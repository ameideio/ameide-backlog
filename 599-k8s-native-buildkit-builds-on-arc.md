# 599: k8s-native image builds on ARC (BuildKit + buildctl)

**Status:** Implemented (local)  
**Owner:** Platform DX / CI  
**Scope:** Enable container image builds on `arc-local` runners without relying on Docker-in-Docker.

---

## Problem

ARC runner pods often run without a local Docker daemon (and should not require privileged Docker access).
Workflows that build images must be able to run on `runs-on: arc-local` without rewriting Dockerfiles or leaking secrets.

We already rely on BuildKit features (e.g. `RUN --mount=type=secret`) for safe build-time secret handling, so the build strategy must be **BuildKit-compatible**.

**Target model (explicit):** local ARC runners (`arc-local`) must not rely on Docker-in-Docker; image builds use in-cluster BuildKit (`buildctl` → `buildkitd`).

---

## Proposed Approach

1) Deploy a **local-only** BuildKit daemon (`buildkitd`) inside the cluster (GitOps-managed).
2) In workflows running on ARC, install `buildctl` and connect to the in-cluster daemon (`tcp://...`).
3) For ARC smoke runs, export images as **OCI tar** (`--output type=oci,dest=...`) to validate builds without pushing.
4) Keep publish workflows guarded (`push=false` / `publish=false` by default on `workflow_dispatch`).

---

## Contracts

- Runner label: `arc-local` (runner scale set name)
- BuildKit endpoint variable: `AMEIDE_BUILDKIT_ADDR` (repo/org variable)
  - Default: `tcp://buildkitd.buildkit.svc.cluster.local:1234`
  - Local runner pods also export `AMEIDE_BUILDKIT_ADDR` directly, so ARC workflows can rely on it without hardcoding.

---

## Implementation (GitOps, local cluster)

### Deployed resources

- Namespace: `buildkit`
- Deployment: `buildkitd` (image: `docker.io/moby/buildkit@sha256:86c0ad9d1137c186e9d455912167df20e530bdf7f7c19de802e892bb8ca16552`)
- Service: `buildkitd` (ClusterIP) on port `1234`
- NetworkPolicy: restrict ingress to `arc-runners` (and same-namespace)
- DaemonSet: `binfmt` (installs `amd64` emulation on arm64 k3d nodes)

GitOps sources:

- Component: `environments/local/components/cluster/configs/buildkit/component.yaml`
- Manifests: `sources/values/_shared/cluster/buildkit.yaml`

### Runner wiring

- Runner pods export `AMEIDE_BUILDKIT_ADDR=tcp://buildkitd.buildkit.svc.cluster.local:1234` so workflows don’t need to hardcode it.
- `arc-local` uses a pinned runner image with baseline tools (including `buildctl` and `skopeo`) to avoid per-workflow installer glue.

### BuildKit security posture (local-only)

- **Default (current):** privileged `buildkitd` in local cluster only.
- **Why:** reliable Dockerfile builds on k3d/k3s, while keeping ARC runner pods unprivileged.
- **If privileged is not acceptable:** choose rootless BuildKit (requires extra kernel/filesystem support like `fuse-overlayfs`) or run privileged BuildKit on a dedicated, tainted node pool with strict admission.

### Variable management (avoid hardcoding)

- Preferred: org-level Actions variable `AMEIDE_BUILDKIT_ADDR` so all repos share one endpoint.
- Fallback: per-repo variable (works today if org-level access is restricted).
- Helper: `scripts/github/set-actions-variable.sh`.

### “Push” policy (ARC)

- `arc-local` is for smoke builds by default: validate Dockerfile and produce artifacts (OCI tar), **no publishing** on `workflow_dispatch` unless explicitly opted-in.
- Real image pushes should run on GitHub-hosted runners or a separate trusted runner set with locked-down secrets and review gates.
  - If you accept pushes from ARC, use dedicated `GHCR_USERNAME` / `GHCR_TOKEN` secrets (avoid relying on `GITHUB_TOKEN` permissions for org-owned packages).

### Network / egress requirements

- BuildKit needs outbound access to registries and build inputs (common: `ghcr.io`, `registry-1.docker.io`, npm, Go proxy, BSR).
- Runner pods need working cluster DNS and CA trust; if default-deny egress is introduced later, add explicit allowances for `arc-runners` and `buildkit`.

---

## Definition of Done

- Cluster has `Service/buildkitd` reachable from `arc-runners` runner pods.
- `cd-service-images` and `cd-devcontainer-image` workflows can perform a BuildKit build on ARC (no Docker), at least as a smoke path.
- Workflows remain safe by default when manually dispatched (no unintended pushes).
