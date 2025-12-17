# 300 – AMEIDE Metamodel Unification

## Objective

Design and introduce a universal element-based graph that replaces artifact-centric storage with a graph-native representation. Every concept (nodes, edges, views, documents, binaries) becomes an `Element`, enabling consistent tooling, cross-ontology traceability, and incremental versioning.

## Key Decisions / Open Questions

1. **Schema Shape**
   - Core fields: `element_id`, `tenant_id`, `organization_id`, `repository_id`, `element_kind`, `ontology`, `type_key`, `body`, `metadata`.
   - Versioning strategy: append-only revisions per element with a derived “current” view.
   - Storage layout (single `universal_elements` table + `element_relationships` vs. partitioned tables).

2. **Ontology Payloads**
   - Embed existing generated protos (e.g., `archimate.ArchiMateElement`, `bpmn.BpmnElement`) into `body`.
   - Define lightweight payload contracts for Markdown/PDF/document structures.
   - Extensibility for future ontologies (TOGAF, C4, Radar, etc.).

3. **Relationship Modeling**
   - Standardize structural edges (`contains`, `references`, `realizes`, etc.) as relationship elements.
   - Enforce referential integrity and ontology-specific rules in service layer.
   - Determine whether platform resources such as repositories and transformations become first-class `Element` instances (with their own relationships) or remain separate aggregates referencing elements.

## Deliverables

1. **Metamodel Specification**
   - Formal definition of `Element`, relationships, versioning semantics.
   - Update proto schemas (`@ameide/core-proto`) with new element messages and services.
   - **Status:** ✅ Element proto definitions merged (`graph/v1/graph.proto`). Core schema captured in documentation.

2. **Element Service MVP**
   - API surface: create/update/get/list elements and relationships, query by ontology/type, inspect version history.
   - Backing persistence using Postgres (leveraging existing graph DB) or dedicated graph store.
   - Event stream for element changes to feed projections and analytics.
   - **Status:** ✅ Service modularized under `services/graph/src/graph/` (elements, relationships, versions) composed in `graph/service.ts` and registered via `server.ts`.

## Proposed Table Schema

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `elements` | Current (mutable) state of each element | `id` (PK), `tenant_id`, `organization_id`, `repository_id`, `element_kind`, `ontology`, `type_key`, `body` (JSONB/BLOB), `metadata` (JSONB), `created_at`, `created_by`, `updated_at`, `updated_by` |
| `element_versions` | Immutable history of element revisions (append-only) | `element_id`, `version_id` (PK), `version_number`, `body`, `metadata`, `created_at`, `created_by` |
| `element_relationships` | First-class relationship elements (semantic/structural) | `id` (PK), `tenant_id`, `organization_id`, `repository_id`, `element_kind`, `ontology`, `type_key`, `source_element_id`, `target_element_id`, `body`, `metadata`, `created_at`, `created_by`, `updated_at`, `updated_by` |
| `relationship_versions` | Revision history for relationship elements | `relationship_id`, `version_id` (PK), `version_number`, `source_element_id`, `target_element_id`, `body`, `metadata`, `created_at`, `created_by` |
| `attachments` (optional) | External binaries (PDFs, images) referenced by elements | `id` (PK), `element_id`, `storage_uri`, `hash`, `media_type`, `size_bytes`, `metadata`, `created_at`, `created_by` |

## Ontology Mapping Examples

- **ArchiMate**
  - Each ArchiMate concept is stored as an `elements` row with `ontology = 'archimate'`, `type_key` matching the ArchiMate element type, and `body` containing the generated `archimate.ArchiMateElement` message. Semantic relationships (Realization, Serving, etc.) are `element_relationships` rows pointing to the source/target element IDs and carrying the `archimate.ArchiMateRelationship` payload. Views (diagrams) become dedicated elements (`type_key = 'archimate_view'`) whose bodies hold layout/viewpoint metadata while relationship elements describe which nodes/edges they display (`view_contains_element`, `view_contains_relationship`).
- **BPMN**
  - BPMN flow nodes (tasks, events, gateways) map to `elements` with `ontology = 'bpmn'` and a `body` serialized from the BPMN proto. Sequence flows, message flows, and associations appear in `element_relationships`, preserving `source_id`/`target_id` references. Process diagrams or collaborations are represented as view elements with structural relationships connecting the diagram to its constituent nodes and edges.
- **Documents / PDF**
  - Markdown or PDF files are modeled as document elements (`ontology = 'document'`, `type_key = 'markdown' | 'pdf'`). The `body` holds structured metadata (parsed Markdown AST, PDF metadata, etc.), while large binaries live in `attachments` referenced by the document element. Optional child elements (e.g., sections, pages) can be created and linked via `element_relationships` to allow fine-grained referencing inside documents.

## Implementation Progress

- ✅ **Proto Layer**  
  `graph/v1/graph.proto` added with Element, Relationship, and version RPC definitions. Re-exported through `@ameide/core-proto`.

- ✅ **Database**  
  `V4__element_graph.sql` establishes core tables, `V5__seed_element_graph.sql` seeds representative elements/relationships, `V6__migrate_memberships_to_relationships.sql` backfills legacy memberships, and `V7__relax_relationship_foreign_keys.sql` unlocks view→relationship containment in Postgres.

- ✅ **Service Layer**  
  Repository service hosts the Element Service via modular handlers (`graph/elements`, `graph/relationships`, `graph/versions`) assembled in `graph/service.ts` and wired through `server.ts`, including create/update/delete/list coverage.

- ✅ **Version APIs**  
  `getElementVersion`, `listElementVersions`, `getRelationshipVersion`, and `listRelationshipVersions` handlers implemented with paging support and wired to the Postgres-backed history tables.
- ✅ **Repository Bridge**  
  A barebones `RepositoryService` persists repositories to the system table while emitting corresponding graph elements (`services/graph/src/graph/service.ts`), laying the groundwork for element-native containment.
- ✅ **Testing Footprint**  
  Unit suites cover version handlers, and an end-to-end view lifecycle scenario exercises graph → element → relationship flows using Testcontainers-backed Postgres (`src/__tests__/unit/**`, `src/__tests__/e2e/**`).

- ⏳ **Versioning Events / Streaming**  
  Table rows track versions, but no event stream or change notification pipeline yet.

- ⏳ **Ontology-specific adapters**  
  ArchiMate/BPMN/document ingestion still routes through legacy artifacts; converters to/from the element service remain TODO.

## Next Steps

1. **Graph Projections & Search**
   - Build traversal endpoints (e.g., children, dependents, impact analysis) atop `element_relationships`.
   - Evaluate indexes/queries needed for large-scale usage and add pagination to list operations.

2. **Eventing & Audit**
   - Publish create/update/delete events for elements and relationships to the platform event bus.
   - Add auditing hooks and RBAC enforcement aligned with tenant/graph scoping.

3. **Ontology Adapters & Legacy Integration**
   - Replace legacy artifact ingestion with adapters that translate to Element Service payloads.
   - Define the dual-write/cutover strategy so existing workflows emit element updates.

4. **Repository & Platform Integration**
   - Keep repositories authoritative in the system table while expressing containment through `Element` graph relations.
   - Apply the same element-native projection to other aggregates (tenants, users, teams, transformations) with canonical relationships (`tenant_owns_graph`, `user_assigned_role`, etc.).

5. **ETL / Migration Tooling**
   - Backfill legacy data into the element tables using proto payloads and validate regenerated views.

6. **Testing & Observability**
   - Add automated coverage for element/relationship lifecycle and version APIs.
   - Introduce metrics/logging dashboards to track graph usage and latency.
