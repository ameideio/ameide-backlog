# 527 Transformation - E2E Execution Sequence (Intake; Implementation)

**Status:** Draft  
**Parent:** `backlog/527-transformation-e2e-sequence.md`

- Typical posture: no Temporal workflow yet; UI + domains handle drafting.
- Domain writes are normal commands (e.g., create/edit elements and relationships); resulting domain facts are emitted via outbox for audit/projection.

