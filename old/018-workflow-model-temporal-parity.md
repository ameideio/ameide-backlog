# 018 – Workflow Model ⇄ Temporal Feature Parity

Date: 2025-07-23

## Context

`packages/ameide_core-workflows-model` delivers a **vendor-neutral workflows IR**.  Down-stream
modules such as `core-workflows-model2temporal` (not yet implemented) will turn
this IR into executable code for the Temporal platform.  To size the gap we
compared:

* The current IR (`models.py`), and
* Temporal’s public API surface as described by the protobuf definitions in
  `api/temporal/api/*` (v1).

This backlog item enumerates the delta and actions to reach **coverage that is
sufficient for end-to-end generation of idiomatic Temporal workflows**.

## Feature Mapping Snapshot

| Category | Temporal API/Concept | IR Support today? | Notes |
| -------- | ------------------- | ----------------- | ----- |
| **Core Workflow & Activities** | Workflow function, Activity task, TaskQueue, RetryPolicy, Timeouts | ✅ (tasks, retry_policy, task_queue, timeout) | Activity options mostly present but need separate Activity vs LocalActivity distinction. |
| **Signals** | `SignalWorkflow`, signal payload schema | ✅ (`SignalDefinition`) | Missing signal *external* vs *internal* direction and blocking/non-blocking semantics. |
| **Queries** | `QueryWorkflow`, typed request/response | ✅ (`QueryDefinition`) | Need consistency requirement flag (strong/eventual). |
| **Child Workflows** | `ExecuteChildWorkflowCommandAttributes` | 🟡 (`SUB_PROCESS` task type) | Lacks child-specific options (workflows_id, namespace, parent_close_policy, retry_policy). |
| **Timers / Sleep** | `TimerCommandAttributes` | ✅ (`TimerEventDefinition`) | Must add cancellation token support. |
| **Cron / ContinueAsNew** | Cron schedules or size-based continue-as-new | ❌ | Not modelled. |
| **Search Attributes & Memo** | Upsert search attributes, memo map | ❌ | Not modelled. |
| **Versioning & Patching** | `GetVersion`, `WorkflowPatching` | ❌ | Not modelled. |
| **Signals **to child/workflows updates** | Temporal Update (beta) | ❌ | Not modelled. |
| **Cancellation Scopes** | Scoped cancellation, propagated cancellation | ❌ | Not modelled. |
| **Schedules API** | Temporal schedules (server-side cron + catch-up) | ❌ | Not modelled. |
| **Interceptors / Middleware** | Inbound & outbound interceptors | ❌ | Cross-cutting concern – model via runtime hints. |
| **Failure & Compensation** | Retry, Error types, Compensation | 🟡 (ErrorBoundary, CompensationLink) | Need failure type enum, backoff reset points. |
| **Search / Visibility Filters** | Query on search attributes | ❌ | Derived from Search Attributes above. |
| **DataConverter & Encoding** | Custom payload mapper | ❌ | Not modelled. |
| **Workflow Update** | New ‘update’ RPC | ❌ | Not modelled. |

Legend: ✅ covered, 🟡 partially covered, ❌ missing.

## Outcome

1. The Workflow IR contains **first-class objects** for every Temporal feature
   marked 🟡 or ❌ above *without* baking in Temporal semantics—i.e. they should
   be generic enough to map to other runtimes (Camunda 8, Cadence, …).
2. A `core-workflows-model2temporal` transformer/generator can rely solely on
   the IR + `RuntimeHints.temporal` for engine-specific dials.

## Tasks

### 1. IR Extensions

* **ChildWorkflowOptions** – new model referenced from `Task` when
  `task_type == SUB_PROCESS`.
* **ContinueAsNewPolicy** – fields: `after_iterations`, `after_history_bytes`,
  `cron_schedule`, `preserve_search_attributes`.
* **SearchAttributeDefinition** – key, type, description; `upsert_on` list of
  events; link to `WorkflowModel`.
* **MemoDefinition** – key↔value mapping evaluated at workflows start.
* **CronSchedule** – ISO 8601 cron/string + jitter/phase.
* **DataConverterConfig** – name, encoding, metadata.
* **UpdateDefinition** – name, request/response schema, idempotent flag.
* **CancellationScope** – modelled as wrapper task or attribute on Task.
* **VersionMarker** – id, change_id, default_version.

### 2. RuntimeHints.temporal

Extend `RuntimeHints` (introduced in backlog 017) with a `temporal` dict to
store:

```python
class TemporalHints(BaseModel):
    namespace: str | None = None
    worker_build_id: str | None = None
    memo: Dict[str, Any] = {}
    search_attributes: Dict[str, Any] = {}
    interceptors: List[str] = []
```

### 3. Transformer Prototype

* Bootstrap **`core-workflows-model2temporal`** package analogous to
  `model2langgraph`.
* Cover conversion of tasks, signals, queries, retry policy.
* Stub out or `NotImplementedError` for advanced features until implemented.

### 4. Validation & Schema

* Implement cross-field validators (e.g., `ContinueAsNewPolicy` cannot coexist
  with `CronSchedule`).
* JSON-Schema export for the updated IR.

### 5. Migration & Deprecation Plan

* Provide conversion script to move `task.subprocess_ref` → `ChildWorkflowOptions`.
* Keep old fields until v2.0 of IR.

### 6. Documentation & Samples

* Add Temporal-specific sample workflows: Order-Fulfilment with child shipment
  workflows, human approval signal, search attributes, and backoff retry.

## Acceptance Criteria

* All new dataclasses validated with unit tests covering serialization round-trip.
* Temporal generator produces runnable code that passes `temporalio` Python SDK
  smoke tests (within sandbox or CI using the temporal-docker-compose stack).
* Legacy workflows continue to validate under deprecation warnings.

## Progress Review (2025-07-23)

No implementation work has started yet on the Workflow IR extensions or the
`core-workflows-model2temporal` transformer.  `core-agent-wfdesc2model` was
looked at but does not relate to this backlog item.

Status: **0 % complete** – awaiting IR refactor kickoff.

## Notes

* Model additions should be **optional** and default-able so existing graph
  transformations (to other targets) remain unaffected.
* Keep an eye on Temporal Update & Schedule APIs—they are evolving; use
  flexible maps for options.
* Reference: `proto/api/workflows_runtime/v1/*.proto` and
  `proto/api/enums/v1/task_queue.proto` for exhaustive option fields.
