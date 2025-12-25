# 527 — Transformation Methodology Dictionary (Canonical → ArchiMate/TOGAF/Scrum/PMI)

**Status:** Draft  
**Parent:** `backlog/527-transformation-capability.md`

This document defines a **cross-methodology dictionary**:

- **Left-most column is canonical**: Ameide/Transformation terms for how the platform stores design-time truth as **Elements + Relationships + Versions** and how workflows reference those artifacts.
- Other columns provide **approximate equivalents** in common methodologies and notations: **ArchiMate**, **TOGAF ADM**, **Scrum**, **PMI**.

Goal: enable **different ProcessDefinitions** (Scrum vs TOGAF ADM vs PMI) to run on different paths while still producing a **harmonized platform handshake**:

- methodology-specific artifacts remain “just elements” in the repository, and
- governance/verification/promotions/releases reference those artifacts by stable `{element_id, version_id}` pointers and evidence bundles.

Operator/CRD boundary:

- Methodology semantics belong in **Elements + workflows + UI + checks/validators** (optionally configured by definitions later). Kubernetes CRDs/operators should only describe runtime wiring (images, env, scaling, permissions), not methodology/business meaning.

Non-goals:

- Not a TOGAF/Scrum/PMI tutorial.
- Not a claim that the methods are identical.
- Not a replacement for proto contracts (protos remain authoritative for runtime semantics).

Related:

- Transformation Domain responsibilities and methodology overlays: `backlog/527-transformation-domain.md`
- Transformation Process (governance workflows): `backlog/527-transformation-process.md`
- Methodology-specific flow-only scenarios:
  - Scrum: `backlog/527-transformation-scenario-scrum.md`
  - TOGAF ADM: `backlog/527-transformation-scenario-togaf-adm.md`
- Proto contract index (what exists vs target): `backlog/527-transformation-proto.md`
- Agent role taxonomy + profile mappings: `backlog/527-transformation-agent.md`
- ArchiMate vocabulary guardrails: `backlog/529-archimate-alignment-470plus.md`
- Legacy mapping (historical docs → target): `backlog/527-transformation-crossreference-303-ontology.md`

---

## Reference relationships contract (harmonized platform handshake)

Regardless of methodology (Scrum vs TOGAF ADM vs PMI), the platform should converge on a tiny set of **stable pointers** that gates/promotions/release consume deterministically (no “search for what counts”).

Recommended anchor: a **change element** (an Element) whose outgoing `REFERENCE` relationships anchor the authoritative snapshots used by governance and workflows.

### Simple posture (recommended): anchors are named relationships

The simplest implementation is to represent anchors as **well-known `ElementRelationship` rows** from the change element.

Minimum anchors (required):

1. `ref:requirement` → authoritative requirement snapshot (`target_element_id` + `metadata.target_version_id`)
2. `ref:deliverables_root` → deliverables package/root snapshot (`target_element_id` + `metadata.target_version_id`)

Optional anchors (add only when needed):

3. `ref:baseline` → `{baseline_id}` (governance baseline; immutable set of `{element_id, version_id}` pointers)
4. `ref:release` → `{release_ref}` (release record element and/or external refs as attachments)
5. `evidence:*` → `{evidence_ref}` (links to evidence elements/attachments; implementation-specific)

Rule: methodology differences determine **what is under `ref:deliverables_root`** and which gates apply, but not how promotions/releases identify authoritative inputs.

Versioning rule (lock-in):

- For `ref:*` anchors, the relationship MUST be `relationship_kind = REFERENCE` and MUST include `metadata.target_version_id`.

---

## ProcessDefinition ID naming (value-stream first)

If R2R (Requirement → Release) is the value stream and Scrum/TOGAF ADM are governance methodologies, prefer **value-stream-first** ProcessDefinition IDs with methodology as a qualifier:

- `r2r.governance.scrum.v1`
- `r2r.governance.togaf_adm.v1`

Selection posture:

- simplest: the change element carries `methodology_key` (`scrum`, `togaf_adm`) and the Process primitive starts the corresponding ProcessDefinition.
- configurable: store/promote a `MethodologyProfileDefinition` that wires the workflow + UI profile + validators, and have the change reference that definition.

---

## Minimal definition set (design-time; methodology as overlays)

The platform can start without any methodology-specific “definition registry types” at all:

- Methodology selection can be a simple `methodology_key` on the change element (e.g., `scrum`, `togaf_adm`).
- Scrum and TOGAF ADM can ship as dedicated UISurfaces and workflows that interpret the same element graph.

When configurability is required (multiple orgs, different gate policies, pluggable validators), introduce design-time definitions (stored/promoted like other definitions) such as:

- `MethodologyProfileDefinition` (wires workflow + UI profile + validation packs + gate policy),
- `ValidationPackDefinition` (deterministic checks),
- `UIProfileDefinition` (UI navigation/templates),
- `GatePolicyDefinition` (approvals/evidence requirements).

## Dictionary (harmonized outputs + workflow evidence)

### 1) Scope + containers

| Canonical (Ameide/Transformation) | ArchiMate (notation) | TOGAF ADM | Scrum | PMI |
|---|---|---|---|---|
| **Scope identity**: `{tenant_id, organization_id, repository_id}` (required everywhere) | N/A (notation-level) | Architecture repository scope / enterprise context | Product context | Project/program context |
| **Repository**: the design-time store for models/docs/definitions (element-centric) | Model repository (ArchiMate model/views) | Architecture Repository (logical concept) | Product artifacts (often tool-specific) | Project repository / PMIS |
| **Initiative**: governance container for a transformation effort | Work Package / Plateau (often used to model change effort) | Architecture work / work package grouping | Epic / Initiative (tool term; not Scrum-normative) | Program/Project/Phase container |
| **Change element**: the tracked unit of change (an Element that anchors deliverables + status) | Work Package / Deliverable (depending on modeling approach) | Architecture Change Request / Request for Architecture Work | PBI / Epic (depending on how you model it) | Change Request / Work Package |

### 2) Requirement articulation (agent-assisted) + readiness

| Canonical (Ameide/Transformation) | ArchiMate (notation) | TOGAF ADM | Scrum | PMI |
|---|---|---|---|---|
| **Requirement draft snapshot**: structured requirement text + examples + glossary + constraints (anchored by reference relationship) | Requirements are usually *external* to ArchiMate; may be represented as Motivation elements (e.g., Requirement/Constraint) if modeled | Architecture Requirements Specification (ARS) (conceptual equivalent) | Story narrative + Acceptance Criteria | Requirements Documentation |
| **Requirement status (Domain fact)**: `DRAFT \| NEEDS_INPUT \| STABILIZED \| CANCELED` | N/A (notation-level) | Requirements managed through ADM; “requirements baseline” is a common practice | Definition of Ready (DoR) / refinement outcome | Requirements baseline / approved requirements |
| **Readiness gate**: explicit recorded decision that the requirement is stable enough to start delivery workflow | N/A | Phase gate / Architecture Board checkpoint (often before committing downstream phases) | DoR gate for pulling into Sprint | Phase gate / integrated change control checkpoint |

### 3) Architecture/design packages (method-dependent, stored as harmonized artifacts)

| Canonical (Ameide/Transformation) | ArchiMate (notation) | TOGAF ADM | Scrum | PMI |
|---|---|---|---|---|
| **Architecture package**: set of versioned Elements/Views/Docs that constrain the change (can be light or deep) | The ArchiMate models/views themselves | ADM deliverables and catalogs/matrices across phases | Often lightweight: just enough design to build; may be ADRs + diagrams | Plans + supporting design docs (varies by org) |
| **Business architecture slice**: what changes in business processes/capabilities/roles | Business layer elements/views | Phase B | May exist as story mapping / impact notes | Scope + process impacts |
| **Information systems architecture slice**: data + application responsibilities/boundaries | Application + Data (Data Object) views | Phase C (Data + Application) | Typically expressed as contract/API changes + ownership notes | Requirements decomposition + solution design |
| **Technology architecture slice**: runtime topology constraints (queues, jobs, env, NFRs) | Technology layer views | Phase D | Usually “build/deploy constraints” in engineering docs | Infrastructure plan / constraints |

### 4) Contracts, definitions, and execution plans (platform-owned)

| Canonical (Ameide/Transformation) | ArchiMate (notation) | TOGAF ADM | Scrum | PMI |
|---|---|---|---|---|
| **Contract package (proto diffs)**: the behavior schema (services/messages/facts) that implements the change | Application Services/Interfaces/Events + Data Objects | Often derived after architecture decisions; not a TOGAF-native artifact but part of solution realization | Interfaces/contracts updated as part of implementation | Requirements → design specifications → implementation artifacts |
| **ProcessDefinition**: versioned definition of a governance/delivery workflow (BPMN as canonical authoring source) | Business Process (model) vs Application-level orchestration are often represented separately; BPMN is a different notation | ADM governance workflow definition (conceptual) | Scrum events/ceremonies are not typically captured as executable workflow definitions | Stage gates / management processes (often documented) |
| **AgentDefinition**: versioned definition of an agent role (tool grants, risk tier, policy) | N/A (notation-level) | “Roles/responsibilities + controls” (conceptual) | Working agreements / team policies (conceptual) | RACI / role definitions + controls (conceptual) |
| **ExtensionDefinition**: versioned auxiliary definition (e.g., BPMN execution profile bindings) | N/A | Architecture building blocks/extensions (conceptual) | Tooling/config conventions (conceptual) | Management plan annexes (conceptual) |

### 5) Evidence, verification, promotion, release

| Canonical (Ameide/Transformation) | ArchiMate (notation) | TOGAF ADM | Scrum | PMI |
|---|---|---|---|---|
| **WorkRequest**: domain-owned record for long-running tool/agent work (scaffold/generate/verify), with idempotency + evidence refs | N/A | Execution task record (conceptual) | CI job/run + checklist (conceptual) | Work performance data / task record (conceptual) |
| **Evidence bundle**: immutable outputs proving what ran and what happened (logs, reports, artifacts), linked from domain state | N/A | Architecture compliance evidence / governance artifacts | DoD evidence (tests, builds, review notes) | Verification/validation evidence; audit artifacts |
| **Verification baseline**: fixed toolchain versions + checks that define “verify” | N/A | Compliance baseline / governance criteria | Definition of Done (DoD) + quality gates | Quality baselines + acceptance criteria |
| **Promotion**: governed decision to “publish” a set of versions/definitions/baseline | Plateau transition / Deliverable acceptance | Phase gate approvals; Architecture Board sign-off (conceptual) | Acceptance of Increment / release readiness | Integrated change control approval |
| **Release record**: what shipped + where + evidence chain | Implementation & Migration artifacts (Deliverable/Plateau) | Implementation Governance (Phase G) + change management (Phase H) | Release/Increment outcomes | Transition/Close + deployment approvals |

---

## Harmonized outputs (what every methodology workflow must produce)

Regardless of whether the ProcessDefinition follows Scrum, TOGAF ADM, or PMI, the workflow should converge on these **canonical domain outputs** (as versioned snapshots + domain facts):

- Requirement snapshot + `requirement_status=STABILIZED` (or `CANCELED`).
- Architecture package (depth is profile-dependent; may be “lightweight” for Scrum).
- Contract package (proto diff + derived codegen outputs).
- Delivery plan / execution intents (WorkRequests for scaffold/generate/verify).
- Verification evidence bundle and pass/fail.
- Promotion record (published versions + promoted definitions/baseline).
- Release record + deployment evidence.

These outputs are the common substrate for projections, audit timelines, and agent handoffs.
