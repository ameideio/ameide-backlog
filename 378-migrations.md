# Migrations in local GitOps (dev)

Summary: how database migrations are delivered in the dev GitOps flow, when they run, and how to verify/fix them.

## Flow overview
- AppSet: `environments/dev/argocd/apps/data-services.yaml` generates `data-db-migrations` (wave `24`, dependencyPhase `state`).
- Chart: `sources/charts/platform-layers/db-migrations` is hook-only (pre-install/pre-upgrade Job via Helm hook).
- Values: `_shared` + `dev` value files feed image/secrets and Flyway settings; schema setup in other charts is disabled in favor of this Job.
- Ordering: RollingSync waves run CNPG (wave 20) → migrations (wave 24) → Temporal (wave 26) → the rest of data services.
- Automation: AppSet template + templatePatch set `syncPolicy.automated` (prune/selfHeal + CreateNamespace/RespectIgnoreDifferences) and the AppProject allows auto-sync (manualSync=false), so migrations can auto-fire with the wave when the spec reflects automation.

## What to expect on a healthy sync
- `Application/data-db-migrations` carries `spec.syncPolicy.automated` and wave label `24`.
- A sync creates a Job named `data-db-migrations` (hook) in `ameide`; pod runs Flyway then exits Completed.
- No other schema setup jobs should run in Temporal charts (those are disabled).

## Verifying quickly
```bash
kubectl -n argocd get app data-db-migrations -o jsonpath='{.spec.syncPolicy}'  # expect automated present
kubectl -n ameide get pods -l job-name=data-db-migrations                      # should be Completed or absent post-hook
kubectl -n ameide logs -f job/data-db-migrations                               # Flyway output
```

## Forcing a rerun (safe)
```bash
kubectl -n argocd delete app data-db-migrations          # AppSet recreates with latest spec
argocd app sync data-db-migrations                       # runs the hook (use devcontainer wrapper)
kubectl -n ameide get pods -l job-name=data-db-migrations -w
kubectl -n ameide logs -f job/data-db-migrations
```

## Common failure modes
- Automation missing: App shows only syncOptions, no automated → delete app and let AppSet recreate after refresh; ensure AppProject manualSync=false.
- Hook never created: sync not triggered → run `argocd app sync data-db-migrations`.
- DB not ready: CNPG not Ready when migrations run → wait for `postgres-ameide` Ready, rerun migrations.
- Temporal crashloops `schema_version` missing: migrations not run or failed → rerun migrations, then restart Temporal deployments.

## Devcontainer CLI note
- `.devcontainer/postCreate.sh` installs `argocd-core` wrapper (`alias argocd=argocd-core`) that uses `--core` against the local cluster with `ARGOCD_NAMESPACE=argocd`. No port-forward/login needed for `argocd app sync ...`.

## Tilt helpers (manual migrations path)
- Tilt resources under the `ops.migrations` label let you build and run migrations without Argo:
  - `ops-migrations-image`: builds/imports the Flyway image (`infra/kubernetes/scripts/ensure-migrations-image.sh`).
  - `ops-flyway-cli`: drop into Flyway CLI (`infra/kubernetes/scripts/run-flyway-cli.sh`).
  - `ops-db-migrate-platform`: runs migrations locally (`infra/kubernetes/scripts/run-migrations.sh`).
  - `ops-db-migrations-logs`: tails the migration Job logs in `ameide`.
- These are manual-trigger local_resources (auto_init=false); use Tilt UI or `tilt trigger <resource>` when you need to rerun migrations outside the Argo wave.

## Alignment with Flyway layout
- Schema files live in `db/flyway/sql` (versioned `V__` files) and `db/flyway/repeatable` (`R__` files); documented in `db/flyway/README.md`.
- The Flyway image (`migrations/Dockerfile.dev` for Tilt/local helpers, `migrations/Dockerfile.release` for GitOps/CI) copies those directories; both paths stay in sync.
- Local value `migrations.seedDemoData` controls the demo seed repeatable; the Job sets the placeholder when enabled (see Flyway README and chart values).

## No Helmfile layers
- The legacy Helmfile “layer 42” flow no longer exists; migrations deploy via the Argo Application (`data-db-migrations`) or the direct script `infra/kubernetes/scripts/run-migrations.sh` (Helm install from `gitops/ameide-gitops/sources/charts/platform-layers/db-migrations`).
- Keep using the shared Flyway image/tag across Argo and Tilt so both paths run the same SQL.
