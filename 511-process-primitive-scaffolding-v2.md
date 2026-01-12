---
title: "511 – Process Primitive Scaffolding (v2): BPMN executed by Zeebe"
status: draft
owners:
  - platform
created: 2026-01-12
---

# 511 – Process Primitive Scaffolding (v2): BPMN executed by Zeebe

## 0) Decision summary (this replaces v1 compile-to-Temporal)

We adopt a single orchestration posture for Ameide business capabilities:

1. **BPMN-authored processes run on Camunda 8 / Zeebe.**
   - A “Process primitive” remains a first-class concept, but its runtime is **a BPMN definition deployed to Zeebe**, not a Temporal Workflow compiled from BPMN.
2. **Temporal is platform-only** (internal platform concerns) and is **not part of Ameide business capabilities** or BPMN-driven orchestration.
3. **The BPMN→Temporal transpilation effort is discontinued** (no shims, no backward compatibility layers).

## 1) What “Process primitive” means in v2

In v2, a Process primitive is:

- **A BPMN process definition** (and related assets) that is deployed to Zeebe as the **runtime program**, and
- **A single worker microservice** that implements *all* Zeebe job workers required by that BPMN (“process solution” packaging), and
- **A verification + conformance test gate** that proves the diagram is deployable and that worker coverage exists.

The Process primitive is **not** a Temporal worker image generated from BPMN.

## 2) BPMN extensions: what they mean now (design-time only)

We still want BPMN to be the coordination surface for agentic delivery, but with Zeebe as the executor:

- **Zeebe/Camunda extensions** (engine-level) define what is runnable on Zeebe (e.g. `zeebe:taskDefinition type="..."` for service tasks, `zeebe:subscription` on messages).
- **Ameide extensions** (`ameide:*`) are **design-time contracts** that drive:
  - worker scaffolding (what side effects must exist),
  - verification (lint + coverage),
  - evidence/traceability requirements,
  - but are **not** relied on for Zeebe runtime semantics.

This preserves “BPMN is the single authoring truth” while preventing “engine semantics drift”:
Zeebe executes BPMN; Ameide verifies that the process is safe/operable and that required worker capabilities exist.

## 3) Runtime posture (Zeebe orchestration; primitives implement side effects)

### 3.1 Side effects are not “inside the process engine”

Zeebe orchestrates; **workers perform side effects**.

In Ameide terms, the workers should be implemented as primitives:

- **Agent primitives**: implement coding/design tasks as Zeebe job workers.
- **Integration primitives**: implement tool executions, CI operations, GitHub interactions, etc., as Zeebe job workers.
- **Domain primitives**: remain the system of record; workers call domain APIs/commands; domains emit facts.

The process engine MUST NOT become a canonical state store. It coordinates, it doesn’t own truth.

### 3.2 Human tasks

Human approvals and decisions are expressed as BPMN user tasks and executed via:

- Camunda Tasklist (default), or
- Ameide UI bridging to Tasklist semantics (explicitly modeled; no hidden custom runtime).

## 4) What the CLI/scaffolder does now (v2, implemented)

The CLI shifts from “compile BPMN into Temporal code” to “verify + scaffold worker contracts + deploy to Zeebe”:

### 4.1 `verify` (normative guardrail)

Verification must answer:

1) **Is this BPMN runnable on Zeebe?**
  - required Zeebe bindings present where needed
  - no unsupported constructs for our chosen engine profile

2) **Is this BPMN acceptable under Ameide’s governance profile?**
  - required metadata exists (process identity, correlation rules, evidence expectations)
  - “diagram must not lie”: if a construct is present, we must have an execution + ops meaning for it

3) **Do we have worker coverage for every side-effect step?** (Zeebe job types)
  - every Zeebe job type declared in BPMN has a corresponding worker handler implementation
  - ownership is explicit: by definition, the Process primitive owns the full set of workers for its BPMN

### 4.2 `scaffold` (developer/agent enablement)

Scaffolding is no longer “generate workflow code”.

Instead it generates/updates:

- a Zeebe-first Process primitive skeleton (BPMN + worker microservice),
- deterministic worker artifacts generated from BPMN job types,
- idempotent handler stubs per job type (created once, never overwritten),
- a conformance suite that can deploy BPMN + drive workers in a real cluster.

**Repository shape (scaffold output, high level):**

```text
primitives/process/<name>/
├── bpmn/process.bpmn
├── cmd/worker/main.go
└── internal/worker/
    ├── worker.go                  # runtime wiring (client + worker registration)
    ├── handlers.go                # handler registry, shared helpers
    ├── job_types_gen.go           # generated list of job types found in BPMN
    ├── handlers_gen.go            # generated registration stubs
    └── handler_<jobtype>.go       # implementation stubs (created if missing)
```

## 5) What to deprecate (v1 artifacts)

The v1 stack is kept for historical context but deprecated:

- BPMN→Temporal compilation, generated `_gen.go`, `compile.lock.json`-as-compiler output
- Temporal testsuite-based BPMN execution semantics conformance (for BPMN-authored processes)

## 6) Conformance suite and cluster requirements (implemented)

The Process primitive posture is enforced by an end-to-end conformance suite that runs against the dev Camunda cluster:

- `primitives/process/bpmn_conformance_v2` (deploy BPMN, start instances, activate/complete/fail jobs, publish messages, assert incidents).
- Important vendor constraint: `/v2/jobs/activation` cannot be scoped to a single `processInstanceKey`, so the conformance runner deploys a per-run rewritten BPMN (unique process id + job types + message name) to avoid cross-run interference.

Cluster wiring required for CI/M2M deployability:

- OIDC `client-id-claim` must be `"azp"` so client_credentials tokens map to clients correctly.
- A dedicated machine principal (e.g. `camunda-deployer`) must be mapped to an admin/deployer default role and have its secret present in-cluster (e.g. `camunda-oidc-client-secrets` key `deployer`).

## 7) Follow-up edits required in other backlogs

This decision implies updates to:

- `backlog/520-primitives-stack-v2.md`: replace “Temporal-backed Process primitives” posture with Camunda-only capability orchestration (see `backlog/520-primitives-stack-v3.md`).
- `backlog/527-transformation-*.md`: remove/replace compile-to-Temporal execution assumptions for BPMN-authored governance workflows; align “runnable BPMN” with Zeebe.
- `backlog/511-*` v1 docs: mark deprecated and point to this document as the v2 direction.
