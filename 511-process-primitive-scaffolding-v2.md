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

- **A BPMN process definition** (and related assets) that is:
  - versioned/promoted as design-time data (Transformation/Definition Registry posture),
  - deployed to Zeebe as the **runtime program**, and
  - observed via Camunda Operate/Tasklist for execution state and incidents.

The Process primitive is **not** a Temporal worker image generated from BPMN.

## 2) BPMN extensions: what they mean now (design-time only)

We still want BPMN to be the coordination surface for agentic delivery, but with Zeebe as the executor:

- **Zeebe/Camunda extensions** (engine-level) define what is runnable on Zeebe.
  - Example intent: service task job type, retries, etc. (Zeebe needs runtime bindings.)
- **Ameide extensions** (`ameide:*`) become **design-time contracts** that:
  - declare what workers/agents must implement for each side effect,
  - define idempotency/correlation/evidence requirements,
  - drive scaffolding and verification (“coverage”) tooling,
  - but are not relied on for runtime semantics.

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

## 4) What the CLI/scaffolder does now (v2)

The CLI shifts from “compile BPMN into Temporal code” to “verify + scaffold worker contracts + deploy to Zeebe”:

### 4.1 `verify` (normative guardrail)

Verification must answer:

1) **Is this BPMN runnable on Zeebe?**
  - required Zeebe bindings present where needed
  - no unsupported constructs for our chosen engine profile

2) **Is this BPMN acceptable under Ameide’s governance profile?**
  - required metadata exists (process identity, correlation rules, evidence expectations)
  - “diagram must not lie”: if a construct is present, we must have an execution + ops meaning for it

3) **Do we have worker coverage for every side-effect step?**
  - every service task/external task type maps to an owning primitive worker implementation
  - ownership is explicit (no heuristics)

### 4.2 `scaffold` (developer/agent enablement)

Scaffolding is no longer “generate workflow code”.

Instead it generates/updates:

- a worker contract manifest (“these job types exist, these primitives own them, these APIs must be called”),
- stub worker handlers in the owning primitive(s) where applicable (or at minimum TODO stubs + tests),
- test fixtures that prove:
  - the process can be deployed to Zeebe,
  - workers can poll and complete a simple happy path,
  - failures produce incidents and are observable.

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
  - CI/test wiring compatible with `./ameide dev inner-loop-test` (no ad-hoc runners).
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

## 7) Follow-up edits required in other backlogs

This decision implies updates to:

- `backlog/520-primitives-stack-v2.md`: replace “Temporal-backed Process primitives” posture with Camunda-only capability orchestration (see `backlog/520-primitives-stack-v3.md`).
- `backlog/527-transformation-*.md`: remove/replace compile-to-Temporal execution assumptions for BPMN-authored governance workflows; align “runnable BPMN” with Zeebe.
- `backlog/511-*` v1 docs: mark deprecated and point to this document as the v2 direction.
