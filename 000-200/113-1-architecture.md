# Artifact-Centric Graph Architecture - Core Architecture

## Architecture Decision Records

### ADR-001: Event Store as Single Source of Truth
- **Status**: Accepted
- **Decision**: Event Store owns all truth; graph is read-only projection
- **Consequences**: Graph can be dropped and rebuilt anytime from events

### ADR-002: Repository as Security Boundary
- **Status**: Accepted
- **Decision**: Tenant → Repository is the only security boundary
- **Consequences**: All queries must start from this root; no cross-graph edges

### ADR-003: Kafka Partitioning Strategy
- **Status**: Accepted
- **Decision**: Partition by graph_id; one consumer per partition
- **Consequences**: Per-graph ordering guaranteed; horizontal scaling by graph

### ADR-004: Relationship Uniqueness Strategy
- **Status**: Accepted
- **Decision**: Rely on Kafka ordering + MERGE with edge_key
- **Consequences**: Simpler implementation; depends on proper Kafka configuration

### ADR-005: Typed Edges Behind Feature Flag
- **Status**: Accepted
- **Decision**: Whitelist specific relationships; control via ENABLE_TYPED_EDGES
- **Consequences**: Performance gains for hot paths; fallback to RELATES if disabled

### ADR-006: Outbox Pattern Over Direct Publishing
- **Status**: Accepted
- **Decision**: Use transactional outbox + Debezium CDC
- **Consequences**: Guaranteed delivery; no dual-write problem; slight latency increase

## Core Architecture

### Single Source of Truth
- Event Store is the only truth
- Graph is a read-only projection (can be dropped and rebuilt)
- No graph write APIs - only the projector writes to graph
- Every node/edge carries `origin_stream`, `origin_seq`, `lastEventSeq`

### Security Model
- Tenant owns Repositories (no separate Workspace concept)
- Repository is the security boundary
- All queries anchor at `(Tenant)->(Repository)`
- Service layer validates JWT → tenant/repo claims

### Consistency Strategy
- Transactional Outbox pattern for dual-write safety
- Idempotent projections with `lastEventSeq` guards
- Eventually consistent with SLO: p95 lag ≤ 5s
- **Effectively-once semantics**: At-least-once delivery with idempotent processing
- Kafka offsets committed AFTER successful Neo4j transaction

### Rollback Guarantees
- Feature flag: `GRAPH_READS_ENABLED` (kill switch)
- Rebuild command: `graphctl rebuild --graph=<id>`
- Dark reads: Shadow execution in staging
- Graph queries return 501 when disabled

## Graph Model

### Node Structure
```cypher
:Tenant {id, name}
:Repository {id, name, visibility, createdAt}
:Artifact {id, kind, name, lastEventSeq, origin_stream, origin_seq, deleted, deletedAt, deletedBy}
:Relation {id, kind, context, valid_from, valid_to}  // Reified when needed
```

### Temporal Semantics for Nodes

#### Soft Delete Pattern
```cypher
// Soft delete marks nodes as deleted without removal
(:Artifact {
  id: "artifact-123",
  deleted: true,              // Soft delete flag
  deletedAt: datetime(),       // When deleted
  deletedBy: "user-456",       // Who deleted
  deletedReason: "Obsolete",   // Optional reason
  lastEventSeq: 42            // Points to delete event
})

// Queries must filter deleted nodes
MATCH (a:Artifact)
WHERE coalesce(a.deleted, false) = false
RETURN a
```

#### Temporal Validity Pattern
```cypher
// Nodes can have temporal validity windows
(:Artifact {
  id: "artifact-789",
  validFrom: datetime("2024-01-01T00:00:00Z"),
  validTo: datetime("2024-12-31T23:59:59Z"),
  supersededBy: "artifact-890",  // Link to replacement
  version: 2
})

// Query for artifacts valid at specific time
MATCH (a:Artifact)
WHERE datetime($asOf) >= a.validFrom 
  AND (a.validTo IS NULL OR datetime($asOf) <= a.validTo)
  AND coalesce(a.deleted, false) = false
RETURN a
```

#### Version Chain Pattern
```cypher
// Version chain using edges
(:Artifact {id: "v1"})-[:NEXT_VERSION]->(:Artifact {id: "v2"})-[:NEXT_VERSION]->(:Artifact {id: "v3"})

// Find latest version
MATCH (a:Artifact {id: $rootId})
MATCH path = (a)-[:NEXT_VERSION*0..]->(:Artifact)
WHERE NOT (last(nodes(path)))-[:NEXT_VERSION]->()
RETURN last(nodes(path)) as latestVersion

// Find version at point in time
MATCH (a:Artifact {id: $rootId})
MATCH path = (a)-[:NEXT_VERSION*0..]->(:Artifact)
WHERE ALL(n IN nodes(path) WHERE n.createdAt <= datetime($asOf))
RETURN last(nodes(path)) as versionAsOf
```

### Edge Patterns

#### Simple Relations
Direct edges with unique key per from/kind/to:
```cypher
(Artifact)-[:RELATES {kind, edge_key}]->(Artifact)
```
- Edge key: `sha256(tenant|repo|from|kind|to)`
- Properties: kind, lastEventSeq, createdAt, createdBy, properties

#### Rich Relations
Reified nodes for multiple, temporal, weighted relations:
```cypher
(Artifact)-[:OUT]->(Relation)-[:IN]->(Artifact)
```
- Multiple relations of same kind with different contexts
- Temporal: valid_from/valid_to for "as of" queries
- Contextual: Different contexts for same from/to/kind triple

### Invariants
- No cross-graph edges between artifacts
- No cross-tenant edges (implied by above)
- All queries start at `(Tenant)->(Repository)`

## Edge Key Canonicalization

```python
def compute_edge_key(tenant_id: str, repo_id: str, from_id: str, kind: str, to_id: str) -> str:
    """Compute canonical edge key for relationship uniqueness."""
    # Normalize all inputs
    tenant_id = tenant_id.strip().lower()
    repo_id = repo_id.strip().lower()
    from_id = from_id.strip().lower()
    kind = kind.strip().lower()  # Always use canonical string
    to_id = to_id.strip().lower()
    
    # Concatenate with pipe separator
    canonical = f"{tenant_id}|{repo_id}|{from_id}|{kind}|{to_id}"
    
    # Compute SHA256
    return hashlib.sha256(canonical.encode('utf-8')).hexdigest()
```

**Requirements:**
- Always lowercase all inputs before hashing
- Always strip whitespace from all inputs
- Kind must be canonical string (e.g., `"bpmn::sequenceflow"` - lowercase)
- Separator is pipe (`|`) character
- Hash is lowercase hex string

## Relationship Uniqueness

Neo4j 5+ doesn't enforce uniqueness on relationships. Our approach:

### Kafka Partitioning + MERGE (Chosen)
- Kafka partitions by `graph_id` ensure sequential processing
- Use `MERGE` with computed `edge_key`
- Simple, leverages existing infrastructure

```cypher
MERGE (from)-[r:RELATES {edge_key: $edge_key}]->(to)
  ON CREATE SET r.kind = $kind, r.properties = $properties
```

## Portability Rules

### Write Path Contract
- **NO APOC procedures** in write operations - plain Cypher only
- **Parameterized queries only** - no string interpolation
- **Reserved properties** that must not be used by applications:
  - `id`, `kind`, `lastEventSeq`
  - `origin_stream`, `origin_seq`
  - `createdAt`, `updatedAt`, `deleted`
  - `createdBy`, `updatedBy`, `deletedBy`

### Provider Compatibility
- Neo4j-specific optimizations (typed edges) must be feature-flagged
- All queries must work with generic RELATES edges
- No vendor-specific functions in core projections

## Performance Strategy

### Typed Edges for Hot Paths
Whitelist of typed edges for 10-50x performance improvement:

```python
TYPED_EDGE_MAPPING = {
    "structural::haspart":      "HAS_PART",
    "doc::haschapter":          "DOC_HAS_CHAPTER",
    "doc::hassection":          "DOC_HAS_SECTION",
    "bpmn::sequenceflow":       "BPMN_SEQUENCE_FLOW",
    "bpmn::messageflow":        "BPMN_MESSAGE_FLOW",
    "archimate::realizes":      "ARCHIMATE_REALIZES",
    "archimate::serves":        "ARCHIMATE_SERVES",
}
```

When `ENABLE_TYPED_EDGES=true` and kind is whitelisted, create both:
1. Generic `RELATES` edge with full properties
2. Typed edge for performance-critical queries