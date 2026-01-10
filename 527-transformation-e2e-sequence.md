# 527 Transformation - E2E Execution Sequence (Document Set Index)

**Status:** Draft (normative intent; implementation exists for Transformation work execution)  
**Parent:** `backlog/527-transformation-capability.md`

This document is an index. The E2E execution sequence is split into a per-phase document set:

- Overview:
  - `backlog/527-transformation-e2e-sequence-01-overview-functional.md`
  - `backlog/527-transformation-e2e-sequence-02-overview-implementation.md`
- Kanban phases (recommended keys): `intake`, `triage`, `awaiting_approval`, `execution`, `acceptance`, `release`, `terminal`
  - `backlog/527-transformation-e2e-sequence-10-intake-functional.md`
  - `backlog/527-transformation-e2e-sequence-11-intake-implementation.md`
  - `backlog/527-transformation-e2e-sequence-20-triage-functional.md`
  - `backlog/527-transformation-e2e-sequence-21-triage-implementation.md`
  - `backlog/527-transformation-e2e-sequence-30-awaiting-approval-functional.md`
  - `backlog/527-transformation-e2e-sequence-31-awaiting-approval-implementation.md`
  - `backlog/527-transformation-e2e-sequence-40-execution-functional.md`
  - `backlog/527-transformation-e2e-sequence-41-execution-implementation.md`
  - `backlog/527-transformation-e2e-sequence-50-acceptance-functional.md`
  - `backlog/527-transformation-e2e-sequence-51-acceptance-implementation.md`
  - `backlog/527-transformation-e2e-sequence-60-release-functional.md`
  - `backlog/527-transformation-e2e-sequence-61-release-implementation.md`
  - `backlog/527-transformation-e2e-sequence-70-terminal-functional.md`
  - `backlog/527-transformation-e2e-sequence-71-terminal-implementation.md`

Hard rule (shared across the set): workflows progress on workflow-local state, Activity results, and explicit user task completions; domain facts are audit/projection inputs and must not be used as internal workflow step-completion control flow.

Related BPMN sketches (phase-split; WIP):

- `backlog/527-transformation-r2d-phase1-triage.bpmn`
- `backlog/527-transformation-r2d-phase2-development-workpackage.bpmn`
- `backlog/527-transformation-r2d-phase3-release-package.bpmn`
