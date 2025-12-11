# 033 – Proto Service Integration & Schema Evolution

Status: **In Progress**  
Author: platform-architecture  
Created: 2025-07-24  
Updated: 2025-07-24
Parent: 026-proto-integration

## Progress Update (2025-07-24)

### Completed
- ✅ API services now use proto objects directly (WorkflowService, AgentService, TransformationService)
- ✅ Proto-based route generation script created (`generate_routes.py`)
- ✅ Service interfaces defined with proper proto types

### In Progress
- 🔄 Full proto-driven OpenAPI generation from schemas
- 🔄 Service adapter patterns being established through SDK

### Remaining
- ⏳ Runtime service adapters (Temporal, LangGraph)
- ⏳ Snapshot testing infrastructure
- ⏳ Schema versioning strategy  

## Context

Following the partial completion of proto integration (026), this backlog item tracks the remaining work needed to fully integrate proto objects into the service layer and establish proper schema evolution practices.

## Problem

1. Runtime services still use ad-hoc dictionaries instead of proto objects
2. No automated schema evolution testing or migration strategy
3. Service adapters for LangGraph and Temporal runtimes not implemented
4. Missing schema versioning strategy for backward compatibility

## Goals

* Complete proto integration into all runtime services
* Establish snapshot testing for schema evolution
* Implement service adapters that bridge proto ↔ runtime formats
* Add schema versioning with migration support

## Work-Streams

### A. Service Adapter Implementation

1. **LangGraph Service Adapter** (`core-agent-model2langgraph`):
   - Create `ProtoAdapter` class that converts `AgentModelProto` → LangGraph graph
   - Handle streaming responses using `ServiceResponse` proto
   - Map proto error types to LangGraph exceptions
   
2. **Temporal Service Adapter** (`core-workflows-model2temporal`):
   - Create adapter for `WorkflowModelProto` → Temporal workflows definitions
   - Map proto retry policies to Temporal retry options
   - Handle activity timeouts and compensations

3. **Common Adapter Interface**:
   ```python
   class ProtoServiceAdapter(Protocol):
       def to_runtime(self, proto: AmeideModel) -> Any: ...
       def from_runtime(self, runtime_obj: Any) -> AmeideModel: ...
       def handle_error(self, error: Exception) -> ServiceError: ...
   ```

### B. Snapshot Testing Infrastructure

1. **Test Framework**:
   - Use pytest-snapshot or similar for JSON serialization tests
   - Store versioned fixtures in `__snapshots__/v1/`, `__snapshots__/v2/`, etc.
   - Add CI job to verify schema compatibility

2. **Schema Evolution Tests**:
   ```python
   def test_agent_model_v1_to_v2_migration():
       v1_data = load_snapshot("agent_model_v1.json")
       v2_model = migrate_v1_to_v2(v1_data)
       assert_snapshot(v2_model, "agent_model_v2.json")
   ```

### C. Schema Versioning Strategy

1. **Version Field**:
   - Add `schema_version: str = "1.0.0"` to `AmeideModel` base
   - Use semantic versioning for schema changes
   - Document breaking vs non-breaking changes

2. **Migration Framework**:
   ```python
   class SchemaMigrator:
       migrations = {
           ("1.0.0", "1.1.0"): migrate_1_0_to_1_1,
           ("1.1.0", "2.0.0"): migrate_1_1_to_2_0,
       }
       
       def migrate(self, data: dict, from_version: str, to_version: str) -> dict:
           # Apply migrations in sequence
   ```

### D. Runtime Service Updates

1. **Update Service Implementations**:
   - Replace dict parameters with proto objects
   - Add proto validation at service boundaries
   - Update error handling to use `ServiceError` proto

2. **Backward Compatibility**:
   - Support both dict and proto inputs for one release cycle
   - Add deprecation warnings for dict usage
   - Update client libraries to use protos

## Acceptance Criteria

1. All runtime services accept and return proto objects
2. Snapshot tests cover 100% of proto models
3. Schema migration framework handles version upgrades
4. Zero regression in existing functionality
5. Performance impact < 5% (measure serialization overhead)

## Non-Goals

* Rewriting runtime engines (keep LangGraph/Temporal as-is)
* Changing wire protocol (still JSON/gRPC)
* Supporting multiple schema versions simultaneously

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Performance overhead | Cache proto conversions, use __slots__ |
| Breaking changes | Extensive snapshot testing, staged rollout |
| Complex migrations | Keep schema changes minimal, test thoroughly |

## Dependencies

* Completion of 026-proto-integration fundamentals
* Core-proto package must be stable
* CI/CD pipeline for snapshot testing

## Timeline

| Week | Deliverable |
|------|-------------|
| W31 | LangGraph adapter + tests |
| W32 | Temporal adapter + tests |
| W33 | Snapshot testing framework |
| W34 | Schema versioning + migrations |
| W35 | Service updates + documentation |

---

_"Refactor first – clean the nearest code & tests, delete the obsolete, outlaw flaky specs" – CLAUDE.md Guideline 1_


# 505 – API Proto-Based Generation Enhancement

Status: **Planned**  
Created: 2025-07-24

## Overview

Enhance the API layer to be fully generated from proto definitions, ensuring consistency and reducing manual coding.

## Current State

- Manual route implementations exist for workflows, agents, transformations
- `generate_routes.py` script created but not fully integrated
- OpenAPI spec generation exists but not proto-driven

## Target State

- All routes auto-generated from proto Request/Response pairs
- OpenAPI spec derived entirely from proto schemas
- Service interfaces generated with proper async patterns
- GraphQL schema generated from proto types

## Implementation Tasks

### 1. Route Generation Pipeline
- [ ] Enhance `generate_routes.py` to handle all patterns
- [ ] Add streaming endpoint generation for agents
- [ ] Generate proper error handling
- [ ] Support file upload endpoints

### 2. OpenAPI Generation
- [ ] Replace manual OpenAPI spec with proto-derived
- [ ] Add proper schema references
- [ ] Include authentication schemas
- [ ] Generate example requests/responses

### 3. Service Interface Generation
- [ ] Generate abstract base classes for services
- [ ] Add proper type hints for all methods
- [ ] Include docstrings from proto
- [ ] Generate test stubs

### 4. GraphQL Schema Generation
- [ ] Map proto types to GraphQL types
- [ ] Generate resolvers from services
- [ ] Add subscription support for streaming
- [ ] Include proper pagination

### 5. CI/CD Integration
- [ ] Add generation step to Makefile
- [ ] Verify no manual changes in CI
- [ ] Auto-update on proto changes
- [ ] Generate migration guides

## Technical Decisions

1. **Merge Strategy**: Generated code in separate files, imported by manual code
2. **Customization**: Use decorators/mixins for custom logic
3. **Versioning**: Track proto version in generated files
4. **Testing**: Generate test cases from proto examples

## Dependencies

- Proto definitions must be complete
- Service layer architecture finalized

## Acceptance Criteria

1. Zero manual route definitions needed
2. OpenAPI spec always in sync with proto
3. Generated code passes all type checks
4. No breaking changes without proto change
5. Documentation auto-generated

## Timeline

Target: 1 week, parallel with SDK Phase 2