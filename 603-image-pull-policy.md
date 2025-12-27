# 603: Image Policy — Implementation Plan (GitOps)

**Status:** Implemented (clean-state enforced)  
**Owner:** Platform SRE / GitOps  
**Target state:** `backlog/602-image-pull-policy.md`

## Clean-State Contract (what “done” means)

- GitOps-managed environments (`local`, `dev`, `staging`, `production`) deploy digest-pinned image refs.
- First-party workloads (charts we control) use a single values knob: `image.ref`.
- Rollouts happen because **Git changed** (PR write-back), not because of `imagePullPolicy`.
- Rendered pod specs set `imagePullPolicy: IfNotPresent` explicitly (boring + stable diffs).

## GitOps Automation (one consistent approach)

### Local + dev (fully automated)

- Producer pushes new first-party images to `ghcr.io/ameideio/<repo>:dev`.
- GitOps automation resolves `:<repo>:dev` → digest and writes it into Git:
  - `.github/workflows/bump-local-dev-images.yaml`
  - `scripts/bump-local-dev-images.sh`
- PR auto-merges after checks → Argo CD sync → deterministic rollout.

Trunk-based note: `:dev` is a producer channel tag for “latest from `main`”, not a branch-derived environment.

This is “per commit” if producer CI triggers `repository_dispatch`; otherwise it runs on schedule.

### Staging + production (promotion PRs; human-gated)

- Promotion is “copy the exact digest forward”:
  - `scripts/promote-images.sh <source_env> <target_env>`
  - `.github/workflows/promote-images.yaml` (manual `workflow_dispatch`) opens the PR
- Humans approve/merge promotion PRs (branch protection) → Argo CD sync → deterministic rollout.

---

## Producer/tooling progress (log)

### 2025-12-26

- `ameide` (producer/tooling): PR https://github.com/ameideio/ameide/pull/404 (in review) removes floating `:dev`/`:main` defaults from local tooling and adds digest/ref support for operator chart + scaffolding.

### 2025-12-25

- `backlog`: PR https://github.com/ameideio/ameide-backlog/pull/50 captured success criteria and expanded the cross-repo checklist (historical context; the GitOps implementation plan has since evolved).

## Repo-Wide Checklist (implemented)

- Charts render `image: {{ required "image.ref is required" .Values.image.ref }}` (or equivalent) and set `imagePullPolicy: IfNotPresent` for every PodSpec (Deployments/Jobs/hooks/initContainers).
- Env overlays (`sources/values/env/{local,dev,staging,production}/**`) pin first-party images by digest.
- Shared values (`sources/values/_shared/**`) pin third-party images by digest (prefer chart `digest:` fields; avoid `tag: <version>@sha256:<digest>` when the chart reuses `tag` in Kubernetes labels).
- Third-party chart values explicitly set `pullPolicy`/`imagePullPolicy` to `IfNotPresent` when the upstream default is `Always` (digest pinning is the rollout mechanism; pull policy stays boring).
- Raw-manifest surfaces do not embed floating tags; embedded `image:` / `ref:` keys are digest pinned.
- CI enforces the policy:
  - `.github/workflows/image-policy.yaml`
  - `scripts/check-image-policy.sh` + `scripts/check-image-policy.py`

## Full GitOps Configuration Checklist (audit surface)

- **Values layering:** ApplicationSet loads base/cluster/env globals, node-profile constraints, shared component defaults, then env overrides (see `argocd/applicationsets/ameide.yaml`).
- **First-party charts:** every rendered PodSpec uses `image.ref` + `imagePullPolicy: IfNotPresent` (includes Jobs/hooks/initContainers).
- **Operators + primitives:** operators pinned by digest; primitive CR `spec.image` pinned by digest; rollouts driven by Git image ref changes.
- **Third-party charts:** pin by digest using chart-native digest support when available; patch vendored charts when necessary to avoid `tag@sha` leaking into labels.
- **Registry + auth:** `ghcr-pull` pull secret materialized before any private mirror images are used; GHCR mirror supports multi-arch for local arm64.
- **Argo CD health:** custom health checks exist for CRDs and core primitives; vendor charts should not emit invalid Kubernetes objects (labels, selectors, etc.).
- **CI automation:**
  - local/dev digest bump PRs auto-merge (`.github/workflows/bump-local-dev-images.yaml`)
  - staging/prod promotion PRs are human-gated (`.github/workflows/promote-images.yaml`)
  - policy gate blocks regressions (`.github/workflows/image-policy.yaml`)
- **Drift/noise:** prefer rendering explicit stable defaults over `ignoreDifferences`; use narrow ignore rules only when upstream charts can’t be made stable.

## Key GitOps Touchpoints

- Local/dev auto-bump PRs: `.github/workflows/bump-local-dev-images.yaml`
- Promotion PRs: `.github/workflows/promote-images.yaml`
- Policy gate: `.github/workflows/image-policy.yaml`
- Helpers: `scripts/bump-local-dev-images.sh`, `scripts/promote-images.sh`, `scripts/check-image-policy.sh`

## Findings Log (from implementation + ArgoCD verification)

- **ArgoCD `ComparisonError` from missing submodule commit:** `backlog` submodule pointed at a non-existent commit; updating the submodule fixed repo-server checkouts.
- **Helm render failures break Argo comparisons:** `cluster-keycloak-operator` had invalid YAML (indentation) and was fixed in GitOps so renders are deterministic.
- **Missing `image.ref` in overlays blocks primitives:** local stacktest primitive overlays were missing pinned refs; added digest-pinned refs to unblock reconciles.
- **Digest pinning surfaces “broken image” reality:** `process-transformation` ingress image was missing `/app/process-transformation-ingress`; GitOps mitigation was rollback to last known-good digest (producer fix required).
- **Third-party chart digest pinning gotcha:** encoding `tag: vX.Y.Z@sha256:...` broke the Alloy chart because it reuses `image.tag` in `app.kubernetes.io/version` labels; fixed by using the chart’s `image.digest` field and a plain semver `tag`.
- **Temporal digest pinning required an operator patch:** the community Temporal operator appended `:version` to `spec.image`, which prevented `repo@sha256:...`; fixed by deploying a patched operator image and pinning Temporal server/ui/admin-tools by digest.
- **Patched Temporal operator image requires pull secret:** because the patched operator is published to `ghcr.io/ameideio/temporal-operator`, the operator chart must set `imagePullSecrets: [{name: ghcr-pull}]` so the controller can pull it.
- **Probe stability fixes are GitOps-owned:** pgAdmin and Langfuse needed probe tuning to avoid kubelet-induced restarts under local contention.
- **CNPG readiness probe timeout can flap on local:** CloudNativePG readiness can time out under local control-plane contention; increasing `cluster.probes.readiness.timeoutSeconds` avoids “no endpoints” cascades where dependent workloads can’t connect to Postgres.
- **CNPG webhook defaulting causes Argo diff noise:** rendering CNPG defaulted fields explicitly in values keeps Argo CD from reporting perpetual OutOfSync on `postgresql.cnpg.io/Cluster`.
- **ArgoCD operational unblocks:** a stuck Application sync operation can require clearing `.operation` and forcing a hard refresh; prefer fixing the root render/apply errors in Git.

## Temporal (now compliant)

- Patched operator image: `sources/values/_shared/cluster/temporal-operator.yaml` points to `ghcr.io/ameideio/temporal-operator@sha256:...`.
- Digest-pinned TemporalCluster images: `sources/values/_shared/data/data-temporal.yaml` sets `cluster.image`, `ui.image`, `admintools.image` as `repo@sha256:...` refs.
- Build/publish helper: `.github/workflows/publish-temporal-operator.yaml` (and local helper `scripts/temporal-operator/build-and-push.sh`).

## Developer Tasks (outside GitOps)

- Publish digest-pinnable images for currently-disabled developer features (so they can be GitOps-managed without floating tags):
  - devcontainer service image
  - agent runtime image for `core-platform-coder`
- Publish internal mirror images as multi-arch manifest lists where local clusters can be arm64.

## Audit Commands

- Policy check: `bash scripts/check-image-policy.sh`
- Find floating tags (should be empty): `rg -n \":(dev|main|latest)\\b\" sources/values/_shared sources/values/env -S`
