> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# Secrets Layer Modularization

> **Superseded:** Active work now lives in `backlog/362-unified-secret-guardrails.md`. This backlog remains as historical reference.

**Created:** Nov 2025  
**Owner:** Platform DX / Infrastructure

## Context

See also:
- backlog/348 – Environment & Secrets Harmonization
- backlog/360 – Secret Guardrail Enforcement

Layer 15 (`secrets-certificates`) currently bundles Vault fixtures, External Secrets, and environment-specific credentials for optional services (Argo CD, observability, Temporal, etc.). As environments diverge, the shared `vault-secrets` manifest has accumulated namespace assumptions that break local bootstrap flows (for example, attempting to manage Argo CD secrets when the namespace is absent). The recent split of Argo CD credentials into `vault-secrets-argocd` demonstrates a healthier pattern where optional integrations live in their own release.

## Problem

- Shared manifests mix baseline prerequisites with optional service wiring, causing Helm installs to fail when an optional namespace or CRD is missing.
- Environment overlays (e.g., local vs. staging vs. production) require brittle `installed` flags to disable unwanted resources.
- Ownership boundaries are unclear: service teams expect their charts to manage credentials, but the layer’s raw manifests mask dependencies.

## Goals

- Modularize Layer 15 so each optional integration has a dedicated release/value file independent from the baseline Vault wiring.
- Ensure clusters without a given service (e.g., Argo CD) never attempt to reconcile its secrets.
- Clarify documentation and ownership so service-specific credentials are maintained alongside their charts.
- Preserve deterministic bootstrap for all environments (local DevContainer, staging, production) without manual toggles.

## Proposed Approach

1. **Inventory optional integrations**  
   Audit `infra/kubernetes/values/platform/vault-secrets.yaml` (and related values) for manifests that target namespaces outside the Layer 15 baseline. Candidate extractions: Argo CD, observability stack (Prometheus/Alertmanager/Grafana), Temporal, MinIO, etc.

2. **Create dedicated releases**  
   For each optional integration:
  - Add a values file under `infra/kubernetes/values/platform/` (e.g., `vault-secrets-observability.yaml`) containing only the manifests/keys required for that service. Temporal now relies on CNPG-managed Secrets instead of a Vault bundle.
   - Introduce a `vault-secrets-<service>` release in `infra/kubernetes/helmfiles/15-secrets-certificates.yaml` whose `installed:` flag can be controlled via the per-environment overlay.
   - Document dependencies (`needs`) so the release only runs after its service namespace is created (potentially in later layers).

3. **Align service charts**  
   Where possible, push secret provisioning into the service’s own chart or layer. For example, Temporal’s Helm release could manage its ESO manifests instead of relying on Layer 15.

4. **Documentation updates**  
   - Update `backlog/339-helmfile-refactor.md` and `backlog/340-secrets.md` to describe the modular structure and how to enable/disable optional bundles.
   - Add a runbook for enabling a new optional service: add namespace chart → enable `vault-secrets-<service>` release → confirm External Secrets health.

5. **CI/Bootstrap validation**  
   Expand `infra/kubernetes/scripts/infra-health-check.sh` to keep the layered Helmfile sync/tests green and document when optional services rely on their own charts for ExternalSecrets.

## Status – 7 Nov 2025

- Layer 15 now emits discrete releases:
  - `vault-secret-store` renders only the shared `ClusterSecretStore` (plus the local Vault sync hook).
  - `vault-secrets-cert-manager`, `vault-secrets-k8s-dashboard`, `vault-secrets-neo4j`, and `vault-secrets-observability` each ship a narrow set of ExternalSecrets (Azure DNS, dashboard OAuth, Neo4j, Grafana/Loki/Tempo/MinIO). Temporal DB/visibility now come from CNPG, so no `vault-secrets-temporal` bundle remains.
  - `vault-secrets-platform` and `vault-secrets-argocd` depend on the secret store to ensure consistent ordering.
- Shared values (Vault endpoint, secret key names) live in `infra/kubernetes/values/platform/vault-secrets.base.yaml`. Module-specific manifests live beside it (for example `vault-secrets-observability.yaml`), and each release now exposes an environment overlay file (e.g., `infra/kubernetes/environments/<env>/platform/vault-secrets-observability.yaml`) where operators can set `installed: false` if a cluster must skip that bundle.
- Helmfile `15-secrets-certificates` now wires the per-release overlays into `installed:` clauses so optional bundles can be disabled without touching the shared manifests. `secrets-smoke` was updated to wait on the new releases before running readiness checks.
- Optional integrations that already own their ExternalSecrets inside their Helm charts (pgAdmin, Plausible, Langfuse bootstrap) are tracked below so operators know no Layer 15 release is required for those namespaces. New optional services should either follow that pattern or add a dedicated `vault-secrets-<service>` bundle with matching documentation.

## Deliverables

- Refactored Layer 15 Helmfile with per-service releases.
- New values files for each optional integration.
- Updated documentation covering the modular pattern.
- Validation scripts/tests ensuring optional services do not leak into environments where they are disabled.

## Recent Updates (Dec 2025 – Mar 2026)

- Added a dedicated `vault-secrets-platform` release plus per-environment override hook so the platform DB credential ExternalSecret is owned by Layer 15 instead of the application chart.
- Local, staging, and production overlays now rely on the shared `platform-db-credentials` secret, keeping the guardrail that forbids inline PostgreSQL passwords while unblocking Tilt deployments.
- **2026-03-09 – Optional namespaces managed by service charts:**  
  - `pgadmin` renders its OAuth client + bootstrap credentials through `infra/kubernetes/charts/platform/pgadmin/templates/externalsecret.yaml`, so enabling the pgAdmin Helm release no longer depends on Layer 15.  
  - `plausible` provisions both the application secret and admin bootstrap credentials via `infra/kubernetes/charts/platform/plausible/templates/externalsecrets.yaml`.  
  - `langfuse`/`langfuse-bootstrap` rely on the chart-owned ExternalSecrets in `infra/kubernetes/charts/platform/langfuse-bootstrap/templates/externalsecrets.yaml`, keeping the Local/AKV Vault keys in sync.  
  Documenting these ownership boundaries satisfies the outstanding “carve out optional bundles” action for pgAdmin, Plausible, and Langfuse without introducing additional Helmfile releases.
- **2026-03-09 – Helmfile simplification & smoke wait tuning:** Layer 15 releases now read `installed` directly from their per-environment `15-secrets.values.yaml` files (no more `vault-secret-bundles.yaml` indirection), eliminating the brittle `.Values | dig` path and letting clusters opt out by setting `installed: false` in the shared values file. The `secrets-smoke` job gained configurable ExternalSecret wait knobs so Vault materialization tests have enough runway on slower clusters, and the `vault-secret-store` chart now emits only the canonical `ameide-vault` store (the legacy alias was removed once every workload referenced `ameide-vault` directly).

## Outstanding work – Local db credentials (Mar 2026)

- The `db-migrations` Helm hook now enforces the presence of `graph-db-credentials`, `transformation-db-credentials`, `threads-db-credentials`, and `agents-runtime-db-secret` before Flyway runs. On fresh local clusters (and in CI) those secrets are missing because the corresponding service charts are disabled, so `kubectl logs -n ameide job/db-migrations` reports “Missing … database URL/user/password” and the hook exits.
- Add a data-plane bundle (working name `vault-secrets-data-plane`) that lives in Layer 15 and renders the ExternalSecrets for the services listed above, along with their Vault key mappings. Per-environment overlays should default the bundle to `installed: true` so migrations and guardrails always see the credentials even when the workloads are not deployed.
- Extend the Vault fixture tooling (`scripts/vault/ensure-local-secrets.py`) and the local runbook (`infra/kubernetes/environments/local/platform/README.md`) so developers can reseed those credentials and force an ExternalSecret reconciliation without installing the services.
- Wire the new bundle into `secrets-smoke` and the Helm template validation so future regressions fail fast in CI/Tilt, and cross-reference backlog/360/backlog/348 when the guardrail revalidation is complete.

### Runbook – enabling or disabling optional secret bundles

1. **Edit the release overlay** – Open `infra/kubernetes/environments/<env>/platform/15-secrets.values.yaml` and set `releases.<bundle>.installed: false` (or true) as needed. Each Layer 15 bundle’s settings live in that single file so overrides stay localized.  
2. **Render + apply Layer 15** – Run `infra/kubernetes/scripts/run-helm-layer.sh infra/kubernetes/helmfiles/15-secrets-certificates.yaml` with `HELMFILE_ENVIRONMENT=<env>` so Helm installs (or removes) the releases based on the updated overlay.  
3. **Validate guardrails** – Execute `infra/kubernetes/scripts/infra-health-check.sh <env>` and inspect the `secrets-smoke` job output. The smoke tests now wait up to five minutes (configurable via `EXTERNALSECRET_WAIT_ATTEMPTS`/`EXTERNALSECRET_WAIT_SLEEP`) for ExternalSecrets to report `Ready=True`. Failures indicate misconfigured overlays or Vault materialization regressions.  
4. **Document ownership** – When adding a brand-new optional namespace, decide whether the service chart owns its ExternalSecrets (pgAdmin/Plausible/Langfuse pattern) or if a new `vault-secrets-<service>` bundle is required. Update this backlog and the overlay comments accordingly so operators know which runbook to follow.

## Risks

- Splitting manifests may introduce ordering issues if optional releases expect namespaces or CRDs created later in the pipeline.
- Service teams must coordinate to keep their credential manifests aligned with chart changes.

Mitigation: Use Helmfile `needs`, document dependencies, and run the refactored layers through CI and DevContainer bootstrap before rollout.
