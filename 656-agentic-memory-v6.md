---
title: "656 — Agentic Memory (v6: Git-backed repository, projection-owned memory, model TBD)"
status: draft
owners:
  - platform
  - transformation
created: 2026-01-18
supersedes:
  - 656-agentic-memory-v1.md
related:
  - 694-elements-gitlab-v6.md
  - 496-eda-principles-v6.md
  - 520-primitives-stack-v6.md
  - 509-proto-naming-conventions-v6.md
  - 657-transformation-domain-clean-target-v2.md
  - 534-mcp-protocol-adapter.md
  - 535-mcp-read-optimizations.md
  - 536-mcp-write-optimizations.md
---

# 656 — Agentic Memory (v6: Git-backed repository, projection-owned memory, model TBD)

## Functional storyline (v6)

This is how “organizational memory” behaves from the perspective of humans and agents.

1) People author and review knowledge as files
- Architecture artifacts, requirements, diagrams, BPMN, and relationship references are stored as files in the tenant repository (relationships are expressed as inline references inside those files).
- “Published truth” is what’s merged to `main` (baseline commit SHA, optionally tagged).

2) People and agents retrieve context in a reproducible way
- Queries are served by a derived Projection (search/graph/context assembly).
- Every answer includes `read_context` plus citations so you can reproduce “what was read”.

3) Agents propose; curators publish
- Execution agents can propose changes (patches + evidence), but they do not publish.
- Humans (or governed workflows) review and approve; the platform publishes by advancing the baseline.

4) Memory is rebuildable
- Indexes/graphs/embeddings are rebuildable from Git content plus owner audit pointers.
- Deleting and rebuilding the projection is always a valid recovery path.

## Technical posture (v6)

This v6 reframes “organizational memory” to match the platform’s Git-backed canonical store posture:

- Canonical design-time truth is **files in a tenant Git repository** (`backlog/694-elements-gitlab-v6.md`).
- “Memory” is primarily a **Projection concern**: indexing + retrieval + citations over canonical Git content, anchored by owner audit pointers.
- The **memory model** (stable IDs, citations, and how we represent “elements/versions/baselines” over Git) is **TBD**.

This document supersedes `backlog/656-agentic-memory-v1.md` (303-first / elements-first posture) while keeping its intent and safety invariants.

## GitLab product analogies (memory as a derived view over repos)

To keep the “Git-first + projection-owned memory” posture concrete, use GitLab’s own product behavior as the analogy:

- **Canonical knowledge** ≈ the repository content you can browse in GitLab (files at a commit SHA); our platform treats that as the source of truth.
- **Citations** ≈ GitLab permalinks to a specific blob/commit/path (optionally anchored to a location inside the file).
- **Memory retrieval** ≈ GitLab’s derived repository views (search/code intelligence/Knowledge Graph): they accelerate discovery, but never replace the repo as truth.

## Projection contract (v6; required)

“Memory” is projection-owned, but that is an explicit contract (not just an implementation detail):

- **Rebuildable:** deleting and rebuilding the projection is always a valid recovery path; correctness cannot depend on projection state.
- **Incrementally maintainable:** projections may update incrementally (facts, merge events, periodic reconciles), but must remain compatible with full rebuilds.
- **Citation-grade:** every returned result (search hit, graph edge/backlink, context bundle) must be explainable via citations anchored to a specific `read_context` (commit SHA + path + anchor).
- **Eventually consistent:** the UI and agents must tolerate lag between publication and projection availability, and must be able to show “what commit SHA this view reflects”.

## 0) Non-negotiables (still required under v6)

1) **Permission-trimmed retrieval is mandatory.**  
No post-hoc filtering; authz must be enforced at query time (`backlog/300-400/329-authz.md`).

2) **Owner-only writes is mandatory.**  
Execution agents, processes, and UI surfaces do not mutate canonical memory/artifacts directly; they route writes through the owning Domain (`backlog/496-eda-principles-v6.md`).

3) **Facts are emitted only after commit.**  
For Git-backed owners, “commit” means commit/merge/tag plus durable audit pointer recording before emitting facts (`backlog/496-eda-principles-v6.md`).

4) **Reproducible reads: `read_context` + citations.**  
Every retrieval must state the effective `read_context`, and return durable citations so answers can be replayed/audited (`backlog/534-mcp-protocol-adapter.md`).

5) **Execution agents are proposal-first.**  
Agents propose changes; humans (or a governed workflow) review/approve before publication.

## 1) What “memory” means under Git-first

Under v6, “organizational memory” is:

- Canonical content: files in the tenant repository (docs, diagrams, BPMN, code, etc.), with relationships expressed as inline references within those files.
- Governance truth: minimal Domain-owned state (tenancy, policy, approvals, audit pointers).
- Retrieval truth: Projection-owned derived read models (indexes/graphs/context assembly), rebuildable from Git + owner audit pointers.

## 2) Memory model (TBD)

The v6 posture intentionally does not yet decide many details, but it does lock one requirement:

- **Metadata blocks are mandatory:** canonical artifacts MUST embed a frontmatter/metadata block that carries a stable `id` and a `refs` list (in-file, not a sidecar). This is required so relationships can be authored consistently and so projections can rebuild a citeable graph over Git content.

Open questions that remain TBD:

- the exact metadata schema per file type (Markdown/BPMN/model formats), as long as it remains in-file and citeable,
- the canonical citation format (commit SHA + path + anchor vs content hash vs owner-issued IDs),
- how we represent “baselines” beyond `main` commit anchors and optional tags,
- whether any “elements/versions/relationships” abstraction remains as a derived UX model or a canonical write model.

Until decided, treat “memory” as a Projection-derived read model over Git, anchored by the Domain’s audit pointers.

## 3) Retrieval contract (what clients can rely on)

Regardless of the final memory model, the retrieval contract remains:

- `read_context` selector vocabulary is stable (e.g., `head | published | baseline_ref | version_ref`).
- responses include citations sufficient to reproduce what was read.
- relationship-aware retrieval is allowed (graph expansion), but remains derived.

## 4) Write / curation contract (what clients can rely on)

Clients (humans and agents) may request changes, but:

- only the owning Domain performs canonical writes (Git operations),
- proposals are first-class and reviewable,
- publication happens by advancing the baseline (`main` merge + optional tagging) under governance.

## 5) Historical context

The v1 contract remains as historical context for the earlier “elements + versions + relationships” substrate:

- `backlog/656-agentic-memory-v1.md`
