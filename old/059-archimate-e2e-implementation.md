# ArchiMate End-to-End Implementation

## Overview
Implement comprehensive end-to-end testing infrastructure for ArchiMate models, following the same patterns established for BPMN tests. This will ensure robust validation, transformation, and integration testing for ArchiMate models throughout the platform.

## Background
ArchiMate is a key enterprise architecture modeling language supported by the platform. While we have core domain models and validators, we lack comprehensive end-to-end tests that validate the complete lifecycle of ArchiMate models from creation through visualization.

## Goals
1. Create comprehensive ArchiMate service infrastructure with proto definitions
2. Implement robust integration tests covering all CRUD operations
3. Add security and error handling tests
4. Create UI integration tests for ArchiMate modeling
5. Ensure performance and scalability for large enterprise models

## Technical Design

### 1. Proto Service Definition
Create `packages/ameide_core_proto/proto/ameide_core_proto/archimate/v1/archimate_service.proto`:
```protobuf
service ArchiMateService {
  rpc CreateArchiMateModel(CreateArchiMateModelRequest) returns (CreateArchiMateModelResponse);
  rpc GetArchiMateModel(GetArchiMateModelRequest) returns (GetArchiMateModelResponse);
  rpc UpdateArchiMateModel(UpdateArchiMateModelRequest) returns (UpdateArchiMateModelResponse);
  rpc DeleteArchiMateModel(DeleteArchiMateModelRequest) returns (DeleteArchiMateModelResponse);
  rpc ListArchiMateModels(ListArchiMateModelsRequest) returns (ListArchiMateModelsResponse);
  rpc ValidateArchiMateModel(ValidateArchiMateModelRequest) returns (ValidateArchiMateModelResponse);
  rpc ExportArchiMateModel(ExportArchiMateModelRequest) returns (ExportArchiMateModelResponse);
  rpc ImportArchiMateModel(ImportArchiMateModelRequest) returns (ImportArchiMateModelResponse);
}
```

### 2. Domain Model Extensions
Extend `packages/ameide_core-domain/src/ameide_core_domain/archimate/`:
- `ArchiMateModel` - Complete model representation
- `ArchiMateDocument` - Storage model with XML content
- Integration with existing `ArchiMateElement`, `ArchiMateRelationship`

### 3. Service Implementation
Create `services/core-api-grpc-rest/grpc_services/archimate_service.py`:
- Implement all RPC methods
- Use graph pattern for storage
- Integrate with ArchiMateValidator
- Add proper error handling and logging

### 4. Test Structure

#### Integration Tests (`tests/integration/test_archimate_service_integration.py`)
- In-memory graph implementation
- CRUD operation tests
- Validation tests using ArchiMateValidator
- Error scenarios:
  - Malformed XML
  - Invalid relationships
  - Cross-layer constraint violations
  - Concurrent modifications
- Security tests:
  - SQL injection prevention
  - XSS prevention
  - Tenant isolation
  - XXE attack prevention
- Performance tests for large models (1000+ elements)

#### E2E Pipeline Tests (`tests/e2e/test_archimate_pipeline.py`)
- Model transformation tests
- Complex relationship validation
- Edge cases:
  - Orphaned elements
  - Circular dependencies
  - Invalid layer assignments
  - Missing required relationships
- Viewpoint filtering tests
- Export/import round-trip tests

#### UI Integration Tests (`tests/e2e/test_ui_archimate_integration.py`)
- Model creation via UI with data-testid attributes
- Element drag-and-drop across layers
- Relationship creation with validation
- Viewpoint switching
- Search and filter functionality
- Concurrent user operations

#### Container Tests (`tests/integration/test_archimate_e2e_container.py`)
- Multi-service orchestration
- Health checks and retry logic
- Integration with IPA service
- Integration with visualization service

### 5. Test Fixtures and Utilities
Create `tests/shared/archimate_fixtures.py`:
- Sample models for each viewpoint
- Element generators for all 40+ types
- Relationship generators with valid constraints
- Large model generators for performance testing

### 6. Validation Coverage
Ensure tests cover:
- All 7 ArchiMate layers
- 40+ element types
- 12 relationship types
- Cross-layer constraints
- Access mode validation
- Influence strength validation
- Viewpoint-based filtering

## Implementation Steps

1. **Phase 1: Proto and Domain Models**
   - Create proto service definition
   - Extend domain models
   - Generate proto stubs

2. **Phase 2: Service Implementation**
   - Implement ArchiMate service
   - Create graph abstraction
   - Add to core-api-grpc-rest

3. **Phase 3: Integration Tests**
   - Implement in-memory graph
   - Add CRUD tests
   - Add validation tests
   - Add error and security tests

4. **Phase 4: E2E Tests**
   - Pipeline tests
   - UI integration tests
   - Container tests

5. **Phase 5: Performance and Scale**
   - Large model tests
   - Concurrent operation tests
   - Benchmark performance

## Success Criteria
- All ArchiMate operations have >95% test coverage
- Tests follow SOLID principles and Given/When/Then pattern
- All security vulnerabilities are tested
- Performance tests pass with models of 1000+ elements
- UI tests use resilient data-testid selectors
- Container tests demonstrate proper service integration

## Dependencies
- Existing ArchiMate domain models
- ArchiMateValidator implementation
- Proto generation infrastructure
- Test container utilities

## Risks and Mitigations
- **Risk**: ArchiMate spec complexity
  - **Mitigation**: Focus on most common patterns first, expand incrementally
- **Risk**: Performance with large models
  - **Mitigation**: Implement pagination and lazy loading
- **Risk**: UI test brittleness
  - **Mitigation**: Use data-testid attributes exclusively

## Future Enhancements
- ArchiMate model diffing
- Model migration between versions
- Advanced viewpoint customization
- Model analysis and metrics
- Integration with external ArchiMate tools

## References
- ArchiMate 3.1 Specification
- Existing BPMN test implementation
- ArchiMate graph constraints