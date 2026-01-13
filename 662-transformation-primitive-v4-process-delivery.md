---
title: 662 – Transformation Primitive v4 (Process B: Delivery Batch)
status: draft
owners:
  - transformation
created: 2026-01-13
---

# 662 – Transformation Primitive v4 (Process B: Delivery Batch)

This document specifies the **Delivery batch** Zeebe process for Transformation v4. Global conventions are defined in `backlog/660-transformation-primitive-v4-goals-invariants.md`.

## Purpose

- Deliver one batch covering N requirements.
- Invoke external execution (Coder or other executors) via request→wait→resume.
- Produce deliverables and mark them ready for acceptance through the domain.

## Process identity

- BPMN process id: `transformation_delivery_batch_v4`
- Process instance correlation key: `delivery_batch_id` (batch aggregator instance)

## Start trigger (EDA → Zeebe)

Start message:
- `messageName`: `com.ameide.transformation.fact.requirement.ready_for_delivery.v1`
- `correlationKey`: `delivery_batch_id`

Required variables on first start:
- `transformation_id`
- `delivery_batch_id`
- `requirement_id`

## Aggregation window

Delivery uses message aggregation semantics:

- The **start message is consumed** by the message start event. Therefore the process MUST
  initialize the collection with the `requirement_id` carried on the start message before it waits for the next message:
  - `requirements_to_deliver = [requirement_id]` (first item)
- While in “collect requirements”, the process then receives repeated
  `com.ameide.transformation.fact.requirement.ready_for_delivery.v1` messages
  correlated by `delivery_batch_id` and appends `requirement_id` to `requirements_to_deliver`.
- The aggregation window closes via:
  - a timer boundary (“collection window elapsed”), or
  - an explicit human “start delivery now” user task (optional).

### BPMN template (normative)

Implement the aggregation window as an explicit loop:

1) message start event → init list with first item
2) loop:
   - receive task / message catch for “next item” (correlated by `delivery_batch_id`)
   - append item id
   - exit when:
     - interrupting boundary timer fires, or
     - human “start now” gate completes

## Functional flow (happy path)

1) **Collect requirements** (aggregation)
- Receive/message catch for ready-for-delivery facts
- Exit on timer/human action

2) **Delivery work (per requirement; multi-instance)**
- Service task job type: `transformation.delivery.coder.request.v1` (or equivalent executor request)
- Wait for completion correlated by `work_id` using one of:
  - message `work.completed.v1` (success), or
  - message `work.failed.v1` (failure), or a single completion message with `status` that is branched on immediately.
- Outputs captured: `pr_url`, `commit_sha`, evidence refs, etc.

### Coder execution contract (normative)

When Delivery uses Coder as the execution backend, the worker implementation MUST be explicit about what it creates and how completion is detected:

- **Preferred execution object:** a **Coder Task** (not “a workspace build”) so Delivery can represent long-running background work as a first-class unit.
- **Template contract:** the selected Coder workspace template MUST support Tasks (the “tasks enabled” template capability is a prerequisite).
- **Idempotency contract:** retries MUST NOT create duplicate work; encode `work_id` into the created task identity (name/metadata) and persist `work_id → coder_task_id`.
- **Completion contract (poll-first):**
  - do not assume webhook-only completion,
  - poll the Coder API until terminal, then resume via Publish Message (TTL + messageId) correlated by `work_id`.

### Multi-instance safety (normative)

Default in v4 is **sequential** multi-instance delivery to avoid shared-variable races.
Parallel multi-instance execution is allowed only when:
- the body does not mutate shared variables, and
- output mapping/collection is explicit (collect results into `deliverables[]` deterministically).

3) **Record deliverable in domain**
- Service task job type: `transformation.delivery.deliverable.record.request.v1`
- Wait for domain fact (optional): `com.ameide.transformation.fact.deliverable.recorded.v1`

4) **Run tests / validations (per deliverable)**
- Service task job type: `transformation.delivery.tests.request.v1`
- Wait for completion correlated by `work_id`

5) **Final human gate**
- User task: `transformation.delivery.final_gate.v1`
- Decision variables (normative):
  - `delivery_decision` ∈ {`approve`, `rework`, `abort`}

6) **Mark ready for acceptance (domain)**
- Service task job type: `transformation.delivery.complete.request.v1`
- Domain emits `com.ameide.transformation.fact.deliverable.ready_for_acceptance.v1`
  including `acceptance_batch_id`.

## Failure / rework (minimal modeled outcomes)

- External executor never completes: timer boundary → triage user task → retry (new `work_id`) or abort.
- Tests fail: loop back to “delivery work” for affected items.
- Worker job retries exhausted: incident is expected; triage user task path exists for “recover”.
