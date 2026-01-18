# 527 Transformation — Process Primitive Specification (v4: Zeebe Request→Wait→Resume semantics)

This document supersedes `backlog/527-transformation-process-v3.md` to make the **Zeebe runtime semantics operable** for long-running steps.

## Intent

Define how the Transformation governance process is shipped as a Process primitive:

- BPMN executes on Camunda 8 / Zeebe.
- One worker microservice implements all job types for the BPMN.
- Long-running work is modeled as **explicit BPMN wait states** and resumes via **publish message** (TTL + messageId), not by holding a Zeebe job lease.

## Canonical assets

- Narrative (latest): `backlog/527-transformation-e2e-sequence-v6.md`
- Executable shape (current): `primitives/process/transformation_v4/bpmn/process.bpmn`
- Implementation (current): `primitives/process/transformation_v4/`

## Job types (request steps)

Transformation v4 uses these Zeebe job types (service tasks) and the worker MUST implement all of them:

- `transformation.requirements.agent.request.v1`
- `transformation.requirements.publish.request.v1`
- `transformation.delivery.batch.init.v1`
- `transformation.delivery.batch.collect_append.v1`
- `transformation.delivery.coder.request.v1`
- `transformation.delivery.tests.request.v1`
- `transformation.delivery.deliverable.record.request.v1`
- `transformation.delivery.complete.request.v1`
- `transformation.acceptance.batch.init.v1`
- `transformation.acceptance.batch.collect_append.v1`
- `transformation.acceptance.preview_validate.request.v1`
- `transformation.acceptance.decision.record.request.v1`
- `transformation.release.batch.init.v1`
- `transformation.release.batch.collect_append.v1`
- `transformation.release.build_publish.request.v1`
- `transformation.release.gitops_promote.request.v1`
- `transformation.release.rollout_verify.request.v1`
- `transformation.release.complete.request.v1`

Each request step MUST be followed by a BPMN wait state correlated by `work_id` (message catch / receive task).

## `work_id` contract (v4)

`work_id` is the correlation key for request→wait→resume. Under the v6 Git-backed posture:

- `work_id` SHOULD be **Domain-issued** (a stable WorkRequest/external action id), not derived from the Zeebe job key.
- `messageId = work_id` is defense-in-depth (buffer-level idempotency); end-to-end idempotency remains the responsibility of the Domain and the consumer.

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

## Primitive boundaries (EDA)

The Transformation worker is the “process solution” glue:
- it requests side effects by calling other primitives (domains/integrations/agents),
- it consumes primitive facts as at-least-once inputs (where needed),
- it does not use EDA as internal control flow; Zeebe BPMN is the control flow.

Interpret “EDA” here as the v6 integration posture (`backlog/496-eda-principles-v6.md`) and the Git-backed repository ownership posture (`backlog/694-elements-gitlab-v6.md`).
