---
title: "527 Transformation — Projection Primitive (v6: TOGAF hierarchy projection + Git-backed elements)"
status: draft
owners:
  - transformation
  - platform
created: 2026-01-19
supersedes:
  - 527-transformation-projection.md
related:
  - 701-repository-ui-enterprise-repository-v6.md
  - 694-elements-gitlab-v6.md
  - 520-primitives-stack-v6.md
  - 496-eda-principles-v6.md
  - 656-agentic-memory-v6.md
---

# 527 Transformation — Projection Primitive (v6: TOGAF hierarchy projection + Git-backed elements)

This v6 refocuses the Transformation Projection on the current product story:

- Canonical authored artifacts live as **Git-backed elements** in the tenant Enterprise Repository (`backlog/694-elements-gitlab-v6.md`).
- The primary repository UX is a **TOGAF 10 Architecture Repository hierarchy** (`backlog/701-repository-ui-enterprise-repository-v6.md`).
- That hierarchy is **purely derived** (projection-owned). There is no canonical “workspace tree” / “workspace node” write model.

## 1) Projection responsibilities (v6)

Scope identity (required everywhere): `{tenant_id, organization_id, repository_id}`.

The Projection provides rebuildable read models for:

- TOGAF hierarchy browsing (categories, counts, filtered lists),
- element search and retrieval (full-text + faceted),
- relationship reconstruction and graph traversal (derived from in-file references + optional `relationships/**`),
- baseline history and audit timelines (facts and Git audit pointers),
- process catalog views (ProcessDefinitions as BPMN files under `processes/**`, with deploy/validation status),
- agent memory retrieval (context assembly + citations; model details TBD).

## 2) Key read models (v6 inventory)

### 2.1 TOGAFHierarchyProjection (new; replaces “workspace tree”)

Derived hierarchy that renders TOGAF categories over a repository’s published baseline:

- Categories:
  - Architecture Landscape
  - Reference Library
  - Standards Information Base
  - Governance Log
  - Architecture Capability
- Inputs:
  - Git-backed element inventory (paths + metadata),
  - optional path conventions (e.g., `elements/**`, `processes/**`),
  - optional element metadata conventions (frontmatter, labels).
- Outputs:
  - category lists, counts, and navigation,
  - stable element references for editor opening.

This is the contract the UISurface uses to render the repository UX; it is intentionally not canonical state.

### 2.2 RelationshipGraphProjection (derived)

Reconstruct a graph from:

- normal references inside element content (links/IDs),
- optional relationship files under `relationships/**`.

Used for backlinks, impact analysis, and agent context expansion.

### 2.3 ProcessCatalogProjection (derived)

Index ProcessDefinitions from:

- `processes/<module>/<process_key>/v<major>/process.bpmn`

and expose:

- available processes by module/key/version,
- deployability/conformance results per runtime profile (Zeebe/Flowable),
- deployment pointers (what is deployed where) derived from audit pointers and GitOps evidence.

## 3) Query surfaces (what the UI/agents call)

This doc does not prescribe service names, but it does prescribe capabilities:

- List repositories (within org scope) and select published baseline.
- Browse TOGAF hierarchy:
  - list categories and items per category
  - open element content (by reference + read_context)
- Search:
  - full-text and faceted over elements
  - filter by TOGAF category and/or element kind/type
- Relationships:
  - backlinks for an element
  - neighbors/path queries (bounded)
- Process catalog:
  - list/get ProcessDefinitions and show runtime validation + deploy status

All queries must return:

- `read_context` (head/published/baseline selectors),
- `citations[]` sufficient for audit-grade replay where required.

## 4) Rebuildability + ingestion posture (v6)

- Projections are rebuildable from Git content plus owner audit pointers/facts.
- The clean target ingestion posture is event-plane delivery (CloudEvents + Protobuf) per `backlog/496-eda-principles-v6.md`.
- Local/dev may keep a relay/tailer as debug/recovery only (not a runtime dependency).

## 5) Historical note (what this replaces)

Earlier Transformation docs used a canonical “workspace tree” / `WorkspaceNode` posture.

Under v6, repository hierarchy is rendered as a **TOGAFHierarchyProjection** derived from Git-backed elements; any “workspace node” concepts remain historical context only.
