# 527 Transformation — Process Primitive Specification (v4: Zeebe Request→Wait→Resume semantics)

This document supersedes `backlog/527-transformation-process-v3.md` to make the **Zeebe runtime semantics operable** for long-running steps.

## Intent

Define how the Transformation governance process is shipped as a Process primitive:

- BPMN executes on Camunda 8 / Zeebe.
- One worker microservice implements all job types for the BPMN.
- Long-running work is modeled as **explicit BPMN wait states** and resumes via **publish message** (TTL + messageId), not by holding a Zeebe job lease.

## Canonical assets

- Narrative: `backlog/527-transformation-e2e-sequence-v5.md`
- Executable shape: `backlog/527-transformation-e2e-sequence-v5.bpmn`
- Implementation: `primitives/process/transformation_v3/`

## Job types (request steps)

Transformation v4 renames job types so the contract is explicit:

- `transformation.r2d.delivery_loop.code_change.request.v1`
- `transformation.r2d.delivery_loop.tests.request.v1`
- `transformation.r2d.delivery_loop.final_gate.request.v1`
- `transformation.r2d.acceptance.preview_validate.request.v1`
- `transformation.r2d.release.build_publish.request.v1`
- `transformation.r2d.release.gitops_promote.request.v1`
- `transformation.r2d.release.rollout_verify.request.v1`

Each request step MUST be followed by a BPMN wait state correlated by `work_id` (message catch / receive task).

## Completion message contract

- Message name: `work.completed.v1`
- Correlation key: `work_id` (must be set before entering the wait state).
- Publisher: the Transformation worker service (callback endpoint) or a dedicated completion bridge.
- Publish semantics (runtime reliability):
  - `timeToLive > 0` to buffer “completion arrives early” races.
  - `messageId = work_id` for idempotency while buffered.

## Human tasks

Human decisions remain explicit BPMN user tasks:
- requirement authoring + DoR gate
- accept/reject (with feedback)

## Primitive boundaries (EDA v2)

The Transformation worker is the “process solution” glue:
- it requests side effects by calling other primitives (domains/integrations/agents),
- it consumes primitive facts as at-least-once inputs (where needed),
- it does not use EDA as internal control flow; Zeebe BPMN is the control flow.

