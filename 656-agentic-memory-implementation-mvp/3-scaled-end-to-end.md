# 656 — Agentic Memory MVP Increment 3: Scaled End-to-End (Hybrid + Graph Modes + CI Gates)

**Goal:** keep the same end-to-end loops, but make retrieval and governance scale and stay stable as the graph grows:

- hybrid retrieval (BM25 + vectors) with model/versioned embeddings
- relationship-aware retrieval modes (GraphRAG-inspired)
- version-aware routing (baseline/time/validity semantics)
- evaluation and drift monitoring as CI gates
- index lifecycle management (blue/green, re-embedding)

---

## 1) Read loop upgrades (hybrid + graph-aware retrieval)

### 1.1 Hybrid retrieval as default

- Candidate recall uses both:
  - full-text/BM25 (keyword + metadata)
  - vector search over derived chunks
- Keep strict metadata filters (baseline/lifecycle/validity/ACL) in both recall paths.

### 1.2 Embeddings are ephemeral artifacts

Store embedding provenance per chunk:

- `embedding_model_name`, `embedding_model_version`
- embedding params (dim, normalization, chunker version)
- index generation id (“blue/green”)

### 1.3 Relationship-aware modes (GraphRAG-inspired, minimal)

Implement retrieval modes (projection-owned) for different query intents:

- **Local/entity mode:** retrieve a seed set + bounded neighborhood expansion with explainable relationship traces.
- **Global mode:** return higher-level summaries for broader questions (requires caching and stricter budgets).
- **Drift/iterative mode:** multi-stage retrieval when initial recall is weak (budgeted, opt-in).

### 1.4 Context distillation

Add a distillation stage that produces a smaller, higher-signal context bundle while preserving:

- citations pinned to versions
- relationship traces (why included)

---

## 2) Propose + curate loop upgrades (validators as enforceable gates)

### 2.1 Validator evidence becomes part of promotion

- Profile validators (ArchiMate/BPMN) run automatically and attach results as evidence to proposals/decisions.
- Promotions can require “green evidence” for certain types/risks.

### 2.2 Contradiction / drift heuristics (minimum viable)

Implement at least heuristic checks:

- “this proposal contradicts current baseline” flags
- “this references superseded content” flags

These do not block by default, but are visible and can be gated by policy for high-risk changes.

---

## 3) Publish loop upgrades (policy-driven promotions)

- Promotion policy can key off:
  - validator results
  - required approvers
  - risk level / classification
  - environment validity (`applies_to_environment`)

---

## 4) Ingest loop upgrades (multi-format + entity extraction)

### 4.1 Multi-format ingestion

- BPMN: chunk by process/subprocess/lane/activity; preserve element ids.
- ArchiMate: chunk by element/view + relationship triples; preserve ids.
- Docs: heading-aware chunking remains default.

### 4.2 Entity extraction and linking (derived projection)

- Extract entities and link them back to canonical element ids to improve graph expansion and disambiguation.
- Keep this as a projection output, not canonical truth.

---

## 5) Quality loop upgrades (evaluation, drift, and cost controls)

### 5.1 CI-gated evaluation

Expand the harness:

- retrieval metrics (precision/recall, nDCG/MRR)
- version drift metrics (published/valid content preference)
- citation correctness (claims supported by cited versions)
- resilience tests (“bad retrieval” → model should degrade gracefully with evidence)

### 5.2 Observability and budgets

- per-mode budgets (local cheaper; global expensive and cached/explicit)
- tracing from retrieval request through final context assembly (for debugging regressions)

### 5.3 Index lifecycle management

- blue/green index builds
- incremental updates
- scheduled re-embedding jobs when models change
- caching keyed by `(baseline_id, query_hash, filters, mode)`

