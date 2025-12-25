# 303 — Elements: Elements-Only Canonical Storage

**Epic:** Unified Data Model  
**Status:** Draft (locked decisions; implementation plateaus tracked in 527)  
**Priority:** High  

---

## Decision (locked)

1. **Elements-only canonical storage.** All canonical repository content is stored as `Element`/`ElementRelationship`/`ElementVersion` (and attachments/evidence as non-canonical blobs referenced from elements). There is no separate canonical content model per notation.
2. **Remove the legacy typed content model.** The platform does not expose a separate canonical noun for “typed content” outside the element substrate. Use `Element`/`ElementVersion`/`ElementRelationship` for canonical repository content, and `Attachment`/`EvidenceBundle`/Definition Registry entries for non-graph payloads and governance. Legacy typed tables/services/protos are deprecated and must be removed; new work must not depend on them.
3. **Identity/scoping is fixed.** Every repository operation is scoped by `{tenant_id, organization_id, repository_id}`. `repository_id` is the only repository identifier exposed in contracts and APIs (no separate `graph_id`/`graphId`).
4. **Profiles, not forks.** ArchiMate/BPMN/etc. are notation (metamodel) profiles over the element model, governed by `element_kind` + namespaced `type_key` values.

This document defines the canonical *data model substrate*. Delivery sequencing and DoD is owned by `backlog/527-transformation-implementation-migration.md`.

---

## 1) Canonical data model

### 1.1 Elements

- `Element` is the universal atomic unit (node/view/document/binary/etc.).
- `ElementVersion` is the immutable payload revision for an element.
- Versioning rule: **no cascading versioning** (versioning a view/document does not version referenced elements).

Minimum canonical fields:

- Scope: `tenant_id`, `organization_id`, `repository_id`
- Identity: `element_id`
- Classification: `element_kind`, `type_key` (namespaced)
- Payload: `body` + `metadata` (and a version history in `element_versions`)

### 1.2 Relationships

All relationships (semantic + containment + reference) are represented as `ElementRelationship` rows.

Relationship taxonomy (recommended):

- `relationship_kind` ∈ `{SEMANTIC, CONTAINMENT, REFERENCE}`
- `type_key` is namespaced:
  - standards namespaces: `archimate:*`, `bpmn:*`, `c4:*`, etc.
  - extension namespace: `ameide:*`

**Versioned references (lock-in):**

- A `REFERENCE` relationship MAY include `target_version_id` in its `metadata` to point to a specific immutable `ElementVersion`.
- For governance/workflow anchoring (e.g., “requirement snapshot”, “deliverables package root”), `target_version_id` MUST be present so reads are reproducible (no “latest version” ambiguity).

### 1.3 Attachments and evidence (non-canonical)

External file formats and evidence bundles are **not canonical state**.

- Attachments/evidence are blobs (storage objects) referenced from elements/versions using REFERENCE relationships (and surfaced through projections).
- Import/export pipelines transform between attachments and canonical elements.

### 1.4 Workspace hierarchy (no “second folder system”)

Repository organization (folders, navigation tree) is represented as a **workspace tree**, not as Elements:

- `WorkspaceNode` (tree) stores “where things live” in the repository browser.
- `ElementAssignment` links elements into the workspace tree (with ordering and optional fixed version references).

Rule: do not introduce parallel “folder elements” for repository organization. If a notation needs internal structure (e.g., sections inside a document), represent it with `ElementRelationship` of kind `CONTAINMENT` within the element graph, not by duplicating the workspace tree.

---

## 2) Metamodel conformance (standards-compliant vs extended)

Conformance is profile-driven and enforced as gates:

- **Standards-compliant profiles** (e.g., ArchiMate 3.2, BPMN 2.0)
  - constrain allowed `type_key` namespaces to the relevant standards (`archimate:*`, `bpmn:*`)
  - enforce relationship validity matrices (warn/block)
  - enable standard exports from promoted truth
- **Extended profiles**
  - allow additional namespaces (e.g., `ameide:*`)
  - define custom constraints and export targets (downstream consumers must opt in explicitly)

Profile selection is stored on the model/view element (metadata/properties) or referenced via a ProfileDefinition in the Definition Registry.

---

## 3) Baselines, promotion, and “normative truth”

The platform supports “normative” design-time truth via baseline/promotion workflows:

- Baseline = stable set of pointers `{element_id, version_id}` (and definition-version pointers where applicable).
- Promotion updates “published” pointers or promoted reference sets, without forcing cascaded versioning.
- All decisions and transitions are auditable via domain facts + process facts.

---

## 4) Implications for services and UI

### 4.1 Services

- Domains are single-writer aggregates for element graph writes (persist → outbox → facts).
- Projections are the only browse/search/history/diff read path (facts → read models).

### 4.2 UI

- UI is a **generic editor** over elements (canvas/document/etc.) configured by a selected notation profile.
- Routing and scoping are repository-first:
  - `/org/[orgId]/repo/[repositoryId]` is the primary context.

---

## 5) Removal scope (legacy typed content model)

The following are deprecated and must be removed (no new dependencies):

- legacy typed tables and CRUD/query services
- legacy typed proto packages and generated SDK surface
- UI coupling to legacy typed APIs/entities as canonical backend state

Replacement:

- use element kinds (views/diagrams, documents, binaries) inside `{tenant_id, organization_id, repository_id}`.

---

## 6) Where execution is tracked

Authoritative delivery plan and plateau DoD lives in:

- `backlog/527-transformation-implementation-migration.md`

And contracts/invariants live in:

- `backlog/527-transformation-proto.md`
