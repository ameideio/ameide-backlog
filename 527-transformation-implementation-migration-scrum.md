# 527 Transformation — Implementation Plan (Scrum overlay)

**Status:** Draft  
**Parent:** `backlog/527-transformation-implementation-migration.md`  
**Scenario:** `backlog/527-transformation-scenario-scrum.md`

This document tracks the **Scrum methodology overlay** implementation: Scrum-native UISurface views, checks/validators (optional validation packs), and ProcessDefinitions that operate over the same canonical **Elements + Relationships + Versions** substrate.

The overlay is considered “done” when a Scrum-driven workflow can create/update the required elements, run verification WorkRequests, and promote/release using the simple handshake (pins as named links; see `backlog/527-transformation-methodology-dictionary.md`).

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

## 1) Overlay deliverables (what must exist)

### 1.1 Minimal v0 handshake (pins as links)

- [ ] Change element type convention (e.g., `ameide:change`) is supported by UI/workflows.
- [ ] Change element stores `methodology_key = scrum`.
- [ ] Change element uses named links (“pins”):
  - [ ] `ref:requirement` → authoritative requirement `{element_id, version_id}`
  - [ ] `ref:deliverables_root` → deliverables package/root `{element_id, version_id}`
  - [ ] optional: `evidence:*`, `ref:baseline`, `ref:release`

### 1.2 UISurface (Scrum-native UX)

- [ ] Scrum overlay UI exists (either a dedicated UISurface deployment or a profile-driven mode in one UISurface):
  - backlog/refinement workspace
  - DoR outcome + missing deliverables guidance (recorded as evidence links, not a special bundle type)
  - DoD outcome + evidence viewer
  - promotion/release view (pins + evidence chain)

### 1.3 Optional v1 (configurable overlays via Definition Registry)

- [ ] `UIProfileDefinition` for Scrum UI (backlog/refinement/DoR/DoD flows).
- [ ] `ValidationPackDefinition`: DoR and DoD (deterministic; WorkRequest-executable).
- [ ] `GatePolicyDefinition`: Scrum gates (approvers + required evidence).
- [ ] `MethodologyProfileDefinition` for Scrum (wires UI profile + validators + workflow).
- [ ] `ProcessDefinition` (BPMN) for R2R governance (Scrum): `r2r.governance.scrum.v1`.

---

## 2) Work packages (incremental)

### WP-S1 — “Change element” support in UI + Domain

- [ ] UI can create a change element, set `methodology_key`, and maintain the pin links (`ref:requirement`, `ref:deliverables_root`).
- [ ] Projection exposes pin links + status in a stable query surface.

### WP-S2 — DoR validation pack

- [ ] Implement DoR checks (start simple: record pass/fail evidence linked from the change element).
- [ ] Optional: implement deterministic validators as WorkRequests.

### WP-S3 — Scrum workflow definition (ProcessDefinition)

- [ ] BPMN authoring pattern for Scrum overlay is locked.
- [ ] Workflow emits process facts for each transition/gate and cites pinned refs.

### WP-S4 — DoD validation pack + promotion gate wiring

- [ ] DoD checks block promotion on failure (evidence is linked from the change element).
- [ ] Promotion can be represented by `ref:baseline` (or baseline tables) and a `ref:release` link when shipped.

### WP-S5 — “Happy path” e2e slice

- [ ] Scenario passes end-to-end per `backlog/527-transformation-scenario-scrum.md`:
  - requirement pinned → DoR pass
  - deliverables package pinned → verify pass
  - baseline promoted → release recorded
