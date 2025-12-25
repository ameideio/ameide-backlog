# 599: k8s-native image builds on ARC (BuildKit + buildctl)

**Status:** Draft / Proposed  
**Owner:** Platform DX / CI  
**Scope:** Enable container image builds on `arc-local` runners without relying on Docker-in-Docker.

---

## Problem

ARC runner pods often run without a local Docker daemon (and should not require privileged Docker access).
Workflows that build images must be able to run on `runs-on: arc-local` without rewriting Dockerfiles or leaking secrets.

We already rely on BuildKit features (e.g. `RUN --mount=type=secret`) for safe build-time secret handling, so the build strategy must be **BuildKit-compatible**.

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

---

## Definition of Done

- Cluster has `Service/buildkitd` reachable from `arc-runners` runner pods.
- `cd-service-images` and `cd-devcontainer-image` workflows can perform a BuildKit build on ARC (no Docker), at least as a smoke path.
- Workflows remain safe by default when manually dispatched (no unintended pushes).

