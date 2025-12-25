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

This is “per commit” if producer CI triggers `repository_dispatch`; otherwise it runs on schedule.

### Staging + production (promotion PRs; human-gated)

- Promotion is “copy the exact digest forward”:
  - `scripts/promote-images.sh <source_env> <target_env>`
  - `.github/workflows/promote-images.yaml` (manual `workflow_dispatch`) opens the PR
- Humans approve/merge promotion PRs (branch protection) → Argo CD sync → deterministic rollout.

## Repo-Wide Checklist (implemented)

- Charts render `image: {{ required "image.ref is required" .Values.image.ref }}` (or equivalent) and set `imagePullPolicy: IfNotPresent` for every PodSpec (Deployments/Jobs/hooks/initContainers).
- Env overlays (`sources/values/env/{local,dev,staging,production}/**`) pin first-party images by digest.
- Shared values (`sources/values/_shared/**`) pin third-party images by digest (via chart `digest:` fields, or `tag: <version>@sha256:<digest>` where the chart only supports `repository`+`tag`).
- Raw-manifest surfaces do not embed floating tags; embedded `image:` / `ref:` keys are digest pinned.
- CI enforces the policy:
  - `.github/workflows/image-policy.yaml`
  - `scripts/check-image-policy.sh` + `scripts/check-image-policy.py`

## Key GitOps Touchpoints

- Local/dev auto-bump PRs: `.github/workflows/bump-local-dev-images.yaml`
- Promotion PRs: `.github/workflows/promote-images.yaml`
- Policy gate: `.github/workflows/image-policy.yaml`
- Helpers: `scripts/bump-local-dev-images.sh`, `scripts/promote-images.sh`, `scripts/check-image-policy.sh`

## Known Exceptions / Constraints

- TemporalCluster CR images (`sources/values/_shared/data/data-temporal.yaml`) are repository-only because the Temporal community operator appends `:version`. Digest pinning requires operator support for full `repo@sha256` refs (or a no-append mode).

## Developer Tasks (outside GitOps)

- Ensure the Temporal operator / TemporalCluster supports digest-pinned images (or migrate Temporal to a workload where `image.ref` is possible).
- Publish digest-pinnable images for currently-disabled developer features (so they can be GitOps-managed without floating tags):
  - devcontainer service image
  - agent runtime image for `core-platform-coder`
- Publish internal mirror images as multi-arch manifest lists where local clusters can be arm64.

## Audit Commands

- Policy check: `bash scripts/check-image-policy.sh`
- Find floating tags (should be empty): `rg -n \":(dev|main|latest)\\b\" sources/values/_shared sources/values/env -S`
