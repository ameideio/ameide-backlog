# 602: Image References + Pull Policy (GitOps)

**Status:** Target state (normative)  
**Owner:** Platform SRE / GitOps  
**Scope:** GitOps-managed environments (`local`, `dev`, `staging`, `production`)  
**Related:** `backlog/603-image-pull-policy.md`, `backlog/503-operators-helm-chart.md`, `backlog/495-ameide-operators.md`, `backlog/456-ghcr-mirror.md`, `backlog/519-gitops-fleet-policy-hardening.md`

## Problem

Kubernetes tags are **mutable pointers**. If Git says `image: foo:main` and a new `foo:main` is pushed:

- Git does not change → Argo CD does not necessarily trigger a rollout.
- A pod restart can pull “whatever `:main` means now”, which is not deterministic.

We need a policy where a Git commit maps to **one exact image artifact**, with auditability and promotion that is a **Git operation**.

## Terms

- **Immutable identity:** `@sha256:<digest>` (the deployment identity).
- **Readable ref (still immutable):** `:<sha>@sha256:<digest>` (allowed, still pinned by digest).
- **Floating tag:** `:main`, `:latest` (MUST NOT be referenced by GitOps).
- **Semantic version tag:** `:vX.Y.Z` (not floating; allowed only when paired with a digest).
- **`imagePullPolicy`:** a pull behavior knob; it does **not** create rollouts by itself.

## Policy (target state)

### Rule 1 — GitOps deploys digest-pinned image refs only

All images referenced by GitOps MUST be pinned by digest:

- `repo@sha256:<digest>` (preferred)
- `repo:<sha>@sha256:<digest>` (allowed for readability)

Refs without a digest are non-compliant.

GitOps MUST NOT use branch-like tags (e.g. `:main`) even when combined with a digest (`repo:main@sha256:...`). If a tag is present, it MUST be either:

- a build SHA tag (recommended), or
- a semantic version tag (for third-party images), and still paired with `@sha256:...`.

### Rule 2 — Single image configuration contract: `image.ref`

For **custom charts we control** in this GitOps repo, the only supported “knob” for images is a single string value:

- `image.ref: <repo>[@sha256:<digest> | :<sha>@sha256:<digest>]`

Do not model first-party service images as `repository`/`tag`/`digest` in GitOps values. That split invites drift and makes it easy to accidentally deploy a floating tag.

For **third-party charts** (vendored under `sources/charts/third_party/**`), follow the chart’s supported values, but the rendered image MUST still be digest-pinned. In practice, that means one of:

- chart supports a `digest:` field → set it (`sha256:<digest>`) and keep `tag:` as a plain semantic version (`vX.Y.Z`) if the chart uses the tag for labels.
- chart supports `repository` + `tag` only → encode as `tag: <version>@sha256:<digest>` only if the chart does not reuse `tag` in Kubernetes labels/annotations (otherwise it can render invalid label values like `vX.Y.Z@sha256:...`).

If a third-party chart does not support digest pinning cleanly, patch the vendored chart to add a digest value (preferred) rather than relaxing the policy.

### Rule 3 — Pull policy is boring and consistent

All pod templates MUST use (Deployments/StatefulSets/DaemonSets, Jobs/CronJobs, Helm hooks, initContainers):

- `imagePullPolicy: IfNotPresent`

`Always` MUST NOT be used as a rollout mechanism.

### Rule 4 — Fast-moving lanes: `local` + `dev`, updated by PRs

There are exactly two “fast-moving” GitOps lanes: `local` and `dev`.

- Default/preferred: CI opens PRs that update `image.ref` digests in `local` and `dev` and auto-merges them once required checks pass (no human step for `local`/`dev`).
- Allowed (dev only): a controller may write directly to the GitOps tracking branch if it is a dedicated bot identity and changes remain deterministic and auditable (see `backlog/623-kargo-gitops-promotion.md`).
- Git change → Argo auto-sync → rollout (deterministic) in both environments.

Implementation options (GitOps repo):

- **Kargo (preferred direction):** Warehouse tracks the digest behind the producer channel tag `:main`, and a `dev` Stage writes digest-pinned `image.ref` values into `sources/values/env/dev/apps/**` (see `backlog/623-kargo-gitops-promotion.md`).
- **Legacy CI PR write-back:** scheduled/dispatch workflows that resolve `ghcr.io/ameideio/<repo>:main` → digest and rewrite env values, then open/auto-merge a PR.

Trunk-based note: the producer `:main` tag represents “latest built from `main`” (a channel tag).

This is “fully automated” for local/dev once:

- producer CI pushes `:<repo>:main` channel tags, and
- the bump workflow is enabled with `GHCR_TOKEN` (read access) configured.

`local` is not an exception to the digest-only policy: it follows the same “digest-pinned refs only” rules as `staging`/`production`. The only difference is that `local` advances automatically (via auto-merged PRs).

`staging` and `production` MUST only move via promotion PRs that copy the exact same digest forward.

### Rule 5 — Operators and primitives follow the same policy

Cluster-scoped operators and primitive runtimes are dependencies:

- Operators MUST be pinned by digest in GitOps.
- Primitive CRs MUST set `spec.image` to a digest-pinned ref.
- GitOps MUST NOT rely on primitive `spec.imagePullPolicy` for rollouts in any environment.

## Build output requirements (producer side)

Every image build MUST output:

- a unique, immutable SHA tag (example: `repo:<gitsha>`)
- the resolved digest (example: `repo@sha256:<digest>`)

CI MUST write the digest into Git (`image.ref`) via PRs; Argo rollouts MUST be driven by that Git change.

## Version bumping and promotion (GitOps)

- `local` and `dev`: updated automatically via PR write-back (CI opens a PR that updates digest-pinned `image.ref` values and auto-merges once required checks pass).
- `staging` and `production`: updated only via **promotion PRs** that copy the exact same digest forward; PR approval is a human gate (branch protection), but the mechanism is still “Git change → Argo rollout”.

Implementation (this repo):

- `scripts/promote-images.sh <source_env> <target_env>` rewrites digests in the target env to match the source env.
- `.github/workflows/promote-images.yaml` (manual `workflow_dispatch`) opens a promotion PR; it is not auto-merged.

## Enforcement (required)

- CI MUST fail if any GitOps-managed values or manifests reference an image ref without `@sha256:...`.
- CI MUST fail if any floating tags (`:main`, `:latest`) are referenced by GitOps.
- There is no allowlist for floating tags in `local`.

Implementation (this repo): `.github/workflows/image-policy.yaml` runs `scripts/check-image-policy.sh` on PRs/changes in `sources/values/**`.

### Enforcement scope (paths)

The enforcement checks apply at minimum to:

- `sources/values/env/local/**`
- `sources/values/env/dev/**`
- `sources/values/env/staging/**`
- `sources/values/env/production/**`
- shared component defaults (at minimum `sources/values/_shared/**`)
- cluster-scoped operators and dependencies (at minimum `sources/values/_shared/cluster/ameide-operators.yaml`)

## Notes

- `imagePullPolicy` is a correctness *aid*, not a rollout mechanism.
- Third-party images are pinned by digest too: GitOps determinism applies equally to upstream charts (otherwise `:latest`/`vX.Y.Z` retags can change meaning without a Git diff).
- Multi-arch and pull-secrets are separate concerns; see `backlog/456-ghcr-mirror.md`.
- If you need mutable tags for an inner-loop workflow, keep it outside GitOps-managed surfaces (e.g., Tilt-only ephemeral releases that are not committed to the GitOps repo).

---

## Implementation progress (log)

### 2025-12-26

- `ameide` (producer/tooling): PR https://github.com/ameideio/ameide/pull/404 (in review) removes floating tag defaults from local build/publish + scaffolding, and adds digest/ref support to the operators Helm chart to unblock 602 in GitOps-managed environments.
- `backlog`: PR https://github.com/ameideio/ameide-backlog/pull/50 (merged) defined success criteria and expanded the refactoring checklist in 603.
