> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# backlog/360 – Secret Guardrails Completion

> **Superseded:** Guardrail status is consolidated in `backlog/362-unified-secret-guardrails.md`. Leave this backlog unchanged except for archival notes.

**Created:** Nov 2025  
**Owner:** Platform DX / Infrastructure  
**Related work:** backlog/338 (DevContainer Startup Hardening), backlog/341 (Local Secret Store), backlog/348 (Environment & Secrets Harmonization), backlog/355 (Secrets Layer Modularization)

## Context

Backlog 348 drove the initial sweep to separate config from secrets, but several services still rely on dev defaults, inline credentials, or lack validation. Backlog 355 called for modularizing Layer 15 so optional integrations ship their own Vault releases; the new `vault-secret-store` family covers observability/Temporal/MinIO, yet we still need to finish the remaining bundles (pgAdmin/Plausible/etc.) and add CI guardrails so regressions are caught early.

## Problem

1. **Service-level gaps** – Agents, agents-runtime, db-migrations, threads, transformation, workflows, and Langfuse still have to-do items from backlog 348 (e.g., move `environment` out of shared values, finish validation checklists, document rotation runbooks).  
2. **Layer 15 follow-through** – Observability/Temporal/MinIO now live in dedicated releases, but other optional namespaces (pgAdmin, Plausible, Langfuse add-ons) still rely on shared manifests, and nothing ensures their release overlays set `installed` consistently across environments.  
3. **No automated enforcement** – Helm template/lint jobs do not yet assert that required secrets exist, nor do we have smoke checks ensuring optional releases stay disabled where expected.

## Gap Log

### Outstanding items from backlog/348

| Service | Gap | Source |
| --- | --- | --- |
| graph / threads / transformation / agents-runtime | ⚠ Regression – db-migrations fails unless their database secrets exist before the charts install. Need Layer 15 bundle + local fixture/runbook updates so the guardrail passes. | backlog/348-envs-secrets-harmonization.md + backlog/355 |

### Recently completed (service scope)

- **2025-11-07 – Agents**: Removed shared MinIO defaults, enforced Helm `required(...)` on both MinIO config and secret references, and validated with `helm template` (see backlog/348-envs-secrets-harmonization.md, “agents”). This closes the agents row from the gap log.
- **2025-11-07 – Layer 15 modularization**: Split the monolithic `vault-secrets` release into service-scoped bundles (`vault-secret-store`, `vault-secrets-cert-manager`, `vault-secrets-k8s-dashboard`, `vault-secrets-neo4j`, `vault-secrets-observability`) and retained the dedicated `vault-secrets-platform`/`vault-secrets-argocd` bundles. Temporal moved to CNPG-owned secrets, so no `vault-secrets-temporal` bundle remains. Each release consumes the shared `vault-secrets.base.yaml`, and per-release overlays now control `installed`, keeping optional integrations explicit without a separate toggle file. `secrets-smoke` was updated to gate on the new releases so guardrails run automatically.
- **2025-11-07 – Agents-runtime**: Cleared default MinIO/OTEL values from the chart, added `required(...)` guardrails for config + secrets, and validated via `helm template` with local overlays (see backlog/348-envs-secrets-harmonization.md, “agents-runtime”). Gap removed.
- **2025-11-07 – db-migrations**: Deleted the inline secret template, removed the `createSecret` toggle, required `existingSecrets`, and validated the chart with local overlays (see backlog/348-envs-secrets-harmonization.md, “db-migrations”). The db-migrations gap is now closed.
- **2025-11-07 – inference-gateway**: Added optional Vault-backed Redis credentials plus `required(...)` guards for `environment`, OTEL endpoint, and Redis config when enabled; validated via `helm template` with local overlays (see backlog/348-envs-secrets-harmonization.md, “inference-gateway”). Gap removed.
- **2025-11-07 – Threads**: Ran helm template for local/staging overlays to confirm Vault-only credential flow; validation checkbox flipped in backlog/348 (“threads”).
- **2025-11-07 – Transformation**: Helm template validation (local/staging) completed; no outstanding plaintext credentials (see backlog/348, “transformation”).
- **2025-11-07 – Workflows**: Helm template validation (local/staging) completed; remaining env-var cleanup deferred (see backlog/348, “workflows”).
- **2025-11-07 – Langfuse**: Documented the rotation/runbook steps for `langfuse-secrets` and `langfuse-bootstrap` credentials inside backlog/348 (“langfuse”), closing the outstanding doc gap.
- **2025-11-07 – Helm template CI scaffold**: Added `infra/kubernetes/scripts/validate-hardened-charts.sh`, which renders the hardened charts (agents*, db-migrations, inference-gateway, threads, transformation, workflows) for the local/staging overlays. Wire this script into CI next so regressions surface automatically.
- **2026-03-09 – Layer 15 smoke hardening**: `infra/kubernetes/helmfiles/15-secrets-certificates.yaml` now reads `installed` directly from each environment values file (no shared toggle file), and the `secrets-smoke` test gained configurable wait knobs (`EXTERNALSECRET_WAIT_ATTEMPTS` and `EXTERNALSECRET_WAIT_SLEEP`), giving ExternalSecrets up to five minutes to reconcile before failing. `vault-secret-store.yaml` now provisions the canonical `ameide-vault` store; the temporary `vault-secret-store` alias was removed once all charts targeted `ameide-vault` directly.
- **2026-03-09 – CI enforcement**: The Helm lint workflow (`.github/workflows/ci-helm.yml`) now invokes `infra/kubernetes/scripts/validate-hardened-charts.sh`, so PRs fail immediately when the hardened charts violate `required(...)` guardrails or reference missing ExternalSecrets.

### Regression – local db-migrations guard (Mar 2026)

Running `bash infra/kubernetes/scripts/run-migrations.sh` against a fresh k3d cluster now fails before Flyway executes. The `db-migrations` hook validates `GRAPH_DATABASE_*`, `TRANSFORMATION_DATABASE_*`, `THREADS_DATABASE_*`, and `AGENTS_RUNTIME_DATABASE_*` variables, but those secrets (`graph-db-credentials`, `transformation-db-credentials`, `threads-db-credentials`, `agents-runtime-db-secret`) are rendered only after their service charts install. Because local workflows run migrations long before those releases, `kubectl logs -n ameide job/db-migrations` shows “Missing … database URL/user/password” for every absent secret and the job exits with `BackoffLimitExceeded`.

Actions to close the regression:

- Teach Layer 15 (backlog/355) to pre-provision these database secrets even when the service Helm releases are disabled. A dedicated `vault-secrets-data-plane` (or equivalent) bundle keeps the guardrails aligned with the modular secret store.
- Expand the local secret fixtures (`scripts/vault/ensure-local-secrets.py`) and `infra/kubernetes/environments/local/platform/README.md` so developers can re-seed the credentials without installing the services, then rerun `kubectl annotate externalsecret ... --overwrite` to force reconciliation.
- Update `infra/kubernetes/scripts/run-migrations.sh` / `scripts/infra/bootstrap-db-migrations-secret.sh` to fail fast when any of the required secrets is missing, and add CI coverage (either via `validate-hardened-charts` or a smoke test) that exercises this path so regressions surface before release.

### Outstanding items from backlog/355

| Integration | Gap | Source |
| --- | --- | --- |
| Observability stack (Grafana/Prometheus/Alertmanager/Loki/Tempo) | ✅ Completed (Nov 7 2025). Secrets now rendered by `vault-secrets-observability`; disable the bundle by setting `installed: false` in the release overlay if needed. | backlog/355-secrets-layer-modularization.md |
| Temporal | ✅ Completed (Nov 7 2025). CNPG owns the Temporal DB + visibility secrets; Vault is optional as a mirror only (no `vault-secrets-temporal` bundle). | backlog/355-secrets-layer-modularization.md |
| MinIO / S3 | ✅ Completed (Nov 7 2025). MinIO service users live inside the observability bundle; Grafana/Loki/Tempo releases now depend on the modular Layer 15 secrets. | backlog/355-secrets-layer-modularization.md |
| pgAdmin / Plausible / Langfuse | ✅ Completed (Mar 9 2026). Each service chart now owns its ExternalSecrets, so enabling those namespaces no longer depends on Layer 15 bundles. | backlog/355-secrets-layer-modularization.md |
| Automation | Helmfile does not yet include additional `vault-secrets-<service>` releases; CI / health-check scripts do not verify optional releases. | backlog/355-secrets-layer-modularization.md |
| Data-plane app DB secrets (graph/threads/transformation/agents-runtime) | ⚠ Pending (Mar 11 2026). db-migrations now fails unless these secrets exist before the service charts install; Layer 15 needs a bundle (or hook) that always provisions them plus local fixtures so Tilt/CI runs succeed. | backlog/355-secrets-layer-modularization.md |

## Goals

1. Close every outstanding action item in backlog 348 (agents, agents-runtime, db-migrations, inference-gateway redis auth, langfuse runbook, threads/transformation/workflows validation).  
2. Split the remaining optional integrations out of `vault-secrets`: observability (Grafana/Prometheus/Alertmanager/Loki/Tempo), Temporal, MinIO/S3, pgAdmin, and any other namespace-specific bundles.  
3. Add CI guardrails that render each chart with local/staging values and fail when `required(...)` constraints or ExternalSecret references are missing.  
4. Extend the infra health check to keep layered Helmfile runs green and surface ExternalSecret regressions automatically.

## Scope

- Services tracked in backlog 348 with open or regressed tasks: agents, agents-runtime, db-migrations, graph, threads, transformation, workflows, inference-gateway (future Redis creds), langfuse (app + bootstrap), plus any chart whose secrets must exist before db-migrations runs.
- Optional secret bundles listed in backlog 355: observability stack, Temporal, MinIO, pgAdmin/Plausible/Langfuse, and the new data-plane database bundle required for migrations.
- Tooling updates: Helm template CI job, `scripts/infra/infra-health-check.sh` (or equivalent), documentation in each chart README + Layer 15 runbook.

## Approach

1. **Per-service cleanups**
   - Agents/agents-runtime: move `environment` out of shared values, relocate MinIO/OTEL knobs to overlays, remove inline secrets.
   - db-migrations: finish the refactor PR by mandating Vault secrets and documenting ENV requirements.
   - Langfuse: document rotation + bootstrap steps for Vault keys.
   - Threads/Transformation/Workflows: run helm template + Tilt smoke tests and record the validation status; remove leftover plaintext fallbacks.
   - Inference-gateway: design Redis credential wiring (ExternalSecret + values) and land it once buffering is enabled.

2. **Layer 15 modularization**
   - Maintain per-service values (`vault-secrets-*.yaml`) plus per-environment overlays that can set `installed: false` when a cluster must skip a bundle; keep the `needs` relationships so ordering stays deterministic.
   - PgAdmin, Plausible, and Langfuse now provision their own ExternalSecrets inside the service charts; future optional namespaces should explicitly document whether they follow that pattern or require a dedicated `vault-secrets-<service>` bundle.

3. **Automation**
   - Extend CI to run `helm template` for every chart + environment overlay; fail when templates emit `required(...)` errors or ExternalSecret refs are missing.
   - Keep `infra/kubernetes/scripts/infra-health-check.sh` running the layered Helmfile sync/tests so ExternalSecret regressions and release failures surface immediately.

4. **Documentation**
   - Append outcomes to backlog 348 per-service sections, noting completion dates and validation evidence.
   - Update Layer 15 README / backlog 355 references to describe the new modular structure and validation workflow.

## Deliverables

- Updated values/overlays for the services above with dev defaults removed and validation checklists completed.
- Separate `vault-secrets-<service>` releases plus values files for each optional integration (observability/Temporal/MinIO ✅, pgAdmin and future services pending).
- Dedicated Layer 15 (or bootstrap) coverage for the graph/threads/transformation/agents-runtime database secrets so `db-migrations` guardrails always find them, even when those applications stay disabled.
- CI job (or GitHub Action) that templates each chart with local + staging values; pipeline fails on missing secrets or invalid URLs.
- Enhanced infra health check script + documentation for enabling/disabling optional secret bundles via the release overlays.
- Backlog documentation updated with completion notes and regression follow-ups.

## Recent Updates (Dec 2025 – Mar 2026)

- Platform chart no longer renders its own ExternalSecret and now fails templating unless `postgresql.auth.existingSecret` is set in every environment (local/staging/production now point to `platform-db-credentials`).
- Layer 15 owns the platform DB ExternalSecret through the new `vault-secrets-platform` release, reinforcing the guardrail that secrets must originate from the shared store instead of inline chart defaults.
- **2026-03-11 – Tilt guardrail for ameide-vault:** `Tiltfile` automatically adds the baseline Helm layers (core proto + infra:10/12/15) to every `--only` run so the `vault-secret-store` release always reconciles `ClusterSecretStore/ameide-vault` before `www-ameide-platform` starts. This prevents the `externalsecret ... could not get ClusterSecretStore "ameide-vault"` error that blocked local rollouts.
- Added `infra/kubernetes/charts/infrastructure/temporal-namespace-bootstrap` to Helmfile layer 45 so local clusters auto-create the `ameidemporal namespace via admin-tools before workflows-runtime guards execute; Tilt `helm upgrade workflows-runtime` now fails fast only when Temporal itself is unreachable.
- **2026-03-11 – Local db-migrations regression triaged:** `kubectl logs -n ameide job/db-migrations` surfaced missing graph/transformation/threads/agents-runtime/langgraph secrets on fresh local clusters. This backlog now tracks the Layer 15 + documentation work required to seed those credentials before the hook runs.

## Risks & Mitigations

- **Ordering issues** when new `vault-secrets-*` releases depend on namespaces or operators from later Helmfile layers. Mitigate by using `needs` and documenting prerequisites.  
- **CI noise** if helm template runs without required secrets. Provide stub values via `.gotmpl` overlays or mock secrets to satisfy `required(...)`.  
- **Operational drift** once secrets move into service-specific releases. Assign ownership in chart READMEs and include rotation steps.
