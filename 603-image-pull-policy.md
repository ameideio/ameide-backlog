# 603: Image Pull Policy — Refactoring Plan (GitOps + Operators)

**Status:** Ready to implement  
**Owner:** Platform SRE / GitOps  
**Scope:** Implement the target state in `backlog/602-image-pull-policy.md`  
**Related:** `backlog/602-image-pull-policy.md`, `backlog/495-ameide-operators.md`, `backlog/503-operators-helm-chart.md`, `backlog/450-argocd-service-issues-inventory.md`, `backlog/519-gitops-fleet-policy-hardening.md`, `backlog/456-ghcr-mirror.md`

## Goal

Make “Git commit → deployed artifact” deterministic in GitOps-managed environments by:

- eliminating reliance on mutable tags for rollouts
- pinning by digest (keeping the SHA tag in the ref for readability is allowed)
- completely removing floating `:dev` / `:main` from GitOps-managed deployments (aliases are never referenced by GitOps)
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

**Target:** primitive runtime images are configured via a single digest-pinned image ref, and GitOps does not rely on `imagePullPolicy` for rollouts.

**DoD**
- Primitive CR manifests set `spec.image` from a single value (`image.ref`) that includes `@sha256:...`.
- Primitive CR manifests do not set `spec.imagePullPolicy` (operators should default to `IfNotPresent`).
- No `dig`/`get` usage that fails when `.Values` is not a plain map in `tpl` contexts.
- Docs updated: `backlog/495-ameide-operators.md` and `backlog/520-primitives-stack-v2-tdd.md` reference 602/603.

### WP-3: Single image contract everywhere (`image.ref`) (repo: `ameide-gitops`)

**Target:** all workloads deployed by this GitOps repo are configured via a single digest-pinned image ref.

**Approach**
- Standardize on `image.ref` (string) and render it verbatim into pod templates (or into primitive `spec.image`).
- Do not split images into `repository`/`tag`/`digest` in GitOps values.

**DoD**
- Every env overlay sets `image.ref` values that include `@sha256:...`.
- Staging/production use the exact same digest that was validated in local (promotion is copying refs).

### WP-4: Continuous update mechanism for the “fast-moving” environment

There is one fast-moving GitOps environment: `local`. It moves via Git PRs that update digests.

**DoD**
- Every successful build opens a PR updating `image.ref` in `sources/values/env/local/**`.
- Merge → Argo sync → deterministic rollout without manual restarts.
- Promotion to staging/production is a PR copying the exact same `image.ref` forward.

### WP-5: Policy enforcement (CI)

**CI enforcement (required)**
- Reject any image ref without `@sha256:...` in GitOps-managed environment directories.
- Reject any floating tag (`:dev`, `:main`, `:latest`) anywhere in GitOps-managed values/manifests.
- Require the same digest-pinning for cluster-scoped operators and primitives.

**DoD**
- CI fails fast when the policy is violated.

### WP-6: Reduce Argo drift/noise related to defaulted image fields

**Target:** Argo stays boring: no spurious OutOfSync from defaulted `imagePullPolicy`.

**DoD**
- All pod templates set `imagePullPolicy: IfNotPresent` explicitly (no reliance on Kubernetes defaulting).
- Drift guidance is documented in `backlog/519-gitops-fleet-policy-hardening.md` and referenced from 602/603.

## Codebase audit: where floating `:dev` / `:main` shows up today

This is the starting map for “what must change” if we implement 602 and want to fully remove floating `:dev`/`:main` from GitOps-managed environments.

- **GitOps env overlays:** `sources/values/env/**` contains `tag: dev`, `tag: main`, and `tag: latest` today.
- **Cluster-scoped operators:** `sources/values/_shared/cluster/ameide-operators.yaml` and `sources/values/cluster/local/ameide-operators.yaml` rely on `tag: dev` and pull-policy forcing today.
- **Smoke job runner image:** `sources/values/_shared/apps/*-smoke.yaml` uses `ghcr.io/ameideio/grpcurl-runner:dev` today.
- **Embedded raw YAML strings:** any `manifests: |-` / “raw manifest” surfaces can hide `:main`/`:dev` refs and must be audited.
- **CLI publish path:** `packages/ameide_core_cli/internal/commands/primitive_publish.go` forces `:dev` (explicitly rejects other tags).
- **Scaffolding defaults:** `packages/ameide_core_cli/internal/commands/primitive_scaffold.go` and `packages/ameide_coding_helpers/scaffold/mcp_adapter.go` generate `:dev` image references by default.
- **CI publishes aliases:** `.github/workflows/cd-service-images.yml` updates `:dev` and `:main` tags as moving pointers; GitOps must never reference them.

## Refactoring checklist: no floating `:dev` / `:main` in GitOps-managed environments

### GitOps (repo: `ameide-gitops`)

- [ ] Replace image configuration in all env overlays with `image.ref` (digest-pinned) and remove `tag:` fields across `sources/values/env/{local,staging,production}/**`.
- [ ] Remove the `env/dev` directory and references (target state is `local`/`staging`/`production` only).
- [ ] Replace all direct `:dev` / `:main` / `:latest` image strings (example: `grpcurl-runner:dev` in `_shared/apps/*-smoke.yaml`) with digest-pinned refs.
- [ ] Update any embedded raw manifest strings to use digest-pinned refs (these frequently evade enforcement).
- [ ] Refactor cluster-scoped operators to use `operators.<name>.image.ref` (digest-pinned) and set `imagePullPolicy: IfNotPresent` (no “force pulls” behavior).

### CI + automation (repo: `ameide` + GitOps update mechanism)

- [ ] Ensure every published image has an immutable identity available to automation (digest + unique tag); make the mapping easy to consume for GitOps write-back.
- [ ] Implement WP-4: CI opens PRs that update `image.ref` digests in `sources/values/env/local/**` so “new build” always produces a Git change → Argo rollout.
- [ ] If `:dev` / `:main` aliases are published, keep them out of GitOps (aliases are never referenced by GitOps).
- [ ] Update devcontainer publishing/consumers so deployments do not depend on `devcontainer:main` / `devcontainer-service:main` floating tags.

### Developer tooling defaults (repo: `ameide`)

- [ ] Update `ameide primitive publish` to support publishing non-`:dev` tags (e.g., `:<sha>` / `dev-<sha>` / `main-<sha>`) and to surface the resolved digest; remove the “only :dev is supported” restriction.
- [ ] Update scaffolding (`primitive scaffold`, coding helpers) to avoid generating committed manifests that reference floating `:dev`; prefer placeholders or digest-aware value patterns aligned to 602.

### Enforcement (CI)

- [ ] Add CI enforcement in `ameide-gitops` that rejects any `image`/`image.ref` without `@sha256:...`.
- [ ] Enforce the same rule for cluster-scoped operators and any raw manifest surfaces (embedded YAML, Jobs, hooks).
- [ ] Remove/retire any admission policies that mutate image refs (Git must be the source of truth; mutation creates Argo drift).

### Docs cleanup

- [ ] Update docs that currently teach `tag: dev` / `tag: main` (example: `operators/helm/README.md`) to show digest/SHA-tag patterns consistent with 602.
- [ ] Update runbooks that recommend “restart pods to pull the new `:dev`” so the new normal is “merge digest update PR → Argo rollout”.
## GitOps configuration checklist (end-to-end)

Use this as a “touch every place images are defined” checklist for implementing `backlog/602-image-pull-policy.md` across `ameide-gitops`.

### 1) Decide and standardize the values contract (stop “tag sprawl”)

- [ ] Standardize on `image.ref` everywhere and remove `repository`/`tag`/`digest` splits from GitOps values.
- [ ] Remove primitive global tag defaults (example: `global.ameide.primitives.imageTag` in `sources/values/base/globals.yaml` and `sources/values/env/*/globals.yaml`).
- [ ] Make pull policy explicit and consistent: set `imagePullPolicy: IfNotPresent` everywhere; remove `Always` in `sources/values/cluster/azure/globals.yaml`, `sources/values/cluster/local/ameide-operators.yaml`, and any shared app values.

### 2) Make every chart/template accept digests (operators are the outlier)

- [ ] Refactor first-party charts under `sources/charts/apps/*` to accept `image.ref` and render it verbatim.
- [ ] Refactor `sources/charts/platform/ameide-operators/**` to accept `operators.<name>.image.ref` and render it verbatim.
- [ ] Refactor `sources/charts/platform/db-migrations/**` to accept `migrations.image.ref` and render it verbatim.

### 3) Convert environment overlays to digest pinning (staging/prod first)

- [ ] Remove `sources/values/env/dev/**` and remove `env: dev` from `config/clusters/azure.yaml` (target state is no `dev` environment).
- [ ] Replace all `tag: main` / `tag: dev` / `tag: latest` usage with digest-pinned `image.ref` in:
  - [ ] `sources/values/env/local/**`
  - [ ] `sources/values/env/staging/**`
  - [ ] `sources/values/env/production/**`
- [ ] Eliminate `:latest` usage by pinning digests in local overlays:
  - [ ] `sources/values/env/local/data/data-minio.yaml`
  - [ ] `sources/values/env/local/platform/platform-backstage.yaml`
- [ ] Fix test/integration jobs that still use mutable tags:
  - [ ] `sources/values/env/staging/tests/integration/transformation.yaml` (currently `tag: dev`)
  - [ ] `sources/values/env/production/tests/integration/transformation.yaml` (currently `tag: dev`)
- [ ] Stop using `tag: dev-placeholder` in UI tilt values and use digest-pinned `image.ref`:
  - [ ] `sources/values/env/local/apps/www-ameide-tilt.yaml`

### 4) Convert primitives (CR manifests) to digest pinning

Primitives must set `spec.image` to a digest-pinned ref and must not rely on tag inheritance.

- [ ] Update all primitive value templates under `sources/values/_shared/apps/*-v0*.yaml` to set `spec.image` from `image.ref` and remove `tag:` usage (example file: `sources/values/_shared/apps/domain-transformation-v0.yaml`).
- [ ] Remove `_shared` defaults that force floating tags for primitive runtime images and require env overlays to set digest-pinned `image.ref`.
- [ ] Update primitive smoke tests to pin `grpcurl-runner` by digest (no `:dev`):
  - [ ] `sources/values/_shared/apps/*-smoke.yaml` (examples: `sources/values/_shared/apps/domain-transformation-v0-smoke.yaml`, `sources/values/_shared/apps/process-ping-v0-smoke.yaml`)

### 5) Convert cluster-scoped operators to digest pinning (no floating tags)

- [ ] Replace `tag: dev` with digest-pinned `operators.*.image.ref` for all operators in `sources/values/_shared/cluster/ameide-operators.yaml`.
- [ ] Remove “force pulls because :dev moves” behavior from `sources/values/cluster/local/ameide-operators.yaml` and set `imagePullPolicy: IfNotPresent`.

### 6) Update CI/render validation and chart tests to match digest-first policy

- [ ] Update `scripts/validate-hardened-charts.sh` to render `local staging production` and to validate `image.ref` includes `@sha256:...` everywhere.
- [ ] Update Helm unit tests in charts that hard-code `:dev` expectations:
  - [ ] `sources/charts/apps/*/tests/image_test.yaml` (examples seen under `sources/charts/apps/agents/tests/image_test.yaml`)
- [ ] Add a repo-level CI gate that fails if any non-digest image refs appear in GitOps-managed env overlays:
  - [ ] Enforce on `sources/values/env/local/**`, `sources/values/env/staging/**`, and `sources/values/env/production/**`
  - [ ] Enforce on cluster-scoped operator values (`sources/values/_shared/cluster/ameide-operators.yaml`)

### 7) Bootstrap/tooling alignment (remove “dev-tag” assumptions)

- [ ] Fix bootstrap build hooks that reference non-existent scripts and “dev-only” flows:
  - [ ] `bootstrap/bootstrap.sh` (references `${REPO_ROOT}/scripts/build-all-images-dev.sh`, which does not exist in this repo)
  - [ ] Remove bootstrap image build/push; bootstrap must only deploy digest-pinned `image.ref` from Git.
- [ ] Update db-migrations helper scripts to stop defaulting to `:dev` and to read digest-pinned `image.ref` from GitOps values:
  - [ ] `scripts/run-migrations.sh` (currently defaults to `sources/values/env/dev/data/data-db-migrations.yaml` and `ghcr.io/ameideio/migrations:dev`; target default is `sources/values/env/local/data/data-db-migrations.yaml` and `migrations.image.ref`)
  - [ ] `scripts/run-flyway-cli.sh`

### 8) Drift management (Argo)

- [ ] Avoid admission-time mutation of images; Git must already be digest-pinned.
- [ ] Keep Argo diff stable by ensuring charts render `imagePullPolicy: IfNotPresent` explicitly (no reliance on defaulting).

### 9) Quick inventory commands (to confirm nothing is missed)

- [ ] Run `rg -n \"\\b(dev|main|latest)\\b\" sources/values/env -S` and eliminate any image refs that still contain floating tags.
- [ ] Run `rg -n \"pullPolicy:\\s*Always|imagePullPolicy:\\s*Always\" sources -S` and eliminate all remaining `Always`.
- [ ] Run `rg -n \"grpcurl-runner:dev\" sources -S` and pin by digest.

## Verification checklist

- Argo applications converge without manual restarts after a new image build.
- `kubectl get deploy -A -ojsonpath='{..image}{\"\\n\"}'` shows digest-pinned refs.
- Primitive CRs set `spec.image` to a digest-pinned ref and omit `spec.imagePullPolicy`.

## Risks / gotchas (track explicitly)

- GHCR mirror tags must remain multi-arch for local `arm64` (see `backlog/456-ghcr-mirror.md`).
- Pull secrets must exist in every namespace that runs private images (see `backlog/450-argocd-service-issues-inventory.md`).
- Floating tags (`:dev`, `:main`) must not bleed into GitOps-managed environments once enforcement is on.
