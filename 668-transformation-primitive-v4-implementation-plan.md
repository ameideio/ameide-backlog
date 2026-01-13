---
title: 668 – Transformation Primitive v4 (Implementation Plan)
status: superseded
owners:
  - transformation
  - platform
created: 2026-01-13
---

# 668 – Transformation Primitive v4 (Implementation Plan)

> **Superseded:** see `backlog/668-transformation-primitive-v4-implementation-plan-v2.md` (capability-complete: domain + process + agent + projection; upstream refactors + test success criteria).

## Status (as of 2026-01-13)

This document is kept for historical context only.

Current plan of record:
- `backlog/668-transformation-primitive-v4-implementation-plan-v2.md`

This is the execution checklist for the v4 spec set:
- `backlog/660-transformation-primitive-v4-goals-invariants.md`
- `backlog/661-664-transformation-primitive-v4-process-*.md`
- `backlog/665-transformation-primitive-v4-primitives-interactions.md`
- `backlog/666-transformation-primitive-v4-e2e-tests.md`
- `backlog/667-transformation-primitive-v4-deploy.md`

The plan is **430-aligned**:
- Phase 0/1/2 remain local-only,
- cluster semantics are validated via `ameide ci smoke`,
- UI E2E remains Playwright and runs only when needed (`ameide ci e2e`).

## Target deliverables (Definition of Done)

Done means:
1) v4 processes are deployed to Zeebe in dev and can be started/correlated via Kafka-delivered domain facts (not manual correlation),
2) one worker microservice owns all job types for the v4 BPMNs and is deployed in dev,
3) Agent primitive is deployed and can execute long-running work via EDA intents/facts (replay-safe),
4) Kafka→Zeebe ingress is deployed and replay-safe,
5) `ameide primitive verify --kind process --name transformation_v4` is green (including GitOps wiring),
6) `ameide ci smoke` is green for v4 (cluster),
7) at least one end-to-end scenario is proven from “domain fact → process instance → user task → request→wait→resume → next domain fact”.

## Execution plan (delivery order)

### 1) Lock the runtime contracts (no ambiguity)

- Confirm the v4 process contracts are final (message names, correlation keys, batch ids, work_id contract).
- Confirm the transport standard: **Kafka** is the EDA substrate; CloudEvents remain an envelope/format carried over Kafka.
- Confirm the “vendor hard edges” are enforced as invariants:
  - workers are short + idempotent (at-least-once jobs),
  - request→wait→resume is mandatory for long work,
  - Zeebe `messageId` is buffer-only dedupe; replay safety is by Kafka consumer offsets + domain/process idempotency.

### 2) Implement the v4 process solution primitive

Primitive: `primitives/process/transformation_v4/`

Required:
- BPMN with 4 distinct Zeebe processes (Requirements / Delivery / Acceptance / Release).
- User tasks must be Camunda user tasks (`zeebe:userTask`) and be completable via Orchestration Cluster API.
- Worker service must implement **all** `zeebe:taskDefinition type="..."` job types used by those BPMNs.
- Completion ingress must publish buffered Zeebe messages (Publish Message with TTL + messageId).
- For long work, worker request handlers MUST emit Agent intents and immediately complete jobs (no long work under lease).

Acceptance (local):
- `ameide primitive verify --kind process --name transformation_v4` passes all non-cluster checks.

Acceptance (cluster):
- `ameide ci smoke` (or equivalent `//go:build cluster` suite) proves each of the 4 processes can:
  - deploy to Zeebe,
  - activate/complete the request jobs,
  - create/complete user tasks,
  - resume via publish-message to explicit wait states.

### 3) Implement Kafka→Zeebe ingress (bridge)

Primitive: `primitives/integration/<name>` or a dedicated platform component (per `backlog/665-transformation-primitive-v4-primitives-interactions.md`).

Required:
- Consume Kafka facts emitted by the Transformation domain primitive.
- For each inbound fact, publish a Zeebe message using **Publish Message**:
  - `name = event.type` (or a deterministic mapping),
  - `correlationKey = <batch_id/work_id>`,
  - `messageId = event.id` (defense-in-depth only),
  - `timeToLive` chosen per message class (buffering policy).
- Replay-safe operation:
  - idempotent consumption keyed by Kafka offsets + consumer group,
  - do not rely on Zeebe messageId for global dedupe.

### 4) Implement Agent primitive (executor)

Primitive: `primitives/agent/<name>` (contract v1).

Required:
- Consume agent intents from Kafka (`io.ameide.agent.intent.*`).
- Persist an `AgentRun` state machine and emit run outcome facts after commit (`io.ameide.agent.fact.*`).
- Implement execution kinds behind the same seam:
  - LangGraph (durable; thread_id = work_id),
  - Coder Task (poll-first completion),
  - one-shot LLM (bounded).

Acceptance (cluster):
- When the process emits `io.ameide.agent.intent.run.requested.v1`, the agent emits a completion fact and the process resumes via Kafka→Zeebe ingress.

Acceptance (cluster):
- When the domain emits a start fact to Kafka, the matching Zeebe process instance appears without manual correlate-message.

### 5) Implement Transformation domain → facts and idempotency hooks

Primitive: Transformation domain (system of record).

Required:
- Emit the v4 fact types to Kafka after commit (outbox).
- Own idempotency of business actions (replay-safe) so reprocessing a fact does not duplicate state transitions.
- Maintain the canonical status model that drives Kanban/projections.

Acceptance (cluster):
- Full “domain emits → bridge publishes → process starts” chain is observable and deterministic.

### 6) Add capability-level cluster smokes (not generic conformance)

Scope: `backlog/666-transformation-primitive-v4-e2e-tests.md`

Required minimum smoke:
- Requirements: start from domain fact, create user tasks, complete DoR, publish domain intent, wait for domain “published” fact.
- Delivery: aggregate two requirements into one batch id, run one request→wait→resume loop, reach the final human gate.

Optional (later, keep separate from generic conformance):
- One “real executor” integration (Coder poll-first) proving end-to-end completion without synthetic completion messages.
- One “analysis executor” integration (LangGraph durable execution), if enabled.

### 7) GitOps deployment (dev only)

Per `backlog/667-transformation-primitive-v4-deploy.md`:
- Deploy the worker microservice via GitOps (image digest).
- Deploy the Kafka→Zeebe ingress via GitOps.
- Run CI deploy of BPMN resources to Zeebe (Orchestration Cluster REST API) as part of the deployment workflow.
- Ensure required OIDC client secrets exist in-cluster before deploy jobs run.

Acceptance:
- A clean Argo sync produces the worker + ingress running in `ameide-dev`.
- Cluster smokes are green without manual steps.

## Notes / non-goals for v4 implementation

- Do not fold UI Playwright tests into v4 process conformance; keep UI E2E focused on UI surfaces and a single minimal happy-path.
- Do not implement “long-running job under Zeebe lease” patterns; if work takes minutes, model it as request→wait→resume.
