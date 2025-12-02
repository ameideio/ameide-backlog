# 422: Plausible ArgoCD alignment (dev)

## What Plausible now does

- **External DBs only.** `DATABASE_URL` points to the CNPG Postgres service (`platform-postgres-clusters-rw.ameide.svc:5432/plausible`) using the CNPG-owned secret `plausible-db-credentials` (username/password keys). `CLICKHOUSE_DATABASE_URL` / `_READ` point to the Altinity ClickHouse HTTP endpoint on 8123 with creds from `clickhouse-auth` (user `plausible`).
- **Vendor init flow enabled.** Deployment now ships an initContainer that runs `/entrypoint.sh db createdb && /entrypoint.sh db migrate` before `run`. This creates/updates both Postgres and ClickHouse schema exactly as the Plausible image expects.
- **Secrets aligned with CNPG ownership.** The Deployment no longer consumes the cluster admin secret `postgres-ameide-auth`; it uses the per-app CNPG secret (`plausible-db-credentials`), satisfying backlog/412 (CNPG owns Postgres creds).
- **Flyway opt-out.** Plausible is removed from the Flyway image and db-migrations Job; schema creation/migrations are solely handled by the Plausible entrypoint. Remaining db-migrations values (local/staging/production) no longer list `plausible-db-credentials`, so Flyway stops mounting Plausible creds entirely.

## Issues encountered

1) **Schema ownership conflict**
   - Flyway tried to own Plausible schema via `plausible-db-credentials`, while the runtime used the shared admin secret. ClickHouse schema was unmanaged because Plausible entrypoint migrations were never run.
   - Fix: Runtime now uses `plausible-db-credentials` and an initContainer to run Plausible’s entrypoint migrations; Flyway no longer touches Plausible.

2) **ArgoCD deletion loop on apps-plausible**
   - The Application was stuck deleting (seed Job finalizer), leaving Deployment missing.
   - Fix: Cleared finalizers, let the ApplicationSet recreate; now Synced/Healthy with the initContainer in place.

## How to verify (dev)

- `argocd app get argocd/apps-plausible` should show Synced/Healthy and the Deployment spec containing `initContainers: plausible-db-init` with the entrypoint command.
- Pods should start with the init phase completing and main container running `run`; ClickHouse/Postgres should have the Plausible tables post-start.
- No Plausible-related Flyway locations or secrets remain in `data-db-migrations` values or Job template.

## Guardrails / follow-ups

- Keep Plausible schema exclusively managed by Plausible entrypoint migrations; do not reintroduce it into Flyway.
- Mirror the initContainer enablement and CNPG secret wiring in any non-dev overlays.
- Monitor first rollout after image bumps to ensure `/entrypoint.sh db migrate` succeeds against ClickHouse/Postgres.***
