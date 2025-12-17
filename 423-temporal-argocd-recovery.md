# 423: Temporal ArgoCD recovery (operator-based)

> ✅ **Current deployment model:** Temporal is deployed via the community Temporal operator (`cluster-temporal-operator`) and per-environment `TemporalCluster`/`TemporalNamespace` resources (`data-temporal`).

## What is supposed to exist

- ArgoCD apps:
  - `cluster-crds-cert-manager` (cert-manager CRDs, shared dependency)
  - `cluster-cert-manager` (cluster-shared cert-manager install in `cert-manager`)
  - `cluster-crds-temporal-operator` (CRDs)
  - `cluster-temporal-operator` (controller, runs in `ameide-system`, watches all namespaces)
  - `{env}-data-temporal` (per-environment TemporalCluster + TemporalNamespace in `ameide-{env}`)
- In each environment namespace (e.g. `ameide-dev`):
  - A `TemporalCluster` named `temporal` (services: `temporal-frontend`, `temporal-ui`, etc.)
  - `TemporalNamespace` resources for the product namespaces we use (e.g. `default`, `ameide`)

## Reproducibility / self-healing guardrails

`data-temporal` includes an idempotent DB preflight hook so the normal path requires no manual SQL:

- **Job**: `temporal-db-preflight` (Argo `PreSync`, sync-wave `-1`)
- **Does**:
  - Wait for Postgres connectivity.
  - Ensure `namespace_metadata(partition_id=54321)` exists (prevents startup crash `Failed to lock namespace metadata. sql: no rows in result set`).
  - Detect stale `cluster_metadata_info` cluster names; it fails-fast by default. Local may opt-in to auto-delete an allowlisted stale name to self-heal previous rename experiments.

`data-temporal` also includes a deterministic Temporal schema setup hook (so preflight never deadlocks on missing tables):

- **Job**: `temporal-db-schema` (Argo `PreSync`)
- **Does**: run `temporal-sql-tool setup-schema` + `update-schema` using the vendor admin-tools image, matching the upstream Temporal schema tooling.

## Recovery checklist (no manual SQL)

1) Ensure cert-manager + operator CRDs + controller are Healthy
   - ArgoCD apps: `cluster-crds-cert-manager`, `cluster-cert-manager`, `cluster-crds-temporal-operator`, `cluster-temporal-operator`
   - Why: the Temporal operator admission webhooks depend on cert-manager CA injection (provided cluster-wide by `cluster-cert-manager`) in the operator namespace (`ameide-system`).
   - Operator logs: `kubectl -n ameide-system logs deploy/temporal-operator-controller-manager`
   - If the operator pod is stuck `ContainerCreating` with `secret "webhook-server-cert" not found`, verify cert-manager webhook/cainjector are Running in `cert-manager` and the webhook configurations have an injected `caBundle`.

2) Ensure the environment Temporal app is Healthy
   - ArgoCD app: `{env}-data-temporal`
   - Check the DB preflight hook logs: `kubectl -n <env-namespace> logs job/temporal-db-preflight`

3) Inspect the `TemporalCluster` reconcile status and schema Jobs
   - `kubectl -n <env-namespace> get temporalcluster temporal -o yaml`
   - Check GitOps schema/preflight hook Jobs and logs:
     - `kubectl -n <env-namespace> logs job/temporal-db-schema`
     - `kubectl -n <env-namespace> logs job/temporal-db-preflight`
   - If the operator still creates internal schema Jobs (version-dependent), inspect those as well.

3.1) If schema/preflight Jobs are stuck `Pending` on AKS
   - Symptom: `Job/temporal-db-schema` (or `temporal-db-preflight`) stays Active with a Pod that never schedules.
   - Root cause: environment node pools are tainted (e.g., `ameide.io/environment=dev:NoSchedule`), but the hook Jobs lacked the `nodeSelector`/`tolerations` from the node profile.
   - Fix: ensure the `data-temporal` chart applies `.Values.nodeSelector` and `.Values.tolerations` to hook Jobs so they schedule on the correct env pool (values are provided via `config/node-profiles/<nodeProfile>.yaml`).

4) If the DB has stale cluster metadata (duplicate cluster names)
   - Prefer a GitOps “break-glass” change over manual SQL:
     - Local: enable `preflight.autoFix.clusterMetadataInfo` and allowlist the stale name (e.g. `active`).
     - Staging/prod: open a PR to temporarily allowlist + enable the auto-fix for the single stale cluster name, resync, then revert the PR.

## Notes

- The Temporal “namespace” concept is **not** a Kubernetes namespace. Temporal namespaces are declared via `TemporalNamespace` CRs shipped with `data-temporal`.
- Schema setup/update is owned by the operator reconcile (not Flyway, and no separate `data-migrations-temporal` app).

### Addendum (2025-12-13): Naming and local DNS

- We keep the service name **Temporal** (and `TemporalCluster` name `temporal`) as the canonical naming.
- Local DNS/HTTPRoute hostnames should use the `*.local.ameide.io` convention (e.g. `temporal.local.ameide.io`) to avoid mixing “dev” and “local” semantics.
