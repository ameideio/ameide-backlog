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

- Implement `read_context` selector (`published` default) and return `citations[]` for every context item:
  - citations MUST pin `{element_id, version_id}`.
- Implement basic candidate recall:
  - keyword/metadata search is enough for Increment 1.

### 1.2 Permission-trimmed retrieval (must exist in v1)

- Retrieval must enforce authorization **before** returning results:
  - scope: `{tenant_id, organization_id, repository_id}`
  - principal context: `{principal_id, roles, purpose}`
- Index and retrieve at the same granularity you return (document/section/chunk), so filtering is not “post-hoc”.

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
  - `metadata.target_version_id` MUST be set (deterministic review).

### 2.2 Minimal proposal payload

- Payload can be “full replacement body” for a single target element version (doc update).
- Capture provenance: `{actor_type, actor_id, tool_id, created_at}` and a short summary.

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

---

## 4) Publish loop (baseline promotion)

- Provide a single “published truth” selector:
  - either published pointers or “active baseline pointer”, but it must be deterministic.
- Curator can promote accepted versions into the published set.

---

## 5) Ingest loop (seed content)

- Implement backlog mirror as a manual command/job:
  - `backlog/**/*.md` → `ameide:ingest.backlog_md` elements
  - provenance includes `{source_repo, commit_sha, path}`
- Ingested sources are **not** treated as canonical truth, but are retrievable as evidence.

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

