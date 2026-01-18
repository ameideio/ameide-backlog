# 656 — Agentic Memory MVP Increment 1 (v6): Minimal End-to-End (Safe + Git-citeable)

Increment 1 proves one complete “organizational memory” loop under the v6 posture:

- canonical knowledge is **files in Git**,
- memory is a **Projection** over those files,
- agents **propose** but do not publish,
- publishing advances the **published baseline** (`main`),
- every answer is reproducible via `read_context` + **Git-anchored citations**.

Normative posture:

- Git-backed canonical repository: `backlog/694-elements-gitlab-v6.md`
- Owner-only writes + facts-after-commit: `backlog/496-eda-principles-v6.md`
- Runtime posture: `backlog/520-primitives-stack-v6.md`
- Memory contract (model TBD): `backlog/656-agentic-memory-v6.md`
- Memory implementation mapping: `backlog/656-agentic-memory-implementation-v6.md`

## Functional storyline (what a user/agent experiences)

1) A curator publishes a “Testing SOP” by merging a change to `main`.
2) An agent asks: “how do I run Phase 0/1/2 tests?”
3) The system returns a context bundle that cites exactly what was read (commit SHA + file anchors).
4) The agent proposes an update (patch + evidence) against the cited baseline.
5) A curator reviews and accepts; the platform publishes by merging to `main`.
6) The index updates; the next query returns the new published truth by default.

## Non-negotiables (must hold in Increment 1)

1) **Permission-trimmed retrieval** (no post-hoc filtering).
2) **Owner-only writes**: agents/process/UI do not mutate canonical Git directly; they request changes via the owning Domain.
3) **Reproducible reads**: every answer includes `read_context` and citations sufficient to reproduce “what was read”.
4) **Proposal-only agents**: execution agents can propose; only governed publish advances `main`.

## 0) Implementation anchors (existing primitives)

No new deployables; reuse existing primitives:

- **Domain primitive (Transformation owner):**
  - owns governance state + audit pointers (MR id, commit SHA, pipeline ids),
  - performs canonical Git operations and emits facts after commit.
  - See `backlog/527-transformation-domain-v3.md` and `backlog/527-transformation-e2e-sequence-v6.md`.
- **Projection primitive (Transformation projection):**
  - indexes repo content (docs + inline links + relationship files where present),
  - serves `memory.get_context` as a query API (permission-trimmed).
- **Integration primitive (MCP adapter):**
  - exposes `memory.get_context` (read) and `memory.propose` (write).
  - See `backlog/534-mcp-protocol-adapter.md`.

## 1) Read loop (Projection)

### 1.1 Retrieval contract (MVP, stable)

The goal is not “perfect relevance”; it is **safety + reproducibility**.

**Request MUST include:**

- `scope`: `{tenant_id, repository_id}`
- `principal`: `{principal_id, roles[], purpose}`
- `read_context.selector`:
  - `published` (default) → resolve to `main` head commit at query time
  - `baseline_ref` → caller provides an explicit baseline reference (tag or commit SHA)
  - `head` → caller provides a specific ref (e.g., MR ref) to read from
  - `version_ref` → caller provides an explicit commit SHA to read from
- `query` (string) OR `paths[]` (direct fetch)

**Response MUST include:**

- `read_context.resolved`:
  - `commit_sha` (the exact Git commit used for all reads)
  - `ref` (what was resolved: `main`, tag, MR ref, etc.)
- `items[]` where each item has:
  - `path`
  - `content` (full body or excerpt)
  - `citations[]` (Git-anchored; see below)
  - `why_included` (minimal: `keyword_match|direct_fetch|link_graph`)

### 1.2 Citation contract (MVP, Git-anchored)

Until the v6 memory model is finalized, Increment 1 uses a minimal Git-anchored citation format:

- `repository_id`
- `commit_sha`
- `path`
- `anchor` (one of):
  - `lines: {start, end}` (1-based, inclusive), or
  - `heading: {slug}` (for Markdown/GLFM anchors), or
  - `byte_range: {start, end}` (binary-safe fallback)

Agents MUST only cite items returned in `read_context.items[].citations[]` (no invented citations).

### 1.3 Permission trimming (must exist)

Authorization is enforced at query time:

- a caller only receives items within their allowed `{tenant_id, repository_id}` scope,
- and only for paths they are permitted to view (if path-level ACL exists).

The projection must index at the same granularity it returns so filtering is not “best effort”.

## 2) Propose loop (Domain; proposal-only for agents)

### 2.1 Proposal shape (MVP)

A proposal is a request to the owning Domain to:

- create/update a working branch,
- apply a patch (file edits),
- open or update an MR targeting `main`,
- attach evidence pointers (links to CI results, agent run ids, etc.).

The request MUST include:

- an idempotency key,
- the `read_context.resolved.commit_sha` the proposal was based on,
- and the patch payload (paths + diffs or full replacements).

### 2.2 Staleness rule (must be defined)

The Domain MUST block silent overwrites:

- if the published baseline (`main`) has advanced since the proposal’s base `commit_sha`,
  the proposal is **stale** and must be rebased (Domain-managed rebase/update MR).

## 3) Curate + publish loop (review + merge to `main`)

- Curators review proposals in the product UI (governance truth lives in Domain state).
- Publishing happens by merging the MR to `main` (optionally tagging the resulting commit).
- The Domain records audit pointers (MR id, merge commit SHA, pipeline ids) durably and only then emits facts.

## 4) Ingest loop (seed content for retrieval)

Increment 1 assumes the canonical repository already contains:

- the curated “Testing SOP” doc file, and optionally
- additional evidence files committed as plain files.

The Projection indexes what is in Git at the resolved commit SHA; ingestion from other sources is out of scope for Increment 1 unless it results in Git-traceable citations.

## 5) Quality loop (single smoke)

One “memory smoke” proves the loop end-to-end:

1) Ensure `main` contains a curated `Testing SOP` file.
2) Query “how do I run Phase 0/1/2 tests?” with `selector=published`.
3) Assert response includes `read_context.resolved.commit_sha` and citations anchored to that SHA.
4) Create a proposal to update the SOP, using the resolved commit SHA as the base.
5) Accept + publish (merge to `main`), record audit pointers.
6) Re-run the query and assert the returned SOP content comes from the new `main` commit and is cited correctly.
