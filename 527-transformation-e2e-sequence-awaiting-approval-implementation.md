# 527 Transformation - E2E Execution Sequence (Awaiting Approval; Implementation)

**Status:** Draft  
**Parent:** `backlog/527-transformation-e2e-sequence.md`

- Preferred implementation: Temporal Update for approval/reject with explicit `UpdateId` (idempotency key).
- Domain records the baseline decision via Governance write surfaces; domains emit facts via outbox; workflow emits a corresponding process fact (`GateDecisionRecorded`).

