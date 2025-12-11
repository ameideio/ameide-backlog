# 029 – Establish `core-domain` Shared Business Model Package

Status: **Completed**  
Author: platform-architecture  
Created: 2025-07-23  
Completed: 2025-07-23

## Purpose

Create a single, versioned package `core-domain` that holds **business-domain
objects** shared across services (Users, Agents, Workflows, Executions, etc.).

This removes duplication of rich models, aligns invariants, and provides a
stable target for mappings between `core-proto` (wire) and `core-storage`
(persistence).

## Design Principles

* **Pure domain** – no imports from storage adapters, frameworks, or REST
  routers (Guideline 2: SRP).
* Behaviour allowed (methods that mutate immutable copies) but no side-effects
  outside the object.
* Organised by bounded context inside the package.

```
ameide_core_domain/
    users/
        models.py     # User, Role
        mappers.py    # ↔ proto / storage helpers
    agents/
        models.py     # AgentRun, Message, ToolCall
    workflows/
        models.py     # Workflow, Task, Gateway
```

## Tasks

1. **Package skeleton** – Poetry project, ruff/mypy, tests.
2. **Move existing domain-like classes** from agent/workflows packages into
   bounded contexts; keep import shims for one release.
3. **Mapper utilities** – functions to convert between:
   * `core-proto` DTOs ↔ domain objects.
   * domain objects ↔ storage entities (`core-storage`).
4. **Versioning** – add `__version__` and CHANGELOG; services pin minor
   version.
5. **CI** – add coverage (≥ 90 %), ruff, mypy strict.

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| 1 | `packages/ameide_core-domain` published and imported by services |
| 2 | No duplicate business objects remain in service codebases |
| 3 | Builders & APIs map via mappers; tests green |
| 4 | Coverage ≥ 90 %, ruff/mypy strict pass |

## Timeline (indicative)

| Week | Deliverable |
|------|-------------|
| W33 | Package skeleton + Users context migrated |
| W34 | Agents & Workflows contexts migrated |
| W35 | Mappers, shims, CI + docs |

## Implementation Details

### 1. Package Creation ✓

Created `/packages/ameide_core-domain/` with:
- Full Poetry project setup
- Makefile with standard targets
- README documenting domain architecture
- CHANGELOG.md for version tracking

### 2. Bounded Contexts Implementation ✓

#### Users Context (`users/`)
- **models.py**: 
  ```python
  class UserDomain(BaseModel):
      user_id: str
      email: str
      roles: list[RoleDomain]
      preferences: UserPreferencesDomain
      created_at: datetime
      
      def grant_role(self, role: RoleDomain) -> "UserDomain": ...
      def update_preferences(self, prefs: dict) -> "UserDomain": ...
  ```
- **mappers.py**: Proto ↔ Domain conversion utilities

#### Agents Context (`agents/`)
- **models.py**:
  ```python
  class AgentExecutionDomain(BaseModel):
      execution_id: str
      agent_id: str
      status: ExecutionStatus
      messages: list[MessageDomain]
      tool_calls: list[ToolCallDomain]
      
      def add_message(self, message: MessageDomain) -> "AgentExecutionDomain": ...
      def complete(self, result: Any) -> "AgentExecutionDomain": ...
  ```
- **mappers.py**: Conversion to/from AgentProto

#### Workflows Context (`workflows/`)
- **models.py**:
  ```python
  class WorkflowExecutionDomain(BaseModel):
      execution_id: str
      workflows_id: str
      status: ExecutionStatus
      current_task: str | None
      variables: dict[str, Any]
      
      def advance_to_task(self, task_id: str) -> "WorkflowExecutionDomain": ...
      def set_variable(self, key: str, value: Any) -> "WorkflowExecutionDomain": ...
  ```
- **mappers.py**: BPMN/Temporal model conversions

#### Orchestration Context (`orchestration/`)
- **models.py**: Unified orchestration domain models
  ```python
  class GraphDomain(BaseModel):
      graph_id: str
      graph_type: GraphType  # workflows, agent, pipeline
      nodes: dict[str, NodeDomain]
      edges: list[EdgeDomain]
      status: ModelStatus
      metadata: GraphMetadata
      
      def validate_structure(self) -> list[str]: ...
      def publish(self) -> "GraphDomain": ...
      def archive(self) -> "GraphDomain": ...
  ```
- **mappers.py**: GraphProto ↔ GraphDomain conversions

### 3. Design Patterns Applied ✓

1. **Pure Domain Objects**:
   - No framework dependencies
   - No storage/persistence logic
   - No HTTP/gRPC concerns
   
2. **Immutable Updates**:
   - All mutation methods return new instances
   - Original objects remain unchanged
   - Thread-safe by design

3. **Rich Business Logic**:
   - Validation methods
   - State transitions
   - Business rule enforcement

### 4. Mapper Implementation ✓

Each context has bidirectional mappers:
```python
# Proto ↔ Domain
def user_proto_to_domain(proto: UserProto) -> UserDomain: ...
def user_domain_to_proto(domain: UserDomain) -> UserProto: ...

# Storage ↔ Domain  
def user_entity_to_domain(entity: UserEntity) -> UserDomain: ...
def user_domain_to_entity(domain: UserDomain) -> UserEntity: ...
```

### 5. Testing & Quality ✓

- Comprehensive test suite for all models
- Mapper round-trip tests
- Business logic validation tests
- Coverage: Meeting 90% target
- Strict mypy typing throughout
- Ruff linting configured

### 6. Integration Points

The domain layer integrates cleanly:
```
Services → Domain ← Storage
    ↓        ↕        ↑
  Proto   Mappers   SQL/Graph
```

All services now:
1. Receive proto objects
2. Convert to domain models
3. Apply business logic
4. Persist via storage layer
5. Return proto responses

---

_Guideline 9: "Design for deletion – modules small enough to rewrite in < 30 min."_  
Successfully applied - each bounded context is focused and modular.
