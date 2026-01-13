---
title: 665 – Transformation Primitive v4 (Primitive Interactions)
status: draft
owners:
  - transformation
  - platform
created: 2026-01-13
---

# 665 – Transformation Primitive v4 (Primitive Interactions)

This document defines how Transformation v4 processes interact with other primitives (domain, projection, agents, integrations) without repeating the per-process details.

## Canonical ownership boundaries

- **Transformation domain primitive** is the system of record:
  - owns `transformation`, `requirement`, `deliverable`, and batch entities
  - emits facts (CloudEvents) after commit
  - consumes intents/commands from process/workers

- **Transformation process solution (Zeebe)** owns orchestration and human gates:
  - hosts BPMN definitions (4 processes)
  - hosts one worker microservice (job types)
  - waits/resumes via Zeebe messages, not broker choreography

- **Projection primitives** own read models:
  - build Kanban and timelines from domain facts (truth)
  - optionally link to Zeebe/Operate for audit trails (non-canonical)

- **Agent primitives / Integration primitives** are executors:
  - invoked via request→wait→resume pattern
  - publish completion messages (directly or via completion ingress)

## Domain Kanban model (v4)

Kanban is domain-driven (single mental model):

- Requirement states (example):
  - Drafting → Published/DoR → Ready for Delivery → Delivering → Ready for Acceptance → Accepted/Rejected → Released

- Batch states (example):
  - Delivery batch: Collecting → Executing → Final Gate → Completed/Aborted
  - Acceptance batch: Collecting → Validating → Decision → Completed
  - Release batch: Collecting → Releasing → Verifying → Completed/Failed

The exact state machine is owned by the domain; processes advance it via intents and domain emits facts for projections.

## EDA → Zeebe bridge (v4)

The Kafka→Zeebe ingress is the cross-primitive seam that makes “domain facts start/correlate processes” reliable:

- consumes CloudEvents from Kafka topics (EDA v3 posture)
- MUST publish Zeebe messages using **Publish Message** semantics (buffering) and MUST NOT use correlate-message for normal delivery.
- publishes Zeebe messages using:
  - `messageName = ce.type`
  - `messageId = ce.id`
  - `correlationKey = batch_id` or `entity_id` as defined per process
  - TTL > 0 by default (buffering)
- does not interpret business semantics beyond mapping + routing (no embedded workflow logic).

### Vendor hard edges (normative)

- Zeebe `messageId` is only buffered-dedupe defense-in-depth; the ingress MUST be replay-safe by Kafka consumer offsets and/or durable inbox dedupe keyed on CloudEvents `id`.
- Prefer Publish Message for all async delivery because correlate-message is not buffered; correlate-message is diagnostic-only.

## Completion ingress (v4)

Long-running external work (Coder, validation runners, build/publish, GitOps promotion) completes by:

- either pushing completion to the process solution completion ingress with `{work_id, outputs}`, or
- being observed by a process-solution-owned poller which then publishes the Zeebe resume message.

Completion ingress MUST be idempotent on `work_id` (or a stable completion event id) because retries are normal.

### Coder (normative)

Coder integration MUST assume **poll-first completion**:

- request handlers persist `work_id → coder_task_id`,
- a poller observes task status/logs until terminal and then publishes the resume message,
- notifications/webhooks MAY exist only to reduce polling frequency; they MUST NOT be the sole completion mechanism.

### LangGraph (normative)

LangGraph integration (analysis/extraction) MUST be treated as an external executor with its own durable execution model:

- request handlers set `work_id` and pass it through as LangGraph `thread_id`,
- LangGraph runs with persistence enabled so long-running analysis survives restarts,
- if LangGraph pauses for input (“interrupt”), the process MUST surface a Tasklist user task and resume LangGraph using the same `thread_id` after completion,
- completion resumes Zeebe via Publish Message (buffered), using the standard `work_id` correlation key.

This keeps:
- the domain as truth,
- Zeebe as orchestration,
- Kafka EDA as cross-primitive messaging,
without forcing long work inside job leases.
