# 527 Transformation — Projection Primitive Specification

**Status:** Draft (Enterprise Knowledge ingestion + MVP reads implemented; advanced read models pending)  
**Parent:** [527-transformation-capability.md](527-transformation-capability.md)

This document specifies the **Transformation Projection primitive** — read models for the portal, agents, and architecture impact analysis.

**Extensions:**
- Semantic search (pgvector + contextual retrieval): `backlog/535-mcp-read-optimizations.md`

---

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (Projection), Application Services (query/read-only), Data Objects (read models).
- **Out-of-scope layers:** Canonical writes (domain responsibility) and workflow orchestration (process responsibility).

## 1) Projection responsibilities

Scope identity (required everywhere): `{tenant_id, organization_id, repository_id}` (no separate `graph_id`).

The `transformation-projection` primitive builds read-optimized views for:

- Workspace browsing (tree, element assignments, revision/promotion state).
- Search (full-text + faceted) and history (audit trail, baseline diffs).
- Architecture impact analysis (graph traversal, matrices, view render payloads).
- Governance dashboards (Scrum/TOGAF/PMI gate state, evidence bundles).
- Definition Registry indexing (versions, promotions, deployment refs).

Projection primitives:

- Are idempotent consumers of domain facts and process facts.
- Provide the primary read path for UISurfaces and Agents.
- Do not write domain state.

## 2) Read model inventory (full scope)

- **WorkspaceTreeProjection**: denormalized workspace tree + element counts + promotion status per node.
- **ElementIndexProjection**: full-text + faceted search over elements/versions (kind, type_key, tags, owners, baseline membership).
- **BaselineHistoryProjection**: promotion/baseline timeline, approval history, diffs between baselines.
- **EnterpriseKnowledgeGraphProjection**: element/relationship graph for impact analysis (neighbors, paths, realizations).
- **ArchiMateViewProjection**: ArchiMate view rendering payloads derived from element versions (nodes/edges/layout/style) for canvas UI.
- **ArchiMateMatrixProjection**: relationship matrices by type.
- **BpmnCatalogProjection**: ProcessDefinitions + versions + links + governance state.
- **MetamodelConformanceProjection**: per element-version conformance results for a selected notation profile (standards-compliant vs extended), including validation outcomes and export readiness.
- **DefinitionRegistryProjection**: definitions with versions, promotion state, deployment references.
- **GovernanceStatusProjection**: gate state per initiative (pending/approved/blocked).
- **AuditTrailProjection**: human/audit friendly timeline from domain facts + process facts.
- **ProcessRunTimelineProjection**: per process instance step timeline (ActivityTransitioned + gate decisions + tool runs) correlated to domain facts for evidence and replay.
  - includes declared projection reads as evidence (`ReadPerformed`).

## 3) Query service surfaces (read-only)

The Transformation capability exposes stable, read-only query APIs; they are projection-backed for scale:

- `TransformationQueryService` — list/get initiatives; milestone/metrics/alerts dashboards.
- `WorkspaceQueryService` — get workspace tree, browse nodes, list/search elements.
- `EnterpriseKnowledgeQueryService` — list/get elements, list relationships, get head/published versions, diff versions, list attachments, show baseline membership.
- `BaselineQueryService` — list/get baselines, compare baselines, show approval history.
- `ArchiMateQueryService` — convenience read surface for ArchiMate profile (filtered element/relationship subsets, viewpoints) backed by Enterprise Knowledge projections.
- `BpmnQueryService` — list definitions/versions, get spec refs, list linked elements.
- `DefinitionRegistryQueryService` — list/get definitions, show version/promotion status.

Query rules:

- Read-only RPCs only (no mutations).
- Pagination/filtering are mandatory for list/search surfaces.

## 4) Implementation notes

### Scaffold command

```bash
ameide primitive scaffold --kind projection --name transformation --include-gitops
```

## 4.1) Implementation progress (repo snapshot; checklists)

Delivered (MVP slice):

- [x] Projection runtime exists at `primitives/projection/transformation` (query service + store).
- [x] Migrations are embedded and applied at startup (`internal/adapters/postgres/migrations/*.sql`).
- [x] Idempotent domain-fact ingestion is implemented:
  - [x] `Store.ApplyDomainFact` applies `TransformationKnowledgeDomainFact` events into materialized tables.
  - [x] Idempotency enforced via `projection_inbox(tenant_id,message_id)`.
- [x] Ingestion runner exists (bridge mode; Kafka is the normative transport):
  - [x] `primitives/projection/transformation/cmd/relay` tails the Domain outbox and applies facts with durable offsets.
  - [ ] Replace bridge mode with a Kafka consumer that subscribes to the relevant topic families (per `backlog/527-transformation-proto.md`) once the Kafka wiring is the system-of-record transport.
- [x] Gate: `bin/ameide primitive verify --kind projection --name transformation --mode repo` passes.

Not yet delivered (full projection meaning):

- [ ] Process-facts ingestion + run-timeline materialization (ActivityTransitioned/GateDecisionRecorded/ToolRunRecorded/ReadPerformed).
- [ ] `read_context` + `citations[]` returned by query services (audit-grade reproducibility).
- [ ] Search/history/diff/impact-analysis read models beyond MVP browse/list/get.

## 4.2) Clarification requests (next steps)

Confirm/decide:

- The canonical `read_context` contract (baseline/time/revision) for all query surfaces and what citation fields are mandatory in every response.
- Whether the projection starts in “bridge mode” (reading Domain DB) or immediately in “facts → read model” mode for the v1 acceptance slice.
- The minimum viable impact analysis queries needed for v1 (baseline compare, view render, graph traversal).
- The canonical “process run view” contract: how a run timeline joins process facts to correlated domain facts, and which citation fields are mandatory per step.

## 4.2.1 Read context + citations (normative; v1)

Projection queries must be reproducible and auditable. v1 standardizes two cross-cutting response fields:

- `read_context` (what state was read)
- `citations[]` (why a returned value is believed; what facts/versions it is derived from)

**`read_context` (conceptual shape)**

- Scope: `{tenant_id, organization_id, repository_id}`
- Selector (exactly one):
  - `head` (latest)
  - `published` (promoted/published pointers)
  - `baseline_ref` (baseline id + version)
  - `version_ref` (element id + version id) for pinning
- Optional:
  - `as_of_time` (for time-travel when supported)
  - `include_uncommitted` (must be false for audit views)

**`citations[]` (conceptual shape)**

A citation references the precise inputs used to compute a read-model field:

- `kind`: `domain_fact | process_fact | element_version | attachment | evidence_bundle | work_request | external_link`
- `ref`:
  - for facts: `{topic_family, message_id}` (and optionally a content hash)
  - for element versions: `{element_id, version_id}`
  - for attachments/evidence: `{attachment_id}` / `{evidence_bundle_id}`
  - for work execution: `{work_request_id}`
- optional `note` (human-readable)

**Rules**

- All query responses MUST include `read_context`.
- Audit-/approval-grade views (baseline diffs, conformance results, process run timelines) MUST include citations sufficient to replay “why this was shown”.

## 4.3) Process run timelines (step events)

To support “each BPMN step produces an event” in a way that is queryable and auditable:

- The Process primitive emits step transitions as process facts (see `backlog/527-transformation-proto.md` §3.1).
- The projection materializes a **run timeline** per `{tenant_id, organization_id, repository_id, process_instance_id}`.

Minimum read model expectations (v1):

- Timeline rows for:
  - `ProcessInstanceStarted/Completed/Failed`
  - `ActivityTransitioned` (STARTED/COMPLETED/FAILED) with attempt/retry counts (includes gateways as `activity_id = gateway_id`)
  - `GateDecisionRecorded` (Approve/Reject/Override/Cancel)
  - `ToolRunRecorded` (scaffold/generate/verify/build outputs)
  - `ReadPerformed` (projection reads used for decisions, including `read_context` + citations)
- Deterministic correlations:
  - join step evidence to domain facts using `correlation_id`/`causation_id` links (and/or explicit refs on the process fact)
  - join tool-run process facts (`ToolRunRecorded`) to Domain-owned WorkRequest lifecycle facts (`WorkRequested`/`WorkCompleted`/`WorkFailed`) using `work_request_id` when available
- Citations:
  - each timeline row carries citations referencing the underlying process fact (and any joined domain fact), suitable for audit replay

This read model is the primary UX surface for workflow observability; it is not a domain read.

## 5) Acceptance criteria

1. UISurface and Agents read via projection query services (not domain DB reads).
2. Projections are idempotent and replay-safe.
3. Search/history/diff/impact analysis are projection-backed.
