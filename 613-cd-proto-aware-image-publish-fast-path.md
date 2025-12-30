---
title: 613 – CD: Proto-Aware Image Publish Fast Path
status: active
owners:
  - platform
  - test-infra
created: 2025-12-28
updated: 2025-12-30
---

## Summary

Reduce wasted compute in `CD / Service Images` by making proto-related work (Buf push + SDK/primitive codegen) **conditional**:

- Run proto/tooling + regeneration only when proto surfaces (or generators) change
- Otherwise, build/publish selected images using already-checked-in generated sources
- Keep explicit overrides for releases / special cases (e.g. workflow dispatch input)

This is a follow-on to `backlog/610-ci-rationalization.md`.

## Context

`CD / Service Images` currently performs a number of proto/codegen steps before building images. For many commits, images change due to service code only, while proto artifacts remain unchanged and already tracked in git.

Even when image publishing is correctly scoped (diff-based matrix), always running proto steps can still add non-trivial time and queue pressure on ARC runners.

## Goals

1. Only pay proto/codegen cost when it can affect artifacts being published.
2. Keep release/tag workflows safe (ability to force proto regen and/or build-all).
3. Preserve correctness by relying on existing “regen-diff” gates in CI before merge.

## Non-goals

- Changing the proto/SDK generation contract.
- Replacing Buf/BSR usage.

## Triggering (prevent run creation on irrelevant pushes)

For **CD** workflows (publish/push/sign), prefer trigger-level filtering to avoid creating workflow runs at all:

- Use `on.push.paths` (and/or `on.push.paths-ignore`) so irrelevant pushes do not create runs.
- Keep `workflow_dispatch` and tag/release triggers unfiltered (always available).

Unlike required CI workflows, CD workflows should not be configured as required checks. This lets us use trigger-level scoping safely without risking “required check stuck Pending” behavior.

## Implementation sketch

Recommended architecture inside `CD / Service Images`:

1. Compute diffs and emit booleans:
   - `needs_proto`: proto surfaces / generators / buf config changed
   - optionally: `needs_images`: service/image inputs changed (if you already have a diff-based image matrix gate)
2. Run proto/codegen conditionally:
   - `proto` job runs only when `needs_proto` is true, or when forced via `workflow_dispatch` input.
3. Keep image publishing independent of proto unless strictly required:
   - Prefer *not* making `build_images` depend on `proto` if `proto` is optional.
   - If `build_images` must `needs: proto`, ensure it still runs when `proto` is skipped/fails by using `if: ${{ always() }}` and then enforcing correctness inside the job (fail fast if proto was required but didn’t succeed).

## Work items

- Add a `needs_proto` gate in `CD / Service Images` based on changed paths:
  - `packages/ameide_core_proto/**`, `buf.yaml`, `buf.work.yaml`
  - `plugins/**`
  - any `buf.gen.*.local.yaml` templates used for primitives
- Ensure manual dispatch and tag releases can force proto/codegen (e.g. `workflow_dispatch` input).
- Capture before/after metrics (runner occupancy + p50/p90 wall time). If existing tooling relies on GitHub’s `/actions/runs/{run_id}/timing` endpoints, treat it as best-effort because those endpoints are closing down; prefer job timestamp–based measurement (`started_at`/`completed_at`) via workflow jobs APIs.
