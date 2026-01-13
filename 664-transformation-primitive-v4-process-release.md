---
title: 664 – Transformation Primitive v4 (Process D: Release Batch)
status: draft
owners:
  - transformation
created: 2026-01-13
---

# 664 – Transformation Primitive v4 (Process D: Release Batch)

This document specifies the **Release batch** Zeebe process for Transformation v4. Global conventions are defined in `backlog/660-transformation-primitive-v4-goals-invariants.md`.

## Purpose

- Release N accepted deliverables in a batch.
- Execute build/publish, GitOps promote, rollout verify as request→wait→resume steps.
- Record “released” truth in the Transformation domain.

## Process identity

- BPMN process id: `transformation_release_batch_v4`
- Process instance correlation key: `release_batch_id` (batch aggregator instance)

## Start trigger (EDA → Zeebe)

Start message:
- `messageName`: `io.ameide.transformation.fact.deliverable.accepted.v1`
- `correlationKey`: `release_batch_id`

Required variables on first start:
- `transformation_id`
- `release_batch_id`
- `deliverable_id`

## Aggregation window

- Start message is consumed by the message start event; initialize:
  - `deliverables_to_release = [deliverable_id]` (first item)
- Receives repeated `deliverable.accepted` facts and appends `deliverable_id` to `deliverables_to_release`.
- Closes collection via timer boundary or human “start release” action.

## Functional flow (happy path)

1) **Collect deliverables** (aggregation)

2) **Build/publish (batch)**
- Service task job type: `transformation.release.build_publish.request.v1`
- Wait for completion correlated by `work_id`
- Output must include pinned digests (so release is immutable).

### Failure modeling (normative)

Receive-task/message-catch waits must model failure explicitly:
- either a dedicated `work.failed.v1` message, or
- a single completion message that includes a failure status and is branched on immediately.

Every long-running wait must also have a “no response” timer boundary that routes to triage.

3) **GitOps promote (batch)**
- Service task job type: `transformation.release.gitops_promote.request.v1`
- Wait for completion correlated by `work_id`

4) **Rollout verify (batch)**
- Service task job type: `transformation.release.rollout_verify.request.v1`
- Wait for completion correlated by `work_id`

5) **Optional human release gate**
- User task: `transformation.release.approval.v1` (optional)

6) **Mark released in domain**
- Service task job type: `transformation.release.record_released.request.v1`
- Domain emits `io.ameide.transformation.fact.release.batch.ready.v1` (v4 minimal path for chaining)
- (future) `io.ameide.transformation.fact.deliverable.released.v1` / `io.ameide.transformation.fact.transformation.completed.v1`

## Status (as of 2026-01-13)

Implemented in repo (not yet deployed):
- Release BPMN exists in `primitives/process/transformation_v4/bpmn/process.bpmn` under process id `transformation_release_batch_v4`.
- Aggregation is implemented in BPMN as:
  - `transformation.release.batch.init.v1`
  - repeated message catch on `io.ameide.transformation.fact.deliverable.accepted.v1` (correlated by `release_batch_id`)
  - `transformation.release.batch.collect_append.v1`

Pending:
- Implement the real job handlers (build/publish, gitops promote, rollout verify, complete).
- Decide final “released” domain fact set once release pipeline is implemented.
