# 599: k8s-native image builds on ARC (BuildKit + buildctl)

**Status:** Implemented (local + AKS)  
**Owner:** Platform DX / CI  
**Scope:** Enable container image builds on ARC runners (local k3d + AKS) without relying on Docker-in-Docker.

## Implementation status (operational)

- GitOps landed and validated on AKS:
  - `ameideio/ameide-gitops#439` (BuildKit StatefulSet recreated as `buildkitd-v2` to apply `64Gi` PVC)
  - `ameideio/ameide-gitops#441` (add read-only AKS inspect script; guard local probe script from cloud)
  - `ameideio/ameide-gitops#453` (enable BuildKit mTLS over TCP; mount client certs into ARC runners)
- Backlog docs updates:
  - `ameideio/ameide-backlog#225` (runbook gates + mTLS invariant)
  - `ameideio/ameide-backlog#226` (clarify local vs AKS probe tooling)
- Verified signals (AKS):
  - `Service/buildkitd` has endpoints to a Ready pod selected by `app.kubernetes.io/name=buildkitd`
  - `buildctl debug workers -v` reports `linux/arm64` and `linux/amd64` platforms

Known gaps / follow-ups:

- Orphaned legacy PVCs may remain after migrations (requires a GitOps/CI-sanctioned cleanup path; avoid manual deletes).

---

## Problem

ARC runner pods often run without a local Docker daemon (and should not require privileged Docker access).
Workflows that build images must be able to run on ARC runners without rewriting Dockerfiles or leaking secrets.

We already rely on BuildKit features (e.g. `RUN --mount=type=secret`) for safe build-time secret handling, so the build strategy must be **BuildKit-compatible**.

**Target model (explicit):** ARC runners (`arc-local` and `arc-aks`) must not rely on Docker-in-Docker; image builds use in-cluster BuildKit (`buildctl` → `buildkitd`).

---

## Proposed Approach

1) Deploy a BuildKit daemon (`buildkitd`) inside each cluster (GitOps-managed).
2) In workflows running on ARC, install `buildctl` and connect to the in-cluster daemon (`tcp://...`).
3) For ARC smoke runs, export images as **OCI tar** (`--output type=oci,dest=...`) to validate builds without pushing.
4) Keep publish workflows guarded (`push=false` / `publish=false` by default on `workflow_dispatch`).

---

## Contracts

- Runner label: set via GitHub variable `AMEIDE_RUNS_ON` (no workflow defaults)
  - `arc-local` for local k3d ARC
  - `arc-aks-v2` for AKS ARC
- BuildKit endpoint variable: `AMEIDE_BUILDKIT_ADDR` (repo/org variable)
  - Default: `tcp://buildkitd.buildkit.svc.cluster.local:1234`
  - Local runner pods also export `AMEIDE_BUILDKIT_ADDR` directly, so ARC workflows can rely on it without hardcoding.
- BuildKit API safety invariant: **BuildKit over TCP must be protected with mTLS** (NetworkPolicy limits reach, not identity).
  - Implemented via a dedicated CA + cert-manager-issued server/client certs, mounted into runner pods.
  - Runner pods expose:
    - `AMEIDE_BUILDKIT_TLS_CA`, `AMEIDE_BUILDKIT_TLS_CERT`, `AMEIDE_BUILDKIT_TLS_KEY`, `AMEIDE_BUILDKIT_TLS_SERVERNAME`

---

## Implementation (GitOps, local + AKS)

### Deployed resources

- Namespace: `buildkit`
- StatefulSet (label `app.kubernetes.io/name=buildkitd`) (image: `ghcr.io/ameideio/mirror/buildkit@sha256:86c0ad9d1137c186e9d455912167df20e530bdf7f7c19de802e892bb8ca16552`)
  - Replica count is explicit per cluster (`buildkitd.replicas`); **AKS currently runs 1**.
  - Per-pod PVC-backed cache mounted at `/var/lib/buildkit` to avoid cold-cache rebuild storms and to survive pod restarts.
- PodDisruptionBudget: `buildkitd` (`minAvailable: 1`)
- Service: `buildkitd` (ClusterIP) on port `1234` (load-balances across StatefulSet pods)
- NetworkPolicy: restrict ingress to `arc-runners` (and same-namespace)
- DaemonSet: `binfmt` (installs emulation for `amd64,arm64` to support cross-arch builds)

GitOps sources:

- Components:
  - Local: `environments/local/components/cluster/configs/buildkit/component.yaml`
  - Azure: `environments/azure/components/cluster/configs/buildkit/component.yaml`
- Values:
  - Shared manifests: `sources/values/_shared/cluster/buildkit.yaml`
  - Azure overrides (pool + replicas + storage): `sources/values/cluster/azure/buildkit.yaml`

### Runner wiring

- Runner pods export `AMEIDE_BUILDKIT_ADDR=tcp://buildkitd.buildkit.svc.cluster.local:1234` so workflows don’t need to hardcode it.
- `arc-local` uses a pinned runner image with baseline tools (including `buildctl` and `skopeo`) to avoid per-workflow installer glue.
- `arc-aks-v2` uses the same runner image family, published multi-arch and pinned by manifest digest.

### BuildKit security posture

- **Default (current):** privileged `buildkitd` (runner pods remain unprivileged).
- **Why:** reliable Dockerfile builds while keeping ARC runner pods unprivileged.
- **If privileged is not acceptable:** choose rootless BuildKit (requires extra kernel/filesystem support like `fuse-overlayfs`) or run privileged BuildKit on a dedicated, tainted node pool with strict admission.
- **Remote API:** `buildkitd` exposes a gRPC API on TCP (`:1234`); baseline requirement is **mTLS** when using remote builds.
  - Server cert: `Secret/buildkitd-server-tls` in `buildkit`
  - Client cert: `Secret/buildkitd-client-tls` in `arc-runners`

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

### Stability under parallel builds (why the StatefulSet + PVC matters)

`CD / Service Images` can legitimately fan out into many concurrent image builds (especially when shared build inputs change and we rebuild everything).
If `buildkitd` is deployed as a single pod with an `emptyDir` state volume, “full rebuild” events turn into:

- cold-cache downloads (registries + language package managers + Go module proxy) repeated across many builds
- high memory / CPU pressure on the single builder
- intermittent client failures like:
  - `dial tcp ...:1234: i/o timeout`
  - `frontend grpc server closed unexpectedly`
  - Go module proxy fetch timeouts (`proxy.golang.org ... TLS handshake timeout`)

The required posture is therefore:

- availability: at least 1 builder replica behind the Service (2+ recommended once the buildkit node pool has capacity)
- persistence: PVC-backed `/var/lib/buildkit` so caches survive restarts and reduce repeated network fetches
- health: readiness/liveness probes so the Service doesn’t route to unhealthy builders

### Debugging / runbook

#### Runbook gates (copy/paste triage order)

1) **Argo renders clean** (no `ComparisonError`; cluster is on the intended overlay for BuildKit values).
2) **Endpoints exist + pods Ready**
   - `kubectl -n buildkit get endpoints buildkitd -o yaml`
   - `kubectl -n buildkit get sts -l app.kubernetes.io/name=buildkitd -o wide`
   - `kubectl -n buildkit get pods -l app.kubernetes.io/name=buildkitd -o wide`
3) **Reachability from runner**
   - Preferred (canonical signal): `buildctl --addr "${AMEIDE_BUILDKIT_ADDR}" debug workers` from an ARC runner pod.
   - Local k3d smoke check: `scripts/local/test-buildkit.sh` (creates/deletes a probe pod; local-only by default).
   - AKS admin read-only check (does **not** validate runner network path): `scripts/aks/inspect-buildkit.sh`
4) **Security invariant (mTLS for remote TCP)**
   - If `AMEIDE_BUILDKIT_ADDR` is `tcp://...:1234` and no client certs are provided, treat it as a policy gap (don’t “solve” with ad-hoc per-repo wiring).
5) **Platform support (multi-arch capability)**
   - `buildctl --addr "${AMEIDE_BUILDKIT_ADDR}" debug workers -v` and confirm the required `Platforms:` list (e.g. includes `linux/amd64` and `linux/arm64`)
   - If connected but missing platforms: check `binfmt` DaemonSet and worker config.
6) **Scheduling pinned (pool exists + labels/taints match)**
   - `kubectl get nodes -l ameide.io/pool=buildkit -o wide`
   - If builders are `Pending`, check node pool capacity and taints/tolerations.

#### Symptom mapping

- `dial tcp ...:1234: connect: connection refused` → Service routes to nothing listening (no ready endpoints / wrong port / crashloop).
- `dial tcp ...:1234: i/o timeout` → network policy / DNS / routing / nodes not reachable.
- `buildctl debug workers` works but `Platforms:` missing → emulation/binfmt or worker config issue.
- `proxy.golang.org ... TLS handshake timeout` → typically egress reliability; persistent cache reduces repeat fetch blast radius.

#### Logs

- builder logs: `kubectl -n buildkit logs -l app.kubernetes.io/name=buildkitd --tail=200`

---

## Definition of Done

- Cluster has `Service/buildkitd` reachable from `arc-runners` runner pods.
- `cd-service-images` and `cd-devcontainer-image` workflows can perform a BuildKit build on ARC (no Docker), at least as a smoke path.
- Workflows remain safe by default when manually dispatched (no unintended pushes).
