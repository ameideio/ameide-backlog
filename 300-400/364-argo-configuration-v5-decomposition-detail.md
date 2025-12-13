# backlog/364 – Argo configuration v5 (Helmfile decomposition detail)

This appendix expands the migration plan: for **each helmfile layer** under `infra/kubernetes/helmfiles`, it lists every release, the target **GitOps component path**, and the **ApplicationSet tier/wave** we’ll assign. Use this table when you move charts from Helmfile → GitOps.

> **Helmfile retention policy:** even after a layer is migrated, keep the Helmfile record checked into git so engineers can continue to run `helm template` (or automation like `render.sh`) when they need freshly rendered YAML. Add a `# RENDER-ONLY` comment to the top of migrated Helmfiles to remind folks they must not be applied to clusters.
>
> **Repo layout reminder:** environment value files now live under `sources/values/<env>/<domain>` inside `gitops/ameide-gitops`, charts live under `sources/charts/{foundation,platform-layers,apps,...}`, and component directories contain metadata only (`component.yaml`).

Legend:

* **Tier** → RollingSync label (`infra`, `data`, `apps`, etc.).
* **Wave** → string/int used in `catalog/*.yaml` and RollingSync steps (negative waves run first).
* **Component path** → directory under `gitops/ameide-gitops/environments/<env>/components/...`.
* **Notes** → sync options, health checks, or special handling.

> When you add tiers beyond `infra/data/apps` (e.g., `platform`, `qa`), remember to mirror them in the RollingSync steps of the relevant ApplicationSet or collapse them into one of the core tiers. The table keeps the richer vocabulary so you can decide whether to expand the RollingSync matchers or simply map extra tiers back to `infra/data/apps`.

---

## Layer 00 – Argo CD

| Helm release | Action | Target component | Wave | Notes |
| --- | --- | --- | --- | --- |
| `argocd` (chart) | ✅ Already replaced by bootstrap manifest (`install.yaml`). | N/A | N/A | Delete helmfile once unused. |

---

## Layer 10 – Bootstrap (namespaces, smoke tests)

| Release | Target component path | Tier | Wave | Notes |
| --- | --- | --- | --- | --- |
| `namespace` (namespace chart) | `foundation/namespaces/ameide` | `infra` | `-20` | Replace chart with raw Namespace YAML; ApplicationSet sets `CreateNamespace=true`. |
| `bootstrap-smoke` (helm-test jobs) | `foundation/bootstrap-smoke` | `infra` | `-15` | Implement as Job with `argocd.argoproj.io/hook: Sync`; ensures cluster ready before next waves. |

**Status:** ✅ Migrated to `foundation-dev` ApplicationSet on 2025-11-12 (`wave00` namespaces, `wave05` smoke job).

---

## Layer 12 – CRDs

| Release | Target component path | Tier | Wave | Notes |
| --- | --- | --- | --- | --- |
| `gateway-api-crds`, etc. | `foundation/crds/gateway-api`, `foundation/crds/<name>` | `infra` | `-10` | Use raw manifests or Helm chart; set `SkipDryRunOnMissingResource=true` for CR consumers. |
| Additional CRDs (cert-manager, ext-dns, etc.) | similar | `infra` | `-10` | Keep each CRD family in its own component for easier reuse. |
| `redisfailovers.databases.spotahome.com` | `foundation/crds/redis` | `infra` | `-10` | Mirrors the Helm doc guidance: CRD rendered once via raw manifest, while the operator chart sets `helm.skipCrds=true` so upgrades flow through this component. |

**Status (2025-11-12):** ✅ `foundation/crds/gateway-api` (standard install manifest) and `foundation/crds/prometheus-operator` (rendered from `prometheus-community/prometheus-operator-crds` v17.0.0) now live under `gitops/ameide-gitops/environments/dev/components/foundation/crds/` with metadata `wave10`. The `foundation-dev` ApplicationSet gained a `wave10` RollingSync step so these CRD Applications run immediately after namespaces + smoke. **Nov‑14 refresh:** the Spotahome RedisFailover CRD (`databases.spotahome.com_redisfailovers`) is now managed by `foundation/crds/redis`, and the operator Application sets `helm.skipCrds=true` so day‑2 CRD updates always flow through the raw manifest component. **Next:** add certificate/secrets components (Helmfile 15) so we can delete `12-crds.yaml`.

---

## Layer 15 – Secrets & Certificates

| Release | Target component path | Tier | Wave | Notes |
| --- | --- | --- | --- | --- |
| `cluster-issuers`, `cert-manager bootstrap` | `foundation/certificates/issuers` | `infra` | `-5` | Requires cert-manager CRDs/operator from earlier wave. |
| Secrets bootstrap (e.g., KMS/ESO templates) | `foundation/secrets/bootstrap` | `infra` | `-5` | Use `RespectIgnoreDifferences=true` if controllers mutate Secret data. |

**Status (2025-11-12):** cert-manager and External Secrets operators now live under `foundation/operators/**` (wave15) as rendered manifests, so ApplicationSet RollingSync deploys them right after the CRDs. The remainder of the layer (Vault PVC/server, cluster secret stores, all `vault-secrets-*` bundles, integration-secrets job, and the secrets smoke test) now lives under `foundation/secrets/**` with waves 16–21. Helmfile `15-secrets-certificates.yaml` has been removed. Every ExternalSecret reports `Ready=True`; Argo still flags the resources OutOfSync because the ExternalSecrets controller injects default fields (we added `argocd.argoproj.io/compare-options: IgnoreExtraneous` annotations to keep the diff noise local).

---

## Layer 18 – CoreDNS config

| Release | Target component path | Tier | Wave | Notes |
| --- | --- | --- | --- | --- |
| `coredns-config` (ConfigMap patch) | `foundation/coredns/config` | `infra` | `-5` | Release runs in `foundation-coredns`; chart templates both the `foundation-coredns` namespace and the `kube-system/coredns-custom` ConfigMap. |

**Status (2025-11-14):** ✅ ApplicationSet now emits the component into the dedicated `foundation-coredns` namespace (created by the same raw-manifests chart). The ConfigMap still targets `kube-system/coredns-custom`, but Argo no longer tracks the entire kube-system namespace, so orphan detection is clean. `platform` AppProject explicitly allows writes to both namespaces, keeping permissions tight while supporting the rewrite patch. Helmfile `18-coredns-config.yaml` remains render-only.

---

## Layer 20 – Core operators

| Release | Target component path | Tier | Wave | Notes |
| --- | --- | --- | --- | --- |
| `cert-manager` | `foundation/operators/cert-manager` | `infra` | `-5` | Helm chart; apply `SkipDryRunOnMissingResource=true` for CRDs. |
| `external-secrets`, `external-dns`, etc. | `foundation/operators/<name>` | `infra` | `-5` | Each operator gets its own Application; add health checks if needed. |
| Ingress/gateway controllers | `platform/control-plane/gateway` | `infra` | `0` | Controllers go after CRDs/operators. |

**Status (2025-11-12):** ✅ All Helmfile 20 releases were rendered into GitOps components:

* `foundation/operators/cloudnative-pg` (project=data-services, namespace `cnpg-system`)
* `foundation/operators/strimzi`
* `foundation/operators/redis`
* `foundation/operators/clickhouse`
* `foundation/operators/keycloak`
* `foundation/operators-smoke` (helm-test job with Argo hooks)

Each component sets `CreateNamespace=true`, `SkipDryRunOnMissingResource=true`, and `RespectIgnoreDifferences=true`, and runs in `wave15` alongside cert-manager + ESO. Helmfile `20-core-operators.yaml` is deleted. RollingSync now gates this entire operator wave before Vault/secrets proceed.

---

## Layer 21 – Storage primitives (local dev)

| Release | Target component path | Tier | Wave | Notes |
| --- | --- | --- | --- | --- |
| `managed-csi` StorageClass | `foundation/storage/managed-storage` | `infra` | `10` | Raw-manifests chart now templates both the `platform-storage` namespace and the `managed-csi` StorageClass; deployed via the dedicated `platform-storage` AppProject. |

**Status (2025-11-14):** ✅ `foundation/storage/managed-storage` moved off `kube-system` and now deploys from the `platform-storage` namespace. The chart creates the namespace + StorageClass in one release, and a scoped AppProject (`platform-storage`) only whitelists the cluster resources it needs (`Namespace`, `StorageClass`). This eliminated the blanket `orphanedResources.warn=false` workaround while keeping RollingSync noise-free.

---

## Layer 22 – Control plane

| Release | Target component path | Tier | Wave | Notes |
| --- | --- | --- | --- | --- |
| Ingress/Gateway classes, network policies, cluster issuers | `platform/control-plane/*` | `infra` | `0` | These depend on operators; include `CreateNamespace=true` where needed. |
| Any cluster-scoped configs (e.g., default RBAC) | `platform/control-plane/rbac` | `infra` | `0` | Keep in platform AppProject (allows cluster resources). |

---

## Layer 24 – Auth

| Release | Target component path | Tier | Wave | Notes |
| --- | --- | --- | --- | --- |
| Keycloak chart | `platform/auth/keycloak` | `platform` | `1` | Helm chart with env overlays; add DB dependency (tier `data`). |
| Realm/clients config (raw manifests) | `platform/auth/config` | `platform` | `2` | Use `RespectIgnoreDifferences=true` if controller mutates config. |

---

## Layer 40 – Core data plane

| Release | Target component path | Tier | Wave | Notes |
| --- | --- | --- | --- | --- |
| Platform services (API, graph, inference, etc.) | `apps/<service>` | `apps` | `3`/`4` | Already started (e.g., `apps/ameide` for dev). Add one component per service. |
| Envoy configs (if local) | `platform/envoy/config` | `platform` | `1` | As needed. |

---

## Layer 42 – DB migrations

| Release | Target component path | Tier | Wave | Notes |
| --- | --- | --- | --- | --- |
| `db-migrations` jobs | `platform/db-migrations` | `data` | `2` | Implement as Application with PreSync hooks; ensure it runs after DB instances but before apps. |

---

## Layer 45 – Data plane extensions

| Release | Target component path | Tier | Wave | Notes |
| --- | --- | --- | --- | --- |
| MinIO | `data/extended/minio` | `data` | `wave25` | Shared values keep it off; the local overlay flips `enabled: true`, forces `local-path` storage, and caps replicas at one so only k3d/dev gets buckets. |
| Neo4j | `data/extended/neo4j` | `data` | `wave25` | The component lives as `component.disabled.yaml`; rename it to `component.yaml` when we actually need Neo4j. |
| Temporal core | `data/extended/temporal` | `data` | `wave26` | Operator-based: deploys `TemporalCluster` + `TemporalNamespace` CRs (no separate namespace bootstrap job). Requires CNPG/Postgres; keep `SkipDryRunOnMissingResource=true`. |
| Temporal OAuth proxy | `data/extended/temporal-oauth2-proxy` | `data` | `wave28` | Bitnami oauth2-proxy chart with Azure AD config. |
| Data plane extended smoke | `data/extended/data-plane-ext-smoke` | `data` | `wave29` | Helm test job; stays disabled until foundational components are Healthy. |

---

## Layer 50 – Product services

| Release | Target component path | Tier | Wave | Notes |
| --- | --- | --- | --- | --- |
| Platform, graph, inference, threads, etc. | `apps/<service>` | `apps` | `3` (backends), `4` (frontends) | For each service: add Kustomize/Helm overlay to `apps-config` repo, reference via catalog. |
| Frontend (Next.js) apps | `apps/<frontend>` | `apps` | `4` | Use `syncOptions: [CreateNamespace=true]`. |

---

## Layer 55 – Observability

| Release | Target component path | Tier | Wave | Notes |
| --- | --- | --- | --- | --- |
| Prometheus stack | `platform/observability/prometheus` | `platform` | `0` | Helm chart (prometheus-community). |
| Grafana | `platform/observability/grafana` | `platform` | `0` | Include dashboards as ConfigMaps/CRDs. |
| OpenTelemetry, Langfuse, logging | `platform/observability/<name>` | `platform` | `0` | Each service becomes its own component. |

---

## Layer 60 – QA / Smoke tests

| Release | Target component path | Tier | Wave | Notes |
| --- | --- | --- | --- | --- |
| `platform-smoke` | `platform/qa/platform-smoke` | `platform` | `5` (after apps) | Implement as Job with PostSync hook so it runs after backends/frontends are Healthy. |

---

## 9. Implementation checklist

1. **Create component directories** per table above (one per release). Include `component.yaml` with metadata:
   ```yaml
   name: platform-smoke
   tier: platform
   project: platform
   wave: "5"
   namespace: ameide
   repoURL: https://github.com/ameideio/ameide-gitops
   targetRevision: HEAD
   path: environments/dev/components/platform/qa/platform-smoke
   syncOptions:
     - CreateNamespace=true
     - RespectIgnoreDifferences=true
   ```
2. **Author ApplicationSets** per environment (dev/staging/prod). Ensure RollingSync steps match wave labels (infra → data → apps → qa).
3. **Add Lua health checks** for third-party CRDs (Postgres operator/instances, etc.) in `argocd-cm`.
4. **Test** each component by syncing via ApplicationSet, then remove it from Helmfile once stable.
5. **Delete Helmfile** once all releases are migrated (keep the table up-to-date).

This level of detail ensures every service previously managed via Helmfile has a 1:1 mapping in GitOps, following the ApplicationSet-only, health-gated pattern from the vendor reference.

---

## 10. Runtime verification snapshot (2025‑11‑13)

* **Charts are now vendored, not downloaded.** Every third-party `.tgz` under `gitops/ameide-gitops/sources/charts/third_party/**` was unpacked and committed, so Argo’s repo-server reads `Chart.yaml` directly; Helmfile and CI/CD are the only places allowed to fetch upstream artifacts.
* **ApplicationSets mount shared/local values via `$values`.** Each template declares a second Git source (`ref: values`) so Helm sees `_shared/<tier>/<name>.yaml` + `<env>/<tier>/<name>.yaml`. Any new ApplicationSet must follow this pattern to keep rendered manifests deterministic.
* **Redis operator currently absent.** `kubectl -n redis-operator get pods` (run from the devcontainer) returns “No resources found”, meaning the operator hasn’t rolled out yet. Until that Application completes, the downstream `data-redis-failover` component will remain `OutOfSync/Degraded`. Plan to re-sync the foundation tier after fixing the image reference (quay.io/spotahome/redis-operator:v1.2.4) or mirroring it to a reachable registry.
* **Next steps before “all green”:**
  - Re-run the foundation → data → platform → apps waves (either `argocd app sync -l tier=<tier>` or let RollingSync drive) after the operator images & secrets are in place.
  - Restore pgAdmin bootstrap/OAuth secrets (or keep OAuth disabled) so the pod leaves `CreateContainerConfigError`.
  - Publish accessible versions of every `docker.io/ameide/*:dev` image (agents, platform, workflows runtime, namespace guard) or add `imagePullSecrets` in component values.
  - Delete the failed Keycloak realm import Jobs and re-sync `platform-keycloak-realm` to exercise the updated realm JSON.
  - MinIO is now scoped to local dev only (shared values keep it off elsewhere and the local overlay forces `local-path` + single replica); the Neo4j component is stored as `component.disabled.yaml`, so the ApplicationSet no longer emits an empty/failed Application for it.
  - Prune Helmfile-era resources once Argo owns the namespaces to clear the fleet-wide `OrphanedResourceWarning`.
