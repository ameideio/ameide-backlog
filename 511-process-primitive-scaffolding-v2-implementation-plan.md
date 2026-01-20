# 511 – Process Primitive Scaffolding (v2) — Implementation Plan

This document turns `backlog/511-process-primitive-scaffolding-v2.md` and `backlog/511-process-primitive-bpmn-conformance-suite-v2.md` into a concrete repo plan.

> **Superseded:** see `backlog/511-process-primitive-scaffolding-v3-implementation-plan.md` (aligns execution + test gates to `backlog/430-unified-test-infrastructure-v2-target.md` and codifies Request → Wait → Resume).
>
> **Update (2026-01):** `ameide test` is Phase 0/1/2 only (local-only). Any cluster-only gates described below must run via `ameide test cluster` / `ameide test cluster` (not via the front door).

## Goal

Deliver a repo front-door that, for a BPMN-authored Process primitive:

1) scaffolds a **process solution** (BPMN + one worker microservice implementing all job types),  
2) verifies “diagram must not lie” (profile + deployability + worker coverage),  
3) deploys BPMN + worker to the dev cluster, and  
4) runs deterministic in-cluster smokes using the conformance BPMN.

## Non-negotiables

- **Zeebe is the runtime** for BPMN-authored processes (Camunda 8 / Zeebe).
- **Process primitive bundles all job workers** for its BPMN (one worker service per process primitive).
- **EDA v2**: facts on broker; commands/intents via gRPC through a deployable Command Bus (`backlog/496-eda-principles-v2.md`).
- **Conformance runner uses Orchestration Cluster REST API** (not Operate/Tasklist APIs) (`backlog/511-process-primitive-bpmn-conformance-suite-v2.md`).
- **Cluster-first**: dev namespace is the truth; no local runtime is required for acceptance.

## Work breakdown (in delivery order)

### 0) Backlogs and names are the contract

- Merge the backlog PR(s) that define:
  - process primitive = BPMN + bundled worker service (511/520/527),
  - EDA v2 contract (496),
  - Zeebe conformance suite v2 (511).

### 1) CLI: `verify` (design-time)

Implement/extend repo verification so it can answer, for a Process primitive:

- BPMN profile conformance (supported constructs + required Zeebe bindings).
- Zeebe deployability (structural checks that match runtime requirements).
- Worker coverage:
  - extract all `zeebe:taskDefinition type="..."` from BPMN,
  - assert the Process primitive worker service registers a handler for each type.

Expected output: stable, actionable errors (no “mystery failures in cluster”).

### 2) CLI: `scaffold` (process solution)

Update Process scaffolding so it outputs:

- `bpmn/` directory with a Zeebe-executable BPMN skeleton (and required documentation).
- `cmd/worker` + `internal/worker` with a worker registry that:
  - implements **all** job types used by the BPMN,
  - is safe under retries and idempotent,
  - communicates with other primitives via EDA v2 seams (Command Bus + broker facts).

### 3) Conformance primitive v2 (fixture + worker)

Add `primitives/process/bpmn_conformance_v2/`:

- Positive fixture BPMN designed for segmented execution (A–G).
- Negative fixtures for unsupported constructs.
- A worker microservice that deterministically completes/fails jobs as required by each segment.

This primitive is the repo-owned “diagram must not lie” guardrail.

### 4) In-cluster conformance runner (smokes)

Implement an in-cluster test runner that:

- Deploys the conformance BPMN to Zeebe via Orchestration Cluster REST API.
- Starts segment instances and drives them via job activation/completion/failure + message publish/correlate.
- Asserts:
  - worker coverage (jobs activatable for every type),
  - message buffering/uniqueness/cardinality semantics,
  - timer semantics (“not earlier”),
  - incident semantics (retries exhausted → incident).

### 5) Front door wiring

Wire the conformance runner into `ameide test cluster` as the cluster-only gate.

### 6) Migration / decommission

- Keep the old Temporal-based BPMN conformance suite as historical only.
- Remove it from required gates once Zeebe conformance v2 is green.

### 7) Apply to Transformation (527)

Refactor the Transformation governance process primitive so it matches the same shape:

- BPMN deployed to Zeebe.
- A single worker microservice implementing all job types in that BPMN.
- Verify + conformance gating prevents drift.

## Definition of Done (implementation)

This plan is “done” when:

- `verify` prevents deploying any BPMN that lacks worker coverage or violates the profile.
- A process scaffold produces a runnable Zeebe BPMN + bundled worker service without manual wiring.
- The conformance suite runs in the dev namespace and reliably proves:
  - deploy → run segments → assert engine semantics,
  - incidents and sequence flows are observable via Orchestration Cluster API.
- `ameide test cluster` runs these smokes against the dev cluster (cluster-only), while `ameide test` stays Phase 0/1/2 only (local-only).
