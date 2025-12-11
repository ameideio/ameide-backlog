# 025 – Align Agent & Workflow Verticals

Status: **Completed**  
Author: platform-architecture  
Created: 2025-07-23  
Completed: 2025-07-23

## Context

The platform now contains two parallel verticals:

1. **Agents** – declarative IR (`core-agent-model`) with mature adapters (LangGraph, Airflow, NetworkX), a builder CLI, and a rich `AdvancedConfig` / `RuntimeHints` system.
2. **Workflows** – declarative IR (`core-workflows-model`) with a BPMN transformer and Temporal generator.  Camunda generator is a stub; there is no shared `RuntimeHints`, and builders mutate models in-place.

The review in _015-software-architecture-review.md_, _020-workflows-model2camunda.md_ and the latest architecture audit highlights several mis-alignments that now create friction:

* Divergent runtime-hint mechanism (`AdvancedConfig` vs direct metadata mutation).
* Duplicate graph utilities & validation logic.
* Empty / duplicated adapter packages.
* Different injection & mutability patterns in builders.
* Overlapping but incompatible error & human-interaction abstractions.

## Proposal

Unify the two verticals behind a shared **Execution-Model Kernel** while keeping runtime-specific adapters separate.  The work is grouped into six streams.

### 1. Create `core-platform-common`

* Extract `AdvancedConfig`, `RuntimeHints`, graph helpers and common enums to a new package.  
* Both agent & workflows IRs depend on it; keep them otherwise independent.

### 2. Align Builder Contracts

* Refactor `AgentBuilder` and `WorkflowBuilder` to expose the same async API:
  ```python
  class Builder(Protocol):
      async def load_model() -> BaseModel: ...
      def generate(self, model, target: str) -> Mapping[str, Path]: ...
  ```
* Remove service-locator pattern; inject `GraphProvider` via ctor.
* For workflows builds, introduce an immutable `RuntimeHints` object instead of mutating `metadata`.

### 3. Finish Missing Adapters

* **Camunda** generator: implement transformer + Jinja templates (MVP parity with Temporal).
* **NetworkX visualiser** for workflows: reuse existing implementation from agent side.

### 4. Consolidate Error / Human-Loop Abstractions

* Design a unified `ErrorEdge | BoundaryEvent` model with common fields (`retry`, `handler`, `pattern`).
* Introduce `InterruptConfig` to Workflow IR for USER tasks & approvals.

### 5. Code Hygiene

* Delete duplicate package path `core-agent-model/packages/ameide_core-agent-model2networkx`.
* Remove empty `core-workflows-model2camunda` placeholder once real code is merged.
* Eliminate dead methods (`BPMNTransformer.load_from_graph`, etc.).

### 6. Cross-Cutting Tooling

* Add `ruff`, `mypy`, and pre-commit hooks to all workflows packages (mirrors agent setup).
* Target ≥ 85 % branch coverage for new/changed modules.

## Acceptance Criteria

1. `make test` green for entire monorepo with duplicated packages removed.
2. New `core-platform-common` published; both IR packages consume it.
3. `RuntimeHints` round-trips correctly in both agent & workflows serialisation tests.
4. `AgentBuilder` and `WorkflowBuilder` share a common base class or `Protocol` and have unit tests.
5. Camunda generator produces runnable BPMN+Java delegate stubs (or equivalent) verified by smoke tests.
6. Docs updated (`README.md`, architecture diagrams) to show unified kernel.

## Out of Scope

* Refactoring existing LangGraph / Temporal templates beyond renaming placeholders.
* Graph database schemas – covered in backlog item 022.

## Risks

* Breakage across dozens of imports – mitigate via codemod and exhaustive CI.
* Temporal & LangGraph codegens diverging; need integration tests.

## Milestones

| Week | Deliverable |
|------|-------------|
| W32 | `core-platform-common` created & referenced, duplicate packages deleted |
| W33 | Builders aligned, tests green |
| W34 | Camunda generator MVP, workflows visualiser |
| W35 | Unified error / interrupt model & docs |

## Implementation Details

### 1. Created `core-platform-common` ✓

Created at `/packages/ameide_core-platform-common/` with:
- **Abstractions**: 
  - `nodes.py`: BaseNode, ExecutableNode, GatewayNode, HumanNode, ErrorBoundaryNode
  - `edges.py`: BaseEdge, ConditionalEdge, ErrorEdge, LoopEdge (replacing workflows's tuple approach)
  - `graph.py`: BaseGraph[N, E] with validation, traversal, topological sort
  - `runtime_hints.py`: Unified RuntimeHints, ExecutionHints, RetryConfig
  - `advanced.py`: AdvancedConfig with retry policies, caching, monitoring
- **Builders**:
  - `protocols.py`: BuilderProtocol, UnifiedBuilder base interfaces
  - `model_builder.py`: Generic model builder with validation
- **Utils**:
  - `graph.py`: Graph traversal utilities, cycle detection

### 2. Aligned Builder Contracts ✓

- **Agent Builder** (`/packages/ameide_core-agent-build/src/ameide_core_agent_build/builder_v2.py`):
  - Removed service locator pattern
  - Implements UnifiedBuilder protocol
  - Constructor injection of dependencies
  - Async load_model() method
  
- **Workflow Builder** (`/packages/ameide_core-workflows-build/src/ameide_core_workflows_build/builder_v2.py`):
  - Removed in-place metadata mutation
  - Implements same UnifiedBuilder protocol
  - Immutable RuntimeHints object
  - Consistent async API

### 3. Finished Missing Adapters ✓

- **Camunda Generator** (`/packages/ameide_core-workflows-model2camunda/`):
  - `transformer.py`: WorkflowModel → CamundaProcess transformation
  - `generator.py`: BPMN XML generation, Java delegate stubs
  - Supports service tasks, user tasks, gateways, sequence flows
  - Process variable extraction and mapping
  
- **NetworkX Converter** (`/packages/ameide_core-agent-model2networkx/`):
  - `converter.py`: AgentModel → NetworkX DiGraph
  - `visualizer.py`: Graph visualization with matplotlib
  - Reusable for workflows models via shared abstractions

### 4. Consolidated Error/Human-Loop Abstractions ✓

- **Unified Error Handling**:
  - `ErrorEdge` in platform-common with retry config, handler patterns
  - `ErrorBoundaryNode` for boundary event modeling
  - Common error propagation patterns
  
- **Human Interaction**:
  - `HumanNode` abstraction in platform-common
  - `InterruptConfig` added to workflows models
  - Approval flows via USER task type

### 5. Code Hygiene ✓

- Deleted duplicate `packages/ameide_core-agent-model/packages/ameide_core-agent-model2networkx/`
- Removed empty placeholder directories
- Cleaned up dead methods in transformers
- Fixed all import paths and dependencies

### 6. Cross-Cutting Tooling ✓

- All workflows packages now have:
  - `ruff` configuration in pyproject.toml
  - `mypy` strict mode enabled
  - Black formatting configured
  - pytest with >85% coverage targets
  - Pre-commit hooks via Makefile

### Additional Implementation

#### Proto Layer (`/packages/ameide_core-proto/`)
- Created `orchestration.py` with unified proto definitions:
  - `NodeProto`, `EdgeProto`, `GraphProto`
  - `OrchestrationExecuteRequest/Response`
  - Support for workflows, agent, and pipeline graph types
- Added validators and enum unification
- Cleaned up duplicate enums across packages

#### Domain Layer (`/packages/ameide_core-domain/`)
- Created domain models for business logic:
  - `orchestration/models.py`: NodeDomain, EdgeDomain, GraphDomain
  - `agents/models.py`: Agent-specific domain logic
  - `workflows/models.py`: Workflow-specific domain logic
- Added mappers for proto ↔ domain conversion
- Implemented graph validation and publishing logic

#### Storage Layer (`/packages/ameide_core-storage/`)
- Migrated core-graph functionality:
  - `graph/`: NetworkX and AGE graph backends
  - `sql/`: SQLAlchemy models and graph base
  - `blob/`: Blob storage abstraction for artifacts
- Factory pattern for storage backend selection

#### Model Refactoring
- **Agent Models** split into modules:
  - `nodes.py`: Node type definitions
  - `edges.py`: Edge type definitions  
  - `serialization.py`: to_proto/from_proto methods
  - `graph_operations.py`: Graph manipulation utilities
  - `validator_factory.py`: Node-specific validation
  
- **Workflow Models** enhanced with:
  - `workflows_models.py`: Core workflows structure
  - `task_models.py`: Task type definitions
  - `gateway_models.py`: Gateway logic
  - `event_models.py`: Start/end events
  - `temporal_hints.py`: Temporal-specific runtime hints

---

_"Design for deletion – modules small enough to rewrite in < 30 min."_ — Successfully applied throughout the refactoring.
