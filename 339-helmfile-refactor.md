> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# Helmfile Layered Refactor

**Created:** Nov 2025  
**Owner:** Platform DX / Infrastructure

## Context

`infra/kubernetes/helmfile.yaml` ballooned past 1,300 lines as every release lived in a single document. Even with anchors, the monolith hid stage boundaries, made review diffs noisy, and forced contributors to scroll through unrelated layers. CI pipelines also struggled to target subsets cleanly without bespoke selectors.

## Goal

Introduce a layered structure that keeps the execution order deterministic while letting each lifecycle slice evolve independently. The refactor should:

- Keep helm defaults and the canonical `&default` template defined once.
- Split releases into logical groups (bootstrap → core operators → control plane → auth & SSO → package registries → data plane → product services → QA).
  - Preserve existing `stage` labels and environment toggles so selectors and automation keep working; trim `needs` edges on optional releases (e.g., product services) to avoid warning noise when they stay disabled.
- Provide a consistent execution surface: the new `infra/kubernetes/scripts/run-helm-layer.sh` drives the sync+test cycle, while Tilt materialises `infra:<layer>` resources (labelled `infra.layer-tests.*`) so engineers can replay any layer from the UI or CLI.

> **Update – February 2026:** Layers 25–28 (package registries) were removed when Verdaccio/devpi/Athens were decommissioned in favour of Buf Managed Mode. The layering strategy remains, but the registry-specific releases, Tilt resources, and helmfiles referenced below now serve as historical context.

> **Update – November 2026:** Follow-up cleanup dropped the lingering CoreDNS rewrites, Envoy routes, and platform smoke tests that pointed at the retired package registries. Control plane checks now gate on the observability/app routes that remain in the gateway chart, and the platform QA layer focuses on external secret health only.

> **Update – March 2027:** Layer 15 now mirrors the other helmfiles: each environment ships a single `platform/15-secrets.values.yaml` file whose `releases.<bundle>.installed` key drives whether a bundle runs. The Helmfile consumes that map directly, so there are no bespoke toggle manifests or custom health-check logic.

Buf’s Hosted module (`buf.build/ameideio/ameide`) is the single source of truth for generated SDKs. The refactor must keep Buf publication ahead of every Helm run so clusters download SDKs from BSR rather than registry sidecars.

## Approach

1. Extract a reusable defaults snippet (`helmfile/_defaults.yaml.gotmpl`) that declares `helmDefaults` plus the shared `templates.default` anchor.
2. Create per-layer helmfiles for the active infrastructure slices:
   - `helmfiles/10-bootstrap.yaml`
   - `helmfiles/12-crds.yaml`
   - `helmfiles/15-secrets-certificates.yaml`
   - `helmfiles/18-coredns-config.yaml`
   - `helmfiles/20-core-operators.yaml`
   - `helmfiles/22-control-plane.yaml`
   - `helmfiles/24-auth.yaml`
   - `helmfiles/40-data-plane.yaml`
   - `helmfiles/42-db-migrations.yaml`
   - `helmfiles/45-data-plane-ext.yaml`
   - `helmfiles/50-product-services.yaml`
   - `helmfiles/55-observability.yaml`
     - Includes the core telemetry stack (Prometheus, Loki, Tempo, OTEL collector, Grafana) **and Langfuse + bootstrap** so tracing UX is always deployed with observability rather than product services.
   - `helmfiles/60-qa.yaml`
   Keep templating to the bare essentials (e.g., `.Environment`-scoped value overlays or shared defaults); avoid introducing new complex Go templates. Legacy package registry layers (`25-package-registries`, `26-package-npm`, `27-package-go`, `28-package-py`) stay deleted—Buf Managed Mode now publishes SDK artefacts to BSR so Helmfile no longer bootstraps Verdaccio/devpi/Athens components or their publisher images. Layer 15 follows this pattern as well: each environment feeds a single `platform/15-secrets.values.yaml` file whose `releases.<bundle>.installed` keys control the optional bundles.
   Vendor every third-party chart under `../charts/third_party/.../<chart>-<version>.tgz` and update each release to reference that path—no direct repository lookups remain and Helmfile runs fully offline.
3. Introduce a pre-Helm proto publication step: add (or keep) a `ameide_core_proto` local_resource that calls `infra/kubernetes/scripts/buf-build-push.sh`. The helper runs `buf build` + `buf push`, requires a valid `BUF_TOKEN`, and should fail fast when the token is absent so developers configure secrets before Helmfile layers execute. Mirror the step in CI so the Buf module is authoritative for every environment.
4. Trim the root `helmfile.yaml` to environments, helmDefaults, hooks, and an ordered `helmfiles:` block (repositories removed now that every third-party chart is vendored locally). Add a tiny `meta-noop` release so the root state stays non-empty.
5. Ensure each operator chart installs its own CRDs from the local package; remove the shared `third-party-crds` bundle.
6. Validate with `helmfile -f helmfiles/<layer>.yaml sync/status` (layer-by-layer) and `helmfile -e <env> list` (orchestrator) to confirm ordering and offline execution.

### Tilt integration
- `Tiltfile` consumes `INFRA_LAYER_SPECS` and derives `INFRA_CRITICAL_PATH` (ordered list of `infra:<layer>` resources). Every application resource declares `resource_deps=INFRA_CRITICAL_PATH + […]`, so keeping the list in sync guarantees workloads wait for all Helm layers before building images or running smoke jobs.
- A `ameide_core_proto` local_resource now precedes the infra layers. It executes `infra/kubernetes/scripts/buf-build-push.sh`, loads `BUF_TOKEN` from `.env` or the environment, and fails loudly when the token is missing so engineers configure Buf credentials before Helm kicks in.
- DevContainer bootstrap still stops once k3d + Buildx are provisioned and delegates reconciliation entirely to Tilt (documented in `backlog/354-devcontainer-tilt.md`). With package registries retired, Tilt steps through `ameide_core_proto` and then the infra layers—no Verdaccio/devpi/Athens resources remain.

### Developer workflow
- On DevContainer create/update the bootstrap script provisions k3d, installs the Kubernetes Buildx builder, and starts Tilt in detached mode—no Helmfile sync occurs outside Tilt (see `backlog/354-devcontainer-tilt.md`).
- This reinforces the principle that DevContainer work stops at prerequisites; Helm and Tilt layers own the actual deployments once the environment is ready.
- Tilt first runs `ameide_core_proto` (Buf build + push) and then reconciles `infra:<layer>` resources in the order defined by `INFRA_CRITICAL_PATH`, keeping proto publication and Helm smoke tests aligned.
- Document Buf credential management as part of the workflow. Developers need to provision a `BUF_TOKEN` (stored in `.env`, `doppler run`, etc.) before starting Tilt; CI must inject the token via secrets as well. Legacy registry knobs (`SKIP_PACKAGE_IMAGE_BUILD`, Verdaccio URLs) can be removed once scripts stop referencing them.

## Layer scope & tests

- **Layer 10 – Bootstrap**
  - Releases: `namespace` (chart `infra/kubernetes/charts/platform/namespace`).
  - Tests: `infra/kubernetes/scripts/run-helm-layer.sh infra/kubernetes/helmfiles/10-bootstrap.yaml` (or Tilt `infra:10-bootstrap`) waits for the namespace smoke job and surfaces logs inline.
- **Layer 12 – Cluster CRDs**
  - Releases:
    - `prometheus-crds` – installs `prometheus-community/prometheus-operator-crds` v17.0.0 in the `default` namespace so ServiceMonitor-dependent operators (CNPG, redis-operator, kubernetes-dashboard) can reconcile cleanly.
  - Tests: none yet—sync is fast and purely declarative; we may add a `kubectl api-resources` smoke script if more CRDs join.
- **Layer 15 – Secrets & certificates**
  - Releases: `vault`, `vault-secrets` raw bundle, `cert-manager`, `external-secrets`.
  - Presync actions: in local environments the hook runs `scripts/vault/ensure-local-secrets.py` to mirror KV data before External Secrets reconciles.
  - Optional bundles (e.g., Argo CD credentials) now live in dedicated releases so clusters without Argo skip them automatically.
  - Tests: `run-helm-layer.sh` (or Tilt `infra:15-secrets-certs`) runs the `secrets-smoke` Helm job.
  - Resilience: optional manifests such as Argo CD credentials moved into their own Helm release (`vault-secrets-argocd`), so clusters that do not run Argo simply leave that release uninstalled.
- **Layer 20 – Core operators**
  - Releases: `cloudnative-pg`, `strimzi-kafka-operator`, `redis-operator`, `keycloak-operator`, `clickhouse-operator`, `gitlab-operator`, `tekton-operator`, and (optional) `operators-smoke`.
  - Tests: `run-helm-layer.sh` (Tilt `infra:20-operators`) executes the operators smoke chart which waits on deployments, CRDs, and node readiness; it now verifies the Altinity ClickHouse operator deployment and CRDs before later layers reconcile ClickHouse installations.
- **Layer 22 – Control plane (planned)**
  - Releases: Envoy Gateway, the `gateway` chart, CoreDNS configuration, and the `control-plane-smoke` Helm test job.
  - Tests: `run-helm-layer.sh` / `infra:22-control-plane` handles the smoke job and checks the Gateway Programmed condition.
- **Layer 24 – Auth & SSO (planned)**
  - Releases: `postgres-clusters` (shared CNPG instance), `keycloak`, `keycloak-realm`, shared identity bootstrap tooling (e.g., admin user/job, base OAuth2 proxy configuration) plus an `auth-smoke` Helm test job that exercises login flows; the layer depends on core operators (including `keycloak-operator`) being reconciled.
  - Tests: `run-helm-layer.sh` / `infra:24-auth` to drive the auth smoke job (realm reconciliation, admin API checks).
- **Layer 25 – Package registries (retired)**
  - Removed in February 2026. Shared DNS/config maps for Verdaccio/devpi/Athens are no longer reconciled; Buf’s Managed Mode provides language artifacts directly from BSR.
- **Layer 26 – npm (retired)**
  - Verdaccio and the TypeScript package runner were removed. TypeScript services now install the published Buf SDK packages (`@buf/ameideio_ameide.*`) without in-cluster registries.
- **Layer 27 – Go packages (retired)**
  - Athens, the Go module uploader, and the runner Helm test were deleted. Go services import Buf generated modules via `buf.build/gen/go/...` and CI smoke tests resolve them from BSR.
- **Layer 28 – Python packages (retired)**
  - Devpi and its runner no longer exist; Python services fetch Buf-published wheels (`ameideio-ameide-*`) instead of mirroring into a cluster registry.
- **Layer 40 – Data plane core**
  - Releases: `redis-failover`, `kafka-cluster`, `clickhouse`, `pgadmin` (and future always-on storage backends). The ClickHouse installation assumes the operator from Layer 20 is already healthy; `needs` now pins it to `clickhouse-operator/clickhouse-operator` so Langfuse/Plausible can rely on a shared data plane without bundling their own ClickHouse bits.
  - Tests: `run-helm-layer.sh` / `infra:40-data-plane` runs the `data-plane-smoke` Helm job (Redis/Kafka/CNPG/ClickHouse readiness + diagnostics).
  - Prerequisites: layer 20 (core operators) must be applied first so the redis, Strimzi, and ClickHouse operators own their CRDs. Vault must expose `graph-db-*`, `threads-db-*`, and `workflows-db-*` credentials; the local bootstrap (`scripts/vault/ensure-local-secrets.py`) now seeds these keys so Helmfile can pull them via ExternalSecrets.
- **Layer 42 – Database migrations**
  - Releases: `db-migrations` (local-only sync) with dependencies on the shared Postgres cluster and Vault-provisioned secrets. When running locally, ensure the Flyway image `docker.io/ameide/ameide-migrations:dev` is present in k3d (`docker build … && k3d image import …`) so the pre-upgrade hook can start without hitting `ImagePullBackOff`.
  - Tests: `run-helm-layer.sh` / `infra:42-db-migrations`; operators can set `installed=true` temporarily per environment to replay flyway jobs during cutovers. The chart now falls back to `DB_USERNAME`/`DB_PASSWORD` from bundled secrets when explicit Flyway credentials are omitted. The layer helper calls `scripts/infra/bootstrap-db-migrations-secret.sh` before syncing so `ameide/db-migrations-local` is always materialised with `FLYWAY_URL` / `FLYWAY_USER` / `FLYWAY_PASSWORD`, keeping the pre-upgrade job healthy after restarts without DevContainer involvement.
- **Layer 45 – Data plane ext**
  - Releases: `minio`, `neo4j`, `temporal` (and other optional/tenant-specific stateful workloads).
  - Tests: `run-helm-layer.sh` / `infra:45-data-plane-ext`; individual checks handle optional workloads gracefully and log skip reasons.
- **Layer 50 – Product services (scaffolded)**
  - Releases: workflow runtime, inference services, web frontends, and supporting jobs; Helmfile definitions exist behind the `productServices.installViaHelmfile` flag (default `false`) so Tilt/Argo remain the primary orchestrators until we toggle the layer on for bootstrap runs.
  - Tests: Existing Helm test hooks per chart plus HTTP smoke curls defined in chart templates.
  - Dependency notes: `needs` edges were removed from this layer so running `helmfile diff` while the flag is `false` no longer spams warnings about disabled parents; reintroduce them only once we switch the layer on by default.
- **Layer 55 – Observability (planned)**
  - Releases: `prometheus`, `prometheus-oauth2-proxy`, `alertmanager-oauth2-proxy`, `loki`, `tempo`, `otel-collector`, `grafana`, `observability-smoke` helpers (plus any future dashboards we re-home from GitOps).
  - Tests: `run-helm-layer.sh` / `infra:55-observability` executes `observability-smoke` (Prometheus, Tempo, Loki, Grafana, telemetry routes).
- **Layer 60 – QA**
  - Releases: `platform-smoke` and future integration suites.
  - Tests: `run-helm-layer.sh` / `infra:60-qa` which fetches the smoke job logs; the helper downgrades “pod not found” races to warnings once the test job succeeds.

## Follow-up

- Document the new layout in `infra/kubernetes/README.md` and update CI/Tilt references.
- Document the CRD lane expectations (Layer 12) so contributors know where to land future standalone CRD bundles.
- Consider templating repeated QA job definitions (integration tests) with Go template loops for further deduplication.
- Keep `stage` and `layer` labels aligned with the new layer names (e.g., `stage=data-plane-ext`) so CI selectors and Argo filters continue to work.
- Explore adding layer-specific selectors (`stage=control-plane`, `stage=data-plane`, `stage=data-plane-ext`, etc.) to speed up smoke deployments in CI.

## Test approach principles

- **Helm-native smoke jobs:** Each layer carries a dedicated Helm test release (`helm-test-jobs`) so `infra/kubernetes/scripts/run-helm-layer.sh` (and Tilt) can drive a consistent sync → test loop and capture logs.
- **Cache-friendly isolation:** Smoke jobs rely on utility images we preload (kubectl/curl) and poke only the services owned by their layer, keeping runs offline and reproducible.
- **Real service probes:** Checks use `kubectl wait`, HTTP/dns curls, and CR lookups against the actual cluster so we catch integration issues without resorting to mocks.
- **Self-cleaning hooks:** Test releases set `hookDeletePolicy: before-hook-creation` (or equivalent) so stale Jobs don’t block reruns; Helmfile/scripts should prune leftover test Jobs before relaunching to keep loops fast.
- **Environment overlays first:** Treat the shared chart values as the baseline; every environment-specific tweak lives under `infra/kubernetes/environments/<env>/...`, keeping drift visible and ensuring smoke jobs receive the exact overlays they exercise.
