# 359 – Resilient Data-Plane Deploy Sequencing

## Summary

- **Symptoms (Nov 6‑7 2025):**
  - `infra:10-bootstrap` and `infra:45-data-plane-ext` failed repeatedly because the `ameide` namespace was mid‑deletion while Helmfile was trying to create release secrets.
  - `minio` stayed in `pending-install` (`another operation … in progress`) after a prior Helm crash, blocking every subsequent `helmfile sync`.
  - Temporal pods (`temporalio/server:1.29.0`) and MinIO bucket bootstrap (`minio/mc:latest`) hit Docker Hub rate limits and stayed in `ImagePullBackOff`, even though Helm ran with `--atomic`.
- **Impact:** Local devcontainer bootstrap never completes unattended; engineers must babysit Helmfile runs, uninstall stuck releases, and rerun stages whenever Docker Hub throttles the cluster.
- **Goal:** Make the data-plane layers reproducible and fault tolerant so that devcontainer post-create, CI smoke tests, and `run-helm-layer.sh` succeed without manual cleanup.

## Pain Points

1. **Namespace + Cluster Coordination**
   - `scripts/infra/ensure-k3d-cluster.sh` recreates the entire cluster when drift is detected. Helmfile had no awareness, so it raced the namespace teardown and failed with “namespace being terminated.”
   - Manual Helm commands (`helm upgrade minio …`) overlapped with Helmfile, leaving releases stuck in `pending-install`.

2. **Registry / Image Pull Fragility**
   - Both MinIO and Temporal pull from Docker Hub anonymously; k3d nodes hit the free rate limit (429) during every bootstrap.
   - Even after a host `docker login`, kubelet remains unauthenticated until a Kubernetes `imagePullSecret` or local mirror is configured.

3. **Lack of Self-Healing**
   - Stuck Helm releases require manual `helm uninstall`.
   - Namespace deletion only aborts the run; no automated wait/ retry loop exists.
   - Helm guard rails are limited to `--atomic`; they do not watch for prerequisites (registry ready, upstream layers healthy).

## Proposal

### 1. Deploy Sequencing Guard Rails

- Promote the new k3d lock/ready files to a shared contract:
  - `scripts/infra/ensure-k3d-cluster.sh` writes `k3d-cluster.lock` while reconciling and `k3d-cluster-ready` on success.
  - `infra/kubernetes/scripts/helmfile-sync.sh` now waits for the lock before touching the cluster and expands the namespace termination preflight to `ameide`, `ameide-int-tests`, and operator namespaces.
- Extend this pattern to all automation entry points (Tilt, CI, devcontainer post-create). Anything that runs Helmfile must first block on the lock and ensure namespaces are `Active`.
- Add a Helm release watchdog:
  1. Before each `run-helm-layer.sh`, scan `helm list -A` for releases in `pending-*`.
  2. Auto-run `helm uninstall <release>` (with confirmation in CI) when a release has been pending for >N minutes.

### 2. Deterministic Images

- Mirror Temporal + MinIO artifacts into `k3d-dev-reg` (or central ACR) as part of bootstrap:
  ```
  docker pull temporalio/server:1.29.0
  docker tag … k3d-dev-reg:5000/third-party/temporal/server:1.29.0
  docker push …
  ```
- Pin Helm values to those mirrored repositories (or digests) so k3d never needs to talk to Docker Hub.
- As a fallback, publish a shared Docker Hub pull secret and inject it via `imagePullSecrets` in `infra/kubernetes/values/infrastructure/temporal.yaml` and `minio` values.

### 3. Health Prechecks & Retries

- Add a `preflight_data_plane_ext()` helper to `run-helm-layer.sh`:
  - Wait for `postgres-ameide` RW service, `redis-master`, and `minio` service to become reachable before syncing.
  - Verify the local registry (`http://localhost:5000/v2/_catalog`) prior to any Helm run (reuse the devcontainer probe).
- Enhance `run-helm-layer.sh` retry logic:
  - On `ErrImagePull` with 429, automatically sleep/backoff and re-run `helm upgrade` after ensuring credentials/mirrors exist.
  - On `another operation is in progress`, call the release watchdog to clean up the stuck revision, then retry once.

### 4. Documentation & Onboarding

- Add a “Data Plane Ready” checklist to `backlog/314-full-stack-integration-pipeline.md` covering:
  1. k3d reconcile lock absence.
  2. Namespace `Active`.
  3. Local registry mirror warm.
  4. Required pull secrets created (`dockerhub-creds`).
  5. `helm list -n ameide` contains no `pending-*`.
- Update `.devcontainer/postCreateCommand.sh` to log these checkpoints (pass/fail) so developers get immediate feedback.

## Success Criteria

- Devcontainer bootstrap (post-create) finishes `helmfile sync -l stage=data-plane-ext` without manual intervention.
- CI or local scripts can rerun `run-helm-layer.sh infra/kubernetes/helmfiles/45-data-plane-ext.yaml` back-to-back without hitting namespace termination, Docker Hub rate limits, or Helm release locks.
- Observability: logs clearly show whether the blocker was registry auth, namespace deletion, or a stuck release, enabling self-serve remediation.
