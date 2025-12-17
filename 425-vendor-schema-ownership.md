# 425 – Vendor-owned schemas vs. Flyway-owned schemas

## Problem / Proposal

Now that Temporal’s persistence schemas are handled by the community Temporal operator (via `TemporalCluster` reconcile and its setup/update Jobs), we have a concrete pattern for how vendor-owned schemas should be managed. This backlog explores what it would take to pivot fully to that model:

- **Vendors own their internal schemas** – we never attempt to apply or version their tables via Flyway.
- **Our application schemas stay consolidated** – Flyway remains the single entry point for Ameide-authored services.

## Current state

| Schema domain                      | Owner today                          | Tooling                                     |
|-----------------------------------|---------------------------------------|---------------------------------------------|
| Temporal default/visibility DBs   | Temporal (vendor-owned) via `data-temporal` schema hook | `temporal-sql-tool` (setup-schema, update)  |
| Keycloak, agents, workflows, etc. | `data-db-migrations` Argo app (Flyway) | Custom Flyway SQL under `/flyway/sql/**`    |

Some vendors (Temporal, CNPG) require their own tooling and provide clear “don’t touch” guidance. Others (e.g., third-party apps installed via Helm) may or may not have opinionated schema tooling.

## Desired end state

1. **Every vendor-provided product with protected schemas** declares and owns a dedicated migration component, similar to Temporal:
   - Example: if we adopt upstream Keycloak schema management (Liquibase or bundled tools), we create a `platform-keycloak-migrations` Argo app that runs their scripts before the Keycloak chart deploys.
   - Each vendor app’s Argo component lists its schema hook in dependencies to guarantee ordering.
2. **Flyway only manages Ameide-authored schemas**:
   - Services we build (agents, workflows, PG schemas for platform components) continue to share the `data-db-migrations` hook.
   - Flyway’s `existingSecrets` list shrinks to only Ameide-owned DB credentials; vendor secrets move to their respective hooks.
3. **Documentation and runbooks** explicitly call out which Argo app owns each schema and which tool to use for emergency fixes.

## Workstreams / Steps

1. **Inventory vendor-managed schemas**:
   - List every Helm chart / operator that owns its own persistence (Temporal, Keycloak, Prometheus, etc.) and determine whether upstream tooling exists for schema setup/upgrades.
   - Decide per vendor whether it’s worth delegating schema ownership (e.g., Keycloak may still be simpler via Flyway if upstream tooling is painful).
2. **Template a “vendor schema hook” chart**:
   - Base it on the **GitOps hook pattern** we use for Temporal preflight (`temporal-db-preflight`): small, idempotent, retryable, and safe-by-default (fail-fast in staging/prod, optional auto-fix only when explicitly enabled).
   - Parameterize for `install_pkg`, CLI download, and command sequences.
3. **Migrate candidates**:
   - For each vendor we choose to delegate, add a new Argo component and shared values file (mirroring Temporal’s structure).
   - Remove the vendor’s secrets/env vars from `data-db-migrations`.
   - Update ordering in the ApplicationSet so vendor schemas run before their Helm chart, but after infrastructure (CNPG, Vault secrets).
4. **Update runbooks**:
   - Document how to rerun each vendor schema hook (e.g., `argocd app sync platform-keycloak-migrations`).
   - Note which tool/logs to inspect when a vendor schema fails.

## Risks / Open questions

- Some vendors may not have a CLI suitable for automation. If we cannot run their tool non-interactively, Flyway (or our own scripts) might remain the only option.
- Additional Argo apps increase dependency graph complexity. We need clear waves/rollout-phase annotations so that vendor schema hooks run before their workloads but after databases and secrets.
- Auditing: splitting secrets across multiple hooks means we must ensure each hook has only the credentials it needs and that RBAC remains tight.

## Definition of done

- Temporal pattern documented and applied to any other vendor that requires it.
- `data-db-migrations` owns only Ameide-authored schemas; all vendor internals are migrated elsewhere.
- Runbooks clearly specify “which Argo app to sync when schema X needs attention”.

## Step 1 – Inventory snapshot (2025-11-30)

| Component / Chart                                      | DB backend / storage                                                                             | Current migration owner                    | Vendor tooling available? | Notes / Next action                                                                                   |
|-------------------------------------------------------|--------------------------------------------------------------------------------------------------|-------------------------------------------|---------------------------|--------------------------------------------------------------------------------------------------------|
| **Temporal** (`data-temporal`)                        | Postgres (`temporal`, `temporal_visibility` in CNPG cluster)                                      | Temporal operator (reconcile)             | Yes (`temporal-sql-tool`) | ✅ Operator owns schema setup/update; no Flyway involvement.                                            |
| **Keycloak** (`platform-keycloak`)                    | Postgres (`keycloak` DB managed by CNPG)                                                         | `data-db-migrations` (Flyway)             | **Yes** (Keycloak uses embedded Liquibase auto-migrations) | Needs exploration: upstream chart can run Liquibase on startup; evaluate replacing Flyway SQL with vendor hook. |
| **Prometheus / Alertmanager / Grafana stack**         | Prometheus TSDB, Alertmanager files, Grafana SQLite/Postgres (we currently run SQLite)           | Not applicable / in-container migrations  | Yes (managed internally)  | No relational schema we touch; nothing to migrate.                                                     |
| **MinIO, ClickHouse, Redis, CNPG operators**          | Each operator manages its own CRDs/state; no Ameide-authored SQL                                 | Operators themselves                      | Yes (operators reconcile) | Out of scope (no Flyway involvement today).                                                            |
| **Platform services (agents, workflows, threads, etc.)** | Postgres schemas (CNPG)                                                                           | `data-db-migrations` (Flyway)             | N/A (our code)            | Remain consolidated under Flyway.                                                                      |

Action item from this inventory: **Keycloak** is the only vendor product still using the shared Flyway job despite having native schema tooling (Liquibase baked into the Keycloak image). Next steps for step 2:

1. Validate whether the upstream Keycloak Helm chart exposes a hook/Job for Liquibase (or if we need to run the image with `KEYCLOAK_ADMIN` environment and `kc.sh start --optimized` to trigger migrations).
2. Decide if switching reduces risk vs. current Flyway SQL (which we maintain ourselves).
3. If yes, template a `platform-keycloak-migrations` chart following the Temporal pattern and remove Keycloak tables from Flyway.

If we uncover other vendor schemas later (e.g., future vendors with their own tooling), add them to this table.
