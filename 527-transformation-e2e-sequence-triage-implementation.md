# 527 Transformation - E2E Execution Sequence (Triage; Implementation)

> **DEPRECATED (2026-01-12):** Superseded by the Zeebe-based sequence.  
> See `backlog/527-transformation-e2e-sequence-v4.md`.

**Status:** Draft  
**Parent:** `backlog/527-transformation-e2e-sequence.md`

- Temporal is optional until a durable run/timeline is needed.
- If a run exists, entering triage is recorded as a process fact (`PhaseEntered(triage)`), emitted idempotently via an Activity.
