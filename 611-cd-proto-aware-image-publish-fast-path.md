---
title: 611 – CD: Proto-Aware Image Publish Fast Path
status: active
owners:
  - platform
  - test-infra
created: 2025-12-28
updated: 2025-12-28
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

## Work items

- Add a `needs_proto` gate in `CD / Service Images` based on changed paths:
  - `packages/ameide_core_proto/**`, `buf.yaml`, `buf.work.yaml`
  - `plugins/**`
  - any `buf.gen.*.local.yaml` templates used for primitives
- Ensure manual dispatch and tag releases can force proto/codegen (e.g. `workflow_dispatch` input).
- Capture before/after metrics (runner minutes + p50/p90 wall time) using the repo’s Actions-minute reporting tooling.

