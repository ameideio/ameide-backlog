# 528 — Transformation (Implementation & Migration layer)

## Layer header (Implementation & Migration)

- **Primary ArchiMate element types used:** Work Package, Plateau (optional).
- **Goal:** describe deliverable slicing and acceptance seams, not application contract details.

## Work packages (starter slicing)

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

## Implementation approach (explicit)

Transformation is implemented by applying the platform primitive stack guardrails in `520-primitives-stack-v2.md` and the per-kind implementation guidance in `520-primitives-stack-v2-*.md`.

**Scaffold stance mapping (non-negotiable)**

- Domain (`transformation-domain`): won’t compile until wired
- Process (governance orchestrators): won’t compile until wired
- Projection (workspace/read models/graph/vector): won’t compile until wired
- UISurface: compile-clean, but fails structural checks until wired
- Integration: compile-clean, but fails structural checks until wired
- Agent: compile-clean, but fails structural checks until wired

## Current platform implementation vs 528 target (documented gap)

This section records the current state of the codebase and how it differs from the `528*` target stack. It is informational and intended to guide sequencing and seam placement.

### Current state (observed)

- The web platform reads Transformation data via `TransformationService` RPCs (workspace, milestones, metrics, alerts).
- The concrete `GetTransformationWorkspace` behavior used by the UI is implemented in platform services (`services/graph` / `services/repository`), not in `services/transformation`.
- `services/transformation` implements transformation CRUD and agent definition upsert, but leaves workspace-oriented APIs unimplemented (it returns `UNIMPLEMENTED` for `GetTransformationWorkspace` and `ListTransformationElements`).
- Repository/deliverables used in the Transformation UI are currently backed by the generalized graph/metamodel element store:
  - element rows in `elements` with `element_kind` carrying notation-specific kinds (ArchiMate/BPMN/etc.)
  - proto shape is `ameide_core_proto.graph.v1.Element` with `ElementKind` including `ELEMENT_KIND_ARCHIMATE_*` and `ELEMENT_KIND_BPMN_*`
- Scrum governance has partial foundation (Scrum protos exist; DB migration for Scrum tables + outbox exists), but the domain primitive and outbox dispatcher are not yet implemented end-to-end.

### Target state (528 stack)

- Transformation architecture truth is owned and persisted by the **Transformation Domain primitive** as the canonical writer (architecture truth + governance work models).
- Architecture repository objects are first-class and partitioned (documentation target):
  - repository object types: `ameide_core_proto.transformation.repository.v1`
  - architecture-truth contracts (intents/facts/query): `ameide_core_proto.transformation.architecture.v1`
  - governance work models: `ameide_core_proto.transformation.governance.<method>.v1`
- Writes are event-driven (intents/commands → domain persistence/outbox → domain facts; process facts for orchestration), reads are projection-backed query services.

## Migration seam (recommended)

### Keep `TransformationService` as the initial UI facade

The UI currently depends on `TransformationService` for workspace + lists. Preserve that façade while moving the underlying sources of truth:

- Phase 1: `TransformationService` reads remain backed by existing platform data models (graph/metamodel store) for continuity.
- Phase 2: introduce `TransformationArchitectureQueryService` (projection-backed) and pivot `TransformationService.GetTransformationWorkspace` to compose its response from:
  - architecture projections (repository objects + baseline/evidence read models)
  - governance projections (Scrum/TOGAF/PMI dashboards)
- Phase 3: deprecate direct coupling to graph/metamodel element storage for Transformation deliverables; treat graph as a derived index/projection.

### Writer boundary shift (non-negotiable)

- Move canonical writes behind `transformation-domain` write surfaces (write services and/or domain intents).
- Do not allow UISurfaces to become competing writers (no “UI intent taxonomy”; only domain intents or service calls).

## Legacy strengths to preserve (intentionally)

- Keep a single UI-friendly workspace read seam (compose a workspace read model; keep `GetTransformationWorkspace` as transitional facade).
- Preserve notation-aware flexibility for early UI/editor iteration, but migrate canonical truth to typed repository objects and governed versions.
- Reuse proven multi-tenant / organization scoping enforcement patterns at RPC boundaries while shifting the canonical writer.

## 06* compatibility/deprecation (explicit)

The 06* vision documents (`064–068`) remain useful for mechanics (client command stack, snapshots, governance gates, projection patterns), but in Transformation scope their boundaries are superseded by `528*`. See `528-transformation-064-068-supersession.md`.

## Acceptance (evidence)

- Scrum seam UAT: `531-transformation-scrum-uat.md`
- TOGAF ADM seam UAT: `532-transformation-togafadm-uat.md`
