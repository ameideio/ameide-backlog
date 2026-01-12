# 527 Transformation — Domain Specification (v2: Zeebe BPMN runtime)

This document replaces the parts of `backlog/527-transformation-domain.md` that assume BPMN is compiled to Temporal-backed `CompiledWorkflowDefinition`.

## Summary

Transformation Domain remains the design-time and canonical data owner for:

- Enterprise Knowledge substrate (Elements + Versions + Relationships).
- Definition Registry (promotable definitions and promotion state).
- Governance artifacts (baselines, approvals, evidence references).

What changes in v2 is the runtime binding for BPMN-authored processes:

- BPMN ProcessDefinitions are deployed to **Zeebe** and executed there.
- The Definition Registry no longer needs “compiled workflow IR for Temporal” as a mandatory derived artifact for BPMN.

## Definition Registry (v2)

### Still first-class (design-time)

- `ProcessDefinition` (BPMN as canonical authoring payload; versioned/promotable).
- `ScaffoldingPlanDefinition` (optional, if we keep a first-class “worker scaffolding plan” derived from BPMN + contracts).
- `AgentDefinition` (agent behaviors and runtime config, where applicable).
- `ExtensionDefinition` (profile definitions, validators, editor metadata, etc.).

### Replaced / removed for BPMN processes

- `CompiledWorkflowDefinition` is no longer the normative derived artifact for BPMN process execution.

Instead, we require explicit **deployment identity** and **worker coverage** contracts:

- `ProcessDeploymentRef` (concept): links `{process_definition_id, promoted_version}` to Zeebe `{process_key, version}` in an environment.
- `WorkerCoverageDefinition` (concept): enumerates required job types and maps them to owning primitives/workers in an environment.

These are enforced by verify/promotion/deploy gates rather than by compiling BPMN into another orchestration runtime.

## Canonical truth boundaries (unchanged)

- Domains remain the system of record for business truth and emit facts after commit (outbox).
- Zeebe execution history is orchestration history, not canonical business truth.
- Workers (agent/integration) call domain APIs/commands; domain facts remain the audit spine.

