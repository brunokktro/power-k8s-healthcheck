# Cost Optimization Checks

## Overview

Cost optimization validation based on:
- [EKS Best Practices - Cost Optimization](https://docs.aws.amazon.com/eks/latest/best-practices/cost-opt.html)

---

## Check: Resource Right-Sizing

**Severity:** MEDIUM

**What to check:**
1. Pods with CPU requests > 2x average usage over 7 days
2. Pods with memory requests > 1.5x peak usage
3. VPA recommendations available but not applied
4. Containers requesting resources but consistently idle

**How to check (EKS with Container Insights):**
```text
get_cloudwatch_metrics:
  cluster_name: <cluster>
  metric_name: cpu_usage_total
  namespace: ContainerInsights
  dimensions: {ClusterName: <cluster>, Namespace: <ns>, PodName: <pod>}
  minutes: 10080  # 7 days
  stat: Average
```

Compare with:
```text
kubectl_get:
  resourceType: pods
  namespace: <ns>
  output: json
```
Extract `.spec.containers[].resources.requests.cpu`

**Finding logic:**
- Request > 3x actual avg usage -> HIGH (significant waste)
- Request > 2x actual avg usage -> MEDIUM
- Request within 1.2-2x -> PASS (healthy headroom)
- Request < actual usage -> HIGH (reliability risk, covered in reliability-checks)

**Recommendation:**
- Use VPA in recommendation mode to get sizing suggestions
- Review resource utilization monthly
- Set requests based on P95 usage + 20% headroom

---

## Check: Idle and Abandoned Resources

**Severity:** MEDIUM

**What to check:**
1. Deployments scaled to 0 for > 7 days
2. PersistentVolumeClaims not mounted to any pod
3. ConfigMaps/Secrets not referenced by any workload
4. Services with no endpoints and no recent traffic
5. Completed Jobs not cleaned up (TTL not set)
6. Failed pods not automatically cleaned

**How to check:**
```text
kubectl_get:
  resourceType: deployments
  allNamespaces: true
  output: json
# Filter: .spec.replicas == 0

kubectl_get:
  resourceType: persistentvolumeclaims
  allNamespaces: true
  output: json
# Cross-reference with pod volume mounts

kubectl_get:
  resourceType: jobs
  allNamespaces: true
  output: json
# Filter: .status.completionTime older than 7 days AND no ttlSecondsAfterFinished
```

**Finding logic:**
- Unattached PVC (especially GP3/io2 volumes) -> MEDIUM (paying for unused storage)
- Deployment at 0 replicas > 30 days -> LOW
- Completed Jobs without TTL cleanup -> LOW
- Services with zero endpoints -> MEDIUM

**Recommendation:**
- Set `ttlSecondsAfterFinished` on Jobs
- Use `kubectl get pvc` cross-referenced with pod mounts to find orphans
- Regularly audit zero-replica deployments

---

## Check: Node Instance Type Optimization

**Severity:** MEDIUM

**What to check:**
1. Nodes consistently under 40% CPU utilization
2. Nodes consistently under 50% memory utilization
3. Mix of instance families for cost efficiency
4. Graviton (ARM) instances considered for compatible workloads
5. Spot instances used for fault-tolerant workloads

**How to check (EKS):**
```text
get_cloudwatch_metrics:
  cluster_name: <cluster>
  metric_name: node_cpu_utilization
  namespace: ContainerInsights
  dimensions: {ClusterName: <cluster>, InstanceId: <id>}
  minutes: 10080
  stat: Average
```

```text
kubectl_get:
  resourceType: nodes
  output: json
```
Check `node.kubernetes.io/instance-type` label.

**Finding logic:**
- Average node CPU < 30% -> HIGH (significantly over-provisioned)
- Average node CPU 30-50% -> MEDIUM
- No Spot instances and workloads are fault-tolerant -> MEDIUM
- No Graviton instances -> LOW (cost opportunity)

---

## Check: Karpenter / Cluster Autoscaler Configuration

**Severity:** MEDIUM

**What to check:**
1. Autoscaler installed and healthy
2. Karpenter: consolidation policy enabled
3. Karpenter: NodePool constraints appropriate (instance families, AZs)
4. CAS: scale-down enabled, appropriate thresholds
5. Unschedulable pods due to insufficient capacity

**How to check:**
```text
kubectl_get:
  resourceType: pods
  namespace: kube-system
  labelSelector: "app.kubernetes.io/name=karpenter"

kubectl_get:
  resourceType: nodepools.karpenter.sh
  output: json

kubectl_get:
  resourceType: pods
  allNamespaces: true
  fieldSelector: "status.phase=Pending"
```

**Finding logic:**
- No autoscaler installed -> MEDIUM
- Karpenter without consolidation -> MEDIUM (missing cost savings)
- Pending pods due to capacity -> HIGH
- CAS with scale-down disabled -> MEDIUM

**Recommendation:**
- Enable Karpenter consolidation (`consolidationPolicy: WhenEmptyOrUnderutilized`)
- Use diverse instance types for better Spot availability
- Set appropriate TTL for empty nodes

---

## Check: Namespace-Level Cost Allocation

**Severity:** LOW

**What to check:**
1. ResourceQuotas defined per namespace
2. Labels for cost allocation (team, project, environment)
3. Namespace mapping to cost centers/teams
4. Kubecost or similar cost visibility tool deployed

**How to check:**
```text
kubectl_get:
  resourceType: resourcequotas
  allNamespaces: true

kubectl_get:
  resourceType: namespaces
  output: json
```
Check for cost-related labels/annotations.

**Finding logic:**
- No ResourceQuotas in shared cluster -> MEDIUM
- No cost allocation labels -> LOW
- No cost visibility tooling -> INFO

---

## Check: Storage Cost Optimization

**Severity:** LOW

**What to check:**
1. StorageClass using GP3 instead of GP2 (better price/performance)
2. Oversized PVCs (requested >> actual usage)
3. Snapshot lifecycle policies for EBS volumes
4. EFS vs EBS selection appropriate for access patterns

**How to check:**
```text
kubectl_get:
  resourceType: storageclasses
  output: json

kubectl_get:
  resourceType: persistentvolumes
  output: json
```

**Finding logic:**
- Using GP2 StorageClass -> MEDIUM (GP3 is cheaper and faster)
- PVC > 100GB without justification -> LOW
- No snapshot policy -> LOW

---

## Check: Data Transfer Costs

**Severity:** MEDIUM

**What to check:**
1. Cross-AZ traffic patterns (pods communicating across AZs)
2. Services using `externalTrafficPolicy: Cluster` (causes extra hops)
3. NAT Gateway usage for pods needing internet (consider VPC endpoints)
4. Inter-region traffic where not needed

**How to check:**
```text
kubectl_get:
  resourceType: services
  allNamespaces: true
  output: json
```
Check `externalTrafficPolicy` field.

For node distribution:
```text
kubectl_get:
  resourceType: nodes
  output: json
```
Check AZ labels vs pod placement patterns.

**Recommendation:**
- Use topology-aware routing (`service.kubernetes.io/topology-mode: Auto`)
- Deploy VPC endpoints for ECR, S3, CloudWatch, STS
- Consider `externalTrafficPolicy: Local` where applicable
