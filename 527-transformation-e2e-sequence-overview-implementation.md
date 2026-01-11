# 527 Transformation - E2E Execution Sequence (Overview; Implementation)

**Status:** Draft  
**Parent:** `backlog/527-transformation-e2e-sequence.md`  
**Related:** `backlog/527-transformation-domain.md`, `backlog/527-transformation-proto.md`, `backlog/527-transformation-integration.md`

## The delegated execution seam (always the same)

1. Workflow decides and stays deterministic.
2. Workflow calls an Activity (the only side-effect boundary).
3. Activity submits a Domain command/intent (e.g., `RequestWork`, idempotent).
4. Domain persists and emits:
   - **Facts** (audit/outcomes) via outbox to fact topics, and
   - **Execution intents** (point-to-point) to execution queue topics when delegation is required.
5. Executor consumes execution intent, runs work, then calls the owning Domain write surface to record started/outcome (idempotent); Domain emits completion facts (outbox).
6. Workflow waits for completion (Signal/Update; timers once supported), then continues.
7. Workflow emits process facts for the timeline; projection joins process facts + domain facts.

Hard rule: executors (in-cluster or external) do not publish facts; they only call the owning Domain write surface; the Domain emits facts (outbox).
