# Networking Checks

## Overview

Networking validation based on:
- [EKS Best Practices - Networking](https://docs.aws.amazon.com/eks/latest/best-practices/networking.html)
- [Kubernetes Configuration Good Practices - Service Configuration](https://kubernetes.io/blog/2025/11/25/configuration-good-practices/)

---

## Check: VPC CNI Configuration (EKS)

**Severity:** HIGH

**What to check:**
1. VPC CNI add-on version (should be latest compatible)
2. WARM_ENI_TARGET / WARM_IP_TARGET / MINIMUM_IP_TARGET settings
3. Prefix delegation mode enabled for high pod density
4. Custom networking enabled if using secondary CIDRs
5. ENABLE_POD_ENI for security groups for pods

**How to check (EKS):**
```text
list_k8s_resources:
  cluster_name: <cluster>
  kind: DaemonSet
  api_version: apps/v1
  namespace: kube-system
  label_selector: "k8s-app=aws-node"
```
Check environment variables in the aws-node DaemonSet.

**Finding logic:**
- VPC CNI > 2 versions behind -> HIGH
- Default WARM_ENI_TARGET without tuning on large clusters -> MEDIUM
- Prefix mode not enabled with > 100 pods/node -> MEDIUM
- Using custom networking without documenting rationale -> INFO

**Recommendation:**
- Keep VPC CNI up to date
- Enable prefix delegation for high pod density (`ENABLE_PREFIX_DELEGATION=true`)
- Tune warm pool settings based on workload patterns

---

## Check: IP Address Exhaustion

**Severity:** CRITICAL

**What to check:**
1. Available IPs in each subnet used by nodes
2. Subnet utilization percentage
3. WARM pool consuming IPs unnecessarily
4. IPv6 cluster consideration for large deployments

**How to check (EKS):**
```text
get_eks_vpc_config:
  cluster_name: <cluster>
```
Cross-reference subnet CIDRs with node count and pods per node.

**Finding logic:**
- Subnet < 20% available IPs -> CRITICAL
- Subnet < 40% available IPs -> HIGH
- No plan for IP growth -> MEDIUM

**Recommendation:**
- Add secondary CIDRs (100.64.0.0/10)
- Enable prefix delegation
- Consider IPv6 for new clusters
- Use custom networking to separate pod and node IPs

---

## Check: DNS Configuration

**Severity:** MEDIUM

**What to check:**
1. CoreDNS replicas (should be >= 2, scale with cluster size)
2. CoreDNS resource requests adequate for cluster size
3. ndots configuration in pods (default 5 causes excessive DNS lookups)
4. DNS policy set appropriately

**How to check:**
```text
kubectl_get:
  resourceType: deployments
  namespace: kube-system
  labelSelector: "k8s-app=kube-dns"
  output: json
```

**Finding logic:**
- CoreDNS replicas < 2 -> HIGH
- CoreDNS without HPA on clusters > 50 nodes -> MEDIUM
- Pods with default ndots=5 causing performance issues -> LOW

**Recommendation:**
- Scale CoreDNS with cluster: ~1 replica per 50 nodes
- Consider `dnsConfig.options: [{name: ndots, value: "2"}]` for external-heavy workloads
- Use NodeLocal DNSCache for large clusters

---

## Check: Avoid hostPort and hostNetwork

**Severity:** MEDIUM

**What to check:**
1. Pods using `hostPort` (limits scheduling, causes port conflicts)
2. Pods using `hostNetwork: true` (bypasses network isolation)
3. Exceptions: CNI plugins, kube-proxy, node-level monitoring (legitimate uses)

**How to check:**
```text
kubectl_get:
  resourceType: pods
  allNamespaces: true
  output: json
```
Filter: `.spec.hostNetwork == true` OR `.spec.containers[].ports[].hostPort != null`

**Finding logic:**
- Application pod with hostNetwork -> HIGH
- Application pod with hostPort -> MEDIUM
- System pod (kube-system) with hostNetwork -> PASS (expected)

**Recommendation:** Use Services (ClusterIP, NodePort, LoadBalancer) or Ingress instead. hostPort/hostNetwork should be reserved for system-level daemons only.

---

## Check: Service Configuration

**Severity:** MEDIUM

**What to check:**
1. Services created BEFORE their backend workloads (env var injection)
2. Services without selectors (headless or external)
3. LoadBalancer services without annotations for internal/external
4. ExternalTrafficPolicy considerations (Local vs Cluster)
5. Services with no matching endpoints

**How to check:**
```text
kubectl_get:
  resourceType: services
  allNamespaces: true
  output: json

kubectl_get:
  resourceType: endpoints
  allNamespaces: true
  output: json
```

**Finding logic:**
- Service with empty endpoints (not headless) -> HIGH
- LoadBalancer without scheme annotation -> MEDIUM
- ExternalTrafficPolicy: Cluster on latency-sensitive services -> LOW

**Recommendation:**
- Always verify Services have healthy endpoints
- Set `service.beta.kubernetes.io/aws-load-balancer-scheme: internal` for private services
- Use `externalTrafficPolicy: Local` to preserve source IP when needed

---

## Check: Ingress Controller Configuration

**Severity:** MEDIUM

**What to check:**
1. Ingress controller deployed and healthy
2. Multiple replicas for HA
3. TLS termination configured
4. Rate limiting / WAF integration (EKS: AWS WAF with ALB)
5. Ingress resources without TLS

**How to check:**
```text
kubectl_get:
  resourceType: ingresses
  allNamespaces: true
  output: json

kubectl_get:
  resourceType: pods
  namespace: <ingress-namespace>
  labelSelector: "app.kubernetes.io/name=ingress-nginx"
```

**Finding logic:**
- No ingress controller installed -> INFO (might use other patterns)
- Ingress without TLS -> HIGH
- Single replica ingress controller -> MEDIUM
- No rate limiting -> LOW

---

## Check: Security Groups for Pods (EKS)

**Severity:** LOW

**What to check:**
1. SecurityGroupPolicy resources defined
2. Pods that need fine-grained network access using SGP
3. Trunk ENI mode enabled on nodes

**How to check:**
```text
kubectl_get:
  resourceType: securitygrouppolicies
  allNamespaces: true
```

**Finding logic:**
- Multi-tenant cluster without SGP -> MEDIUM
- SGP defined but not matching pods -> LOW
- Properly configured -> PASS

---

## Check: Service Mesh Health

**Severity:** LOW (only if mesh is deployed)

**What to check:**
1. Istio/Linkerd sidecar injection enabled
2. Pods without sidecar in meshed namespaces
3. mTLS mode (strict vs permissive)
4. Control plane health (istiod/linkerd-destination)

**How to check:**
```text
kubectl_get:
  resourceType: pods
  allNamespaces: true
  output: json
```
Filter for sidecar containers (istio-proxy, linkerd-proxy).

Check namespace labels for injection:
- `istio-injection: enabled`
- `linkerd.io/inject: enabled`

**Finding logic:**
- Mesh installed but mTLS in permissive mode -> MEDIUM
- Pods in meshed namespace without sidecar -> MEDIUM
- Mesh control plane unhealthy -> HIGH
