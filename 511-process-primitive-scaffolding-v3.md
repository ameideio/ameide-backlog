---
title: "511 – Process Primitive Scaffolding (v3): Zeebe long-work pattern (request → wait → resume)"
status: draft
owners:
  - platform
created: 2026-01-13
---

# 511 – Process Primitive Scaffolding (v3): Zeebe long-work pattern (request → wait → resume)

This document supersedes `backlog/511-process-primitive-scaffolding-v2.md` by tightening the **runtime meaning** of BPMN service tasks for **multi-minute / multi-hour** side effects (agentic coding, CI runs, builds, rollout verification).

Everything else from v2 remains true:

- BPMN-authored processes execute on **Camunda 8 / Zeebe**.
- A Process primitive ships as a **process solution**: **one executable BPMN + one worker microservice** that owns all job types.
- EDA v2 remains **between primitives** (facts on broker; commands/intents via gRPC Command Bus).

## The Zeebe-aligned default rule

If a step can exceed “seconds”, it MUST be modeled as:

1) **Request** (short Zeebe job) → 2) **Wait** (BPMN wait state) → 3) **Resume** (publish message)

This avoids holding a job lease while executing long work and embraces Zeebe’s at-least-once job handling.

## Pattern: Request → Wait → Resume (normative)

### A) Request (serviceTask, short, idempotent)

**BPMN:** `bpmn:serviceTask` with `zeebe:taskDefinition type="...request..."`.

**Worker handler responsibilities:**

1. Derive a stable `work_id` (correlation key).
   - Default: `work_id = "<processInstanceKey>-<jobKey>"` (or another deterministic scheme).
2. Start external work **durably** (fast):
   - call a primitive command API (gRPC Command Bus), or
   - call an external system API (e.g., Coder), or
   - write a durable record in an owning domain (preferred when available).
3. Persist the handover in Zeebe variables (minimum):
   - `work_id`, `work_kind`, `work_handle` (external ID), inputs, and trace context.
4. **Complete the Zeebe job quickly.**

**Idempotency requirement:** repeated execution of the request handler MUST NOT create duplicated external work. Use `work_id` as the idempotency key.

### B) Wait (message catch / receive task, explicit BPMN wait state)

**BPMN:** `bpmn:intermediateCatchEvent` (message) or `bpmn:receiveTask`.

- Message name: process-owned (e.g., `work.completed` or capability-specific).
- Correlation key: `= work_id` (FEEL expression via `zeebe:subscription correlationKey="=work_id"` on the message).

This makes “waiting for Coder/CI/agents” an honest, durable wait state in the diagram and engine.

### C) Resume (publish message with TTL + messageId)

When the external work finishes, resume the process by calling **publish message** (buffered) rather than correlate (non-buffered).

Required envelope:

- `name`: the wait state’s message name
- `correlationKey`: `work_id`
- `timeToLive`: > 0 (buffering to avoid races)
- `messageId`: `work_id` (idempotency while buffered)
- variables: structured outputs (URLs, digests, evidence links, status, error summary)

## What we avoid by default (anti-pattern)

Do NOT implement “minutes of work” while holding a single Zeebe job lease as the default posture.

Zeebe supports extending timeouts, but that should be treated as an escape hatch for rare bounded operations, not the standard for agentic delivery loops or CI/build pipelines.

## Worker packaging (ownership vs operations)

“One worker service owns all job types” means:

- One logical deployable (one image) owns the job-type contract for the BPMN.
- It MUST be horizontally scalable (multiple replicas) and MUST assume duplicate delivery can occur.
- Idempotency is mandatory for request handlers and completion publishing.

## Tooling implications (CLI + verify)

The CLI/verify posture remains:

- Profile/lint: BPMN is deployable to Zeebe and uses supported constructs.
- Worker coverage: every `zeebe:taskDefinition type="..."` has a handler in the owning worker service.
- Conformance: runtime semantics are proven against the dev cluster (messages, timers, incidents) via Orchestration Cluster REST API (`backlog/511-process-primitive-bpmn-conformance-suite-v2.md`).

This v3 adds a modeling constraint:

- Any step that can exceed “seconds” MUST be modeled using Request → Wait → Resume.

