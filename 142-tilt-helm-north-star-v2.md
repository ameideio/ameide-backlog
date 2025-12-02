> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# Tilt + Helm North Star v2

**STATUS**: Partially Implemented (Infrastructure/Application Separation Complete)
**Last Updated**: 2025-08-29

This proposal formalizes a clean separation of concerns for local development:

## Current State (Implemented)

### Infrastructure/Application Separation ✅
- **Infrastructure**: Deployed exclusively via Helmfile (`helmfile -e local sync`)
- **Applications**: Deployed via Tilt for fast development iteration
- **No Conflicts**: Tilt no longer deploys infrastructure, preventing resource ownership conflicts

### What Was Fixed
- Removed all infrastructure `helm_resource()` calls from Tiltfile
- Enhanced infrastructure health checks to verify all components
- Fixed certificate deletion issue caused by dual Helm ownership
- Added `ops-deploy-infrastructure` recovery tool for missing infrastructure

## Next Phase (To Be Implemented)

### Digest-Based Deployments
- Tilt handles file watching, rebuilding images, and publishing them to the local registry
- Helm remains authoritative for all Kubernetes resources and image selection
- Deployments use immutable image digests, never ambiguous floating tags like `:dev`

## Goals

- Deterministic, reproducible deploys during local dev.
- Fast inner loop (Tilt) without coupling to Kubernetes YAML templating.
- Helm as the single source of truth for manifests and values (even in local).
- Remove ambiguity and drift caused by multiple `:dev` tags.

## Summary

- Tilt builds an image and pushes it to the local registry using a content-based tag (`$EXPECTED_REF`).
- Tilt resolves the pushed image digest and writes a small Helm values file per service:
  - `image.graph: <registry>/<service>`
  - `image.digest: sha256:...`
- Tilt calls `helm_resource()` with base values + environment values + the Tilt‑generated “tilt-values” file.
- Helm templates the Deployment using the digest (digest-first), then applies it to the cluster.

## Prerequisites

### Current Setup (Working)
- k3d cluster with integrated registry exposed via Envoy at `docker.io/ameide`
- Infrastructure deployed via Helmfile before starting Tilt
- Separation of concerns: Helmfile owns infrastructure, Tilt owns applications

### For Digest Implementation (Future)
- Local registry reachable by both host and cluster at `docker.io/ameide`
  - Containerd mirror + DNS wiring now ships automatically via the `registry-mirror` Helm component (see backlog 337)
- Tilt and Docker available on the dev machine
- `crane` (gcr.io/go-containerregistry/crane) for resolving digests

## Helm Chart Requirements

Deployments must prefer digest when provided:

```yaml
{{- $repo := .Values.image.graph -}}
{{- if .Values.image.digest }}
image: "{{ $repo }}@{{ .Values.image.digest }}"
{{- else if .Values.image.tag }}
image: "{{ $repo }}:{{ .Values.image.tag }}"
{{- else }}
image: "{{ $repo }}"
{{- end }}
```

- Inference chart already supports digest-first:
  - `infra/kubernetes/charts/platform/inference/templates/deployment.yaml:1`
- Update any tag‑only charts (e.g., inference_gateway) to digest-first using the snippet above:
  - `infra/kubernetes/charts/platform/inference_gateway/templates/deployment.yaml:1`

## Tiltfile Pattern (per service)

### Build + Publish + Write Digest Values

**IMPORTANT**: Use `local_resource` instead of `custom_build` for reliable builds. The `custom_build` approach may not trigger if nothing directly references the image name.

```python
# Build inference image and generate digest file
local_resource(
    'inference-build',
    cmd='bash -c "set -euo pipefail; \
    REGISTRY=docker.io/ameide; \
    REF=\\"\\$REGISTRY/inference:tilt-\\$(date +%s)\\"; \
    docker build -f services/inference/Dockerfile -t \\"\\$REF\\" .; \
    docker push \\"\\$REF\\"; \
    DIGEST=\\$(crane digest --insecure \\"\\$REF\\"); \
    mkdir -p infra/kubernetes/environments/local/tilt-values; \
    {
      echo \\"image:\\"; \
      echo \\"  graph: \\$REGISTRY/inference\\"; \
      echo \\"  digest: \\$DIGEST\\"; \
    } > infra/kubernetes/environments/local/tilt-values/inference.values.yaml; \
    echo \\"Built and pushed \\$REF with digest \\$DIGEST\\""',
    deps=['services/inference', 'packages/ameide_core_inference_agents'],
    labels=['build.inference']
)
```

Notes:
- `local_resource` ensures the build always executes when dependencies change
- **Must use `bash -c`** instead of `sh` for `pipefail` support
- Use escaped quotes and variables for proper shell execution
- Use echo commands instead of heredocs for simpler escaping
- Generate unique tags with timestamp to avoid conflicts
- Echo status for debugging

### Deploy with Helm + Include Tilt Values

```python
# Sentinel resource that signals digest file is ready
local_resource(
    'inference-digest-ready',
    cmd='test -f infra/kubernetes/environments/local/tilt-values/inference.values.yaml && echo "Digest ready" || echo "Waiting for digest"',
    deps=['infra/kubernetes/environments/local/tilt-values/inference.values.yaml'],
    resource_deps=[]  # CRITICAL: No deps to avoid dependency cycles
)

# Deploy using helm_resource - NO image linking, only digest file
helm_resource(
    'inference',
    'infra/kubernetes/charts/platform/inference',
    namespace='ameide',
    flags=[
        '--values=infra/kubernetes/values/platform/inference.yaml',
        '--values=infra/kubernetes/environments/local/platform/inference.yaml',
        '--values=infra/kubernetes/environments/local/tilt-values/inference.values.yaml',
        '--wait',
        '--atomic',
    ],
    deps=[
        'infra/kubernetes/charts/platform/inference',
        'infra/kubernetes/values/platform/inference.yaml',
        'infra/kubernetes/environments/local/platform/inference.yaml',
        'infra/kubernetes/environments/local/tilt-values/inference.values.yaml',
    ],
    image_keys=[],  # CRITICAL: Disable automatic image injection
    labels=['inference'],
    resource_deps=['infra-health-check', 'inference-digest-ready']
)
```

**Critical settings:**
- `image_keys=[]` - Prevents helm_resource from injecting `--set image.*` flags
- NO `image_deps` - Avoid linking builds to deploys which triggers name mangling
- Sentinel resource without `resource_deps` to avoid circular dependencies
- `--wait --atomic` flags for reliable deployments

### For Services Without Digest Support

- Preferred: patch the chart to support digest-first (see snippet above).
- Interim: write `image.tag` in the Tilt-generated file instead of `image.digest`.

```python
# Interim for a tag-only chart (not recommended long-term)
custom_build(
    'docker.io/ameide/inference_gateway',
'''docker build -f services/inference_gateway/Dockerfile.release -t $EXPECTED_REF services/inference_gateway && \
    docker push $EXPECTED_REF && \
    TAG=${EXPECTED_REF##*:} && \
    mkdir -p infra/kubernetes/environments/local/tilt-values && \
    cat > infra/kubernetes/environments/local/tilt-values/inference_gateway.values.yaml <<EOF
image:
  graph: docker.io/ameide/inference_gateway
  tag: $TAG
EOF
    ''',
    deps=['services/inference_gateway'],
    skips_local_docker=False,
)
```

## Directory Layout

- Base values: `infra/kubernetes/values/platform/<service>.yaml`
- Env overrides: `infra/kubernetes/environments/local/platform/<service>.yaml`
- Tilt‑generated overrides: `infra/kubernetes/environments/local/tilt-values/<service>.values.yaml`

## Migration Steps

- Add digest-first support to any tag‑only charts (e.g., inference‑gateway) using the template above.
- Update Tiltfile for each app service:
  - Add custom_build with build → push → digest → write tilt-values.
  - Update `helm_resource` to include the tilt-values file and remove any `--set=image.graph=…` overrides (tilt-values now sets repo+digest).
- Keep infrastructure Helmfile flows as-is for non-local environments.

## Verification

### Current Setup Verification
- Check infrastructure is deployed:
  ```bash
  helm list -n ameide | grep -E "(gateway|cert-manager|redis|postgres)"
  kubectl get gateway ameide -n ameide
  kubectl get secret ameide-wildcard-tls -n ameide
  ```
- Verify HTTPS access:
  ```bash
  curl -k https://www.dev.ameide.io/
  curl -k https://platform.dev.ameide.io/
  ```
- Check Tilt is not deploying infrastructure:
  ```bash
  grep "helm_resource" Tiltfile | grep -v "^#" | wc -l  # Should be 0 (except load statement)
  ```

### Future Digest Verification
- Confirm DNS + registry wiring before builds:
  ```bash
  scripts/infra/helm-ops.sh preflight dns docker.io
  ```
- After a build, confirm Tilt generated values:
  - `ls -l infra/kubernetes/environments/local/tilt-values/`
  - `cat infra/kubernetes/environments/local/tilt-values/inference.values.yaml`
- Confirm Deployment uses digest:
  - `kubectl -n ameide get deploy inference -o jsonpath='{.spec.template.spec.containers[0].image}'`
  - Expect: `docker.io/ameide/inference@sha256:…`

## Configuration

### Timeout Settings

```python
# At the top of your Tiltfile
update_settings(
    k8s_upsert_timeout_secs=600,     # 10 minutes for complex Helm deployments
    suppress_unused_image_warnings='*'  # optional, keeps UI quiet for local-only builds
)
```

The default 30-second timeout is insufficient when using `helm --wait` for complex deployments. We found 600 seconds (10 minutes) works reliably for all services.

## Current Implementation Details

### Infrastructure Components (Helmfile-managed)
```yaml
# All deployed via: helmfile -e local sync
- PostgreSQL clusters (CloudNativePG operator)
- Kafka cluster (Strimzi operator)
- Redis, MinIO, Neo4j, Keycloak, Temporal
- Prometheus, Grafana, Loki, Tempo (observability)
- Envoy Gateway and Gateway configuration
- Cert-Manager with TLS certificates
```

### Application Services (Tilt-managed)
```python
# Pattern used in current Tiltfile:
docker_build(
    'docker.io/ameide/service-name',
    '.',
    dockerfile='services/service_name/Dockerfile',
)

yaml = helm(
    'infra/kubernetes/charts/platform/service-name',
    name='service-name',
    namespace='ameide',
    values=['...'],
    set=['image.graph=docker.io/ameide/service-name'],
)
k8s_yaml(yaml, allow_duplicates=True)

k8s_resource('service-name',
    labels=['app.category'],
    resource_deps=['infra-health-check']
)
```

## Troubleshooting

- HTTP response to HTTPS client on registry:
  - Envoy exposes `docker.io/ameide`. If using HTTPS locally, keep the listener in sync; otherwise rely on the Helm-managed containerd mirror for HTTP.
- ErrImagePull (connection refused):
  - Registry not reachable from cluster network; ensure it's started and routed.
- Helm error "Kubernetes cluster unreachable": 
  - Check kubeconfig/context Tilt/Helm are using.
- No tilt-values file:
  - Ensure the local_resource ran and the path matches the helm_resource flag.
- Helm deployment timeout:
  - Increase `k8s_upsert_timeout_secs` in Tiltfile (default 30s is often too short)
- Dependency cycle detected:
  - Check sentinel resources don't have `resource_deps` pointing back to helm_resource
- Build not triggering:
  - Use `local_resource` instead of `custom_build` for guaranteed execution
- Image name mangling (docker.io_5000_):
  - Ensure `image_keys=[]` is set on helm_resource to disable injection
- Shell syntax errors ("set: Illegal option -o pipefail"):
  - Use `bash -c` instead of `sh` as pipefail is bash-specific
- Bazel proto path errors ("glob pattern 'proto/**/*.proto' didn't match"):
  - Update BUILD.bazel to use `src/proto/**/*.proto` if protos are in src directory

## Practical Notes From Implementation

- Avoid Tilt image rewriting when using digests
  - Set `image_keys=[]` on every `helm_resource` that consumes tilt‑values (this explicitly disables the extension’s image injection and prevents `--set image.*` from being added to the Helm command).
  - Do not use `image_deps` with `helm_resource` in this pattern (it links builds to deploys and triggers image injection/mangling).
  - Using automatic image linking can rewrite names (e.g., `docker.io_5000_service`) and override your digest values. Keep Helm in control via tilt‑values.
  - Use `resource_deps` (or a dummy `local_resource` sentinel that depends on the generated values file) to order Helm after the build without exposing image maps.

### Key Implementation Lessons

1. **Use `local_resource` not `custom_build`**:
   - `custom_build` may not trigger if the image name isn't directly referenced
   - `local_resource` guarantees execution when dependencies change
   - Clean names (like 'inference') work better than suffixed names ('inference-img')

2. **Prevent helm_resource image injection completely**:
   - Set `image_keys=[]` on every helm_resource
   - Never use `image_deps` - it triggers name mangling
   - The extension is aggressive about injecting `--set image.*` flags

3. **Avoid dependency cycles**:
   - Sentinel resources must have `resource_deps=[]`
   - Use sentinels to order Helm after builds without creating cycles

4. **Registry routing**:
   - Envoy now fronts the k3d registry at `docker.io/ameide` for both host pushes and in-cluster pulls.
   - The Helm-managed `registry-mirror` DaemonSet keeps containerd mirrors + trust settings in sync automatically.
   - Always populate tilt-values with the Envoy host (`docker.io/ameide/...`) to avoid drift.

### Complete Working Example

```python
# 1. Build and push with local_resource
local_resource(
    'inference-build',
    cmd='bash -c "set -euo pipefail; \
    REGISTRY=docker.io/ameide; \
    REF=\\"\\$REGISTRY/inference:tilt-\\$(date +%s)\\"; \
    docker build -f services/inference/Dockerfile -t \\"\\$REF\\" .; \
    docker push \\"\\$REF\\"; \
    DIGEST=\\$(crane digest --insecure \\"\\$REF\\"); \
    mkdir -p infra/kubernetes/environments/local/tilt-values; \
    {
      echo \\"image:\\"; \
      echo \\"  graph: \\$REGISTRY/inference\\"; \
      echo \\"  digest: \\$DIGEST\\"; \
    } > infra/kubernetes/environments/local/tilt-values/inference.values.yaml; \
    echo \\"Built and pushed \\$REF with digest \\$DIGEST\\""',
    deps=['services/inference', 'packages/ameide_core_inference_agents'],
    labels=['build.inference']
)

# 2. Sentinel to signal digest readiness
local_resource(
    'inference-digest-ready',
    cmd='test -f infra/kubernetes/environments/local/tilt-values/inference.values.yaml && echo "Ready"',
    deps=['infra/kubernetes/environments/local/tilt-values/inference.values.yaml'],
    resource_deps=[]  # No deps to avoid cycles
)

# 3. Deploy with Helm - clean, no image injection
helm_resource(
    'inference',
    'infra/kubernetes/charts/platform/inference',
    namespace='ameide',
    flags=[
        '--values=infra/kubernetes/values/platform/inference.yaml',
        '--values=infra/kubernetes/environments/local/platform/inference.yaml',
        '--values=infra/kubernetes/environments/local/tilt-values/inference.values.yaml',
        '--wait', '--atomic',
    ],
    deps=[
        'infra/kubernetes/charts/platform/inference',
        'infra/kubernetes/environments/local/tilt-values/inference.values.yaml',
    ],
    image_keys=[],  # Critical: disable image injection
    resource_deps=['infra-health-check', 'inference-digest-ready'],
)
```

- Registry routing (local)
  - Host and cluster traffic use `docker.io/ameide`
  - The `registry-mirror` DaemonSet configures containerd for HTTP automatically; no manual mirror script required
  - Ensure Envoy/TLS listeners stay aligned if you opt into HTTPS locally

- Generated files hygiene
  - Add `infra/kubernetes/environments/local/tilt-values/` to `.gitignore` so per‑developer digests don’t get committed.

## Common Pitfalls to Avoid

1. **DON'T use custom_build with clean names** - It won't trigger unless something references the image
2. **DON'T add `-img` suffixes** - Keep names clean and simple
3. **DON'T forget `image_keys=[]`** - The helm_resource extension is aggressive about injection
4. **DON'T create dependency cycles** - Sentinels must not depend on the resources they guard
5. **DON'T bypass the Envoy registry host** - Always use `docker.io/ameide` in tilt-values
6. **DON'T skip the timeout increase** - Helm --wait needs more than 30 seconds

## Migration Checklist

### Phase 1: Infrastructure Separation (Completed 2025-08-29)
- [x] Removed all infrastructure `helm_resource()` from Tiltfile
- [x] Infrastructure deployed exclusively via Helmfile
- [x] Enhanced health checks for all infrastructure components
- [x] Fixed certificate ownership conflicts
- [x] Application services use `docker_build()` + `helm()` + `k8s_yaml()`

### Phase 2: Digest-Based Deployments (To Do)
- [ ] Charts render digest‑first (inference, inference‑gateway, others as needed)
- [ ] Tiltfile per‑service using `local_resource`: build → push → `crane digest` → write tilt‑values
- [ ] Helm consumes: base values + env values + tilt‑values
- [ ] Disabled image injection: `image_keys=[]` on helm_resource; no `image_deps`
- [ ] Use sentinel `local_resource` for ordering without creating cycles
- [ ] `.gitignore` excludes `infra/kubernetes/environments/local/tilt-values/`
- [x] Increased timeout: `update_settings(k8s_upsert_timeout_secs=600)`
- [ ] Verified Deployments reference `repo@sha256:…` with clean names (no prefix mangling)

## FAQ

- Can we still use live_update for frontends?
  - Yes. This pattern targets backend images; frontends can keep `docker_build` + live_update for hot reload.

- What about CI/CD?
  - CI builds and pushes, resolves digests, and Helmfile deploys with digest overrides — the same contract as local.

- Do we need to keep `:dev` tags?
  - No. They are not required when using digest‑based deploys. Keep them only if your tooling still references tags.

## CI/CD Alignment

- In CI, build and push images, then resolve digests:
  - `docker buildx build --push -t $IMAGE:$GIT_SHA …`
  - `DIGEST=$(crane digest $IMAGE:$GIT_SHA)`
- Helmfile deploys with `image.graph` + `image.digest` (no tags), enabling rollback by promoting previous digests.

## Decision Log

### 2025-08-29: Infrastructure/Application Separation
- **Problem**: Tilt was deploying infrastructure via `helm_resource()`, causing ownership conflicts
- **Example**: Gateway deployment by Tilt deleted cert-manager certificates created by Helmfile
- **Solution**: Removed all infrastructure deployments from Tilt, kept only application services
- **Result**: Clean separation - Helmfile owns infrastructure, Tilt owns applications

### Future: Digest-Based Deployments
- Remove floating tags (`:dev`) in favor of digests to avoid ambiguity
- Keep Helm authoritative for all cluster changes; Tilt only prepares immutable inputs
- Adopt a per-service tilt-values layer to keep local dev changes encapsulated
