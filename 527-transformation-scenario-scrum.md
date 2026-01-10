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

## Prerequisites (implementation)

These are the minimum implementation capabilities required for this scenario to run end-to-end.

- [ ] **Domain (Enterprise Knowledge)** supports Elements/Versions/Relationships writes for `{tenant_id, organization_id, repository_id}`.
- [ ] **REFERENCE relationships are versioned**: governance anchors (e.g., `ref:requirement`, `ref:deliverables_root`) include `metadata.target_version_id` (no “latest version” ambiguity).
- [ ] **Projection** can read and render:
  - [ ] change element + its outgoing `ref:*` relationships (including `metadata.target_version_id`)
  - [ ] requirement status (as domain facts or a derived read model)
  - [ ] deliverables root membership (relationships under the deliverables root)
  - [ ] evidence links as versioned `ref:evidence:*` relationships (e.g., `ref:evidence:dor`, `ref:evidence:dod`)
- [ ] **WorkRequests substrate** works end-to-end:
  - [ ] Domain records WorkRequests and emits `transformation.work.domain.facts.v1`
  - [ ] executor processes tool-run queues and records outcomes + evidence back to Domain
  - [ ] Process emits `ToolRunRecorded` and step transitions on `transformation.process.facts.v1`
- [ ] **UISurface** can create/edit:
  - [ ] a change element (`methodology_key = scrum`)
  - [ ] requirement draft element(s) and snapshots (ElementVersions)
  - [ ] anchor relationships (`ref:requirement`, `ref:deliverables_root`) with `metadata.target_version_id`
- [ ] **Cluster E2E harness (non-agentic) exists** (optional gate; required for UI-affecting changes):
  - [ ] BuildKit build path exists in-cluster (no Docker-in-Docker requirement)
  - [ ] Gateway API overlay routing exists (Envoy Gateway): runner can create/delete `HTTPRoute` rules that match a run header (recommended `X-Ameide-Run-Key=<nonce>`) and route to shadow Services while stable URLs remain unchanged
  - [ ] Harness fails closed unless it can prove routing correctness:
    - [ ] waits for `HTTPRoute` status `Accepted=True` and `ResolvedRefs=True` (or controller equivalent)
    - [ ] verifies an overlay-only marker (preferred: response header modifier) is present with the run header and absent without it
  - [ ] Harness uses a run anchor object (`ConfigMap wut-run-<work_request_id>`) + `ownerReferences` for deterministic cleanup, with a TTL janitor sweep as a safety net
  - [ ] Playwright E2E runner can hit the stable URL and inject the run header on every request (artifacts under `/artifacts/e2e/*` per 430)

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
- **Output:** Requirement status recorded on the change element (v0: `Element.lifecycle_state`, e.g., `DRAFT | NEEDS_INPUT | STABILIZED | CANCELED`) and the anchor reference relationship is updated: `ref:requirement` (`relationship_kind=REFERENCE`, `metadata.target_version_id=<requirement_version_id>`).
- **Next:** If `STABILIZED` → **Trigger** 1.4; if `CANCELED` → **End**; else **Loop** 1.2

### 1.4 DoR validation (Scrum overlay)
- **Input:** `ref:requirement` + Scrum checklist/rules.
- **Output:** DoR outcome recorded as evidence (e.g., an evidence element/version) linked from the change element (e.g., `ref:evidence:dor`).
- **Next:** If DoR fails → **Loop** 1.2; else **Trigger** 1.5

### 1.5 Start delivery workflow (only after DoR)
- **Input:** Change element id/scope + anchored stabilized requirement ref(s) + DoR evidence ref(s).
- **Output:** R2R workflow instance started (using `r2r.governance.scrum.v1`); “requirements captured” process fact emitted.
- **Next:** **Trigger** Phase 2

### Phase 1 — Implementation checklist

- [ ] **Domain**: requirement status fact/field exists (`DRAFT | NEEDS_INPUT | STABILIZED | CANCELED`) and is queryable.
- [ ] **Domain**: `ref:requirement` relationship stores `metadata.target_version_id` and is updatable idempotently.
- [ ] **UISurface**: supports iterative requirement articulation (draft → snapshot → stabilize) and records DoR evidence (manual evidence element/attachment is fine for v0).
- [ ] **Projection**: exposes “requirement stabilized” and the anchored requirement snapshot on the change view.

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
- **Input:** BPMN model + bindings (Activity types/policies/IO mappings; explicit waits only when an external message/timer is the intended control-flow surface).
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

### Phase 2 — Implementation checklist

- [ ] **Domain**: supports a deliverables package/root element and versioning it (ElementVersion snapshots).
- [ ] **Domain**: `ref:deliverables_root` relationship stores `metadata.target_version_id`.
- [ ] **UISurface**: can assemble the deliverables root contents (views/docs/elements) and maintain the root’s membership links deterministically.
- [ ] **Projection**: can render the deliverables root and its members for review/gates.
- [ ] **(Optional)** BPMN authoring/binding surfaces exist if executable workflows are in-scope for this slice.

---

## Phase 3 — Realize (sprint execution + tool runs + verification)

### 3.1 Iteration planning (optional; Scrum overlay)
- **Input:** Approved design artifacts + organizational cadence.
- **Output:** Optional planning artifacts captured as elements (notes, plan docs) and linked from the change element.
- **Next:** If no iteration cadence → **Skip** 3.1; else **Trigger** 3.2

### 3.2 Request scaffold/codegen tool run
- **Input:** Accepted design diffs; repo coordinates; idempotency key; `action_kind=scaffold|generate`.
- **Output:** Domain WorkRequest created; `WorkExecutionRequested` execution intent emitted to the execution queue topic (e.g., `transformation.work.queue.toolrun.generate.v1`); `WorkRequested` fact emitted to `transformation.work.domain.facts.v1`.
- **Next:** **Trigger** Integration executor job; **Wait** 3.2a

### 3.2a Tool run outcome recorded
- **Input:** WorkRequest id.
- **Output:** Evidence bundle + terminal status recorded in Domain; Process emits `ToolRunRecorded` referencing evidence.
- **Next:** If success → **Trigger** 3.3; if failure → **Loop** 3.2 or **Wait** for repair then re-**Trigger** 3.2

### 3.3 Request Dev WorkRequest (agentic; Codex mocked; creates PR)
- **Input:** Accepted diffs + task prompt + repo ref.
- **Output:** Domain WorkRequest created:
  - `work_kind=agent_work`
  - `action_kind=publish` (agent produces and publishes a PR as the outcome)
- **Next:** **Trigger** agent-work runner job; **Wait** 3.3a

### 3.3a Dev outcome recorded (PR created)
- **Input:** Dev WorkRequest id.
- **Output:** PR metadata recorded as WorkRequest result outputs + evidence:
  - `outputs.pr_url`
  - `outputs.commit_sha` (head SHA for verification)
- **Next:** If success → **Trigger** 3.4; else **Loop** 3.3 (revise prompt / fix runner)

### 3.4 Request verification tool run
- **Input:** `outputs.commit_sha`; `action_kind=verify`; verification baseline.
- **Output:** Domain WorkRequest created; `WorkExecutionRequested` execution intent emitted to the execution queue topic (e.g., `transformation.work.queue.toolrun.verify.v1`); `WorkRequested` fact emitted to `transformation.work.domain.facts.v1`.
- **Next:** **Trigger** Integration verifier job; **Wait** 3.4a

### 3.4a Verification outcome recorded
- **Input:** WorkRequest id.
- **Output:** Verification report + evidence bundle; pass/fail status.
- **Next:** If pass → **Trigger** 3.5 (when E2E gate is required) else **Trigger** Phase 4; if fail → **Loop** 3.3 (fix) or if “proto breaking” → **Loop** 2.2 (redesign)

### 3.5 Automatically request cluster UI harness verification (stable URLs; gateway overlay)
- **Input:** Verified PR ref + stable base URL + service selection manifest.
- **Output:** Domain WorkRequest created with:
  - `action_kind=verify`
  - `verification_suite_ref=transformation.verify.ui_harness.gateway_overlay.v1`
  - recommended queue: `transformation.work.queue.toolrun.verify.ui_harness.v1`
- **Next:** If the manifest indicates no `edge_routable` services are affected → **Skip** 3.5 and **Trigger** Phase 4; else **Trigger** E2E runner job; **Wait** 3.5a

### 3.5a E2E outcome recorded
- **Input:** E2E WorkRequest id.
- **Output:** Playwright artifacts + pass/fail evidence recorded in Domain and cited by Process facts.
- **Next:** If pass → **Trigger** Phase 4; if fail → **Loop** 3.3 (fix) or **Loop** 2.1/2.2 (redesign) depending on root cause

### Phase 3 — Implementation checklist

- [ ] **Domain**: WorkRequest lifecycle is persisted and emits `WorkRequested/Started/Completed/Failed` facts.
- [ ] **Integration runner**: can execute `scaffold|generate|verify` deterministically for a given `commit_sha` and record evidence (logs + artifacts).
- [ ] **Integration runner (E2E)**: can build/run shadow services in-cluster, apply Gateway API overlay routes, run Playwright against stable URLs with a run header, and record artifacts under `/artifacts/e2e/*`.
- [ ] **Process**: emits step-level process facts and correlates to WorkRequests (so Projection can build a run timeline).
- [ ] **Projection/UISurface**: can show WorkRequest status + evidence links and the process run timeline.

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

### Phase 4 — Implementation checklist

- [ ] **Domain**: baselines exist as immutable `{element_id, version_id}` sets and can be promoted.
- [ ] **Projection**: can render baseline membership and promotion history for audit.
- [ ] **Process/UISurface**: gate decisions are recorded and visible (approve/reject/override/cancel).

---

## Phase 5 — Release (merge + deploy + DoD visibility)

### 5.1 Merge + CI build + deploy
- **Input:** Approved PR + promoted definitions/baseline.
- **Output:** CI pipeline run + deploy execution; deployment evidence captured.
- **Next:** **Wait** CI/deploy status; if fail → **Loop** Phase 3 (fix) or **Trigger** rollback/disable if already partially deployed

### 5.2 DoD validation + close-out
- **Input:** Domain facts + process facts + deployment evidence + anchored deliverables.
- **Output:** DoD outcome recorded as evidence linked from the change element (e.g., `ref:evidence:dod`); portal/read model shows evidence chain and run timeline.
- **Next:** **Trigger** workflow completion + close change element

### Phase 5 — Implementation checklist

- [ ] **Domain**: can record a release outcome (as an element/attachment + reference relationships) linked to the change.
- [ ] **Projection/UISurface**: can show “what shipped” + evidence chain + timelines in one view.
- [ ] **DoD evidence**: captured deterministically (even if it is initially manual evidence attachments for v0).
