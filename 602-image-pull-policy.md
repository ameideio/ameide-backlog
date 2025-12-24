# 602: Image Pull Policy + Image Reference Policy (GitOps)

**Status:** Draft (target-state policy)  
**Owner:** Platform SRE / GitOps  
**Scope:** GitOps-managed environments (including `local`)  
**Related:** `backlog/603-image-pull-policy.md`, `backlog/503-operators-helm-chart.md`, `backlog/495-ameide-operators.md`, `backlog/456-ghcr-mirror.md`, `backlog/519-gitops-fleet-policy-hardening.md`

## Problem

Kubernetes tags are **mutable pointers**. If Git says `image: foo:dev` and a new `foo:dev` is pushed:

- Git does not change → Argo CD does not necessarily trigger a rollout.
- A pod restart can pull “whatever `:dev` means now”, which is not deterministic.

We need a policy where a Git commit maps to **one exact image artifact**, with auditability and promotion that is a **Git operation**.

## Terms

- **Immutable identity:** `@sha256:<digest>` (preferred deployment reference).
- **Unique build tag:** `:<sha>` (human-friendly, still immutable if never republished).
- **Floating tag:** `:dev`, `:main`, `:latest` (allowed to exist, not deployed directly in GitOps-managed environments).
- **`imagePullPolicy`:** controls pull behavior on pod start; it does **not** create rollouts by itself.

## Policy (target state)

### Rule 1 — GitOps-managed environments deploy immutable image references

Manifests must reference images using one of:

1) `repo@sha256:<digest>` (preferred)  
2) `repo:<tag>@sha256:<digest>` (allowed for readability)

Floating tags (`:dev`, `:main`, `:latest`) are not deployed directly.

### Rule 2 — Every build publishes a unique tag and a digest

Required output per build:

- `repo:<gitsha>` (or `repo:sha-<gitsha>`)
- `repo@sha256:<digest>` (recorded by CI)

Optional:

- release tags (`vX.Y.Z`, `vX.Y.Z-rc.N`) that point at the **same digest** as the SHA tag

### Rule 3 — Promotion is “copy the same digest forward”

Promotion across stages is a Git change that moves the same digest reference forward.

- Rollback is Git revert.
- Staging/prod prove the exact artifact that passed earlier stages.

### Rule 4 — Operators + primitives follow the same immutability rules

Cluster-scoped operators and primitive runtimes are dependencies:

- pin them by digest in GitOps-managed environments
- treat floating tags as a transient dev convenience only (not the deployment reference)

## Transitional allowances (explicitly time-boxed)

If we temporarily deploy floating tags (e.g., for rapid iteration), we must still make behavior predictable:

- set explicit pull policies (`spec.imagePullPolicy: Always` for primitives; chart-specific equivalents elsewhere)
- ensure Git changes on every rollout (e.g., write back a resolved digest into Git, or bump a rollout annotation)

This is tracked and refactored under `backlog/603-image-pull-policy.md`.

## Compatible implementation patterns (choose one)

### A) Pure Git PR promotion (minimal moving parts)

1. CI builds and pushes `repo:<sha>`, records the digest.
2. CI opens a PR updating GitOps values/manifests to `repo@sha256:<digest>`.
3. Merge → Argo sync → deterministic rollout.
4. Promotion is another PR copying the same digest forward.

### B) Argo CD Image Updater with Git write-back

If we want continuous updates without humans:

- image updater follows a constrained tag set (e.g., `sha-*`)
- writes the resolved digest back to Git (Git remains source of truth)

### C) Kargo (promotion controller)

If we want a first-class “promote artifact between stages” controller:

- Kargo watches artifacts and commits new refs into the GitOps repo per stage

## Notes

- `imagePullPolicy` is a correctness *aid*, not a rollout mechanism.
- Multi-arch and pull-secrets are separate concerns; see `backlog/456-ghcr-mirror.md`.
