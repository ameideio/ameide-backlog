# backlog/364 – Argo configuration v5 (Helmfile → ApplicationSet migration)

Goal: retire every `infra/kubernetes/helmfiles/*.yaml` release from the **runtime apply path** and manage **all infra/platform/app workloads** via Argo CD **ApplicationSets** that implement the **wave + health gating** model described in the vendor-aligned reference.

> We are keeping the Helmfiles in-repo as **rendering blueprints** (developers still run `helm template` against them when bumping chart versions), but once a layer is migrated those Helmfiles must not be applied to a cluster. Treat them as documentation + input to chart rendering only.

This document captures the target Git layout, ApplicationSet strategy, and a migration checklist so we can fence Helmfile into that rendering-only role once the v5 GitOps setup is complete.

> **Bootstrap reference:** Execution steps that previously called `tools/bootstrap/bootstrap-v2.sh` now map to `ameideio/ameide-gitops: bootstrap/bootstrap.sh`. DevContainers in `ameideio/ameide` use `tools/dev/bootstrap-contexts.sh` solely to configure kubectl/Telepresence/argocd against the shared AKS cluster per [435-remote-first-development.md](435-remote-first-development.md).

---

## 1. Target Git structure (config repo)

```
gitops/
  ameide-gitops/
    environments/
      dev/
        argocd/
          projects/
            platform.yaml
            data-services.yaml
            apps.yaml
          apps/
            platform.yaml            # ApplicationSet (RollingSync) for infra/platform
            data.yaml                # ApplicationSet for DB/queues
            apps.yaml                # ApplicationSet for first-party apps
          repos/
            ameide-gitops.yaml       # repository secret (type=repository)
          ops/
            argocd-cm.yaml           # health checks, ApplicationSet feature flags, etc.
        components/                  # metadata only (no rendered manifests)
          foundation/
          platform/
          data/
          apps/
```

* Every component directory holds Helm/Kustomize manifests plus a small metadata file (e.g., `component.yaml`) specifying `project`, `tier`, `namespace`, `syncOptions`, etc.
* ApplicationSets generate per-component Applications from those directories and roll them out in tiered waves (infra → data → apps) using RollingSync.

> **Values + charts layout:** environment overrides and charts now live inside the GitOps repo (`sources/values/<env>/{foundation,platform,apps,tests}` and `sources/charts/{foundation,platform-layers,apps,...}`). Component metadata references those paths via the `chart` block, so Argo needs only the GitOps repo to render everything.

> **New required value (Dec 2025):** the `www-ameide-platform` chart now emits `NEXT_PUBLIC_DEFAULT_ORG`, `AUTH_DEFAULT_ORG`, and `WWW_AMEIDE_PLATFORM_ORG_ID` from its ConfigMap. GitOps environments **must** set `services.www_ameide_platform.organization.defaultOrg` (see `services/www_ameide_platform/values.example.yaml`) or Helm rendering will fail via `required(...)`. This keeps pod + Telepresence env files in sync and avoids runtime crashes during login.

---

## 2. ApplicationSet orchestration

Example (platform ApplicationSet with RollingSync):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: platform
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/ameideio/ameide-gitops.git
        revision: HEAD
        files:
          - path: environments/dev/components/**/component.yaml
  template:
    metadata:
      name: '{{name}}'
      labels:
        tier: '{{tier}}'
    spec:
      project: '{{project}}'
      source:
        repoURL: https://github.com/ameideio/ameide-gitops.git
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        syncOptions: '{{syncOptions}}'
  strategy:
    type: RollingSync
    rollingSync:
      steps:
        - matchExpressions: [{ key: tier, operator: In, values: [infra] }]
        - matchExpressions: [{ key: tier, operator: In, values: [data] }]
        - matchExpressions: [{ key: tier, operator: In, values: [apps] }]
```

* Each `component.yaml` sets `name`, `tier`, `project`, `namespace`, optional `syncOptions`.
* RollingSync waits for tier=infra to be **Healthy** before tier=data, etc., mirroring the wave table.
* Enable Progressive Syncs in `argocd-cm` (`applicationsetcontroller.enable.progressive.syncs: "true"`).
* The controller takes over sync orchestration, so we intentionally omit the `automated` block—Progressive Syncs temporarily disable it on generated Applications anyway.

> **Multi-AppSet vs single AppSet:** the migration plan still favors one ApplicationSet per project/tier (`platform`, `data-services`, `apps`). Each spec matches the snippet above; only the `matchExpressions` and generator path differ. Use the shared snippet as the baseline template, then duplicate it per tier if you need stricter guardrails or differing RollingSync steps.

---

## 3. AppProjects & guardrails

Create three AppProjects (`platform`, `data-services`, `apps`) with:

* `sourceRepos`: restrict to `https://github.com/ameideio/ameide-gitops.git`.
* `destinations`: per tier (platform components can deploy cluster-scoped resources; apps are namespace-scoped only).
* `clusterResourceWhitelist` / `namespaceResourceWhitelist` to permit only the necessary Kinds (e.g., `Namespace` allowed in platform project).
* Roles/permissions if required (e.g., OIDC groups per team).

These live under `environments/<env>/argocd/projects/*.yaml` and are applied during bootstrap.

---

## 4. Wave mapping (tier labels → sync order)

| Tier (`tier` label) | Sync wave | Contents                                 | Notes |
| --- | --- | --- | --- |
| `infra` | -10 / -5 / 0 | CRDs, namespaces, operators, platform primitives | Use `SkipDryRunOnMissingResource=true` when CRDs + CRs in same graph. |
| `data` | 1 / 2 | Data operators and instances (Postgres, queues) | Add Lua health checks so RollingSync waits for DB “Ready=True”. |
| `apps` | 3 / 4 | First-party services (backends, frontends) | `CreateNamespace=true` to avoid manual namespace creation. |

RollingSync steps reference `tier` labels; add more tiers if needed (e.g., `ops`, `qa`).

> **Fine-grained waves:** if you want the `-20 → 5` ordering from the detail appendix, keep the `tier`-based RollingSync steps for coarse sequencing and add a `wave` label sourced from `component.yaml`. You can then declare additional RollingSync steps (or sub-matchers inside each tier) to respect the finer ordering without giving up readability.

---

## 5. Helmfile decomposition plan

For each helmfile (track status as we migrate):

| Helmfile | Action | Status (2025-11-12) |
| --- | --- | --- |
| `00-argocd` | Already replaced by bootstrap manifest. Keep Helmfile for `helm template`, but mark it “render-only” and never apply. | ✅ handled historically by `tools/bootstrap/bootstrap-v2.sh` (now `ameide-gitops/bootstrap/bootstrap.sh`), which runs automatically from the GitOps bootstrap |
| `10-bootstrap` | Convert namespace chart + smoke jobs into components (`foundation/namespaces`, `foundation/bootstrap-smoke`). Keep Helmfile as rendering input. | ✅ migrated → `foundation-dev` ApplicationSet (`wave00` / `wave05`) |
| `12-crds` | Components per CRD package (`foundation/crds/*`). Helmfile retained for rendering. | ✅ migrated → `foundation/crds/gateway-api` + `foundation/crds/prometheus-operator` (wave10) |
| `15-secrets-certificates` | `foundation/certificates`, `foundation/secrets`. Helmfile retained for rendering only. | ✅ cert-manager + External Secrets operators + Vault/secret-store bundles + smoke job migrated (waves 15–21) |
| `18-coredns-config` | `foundation/coredns-config`. Helmfile retained for rendering only. | ✅ migrated → component now deploys from the dedicated `foundation-coredns` namespace while writing the `kube-system/coredns-custom` ConfigMap (wave12). |
| `20-core-operators` | `foundation/operators/<name>`. Helmfile retained for rendering only. | ✅ cnpg, Strimzi, Redis, ClickHouse, Keycloak operators + operators smoke job now live under `foundation/operators/**` (wave15). |
| `22-control-plane` | `platform/control-plane/*`. | ✅ envoy gateway, platform gateway, cert-manager config, smoke jobs mapped to waves 30–33. |
| `24-auth` | `platform/auth-stack`. | ✅ postgres clusters, Keycloak instance/realm, auth smoke (waves 34–37). |
| `40-data-plane` | `platform/data-plane/*`. | ✅ redis failover, Kafka cluster, ClickHouse, pgAdmin, data-plane smoke (waves 20–24). |
| `42-db-migrations` | `platform/db-migrations` (hook jobs). | ✅ `data/db/db-migrations` component uses Helm chart + PreSync job. |
| `45-data-plane-ext` | `platform/data-plane-ext/*`. | ✅ temporal stack plus MinIO/Neo4j components; MinIO stays local-only via the `local` overlay (single replica, `local-path` storage), and Neo4j lives as `component.disabled.yaml` until we intentionally re-enable it. |
| `50-product-services` | `apps/<service>` (use ApplicationSet generator for app directories). | ✅ all platform services (platform, graph, inference, runtime agents, www, etc.) now under `components/apps/**`. |
| `55-observability` | `platform/observability/*`. | ✅ Prometheus stack, Grafana + datasources, Loki/Tempo, OTEL collector, Langfuse (+ bootstrap) wired into waves 40–45. |
| `60-qa` | `platform/qa/platform-smoke`. | ✅ platform smoke job as Helm test (wave55). |

_Current status:_ `foundation-dev` ApplicationSet now gates namespaces → CRDs → CoreDNS → all operators (cert-manager, ESO, CNPG, Strimzi, Redis, ClickHouse, Keycloak) → Vault/secrets across waves `00`–`21`, and the operators smoke job runs in the same step. _Next:_ migrate Helmfile **22-control-plane** into GitOps components so the platform tier (Ingress/Gateway, policies, etc.) is fully declarative.

_Dec 2025 checkpoint:_ new Tier 1 workloads (e.g., `extensions-runtime` from Backlog 480) and refreshed frontends (`www-ameide-platform` default org wiring) are entering the platform tier. Action items:

- Add `components/apps/extensions-runtime` with `tier: apps`, `wave: 36`, matching the service’s Helm chart + image tags so ApplicationSets own its lifecycle (Tilt now builds/pushes both dev + release images to GHCR).
- Ensure `services/www_ameide_platform` component consumes the new values contract (`services.www_ameide_platform.organization.defaultOrg`) and that dev/staging/prod overlays set it, otherwise Helm rendering fails per §1 (“New required value”).
- Delete or re-label any legacy Helmfile-applied Deployments/Secrets in namespaces now owned by ApplicationSets (especially platform gateways and Temporal) so Argo’s orphan detection surfaces real drift instead of false positives.

_Layer 15 progress:_ `foundation-cert-manager`, `foundation-external-secrets`, Vault server + PVC, ClusterSecretStores, every `vault-secrets-*` bundle, integration secrets, and the secrets smoke job now live under `environments/dev/components/foundation/secrets/**` (waves 15–21). Helmfile `15-secrets-certificates.yaml` has been deleted. All ExternalSecrets report `Ready=True`; Argo shows them Healthy (OutOfSync only because the controller injects provider defaults).

_Data-plane extensions policy:_ MinIO and Neo4j charts are vendored like any other dependency, but shared values pin `enabled: false`. Only the local overlay (`sources/values/env/local/data/data-minio.yaml`) flips MinIO on, forces `mode: standalone`, sets `statefulset.replicaCount: 1`, and binds both `global.defaultStorageClass` + `persistence.storageClass` to `local-path` so k3d/dev gets object storage backed by the devcontainer disk. Neo4j stays disabled everywhere—the component lives as `component.disabled.yaml`, so ApplicationSets skip it entirely until we intentionally rename it to `component.yaml` and provide environment overrides. Temporal core, namespace bootstrap, and oauth proxy are standard components (waves 26–28), followed by the `data-plane-ext-smoke` job (wave29).

_Nov‑13 refresh:_ All third‑party chart archives were unpacked into real directories so Argo’s repo-server can read `Chart.yaml` without a pre-render step, and every ApplicationSet now pulls shared + environment values via the multi-source `$values/...` pattern. This lets us drive Helm entirely from Git (no pre-render), but it also surfaced runtime debt we still need to burn down:

- Foundation/app/data operators now render, but several CR workloads can’t start because referenced images/Secrets are missing (redis operator tag `v1.2.4`, pgAdmin OAuth secret, private `docker.io/ameide/*:dev` images, workflows-runtime namespace guard image, etc.).
- The Spotahome RedisFailover CRD now lives in `foundation/crds/redis`, mirroring the upstream Helm guidance (`kubectl replace ...databases.spotahome.com_redisfailovers.yaml`). The operator Application sets `helm.skipCrds=true` so future chart bumps don’t re-install the CRD out-of-band.
- Keycloak realm import JSON was cleaned up; we still need to delete the failed `master-realm-*` Jobs and re-sync `platform-keycloak-realm` to prove the new export is valid.
- Workflows runtime’s main Deployment still points at an unpublished image.
- MinIO now reconciles only in the local environment (the shared value set keeps it off elsewhere and the local overlay pins `local-path` storage); the Neo4j component is parked as `component.disabled.yaml`, so the ApplicationSet no longer generates an empty/failed Application for it.
- Helmfile-applied resources remain in the cluster, which is why every Application shows `OrphanedResourceWarning`. Once Argo owns a namespace, delete (or re-label) the legacy objects so drift detection can surface real issues.

Migration steps:

1. Create component directories + metadata.
2. Ensure ApplicationSet picks them up (labels, tier).
3. Apply + sync via Argo CD; verify health gating works.
4. Remove release from helmfile.
5. Delete helmfile once empty.

Track progress in a simple checklist table (foundation → platform → data → apps).

> **Rendering workflow:** when a chart upgrade is required, run `helm template` against the original Helmfile release, review the rendered diff, commit the updated manifests under `gitops/ameide-gitops`, and leave the Helmfile untouched except for version/values that future renders depend on. Helmfile should never be pointed at a real cluster again.

---

## 6. Health checks & sync options

* Add Lua health checks in `argocd-cm` for:
  * Postgres clusters / operators.
  * Gateway API, Ingress controllers, etc.
* Common `syncOptions` per component:
  * `CreateNamespace=true` when target namespace doesn’t exist.
  * `SkipDryRunOnMissingResource=true` for CRDs preceding CRs.
  * `PruneLast=true` for delicate workloads.
  * `RespectIgnoreDifferences=true` if ignoring controller-managed fields.

These can be set in `component.yaml` so ApplicationSet template injects them.

---

## 7. Bootstrap automation

`.devcontainer/postCreate.sh` already:

1. Installs k3d (registry + cluster).
2. Installs Argo CD (`install.yaml`) and waits for controllers.
3. Applies repo secrets (`repos/ameide-gitops.yaml`) from `.env` (`GITHUB_TOKEN`).
4. Applies AppProjects + ApplicationSets + ops config from `gitops/ameide-gitops`.
5. Starts a background port-forward to http://localhost:8443.

After ApplicationSets exist, bootstrap will automatically render all components; helmfile is no longer needed for local dev.

---

## 8. Next steps

1. Finish validating the foundation → data → platform → apps waves with the new multi-source ApplicationSets (use `argocd app sync -l tier=<tier>` so RollingSync can gate on health).
2. Resolve outstanding runtime blockers:
   - mirror or pin a redis-operator image that actually exists (current check `kubectl -n redis-operator get pods` returns “No resources found”), then re-sync `foundation-redis-operator` / `data-redis-failover`.
   - recreate pgAdmin’s bootstrap & OAuth secrets (or keep OAuth disabled in dev) so the Deployment can start.
   - publish/pull-through every `docker.io/ameide/*:dev` image (agents, platform, workflows, namespace guard) or add `imagePullSecrets`.
   - delete the failed `master-realm-*` Jobs, re-sync `platform-keycloak-realm`, and confirm the import succeeds with the new JSON.
3. Add Lua health checks for operators/data services so RollingSync can meaningfully block on `Healthy`.
4. Once Argo owns a namespace, prune the legacy Helmfile resources to clear the `OrphanedResourceWarning` noise.
5. Remove helmfiles as each tier is migrated; update docs/backlog to declare helmfile deprecated (they now exist purely for `helm template` when bumping charts).

Once done, both devcontainer + shared clusters use the same GitOps pipeline, with ApplicationSet progressive sync delivering health-gated waves per the vendor guidance.
