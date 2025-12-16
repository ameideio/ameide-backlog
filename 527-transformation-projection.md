# 527 Transformation — Projection Primitive Specification

**Status:** Draft (scaffold implemented; ingestion/read models pending)  
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

The `transformation-projection` primitive builds read-optimized views for:

- Workspace browsing (tree, artifact membership, revision/promotion state).
- Search (full-text + faceted) and history (audit trail, baseline diffs).
- Architecture impact analysis (graph traversal, matrices, view render payloads).
- Governance dashboards (Scrum/TOGAF/PMI gate state, evidence bundles).
- Definition Registry indexing (versions, promotions, deployment refs).

Projection primitives:

- Are idempotent consumers of domain facts and process facts.
- Provide the primary read path for UISurfaces and Agents.
- Do not write domain state.

## 2) Read model inventory (full scope)

- **WorkspaceTreeProjection**: denormalized workspace tree + artifact counts + promotion status per node.
- **ArtifactIndexProjection**: full-text + faceted search over artifacts/revisions (type, tags, owners, baseline membership).
- **BaselineHistoryProjection**: promotion/baseline timeline, approval history, diffs between baselines.
- **ArchiMateGraphProjection**: element/relationship graph for impact analysis (neighbors, paths, realizations).
- **ArchiMateViewProjection**: view rendering payloads (nodes/edges/layout/style) for canvas UI.
- **ArchiMateMatrixProjection**: relationship matrices by type.
- **BpmnCatalogProjection**: ProcessDefinitions + versions + links + governance state.
- **DefinitionRegistryProjection**: definitions with versions, promotion state, deployment references.
- **GovernanceStatusProjection**: gate state per initiative (pending/approved/blocked).
- **AuditTrailProjection**: human/audit friendly timeline from domain facts + process facts.

## 3) Query service surfaces (read-only)

The Transformation capability exposes stable, read-only query APIs; they are projection-backed for scale:

- `TransformationQueryService` — list/get initiatives; milestone/metrics/alerts dashboards.
- `WorkspaceQueryService` — get workspace tree, browse nodes, list/search artifacts.
- `ArtifactQueryService` — get artifact, list revisions, diff revisions, list attachments, show baseline membership.
- `BaselineQueryService` — list/get baselines, compare baselines, show approval history.
- `ArchiMateQueryService` — get model, list elements/relationships, graph traversal.
- `ArchiMateViewQueryService` — get view render payload, list views, get view revision.
- `BpmnQueryService` — list definitions/versions, get spec refs, list linked artifacts.
- `DefinitionRegistryQueryService` — list/get definitions, show version/promotion status.

Query rules:

- Read-only RPCs only (no mutations).
- Pagination/filtering are mandatory for list/search surfaces.

## 4) Implementation notes

### Scaffold command

```bash
ameide primitive scaffold --kind projection --name transformation --include-gitops
```

## 4.1) Implementation progress (repo snapshot)

Delivered (scaffold + tests):

- Projection scaffold exists at `primitives/projection/transformation` (query service skeleton, non-root container, tests).
- Scaffold tests run: `cd primitives/projection/transformation && go test ./...`.
- Note: `ameide primitive verify` does not currently support `--kind projection` (repo guardrails are not yet enforced via CLI for Projection).

Not yet delivered (projection meaning):

- Idempotent ingestion/materialization from domain facts + process facts into the listed read models is not implemented end-to-end.
- Agent-grade read semantics (`read_context` + citations + audit replay, semantic search) are not implemented end-to-end.

## 4.2) Clarification requests (next steps)

Confirm/decide:

- The canonical `read_context` contract (baseline/time/revision) for all query surfaces and what citation fields are mandatory in every response.
- Whether the projection starts in “bridge mode” (reading Domain DB) or immediately in “facts → read model” mode for the v1 acceptance slice.
- The minimum viable impact analysis queries needed for v1 (baseline compare, view render, graph traversal).

## 5) Acceptance criteria

1. UISurface and Agents read via projection query services (not domain DB reads).
2. Projections are idempotent and replay-safe.
3. Search/history/diff/impact analysis are projection-backed.
