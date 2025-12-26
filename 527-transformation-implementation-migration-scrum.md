# 527 Transformation — Implementation Plan (Scrum overlay)

**Status:** Draft  
**Parent:** `backlog/527-transformation-implementation-migration.md`  
**Scenario:** `backlog/527-transformation-scenario-scrum.md`

This document tracks the **Scrum methodology overlay** implementation: Scrum-native UISurface views, checks/validators (optional validation packs), and ProcessDefinitions that operate over the same canonical **Elements + Relationships + Versions** substrate.

The overlay is considered “done” when a Scrum-driven workflow can create/update the required elements, run verification WorkRequests, and promote/release using the simple handshake (anchor `REFERENCE` relationships with `metadata.target_version_id`; see `backlog/527-transformation-methodology-dictionary.md`).

---

## Layer header (Implementation & Migration)

- **Primary ArchiMate element types used:** Work Package, Deliverable, Plateau, Gap.
- **Goal:** ship a Scrum overlay that is data-driven (definitions + elements), not CRD-driven.

---

## 0) Non-goals

- No separate “Scrum canonical storage model” that forks design-time truth away from Elements.
- No Scrum methodology semantics encoded in Kubernetes CRDs/operators.
- No attempt to fully replicate Jira/Azure DevOps; v1 is “enough to govern Requirement → Release”.

---

## 0.1) Implementation progress (code snapshot; preserve context)

Implemented in code (Dec 2025):

- [x] **Versioned anchor relationships**: Domain accepts `relationship_kind=REFERENCE` relationships with `metadata.target_version_id` and validates that the target version exists for the target element.
- [x] **R2R governance workflow (Scrum)**: Process primitive has a v0 Temporal workflow that:
  - records requirement stabilization status on the change element (v0: `Element.lifecycle_state`),
  - creates/updates `ref:requirement` and `ref:deliverables_root` versioned reference relationships,
  - scaffolds deliverables membership via `CONTAINMENT` relationships (e.g., `contains:deliverable`),
  - requests one scaffold WorkRequest (`ActionKind=SCAFFOLD`, routed to `transformation.work.queue.toolrun.generate.v1`),
  - requests two verify WorkRequests (`dor.verify`, `dod.verify`),
  - records evidence as elements linked via versioned `ref:evidence:*` relationships, and
  - records a release element linked via `ref:release`, then promotes a governance baseline.
- [x] **E2E tests**: capability pack runs the Scrum R2R flow end-to-end in repo-mode and cluster-mode under the same test implementation (430 posture).

Still pending (target state):

- [ ] Scrum-native UISurface experience for refinement/DoR/DoD.
- [ ] Stored `ProcessDefinition` (BPMN) `r2r.governance.scrum.v1` executed from the Definition Registry (v0 workflow is hard-coded).
- [ ] DoR/DoD as DefinitionRegistry-driven ValidationPack + GatePolicy definitions (optional v1).

## 1) Overlay deliverables (what must exist)

### 1.1 Minimal v0 handshake (anchor reference relationships)

- [ ] Change element type convention (e.g., `ameide:change`) is supported by UI/workflows.
- [ ] Change element stores `methodology_key = scrum`.
- [ ] Change element uses anchor `REFERENCE` relationships (versioned via `metadata.target_version_id`):
- [ ] Change element uses anchor `REFERENCE` relationships (versioned via `metadata.target_version_id`):
  - [ ] `ref:requirement` → authoritative requirement snapshot (`target_element_id` + `metadata.target_version_id`)
  - [ ] `ref:deliverables_root` → deliverables package/root snapshot (`target_element_id` + `metadata.target_version_id`)
  - [ ] optional: `ref:evidence:*`, `ref:baseline`, `ref:release`

### 1.2 UISurface (Scrum-native UX)

- [ ] Scrum overlay UI exists (either a dedicated UISurface deployment or a profile-driven mode in one UISurface):
  - backlog/refinement workspace
  - DoR outcome + missing deliverables guidance (recorded as evidence links, not a special bundle type)
  - DoD outcome + evidence viewer
  - promotion/release view (anchor relationships + evidence chain)

### 1.3 Optional v1 (configurable overlays via Definition Registry)

- [ ] `UIProfileDefinition` for Scrum UI (backlog/refinement/DoR/DoD flows).
- [ ] `ValidationPackDefinition`: DoR and DoD (deterministic; WorkRequest-executable).
- [ ] `GatePolicyDefinition`: Scrum gates (approvers + required evidence).
- [ ] `MethodologyProfileDefinition` for Scrum (wires UI profile + validators + workflow).
- [ ] `ProcessDefinition` (BPMN) for R2R governance (Scrum): `r2r.governance.scrum.v1`.

---

## 2) Work packages (incremental)

### WP-S1 — “Change element” support in UI + Domain

- [ ] UI can create a change element, set `methodology_key`, and maintain the anchor reference relationships (`ref:requirement`, `ref:deliverables_root`, with `metadata.target_version_id`).
- [ ] Projection exposes anchor relationships + status in a stable query surface.

### WP-S2 — DoR validation pack

- [ ] Implement DoR checks (start simple: record pass/fail evidence linked from the change element).
- [ ] Optional: implement deterministic validators as WorkRequests.

### WP-S3 — Scrum workflow definition (ProcessDefinition)

- [ ] BPMN authoring pattern for Scrum overlay is locked.
- [ ] Workflow emits process facts for each transition/gate and cites anchored references.

### WP-S4 — DoD validation pack + promotion gate wiring

- [ ] DoD checks block promotion on failure (evidence is linked from the change element).
- [ ] Promotion can be represented by `ref:baseline` (or baseline tables) and a `ref:release` link when shipped.

### WP-S5 — “Happy path” e2e slice

- [x] Scenario passes end-to-end (Process + Domain + executor; UISurface pending) per `backlog/527-transformation-scenario-scrum.md`:
  - requirement anchored (reference relationship + `metadata.target_version_id`) → DoR pass
  - deliverables package anchored (reference relationship + `metadata.target_version_id`) → verify pass
  - baseline promoted → release recorded
