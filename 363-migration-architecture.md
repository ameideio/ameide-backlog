> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# backlog/363 – Unified Flyway Architecture

**Created:** Mar 2026  
**Owner:** Platform DX / Infrastructure  
**Supersedes:** backlog/332-flyway.md (v1), backlog/332-flyway-v2.md (v2)

---

## Why another backlog?

The first two Flyway backlogs focused on tactical fixes (local CLI helper, secret templating, devcontainer bootstrap). Since then we rebuilt the entire pipeline:

* db-migrations is a standalone Helm chart deployed via Layer 42.
* The Flyway image is managed by Tilt using the v3 registry workflow (see backlog/343).
* Layer 15 pre-provisions every database secret, and `scripts/infra/bootstrap-db-migrations-secret.sh` now **verifies** rather than fabricating credentials.
* The Tilt/Tekton workflows use the same `ops-db-migrate-platform` helper for both local and CI.

This backlog is the canonical reference for how migrations work today and where we plan to take them next. The older flyway backlogs remain for historical context only.

---

## Architecture Overview

### Components

| Component | Responsibility |
| --- | --- |
| `migrations/Dockerfile.dev` | Ring 1 Flyway image for Tilt/local Helm helpers; builds from `flyway/flyway:10.17` and copies `db/flyway/sql/**` plus repeatables directly from the workspace. |
| `migrations/Dockerfile.release` | Ring 2 Flyway image used by GitOps/CI; identical build but installs from the stamped release ref published by `cd-service-images`. |
| Tilt migrations image build | Built via `docker_build(registry_image('ameide-migrations'), ...)` in Tiltfile. Auto-rebuilds on SQL file changes; manually trigger Layer 42 sync with `tilt trigger infra:42-db-migrations-run`. Uses `default_registry(host='localhost:5000', host_from_cluster='k3d-dev-reg:5000')` so Helm always pulls the fresh image. |
| Helmfile Layer 42 (`infra/kubernetes/helmfiles/42-db-migrations.yaml`) | Deploys the `db-migrations` Helm release (`infra/kubernetes/charts/platform/db-migrations`). Depends on core infra layers and Layer 15 secrets. |
| Helm chart (`infra/kubernetes/charts/platform/db-migrations`) | Runs Flyway as a pre-install/upgrade Job. Migrates platform + satellite databases (agents, graph, transformation, threads, agents-runtime, workflows, LangGraph). Validates credentials before any Flyway command. |
| Secrets bundle (`vault-secrets-platform`) | Layer 15 release that materialises `platform-db-credentials`, `agents-db-secret`, `graph-db-credentials`, etc., from Vault in every environment (local/staging/prod). |
| Bootstrap helper (`scripts/infra/bootstrap-db-migrations-secret.sh`) | Applied before migrations. Creates missing database secrets from Vault defaults, then verifies each secret exposes the required `*_DATABASE_URL/USER/PASSWORD` keys. Exits early with guidance if anything is missing. |
| Smoke tests (`infra/kubernetes/values/tests/secrets-smoke.yaml`) | CI job that asserts all Layer 15 secrets reconcile (including the data-plane bundle) before migrations run. |
| CLI entry (`infra/kubernetes/scripts/run-migrations.sh`) | Wrapper that triggers the image build, ensures secrets, runs `helm upgrade --install --wait --atomic`, streams job logs, and prints Flyway status. Exposed via Tilt (`ops-db-migrate-platform`) and `pnpm migrate`. |

### Execution flow (local & CI)

1. **Rebuild Flyway image** – Tilt auto-rebuilds on file changes to `db/flyway/sql/**`; manually trigger with `tilt trigger infra:42-db-migrations-run` to rebuild and push `k3d-dev-reg:5000/ameide/ameide-migrations:dev`.
2. **Ensure secrets** – `run-migrations.sh` invokes the bootstrap script; if any secret/key is missing, it fails immediately with instructions to fix Vault / rerun Layer 15.
3. **Helm upgrade** – Layer 42 release uses the freshly pushed image. Hooks run `flyway validate/migrate` sequentially for each database, printing before/after schema versions.
4. **Result surface** – Tilt UI + CLI log the job output; `kubectl logs -n ameide job/db-migrations` remains the single source of truth.
5. **Verification** – CI smoke tests (`secrets-smoke` + future `flyway-smoke`) gate infra layers. Integration jobs (threads, workflows, etc.) run after migrations to prove schema compatibility.

### Secrets & credentials

* `vault-secrets-platform` charts define the ExternalSecrets for each database.
* `scripts/vault/ensure-local-secrets.py` deterministically generates local values for all `*_DATABASE_URL/USER/PASSWORD` keys (and other Vault entries) and seeds k3d clusters during bootstrap.
* The bootstrap script only **verifies** secrets. This enforces the principle in backlog/362 that secrets originate from Vault, not helper scripts.

### Image management (tying to backlog/343)

* Images are referenced via `registry_image('ameide-migrations')` (which resolves to `ameide/ameide-migrations`).
* Tilt’s `default_registry` rewrites host vs. cluster registries, eliminating the old “docker build to ameide/… then k3d import” divergence.
* Because Helm now pulls the same tag we pushed, Flyway jobs always run the latest SQL (e.g., `V3__threads_add_tenant_metadata_columns.sql` finally landed once this refactor was done).

---

## Current State (Mar 2026)

| Area | Status | Notes |
| --- | --- | --- |
| Image workflow | ✅ | Ops-migrations-image uses `docker build` + push; Helm pulls refreshed tags automatically. |
| Secrets guardrail | ✅ | Layer 15 data-plane bundle + bootstrap verification + secrets-smoke CI. |
| Fail-fast Flyway guardrail | ✅ | db-migrations Job validates every secret before running Flyway; CLI exits early if credentials missing. |
| Documentation | ✅ | `infra/kubernetes/charts/platform/db-migrations/README.md`, environment READMEs, backlog/362 cover the process. |
| CI coverage | ✅ | Secrets smoke test validates all db secrets. Future: add "flyway info" job to verify schema versions post-Layer 42. |
| Multi-environment parity | ✅ | Layer 15 ensures the same secrets exist in local/staging/prod; Helm chart identical across envs (values files handle differences like image tag). |

---

## Roadmap / Open Items

1. **CI Flyway smoke test** – Add a tekton/Tilt job that runs `flyway info` (read-only) post Layer 42 to confirm latest versions before app deploys.
2. **Schema drift dashboard** – Expose Flyway schema versions via metrics or a status page so operators can see per-database state without tailing job logs.
3. **Automated rollback guidance** – Document how to trigger targeted migrations (e.g., only threads DB) for hotfixes; consider parameterizing the Helm chart for per-db toggles.
4. **Flyway upgrade cadence** – Track upstream Flyway + PostgreSQL support matrix (log currently warns about Postgres 17). Schedule quarterly upgrades so we stay within supported ranges.
5. **Archive legacy tooling** – Remove obsolete scripts (`scripts/flyway.sh`, old docker-compose helpers) once every consumer uses `run-migrations.sh` / Tilt.

---

> **Note:** backlog/343:220 also references `ops-migrations-image` and needs updating to reflect the inline `docker_build()` pattern.

## References

* `infra/kubernetes/charts/platform/db-migrations/README.md` – Chart usage + CLI helper details.
* `infra/kubernetes/scripts/run-migrations.sh` – Canonical wrapper.
* `Tiltfile` (section “Database Migrations Image”) – Image build/push logic following backlog/343.
* `backlog/362-unified-secret-guardrails.md` – Secret & guardrail tracking.
* `backlog/343-tilt-helm-north-start-v3.md` – Registry/image injection rules referenced above.
