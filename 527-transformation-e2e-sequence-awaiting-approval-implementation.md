# 527 Transformation - E2E Execution Sequence (Awaiting Approval; Implementation)

> **DEPRECATED (2026-01-12):** Superseded by the Zeebe-based sequence.  
> See `backlog/527-transformation-e2e-sequence-v4.md`.

**Status:** Draft  
**Parent:** `backlog/527-transformation-e2e-sequence.md`

- Preferred implementation: Temporal Update for approval/reject with explicit `UpdateId` (idempotency key).
- Domain records the baseline decision via Governance write surfaces; domains emit facts via outbox; workflow emits a corresponding process fact (`GateDecisionRecorded`).
