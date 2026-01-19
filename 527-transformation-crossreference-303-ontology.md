# 527 — Transformation Crossreference (303/527 Target; element-based ontologies)

**Status:** Draft  
**Audience:** Anyone reading legacy backlogs  
**Scope:** Provide a single mapping table that explains how older documents map to the current target posture:

- **Target is `303/527`**: canonical storage is **Elements + Relationships + Versions** (relational Postgres), scoped by `{tenant_id, organization_id, repository_id}`.
- **Methodologies are ontologies**: Scrum/TOGAF “contracts” are **type_key namespaces + UI/workflows + projections**, not separate canonical tables/aggregates.
- **Graph + vector are read projections**: traversal and semantic search are derived read models (Projection responsibility).

Canonical references:

- Elements-only canonical storage: `backlog/300-400/303-elements.md`
- Transformation capability & primitive specs: `backlog/527-transformation-capability.md`
- Integration/EDA invariants: `backlog/496-eda-principles-v6.md`
- Projection + vector posture: `backlog/520-primitives-stack-v2-projection.md`, `backlog/535-mcp-read-optimizations.md`

---

## Crossreference table (legacy → target)

| Legacy concept / wording | Target concept | Meaning under 303/527 | Canonical reference(s) | Notes |
|---|---|---|---|---|
| `graph_id` | `repository_id` | Repository scope identifier used everywhere; no separate `graph_id` in new contracts. | `backlog/300-400/303-elements.md` | Treat `graph_id` in older docs as a historical name for repository identity. |
| “Graph service” as a writer | Domain primitive writes; Graph is read-only | Canonical writes happen in Domain primitives; Graph is projection-only. | `backlog/470-ameide-vision.md`, `backlog/496-eda-principles-v6.md` | Graph may still exist as a traversal-optimized projection, but never as a write surface. |
| “Graph DB is the canonical store” | Postgres is canonical; graph is derived | The graph structure is stored as `ElementRelationship` rows; graph traversal can be implemented as a projection/read model. | `backlog/300-400/303-elements.md`, `backlog/527-transformation-projection.md` | A dedicated graph DB is optional infrastructure; it is never canonical truth. |
| “Vector DB is canonical knowledge” | Vector index is a Projection read model | Embeddings are derived from element-version content and keyed by `{element_id, version_id}` provenance. | `backlog/520-primitives-stack-v2-projection.md`, `backlog/535-mcp-read-optimizations.md` | Projection owns embedding/indexing; domains do not depend on vectors. |
| Work item as primary object | Element as primary object | “Change”, “requirement”, “deliverable”, “story”, “deliverable package”, “run”, etc. are all represented as Elements + links. | `backlog/300-400/303-elements.md`, `backlog/527-transformation-domain.md` | UI may use work-item vocabulary; storage remains elements. |
| “Pins” as a typed object (`ChangePins`, `*_ref` fields) | Versioned REFERENCE relationships | The simplest handshake uses well-known `REFERENCE` relationships from the change element (e.g., `ref:requirement`, `ref:deliverables_root`) with explicit version selection (e.g., `metadata.target_version_id`). | `backlog/527-transformation-methodology-dictionary.md`, `backlog/300-400/303-elements.md` | Avoid a separate “pins” object; just use relationships with explicit version anchoring for audit. |
| Scrum as separate canonical aggregates/tables | Scrum as an ontology over elements | Scrum artifacts are element types (`type_key` namespace like `scrum:*`) plus views/workflows; no additional canonical tables are required. | `backlog/300-400/303-elements.md`, `backlog/527-transformation-capability.md` | This supersedes plans that introduce `transformation.scrum_*` tables as canonical state. |
| TOGAF ADM deliverables as a separate model | TOGAF as an ontology over elements | ADM deliverables are just elements linked under a deliverables root; phase gates are workflow/process evidence. | `backlog/527-transformation-scenario-togaf-adm.md` | The ADM UI can still be dedicated; storage remains shared. |
| ChangeSet aggregate + event-store preview model | Proposal elements + baselines/promotions | Reviewable proposals are represented as draft element versions and/or proposal elements; promotions/baselines provide normative truth and audit. | `backlog/527-transformation-domain.md`, `backlog/527-transformation-projection.md` | Legacy ChangeSet designs are kept for historical context; the target approach uses element versions + governance. |

---

## Deprecated legacy docs (kept for history)

These documents contain assumptions that conflict with the `303/527` target posture. They are preserved for context but should not be used as implementation targets:

- `backlog/200-300/220-repository-entity-model.md` (uses `graph_id` as primary identifier)
- `backlog/000-200/113-10-changeset-hitl.md` (event-store + ChangeSet model; `graph_id` routing)
- `backlog/300-400/367-1-scrum-transformation.md` (plans Scrum-first tables/aggregates)
- `backlog/506-scrum-vertical-v2.md` (contains “Scrum tables” implementation language; treat as historical storage guidance if present)

For each, interpret terms using the crossreference table above and prefer `303/527` as authoritative going forward.
