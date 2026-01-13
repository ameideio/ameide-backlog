---
title: 668 – Transformation Primitive v4 (Implementation Plan v2)
status: draft
owners:
  - transformation
  - platform
created: 2026-01-13
updated: 2026-01-13
supersedes:
  - 668-transformation-primitive-v4-implementation-plan.md
---

# 668 – Transformation Primitive v4 (Implementation Plan v2)

This plan turns the v4 spec set (660–667) into an incremental, implementable delivery plan aligned to:

- `backlog/430-unified-test-infrastructure-v2-target.md` (Phase 0/1/2 are local-only; E2E is separate),
- `backlog/671-primitive-development-workflow-unified.md` (core repo produces “eligible to deploy”; Argo runs smokes),
- Camunda/Zeebe vendor constraints (short/idempotent jobs; wait states; publish-message TTL buffering).

## Definition of Done (v4)

Done means, in dev:

1. All four v4 BPMN processes are deployed to Zeebe (Requirements / Delivery / Acceptance / Release).
2. The v4 worker microservice is deployed and owns all job types used by those BPMNs.
3. Deployed-system assertions run as a primitive-owned **smoke binary** executed by ArgoCD PostSync hook jobs (same image digest as the worker).
4. `ameide primitive verify --kind process --name transformation_v4` is green (design-time + unit/integration per 430).
5. At least one end-to-end scenario is proven (domain fact → process start → user task → request→wait→resume → next domain fact), using the real Kafka→Zeebe ingress and real primitives (no manual correlation).

## Responsibility split (non-negotiable)

### Core repo (this repo)

Owns:
- BPMN definitions, worker implementation, smoke implementation.
- Unit + integration tests (430 Phase 1/2).
- `ameide primitive verify` gates (deterministic; “eligible to deploy”).

Does not own:
- Running cluster smokes as CI truth.

### GitOps repo (`ameide-gitops`)

Owns:
- Enabling the primitive in `ameide-dev` (digest-pinned image).
- Running primitive-owned smoke jobs as Argo PostSync hooks.

## Increment plan

Each increment is “done” only when:

- unit + integration tests are green locally (430),
- `ameide primitive verify` is green,
- the dev cluster deploy is healthy,
- Argo PostSync smoke job is green.

### Increment 0 — Process solution is deployable and smokable (process-only)

Goal: prove the process primitive can be deployed and exercised in Zeebe without relying on a local cluster harness.

Deliver:
- `primitives/process/transformation_v4/bpmn/process.bpmn` includes all four processes and uses vendor-honest wait states.
- `primitives/process/transformation_v4/cmd/worker` implements all job types (short/idempotent handlers).
- `primitives/process/transformation_v4/cmd/smoke` exists and runs in-cluster:
  - deploys BPMN + forms (if applicable),
  - starts each process (via message correlation),
  - drives waits via publish-message (TTL + messageId),
  - completes user tasks via Orchestration Cluster API,
  - asserts no incidents for happy paths.
- The primitive image contains both `worker` and `smoke` binaries and ships BPMN/forms as runtime resources.

### Increment 1 — Kafka→Zeebe ingress (primitive facts start/resume processes)

Goal: remove “manual publish to Zeebe” from capability flow.

Deliver:
- Kafka→Zeebe ingress consumes v4 domain/agent facts from Kafka (CloudEvents envelope per 496 v3) and publishes Zeebe messages using **Publish Message** (TTL buffering; retry/backoff on 503).
- Replay safety is owned by Kafka offsets + primitive idempotency; do not rely on Zeebe `messageId` as global dedupe.

### Increment 2 — Domain + projection (Kanban truth)

Goal: make domain canonical truth and projections sufficient to power Kanban.

Deliver:
- Transformation domain emits v4 fact types after commit (outbox → Kafka).
- Projection materializes Kanban-ready views from domain facts (cross-reference the Kanban/projection backlogs).
- Capability smoke asserts observable Kanban progression for a minimal scenario.

### Increment 3 — Agent primitive wiring (3 agent kinds)

Goal: make agent execution a first-class primitive seam used by the v4 processes.

Deliver:
- Agent primitive supports:
  - `LLM_ONE_SHOT` (synchronous API remains available; EDA is not “Kafka only”),
  - `LANGGRAPH` (durable execution + interrupts),
  - `CODER_TASK` (poll-first completion).
- Transformation and conformance include at least one example of each kind.

### Increment 4 — Capability E2E (domain + process + agent + projection)

Goal: prove the entire capability, not just the BPMN.

Deliver:
- One “Requirements → Delivery” chain runs end-to-end in dev:
  - domain fact starts Requirements,
  - Requirements hits agent, resumes via ingress,
  - DoR is completed (user task),
  - domain emits “ready for delivery”, starting Delivery batch,
  - Delivery completes one request→wait→resume loop and reaches final gate.
- The smoke job remains deterministic (no UI; no Playwright).
  UI E2E remains owned by `ameide ci e2e` (Playwright) and is not conflated with process smokes (430/671).

