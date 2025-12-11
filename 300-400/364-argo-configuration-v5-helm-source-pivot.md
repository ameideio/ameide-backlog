> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# backlog/364 – Helm-source pivot for Argo configuration v5

We are extending the v5 migration from Helmfile → Argo CD by switching **all components** (foundation, platform, data, apps) from “committed rendered YAML” to **Helm-sourced Applications**. ApplicationSets + RollingSync remain the orchestrator, but every layer now relies on Helm charts as the single manifest source.

The goal: **faster upgrades and less YAML churn everywhere** while preserving the existing guardrails (health gating, diff customisation, AppProjects).

---

## 1. What changes vs. the current plan

| Topic | Previous approach | New pivot |
| --- | --- | --- |
| Source of manifests | Every component stores rendered manifests (`manifests.yaml`) committed under `gitops/ameide-gitops/environments/<env>/components/**`. | **All** components (foundation + platform + data + apps) will ultimately reference Helm charts; rendered manifests are a temporary artifact until the migration completes. |
| Helmfile role | Render-only helper after each layer migrates; engineers ran `helm template` and copied YAML into GitOps repo. | Helmfile still exists for rendering when needed, but most Helm rendering happens inside Argo. No more manual copy/paste for app releases. |
| ApplicationSet template | Always emitted `source.path`. | Templates emit Helm sources exclusively (rendered from a higher-level template) so every ApplicationSet drives Helm rendering. |
| Values hierarchy | Stored under `infra/kubernetes/environments/**` and baked into rendered manifests. | Value files become inputs to the Helm source (listed in `component.yaml`). We can keep the existing values tree or relocate it next to components. |
| Helm hooks | Executed during `helm template`/Helmfile runs (or rewritten into Argo hooks). | Helm hooks still don’t run under Argo; critical ones must be converted to Argo resource hooks or separate components before flipping to Helm source. |

---

## 2. Scope of the pivot

*Every* Helmfile layer (00 → 60) will transition to Helm-sourced Applications. We’ll restart from the **foundation layers first**, then work upward, aiming for **zero committed manifests** under `gitops/ameide-gitops/environments/<env>/components/**` once the pivot completes.

Implications:

* Foundation tiers (namespaces, CRDs, operators, secrets) need Helm definitions if they don’t already exist; where we previously rendered raw YAML, we’ll point Argo to the chart (or wrap raw manifests in a “raw” chart) so Helm can render them.
* Platform/data/app tiers already have charts; they’ll convert after foundation is stable.
* During migration we can keep rendered files for safety, but each component’s definition-of-done includes deleting `manifests.yaml` once the Helm source proves stable.

**Repo layout checkpoints**

* `gitops/ameide-gitops/sources/values/<env>/...` now houses every environment value file (mirroring the old `infra/kubernetes/values` layout). Use `_shared/<tier>/<component>.yaml` + `<env>/<tier>/<component>.yaml` as the standard pair and set `ignoreMissingValueFiles=true`.
* `gitops/ameide-gitops/sources/charts/` contains all Helm charts (foundation, platform layers, apps, vendor CRDs). Components set `chart.path: sources/charts/...` so Argo pulls everything from the GitOps repo.
* `gitops/ameide-gitops/environments/<env>/components/` remains metadata-only; each component folder contains only `component.yaml` (no rendered manifests) plus optional docs.

---

## 3. Metadata changes (`component.yaml`)

Every component now **must** define Helm metadata; rendered YAML is no longer supported.

```yaml
name: platform-gateway
project: platform
namespace: ameide
tier: platform
wave: "wave31"
syncOptions:
  - CreateNamespace=true
  - RespectIgnoreDifferences=true

chart:
  repoURL: https://github.com/ameideio/infra-charts.git   # or oci://, or Helm repo URL
  path: charts/platform/gateway                            # optional (Git-based charts)
  chart: gateway                                           # optional (Helm repo/OCI)
  version: 1.5.4
  valueFiles:
    - gitops/ameide-gitops/sources/values/_shared/apps/platform/gateway.yaml
    - gitops/ameide-gitops/sources/values/env/dev/apps/platform/gateway.yaml
  ignoreMissingValueFiles: true
```

Fields:

* `repoURL` – required (Git, Helm repo, or OCI).
* `version` – required for Helm repo/OCI; optional when charts come from Git.
* `path` vs. `chart` – mutually exclusive depending on repo type.
* `valueFiles` – optional list; order matters (last wins). Leave it unset to inherit the default `_shared/<tier>/<component>.yaml` + `<env>/<tier>/<component>.yaml` pair emitted by the ApplicationSet; set it only when the component lives somewhere else (for example, `platform-gateway` uses `apps/platform/gateway.yaml`).
* `ignoreMissingValueFiles` – mirrors Argo’s Helm option.

Any `ignoreDifferences` or `syncOptions` already defined still apply. Lua health checks remain in `argocd-cm`.

---

## 4. ApplicationSet template updates

Argo CD already supports Go templating inside ApplicationSets via `spec.goTemplate: true` (Sprig enabled), so we stick with native templating—but **remember the official limitation:** only string fields can be templated. Lists/objects must be fixed in the template (or injected via `templatePatch` strings) and you can use `ignoreMissingValueFiles: true` to list a superset of value files per component.

Recommended pattern:

```yaml
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - git:
        ...
  template:
    metadata:
      name: '{{ .name }}'
      labels:
        tier: '{{ .tier }}'
        wave: '{{ .wave }}'
    spec:
      project: '{{ .project }}'
      source:
        repoURL: '{{ .chart.repoURL }}'
        path: '{{ .chart.path }}'             # or chart: '{{ .chart.chart }}'
        targetRevision: '{{ default "HEAD" .chart.version }}'
        helm:
          releaseName: '{{ .name }}'
          {{- $valuesEnv := default "local" (get $.metadata.labels "gitops.ameide.io/values-environment") }}
          {{- $defaultValueFiles := list (printf "$values/sources/values/_shared/%s/%s.yaml" .tier .name) (printf "$values/sources/values/%s/%s/%s.yaml" $valuesEnv .tier .name) }}
          {{- $valueFiles := default $defaultValueFiles .chart.valueFiles }}
          valueFiles:
          {{- range $valueFiles }}
            - '{{ . }}'
          {{- end }}
          ignoreMissingValueFiles: true
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{ .namespace }}'
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
          - RespectIgnoreDifferences=true
```

> If a component truly needs variable-length lists, use `templatePatch` (string) with `valueFiles` rendered via `toYaml` and remember that patches replace lists entirely—patch only the components that deviate.

> The `chart.valueFiles` override is intentionally optional. Most components inherit the `_shared/<tier>/<component>` + `<env>/<tier>/<component>` pair generated above, but edge cases (e.g., charts that live under `apps/platform/**`) can set their own list without forking the ApplicationSet template.

---

## 5. Migration plan (restart at foundation)

1. **Template/metadata prep (all tiers)**  
   * Extend `component.yaml` schema with the Helm fields.  
   * Enable `goTemplate: true` in each ApplicationSet (or split generators) so Helm metadata flows straight into `spec.source`. No external rendering step is needed; we rely on the controller’s native templating support.

2. **Foundation pass (layers 00–21)**  
   * For each component (namespaces, bootstrap, CRDs, secrets, CoreDNS, operators), ensure a Helm chart exists. Where only raw YAML exists, package it as a minimal Helm chart (values + templates).  
   * Update the component metadata to reference the chart + value files (likely reusing `infra/kubernetes/environments/**`).  
   * Apply via ApplicationSet, confirm RollingSync waves stay green, then delete the old `manifests.yaml`.

3. **Platform/control plane (layer 22) & auth (24)**  
   * Convert Envoy Gateway, platform gateway, cert-manager config, smoke jobs, and Keycloak/auth charts to Helm sources.  
   * Rewrite Helm test hooks as Argo resource hooks where necessary.

4. **Data plane + extensions (layers 40/45)**  
   * Migrate Redis/Kafka/ClickHouse/pgAdmin plus optional extensions (MinIO, extra queues) to Helm sources.  
   * Ensure data-layer health checks (CNPG, Strimzi, etc.) continue to gate RollingSync waves.

5. **Apps, observability, QA (layers 50/55/60)**  
   * Point application components at their Helm/OCI charts instead of keeping rendered manifests under `gitops/.../apps`.  
   * Observability stacks and QA smoke suites follow the same pattern.

6. **Backlog + tooling cleanup**  
   * Update the decomposition/detail docs with Helm-source status per layer.  
   * Remove rendered manifests, adjust scripts/docs to reflect Argo+Helm as the only runtime path, and archive any Helmfile automation that is no longer needed.

---

## 6. CRDs, hooks, and Helm dependencies

* **Charts shipping CRDs:** either move CRDs into an earlier Helm component and set `helm.skipCrds: true` on the charted workload, or keep CRDs with the chart but set `SkipDryRunOnMissingResource=true` (and `PruneLast=true` when needed) so RollingSync doesn’t choke while CRDs register. This mirrors the vendor guidance.
* **Large CRDs (e.g., CloudNativePG poolers):** Argo’s client-side apply path stores a full `last-applied-configuration` blob under `metadata.annotations`, which easily busts the 256 KB limit for fat CRDs. Enforce `ServerSideApply=true` in the ApplicationSet template (and per-component overrides when needed) so Argo uses SSA for creation/patching. We hit this directly on `foundation-cloudnative-pg`, where the `poolers.postgresql.cnpg.io` CRD refused to apply until SSA was enabled and the operator restarted; the runbook is now “flip SSA, resync the Application, then delete the controller pod so it boots with the newly registered CRDs.”
* **Git-hosted charts with dependencies:** Argo’s repo-server doesn’t run `helm dependency update`. Prefer OCI/Helm repos, or vendor the dependency charts under `charts/<name>/charts/` so they’re resolvable at render time.
* **Private/OCI repos:** make sure repo creds are registered in Argo and set `helm.passCredentials: true` when pulling charts across domains.  
* **Helm hooks:** leave Helm hook annotations in place when they already match the desired lifecycle (Argo maps most of them). When you need precise control, convert the job into an Argo resource hook (`argocd.argoproj.io/hook: PreSync|Sync|PostSync`).  
* **Multiple sources:** when values live in a separate repo, use Argo’s `sources` + `$values` support instead of a bespoke render step.

---

## 6. Day-2 workflows with Helm source

* **Upgrade**: bump `chart.version` or edit value files, merge PR, let Argo render+sync.
* **Rollback**: revert the Git commit; Argo syncs back to the previous chart version/values. No Helm release state to manually roll back.
* **Diff/drift**: keep using per-component `ignoreDifferences` + `RespectIgnoreDifferences=true`. Controller-mutated CRDs (ESO, CNPG) still need them; Helm at source doesn’t change that.
* **Health gating**: unchanged—RollingSync waits for Applications (rendered via Helm or otherwise) to reach Healthy before progressing.

---

## 7. Risks & mitigations

| Risk | Mitigation |
| --- | --- |
| ApplicationSet templates currently include Go template control blocks that `kubectl apply` can’t parse. | Introduce a render step (gomplate, Cue, ytt) before applying, or refactor templates to plain YAML with parameter placeholders. This is a prerequisite for any Helm-source logic anyway. |
| Helm hooks not executed by Argo. | Convert critical hooks to Argo `PreSync`/`PostSync` resources, matching the pattern already described in `backlog/364-argo-configuration-v5-principles.md`. |
| Values drift between `infra/kubernetes` and GitOps repo. | Long-term, relocate value files into `gitops/.../values`. Short-term, reference the existing files via relative paths so there’s a single copy. |
| Engineers still run Helmfile against clusters. | Keep `RENDER-ONLY` banners in `infra/kubernetes/helmfiles/*.yaml`, set `installed: false` on migrated releases (e.g., `22-control-plane.yaml` → `gateway`), and document that Helmfile is used solely for templating/testing, not deployment. |

---

## 8. What moves, what stays, what gets deleted

| Item | Outcome | Notes |
| --- | --- | --- |
| `gitops/ameide-gitops/environments/<env>/components/<tier>/<component>/manifests.yaml` (all tiers) | **Deleted** once the component runs as Helm source. | We keep the directory + `component.yaml`, but drop the rendered manifests to avoid dual sources. |
| Foundation component manifests (namespaces, CRDs, operators, secrets) | **Deleted once Helm-sourced** | Temporary copies can exist during migration, but final state removes all rendered manifests. |
| `component.yaml` files | **Kept & extended** | Metadata gains optional `chart.*` fields; existing fields (tier, wave, syncOptions, ignoreDifferences) remain. |
| ApplicationSet files under `gitops/.../argocd/apps/*.yaml` | **Kept & updated** | Generated YAML always references Helm sources; RollingSync, deletion order, and projects stay intact. |
| `infra/kubernetes/helmfiles/*.yaml` | **Kept as render-only** | Already tagged `RENDER-ONLY`; they serve as references for chart versions/values but are never applied to clusters. |
| `infra/kubernetes/charts/**` and `infra/kubernetes/values/**` | **Kept (referenced)** | Helm sources still need these chart definitions and value files. Later we may move values into the GitOps repo, but no immediate deletions. |
| Render-only helper scripts (e.g., `scripts/infra/ensure-k3d-cluster.sh`) | **Kept until new workflows exist** | Some scripts may be retired after Argo fully owns reconciliation; for now they provide local dev parity. |

---

## 9. Definition of done

* ApplicationSet templates render plain YAML for Helm-sourced Applications (no path-based fallback).
* Layers 00–60 in the dev environment sync successfully as Helm-sourced Applications (with health gates + diff ignores intact).
* Rendered manifests are removed for **all** components; GitOps repo only stores component metadata + values (and value files).
* Backlog docs note which tiers are Helm-sourced vs. rendered to keep auditors clear on the hybrid model.

---

## 10. Status snapshot (2025‑11‑13)

* **Chart vendoring completed.** Every third-party Helm archive under `sources/charts/third_party/**` was extracted and committed, so Argo’s repo-server never needs to download `.tgz` artifacts again.
* **ApplicationSets now use `$values`.** Each tier’s template adds a secondary Git source (`ref: values`) so Helm always loads `_shared/<tier>/<name>.yaml` plus the environment-specific overlay. Any new ApplicationSet must follow the same pattern; otherwise Helm will render without env overrides.
* **Redis operator still absent.** Running `kubectl -n redis-operator get pods` currently prints “No resources found”, so neither the operator nor the `data-redis-failover` instance can progress. Confirm the image/tag (`quay.io/spotahome/redis-operator:v1.2.4`) exists or mirror it to a reachable registry, then re-sync the foundation/data waves.
* **Blocking tasks before we can declare “all green”:**
  1. Publish accessible dev images for every `docker.io/ameide/*:dev` workload (agents, platform, workflows runtime, namespace guard) or add `imagePullSecrets`.
  2. Restore pgAdmin bootstrap + OAuth secrets (or keep OAuth disabled) so the Deployment leaves `CreateContainerConfigError`.
  3. Delete failed `master-realm-*` jobs and re-sync `platform-keycloak-realm` to exercise the new realm JSON.
  4. Re-run RollingSync wave-by-wave (foundation → data → platform → apps) after the above fixes land.
  5. Prune the legacy Helmfile resources so the fleet-wide `OrphanedResourceWarning` reflects real drift.
