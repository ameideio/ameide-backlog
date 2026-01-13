# 527 Transformation — Process Primitive Specification (v4: Zeebe long-work posture)

This document supersedes `backlog/527-transformation-process-v3.md` by making the Zeebe runtime posture explicit for **multi-minute/multi-hour** Transformation steps (agentic coding, tests, build/publish, rollout verification).

## Intent

Define the *to-be* execution meaning for the Transformation governance Process primitive:

- BPMN is executable on **Camunda 8 / Zeebe**.
- The Process primitive bundles **one worker microservice** implementing **all** Zeebe job types for the BPMN.
- **Long work is not performed while holding a Zeebe job lease by default.**
- Long work is modeled as **Request → Wait → Resume** using BPMN wait states and Zeebe messages.

## As-is (scaffold)

`primitives/process/transformation_v3/bpmn/process.bpmn` currently models large steps as direct `serviceTask` nodes (job types like `transformation.r2d.*.v1`).

That scaffold is useful for naming + ownership, but it is “short-job shaped” until we refactor the BPMN to include explicit wait states/messages for long work.

## To-be (normative pattern for each large step)

For every step that can exceed “seconds”, model it as:

1) **Request** (serviceTask; worker completes quickly)  
2) **Wait** (message catch / receive task; process instance waits)  
3) **Resume** (publish message with TTL + messageId; process instance continues)

### Work identity (correlation key)

Define and persist a `work_id` per requested unit of work:

- Default: `work_id = "<processInstanceKey>-<jobKey>"`
- Store it in Zeebe variables at request-time.

### Completion message contract (resume)

Completion resumes the process via **publish message**:

- `name`: `transformation.r2d.work.completed.v1` (process-owned message name)
- `correlationKey`: `work_id`
- `timeToLive`: `PT10M` (or similar; >0 required for buffering)
- `messageId`: `work_id` (idempotency while buffered)
- variables (payload): structured outputs (PR URL, preview URL, digests, evidence links, status, error summary)

### Step semantics (examples)

Delivery loop (agentic + CI):

- `code_change`: request agentic work (Coder/task execution), then wait for completion message with PR/commit outputs.
- `iterate_tests`: request test run, then wait for completion message with evidence + pass/fail outcome.
- `final_gate`: request final verification, then wait for completion message with “ready for acceptance” outputs.

Release:

- `build_publish`: request build/publish and capture immutable digests in completion payload.
- `gitops_promote`: request GitOps PR/merge/promotion, then wait for completion payload.
- `rollout_verify`: request rollout verification, then wait for completion payload.

## Coder posture (external execution backend, not a primitive)

Coder is treated as an external system invoked by worker handlers:

- Request handler calls Coder API to start work and records `work_handle` (Coder workspace/job ID) keyed by `work_id`.
- Completion is delivered back to Zeebe by publishing the completion message:
  - preferred: Coder callback → platform endpoint → publish message
  - fallback: poller service → publish message

No broker-mediated internal choreography is introduced “just to run Coder”.

## Primitive boundaries (496)

Worker handlers call other primitives via platform seams:

- **Commands/intents:** gRPC via the deployable Command Bus (EDA v2 posture).
- **Facts/events:** broker facts are emitted by the owning primitive after commit; workers consume facts as at-least-once inputs when needed.
- **Process progression:** owned by Zeebe BPMN execution + Zeebe messages/timers/user tasks, not by broker events.

## Verification posture

Promotion/deployment must be blocked unless:

- the BPMN is deployable to Zeebe,
- the worker microservice has complete handler coverage for all job types in the BPMN,
- and long-running steps follow the Request → Wait → Resume modeling rule (no “hold the lease for minutes” defaults).

