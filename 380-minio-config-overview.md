# 380: MinIO configuration – guardrail-compliant overview

**Scope:** GitOps-managed MinIO component (`data-minio`), chart `bitnami/minio` 17.0.21, values under `gitops/ameide-gitops/sources/values`.

## Deployment shape
- **Mode:** Standalone with 1 replica (Deployment replicaCount, no StatefulSet knobs in this mode).
- **Images:** `quay.io/minio/minio:RELEASE.2025-09-07T16-13-09Z`, `quay.io/minio/mc:latest`, console `quay.io/minio/console:v0.30.0`.
- **Storage:** 10 Gi PVC, default `managed-csi`; dev overlay forces `local-path`.
- **Args:** Explicit `server /bitnami/minio/data --address=:9000 --console-address=:9001`.
- **Services:** API 9000, console 4002, both ClusterIP.
- **Buckets:** Created on startup via `MINIO_DEFAULT_BUCKETS` (`artifacts,models,backups,langfuse-events,langfuse-batch,langfuse-media,loki,tempo-traces`).

## Secrets & guardrails (backlog/362-v2 aligned)
- **Secret source:** Vault via `ClusterSecretStore/ameide-vault` (Kubernetes auth, service account `foundation-external-secrets`).
- **ExternalSecrets (chart-owned, no inline creds):**
  - `minio-root-credentials-sync` → `Secret/minio-root-credentials` (keys `root-user`, `root-password`; chart wired via `auth.existingSecret: minio-root-credentials`, `auth.rootUserSecretKey: root-user`, `auth.rootPasswordSecretKey: root-password`, sourced from Vault keys `minio-root-user`, `minio-root-password`).
  - `minio-service-users` → user config files for MC provisioning; pulls `loki-storage-secret-key` and `tempo-storage-secret-key` from Vault and renders:
    - `loki-user`: username `loki-storage`, policy `readwrite`, enabled.
    - `tempo-user`: username `tempo-storage`, policy `readwrite`, enabled.
- **Charts consume secrets via `existingSecret` only; `createSecret` toggles remain off.**

## Provisioning hooks
- **Bucket bootstrap Job:** Hook `data-minio-bucket-bootstrap` creates/suspends versioning for the default buckets using `mc` and the root secret.
- **Service-user provisioning Job:** Uses `minio-service-users` Secret to create/enable users and attach/detach policies via `mc admin`. Waits for API availability, restarts server to apply changes.
- Both hooks are part of `extraDeploy` in `sources/values/_shared/data/data-minio.yaml`; Argo health remains on the main Deployments/Services (hooks are short-lived).

## Chart default overrides
- **Init container override:** `bitnami/os-shell:latest` for volume permissions (upstream tag removed).
- **Security:** `global.security.allowInsecureImages: true` (quay.io images), seccomp RuntimeDefault.
- **Metrics:** Disabled (Prometheus ServiceMonitor off by default).
- **Networking:** NetworkPolicies for API, console, and provisioning jobs shipped via chart defaults.

## Environments & layering
- Shared values: `sources/values/_shared/data/data-minio.yaml` (all knobs above).
- Dev overlay: `sources/values/dev/data/data-minio.yaml` (`enabled: true`, `local-path` storage class, 1 replica).
- Other environments: MinIO is off by default; enable by providing per-env value file and storage class.

## Operational notes
- **Flux/Argo determinism:** Ignore-diffs not required; ExternalSecrets own secret materialization, and provisioning hooks expect those Secrets before running.
- **Rotation:** Update Vault keys (`minio-root-user/minio-root-password`, `loki-storage-secret-key`, `tempo-storage-secret-key`), then annotate ExternalSecrets to force sync; provisioning job will recreate users/policies.
- **Failure modes:** Missing `ClusterSecretStore` or ExternalSecrets will block pods (`CreateContainerConfigError` or provisioning Job wait). Verify with `kubectl get externalsecret -n ameide` before syncs.
