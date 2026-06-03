# Configuration Checks

## Overview

Configuration hygiene validation based on:
- [Kubernetes Configuration Good Practices](https://kubernetes.io/blog/2025/11/25/configuration-good-practices/)
- [Kubernetes Common Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)
- [Kubernetes Setup Best Practices](https://kubernetes.io/docs/setup/best-practices/)

---

## Check: Semantic Labels

**Severity:** MEDIUM

**What to check:**
1. Workloads using recommended Kubernetes labels:
   - `app.kubernetes.io/name`
   - `app.kubernetes.io/instance`
   - `app.kubernetes.io/version`
   - `app.kubernetes.io/component`
   - `app.kubernetes.io/part-of`
   - `app.kubernetes.io/managed-by`
2. Custom labels for operational purposes (team, environment, cost-center)
3. Consistent labeling scheme across namespaces

**How to check:**
```text
kubectl_get:
  resourceType: deployments
  allNamespaces: true
  output: json
```
Check `.metadata.labels` and `.spec.template.metadata.labels` for standard keys.

**Finding logic:**
- No `app.kubernetes.io/name` label -> MEDIUM
- No version label on any workload -> LOW
- Inconsistent label schemes across namespaces -> LOW
- Proper semantic labels -> PASS

**Recommendation:**
```yaml
metadata:
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: myapp-production
    app.kubernetes.io/version: "1.2.3"
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: platform
    app.kubernetes.io/managed-by: helm
```

---

## Check: Meaningful Annotations

**Severity:** LOW

**What to check:**
1. Resources with `kubernetes.io/description` annotation
2. Deployments with change-cause annotation (rollout history)
3. Contact/owner annotations for operational clarity
4. Links to runbooks or dashboards in annotations

**How to check:**
```text
kubectl_get:
  resourceType: deployments
  allNamespaces: true
  output: json
```
Check `.metadata.annotations`.

**Finding logic:**
- No description annotation -> LOW
- No owner/team annotation -> LOW
- Change-cause annotation present -> PASS (good practice)

**Recommendation:**
```yaml
metadata:
  annotations:
    kubernetes.io/description: "Main API gateway handling external traffic"
    team: platform-engineering
    oncall: "#platform-oncall"
    runbook: "https://wiki.example.com/api-gateway"
```

---

## Check: Latest Stable API Versions

**Severity:** MEDIUM

**What to check:**
1. Resources using beta API versions when stable exists
2. Resources using alpha API versions in production
3. Resources using older API versions when newer stable is available

**How to check:**
```text
kubectl_get:
  resourceType: deployments
  allNamespaces: true
  output: json
# Check apiVersion field

list_api_resources (kubernetes MCP)
# Get current preferred versions
```

**Finding logic:**
- Using v1beta1 when v1 is available -> MEDIUM
- Using alpha API in production -> HIGH
- Using current stable version -> PASS

**Recommendation:** Always use the latest stable API version. Check with `kubectl api-resources` for current preferred versions.

---

## Check: YAML Hygiene

**Severity:** LOW

**What to check:**
1. Resources setting default values explicitly (unnecessary verbosity)
2. Boolean values using non-standard representations (yes/no/on/off instead of true/false)
3. Resources without namespace explicitly set (relies on context default)
4. Multi-resource files grouped logically (related objects together)

**How to check:** This is primarily a manifest-level check. At runtime, verify:
```text
kubectl_get:
  resourceType: pods
  allNamespaces: true
  output: json
```
Check for pods in `default` namespace (often indicates missing namespace in manifest).

**Finding logic:**
- Workloads in `default` namespace -> MEDIUM (indicates no namespace governance)
- Non-system pods in `kube-system` -> HIGH (namespace misuse)

**Recommendation:**
- Never deploy application workloads to `default` namespace
- Use YAML, not JSON, for manifests
- Use `true`/`false` for booleans, never `yes`/`no`
- Group related objects in single files

---

## Check: Namespace Isolation

**Severity:** MEDIUM

**What to check:**
1. Proper namespace separation by environment (dev/staging/prod)
2. Proper namespace separation by team or service
3. ResourceQuotas per namespace
4. LimitRanges per namespace
5. Excessive number of resources in a single namespace

**How to check:**
```text
kubectl_get:
  resourceType: namespaces
  output: json

kubectl_get:
  resourceType: resourcequotas
  allNamespaces: true

kubectl_get:
  resourceType: limitranges
  allNamespaces: true
```

**Finding logic:**
- All workloads in one namespace -> HIGH
- Namespace without ResourceQuota in shared cluster -> MEDIUM
- Namespace without LimitRange -> LOW
- Proper isolation -> PASS

**Recommendation:**
- One namespace per service or team
- Always set ResourceQuotas in shared clusters
- Use LimitRanges as defaults for pods without explicit resources

---

## Check: Version Control and GitOps

**Severity:** MEDIUM

**What to check:**
1. Resources with `managed-by: helm` or ArgoCD/Flux annotations (GitOps)
2. Resources without any management annotation (likely manually applied)
3. Config drift indicators (resources modified since last sync)
4. Helm releases in failed or pending state

**How to check:**
```text
kubectl_get:
  resourceType: deployments
  allNamespaces: true
  output: json
```
Check for:
- `app.kubernetes.io/managed-by: Helm`
- `argocd.argoproj.io/managed-by`
- `fluxcd.io/sync-checksum`

```text
kubectl_generic:
  command: helm
  subCommand: list
  flags: {all-namespaces: "", output: json}
```

**Finding logic:**
- > 50% of resources without management tool -> MEDIUM
- Helm releases in failed state -> HIGH
- ArgoCD apps out of sync -> MEDIUM
- All resources managed by GitOps -> PASS

---

## Check: ConfigMap and Secret Best Practices

**Severity:** MEDIUM

**What to check:**
1. Secrets referenced as env vars (appear in logs) vs mounted as volumes
2. ConfigMaps with large data (> 1MB approaching etcd limit)
3. Immutable ConfigMaps/Secrets used where appropriate
4. ConfigMap changes triggering rollouts (checksum annotation pattern)

**How to check:**
```text
kubectl_get:
  resourceType: pods
  allNamespaces: true
  output: json
```
Check `.spec.containers[].env[].valueFrom.secretKeyRef` vs `.spec.volumes[].secret`.

```text
kubectl_get:
  resourceType: configmaps
  allNamespaces: true
  output: json
```
Check data size and `immutable` field.

**Finding logic:**
- Secrets in env vars -> MEDIUM (log exposure risk)
- ConfigMap > 500KB -> MEDIUM (approaching limits)
- No immutable ConfigMaps for static config -> LOW
- Proper volume mount usage -> PASS

---

## Check: Controller Usage (No Naked Pods)

**Severity:** HIGH

**What to check:**
1. Pods without ownerReferences (naked pods)
2. Use Deployments for stateless apps that should always run
3. Use StatefulSets for stateful workloads
4. Use Jobs/CronJobs for batch tasks
5. Use DaemonSets for node-level services

**How to check:**
```text
kubectl_get:
  resourceType: pods
  allNamespaces: true
  output: json
```
Filter: `.metadata.ownerReferences` empty or null (excluding system namespaces).

**Finding logic:**
- Naked pod in production -> HIGH
- Long-running pod as Job (not Deployment) -> MEDIUM
- Appropriate controller usage -> PASS

**Recommendation:** From K8s Configuration Good Practices:
- "A Deployment, which both creates a ReplicaSet and specifies a strategy to replace Pods, is almost always preferable to creating Pods directly"
- "A Job is perfect when you need something to run once and then stop"
