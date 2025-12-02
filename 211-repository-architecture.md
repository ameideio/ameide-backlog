Title: Configurable Architecture Repository (TOGAF ADM) — Design & Plan

Status: Draft

Goals

- Multi-tenant, multi-graph organization of all architecture artifacts (BPMN, ArchiMate, UML, DMN, documents, etc.)
- UI organized as a TOGAF ADM-style graph, but fully configurable by end users
- Flexible hierarchy with both concrete folders and smart (virtual) folders
- Strong linking to existing artifacts.v1 Aggregate/ArtifactView and cross-references

Key Decisions

- Domain modeled as a distinct Repository bounded context with its own protos and services
- Store tree as materialized path (ltree-compatible string) for efficient queries; parent_id maintained for adjacency
- Assignments link graph nodes to artifacts; one artifact can appear in multiple nodes
- Configurable node types and constraints (allowed children, allowed artifact kinds)
- Smart folders are virtual nodes with a filter expression over ArtifactView (server-evaluated)

Protobufs

- Files: `packages/ameide_core_proto/src/ameide_core_proto/graph/v1/`
  - `graph.proto`: Repository, RepositoryConfig, NodeType, Node, Assignment, NodeDefinition, RepositoryStats
  - `graph_service.proto`: RepositoryAdminService, RepositoryTreeService, RepositoryAssignmentService, RepositoryQueryService
- Integration: keeps artifact identity (`artifact_id`, `artifact_type`) and uses existing `artifacts.v1` query APIs to fetch details when needed

DB Schema (PostgreSQL)

- Table: `repositories`
  - `id` (uuid/text pk), `tenant_id`, `key` (slug, unique per tenant), `name`, `description`, `template_id` (nullable), `attributes` (jsonb), `config` (jsonb), timestamps, `version`
  - Indexes: `(tenant_id,key) unique`, `(tenant_id,name)`
- Table: `graph_nodes`
  - `id`, `tenant_id`, `graph_id` fk, `parent_id` fk null, `type_key`, `name`, `slug`, `path` (ltree/text), `order` (int), `is_virtual` (bool), `filter_expression` (text), `attributes` (jsonb), timestamps, `version`
  - Indexes: `(graph_id,parent_id)`, `(graph_id,path)`, `(tenant_id,type_key)`
  - Consider `ltree` extension for path and a GIST index if available
- Table: `graph_assignments`
  - `id`, `tenant_id`, `graph_id` fk, `node_id` fk, `artifact_id`, `artifact_type`, `pinned_version` (int), `role` (text), `order` (int), `attributes` (jsonb), timestamps, `version`
  - Indexes: `(graph_id,node_id)`, `(artifact_id)`, `(tenant_id,artifact_type)`
- Optional: `graph_templates` (id, tenant_id nullable, name, description, definition jsonb, is_default)

Multi-Tenancy

- All tables include `tenant_id` and enforce tenant filters in repositories
- Unique constraints scoped by tenant for keys
- `TenantRepository` helper can be reused for isolation

Hierarchy & Configurability

- Node types define constraints and UI hints (e.g., icon, color)
- Default template seeds a tree (ADM Cycles, Phases, Architecture Landscape, Standards, Reference Library, Governance, Requirements, Solutions)
- End users can add/move/rename nodes when `allow_custom_nodes` is enabled
- Smart folders evaluate server-side queries (initially parameterized filters; later CEL)

UI Integration (services/www_ameide_platform)

- Replace current mock graph with backend calls:
  - `RepositoryQueryService.GetTree` to build the left tree
  - `RepositoryAssignmentService.ListNodeAssignments` to list contents
  - Use `ArtifactQueryService` for details (name, tags, status) or denormalize minimal fields in assignment
- Support drag-and-drop to `AssignArtifact` / `UnassignArtifact`
- Provide views by ADM phases, by cycles, and by type as predefined nodes

Phased Implementation Plan

1) Protos & Docs (this change)
   - Add graph domain protos and design doc

2) Storage Layer
   - SQLAlchemy models + repositories for `repositories`, `graph_nodes`, `graph_assignments`
   - Alembic migration: tables + ltree extension (if used)

3) Minimal Service
   - New service `services/core_graph` (Python gRPC) implementing Admin/Tree/Assignment APIs
   - Integrate auth/tenant from `RequestContext`

4) Frontend Wiring
   - Replace `mock-graph` with SDK client calls
   - Tree, list, grid, search; DnD assignments; badges by type/status

5) Smart Folders & Policies
   - Add virtual folders with simple filter configuration (server-side evaluation)
   - Add per-node RBAC inheritance (optional)

6) Advanced: Templates & Imports
   - Tenant-defined templates; bulk seed; export/import templates as JSON

Runtime Notes

- Consistency: optimistic versioning on entities; simple `MoveNode` recalculates `path` for subtree in a transaction
- Performance: index `path` for subtree reads, paginate assignments; cache stats in Redis (existing infra)
- Backfill: migration script to seed a default ADM graph for each tenant (optional)

Open Questions

- Expression language for smart folders (CEL vs. simple filter DSL) — start with filter map in proto, evolve later
- Denormalization scope in `Assignment` to minimize cross-service calls for ArtifactView fields
- Whether to expose a consolidated `RepositoryBrowser` query that returns nodes + minimal artifact views in one request
