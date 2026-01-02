---
title: 611 – Trunk-Based `main` + GitOps Environment Promotion (dev → staging → prod)
status: active
owners:
  - platform
  - release
  - sre
created: 2025-12-27
updated: 2025-12-30
---

## Summary

Move to a **single-trunk** development model where:

- `main` is the only integration branch (all changes land via PR → `main`)
- **dev environment tracks `main` via artifacts** (digest-pinned images), not via a long-lived `dev` branch
- **staging/prod are promotions of immutable artifacts built from `main`** (build once → deploy many)
- ARC (`arc-local`) is the default CI substrate; Docker-in-Docker is not required (BuildKit in-cluster)

This eliminates duplicated “promotion PR” CI, reduces queued/cancelled runs, and aligns GitOps with environment promotion.

## Context

We currently conflate:

- source control branching (`dev`/`main`)
- deployment environments (dev/staging/prod)

This leads to waste patterns:

- `dev → main` PRs re-run heavy CI even though `dev` already enforced it
- CD workflows create runs on many pushes and later self-cancel via `should-run`, causing queue/noise
- “promotion” becomes “rebuild/retest” instead of “promote immutable digests”

Related backlogs:

- `610-ci-rationalization.md` (cost/latency controls)
- `598-github-arc.md` (ARC local runner substrate)
- `599-k8s-native-buildkit-builds-on-arc.md` (in-cluster BuildKit)
- `602-image-pull-policy.md` / `603-image-pull-policy.md` (digest pinning and rollout discipline)

## Goals

1. **Single trunk**: all feature work merges via PR → `main`.
2. **Build once, deploy many**: the same image digests move dev → staging → prod.
3. **No duplicated CI**: no “full CI twice” on branch promotions.
4. **Low-noise pipelines**: avoid creating workflow runs when paths are irrelevant.
5. **ARC-first CI**: most PR/push verification runs on `arc-local` without Docker daemon semantics.

## Non-goals

- Introducing new permanent “flag” systems for choosing runner/publish behavior per workflow.
- Making staging/prod deploy directly from branch pointers instead of immutable digests.
- Running untrusted fork PR code on self-hosted runners.

## Target Model

### Branching

- **`main` only** for integration.
- Optional: short-lived release branches only if needed for backports (policy-defined); default is tag-based releases from `main`.
- Remove “`dev` branch as environment proxy”.

### Environments and promotions (GitOps)

- Dev/staging/prod are represented by **GitOps overlays** and **digest-pinned `image.ref` values**.
- Producer CI publishes a **mainline channel tag** `ghcr.io/ameideio/<repo>:main` that always points at “latest built from `main`”; GitOps automation resolves it to digests for local/dev.
- Mainline image contract:
  - `:<repo>:main` MUST be a multi-arch manifest list (at least `linux/amd64` + `linux/arm64`) so local clusters and ARC runners don’t break by architecture.
  - `:main` is the only moving tag used for mainline automation; do not publish or depend on a second moving tag.
  - If a service needs a separate “dev-server” image (e.g., Next.js `pnpm dev`), model it as a distinct artifact (different image/repo or an explicit suffix), not a second moving tag that GitOps might accidentally consume.
- Promotion is a GitHub PR in `ameide-gitops` that copies digests forward:
  - dev → staging
  - staging → prod
- Promotion does **not** rebuild images; it reuses already-produced digests.

### CI vs CD responsibilities

**CI (verification)**
- Runs on PRs to `main` (ARC by default) with cancellation + scoped work.
- Never publishes externally by default.

**CD (publication/promotion)**
- Only runs when explicitly intended:
  - tags/releases
  - manual dispatch
  - PR-merged promotion in GitOps (digest copy)
- Avoid “always-on publish on every push”.

### Runner switching (simple)

- One variable to switch CI execution substrate per repo:
  - `AMEIDE_RUNS_ON=arc-local` (default)
  - `AMEIDE_RUNS_ON=ubuntu-latest` (fallback)
- Publishing workflows should run on a **trusted substrate**; in `ameide-gitops` we keep ARC-first and treat `arc-local` as the trusted substrate (locked down and fork-guarded), so workflows should not hardcode `ubuntu-latest`.

## Required-check policy

- Required checks should map to **PRs into `main`** only.
- Avoid policies that require a “promotion PR path” (e.g., “PRs to main must come from dev”), as that recreates duplication and complexity.

### Vendor constraints (GitHub Actions semantics)

- **Workflow skipped ⇒ required checks Pending (merge blocks)**: trigger-level filters (`paths`, `branches`) can skip an entire workflow; if its check is required, merges can block with “Pending”.
- **Job skipped via `if:` ⇒ reports Success**: job-level skips are not enforcement by themselves.
- **`needs:` + failure/skip ⇒ downstream skips unless `if: ${{ always() }}`**: use a final “gate” job (forced to run) if you need a single required check while still scoping internal work.

### `ameide-gitops` required check (recommended)

- Require only the single check `GitOps / Gate` (implemented in this repo at `.github/workflows/gitops-gate.yaml`).
- Do not require path-filtered workflows directly; keep them as implementation detail called by the gate (or as push/schedule/manual automation).

## Definition of Done

- `dev` branch is not required for the standard change path; all work merges via PR → `main`.
- `dev` environment deploys the latest approved `main` artifacts via GitOps digest pins.
- Producer publishes `ghcr.io/ameideio/<repo>:main` for every first-party image consumed by GitOps local/dev, and it is multi-arch (`linux/amd64` + `linux/arm64`).
- GitOps local/dev digest bump automation resolves `:<repo>:main` only and fails fast if `:main` is missing or not multi-arch.
- staging/prod promotions are PR-based digest copies; no rebuilds.
- CI creates minimal queued/cancelled noise (path filters + cancellation).
- ARC runners are the default for CI; BuildKit-in-cluster is the image build path.

---

## Transition Plan (Actions)

### A) Repository governance (source of truth)

1. Make `main` the only required PR target for normal development (document this in repo README/contributing).
2. Remove/relax any policies that enforce “PRs to main must come from dev” (required-check and workflow policy alignment).
3. Decide whether to keep a `dev` branch at all:
   - Preferred: keep temporarily as a compatibility branch, then archive/delete once unused.
   - If kept: it must not be an environment proxy.

### A.1) GitHub branch rules / rulesets (concrete)

Target behavior: **all code merges via PR → `main`**; “promotion” is GitOps digest-copy PRs (dev→staging→prod), not `dev → main`.

- Protect `main` (all repos that participate in CI/CD):
  - Require PRs (no direct pushes), block force-push and deletion
  - Require approvals + CODEOWNERS, require conversation resolution
  - Require status checks appropriate for PR→`main` (CI workflows only; avoid making CD/publish required unless intended)
  - Optional: require linear history (recommended) and signed commits (policy choice)
- Remove legacy constraints:
  - Remove any “source branch must be `dev`” rules/checks
  - Remove any “promotion PR `dev → main`” required check logic
- ARC safety:
  - Ensure fork PRs do not execute on self-hosted ARC runners (policy + workflow guards); keep required checks compatible with that constraint.

### B) CI restructuring (ameide repo)

1. Change CI triggers to be trunk-based:
   - Heavy CI runs on `pull_request` targeting `main` (fork-guarded).
   - `push` to `main` runs a smaller “post-merge” validation set (optional).
2. Add/strengthen path scoping inside heavyweight workflows:
   - Gate pnpm install + JS suites behind path filters.
   - Gate uv sync + Python suites behind path filters.
   - Gate operators and integration packs behind path filters.
3. Ensure aggressive cancellation:
   - `concurrency.cancel-in-progress: true` for PR workflows grouped by PR number.
4. Ensure CI is ARC-safe:
   - No Docker daemon assumptions.
   - Any image builds use BuildKit (`AMEIDE_BUILDKIT_ADDR`).

### C) CD workflow triggers (ameide repo)

1. Prevent unnecessary run creation:
   - Add `on.push.paths` filters to CD workflows so irrelevant pushes don’t create runs at all:
     - `.github/workflows/cd-service-images.yml`
     - `.github/workflows/cd-packages.yml`
     - `.github/workflows/cd-devcontainer-image.yml`
   - Note: `on.push.paths` prevents run creation; job-level `if:` only skips after the run exists (so it does not solve queue/noise).
2. Turn on branch-run cancellation:
   - For branch pushes, set `concurrency.cancel-in-progress: true` (group by workflow+branch).
   - Keep tags/releases separate if needed (no cancellation).
3. Separate “publish” from “smoke”:
   - Smoke builds may run on ARC and produce OCI/tar artifacts without pushing.
   - Real publishing/signing should run on a trusted substrate.

### D) GitOps environment model (ameide-gitops repo)

1. Define explicit environment overlays and promotion PR flow:
   - dev overlay consumes “latest main” channel (digest-pinned).
   - staging overlay consumes promoted digests from dev.
   - prod overlay consumes promoted digests from staging.
2. Standardize image references:
   - Always digest-pin `image.ref` (enforced by `602`/`603` policy).
3. Promotion scripts:
   - Ensure `scripts/promote-images.sh` copies digests forward without rebuilding.
   - Ensure promotion PRs are labeled/owned and auto-mergeable when policy allows.

### E) Argo CD + Applicationsets

1. Ensure dev/staging/prod are driven by GitOps overlays (not branch pointers).
2. Ensure Argo CD sync waves reflect “foundation → platform → workloads”.
3. Confirm rollouts happen because Git changed (promotion PR merged), not because tags float.

### F) ARC + BuildKit (local cluster)

1. Keep ARC default runner (`arc-local`) CI-friendly:
   - baseline tools in runner image (git/curl/tar/jq/yq/rg/skopeo/rsync/buildctl)
   - `AMEIDE_BUILDKIT_ADDR` injected into runner pods and set as repo variable
2. Keep BuildKit in-cluster reachable (`buildkitd.buildkit.svc.cluster.local:1234`) and restrict access appropriately for local.

### G) External integrations + release artifacts

1. Decide the publication boundary:
   - what is published on tag only vs every merge to main
   - where signing happens (hosted or trusted runner set)
2. Align SDK publishing and container publishing to the same release signal (tags on `main`).
3. Update documentation (runbooks) for “how to promote dev → staging → prod”.

### H) Measurement and guardrails

1. Update CI reporting to measure:
   - runner occupancy (sum of job `startedAt→completedAt`)
   - queue time separately (run created→job started)
   - prefer job timestamp–based measurement over GitHub’s `/actions/runs/{run_id}/timing` endpoints (which are closing down as part of billing platform changes)
2. Track cancelled/superseded runs and enforce budgets (see `610`).

---

## Research log (what must change / align)

This section records concrete deltas discovered while drafting 611, so related backlogs/docs can be adjusted coherently (no “flag proliferation”, no band-aids).

### Existing backlogs that need edits

- `backlog/610-ci-rationalization.md`: the section “E) Required-check policy for promotion PRs (`dev → main`)" conflicts with trunk-based `main` only; rewrite it to cover **PR → `main` only** and treat “promotion” as **GitOps digest-copy PRs** (dev→staging→prod) rather than branch promotions.
- `backlog/598-github-arc.md`: “Workflow onboarding” currently assumes `dev` exists as an integration path; update onboarding guidance to **PR → `main` only** and treat `dev` as an environment lane (GitOps), not a branch.
- `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md`: keep the digest-pinning model, but explicitly clarify that `:main` is **a producer channel tag** (resolved to digest by GitOps automation), not a branch/environment semantic.

### Process docs (AGENTS.md) that conflict with this direction

These docs explicitly instruct “feature PR → `dev`” then “promotion PR `dev → main`”, and even recommend merging `origin/main` into `dev` regularly. They must be updated to a trunk-only flow:

- `.external/ameide-repo/AGENTS.md`: remove the `dev` promotion workflow; document “feature PR → `main`” and “environment promotion happens in `ameide-gitops` via digest-copy PRs”.
- `.external/ameide/AGENTS.md`: same update (duplicate policy exists in a second clone).

### CI/CD workflow refactors needed in `ameide`

Current workflow logic mixes “branch == environment/channel” assumptions that are incompatible with the trunk-based model.

- `.external/ameide-repo/.github/workflows/README.md`: update trigger language that says “PRs to `dev`/`main`” and the “main == release” narrative; describe **`main` == mainline channel**, and **tags == release**.
- `.external/ameide-repo/.github/workflows/cd-packages.yml`:
  - Remove `push.branches: [dev]` once trunk-based.
  - Change `workflow_dispatch.inputs.ref` default from `dev` → `main`.
  - Delete branch-derived channel logic (`main => release`), and use `scripts/ci/compute_sdk_versions.sh` outputs (`channel`, `is_release`, `ghcr_extra_tags`) as the single source of truth.
- `.external/ameide-repo/.github/workflows/cd-packages-probe.yml`: the “channel” determination currently treats `refs/heads/main` as `release`; change this to **tag-only release** and derive tags to probe from `compute_sdk_versions.sh` outputs.
- `.external/ameide-repo/scripts/ci/verify_ghcr_aliases.sh`: currently skips verification when `BRANCH_CHANNEL=dev`; align this with tag-based releases (e.g., verify aliases when `IS_RELEASE=true` / `CHANNEL=release`), and decide explicitly whether mainline-channel aliases (like `:main`) should also be verified.
- `.external/ameide-repo/.github/workflows/cd-service-images.yml` and `.external/ameide-repo/.github/workflows/cd-devcontainer-image.yml`:
  - Remove `push.branches: [dev]` under trunk-based.
  - Add `on.push.paths` so irrelevant pushes do not create runs at all (keep `should-run` as defense-in-depth).
  - Enable cancellation for branch pushes (`concurrency.cancel-in-progress: true` for `main`), while keeping tag/release behavior separate if desired.
- `.external/ameide-repo/.github/workflows/ci-proto.yml`: currently publishes to BSR and SDKs on `refs/heads/main`; confirm whether that is intended as “dev channel publication” or whether external publication should be tag-only (if tag-only, move publish gates to `refs/tags/v*`).

### Build/CI script refactors

- `.external/ameide-repo/scripts/ci/compute_sdk_versions.sh`: already has correct “release == semver tag” semantics; remove downstream branch heuristics in workflows so this remains authoritative.
- `.external/ameide-repo/scripts/ci/report_actions_minutes.sh`: improve measurement to separate **runner occupancy** vs **queue time** by summing job `startedAt→completedAt` from `gh run view --json jobs`, and report queue as (run created→first job started). This better matches ARC capacity pressure than `run_duration_ms` alone.

---

## Implementation progress (log)

### 2025-12-27

- `ameide` (merged): PR `ameideio/ameide#428` removes `dev` branch triggers, makes release semantics tag-only, adds `on.push.paths` to CD workflows, and aligns channel tagging to `:main` (from `main`) + `vX.Y.Z` tags.
- `backlog` (this repo): updated `598`, `602`, `603`, `610` to reflect trunk-based semantics while preserving pre-trunk notes; updated `611` research log with concrete refactor targets.
- GitHub governance (done):
  - Enabled `main` branch protection for `ameideio/ameide`, `ameideio/ameide-gitops`, `ameideio/ameide-backlog` via `gh api`.
  - `ameideio/ameide`: requires 1 approval + CODEOWNERS, conversation resolution, linear history; retains the existing required status check context `Verify PR source branch` (now advisory).
  - `ameideio/ameide-gitops` + `ameideio/ameide-backlog`: requires 1 approval + CODEOWNERS, conversation resolution, linear history (no required status checks yet).

### 2025-12-30

- `ameide-gitops` (this repo): added a single always-run PR-required gate workflow `GitOps / Gate` and converted path-scoped checks to be called via `workflow_call` (no PR path-filter required-check footguns).
