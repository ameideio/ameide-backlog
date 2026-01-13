# 656 — Agentic Memory MVP Increment 1: Minimal End-to-End (Safe + Citeable)

**Goal:** ship one complete loop that is safe, deterministic, and usable by agents/humans:

- **Read:** retrieve citeable context (version-pinned)
- **Propose:** execution agents can only propose
- **Curate:** curator can accept/reject
- **Publish:** promote to “published truth”
- **Ingest:** seed sources (backlog mirror) exist
- **Quality:** a smoke proves the whole loop

This increment intentionally prioritizes **version drift prevention** and **permission-trimmed retrieval** over “fancy embeddings”.

---

## 1) Read loop (projection)

### 1.1 Retrieval contract

Implement a minimal, stable `read_context` contract that agents can hardcode against.

**Request MUST include:**

- `scope`: `{tenant_id, organization_id, repository_id}`
- `principal`: `{principal_id, roles[], purpose}`
- `selector`: `published` (default) OR `baseline_id` (explicit)
- `query` (string) OR `element_id` (direct fetch)

**Response MUST include:**

- `scope` echo
- `baseline_selector_used`: `{kind: "published"|"baseline_id", baseline_id?}`
- `items[]` where each item has:
  - `repository_id`, `element_id`, `version_id`
  - `title` (or stable label), `content` (full body or excerpt)
  - `citations[]` (version pinned; see below)
  - `why_included` (minimal: `keyword_match` / `relationship_path` / `direct_fetch`)

**Citations MUST be machine-checkable:**

- every citation includes `{repository_id, element_id, version_id}`
- optional: `excerpt` `{chunk_key?, start?, end?}` for UI highlighting (non-canonical convenience)

Candidate recall in Increment 1 can be keyword/metadata search only.

### 1.2 Permission-trimmed retrieval (must exist in v1)

- Retrieval must enforce authorization **before** returning results:
  - scope: `{tenant_id, organization_id, repository_id}`
  - principal context: `{principal_id, roles, purpose}`
- Index and retrieve at the same granularity you return (document/section/chunk), so filtering is not “post-hoc”.

**V1 granularity decision:** the returned unit is an `ElementVersion` (optionally with an excerpt); derived chunks are internal projection artifacts that must always back-reference the cited `version_id`.

### 1.3 Structure-aware chunking (minimum viable)

Even if the retrieval is keyword-first, store text in **structure-aware units** so later embedding/rerank is stable:

- Documents: chunk by heading/section path (`H1 > H2 > …`) with stable `chunk_key`.
- BPMN/ArchiMate: store at least one chunk per element/view with stable ids.

---

## 2) Propose loop (domain writes; proposal-only for execution agents)

### 2.1 Proposal object (elements + relationships)

- `ameide:curation.proposal` element with required metadata.
- Required relationship: `ameide:curation.targets_version`:
  - `relationship_kind=REFERENCE`
  - `metadata.target_version_id` MUST be set (deterministic review; aligns with `backlog/300-400/303-elements.md`).

### 2.2 Minimal proposal payload

- Payload can be “full replacement body” for a single target element version (doc update).
- Capture provenance: `{actor_type, actor_id, tool_id, created_at}` and a short summary.

### 2.3 Proposal staleness + rebase (must be defined in v1)

Accepting a proposal must not silently overwrite newer truth.

- A proposal is **stale** if its `targets_version.metadata.target_version_id` is not the currently-selected `published` (or explicitly-selected `baseline_id`) version for that element.
- If stale, the system MUST block `accept` and require a `rebase` first.
- `rebase` creates a new proposal that targets the current version and links:
  - old proposal → new proposal via `ameide:curation.rebased_to` (REFERENCE).

---

## 3) Curate loop (accept/reject)

### 3.1 List + view proposals

- A curator can list proposals by status and open one proposal to see:
  - target element + pinned target version
  - proposed replacement body (or attachment)

### 3.2 Accept/reject actions

- Accept:
  - creates a new `ElementVersion` for the target element
  - links proposal → new version via `ameide:curation.accepted_as` (REFERENCE pinned to the new version)
- Reject:
  - records a decision/rationale element and links `ameide:curation.rejected_because`

RBAC: execution agents cannot accept/reject.

**Audit MUST exist in v1:** accept/reject/rebase writes an explicit decision artifact (element + relationships) so later trust scoring has deterministic inputs.

---

## 4) Publish loop (baseline promotion)

Define “published truth” deterministically in v1:

- Each repository has exactly one `published baseline pointer` → `baseline_id`.
- `selector=published` resolves to that baseline id.
- Promotion updates that pointer (and is auditable).

---

## 5) Ingest loop (seed content)

- Implement backlog mirror as a manual command/job:
  - `backlog/**/*.md` → `ameide:ingest.backlog_md` elements
  - provenance includes `{source_repo, commit_sha, path}`
- Ingested sources are **not** treated as canonical truth, but are retrievable as evidence.

**V1 trust label:** ingested evidence elements/versions MUST carry a stable trust marker (e.g., `source_kind=INGESTED_EVIDENCE`) so trust ranking can be deterministic in Increment 2.

---

## 6) Quality loop (MVP smoke)

Add a smoke that proves the end-to-end path:

1. Create/ensure an ingested doc element exists.
2. Retrieve context (published) → returns citations.
3. Create a proposal against a pinned target version.
4. Curator accepts → new version exists.
5. Curator promotes → retrieval (published) returns the new version by default.

Smoke must also assert permission trimming:

- unauthorized principal cannot retrieve or propose outside scope.

Add one negative assertion to prevent “freeform memory” leaks:

- answers produced by agent workflows must only cite `{repository_id, element_id, version_id}` that came from `read_context.items[].citations[]` (no invented citations).
