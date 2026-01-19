---
title: 661 – Transformation Primitive v4 (Process A: Requirements)
status: draft
owners:
  - transformation
created: 2026-01-13
---

# 661 – Transformation Primitive v4 (Process A: Requirements)

This document specifies the **Requirements** Zeebe process for Transformation v4. Global goals/invariants and shared conventions are defined in `backlog/660-transformation-primitive-v4-goals-invariants.md`.

## Purpose

- Produce and stabilize requirement artifacts for a transformation.
- Capture explicit human gates (review + DoR).
- Publish “ready for delivery” facts through the Transformation domain.

## Process identity

- BPMN process id: `transformation_requirements_v4`
- Process instance correlation key: `requirements_batch_id` (default)

## Start trigger (EDA → Zeebe)

Start message:
- `messageName`: `io.ameide.transformation.fact.transformation.requested.v1`
- `correlationKey`: `requirements_batch_id`

Required variables on start:
- `transformation_id`
- `requirements_batch_id`
- optional: repo/target context, initial notes

## Functional flow (happy path)

1) **Request analysis** (optional, automated)
- Service task job type: `transformation.requirements.agent.request.v1`
- Output variables: `work_id`, `work_kind`

2) **Wait analysis**
- Event-based gateway waits for one of:
  - `io.ameide.agent.fact.run.completed.v1`
  - `io.ameide.agent.fact.run.needs_input.v1`
  - `io.ameide.agent.fact.run.failed.v1`
  correlated by `work_id`

Receive-task/message-catch errors must be modeled explicitly:
- either a dedicated `work.failed.v1` message, or
- a single completion message that includes a failure status and is branched on immediately.

Also include a “no response” timer boundary that routes to triage.

3) **Human review**
- User task: `transformation.requirements.review.v1`

4) **DoR gate**
- User task: `transformation.requirements.dor_gate.v1`
- Decision variables (normative):
  - `dor_decision` ∈ {`approve`, `rework`, `cancel`}
  - optional: `dor_feedback`

5) **Publish requirements to domain**
- Service task job type: `transformation.requirements.publish.request.v1`
- Worker emits intent/command to Transformation domain.

6) **Wait for domain fact (truth)**
- Message catch on `io.ameide.transformation.fact.requirements.published.v1`
- Correlation key: `requirements_batch_id`

## Analysis executor contract (LangGraph) (normative)

If Requirements uses LangGraph as the analysis executor, the process solution MUST treat it as an external long-running executor that supports pause/resume:

- **Durability:** LangGraph analysis MUST run with persistence enabled (checkpointer) so retries/restarts do not lose progress.
- **Identity mapping:** `work_id` MUST be used as the LangGraph `thread_id` (one analysis thread per Zeebe request→wait cycle).
- **Idempotency:** side effects inside analysis MUST be replay-safe (LangGraph supports “task-like” wrapping); the process solution must still assume at-least-once semantics on its own request job.
- **Human input:** humans live in **Tasklist user tasks**, not “hidden” inside LangGraph. If analysis hits an “interrupt / needs input” state, treat it as a failure/needs-input completion that routes to a Tasklist user task. On user completion, the worker resumes LangGraph using the same `thread_id` and then completes the analysis cycle (publishes the Zeebe completion message).
- **Completion delivery:** completion MUST resume Zeebe via Publish Message (buffered). Do not depend on webhook-only completion; polling is the safe default unless a push contract is explicitly guaranteed.

## Outputs (domain facts)

The domain emits:
- `io.ameide.transformation.fact.requirement.ready_for_delivery.v1` (one per requirement)
  - includes `delivery_batch_id` assignment (batching decision)
  - includes `requirement_id`, `transformation_id`

## Status (as of 2026-01-13)

Implemented in repo (not yet deployed):
- Design-time ProcessDefinition (target): `processes/transformation/v4/process.bpmn` (tenant Enterprise Repository; published by advancing `main`).
- Requirements BPMN exists in the repo fixture `primitives/process/transformation_v4/bpmn/process.bpmn` under process id `transformation_requirements_v4`.
- Worker job types are scaffolded for `transformation.requirements.agent.request.v1` and `transformation.requirements.publish.request.v1`.

Pending:
- Implement the real worker handlers (request must emit agent intent + domain command).
- Implement the pre-cluster contract test coverage (see `backlog/666-...`) as the local gate.
- Tasklist user task assignment + forms are specified but not yet wired to real UI/forms.

## Failure / rework (minimal modeled outcomes)

- DoR rejects: loop to review (or end “canceled”).
- Publish rejected by domain: go to a human triage user task (reason captured).

## User task contract (normative)

All human gates must be modeled as `zeebe:userTask` (Tasklist), with:
- assignment (`candidateGroups`/`candidateUsers` or expression-based assignment),
- a form reference strategy (Camunda form id or external reference),
- a test completion strategy (cluster smoke can complete tasks via the Orchestration Cluster API).
