Note: this document is supporting background; the consolidated, normative spec is `backlog/520-primitives-stack-v2.md`. Examples use placeholder namespaces (e.g., `acme.*`)—the canonical repo proto namespace is `ameide_core_proto.*` per `backlog/509-proto-naming-conventions.md`.

Below is a concrete “best trio” design that keeps **proto = interfaces/schemas**, **generated code = stable scaffold**, and **operator = runtime config/secrets only**.

I’m grounding the LangGraph / LangChain / Buf / Kubernetes pieces in official docs:

* LangGraph Graph API basics: **state schema + reducers**, **nodes/edges**, **must compile** (compile also configures checkpointers/breakpoints) ([LangChain Docs][1])
* LangGraph persistence: **checkpointers**, **thread_id in config**, checkpoints per “super-step” enabling memory/time-travel/interrupts ([LangChain Docs][2])
* LangGraph testing patterns: compile with a **fresh checkpointer per test**, invoke with `configurable.thread_id`, test nodes via `compiled_graph.nodes[...]` and partial execution via `update_state`/interrupts ([LangChain Docs][3])
* LangChain tools: schema inference needs **type hints**, tools can accept **args_schema**, and you can **inject state** via `InjectedState` while keeping it hidden from the model’s tool schema ([LangChain Documentation][4])
* Tool calling standardization: `ChatModel.bind_tools()` accepts Pydantic classes / tools / functions; providers differ in wire format but LangChain gives a common interface, and `AIMessage.tool_calls` normalizes tool invocations ([LangChain Blog][5])
* LangChain testing: use `GenericFakeChatModel` for deterministic unit tests; use in-memory checkpointers to simulate multi-turn persistence. ([LangChain Docs][6])
* Protobuf plugin contract: a compiler plugin reads `CodeGeneratorRequest` from stdin and writes `CodeGeneratorResponse` to stdout ([Protocol Buffers][7])
* Buf descriptors/custom options: custom options are **extensions on options messages** (FileOptions/MessageOptions/FieldOptions/MethodOptions) ([Buf][8])
* Buf remote/custom plugins: remote plugins MUST be deterministic—**no filesystem/network dependence** beyond the request ([Buf][9])
* Kubernetes operator pattern & CRDs: operators reconcile desired state using custom resources/controllers ([Kubernetes][10])
* Secrets/config injection patterns: inject via env / envFrom or mount via projected volumes ([Kubernetes][11])
* Autoscaling via HPA and HTTP exposure via Gateway API / HTTPRoute ([Kubernetes][12])

---

## 0) LangGraph v1.0 GA alignment (memory + durability for DAGs)

This research note assumes the LangGraph “state is memory” model; for Ameide we make the following points explicit so baseline agents and scaffolding remain aligned:

- **State is the memory:** nodes return state updates; avoid in-place mutation.
- **Parallelism requires reducers:** any state field written by multiple nodes in a superstep must define a reducer (append/merge/replace).
- **Superstep semantics:** parallel node updates apply together at the end of a superstep, so deterministic merges matter.
- **Durability boundary:** if persistence is enabled, `thread_id` is required; replay/continue may use `checkpoint_id`.
- **Interrupt safety:** on resume, nodes can be re-run from the beginning; side-effects must be idempotent or pushed behind stable boundaries.
- **Short-term vs long-term memory:** per-thread checkpoints are short-term; any shared store is long-term and must remain non-authoritative for business truth.

In Ameide terms:

- Graph state/checkpoints are workflow memory only.
- Domain/Process remain canonical truth (facts/intents/process facts) and the only writers of durable truth.
- Blob artifacts are out-of-band; state stores only references.

## 1) Proto shape

### Goals

1. **Proto defines:**

   * Agent **public API shapes** (request/response envelopes).
   * Tool **I/O schemas** (input/output messages).
   * Optional **event/fact** message schemas (typed emissions).
   * Agent **state schema** (what can be checkpointed / validated).

2. **Proto does not define:**

   * Prompts, routing logic, heuristics, tool selection policy, hidden reasoning.

3. **Proto includes only minimal “graph hints”:**

   * `entry_node`
   * `required_tool_ids`
   * `state_type` is not supported in v2.
   * State reducers are required per field (reducers are part of state semantics in LangGraph) ([LangChain Docs][1])

### State discipline (LangGraph persistence reality)

LangGraph persistence keys checkpoint history by `thread_id` (passed via `config={"configurable":{"thread_id":"..."}}`). If persistence is enabled, treat `thread_id` as required input and treat state size as an operational constraint:

* **`thread_id` is required when persistence is enabled.** Generated scaffolding MUST enforce this at the API boundary (reject requests without it) and pass it into LangGraph config consistently.
* **Persisted state stays small.** Persist IDs, cursors, keys, and compact summaries; do not store large blobs (documents, images, long transcripts) directly in state.
* **Large artifacts are out-of-band.** Generated code MUST include blob-store hooks (interface + stub implementation) and state MUST store only references (URI/object key + checksum + size + content-type).
* **Checkpoint scope is explicit.** If you generate field-level “checkpoint” markers (e.g., `(state_field).checkpoint`), default to checkpointing only the minimal subset required for replay/resume.

Additions that keep the system safe under durable execution:

* **Replay is first-class (optional):** if supported, accept an optional `checkpoint_id` in the API envelope/config so callers can replay/continue at a known checkpoint.
* **Interrupts re-run nodes:** treat any node that can be replayed as (a) pure or (b) idempotent, or (c) emitting a stable intent to a domain/process boundary rather than performing the side-effect in-process.

### Fan-out/fan-in discipline (Send + reducers)

If you want real DAG parallelism:

* Use LangGraph `Send` for map-reduce style fan-out, not “LLM decides to parallelize”.
* Write fan-out results into reducer-backed state fields (append/merge), then join with a deterministic gate node.

### Durable side effects (Ameide boundaries)

Treat these as **side-effects** (must be idempotent under replay/interrupt):

* A2A “create task” / “send message”
* emitting a Domain intent (command over the bus)
* calling an Integration seam

Use stable dedupe keys such as:

* A2A: `{thread_id, node_id, logical_task_key}`
* Domain intent: `{aggregate_id, intent_type, causation_id}` (plus tenant axis)

### Long-term memory (LangGraph store) usage in Ameide

If you introduce LangGraph “store” (cross-thread memory), constrain it to:

* agent-private preferences/cached summaries
* non-authoritative hints and retrieval aids

Do not store canonical business truth there; persist truth in Domain/Process and reference it by stable IDs.

### Streaming standardization

For operator-facing agents, standardize on:

* machine-readable progress: `stream_mode="updates"`
* token/message streaming: opt-in only when needed
* a single custom channel for Ameide-native status fields (`phase`, `reason`, `artifact_ref`, `correlation_id`)

### Recommended proto organization

Use 3 packages (separate files/modules to avoid mixing concerns):

* `acme.agent.v1`
  Agent service, state messages, agent API envelopes.
* `acme.tools.v1`
  Tool RPC contracts (request/response per tool).
* `acme.domain.facts.v1` / `acme.process.facts.v1` / `acme.domain.intents.v1`
  Async contracts your runtime can emit/consume. Avoid `acme.events.v1` synonyms; publish facts/intents on stable topic families instead (for shared message metadata, reuse `ameide_core_proto.common.v1.event_fact` from `packages/ameide_core_proto/src/ameide_core_proto/common/v1/eventing.proto`).

And one shared annotations file:

* `acme.annotations.v1`
  Custom options for labeling message roles and providing minimal hints.

### How to mark “state” vs “tool I/O” vs “events/facts”

Use **custom options** on `google.protobuf.MessageOptions` and `FieldOptions` and `ServiceOptions`. Custom options are a standard Protobuf mechanism implemented as extensions on the options messages ([Buf][8]).

**Hard rule:** mark Ameide-specific options as **SOURCE-retained** by default so they are available to codegen but do not leak into runtime descriptor pools; generators must read such options from `CodeGeneratorRequest.source_file_descriptors`.

Concretely:

* `MessageOptions` → classify a message as:

  * `STATE`
  * `TOOL_INPUT`
  * `TOOL_OUTPUT`
  * `EVENT`
  * `FACT`

* `FieldOptions` → state reducer semantics (LangGraph reducers control how updates are applied) ([LangChain Docs][1]):

  * `REPLACE` (default)
  * `APPEND_LIST`
  * `MERGE_MAP`
  * `ADD_MESSAGES` (only if you deliberately model messages similarly to LangGraph message reducers; otherwise keep your own list append)

* `ServiceOptions` (on the Agent service) → minimal graph hints:

  * `agent_id`
  * `entry_node`
  * `required_tool_ids`

* `MethodOptions` (on Tool service RPCs) → tool metadata for adapters:

  * stable `tool_id` (used by operator/runtime config mapping)
  * `display_name`, `description`
  * `idempotent` (helps caching/retries)
  * `default_timeout_ms` (hint only; runtime can override)

This is still “interface + operational metadata”, not “behavior DSL”.

---

## 2) Buf plugin outputs (Python + LangGraph scaffold)

### Why a Buf plugin here

* Protobuf plugins are the standard way to generate code: plugin reads `CodeGeneratorRequest` and writes `CodeGeneratorResponse` ([Protocol Buffers][7]).
* If you want this to work as a Buf remote plugin, keep generation deterministic (no filesystem/network dependencies) ([Buf][9]).
* **Hard rule:** implement the generator on a standard plugin harness (e.g., Buf’s `bufbuild/protoplugin`) and add golden tests that assert byte-for-byte deterministic output from fixed descriptor inputs.

### Generated outputs you asked for

You wanted:

* `state.generated.py` (Pydantic models)
* `graph.generated.py` (LangGraph skeleton)
* `tools/generated/*.py` (typed adapters)
* `nodes_impl.py` / `policy_impl.py` boundary
* RED tests scaffolding + harness
* Regeneration/ownership rules

#### Proposed file layout

This is compatible with LangGraph’s “agent package + utils/state/nodes/tools” style ([LangChain Docs][13]), while keeping generated code clearly separated:

```
repo/
  packages/
    ameide_core_proto/
      src/ameide_core_proto/…       # proto sources

    ameide_sdk_python/
      gen/python/…                  # protoc Python + gRPC outputs (SDK stubs)

  build/
    generated/
      agent/
        research-agent/             # GENERATED-ONLY (safe to delete)
          state.generated.py
          graph.generated.py
          runtime_contract.generated.py
          tools/generated/*.py
          tests/generated/*.py
          harness/generated/*.py

  primitives/
    agent/
      research-agent/               # Implementation-owned runtime package (never cleaned)
        __init__.py
        app.py                      # tiny entrypoint; imports SDK stubs + generated wiring
        nodes_impl.py               # node implementations (implementation-owned)
        policy_impl.py              # routing/tool policy (implementation-owned)
        tools_impl/                 # only if you sometimes implement tools in-process
          __init__.py
          web_search.py
```

### What each generated file does

#### `state.generated.py` (Pydantic models + reducer helpers)

* Generates Pydantic `BaseModel` equivalents for:

  * state messages marked `(msg_role)=STATE`
  * tool input/output messages
  * event/fact messages
* Exposes:

  * `StateModel` class (Pydantic) — LangGraph supports Pydantic as state schema ([LangChain Docs][1])
  * reducer functions (pure Python) that match LangGraph reducer signatures (left, right) → merged value ([LangChain Docs][1])
  * JSON-schema export utilities for tool binding

> Note: LangGraph docs mention TypedDict/dataclass/Pydantic tradeoffs (Pydantic validation vs performance) ([LangChain Docs][1]). You can still generate Pydantic but keep hot-path state small.

**Structured output and validation (required):**

Since LangChain/LangGraph structured output is typically enforced via JSON/Pydantic validation, generating Pydantic models from proto is a practical contract:

* Validate tool inputs/outputs and any “final response” shape at runtime against generated Pydantic models.
* Treat validation failures as contract violations (surface as structured errors and metrics), not as “best-effort” parsing.

#### `tools/generated/*.py` (tool adapters)

Each tool adapter is a thin wrapper that:

* Validates inputs using the generated Pydantic input model.
* Transports the call to the configured endpoint (HTTP/gRPC/etc; chosen at runtime).
* Validates output using the generated Pydantic output model.
* Exposes a LangChain tool object:

  * Implements `BaseTool` or uses the `tool()` wrapper with `args_schema=...` (LangChain tools support `args_schema` and require type hints for inference if you rely on inference) ([LangChain Documentation][4])
* Supports `InjectedState` for “hidden” runtime-provided params if you want (e.g., tenant id, auth context) while keeping those args out of the model-visible schema ([LangChain Documentation][4])

Critically: **proto defines tool I/O**, operator/runtime provides **endpoints and credentials**.

#### `graph.generated.py` (LangGraph wiring skeleton)

Generated graph builder includes:

* `StateGraph(StateModel)` initialization and `.compile(...)` call ([LangChain Docs][1])
* A standard place to inject:

  * `checkpointer` (required) ([LangChain Docs][2])
  * runtime config (model + tools)
* Adds nodes by importing **implementation-owned** implementations.

LangGraph requires compilation before use, and compilation is also where checkpointers/breakpoints are configured ([LangChain Docs][1]).

#### `nodes_impl.py` and `policy_impl.py` boundary

* Generated code imports *only* these as extension points.
* Humans implement:

  * Node logic in `nodes_impl.py`
  * Edge/routing logic in `policy_impl.py`
* Proto only provides **entry node name** and **required tools**, not edge logic.

#### RED tests scaffolding

Use the official testing patterns:

* Build graph per test and compile with a fresh in-memory checkpointer ([LangChain Docs][3])
* Invoke with `config={"configurable":{"thread_id":"..."}}` when persistence is enabled ([LangChain Docs][2])
* Unit test nodes through `compiled_graph.nodes["node"].invoke(...)` ([LangChain Docs][3])
* For tool-calling behaviors, use `GenericFakeChatModel` to deterministically emit tool_calls ([LangChain Docs][6])

The generator can produce tests that initially fail with clear messages like:

* “Node `plan` is still NotImplemented”
* “Required tool `web_search` not registered”
* “State reducer mismatch for field X”

### Regeneration + ownership rules

**Hard rule:** anything under `build/generated/agent/**` is overwritten on every `buf generate`.

* `*.generated.py` files: always overwritten, contain header:

  * “DO NOT EDIT; generated by protoc-gen-acme-agent”
* Implementation-owned files live outside generator outputs (e.g., `primitives/agent/<agent-id>/{nodes_impl.py,policy_impl.py,...}`) and are never touched by generators.
* If you want “starter” implementations, generate **templates** under generated roots (e.g., `build/generated/agent/templates/nodes_impl.py.tmpl`) and copy them into `primitives/agent/<agent-id>/` once.
* If you want stronger enforcement, CI can fail if:

  * a generated file differs from `buf generate` output
  * a generated file was edited without changing its “generated hash” footer (cheap tamper check)

---

## 3) Operator internals

### Operator responsibilities (and only these)

The operator is a Kubernetes controller implementing the operator pattern (reconcile loop managing CRDs) ([Kubernetes][10]).

It MUST manage:

* Deployments/Services
* HTTPRoute attachment to Gateway (ingress)
* Secrets and ConfigMaps injection
* Optional autoscaling (HPA)

It must **not** embed prompts, graph logic, tool policies.

### CRD sketch: `Agent` (namespaced)

**Spec fields (minimal but complete):**

* `image`: container image for runtime
* `replicas` OR `autoscaling` config
* `http` exposure:

  * `gatewayRef` (name/namespace)
  * `hostnames[]`, `pathPrefix`, `port`
* `model` config:

  * `provider` (openai/anthropic/…)
  * `model` (string)
  * `params` (temperature, max_tokens, etc.)
  * `secretRefs` for provider keys
* `tools[]`:

  * `toolId`
  * `endpointRef` (Service/URL)
  * `authSecretRef` (required)
  * `timeouts/retries` (runtime hints)
* `persistence`:

  * `enabled`
  * `backend` (memory/sqlite/postgres/…)
  * `dsnSecretRef` (if needed)
* `resources`, `serviceAccountName`, `nodeSelector`, etc.

**Status fields:**

* `conditions[]`
* `observedGeneration`
* `url` (resolved external URL)
* `resolvedTools[]` (toolId → endpoint)
* `effectiveModel` (provider/model actually injected)

### Secrets/config injection patterns

Use standard K8s mechanisms:

* Env vars via `env`/`envFrom` ([Kubernetes][14])
* Secrets mounted as files (required for keys) using projected volumes that combine Secret + ConfigMap + DownwardAPI ([Kubernetes][15])

### Autoscaling

If `spec.autoscaling` is set, operator creates an HPA targeting the Deployment. HPA is the standard resource that updates replicas to match demand based on CPU/memory/custom metrics ([Kubernetes][12]).

### HTTP exposure

Operator creates:

* Service `agent-runtime` (ClusterIP)
* HTTPRoute attaching to the chosen Gateway via `parentRefs` and routing to Service via `backendRefs` ([Kubernetes Gateway API][16])

---

## Minimal operator ↔ runtime contract

This is the “narrow waist” between operator and generated runtime.

| Contract item                   | Delivered by operator           | Consumed by runtime                   | Notes                                                 |
| ------------------------------- | ------------------------------- | ------------------------------------- | ----------------------------------------------------- |
| `AGENT_ID`                      | env var                         | selects which generated graph to load | stable identifier                                     |
| `PORT_HTTP`                     | env var                         | server bind                           | default 8080                                          |
| `MODEL_PROVIDER` / `MODEL_NAME` | env var                         | choose ChatModel impl                 | operator-controlled config                            |
| `MODEL_PARAMS_JSON`             | ConfigMap-mounted file or env   | model init                            | keep it JSON to stay language-agnostic                |
| Provider API key                | Secret mounted file (required) | model init reads file path            | MUST NOT use plaintext env vars for secrets ([Kubernetes][11]) |
| `TOOLS_CONFIG_JSON`             | ConfigMap-mounted file          | tool adapters resolve endpoints/auth  | maps toolId → url + authRef                           |
| Tool auth tokens                | Secret mounted files            | tool adapters attach auth             | per-tool                                              |
| `PERSISTENCE_ENABLED`           | env var                         | enable checkpointer                   |                                                       |
| `PERSISTENCE_DSN`               | Secret file path                | checkpointer init                     | only if non-memory                                    |
| `LOG_LEVEL`                     | env var                         | logging                               | required (default `info`)                             |

This keeps operator purely operational and runtime purely behavioral.

**Request identity contract (when persistence enabled):**

* Clients MUST provide `thread_id` (e.g., as a request field or HTTP header mapped into runtime config).
* Runtime MUST map that value to `configurable.thread_id` when invoking the compiled graph.

### Conditions (suggested)

* `SecretsReady` — all referenced secrets exist/mounted
* `DependenciesReady` — model config + provider key present AND tool endpoints resolved + config rendered
* `WorkloadReady` — Deployment available
* `RouteReady` — HTTPRoute accepted/attached
* `Ready` — umbrella condition: all required are true

(Use `metav1.Condition` everywhere and set `observedGeneration` to the Agent CR’s `.metadata.generation` when updating conditions; keep condition types positive-polarity and use `reason`/`message` for details.)

(These are status conditions only; they don’t encode logic.)

---

## Deliverable: Example proto snippets (annotations + agent + tool + state)

Important: **custom options are extensions**, and extension declarations MUST live in a dedicated `syntax = "proto2";` file. Use normal `proto3` files for your agent/tool APIs and *import* the annotations.

### `acme/annotations/v1/agent_annotations.proto` (options/extensions)

```proto
syntax = "proto2";

package acme.annotations.v1;

import "google/protobuf/descriptor.proto";

enum MessageRole {
  MESSAGE_ROLE_UNSPECIFIED = 0;
  STATE = 1;
  TOOL_INPUT = 2;
  TOOL_OUTPUT = 3;
  EVENT = 4;
  FACT = 5;
}

message MessageRoleOption {
  optional MessageRole role = 1;
}

extend google.protobuf.MessageOptions {
  optional MessageRoleOption msg_role = 51001;
}

enum ReducerKind {
  REDUCER_KIND_UNSPECIFIED = 0; // defaults to REPLACE in codegen
  REPLACE = 1;
  APPEND_LIST = 2;
  MERGE_MAP = 3;
}

message StateFieldOption {
  optional ReducerKind reducer = 1;
  optional bool checkpoint = 2; // hint: persisted/checkpointed
  optional bool public_input = 3;  // hint: allowed in graph input schema
  optional bool public_output = 4; // hint: exposed in graph output schema
}

extend google.protobuf.FieldOptions {
  optional StateFieldOption state_field = 51002;
}

message AgentServiceOption {
  optional string agent_id = 1;
  optional string entry_node = 2;
  repeated string required_tool_ids = 3;
}

extend google.protobuf.ServiceOptions {
  optional AgentServiceOption agent = 51003;
}

message ToolMethodOption {
  optional string tool_id = 1;
  optional string display_name = 2;
  optional string description = 3;
  optional bool idempotent = 4;
  optional uint32 default_timeout_ms = 5;
}

extend google.protobuf.MethodOptions {
  optional ToolMethodOption tool = 51004;
}
```

### `acme/agent/v1/research_agent.proto` (agent API + state)

```proto
syntax = "proto3";

package acme.agent.v1;

import "acme/annotations/v1/agent_annotations.proto";
import "google/protobuf/timestamp.proto";

message ResearchState {
  option (acme.annotations.v1.msg_role) = { role: STATE };

  repeated string notes = 1 [
    (acme.annotations.v1.state_field) = {
      reducer: APPEND_LIST
      checkpoint: true
      public_input: false
      public_output: true
    }
  ];

  map<string, string> facts = 2 [
    (acme.annotations.v1.state_field) = {
      reducer: MERGE_MAP
      checkpoint: true
      public_input: false
      public_output: true
    }
  ];

  string user_query = 3 [
    (acme.annotations.v1.state_field) = {
      reducer: REPLACE
      checkpoint: true
      public_input: true
      public_output: false
    }
  ];

  google.protobuf.Timestamp started_at = 4 [
    (acme.annotations.v1.state_field) = {
      reducer: REPLACE
      checkpoint: true
      public_input: false
      public_output: true
    }
  ];
}

message AgentInvokeRequest {
  string request_id = 1;
  ResearchState input = 2;
}

message AgentInvokeResponse {
  string request_id = 1;
  ResearchState output = 2;
}

service ResearchAgent {
  option (acme.annotations.v1.agent) = {
    agent_id: "research-agent"
    entry_node: "entry"
    required_tool_ids: "web_search"
  };

  rpc Invoke(AgentInvokeRequest) returns (AgentInvokeResponse);
}
```

### `acme/tools/v1/tools.proto` (tool contracts)

```proto
syntax = "proto3";

package acme.tools.v1;

import "acme/annotations/v1/agent_annotations.proto";

message WebSearchRequest {
  option (acme.annotations.v1.msg_role) = { role: TOOL_INPUT };
  string query = 1;
  uint32 max_results = 2;
}

message WebSearchResponse {
  option (acme.annotations.v1.msg_role) = { role: TOOL_OUTPUT };
  repeated string urls = 1;
  repeated string snippets = 2;
}

service Tools {
  rpc WebSearch(WebSearchRequest) returns (WebSearchResponse) {
    option (acme.annotations.v1.tool) = {
      tool_id: "web_search"
      display_name: "Web Search"
      description: "Search the web for relevant documents."
      idempotent: true
      default_timeout_ms: 8000
    };
  }
}
```

Why this meets your success criteria:

* Proto defines **interfaces and schemas** (what goes in/out), plus very small **structural hints** (entry node + required tools).
* Reducer hints are “data semantics” for state update behavior, aligned with LangGraph’s state/reducer model ([LangChain Docs][1]).
* No prompts, no policies, no routing.

---

## Deliverable: Sample generated `graph.generated.py` skeleton

This is what your plugin MUST generate (simplified). It uses LangGraph’s “define → add nodes/edges → compile” lifecycle ([LangChain Docs][1]) and leaves behavior to humans.

```python
# build/generated/agent/acme_agent/graph.generated.py
# DO NOT EDIT. Generated by protoc-gen-acme-agent.

from __future__ import annotations

from langgraph.graph import StateGraph, START, END

from acme_agent.generated.state.generated import ResearchStateModel
from acme_agent.generated.tools.generated.registry import build_tool_registry
from acme_agent.nodes_impl import entry  # implementation-owned
from acme_agent.policy_impl import register_edges  # implementation-owned


AGENT_ID = "research-agent"
ENTRY_NODE = "entry"
REQUIRED_TOOL_IDS = ("web_search",)


def build_graph(*, runtime, checkpointer=None):
    """
    runtime: runtime handle providing model + tool endpoint config.
    checkpointer: required LangGraph checkpointer (operator config).
    """
    tools = build_tool_registry(runtime=runtime, required_tool_ids=REQUIRED_TOOL_IDS)

    g = StateGraph(ResearchStateModel)

    # Required entry node exists as a function in nodes_impl.py
    g.add_node("entry", entry(runtime=runtime, tools=tools))

    # Humans own edges/policy to avoid 'proto as behavior DSL'
    register_edges(g)

    # Minimal safety: ensure START is wired to entry unless overridden
    # (policy_impl overrides this by adding its own START edge).
    g.add_edge(START, ENTRY_NODE)

    # compile is required, and is where checkpointers/breakpoints are set
    compiled = g.compile(checkpointer=checkpointer)
    return compiled
```

The generator MUST NOT wire `ENTRY_NODE -> END` by default; missing edge logic MUST be caught early by tests.

---

## Deliverable: What *not* to encode in proto

To avoid “proto as prompt/behavior DSL” pitfalls:

1. **No system prompts, rubrics, exemplars, or chain-of-thought scaffolds.**
   These change frequently and are not interface contracts.

2. **No semantic routing rules** like “if query contains X call tool Y”.
   That’s policy logic; keep it in `policy_impl.py` + tests.

3. **No model/provider secrets or endpoints**.
   Operator injects secrets/config at runtime via standard K8s primitives ([Kubernetes][11]).

4. **No per-environment config** (temperature, max_tokens, retries).
   Those belong in CRD spec or runtime config, not proto.

5. **No graph topology beyond minimal hints** (entry node + required tools).
   If you encode full edges/conditions in proto, it will inevitably become an implicit behavior DSL.

---

## CI guardrails (practical and enforceable)

### Proto / generation

* `buf lint` + `buf breaking` (API hygiene + compatibility)
* `buf generate` followed by a **clean git diff check**:

  * fail CI if generated files changed but weren’t committed
* `buf.gen.yaml` pins remote plugin versions (and optionally revisions) and uses `clean: true` for generated-only directories. (No bespoke CLI; `buf generate` is canonical.) ([Buf][9], [Buf][17])

### Python quality gates

* Formatting/lint: `ruff` + `black` (or ruff-format)
* Type checking: `mypy`
* Unit tests: `pytest`

### Import policy guardrail

Enforce that generated code only imports:

* stdlib
* LangGraph/LangChain runtime libs
* `nodes_impl.py` and `policy_impl.py` (the only implementation-owned imports)

…and that human code never edits `build/generated/agent/**`.

Also enforce that runtimes import **generated SDK outputs** (e.g., `packages/ameide_sdk_python/gen/python/**`) rather than importing proto source trees directly.

### Test scaffolding rules (RED tests)

Generate tests that:

1. Build graph per test and compile with a fresh checkpointer ([LangChain Docs][3])
2. Fail fast if any required node returns NotImplemented
3. Validate required tools are registered
4. Optionally test tool-call flows using `GenericFakeChatModel` emitting tool_calls ([LangChain Docs][6])

---

## Why this trio works

* **Proto is the contract**: stable tool I/O, state/event shapes, minimal wiring hints.
* **Generated code is a scaffold**: it compiles the LangGraph correctly, provides typed models and adapters, and forces implementation via RED tests.
* **Operator is operational only**: reconciles Deployments/Routes/Secrets/HPA using standard Kubernetes patterns ([Kubernetes][10]), injects config/secrets via env/mounts ([Kubernetes][14]), and never touches behavior.

If you want, I can also provide:

* a concrete `buf.gen.yaml` that runs **standard python protos + your custom plugin** (remote or local),
* a sample `Agent` CRD YAML manifest + reconciled Deployment/HTTPRoute YAML templates,
* and a minimal runtime “server” contract (HTTP endpoints `/invoke` + `/stream`) that stays language-agnostic while still using the generated graph.

[1]: https://docs.langchain.com/oss/python/langgraph/graph-api "Graph API overview - Docs by LangChain"
[2]: https://docs.langchain.com/oss/python/langgraph/persistence "Persistence - Docs by LangChain"
[3]: https://docs.langchain.com/oss/python/langgraph/test "Test - Docs by LangChain"
[4]: https://reference.langchain.com/python/langchain/tools/ "Tools | LangChain Reference"
[5]: https://blog.langchain.com/tool-calling-with-langchain/ "Tool Calling with LangChain"
[6]: https://docs.langchain.com/oss/python/langchain/test "Test - Docs by LangChain"
[7]: https://protobuf.dev/reference/cpp/api-docs/google.protobuf.compiler.plugin.pb/ "google.protobuf.compiler.plugin.pb.h | Protocol Buffers"
[8]: https://buf.build/docs/reference/descriptors/ "Descriptors - Buf Docs"
[9]: https://buf.build/docs/bsr/remote-plugins/custom-plugins/ "Custom plugins - Buf Docs"
[10]: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/ "Operator pattern | Kubernetes"
[11]: https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/ "Distribute Credentials Securely Using Secrets | Kubernetes"
[12]: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/ "Horizontal Pod Autoscaling | Kubernetes"
[13]: https://docs.langchain.com/oss/python/langgraph/application-structure "Application structure - Docs by LangChain"
[14]: https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/ "Define Environment Variables for a Container | Kubernetes"
[15]: https://kubernetes.io/docs/concepts/storage/projected-volumes/ "Projected Volumes | Kubernetes"
[16]: https://gateway-api.sigs.k8s.io/api-types/httproute/ "HTTPRoute - Kubernetes Gateway API"
[17]: https://buf.build/docs/configuration/v2/buf-gen-yaml/ "buf.gen.yaml - Buf Docs"
