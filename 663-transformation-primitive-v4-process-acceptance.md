---
title: 663 – Transformation Primitive v4 (Process C: Acceptance Batch)
status: draft
owners:
  - transformation
created: 2026-01-13
---

# 663 – Transformation Primitive v4 (Process C: Acceptance Batch)

This document specifies the **Acceptance batch** Zeebe process for Transformation v4. Global conventions are defined in `backlog/660-transformation-primitive-v4-goals-invariants.md`.

## Purpose

- Accept/reject N deliverables (batched) with automation + explicit human decision.
- Record acceptance outcomes in the Transformation domain (truth).

## Process identity

- BPMN process id: `transformation_acceptance_batch_v4`
- Process instance correlation key: `acceptance_batch_id` (batch aggregator instance)

## Start trigger (EDA → Zeebe)

Start message:
- `messageName`: `com.ameide.transformation.fact.deliverable.ready_for_acceptance.v1`
- `correlationKey`: `acceptance_batch_id`

Required variables on first start:
- `transformation_id`
- `acceptance_batch_id`
- `deliverable_id`

## Aggregation window

- Start message is consumed by the message start event; initialize:
  - `deliverables_to_accept = [deliverable_id]` (first item)
- Receives repeated `deliverable.ready_for_acceptance` facts and appends `deliverable_id` to `deliverables_to_accept`.
- Closes collection via timer boundary or human “start acceptance” action.

## Functional flow (happy path)

1) **Collect deliverables** (aggregation)

2) **Automated validation (per deliverable)**
- Service task job type: `transformation.acceptance.preview_validate.request.v1`
- Wait for completion correlated by `work_id` and branch on:
  - success completion message, or
  - failure completion message/status (receive-task/message-catch errors must be modeled explicitly).

3) **Human accept/reject**
Two supported shapes (choose one for v4):
- per-deliverable user task (multi-instance), or
- one batch user task with a list/form.

### User task contract (normative)

All human gates must be modeled as `zeebe:userTask` (Tasklist), with:
- assignment (`candidateGroups`/`candidateUsers` or expression-based assignment),
- a form reference strategy (Camunda form id or external reference),
- a test completion strategy (cluster smoke can complete tasks via the Orchestration Cluster API).

Decision variables (normative):
- `acceptance_decision` ∈ {`accept`, `reject`}
- optional: `acceptance_feedback`

4) **Write decision to domain**
- Service task job type: `transformation.acceptance.record_decision.request.v1`
- Domain emits:
  - `com.ameide.transformation.fact.deliverable.accepted.v1` (includes `release_batch_id`), or
  - `com.ameide.transformation.fact.deliverable.rework_requested.v1` (points back to delivery semantics).
