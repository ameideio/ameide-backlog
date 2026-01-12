# 527 Transformation - E2E Execution Sequence (Awaiting Approval; Functional)

> **DEPRECATED (2026-01-12):** Superseded by the Zeebe-based sequence.  
> See `backlog/527-transformation-e2e-sequence-v4.md`.

**Status:** Draft  
**Parent:** `backlog/527-transformation-e2e-sequence.md`

## Intent

Collect the DoR (baseline/definition readiness) decision as an explicit human completion.

## Rules

- This is the first of **two** allowed `userTask`s in the Transformation R2R profile (DoR + Acceptance).
- The workflow MUST block only on explicit completion (user task completion), not on broker facts.
- Decision outcomes MUST be explainable and leave durable evidence references.
