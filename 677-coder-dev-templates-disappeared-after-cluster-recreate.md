Incident: Coder dev templates disappeared after AKS/namespace recreate

Date

  - 2026-01-14 → 2026-01-15 (dev)

Summary

  - After the AKS dev cluster/namespace was recreated, Coder started behaving like a fresh install (`/setup` shown, templates missing).
  - Root cause was not “Coder lost a PV” on `coderd` (it doesn’t use one), but that the **CNPG Postgres cluster/PVCs backing Coder were new** and the StorageClass reclaim policy is `Delete`.
  - Secondary issue: template rehydration CI failed because the GitHub Actions `CODER_SESSION_TOKEN` secret had expired and Coder’s default admin-token max lifetime is 168h.

Impact

  - `https://coder.dev.ameide.io` required re-bootstrap (first-user / admin) and had no `ameide-dev` / `ameide-gitops` templates available in the UI.
  - Developers could not create the standard dev workspaces until templates were republished.

What changed / evidence

  - `ameide-dev` namespace and `clusters.postgresql.cnpg.io/postgres-ameide` were newly created (~11h age at time of investigation).
  - Coder was configured to use `Secret/coder-db-url` → `postgres-ameide-rw:5432/coder`.
  - Coder templates are stored in the Coder control-plane database (Postgres-backed), so a fresh DB means templates are absent.

Root cause

  - The dev cluster/namespace recreation caused CNPG PVCs to be recreated; Azure managed disk reclaim policy is `Delete`, so removing PVCs deletes the disk and therefore the DB contents.

Contributing factors

  - Templates are **not GitOps-native**; they live in Coder DB and require a publisher (CI/CLI) to push them back after a DB reset.
  - The platform previously relied on out-of-cluster/manual publishing (CI/CLI) and did not have an always-on in-cluster reconciler to rehydrate templates after DB resets.

Resolution (GitOps)

  - Treat template publishing as **seeding** (per `backlog/713-seeding-contract.md`) and make it self-healing.
  - Add in-cluster reconcilers (ArgoCD-managed):
    - `CronJob/coder-bootstrap-admin`: ensures “first user exists” and reconciles bootstrap admin `login_type` to `oidc` (avoids `/setup` + “Incorrect login type”).
    - `CronJob/coder-templates-reconciler`: clones `ameideio/ameide-gitops` using `Secret/gh-auth` and runs `coder templates push` for `ameide-dev` + `ameide-gitops` using `Secret/coder-bootstrap-token`.
    - `CronJob/coder-cli-session-token`: maintains `coder-cli-session-token` in Vault KV so new workspaces can run `coder whoami` without interactive login.
  - Keep admin token lifetime caps raised in dev (so `coder-bootstrap-token` can be long-lived and deterministic).

Verification

  - After reconciliation:
    - UI no longer shows `/setup` (first user exists).
    - `coder templates list` contains:
      - `ameide-dev`
      - `ameide-gitops`
    - Fresh workspaces can clone private repos and run the default toolchain without manual auth.

Follow-ups (recommended)

  - Decide whether dev Coder DB is “ephemeral by design”:
      - If yes: keep the “rehydrate templates via CI” workflow as the documented recovery procedure.
      - If no: add CNPG backups (object storage) and/or adjust reclaim policy (Retain) in the storage layer.
  - Keep template publishing in-cluster (CronJob reconciler) to eliminate the “reset requires manual/CI intervention” class of outages.

Update (2026-01-15): workspace health hardening + orphan cleanup

  - Vendor-aligned template refactor landed in the GitOps repo:
    - keep `startup_script` minimal and non-fatal; move setup into `coder_script`
    - use the vendor-supported `code-server` module instead of hand-managed daemon logic
    - pin `code-server` module version (`registry.coder.com/modules/code-server/coder`), and pin Codex CLI by default (`DEVCONTAINER_CODEX_VERSION=0.87.0`) for determinism
    - Codex auth policy: `CODEX_ACCOUNT_SLOT=auto` is non-fatal; `CODEX_ACCOUNT_SLOT=0|1|2` is fail-fast if the requested slot secret is missing
  - Storage: switch `/workspaces` from `emptyDir` to a per-workspace `managed-csi-premium` PVC (`16Gi`) so stop/start and pod reschedules keep the repo + editor caches.
  - Added a break-glass GitHub workflow to delete orphaned Coder workspace namespaces when Coder state diverges from cluster resources:
    - `.github/workflows/aks-delete-orphan-coder-namespace.yaml`
