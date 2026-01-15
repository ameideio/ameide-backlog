Current Argo CD wave layout (dev), aligned to what is defined in `gitops/ameide-gitops` today.

> **UI performance:** See `backlog/681-argocd-ui-performance.md` for tuning notes with ~400 Applications.

- Foundation: 10 -> 20 -> 30 -> 40 -> 50 -> 60 -> 99
- Platform: 29 -> 31 -> 30 -> 32 -> 39
- Data core: 30 -> 31 -> 39
- Data extended: 34 -> 36 -> 39
- Apps backends: 80 -> 89
- Apps frontends: 90 -> 99 (99 currently unused)

---

## Principles (wave reorg)

- Keep base-10 spacing inside each ApplicationSet (10/20/30…) with 99 reserved for smokes/tests.
- Order by dependency depth: namespaces first, then CRDs, then controllers/operators, then secrets/configs, then bootstraps, then smokes.
- Move shared edge/TLS prereqs early so ingress/gateway consumers don’t stall downstream.
- Keep platform/data/apps in separate ApplicationSets but align ranges so humans can spot intent; avoid hidden cross-set coupling beyond top-level sync order and explicit DependsOn where needed.
- Minimize special cases—only add extra waves when a real dependency requires sequencing.

Top-level Argo Application sync order (`environments/dev/argocd/applications/root/*.yaml`): foundation 0 -> platform 1/2 -> data-core 4 -> data-extended 5 -> apps 9.

---

## Foundation (current 10/20/30/40/50/60/99)

ApplicationSet: `environments/dev/argocd/apps/foundation.yaml` (sync-wave 0).

- Current RollingSync: 10 -> 20 -> 30 -> 40 -> 50 -> 60 -> 99.
  - 10: namespaces — `foundation-namespaces` (`foundation/common/raw-manifests`).
  - 20: CRDs — redis, prometheus-operator, gateway-api (+ ExternalSecrets CRDs if not preinstalled) (`foundation/common/raw-manifests`).
  - 30: secret store + storage — `foundation-vault-secret-store`, `managed-storage` (`foundation/common/raw-manifests`).
  - 40: operators — cnpg-monitoring (`common/raw-manifests`), cloudnative-pg (`third_party/cnpg/cloudnative-pg/0.26.1`), clickhouse (`third_party/altinity/altinity-clickhouse-operator/0.25.5`), external-secrets-ready (`foundation/helm-test-jobs`), vault-webhook-certs (`foundation/vault-webhook-certs`), redis (`third_party/spotahome/redis-operator/3.3.0`), cert-manager (`third_party/jetstack/cert-manager/v1.19.1`), vault-bootstrap (`foundation/vault-bootstrap`), external-secrets (`third_party/external-secrets/external-secrets/0.20.4`), keycloak (`foundation/operators/keycloak-operator`), vault (`third_party/hashicorp/vault/0.28.0`), strimzi (`third_party/strimzi/strimzi-kafka-operator/0.48.0`).
  - 50: secrets — vault-secrets-{observability,temporal,platform} (`foundation/common/raw-manifests`), clickhouse-secrets (`data/clickhouse-secrets`), keycloak-db-credentials (`foundation/common/raw-manifests`).
  - 60: configs — coredns-config (`foundation/common/raw-manifests`).
  - 99: smokes — `foundation-operators-smoke`, `foundation-bootstrap-smoke` (`foundation/helm-test-jobs`).

_Proposal (not applied):_ keep as-is.
---

## Platform (waves 29/31/30/32/39)

ApplicationSet: `environments/dev/argocd/apps/platform.yaml` (sync-wave 2, RollingSync order 29 -> 31 -> 30 -> 32 -> 39).

1. 29: `platform-cert-manager-config` (issuers/certs for envoy/platform TLS).
2. 31: `platform-envoy-gateway` (Envoy Gateway control plane).
3. 30: core platform payloads (maxUpdate 10):
   - Control plane: envoy-crds.
   - Auth: keycloak, postgres-clusters.
   - Observability: prometheus, otel-collector, loki, tempo, grafana, grafana-datasources, langfuse, langfuse-bootstrap, alloy-logs.
4. 32: `platform-gateway` and `platform-keycloak-realm` (maxUpdate 1).
5. 39: smokes (`control-plane-smoke`, `secrets-smoke`, `observability-smoke`, `auth-smoke`).

Gateway/GatewayClass health overrides live in `gitops/ameide-gitops/sources/values/common/argocd.yaml` so these waves do not stall on Accepted/Programmed-only status.

_Proposal (not applied):_ normalize ranges: 10 edge/TLS prereqs (cert-manager-config, envoy-gateway), 20 platform runtimes (envoy-crds, auth, observability), 30 platform bootstraps (gateway, keycloak-realm), 99 smokes.

---

## Data services (waves 30/31/39 and 34/36/39)

Two ApplicationSets with separate RollingSync ladders.

- `data-core-dev` (`environments/dev/argocd/apps/data-core.yaml`, sync-wave 4)
  - 30: clickhouse, kafka-cluster, redis-failover, minio.
  - 31: db-migrations.
  - 39: data-plane-smoke.

- `data-extended-dev` (`environments/dev/argocd/apps/data-extended.yaml`, sync-wave 5)
  - 34: temporal, pgadmin.
  - 36: (removed) Temporal namespaces are now declared via `TemporalNamespace` CRs alongside `data-temporal`.
  - 39: data-plane-ext-smoke.

_Proposal (not applied):_ align to 20/30/99 per set — data-core: 20 runtimes (clickhouse/kafka/redis/minio), 30 db-migrations, 99 smokes; data-extended: 20 temporal/pgadmin (includes TemporalNamespace CRs), 99 smokes.

---

## Apps (waves 80/89/90/99)

- `apps-backends-dev` (`environments/dev/argocd/apps/apps-backends.yaml`, sync-wave 9)
  - 80: platform, transformation, graph, workflows, workflows-runtime, threads, agents, agents-runtime, inference, inference-gateway.
  - 89: apps-platform-smoke.

- `apps-frontends-dev` (`environments/dev/argocd/apps/apps-frontends.yaml`, sync-wave 9)
  - 90: www-ameide, www-ameide-platform.
  - 99: reserved (no components today).
