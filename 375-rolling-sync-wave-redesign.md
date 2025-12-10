# backlog/375 – RollingSync wave redesign

> **Bootstrap location:** This backlog still references the legacy `tools/bootstrap/bootstrap-v2.sh` helper for rerunning RollingSync in dev. That CLI now lives in `ameide-gitops/bootstrap/bootstrap.sh`; when operating from the application repo, use the GitOps bootstrap for cluster convergence and `ameide-core/tools/dev/bootstrap-contexts.sh` only for developer context setup.

## 1. Why revisit waves now?
RollingSync was introduced while Helmfile layers still defined our world. We mirrored those layers (foundation/data/platform/apps) even though dependencies like “Keycloak needs Postgres” made the tier boundaries fuzzy—Postgres lives in the platform tier purely because Keycloak depends on it. As the stack grew, that layer-centric layout left us with (historical snapshot _before_ this redesign):
* **24+ serialized steps** across four ApplicationSets (`foundation-dev`, `data-services-dev`, `platform-dev`, `apps-dev`). Each step has `maxUpdate=1`, even when the Applications are independent. A full redeploy can take ~30–40 minutes because Argo CD waits for every Application to go Healthy before touching the next wave.
* **Overloaded foundation waves.** `wave15` contains Vault, CNPG, Strimzi, cert-manager, ESO, and a smoke job. `wave16` holds every Vault secret bundle. Any flaky Application in those groups stalls the entire infra tier.
* **Misplaced smoke / QA checks.** Validation jobs run in the same wave as the workloads they validate. A failing hook keeps the wave OutOfSync, preventing retry loops on the actual operator or chart.
* **Implicit cross-tier dependencies.** Platform Keycloak depends on data-plane Postgres clusters; app workloads depend on both. Because ApplicationSets run independently, a green foundation tier does not prevent platform/apps from syncing while data remains unhealthy.
* **Unused/opaque labels.** Waves are encoded as strings (`"wave15"`) and we currently define steps that have zero members (`wave17-21`). Adding new components requires hunting for unused strings and massaging match expressions.

This backlog attempts a full redesign so waves reflect real dependencies, allow safe parallelism, and keep runbooks simple.

## 2. Design goals & constraints

1. **Health-gated tiers stay intact.** We keep RollingSync as the orchestrator; this exercise changes labels, grouping, and `maxUpdate` but not the overall progressive sync model.
2. **Concrete readiness gates.** Each wave should map to a precondition the next wave truly needs (CRDs → controllers → CR consumers, DB → migrations → workloads, secrets → consumers → smoke).
3. **Parallelism where safe.** Independent workloads in a wave should sync concurrently. For example, Loki and Tempo don’t depend on each other; they should not be serialized.
4. **Smoke isolation.** Validation jobs get dedicated waves immediately *after* the resources they test. If they fail, operators can debug without blocking follow-up syncs.
5. **Cross-tier ordering.** Tier-level Applications should obey `foundation → data → platform → apps` so we don’t deploy frontends when foundational services are broken.
6. **Predictable labeling.** Shift to numeric (or at least sortable) labels and remove unused waves. Future components should have a documented slot without guesswork.
7. **Backward compatibility awareness.** Existing Lua health checks and ignore rules must continue to work. We shouldn’t break existing sync windows or AppProject restrictions.

### Implementation snapshot (Nov 2025)
- Dev environment now follows the target numeric rollout-phase layout across every tier. ApplicationSets (`foundation-dev`, `data-services-dev`, `platform-dev`, `apps-dev`) enforce the tables in §3 with the documented `maxUpdate` values and dedicated smoke waves.
- `wave_lint.sh` generates (and lint-checks) the inventory consumed in §3.5 and CI can reuse it to fail fast when a component lacks a `rolloutPhase` label.
- `gitops/ameide-gitops/README.md` hosts the full “RollingSync verification & inventory” workflow (inventory command, bootstrap rerun, manual `argocd app sync`/`app wait`, parent orchestrator inspection, controller log tailing). Backlog references point there to avoid duplicated runbooks.
- The parent orchestrator (`environments/dev/argocd/applications/ameide.yaml`) plus its root kustomization ensure tier-level sync-waves (`0/1/2/3`). Operators can still sync tiers manually, but RollingSync now guarantees foundation → data → platform → apps sequencing by default.
- Smoke / QA Applications across all domains live in dedicated waves immediately after the workloads they test, satisfying the “smoke isolation” requirement.
- Remaining TODO: when new environments (staging/prod) are added, copy the component/ApplicationSet structure, re-run the inventory tooling, and follow the verification steps to validate RollingSync end-to-end.

## 3. Current vs proposed wave maps

### 3.1 Foundation ApplicationSet

**Previous layout (pre-refactor):**  
* `wave00` namespaces  
* `wave05` bootstrap smoke  
* `wave10` CRDs/storage + managed storage  
* `wave12` CoreDNS config  
* `wave15` **(overloaded)** – Vault core, Vault bootstrap, cert-manager, ESO, CNPG, Strimzi, Redis operator, Keycloak operator, operators smoke, etc.  
* `wave16` **(overloaded)** – every Vault secret bundle + ClusterSecretStore  
* `wave17-21` defined in RollingSync spec but unused.

**Pain points:** When Vault or CNPG is unhealthy, all other operators in `wave15` churn even if they’re fine. Operators smoke jobs can’t run until every operator is green, but when they fail they keep the wave degraded. Secret bundles all compete in the same wave which hides dependencies (secret store vs bundles vs consumers). `maxUpdate=2` means we only sync two Applications per wave even though half a dozen charts are independent.

**Current status (Nov 2025):** Foundation, data, platform, and apps components now publish numeric `rolloutPhase` labels and each dev ApplicationSet’s RollingSync strategy enforces the tables below (with the documented `maxUpdate` values). Supporting tooling (`wave_lint.sh` and the “RollingSync verification & inventory” section in `gitops/ameide-gitops/README.md`) keeps the layout auditable and documents the verification workflow so staging/prod can reuse it.

**Target layout (labels become numeric only—drop the `wave` prefix everywhere):**

| Wave label | Max update | Rationale / members |
| --- | --- | --- |
| `00` | 2 | All namespaces (foundation + shared). No change. |
| `05` | 1 | Bootstrap smoke (kicks off hooks that verify namespaces). |
| `10` | 3 | CRDs + managed storage (Gateway API, Redis CRDs, Prometheus CRDs, PVC setup). |
| `12` | 2 | CoreDNS config and other cluster-scoped configmaps; safe once CRDs exist. |
| `15` | 3 | “Pure” operators that only depend on CRDs (cert-manager, ESO, Strimzi, ClickHouse operator, Redis operator, CNPG operator). `maxUpdate=3` lets them roll together. |
| `16` | 2 | Vault StatefulSet + bootstrap CronJob (depends on storage + operators). |
| `17` | 2 | ClusterSecretStore + shared secret bundles (platform, observability, temporal). Split secret store into `wave17` so bundlers don’t bounce. |
| `18` | 2 | Secret validation jobs + smoke suites (operators smoke, secrets smoke). Failures now isolate to their wave. |
| `19-21` | Reserved | For future shared services (ingress controllers, cluster-wide RBAC). |

Implementation details:
* Rename component labels from `"waveNN"` to `"NN"` so RollingSync matchers can rely on clean numeric values (`In ["15","16"]`, `Gt 18`, etc.).
* Update `foundation-vault-secret-store` to `wave17` while keeping its dependencies documented (`foundation-secrets/vault-secret-store/component.yaml`).
* Add `maxUpdate:3` for `wave15` (operators) and `wave17` (secret bundles) to shorten rollout time.
* Remove unused steps from the RollingSync spec once relabeling is complete.

### 3.2 Data ApplicationSet

**Previous layout (pre-refactor):** sequential waves `20-30` with `maxUpdate=1`. CNPG clusters, Redis, Kafka all run one at a time, followed by ClickHouse obs, pgAdmin, data-plane smoke, db-migrations, optional storage (MinIO), Temporal, and extended smoke.

**Target (numeric labels):**  

| Wave | Max update | Contents / notes |
| --- | --- | --- |
| `20` | 2 | Stateful primitives (CNPG clusters, Redis failover, Kafka). Running 2 at a time keeps API load manageable while reducing total time. |
| `21` | 2 | Secret dependencies or shared bundles needed before the core data apps (e.g., `foundation-vault-secrets-observability`). |
| `22` | 1 | ClickHouse / pgAdmin (depends on CNPG + secrets). |
| `23` | 1 | Data-plane smoke job (verifies base plane). |
| `24` | 1 | DB migrations (runs after the plane is validated). |
| `25` | 2 | Optional storage providers (MinIO) or rarely enabled features. |
| `26` | 1 | Temporal core |
| `27` | 1 | Temporal namespace bootstrap |
| `28` | 1 | Temporal OAuth proxy or additional controllers |
| `29` | 1 | Extended smoke (data-plane-ext) |

Notes:
* This keeps Temporal pieces isolated but still sequential. If gating becomes too slow we can combine `26/27` with a `maxUpdate=1` cluster.
* Document that `data-plane-smoke` must succeed before `db-migrations` runs; hooking them to the same wave would revert the old behavior we want to avoid.
**Status (Nov 2025):** Complete in dev — every data component now includes a numeric `wave` label and the `data-services-dev` ApplicationSet gates waves `20-29` with the `maxUpdate` values above.

### 3.3 Platform ApplicationSet

**Previous layout (pre-refactor):** `maxUpdate=1` for every wave from `wave30` through `wave46`. Observability workloads (Loki, Tempo, Prometheus, Alloy, Grafana datasources) roll one at a time even though they’re independent. Auth stack includes Postgres clusters -> Keycloak -> realm -> smoke, all sequential. Observability QA at the end.

**Target (numeric labels):**  

| Wave | Max update | Description |
| --- | --- | --- |
| `30` | 1 | Envoy Gateway chart |
| `31` | 1 | Gateway API resources (Gateway, HTTPRoute, etc.) |
| `32` | 1 | Cert-manager config for control plane |
| `33` | 1 | Control-plane smoke job |
| `34` | 1 | Postgres clusters for Keycloak (or move this component under data tier wave `20`) |
| `35` | 1 | Keycloak |
| `36` | 1 | Keycloak realm |
| `37` | 1 | Auth smoke |
| `40` | 3 | Loki / Tempo / Prometheus (parallel). Set `maxUpdate=3`. |
| `41` | 3 | Alloy logs, Grafana datasources, OTEL collectors (parallel). |
| `42` | 1 | Grafana |
| `43` | 1 | Langfuse bootstrap |
| `44` | 1 | Langfuse runtime |
| `45` | 1 | Observability smoke / QA |
| `46` | reserve | Additional platform QA or future controllers |

Observability waves (`40/41`) benefit most from parallelism. We also encourage a conversation about moving `platform/auth/postgres-clusters` to the data tier so databases live alongside other data-plane primitives. If not feasible, document the dependency explicitly (platform Keycloak waits for data wave `20` to become Healthy).
**Status (Nov 2025):** Complete in dev — platform components now use waves `30-45` with RollingSync enforcing the new batch sizes (3-way parallelism for observability workloads).

### 3.4 Apps ApplicationSet

**Previous layout (pre-refactor):** `wave50-55` with `maxUpdate=1`. Platform services (platform, graph, transformation) share `wave50` but still serialize. Runtime services (workflows, threads, agents) and frontends all serialize as well.

**Target (numeric labels):**

| Wave | Max update | Contents |
| --- | --- | --- |
| `50` | 2 | Core backend services (platform, graph, transformation). Helm release dependencies ensure proper ordering inside each Application. |
| `51` | 2 | Workflows, threads. |
| `52` | 2 | Runtime services (agents runtime, runtime APIs). |
| `53` | 2 | Inference / runtime extras. |
| `54` | 3 | Frontend apps (www-ameide, www-ameide-platform). |
| `55` | 1 | App smoke / QA (future). |

`maxUpdate` increases provide faster rollouts without sacrificing cross-Application health gating. If certain services truly depend on others, label them into later waves accordingly.
**Status (Nov 2025):** Complete in dev — all apps components publish waves `50-55` and the ApplicationSet now syncs each cohort with the desired parallelism.

### 3.5 Current wave inventory (dev state)
Generated via `./wave_lint.sh` (it scans `gitops/ameide-gitops/environments/dev/components/**/component.yaml`). Format: `component-name` (relative path) – tier / project / namespace.

- **wave00**
  - `foundation-namespaces` (foundation/namespaces/component.yaml) – foundation / platform / argocd
- **wave05**
  - `foundation-bootstrap-smoke` (foundation/bootstrap-smoke/component.yaml) – foundation / platform / ameide
- **wave10**
  - `foundation-crds-gateway` (foundation/crds/gateway-api/component.yaml) – foundation / platform / argocd
  - `foundation-crds-prometheus` (foundation/crds/prometheus-operator/component.yaml) – foundation / platform / argocd
  - `foundation-crds-redis` (foundation/crds/redis/component.yaml) – foundation / platform / argocd
  - `foundation-managed-storage` (foundation/storage/managed-storage/component.yaml) – foundation / platform-storage / platform-storage
- **wave12**
  - `foundation-coredns-config` (foundation/coredns-config/component.yaml) – foundation / platform / foundation-coredns
- **wave15**
  - `foundation-cert-manager` (foundation/operators/cert-manager/component.yaml) – foundation / platform / cert-manager
  - `foundation-clickhouse-operator` (foundation/operators/clickhouse/component.yaml) – foundation / data-services / clickhouse-operator
  - `foundation-cloudnative-pg` (foundation/operators/cloudnative-pg/component.yaml) – foundation / data-services / cnpg-system
  - `foundation-cnpg-monitoring` (foundation/operators/cnpg-monitoring/component.yaml) – foundation / data-services / ameide
  - `foundation-external-secrets` (foundation/operators/external-secrets/component.yaml) – foundation / platform / external-secrets
  - `foundation-keycloak-operator` (foundation/operators/keycloak/component.yaml) – foundation / platform / ameide
  - `foundation-redis-operator` (foundation/operators/redis/component.yaml) – foundation / data-services / redis-operator
  - `foundation-strimzi-operator` (foundation/operators/strimzi/component.yaml) – foundation / data-services / strimzi-system
- **wave16**
  - `foundation-vault-bootstrap` (foundation/operators/vault-bootstrap/component.yaml) – foundation / platform / vault
  - `foundation-vault-core` (foundation/operators/vault/component.yaml) – foundation / platform / vault
- **wave17**
  - `foundation-vault-secret-store` (foundation/secrets/vault-secret-store/component.yaml) – foundation / platform / argocd
  - `foundation-vault-secrets-observability` (foundation/secrets/vault-secrets-observability/component.yaml) – foundation / platform / ameide
  - `foundation-vault-secrets-platform` (foundation/secrets/vault-secrets-platform/component.yaml) – foundation / platform / ameide
- **wave18**
  - `foundation-operators-smoke` (foundation/operators-smoke/component.yaml) – foundation / platform / ameide
- **wave20**
  - `data-kafka-cluster` (data/core/kafka-cluster/component.yaml) – data / data-services / ameide
  - `data-redis-failover` (data/core/redis-failover/component.yaml) – data / data-services / ameide
- **wave22**
  - `data-clickhouse` (data/core/clickhouse/component.yaml) – data / data-services / ameide
  - `data-pgadmin` (data/core/pgadmin/component.yaml) – data / data-services / ameide
- **wave23**
  - `data-data-plane-smoke` (data/core/data-plane-smoke/component.yaml) – data / data-services / ameide
- **wave24**
  - `data-db-migrations` (data/db/db-migrations/component.yaml) – data / data-services / ameide
- **wave25**
  - `data-minio` (data/extended/minio/component.yaml) – data / data-services / ameide
- **wave26**
  - `data-temporal` (data/extended/temporal/component.yaml) – data / data-services / ameide
- **wave27**
  - `data-temporal-namespace-bootstrap` (data/extended/temporal-namespace-bootstrap/component.yaml) – data / data-services / ameide
- **wave29**
  - `data-data-plane-ext-smoke` (data/extended/data-plane-ext-smoke/component.yaml) – data / data-services / ameide
- **wave30**
  - `platform-envoy-gateway` (platform/control-plane/envoy-gateway/component.yaml) – platform / platform / ameide
- **wave31**
  - `platform-gateway` (platform/control-plane/gateway/component.yaml) – platform / platform / ameide
- **wave32**
  - `platform-cert-manager-config` (platform/control-plane/cert-manager-config/component.yaml) – platform / platform / ameide
- **wave33**
  - `platform-control-plane-smoke` (platform/control-plane/control-plane-smoke/component.yaml) – platform / platform / ameide
- **wave34**
  - `platform-postgres-clusters` (platform/auth/postgres-clusters/component.yaml) – platform / platform / ameide
- **wave35**
  - `platform-keycloak` (platform/auth/keycloak/component.yaml) – platform / platform / ameide
- **wave36**
  - `platform-keycloak-realm` (platform/auth/keycloak-realm/component.yaml) – platform / platform / ameide
- **wave37**
  - `platform-auth-smoke` (platform/auth/auth-smoke/component.yaml) – platform / platform / ameide
- **wave40**
  - `platform-loki` (platform/observability/loki/component.yaml) – platform / platform / ameide
  - `platform-prometheus` (platform/observability/prometheus/component.yaml) – platform / platform / ameide
  - `platform-tempo` (platform/observability/tempo/component.yaml) – platform / platform / ameide
- **wave41**
  - `platform-alloy-logs` (platform/observability/alloy-logs/component.yaml) – platform / platform / ameide
  - `platform-grafana-datasources` (platform/observability/grafana-datasources/component.yaml) – platform / platform / ameide
  - `platform-otel-collector` (platform/observability/otel-collector/component.yaml) – platform / platform / ameide
- **wave42**
  - `platform-grafana` (platform/observability/grafana/component.yaml) – platform / platform / ameide
- **wave43**
  - `platform-langfuse-bootstrap` (platform/observability/langfuse-bootstrap/component.yaml) – platform / platform / ameide
- **wave44**
  - `platform-langfuse` (platform/observability/langfuse/component.yaml) – platform / platform / ameide
- **wave45**
  - `platform-observability-smoke` (platform/observability/observability-smoke/component.yaml) – platform / platform / ameide
- **wave50**
  - `apps-graph` (apps/core/graph/component.yaml) – apps / apps / ameide
  - `apps-platform` (apps/core/platform/component.yaml) – apps / apps / ameide
  - `apps-transformation` (apps/core/transformation/component.yaml) – apps / apps / ameide
- **wave51**
  - `apps-threads` (apps/core/threads/component.yaml) – apps / apps / ameide
  - `apps-workflows` (apps/core/workflows/component.yaml) – apps / apps / ameide
  - `apps-workflows-runtime` (apps/core/workflows-runtime/component.yaml) – apps / apps / ameide
- **wave52**
  - `apps-agents` (apps/runtime/agents/component.yaml) – apps / apps / ameide
  - `apps-agents-runtime` (apps/runtime/agents-runtime/component.yaml) – apps / apps / ameide
- **wave53**
  - `apps-inference` (apps/runtime/inference/component.yaml) – apps / apps / ameide
  - `apps-inference-gateway` (apps/runtime/inference-gateway/component.yaml) – apps / apps / ameide
- **wave54**
  - `apps-www-ameide` (apps/web/www-ameide/component.yaml) – apps / apps / ameide
  - `apps-www-ameide-platform` (apps/web/www-ameide-platform/component.yaml) – apps / apps / ameide
- **wave55**
  - `apps-platform-smoke` (apps/qa/platform-smoke/component.yaml) – apps / apps / ameide


Use `./wave_lint.sh` (Markdown) or `./wave_lint.sh --format json` whenever you need to refresh this list.
## 4. Cross-tier guardrails

Even with perfect intra-tier waves, we still deploy tiers in parallel. The dev environment now uses **option #1 (parent orchestrator Application)** so that foundation → data → platform → apps sequencing is enforced declaratively. For historical completeness, other options are retained below, but option #1 is the canonical implementation:
2. **Sync windows or manual gating.** Alternatively, use AppProject Sync Windows to allow data/platform/apps only when foundation is Healthy. Downside: operators must manage windows manually.
3. **Scripting / automation.** As a fallback, Jenkins/Tilt pipeline could call `argocd app sync foundation-dev --prune --retry`, wait for Healthy, then sequentially sync data → platform → apps. This is less declarative but requires zero new manifests.

### 4.1 Parent orchestrator Application details

*Manifest location & shape.* Add `gitops/ameide-gitops/environments/dev/argocd/applications/ameide.yaml` that renders a single Argo CD `Application` sourcing a small kustomization containing the four tier `Application` manifests. Each child `Application` keeps its existing spec, but receives a `metadata.annotations.argocd.argoproj.io/sync-wave` value (`"0"` foundation, `"1"` data, `"2"` platform, `"3"` apps). The orchestrator itself carries `sync-wave: "-1"` so it always reconciles before the tier Applications.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ameide
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  project: platform
  destination:
    name: in-cluster
    namespace: argocd
  source:
    repoURL: https://github.com/ameideio/ameide-gitops.git
    path: environments/dev/argocd/applications/root
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

The `root` kustomization (`environments/dev/argocd/applications/root/`) lists the four tier `Application` YAML files; keeping them separate lets operators keep RBAC/project scoping identical to today. Each tier Application carries the `sync-wave` annotations shown above.

*Responsibilities.* The orchestrator only enforces ordering. Operators can still sync `platform-dev` directly if needed, but RollingSync will now wait for the preceding tier to report `Healthy` before allowing the next one to advance. Runbooks instruct operators to use `argocd app get ameide` as the single pane of glass for tier orchestration.

## 5. Migration plan

1. **Inventory mapping.** _Complete_ – enforced by `wave_lint.sh`.
2. **Relabel components.** _Complete_ – all dev components carry numeric `wave` labels.
3. **Update ApplicationSets.** _Complete_ – RollingSync steps match the tables in §3.
4. **Add the parent orchestrator.** _Complete_ – `applications/ameide.yaml` wires tier sync-waves together.
5. **Smoke job refactor.** _Complete_ – every QA Application sits in a dedicated wave.
6. **Test progressive sync.** Covered via the verification checklist below / `gitops/ameide-gitops/README.md` (“RollingSync verification & inventory”).
7. **Document runbooks.** _Complete_ – backlog/364 + the new docs reference this design.
8. **Backport to other environments.** Ready for reuse – when additional envs appear, copy the components + rerun the tooling to validate.

### 5.1 Dev verification checklist
1. **Inventory + lint:** `./wave_lint.sh` fails fast if a component is missing its `rolloutPhase` label. `.devcontainer/postCreate.sh` now runs the JSON variant automatically (unless `DEVCONTAINER_SKIP_WAVE_VALIDATION=1`) so missing labels stop bootstrap immediately.
2. **Re-run bootstrap (preferred):** `../ameide-gitops/bootstrap/bootstrap.sh --config bootstrap/configs/dev.yaml --reset-k3d --show-admin-password --port-forward`. This reapplies repo credentials, recreates the parent `ameide` Application (if missing), and ensures the RollingSync ApplicationSet picks up the updated spec in a clean state.
3. **Trigger RollingSync manually (fallback):**
   ```bash
   argocd app sync foundation-dev --server localhost:8080 --plaintext --insecure --retry-strategy backoff
   argocd app wait foundation-dev --timeout 600 --health
   argocd app sync data-services-dev --server localhost:8080 --plaintext --insecure --retry-strategy backoff
   # repeat for platform-dev and apps-dev once the previous tier is Healthy
   ```
4. **Validate the parent orchestrator:** `argocd app get ameide` should show the tier Applications as children with the expected `Sync Wave` column values. Confirm that pausing `foundation-dev` keeps the higher tiers OutOfSync but not progressing—this proves the guardrail works.
5. **Inspect controller logs:** `kubectl logs -n argocd deploy/argocd-applicationset-controller -f | rg rolling-sync` to confirm each wave transitions as expected and `maxUpdate` caps are respected.
6. **Document findings:** capture health + timing deltas in the GitOps README section so we can compare against staging/prod once those environments are migrated.

## 6. Risks & mitigations
| Risk | Mitigation |
| --- | --- |
| Parallel ops overwhelm API server | Start with conservative `maxUpdate` (2–3) and monitor Argo CD metrics. Reduce if throttling occurs. |
| Component mis-labeled, causing dependency issues | Require owner sign-off during mapping; run canary syncs per tier before merging. |
| Smoke jobs still block due to long runtimes | Convert long-running checks into `PostSync` hooks with timeouts; keep Applications minimal. |
| Parent Application introduces new single point of failure | Keep tiers independently manageable; parent orchestrator simply enforces order, but operators can still sync tiers manually if needed. |
| Numeric-only labels become opaque to operators | Publish and maintain the numeric wave map in backlog/364 + component READMEs; add comments in each `component.yaml` describing the wave’s readiness gate. |

## 7. Open questions
* Should platform auth Postgres clusters move to data tier or stay under platform?
* Do we need separate waves for CRDs vs CRDs+operators when emitting from raw manifests?
* How do we handle optional/disabled components? (Proposal: keep them labeled but rely on environment-specific `enabled` flags.)
* Numeric-only labels remove the “wave” mnemonic—should we add linting or doc tooling to keep the number ↔ readiness gate mapping obvious?

### 7.1 Proposed resolutions
- **Platform auth Postgres placement.** Treat Postgres clusters as data-tier assets even if the consumers live in platform. Moving `platform-auth` CNPG clusters into the data ApplicationSet eliminates the last cross-tier dependency: the database becomes part of the earlier tier, and the platform components simply depend on the published connection secrets.
- **CRDs vs operator waves.** Keep the two-phase layout already drafted (`10` for CRDs/storage primitives, `15` for operators/controllers). This ensures k3s/kube-apiserver doesn’t churn when we upgrade controllers and lets us fail fast if CRDs don’t apply cleanly. Only merge them when Helm charts own both CRD + controller installation and we’re confident about upgrade ordering.
- **Optional components.** Keep every component labeled with its canonical wave even when `enabled: false` so operators know where it would slot. Environment overlays flip the `enabled` flag (or omit the Application) without touching the labeling scheme, which keeps linting + documentation in sync.
- **Numeric label clarity.** Automate enforcement in CI by wiring `scripts/wave-inventory.py` (or a lightweight Go/Bash port) into `pnpm lint`. The job will fail when waves are missing, duplicated, or out of order. Pair that with a short comment inside each `component.yaml` explaining the readiness gate (“`# Wave 15: operators that only depend on CRDs`”) so humans don’t need to remember the mapping.

## 8. Tracking
* **Owner:** Argo CD platform team  
* **Dependencies:** backlog/364 (Helmfile decomposition), backlog/362 (secret guardrails)  
* **Deliverable:** PR series relabeling components, updating ApplicationSets, and adding the tier orchestrator.
