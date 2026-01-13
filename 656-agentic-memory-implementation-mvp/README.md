# 656 — Agentic Memory MVP: Increment Plan (Index)

This directory splits the MVP implementation into **end-to-end increments** (each increment advances *all loops*).

**Contract:** `backlog/656-agentic-memory.md`  
**Overview plan:** `backlog/656-agentic-memory-implementation.md`

## Non-negotiables (all increments)

- Permission-trimmed retrieval (no post-hoc filtering).
- Version-pinned citations (`{repository_id, element_id, version_id}`).
- Execution agents are proposal-only writers (no direct canonical mutation).
- One selector vocabulary (aligned to 527): `head | published | baseline_ref | version_ref`.

## Increments

1. `backlog/656-agentic-memory-implementation-mvp/1-minimal-end-to-end.md`
2. `backlog/656-agentic-memory-implementation-mvp/2-operational-end-to-end.md`
3. `backlog/656-agentic-memory-implementation-mvp/3-scaled-end-to-end.md`

## One scenario (expanded each increment)

Use a single scenario so the test surface grows without changing the story:

**Scenario: “How do I run tests?” (agent + curated org memory)**

1. Backlog markdown about testing/inner loop is ingested as evidence (`ameide:ingest.backlog_md`).
2. A curator publishes a canonical “Testing SOP” element (plus minimal ArchiMate links) into the published baseline.
3. An execution agent retrieves context for “how do I run Phase 0/1/2 tests?” and receives version-pinned citations.
4. The agent proposes an update (e.g., add Phase 3 / cluster-only caveat); the proposal targets a pinned `version_id`.
5. Curator rebases (if stale), accepts, and promotes; retrieval now returns the new version as published truth.

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
