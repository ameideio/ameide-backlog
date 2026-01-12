# Primitives Stack v3 (Backlog 520) — Zeebe for BPMN, Temporal for coded platform workflows

This document replaces `backlog/520-primitives-stack-v2.md` for the orchestration/runtime posture.

## Summary (the single decision that changes everything)

We adopt a **dual-orchestrator** platform posture with strict scope:

1. **BPMN-authored processes execute on Camunda 8 / Zeebe.**
   - BPMN is the runnable program (engine semantics).
   - “Process primitive” remains a first-class concept, but for BPMN processes it means “BPMN definition + worker implementations + verification”, not “Temporal workflow service”.
2. **Temporal remains part of the platform**, but is used for **platform workflows authored as code without BPMN design-time**.
3. The BPMN→Temporal transpilation/compile-to-IR effort is **discontinued** (no backward compatibility layers).

The non-negotiable rule still holds: **the diagram must not lie**.

## What stays the same (v2 principles that still apply)

The following v2 platform constitution remains true:

- Operators are operational only; they never encode business logic.
- Domains are systems of record; processes/orchestration never become canonical truth.
- Protos define contracts; generation is deterministic; CI gates enforce drift discipline.
- EDA rules remain (idempotency, outbox for writers, at-least-once consumers).
- Configuration has a single authority (cluster conventions vs GitOps-injected vs request payloads).

## What changes (v2 → v3)

### A) Process primitive runtime changes

**Before (v2):**
- Process primitives = Temporal workflows + Activities/Updates, optionally derived from BPMN via compilation.

**Now (v3):**
- BPMN-authored Process primitives = Zeebe deployments + workers that implement side effects.
- Temporal is not used as the execution engine for BPMN processes.

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

## Backlog mapping / replacements

This v3 posture requires v2 backlog documents that assume “Temporal-backed Process primitives” to be deprecated and replaced by Zeebe-specific versions, notably:

- `backlog/511-*` Process primitive scaffolding/compiler docs (superseded by Zeebe-based direction).
- `backlog/527-*` Transformation Process/Proto docs that assume compiled Workflow IR for Temporal execution.

