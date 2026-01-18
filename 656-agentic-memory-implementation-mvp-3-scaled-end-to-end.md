---
title: "656 — Agentic Memory MVP Increment 3 (v6): Scaled End-to-End (Hybrid + Graph Modes + CI Gates)"
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
  - 527-transformation-v6-index.md
---

# 656 — Agentic Memory MVP Increment 3 (v6): Scaled End-to-End (Hybrid + Graph Modes + CI Gates)

Increment 3 makes memory retrieval and publication stable at scale:

- hybrid retrieval (BM25 + vectors),
- relationship-aware retrieval modes (graph expansion),
- evaluation and drift monitoring as CI gates,
- index lifecycle management (blue/green + re-embedding),
- strict filtering remains enforceable (permission trimming + baseline-aware reads).

Normative posture: `backlog/656-agentic-memory-v6.md`, `backlog/656-agentic-memory-implementation-v6.md`, `backlog/496-eda-principles-v6.md`, `backlog/694-elements-gitlab-v6.md`.

## Functional storyline (what changes from Increment 2)

1) Users can find the right answer even with synonyms and scale (“phase zero”, “inner loop”).
2) Agents can ask for graph-local context (“show related runbooks and linked diagrams”) with bounded budgets.
3) Retrieval changes are gated by CI evaluation (no silent regression in ranking or citations).

## 1) Read loop upgrades (hybrid + graph-aware retrieval)

### 1.1 Hybrid retrieval as default

- Candidate recall uses both:
  - full-text (keyword + metadata),
  - vector search over derived chunks (embeddings).
- Strict filtering must be supported in both paths:
  - permission trimming,
  - baseline membership / lifecycle / validity metadata (where present).

Recommended default: keep embeddings in Projection Postgres (`pgvector`) so filtering can be enforced via SQL joins; shard/partition per repo only when scale requires it.

### 1.2 Embeddings are ephemeral projection artifacts

Store provenance per embedded chunk:

- `embedding_model_name`, `embedding_model_version`
- chunker version and parameters
- index generation id (“blue/green”)

### 1.3 Relationship-aware modes (GraphRAG-inspired, minimal)

Add explicit retrieval modes (projection-owned) with budgets:

- `local`: retrieve seeds + bounded neighborhood expansion
- `global`: return higher-level summaries (cached; opt-in)
- `drift`: multi-stage retrieval (budgeted; opt-in)

Mode budgets MUST be first-class request parameters:

- `mode`, `budget: {max_candidates, max_hops, max_tokens, max_seconds}`

### 1.4 Context distillation

Add a distillation stage that produces a smaller context bundle while preserving:

- citations anchored to commit SHA + file anchors,
- relationship traces (“why included” paths).

## 2) Propose + curate upgrades (validators as enforceable gates)

### 2.1 Validator evidence as publish gates

When relevant, validators run automatically and attach evidence to the proposal:

- passing evidence can be required for publish by policy,
- failures can block publish (policy-controlled).

### 2.2 Contradiction / drift heuristics (minimum viable)

Add at least heuristic warnings:

- “proposal contradicts published baseline”
- “proposal references superseded content”

Non-blocking by default, but visible and gateable for high-risk changes.

## 3) Quality loop upgrades (evaluation, drift, and cost controls)

### 3.1 CI-gated evaluation

Expand the harness:

- retrieval metrics (precision/recall, MRR/nDCG),
- drift metrics (published preference under ambiguity),
- citation correctness (answers supported by cited anchors),
- resilience tests (“bad retrieval” → graceful fallback).

### 3.2 Index lifecycle management

- blue/green index builds
- canary evaluation: run golden suite on both before cutover
- scheduled re-embedding when models change
- caching keyed by `(baseline_commit_sha, query_hash, filters, mode)`
