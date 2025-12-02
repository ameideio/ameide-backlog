> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# backlog/364 – Argo configuration v5 (Helm pivot blueprint)

We already decomposed most Helmfile releases into `gitops/ameide-gitops/environments/dev/components/**`, but the legacy `infra/kubernetes` tree (charts, values, Helmfile orchestration) still exists and keeps growing. This note captures:

1. What currently lives under `infra/kubernetes` (and why it feels “messy”).
2. The target operating model where Argo CD consumes **Helm sources** directly (no rendered YAML committed for app/service tiers).
3. A step-by-step pivot plan so we can keep the good parts (charts, values, smoke jobs) while handing rollout control to Argo ApplicationSets.

---

## 1. Current state under `infra/kubernetes`

| Area | What we have today | Notes |
| --- | --- | --- |
| Helmfile orchestration | `infra/kubernetes/helmfile.yaml` fans out to 13 layer helmfiles (`00` → `60`) with strict `needs` ordering and stage selectors (`infra/kubernetes/helmfile.yaml:1-52`). | Layers map 1:1 with the v5 decomposition tables but still assume Helmfile owns reconciliation. |
| Charts & raw manifests | Custom charts for operators, platform services, tests, third-party overlays (`infra/kubernetes/charts/**`). | Many are thin wrappers around upstream charts; some (Envoy Gateway, smoke jobs) are bespoke. |
| Values hierarchy | Shared defaults + per-env overrides under `infra/kubernetes/environments/{_shared,local,staging,production}` plus ad-hoc values stitched inside charts (`infra/kubernetes/values/**`). | Environment merges happen at Helmfile runtime; nothing enforces schema or keeps envs in sync. |
| Operational scripts | `scripts/infra/ensure-k3d-cluster.sh`, `infra/kubernetes/scripts/infra-health-check.sh`, vendor chart downloaders, etc. | Tooling assumes `helmfile sync` is authoritative for infra provisioning. |
| Helmfile-specific glue | Hooks, selectors, `needs`, custom smoke tests, and the expectation that `helmfile diff/sync/test` gate every environment. | These concepts need translation when Argo takes over orchestration. |

**Why it feels messy now**

* Dual control planes: Helmfile still contains live releases even after we migrated the same workloads into GitOps components. A new engineer can’t tell which source of truth to trust without tribal knowledge.
* Values sprawl: service defaults live in `infra/kubernetes/values`, env overrides in `infra/kubernetes/environments/<env>`, and the new GitOps components have yet another set of env-specific manifests. Staying DRY across all three is nearly impossible.
* Helmfile-only features (selectors, `needs`, `helmfile test`) don’t map directly to Argo’s RollingSync/App hooks, so the behavior drifts between local Helmfile runs and Argo-managed clusters.

---

## 2. Target: Argo ApplicationSets with Helm sources

Principle recap:

* **Argo CD stays the reconciler**. ApplicationSets + RollingSync remain our wave orchestrator (`gitops/ameide-gitops/environments/dev/argocd/apps/*.yaml`).
* **Helm becomes the source for app/service tiers**: each generated Application points to a chart + value files instead of committed manifests. Helm renders on every sync, so day-2 upgrades are just PRs that bump `targetRevision` or values.
* **Rendered YAML remains** for CRD-heavy/operator tiers (foundation). Deterministic diffs there are helpful, and those layers already work via the current component directories.
* **Helmfile lives on as a rendering aide**: we keep the charts and helmfiles in-tree, but clearly mark them `RENDER-ONLY` and use them solely to template manifests or generate Helm values for components.

Implications:

1. Each component metadata file (`component.yaml`) must be able to describe the Helm chart source (repo URL/OCI ref, chart version, value files). We can extend the metadata schema with optional `chart`, `chartRepoURL`, `valueFiles`, etc.
2. ApplicationSet templates need dual-source support: if a component declares `chart`, we render via Helm; otherwise we fallback to committed manifests. (Today’s Go template logic only supports the latter.)
3. Environment overrides move next to the component, eliminating the current `_shared` + `environments/<env>` merge pipeline. We can still re-use the existing `values/` files by referencing them in `valueFiles`.

---

## 3. Pivot plan

### Phase 0 – Document and freeze Helmfile (in progress)

* [x] Mark helmfiles as `## RENDER-ONLY` and update backlog docs to state they are not applied anymore (`infra/kubernetes/helmfiles/*.yaml` already carries the banner for layers 00/10/12/18/20).
* [ ] Add a top-level README blurb (or `CLAUDE.md`) that mirrors the new policy so future contributors don’t run `helmfile sync` against shared clusters.
* [ ] Create a helper script (`scripts/render-helmfile.sh <layer>`) that shells into Helmfile purely to run `helm template` for a release. This keeps the workflow alive for engineers who prefer Helmfile to gather values/Deps when regenerating YAML.

### Phase 1 – Inventory and classify releases

* Build a table (derived from `helmfile.yaml` + the layer-specific files) that tags each release as **rendered YAML** vs **Helm source**:
  * Foundation tiers (namespaces, CRDs, operators, Vault/secrets) stay rendered.
  * Platform/data/app tiers become Helm-sourced unless they ship CRDs/hooks that Argo can’t run.
* Capture extra metadata needed for Helm sources: chart repo, path, version, value files currently referenced in Helmfile. Most of this sits inside `infra/kubernetes/charts` + `infra/kubernetes/environments/<env>`.

### Phase 2 – Extend component metadata + ApplicationSets

* Update `component.yaml` schema to include optional Helm fields:

  ```yaml
  chart:
    repoURL: oci://docker.io/envoyproxy/gateway-helm
    chart: gateway
    version: 1.5.4
    valueFiles:
      - gitops/ameide-gitops/sources/values/_shared/apps/platform/gateway.yaml
      - gitops/ameide-gitops/sources/values/dev/apps/platform/gateway.yaml
    ignoreMissingValueFiles: true
  ```

* Teach the ApplicationSet templates (`gitops/ameide-gitops/environments/dev/argocd/apps/*.yaml`) to emit either:
  * `source.path` (current behavior) when `chart` is absent.
  * `source.chart`/`source.repoURL` + `source.helm.valueFiles` when `chart` is present.

* Retain RollingSync, deletion order, and per-component `syncOptions` regardless of source type.

### Phase 3 – Migrate layer 22 (control plane) as the wedge

1. For `platform/control-plane/*` components, replace `manifests.yaml` with Helm source references:
   * `platform-envoy-gateway` → Envoy Gateway chart (currently defined in `helmfiles/22-control-plane.yaml:14-41` with values from `charts/values/infrastructure/envoy-gateway.yaml` + env overrides).
   * `platform-gateway` → internal chart `charts/platform/gateway`.
   * `platform-cert-manager-config` and `control-plane-smoke` can remain rendered YAML until the Helm logic for hooks is ready.
2. Update the matching `component.yaml` files with the Helm metadata, plus `valueFiles` pointing to the existing values.
3. Apply ApplicationSets and verify Argo renders the charts correctly. Watch `argocd-applicationset-controller` logs for Helm rendering errors.

### Phase 4 – Expand to data plane + app tiers

* Repeat the process for `infra/kubernetes/helmfiles/40-data-plane.yaml` and beyond. Many releases there already reference charts in `infra/kubernetes/charts/operators-config/*`.
* For first-party services (layer 50), we can direct Argo to the repo/OCI artifact that already contains the app chart instead of committing rendered manifests under `gitops/ameide-gitops/environments/dev/apps/**`.

### Phase 5 – Clean up values & tooling

* Once a tier is Helm-sourced from Argo, delete the redundant rendered manifests under `gitops/ameide-gitops/environments/dev/components/<tier>/.../manifests.yaml`.
* Move any remaining shared values from `infra/kubernetes/environments/_shared` into a `gitops/.../values/` directory that sits alongside components so everything lives inside the GitOps repo.
* Deprecate Helmfile-specific scripts (`infra/kubernetes/scripts/infra-health-check.sh`) or teach them to call Argo CLI / kubectl instead.

---

## 4. Day-2 operations under the Helm pivot

* **Chart upgrades:** modify `component.yaml` (`chart.version` or value file) and submit a PR; Argo re-renders automatically on merge. No local `helm template` required unless you want to preview diffs.
* **Diff/drift controls:** keep the existing `ignoreDifferences` blocks inside components and `RespectIgnoreDifferences=true` in ApplicationSets. Operator-managed CRDs (ExternalSecrets, CNPG) still need those guardrails no matter how the manifests were produced.
* **Health gating:** RollingSync + Lua health checks remain unchanged; Argo waits for Helm-rendered Applications to become Healthy before advancing the wave.
* **Smoke tests / hooks:** convert Helm test pods into Argo resource hooks (`argocd.argoproj.io/hook: PostSync`, etc.). This ensures they run even though Helm hooks aren’t executed by Argo.

---

## 5. Risks & mitigations

| Risk | Mitigation |
| --- | --- |
| Helmfile remains in muscle memory; someone runs `helmfile sync` against dev/prod. | Prominent warnings + CI guardrail (GitHub Action that fails if Helmfile diffs vs GitOps drift?), plus docs/automation that only ever call Argo. |
| ApplicationSet template currently uses Go template control blocks that `kubectl apply` rejects. | Introduce a render step (e.g., gomplate, Cue) or refactor templates to plain YAML + `$values` substitution before we rely on them in automation. |
| Value files referenced from Helm sources drift relative to the GitOps repo. | Co-locate value files with components (or at least under `gitops/ameide-gitops`) and treat `infra/kubernetes/environments` as deprecated once migrations finish. |
| Helm hooks not running under Argo. | During Phase 3+ we must rewrite Helm test jobs into Argo hooks or standalone components so we don’t lose current health gates. |

---

## 6. Definition of done

* Helmfile is clearly flagged as render-only, with automation/tests ensuring it is never applied to a live cluster.
* Every ApplicationSet in `gitops/ameide-gitops/environments/<env>/argocd/apps/*.yaml` can render both manifest- and Helm-backed components.
* Layer 22 (control plane) is Helm-sourced and syncing green via Argo.
* Layer 40+ components (data plane, product services) reference Helm charts directly; rendered YAML exists only for operator foundations and intentionally “frozen” manifests.
* `infra/kubernetes/environments/*` is either deleted or mirrored inside the GitOps repo so there’s a single path for environment overrides.

Once these conditions are met, “Argo + Helm” becomes the default story for platform/app layers, while foundations continue to enjoy the determinism of committed manifests. This gives us the faster upgrade loop we want without losing the guardrails already codified in backlog/364.

---

## 5. Decision log (2025‑11‑13)

* **MinIO stays local-only.** Shared values keep it disabled everywhere, while the local overlay flips it on, forces single-replica/standalone mode, and pins both `global.defaultStorageClass` + `persistence.storageClass` to `local-path` so PVCs land on the devcontainer disk.
* **Neo4j remains disabled everywhere.** The component lives as `component.disabled.yaml`, so ApplicationSets ignore it entirely until we intentionally rename the file and supply environment overrides; the chart/values stay vendored for future use.
* **Third-party charts are stored unpacked.** Every Helm tarball under `sources/charts/third_party/**` has been extracted so Argo’s repo-server can read `Chart.yaml` directly; do not re-commit `.tgz` bundles or the controller will regress to `Chart.yaml: no such file`.
* **ApplicationSets use multi-source `$values`.** All tier AppSets now declare a second source with `ref: values` so `$values/sources/values/_shared|local/<tier>/<name>.yaml` is available to Helm. Any new ApplicationSet must follow the same pattern otherwise the charts will render without env overrides.

**Operational backlog (blocking “all green”)**

1. Verify the redis-operator image (`quay.io/spotahome/redis-operator:v1.2.4`) exists or publish a mirror and update `foundation-redis-operator` values.
2. Restore pgAdmin bootstrap/OAuth secrets (or keep OAuth disabled) so the Deployment can leave `CreateContainerConfigError`.
3. Publish public images for every `docker.io/ameide/*:dev` workload (agents, platform, workflows, namespace-guard) or add `imagePullSecrets` so the pods don’t die with `ImagePullBackOff`.
4. Delete the failed `master-realm-*` Jobs and re-sync `platform-keycloak-realm` to confirm the regenerated realm export succeeds.
5. Produce an accessible image for `workflows-runtime` (the namespace guard hook is now optional, but the core Deployment still needs the dev image).
6. After Argo owns a namespace, prune/remap the legacy Helmfile resources to eliminate the `OrphanedResourceWarning` noise.

These decisions live in the GitOps repo (component metadata + values) and are documented here so future layers don’t reintroduce “dual mode” deployments.
