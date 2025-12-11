> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

> **Planned Change**: The current AgentRuntime architecture (agents control
> plane + agents_runtime Python execution) will be replaced by a **Backstage‑based
> Agent Controller** that aligns with the platform factory model in [471 §4](471-ameide-business-architecture.md#4-backstage-as-the-platform-factory-business-view).
>
> Key constraint from 471: "Agents are always invoked via domain/process APIs;
> they do **not** become a separate source of truth."

# 310: Agents Platform V2 (n8n-Aligned)

**Status**: Draft
**Priority**: High
**Complexity**: Large

---

## 1. Why We're Rewriting the Agents Platform

The first-generation threads experience built agents directly inside the inference service with ad-hoc configs. That approach stalled for three reasons:

1. **Configuration Sprawl** – Hardcoded Python dictionaries (`services/inference/context.py`) made it impossible for UI teams to discover metadata, let alone manage per-tenant variations.
2. **No Shared Schema** – The threads UI, inference service, and future automation planes all shaped agent payloads differently, causing drift every time we tweaked prompts or routing behaviour.
3. **Limited Extensibility** – Tools, chains, retrievers, and memories were wired as bespoke Python closures. Adding a new capability required deployments instead of configuration.

n8n solved these problems years ago in `@n8n/nodes-langchain` by treating every LangChain primitive as a **versioned, declarative node** with consistent metadata, UI generation hints, and co-located execution logic. This document captures how we transplant those foundations into Ameide’s stack (LangGraph services, Connect/Connect-Web clients, Next.js UI, PostgreSQL persistence) while preserving our governance standards.

---

## 2. n8n Foundations (Authoritative References)

| Pattern | n8n Reference | Key Takeaway |
|---------|---------------|--------------|
| Versioned node registry | `nodes/agents/Agent/Agent.node.ts` | One logical “Agent” backed by multiple compatible versions. |
| Declarative UI schema | `nodes/agents/Agent/V3/AgentV3.node.ts` | `INodeTypeDescription.properties` encode prompts, toggles, notices. |
| Reusable option bundles | `nodes/agents/Agent/agents/ToolsAgent/options.ts` & `.../V3/description.ts` | Common configuration blocks (system messages, streaming, batching). |
| Execution wrapper | `nodes/agents/Agent/agents/ToolsAgent/V3/execute.ts` | Structured input → LangChain runtime w/ streaming + tool calls. |
| Tool primitives | `nodes/tools/ToolCalculator/ToolCalculator.node.ts` | Identical metadata/execution split across tool nodes. |
| Tool executor | `nodes/ToolExecutor/ToolExecutor.node.ts` | Generic node that executes any connected tool or toolkit. |
| Chains, LLMs, memories | `nodes/chains/*`, `nodes/llms/*`, `nodes/memory/*` | Every primitive follows the same metadata + runtime recipe. |

**Conclusion**: We need a shared description format (proto) that can express *any* node: agent, chain, tool, LLM, memory, retriever. The runtime (LangGraph) pulls from that description to build graphs; the UI renders configuration from the same schema; all communication happens through generated SDKs.

---

## 3. Ameide Technology Alignment

| Concern | n8n Artifact | Ameide Translation |
|---------|--------------|--------------------|
| Node registry & versioning | `VersionedNodeType` | `agents.proto` + Postgres tables track major/minor revisions; inference loads versioned configs. |
| Declarative properties | `INodeTypeDescription.properties` | Proto `PropertySchema` -> generated TS components (Radix + Tailwind) and Pydantic wrappers for FastAPI. |
| Execution context | `toolsAgentExecute` | LangGraph builder that instantiates prompts, callbacks, tools based on proto config. |
| Tool kits / executors | `ToolExecutor.node.ts` | Shared `ToolRuntime` in inference service; Agents service exposes CRUD for tool definitions. |
| Catalog metadata | `codex` block | Proto `CatalogMetadata` used by UI navigation, search, docs linking. |
| Middleware & content blocks | `create_agent` (LangChain v1) | Publish artifacts capture middleware stacks + standard `content_blocks`; inference forwards them verbatim to clients (direct gRPC and Temporal activity paths). |
| Tests | co-located `*.test.ts` | Pytest & Playwright suites stored next to runtime modules/React features. |

> **Runtime parity vs. runtime reuse**  
> n8n’s executor is a LangChain loop implemented in TypeScript. We treat it as the behavioural spec for prompts, fallback logic, output parsing, and tool wiring, but Ameide never runs that code in production. Instead, the publish flow compiles every agent definition into a LangGraph DAG (via `langchain.agents.create_agent`), caches the artifact in the registry, and ships only the compiled graph to the Python runtime. This keeps configuration parity while letting the runtime exploit LangGraph’s deterministic graphs, checkpointing, and parallel branches.

## Progress & Gaps (2025-02)

- **✅ Control plane foundations:** Service is implemented in Go (`services/agents/cmd/agents`) with Postgres persistence and MinIO integration; node/instance CRUD, routing evaluation, and gRPC endpoints are live (`services/agents/internal/service/service.go`). Publish now invokes a compiler and artifact store prior to updating Postgres.
- **⚠️ Artifact fidelity:** Compiler now emits a `plan.v1` bundle (prompt/model/tool ids) but still lacks LangGraph DAGs and middleware execution payloads (`services/agents/internal/compiler/compiler.go`).
- **⚠️ Catalog diffusion:** `StreamCatalog` subscribers only receive node updates; `CreateAgent`/`UpdateAgent` skip `BroadcastCatalogSnapshot`, so agent-instance deltas are invisible until the next manual refresh.
- **⚠️ Snapshot streaming:** `StreamCatalog` now pushes both nodes and agents with per-tenant filtering, and create/update/publish broadcasts refreshed snapshots, but the Python runtime still polls instead of maintaining a streaming cache subscription.
- **✅ Integration job:** The service image bakes its gRPC integration suite and Tilt exposes `test:integration-agents_runtime`, aligning with the platform’s Helm-based runner.
- **⚠️ Governance events:** Required domain events (`agent.instance.published`, `agent.routing.changed`) are not emitted, leaving analytics blind to publish activity.
- **⚠️ Tool grants pending:** The schema exists (`services/agents/internal/store/postgres/postgres.go`), but the service does not yet expose grant/ revoke APIs or enforce grants at publish time.
- **⚠️ Tenant UX:** UI SDKs and guardrails are not wired to surface compilation failures or prevent publishing when artifact upload fails.

> **Next steps:** replace the JSON snapshot with a `create_agent`-backed LangGraph plan (including middleware + `content_blocks` schema), emit publish/routing events, extend streaming to cover agent-instance mutations, and integrate generated SDKs so tenants receive immediate feedback during publish.

To replicate those behaviours we explicitly capture:
- Property-level flags (`noDataExpression`, notices, callouts, display conditions) as seen in [`AgentV3`](../reference-code/n8n/packages/@n8n/nodes-langchain/nodes/agents/Agent/V3/AgentV3.node.ts#L62).
- Version wiring (`inputs`, `outputs`, `groups`, `hints`) so generated canvases behave like n8n’s node runtime.
- Version compatibility: several semantic versions can point to the same executable payload, echoing [`Agent.node.ts`](../reference-code/n8n/packages/@n8n/nodes-langchain/nodes/agents/Agent/Agent.node.ts#L17) where `1.1`–`1.9` reuse the V1 implementation.
- Tool bindings that preserve metadata emitted by [`getConnectedTools`](../reference-code/n8n/packages/@n8n/nodes-langchain/utils/helpers.ts#L201), enabling inference to rebuild toolkits and structured output helpers.

---

## 4. Target Stack Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                                UI Layer                              │
│  - Next.js app (`services/www_ameide_platform`)                      │
│  - Generated TS SDK (`packages/ameide_sdk_ts`)                         │
│  - Agent Designer (n8n-style canvas, property inspector, test pane)  │
│  - Chat interface with thread history                                │
└────────▲─────────────────────────────────────────────────────────────┘
         │ Connect-Web / gRPC-Web (HTTP/1.1 + JSON)
┌────────┴─────────────────────────────────────────────────────────────┐
│                     Agents Service (Go)                              │
│  - `agents/v1/agents_service.proto` (Control plane APIs)             │
│  - Connect protocol support (h2c for browser compatibility)          │
│  - Postgres tables: `agents`, `agent_versions`, `agent_routes`,      │
│    `tools`, `tool_versions`, `tool_access`                           │
│  - Catalog streaming via StreamCatalog                               │
└────────▲─────────────────────────────────────────────────────────────┘
         │ Native gRPC (catalog subscription)
┌────────┴─────────────────────────────────────────────────────────────┐
│                 Agents Runtime Service (Python)                      │
│  - `agents_runtime/v1/agents_runtime_service.proto`                  │
│  - LangGraph execution runtime with tool injection                   │
│  - Subscribes to catalog updates from Agents service                 │
│  - Invoke/Stream RPCs for agent execution                            │
│  - Calls downstream services for context & execution                 │
└───────┬────────────────────────────────────────────────────────┬─────┘
        │                                                        │
        │ gRPC (for threads history context)                        │ gRPC (other tools)
        │                                                        │
┌───────▼───────────┐                                    ┌───────▼────────────┐
│  Chat Service     │                                    │ Platform Services  │
│  (TypeScript)     │                                    │ - Repository       │
│  - Thread history │                                    │ - Platform (Orgs)  │
│  - Messages       │                                    │ - Workflow         │
│  - Postgres DB    │                                    │ - Inference        │
└───────────────────┘                                    └────────────────────┘
         │
         │
┌────────▼─────────────────────────────────────────────────────────────┐
│                        Downstream Integrations                       │
│  - Workflow orchestration (Temporal via workflows service)            │
│  - Evaluations (Langfuse Datasets)                                   │
│  - Governance/analytics streams                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.1 Target Application Landscape

```
                                      ┌───────────────────────┐
                                      │       Clients         │
                                      │  - Web UI (Next.js)   │
                                      │  - Internal Services  │
                                      │  - Automation Jobs    │
                                      └───────────┬───────────┘
                                                  │ HTTPS / Connect-Web
                                      ┌───────────▼───────────┐
                                      │    Envoy Gateway      │
                                      │  TLS termination      │
                                      │  GRPCRoutes per svc   │
                                      └───────────┬───────────┘
              ┌──────────────┬────────────────────┼────────────────┬─────────────┐
              │              │                    │                │             │
┌─────────────▼─────┐ ┌──────▼──────┐ ┌──────────▼────────┐ ┌─────▼─────┐ ┌────▼────┐
│  Agents (Go)      │ │ Repository  │ │  Platform         │ │  Chat     │ │Workflow │
│  Control plane    │ │ Artifacts   │ │  Orgs/Teams/Roles │ │ Threads & │ │Temporal │
│  Catalog & Routes │ │ Versions    │ │  User management  │ │ Messages  │ │Defs     │
└─────────────┬─────┘ └─────────────┘ └───────────────────┘ └─────┬─────┘ └─────────┘
              │                                                     │
              │ StreamCatalog (node/agent updates)                 │ gRPC
              │                                                     │ (history)
      ┌───────▼────────────────────────────────────────────────────▼─────────┐
      │               Agents Runtime (Python)                                │
      │  - LangGraph execution engine                                        │
      │  - Tool injection (graph, threads, platform APIs)                  │
      │  - Streaming responses via AgentsRuntimeService                      │
      │  - Metrics & tracing (Langfuse)                                      │
      └──────────────────────────────┬───────────────────────────────────────┘
                                     │
                                     │ Tool calls (gRPC)
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
            ┌───────▼────────┐ ┌─────▼──────┐ ┌──────▼────────┐
            │  Repository    │ │  Platform  │ │  Inference    │
            │  (artifacts)   │ │  (context) │ │  (LLM calls)  │
            └────────────────┘ └────────────┘ └───────────────┘
                                     │
                    ┌────────────────▼────────────────┐
                    │   Downstream Systems            │
                    │   - Temporal workflows          │
                    │   - Langfuse traces/datasets    │
                    │   - Analytics/governance events │
                    └─────────────────────────────────┘

<────── Agent Invocation Flow: UI → Envoy → Agents Runtime → Tools ──────>
<────── Catalog Management: UI → Envoy → Agents (Control Plane) ──────>
```

**Data Stores**
- **agents** (Go) → Postgres (`agents`, `agent_versions`, `routing_rules`, `agent_tools`, `tool_grants`)
- **agents_runtime** (Python) → Redis/Postgres (execution state & checkpointing), subscribes to catalog from agents
- **threads** (TypeScript) → Postgres (`threads`, `messages`, `thread_participants`) - Thread history for agent context
- **platform** (TypeScript) → Postgres (organizations, teams, roles, users)
- **graph** (TypeScript) → Postgres (artifacts, versions, metadata)
- **workflows** (TypeScript) → Temporal workflows definitions
- **workflows_runtime** (Python) → Temporal workers; reuse LangGraph executor from agents_runtime
- **inference** (Python) → Redis/Postgres (base LangGraph runtime), Langfuse

**Event Streams**
- `agent.instance.published`, `agent.routing.changed` → analytics bus
- Agents runtime & inference emit Langfuse traces + tool usage metrics
- Temporal workflows emit execution telemetry correlated with agent instance IDs (activity heartbeat metadata) – orchestration patterns detailed in [305-workflows](305-workflows.md)

**Temporal activity expectations**
- Agent bridge activities and downstream connector activities must attach business idempotency keys (`workflowsId` + sequence or domain-specific identifiers) to every external call so Temporal retries remain safe.
- Multi-step side effects expose compensating actions that workflows can execute on failure paths.
- Activity streaming follows the chunked-update guidance in [309-inference](309-inference.md) to keep workflows event histories bounded.

---

## 5. Protobuf Contracts

### 5.1 Agents Control Plane (`ameide_core_proto/agents/v1/`)

**Location**: `packages/ameide_core_proto/src/ameide_core_proto/agents/v1/agents_service.proto`

Exposes catalog management, agent instance lifecycle, and routing APIs via the **agents** Go service.

```proto
syntax = "proto3";
package ameide.agents.v1;

import "google/protobuf/struct.proto";
import "google/protobuf/timestamp.proto";

enum NodeKind {
  NODE_KIND_UNSPECIFIED = 0;
  NODE_KIND_AGENT = 1;
  NODE_KIND_TOOL = 2;
  NODE_KIND_CHAIN = 3;
  NODE_KIND_LLM = 4;
  NODE_KIND_MEMORY = 5;
  NODE_KIND_RETRIEVER = 6;
}

enum PropertyType {
  PROPERTY_TYPE_UNSPECIFIED = 0;
  PROPERTY_TYPE_STRING = 1;
  PROPERTY_TYPE_TEXT = 2;
  PROPERTY_TYPE_NUMBER = 3;
  PROPERTY_TYPE_BOOLEAN = 4;
  PROPERTY_TYPE_OPTIONS = 5;
  PROPERTY_TYPE_COLLECTION = 6;
  PROPERTY_TYPE_FIXED_COLLECTION = 7;
  PROPERTY_TYPE_JSON = 8;
  PROPERTY_TYPE_NOTICE = 9;
  PROPERTY_TYPE_CALLOUT = 10;      // mirrors n8n's `type: 'callout'`
  PROPERTY_TYPE_CREDENTIAL = 11;
  PROPERTY_TYPE_RESOURCE_LOCATOR = 12;
  PROPERTY_TYPE_CUSTOM = 99;       // allows forward-compat for n8n-specific widgets
}

message PropertyOption {
  string name = 1;
  string value = 2;
  string description = 3;
}

message StringList {
  repeated string values = 1;
}

message DisplayOptions {
  map<string, StringList> show = 1;
  map<string, StringList> hide = 2;
  map<string, StringList> equals = 3;
  string expression = 4; // optional JS-style expression evaluated client-side
}

message CredentialBinding {
  string credential_urn = 1;
  string credential_type = 2; // e.g., "openaiApi"
  repeated string scopes = 3;
}

message PropertySchema {
  string name = 1;
  string display_name = 2;
  PropertyType type = 3;
  google.protobuf.Value default_value = 4;
  repeated PropertyOption options = 5;
  repeated PropertySchema children = 6; // for collection/fixed collection
  DisplayOptions display_options = 7;
  map<string, google.protobuf.Value> type_options = 8;
  bool required = 9;
  string description = 10;
  string placeholder = 11;
  string hint_markdown = 12;
  bool no_data_expression = 13;    // align with `noDataExpression` flags in n8n nodes
  repeated string shared_bundle_ids = 14; // references reusable option bundles (e.g., common agent options)
  string custom_type = 15;         // string identifier for client-side custom widgets
  CredentialBinding credential = 16;
}

message CatalogMetadata {
  string icon = 1;
  string icon_color = 2;
  repeated string alias = 3;
  repeated string categories = 4;
  map<string, DocumentationLinks> documentation = 5;
}

message DocumentationLinks {
  repeated string urls = 1;
}

enum ConnectionKind {
  CONNECTION_KIND_UNSPECIFIED = 0;
  CONNECTION_KIND_MAIN = 1;
  CONNECTION_KIND_AI_TOOL = 2;
  CONNECTION_KIND_AI_LANGUAGE_MODEL = 3;
  CONNECTION_KIND_AI_OUTPUT_PARSER = 4;
  CONNECTION_KIND_AI_MEMORY = 5;
}

message NodeInput {
  ConnectionKind kind = 1;
  string expression = 2; // e.g., serialized JS expression from n8n's getInputs()
}

message NodeOutput {
  ConnectionKind kind = 1;
  string name = 2;
}

message NodeHint {
  string message = 1;
  string type = 2;
  string location = 3;
  string when_expression = 4; // evaluated client-side similar to n8n hint display conditions
}

message NodeVersion {
  string version = 1;                  // e.g., "3.1"
  NodeKind kind = 2;
  CatalogMetadata catalog = 3;
  repeated PropertySchema properties = 4;
  map<string, google.protobuf.Value> defaults = 5;
  map<string, google.protobuf.Value> runtime_config = 6; // e.g., streaming enabled
  google.protobuf.Timestamp created_at = 7;
  string created_by = 8;
  string change_log_markdown = 9;
  repeated NodeInput inputs = 10;
  repeated NodeOutput outputs = 11;
  repeated string groups = 12;
  repeated NodeHint hints = 13;
}

message NodeDefinition {
  string node_id = 1;                   // unique slug e.g., "agent.general"
  string display_name = 2;
  string description = 3;
  repeated NodeVersion versions = 4;
  string default_version = 5;
}

message ToolInstance {
  string tool_node_id = 1;
  string version = 2;
  map<string, google.protobuf.Value> parameters = 3;
  map<string, google.protobuf.Value> metadata = 4; // includes source_node_name/is_from_toolkit flags
}

message AgentInstance {
  string agent_instance_id = 1;
  string node_id = 2;
  string version = 3;
  map<string, google.protobuf.Value> parameters = 4;
  repeated ToolInstance tools = 5;
  map<string, google.protobuf.Value> runtime_overrides = 6;
  google.protobuf.Timestamp published_at = 7;
  string tenant_id = 8;
  string name = 9;
  string description = 10;
  repeated string allowed_roles = 11;
  repeated string ui_pages = 12;
  repeated RoutingRule routing_rules = 13;
  string compiled_artifact_uri = 14;
  string compiled_artifact_hash = 15;
  google.protobuf.Timestamp compiled_at = 16;
}

message RoutingCondition {
  string ui_page = 1;
  string user_role = 2;
  bool requires_graph = 3;
  bool requires_selection = 4;
  string element_kind = 5;
  map<string, string> custom_metadata = 6;
}

message RoutingRule {
  string rule_id = 1;
  int32 priority = 2;
  RoutingCondition condition = 3;
  string agent_instance_id = 4;
}

message ListNodesRequest {}
message ListNodesResponse { repeated NodeDefinition nodes = 1; }

message CreateAgentRequest { AgentInstance agent = 1; }
message CreateAgentResponse { AgentInstance agent = 1; }

message UpdateAgentRequest { AgentInstance agent = 1; }
message UpdateAgentResponse { AgentInstance agent = 1; }

message PublishAgentRequest { string agent_instance_id = 1; }
message PublishAgentResponse { AgentInstance agent = 1; }

message ListAgentsRequest { string tenant_id = 1; }
message ListAgentsResponse { repeated AgentInstance agents = 1; }

message StreamCatalogRequest {}
message StreamCatalogResponse {
  repeated NodeDefinition nodes = 1;
  google.protobuf.Timestamp updated_at = 2;
}

message SimulateRouteRequest {
  string tenant_id = 1;
  RoutingCondition condition = 2;
}

message SimulateRouteResponse {
  RoutingRule matched_rule = 1;
  AgentInstance agent = 2;
}

service AgentsService {
  rpc ListNodes(ListNodesRequest) returns (ListNodesResponse);
  rpc StreamCatalog(StreamCatalogRequest) returns (stream StreamCatalogResponse);
  rpc CreateAgent(CreateAgentRequest) returns (CreateAgentResponse);
  rpc UpdateAgent(UpdateAgentRequest) returns (UpdateAgentResponse);
  rpc PublishAgent(PublishAgentRequest) returns (PublishAgentResponse);
  rpc ListAgents(ListAgentsRequest) returns (ListAgentsResponse);
  rpc SimulateRoute(SimulateRouteRequest) returns (SimulateRouteResponse);
}
```

### 5.2 Agents Runtime Execution (`ameide_core_proto/agents_runtime/v1/`)

**Location**: `packages/ameide_core_proto/src/ameide_core_proto/agents_runtime/v1/agents_runtime_service.proto`

Exposes agent execution APIs via the **agents_runtime** Python service. Handles invocation, streaming, and runtime catalog synchronization.

```proto
syntax = "proto3";
package ameide_core_proto.agents_runtime.v1;

service AgentsRuntimeService {
  rpc Invoke(InvokeRequest) returns (InvokeResponse);
  rpc Stream(StreamRequest) returns (stream StreamResponse);
  rpc ListAgents(ListAgentsRequest) returns (ListAgentsResponse);
}

// Message types defined in agents_runtime_types.proto
// Include: Message, Options, Usage, StreamEvent, etc.
```

**Separation Rationale**:
- **agents** (Go): Control plane focused on catalog management, versioning, publishing, and routing
- **agents_runtime** (Python): Execution plane focused on LangGraph runtime, tool injection, and streaming execution
- Clear proto namespace separation allows independent evolution and deployment
- Browser clients connect to **agents** via Connect protocol for catalog operations
- Backend services invoke **agents_runtime** via native gRPC for agent execution

**Generated Artifacts**
- **TypeScript SDK**: `packages/ameide_sdk_ts` via `buf generate` (Connect-Web + Zod wrappers)
- **Python models**: `packages/ameide_sdk_python` for agents_runtime binding
- **Go stubs**: `packages/ameide_sdk_go` for agents service runtime and other Go services

### 5.3 Property rendering & validation contract

| Proto field | UI/runtime usage | n8n analogue |
|-------------|------------------|--------------|
| `PropertyType` / `custom_type` | Maps to React component registry: `STRING → TextArea`, `TEXT → RichText`, `OPTIONS → Select`, `CREDENTIAL → CredentialPicker`, `RESOURCE_LOCATOR → ResourceLocator`. Unknown values fall back to `JSONEditor` using `custom_type`. | `type` column in `INodeProperties` |
| `DisplayOptions` | Converted into show/hide predicates using the same evaluator that powers n8n’s [`displayOptions`](../reference-code/n8n/packages/@n8n/nodes-langchain/nodes/agents/Agent/V3/AgentV3.node.ts#L36). `expression` is executed inside a sandboxed JS runtime shared with the React form layer. | `displayOptions.show/hide` |
| `shared_bundle_ids` | Loads reusable property bundles (`common.agent.options`, `common.tool.options`) from the catalog, matching [`commonOptions`](../reference-code/n8n/packages/@n8n/nodes-langchain/nodes/agents/Agent/agents/ToolsAgent/options.ts#L4). | Spread operator composition |
| `CredentialBinding` | UI renders credential selector + scope chips; backend enforces that resolved credential URN is injected into `ToolDependencies`. | `type: 'credentials'` properties |
| `type_options` | Passed unchanged to component-specific helpers (e.g., `rows`, `maxValue`). | `typeOptions` |
| `no_data_expression` | Prevents automatic expression evaluation when false, mirroring `noDataExpression` semantics. | `noDataExpression` |

Validation pipeline:
1. TS SDK (Zod) enforces shape + required fields.
2. React form performs synchronous validation, including display-dependent required checks.
3. Agents service re-validates against generated Pydantic models before persisting.

Expression evaluation strategy:
- Use `@meide/expression-runtime`, a sandboxed JS interpreter that mirrors n8n’s expression engine for `{{ }}` templates.
- Provide a compatibility layer where legacy `inputs` expressions (e.g., from [`getInputs`](../reference-code/n8n/packages/@n8n/nodes-langchain/nodes/agents/Agent/V3/AgentV3.node.ts#L29)) are stored in `NodeInput.expression` and evaluated only in the designer runtime, never server-side.

---

## 6. Data Model & Persistence

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `agents.nodes` | Catalog of reusable node definitions (agent/tool/chain). | `node_id`, `kind`, `default_version`, `catalog_metadata`, `created_at`. |
| `agents.node_versions` | Concrete version payloads. | `node_id`, `version`, `properties_jsonb`, `defaults_jsonb`, `runtime_config_jsonb`, `change_log_md`. |
| `agents.instances` | Tenant-scoped agent instances. | `agent_instance_id`, `node_id`, `version`, `tenant_id`, `name`, `description`, `parameters_jsonb`, `runtime_overrides_jsonb`, `allowed_roles`, `compiled_artifact_uri`, `compiled_artifact_hash`, `compiled_at`. |
| `agents.routing_rules` | Declarative routing. | `rule_id`, `tenant_id`, `agent_instance_id`, `priority`, `condition_jsonb`. |
| `agents.agent_tools` | Materialized tool bindings per agent instance (parameters + metadata). | `agent_instance_id`, `tool_node_id`, `version`, `parameters_jsonb`, `metadata_jsonb`. |
| `agents.tool_grants` | Tool access control per agent instance. | `agent_instance_id`, `tool_node_id`, `granted_by`, `granted_at`. |

Migrations live in `migrations/agents/*.sql` and run via Goose/Flyway depending on service language.

### 6.1 Schema evolution & compatibility

- **Semver policy**: minor versions (`3.1`) may reuse identical execution logic (like n8n’s [`Agent.node.ts`](../reference-code/n8n/packages/@n8n/nodes-langchain/nodes/agents/Agent/Agent.node.ts#L17)); breaking property removals require major bumps.
- **Buf breaking checks**: CI enforces non-breaking proto updates; a detected break prompts regeneration of SDKs plus migration stubs.
- **Catalog migrations**: Flyway scripts populate `agents.node_versions` with new defaults and optionally backfill existing agent instances. The service rejects publishing if required parameters are missing after a schema shift.
- **Fallback rendering**: UI treats unknown `PropertyType` as `PROPERTY_TYPE_CUSTOM` and renders JSON editors, preventing hard failures during phased rollouts.

---

## 7. Agents Service Responsibilities (Go Control Plane)

The **agents** service (Go, located in `services/agents/`) serves as the control plane for agent catalog and configuration:

1. **Catalog APIs** – Serve node definitions & versions from Postgres, streaming updates to agents_runtime and UI via Connect protocol with h2c support for browser compatibility.
2. **Instance Lifecycle** – CRUD + publish for tenant-scoped agent instances with optimistic locking and n8n-compatible tool bindings (mirrors [`AgentV3` defaults/inputs](../reference-code/n8n/packages/@n8n/nodes-langchain/nodes/agents/Agent/V3/AgentV3.node.ts#L21)).
3. **Routing Management** – Evaluate declarative rules; provide `SimulateRoute` endpoint for UI sandbox ("what agent would handle this context?").
4. **Registry Sync** – Push incremental diffs (nodes, versions, tool metadata) via `StreamCatalog` to agents_runtime service; updates trigger runtime cache refresh.
5. **Audit & Telemetry** – Emit `agent.instance.published`, `agent.routing.changed`, and `agent.tool.granted` events to Platform Analytics.
6. **Connect Protocol Support** – Implements Connect protocol adapter pattern to enable browser-based clients (Connect-Web) alongside native gRPC backends.

### 7.1 Publish & Artifact Pipeline

- `PublishAgent` fetches the target `NodeDefinition`/`NodeVersion`, renders the LangGraph bundle, and produces a content-addressed artifact (`sha256`) that is uploaded to MinIO (`s3://<bucket>/<prefix>/<agent_instance_id>/<hash>.json`).
- The Agents service persists `compiled_artifact_uri`, `compiled_artifact_hash`, and `compiled_at` on the agent instance so inference can hydrate graphs without recomputing.
- Artifact storage is abstracted behind a simple S3-compatible interface to support MinIO locally and AWS S3 in production; buckets/prefixes are configured via environment variables.
- Publishing finishes only after the artifact is durably stored and the catalog broadcaster notifies subscribers; failures roll back the publish transaction.
- Domain events include artifact metadata, enabling downstream cache warmers to prefetch into Redis or other runtime stores.

---

## 8. Agents Runtime Integration (Python Execution Plane)

The **agents_runtime** service (Python, located in `services/agents_runtime/`) handles agent execution:

- **Registry Client**: Subscribe to `StreamCatalog` from agents service to receive node definitions; persist in-memory registry keyed by `node_id:version`.
- **Agent Builder**: Translate `AgentInstance` from catalog into LangGraph graphs. Prompts, streaming flags, fallback models come from proto fields.
- **Tool Loader**: Map enabled tool IDs to runtime factories using proto metadata (input schema, expected outputs). Tools injected into LangGraph executor context.
- **Routing**: Query agents service for routing evaluation; agents service returns matched agent instance based on `RoutingCondition`.
- **Invocation envelope**: Accept `InvocationContext` via `AgentsRuntimeService/Invoke` (tenant, agent id, `ui_page`, `user_role`, graph/selection flags, custom metadata).
- **Metrics**: Tag Langfuse traces and service logs with `agent_instance_id`, `node_id`, `version`.
- **Proto**: Exposes `ameide_core_proto.agents_runtime.v1.AgentsRuntimeService` for execution RPCs.

### 8.1 Service Architecture & Tool Integration

**Agents Runtime** acts as the orchestration layer, calling downstream services as tools:

1. **Chat Service** (TypeScript, `services/threads/`):
   - Provides thread history and message context for agents
   - Agents query threads history to maintain conversation context
   - Stores agent responses back to threads
   - Proto: `ameide_core_proto.threads.v1.ChatService`

2. **Repository Service** (TypeScript, `services/graph/`):
   - Artifact retrieval (ArchiMate models, BPMN diagrams, etc.)
   - Version history and metadata queries
   - Tool for agents to access architectural knowledge

3. **Platform Service** (TypeScript, `services/platform/`):
   - Organization, team, and user context
   - Role-based access control information
   - Workspace configuration for agent routing

4. **Inference Service** (Python, `services/inference/`):
   - Base LangGraph runtime (not agent-specific)
   - Model provider integrations (OpenAI, Azure OpenAI, etc.)
   - Streaming inference APIs
   - Called by agents_runtime for LLM execution

This separation ensures agents_runtime focuses on orchestration while domain services maintain their specialized data and logic.

---

## 9. UI & Designer Experience

1. **Catalog Explorer** – Uses generated TS SDK to fetch node definitions. Cards display `display_name`, `description`, and `catalog.metadata`.
2. **Agent Designer** – React/Canvas experience that mirrors n8n:
   - Palette of available nodes (agents, tools, chains).
   - Property panel driven by `PropertySchema` (rendered via generic components).
   - Test console built on top of inference streaming endpoint.
3. **Workspace Configuration** – Repository/Initiative settings consume `ListAgents` to choose defaults per UI page, toggle availability, and preview routing outcomes.
4. **Routing Sandbox** – UI enters sample context (role, ui_page, graph ID) and calls simulation API; result shows matched rule + agent preview.

All UI uses the generated TypeScript SDK (Connect-Web client) for schema accuracy.

---

## 10. Operations & Governance

- **Deployments**:
  - **agents** (Go): Deployed as `deployment/agents` in platform namespace, exposes port 8080 for Connect/gRPC
  - **agents_runtime** (Python): Deployed as `deployment/agents_runtime`, exposes port 9000 for gRPC
  - Both services managed via Helm charts in `infra/kubernetes/charts/platform/`
  - Auto-scaling based on request volume and CPU/memory metrics
- **Routing**: Envoy Gateway with separate GRPCRoutes:
  - `grpcroute-agents` matches `ameide_core_proto.agents.v1.AgentsService`
  - `grpcroute-agents_runtime` matches `ameide_core_proto.agents_runtime.v1.AgentsRuntimeService`
- **Secrets**: Tool/LLM credentials remain in Azure KeyVault (production/staging) or k8s secrets (local); agent configs reference credential IDs via proto `parameters`.
- **Auditing**: Publishing an agent writes governance records to analytics bus. Rollbacks allowed by republishing older version.
- **Testing**:
  - Unit: Schema validation, routing evaluation, LangGraph builders
  - Integration: Helm-based integration test jobs (`test-agents`, `test-agents_runtime`) via Tilt
  - E2E: Playwright flows exercise designer → publish → invoke via agents_runtime

---

## 11. Migration Plan

| Phase | Deliverable | Status | Notes |
|-------|-------------|--------|-------|
| **P0** | Proto separation: `agents.proto` + `agents_runtime.proto` | ✅ Complete | Separate namespaces for control plane and execution. |
| **P1** | Agents service (Go) with Connect protocol support | ✅ Complete | Control plane live with h2c adapter for browser clients. |
| **P2** | Agents-runtime service (Python) with LangGraph executor | ✅ Complete | Execution plane deployed, subscribes to catalog updates. |
| **P3** | GRPCRoute separation via Envoy Gateway | ✅ Complete | Distinct routes for AgentsService and AgentsRuntimeService. |
| **P4** | Service naming cleanup (remove `-service` suffixes) | ✅ Complete | All services renamed: threads, platform, workflows. |
| **P5** | UI catalog + designer MVP | 🚧 In Progress | Feature-flagged; catalog APIs functional, designer under development. |
| **P6** | Tool migration (graph queries, ArchiMate tools, etc.) | ⏳ Pending | Tools to be registered via agents service; runtime pulls from proto metadata. |
| **P7** | Legacy inference agent removal | ⏳ Pending | Remove hardcoded agent configs after migration complete. |

---

## 12. Connect Protocol Implementation

The agents service implements Connect protocol support to enable browser-based clients (Connect-Web) alongside native gRPC backends.

### Architecture

**Connect Protocol Adapter Pattern** (`services/agents/internal/service/connect_adapter.go`):
- Wraps the native gRPC service implementation
- Converts Connect requests (`*connect.Request[T]`) to gRPC messages (`*T`)
- Converts gRPC responses back to Connect responses (`*connect.Response[T]`)
- Handles streaming via `grpcStreamAdapter` that bridges Connect and gRPC stream interfaces

**HTTP/2 Cleartext (h2c) Server** (`services/agents/cmd/agents/main.go`):
- Replaces native gRPC server with HTTP server using `h2c.NewHandler`
- Enables both HTTP/1.1 (Connect-Web from browsers) and HTTP/2 (native gRPC)
- Single endpoint serves both protocol variants based on client capabilities

**Benefits**:
- Browser clients can call agents service directly via Connect-Web (HTTP/1.1 + JSON)
- Backend services continue using efficient native gRPC (HTTP/2 + protobuf)
- No separate REST API or proxy layer needed
- Same service logic handles both protocol variants

**Pattern Reusability**:
- This Connect protocol adapter pattern can be applied to any Go gRPC service that needs browser compatibility
- Provides a standard approach for enabling Connect-Web alongside native gRPC without code duplication
- Other Go services in the platform can adopt the same pattern for browser-friendly APIs

---

## 13. Open Questions

1. **Schema Evolution Tooling** – Automate diffs between successive `PropertySchema` payloads so backfills can be generated when bundles change.
2. **Credential Lifecycle** – Define how `CredentialBinding.credential_urn` rotates when secrets are rotated or revoked (shared with platform secrets team).
3. **Tool Execution Limits** – Need quotas & guardrails similar to n8n’s `maxIterations`, `returnIntermediateSteps`.

---

## 14. References

- n8n LangChain package: `reference-code/n8n/packages/@n8n/nodes-langchain/`
- `Agent.node.ts` (versioned registry)
- `Agent/V3/AgentV3.node.ts` (declarative schema)
- `Agent/agents/ToolsAgent/options.ts` + `.../V3/description.ts` (options bundles)
- `Agent/agents/ToolsAgent/V3/execute.ts` (runtime)
- `tools/ToolCalculator/ToolCalculator.node.ts` (tool metadata)
- `ToolExecutor/ToolExecutor.node.ts` (generic tool execution pipeline)

---

**Author**: Architecture Team
**Last Updated**: 2025-10-29
**Status**: Reflects current production architecture with agents/agents_runtime separation
