---
title: "511 – Process Primitive Scaffolding (v2): BPMN executed by Zeebe"
status: draft
owners:
  - platform
created: 2026-01-12
superseded_by:
  - 511-process-primitive-scaffolding-v3.md
---

# 511 – Process Primitive Scaffolding (v2): BPMN executed by Zeebe

> **Superseded:** see `backlog/511-process-primitive-scaffolding-v3.md` (aligns process packaging + test posture to `backlog/430-unified-test-infrastructure-v2-target.md` and codifies Request → Wait → Resume for long work).

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

### 3.3 Inter-primitive messaging posture (EDA v2)

This backlog is about process execution on Zeebe, but the worker microservice is still a Kubernetes service and must follow the platform messaging standard:

- Inter-primitive messaging inside Kubernetes follows `backlog/496-eda-principles-v2.md` (CloudEvents + Knative Broker/Trigger).
- Any “ingress” component that consumes inter-primitive facts/intents MUST be a Trigger subscriber and MUST NOT rely on stdin JSONL envelopes as an operational posture.

Zeebe job activation/completion is a Zeebe-internal mechanism; it does not replace the EDA v2 fact spine for canonical truth (domains still emit facts after commit via outbox).

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

## 4.3 Conformance gate (v2)

The v2 “diagram must not lie” guardrail is a **Zeebe runtime conformance suite**:

- a single “coverage BPMN” fixture tested by **small, deterministic segments**
- engine truth asserted via **Orchestration Cluster REST API** (not Operate/Tasklist APIs)
- message semantics tested using buffering/uniqueness/cardinality behavior (TTL + messageId)
- timer semantics asserted as “not earlier; may be later”
- incidents asserted via engine incident search for the process instance

See `backlog/511-process-primitive-bpmn-conformance-suite-v2.md`.

## 5) Definition of Done (scaffold + verify + deploy + smoke)

This is the Definition of Done for the v2 Process primitive toolchain:

### A) Scaffolding (repo state)

- A new Process primitive exists with:
  - a BPMN definition intended for Zeebe execution,
  - one worker microservice that implements **all** BPMN job types used by that definition,
  - CI/test wiring compatible with `ameide test` (no ad-hoc runners).
- The scaffold produces a worker entrypoint that:
  - registers handlers for each `zeebe:taskDefinition type="..."`,
  - calls other primitives only via the seams defined in `backlog/496-eda-principles-v2.md` (facts on broker; commands via gRPC/Command Bus),
  - is idempotent and safe under retries.

### B) Verification (gate)

- `verify` fails fast with stable, actionable errors when:
  - BPMN is not deployable to Zeebe (missing/bad runtime bindings),
  - unsupported constructs exist (profile rejection),
  - any job type is missing a worker handler in the Process primitive worker microservice,
  - any required EDA v2 metadata contracts are violated (e.g., deprecated proto namespaces referenced).

### C) Deployment (dev cluster)

- The BPMN is deployed to Zeebe in the dev namespace (GitOps-driven).
- The Process primitive worker microservice is deployed, healthy, and connected to Zeebe.

### D) Smoke / conformance (dev cluster)

- A conformance BPMN is used as the smoke gate (segment-based, deterministic).
- Smokes run in-cluster (dev namespace) and prove:
  - jobs can be activated/completed for every job type used,
  - message publish/correlate behavior matches engine semantics,
  - timers are asserted as “not earlier” (may be later, within timeout),
  - incidents are created and observable when retries are exhausted,
- negative fixtures are rejected at verify time with stable errors.

See `backlog/511-process-primitive-scaffolding-v2-implementation-plan.md`.

## 6) What to deprecate (v1 artifacts)

The v1 stack is kept for historical context but deprecated:

- BPMN→Temporal compilation, generated `_gen.go`, `compile.lock.json`-as-compiler output
- Temporal testsuite-based BPMN execution semantics conformance (for BPMN-authored processes)

## 6.1 Legacy primitives to treat as non-normative

The repo currently contains v1/legacy Process primitives that are Temporal/JSONL-shaped (e.g., `primitives/process/transformation_v2`, `primitives/process/bpmn_conformance_v1`).

They are useful as historical reference, but they are **not the v2 target** and should not drive new work. The v2 target is “BPMN deployed to Zeebe + worker coverage + EDA v2 boundaries”.

## 7) Conformance suite and cluster requirements (implemented)

The Process primitive posture is enforced by an end-to-end conformance suite that runs against the dev Camunda cluster:

- `primitives/process/bpmn_conformance_v2` (deploy BPMN, start instances, activate/complete/fail jobs, publish messages, assert incidents).
- Important vendor constraint: `/v2/jobs/activation` cannot be scoped to a single `processInstanceKey`, so the conformance runner deploys a per-run rewritten BPMN (unique process id + job types + message name) to avoid cross-run interference.

Cluster wiring required for CI/M2M deployability:

- OIDC `client-id-claim` must be `"azp"` so client_credentials tokens map to clients correctly.
- A dedicated machine principal (e.g. `camunda-deployer`) must be mapped to an admin/deployer default role and have its secret present in-cluster (e.g. `camunda-oidc-client-secrets` key `deployer`).

## 8) Follow-up edits required in other backlogs

This decision implies updates to:

- `backlog/520-primitives-stack-v2.md`: replace “Temporal-backed Process primitives” posture with Camunda-only capability orchestration (see `backlog/520-primitives-stack-v3.md`).
- `backlog/527-transformation-*.md`: remove/replace compile-to-Temporal execution assumptions for BPMN-authored governance workflows; align “runnable BPMN” with Zeebe.
- `backlog/511-*` v1 docs: mark deprecated and point to this document as the v2 direction.
