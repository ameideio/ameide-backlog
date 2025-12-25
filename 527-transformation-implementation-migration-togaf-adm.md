# 527 Transformation — Implementation Plan (TOGAF ADM overlay)

**Status:** Draft  
**Parent:** `backlog/527-transformation-implementation-migration.md`  
**Scenario:** `backlog/527-transformation-scenario-togaf-adm.md`

This document tracks the **TOGAF ADM methodology overlay** implementation: ADM-native UISurface views, checks/validators (optional validation packs), and ProcessDefinitions that operate over the same canonical **Elements + Relationships + Versions** substrate.

The overlay is considered “done” when an ADM-driven workflow can assemble phase deliverables as elements under a deliverables package, run verification checks, and promote/release using the simple handshake (pins as named links; see `backlog/527-transformation-methodology-dictionary.md`).

---

## Layer header (Implementation & Migration)

- **Primary ArchiMate element types used:** Work Package, Deliverable, Plateau, Gap.
- **Goal:** ship an ADM overlay that is data-driven (definitions + elements), not CRD-driven.

---

## 0) Non-goals

- No separate “ADM canonical storage model” that forks design-time truth away from Elements.
- No attempt to fully encode the entire TOGAF spec; v1 focuses on Phase A–D driven delivery plus governance gates and evidence.
- No ADM semantics embedded in Kubernetes CRDs/operators.

---

## 1) Overlay deliverables (what must exist)

### 1.1 Minimal v0 handshake (pins as links)

- [ ] Change element type convention (e.g., `ameide:change`) is supported by UI/workflows.
- [ ] Change element stores `methodology_key = togaf_adm`.
- [ ] Change element uses named links (“pins”):
  - [ ] `ref:requirement` → authoritative requirement `{element_id, version_id}`
  - [ ] `ref:deliverables_root` → deliverables package/root `{element_id, version_id}`
  - [ ] optional: `evidence:*`, `ref:baseline`, `ref:release`

### 1.2 Elements (ADM deliverables as “just elements”)

- [ ] Deliverables package pattern is established:
  - a single “deliverables root” element
  - child elements (or references) representing Phase A/B/C/D deliverables

### 1.3 UISurface (ADM-native UX)

- [ ] ADM overlay UI exists (either a dedicated UISurface deployment or a profile-driven mode in one UISurface):
  - Phase A–D navigation and templates
  - deliverables completeness status per phase
  - conformance/validation findings and evidence viewer
  - promotion/release view (pins + evidence chain)

### 1.4 Optional v1 (configurable overlays via Definition Registry)

- [ ] `UIProfileDefinition` for ADM UI (Phase A–D workspace navigation + deliverables templates).
- [ ] `ValidationPackDefinition`: Phase A–D completeness + conformance.
- [ ] `GatePolicyDefinition`: Architecture Board / phase gate policy (approvers + required evidence).
- [ ] `MethodologyProfileDefinition` for TOGAF ADM (wires UI profile + validators + workflow).
- [ ] `ProcessDefinition` (BPMN) for R2R governance (TOGAF ADM): `r2r.governance.togaf_adm.v1`.

---

## 2) Work packages (incremental)

### WP-A1 — Deliverables package template + UI scaffolding

- [ ] UI can create `ref:deliverables_root` and scaffold Phase A–D placeholders as elements.
- [ ] Projection can render “phase completeness” from pinned refs + relationships + validation evidence.

### WP-A2 — Phase A–D checks/validators

- [ ] Each phase validator is deterministic and WorkRequest-executable.
- [ ] Validators reference notation profile conformance rules when applicable (standards-compliant vs extended).

### WP-A3 — ADM workflow definition (ProcessDefinition)

- [ ] BPMN authoring pattern for ADM overlay is locked.
- [ ] Workflow emits process facts for each transition/gate and cites pinned refs.

### WP-A4 — Governance gates + promotion wiring

- [ ] Phase gate decisions are recorded as evidence and reflected in projection timelines.
- [ ] Promotion gate requires required phase evidence + pinned baseline refs.

### WP-A5 — “Happy path” e2e slice

- [ ] Scenario passes end-to-end per `backlog/527-transformation-scenario-togaf-adm.md`:
  - requirement pinned
  - deliverables root pinned + Phase A–D complete
  - verify pass
  - baseline promoted → release recorded (if in-scope)
