# Image Build Checks

## Overview

Container image build best practices validation based on:
- [AWS Well-Architected Container Build Lens](https://docs.aws.amazon.com/wellarchitected/latest/container-build-lens/container-build-lens.html)
- [EKS Best Practices - Image Security](https://docs.aws.amazon.com/eks/latest/best-practices/security.html)

**Note:** These checks are partially runtime-verifiable (image tags, pull policies, registry sources) and partially guidance-based (build process, Dockerfile practices). The runtime checks use MCP tools; the build practices are advisory.

---

## Check: Image Tags and Immutability

**Severity:** HIGH

**What to check:**
1. Containers using `:latest` tag
2. Containers using mutable tags (no SHA digest pinning)
3. Containers with no tag specified (implies `:latest`)
4. Image pull policy not set to `Always` for mutable tags

**How to check:**
```text
kubectl_get:
  resourceType: pods
  allNamespaces: true
  output: json
```
Filter: `.spec.containers[].image` patterns:
- Ends with `:latest` -> HIGH
- Has no `:` after repository name -> HIGH (implicit latest)
- Contains `@sha256:` -> PASS (immutable)
- Has specific tag (e.g., `:1.2.3`) -> LOW (mutable but acceptable)

**Finding logic:**
- Using `:latest` in production -> HIGH
- Using mutable tag without `imagePullPolicy: Always` -> MEDIUM
- Using SHA digest -> PASS
- Using semantic version tag -> LOW (acceptable with proper CI/CD)

**Recommendation:**
- Use SHA digests for production: `image: myapp@sha256:abc123...`
- At minimum, use semantic version tags: `image: myapp:1.2.3`
- Set `imagePullPolicy: Always` for any mutable tag
- Enable tag immutability in ECR

---

## Check: Base Image Selection

**Severity:** MEDIUM (advisory)

**What to check:**
1. Containers using full OS base images (ubuntu, centos, debian)
2. Containers using distroless or minimal base images
3. Known vulnerable base images
4. Base image age (old images accumulate CVEs)

**How to check (runtime indicators):**
```text
exec_in_pod:
  name: <pod>
  namespace: <ns>
  command: ["cat", "/etc/os-release"]
```
Note: Only works if the image has a shell (distroless won't have one).

```text
kubectl_get:
  resourceType: pods
  allNamespaces: true
  output: json
```
Check image names for indicators: `ubuntu`, `centos`, `alpine`, `distroless`, `scratch`.

**Finding logic:**
- Base image is full OS (ubuntu:22.04, centos:7) -> MEDIUM
- Base image is minimal (alpine, distroless) -> PASS
- Base image is `scratch` -> PASS (minimal attack surface)
- Cannot determine base -> INFO

**Recommendation (Container Build Lens):**
- Use minimal base images (distroless, alpine, or scratch)
- Regularly update base images (monthly cadence minimum)
- Pin base image version with digest
- Consider multi-arch base images for Graviton compatibility

---

## Check: Multi-Stage Build Indicators

**Severity:** LOW (advisory)

**What to check:**
1. Image size (large images suggest no multi-stage build)
2. Build tools present in runtime image (compilers, package managers beyond what's needed)
3. Dev dependencies in production image

**How to check (runtime indicators):**
```text
exec_in_pod:
  name: <pod>
  namespace: <ns>
  command: ["which", "gcc"]
# If gcc/make/npm/pip exist in production image -> likely no multi-stage

kubectl_describe:
  resourceType: pods
  name: <pod>
  namespace: <ns>
```
Check image size from container runtime info.

**Finding logic:**
- Build tools found in running container -> MEDIUM
- Image > 1GB -> MEDIUM (likely unoptimized)
- Image < 100MB for Go/Rust app -> PASS
- Image < 500MB for Java/Node app -> acceptable

**Recommendation (Container Build Lens):**
```dockerfile
# Build stage
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server

# Runtime stage
FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app/server /server
USER 65534
ENTRYPOINT ["/server"]
```

---

## Check: Image Scanning and Vulnerabilities

**Severity:** HIGH

**What to check:**
1. ECR image scanning enabled (basic or enhanced)
2. Images with CRITICAL/HIGH CVEs in use
3. Admission controller enforcing scan results (e.g., Kyverno, OPA Gatekeeper)
4. Scan results freshness (scanned within last 30 days)

**How to check (EKS/ECR - advisory):**
This requires ECR API access. At runtime, check for admission controllers:
```text
kubectl_get:
  resourceType: validatingwebhookconfigurations
  output: json
```
Look for image scanning webhooks (Kyverno, OPA, Prisma Cloud, etc.)

```text
kubectl_get:
  resourceType: clusterpolicies.kyverno.io
  output: json
# Look for image verification policies
```

**Finding logic:**
- No image scanning admission controller -> MEDIUM
- Admission controller present but in audit mode -> LOW
- Admission controller in enforce mode -> PASS

**Recommendation:**
- Enable ECR enhanced scanning (Inspector integration)
- Deploy admission controller to block images with CRITICAL CVEs
- Integrate scanning into CI/CD pipeline (shift-left)
- Re-scan images periodically (CVEs discovered post-deploy)

---

## Check: Registry Configuration

**Severity:** MEDIUM

**What to check:**
1. Pulling from public registries (docker.io) in production
2. ECR pull-through cache configured for public dependencies
3. Image pull secrets configured for private registries
4. Cross-region ECR replication for multi-region deployments

**How to check:**
```text
kubectl_get:
  resourceType: pods
  allNamespaces: true
  output: json
```
Check image prefixes:
- `docker.io/` or no prefix -> public registry
- `<account>.dkr.ecr.<region>.amazonaws.com/` -> private ECR
- `gcr.io/`, `quay.io/` -> third-party registries

**Finding logic:**
- Production pods pulling from docker.io directly -> MEDIUM (rate limits, availability)
- No pull-through cache for public dependencies -> LOW
- Using private ECR -> PASS
- Missing imagePullSecrets for private registry -> HIGH (pods will fail)

**Recommendation:**
- Use ECR pull-through cache for public images
- Mirror critical third-party images to private ECR
- Enable cross-region replication for DR
- Never store credentials in Dockerfiles or ConfigMaps

---

## Check: Image Signing and Provenance

**Severity:** LOW (advisory, becoming industry standard)

**What to check:**
1. Images signed with Sigstore/cosign
2. SBOM (Software Bill of Materials) attached to images
3. Admission controller verifying signatures
4. Provenance attestations (SLSA level)

**How to check (runtime):**
```text
kubectl_get:
  resourceType: clusterpolicies.kyverno.io
  output: json
# Look for image verification rules (verifyImages)
```

**Finding logic:**
- No signature verification -> LOW (recommended but not yet universal)
- Kyverno/OPA image verification in audit mode -> PASS
- Full signature verification in enforce mode -> PASS (excellent)

**Recommendation:**
- Sign images with cosign in CI/CD
- Generate and attach SBOM (syft, trivy)
- Deploy admission controller with signature verification
- Target SLSA Level 2+ for critical workloads

---

## Check: Layer Optimization

**Severity:** LOW (advisory)

**What to check:**
1. Image layer count (excessive layers = larger image, slower pulls)
2. Unnecessary files in image (docs, tests, source code)
3. Package manager cache not cleaned
4. Secrets baked into image layers

**How to check (runtime indicators):**
```text
exec_in_pod:
  name: <pod>
  namespace: <ns>
  command: ["find", "/", "-name", "*.test.*", "-o", "-name", "*.md"]
```
Note: Limited by container filesystem access.

**Recommendation (Container Build Lens):**
- Combine related RUN commands to reduce layers
- Clean package manager caches in the same layer: `RUN apt-get install -y pkg && rm -rf /var/lib/apt/lists/*`
- Use `.dockerignore` to exclude unnecessary files
- Never `COPY . .` without a proper `.dockerignore`
- Never store secrets in Dockerfile (use build secrets or runtime injection)

---

## Check: Non-Root User

**Severity:** HIGH

**What to check:**
1. Containers running as root (UID 0)
2. Dockerfile missing `USER` instruction
3. Pod securityContext not setting `runAsNonRoot: true`
4. Image built with root as default user

**How to check:**
```text
kubectl_get:
  resourceType: pods
  allNamespaces: true
  output: json
```
Check:
- `.spec.securityContext.runAsNonRoot` != true
- `.spec.containers[].securityContext.runAsUser` == 0 or not set

```text
exec_in_pod:
  name: <pod>
  namespace: <ns>
  command: ["id"]
```
Check if running as uid=0.

**Finding logic:**
- Running as root without explicit override -> HIGH
- Running as root with documented justification -> MEDIUM
- Running as non-root user -> PASS

**Recommendation:**
```dockerfile
# In Dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# In Pod spec (defense in depth)
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
```
