# Enterprise Architecture Graph Projection

## 1 · Purpose & Background

This epic extends the Unified Artifact Framework (backlog item 067) to support hyperlinking between diagram elements and a wider Enterprise Architecture (EA) graph. By projecting the event stream into a graph database, we enable AI systems and analysts to traverse relationships across the entire architecture landscape, providing rich context beyond individual diagrams.

## 2 · Vision Statement

> *"Transform isolated architecture artifacts into an interconnected knowledge graph where every BPMN process, ArchiMate component, and specification document is linked to the broader enterprise architecture, enabling AI-powered insights and impact analysis."*

## 3 · Scope

| In‑Scope | Out‑of‑Scope |
|----------|--------------|
| • Graph database projection from artifact event streams<br>• Hyperlinking commands (LinkElement, UnlinkElement)<br>• Graph traversal APIs for AI context enrichment<br>• Consistency patterns between stores<br>• Read-optimized GraphQL/Cypher endpoints<br>• Link validation and broken link detection | • Modifying core artifact framework<br>• Real-time graph updates (eventual consistency is acceptable)<br>• Graph-based authorization (use existing RBAC)<br>• Automatic link discovery<br>• Graph visualization UI |

## 4 · Goals & Non‑Goals

### 4.1 Goals

1. **Rich Context**: Enable AI to traverse from a BPMN task to related systems, capabilities, and requirements
2. **Impact Analysis**: Understand downstream effects of changes across the enterprise architecture
3. **Relationship Integrity**: Maintain consistency between artifact elements and EA graph entities
4. **Query Performance**: Sub-second graph traversals for typical AI context queries
5. **Incremental Adoption**: Works alongside existing artifact framework without disruption

### 4.2 Non‑Goals

* Replace the event store as source of truth
* Provide real-time graph updates (5-10 second lag acceptable)
* Support arbitrary graph schemas (fixed EA graph)
* Handle non-EA relationships (e.g., social graphs)

## 5 · Technical Architecture

### 5.1 High-Level Design

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Artifact       │────▶│   Projection    │────▶│  Graph DB       │
│  Event Stream   │     │   Service       │     │  (Neo4j)        │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                               │                          │
                               ▼                          ▼
                        ┌─────────────────┐     ┌─────────────────┐
                        │  PostgreSQL     │     │  GraphQL API    │
                        │  (Link Metadata)│     │  (Read Model)   │
                        └─────────────────┘     └─────────────────┘
```

### 5.2 New Command Types

```protobuf
// Additional commands for artifact hyperlinking
message LinkElementCommand {
  string element_id = 1;
  string graph_entity_id = 2;  // EA graph entity UUID
  LinkType link_type = 3;
  map<string, string> metadata = 4;  // Optional link metadata
}

message UnlinkElementCommand {
  string element_id = 1;
  string graph_entity_id = 2;
}

enum LinkType {
  LINK_TYPE_UNSPECIFIED = 0;
  LINK_TYPE_IMPLEMENTS = 1;      // Process implements capability
  LINK_TYPE_USES = 2;            // Component uses system
  LINK_TYPE_DEPENDS_ON = 3;      // Element depends on another
  LINK_TYPE_DOCUMENTED_BY = 4;   // Element documented by spec
  LINK_TYPE_REALIZES = 5;        // Component realizes requirement
}
```

### 5.3 Graph Schema

```cypher
// Node types
(artifact:Artifact {
  id: String,
  tenant_id: String,
  kind: String,
  name: String,
  current_revision: String
})

(element:Element {
  id: String,
  artifact_id: String,
  type: String,
  name: String
})

(ea_entity:EAEntity {
  id: String,
  graph: String,
  type: String,  // System, Capability, Component, Requirement
  name: String
})

// Relationships
(artifact)-[:CONTAINS]->(element)
(element)-[:IMPLEMENTS|USES|DEPENDS_ON|DOCUMENTED_BY|REALIZES]->(ea_entity)
(ea_entity)-[:RELATED_TO]->(ea_entity)
```

### 5.4 Projection Service

```python
# packages/ameide_core-services/src/ameide_core_services/projections/graph_projection.py
from ameide_core_platform_common.observability import business_operation

class GraphProjectionService:
    """Projects artifact events to graph database."""
    
    def __init__(
        self,
        event_store: ArtifactEventStore,
        graph_db: Neo4jClient,
        link_validator: LinkValidator
    ):
        self._event_store = event_store
        self._graph = graph_db
        self._validator = link_validator
    
    @business_operation("projection.graph.update")
    async def process_events(self, from_sequence: int) -> int:
        """Process new events and update graph."""
        events = await self._event_store.get_events_after(from_sequence)
        
        for event in events:
            if event.type == "LinkElementCommand":
                await self._process_link(event)
            elif event.type == "UnlinkElementCommand":
                await self._process_unlink(event)
            elif event.type in ["CreateElementCommand", "DeleteElementCommand"]:
                await self._sync_element(event)
        
        return events[-1].sequence if events else from_sequence
    
    async def get_element_context(
        self, 
        element_id: str, 
        depth: int = 2
    ) -> GraphContext:
        """Get graph context for AI analysis."""
        query = """
        MATCH (e:Element {id: $element_id})
        OPTIONAL MATCH path = (e)-[*1..$depth]-(related)
        RETURN e, path, related
        """
        
        result = await self._graph.query(
            query, 
            element_id=element_id, 
            depth=depth
        )
        
        return self._build_context(result)
```

### 5.5 AI Integration Points

```python
# AI context enrichment example
async def analyze_bpmn_process(artifact_id: str, ai_service: AIService):
    # 1. Get artifact from event store
    artifact = await artifact_service.get_artifact(artifact_id)
    
    # 2. Get graph context
    context = await graph_service.get_artifact_context(
        artifact_id=artifact_id,
        include_elements=True,
        include_linked_entities=True,
        depth=3
    )
    
    # 3. Build enriched prompt
    prompt = f"""
    Analyze this BPMN process:
    - Artifact: {artifact.name}
    - Elements: {len(context.elements)}
    - Linked Systems: {context.linked_systems}
    - Capabilities: {context.linked_capabilities}
    - Related Requirements: {context.requirements}
    
    Consider the enterprise context when analyzing...
    """
    
    # 4. Get AI insights with full context
    return await ai_service.analyze(prompt, context)
```

## 6 · Implementation Considerations

### 6.1 Consistency Patterns

1. **Eventually Consistent**: Graph lags event store by seconds
2. **Compensation**: Handle retrospective link corrections
3. **Validation**: Verify EA entities exist before linking
4. **Orphan Detection**: Regular sweeps for broken links

### 6.2 Performance Optimization

1. **Batch Processing**: Group graph updates in transactions
2. **Incremental Updates**: Only process new events
3. **Caching**: Cache frequent traversal patterns
4. **Indexes**: Create indexes on common query paths

### 6.3 Security

1. **Tenant Isolation**: Graph queries respect tenant boundaries
2. **Link Authorization**: Users can only link to authorized EA entities
3. **Audit Trail**: All link operations in event stream
4. **Read Access**: Separate read permissions for graph queries

## 7 · Success Metrics

| Metric | Target |
|--------|--------|
| Graph query response time (p95) | < 500ms |
| Projection lag (p95) | < 10s |
| Link validation accuracy | > 99% |
| AI context enrichment improvement | 40% more relevant insights |

## 8 · Dependencies

* Unified Artifact Framework (067) must be deployed
* EA Repository API must expose entity metadata
* Graph database infrastructure (Neo4j cluster)
* AI services must support context injection

## 9 · References

* Backlog item 067 — Unified Artifact Framework
* Enterprise Architecture Metamodel specification
* Neo4j Graph Data Science documentation
* GraphQL Federation patterns for distributed graphs