# 527 — Transformation Scenario (TOGAF ADM track; flow-only)

**Status:** Draft  
**Parent:** `backlog/527-transformation-capability.md`  
**Methodology overlay:** TOGAF ADM (dedicated UISurface + workflow over the same element graph)
**ProcessDefinition ID (recommended):** `r2r.governance.togaf_adm.v1`

This is a **flow-only** view of R2R (“Requirement → Release”) when the change is governed by a **TOGAF ADM** overlay.

The path is ADM-native (A→B→C→D→… with explicit deliverables), but all design-time truth is stored as **Elements + Relationships + Versions**.

## Legend (used in “Next”)

- **Trigger**: start the next action immediately
- **Wait**: block until a signal/fact/status is observed
- **Skip**: conditional bypass (step is not applicable)
- **Loop**: return to an earlier step
- **End**: terminate the workflow

---

## Prerequisites (implementation)

These are the minimum implementation capabilities required for this scenario to run end-to-end.

- [ ] **Domain (Enterprise Knowledge)** supports Elements/Versions/Relationships writes for `{tenant_id, organization_id, repository_id}`.
- [ ] **REFERENCE relationships are versioned**: governance anchors (e.g., `ref:requirement`, `ref:deliverables_root`) include `metadata.target_version_id`.
- [ ] **Projection** can read and render:
  - [ ] change element + its outgoing `ref:*` relationships (including `metadata.target_version_id`)
  - [ ] deliverables root membership (relationships under the deliverables root)
  - [ ] phase gate evidence and “phase completeness” signals derived from relationships + evidence
- [ ] **WorkRequests substrate** works end-to-end:
  - [ ] Domain records WorkRequests and emits `transformation.work.domain.facts.v1`
  - [ ] executor processes tool-run queues and records outcomes + evidence back to Domain
  - [ ] Process emits `ToolRunRecorded` and step transitions on `transformation.process.facts.v1`
- [ ] **UISurface** can create/edit:
  - [ ] a change element (`methodology_key = togaf_adm`)
  - [ ] requirement draft element(s) and snapshots (ElementVersions)
  - [ ] anchor relationships (`ref:requirement`, `ref:deliverables_root`) with `metadata.target_version_id`

---

## Phase 1 — Initiate (intake + articulation + Phase A readiness)

### 1.1 Capture change element (request for change / work)
- **Input:** Raw ask (often incomplete); target capability; initial “agentic decision” flag if known.
- **Output:** Change element created with canonical scope `{tenant_id, organization_id, repository_id}` and `methodology_key = togaf_adm` (stored on the change element body).
- **Next:** **Trigger** 1.2

### 1.2 Requirement articulation (agent-assisted; iterative)
- **Input:** Change element; stakeholder concerns; existing repository content/definitions/contracts.
- **Output:** Requirement draft element(s) (docs/views) linked from the change element (drivers, goals, constraints, scenarios/examples, acceptance criteria, glossary).
- **Next:** If still ambiguous → **Loop** 1.2; else **Trigger** 1.3

### 1.3 Record requirement status (Domain fact; anchored element version)
- **Input:** Latest requirement draft element version(s).
- **Output:** Requirement status recorded as a Domain fact on the change element (e.g., `DRAFT | NEEDS_INPUT | STABILIZED | CANCELED`) and the anchor reference relationship is updated: `ref:requirement` (`relationship_kind=REFERENCE`, `metadata.target_version_id=<requirement_version_id>`).
- **Next:** If `STABILIZED` → **Trigger** 1.4; if `CANCELED` → **End**; else **Loop** 1.2

### 1.4 Start governance workflow (ADM run)
- **Input:** Change element id/scope + anchored stabilized requirement ref(s) (`ref:requirement`).
- **Output:** R2R workflow instance started (using `r2r.governance.togaf_adm.v1`); “requirements captured” process fact emitted; ADM run context initialized.
- **Next:** **Trigger** Phase 2

### Phase 1 — Implementation checklist

- [ ] **Domain**: requirement status fact/field exists and is queryable.
- [ ] **Domain**: `ref:requirement` relationship stores `metadata.target_version_id` and is updatable idempotently.
- [ ] **UISurface**: supports requirement articulation loop and “stabilize” decision recording.
- [ ] **Projection**: exposes the anchored requirement snapshot on the change view.

---

## Phase 2 — Architect (ADM A/B/C/D; deliverables + gates)

### 2.A Architecture Vision (ADM Phase A)
- **Input:** Stabilized requirement snapshot; stakeholder concerns; baseline repository state.
- **Output:** Deliverables package element created/updated and anchored via `ref:deliverables_root` (`relationship_kind=REFERENCE`, `metadata.target_version_id=<deliverables_root_version_id>`); Phase A deliverables stored as Elements/Views/Docs under that package (vision, scope, principles, stakeholder map).
- **Next:** **Trigger** 2.B

### 2.B Business Architecture (ADM Phase B)
- **Input:** Phase A outputs.
- **Output:** Business architecture slice stored as Elements/Views/Docs and linked; gaps/impacts captured.
- **Next:** **Trigger** 2.C

### 2.C Information Systems Architecture (ADM Phase C)
- **Input:** Phase B outputs + existing contracts.
- **Output:** Data + application architecture slices stored as Elements/Views/Docs and linked (semantics, ownership boundaries, integration touchpoints).
- **Next:** **Trigger** 2.D

### 2.D Technology Architecture (ADM Phase D)
- **Input:** Phase C outputs + platform constraints.
- **Output:** Technology architecture slice stored as Elements/Views/Docs and linked (runtime topology constraints, NFRs, env boundaries, evidence requirements).
- **Next:** **Trigger** 2.1

### 2.1 Define contract + message boundary (derived from A–D)
- **Input:** Anchored A–D deliverables + requirement snapshot; current capability protos.
- **Output:** Proto diff spec and checklist (touched primitives, migrations/tests); derived contract package linked from the change element.
- **Next:** **Trigger** 2.2

### 2.2 Update capability BPMN model (optional)
- **Input:** Capability workflow model (if it exists).
- **Output:** New versioned BPMN element reflecting the change (tasks/decisions/message boundaries).
- **Next:** If BPMN not used → **Skip** 2.2 and **Trigger** 2.3; else if model-only → **Trigger** 2.3; else if executable → **Trigger** 2.2a

### 2.2a Bind executable BPMN (only if executable)
- **Input:** BPMN model + bindings (send intents / await facts / correlation rules).
- **Output:** Promotable BPMN execution profile definition (recommended as an `ExtensionDefinition`).
- **Next:** **Trigger** 2.3

### 2.3 Define agent scope (only if agentic)
- **Input:** “Agentic decision” flag; desired tool grants; risk tier.
- **Output:** AgentDefinition diff recorded as a promotable definition.
- **Next:** If no agentic component → **Skip** 2.3; **Trigger** 2.4

### 2.4 Architecture gate (A–D complete)
- **Input:** Proposed A–D deliverables + contract package + (optional) BPMN artifacts + (optional) AgentDefinition.
- **Output:** Gate decision recorded (Approve/Reject) as workflow signal + process facts; validation evidence is recorded as evidence linked from the change element (e.g., `evidence:adm_phase_gates`) and can assert “A–D complete” against `ref:deliverables_root`.
- **Next:** **Wait** gate decision; if Approve → **Trigger** Phase 3; if Reject → **Loop** Phase 2 (revise deliverables); if Cancel → **End**

### Phase 2 — Implementation checklist

- [ ] **Domain**: deliverables package/root element exists and is versioned; `ref:deliverables_root` stores `metadata.target_version_id`.
- [ ] **UISurface**: provides Phase A–D workspace navigation/templates (even if minimal) that create/update deliverables as elements.
- [ ] **Projection**: can render phase completeness deterministically from relationships + evidence (no “search for what counts”).
- [ ] **Validation**: Phase A–D completeness checks exist (v0 can be manual evidence + simple projection rules; v1 can be WorkRequest-executed validators).

---

## Phase 3 — Realize (work packages + tool runs + verification)

### 3.1 Request scaffold/codegen tool run
- **Input:** Accepted design artifacts; repo coordinates; idempotency key; `action_kind=scaffold|generate`.
- **Output:** Domain WorkRequest created; `WorkRequested` emitted to work queue (e.g., `transformation.work.queue.toolrun.generate.v1`).
- **Next:** **Trigger** Integration executor job; **Wait** 3.1a

### 3.1a Tool run outcome recorded
- **Input:** WorkRequest id.
- **Output:** Evidence bundle + terminal status recorded in Domain; Process emits `ToolRunRecorded` referencing evidence.
- **Next:** If success → **Trigger** 3.2; if failure → **Loop** 3.1 or **Wait** for repair then re-**Trigger** 3.1

### 3.2 Implement repo changes (agent or human)
- **Input:** Scaffolded repo state + accepted artifacts.
- **Output:** PR/branch with: proto changes, regenerated outputs, capability domain/process/agent/projection/UI updates, tests.
- **Next:** **Trigger** 3.3

### 3.3 Request verification tool run
- **Input:** PR ref; `action_kind=verify`; verification baseline.
- **Output:** Domain WorkRequest created; `WorkRequested` emitted (e.g., `transformation.work.queue.toolrun.verify.v1`).
- **Next:** **Trigger** Integration verifier job; **Wait** 3.3a

### 3.3a Verification outcome recorded
- **Input:** WorkRequest id.
- **Output:** Verification report + evidence bundle; pass/fail status.
- **Next:** If pass → **Trigger** Phase 4; if fail → **Loop** 3.2 (fix) or if “architecture-invalidates-contract” → **Loop** Phase 2 (revise A–D then re-verify)

### Phase 3 — Implementation checklist

- [ ] **Domain**: WorkRequest lifecycle is persisted and emits `WorkRequested/Started/Completed/Failed` facts.
- [ ] **Integration runner**: can execute `scaffold|generate|verify` deterministically and record evidence (logs + artifacts).
- [ ] **Process**: emits step-level process facts and correlates to WorkRequests.
- [ ] **Projection/UISurface**: can show WorkRequest status + evidence links and the process run timeline.

---

## Phase 4 — Promote (governed release decision)

### 4.1 Promotion gate
- **Input:** PR contents + verification evidence + phase completion evidence.
- **Output:** Gate request for promoting: definitions (Process/Agent/Extension), and release baseline; decision recorded as evidence.
- **Next:** **Wait** Approve/Reject/Override/Cancel; if Approve/Override → **Trigger** 4.2; if Reject → **Loop** Phase 3; if Cancel → **End**

### 4.2 Record promotions (system-of-record)
- **Input:** Approved promotion decisions.
- **Output:** Domain records promoted versions/baseline; promotion facts emitted; projection reflects audit trail.
- **Next:** **Trigger** Phase 5

### Phase 4 — Implementation checklist

- [ ] **Domain**: baselines exist as immutable `{element_id, version_id}` sets and can be promoted.
- [ ] **Projection**: can render baseline membership and promotion history for audit.
- [ ] **Process/UISurface**: phase gate decisions and promotion decisions are recorded and visible.

---

## Phase 5 — Release (merge + deploy + governance visibility)

### 5.1 Merge + CI build + deploy
- **Input:** Approved PR + promoted definitions/baseline.
- **Output:** CI pipeline run + deploy execution; deployment evidence captured.
- **Next:** **Wait** CI/deploy status; if fail → **Loop** Phase 3 (fix) or **Trigger** rollback/disable if already partially deployed

### 5.2 Post-release visibility (Phase G/H-style evidence)
- **Input:** Domain facts + process facts + deployment evidence.
- **Output:** Portal/read model shows: what shipped, what’s approved, what’s running, evidence chain, run timeline; optional post-release evidence can be captured as evidence elements/attachments linked from the change element (e.g., `evidence:adm_post_release`).
- **Next:** **Trigger** workflow completion + close change element

### Phase 5 — Implementation checklist

- [ ] **Domain**: can record a release outcome (as an element/attachment + reference relationships) linked to the change.
- [ ] **Projection/UISurface**: can show “what shipped” + evidence chain + timelines in one view.
- [ ] **ADM Phase H posture**: follow-on changes can be represented as new change elements linked to prior releases/baselines.
