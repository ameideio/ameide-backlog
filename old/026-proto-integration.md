# 026 – Core-Proto Integration Across Agents & Workflows

Status: **In Progress**  
Author: platform-architecture  
Created: 2025-07-23  
Updated: 2025-07-24  

## Progress Update (2025-07-24)

### Completed
- ✅ Proto hygiene: Basic cleanup and organization
- ✅ Unified enums: Used across API implementations
- ✅ API Integration: All new API routes use proto types directly
- ✅ Service definitions: WorkflowService, AgentService, TransformationService

### In Progress
- 🔄 Route generation from proto (generate_routes.py created)
- 🔄 Full proto-driven OpenAPI spec generation

### Remaining
- ⏳ Service adapters for actual runtime integration
- ⏳ Snapshot testing for proto schema changes
- ⏳ Schema versioning strategy

## Context

`core-proto` defines vendor-neutral ServiceRequest / ServiceResponse DTOs and common
enums (graph, runtime, agent messaging, etc.).  It is **the public integration
surface that travels on the wire** – consumed by platform client libraries,
external integrators *and* our own services.  It remains persistence-agnostic;
store schemas live in `core-storage` (backlog 027) while rich business logic
lives in `core-domain` (backlog 029).

Right now no agent/workflows package imports it and the module itself carries
place-holders and duplicated concepts.  This backlog item makes `core-proto`
fully authoritative and integrates it across services while staying compliant
with CLAUDE.md guidelines.

## Problem

1. Duplicate concepts: agent/workflows packages re-declare `AgentStatus`,
   `DeploymentStatus`, retry structures, etc.
2. Builders and runtime generators pass around ad-hoc dicts instead of strongly
   typed `ServiceRequest` / `RuntimeDeployRequest` contracts.
3. `core-proto` itself needs clean-up (ruff, mypy, versioning, dead code).

## Goals

* `core-proto` becomes the **single source of truth** for all cross-service DTOs.
* Agent + Workflow stacks import those enums / models instead of duplicating.
* Builder CLIs natively emit/consume proto objects (e.g., `RuntimeDeployRequest`).
* All changes follow CLAUDE guidelines (# refactor-first, SOLID, one tool-chain,
  test-loop, zero hacks).

## Work-Streams

### A. Proto Package Hygiene

1. Delete placeholder `transformation.py` or implement minimal DTOs.
2. Run `ruff --fix` + `mypy --strict`; fix unused imports, regex patterns, forward
   reference rebuilds (`model_rebuild()`).
3. Split oversized modules:
   * `agent.py` → `agent/execution.py`, `agent/definition.py`, `agent/events.py`.
   * Re-export in `agent/__init__.py` to avoid breaking clients.
4. Add cross-field validators (e.g., `BatchRequest.max_batch_size`,
   `RuntimeDeployRequest.replicas`).
5. Introduce `schema_version: str` on `AmeideModel` for evolutionary tracking.
6. Add snapshot tests that serialise every public DTO to JSON and compare against
   stored fixtures (guideline 8 coverage ≥ 85 %).

### B. Unify Shared Enums & Constants

1. Move duplicates (`AgentStatus`, `DeploymentStatus`, `HealthStatus`, etc.) to
   `core-proto`.  
2. Refactor `core-agent-*` and `core-workflows-*` packages to import them.
3. Delete old enum definitions; add shim imports for one minor release to avoid
   sudden breakage.

### C. Builder & Runtime Integration

1. `AgentBuilder.generate_langgraph` returns `RuntimeDeployRequest` &
   `RuntimeDeployResponse` instead of raw dicts.
2. `WorkflowBuilder.generate_temporal` does the same.
3. Replace in-place `metadata` mutation with creation of a
   `RuntimeDeployRequest` that carries runtime-specific hints.

### D. Service Layer Alignment

1. Introduce thin adapter in each runtime package that converts proto
   requests/responses to framework-native calls (e.g., Temporal client).
2. Use `AmeideService` `Protocol` to type-check service implementations.

### E. Documentation & CI

1. Update architecture docs & diagrams to show proto as central layer.
2. Add `pre-commit` entry for proto tests & snapshots.
3. Ensure `make test` targets run proto snapshot tests.

## Acceptance Criteria

1. No duplicate enum or DTO definition across repo; pylint/ruff fails if new
   duplicates are introduced.
2. All builders compile & tests pass with proto-based request/response objects.
3. `core-proto` test coverage ≥ 85 %, zero lint errors, mypy strict pass.
4. Snapshot tests added and verified in CI.
5. Docs updated, empty/stub modules removed.

## Non-Goals

* Refactoring of runtime template generation beyond replacing hard-coded enums.
* Back-porting proto usage to external closed-source services.

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Wide-spread import churn | Provide shim re-exports for one release cycle |
| Snapshot tests brittle | Store fixtures under versioned directory; bump only on intentional schema change |
| Proto bloat | Enforce module size < 300 LOC (Guideline 9) via CI check |

## Timeline (indicative)

| Week | Deliverable |
|------|-------------|
| W32 | Proto hygiene & tests green |
| W33 | Duplicate enums removed, builders refactored |
| W34 | Runtime adapters emit proto responses |
| W35 | Docs, CI, release notes |

## Implementation Details (Partial)

### A. Proto Package Hygiene ✓

1. **Cleaned up core-proto package**:
   - Fixed imports and forward references
   - Added `validators.py` for cross-field validation
   - Removed placeholder code
   - Added proper type hints throughout

2. **Module Organization**:
   - Created `enums.py` for shared enumerations
   - Created `orchestration.py` for unified graph/node/edge protos
   - Split responsibilities across focused modules

3. **Testing**:
   - Added test suite in `__tests__/` directory
   - Implemented validation tests
   - Added enum consistency tests

### B. Unify Shared Enums & Constants ✓

1. **Created Unified Enums** in `/packages/ameide_core-proto/src/ameide_core_proto/enums.py`:
   - `ModelStatus`: DRAFT, PUBLISHED, DEPRECATED, ARCHIVED
   - `DeploymentStatus`: PENDING, DEPLOYING, DEPLOYED, FAILED, TERMINATING
   - `HealthStatus`: HEALTHY, UNHEALTHY, DEGRADED, UNKNOWN
   - `RetryStrategy`: EXPONENTIAL_BACKOFF, FIXED_DELAY, LINEAR_BACKOFF

2. **Removed Duplicates**:
   - Agent and workflows packages now import from core-proto
   - Deleted duplicate enum definitions
   - Added compatibility imports where needed

### C. Builder & Runtime Integration ✓

1. **Created Orchestration Proto** (`orchestration.py`):
   ```python
   class NodeProto(AmeideModel):
       node_id: str
       node_type: str  # "task", "agent", "gateway", etc.
       node_name: str
       metadata: dict[str, Any]
   
   class GraphProto(AmeideModel):
       graph_id: str
       graph_type: str  # "workflows", "agent", "pipeline"
       nodes: list[NodeProto]
       edges: list[EdgeProto]
   ```

2. **Builder Integration**:
   - Builders now use proto objects internally
   - Added serialization methods (to_proto/from_proto)
   - Validators ensure proto compliance

3. **Agent Model Proto Serialization** (2025-07-24):
   - Created `proto_serialization.py` in core-agent-model
   - Implemented `AgentModelProto` for wire format
   - Added complete serialization for all edge types (simple, conditional, loop, error)
   - Created comprehensive round-trip tests
   - Handles all node types and preserves extensions

### D. Service Layer Alignment (Pending)

- Proto adapters for runtime services not yet implemented
- This requires updating service implementations

### E. Documentation & CI ✓

1. **CI Integration**:
   - Added Makefile with test targets
   - Configured mypy strict mode
   - Added ruff linting configuration

2. **Documentation**:
   - Updated README with proto layer architecture
   - Added docstrings to all public APIs

## Outstanding Work

1. **Complete Service Integration**:
   - Update runtime services to use proto objects
   - Add adapters for Temporal/LangGraph services

2. **Snapshot Testing**:
   - Implement JSON snapshot tests for schema evolution
   - Add fixtures for all proto models

3. **Schema Versioning**:
   - Add schema_version field to AmeideModel
   - Implement migration strategy

---

_Remember Guideline 1 – delete the obsolete, outlaw flaky specs; write once, write right._
