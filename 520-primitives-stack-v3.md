# Primitives Stack v3 (Backlog 520) — Camunda-only orchestration for Ameide business domains

> **Superseded:** replaced by `backlog/520-primitives-stack-v6.md` (hybrid integration posture + `io.ameide.*` naming made normative).

> **Terminology update:** the term “capability” is deprecated in this repo’s core runtime posture docs. Use **module** to mean a **business domain grouping** (e.g., `transformation`, `sales`). Use **primitive** to mean the **deployable microservice unit** (Domain/Process/Projection/Integration/Agent/UISurface).
>
> Important: “module” is an organizational label for ownership and code layout; it is not, by itself, a hard integration boundary. Integration rules are governed by owner-only writes + proto contracts (see `backlog/496-eda-principles-v6.md`).

This document replaces `backlog/520-primitives-stack-v2.md` for the orchestration/runtime posture.

## Summary (the single decision that changes everything)

We adopt a Camunda-only orchestration posture for Ameide business domains:

1. **BPMN-authored processes execute on Camunda 8 / Zeebe.**
   - BPMN is the runnable program (engine semantics).
   - “Process primitive” remains a first-class concept, but it means “BPMN definition + worker implementations + verification”, not “Temporal workflow service”.
2. **Temporal is platform-only** (internal platform concerns) and is **not part of Ameide business domains** or BPMN-driven orchestration.
3. The BPMN→Temporal transpilation/compile-to-IR effort is **discontinued** (no backward compatibility layers).

The non-negotiable rule still holds: **the diagram must not lie**.

## What stays the same (v2 principles that still apply)

The following v2 platform constitution remains true:

- Operators are operational only; they never encode business logic.
- Domains are systems of record; processes/orchestration never become canonical truth.
- Protos define contracts; generation is deterministic; CI gates enforce drift discipline.
- Integration rules remain (owner-only writes, facts-after-commit, idempotency, at-least-once consumers).
- Configuration has a single authority (cluster conventions vs GitOps-injected vs request payloads).

## What changes (v2 → v3)

### A) Process primitive runtime changes

**Before (v2):**
- Process primitives = Temporal workflows + Activities/Updates, optionally derived from BPMN via compilation.

**Now (v3):**
- BPMN-authored Process primitives = Zeebe deployments + workers that implement side effects.
- Temporal is not used as an execution engine for Ameide business domain orchestration.

### B) Meaning of BPMN extensions changes

We distinguish:

- **Zeebe/Camunda runtime bindings**: whatever the engine requires to execute BPMN (e.g., job type bindings).
- **Ameide extensions (`ameide:*`)**: design-time contracts that drive:
  - verification (profile conformance + operability checks),
  - scaffolding of worker stubs/tests,
  - ownership mapping (which primitive implements which job types),
  - evidence/traceability requirements.

`ameide:*` extensions are not relied on for runtime semantics.

## Tooling contract (CLI + CI)

The Ameide CLI remains the orchestration/guardrails tool, but its process responsibilities change:

- Verify:
  - validates BPMN against the selected profile(s),
  - validates Zeebe deployability constraints,
  - validates worker coverage for all side-effect steps.
- Scaffold:
  - produces worker stubs/tests in the owning primitive repos,
  - produces deployment manifests (GitOps references) for Zeebe definitions where applicable,
  - does not generate Temporal workflow code from BPMN.

## CI conformance gate (runtime truth)

To enforce “diagram must not lie” at runtime, we keep a cluster-backed conformance suite that:

- deploys a BPMN fixture to Zeebe,
- starts process instances for targeted segments,
- activates/completes/fails jobs using an in-test worker loop,
- publishes/correlates messages, and
- asserts incidents are created for exhausted retries.

Vendor constraint to account for: job activation cannot be scoped to a single process instance, so test fixtures must avoid cross-run job-type collisions (e.g. per-run job type namespace or per-run process id).

Vendor “physics” to make explicit: Zeebe job `timeout` is the **lease duration** after which the engine may reassign a job; it is not an “execution time budget”. This is why the platform default is short, idempotent request handlers and explicit BPMN wait states for long work.

## Backlog mapping / replacements

This v3 posture requires v2 backlog documents that assume “Temporal-backed Process primitives” to be deprecated and replaced by Zeebe-specific versions, notably:

- `backlog/511-*` Process primitive scaffolding/compiler docs (superseded by Zeebe-based direction).
- `backlog/527-*` Transformation Process/Proto docs that assume compiled Workflow IR for Temporal execution.
- EDA posture docs that assume “Kafka-only” routing: superseded by `backlog/496-eda-principles-v6.md`.

## Naming posture (event types)

Event type namespaces are owned by Ameide and use `io.ameide.*` (not `com.ameide.*`).
This matters because Zeebe message names are mapped directly from CloudEvent `type` strings (`messageName == ce.type`).

## Integration rule (EDA scope)

This repo uses an industry-aligned **hybrid** integration posture (`backlog/496-eda-principles-v6.md`):

- **Commands** (often RPC) request a change from an owner.
- **Facts** (often events) propagate committed outcomes for replay/fan-out and for derived read models.
- **Owner-only writes** is the core invariant: no primitive mutates another owner’s canonical store (DB or Git).

“Intra-module vs inter-module” is not a correctness rule; it is a coupling/operability choice:

- Cross-domain workflows (processes) spanning multiple owners are normal (Saga / process manager pattern).
- Prefer events for propagation and loose coupling; prefer RPC for command issuance and synchronous validation.
- Avoid using broker events as internal control flow inside a single process primitive; Zeebe BPMN is the control flow.

### Git-backed owners (canonical store is not always a DB)

Domains may be Git-backed owners (e.g., canonical content as files in a tenant repository). In that case:

- “commit” means Git commit/merge/tag + durable audit pointers, and only then emit facts.
- Process/Agent/Integration/UISurface never write to the tenant repo directly; they route writes via the owning Domain.
