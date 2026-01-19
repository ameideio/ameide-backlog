---
title: "656 — Agentic Memory MVP (v6: increment index)"
status: draft
owners:
  - platform
  - transformation
created: 2026-01-18
related:
  - 656-agentic-memory-v6.md
  - 656-agentic-memory-implementation-v6.md
  - 694-elements-gitlab-v6.md
  - 496-eda-principles-v6.md
---

# 656 — Agentic Memory MVP: Increment Plan (Index)

This MVP implementation is split into **end-to-end increments** (each increment advances *all loops*).

**Contract (current):** `backlog/656-agentic-memory-v6.md` (memory model TBD)  
**Implementation plan (current):** `backlog/656-agentic-memory-implementation-v6.md`  
**Historical contract:** `backlog/656-agentic-memory-v1.md`  
**Historical plan:** `backlog/656-agentic-memory-implementation-v1.md`

## Versioning

- **v6 (current):** Git-first increments (`backlog/656-agentic-memory-implementation-mvp-*-end-to-end-v6.md`)
- **v1 (historical):** elements-first increments preserved for context (`backlog/656-agentic-memory-implementation-mvp-*-end-to-end-v1-elements-first.md`)

## Current repo status (2026-01-14)

- Increment 1 is **partially** delivered: `GetReadContext` exists (selector + legacy element/version-pinned citations), and Phase 0/1/2 verification via `ameide test` is green.
- Not yet delivered: permission-trimmed retrieval (authZ), keyword/semantic recall, ingestion jobs, curation queue UX, publish/promotion workflows, and the scenario smoke test.

## Non-negotiables (all increments)

- Permission-trimmed retrieval (no post-hoc filtering).
- Git-anchored citations (minimum viable: `{repository_id, commit_sha, path, anchor}`), until the v6 memory model is locked.
- Execution agents are proposal-only writers (no direct canonical mutation).
- One selector vocabulary (aligned to 527): `head | published | baseline_ref | version_ref`.

## Increments

1. `backlog/656-agentic-memory-implementation-mvp-1-minimal-end-to-end-v6.md`
2. `backlog/656-agentic-memory-implementation-mvp-2-operational-end-to-end-v6.md`
3. `backlog/656-agentic-memory-implementation-mvp-3-scaled-end-to-end-v6.md`

Historical copies:

1. `backlog/656-agentic-memory-implementation-mvp-1-minimal-end-to-end-v1-elements-first.md`
2. `backlog/656-agentic-memory-implementation-mvp-2-operational-end-to-end-v1-elements-first.md`
3. `backlog/656-agentic-memory-implementation-mvp-3-scaled-end-to-end-v1-elements-first.md`

## One scenario (expanded each increment)

Use a single scenario so the test surface grows without changing the story:

**Scenario: “How do I run tests?” (agent + curated org memory)**

1. A curator publishes a canonical “Testing SOP” doc by merging to `main`.
2. An execution agent retrieves context for “how do I run Phase 0/1/2 tests?” with `selector=published`.
3. The response includes `read_context` plus citations anchored to the resolved published commit SHA.
4. The agent proposes an update (e.g., add Phase 3 / cluster-only caveat) against that base commit SHA.
5. Curator rebases (if stale) and publishes by merging to `main`; retrieval now returns the new published truth by default.

This scenario exercises all loops (Read → Propose → Curate → Publish → Ingest → Quality) and becomes the reference for smoke, golden queries, and CI evaluation.

## Verification (by increment)

- **Increment 1:** single “memory smoke” proves the scenario end-to-end, including permission trimming and citation discipline.
- **Increment 2:** golden queries + deterministic diff/rebase checks for the same scenario; retrieval trace logs are required.
- **Increment 3:** CI-gated retrieval evaluation + drift/citation correctness metrics; blue/green index canary runs the golden suite before cutover.

## Where this lands (repo primitives + proto seams)

These increments are implemented by the Transformation capability primitives; “memory” is not a separate storage system.

- **Domain primitive (canonical writes):** `primitives/domain/transformation`
  - Proto: `io.ameide.transformation.knowledge.v1.*`, `io.ameide.transformation.governance.v1.*`
  - Ideal façade: `io.ameide.transformation.memory.v1.*` (front door RPCs for propose/accept/promote)
- **Projection primitive (all reads + retrieval):** `primitives/projection/transformation`
  - Proto: `io.ameide.transformation.knowledge.v1.*`, `io.ameide.transformation.governance.v1.*`
  - Ideal façade: `io.ameide.transformation.memory.v1.*` (front door RPCs for get_context/search)
- **Integration primitive (agent interface):** MCP adapter (see `backlog/534-mcp-protocol-adapter.md`)
  - Tools: `memory.get_context`, `memory.propose`, etc. map 1:1 to the front door RPCs

EDA context (facts → projection) is governed by the active `backlog/496-eda-principles-*` spec; the memory contract requires outbox/inbox + citeable reads, regardless of delivery plane.
