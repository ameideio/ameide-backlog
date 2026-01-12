# 527 Transformation — Proto contract notes (v2: Zeebe BPMN runtime)

This document captures the proto-level impact of adopting **Zeebe as the execution runtime** for BPMN-authored processes.
It replaces the parts of `backlog/527-transformation-proto.md` that assume “compiled Workflow IR for Temporal execution”.

## What changes

### 1) Remove BPMN→Temporal compilation artifacts as normative

The following concepts become non-normative (or removed) for BPMN-authored processes:

- `CompiledWorkflowDefinition` as “the executable representation” of BPMN.
- Any “compile-to-IR → generated Temporal runner” pipeline as the default.

### 2) Introduce Zeebe deployment references as first-class

We need explicit, stable identifiers to join:

- promoted design-time BPMN version
- deployed Zeebe process definition (process key / version)
- worker implementations and evidence

This can be represented by adding (conceptually) a “deployment reference” record/fact that links:

- `process_definition_id` + promoted version
- `zeebe_process_key` + zeebe version
- environment/namespace context

### 3) Worker coverage manifest becomes the enforceable contract

Instead of “generated workflow code drift”, we need:

- a “coverage manifest” that enumerates required job types from BPMN and maps them to owning primitives/workers
- verification that every job type has an implementation in the deployed environment

## What stays the same

- Domains remain the system of record; domain intents/facts remain the canonical audit trail.
- Process-level “facts” (if kept) remain derived/observational; they must not replace domain facts.
- EDA envelope rules (tenant, traceability, idempotency) still apply to worker→domain interactions.

## Immediate backlog edits required

- `backlog/527-transformation-proto.md` must be deprecated and replaced by a Zeebe-first version that:
  - removes Temporal compilation terms as the BPMN execution posture,
  - defines the minimum new facts/intents to represent Zeebe deployment identity and worker coverage.

