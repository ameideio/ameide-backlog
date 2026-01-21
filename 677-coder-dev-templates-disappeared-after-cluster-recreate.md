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
  - The existing GitHub Actions workflow used a `CODER_SESSION_TOKEN` which had expired (401), and admin tokens were capped to 168h by default.
  - The template workflow hard-failed on “GitHub External Auth linked” even though we still want to push+promote template versions without running the workspace E2E.

Resolution (GitOps)

  - Re-enabled GitHub External Auth provider in Coder values (still Keycloak-only login):
      - `CODER_OAUTH2_GITHUB_DEFAULT_PROVIDER_ENABLE=false` (no GitHub login)
      - `CODER_EXTERNAL_AUTH_0_*` + `externalAuth.github.enabled=true`
  - Increased token lifetime defaults for CI in dev:
      - `CODER_DEFAULT_TOKEN_LIFETIME=8760h`
      - `CODER_MAX_ADMIN_TOKEN_LIFETIME=8760h` (required because CI token was minted as admin)
  - Rotated GitHub Actions secret `CODER_SESSION_TOKEN` to a fresh long-lived token.
  - Made `.github/workflows/coder-devcontainer-e2e.yaml` resilient:
      - Template push+promote continues even if GitHub External Auth is not linked; workspace E2E is gated on linkage.

Verification

  - `coder` Postgres DB contains templates:
      - `ameide-dev`
      - `ameide-gitops`
      - `scratch`
  - The “publish templates” workflow completes successfully and is repeatable via `workflow_dispatch`.

Follow-ups (recommended)

  - Decide whether dev Coder DB is “ephemeral by design”:
      - If yes: keep the “rehydrate templates via CI” workflow as the documented recovery procedure.
      - If no: add CNPG backups (object storage) and/or adjust reclaim policy (Retain) in the storage layer.
  - Decide whether template publishing must be Argo-driven (PostSync hook Job) or may remain CI-driven:
      - CI-driven (current) avoids storing a long-lived Coder token in-cluster and can run on GitHub-hosted runners (independent of ARC health).
      - Argo-driven implies storing/passing a long-lived Coder admin token via Vault/ESO, and operating a Job with the right egress/auth to publish templates after a DB reset.

Update (2026-01-15): workspace health hardening + orphan cleanup

  - Vendor-aligned template refactor landed in the GitOps repo:
    - keep `startup_script` minimal and non-fatal; move setup into `coder_script`
    - use the vendor-supported `code-server` module instead of hand-managed daemon logic
    - pin `code-server` module version (`registry.coder.com/modules/code-server/coder`), and pin Codex CLI by default (`DEVCONTAINER_CODEX_VERSION=0.87.0`) for determinism
    - Codex auth policy: `CODEX_ACCOUNT_SLOT=auto` is non-fatal; `CODEX_ACCOUNT_SLOT=0|1|2` is fail-fast if the requested slot secret is missing
  - Storage: switch `/workspaces` from `emptyDir` to a per-workspace `managed-csi-premium` PVC (`16Gi`) so stop/start and pod reschedules keep the repo + editor caches.
  - Added a break-glass GitHub workflow to delete orphaned Coder workspace namespaces when Coder state diverges from cluster resources:
    - `.github/workflows/aks-delete-orphan-coder-namespace.yaml`

Update (2026-01-21): remove CI as the recovery engine (GitOps-only)

  - Template rehydration is now handled in-cluster:
    - `CronJob/coder-templates-reconciler` pushes templates from Git into the Coder DB and retries until they are visible.
  - First-user bootstrap is self-healing and SSO-aligned:
    - `CronJob/coder-bootstrap-admin` ensures the bootstrap user exists and reconciles it to `login_type=oidc` to avoid “Incorrect login type … password”.
  - Resulting contract: after a cluster/namespace recreate (fresh DB), ArgoCD reconciliation + these CronJobs converge the Coder control plane back to “dashboard usable” without manual `/setup` or CI token rotation.
