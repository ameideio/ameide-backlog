# 527 Transformation - E2E Execution Sequence (Document Set Index)

**Status:** Draft (normative intent; implementation exists for Transformation work execution)  
**Parent:** `backlog/527-transformation-capability.md`

This document is an index. The E2E execution sequence is split into a per-phase document set:

- Overview:
  - `backlog/527-transformation-e2e-sequence-overview-functional.md`
  - `backlog/527-transformation-e2e-sequence-overview-implementation.md`
- Kanban phases (recommended keys): `intake`, `triage`, `awaiting_approval`, `execution`, `acceptance`, `release`, `terminal`
  - `backlog/527-transformation-e2e-sequence-intake-functional.md`
  - `backlog/527-transformation-e2e-sequence-intake-implementation.md`
  - `backlog/527-transformation-e2e-sequence-triage-functional.md`
  - `backlog/527-transformation-e2e-sequence-triage-implementation.md`
  - `backlog/527-transformation-e2e-sequence-awaiting-approval-functional.md`
  - `backlog/527-transformation-e2e-sequence-awaiting-approval-implementation.md`
  - `backlog/527-transformation-e2e-sequence-execution-functional.md`
  - `backlog/527-transformation-e2e-sequence-execution-implementation.md`
  - `backlog/527-transformation-e2e-sequence-acceptance-functional.md`
  - `backlog/527-transformation-e2e-sequence-acceptance-implementation.md`
  - `backlog/527-transformation-e2e-sequence-release-functional.md`
  - `backlog/527-transformation-e2e-sequence-release-implementation.md`
  - `backlog/527-transformation-e2e-sequence-terminal-functional.md`
  - `backlog/527-transformation-e2e-sequence-terminal-implementation.md`

Hard rule (shared across the set): workflows progress on workflow-local state, Activity results, and explicit user task completions; domain facts are audit/projection inputs and must not be used as internal workflow step-completion control flow.

