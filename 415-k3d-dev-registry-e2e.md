# 415 – k3d dev registry end-to-end flow (implementation notes)

> ⚠️ **Deprecated workflow:** These notes cover the legacy k3d dev registry. The supported approach is now remote-first AKS + Telepresence (backlog/435). Keep this file for archival purposes only. References to `tools/bootstrap/bootstrap-v2.sh` map to the GitOps bootstrap currently located at `ameide-gitops/bootstrap/bootstrap.sh`.

## Objectives
- Single dev registry endpoint: `k3d-ameide.localhost:5001/ameide`, used by builders, GitOps values, and cluster pulls.
- No `k3d image import`; every dev build is pushed to the registry.
- k3s/containerd is configured to use the HTTP registry (no HTTPS fallback), so pulls succeed out-of-the-box after bootstrap.
- Image naming uses hyphens across the fleet (`agents-runtime`, `www-ameide`, `www-ameide-platform`, etc.); avoid underscore repos to match GitOps values and containerd pulls.

## Bootstrap mechanics
- Entry point: `ameide-gitops/bootstrap/bootstrap.sh --config bootstrap/configs/dev.yaml` (legacy references may still mention `tools/bootstrap/bootstrap-v2.sh` inside `ameide-core`).
- Defaults:
  - `K3D_REGISTRY_PORT=5001`
  - `BUILD_IMAGES=1` (unless `--skip-build-images`)
  - `IMPORT_IMAGES=0`, `PUSH_IMAGES=1`
- k3d creation (`tools/bootstrap/lib/k3d.sh`):
  - Registry: `k3d registry create ameide.localhost --port 0.0.0.0:${K3D_REGISTRY_PORT}` (host 5001 → container 5000).
- Cluster: `k3d cluster create ... --registry-use k3d-ameide.localhost:5001` with `--disable-default-registry-endpoint` for server+agent.
- Registry config: mirrors `k3d-ameide.localhost:5001` to `http://k3d-ameide.localhost:5001` (k3s containerd treats it as HTTP).
- Build stage (if not skipped):
  - Script: `scripts/build-all-images-dev.sh`
  - Registry envs: `AMEIDE_REGISTRY_HOST=k3d-ameide.localhost:5001`, namespace `ameide`; push endpoint default `AMEIDE_REGISTRY_PUSH_HOST=localhost:5001` so the devcontainer talks to the host-published registry port.
  - Tags: `k3d-ameide.localhost:5001/ameide/<service>:dev` (plus local `ameide/<service>:dev`) — registry content is the same regardless of whether the push endpoint is `localhost` or `k3d-ameide.localhost`.
  - Image names are hyphenated in tags to match GitOps values (e.g., `agents-runtime`, `inference-gateway`, `workflows-runtime`, `workflow-worker`, `www-ameide`, `www-ameide-platform`).
  - Concurrency: `JOBS=$(nproc)` (override with `JOBS=`).
  - Secrets: BuildKit secrets from `.env` (e.g., `PKG_GITHUB_COM_TOKEN`, `GHCR_TOKEN`).
  - Outputs: per-service logs under `artifacts/image-build-logs/*_dev.log` and `build-summary-dev.csv`.
- Separation of loops:
  - Argo CD path: uses the bootstrap-built images pushed to `k3d-ameide.localhost:5001/ameide` and deploys them in-cluster.
  - Tilt path: inner-loop builds run from the devcontainer and can push/tag independently (still expected to use the same registry/values); Argo does not consume Tilt-built artifacts.
- GitOps apply:
  - Dev/local values already point to `k3d-ameide.localhost:5001/ameide/...:dev` with `pullPolicy: IfNotPresent`, no pull secrets. (Legacy flow; current GitOps image reference policy is tracked in `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md`.)
  - Argo applies apps; kubelet pulls from the registry directly.

## Expected steady state
- `docker port k3d-ameide.localhost` → `5000/tcp -> 0.0.0.0:5001`
- Registry populated: `docker exec k3d-ameide.localhost ls /var/lib/registry/docker/registry/v2/repositories` lists `ameide/*`.
- Pods pull without `ImagePullBackOff` or `manifest unknown`.

## Common failure modes
1) **Skipped build after reset**  
   - Symptom: registry empty; k3d registry logs show 404 `manifest unknown` for `:dev`.  
   - Fix: rerun `scripts/build-all-images-dev.sh` (let it finish), then Argo resync or wait for kubelet retries.
2) **Host port collision**  
   - Symptom: `k3d registry create ... port ... bind: address already in use`.  
   - Fix: free host port 5001 or change `K3D_REGISTRY_PORT` + all defaults (build script + GitOps values) in lockstep, recreate k3d, rebuild/push.
3) **HTTPS fallback**  
   - Symptom: `http: server gave HTTP response to HTTPS client`.  
   - Fix: ensure `--disable-default-registry-endpoint` is present for k3s, and registries mirror uses `http://...:5001`.
4) **Partial pushes (interrupted build)**  
   - Symptom: `build-summary-dev.csv` says success, but registry has no manifests; logs end with endless `Waiting` on push.  
   - Fix: rerun `scripts/build-all-images-dev.sh` without interruption.
5) **Push endpoint mismatch (push hangs)**  
   - Symptom: `docker push k3d-ameide.localhost:5001/...` hangs on `Waiting` because the devcontainer resolves the hostname to the registry container IP (no host port).  
   - Fix: push via the host-published port (`AMEIDE_REGISTRY_PUSH_HOST=localhost:5001 scripts/build-all-images-dev.sh`) or add `127.0.0.1 k3d-ameide.localhost` to `/etc/hosts` so pushes reach the host port 5001.
6) **Underscore vs hyphen repos**  
   - Symptom: pod ImagePullBackOff with “not found” even though the image was built (e.g., GitOps uses `www-ameide` but registry has `www_ameide`).  
   - Fix: use hyphenated image names in build tags (`agents-runtime`, `inference-gateway`, `workflow-worker`, `workflows-runtime`, `www-ameide`, `www-ameide-platform`) so tags match GitOps values.

## Minimal verification checklist (post-bootstrap)
- Registry contents: `docker exec k3d-ameide.localhost ls /var/lib/registry/docker/registry/v2/repositories/ameide`
- Pods: `kubectl get pods -n ameide | grep -v Running` (ensure no ImagePullBackOff)
- Argo apps: `kubectl get applications -n argocd` (should be Synced/Healthy or Progressing for normal rollouts)
- Pull test: `ctr -n k8s.io images pull k3d-ameide.localhost:5001/ameide/<service>:dev` from a k3d node container if needed.

## Scope

This story covers the **Argo CD / GitOps dev path only**:

- Ensure the k3d dev cluster is created with a local registry (`k3d-ameide.localhost:5001/ameide`) and that k3s/containerd is configured to pull from it over HTTP.
- Ensure dev bootstrap builds and **pushes** images for all services to that registry.
- Ensure dev GitOps values reference `k3d-ameide.localhost:5001/ameide/<service>:dev`.

Tilt’s inner-loop build path and its registry configuration (`registry.localhost:32000` by default) are out of scope for this story and are tracked under 373.

> Staging/prod remain unchanged here: they continue to use the cloud registry (GHCR/ACR) with pull secrets and Argo-only deployment flows; no Tilt involvement.

---

## Implementation progress (602/603 alignment)

### 2025-12-26

- This document’s `:dev`-tagged local registry references are legacy guidance. Target state is to pin GitOps-managed workloads by digest (`image.ref` with `@sha256:...`) even for `local`, with automation advancing `local`/`dev` via PRs (see 602/603).
