# Scalability Checks

## Overview

Scalability validation based on:
- [EKS Best Practices - Scalability](https://docs.aws.amazon.com/eks/latest/best-practices/scalability.html)
- [Kubernetes Setup Best Practices - Large Clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/)

---

## Check: API Server Load

**Severity:** HIGH (at scale)

**What to check:**
1. API server request latency (P99 > 1s indicates stress)
2. Number of watchers on high-cardinality resources
3. LIST calls without pagination (fetching entire resource lists)
4. Custom controllers with aggressive reconciliation loops
5. Informer resync periods too aggressive (< 30s)

**How to check (EKS):**
```text
get_cloudwatch_logs:
  cluster_name: <cluster>
  resource_type: cluster
  log_type: control-plane
  filter_pattern: "latency"
  minutes: 60

get_cloudwatch_metrics:
  cluster_name: <cluster>
  metric_name: apiserver_request_duration_seconds
  namespace: AWS/EKS
  dimensions: {ClusterName: <cluster>}
```

**Finding logic:**
- API server P99 latency > 5s -> CRITICAL
- API server P99 latency > 1s -> HIGH
- > 1000 active watchers on single resource type -> MEDIUM
- Custom controller reconciling every < 10s -> MEDIUM

**Recommendation:**
- Use informer/watch instead of polling
- Paginate LIST calls (limit=500)
- Increase informer resync periods (5-10 min default)
- Use server-side apply to reduce conflict retries
- Consider multiple smaller clusters if hitting API server limits

---

## Check: etcd Object Count

**Severity:** MEDIUM

**What to check:**
1. Total number of objects in etcd (approaching limits)
2. Namespaces with excessive ConfigMaps/Secrets (> 1000)
3. Completed Jobs not cleaned up (accumulate in etcd)
4. Excessive Events (default retention causes bloat)
5. CRD instances count

**How to check:**
```text
kubectl_get:
  resourceType: configmaps
  allNamespaces: true
# Count total

kubectl_get:
  resourceType: secrets
  allNamespaces: true
# Count total

kubectl_get:
  resourceType: events
  allNamespaces: true
# Count total
```

**Finding logic:**
- Total objects > 100K -> HIGH
- Single namespace > 5000 objects -> MEDIUM
- Uncleaned Jobs > 1000 -> MEDIUM
- Events count > 50K -> LOW (usually ephemeral but indicates churn)

**Recommendation:**
- Set `ttlSecondsAfterFinished` on Jobs
- Use event expiry settings
- Archive and clean old CRD instances
- Consider etcd defragmentation schedule
- For EKS: AWS manages etcd, but high object count still impacts performance

---

## Check: Node Density and Pod Limits

**Severity:** MEDIUM

**What to check:**
1. Pods per node approaching instance limit
2. ENI/IP address limits for instance type
3. Prefix delegation enabled for high density
4. Max pods setting configured correctly

**How to check:**
```text
kubectl_get:
  resourceType: nodes
  output: json
```
Check:
- `.status.allocatable.pods` (max pods allowed)
- `.status.capacity.pods`
- Actual pod count per node

```text
kubectl_get:
  resourceType: pods
  allNamespaces: true
  fieldSelector: "spec.nodeName=<node>"
```
Count pods per node.

**Finding logic:**
- Pod count > 80% of node allocatable -> HIGH
- Pod count > 60% of node allocatable -> MEDIUM
- Prefix delegation not enabled with > 50 pods/node -> MEDIUM
- Adequate headroom -> PASS

**Recommendation (EKS):**
- Enable prefix delegation: `ENABLE_PREFIX_DELEGATION=true`
- Use appropriate instance types for pod density needs
- Max pods formula (secondary IP mode): `(ENIs * (IPs per ENI - 1)) + 2`
- Max pods with prefix delegation: `(ENIs * ((IPs per ENI - 1) * 16)) + 2`

---

## Check: Cluster Services Scaling

**Severity:** HIGH (at scale)

**What to check:**
1. CoreDNS scaled appropriately for cluster size
2. Metrics server resource allocation
3. Karpenter/CAS controller resources
4. Ingress controller scaled for traffic volume
5. Service mesh control plane resources

**How to check:**
```text
kubectl_get:
  resourceType: deployments
  namespace: kube-system
  output: json
```
Check replicas and resource requests for critical controllers.

**Scaling guidelines:**
| Component | Guideline |
|-----------|-----------|
| CoreDNS | 1 replica per 50 nodes OR use NodeLocal DNSCache |
| Metrics Server | Increase memory for > 100 nodes |
| Karpenter | 2 replicas (HA), increase memory for > 1000 pods |
| Ingress Controller | Scale with traffic, not node count |
| VPC CNI (aws-node) | DaemonSet, auto-scales |

**Finding logic:**
- CoreDNS with 2 replicas on > 100 node cluster -> HIGH
- Metrics server OOMKilled events -> HIGH
- Karpenter single replica -> MEDIUM
- Appropriate scaling -> PASS

---

## Check: Workload Architectural Patterns

**Severity:** MEDIUM

**What to check:**
1. Services with > 5000 endpoints (single Service scaling limit)
2. Namespaces with > 10000 pods
3. Single Deployment with > 1000 replicas (consider splitting)
4. EndpointSlices enabled (default since K8s 1.21)

**How to check:**
```text
kubectl_get:
  resourceType: endpoints
  allNamespaces: true
  output: json
# Check endpoint count per Service

kubectl_get:
  resourceType: endpointslices
  allNamespaces: true
```

**Finding logic:**
- Service with > 1000 endpoints -> MEDIUM (consider service splitting)
- Single Deployment > 500 replicas -> LOW (works but consider sharding)
- EndpointSlices not in use -> MEDIUM (older cluster)

---

## Check: Node Group Diversity

**Severity:** MEDIUM

**What to check:**
1. Single instance type across all nodes (blast radius)
2. Single AZ deployment (covered in reliability but impacts scalability)
3. Insufficient instance type diversity for Spot (affects Karpenter/CAS)
4. Mix of compute types (On-Demand for baseline, Spot for burst)

**How to check:**
```text
kubectl_get:
  resourceType: nodes
  output: json
```
Check labels:
- `node.kubernetes.io/instance-type`
- `topology.kubernetes.io/zone`
- `karpenter.sh/capacity-type`

**Finding logic:**
- Single instance type -> MEDIUM
- < 3 instance types with Spot -> HIGH (interruption risk)
- Good diversity (5+ types, 3 AZs) -> PASS

**Recommendation:**
- Use 5+ instance types for Spot workloads
- Mix instance families (m5, m6i, m6g, c5, c6i)
- Spread across 3 AZs
- Use Karpenter with broad NodePool constraints

---

## Check: Resource Quotas at Scale

**Severity:** MEDIUM

**What to check:**
1. Namespace-level ResourceQuotas prevent runaway scaling
2. Cluster-level resource limits (node count, total CPU/memory)
3. Service account token generation rate (OIDC at scale)
4. API priority and fairness configuration

**How to check:**
```text
kubectl_get:
  resourceType: resourcequotas
  allNamespaces: true
  output: json

kubectl_get:
  resourceType: prioritylevelconfigurations
  output: json

kubectl_get:
  resourceType: flowschemas
  output: json
```

**Finding logic:**
- No ResourceQuotas in multi-tenant cluster -> MEDIUM
- Default API priority levels not tuned -> LOW (OK for most clusters)
- Custom FlowSchemas for critical controllers -> PASS (good practice at scale)

**Recommendation:**
- Set ResourceQuotas per namespace
- For > 300 nodes: review APF (API Priority and Fairness) settings
- Ensure critical controllers (Karpenter, ingress) have dedicated FlowSchema
- Monitor API server rejection rate due to priority/fairness
