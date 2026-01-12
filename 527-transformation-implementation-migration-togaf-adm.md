# 527 Transformation — Implementation Plan (TOGAF ADM overlay)

> **DEPRECATED (2026-01-12):** Superseded by the Zeebe-based Process primitive posture.  
> See `backlog/527-transformation-process-v2.md`.

**Status:** Draft  
**Parent:** `backlog/527-transformation-implementation-migration.md`  
**Scenario:** `backlog/527-transformation-scenario-togaf-adm.md`

This document tracks the **TOGAF ADM methodology overlay** implementation: ADM-native UISurface views, checks/validators (optional validation packs), and ProcessDefinitions that operate over the same canonical **Elements + Relationships + Versions** substrate.

The overlay is considered “done” when an ADM-driven workflow can assemble phase deliverables as elements under a deliverables package, run verification checks, and promote/release using the simple handshake (anchor `REFERENCE` relationships with `metadata.target_version_id`; see `backlog/527-transformation-methodology-dictionary.md`).

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

## 0.1) Implementation progress (code snapshot; preserve context)

Implemented in code (Dec 2025):

- [x] **Versioned anchor relationships**: Domain accepts `relationship_kind=REFERENCE` relationships with `metadata.target_version_id` and validates that the target version exists for the target element.
- [x] **R2R governance workflow (TOGAF ADM)**: Process primitive has a v0 Temporal workflow that:
  - records requirement stabilization status on the change element (v0: `Element.lifecycle_state`),
  - creates/updates `ref:requirement` and `ref:deliverables_root` versioned reference relationships,
  - scaffolds Phase A and B/C/D deliverables as elements and links them under the deliverables root via `CONTAINMENT` relationships (e.g., `contains:deliverable`),
  - requests one scaffold WorkRequest (`ActionKind=SCAFFOLD`, routed to `transformation.work.queue.toolrun.generate.v1`),
  - requests two verify WorkRequests (`phase_a.verify`, `phase_bcd.verify`),
  - records evidence as elements linked via versioned `ref:evidence:*` relationships, and
  - records a release element linked via `ref:release`, then promotes a governance baseline.
- [x] **Headless E2E tests (governance seam)**: capability pack runs the TOGAF ADM R2R flow end-to-end in repo-mode and cluster-mode under the same test implementation (430 posture).
- [x] **UI E2E harness (Playwright; stable URLs)**: WorkRequest-driven E2E harness exists in code and uses BuildKit + Gateway API header overlays (no Telepresence) to route only test traffic to shadow services and capture artifacts under `/artifacts/e2e/*` (per `backlog/430-unified-test-infrastructure-status.md`).
  - Operational note: this is “real” only when GitOps wiring exists (topic + runner RBAC + ScaledJob), the gateway supports `HTTPRoute` header matches + response header filters, and the E2E persona secret is configured.

Still pending (target state):

- [ ] ADM-native UISurface experience for Phase A–D deliverables.
- [ ] Stored `ProcessDefinition` (BPMN) `r2r.governance.togaf_adm.v1` executed from the Definition Registry (v0 workflow is hard-coded).
- [ ] Phase A–D validators and Architecture Board gates as DefinitionRegistry-driven definitions (optional v1).

## 1) Overlay deliverables (what must exist)

### 1.1 Minimal v0 handshake (anchor reference relationships)

- [ ] Change element type convention (e.g., `ameide:change`) is supported by UI/workflows.
- [ ] Change element stores `methodology_key = togaf_adm`.
- [ ] Change element uses anchor `REFERENCE` relationships (versioned via `metadata.target_version_id`):
  - [ ] `ref:requirement` → authoritative requirement snapshot (`target_element_id` + `metadata.target_version_id`)
  - [ ] `ref:deliverables_root` → deliverables package/root snapshot (`target_element_id` + `metadata.target_version_id`)
  - [ ] optional: `ref:evidence:*`, `ref:baseline`, `ref:release`

### 1.2 Elements (ADM deliverables as “just elements”)

- [ ] Deliverables package pattern is established:
  - a single “deliverables root” element
  - child elements (or references) representing Phase A/B/C/D deliverables

### 1.3 UISurface (ADM-native UX)

- [ ] ADM overlay UI exists (either a dedicated UISurface deployment or a profile-driven mode in one UISurface):
  - Phase A–D navigation and templates
  - deliverables completeness status per phase
  - conformance/validation findings and evidence viewer
  - promotion/release view (anchor relationships + evidence chain)

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
- [ ] Projection can render “phase completeness” from anchored references + relationships + validation evidence.

### WP-A2 — Phase A–D checks/validators

- [ ] Each phase validator is deterministic and WorkRequest-executable.
- [ ] Validators reference notation profile conformance rules when applicable (standards-compliant vs extended).

### WP-A3 — ADM workflow definition (ProcessDefinition)

- [ ] BPMN authoring pattern for ADM overlay is locked.
- [ ] Workflow emits process facts for each transition/gate and cites anchored references.

### WP-A4 — Governance gates + promotion wiring

- [ ] Phase gate decisions are recorded as evidence and reflected in projection timelines.
- [ ] Promotion gate requires required phase evidence + baseline references (immutable `{element_id, version_id}` sets).

### WP-A5 — “Happy path” e2e slice

- [x] Scenario passes end-to-end (Process + Domain + executor; UISurface pending) per `backlog/527-transformation-scenario-togaf-adm.md`:
  - requirement anchored (reference relationship + `metadata.target_version_id`)
  - deliverables root anchored (reference relationship + `metadata.target_version_id`) + Phase A–D complete
  - verify pass
  - baseline promoted → release recorded (if in-scope)
