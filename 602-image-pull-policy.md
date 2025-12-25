# 602: Image References + Pull Policy (GitOps)

**Status:** Target state (normative)  
**Owner:** Platform SRE / GitOps  
**Scope:** GitOps-managed environments (`local`, `staging`, `production`)  
**Related:** `backlog/603-image-pull-policy.md`, `backlog/503-operators-helm-chart.md`, `backlog/495-ameide-operators.md`, `backlog/456-ghcr-mirror.md`, `backlog/519-gitops-fleet-policy-hardening.md`

## Problem

Kubernetes tags are **mutable pointers**. If Git says `image: foo:dev` and a new `foo:dev` is pushed:

- Git does not change → Argo CD does not necessarily trigger a rollout.
- A pod restart can pull “whatever `:dev` means now”, which is not deterministic.

We need a policy where a Git commit maps to **one exact image artifact**, with auditability and promotion that is a **Git operation**.

## Terms

- **Immutable identity:** `@sha256:<digest>` (the deployment identity).
- **Readable ref (still immutable):** `:<sha>@sha256:<digest>` (allowed, still pinned by digest).
- **Floating tag:** `:dev`, `:main`, `:latest` (MUST NOT be referenced by GitOps).
- **`imagePullPolicy`:** a pull behavior knob; it does **not** create rollouts by itself.

## Policy (target state)

### Rule 1 — GitOps deploys digest-pinned image refs only

All images referenced by GitOps MUST be pinned by digest:

- `repo@sha256:<digest>` (preferred)
- `repo:<sha>@sha256:<digest>` (allowed for readability)

Refs without a digest are non-compliant.

### Rule 2 — Single image configuration contract: `image.ref`

In this GitOps repo, the only supported “knob” for images is a single string value:

- `image.ref: <repo>[@sha256:<digest> | :<sha>@sha256:<digest>]`

Do not model images as `repository`/`tag`/`digest` in GitOps values. That split invites drift and makes it easy to accidentally deploy a floating tag.

### Rule 3 — Pull policy is boring and consistent

All pod templates MUST use (Deployments/StatefulSets/DaemonSets, Jobs/CronJobs, Helm hooks, initContainers):

- `imagePullPolicy: IfNotPresent`

`Always` MUST NOT be used as a rollout mechanism.

### Rule 4 — One fast-moving lane: `local`, updated by PRs

There is exactly one “fast-moving” GitOps lane: `local`.

- CI MUST open PRs that update `image.ref` digests in `local` and MUST auto-merge them once required checks pass (no human step for `local`).
- Merge → Argo auto-sync → rollout (deterministic).

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

## Enforcement (required)

- CI MUST fail if any GitOps-managed values or manifests reference an image ref without `@sha256:...`.
- CI MUST fail if any floating tags (`:dev`, `:main`, `:latest`) are referenced by GitOps.
- There is no allowlist for floating tags in `local`.

### Enforcement scope (paths)

The enforcement checks apply at minimum to:

- `sources/values/env/local/**`
- `sources/values/env/staging/**`
- `sources/values/env/production/**`
- cluster-scoped operators and dependencies (at minimum `sources/values/_shared/cluster/ameide-operators.yaml`)

## Notes

- `imagePullPolicy` is a correctness *aid*, not a rollout mechanism.
- Multi-arch and pull-secrets are separate concerns; see `backlog/456-ghcr-mirror.md`.
- If you need mutable tags for an inner-loop workflow, keep it outside GitOps-managed surfaces (e.g., Tilt-only ephemeral releases that are not committed to the GitOps repo).
