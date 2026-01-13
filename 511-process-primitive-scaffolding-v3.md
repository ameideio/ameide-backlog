---
title: "511 – Process Primitive Scaffolding (v3): Zeebe process solution + 430-aligned test contract"
status: draft
owners:
  - platform
created: 2026-01-13
parent: 511-process-primitive-scaffolding-v2.md
supersedes:
  - 511-process-primitive-scaffolding-v2.md
---

# 511 – Process Primitive Scaffolding (v3): Zeebe process solution + 430-aligned test contract

This document updates the v2 Zeebe posture to be **explicitly aligned** with the repo-wide test contract in `backlog/430-unified-test-infrastructure-v2-target.md`.

## Decision summary (v3)

1. **BPMN-authored processes execute on Camunda 8 / Zeebe.**
2. A Process primitive is shipped as a **Zeebe “process solution”**:
   - executable BPMN definition(s), plus
   - **one worker microservice** that implements **all Zeebe job types** referenced by that BPMN.
3. **Long work is never performed while holding a Zeebe job lease** as the default posture.
   - The worker’s job handlers are **short, idempotent “request” steps**.
   - The BPMN explicitly models the wait via **message catch / receive task**, and resumes via **publish message (TTL + messageId)**.
4. **Test contract is 430v2**:
   - `ameide dev inner-loop-test` and `ameide ci test` run **Phase 0/1/2 only** (local-only; no cluster/Telepresence).
   - Cluster runtime semantics are validated by a **separate cluster smoke entrypoint** (not Phase 2, not Playwright E2E).
   - Playwright E2E runs separately via `ameide ci e2e` against preview ingress URLs.

## Canonical runtime template (vendor-aligned): Request → Wait → Resume

For any BPMN step that can take more than “seconds” (build/tests/rollout/coder loops), the canonical Zeebe shape is:

1) **Request** (serviceTask; short job handler)
- Job type naming: `*.request.v1` (recommended convention).
- Worker behavior: compute a stable `work_id`, durably request work, set variables, complete job quickly.

2) **Wait** (message catch / receive task)
- BPMN waits for `work.completed.v1` correlated by `work_id`.
- Correlation key is configured via `zeebe:subscription correlationKey="=work_id"` on the message definition (Zeebe semantics).

3) **Resume** (external completion publishes message)
- Completion publishes a Zeebe message:
  - `timeToLive > 0` to buffer early completion,
  - `messageId = work_id` for idempotency while buffered,
  - outputs (evidence links, PR URL, digest(s), etc.) are included as variables.

This template prevents “long step under job lease” anti-patterns and makes retries/duplicates manageable.

## What the CLI must provide (v3)

### `verify` (Phase 0, design-time contract)

`verify` MUST be able to fail fast with stable errors for:
- BPMN not deployable to Zeebe (missing engine bindings / unsupported constructs).
- Missing worker coverage (BPMN declares job type but worker service does not implement it).
- Request/Wait/Resume contract violations (recommended, for operability):
  - if a job type matches `*.request.*`, the BPMN must contain a following wait state correlated by `work_id`.

### `scaffold` (developer/agent contract)

Scaffolding outputs a process solution skeleton:
- `bpmn/` with Zeebe-executable BPMN (including `bpmn:documentation` describing the functional intent and runtime contract).
- `cmd/worker` + `internal/worker` with a registry of handlers for every job type.
- A completion ingress (HTTP/gRPC) that can publish `work.completed.v1` back to Zeebe.

## Testing posture (must match 430v2)

### Phase 0/1/2 (local-only)

- Phase 0: verify contract + discovery (no cluster).
- Phase 1: unit tests (no cluster).
- Phase 2: integration tests against mocks/stubs/in-memory fakes (no cluster).

### Cluster runtime semantics (separate)

Zeebe runtime semantics are validated by a **cluster-only smoke suite** (Camunda Orchestration Cluster REST API):
- runs as a separate CLI entrypoint (not part of 0/1/2),
- deploys a conformance BPMN, drives segments, asserts message/timer/incident semantics.

See `backlog/511-process-primitive-bpmn-conformance-suite-v3.md`.

