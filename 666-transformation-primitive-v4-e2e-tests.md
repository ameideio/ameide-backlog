---
title: 666 – Transformation Primitive v4 (Test Plan)
status: draft
owners:
  - transformation
  - platform
created: 2026-01-13
---

# 666 – Transformation Primitive v4 (Test Plan)

This document defines the v4 test plan aligned to `backlog/430-unified-test-infrastructure-v2-target.md`.

## Test layers (430-aligned)

- Phase 0/1/2 (local-only):
  - schema/contract checks, unit tests, mocked integration tests
  - no Kubernetes access, no Telepresence

- Cluster smokes (separate):
  - `ameide test cluster` runs `//go:build cluster` suites
  - used to validate Zeebe runtime wiring and cross-primitive message routing

- Deployed E2E (preview):
  - `ameide test cluster` runs Playwright against preview ingress URL

## Conformance vs capability tests

- **BPMN conformance** stays generic (engine semantics):
  - message buffering + messageId idempotency
  - timer semantics (“not earlier”)
  - incident creation semantics

- **Transformation v4 capability smoke** validates the chain:
  - domain emits start facts
  - processes start/correlate via Kafka→Zeebe ingress
  - request→wait→resume works end-to-end

## Pre-cluster contract test (repo-only; mandatory until cluster is ready)

Until the dev cluster is ready, the “wire everything together” contract is a **repo-only** test suite that proves:

- message name alignment (no `com.ameide.*`; `io.ameide.*` only),
- BPMN message subscriptions use the correct correlation keys,
- generated worker registry (`internal/worker/*_gen.go`) matches the BPMN job types,
- the three v4 agents exist and match the intended kinds (langgraph / coder_task / llm_one_shot).

This contract is implemented as a Go test inside the process primitive:
- `primitives/process/transformation_v4/internal/tests/contract_repo_test.go`

Run locally (no Kubernetes, no Telepresence):
- `go test ./primitives/process/transformation_v4/internal/tests -count=1`
- `go test -tags=integration ./primitives/domain/transformation/internal/tests -count=1`

## Cluster smoke scenarios (minimum set)

1) Requirements process starts
- Publish domain fact: `transformation.requested`
- Assert: Requirements process instance exists and user tasks are created.
  - Preferred mechanism: use Orchestration Cluster REST API user task search + completion endpoints.

### Optional: Requirements analysis executor seam (LangGraph)

Add one smoke that proves the analysis request→wait→resume seam end-to-end:

- request handler starts LangGraph analysis using `thread_id = work_id` with persistence enabled,
- simulate completion via completion ingress (baseline) or via a poller when available,
- assert the process advances into the human review gate.

2) Requirements publish → Delivery batch starts
- Complete DoR gate (simulate user task completion)
- Assert domain emits `io.ameide.transformation.fact.requirement.ready_for_delivery.v1`
- Assert Kafka→Zeebe ingress starts/correlates Delivery batch instance by `delivery_batch_id`.

3) Delivery request→wait→resume loop works
- Worker executes request job(s) and sets `work_id`
- Simulate executor completion via completion ingress (deterministic smoke baseline)
- Assert process advances past wait state.

### Optional (capability-level) integration: Coder step

Add one separate cluster smoke that proves the Coder executor seam end-to-end (without turning generic conformance into an integration test):

- request handler creates a deterministic Coder Task for a known “no-op” template/profile,
- the process-solution-owned poller detects completion and publishes the resume message,
- assert the process advances and captures evidence refs/links.

4) Acceptance batch chain
- Domain emits `io.ameide.transformation.fact.deliverable.ready_for_acceptance.v1`
- Acceptance process aggregates and reaches decision gate.

5) Release batch chain
- Domain emits `io.ameide.transformation.fact.deliverable.accepted.v1`
- Release process runs request steps and records release in domain.

## Playwright E2E (preview) scope

One minimal UI scenario to prove wiring:
- user can view transformation Kanban state transitions driven by domain facts
- user can complete at least one gate (DoR or acceptance) via UI, and observe process progress

## Additional required smoke coverage (vendor hard edges)

- Verify the Kafka→Zeebe ingress uses Publish Message (buffering) for normal delivery and does not use correlate-message.
- Verify replay safety:
  - reprocessing the same domain fact (Kafka replay) must not create duplicate business actions (domain/process idempotency),
  - do not rely on Zeebe `messageId` as global dedupe (buffer-only defense-in-depth).
