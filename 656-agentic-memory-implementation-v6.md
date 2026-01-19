---
title: "656 — Agentic Memory Implementation (v6: Git-backed owner + projection; model TBD)"
status: draft
owners:
  - platform
  - transformation
created: 2026-01-18
supersedes:
  - 656-agentic-memory-implementation-v1.md
related:
  - 656-agentic-memory-v6.md
  - 694-elements-gitlab-v6.md
  - 527-transformation-domain-v3.md
  - 527-transformation-e2e-sequence-v6.md
  - 527-transformation-process-v4.md
  - 496-eda-principles-v6.md
  - 509-proto-naming-conventions-v6.md
  - 535-mcp-read-optimizations.md
  - 536-mcp-write-optimizations.md
---

# 656 — Agentic Memory Implementation (v6: Git-backed owner + projection; model TBD)

## Functional contract (v6)

This implementation exists to make these loops work end-to-end in the product:

1) Read
- Humans and agents ask questions and receive context assembled from the published baseline.
- Responses are reproducible (`read_context`) and citeable (citations).

2) Propose
- Agents produce proposals (patches + evidence) without directly changing published truth.
- Proposals are reviewable and traceable to a baseline.

3) Curate and publish
- Curators/humans review proposals, approve/reject, and publish by advancing `main`.
- Publishing records audit pointers and emits facts after commit.

4) Rebuild
- Projection indexes are rebuildable from Git plus audit pointers; rebuild is a supported recovery mode.

## Technical mapping (v6)

This v6 implementation plan aligns “agentic memory” with the Git-backed Enterprise Repository posture:

- Canonical content is Git (`backlog/694-elements-gitlab-v6.md`).
- The Transformation Domain is the owner of canonical writes (Git operations + governance state).
- The Transformation Projection is the read surface (indexing + retrieval + citations).
- “Memory model” specifics (IDs/citations/baselines abstraction) are **TBD** and will be locked in a follow-up backlog.

## 0) What we can implement now (without deciding the memory model)

### A) Owner + audit pointers (Domain)

Implement/require:

- tenant/repo registry (tenant/repo → GitLab project),
- governance state (approvals/policy),
- audit pointers for every canonical update (MR id, commit SHA, pipeline id, tag).

This is the durable spine that retrieval and evidence must reference even while citations are still TBD.

### B) Read path and retrieval discipline (Projection)

Implement/require:

- `read_context` enforced on all memory retrieval tools,
- permission-trimmed retrieval at query time,
- rebuildable indexing over Git content (docs + code + inline links + relationship files),
- deterministic query responses that include citations (even if citation fields evolve).

### C) Agent “front doors” (MCP + proto)

Implement/require:

- `memory.get_context` (read) → Projection
- `memory.propose` (write) → Domain

Agents do not get direct Git/DB access; they always route through these seams.

## 1) What is explicitly deferred (TBD)

The following must be decided before “memory” is treated as complete:

- stable identifiers inside Git files (or owner-issued IDs),
- citation format and replay rules (commit+path+anchor vs other),
- the baseline selector semantics beyond `main` (tags? published pointers?),
- whether “elements/versions/relationships” is (a) a derived UX model or (b) a canonical write model expressed as files.

## 2) Recommended next: lock a minimal citation contract (v6)

Once ready, define a minimal v6 citation contract that supports:

- “what did the agent read?” reproducibility,
- “what changed?” diffability across baselines,
- durable evidence references for governance.

This contract should be anchored to Git + Domain audit pointers (MR id, commit SHA, pipeline id), not to transport/runtime specifics.
