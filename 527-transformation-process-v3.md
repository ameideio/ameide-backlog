# 527 Transformation — Process Primitive Specification (v3: Zeebe “process solution” packaging)

This document supplements `backlog/527-transformation-process-v2.md` with **implementation-facing** details now that we are scaffolding the process as a Zeebe “process solution”.

## Intent

Define how the Transformation governance process is shipped as a Process primitive:

- BPMN is executable on **Camunda 8 / Zeebe**.
- The Process primitive bundles **one worker microservice** implementing **all** Zeebe job types for the BPMN.
- The worker performs side effects by calling other primitives, following EDA principles.

## Canonical assets

- **Design-time narrative (Zeebe posture):** `backlog/527-transformation-e2e-sequence-v4.md`
- **Executable BPMN shape (Zeebe bindings):** `backlog/527-transformation-e2e-sequence-v4.bpmn`
- **Implementation (scaffolded process primitive):** `primitives/process/transformation_v3/`

## Job types (worker coverage contract)

The Transformation process uses these Zeebe job types (service tasks):

- `transformation.r2d.delivery_loop.code_change.v1`
- `transformation.r2d.delivery_loop.iterate_tests.v1`
- `transformation.r2d.delivery_loop.final_gate.v1`
- `transformation.r2d.acceptance.preview_validate.v1`
- `transformation.r2d.release.build_publish.v1`
- `transformation.r2d.release.gitops_promote.v1`
- `transformation.r2d.release.rollout_verify.v1`

The Process primitive worker MUST register a handler for each of the above.

## Human tasks

The process includes explicit BPMN user tasks for human gates:

- `transformation.r2d.requirements.develop_requirement.v1`
- `transformation.r2d.requirements.dor_gate.v1`
- `transformation.r2d.acceptance.accept_reject_gate.v1`

These are completed via Tasklist (or Ameide UI bridged to Tasklist semantics).

## Primitive boundaries (496)

Worker handlers must follow these seams:

- **Commands/intents:** gRPC via the deployable Command Bus (EDA v2 posture).
- **Facts/events:** broker facts are emitted by the owning primitive after commit; workers consume facts as at-least-once inputs.
- **No internal control-flow via EDA:** progression is owned by Zeebe BPMN execution, not by broker event choreography.

## Verification posture

Promotion/deployment must be blocked unless:

- the BPMN is deployable to Zeebe, and
- the worker microservice has complete handler coverage for all job types in the BPMN.

