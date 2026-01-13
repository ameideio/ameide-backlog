# 656 — Agentic Memory MVP Increment 3: Scaled End-to-End (Hybrid + Graph Modes + CI Gates)

**Goal:** keep the same end-to-end loops, but make retrieval and governance scale and stay stable as the graph grows:

- hybrid retrieval (BM25 + vectors) with model/versioned embeddings
- relationship-aware retrieval modes (GraphRAG-inspired)
- version-aware routing (baseline/time/validity semantics)
- evaluation and drift monitoring as CI gates
- index lifecycle management (blue/green, re-embedding)

---

## 0) Implementation anchors (primitives + proto)

Increment 3 is where the retrieval and governance loops become “production-shaped”:

- **Projection primitive:** hybrid retrieval (keyword + vectors), mode/budget parameters, graph-aware retrieval, index lifecycle management.
- **Domain primitive:** validators as attachable evidence + policy-driven promotion gates (still expressed as canonical elements/versions/baselines).
- **EDA posture:** facts remain the rebuild source; embeddings/graph indexes are projections (derived).

See `backlog/656-agentic-memory-implementation.md` §5 and `backlog/535-mcp-read-optimizations.md` for the retrieval/indexing posture.

## 1) Read loop upgrades (hybrid + graph-aware retrieval)

### 1.1 Hybrid retrieval as default

- Candidate recall uses both:
  - full-text/BM25 (keyword + metadata)
  - vector search over derived chunks
- Keep strict metadata filters (baseline/lifecycle/validity/ACL) in both recall paths.

**Implementation note (make constraints explicit):** baseline + lifecycle + validity + ACL filtering must be supported in the vector recall path. In practice this usually requires one (or a combination) of:

- partitioned indexes (per tenant/org/repo) to keep ACL sets small,
- storing high-signal filter metadata alongside embeddings (published/baseline membership, classification, lifecycle),
- and/or doing bounded vector recall first, then a post-recall join/filter against authoritative baseline membership (with careful leakage controls).

**Recommended default for this repo:** keep embeddings in the projection Postgres (`pgvector`) so baseline membership and ACL filtering are enforceable via SQL joins/filters, with optional per-repo partitioning when scale demands.

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

**Mode budgets MUST be first-class request parameters:**

- `mode`: `local|global|drift`
- `budget`: e.g., `{max_candidates, max_hops, max_tokens, max_seconds}`

Default budgets must be conservative; “global” mode should be opt-in.

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

Add a canary evaluation step for rollouts:

- run the golden query suite against both blue and green indexes and compare ranking + citation sets before cutover.

**Reference scenario (v3):** “How do I run tests?”

Use the same scenario, but now assert scaled retrieval properties:

- Hybrid recall: the published SOP must be returned even when the query uses synonyms (“inner loop”, “unit test phases”, “phase zero”).
- Graph mode: local/entity mode can expand from “Ameide CLI” / “Testing SOP” to linked elements (bounded, explainable traces).
- CI evaluation: the golden suite for this scenario is a required gate for index/schema changes (drift + citation correctness).
