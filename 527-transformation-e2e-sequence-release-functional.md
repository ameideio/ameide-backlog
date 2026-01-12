# 527 Transformation - E2E Execution Sequence (Release; Functional)

> **DEPRECATED (2026-01-12):** Superseded by the Zeebe-based sequence.  
> See `backlog/527-transformation-e2e-sequence-v4.md`.

**Status:** Draft  
**Parent:** `backlog/527-transformation-e2e-sequence.md`

## Intent

Publish artifacts (digest-pinned) and promote environments via GitOps, then close the run.

## Rules

- Release sequencing and gates are Process concerns; rollout mechanics are Infra concerns.
- No local registries: images are referenced from GHCR by immutable digest.
