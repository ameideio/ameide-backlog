# backlog/382 – ArgoCD dependency hardening & secrets gating

**Created:** Nov 2025  
**Owner:** Platform DX / Infrastructure  
**Related:** backlog/362-unified-secret-guardrails-v2.md, backlog/367-bootstrap-v2.md

---

## Problem
Wave-only ordering let components reconcile out of sequence (e.g., observability stacks racing ahead of backends, apps starting without Vault-backed secrets). Chart-level band-aids (inline PreSync jobs) masked the gaps and violated the guardrail principle of Argo-managed, fail-fast enforcement.

> **Decision (2026-04):** Do **not** adopt Argo CD `dependsOn`. We standardize on wave ordering + health hooks/smoke jobs to express readiness and gating.

### Terminology
- When this document references “dependency edges” or `dependsOn`, it describes **our own metadata** on component definitions/ApplicationSets. Argo CD itself does not enforce an object-level `dependsOn` primitive. We translate those edges into wave ordering, health checks, and smoke jobs so reconciliation remains deterministic and reproducible.
- Argo primitives we rely on are: ApplicationSet templating, `sync.wave`, `syncOptions`, custom health checks, and Helm/kubectl-based smokes.
- Application and ApplicationSet specs in `gitops/ameide-gitops` must **not** set `spec.dependsOn`; sequencing is handled entirely through waves and health signals.

## Goal
Make dependencies explicit and Argo-native across all tiers so apps only roll when their prerequisites are healthy, and secrets are verified cluster-side without inline hooks or local fallbacks.

## Approach
1. **Wave ordering + health hooks (no `dependsOn`)** – Use waves + component type/phase to sequence controllers → stores → bundles → workloads. Use Argo health hooks to gate on readiness instead of custom `dependsOn`.
2. **Argo smoke > chart hooks** – Replace chart-level PreSync waits with Argo-managed smoke tests (helm-test-jobs) that fail fast if ExternalSecrets or key Secrets are missing:
   - New `platform-secrets-smoke` (wave 43) runs after platform waves; checks ExternalSecret readiness and critical Secrets (grafana-admin-credentials, loki/tempo S3, minio-service-users, platform DB creds, etc.).
3. **Bootstrap alignment** – The bootstrap v2 flow can wait on tier health (e.g., platform wave 41/42 + secrets smoke) so local/CI/prod behave the same and cannot “finish” with missing prerequisites.

## Definition of done
- All environments (dev/staging/prod) keep consistent wave ordering and platform secrets smoke coverage; Argo waits respect health hooks/smokes instead of `dependsOn`.
- No chart-level secret wait hooks remain; failures surface via Argo health/smoke and wave sequencing.
- README/runbooks reference the dependency model and how to re-run smokes.

## Next steps
1) Extend the dependency sweep to remaining gaps: data operators/CRDs → workloads, gateway/proxy → certs/secrets, app-tier per-service DB/queue deps.  
2) Wire bootstrap v2 to fail when tier health or secrets smokes are not green (backlog/367).  
3) Add lightweight CI/tilt checks to ensure new components follow wave/health/smoke gating for secrets/backends before merge.***




Findings

Secret chain gaps (all envs): foundation-external-secrets-ready has no edge to foundation-external-secrets (gitops/ameide-gitops/environments/dev/components/foundation/operators/external-secrets-ready/component.yaml (line 1) etc); foundation-vault-secret-store is not blocked on the readiness check (.../foundation/secrets/vault-secret-store/component.yaml (line 1)); foundation-vault-secrets-{platform,observability} are not gated on the store (.../foundation/secrets/vault-secrets-platform/component.yaml (line 1), etc); foundation-vault-bootstrap does not depend on foundation-vault-core (.../foundation/operators/vault-bootstrap/component.yaml (line 1)). This breaks the “Fail Fast / No Local Fallbacks” chain for all consumers.
Controller/CRD ordering (all envs): Stateful services lack operator edges—data-kafka-cluster vs foundation-strimzi-operator (.../data/core/kafka-cluster/component.yaml (line 1)), data-redis-failover vs foundation-redis-operator (.../data/core/redis-failover/component.yaml (line 1)), data-clickhouse-secrets vs foundation-clickhouse-operator (.../data/core/clickhouse-secrets/component.yaml (line 1)), platform-postgres-clusters vs foundation-cloudnative-pg (.../platform/auth/postgres-clusters/component.yaml (line 1)). CNPG/vault webhooks also aren’t sequenced with foundation-vault-webhook-certs.
Data/services (all envs): Secrets aren’t gated on Vault (data-clickhouse-secrets, data-minio); Temporal now uses CNPG-managed Secrets and should not reference Vault at all. Temporal namespaces are now declared via `TemporalNamespace` CRs alongside `data-temporal` (no separate `data-temporal-namespace-bootstrap` app). data-plane-smoke and data-plane-ext-smoke don’t reference the stacks they validate. data-db-migrations has no edge to platform-postgres-clusters or foundation-vault-secrets-platform (.../data/db/db-migrations/component.yaml (line 1)).
Platform/auth/ingress (staging/prod parity gap): Keycloak and realm lack DB/secret edges (platform-keycloak, platform-keycloak-realm at .../platform/auth/keycloak/component.yaml (line 1) and .../keycloak-realm/component.yaml (line 1)); dev is equally missing today. platform-postgres-clusters not gated on CNPG operator or Vault store. platform-cert-manager-config, platform-gateway, platform-envoy-gateway have no cert-manager/gateway-API ordering (.../platform/control-plane/*.yaml). platform-secrets-smoke has no upstream deps (.../platform/secrets/secrets-smoke/component.yaml (line 1)).
Observability (all envs): platform-grafana uses admin creds but doesn’t point to foundation-vault-secrets-observability; platform-grafana-datasources is wave 41 while grafana is wave 42 and lacks gating to grafana/backends (.../platform/observability/grafana-datasources/component.yaml (line 1)). platform-otel-collector/platform-alloy-logs don’t depend on backends. platform-langfuse & platform-langfuse-bootstrap rely on clickhouse/redis/minio/vault secrets but have no edges (.../platform/observability/langfuse*.yaml (line 1)). platform-observability-smoke has no dependencies.
Apps tier (all envs): None of the apps reference platform-postgres-clusters or foundation-vault-secrets-platform even though all charts require those ExternalSecrets (e.g. apps-platform .../apps/core/platform/component.yaml (line 1), apps-graph .../apps/core/graph/component.yaml (line 1), etc). Service-specific gaps: apps-workflows/apps-workflows-runtime not gated on data-temporal; apps-inference/apps-inference-gateway not gated on data-redis-failover or Vault store; apps-agents/apps-agents-runtime not gated on data-minio; web apps (apps-www-ameide*) not gated on Keycloak realm/gateway/redis. apps-platform-smoke lacks deps.
Smoke/tests (all envs): foundation-bootstrap-smoke, foundation-operators-smoke, platform-control-plane-smoke, platform-auth-smoke, platform-observability-smoke, platform-secrets-smoke, apps-platform-smoke, data-data-plane-smoke, data-data-plane-ext-smoke all have empty dependency metadata, so they don’t actually validate readiness before progressing.
Proposed gating/wave adjustments (minimal)

Foundation chain
Gate foundation-external-secrets-ready on foundation-external-secrets.
Gate foundation-vault-secret-store on foundation-external-secrets-ready; gate foundation-vault-secrets-{platform,temporal,observability} on the secret store.
Gate foundation-vault-bootstrap on foundation-vault-core and foundation-vault-webhook-certs.
Gate foundation-operators-smoke on all layer-15 operators; gate foundation-bootstrap-smoke on the secret store and bundles.
Data layer
Gate operators: data-kafka-cluster -> foundation-strimzi-operator; data-redis-failover -> foundation-redis-operator; data-clickhouse-secrets -> foundation-clickhouse-operator; platform-postgres-clusters -> foundation-cloudnative-pg.
Secret gating: data-clickhouse-secrets, data-minio -> foundation-vault-secret-store; Temporal consumes CNPG-managed Secrets (no Vault gating).
Ordering: data-db-migrations -> platform-postgres-clusters + foundation-vault-secrets-platform; data-plane-smoke -> kafka/redis/clickhouse; data-plane-ext-smoke -> minio + temporal (+ kafka/redis if exercised).
 Data-db-migrations job owns every schema (platform/agents/graph/.../temporal) via a single helper that normalizes JDBC URLs, runs Flyway with explicit location lists, and auto-baselines non-empty schemas before migrating—no inline hooks or per-chart scripts.
Platform/auth + control-plane
platform-postgres-clusters -> foundation-cloudnative-pg + foundation-vault-secret-store.
platform-keycloak -> platform-postgres-clusters + foundation-vault-secrets-platform + foundation-vault-secret-store; platform-keycloak-realm -> platform-keycloak.
platform-cert-manager-config -> foundation-cert-manager; platform-envoy-gateway -> foundation-cert-manager + foundation-crds-gateway; platform-gateway -> platform-envoy-gateway + platform-cert-manager-config. Gate platform-secrets-smoke on vault-secrets-platform and keycloak.
Observability
platform-prometheus/loki/tempo -> foundation-vault-secrets-observability (for admin creds) and foundation-cert-manager if certificates/monitors are required.
platform-grafana -> prom+loki+tempo + foundation-vault-secrets-observability; bump grafana-datasources after grafana (wave > grafana or gate explicitly on grafana/prom/loki/tempo).
platform-otel-collector -> platform-prometheus + platform-tempo; platform-alloy-logs -> platform-loki.
platform-langfuse -> data-clickhouse + data-redis-failover + data-minio + foundation-vault-secrets-observability; platform-langfuse-bootstrap -> platform-langfuse + foundation-vault-secret-store.
platform-observability-smoke -> grafana + datasources + prom + loki + tempo + otel/alloy.
Apps tier
Baseline for every app: gate on platform-postgres-clusters, foundation-vault-secrets-platform, foundation-vault-secret-store.
Additional edges: apps-workflows/apps-workflows-runtime -> data-temporal; apps-inference/app-runtime-agent/langgraph -> data-redis-failover (and data-minio where S3 used); apps-agents/apps-agents-runtime -> data-minio; apps-www-ameide* -> platform-keycloak-realm + platform-gateway + data-redis-failover; apps-inference-gateway -> data-redis-failover + platform-gateway.
apps-platform-smoke -> keycloak + gateway + core apps it validates.
Smoke/tests enforcement
Add explicit gating metadata for each smoke job to the components they validate (foundation operators, secret bundles, data stores, platform auth/control-plane, observability stack, apps as applicable) and ensure smoke waves remain last.
Follow-up risks

Exact secret producers must be confirmed when wiring edges (e.g., which bundle mints www-ameide-auth); above assumes vault bundles remain the single source.
Grafana/Langfuse may also need cert-manager/gateway edges if ingress/certs are enabled—validate per env before setting.
After adding edges, re-evaluate wave numbers to keep monotonic order (especially grafana vs datasources) and avoid circular deps.

## Reproducible stack blueprint
To keep every environment reproducible, treat the stack as four responsibility tiers with clear deliverables:

| Tier | Responsibility | Owners | GitOps path examples | Notes |
| --- | --- | --- | --- | --- |
| **Foundation** | Cluster prerequisites (CRDs, operators, ExternalSecrets store, Vault bundles, cert-manager) | Platform DX | `foundation/operators/*`, `foundation/secrets/*` | Waves 0‑15. Provides identity, PKI, and secret materialization. Health requirements: operator CRDs ready, Vault PKI certs rotated, secret smoke green. |
| **Data core** | Stateful backends (Postgres/CNPG, Kafka, Redis, ClickHouse) and shared migrations | Data Services | `data/core/*`, `data/db/*` | Waves 16‑32. Consume secrets from foundation tier, expose readiness via Lua health checks. Helm hook jobs limited to schema creation (temporal/sql tool). |
| **Data extensions** | Optional runtimes (Temporal, MinIO, Neo4j) and bootstrap jobs | Data Services | `data/extended/*` | Waves 25‑35. Depend on data-core + migrations; ship namespace/bootstrap jobs as separate components (wave+1) to avoid race conditions. |
| **Platform & Apps** | Control planes (gateway/envoy), platform services, product apps, smokes | Platform + App teams | `platform/*`, `apps/*` | Waves 35+. Each component declares required backends (DBs, auth, secrets) via metadata, while smokes (QA tier) sit last to verify entire slice. |

Publishing new components requires: (1) adding a component directory with metadata (tier/domain/wave + dependency list), (2) referencing it from the env ApplicationSet so wave ordering is enforced, and (3) authoring health checks or smoke jobs appropriate for the tier. This structure keeps local/dev/prod identical—bootstrap v2 simply runs `argocd app sync --groups waveX` in order so any failure halts rollouts deterministically.
