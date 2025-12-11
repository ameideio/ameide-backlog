> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# DevContainer Startup Hardening _(partially archived)_

> **Update (2026-02-17):** Historic references to Verdaccio/devpi/Athens image imports now apply only to the retired registry stack. Buf Managed Mode removed those steps; preserve the remainder of this backlog for the general bootstrap checklist.

**Created:** Nov 2025  
**Owner:** Platform DX / Developer Experience

## Goal

Related backlog:
- backlog/341 – Local Secret Store (provider abstraction + seeding)
- backlog/348 – Environment & Secrets Harmonization (service-level config/secret split)
- backlog/355 – Secrets Layer Modularization (Layer 15 refactor)
- backlog/360 – Secret Guardrail Enforcement (gap log + CI/health checks)

Ensure that “Reopen in Container” delivers a ready-to-code environment with no manual recovery steps: the local k3d cluster mirrors the repo, Helmfile converges, and platform smoke tests pass before Tilt starts.

## Declarative Building Blocks

| Layer | Artifact(s) | Purpose |
|-------|-------------|---------|
| Kubernetes cluster | `infra/docker/local/k3d.yaml`, `scripts/infra/ensure-k3d-cluster.sh` | Define the k3d topology, registry mirror, and port mappings; the helper hashes the config and recreates the cluster only when drift is detected (config change or missing registry container). |
| Buf SDK distribution | `packages/ameide_core_proto`, `packages/ameide_sdk_{go,ts,python}/scripts/sync*`, Buf BSR (`buf.build/ameideio/ameide`) | Managed Mode publishes generated SDKs to Buf; DevContainer bootstrap only verifies credentials and runs the sync scripts so services can install the published packages without standing up Verdaccio/devpi/Athens. |
| Vendored Helm dependencies | `infra/kubernetes/charts/third_party/charts.lock.yaml`, `infra/kubernetes/charts/third_party/`, `infra/kubernetes/scripts/vendor-charts.sh` | Cache upstream charts inside the repo so Helmfile runs entirely offline; the lock file pins versions and the script refreshes local `.tgz` archives. Operator charts now install their own CRDs from those archives (no shared bundle). |
| Platform deployment | `infra/kubernetes/helmfile.yaml`, `infra/kubernetes/helmfiles/{10,15,20,…}.yaml`, `infra/kubernetes/scripts/helmfile-sync.sh` | Apply layered Helmfiles in order (namespace bootstrap → secrets & certificates → core operators → control plane → auth & SSO → package registries → core/extended data plane → product services → QA smoke tests). The sync script drives the layers sequentially so cross-layer dependencies such as `namespace` remain satisfied, and per-layer Helm smoke suites plus `helmfile status` gate the flow. |
| Health gates | `infra/kubernetes/charts/helm-test-jobs/`, `infra/kubernetes/charts/platform-smoke/`, `infra/kubernetes/environments/*/platform/platform-smoke.yaml`, `infra/kubernetes/scripts/run-helm-layer.sh`, `infra/kubernetes/scripts/infra-health-check.sh` | Helm-native test jobs validate each layer (namespace, secrets, operators, control plane, auth & SSO, registries, core/extended data plane) plus the platform smoke chart. The shared runner (`run-helm-layer.sh`) performs the sync + test cycle, captures Helm output, and degrades “pod log missing” races to warnings so Tilt never hard-fails after a successful suite. |
| Secret bootstrap | `foundation-vault-bootstrap` (CronJob), `infra/kubernetes/scripts/run-helm-layer.sh` + `scripts/infra/bootstrap-db-migrations-secret.sh` | Vault is initialized/unsealed/seeded in-cluster, and the helper script creates the `db-migrations-local` secret (Flyway URL/user/password) during the Layer 42 sync so migrations run without manual intervention after every container restart. |
| Observability | `/home/vscode/.devcontainer-setup.log`, Tilt UI (`http://localhost:10350`) | Provide a single log stream and visual status once the bootstrap exits cleanly. |

## Bootstrap Flow (postCreateCommand)

1. **Cluster reconciliation**  
   `scripts/infra/ensure-k3d-cluster.sh` compares `infra/docker/local/k3d.yaml` with the live cluster. Hash mismatches or missing registry containers trigger a destroy/recreate; otherwise the cluster is simply started. Legacy registry containers (`dev-reg`, `k3d-docker.io`) are purged automatically.

2. **Helmfile orchestration**  
   `infra/kubernetes/scripts/helmfile-sync.sh local` drives the desired state:
   - Refreshes vendored chart archives from `infra/kubernetes/charts/third_party/` via `infra/kubernetes/scripts/vendor-charts.sh` and uses them directly (no remote repo access).
   - Applies the `foundation-vault-core` and `foundation-vault-bootstrap` releases: Vault comes online automatically, and the CronJob seeds fixture secrets plus Kubernetes auth state. During Tilt/Tiltfile reconciliation the Layer 42 helper invokes `scripts/infra/bootstrap-db-migrations-secret.sh`, ensuring `ameide/db-migrations-local` exists with Flyway connection info.
   - Executes each layer’s Helmfile separately (10-bootstrap → 15-secrets-certificates → …) so that `needs` references remain within the same state file; this guards against namespace install races when developers rerun isolated stages.
   - Applies layered Helmfiles in phase order: `10-bootstrap` (namespace), `15-secrets-certificates` (cert-manager, external-secrets manifests), `20-core-operators`, `22-control-plane`, `24-auth` (Postgres + Keycloak + realm bootstrap), `40-data-plane`, `45-data-plane-ext`, `50-product-services` (placeholder), and `60-qa`, with each phase’s Helm smoke release providing the immediate health gate.
   - Language SDK availability is validated via Buf during package sync scripts (`pnpm --filter @ameideio/ameide-sdk-ts run build`, `node packages/ameide_sdk_go/scripts/sync-proto.mjs`, `uv run python packages/ameide_sdk_python/scripts/sync_proto.py`); no in-cluster registries are primed.
   - Installs `platform-smoke` (local only) and executes Helm tests; `infra/kubernetes/scripts/run-helm-layer.sh` is invoked for each layer so the same behaviour powers both CLI automation and Tilt’s `infra:<layer>` resources. `infra/kubernetes/scripts/infra-health-check.sh` simply loops over the layers with that helper. All smoke jobs delete stale runs before creation to avoid “already exists” failures.
   - Emits a condensed bootstrap summary at exit covering registry images, Helmfile status, package registries, ExternalSecrets, and Tilt readiness, so developers can spot blockers without scrolling the full log.
   - Re-runs `helmfile status`; success triggers Tilt auto-start, failure surfaces remediation guidance and stops the flow.

3. **Tilt bring-up**  
   `scripts/dev/start-tilt.sh` launches Tilt in detached mode. The bootstrap checks `http://127.0.0.1:10350/api/status`; if the API is unreachable, the script exits non-zero so developers see the failure immediately.

## Verification Checklist

Run after post-create or when troubleshooting drift:

- `kubectl get externalsecret -A` — expect `Ready=True`; failures indicate missing secrets or RBAC issues.
- `k3d cluster get ameide` + `docker ps | grep k3d-dev-reg` — ensure the cluster and embedded registry exist.
- `helmfile -e local status` — confirms no pending releases.
- `infra/kubernetes/scripts/infra-health-check.sh` — reruns the layered Helm smoke suites.
- `curl http://127.0.0.1:10350/api/status` — verifies Tilt is serving.

## Operational Commands

| Task | Command(s) |
|------|------------|
| Reconcile k3d manually | `scripts/infra/ensure-k3d-cluster.sh` |
| Refresh vendored Helm charts/CRDs | `infra/kubernetes/scripts/vendor-charts.sh` |
| Re-run Helmfile sync | `infra/kubernetes/scripts/helmfile-sync.sh local` |
| Sync a specific Helm layer | `helmfile -f infra/kubernetes/helmfiles/<layer>.yaml -e local sync` (run `10-bootstrap` first so shared namespaces exist) |
| Smoke tests only | `infra/kubernetes/scripts/infra-health-check.sh` (or trigger Tilt’s `infra:<layer>` resources) |
| Tail bootstrap log | `less +F /home/vscode/.devcontainer-setup.log` |

## Troubleshooting

| Symptom | Likely Cause | Remediation |
|---------|--------------|-------------|
| `foundation-vault-bootstrap` CronJob failing | Vault pod still starting or service account missing auth | Inspect the CronJob pod logs after layer 15 completes; ensure the `foundation-vault-core` StatefulSet is Healthy and the bootstrap service account retains `system:auth-delegator`. |
| `scripts/infra/ensure-k3d-cluster.sh` recreates endlessly | Hash mismatch (config edited) or registry container missing | Inspect diffs in `infra/docker/local/k3d.yaml`; ensure Docker daemon is running. |
| Helmfile fails fetching charts (`repo . not found`, `no such file`) | Vendored archive missing/outdated or local chart path typo | Run `infra/kubernetes/scripts/vendor-charts.sh`, confirm the archive/CRDs are committed, and fix the Helmfile path. |
| `platform-smoke` tests fail | DNS rewrite, registry, or ExternalSecret regression | Inspect job logs (`kubectl logs -n ameide job/<name>`), check CoreDNS config, registry pods, and Vault secret materialisation. |
| Tilt API unreachable | Tilt failed to start or crashed | Tail `/home/vscode/tilt.log`; rerun `scripts/dev/start-tilt.sh`. |

## Status & Next Steps

- ✅ Vault-first bootstrap (deterministic fixtures + ESO wiring)
- ✅ Declarative k3d lifecycle & container registry mirrors
- ✅ Buf Managed Mode bootstrap (credentials + SDK smoke scripts)
- ✅ Helmfile gating with `platform-smoke` Helm tests
- ✅ Vendored Helm charts (CRDs applied by their respective operators)
- ✅ Bootstrap summary/reporting polish (exit summary emitted with per-layer status)

### Outstanding gaps (tracked via backlog/360)

1. **Default secret validation** – `foundation-vault-bootstrap` seeds fixtures blindly; add a post-sync check (or reuse the CI helm-template job) to ensure every ExternalSecret key exists so regressions surface early.  
2. **Optional secret bundles** – DevContainer currently syncs the entire `vault-secrets.yaml`; once backlog/355 splits optional bundles, update the bootstrap flow to seed only the releases required for local mode.  
3. **Gap log integration** – Mirror the backlog/360 service gaps (agents, db-migrations, threads, etc.) by surfacing warnings in `/home/vscode/.devcontainer-setup.log` when helm template/validation steps detect missing credentials or pending refactors.  
4. **Documentation alignment** – Update onboarding docs once backlog/341 finishes the provider-agnostic seeding script so developers know whether Vault, AKV, or another backend supplies the fixtures.

**Upcoming focus:** Gather feedback on the new bootstrap summary and iterate if additional remediation hints or automatic retries are needed, while closing the gaps above in coordination with backlog/341/355/360.
