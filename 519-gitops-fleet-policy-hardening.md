# 519 – GitOps Fleet Policy + Values Schema Hardening

**Status:** In progress (partially executed)  
**Owner:** Platform SRE / GitOps  
**Depends on:** 364 (Argo bootstrap + ApplicationSets), 375 (RollingSync waves), 446 (namespace isolation), 452 (Vault RBAC isolation), 456 (GHCR mirror policy), 462 (secrets origin classification), 505 (devcontainer service target architecture)

---

## 0) Executive summary

This backlog hardens the GitOps repository so Argo CD/ApplicationSet waves stay deterministic across cluster types, Helm value layering is collision-safe, and secrets/images/bootstraps follow operator-first (destination cluster) patterns.

Primary outcomes:
- Eliminate Helm values merge footguns (e.g. top-level `cluster:` collisions) by introducing a namespaced global contract (`global.ameide.*`) and schemas.
- Make “component presence” and “component enablement” explicit and wave-safe per cluster type (local vs azure).
- Remove non-deterministic Helm-generated runtime secrets; converge on Vault KV + External Secrets Operator for stable secrets.
- Stop patching vendor charts directly; introduce wrapper charts as the policy translation layer.

---

## 0.1) Backlog alignment (how this relates)

- **364 / 375:** keep the existing AppSet + RollingSync wave model; only refine how apps are selected per cluster type (without changing `rollout-phase` semantics).
- **446 / 452 / 462:** finish the transition to “destination cluster secrets” by removing Helm-generated stable secrets and formalizing Vault KV → ESO → Secret as the default.
- **456:** turn “mirror vs upstream” into a first-class fleet policy, then translate it in wrappers per chart image schema.
- **505:** normalize “optional workloads” (e.g., devcontainer) behind explicit capability/enablement rather than ad-hoc local-only overrides.

---

## 1) Current state snapshot (as of now)

**What is already aligned**
- **AppSet waves still function**: component definitions retain `rolloutPhase` and RollingSync steps continue to gate rollouts.
- **AppSet strict templating is enabled**: `goTemplateOptions: ["missingkey=error"]` is enforced so mis-templated apps fail fast.
- **A `global.ameide.*` contract exists**: cluster/env globals now publish a Helm-native contract and at least one owned chart enforces it via `values.schema.json`.
- **The fleet contract is available to cluster-scoped apps too**: both local and Azure overlays ensure cluster-scoped operators/config can consume the cluster-type globals file (e.g. `sources/values/cluster/{local,azure}/globals.yaml`) when needed.
- **Cluster-type component set scaffolding exists (local)**: local can now be driven from `environments/local/components/**` (curation/allowlist work still pending). Azure keeps using `_shared` as the canonical full set until it needs to diverge.
- **Destination-cluster secret management is in place** for Vault + SecretStore + GHCR pull secrets; cross-namespace Vault role now explicitly includes `ameide-system`.
- **Argo drift mitigations exist** where needed:
  - CNPG drift fix for `cluster.*` key collisions applied in templates.
  - Argo CD system `resource.customizations` covers Temporal CRs and CRDs to keep health deterministic (no “Unknown” during controller/CRD ordering).
- **Backstage stable session secret is operator-managed**: cookie signing secret is sourced from Vault KV and synced via External Secrets Operator (no Helm randomness, no Argo diff ignores).
- **Postgres credential Secrets are operator-managed**: CNPG user Secrets are now sourced from Vault KV and synced via External Secrets Operator (no Helm `rand*/lookup` loops, no secret payload diff ignores).
- **Schema guardrails are expanding**: additional internal charts now validate the `global.ameide.*` contract (start with Backstage + CNPG config charts).
- **Postgres password drift can be reconciled (local-only)**: a gated PostSync hook Job can reconcile existing DB role passwords from the synced Secrets using the CNPG superuser secret. This is intended as a migration/self-heal tool when switching password sources (e.g. Helm-generated → Vault KV → ESO).
- **A proper chart toggle exists for one vendor chart fork**: Langfuse worker can be disabled (currently implemented inside the vendored chart tree).
- **Temporal bootstrap is operator-native**: Temporal namespaces are managed via `TemporalNamespace` CRs and Temporal DB readiness is gated by an idempotent Argo hook Job (no “run this once” manual bootstrap).

**What remains misaligned (gaps)**
- **Values schema collision risk remains repo-wide**: the worst offender (local top-level `cluster:`) is removed, but we still need to migrate remaining ambiguous root keys into the `global.ameide.*` contract and expand schema guardrails beyond a single chart.
- **Local disable semantics are inconsistent**:
  - Some components are disabled by “empty manifests” / pruning behavior instead of a first-class `enabled` contract.
  - Some components should not exist at all on local and should be excluded via the ApplicationSet generator rather than installed then pruned.
- **Non-deterministic runtime secrets still exist** in some charts and are currently masked via Argo diff ignores (migrate them to Vault KV → ESO → Secret, and remove ignores).
- **Vendor chart modifications exist in-tree** (e.g., Langfuse changes landed under `sources/charts/third_party/...`), which will complicate upstream upgrades.
- **Image policy is not centralized**: multi-arch requirements and “mirror vs upstream” decisions are handled ad-hoc per chart.
- **Bootstrap Jobs are not standardized**: job immutability, wait/retry patterns, and cleanup semantics vary per bootstrap chart.

---

## 2) Goals

1. **Collision-safe values layering**
   - Platform/fleet globals live under `global.ameide.*` (Helm-native propagation).
   - Avoid ambiguous root keys like `cluster`, `name`, `type`, `spec` in global values.
2. **Wave-safe fleet targeting**
   - AppSets install the right component set per cluster type while preserving `rolloutPhase` gating:
     - Azure: uses `environments/_shared/components/**` as the canonical full set.
     - Local: uses `environments/local/components/**` as a curated allowlist/subset.
3. **First-class enablement**
   - Every workload we own or wrap supports `enabled: true|false`.
   - For truly unsupported components, prefer “do not generate the Application” over “install disabled placeholder resources”.
4. **Operator-first secrets**
   - Stable runtime secrets come from Vault KV → ESO → Kubernetes Secret (per 446/452/462).
   - Eliminate Helm-generated stable secrets (no `rand*` + `lookup` loops in runtime charts).
5. **Wrapper charts for third-party**
   - Changes required for fleet policy are implemented in wrappers, not in `sources/charts/third_party/**`.
6. **Central image policy**
   - A single contract for mirror preference + multi-arch requirement, with wrapper translation to each chart’s image schema.
7. **Standard bootstrap runner pattern**
   - Consistent wait/retry/idempotency + cleanup semantics for bootstrap Jobs that gate waves.

---

## 3) Non-goals

- Rewriting the entire repo layout in one shot (must remain incremental and reversible).
- Converting every third-party chart immediately (prioritize the ones that currently cause drift or local incompatibility).
- Introducing new secret providers beyond Vault + ESO.

---

## 4) Target architecture sketch

### 4.1 `global.ameide.*` fleet contract (Helm-native)

New canonical globals contract (examples; exact fields to be finalized in schema):

```yaml
global:
  ameide:
    cluster:
      type: local|azure
      name: k3d-ameide|aks-...
      arch: arm64|amd64
    capabilities:
      operators:
        agentOperator: true|false
      workloads:
        devcontainerService: true|false
        langfuseWorker: true|false
    images:
      policy:
        preferMirror: true|false
        requireMultiArch: true|false
        mirrorRegistry: ghcr.io/ameideio/mirror
```

### 4.2 Cluster-type component sets

- Azure hosted clusters use `environments/_shared/components/**/component.yaml` as the canonical full set.
- `environments/local/components/**/component.yaml` is the curated subset for local k3d; either omits unsupported components or points to wrappers with `enabled=false`.
- If/when Azure needs to diverge from `_shared`, introduce a real `environments/azure/components/**` tree (not a symlink mirror) and document the divergence.

### 4.3 Wrappers for vendor charts

Example wrapper conventions:
- `sources/charts/platform/langfuse` depends on `sources/charts/third_party/langfuse/...`
- Wrapper implements:
  - `enabled` toggles
  - policy translation from `global.ameide.images.policy.*`
  - local capability gating
  - any extra manifests needed for Ameide conventions

### 4.4 Diff customization split (system vs app)

- **System-level (global, kind-based):** Argo CD `resource.customizations` for known controller/webhook mutations we never want to fight repo-wide.
- **App-level (component-specific):** `ignoreDifferences` only where a single component needs it.
- When using ignore rules that should apply during sync (not only diff), prefer enabling `RespectIgnoreDifferences=true` on those Applications (documented and intentional).

---

## 5) Delivery plan (incremental phases)

### Phase A — Fleet contract + compatibility bridge

1. Add `global.ameide.*` to `sources/values/base/globals.yaml` and cluster/env overlays.
2. Add a compatibility bridge for existing charts that read root keys:
   - keep current keys temporarily (`cluster.*`, etc.) but treat them as deprecated.
3. Add `values.schema.json` to at least one internal chart to validate `global.ameide.*` presence and structure.

**Exit criteria**
- `global.ameide.cluster.*` exists across environments.
- One chart fails fast if contract is missing/malformed.

### Phase B — ApplicationSet generator split by clusterType

1. Create `environments/local/components/**` (start minimal).
2. Patch AppSet overlays so the local overlay reads `environments/local/components/**`.
3. Azure remains on `environments/_shared/components/**` by default; only introduce `environments/azure/components/**` if Azure needs to diverge.
3. Enforce strict AppSet templating (fail fast):
   - `goTemplateOptions: ["missingkey=error"]`
4. Preserve wave labels (`rolloutPhase`) and validate RollingSync still progresses.

**Exit criteria**
- Local installs only the intended set of apps (no “unsupported-but-present” apps).
- `argocd app wait --selector rollout-phase=... --health` progresses without stalling on excluded apps.

### Phase C — First-class enablement pattern

1. For all Ameide-owned charts: add `enabled` gates.
2. For vendor charts: add enablement in wrappers (not vendored chart trees).
3. Establish a repo convention:
   - omit from component set when unsupported
   - use `enabled=false` only when we want the app “visible but off”

**Exit criteria**
- Local disable decisions are expressed either by omission or `enabled=false`, never by “empty manifests + prune”.

### Phase D — Secrets: remove Helm-generated stable secrets

1. Move Backstage session secret generation to Vault KV and sync via ESO.
2. Remove Helm `rand*/lookup` stable-secret generation from app charts (or disable those templates by default).
3. Keep `ignoreDifferences` only for truly controller-managed fields, not for secrets we can make deterministic.
4. Document when ESO generators are allowed (rotation-only), and prohibit their use for “must remain stable” secrets.

**Exit criteria**
- No runtime chart uses Helm randomness to produce a stable secret value.
- Backstage app is `Synced/Healthy` without secret payload ignores.

### Phase E — Image policy consolidation

1. Define `global.ameide.images.policy.*` as the only place to decide mirror + multi-arch requirements.
2. Implement translation in wrappers to each vendor chart’s image schema.
3. Add a lightweight lint check (script) that flags images that violate local multi-arch requirement (optional).

**Exit criteria**
- Local no longer needs ad-hoc “use upstream multi-arch image” overrides per chart.

### Phase F — Bootstrap Jobs standardization

1. Create a reusable “bootstrap runner” chart (or a shared template library) that supports:
   - dependency readiness wait
   - bounded retry/backoff
   - idempotent operations
   - TTL cleanup for completed Jobs
2. Migrate Temporal namespace bootstrap to the standard pattern.

**Exit criteria**
- Bootstrap jobs do not fail due to race-to-dependency readiness.
- Bootstrap jobs do not create immutable-template drift loops during Argo sync.

### Phase G — Diff customization and SSA policy

1. Move generic ignore rules into Argo CD system config (`argocd-cm` `resource.customizations`).
2. Keep component-level `ignoreDifferences` only for workload-specific, justified cases.
3. For remaining SSA noise (e.g., CNPG-managed fields), prefer:
   - removing the root cause (values collisions / non-determinism), otherwise
   - a scoped ignore rule + `RespectIgnoreDifferences=true` when appropriate.

**Exit criteria**
- No secret payload ignores remain for stable secrets.
- Any remaining ignore rules are scoped, documented, and consistent across environments.

---

## 6) Acceptance criteria (definition of done)

- Values collision class is eliminated: no chart receives unexpected `spec` keys due to global `cluster.*` merges.
- Local and Azure are cleanly separated by component descriptors while preserving waves.
- No vendored third-party chart trees contain Ameide-specific logic (all policy is in wrappers).
- Stable runtime secrets are sourced from Vault/ESO, not Helm randomness.
- Generic diff ignore rules live in Argo system config; component-level ignores are rare and documented.
- `argocd app list` shows no `Degraded|OutOfSync|Progressing` after a full bootstrap on local and one hosted environment.

---

## 7) Risks & mitigations

- **Large migration surface area** → execute in phases; keep compatibility bridge until wrappers are adopted.
- **Wave stalling if components are miscategorized** → start with small local component set; validate each wave before expanding.
- **Vendor upgrades become painful if we patch vendor charts** → enforce wrapper-only rule; add CI lint to block edits under `sources/charts/third_party/**` except version bumps.

---

## 8) Tracking checklist

- [ ] Phase A: `global.ameide.*` contract + schema
  - [x] AppSet injects `global.ameide.*` for env-scoped apps (missingkey=error)
  - [x] Schema guardrails added to initial owned charts (Backstage + CNPG config + Temporal)
  - [ ] Expand schema guardrails repo-wide; migrate remaining ambiguous root keys
- [ ] Phase B: clusterType component directories + AppSet wiring
  - [x] Local overlay reads `environments/local/components/**`
  - [x] Azure remains canonical on `environments/_shared/components/**`
  - [ ] Curate `environments/local/components/**` into a true local allowlist (omit unsupported apps)
- [ ] Phase C: enablement contract across owned/wrapped charts
- [ ] Phase D: stable secrets moved to Vault/ESO; remove Helm rand secrets
- [ ] Phase E: central image policy + wrapper translation
- [ ] Phase F: standardized bootstrap runner pattern
- [ ] Phase G: system-level diff customizations + SSA policy
  - [x] Scoped app-level drift mitigations added where required (e.g., GitLab defaulted fields)
  - [ ] Move generic drift rules into Argo system config where appropriate; minimize `ignoreDifferences`
