# 527 Transformation — Implementation & Migration (core)

**Status:** Draft (scaffolds implemented; migration plan pending)  
**Parent:** [527-transformation-capability.md](527-transformation-capability.md)

This document captures the Implementation & Migration layer for delivering the Transformation capability, including work packages, migration seams, and explicit gaps between the current codebase and the target architecture.

---

## Layer header (Implementation & Migration)

- **Primary ArchiMate element types used:** Work Package, Plateau (optional), Deliverable, Gap.
- **Goal:** describe deliverable slicing and acceptance seams, not application contract details.

## 1) Work packages (starter slicing)

**Work Package A — Enterprise Repository + baseline seam**

- Entry:
  - domain outbox publishing is operational
  - projection rebuild/replay is supported
- Exit evidence:
  - typed ArchiMate storage is usable via write services/intents
  - baseline draft → submit → approve/reject → promote is end-to-end
  - evidence links are visible in an audit timeline read model

**Work Package B — Governance seam (Scrum / TOGAF ADM)**

- Entry:
  - baseline seam exists (Work Package A exit)
- Exit evidence:
  - governance progression is driven by domain facts and process facts (no runtime RPC reads/writes)
  - gate/timebox decisions require evidence bundles and are attributable
  - UISurface reads projections and invokes domain write surfaces only

## 2) Implementation approach (explicit)

Transformation is implemented by applying:

- `backlog/520-primitives-stack-v2.md` (guardrails)
- `backlog/533-capability-implementation-playbook.md` (capability implementation DAG)
- per-kind scaffolding specs (`backlog/510-*` through `backlog/513-*`)

**Scaffold stance mapping (non-negotiable)**

- Domain (`transformation-domain`): compile-clean scaffold; business logic pending
- Process (governance orchestrators): compile-clean scaffold; workflow logic pending
- Projection (workspace/read models/graph/vector): compile-clean scaffold; ingestion/read models pending
- UISurface: placeholder runtime; portal UX pending
- Integration: MCP adapter scaffold; connector/tool wiring pending
- Agent: compile-clean scaffold; role definitions/tools pending

## 2.1) Implementation progress (repo snapshot)

Delivered (scaffold + guardrails):

- Primitive scaffolds exist for Transformation under `primitives/*/transformation*` and compile/test cleanly.
- Repo-mode verification is green for CLI-supported kinds:
  - `bin/ameide primitive verify --kind domain --name transformation --mode repo`
  - `bin/ameide primitive verify --kind process --name transformation --mode repo`
  - `bin/ameide primitive verify --kind agent --name transformation --mode repo`
  - `bin/ameide primitive verify --kind uisurface --name transformation --mode repo`
- Proto contracts for Scrum and Architecture bounded contexts exist, plus legacy `TransformationService` façade protos (see `backlog/527-transformation-proto.md`).

## 2.2) Clarification requests (next steps)

Confirm/decide:

- Which “acceptance slice” is the next migration target (e.g., Scrum governance loop vs architecture revision/baseline promotion), and what evidence proves success end-to-end.
- Whether `services/transformation` remains a façade (Phase 1/2) or is retired in favor of domain/process/projection primitives (and the deprecation plan).
- Which projection read models are required for the first slice (workspace, audit timeline, baseline compare), and whether projection starts in bridge mode.

## 3) Current platform vs target (documented gap)

### Current state (observed)

- The web platform reads Transformation data via `TransformationService` RPCs (workspace, milestones, metrics, alerts).
- The concrete `GetTransformationWorkspace` behavior used by the UI is implemented in platform services (`services/graph` / `services/repository`), not in `services/transformation`.
- `services/transformation` implements transformation CRUD and agent definition upsert, but leaves workspace-oriented APIs unimplemented (`UNIMPLEMENTED` for `GetTransformationWorkspace` and `ListTransformationElements`).
- Transformation UI deliverables are currently backed by generalized graph/metamodel element storage:
  - element rows in `elements` with `element_kind` carrying notation-specific kinds (ArchiMate/BPMN/etc.)
  - proto shape: `ameide_core_proto.graph.v1.Element` with `ElementKind` including `ELEMENT_KIND_ARCHIMATE_*` and `ELEMENT_KIND_BPMN_*`
- Scrum governance has partial foundation (Scrum protos exist; scaffolds exist for domain/process/projection), but the outbox publishing + process orchestration + portal read models are not yet implemented end-to-end.

### Target state (527 stack)

- Transformation truth is owned and persisted by the **Transformation Domain primitives** as canonical writers.
- Repository objects are first-class and partitioned (documentation target):
  - core repository/definitions: `ameide_core_proto.transformation.core.v1`
  - governance work models: `ameide_core_proto.transformation.governance.<method>.v1` (or per-profile packages)
- Writes are event-driven (intents/commands → domain persistence/outbox → domain facts; process facts for orchestration), reads are projection-backed query services.

## 4) Migration seam (recommended)

### Keep `TransformationService` as an initial UI façade

- Phase 1: `TransformationService` reads remain backed by existing platform data models (graph/metamodel store) for continuity.
- Phase 2: introduce projection-backed query services and pivot `TransformationService.GetTransformationWorkspace` to compose its response from:
  - architecture projections (repository objects + baseline/evidence read models)
  - governance projections (Scrum/TOGAF/PMI dashboards)
- Phase 3: deprecate direct coupling to graph/metamodel element storage for Transformation deliverables; treat graph as a derived index/projection.

### Writer boundary shift (non-negotiable)

- Move canonical writes behind Transformation domain write surfaces (write services and/or domain intents).
- Do not allow UISurfaces to become competing writers.

## 5) Legacy strengths to preserve

- Keep a single UI-friendly workspace read seam (compose a workspace read model; keep `GetTransformationWorkspace` as transitional façade).
- Preserve notation-aware flexibility for early UI/editor iteration, but migrate canonical truth to typed repository objects and governed versions.
- Reuse proven multi-tenant scoping enforcement patterns at RPC boundaries while shifting the canonical writer.

## 6) Legacy mapping (current code ownership)

### `services/transformation` (legacy)

- Implements partial `TransformationService` CRUD and agent definition upsert/get.
- Does not implement workspace/milestones/metrics/alerts.
- Does not implement EDA contracts for Scrum intents/facts or outbox publishing.

Target role:

- Either becomes a façade while Domain primitives take over, or is evolved into a Domain-primitive-shaped implementation (outbox + bus + strict boundaries).

### `primitives/domain/transformation` (scaffold)

- Scaffold exists with outbox/dispatcher shape, migrations, and smoke tests, but handlers remain placeholder-level.
- Target bounded contexts (core repository + methodology profiles) are not implemented end-to-end.

Target role:

- Becomes the canonical implementation of the Transformation bounded contexts (core + scrum), consistent with the primitives stack and 496.

## 7) Full-scope Phase G plan (implementation governance)

This is the “how we govern implementation” plan for delivering the full future state, using:

- `backlog/524-transformation-capability-decomposition.md` as the execution loop
- `backlog/520-primitives-stack-v2.md` as the conformance baseline

### G.0 Governance gates (non-negotiable)

- Contract gates: `buf lint`, `buf breaking`, deterministic `buf generate`, regen-diff, SDK-only import policy.
- Runtime gates: compile/test per primitive runtime; determinism for Process primitives; replay safety for Agents; idempotency for Projections.
- Deployment gates: deterministic builds + scans; GitOps sync + smoke probes; operators surface conditions.
- Promotion gates: artifacts/definitions promoted only through governed flows (process facts + approvals), never by direct DB edits.

### G.1 Work packages (plateaus) for full scope

1. **G1 — Contracts**
   - Lock and publish proto packages:
     - `ameide_core_proto.transformation.core.v1`
     - `ameide_core_proto.transformation.scrum.v1` (validate/extend)
     - `ameide_core_proto.transformation.togaf.v1`
     - `ameide_core_proto.transformation.pmi.v1`
     - `ameide_core_proto.process.{scrum,togaf,pmi}.v1`
   - Lock topic families and envelope invariants (meta + subject + monotonic versions).
2. **G2 — Enterprise Repository (Domain)**
   - Implement typed ArchiMate store + views/layout + revision/promotion model.
   - Implement BPMN ProcessDefinition storage/versioning + linking.
   - Implement transactional outbox publishing + write surfaces.
3. **G3 — Profile domains**
   - Scrum: complete bounded context and outbox publishing per `backlog/506-scrum-vertical-v2.md`.
   - TOGAF/ADM + PMI: implement profile configuration domains + facts.
4. **G4 — Governance execution (Process)**
   - Implement Temporal-backed governance processes (ingress + deterministic workflows + process outbox).
5. **G5 — Projections and portal (Projection + UISurface)**
   - Implement read models and full portal UX surfaces (see `backlog/527-transformation-projection.md`, `backlog/527-transformation-uisurface.md`).
6. **G6 — Agent-driven `524` execution (Agent)**
   - Store AgentDefinitions per role; wire tool grants and risk tiers; execute `524` as stored ProcessDefinition(s).
7. **G7 — Hardening and migration**
   - Multi-tenant isolation, auditability, retention policies, SLOs, backup/restore.
   - Migrate off legacy façades with coexistence windows and explicit deprecation milestones.

### G.2 Delivery mechanics (day-to-day)

- Repeat `524` Step 6 loop for every work package: scaffold → generate → compile/test → publish images → GitOps sync → smoke probes.
- Promote only when contract gates and deployment gates are green.
- Manage drift continuously via `524` Step 7 (TOGAF Phase H).
