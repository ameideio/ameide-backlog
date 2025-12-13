# 423: Temporal ArgoCD recovery (operator-based)

> ✅ **Current deployment model:** Temporal is deployed via the community Temporal operator (`cluster-temporal-operator`) and per-environment `TemporalCluster`/`TemporalNamespace` resources (`data-temporal`).

## What is supposed to exist

- ArgoCD apps:
  - `cluster-crds-temporal-operator` (CRDs)
  - `cluster-temporal-operator` (controller, runs in `temporal-system`, watches all namespaces)
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

## Recovery checklist (no manual SQL)

1) Ensure operator CRDs + controller are Healthy
   - ArgoCD apps: `cluster-crds-temporal-operator`, `cluster-temporal-operator`
   - Operator logs: `kubectl -n temporal-system logs deploy/temporal-operator-controller-manager`

2) Ensure the environment Temporal app is Healthy
   - ArgoCD app: `{env}-data-temporal`
   - Check the DB preflight hook logs: `kubectl -n <env-namespace> logs job/temporal-db-preflight`

3) Inspect the `TemporalCluster` reconcile status and schema Jobs
   - `kubectl -n <env-namespace> get temporalcluster temporal -o yaml`
   - Check operator-created schema Jobs/pods (usually `temporal-setup-*` / `temporal-update-*`) and their logs.

4) If the DB has stale cluster metadata (duplicate cluster names)
   - Prefer a GitOps “break-glass” change over manual SQL:
     - Local: enable `preflight.autoFix.clusterMetadataInfo` and allowlist the stale name (e.g. `active`).
     - Staging/prod: open a PR to temporarily allowlist + enable the auto-fix for the single stale cluster name, resync, then revert the PR.

## Notes

- The Temporal “namespace” concept is **not** a Kubernetes namespace. Temporal namespaces are declared via `TemporalNamespace` CRs shipped with `data-temporal`.
- Schema setup/update is owned by the operator reconcile (not Flyway, and no separate `data-migrations-temporal` app).

