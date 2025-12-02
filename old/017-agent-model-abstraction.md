# 017 – Agent Model Abstraction Layer

Date: 2025-07-23

## Context

`packages/ameide_core-agent-model` currently defines an **Agent IR** (nodes, edges, state, etc.) that is geared towards the LangGraph runtime.  The IR is already consumed by the `core-agent-model2langgraph` generator and works well for threads-style graphs.

However, we purposely want the IR to be *framework-agnostic* so that:

* future generators (🦜 LangChain Flow, Airflow DAGs, Kubeflow, Temporal, bespoke engines, …) can reuse the same definitions without leaking runtime-specific details;
* the same business logic can be ported across runtimes by swapping the generator module only;
* we can serialise / persist the IR as a contract between design tools and execution engines.

At the moment some LangGraph-specific concepts have crept into the model via the `metadata` escape-hatch (e.g. `checkpointer`, `allow_cycles`) or via implicit assumptions in the transformer.

## Outcome

1. A **clean, runtime-agnostic IR** specification housed in `core-agent-model`.
2. One adapter/transformer per target runtime (`…2langgraph`, `…2temporal`, `…2custom`, etc.) that converts the IR into concrete code or configuration.
3. A migration guide + changelog for existing projects.

## Tasks

1. Identify and list all LangGraph-specific fields currently used:
   * `metadata.checkpointer`, `allow_cycles`, `condition_function`, …
   * node-level flags like `is_async`, `is_streaming` – decide if they belong in the IR or in a runtime capability layer.

2. Create **`runtime` sub-objects** on `AgentModel` for optional, engine-specific hints.
   ```py
   class RuntimeHints(BaseModel):
       langgraph: Dict[str, Any] = {}
       temporal: Dict[str, Any] = {}
       airflow: Dict[str, Any] = {}
   ```
   Keep this container 100 % optional so core validation is unaffected.

3. Move current LangGraph hints out of generic `metadata` into `runtime.langgraph`.

4. Audit `core-agent-model2langgraph` transformer:
   * Update it to read from `runtime.langgraph` first, fallback to legacy locations (deprecate with warnings).

5. Documentation
   * Update README in **core-agent-model** to state the abstraction goal and the contract.
   * Add ADR/backlog reference here.

6. (Stretch) Provide **second generator** example – `core-agent-model2airflow` that produces a dummy DAG to validate the abstraction.

### Modelling parity with LangGraph advanced features

The following sub-tasks capture *feature buckets* that exist in LangGraph today but
are **not yet expressed** in the IR.  Each bucket shall result in:

* New, **engine-agnostic dataclasses/enums** inside `core-agent-model`.
* Optional runtime hints inside `RuntimeHints.langgraph` for LangGraph-specific
  tuning.
* Updates to validation utilities.

| Bucket | Required abstractions | LangGraph mapping |
| ------ | -------------------- | ----------------- |
| **Checkpointing** | `CheckpointConfig` (strategy, frequency, store_ref, serializer, cipher) | `langgraph.checkpoint.*` saver classes |
| **Caching** | `CachePolicy` (ttl, namespace, key_fields) | `langgraph.cache.*` |
| **Store / Persistence** | `StoreConfig` (backend, connection_uri, credentials_ref) | `langgraph.store.*` + `with_config(store=…)` |
| **Channels / Message Bus** | `ChannelDef` (id, topic, aggregation_fn, retention) & `ChannelEdge` (writer ➜ channel, channel ➜ subscriber) | `langgraph.channels.*` + Pregel builder |
| **Pregel compute** | `PregelNodeConfig` (reads, writes, meta, retry, cache_policy) plus `GraphMode = {state, pregel}` flag on AgentModel | `langgraph.pregel.NodeBuilder` & `Pregel` |
| **Interrupts & Human-in-the-loop** | `InterruptConfig` (kind, timeout, resume_conditions) | `langgraph.prebuilt.interrupt.*` |
| **Retry / Error surface** | Promote `RetryPolicy`, `ErrorEdge` enhancements (exponential_backoff, jitter, classify_fn) | `langgraph.types.RetryPolicy` |
| **Remote / Distributed execution** | `RuntimeEndpoint` (url, token_ref, concurrency), `DeploymentTarget` on AgentModel | `langgraph.pregel.remote.RemoteGraph`, Swarm / Supervisor |
| **Streaming / Writers** | `StreamWriterConfig` (mode, destination) | `langgraph.types.StreamWriter`, `with_config(stream_writer=…)` |
| **Constants & Versioning** | `RuntimeConstant` registry or simple key/value bag | `langgraph.constants.*` |

Implementation order can follow row order; earlier rows unblock more common
use-cases.

For each bucket:

1. Design & add pydantic models under `ameide_core_agent_model.advanced`.
2. Extend `AgentModel` with optional field (e.g., `checkpoint: CheckpointConfig | None`).
3. Add `runtime.langgraph` adapter mapping _only_ where LangGraph needs extra
   tuning (e.g., `serializer='jsonplus'`).
4. Update transformer/generator to consume the new fields.
5. Cover with unit tests + JSON schema round-trip tests.

### Cross-cutting tasks

* **Schema export** – publish consolidated JSON-Schema so external tools (UI
  editors, workflows importers) can rely on it.
* **Back-compat hooks** – support legacy locations via deprecation warnings.
* **Documentation** – add a “Feature matrix” page mapping IR objects to
  LangGraph, Temporal, Airflow, Custom.
* **Samples** – Upgrade sample agent(s) to showcase a checkpointed, cached,
  streaming, human-interrupt capable graph.

## Acceptance Criteria

* `core-agent-model` passes tests and no longer imports or references LangGraph packages.
* All engine-specific data live under `RuntimeHints` or equivalent, not in base IR.
* `core-agent-model2langgraph` tests still pass using the new hints.
* A small PoC conversion to a different framework demonstrates the swapability.

## Notes

* Keep additive and backwards-compatible for now; we can schedule a breaking change after verifying downstream consumers.
* Inspiration: OpenAPI → code-gen for FastAPI, Express, Spring; Terraform IRs, etc.

## Progress Review (2025-07-23)

| Task | Status | Notes |
| ---- | ------ | ----- |
| 1. Identify LangGraph-specific fields | 🟡 In-progress | Some fields (e.g., `metadata.checkpointer`) remain; initial audit done ad-hoc. |
| 2. Introduce `RuntimeHints` container | ✅ Completed | `RuntimeHints` dataclass added to `core-agent-model.models`, includes `langgraph`, `temporal`, `airflow`, `custom`. |
| 3. Move hints into `runtime.langgraph` | 🟡 Partial | New code can set hints, but several generators still write to `.metadata` (`checkpointer`, `default_model`). |
| 4. Update LangGraph transformer | ✅ Completed | Transformer now reads `agent.runtime_hints.langgraph` and falls back to old metadata. |
| 5. Documentation | ⬜ Not Started | README still references old pattern. |
| 6. Secondary generator PoC | ⬜ Not Started | Awaiting separate backlog item. |

Additional observations
* `core-agent-build` still injects LangGraph-specific values into `agent.metadata` rather than `runtime_hints.langgraph`.  Needs follow-up.
* Unit tests have not yet been updated for the new field; CI still green because they don’t reference RuntimeHints.

Next Steps
1. Update **core-agent-build** to use `RuntimeHints.langgraph` (and purge direct metadata edits).
2. PR to migrate existing sample agent JSON to new field.
3. Kick-off documentation update (#TODO link).
