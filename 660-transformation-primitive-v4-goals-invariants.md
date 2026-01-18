---
title: 660 – Transformation Primitive v4 (Goals + Invariants)
status: draft
owners:
  - transformation
  - platform
created: 2026-01-13
---

# 660 – Transformation Primitive v4 (Goals + Invariants)

This backlog defines the **v4 target** for the Transformation capability: four Zeebe processes chained by cross-primitive events, with the **Transformation domain** as the source of truth.

This document is the **non-repeating** foundation for:
- The four Zeebe processes:
  - `backlog/661-transformation-primitive-v4-process-requirements.md`
  - `backlog/662-transformation-primitive-v4-process-delivery.md`
  - `backlog/663-transformation-primitive-v4-process-acceptance.md`
  - `backlog/664-transformation-primitive-v4-process-release.md`
- `backlog/665-transformation-primitive-v4-primitives-interactions.md`,
- `backlog/666-transformation-primitive-v4-e2e-tests.md`,
- `backlog/667-transformation-primitive-v4-deploy.md`.

## Goals (v4)

- Replace the current “single pipeline BPMN” posture with **four distinct Zeebe processes**:
  - Requirements
  - Delivery (batch)
  - Acceptance (batch)
  - Release (batch)
- Chain processes via **EDA between primitives**:
  - the Transformation **domain primitive** emits facts,
  - a **Kafka→Zeebe ingress** publishes Zeebe messages to start/correlate process instances from Kafka-delivered facts.
- Keep BPMN **vendor-honest** for Camunda 8 / Zeebe:
  - long work modeled as explicit wait states (messages),
  - workers are short, idempotent request handlers.
- Keep the repository **430-aligned**:
  - Phase 0/1/2 are local-only (`ameide test`); cluster semantics live in `ameide test smoke`; Playwright E2E lives in `ameide test e2e`.

## Status (as of 2026-01-13)

Implemented in repo (not yet deployed):
- `primitives/process/transformation_v4/bpmn/process.bpmn` contains the **four v4 processes** and uses `io.ameide.*` message names.
- `primitives/domain/transformation` was refactored to a minimal v4 R2D command surface + v4 fact stable types.
- v4 agent scaffolds exist for all 3 agent kinds:
  - `primitives/agent/transformation-requirements-v4` (langgraph)
  - `primitives/agent/transformation-delivery-v4` (coder_task)
  - `primitives/agent/transformation-acceptance-v4` (llm_one_shot)
- Repo verification/scaffolding guardrails were tightened to keep `ameide primitive verify --mode repo` green.

Pending / blocked (cluster not ready yet):
- Kafka→Zeebe ingress implementation and deployed wiring.
- Deployed-system smokes (Argo PostSync) and cluster-backed runtime conformance.
- Kanban projection end-to-end (domain facts → projection → UI).

## Non-goals (v4)

- “Process engine as system of record”: Zeebe state is not canonical truth.
- Cross-primitive orchestration via internal event choreography (no “broker-driven control-flow” inside a single process).
- Reusing Temporal as a business orchestration engine for Transformation v4.

## Vendor invariants (Camunda/Zeebe “physics”)

These are the constraints the design must not fight:

- **Jobs are at-least-once**: workers must be idempotent; job timeouts can lead to reassignment.
- **Waits belong in BPMN**: async completion resumes via message subscriptions (receive task / message catch).
- **Correlation keys must be set before waiting**: subscriptions are not updated if the variable changes after the wait state is entered.
- **Publish Message is the default ingress** for async completion (TTL buffering + stable messageId for idempotency).

## Platform invariants (Ameide posture)

- **Transformation domain primitive is canonical truth** (statuses, batches, relationships, audit of decisions).
- **Canonical repository substrate is Git-backed** (canonical content as files; owner emits facts after Git commit/merge): `backlog/694-elements-gitlab-v6.md`.
- **EDA v4 applies between primitives**:
  - facts are emitted after commit by the owning primitive,
  - commands/intents are a command surface (not the routing spine),
  - processes do not depend on “fact choreography” as internal “step completion”.
- **Process primitives are “process solutions”**:
  - BPMN definitions deployed to Zeebe,
  - one worker microservice (per primitive) owning the job types for those BPMNs.

## v4 runtime pattern (default)

For any step that can take more than seconds, the default shape is:

1) **Request** (service task → short job handler)
- Sets a stable `work_id` and persists any external handles.
- Completes quickly (no long work under Zeebe lease).

2) **Wait** (message catch / receive task)
- Waits for a completion message correlated by `work_id`.

3) **Resume** (Publish Message)
- Completion ingress publishes a Zeebe message with:
  - `timeToLive > 0` (buffer early completion),
  - `messageId = <event_id or work_id>` (idempotency while buffered),
  - `correlationKey = work_id`.

## External executors (v4)

Transformation v4 treats systems like **Coder** (and LangGraph / runners) as **external executors**:

- The process solution only **requests** work and then **waits**.
- Completion returns via **Publish Message** (buffered) so Zeebe resumes at the explicit wait state.

### Coder completion is poll-first (normative)

Coder does not provide a universal, unambiguous “push completion” contract for tasks across all deployments.
Therefore, when a v4 process step is executed via Coder, the process solution MUST have a **poll-first** completion strategy:

- request handler records a stable `work_id` ↔ `coder_task_id` mapping,
- a poller observes Coder task status/logs until terminal, then publishes the Zeebe resume message.

Notifications/webhooks MAY be used only as an optimization to *wake up* polling; they are not the source of truth.

### LangGraph analysis is durable + resumable (normative)

When v4 uses LangGraph for analysis, the executor contract MUST be durable and resumable:

- analysis runs with persistence enabled (checkpointer),
- `work_id` is used as the LangGraph `thread_id`,
- “needs input / interrupt” is surfaced to BPMN as an explicit outcome and resolved via Tasklist user tasks, then resumed using the same `thread_id`.

### Vendor hard edge: `messageId` is not global dedupe

Zeebe `messageId` only protects against duplicates **while a message is buffered** (defense-in-depth).
End-to-end idempotency MUST be guaranteed by:
- the Kafka consumer (offset-based replay safety) for domain facts, and/or
- the owning primitive (domain/process) recording completion/decisions idempotently.

Do not treat Zeebe `messageId` as the canonical replay/idempotency mechanism across long horizons.

## Canonical identity model (v4)

The following identifiers are treated as stable business keys:

- `transformation_id` (root capability entity)
- `requirements_batch_id` (batch/aggregation for requirements)
- `delivery_batch_id` (batch/aggregation for delivery)
- `acceptance_batch_id` (batch/aggregation for acceptance)
- `release_batch_id` (batch/aggregation for release)
- `requirement_id` (domain entity)
- `deliverable_id` (domain entity)
- `work_id` (external async action id; correlation key for request→wait→resume)

## Canonical event→message mapping (v4)

Cross-primitive chaining uses CloudEvents (EDA) delivered via Kafka and a Kafka→Zeebe ingress (bridge or connectors).

Default mapping:

- `messageName` = CloudEvent `type`
- `messageId` = CloudEvent `id`
- `correlationKey` = the relevant batch id or entity id (defined per process)
- `timeToLive` is explicit per message class (avoid “ghost buffering”):
  - domain fact → process start/correlation: long TTL (buffers deployment lag / transient outages),
  - executor completion: short TTL (buffers only “arrived before wait state” races),
  - control/closing messages: TTL=0 unless buffering is explicitly desired.
- Zeebe message variables = CloudEvent `data`

Per-process definitions (message types and correlation keys) are defined in the per-process docs.
