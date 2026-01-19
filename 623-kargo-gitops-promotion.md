---
title: 623 – Kargo for GitOps Image Promotion (dev → staging → prod)
status: proposed
owners:
  - platform
  - sre
  - release
created: 2026-01-19
updated: 2026-01-19
---

## Summary

Adopt **Kargo** as the standard controller for **artifact-based environment promotion** in GitOps:

- Track **first-party images** published to GHCR from `main` (channel tag `:main`).
- Update GitOps env overlays by writing **digest-pinned** refs.
- Promote immutable digests **dev → staging → prod** as an orchestrated, auditable Git operation.

This replaces bespoke “bump digests” and “promote images” scripts/workflows with a purpose-built promotion layer that complements Argo CD.

## Context and prior art (older docs)

This backlog item is a continuation and consolidation of several existing documents:

- Trunk + artifact-based promotion: `backlog/611-trunk-based-main-and-gitops-environment-promotion.md`
- Digest pinning contract and enforcement: `backlog/602-image-pull-policy.md`, `backlog/603-image-pull-policy.md`
- GitOps fleet hardening / values contracts: `backlog/519-gitops-fleet-policy-hardening.md`
- Argo CD hierarchy and orchestration note (“consider Kargo”): `backlog/300-400/364-argo-configuration-v2.md`
- Registry/multi-arch expectations and pull secret posture: `backlog/456-ghcr-mirror.md`
- Historical operational lessons: `backlog/450-argocd-service-issues-inventory.md`

## Problem

Our GitOps promotion model is correct in principle (digest pinning + immutable promotions), but today it is implemented via **repo-specific glue**:

- scripts that resolve `:main` tags to digests and rewrite env values
- scripts that copy digests between env directories
- GitHub workflows that open/merge PRs based on schedules and/or dispatch hooks

This has recurring costs:

- **Duplication of logic** (discover images, resolve digests, map to values keys, update files, write commit/PR)
- **Hard-to-reason orchestration** when multiple services move together (race conditions across PRs/workflows)
- **Policy drift** risk (different code paths for local/dev vs staging/prod)
- **Operational burden** when we want to add:
  - grouped releases (multiple services promoted together)
  - environment-specific controls (auto in dev, human gate in prod)
  - verification gates (smoke checks) as first-class steps

## Goals

1. **Keep the existing GitOps contract**:
   - Env overlays remain the desired state.
   - Deployments remain digest-pinned (see `backlog/602-image-pull-policy.md`).
2. **Standardize promotion orchestration** using Kargo primitives:
   - Warehouse → Freight → Stage → Promotion.
3. **Dev auto-advances first-party images** with minimal bespoke glue.
4. **Staging and prod move by explicit promotion** of the exact same digests that ran in earlier stages (“build once, deploy many”).
5. **First-party only**: Kargo automation targets only images we build (no third-party image automation).

## Non-goals

- Replace Argo CD, ApplicationSets, or the existing environment directory/values layering model.
- Introduce local registries or any workflow incompatible with the GHCR digest policy.
- Automatically “track latest” in staging/prod.
- Manage third-party chart upgrades (those remain explicit dependency bumps).

## Why Kargo (rationale)

Kargo provides an opinionated model for what we already do informally:

- Observe upstream “artifacts” (images, git commits, charts) via Warehouses.
- Collect artifacts into Freight.
- Promote Freight across ordered Stages.
- Encode promotions as repeatable PromotionTasks/steps (git clone, yaml update, commit, PR, wait).

This aligns with our target state in `backlog/611-trunk-based-main-and-gitops-environment-promotion.md`:

- `main` is the only integration branch.
- `dev` tracks `main` via artifacts (digests), not via a `dev` branch.
- staging/prod are promotions of immutable digests.

## Proposed architecture (Ameide mapping)

### Kargo runtime placement

- Install Kargo into the shared cluster (GitOps-managed), alongside Argo CD.
- Use Argo CD to reconcile Kargo itself and all Kargo CRDs/manifests (single-writer model).

### Scope: what Kargo will manage

Kargo will manage **only first-party image refs** in the GitOps repo:

- Any `ghcr.io/ameideio/<repo>` refs that correspond to images built by Ameide CI.
- Only values keys that are part of the “first-party contract” (typically `*.image.ref` or `image.ref`).

Third-party images remain pinned by explicit dependency bumps and are out of scope.

### Warehouse model

We will use one of these patterns (decide in Phase 0):

**Option A (preferred): Grouped Warehouse**
- A single Warehouse subscribes to multiple image repositories.
- Freight represents a coherent “set” of first-party digests (a release unit).
- Best for “promote everything together” posture.

**Option B: Multiple Warehouses**
- One Warehouse per service/repo image.
- Stages can promote subsets independently.
- Best when services should move independently, but increases config surface.

### Stage model

Stages correspond to GitOps environments:

- `dev` stage: auto-promotion on new Freight (safe lane).
- `staging` stage: promotion by explicit trigger (human gate).
- `production` stage: promotion by explicit trigger (human gate + stricter verification).

### Git write-back model

Kargo promotions update the `ameide-gitops` repository by:

1. cloning the GitOps repo
2. updating env overlay values files (YAML updates)
3. committing and either:
   - pushing directly to `main` (for dev auto-advance), or
   - pushing to a branch and opening a PR targeting `main` (for staging/prod)

This preserves the “Git change → Argo CD sync → rollout” contract.

### Digest pinning behavior

Kargo promotions MUST write digest-pinned refs into Git, consistent with `backlog/602-image-pull-policy.md`:

- Preferred: `ghcr.io/ameideio/<repo>@sha256:<digest>`
- Allowed for readability: `ghcr.io/ameideio/<repo>:<sha>@sha256:<digest>`

The Warehouse may observe the mutable `:main` tag as an input, but the Git write-back is always a digest-pinned ref.

### Verification model (incremental)

Phase 1 can treat Argo CD health/sync as the “verification signal” (status-based).
Later phases can add explicit verification steps per Stage, such as:

- Argo Rollouts AnalysisTemplates for smoke checks
- Kargo verification providers (as available in our stack)

## Implementation plan

### Phase 0 — Design decisions and inventory (1–2 days)

- Decide Warehouse pattern (Grouped vs Multiple).
- Define the authoritative inventory of “first-party images we build”:
  - input list (repo/image names) and
  - mapping to GitOps value keys (e.g. `image.ref`, `migrations.image.ref`).
- Define environment-specific write-back behavior:
  - `dev`: direct commit to `main` vs PR + auto-merge
  - `staging`/`production`: PRs with required human approval/merge
- Confirm required secrets exist (or will be materialized):
  - Git provider token (GitHub) with least privilege for branch/PR operations
  - Registry read auth for GHCR to resolve tags/digests (can reuse existing pull secret patterns)

Deliverable: a short “Kargo config contract” doc fragment and the initial CRD skeletons.

### Phase 1 — Install Kargo (GitOps-managed) (1–2 days)

- Add Kargo installation to the GitOps repo as a cluster-scoped component.
- Add required namespaces/RBAC and secrets materialization via ESO.
- Confirm Kargo CRDs, controller health, and UI access (if enabled) are stable under Argo reconciliation.

Deliverable: Kargo running in-cluster, reconciled by Argo CD.

### Phase 2 — Dev auto-advance for a single service (1 day)

- Create a Kargo Project for Ameide GitOps promotions.
- Create:
  - Warehouse tracking one first-party image repo (tag `main`)
  - `dev` Stage consuming Freight from that Warehouse
  - PromotionTask that updates exactly one `sources/values/env/dev/**` file and commits changes
- Validate:
  - New `:main` push → new Freight detected → dev promotion executed
  - GitOps repo updated with digest-pinned ref
  - Argo CD sync rolls out expected workload

Deliverable: one end-to-end dev auto-advance loop.

### Phase 3 — Expand dev to all first-party images (1–2 days)

- Extend Warehouse subscriptions and YAML update mappings to all first-party images.
- Prefer grouping rules that avoid partial upgrades (e.g., promote a coherent set of images).
- Document how to add a new first-party image repo to the system.

Deliverable: dev auto-advances *all* first-party images.

### Phase 4 — Promotions to staging and prod (2–4 days)

- Add `staging` and `production` Stages.
- Implement PromotionTasks that:
  - copy exact digests forward (dev → staging → prod) by updating env overlay values
  - open PRs for staging/prod promotions (and optionally wait for merge)
- Add clear guardrails:
  - staging/prod promotion must never “re-resolve” `:main` (only promote known digests)
  - verification hooks (at minimum: Argo app health + smoke suites if present)

Deliverable: end-to-end promotion pipeline managed by Kargo.

### Phase 5 — Deprecate bespoke automation (1 day)

- Deprecate or disable GitHub workflows/scripts that Kargo replaces:
  - dev digest bump workflows
  - promote-image workflows/scripts
- Keep `local` developer workflows out of scope (local clusters won’t run Kargo).

Deliverable: one promotion system of record per environment.

## Rollback plan

- Disable auto-promotion in dev Stage.
- Remove/disable Kargo Stages and revert to existing GitHub Action–driven digest bump + promotion PR workflows.
- Since Git remains the source of truth, rollback is “stop producing commits” and/or revert offending digests.

## Open questions

- Grouping: do we promote all first-party images as one unit, or keep groups (platform vs primitives vs agents)?
- Verification: what minimal smoke gates are required before staging/prod promotions are allowed?
- Git branching: do we introduce stage branches (e.g., `stage/staging`) or keep env overlays on `main` only?
- Credentials: use GitHub App vs PAT; define minimum scopes and rotation policy.

