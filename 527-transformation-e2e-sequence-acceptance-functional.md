# 527 Transformation - E2E Execution Sequence (Acceptance; Functional)

> **DEPRECATED (2026-01-12):** Superseded by the Zeebe-based sequence.  
> See `backlog/527-transformation-e2e-sequence-v4.md`.

**Status:** Draft  
**Parent:** `backlog/527-transformation-e2e-sequence.md`

## Intent

Optionally validate in a preview environment and capture an explicit acceptance decision.

## Rules

- This is the second of **two** allowed `userTask`s in the Transformation R2R profile (DoR + Acceptance).
- Acceptance can be modeled as its own phase or as a release-gate inside `execution`/`release` depending on the ProcessDefinition profile.
