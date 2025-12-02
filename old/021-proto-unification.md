# 021 – Unify Agent & Workflow Proto for Inter-Operability

Date: 2025-07-23

## Problem

Agent and Workflow services share a common base (`ServiceRequest`/`Response`) but
still diverge in several areas (status enums, stream events, interrupt
semantics, output field names), making it harder to:

* treat executions uniformly in dashboards and monitoring, and
* embed agents inside workflows (and vice-versa) without ad-hoc mapping code.

## Outcome

`core-proto` offers a **single execution vocabulary** that both
`agent.py` and `workflows.py` build upon, enabling seamless cross-calls and a
unified event stream.

## Proposed Additions / Changes

1. **ExecutionPhase enum (base.py)**
   ```python
   class ExecutionPhase(str, Enum):
       PENDING = "pending"
       RUNNING = "running"
       WAITING = "waiting"      # user input, external signal, timer
       COMPLETED = "completed"
       FAILED = "failed"
       CANCELLED = "cancelled"
   ```
   Both `AgentStatus` and `WorkflowStatus` map to this via helper
   `def to_phase(self) -> ExecutionPhase` methods.

2. **ExecutionStreamEvent (base.py)**
   Generic event for live streaming:
   ```python
   class ExecutionStreamEvent(AmeideModel):
       kind: Literal[
           "message", "task", "metric", "error", "log", "done"
       ]
       payload: Dict[str, Any]
       timestamp: datetime = Field(default_factory=datetime.utcnow)
   ```
   `AgentStreamEvent` and forthcoming `WorkflowStreamEvent` will inherit or
   alias this.

3. **Generic outputs on ServiceResponse**
   Add optional `outputs: Dict[str, Any] = {}` to `ServiceResponse` so both
   `final_answer` (agent) and `output_data` (workflows) become *specialised
   accessors*.

4. **Cross-reference IDs**
   * `TaskExecutionInfo` → add `child_agent_run_id: Optional[str]`.
   * `AgentExecutionResponse` → add `parent_workflows_run_id: Optional[str]`.

5. **Unified interrupt/cancel APIs**
   Introduce `ExecutionInterruptRequest/Response` (base) with
   `execution_type: "agent"|"workflows"`, deprecate separate
   `AgentInterruptRequest` & `WorkflowCancelRequest`.

6. **Standard runtime_overrides**
   Add field to `ServiceRequest`:
   ```python
   runtime_overrides: Dict[str, Any] = Field(default_factory=dict, exclude=True)
   ```
   Concrete services move engine-specific overrides into this dict.

7. **Metrics section alignment**
   Ensure both `AgentQueryResponse` and `WorkflowQueryResponse` use common
   `metrics: Dict[str, Any]` structure.

## Tasks

1. Update `packages/ameide_core-proto` models per changes above.
2. Refactor `agent.py` and `workflows.py` to use new base constructs; provide
   mapping helper functions (`status_to_phase`, etc.).
3. Remove now-redundant fields (`final_answer`, `output_data` kept as
   @property wrappers pointing to `outputs`).
4. Adjust unit tests in `core-proto` and downstream packages.
5. Update documentation and sample payloads.

## Acceptance Criteria

* Both Agent and Workflow gRPC/REST layers exchange `ExecutionStreamEvent` and
  share `ExecutionPhase` mapping.
* CI passes across all packages depending on `core-proto`.
* Dashboards can list mixed agent & workflows executions using only common
  fields (`phase`, `outputs`).

## Notes

Breaking change – as agreed we do *not* maintain backward compatibility.  All
callers must migrate together.
