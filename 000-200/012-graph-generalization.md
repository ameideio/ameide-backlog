# Graph Database Generalization

## ✅ IMPLEMENTATION STATUS: COMPLETED (December 2025)

The graph database abstraction layer has been successfully implemented, enabling the transpiler and other components to work with multiple graph database backends through a unified interface.

## Package Architecture Overview

### 1. **core-graph** (Core Abstractions)
This is the foundation package that provides:
- **GraphProvider Protocol**: Abstract interface that all graph database implementations must follow
- **Data Classes**: GraphNode, GraphEdge, GraphResult, GraphQuery
- **MockProvider**: In-memory implementation for testing
- **Common Utilities**:
  - UUID generation (deterministic, cross-graph references)
  - Schema management framework
  - Base loader classes

### 2. **core-graph-age** (Apache AGE Implementation)
Implements the GraphProvider protocol for Apache AGE:
- **AGEProvider**: Async implementation using asyncpg
- **Sync utilities**: For migrations and bulk operations using psycopg
- **Schema migrations**: AGE-specific schema tracking
- **Model loaders**: ArchiMate and BPMN XML importers (optional)

### 3. **core-graph-neo4j** (Neo4j Implementation)
Implements the GraphProvider protocol for Neo4j:
- **Neo4jProvider**: Async implementation using official Neo4j driver
- Full protocol compliance with the same interface as AGE

## How They Fit Together

```
                    ┌─────────────────┐
                    │ core-transpilers│
                    └────────┬────────┘
                             │ uses
                    ┌────────▼────────┐
                    │   core-graph    │
                    │  (protocols)    │
                    └────────┬────────┘
                             │ implements
                 ┌───────────┴───────────┐
                 │                       │
        ┌────────▼────────┐     ┌───────▼────────┐
        │ core-graph-age  │     │core-graph-neo4j│
        └─────────────────┘     └────────────────┘
```

## Benefits Realized

1. **Database Independence**: Transpiler code doesn't know or care which database is being used
2. **Testability**: MockProvider enables fast unit tests without any database
3. **Future-Proof**: New graph databases can be added by implementing GraphProvider
4. **Performance**: Async operations throughout for high-performance queries
5. **Type Safety**: Protocol-based design with full type hints

## Implementation Details

### GraphProvider Protocol

```python
class GraphProvider(Protocol):
    async def connect(self) -> None: ...
    async def disconnect(self) -> None: ...
    async def set_graph(self, graph_name: str) -> None: ...
    async def get_node(self, label: str, node_id: str) -> Optional[GraphNode]: ...
    async def create_node(self, label: str, node_id: str, properties: Dict[str, Any]) -> GraphNode: ...
    async def get_edges(self, edge_type: str, ...) -> List[GraphEdge]: ...
    async def create_edge(self, edge_type: str, ...) -> GraphEdge: ...
    # ... more methods
```

### Common Utilities Generalized

1. **UUID Generation** (`uuid_utils.py`):
   - Deterministic UUID5 generation for cross-graph references
   - `generate_element_uuid(graph_name, element_id)`
   - `generate_edge_uuid(graph_name, source_id, target_id, edge_type)`

2. **Schema Management** (`schema.py`):
   - Predefined schemas for ArchiMate, BPMN, Temporal, PROV-O
   - `SchemaManager` protocol for tracking graph schemas
   - Label and property validation

3. **Loader Framework** (`loaders.py`):
   - `GraphLoader` base class for async model loading
   - `FileLoader` for file-based imports
   - `BatchLoader` for efficient bulk operations

### Testing Strategy

The provider-agnostic design enables powerful testing patterns:

```python
@pytest.mark.parametrize("provider_fixture", [
    "mock_provider",      # Fast, no dependencies
    "age_provider",       # Requires AGE database
    "neo4j_provider"      # Requires Neo4j instance
])
async def test_transpilation(provider_fixture, request):
    provider = request.getfixturevalue(provider_fixture)
    # Same test code works for all providers
```

## Migration from core-age

The original `packages/ameide_core-age` functionality was successfully:
1. Split into generic utilities (moved to `core-graph`)
2. AGE-specific code (moved to `core-graph-age`)
3. All imports updated across the codebase
4. Original package removed to avoid confusion

## Future Extensions

The architecture supports adding new graph databases:
1. Create `core-graph-{database}` package
2. Implement GraphProvider protocol
3. Add to test fixtures
4. No changes needed to transpiler or other consumers

Examples of potential additions:
- `core-graph-neptune` for AWS Neptune
- `core-graph-janusgraph` for JanusGraph
- `core-graph-tigergraph` for TigerGraph
- `core-graph-dgraph` for Dgraph