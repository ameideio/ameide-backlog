# 535 — MCP Read Optimizations (Semantic Search & Contextual Retrieval)

**Status:** Draft (no repo implementation yet)
**Audience:** Architecture, platform engineering, AI/agent teams
**Scope:** Define **semantic search** as a Projection capability — enabling AI agents to find relevant data using natural language queries via contextual retrieval (pgvector + hybrid search).

**Use with:**
- MCP protocol adapter: `backlog/534-mcp-protocol-adapter.md`
- MCP write optimizations: `backlog/536-mcp-write-optimizations.md`
- Transformation projection: `backlog/527-transformation-projection.md`
- Capability definition: `backlog/528-capability-definition.md`
- Agentic memory contract (v6): `backlog/656-agentic-memory-v6.md`

---

## Implementation progress (current)

Repo status (today):
- [x] This doc defines the target pattern (Projection-owned semantic retrieval; adapter calls Projection, not embedding APIs).
- [ ] No semantic search Projection implementation detected yet (no `semanticSearch` RPCs, pgvector migrations, or indexing pipeline in-tree).

Build-out checklist:
- [ ] Define/confirm proto query surface (e.g. `SemanticSearch`, `GetContext`, `Reindex`, `GetIndexHealth`) and response shape (scores + provenance).
- [ ] Add storage schema + migrations (pgvector for embeddings, tsvector for keyword search, plus metadata tables for chunks/documents).
- [ ] Implement ingestion/indexing pipeline (consume facts/read models → build contextual chunks → embed → upsert index).
- [ ] Implement hybrid retrieval (dense + sparse) with rank fusion + dedupe; add deterministic tie-break rules.
- [ ] Enforce authorization at query time (tenant/role filters; prevent cross-tenant leakage in retrieval).
- [ ] Add observability (index lag, embedding failures, chunk counts, hot queries, cost budgets).
- [ ] Add conformance tests (golden queries + expected top-k; regression harness for chunking and ranking).

## Clarification requests (next steps)

Decide/confirm:
- [ ] Embedding provider strategy (in-process model vs external API; if external, which Integration primitive owns calls + caching + retry).
- [ ] Chunking rules for each artifact type (graph elements vs docs vs code) including relationship traversal depth and maximum token budget per chunk.
- [ ] Multi-tenant partition strategy (single shared index with `tenant_id` filter vs per-tenant tables/indexes) and retention policy.
- [ ] Reranking posture (none in v1 vs lightweight cross-encoder) and latency/cost targets for `semanticSearch`.
- [ ] Re-index triggers (event-driven vs scheduled), backfill strategy, and “index versioning” during schema evolution.

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Service (semantic query), Data Object (contextual chunks, embeddings).
- **Out-of-scope layers:** Strategy/Business (capability design), Technology (vector DB internals).
- **Allowed nouns:** semantic search, contextual retrieval, embedding, chunk, hybrid search, reranking.

## Implementation progress (repo)

- [x] Projection primitives and query services exist (baseline structured reads), e.g. Transformation supports `ListTransformationElements(search_query=...)` in `packages/ameide_core_proto/src/ameide_core_proto/transformation/v1/transformation_service.proto`.
- [ ] No `semanticSearch` query surface (gRPC or MCP tool) implemented yet.
- [ ] No embedding pipeline or contextual chunk generation job implemented.
- [ ] No pgvector schema/migrations (or other vector store) committed for projections.
- [ ] No hybrid retrieval (dense + sparse) merge/dedupe/rerank implementation exists yet.

## Clarifications requested (next steps)

- [ ] Pick the embedding model(s) (provider, dimensions) and the tenancy/isolation model (per-tenant namespace, per-tenant keys, cost controls).
- [ ] Decide where vectors live for v1: Postgres `pgvector` in each Projection DB vs a shared/vector service, and how to handle backups/migrations.
- [ ] Define the first “canonical” semantic search target (Transformation ArchiMate vs SRE incidents/runbooks) to lock the chunk schema and evaluation harness.
- [ ] Decide indexing triggers: event-driven from Domain facts, Projection read-model changes, or scheduled rebuilds (and what “freshness” SLO is required).
- [ ] Define the minimum response contract (scores, citations, stable element IDs, `ReadContext`) so clients can render and cache results consistently.

---

## 0) Problem

When AI agents (Claude Code, browser chat, platform agents) need to find information in capability data (e.g., ArchiMate models), they face a challenge:

**Structured queries require knowing what to ask for:**
```
transformation.listElements(filter={layer: "strategy", type: "Capability"})
```
→ Works if you know the exact layer/type. Fails for exploratory questions.

**Natural language queries need semantic understanding:**
```
"What handles order fulfillment?"
"Show me everything related to customer authentication"
"Which capabilities are affected by the payment gateway?"
```
→ Requires semantic search to find relevant elements.

**The cost of navigation:**
Without semantic search, agents must make multiple expensive queries:
1. List all elements → 2. Filter by keywords → 3. Get relationships → 4. Traverse graph → 5. Repeat

Semantic search provides **direct relevance-based retrieval**.

---

## 1) Definition

**Semantic search** is a Projection capability that enables natural language queries over capability data using:
- **Vector embeddings** for semantic similarity
- **Contextual chunks** that preserve meaning (not orphan fragments)
- **Hybrid retrieval** combining dense (vector) and sparse (keyword) search

Semantic search is **not** a new primitive. It is a query optimization within Projection:

```
Projection Primitive
├── Structured queries (SQL → listElements, getView)
├── Full-text search (tsvector → searchElements)
└── Semantic search (pgvector → semanticSearch, getContext)  ← This document
```

---

## 2) Architectural fit

### 2.1 Semantic search as Projection capability

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Projection Primitive                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐ │
│  │ Structured      │  │ Full-text       │  │ Semantic Search     │ │
│  │ Queries         │  │ Search          │  │ (Contextual         │ │
│  │ (SQL)           │  │ (tsvector)      │  │  Retrieval)         │ │
│  └────────┬────────┘  └────────┬────────┘  └──────────┬──────────┘ │
│           │                    │                      │            │
│           └────────────────────┼──────────────────────┘            │
│                                │                                    │
│                         PostgreSQL                                  │
│                    (with pgvector extension)                        │
└─────────────────────────────────────────────────────────────────────┘
```

**Why Projection owns semantic search:**
1. Projection already owns "read-optimized views for specific access patterns" (per `backlog/520-primitives-stack-v6.md` §Projection)
2. pgvector is just another PostgreSQL index type (like tsvector)
3. Same data source, different query interface
4. No new primitive = less architectural complexity

### 2.2 Data flow

```
Domain Facts
     │
     ▼
Projection (fact consumer)
     │
     ├──► Update structured read models (existing)
     │
     └──► Generate contextual chunks + embeddings (async)
              │
              ▼
         semantic_chunks table (pgvector)
```

Embedding generation is **async** because:
- Embedding API calls are slow (100-500ms)
- Shouldn't block the main projection pipeline
- Can be batched for efficiency

#### 2.2.1 Read-after-write expectations (projections may lag)

Semantic search and contextual retrieval operate on the Projection/indexed view of the model.
After a write:

* The **write tool result** should be treated as the authoritative confirmation of what was created/updated.
* Projection-backed search results may lag briefly depending on indexing/event processing.

Therefore:

* Write tools SHOULD return the created/updated entity (and revision/citation metadata where applicable).
* Agents SHOULD prefer reading by returned `id` if they must verify immediately, rather than assuming search reflects the change instantly.

See also: `backlog/536-mcp-write-optimizations.md` §6.0 for write tool response design.

### 2.3 Consumers

| Consumer | Use Case | Query Type |
|----------|----------|------------|
| AI Agents (MCP) | "Find elements related to order processing" | `semanticSearch` |
| Browser Chat | Conversational queries | `semanticSearch` |
| Platform Agents | "Find capabilities affected by this change" | `semanticSearch` |
| UISurface | Search box with semantic understanding | `semanticSearch` |

All consumers access semantic search via:
- **MCP tools** (agentic access)
- **gRPC QueryService** (direct integration)

---

## 3) Contextual retrieval

### 3.1 The problem with naive chunking

Traditional RAG chunks documents into fragments, but fragments lose context:

```
❌ Naive chunk:
"Handles order processing and fulfillment."

→ What handles it? Which layer? What relationships?
```

### 3.2 Contextual chunks preserve meaning

Following Anthropic's contextual retrieval pattern, each chunk includes context from its source:

```
✓ Contextual chunk:
"OrderService is an ApplicationComponent in the Application layer
of the Commerce capability. It realizes the 'Process Orders' business
process and serves the 'Order Management' application service.
It is used by StorefrontUI and accessed via OrderAPI.
Description: Handles order processing and fulfillment."
```

The context comes from **traversing relationships** during indexing.

### 3.3 Contextual chunk generation for ArchiMate

For each ArchiMate element, generate a contextual chunk:

```
┌─────────────────────────────────────────────────────────────────────┐
│ Contextual Chunk Template                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ {name} is a {type} in the {layer} layer.                           │
│                                                                     │
│ [For each relationship:]                                            │
│ It {verb_phrase} {target_name} ({target_type}).                    │
│                                                                     │
│ [If part of capability:]                                            │
│ It belongs to the {capability_name} capability.                    │
│                                                                     │
│ [If has properties:]                                                │
│ Properties: {key}={value}, ...                                     │
│                                                                     │
│ [If has description:]                                               │
│ Description: {description}                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Relationship verb phrases** (ArchiMate-aware):
| Relationship Type | Verb Phrase |
|-------------------|-------------|
| Realization | "realizes" |
| Serving | "serves" |
| Assignment | "is assigned to" |
| Composition | "is composed of" |
| Aggregation | "aggregates" |
| Flow | "flows to" |
| Triggering | "triggers" |
| Access | "accesses" |
| Association | "is associated with" |

### 3.4 Example contextual chunks

**Strategy layer - Capability:**
```
"Transformation is a Capability in the Strategy layer.
It realizes the 'Change Management' value stream.
It is composed of: Enterprise Repository, ProcessDefinitions (BPMN as files under `processes/**`), Methodology Profiles.
It serves: Platform Engineering, Architecture teams.
Description: The platform's change-the-business capability that captures
what should be built, how it should be built, and how it is governed."
```

**Application layer - Component:**
```
"transformation-v0-domain is an ApplicationComponent in the Application layer.
It belongs to the Transformation capability.
It realizes the 'Transformation Domain' application service.
It serves: transformation-v0-projection, transformation-mcp-adapter.
It accesses: postgres-ameide (database).
It is triggered by: TransformationArchitectureDomainIntent, ScrumDomainIntent.
Description: Canonical writer for ArchiMate models, BPMN definitions,
baselines, and Scrum artifacts."
```

---

## 4) Retrieval pipeline

### 4.1 V1: Practical hybrid retrieval

```
Query: "what manages customer orders?"
   │
   ├─► Embed query (embedding API)
   │
   ├─► Hybrid retrieval (parallel)
   │      ├─ Dense: pgvector cosine similarity on contextual chunks
   │      └─ Sparse: tsvector keyword match
   │
   ├─► Merge & dedupe (rank fusion)
   │
   └─► Return top-k elements with context + scores
```

### 4.2 Retrieval stages

| Stage | V1 (Simple) | V2 (Enhanced) |
|-------|-------------|---------------|
| **Query processing** | Direct embedding | Query rewriting, decomposition |
| **Dense retrieval** | pgvector cosine | pgvector + IVF tuning |
| **Sparse retrieval** | tsvector | BM25 with contextual terms |
| **Fusion** | Reciprocal Rank Fusion (RRF) | RRF with tuned k parameter |
| **Reranking** | None (LLM filters) | Cross-encoder or ColBERT |

**Why RRF is V1 default (not weighted merge):**
- BM25/ts_rank scores and cosine similarity are on **incompatible scales** (0-∞ vs -1 to 1)
- Weighted sum fusion requires careful normalization and is hard to tune
- RRF is **score-agnostic** — fuses by rank position, not raw scores
- Industry standard: Used by Elasticsearch, Vespa, major RAG frameworks

### 4.3 Why hybrid retrieval?

Dense (vector) search alone misses exact matches:
- Query: "OrderService" → May return semantically similar but wrong elements

Sparse (keyword) search alone misses semantic relationships:
- Query: "fulfillment handling" → Misses "OrderService" if description says "order processing"

**Hybrid combines both** for better recall and precision.

---

## 5) MCP tools for semantic search

Add to capability MCP manifests:

```yaml
# In transformation-mcp-manifest.yaml
queries:
  # Existing structured queries
  - name: listElements
    # ...

  # NEW: Semantic search queries
  - name: semanticSearch
    description: Find elements semantically related to a natural language query
    target:
      service: transformation-v0-projection
      port: 50051
      rpc: SemanticQueryService.Search
    input:
      properties:
        query: { type: string, required: true, description: "Natural language query" }
        top_k: { type: integer, default: 10, description: "Number of results" }
        min_score: { type: number, default: 0.5, description: "Minimum similarity score" }
        # ReadContext — required for citation discipline (see §5.2)
        read_context:
          type: object
          description: "Point-in-time read context for reproducible queries"
          properties:
            baseline_id: { type: string, description: "Query against a specific baseline" }
            as_of: { type: string, format: date-time, description: "Query as of a specific timestamp" }
        filters:
          type: object
          properties:
            layer: { type: string, enum: [strategy, business, application, technology] }
            element_type: { type: string }
            capability: { type: string }
    output:
      type: array
      items:
        type: object
        properties:
          element_id: { type: string }
          repository_id: { type: string, description: "Repository scope of the result" }
          element_name: { type: string }
          element_type: { type: string }
          layer: { type: string }
          score: { type: number }
          context: { type: string, description: "Contextual chunk that matched" }
          # Canonical citation + read context echo — always returned
          version_id: { type: string, description: "Immutable version of the element (303/656)" }
          read_context:
            type: object
            properties:
              selector: { type: string }
              baseline_id: { type: string }
              as_of: { type: string, format: date-time }

  - name: getContext
    description: RAG-style retrieval - get relevant context for answering a question about the architecture
    target:
      service: transformation-v0-projection
      port: 50051
      rpc: SemanticQueryService.GetContext
    input:
      properties:
        query: { type: string, required: true }
        max_tokens: { type: integer, default: 4000, description: "Max context size" }
        include_relationships: { type: boolean, default: true }
    output:
      type: object
      properties:
        context: { type: string, description: "Assembled context for LLM" }
        sources:
          type: array
          items:
            type: object
            properties:
              element_id: { type: string }
              element_name: { type: string }
              relevance: { type: number }
```

### 5.1 Tool usage examples

**semanticSearch** - Find relevant elements:
```
User: "What handles order fulfillment?"

Agent: [invokes transformation.semanticSearch]
{
  "query": "order fulfillment handling",
  "top_k": 5,
  "filters": { "layer": "application" }
}

Response:
[
  { "element_id": "...", "element_name": "OrderService", "score": 0.89, "context": "..." },
  { "element_id": "...", "element_name": "FulfillmentProcess", "score": 0.85, "context": "..." },
  ...
]
```

**getContext** - Get assembled context for complex questions:
```
User: "Explain how the platform handles customer authentication"

Agent: [invokes transformation.getContext]
{
  "query": "customer authentication flow security",
  "max_tokens": 4000,
  "include_relationships": true
}

Response:
{
  "context": "Authentication in the platform is handled by...[assembled from multiple relevant chunks]",
  "sources": [
    { "element_id": "...", "element_name": "AuthService", "relevance": 0.92 },
    { "element_id": "...", "element_name": "IdentityProvider", "relevance": 0.87 },
    ...
  ]
}
```

### 5.2 ReadContext and citation discipline

**All query tools MUST support a common `read_context` parameter** to enable reproducible, auditable queries:

```yaml
# ReadContext — common input for all query tools
read_context:
  type: object
  description: "Point-in-time read context for reproducible queries"
  properties:
    selector:
      type: string
      description: "Selector mode (tool-specific default; for agent memory tools, default is published)"
      enum: ["head", "published", "baseline_id", "as_of"]
    baseline_id:
      type: string
      description: "Query against a specific published baseline"
    as_of:
      type: string
      format: date-time
      description: "Query as of a specific timestamp (temporal query)"
```

**Citation metadata — MUST be returned in all query responses:**

```yaml
# Canonical citation — required in all query tool outputs
repository_id:
  type: string
  description: "Repository scope of the cited item"
element_id:
  type: string
  description: "Stable element identifier"
version_id:
  type: string
  description: "Immutable element version (303/656); citeable and reproducible"

# Read context echo (required for replay)
read_context:
  type: object
  description: "The effective read context used to compute this result"
  properties:
    selector: { type: string }
    baseline_id: { type: string }
    as_of: { type: string, format: date-time }
```

**Why citation discipline matters:**

1. **Reproducibility** — Agents can re-run queries and get consistent results
2. **Auditability** — Each answer can be traced to a specific version
3. **Cross-call consistency** — Multiple queries in a session see the same state
4. **Governance** — Impact analysis and baseline comparisons require point-in-time queries

**Implementation requirement:**
- If `read_context` is omitted, the tool MUST apply a documented default:
  - for agent memory context tools (656), default is `selector=published` (published baseline pointer)
  - for interactive browse/editor queries, default may be `selector=head`
- If `baseline_id` is provided, query against that baseline's snapshot
- If `as_of` is provided, use temporal query semantics (bi-temporal if supported)
- Always include citation metadata in response, even for HEAD queries

---

## 6) Implementation

### 6.1 Database schema

```sql
-- Semantic chunks table (pgvector)
CREATE TABLE semantic_chunks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    organization_id UUID NOT NULL,
    repository_id UUID NOT NULL,

    -- Source element/version reference (303)
    element_id UUID NOT NULL,
    version_id UUID NOT NULL,
    element_type VARCHAR(100) NOT NULL,
    element_name VARCHAR(500) NOT NULL,
    layer VARCHAR(50) NOT NULL,

    -- Contextual chunk
    chunk_text TEXT NOT NULL,
    chunk_hash VARCHAR(64) NOT NULL,  -- For deduplication/change detection

    -- Vector embedding
    embedding vector(1536),  -- OpenAI ada-002 dimension (configurable)

    -- Full-text search
    tsv tsvector GENERATED ALWAYS AS (to_tsvector('english', chunk_text)) STORED,

    -- Metadata
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),

    -- Constraints (conceptual)
    -- Do not couple this schema to legacy typed tables (e.g., archimate_*).
    -- The implementation MUST be keyed by canonical provenance `{tenant_id, organization_id, repository_id, element_id, version_id}` (303/656).
);

-- Vector similarity index (IVF for scale)
CREATE INDEX semantic_chunks_embedding_idx
    ON semantic_chunks USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);  -- Tune based on data size

-- Full-text search index
CREATE INDEX semantic_chunks_tsv_idx
    ON semantic_chunks USING gin(tsv);

-- Lookup indexes
CREATE INDEX semantic_chunks_tenant_idx ON semantic_chunks(tenant_id);
CREATE INDEX semantic_chunks_element_idx ON semantic_chunks(element_id);
CREATE INDEX semantic_chunks_layer_idx ON semantic_chunks(tenant_id, layer);
CREATE INDEX semantic_chunks_type_idx ON semantic_chunks(tenant_id, element_type);
```

### 6.2 Projection structure

```
transformation-v0-projection/
├── internal/
│   ├── projections/
│   │   ├── archimate_graph.go       # Existing structured projections
│   │   ├── semantic_index.go        # NEW: Maintains semantic_chunks
│   │   └── semantic_index_test.go
│   │
│   ├── services/
│   │   ├── query_service.go         # Existing structured queries
│   │   ├── semantic_service.go      # NEW: Vector + hybrid search
│   │   └── semantic_service_test.go
│   │
│   ├── embedding/
│   │   ├── chunker.go               # Generate contextual chunks
│   │   ├── chunker_test.go
│   │   ├── embedder.go              # Call embedding API
│   │   ├── embedder_test.go
│   │   └── config.go                # Embedding model config
│   │
│   └── retrieval/
│       ├── hybrid.go                # Hybrid retrieval logic
│       ├── fusion.go                # Rank fusion
│       └── hybrid_test.go
```

### 6.3 Chunk generation flow

```go
// semantic_index.go

func (p *SemanticIndexProjection) OnElementCreatedOrUpdated(ctx context.Context, element Element) error {
    // 1. Fetch relationships for context
    relationships, err := p.relationshipRepo.GetForElement(ctx, element.ID)
    if err != nil {
        return err
    }

    // 2. Build contextual chunk
    chunk := p.chunker.BuildContextualChunk(element, relationships)

    // 3. Check if chunk changed (avoid re-embedding unchanged content)
    chunkHash := sha256Hash(chunk)
    existing, _ := p.chunkRepo.GetByElementID(ctx, element.ID)
    if existing != nil && existing.ChunkHash == chunkHash {
        return nil // No change, skip embedding
    }

    // 4. Generate embedding (async via queue for production)
    embedding, err := p.embedder.Embed(ctx, chunk)
    if err != nil {
        return err
    }

    // 5. Upsert semantic chunk
    return p.chunkRepo.Upsert(ctx, SemanticChunk{
        TenantID:    element.TenantID,
        ElementID:   element.ID,
        ElementType: element.Type,
        ElementName: element.Name,
        Layer:       element.Layer,
        ChunkText:   chunk,
        ChunkHash:   chunkHash,
        Embedding:   embedding,
    })
}
```

### 6.4 Hybrid search implementation

```go
// hybrid.go

func (s *SemanticService) Search(ctx context.Context, req SearchRequest) ([]SearchResult, error) {
    // 1. Embed query
    queryEmbedding, err := s.embedder.Embed(ctx, req.Query)
    if err != nil {
        return nil, err
    }

    // 2. Parallel retrieval
    var wg sync.WaitGroup
    var denseResults, sparseResults []SearchResult

    wg.Add(2)

    // Dense (vector) search
    go func() {
        defer wg.Done()
        denseResults, _ = s.vectorSearch(ctx, queryEmbedding, req)
    }()

    // Sparse (keyword) search
    go func() {
        defer wg.Done()
        sparseResults, _ = s.keywordSearch(ctx, req.Query, req)
    }()

    wg.Wait()

    // 3. Merge with Reciprocal Rank Fusion (RRF)
    merged := s.rrfFusion(denseResults, sparseResults)

    // 4. Apply filters and return top-k
    filtered := s.applyFilters(merged, req.Filters)
    return s.topK(filtered, req.TopK), nil
}

func (s *SemanticService) vectorSearch(ctx context.Context, embedding []float32, req SearchRequest) ([]SearchResult, error) {
    // pgvector cosine similarity
    query := `
        SELECT element_id, element_name, element_type, layer, chunk_text,
               1 - (embedding <=> $1) as score
        FROM semantic_chunks
        WHERE tenant_id = $2
          AND 1 - (embedding <=> $1) >= $3
        ORDER BY embedding <=> $1
        LIMIT $4
    `
    // ...
}

func (s *SemanticService) keywordSearch(ctx context.Context, query string, req SearchRequest) ([]SearchResult, error) {
    // Full-text search with ranking
    query := `
        SELECT element_id, element_name, element_type, layer, chunk_text,
               ts_rank(tsv, plainto_tsquery('english', $1)) as score
        FROM semantic_chunks
        WHERE tenant_id = $2
          AND tsv @@ plainto_tsquery('english', $1)
        ORDER BY score DESC
        LIMIT $3
    `
    // ...
}

// rrfFusion implements Reciprocal Rank Fusion
// RRF score = Σ 1/(k + rank_i) for each result set
// k is a smoothing constant (default: 60) that reduces impact of high rankings
func (s *SemanticService) rrfFusion(dense, sparse []SearchResult) []SearchResult {
    k := s.config.RRFConstant // typically 60
    scores := make(map[string]float64)
    results := make(map[string]SearchResult)

    // Score dense results by rank
    for rank, r := range dense {
        scores[r.ElementID] += 1.0 / float64(k+rank+1) // rank is 0-indexed
        results[r.ElementID] = r
    }

    // Score sparse results by rank
    for rank, r := range sparse {
        scores[r.ElementID] += 1.0 / float64(k+rank+1)
        if _, exists := results[r.ElementID]; !exists {
            results[r.ElementID] = r
        }
    }

    // Sort by RRF score
    var fused []SearchResult
    for id, result := range results {
        result.Score = scores[id]
        fused = append(fused, result)
    }
    sort.Slice(fused, func(i, j int) bool {
        return fused[i].Score > fused[j].Score
    })
    return fused
}
```

---

## 7) Configuration

### 7.1 Embedding model configuration

```yaml
# projection-config.yaml
semantic:
  embedding:
    provider: openai  # or "local", "azure", "cohere"
    model: text-embedding-ada-002
    dimensions: 1536
    batch_size: 100

    # For local models
    # provider: local
    # model_path: /models/all-MiniLM-L6-v2
    # dimensions: 384

  retrieval:
    default_top_k: 10
    min_score: 0.5
    rrf_k: 60  # RRF smoothing constant (standard default)

  indexing:
    async: true
    batch_size: 50
    reindex_on_relationship_change: true
```

### 7.2 Environment variables

```bash
# Embedding API
EMBEDDING_PROVIDER=openai
OPENAI_API_KEY=sk-...

# Or for Azure OpenAI
EMBEDDING_PROVIDER=azure
AZURE_OPENAI_ENDPOINT=https://...
AZURE_OPENAI_KEY=...
AZURE_OPENAI_EMBEDDING_DEPLOYMENT=text-embedding-ada-002
```

---

## 8) Acceptance criteria

1. Semantic search is documented as a Projection capability (not a new primitive).
2. Contextual chunks include element identity + relationships + description.
3. Hybrid retrieval combines pgvector (dense) and tsvector (sparse).
4. **Fusion uses Reciprocal Rank Fusion (RRF) by default** (not weighted merge).
5. MCP tools `semanticSearch` and `getContext` are defined.
6. Embedding generation is async and doesn't block fact consumption.
7. Chunk regeneration is triggered when element or relationships change.
8. Filters (layer, type, capability) can be applied to semantic search.
9. Transformation projection serves as reference implementation.

---

## 9) V1 scope

For initial implementation:

| Feature | V1 | Future |
|---------|-----|--------|
| Vector search | pgvector cosine similarity | IVF tuning, HNSW |
| Contextual chunks | Element + immediate relationships | Full graph traversal |
| Hybrid retrieval | pgvector + tsvector | BM25 tuning |
| Fusion | Reciprocal Rank Fusion (RRF) | k parameter tuning |
| Reranking | None (LLM filters results) | Cross-encoder, ColBERT |
| Embedding model | OpenAI ada-002 | Local/fine-tuned models |
| Async indexing | Background goroutine | Dedicated worker queue |

**V1 design decision: RRF over weighted merge**

RRF is preferred for V1 because it is robust without tuning:
- No need to normalize heterogeneous scores
- Works immediately with default k=60
- Weighted merge requires empirical tuning for each retrieval source

---

## 10) Future considerations

- **Reranking**: Add cross-encoder or late-interaction (ColBERT) reranking for precision
- **Query decomposition**: Break complex queries into sub-queries
- **Multi-hop retrieval**: Follow relationships to find indirectly relevant elements
- **Streaming results**: Return results as they're found for large result sets
- **Feedback loop**: Use click/selection data to improve retrieval quality
- **Custom embeddings**: Fine-tune embedding models on ArchiMate vocabulary
- **Cross-capability search**: Federated semantic search across multiple capabilities
