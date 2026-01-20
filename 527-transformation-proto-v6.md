---
title: "527 Transformation — Proto Contract Notes (v6: Git-backed elements, derived Enterprise repository hierarchy)"
status: draft
owners:
  - transformation
  - platform
created: 2026-01-19
supersedes:
  - 527-transformation-proto.md
related:
  - 694-elements-gitlab-v6.md
  - 701-repository-ui-enterprise-repository-v6.md
  - 520-primitives-stack-v6.md
  - 496-eda-principles-v6.md
  - 509-proto-naming-conventions-v6.md
  - 527-transformation-projection-v6.md
  - 527-transformation-domain-v3.md
  - 527-transformation-e2e-sequence-v6.md
  - 656-agentic-memory-v6.md
---

# 527 Transformation — Proto Contract Notes (v6: Git-backed elements, derived Enterprise repository hierarchy)

This v6 replaces the older 527 proto posture that assumed:

- canonical Elements/Versions/Relationships persisted as Postgres rows, and
- a canonical “workspace tree” (`WorkspaceNode`) as a write model, and
- a Definition Registry as the canonical store for ProcessDefinitions,
- compile-to-Temporal as the BPMN execution target.

v6 posture:

- Canonical repository content is **Git-backed** (files in the tenant Enterprise Repository) (`backlog/694-elements-gitlab-v6.md`).
- UI talks in **elements**, but elements are stored as files in Git (`backlog/701-repository-ui-enterprise-repository-v6.md`).
- Repository hierarchy is an **EnterpriseRepositoryHierarchyProjection** (derived; not canonical state) (`backlog/527-transformation-projection-v6.md`).
- ProcessDefinitions are governed BPMN files under `processes/**` and are deployed to **Zeebe/Flowable** (`backlog/520-primitives-stack-v6.md`).

## 1) What must be true of the proto surfaces (v6 invariants)

Inter-primitive communication is contract-first per `backlog/496-eda-principles-v6.md` (public APIs/event payloads defined in Protobuf; transport may be RPC and/or events). GitLab REST is an internal storage adapter used by owner/projection primitives; it is not an inter-primitive integration surface.

### 1.1 Scope identity is always explicit

Every command, fact, and query must carry scope identity:

- `tenant_id`
- `organization_id`
- `repository_id`

There is no separate “workspace id” for repository navigation in v6.

### 1.2 Element identity is Git-backed (even if the UX says “element”)

v6 requires that inter-primitive contracts can reference canonical content in Git.

Minimum viable element reference (implementation-neutral):

- `repository_id`
- `path` (repo-relative)
- `ref` (commit SHA for audit-grade reads; branch/MR refs allowed for non-audit views)
- optional `anchor` (e.g., Markdown heading, line range, model object id)

This maps to the memory/read discipline (`read_context` + citations) required by projections and agent memory.

GitLab analogy: an audit-grade element reference behaves like a GitLab “permalink” to a blob at a specific commit SHA (with optional in-file anchor), while non-audit preview may use branch/MR refs.

### 1.3 Relationships are not a canonical CRUD surface

Relationships are authored only as normal references inside element content.

Contracts must support:

- retrieving derived backlinks/graph expansions from the Projection,
- citing the underlying Git anchors used to compute those derived relationships.

### 1.4 ProcessDefinitions are files in the repository (no Definition Registry)

ProcessDefinitions are referenced as Git-backed artifacts:

- `processes/<module>/<process_key>/v<major>/process.bpmn`

Promotion requires runtime-profile validation and deploy evidence for Zeebe/Flowable.

### 1.5 Repository hierarchy is projection-derived (no canonical workspace tree)

The repository hierarchy presented in the UI is derived by the Projection from the Git file tree; it is not a Domain aggregate and is not mutated via commands such as “CreateWorkspaceNode”.

If the UI needs curated grouping beyond the raw file hierarchy, treat it as:

- derived metadata (projection-owned), and/or
- element-authored metadata (stored in Git), and/or
- governance policy (domain-owned) only where it affects enforcement.

## 2) Compatibility note (migration posture)

Existing proto packages and implementations may still expose element/version nouns.

Under v6, those nouns MUST be interpreted as Git-backed:

- “version” corresponds to a commit anchor (commit SHA + path) for audit-grade reads.
- “baseline/published” corresponds to `main` commit SHA (optionally tagged).

A follow-up contract refactor may introduce explicit GitRef/Path-based message fields, but this doc is already normative about the semantics.
