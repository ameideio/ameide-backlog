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

1. **BPMN-authored business processes execute on Zeebe (default) or Flowable (supported profile).**
2. A Process primitive is shipped as a **Zeebe “process solution”**:
   - **one worker microservice** that implements **all Zeebe job types** referenced by the process definition, plus
   - deploy logic that deploys the **published** process definition to the selected BPMN runtime (Zeebe or Flowable).
3. **Long work is never performed while holding a Zeebe job lease** as the default posture.
   - The worker’s job handlers are **short, idempotent “request” steps**.
   - The BPMN explicitly models the wait via **message catch / receive task**, and resumes via **publish message (TTL + messageId)**.
4. **Test contract is 430v2**:
  - `ameide test` runs **Phase 0/1/2 only** (local-only; no cluster/Telepresence).
   - Cluster runtime semantics are validated by a **separate cluster smoke entrypoint** (not Phase 2, not Playwright E2E).
  - Playwright E2E runs separately via `ameide test e2e` against preview ingress URLs.

## Design-time governed process definitions (v6 alignment)

Under the Git-backed Enterprise Repository posture, the **canonical BPMN source of truth** is a design-time governed artifact stored as files in the **tenant Enterprise Repository**:

- A ProcessDefinition is stored and versioned in Git (e.g., `processes/<module>/<process_key>/v<major>/process.bpmn`).
- “Publishing” a process definition is advancing the tenant baseline (`main`) under policy.
- A Process primitive is the **agentic-coding-produced implementation** of that governed artifact:
  - it implements the job handlers the BPMN requires,
  - and it deploys the **published** BPMN bytes into the orchestration runtime.
 - Vendor-correct naming: `process_key` SHOULD equal the BPMN `<process id="...">` so runtime deployment/keying semantics align across Zeebe/Flowable.

### What “versioned” means (grounded)

- **Major** is encoded in the repository path (`v<major>`).
- **Minor/patch** are Git commits on `main` (optionally tagged), anchored by commit SHA for audit pointers and rollback.
- “Published process definition version” means: *a concrete commit SHA on `main` plus a process path*.
 - Recommended major policy: a major bump SHOULD introduce a new BPMN process id (new `process_key`) to avoid mixing breaking changes into one runtime key’s version chain; instance migration remains a separate (explicit) concern.

### What lives where

- **Tenant Enterprise Repository (canonical design-time):**
  - `processes/<module>/<process_key>/v<major>/process.bpmn` (canonical BPMN)
  - optional `bindings.yaml` (portable binding metadata / policy hooks)
  - optional `README.md` (functional intent; promotion gates)
- **Process primitive repo/image (runtime implementation):**
  - worker code implementing the job types required by the published BPMN,
  - deploy logic that deploys the published BPMN to Zeebe,
  - optional BPMN copy under `bpmn/` as a **fixture** for local verification/tests (not canonical).

This doc remains focused on the runtime/posture and the CLI guardrails; it intentionally does not define the full “process package” schema.

**Gap (TBD):** multi-tenant mapping rules for deploying tenant-specific process versions without collisions (flagged; not addressed here).

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
- BPMN not deployable to the selected BPMN runtime profile (Zeebe/Flowable) (missing engine bindings / unsupported constructs).
- Missing worker coverage (BPMN declares job type but worker service does not implement it).
- Request/Wait/Resume contract violations (recommended, for operability):
  - if a job type matches `*.request.*`, the BPMN must contain a following wait state correlated by `work_id`.

### `scaffold` (developer/agent contract)

Scaffolding outputs a process solution skeleton:
- `bpmn/` with a Zeebe-executable BPMN skeleton (including `bpmn:documentation` describing the functional intent and runtime contract).
- `cmd/worker` + `internal/worker` with a registry of handlers for every job type.
- A completion ingress (HTTP/gRPC) that can publish `work.completed.v1` back to Zeebe.

**Note:** under the v6 “process definitions live in the tenant Enterprise Repository” posture, the `bpmn/` copy in the Process primitive repo is a scaffold/fixture for local reasoning and tests. The canonical definition is the published ProcessDefinition in the tenant repository.

## Testing posture (must match 430v2)

### Phase 0/1/2 (local-only)

- Phase 0: verify contract + discovery (no cluster).
- Phase 1: unit tests (no cluster).
- Phase 2: integration tests against mocks/stubs/in-memory fakes (no cluster).

### Cluster runtime semantics (separate)

Runtime semantics are validated by a **cluster-only smoke suite** against the selected BPMN runtime:
- runs as a separate CLI entrypoint (not part of 0/1/2),
- deploys a conformance BPMN, drives segments, asserts message/timer/incident semantics.

See `backlog/511-process-primitive-bpmn-conformance-suite-v3.md`.
