# 527 — Transformation Scenario (Scrum track; flow-only)

**Status:** Draft  
**Parent:** `backlog/527-transformation-capability.md`  
**Methodology overlay:** Scrum (dedicated UISurface + workflow over the same element graph)
**ProcessDefinition ID (recommended):** `r2r.governance.scrum.v1`

This is a **flow-only** view of R2R (“Requirement → Release”) when the change is governed by a **Scrum** overlay.

The path is Scrum-native (refinement → DoR → iterate → DoD), but all design-time truth is stored as **Elements + Relationships + Versions**.

## Legend (used in “Next”)

- **Trigger**: start the next action immediately
- **Wait**: block until a signal/fact/status is observed
- **Skip**: conditional bypass (step is not applicable)
- **Loop**: return to an earlier step
- **End**: terminate the workflow

---

## Phase 1 — Initiate (intake + articulation + DoR)

### 1.1 Capture change element (raw intake)
- **Input:** Raw ask (often incomplete); target capability; initial “agentic decision” flag if known.
- **Output:** Change element created with canonical scope `{tenant_id, organization_id, repository_id}` and `methodology_key = scrum` (stored on the change element body).
- **Next:** **Trigger** 1.2

### 1.2 Requirement articulation (agent-assisted; iterative)
- **Input:** Change element; stakeholder context; existing repository content/definitions/contracts.
- **Output:** Requirement draft element(s) (docs/views) linked from the change element; drafts include problem, non-goals, examples, acceptance criteria, glossary/definitions, risks/governance needs.
- **Next:** If still ambiguous → **Loop** 1.2; else **Trigger** 1.3

### 1.3 Record requirement status (Domain fact; anchored element version)
- **Input:** Latest requirement draft element version(s).
- **Output:** Requirement status recorded as a Domain fact on the change element (e.g., `DRAFT | NEEDS_INPUT | STABILIZED | CANCELED`) and the anchor reference relationship is updated: `ref:requirement` (`relationship_kind=REFERENCE`, `metadata.target_version_id=<requirement_version_id>`).
- **Next:** If `STABILIZED` → **Trigger** 1.4; if `CANCELED` → **End**; else **Loop** 1.2

### 1.4 DoR validation (Scrum overlay)
- **Input:** `ref:requirement` + Scrum checklist/rules.
- **Output:** DoR outcome recorded as evidence (e.g., an evidence element or attachment) linked from the change element (e.g., `evidence:dor`).
- **Next:** If DoR fails → **Loop** 1.2; else **Trigger** 1.5

### 1.5 Start delivery workflow (only after DoR)
- **Input:** Change element id/scope + anchored stabilized requirement ref(s) + DoR evidence ref(s).
- **Output:** R2R workflow instance started (using `r2r.governance.scrum.v1`); “requirements captured” process fact emitted.
- **Next:** **Trigger** Phase 2

---

## Phase 2 — Design (lightweight architecture + contracts + definitions)

### 2.1 Produce architecture package (Scrum-appropriate)
- **Input:** Requirement snapshot; existing repository content.
- **Output:** Deliverables package element created/updated and anchored via `ref:deliverables_root` (`relationship_kind=REFERENCE`, `metadata.target_version_id=<deliverables_root_version_id>`); package contains lightweight architecture elements (views/docs) sufficient to constrain the change.
- **Next:** **Trigger** 2.2

### 2.2 Define contract + message boundary
- **Input:** Requirement snapshot + architecture package; current capability protos.
- **Output:** Proto diff spec (new fields/messages; intent/fact boundary) and derived checklist (touched primitives, migrations/tests).
- **Next:** **Trigger** 2.3

### 2.3 Update capability BPMN model (optional)
- **Input:** Capability workflow model (if it exists).
- **Output:** New versioned BPMN element reflecting the change (tasks/decisions/message boundaries).
- **Next:** If BPMN not used → **Skip** 2.3 and **Trigger** 2.4; else if model-only → **Trigger** 2.4; else if executable → **Trigger** 2.3a

### 2.3a Bind executable BPMN (only if executable)
- **Input:** BPMN model + bindings (send intents / await facts / correlation rules).
- **Output:** Promotable BPMN execution profile definition (recommended as an `ExtensionDefinition`).
- **Next:** **Trigger** 2.4

### 2.4 Define agent scope (only if agentic)
- **Input:** “Agentic decision” flag; desired tool grants; risk tier.
- **Output:** AgentDefinition diff recorded as a promotable definition.
- **Next:** If no agentic component → **Skip** 2.4; **Trigger** 2.5

### 2.5 Design gate (Scrum)
- **Input:** Proposed diffs: architecture package refs + proto + (optional) BPMN + (optional) BPMN profile + (optional) AgentDefinition.
- **Output:** Decision recorded (approve/reject) as workflow signal + process facts; anchored design artifact refs remain the canonical inputs to Phase 3.
- **Next:** **Wait** gate decision; if Approve → **Trigger** Phase 3; if Reject → **Loop** 2.1; if Cancel → **End**

---

## Phase 3 — Realize (sprint execution + tool runs + verification)

### 3.1 Iteration planning (optional; Scrum overlay)
- **Input:** Approved design artifacts + organizational cadence.
- **Output:** Optional planning artifacts captured as elements (notes, plan docs) and linked from the change element.
- **Next:** If no iteration cadence → **Skip** 3.1; else **Trigger** 3.2

### 3.2 Request scaffold/codegen tool run
- **Input:** Accepted design diffs; repo coordinates; idempotency key; `action_kind=scaffold|generate`.
- **Output:** Domain WorkRequest created; `WorkRequested` emitted to work queue (e.g., `transformation.work.queue.toolrun.generate.v1`).
- **Next:** **Trigger** Integration executor job; **Wait** 3.2a

### 3.2a Tool run outcome recorded
- **Input:** WorkRequest id.
- **Output:** Evidence bundle + terminal status recorded in Domain; Process emits `ToolRunRecorded` referencing evidence.
- **Next:** If success → **Trigger** 3.3; if failure → **Loop** 3.2 or **Wait** for repair then re-**Trigger** 3.2

### 3.3 Implement repo changes (agent or human)
- **Input:** Scaffolded repo state + accepted diffs.
- **Output:** PR/branch with: proto changes, regenerated outputs, capability domain/process/agent/projection/UI updates, tests.
- **Next:** **Trigger** 3.4

### 3.4 Request verification tool run
- **Input:** PR ref; `action_kind=verify`; verification baseline.
- **Output:** Domain WorkRequest created; `WorkRequested` emitted (e.g., `transformation.work.queue.toolrun.verify.v1`).
- **Next:** **Trigger** Integration verifier job; **Wait** 3.4a

### 3.4a Verification outcome recorded
- **Input:** WorkRequest id.
- **Output:** Verification report + evidence bundle; pass/fail status.
- **Next:** If pass → **Trigger** Phase 4; if fail → **Loop** 3.3 (fix) or if “proto breaking” → **Loop** 2.2 (redesign)

---

## Phase 4 — Promote (governed release decision)

### 4.1 Promotion gate
- **Input:** PR contents + verification evidence.
- **Output:** Gate request for promoting: definitions (Process/Agent/Extension), and release baseline; decision recorded as evidence.
- **Next:** **Wait** Approve/Reject/Override/Cancel; if Approve/Override → **Trigger** 4.2; if Reject → **Loop** Phase 3; if Cancel → **End**

### 4.2 Record promotions (system-of-record)
- **Input:** Approved promotion decisions.
- **Output:** Domain records promoted versions/baseline; promotion facts emitted; projection reflects audit trail.
- **Next:** **Trigger** Phase 5

---

## Phase 5 — Release (merge + deploy + DoD visibility)

### 5.1 Merge + CI build + deploy
- **Input:** Approved PR + promoted definitions/baseline.
- **Output:** CI pipeline run + deploy execution; deployment evidence captured.
- **Next:** **Wait** CI/deploy status; if fail → **Loop** Phase 3 (fix) or **Trigger** rollback/disable if already partially deployed

### 5.2 DoD validation + close-out
- **Input:** Domain facts + process facts + deployment evidence + anchored deliverables.
- **Output:** DoD outcome recorded as evidence linked from the change element (e.g., `evidence:dod`); portal/read model shows evidence chain and run timeline.
- **Next:** **Trigger** workflow completion + close change element
