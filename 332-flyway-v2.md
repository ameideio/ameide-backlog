> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# 332: Flyway Dual-Database Management (v2) _(Superseded – see backlog/363-migration-architecture.md)_

**Status:** In progress  
**Date:** 2025-11-04  
**Owner:** Platform DX / Infrastructure

---

## Why a v2?

The original Flyway rollout (v1) added dual-database migrations but still assumed Azure Key Vault as the source of truth for connection secrets. With the platform moving fully to HashiCorp Vault plus local bootstrap scripts, we need an updated plan that removes Azure dependencies and guarantees the `db-migrations` layer works after every devcontainer restart.

Key v2 goals:

1. Keep the dual-database Flyway pattern (platform + agents, plus any future databases).
2. Replace Azure Key Vault references with Vault-backed defaults and deterministic local secrets.
3. Ensure Layer 42 (database migrations) is self-sufficient: no manual secret creation, no flaky Flyway hooks.

---

## Current pain points

- **`FLYWAY_URL not set` errors** after reopening the devcontainer, caused by missing `db-migrations-local` secret.
- Multiple docs still talk about Azure Key Vault, confusing new contributors.
- Local migrations require manual secret creation or env var exports, while CI/Tilt paths rely on Vault.

---

## Updated architecture

```
Layer 42 (db-migrations Helmfile)
└── db-migrations chart
    ├── Pre-loads Vault fixtures via ensure-local-secrets.py (Layer 15 hook + devcontainer post-start)
    ├── run-helm-layer.sh calls bootstrap-db-migrations-secret.sh before syncing so ameide/db-migrations-local always exists
    └── Helm Job runs Flyway for each database (platform → agents → optional extras)
```

### Secret flow

1. `scripts/vault/ensure-local-secrets.py`
   - Seeds Vault with values produced by `scripts/vault/ensure-local-secrets.py`.
   - Configures External Secrets (ESO) to pull from Vault.

2. `scripts/infra/bootstrap-db-migrations-secret.sh`
   - Reads credentials from `platform-db-credentials` (ESO-managed).
   - Falls back to Vault defaults if the secret is not present yet.
   - Writes `ameide/db-migrations-local` containing:
     - `FLYWAY_URL` (JDBC-safe)
     - `FLYWAY_USER`
     - `FLYWAY_PASSWORD`

3. `infra/kubernetes/environments/local/platform/db-migrations.yaml`
   - References `db-migrations-local` in the `existingSecrets` list so the Helm job mounts it automatically.

Result: Flyway pre-upgrade hook always has connection info, even after a clean devcontainer rebuild.

---

## Required changes

### 1. Documentation updates

- Deprecate Azure Key Vault instructions in:
  - `backlog/332-flyway.md`
  - `infra/kubernetes/values/platform/azure-keyvault-secrets.yaml` (note removal)
  - Any README sections referencing `az keyvault`
- Point contributors to Vault defaults + bootstrap scripts instead.

### 2. Helm chart & values

- **Chart:** `infra/kubernetes/charts/platform/db-migrations`
  - Confirm `existingSecrets` includes `db-migrations-local` (already updated).
  - Ensure env fallbacks prefer Flyway-specific keys before generic `DATABASE_URL`.

- **Values:** `infra/kubernetes/environments/*/platform/db-migrations.yaml`
  - Local: keep `db-migrations-local` in `existingSecrets`.
  - Staging/production: ESO should publish the same keys (`FLYWAY_*`) via Vault.

### 3. Devcontainer bootstrap

- `.devcontainer/postStartCommand.sh`
  - Run `scripts/vault/ensure-local-secrets.py` before starting Tilt (already implemented via Helmfile hook).
  - Add log checkpoints so failures appear in post-start summaries.

### 4. Backlog alignment

- `backlog/339-helmfile-refactor.md`: mention that Layer 42 depends on the new bootstrap logic.
- `backlog/338-devcontainer-startup-hardening.md`: document the secret bootstrap under “Declarative building blocks”.
- **This document** (v2) should replace the v1 “Azure Key Vault Setup” steps with Vault-focused instructions.

---

## Verification plan

1. **Fresh devcontainer**  
   - `Reopen in Container`
   - Confirm post-start log shows:
     - Vault sync success
    - Trigger `infra:42-db-migrations` (auto-runs on start) and verify logs include `db-migrations Flyway secret ensured`
2. **Run migrations manually**
   ```bash
   bash infra/kubernetes/scripts/run-flyway-cli.sh
   bash infra/kubernetes/scripts/run-migrations.sh
   ```
   Expect both commands to succeed without exporting environment variables.
3. **Check secrets**
   ```bash
   kubectl get secret db-migrations-local -n ameide -o yaml
   ```
4. **Layer 42 smoke**
   ```bash
   tilt trigger infra:42-db-migrations
   ```
   Ensure the Helm job completes and smoke test gates pass.

---

## Rollout steps

1. Merge the bootstrap scripts and Helm value updates (already done).
2. Update documentation per the sections above.
3. Announce the new workflow:
   - No more `az keyvault secret set …` for Flyway.
   - Use Vault defaults or environment-specific ESO overlays.
4. Monitor migrations in CI and the first few developer restarts to confirm the new secret flow is stable.

---

## Open questions

- Do we still need Azure Key Vault for any migration-related credential? (Goal: **remove** dependency entirely.)
- Should we add a health check that asserts `db-migrations-local` exists before Layer 42 runs in CI?
- Do we want optional seed data (e.g., demo organizations) bundled in the same bootstrap script?

---

## Next actions

- [ ] Purge Azure Key Vault instructions from older docs.
- [x] Verify staging/production ESO overlays deliver `FLYWAY_URL` / `FLYWAY_USER` / `FLYWAY_PASSWORD`.
- [ ] Add a CI smoke target that runs `infra:42-db-migrations` after devcontainer bootstrap scripts.
- [ ] Archive v1 doc once all consumers migrate.
