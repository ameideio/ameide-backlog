---
title: "511 – Process Primitive Scaffolding (v3) — Implementation Plan (430-aligned)"
status: draft
owners:
  - platform
created: 2026-01-13
parent: 511-process-primitive-scaffolding-v3.md
supersedes:
  - 511-process-primitive-scaffolding-v2-implementation-plan.md
---

# 511 – Process Primitive Scaffolding (v3) — Implementation Plan (430-aligned)

This document turns `backlog/511-process-primitive-scaffolding-v3.md` and `backlog/511-process-primitive-bpmn-conformance-suite-v3.md` into a concrete repo plan.

## Goal

Deliver a repo front-door that, for a BPMN-authored Process primitive:

1) scaffolds a **process solution** (BPMN + one worker microservice implementing all job types),  
2) verifies “diagram must not lie” (profile + deployability + worker coverage),  
3) supports long-running steps via **Request → Wait → Resume**, and  
4) proves Zeebe runtime semantics via a **separate cluster smoke entrypoint**, while keeping `ameide dev inner-loop-test` strictly Phase 0/1/2 per 430.

## Non-negotiables

- Zeebe is the runtime for BPMN-authored processes (Camunda 8 / Zeebe).
- Process primitive bundles all job workers for its BPMN (one worker service per process primitive).
- Request → Wait → Resume is the default for long-running work (no long work while holding a Zeebe job lease).
- 430 test contract is enforced:
  - `ameide dev inner-loop-test` and `ameide ci test` run Phase 0/1/2 only (local-only; no cluster/Telepresence).
  - cluster smokes run via a separate CLI entrypoint (name TBD; e.g. `ameide ci smoke`).
  - Playwright E2E runs via `ameide ci e2e` against preview ingress URLs.

## Work breakdown (delivery order)

### 0) Backlogs are the contract

- Ensure `511 v3`, `527 v4/v5`, and `621 v2` are merged in backlog.

### 1) CLI: `verify` (Phase 0, design-time)

Implement/extend verification so it can answer for a Process primitive:

- BPMN profile conformance (supported constructs + required Zeebe bindings).
- Worker coverage:
  - extract all `zeebe:taskDefinition type="..."` from BPMN,
  - assert the worker service registers a handler for each type.
- Request/Wait/Resume conformance (for `*.request.*` job types):
  - a wait state exists and correlates on `work_id`.

### 2) CLI: `scaffold` (process solution)

Update scaffolding so it outputs:

- `bpmn/` directory with a Zeebe-executable BPMN skeleton (including `bpmn:documentation`).
- `cmd/worker` + `internal/worker` with a worker registry implementing all job types.
- A completion ingress that publishes `work.completed.v1` back to Zeebe (TTL + messageId).

### 3) Conformance primitive (engine semantics fixture)

Keep conformance generic (no Transformation/Coder specifics):

- A conformance BPMN that exercises message buffering/uniqueness, timers, incidents, gateways.
- A deterministic in-process worker that activates/completes/fails jobs and publishes messages.

### 4) Cluster smoke suite (separate CLI entrypoint)

Add a cluster-only smoke runner that:

- deploys the conformance BPMN,
- runs segment tests,
- asserts engine semantics via Orchestration Cluster REST API,
- emits JUnit evidence.

This runner must not be selectable by Phase 2 (`//go:build integration`).

### 5) Apply to Transformation (527)

Refactor `primitives/process/transformation_v3` to:

- use Request → Wait → Resume in BPMN,
- implement short request handlers,
- publish `work.completed.v1` on completion.

Add a Transformation-specific cluster smoke/integration test if needed (separate from conformance).

## Definition of Done

- Local front doors (`ameide dev inner-loop-test`, `ameide ci test`) are Phase 0/1/2 only and stay cluster-free.
- `verify` prevents deploying any BPMN that lacks worker coverage or violates the profile.
- `scaffold` produces a runnable Zeebe BPMN + bundled worker service, including completion ingress.
- Cluster smokes are deterministic and run via the dedicated smoke entrypoint with JUnit evidence.
- Transformation v3 is refactored to Request → Wait → Resume and validated by the same guardrails.

