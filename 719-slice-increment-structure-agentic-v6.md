---
title: "719 — v6 Increment Structure (agentic-first deliverables; Domain-owned evidence; memory is an Agent concern)"
status: draft
owners:
  - platform
  - transformation
  - agents
  - ui
created: 2026-01-21
updated: 2026-01-21
related:
  - 496-eda-principles-v6.md
  - 520-primitives-stack-v6.md
  - 534-mcp-protocol-adapter.md
  - 537-primitive-testing-discipline.md
  - 656-agentic-memory-v6.md
  - 701-repository-ui-enterprise-repository-v6.md
  - 714-togaf-scenarios-v6.md
  - 717-ameide-agents-v6.md
---

# 719 — v6 Increment Structure (agentic-first deliverables)

This backlog defines a **single, repeatable increment structure** for the v6 “Scenario Slices” program (`backlog/714-togaf-scenarios-v6.md`).

Goal: eliminate ambiguity by ensuring every increment:

1) is grounded in a **user story**,
2) specifies what **each primitive kind** must deliver (or explicitly “no new deliverables”),
3) stays aligned to v6 ownership rules (owner-only writes; derived projections; citeable reads),
4) strengthens the **agentic development** loop without inventing new canonical stores.

## 0) Principles (non-negotiable)

### P0.1 Every increment is a user story

- Each increment has exactly one primary user story (persona + capability).
- “Plumbing only” increments are not allowed; if you need plumbing, rewrite it as an operator/developer story with observable acceptance.

### P0.2 All six primitive kinds are first-class in every increment

This structure is scoped to Ameide’s six primitive kinds (per `backlog/520-primitives-stack-v6.md`):

1. **Domain**
2. **Projection**
3. **Process**
4. **Agent**
5. **UISurface**
6. **Integration**

Every increment MUST contain a section for each primitive kind.

If a primitive kind has no new behavior in that increment:

- write **“No new deliverables (must remain compatible)”**, and
- state exactly how that “no change” is proven (test/assertion/evidence).

### P0.3 Git is an adapter behind Domain (write) and Projection (read)

- Git/GitLab is a vendor substrate and a storage adapter, not a canonical platform API surface.
- Canonical writes remain **Domain-owned** (owner-only writes).
- Canonical reads and derived views remain **Projection-owned** (rebuildable; citeable).

### P0.4 “Evidence Spine” is Domain-owned (internal detail)

The “evidence spine” concept is the Domain’s internal audit record for a governed change/publish.

- Other primitives MUST NOT define their own competing “evidence spine” schema.
- Other primitives consume evidence only via:
  - **Domain Facts** (emitted after commit/publish), and/or
  - **Domain Queries** (read access to Domain-owned evidence records).

### P0.5 Memory is an Agent concern (not a separate primitive)

“Memory” is treated as an **agent workflow concern**:

- Agents request context via Projection queries that return citeable content at a `read_context` (commit SHA).
- “Memory behavior” (what context to fetch, how to assemble it, how to cite it) is specified under the **Agent** deliverables.

This does NOT change the v6 posture that:

- content truth is Git (canonical store),
- derived indexes/search/graph are Projection concerns (`backlog/656-agentic-memory-v6.md`).

This principle only removes “Memory” as a separate deliverable bucket in increment writing to reduce confusion.

### P0.6 Remove the word “optional” from increment deliverables

Within an increment’s per-primitive deliverables:

- do not write “optional”.
- write one of:
  - **Deliverables (this increment)**, or
  - **No new deliverables (must remain compatible)**, or
  - **Deferred until Increment N** (explicit and dated).

## 1) Standard increment template (copy/paste)

### 1.0 Increment header

- **Increment**: `<id> — <name>`
- **Tier**: `1|2|3`
- **Primary user story**: `As a <persona>, I can <capability> so that <value>.`
- **User-visible acceptance** (2–5 bullets)

### 1.1 Canonical artifacts (Git)

- **Paths introduced/changed** (minimum set)
- **Minimal content requirements** (IDs/refs rules if applicable)

### 1.2 Contracts touched (delta only)

List only what changes in this increment:

- **Domain**: commands, facts, queries (names + key fields)
- **Projection**: queries (names + key fields)
- **Process**: start/complete signals, facts, required gates
- **Agent**: tool grants, input/output contracts, supervision cadence outputs
- **UISurface**: routes/components, required user actions
- **Integration**: adapters/bindings (e.g., MCP adapter conformance), but only as transport/wiring for the above contracts

### 1.3 Domain-owned evidence (internal)

- **Domain evidence record (internal)**: what Domain persists for audit in this increment
  - example fields: `mr_iid`, `target_branch`, `target_head_sha`, validation evidence refs, citations used/returned, governance decision refs
- **Exposure**:
  - which **Fact(s)** prove completion (post-commit),
  - which **Domain Query** exposes evidence for UI/agents/process (read-only).

### 1.4 Primitive deliverables (mandatory; same structure every time)

For each primitive kind, fill both subsections.

#### 1.4.1 Domain

- **Deliverables (this increment)**:
  - …
- **Prove (contract assertions)**:
  - …

#### 1.4.2 Projection

- **Deliverables (this increment)**:
  - …
- **Prove (contract assertions)**:
  - …

#### 1.4.3 Process

- **Deliverables (this increment)**:
  - …
- **Prove (contract assertions)**:
  - …

#### 1.4.4 Agent

- **Deliverables (this increment)**:
  - include the “memory/context assembly” behavior here (what the agent fetches; how it cites it)
  - include supervision cadence behavior here (`review_interval_minutes`, continue/adjust/escalate)
- **Prove (contract assertions)**:
  - …

#### 1.4.5 UISurface

- **Deliverables (this increment)**:
  - …
- **Prove (contract assertions)**:
  - …

#### 1.4.6 Integration

- **Deliverables (this increment)**:
  - …
- **Prove (contract assertions)**:
  - …

### 1.5 Tests (required)

- **Contract-pass test**
  - steps + assertions; must validate the Domain facts/queries + Projection behavior deterministically at a SHA
- **UX-pass test**
  - minimal UI flow that exercises the same contracts
- **Negative case**
  - at least one deterministic failure (e.g., broken ref, duplicate id, SHA guard mismatch)

## 2) Notes on applying this to 714

- Keep the slice/story writing in `backlog/714-togaf-scenarios-v6.md`, but write each slice so it can be copied into the template above without “optional” language.
- Keep the detailed per-scenario implementer docs (`714-togaf-scenarios-v6-00/01/02.md`), but ensure each one uses the same per-primitive deliverables headings and explicitly says “No new deliverables” where appropriate.

