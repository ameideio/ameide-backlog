# 527 Transformation — Process Primitive Specification (v2: Zeebe-executed BPMN)

This document replaces the “Temporal-backed governance workflow” posture in `backlog/527-transformation-process.md`.

## 0) Intent

Define the runtime posture for Transformation governance processes where:

- BPMN is the canonical authoring format and the executable runtime program.
- Execution is performed by **Camunda 8 / Zeebe**.
- Side effects are performed by the **Transformation Process primitive worker microservice** as Zeebe job workers (one worker service per process primitive).
- Domains remain the system of record; process engines coordinate but do not own truth.

## 1) Execution model

### 1.1 What runs where

- **Zeebe** executes the BPMN process instance state machine (tokens, waits, timers, incidents).
- **Workers** implement BPMN service tasks and other side-effect steps:
  - Agentic work steps (coding, review, drafting)
  - Tooling/external system steps (GitHub, CI, ArgoCD)
  - Domain-facing steps (command submission to domain APIs)

All worker entrypoints live in the Process primitive worker microservice; it communicates with other primitives via the EDA/RPC seams defined in `backlog/496-eda-principles.md`.

### 1.2 Human tasks

Human completion is represented as BPMN user tasks and completed via:

- Camunda Tasklist (default), or
- Ameide UI surface that bridges to Tasklist semantics (explicitly modeled and audited).

## 2) BPMN extensions (design-time contracts)

We use extensions to document and verify “what must be implemented”, not to transpile to another engine:

- Zeebe/Camunda engine bindings define runtime job types and supported execution settings.
- Ameide extensions (`ameide:*`) define:
  - required worker ownership (which primitive implements each job type),
  - required idempotency strategy (keys, dedupe scope),
  - required evidence outputs (links/artifacts/facts),
  - correlation conventions for message-based interactions where applicable.

These extensions are enforced by verify/promotion gates, not interpreted at runtime by Zeebe.

## 3) Promotion / deployment loop (design → verify → promote → deploy → run)

The deployment loop is:

1) Author BPMN (and required extensions) as design-time assets.
2) Verify:
   - profile conformance,
   - deployability to Zeebe,
   - worker coverage exists.
3) Promote the versioned definition (baseline/promotion posture).
4) Deploy the promoted BPMN to Zeebe (GitOps-driven).
5) Run process instances; collect evidence and operational telemetry.

## 4) Observability and “process facts”

Zeebe/Operate provide execution history. Ameide may still emit process facts for:

- cross-system audit joins (domain facts + worker outcomes + process stage),
- projections (Kanban/timeline),
- portability of audit data outside Camunda.

However, the process engine remains the primary execution history for BPMN processes; any “process facts” must not reintroduce hidden control flow outside Zeebe.
