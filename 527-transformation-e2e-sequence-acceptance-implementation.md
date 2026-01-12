# 527 Transformation - E2E Execution Sequence (Acceptance; Implementation)

> **DEPRECATED (2026-01-12):** Superseded by the Zeebe-based sequence.  
> See `backlog/527-transformation-e2e-sequence-v4.md`.

**Status:** Draft  
**Parent:** `backlog/527-transformation-e2e-sequence.md`

- Preview infra details must not leak into process semantics.
- Workflow can orchestrate preview preparation via Activities and then wait on the explicit acceptance decision (user task completion).
