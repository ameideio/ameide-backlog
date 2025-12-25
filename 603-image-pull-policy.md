# 603: Image Pull Policy — Refactoring Plan (GitOps + Operators)

**Status:** Ready to implement  
**Owner:** Platform SRE / GitOps  
**Scope:** Implement the target state in `backlog/602-image-pull-policy.md`  
**Related:** `backlog/602-image-pull-policy.md`, `backlog/495-ameide-operators.md`, `backlog/503-operators-helm-chart.md`, `backlog/450-argocd-service-issues-inventory.md`, `backlog/519-gitops-fleet-policy-hardening.md`, `backlog/456-ghcr-mirror.md`

## Goal

Make “Git commit → deployed artifact” deterministic in GitOps-managed environments by:

- eliminating reliance on mutable tags for rollouts
- pinning by digest (or immutable SHA tag + digest)
- completely removing floating `:dev` / `:main` from GitOps-managed deployments (treat them as aliases-only, or stop publishing them)
- making `imagePullPolicy` usage consistent and intentional (not accidental drift)

## Current state (known reality)

- Some workloads still use floating tags (especially during rapid iteration).
- GitOps values still contain a large amount of `image.tag: dev` / `image.tag: main` usage (i.e., “floating dev/main” is still real in practice).
- `imagePullPolicy` is currently used to force refreshes when tags move, but it does not trigger rollouts on its own.
- Argo can surface drift from Kubernetes-defaulted fields when diff mode changes.

## Refactoring work packages

### WP-1: Primitive CRD contract completeness (repo: `ameide`)

**Target:** all primitive CRDs support `spec.imagePullPolicy`, and operators apply it to generated Deployments/Jobs.

**DoD**
- `spec.imagePullPolicy` exists on Domain/Process/Agent/UISurface/Projection/Integration CRDs.
- Operators copy it into all generated pod templates (Deployments + Jobs, including migrations).
- Tests: at least one controller test per kind asserts `imagePullPolicy` propagation.

### WP-2: GitOps values contract for primitives (repo: `ameide-gitops`)

**Target:** primitives are configured via a single predictable value key, and rendering never relies on brittle template helpers inside `tpl`.

**DoD**
- A single “fleet contract” value exists for primitives (example: `global.ameide.primitives.imagePullPolicy`).
- All primitive CR manifests render `spec.imagePullPolicy` from that value.
- No `dig`/`get` usage that fails when `.Values` is not a plain map in `tpl` contexts.
- Docs updated: `backlog/495-ameide-operators.md` and `backlog/520-primitives-stack-v2-tdd.md` reference 602/603.

### WP-3: Digest pinning support in GitOps values (repo: `ameide-gitops`)

**Target:** every GitOps-managed environment can deploy primitives by digest.

**Approach**
- Prefer setting the primitive CR’s `spec.image` to `repo@sha256:<digest>` directly.
- Values should support either:
  - `image.digest` (preferred), or
  - `image.tag` + `image.digest` (readability), or
  - `image.tag` only (explicitly transitional and tracked).

**DoD**
- Primitive value templates support digests cleanly.
- Staging/prod overlays require digests (policy enforcement in CI; see WP-5).

### WP-4: Continuous update mechanism for the “fast-moving” environment

We need one environment that moves quickly without humans manually restarting pods.

Pick one:

- **Option A (PR-based):** CI opens PRs that update digests in GitOps.
- **Option B (Image Updater):** Argo CD Image Updater with Git write-back.
- **Option C (Kargo):** promotion controller committing refs per stage.

**DoD**
- A new build results in a Git change (digest update) and an Argo-driven rollout without manual intervention.
- Rollout events are auditable (commit shows old/new digest).

### WP-5: Policy enforcement (CI + optional admission control)

**CI enforcement (required)**
- Reject floating tags in GitOps-managed environment directories (except explicit allowlisted local-only transitional cases).
- Require digest pinning for staging/prod (and for cluster-scoped operators).

**Admission control (optional)**
- Gatekeeper/Kyverno policy to require digests in target namespaces.

**DoD**
- CI fails fast when a disallowed tag policy is violated.
- The allowlist is explicit and time-boxed.

### WP-6: Reduce Argo drift/noise related to defaulted image fields

**Target:** Argo stays boring: no spurious OutOfSync from defaulted `imagePullPolicy`.

**DoD**
- Workloads either set `imagePullPolicy` explicitly or `ignoreDifferences` is applied narrowly where defaulting is unavoidable.
- Drift guidance is documented in `backlog/519-gitops-fleet-policy-hardening.md` and referenced from 602/603.

## Codebase audit: where floating `:dev` / `:main` shows up today

This is the starting map for “what must change” if we implement 602 and want to fully remove floating `:dev`/`:main` from GitOps-managed environments.

- **GitOps env overlays:** `gitops/ameide-gitops/sources/values/env/**` uses `image.tag: dev` and `image.tag: main` broadly (dev/local/staging/production).
- **Cluster-scoped operators:** `gitops/ameide-gitops/sources/values/cluster/local/ameide-operators.yaml` pins operators to `tag: dev` and uses `global.imagePullPolicy: Always`.
- **Smoke job runner image:** `gitops/ameide-gitops/sources/values/_shared/apps/*-smoke.yaml` uses `image: ghcr.io/ameideio/grpcurl-runner:dev`.
- **Embedded platform manifest:** `gitops/ameide-gitops/sources/values/_shared/platform/platform-devcontainer-service.yaml` deploys `ghcr.io/ameideio/devcontainer-service:main` in a raw YAML string.
- **CLI publish path:** `packages/ameide_core_cli/internal/commands/primitive_publish.go` forces `:dev` (explicitly rejects other tags).
- **Scaffolding defaults:** `packages/ameide_core_cli/internal/commands/primitive_scaffold.go` and `packages/ameide_coding_helpers/scaffold/mcp_adapter.go` generate `:dev` image references by default.
- **CI publishes aliases:** `.github/workflows/cd-service-images.yml` updates `:dev` and `:main` tags as moving pointers (decide whether to keep as human convenience aliases).

## Refactoring checklist: no floating `:dev` / `:main` in GitOps-managed environments

### GitOps (repo: `ameide-gitops`)

- [ ] Replace `image.tag: dev` / `image.tag: main` with `image.digest` (preferred) or immutable SHA tag + digest across `gitops/ameide-gitops/sources/values/env/{dev,local,staging,production}/**`.
- [ ] Replace all direct `:dev` / `:main` image strings in GitOps values (example: `grpcurl-runner:dev` in `_shared/apps/*-smoke.yaml`) with digest-pinned references.
- [ ] Update embedded manifest strings (example: `platform-devcontainer-service.yaml`) to use digest-pinned images.
- [ ] Add digest support to cluster-scoped operator chart(s) and values:
  - `gitops/ameide-gitops/sources/charts/platform/ameide-operators/**` should render `repo@sha256:<digest>` when a digest is provided.
  - `gitops/ameide-gitops/sources/values/cluster/local/ameide-operators.yaml` should stop using `tag: dev` + `imagePullPolicy: Always` as the “rollout” mechanism.
- [ ] Time-box and remove `imagePullPolicy: Always` usage that exists only to compensate for floating tags; once digests are in Git, the default should be boring (`IfNotPresent`).

### CI + automation (repo: `ameide` + GitOps update mechanism)

- [ ] Ensure every published image has an immutable identity available to automation (digest + unique tag); make the mapping easy to consume for GitOps write-back.
- [ ] Implement WP-4 (Option A/B/C) so “new build” always produces a Git change in the fast-moving environment (digest update) → Argo rollout, without manual restarts.
- [ ] Decide whether `:dev` / `:main` remain as convenience aliases (never referenced by GitOps values) or are removed from publishing entirely.
- [ ] Update devcontainer publishing/consumers so deployments do not depend on `devcontainer:main` / `devcontainer-service:main` floating tags.

### Developer tooling defaults (repo: `ameide`)

- [ ] Update `ameide primitive publish` to support publishing non-`:dev` tags (e.g., `:<sha>` / `dev-<sha>` / `main-<sha>`) and to surface the resolved digest; remove the “only :dev is supported” restriction.
- [ ] Update scaffolding (`primitive scaffold`, coding helpers) to avoid generating committed manifests that reference floating `:dev`; prefer placeholders or digest-aware value patterns aligned to 602.

### Enforcement (CI + optional admission control)

- [ ] Add CI enforcement in `ameide-gitops` that rejects `tag: dev` / `tag: main` in GitOps-managed environment directories, with an explicit allowlist only for Tilt-only placeholder releases.
- [ ] Enforce the same rule for cluster-scoped operators and any raw manifest surfaces (embedded YAML, Jobs, hooks).
- [ ] Optional: admission control (Kyverno/Gatekeeper) to require digests in target namespaces (ensure this complements — not replaces — Git being the source of truth).

### Docs cleanup

- [ ] Update docs that currently teach `tag: dev` / `tag: main` (example: `operators/helm/README.md`) to show digest/SHA-tag patterns consistent with 602.
- [ ] Update runbooks that recommend “restart pods to pull the new `:dev`” so the new normal is “merge digest update PR → Argo rollout”.
## GitOps configuration checklist (end-to-end)

Use this as a “touch every place images are defined” checklist for implementing `backlog/602-image-pull-policy.md` across `ameide-gitops`.

### 1) Decide and standardize the values contract (stop “tag sprawl”)

- [ ] Remove floating defaults for primitives from `sources/values/base/globals.yaml` (`global.ameide.primitives.imageTag: dev`) and replace with an explicit digest-first contract (example: `global.ameide.primitives.imageDigest` or `global.ameide.primitives.imageRef`).
- [ ] Remove `global.ameide.primitives.imageTag: main` from `sources/values/env/staging/globals.yaml` and `sources/values/env/production/globals.yaml` and replace with per-primitive digest pinning (see primitives section below).
- [ ] Audit all “pull policy defaults” and make them intentional:
  - [ ] Change `imagePullPolicy: Always` to `IfNotPresent` in `sources/values/cluster/azure/globals.yaml` once all images are digest-pinned.
  - [ ] Change `global.ameide.primitives.imagePullPolicy: Always` to `IfNotPresent` in `sources/values/env/local/globals.yaml` and `sources/values/env/dev/globals.yaml` once primitives deploy by digest.
- [ ] Remove unconditional `pullPolicy: Always` defaults in shared app values (they were compensating for mutable tags):
  - [ ] `sources/values/_shared/apps/*.yaml` (examples: `sources/values/_shared/apps/graph.yaml`, `sources/values/_shared/apps/workflows.yaml`, `sources/values/_shared/apps/www-ameide-platform.yaml`)
  - [ ] `sources/charts/shared-values/platform/*.yaml`

### 2) Make every chart/template accept digests (operators are the outlier)

- [ ] Confirm (or add) digest support in all first-party app charts under `sources/charts/apps/*` (most already support `image.digest` → `repo@sha256:...`).
- [ ] Add digest support to the operators chart (currently tag-only):
  - [ ] Update templates under `sources/charts/platform/ameide-operators/templates/*-deployment.yaml` to render `image: repo@digest` when `operators.<name>.image.digest` is set.
  - [ ] Update the values schema/README for the operators chart under `sources/charts/platform/ameide-operators/` so each operator image has `repository`, `tag`, and `digest` (digest preferred).
- [ ] Verify the db-migrations chart stays digest-first (it already supports `migrations.image.digest`): `sources/charts/platform/db-migrations/templates/job.yaml`.

### 3) Convert environment overlays to digest pinning (staging/prod first)

- [ ] Remove the “dev vs local” split at the GitOps policy layer (keep only one “fast-moving” environment concept):
  - [ ] Decide whether `env/dev` remains a real cluster namespace (AKS) or is replaced/renamed to `env/local` (or `env/sandbox`).
  - [ ] If removing/renaming `dev`, update `config/clusters/azure.yaml`, `sources/values/env/dev/**`, `argocd/README.md`, and bootstrap defaults in `bootstrap/bootstrap.sh`.
- [ ] Replace `tag: main` + `pullPolicy: Always` with `digest: sha256:...` + `pullPolicy: IfNotPresent` in:
  - [ ] `sources/values/env/staging/apps/*.yaml`
  - [ ] `sources/values/env/production/apps/*.yaml`
- [ ] Eliminate `:latest` usage by pinning digests in local overlays:
  - [ ] `sources/values/env/local/data/data-minio.yaml`
  - [ ] `sources/values/env/local/platform/platform-backstage.yaml`
- [ ] Fix test/integration jobs that still use mutable tags:
  - [ ] `sources/values/env/staging/tests/integration/transformation.yaml` (currently `tag: dev`)
  - [ ] `sources/values/env/production/tests/integration/transformation.yaml` (currently `tag: dev`)
- [ ] Stop using `tag: dev-placeholder` in local/dev UI tilt values and switch to either digest pinning or a deterministic content-based tag that is written back to Git:
  - [ ] `sources/values/env/local/apps/www-ameide-tilt.yaml`
  - [ ] `sources/values/env/dev/apps/www-ameide-tilt.yaml`

### 4) Convert primitives (CR manifests) to digest pinning

Primitives currently construct `spec.image` from `repository:tag` and inherit global tags (`dev`/`main`). Target state is `spec.image: repo@sha256:...` (or `repo:sha@sha256:...`).

- [ ] Update all primitive value templates under `sources/values/_shared/apps/*-v0*.yaml` to support digest pinning (example file: `sources/values/_shared/apps/domain-transformation-v0.yaml`).
  - [ ] If a digest is set, render `spec.image` as `repo@digest`.
  - [ ] If a tag+digest pair is set, render `spec.image` as `repo:tag@digest` (optional readability).
  - [ ] If only a tag is set, treat as explicitly transitional (tracked and time-boxed per 602/603).
- [ ] Remove `_shared` defaults that force `tag: dev` for primitive runtime images (example: `sources/values/_shared/apps/domain-transformation-v0.yaml`) and require env overlays to provide digests for GitOps-managed envs.
- [ ] Update primitive smoke tests to pin `grpcurl-runner` by digest (no `:dev`):
  - [ ] `sources/values/_shared/apps/*-smoke.yaml` (examples: `sources/values/_shared/apps/domain-transformation-v0-smoke.yaml`, `sources/values/_shared/apps/process-ping-v0-smoke.yaml`)

### 5) Convert cluster-scoped operators to digest pinning (no floating tags)

- [ ] Replace `tag: dev` with digest pinning for all operators in `sources/values/_shared/cluster/ameide-operators.yaml`.
- [ ] Remove “force pulls because :dev moves” behavior from local overrides once pinned:
  - [ ] `sources/values/cluster/local/ameide-operators.yaml` (currently sets `global.imagePullPolicy: Always` and `operators.*.image.tag: dev`)

### 6) Update CI/render validation and chart tests to match digest-first policy

- [ ] Update the hardened render script to stop assuming `:dev` and to validate digest pinning rules:
  - [ ] `scripts/validate-hardened-charts.sh` (currently renders `dev staging production` and expects `...:dev` patterns)
- [ ] Update Helm unit tests in charts that hard-code `:dev` expectations:
  - [ ] `sources/charts/apps/*/tests/image_test.yaml` (examples seen under `sources/charts/apps/agents/tests/image_test.yaml`)
- [ ] Add a repo-level CI gate that fails if floating tags appear in GitOps-managed env overlays (allowlist must be explicit + time-boxed):
  - [ ] Enforce on `sources/values/env/staging/**` and `sources/values/env/production/**`
  - [ ] Enforce on cluster-scoped operator values (`sources/values/_shared/cluster/ameide-operators.yaml`)

### 7) Bootstrap/tooling alignment (remove “dev-tag” assumptions)

- [ ] Fix bootstrap build hooks that reference non-existent scripts and “dev-only” flows:
  - [ ] `bootstrap/bootstrap.sh` (references `${REPO_ROOT}/scripts/build-all-images-dev.sh`, which does not exist in this repo)
  - [ ] Decide whether bootstrap should build/push images at all, or only deploy digest-pinned refs produced elsewhere (recommended for determinism).
- [ ] Update migration helper scripts to stop defaulting to `:dev` and to read the digest-pinned GitOps value instead:
  - [ ] `scripts/run-migrations.sh` (defaults to `gitops/.../env/dev/...` and `ghcr.io/ameideio/migrations:dev`)
  - [ ] `scripts/run-flyway-cli.sh`

### 8) Enforcement and drift management (Argo + admission)

- [ ] Align (or retire) the existing Kyverno policies so they match Ameide namespaces and the 602/603 approach:
  - [ ] `policies/deny-latest-tag.yaml` (currently only matches namespace `ameide`)
  - [ ] `policies/require-digest-mutation.yaml` (currently placeholder image refs and namespace; mutation can also create Argo drift)
- [ ] If admission policies mutate images (digest rewrite), decide and document how Argo should treat that drift:
  - [ ] Prefer “Git is already digest-pinned” (no mutation needed) over mutation-based enforcement.
  - [ ] If mutation is kept, add narrowly-scoped `ignoreDifferences` rules in Argo config (see `sources/values/common/argocd.yaml`) to avoid constant OutOfSync.

### 9) Quick inventory commands (to confirm nothing is missed)

- [ ] Run `rg -n \"tag:\\s*(dev|main|latest)\\b\" sources/values/env -S` and eliminate or time-box every match.
- [ ] Run `rg -n \"pullPolicy:\\s*Always\" sources/values sources/charts/shared-values -S` and ensure every remaining `Always` is justified (prefer `IfNotPresent` with digests).
- [ ] Run `rg -n \"grpcurl-runner:dev\" sources -S` and pin by digest.

## Verification checklist

- Argo applications converge without manual restarts after a new image build.
- `kubectl get deploy -A -ojsonpath='{..image}{\"\\n\"}'` shows digests (or SHA tags) in GitOps-managed environments.
- Primitive CRs show `spec.imagePullPolicy` only when needed (transitional), not as a permanent crutch.

## Risks / gotchas (track explicitly)

- GHCR mirror tags must remain multi-arch for local `arm64` (see `backlog/456-ghcr-mirror.md`).
- Pull secrets must exist in every namespace that runs private images (see `backlog/450-argocd-service-issues-inventory.md`).
- Floating tags (`:dev`, `:main`) must not bleed into GitOps-managed environments once enforcement is on.
