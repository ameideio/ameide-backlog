# 527 Transformation — Domain Primitive Specification

**Status:** Draft (scaffold implemented; verification green; business logic pending)  
**Parent:** [527-transformation-capability.md](527-transformation-capability.md)

This document specifies the **Transformation Domain primitives** — the canonical writers for design-time transformation artifacts (Enterprise Repository + Definition Registry) and methodology profile state (Scrum/TOGAF/PMI).

---

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (Domain), Application Services (write + minimal strong reads), Application Events (domain facts), Data Objects (typed artifacts).
- **Out-of-scope layers:** Strategy/Business catalogs (see `backlog/527-transformation-capability.md`); full read/search surfaces (projection responsibility).

## 1) Domain responsibilities (canonical writers)

### 1.1 `transformation-domain` (core: Enterprise Repository + Definition Registry)

The `transformation-domain` primitive is the **single-writer** for core Transformation state (target: `ameide_core_proto.transformation.core.v1`), including:

- **Transformation initiatives** (portfolio/program objects).
- **Enterprise Repository workspace tree** (`WorkspaceNode`) and artifact registry.
- **Typed architecture artifacts** (ArchiMate models/elements/relationships/views/layout) as domain data.
- **Typed process artifacts** (BPMN ProcessDefinitions + versions + links) as domain data.
- **Promotions/baselines** (draft → submit → approve/reject → promote/rollback).
- **Definition Registry** (ProcessDefinition, AgentDefinition, ExtensionDefinition, promotion state).

**Canonical writer rule:** external file formats are attachments/import/export bundles; they are not canonical state.

### 1.2 Methodology profile domains (bounded contexts)

Profiles are methodology state bound to initiatives; they **govern** the repository and execution, but do not fork the repository’s core model:

- `transformation-scrum-domain` — Scrum artifacts and governance state (`ameide_core_proto.transformation.scrum.v1`).
- `transformation-togaf-domain` — TOGAF/ADM profile configuration and governance state (`ameide_core_proto.transformation.togaf.v1`).
- `transformation-pmi-domain` — PMI profile configuration and governance state (`ameide_core_proto.transformation.pmi.v1`).

These domains:

- Accept change via commands (RPC) and/or **domain intents**.
- Emit **domain facts** via transactional outbox.
- Provide minimal, strongly-consistent reads only (get-by-id / bounded lists); full read/search is projection-backed.

## 2) Core aggregates (core domain)

Authoritative aggregate set (from `backlog/527-transformation-proto.md`):

- Initiative
- WorkspaceNode
- Artifact, ArtifactRevision
- Baseline/Promotion
- Typed artifacts (as aggregates or Artifact specializations):
  - ArchiMateModel / Element / Relationship
  - ArchiMateView (+ ViewNode/ViewEdge layout state + ViewRevision)
  - BpmnProcessDefinition (+ ProcessVersion + artifact links)
  - Definition Registry entries (ProcessDefinition / AgentDefinition / ExtensionDefinition)

Idempotency/order rule: all domain facts carry a monotonic per-aggregate version; consumers are idempotent.

## 3) Enterprise Repository storage model (minimum)

The minimum relational model implied by “typed store + revisions/promotions” (illustrative; exact schema defined by protos + migrations):

- Initiatives + workspace:
  - `transformation_initiatives`
  - `workspace_nodes` (tree: parent_id, slug, order, attributes)
- Generic artifacts:
  - `artifacts` (type, workspace_node_id, canonical_ref, tags)
  - `artifact_revisions` (artifact_id, revision_id, created_by, created_at, content_ref?, digest, schema_version)
  - `artifact_promotions` / `baselines` (initiative_id, label, promoted_revision_ids, policy, approved_by, approved_at)
  - `attachments` (artifact_id or revision_id, uri, content_type, size, checksum)
- Typed ArchiMate:
  - `archimate_models`
  - `archimate_elements`, `archimate_relationships`
  - `archimate_views`
  - `archimate_view_nodes`, `archimate_view_edges`
  - `archimate_view_revisions`
- Typed BPMN:
  - `bpmn_process_definitions`, `bpmn_process_versions`, `bpmn_artifact_refs`
- Definition Registry:
  - `process_definitions`, `agent_definitions`, `extension_definitions`

**Non-negotiables:**

- Every table is tenant-scoped and uses stable IDs.
- Facts are emitted only after persistence; facts include monotonic aggregate versions.

## 4) Definition Registry (inside Transformation)

Transformation is the canonical store for design-time definitions consumed by control-plane operators and runtimes:

- ProcessDefinition (referenced by Process CRs; fetched by Process operator)
- AgentDefinition (fetched by Agent operator; governs runtime)
- ExtensionDefinition (drives scaffolding/promotion processes)

This implies schema-backed domain models, not ad-hoc configs:

- CapabilityDefinition (identity/scope/nouns/topology)
- CapabilityEDAContract (topic families, envelopes, intent/fact catalogs)
- CapabilityPrimitiveDecomposition (primitives + responsibilities)

## 5) Domain APIs (write-first)

- **Write surfaces:** prefer business verbs (no CRUD-as-API); accept via RPC and/or domain intents.
- **Reads:** minimal strong reads only (get-by-id / bounded lists); all browse/search/history/diffs go through projections and projection query services.

## 6) Implementation notes

### Scaffold command

```bash
ameide primitive scaffold --kind domain --name transformation --include-gitops
```

### Outbox

All domain facts emitted via transactional outbox; only domains emit facts.

### Idempotency table

Domain WriteService enforces durable idempotency via `client_request_id` (see `backlog/536-mcp-write-optimizations.md` §7 for full two-layer strategy):

```sql
CREATE TABLE idempotency_keys (
    tenant_id         UUID NOT NULL,
    client_request_id TEXT NOT NULL,
    aggregate_type    TEXT NOT NULL,
    aggregate_id      UUID,
    response_payload  JSONB NOT NULL,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at        TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (tenant_id, client_request_id, aggregate_type)
);
```

**Critical constraint:** Idempotency key insertion MUST be in the same transaction as the domain write. This ensures exactly-once semantics even under adapter restarts or horizontal scaling.

## 6.1) Implementation progress (repo snapshot)

Delivered (scaffold + guardrails):

- Domain scaffold exists at `primitives/domain/transformation` (gRPC server, outbox + dispatcher skeleton, Flyway migrations, non-root container).
- Repo-mode verification is green: `bin/ameide primitive verify --kind domain --name transformation --mode repo`.
- Unit/integration smoke tests exist and run: `cd primitives/domain/transformation && go test ./...`.

Not yet delivered (domain meaning):

- The canonical Enterprise Repository write model (initiatives/workspace/artifacts/baselines/definitions) is not implemented end-to-end; handlers remain scaffold-level.
- Outbox dispatcher publishing to a broker and emitting the declared topic families is not implemented end-to-end.

## 6.2) Clarification requests (next steps)

Confirm/decide:

- The canonical artifact taxonomy for “context curation” (what is an Artifact vs Attachment vs EvidenceBundle vs Definition) and the minimum metadata required for reproducibility.
- Baseline/promotion policy: default approval requirements, roles, and whether promotion is per-initiative or cross-initiative.
- Proposal/draft lifecycle: whether proposals are first-class artifacts (states + IDs) or represented as draft revisions/baselines, and which facts are required.
- Durable idempotency: expected retention/TTL for `client_request_id`, and which commands are required to be idempotent vs best-effort.

## 7) Acceptance criteria

1. Core Transformation state and definitions have a single-writer domain boundary.
2. Typed ArchiMate/BPMN are domain data; import/export is attachment-only.
3. All state changes emit domain facts with per-aggregate monotonic versions.
4. Domains do not provide search/portal read APIs; projections do.
5. Durable idempotency via `client_request_id` at WriteService boundary (two-layer strategy).
