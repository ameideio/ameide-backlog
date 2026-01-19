# 534 — MCP Protocol Adapter (Model Context Protocol for Agentic Access)

**Status:** Draft (scaffold + shared runtime implemented)
**Audience:** Architecture, platform engineering, AI/agent teams
**Scope:** Define **MCP** as a protocol adapter pattern for **agentic access** — enabling AI agents (platform agents, browser chat, Claude Code, etc.) to interact with capabilities via a standard tool interface, following CQRS principles.

**Use with:**
- Capability definition template: `backlog/528-capability-definition.md`
- Transformation capability: `backlog/527-transformation-capability.md`
- Projection specification (v6): `backlog/527-transformation-projection-v6.md`
- Agent primitive specification: `backlog/527-transformation-agent.md`
- Primitives stack: `backlog/520-primitives-stack-v6.md`

---

## Implementation progress (current)

Implemented in repo:
- [x] Shared MCP runtime module (`packages/ameide_mcp`) for Streamable HTTP muxing, PRM discovery, origin policy, and token verification hooks (capability-agnostic, leaf dependency).
- [x] MCP adapter scaffold command: `ameide scaffold integration mcp-adapter --capability <cap>` (templates under `packages/ameide_core_cli/internal/commands/templates/integration/mcp_adapter/`).
- [x] MCP adapter shape verification: `ameide primitive verify --check mcp` enforces required files/conventions for `*-mcp-adapter` Integration primitives (tests still run via `ameide test` per 430v2).
- [x] Reference adapter (shape + conformance): `primitives/integration/sre-mcp-adapter` implements stdio + Streamable HTTP, PRM endpoint, origin allowlist, and unit tests; `ameide primitive verify --kind integration --name sre-mcp-adapter --mode repo` passes.
- [!] Reference adapter (HTTP mux spine): `primitives/integration/transformation-mcp-adapter` uses `packages/ameide_mcp.NewHTTPMux(...)` and includes a smoke test; `ameide primitive verify --kind integration --name transformation-mcp-adapter --mode repo` passes but warns that transport/security tests are missing.

Not yet implemented (still design-level in this backlog):
- [ ] Proto-annotated tool/resource exposure (no hand-authored parallel JSON schema; generate from `(ameide.mcp.expose)` or equivalent).
- [ ] Real capability routing (tools/resources that call Projection/Domain gRPC via SDKs; current adapters are scaffolds/placeholders).
- [ ] One “conformance suite” that can be run against any adapter (beyond unit tests) to validate auth/origin/PRM + tool/resource behavior.

## Clarification requests (next steps)

Decide/confirm:
- [ ] Canonical env-var/config contract for adapters (currently mixed prefixes: `AMEIDE_*` vs `MCP_*`) including auth mode, issuer/JWKS, required scopes, and allowed origins.
- [ ] Whether stdio transport is required beyond local/dev (vs HTTP-only in-cluster) and how it is gated.
- [ ] Canonical mapping from MCP identity → Ameide tenant/user (claims, tenant selection, and required scopes per capability).
- [ ] Trace/correlation propagation rule (what goes in HTTP headers vs tool inputs; how it maps to Ameide RequestContext).
- [ ] Error mapping contract (gRPC status/details → MCP JSON-RPC error shape) and what is safe to expose to external clients.
- [ ] Whether the platform standardizes on `mcp:tools`/`mcp:resources` only, or introduces capability-scoped write scopes (e.g. `sales:write`, `transformation:write`) for coarse authz filtering before tool-level enforcement.

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Interface (MCP adapter), Application Service (protocol translation).
- **Primitive classification:** **Integration primitive** — MCP adapter is an Integration primitive per `backlog/520-primitives-stack-v6.md`. It adapts external protocol (MCP/JSON-RPC) to internal protocols (gRPC).
- **Out-of-scope layers:** Strategy/Business (capability design), Technology (deployment specifics).
- **Allowed nouns:** protocol adapter, tool, resource, command, query, agentic access, JSON-RPC.
- **Prohibited unless qualified:** service, endpoint, REST (qualify with protocol context).

---

## Architecture Decisions (ADRs)

| ADR | Decision | Rationale |
|-----|----------|-----------|
| **ADR-01** | **MCP adapter is Integration primitive, not gateway** | Per 520, adapters translate protocols but don't own semantics. One adapter per capability, not a platform-wide MCP gateway. |
| **ADR-02** | **Proto-first tool schemas** | MCP tool schemas MUST be generated from proto. Never hand-author parallel JSON schemas. `(ameide.mcp.expose)` annotation controls exposure. |
| **ADR-03** | **OAuth 2.1 + Protected Resource Metadata** | MCP adapter implements OAuth 2.1 per MCP Authorization spec. Each client authenticates directly; no token brokering. |
| **ADR-04** | **CQRS routing: commands→Domain, queries→Projection** | Reads (`listElements`, `semanticSearch`) route to Projection. Writes (`submitIntent`) route to Domain. MCP adapter doesn't blur this boundary. |
| **ADR-05** | **Projection owns semantic search** | Vector index and contextual chunks belong to Projection (see 535). MCP adapter calls Projection, never builds embeddings. |
| **ADR-06** | **Writes default to proposals** | MCP write tools should default to creating proposals (governed by Process), not direct mutations, unless scope+risk allow it. |
| **ADR-07** | **Internal agents use SDK/gRPC; MCP is external compatibility binding** | Platform agents (AmeidePO, AmeideSA, AmeideCoder) use typed SDK clients for performance and type safety. MCP adapters exist for external AI tools (Claude Code, Cursor, Copilot). This preserves typed invariants and avoids JSON-RPC overhead inside the cluster. |
| **ADR-08** | **Approval governance owned by Domain/Process, not adapter** | MCP command tools create intents/draft revisions. Approval gating is enforced by Domain policies + Process workflows. The adapter never implements approval state machines. |

## Implementation progress (repo)

- [x] Shared MCP HTTP + security spine exists: `packages/ameide_mcp` (Streamable HTTP mux, PRM handler, origin policy, dev + Keycloak JWT verification, conformance tests).
- [x] Capability-scoped MCP adapters exist: `primitives/integration/transformation-mcp-adapter` and `primitives/integration/sre-mcp-adapter`.
- [x] SRE adapter serves `initialize`, `tools/list`, `tools/call`, `resources/list`, `resources/read` over stdio + HTTP, with basic scope gating and PRM endpoint.
- [x] CLI scaffold template exists for Integration MCP adapters (embedded templates under `packages/ameide_core_cli/internal/commands/templates/integration/mcp_adapter`).
- [ ] Tool catalogs are not proto-derived yet (SRE tools are hand-authored; transformation adapter is currently shape-only).
- [ ] `(ameide.mcp.expose)` (or equivalent) proto annotations and the generation pipeline are not implemented.
- [ ] MCP adapters are not consistently referenced by any proto surface yet (e.g., `INTEGRATION/sre-mcp-adapter` has no proto files referencing the primitive, so verification skips naming checks).
- [ ] Trace/correlation propagation from MCP requests into `RequestContext` is not standardized (e.g., `primitives/integration/sre-mcp-adapter/internal/mcp/server.go` generates new IDs and drops trace context).
- [ ] Platform-wide tool registry / generated catalog artifact is not implemented.

## Clarifications requested (next steps)

- [ ] Choose the canonical proto annotation surface for MCP exposure (new `common/v1/mcp.proto` options vs reusing `common/v1/annotations.proto`), and name the option (`ameide.mcp.expose`?).
- [ ] Define minimum “DONE” transport set for adapters: `http` only vs `http + stdio`, and whether localhost-enforcement is mandatory for dev mode.
- [ ] Confirm auth baseline: Keycloak issuer/JWKS discovery inputs, required scopes, and whether PRM should advertise a single auth server or per-tenant realm.
- [ ] Decide whether adapters must always propagate W3C trace context (`traceparent`/`tracestate`) into gRPC metadata and/or `RequestContext.metadata`.
- [ ] Decide if platform agents are allowed to use MCP in-cluster (ADR-07 says “no”); if not, document the required typed SDK tool surfaces that mirror MCP exactly.
- [ ] Confirm whether “write tools default to proposals” is a hard requirement for all capabilities or capability-specific policy.

---

## 0) Definition

**MCP (Model Context Protocol)** is an open protocol for AI-tool integrations. In Ameide, MCP is a **protocol adapter** — another way to expose capability data alongside gRPC and REST.

MCP is **not** a new primitive type. It is a protocol variant that:
- Exposes existing Domain and Projection services to AI agents
- Follows CQRS: commands route to Domain, queries route to Projection
- Runs one adapter per capability (not a platform-wide gateway)
- Provides the standard interface for **agentic access** across all agent types

---

## 1) Architectural fit

### 1.1 MCP adapter as Integration primitive

MCP adapter is an **Integration primitive** — it adapts external protocols (MCP/JSON-RPC) to internal protocols (gRPC). It is not a new primitive type; it follows the Integration primitive pattern from `backlog/520-primitives-stack-v6.md`.

```
┌─────────────────────────────────────────────────────────────┐
│                     Capability                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Domain (writes)              Projection (reads)           │
│   ┌─────────────┐              ┌─────────────┐              │
│   │ WriteService│              │QueryService │              │
│   │   (gRPC)    │              │   (gRPC)    │              │
│   └──────┬──────┘              └──────┬──────┘              │
│          │                            │                     │
│          └────────────┬───────────────┘                     │
│                       │                                     │
│              ┌────────┴────────┐                            │
│              │  Integration    │                            │
│              │  Primitives     │                            │
│              ├─────────────────┤                            │
│              │ gRPC (native)   │                            │
│              │ REST (optional) │                            │
│              │ MCP (AI agents) │ ◄── This document          │
│              └─────────────────┘                            │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 CQRS routing

MCP operations map to CQRS sides:

| MCP Concept | CQRS Side | Target |
|-------------|-----------|--------|
| Tool (command) | Write | Domain WriteService |
| Tool (query) | Read | Projection QueryService |
| Resource | Read | Projection QueryService |

The MCP adapter is a thin translation layer — it does not own state or business logic.

Process posture (non-default):

- Process primitives are orchestration. Do not expose internal workflow steps as MCP tools by default.
- If process operations are exposed, keep them deliberate and high-level (e.g., `StartTransformationRun`, `GetRunStatus`) so governance/invariants are not bypassed.

### 1.3 One adapter per capability

Each capability owns its MCP surface:
- `transformation-mcp-adapter` exposes Transformation tools/resources
- `commerce-mcp-adapter` exposes Commerce tools/resources
- No platform-wide MCP gateway (avoids coupling and single point of failure)

---

## 2) Agentic access architecture

### 2.1 What is agentic access?

**Agentic access** is an access pattern where AI agents (LLMs with tool-calling capabilities) interact with platform capabilities. All agents — regardless of where they run — need tools to:
- **Read** data (query projections)
- **Write** data (submit intents to domains)
- **Reason** about what to do

MCP provides the standard tool interface for agentic access.

### 2.2 Agent types and MCP

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Agent Types                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Platform Agents (headless)         Human-Driven Agents             │
│  ───────────────────────────        ─────────────────────           │
│  - Run inside platform              - Interactive, human in loop    │
│  - React to facts/events/schedules  - Triggered by conversation     │
│  - Autonomous execution             - User reviews actions          │
│  - AgentDefinition governs          - User auth governs             │
│                                                                     │
│  Examples:                          Examples:                       │
│  - architecture-reviewer            - Browser chat agent            │
│  - drift-detector                   - Claude Code                   │
│  - backlog-groomer                  - Copilot                       │
│                                     - Any MCP-compatible client     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │   MCP Tools     │
                    │  (same for all) │
                    └────────┬────────┘
                             │
               ┌─────────────┴─────────────┐
               ▼                           ▼
          Domain                      Projection
         (writes)                      (reads)
```

**Key insight**: Capabilities publish a single, proto-first **tool catalog** (tools/resources), and MCP is one standard protocol binding for consuming that catalog. The difference between agent types is:
- **What triggers them** (facts vs human conversation)
- **What governs them** (AgentDefinition vs user auth)
- **How they invoke tools** (SDK-backed internal tools vs MCP protocol adapter)

### 2.3 AgentDefinition vs Agentic Access

These are two separate concepts:

| Concept | What it is | Scope |
|---------|------------|-------|
| **AgentDefinition** | Deployment artifact for platform agents | Defines runtime, triggers, workflows for agents that run *in* the platform |
| **Agentic Access** | Access pattern via capability tools/resources (MCP-compatible) | How *any* agent (internal or external) interacts with capabilities |

**AgentDefinition** specifies:
- What container/runtime runs the agent
- What triggers the agent (facts, schedules, signals)
- What workflows the agent executes
- What tools the agent is allowed to use (whitelist)

**Agentic Access** is:
- A capability-owned tool/resource surface (queries → Projection; commands → Domain)
- Optionally exposed via MCP (external/devtool compatibility) through an Integration primitive
- Governed by the caller's auth (user or service account)

```
Platform Agent (AgentDefinition)
├── Deployed via AgentDefinition CR
├── Runs with service account
├── Tool access filtered by AgentDefinition.spec.tools.allow
└── Uses capability tools (SDK-backed or MCP) ──► Domain/Projection

External Agent (no AgentDefinition)
├── Runs outside platform (Claude Code, browser, etc.)
├── Runs with user's auth token
├── Tool access = user's permissions
└── Uses MCP tools (via protocol adapter) ──► Domain/Projection
```

### 2.4 Why MCP exists (and why agents don’t depend on it internally)

**Rationale:**

1. **Single tool definition per capability (proto-first)** — Proto annotations are the source of truth for what tools exist; MCP schemas/manifests are generated from proto (no drift).

2. **Portable agent behaviors** — External devtool agents can exercise the same capability surfaces that platform agents use, without forcing platform agents to adopt a JSON-RPC runtime boundary.

3. **Consistent interface testing** — The MCP adapter provides a standardized compatibility surface that can be compliance-tested independently of any specific agent runtime.

4. **LLM-native interface for external ecosystems** — MCP is designed for AI tool calling (JSON schemas, structured errors, resource references).

5. **Ecosystem compatibility** — MCP is an open standard. Capability tool surfaces can be consumed by Claude Code, Copilot, and future assistants.

### 2.5 Governance model

| Agent Type | Identity | Tool Access | Governance |
|------------|----------|-------------|------------|
| Platform agent (headless) | Service account | Filtered by AgentDefinition | Platform-managed |
| Browser chat agent | User token | User's role permissions | User-managed |
| External agent (Claude Code) | User token | User's role permissions | User-managed |

**Platform agents** are constrained by AgentDefinition:
```yaml
apiVersion: ameide.io/v1
kind: AgentDefinition
metadata:
  name: architecture-reviewer
spec:
  runtime:
    image: ameide/architecture-reviewer:v1
  triggers:
    - type: fact
      topic: transformation.architecture.domain.facts.v1
  tools:
    allow:
      - transformation.listElements
      - transformation.getView
      - transformation.searchCapabilities
    deny:
      - transformation.submit*  # read-only agent
```

**Human-driven agents** (browser chat, Claude Code) inherit the user's permissions:
- An Analyst via Claude Code can read what Analyst can read in browser UI
- An Architect via Claude Code can write what Architect can write in browser UI
- MCP is just another protocol, not a privilege boundary

### 2.6 MCP transports (per MCP spec 2025-11-25)

MCP defines two standard transports. Implementations MUST support at least one:

| Transport | Use Case | Binding |
|-----------|----------|---------|
| **stdio** | Local agents (Claude Code, CLI tools) | Subprocess stdin/stdout |
| **Streamable HTTP** | Remote agents (browser chat, platform agents) | HTTP POST with optional SSE streaming |

**Transport selection by agent type:**

| Agent Type | Recommended Transport | Notes |
|------------|----------------------|-------|
| Platform agent (headless) | Streamable HTTP or in-process | Same cluster, service mesh routing |
| Browser chat agent | Streamable HTTP | Web-compatible, firewall-friendly |
| External (Claude Code) | stdio (local) or Streamable HTTP (remote) | stdio for dev, HTTP for shared envs |

**Protocol notes:**
- **stdio**: JSON-RPC messages over stdin/stdout. No HTTP overhead. Best for local subprocess invocation.
- **Streamable HTTP**: JSON-RPC over HTTP POST. Response may be SSE stream for long-running operations.
- **HTTP+SSE (legacy)**: Older pattern with separate SSE endpoint. Deprecated in favor of Streamable HTTP.
- **WebSocket**: Custom transport, not required by MCP spec. Avoid unless specific requirements demand it.

The MCP adapter MUST support stdio and Streamable HTTP. The tool interface is identical across transports.

### 2.7 Client transport compatibility

Not all MCP clients support Streamable HTTP equally. Current client compatibility:

| Client | Streamable HTTP | stdio | HTTP+SSE (legacy) |
|--------|-----------------|-------|-------------------|
| **VS Code Copilot** | ✅ | ✅ | ❌ |
| **Claude Code** | ✅ | ✅ | ❌ |
| **Codex (OpenAI)** | ✅ | ✅ | ? |
| **Cline** | ⚠️ buggy ([#6767](https://github.com/cline/cline/issues/6767)) | ✅ | ✅ |

**Cline users:** Due to ongoing Streamable HTTP issues in Cline, users may need to:
1. Use stdio transport for local development, or
2. Wait for Cline to fix Streamable HTTP support

The platform MCP adapter prioritizes Streamable HTTP as the primary remote transport per MCP spec 2025-11-25. Legacy HTTP+SSE is not implemented unless Cline compatibility becomes a hard requirement.

---

## 3) MCP protocol semantics

MCP uses **JSON-RPC 2.0** over the selected transport. Understanding the protocol semantics is essential for correct implementation.

### 3.1 JSON-RPC method naming (per MCP spec)

MCP defines standard JSON-RPC methods. These are **not REST endpoints**:

| JSON-RPC Method | Purpose | Direction |
|-----------------|---------|-----------|
| `initialize` | Handshake, capability negotiation | Client → Server |
| `tools/list` | List available tools | Client → Server |
| `tools/call` | Invoke a tool | Client → Server |
| `resources/list` | List available resources | Client → Server |
| `resources/read` | Read a resource | Client → Server |
| `prompts/list` | List available prompts | Client → Server |
| `prompts/get` | Get a prompt | Client → Server |

**Example JSON-RPC request:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "transformation.listElements",
    "arguments": {
      "model_id": "platform-architecture",
      "layer": "application"
    }
  }
}
```

**Example JSON-RPC response:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      { "type": "text", "text": "[{\"id\": \"orders-domain\", ...}]" }
    ]
  }
}
```

### 3.2 Tool naming convention

Tool names follow the pattern defined in `backlog/509-proto-naming-conventions-v6.md`:
```
<capability>.<operation>
```

**Commands** (→ Domain WriteService):
```
transformation.submitArchitectureIntent
transformation.submitScrumIntent
transformation.createBaseline
commerce.submitOrderIntent
commerce.cancelOrder
```

**Queries** (→ Projection QueryService):
```
transformation.listElements
transformation.listRelationships
transformation.getView
transformation.searchCapabilities
commerce.getOrder
commerce.listProducts
commerce.searchInventory
```

### 3.3 Tool calling lifecycle (chat → tools/call → domain)

MCP tools are not "run by the LLM". In an agentic chat:

* The **LLM produces a tool call intent** (tool name + JSON arguments).
* The **client/runtime executes** the call over MCP (`tools/call`).
* The **MCP adapter normalizes/enriches** the request and routes it to the correct backend (CQRS).
* The **Domain** is authoritative for validation/invariants and persistence.
* The **tool result** is returned to the LLM, which then writes a human confirmation.

ASCII sequence (write example):

```text
User (chat)
 │  "Create capability X with owner=Supply Chain, criticality=High"
 ▼
LLM (vendor runtime)
 │  - Tool calling enabled
 │  - Structured Output / strict tools ON (if supported)
 │  - Temperature=0 (recommended for writes)
 │  => emits tool call JSON that matches tool schema
 ▼
MCP Client / Agent runtime
 │  JSON-RPC: tools/call
 │  - Schema validation (client-side)
 │  - May show confirmation modal for write operations (ChatGPT behavior)
 ▼
MCP Protocol Adapter
 │  - parse/validate input shape
 │  - enrich (defaults, IDs, mapping)
 │  - route (CQRS → write side)
 ▼
Domain (write side)
 │  - validate invariants (including internal structure rules)
 │  - enforce idempotency
 │  - persist + emit events
 ▼
Adapter returns tool result
 │  canonical created object (final structure + ids + revision + warnings)
 ▼
MCP Client → LLM
 │  tool result processed
 ▼
User sees confirmation + next steps
```

**Key contract:** the tool's JSON schema is the "shape" contract. If the client/model supports strict structured output, configure it so the LLM must produce arguments conforming to the tool schema for write operations.

See also: `backlog/536-mcp-write-optimizations.md` §6 for detailed write flow.

### 3.4 Resource URI convention

```
<capability>://<type>/<id>
```

Resources provide entity-level context injection:
```
transformation://model/platform-architecture
transformation://element/orders-domain
transformation://view/capability-map
transformation://baseline/2024-q4
commerce://order/ord-12345
commerce://product/sku-abc
```

Resources are read-only and resolve via Projection QueryService through `resources/read`.
If a capability interaction requires parameters or computation, model it as a tool, not a resource.

### 3.5 Tool input/output mapping

MCP tools accept JSON arguments and return structured content. The adapter translates:

```
JSON-RPC tools/call                        gRPC Call
─────────────────────────────────────────────────────────────────────
method: "tools/call"
params:
  name: "transformation.listElements"      ArchiMateQueryService.ListElements
  arguments:                               ListElementsRequest {
    model_id: "abc-123"                      model_id: "abc-123"
    layer: "application"                     layer: APPLICATION
  }                                        }

result:                                    ListElementsResponse (protobuf)
  content: [{ type: "text", text: "..." }]   → JSON serialized
```

### 3.6 Structured tool output (preferred)

MCP supports multiple content types in tool responses. **Prefer structured JSON output** over text-wrapped JSON:

**❌ Avoid: JSON wrapped in text**
```json
{
  "content": [
    { "type": "text", "text": "[{\"id\": \"orders-domain\", \"name\": \"Orders\"}]" }
  ]
}
```

**✅ Prefer: Structured JSON content**
```json
{
  "content": [
    {
      "type": "resource",
      "resource": {
        "uri": "transformation://element/orders-domain",
        "mimeType": "application/json",
        "text": "{\"id\": \"orders-domain\", \"name\": \"Orders\", \"type\": \"ApplicationComponent\"}"
      }
    }
  ]
}
```

**Or multiple resources:**
```json
{
  "content": [
    {
      "type": "resource",
      "resource": {
        "uri": "transformation://element/orders-domain",
        "mimeType": "application/json",
        "text": "{\"id\": \"orders-domain\", \"name\": \"Orders\"}"
      }
    },
    {
      "type": "resource",
      "resource": {
        "uri": "transformation://element/inventory-domain",
        "mimeType": "application/json",
        "text": "{\"id\": \"inventory-domain\", \"name\": \"Inventory\"}"
      }
    }
  ]
}
```

**Benefits of structured output:**
1. **Parseable** — Clients can extract individual results without JSON parsing nested strings
2. **Addressable** — Each result has a URI for context injection or follow-up queries
3. **Type-aware** — `mimeType` tells clients how to render/process the content
4. **Composable** — Results can be mixed with text explanations in the same response

**Implementation:**
```go
func (s *Server) formatQueryResults(results []*pb.Element) []MCPContent {
    content := make([]MCPContent, 0, len(results))
    for _, elem := range results {
        content = append(content, MCPContent{
            Type: "resource",
            Resource: &MCPResource{
                URI:      fmt.Sprintf("transformation://element/%s", elem.Id),
                MimeType: "application/json",
                Text:     mustMarshal(elem),
            },
        })
    }
    return content
}
```

### 3.7 ReadContext and citation discipline

**All query tools MUST support `read_context`** for reproducible, auditable queries:

```yaml
# ReadContext — common input parameter
read_context:
  type: object
  description: "Point-in-time read context for reproducible queries"
  properties:
    selector:
      type: string
      description: "Selector mode (tool-specific default; for agent memory tools, default is published)"
      enum: ["head", "published", "baseline_id", "as_of"]
    baseline_id:
      type: string
      description: "Query against a specific published baseline"
    as_of:
      type: string
      format: date-time
      description: "Query as of a specific timestamp"
```

**All query tool responses MUST include citation metadata:**

| Field | Description |
|-------|-------------|
| `repository_id` | Repository scope of the cited item |
| `element_id` | Stable element identifier |
| `version_id` | Immutable element version (303/656) |
| `read_context` | Effective read context used (selector/baseline/as_of) |

**Why this matters:**
- Agents making multiple calls in a session see consistent state
- Impact analysis requires comparing baselines
- Audit trails can reproduce exactly what data an agent saw
- Governance requires traceability to specific versions

See `backlog/535-mcp-read-optimizations.md` §5.2 for full specification.

---

## 4) Capability MCP manifest

Each capability declares its MCP surface via a manifest (conceptual schema):

```yaml
# transformation-mcp-manifest.yaml
apiVersion: ameide.io/v1
kind: MCPManifest
metadata:
  name: transformation
  version: v1
spec:
  # Commands route to Domain
  commands:
    - name: submitArchitectureIntent
      description: Submit an architecture domain intent (create/update/delete ArchiMate elements, relationships, views)
      target:
        service: transformation-v0-domain
        port: 50051
        rpc: TransformationArchitectureWriteService.SubmitIntent
      input:
        schema: TransformationArchitectureDomainIntent

    - name: submitScrumIntent
      description: Submit a Scrum domain intent (backlog items, sprints, goals)
      target:
        service: transformation-v0-domain
        port: 50051
        rpc: ScrumWriteService.SubmitIntent
      input:
        schema: ScrumDomainIntent

  # Queries route to Projection
  queries:
    - name: getArchimateModel
      description: Get an ArchiMate model by ID
      target:
        service: transformation-v0-projection
        port: 50051
        rpc: ArchiMateQueryService.GetArchimateModel
      input:
        properties:
          model_id: { type: string, required: true }

    - name: listElements
      description: List ArchiMate elements with optional filtering
      target:
        service: transformation-v0-projection
        port: 50051
        rpc: ArchiMateQueryService.ListArchimateElements
      input:
        properties:
          model_id: { type: string, required: true }
          layer: { type: string, enum: [strategy, business, application, technology] }
          element_type: { type: string }
          page_size: { type: integer, default: 50 }
          page_token: { type: string }
          # ReadContext — point-in-time query support (see §3.6)
          read_context:
            type: object
            properties:
              baseline_id: { type: string }
              as_of: { type: string, format: date-time }

    - name: listRelationships
      description: List ArchiMate relationships for an element
      target:
        service: transformation-v0-projection
        port: 50051
        rpc: ArchiMateQueryService.ListArchimateRelationships
      input:
        properties:
          model_id: { type: string, required: true }
          element_id: { type: string }
          relationship_type: { type: string }
          read_context:
            type: object
            properties:
              baseline_id: { type: string }
              as_of: { type: string, format: date-time }

    - name: getView
      description: Get an ArchiMate view with layout
      target:
        service: transformation-v0-projection
        port: 50051
        rpc: ArchiMateQueryService.GetArchimateView
      input:
        properties:
          view_id: { type: string, required: true }
          read_context:
            type: object
            properties:
              baseline_id: { type: string }
              as_of: { type: string, format: date-time }

    - name: searchCapabilities
      description: Search capabilities by name, description, or tags
      target:
        service: transformation-v0-projection
        port: 50051
        rpc: ArchiMateQueryService.SearchElements
      input:
        properties:
          query: { type: string, required: true }
          layer: { type: string, default: strategy }
          element_type: { type: string, default: Capability }
          read_context:
            type: object
            properties:
              baseline_id: { type: string }
              as_of: { type: string, format: date-time }

  # Resources for context injection
  resources:
    - pattern: "model/{model_id}"
      description: ArchiMate model
      rpc: ArchiMateQueryService.GetArchimateModel

    - pattern: "element/{element_id}"
      description: ArchiMate element
      rpc: ArchiMateQueryService.GetElement

    - pattern: "view/{view_id}"
      description: ArchiMate view with layout
      rpc: ArchiMateQueryService.GetArchimateView

    - pattern: "baseline/{baseline_id}"
      description: Promoted baseline
      rpc: BaselineQueryService.GetBaseline
```

---

## 5) Authentication, authorization, and tenant isolation

### 5.1 MCP OAuth 2.1 (Primary — per MCP Authorization spec)

Per MCP Authorization specification and industry best practice (GitHub, Port, Claude Code, VS Code), the platform MCP adapter implements **OAuth 2.1**:

- **MCP adapter = Resource Server (RS)** — validates tokens, enforces scopes
- **Keycloak = Authorization Server (AS)** — issues tokens, manages users/roles

**MCP clients handle OAuth themselves.** No local proxy binary or token brokering needed.

```
┌─────────────────────────────────────────────────────────────────────┐
│  MCP Client (Claude Code, VS Code, Cursor)                          │
│                                                                     │
│  1. Discovers Protected Resource Metadata (PRM) from MCP adapter    │
│     GET https://mcp.transformation.ameide.io/.well-known/oauth-...  │
│                                                                     │
│  2. Fetches AS metadata from Keycloak (from PRM.authorization_servers)
│     GET https://auth.ameide.io/realms/ameide/.well-known/openid-... │
│                                                                     │
│  3. Runs OAuth flow (browser redirect with PKCE)                    │
│     User authenticates to Keycloak                                  │
│                                                                     │
│  4. Stores/refreshes tokens automatically                           │
│                                                                     │
│  5. Calls MCP adapter with Authorization: Bearer <token>            │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 Required endpoints (MCP adapter)

Per MCP Authorization spec (RFC 9728 Protected Resource Metadata), the MCP adapter MUST expose:

| Endpoint | Purpose | Spec |
|----------|---------|------|
| `/.well-known/oauth-protected-resource` | Protected Resource Metadata (PRM) | RFC 9728 |

**PRM response example:**
```json
{
  "resource": "https://mcp.transformation.ameide.io",
  "authorization_servers": ["https://auth.ameide.io/realms/ameide"],
  "scopes_supported": ["mcp:tools", "mcp:resources", "openid", "profile"],
  "bearer_methods_supported": ["header"]
}
```

MCP clients use `authorization_servers` to discover Keycloak's OIDC metadata:
- `GET https://auth.ameide.io/realms/ameide/.well-known/openid-configuration`

### 5.3 MCP-specific scopes

Define these scopes in Keycloak and include in access tokens:

| Scope | Purpose | Required for |
|-------|---------|--------------|
| `mcp:tools` | Invoke MCP tools (`tools/call`) | All tool invocations |
| `mcp:resources` | Read MCP resources (`resources/read`) | Resource access |
| `openid` | Standard OIDC | User identity |
| `profile` | User profile claims | Display name |

**Keycloak client scope configuration:**
```yaml
# Create client scopes in Keycloak
clientScopes:
  - name: mcp:tools
    description: "Invoke MCP tools"
    protocol: openid-connect
    attributes:
      include.in.token.scope: "true"
  - name: mcp:resources
    description: "Read MCP resources"
    protocol: openid-connect
    attributes:
      include.in.token.scope: "true"
```

**Scope enforcement in MCP adapter:**
```go
func (s *Server) handleToolsCall(req *JSONRPCRequest) error {
    // Require mcp:tools scope for tool invocations
    if !hasScope(req.Token, "mcp:tools") {
        return &JSONRPCError{
            Code:    -32001,
            Message: "insufficient_scope",
            Data:    map[string]string{"required_scope": "mcp:tools"},
        }
    }
    // ... proceed with tool call
}
```

### 5.4 Dynamic Client Registration (DCR) — Optional

VS Code tries DCR first before falling back to pre-configured client credentials.

**Option A: Enable DCR (best UX for VS Code)**

Keycloak supports DCR via `/realms/{realm}/clients-registrations/openid-connect`. To enable:

1. Create an **Initial Access Token** in Keycloak admin console
2. MCP adapter implements `/register` endpoint that proxies to Keycloak DCR
3. Enforce safety: allowlist redirect URIs, rate-limit registrations

```
POST /register
Content-Type: application/json

{
  "client_name": "VS Code MCP Client",
  "redirect_uris": ["http://127.0.0.1:33418"],
  "grant_types": ["authorization_code"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none"
}
```

**Option B: No DCR (simpler, current default)**

Pre-create Keycloak clients for each MCP client type (VS Code, Claude Code). Users configure client_id manually if prompted.

**Current recommendation:** Start with Option B (pre-configured clients). Add DCR later if VS Code UX feedback requires it.

### 5.5 Keycloak client configuration

For MCP OAuth support, create Keycloak clients for each MCP client type:

```yaml
# VS Code MCP client
- clientId: ameide-mcp-vscode
  protocol: openid-connect
  publicClient: true                    # No client secret (PKCE required)
  standardFlowEnabled: true             # Authorization Code flow
  directAccessGrantsEnabled: false      # No Resource Owner Password
  attributes:
    pkce.code.challenge.method: "S256"  # Require PKCE
  redirectUris:
    - "http://127.0.0.1:33418"          # VS Code MCP loopback (fixed port)
    - "https://vscode.dev/redirect"     # VS Code Web
  webOrigins:
    - "vscode-webview://*"
  defaultClientScopes:
    - openid
    - profile
    - mcp:tools
    - mcp:resources

# Claude Code MCP client
- clientId: ameide-mcp-claude
  protocol: openid-connect
  publicClient: true
  standardFlowEnabled: true
  directAccessGrantsEnabled: false
  attributes:
    pkce.code.challenge.method: "S256"
  redirectUris:
    - "http://127.0.0.1:*"              # Claude Code loopback (dynamic port)
  defaultClientScopes:
    - openid
    - profile
    - mcp:tools
    - mcp:resources

# CLI/Device flow client (for ameide auth token)
- clientId: ameide-mcp-cli
  protocol: openid-connect
  publicClient: true
  standardFlowEnabled: false
  directAccessGrantsEnabled: false
  attributes:
    oauth2.device.authorization.grant.enabled: "true"  # Device Authorization Grant
  defaultClientScopes:
    - openid
    - profile
    - mcp:tools
    - mcp:resources
```

### 5.6 Audience handling (RFC 8707 workaround)

MCP spec references RFC 8707 Resource Indicators for audience-restricted tokens. **Keycloak does not yet support RFC 8707.**

**Workaround: Audience protocol mapper**

Create a protocol mapper in Keycloak to add the MCP adapter as token audience:

```yaml
# Add to each MCP client
protocolMappers:
  - name: mcp-audience
    protocol: openid-connect
    protocolMapper: oidc-audience-mapper
    config:
      included.client.audience: "ameide-mcp-adapter"
      id.token.claim: "false"
      access.token.claim: "true"
```

**MCP adapter token validation:**
```go
func validateToken(token *jwt.Token) error {
    claims := token.Claims.(jwt.MapClaims)

    // Validate audience includes MCP adapter
    aud := claims["aud"]
    if !containsAudience(aud, "ameide-mcp-adapter") {
        return errors.New("invalid audience")
    }

    // Validate issuer
    if claims["iss"] != "https://auth.ameide.io/realms/ameide" {
        return errors.New("invalid issuer")
    }

    return nil
}
```

### 5.7 Bearer token fallback (secondary)

For clients that don't support full OAuth (CI, scripts, Cursor env vars):

```bash
# Cursor / env var mode
claude mcp add transformation \
  --transport http \
  --url https://mcp.transformation.ameide.io \
  --header "Authorization: Bearer ${AMEIDE_TOKEN}"

# CLI helper using Device Authorization Grant
ameide auth token  # Opens browser, returns token
export AMEIDE_TOKEN=$(ameide auth token --quiet)
```

The adapter validates Bearer tokens the same way — no separate code path.

### 5.8 Tenant isolation

Tenant ID is:
- Extracted from token claims (`tenantId` claim)
- Injected into all gRPC request metadata
- Enforced by Domain/Projection (same as direct gRPC access)

### 5.9 Authorization model

MCP tools inherit the same authorization as their underlying gRPC operations:
- `submitArchitectureIntent` requires write permission on Transformation domain + `mcp:tools` scope
- `listElements` requires read permission on Transformation projection + `mcp:tools` scope

**Layered authorization:**
1. **Scope check** — Does token have `mcp:tools` or `mcp:resources`?
2. **Role check** — Does user's role allow this operation?
3. **Tenant check** — Is user authorized for this tenant?

No MCP-specific permissions layer beyond scopes.

---

## 6) Single source of truth (Proto-first)

### 6.1 Proto as authoritative schema

**Proto is the single source of truth** for all API contracts. MCP tool schemas MUST be generated from proto definitions, not hand-written.

```
Proto Definition (authoritative)
        │
        ▼
┌───────────────────┐
│  Code Generation  │
├───────────────────┤
│ • gRPC stubs      │
│ • MCP tool schemas│
│ • OpenAPI specs   │
└───────────────────┘
```

### 6.2 Generation pipeline

```bash
# Proto compilation generates MCP schemas alongside gRPC stubs
buf generate --template buf.gen.yaml

# buf.gen.yaml includes MCP schema plugin
plugins:
  - plugin: go-grpc
    out: gen/go
  - plugin: mcp-schema
    out: gen/mcp
    opt:
      - expose_annotation=ameide.mcp.expose
```

### 6.3 Proto annotations for MCP exposure

Proto services/methods use annotations to control MCP exposure:

```protobuf
import "ameide/mcp/annotations.proto";

service ArchiMateQueryService {
  // Exposed as MCP tool (read-only query)
  rpc ListElements(ListElementsRequest) returns (ListElementsResponse) {
    option (ameide.mcp.expose) = {
      tool_name: "transformation.listElements"
      description: "List ArchiMate elements with optional filtering"
      hints: {
        read_only: true      // Maps to MCP readOnlyHint
        destructive: false   // Maps to MCP destructiveHint
        idempotent: true     // Maps to MCP idempotentHint
      }
    };
  }

  // NOT exposed to MCP (internal only)
  rpc InternalReconcile(ReconcileRequest) returns (ReconcileResponse);
}

service TransformationWriteService {
  // Exposed as MCP tool (write command)
  rpc SubmitArchitectureIntent(SubmitIntentRequest) returns (SubmitIntentResponse) {
    option (ameide.mcp.expose) = {
      tool_name: "transformation.submitArchitectureIntent"
      description: "Submit an architecture intent (create/update/delete elements)"
      hints: {
        read_only: false
        destructive: true    // May modify/delete data
        idempotent: false    // Repeated calls create duplicates
      }
    };
  }
}
```

### 6.4 Tool annotations (per MCP spec 2025-11-25)

MCP spec 2025-11-25 introduces **tool annotations** that help clients make intelligent decisions about tool invocation. The platform MCP adapter SHOULD include these hints:

| Hint | Proto Field | MCP Field | Purpose |
|------|-------------|-----------|---------|
| Read-only | `hints.read_only` | `readOnlyHint` | Tool only reads data, never modifies |
| Destructive | `hints.destructive` | `destructiveHint` | Tool may delete or irreversibly modify data |
| Idempotent | `hints.idempotent` | `idempotentHint` | Repeated calls with same args have same effect |
| Open World | `hints.open_world` | `openWorldHint` | Tool interacts with external systems |

**CQRS mapping:**
- **Queries** (→ Projection): `readOnlyHint: true`, `destructiveHint: false`
- **Commands** (→ Domain): `readOnlyHint: false`, `destructiveHint: true/false` (depends on operation)

Clients may use these hints to:
- Auto-approve read-only tool calls
- Require confirmation for destructive operations
- Retry idempotent operations on transient failures

### 6.5 Proto file locations (per capability)

**Transformation capability:**
- Proto package: `ameide_core_proto.transformation.core.v1`
- File location: `packages/ameide_core_proto/src/ameide_core_proto/transformation/`
- Contract catalog: [527-transformation-proto.md](527-transformation-proto.md)
- Topic families: `transformation.domain.intents.v1`, `transformation.domain.facts.v1`

**MCP annotation definitions:**
- Package: `ameide_core_proto.mcp.annotations.v1`
- File: `packages/ameide_core_proto/src/ameide_core_proto/mcp/annotations.proto`

### 6.6 Benefits of proto-first

1. **No schema drift** — MCP schemas always match gRPC contracts
2. **Single edit point** — Change proto, regenerate everything
3. **Reviewable exposure** — `(ameide.mcp.expose)` annotation is visible in code review
4. **Type safety** — JSON schema derived from proto types

### 6.7 Portable schema constraints

Generated JSON schemas MUST be compatible across major LLM vendors. Follow these constraints:

**Supported JSON Schema features (portable subset):**
- Core types: `object`, `properties`, `required`, `enum`, `array`, `items`, `type`
- String validation: `minLength`, `maxLength`, `pattern`
- Number validation: `minimum`, `maximum`, `exclusiveMinimum`, `exclusiveMaximum`
- Descriptions: `description`, `title`

**Limited or vendor-specific (use with caution):**
- Combinators: `anyOf`, `oneOf`, `allOf` (limited support in Google Vertex AI, Amazon Nova)
- Deep nesting: Keep object nesting ≤ 2 levels (Amazon Nova best practice; deeper nesting degrades performance)
- Advanced keywords: `dependencies`, `patternProperties`, `additionalProperties: false` (inconsistent vendor support)

**Schema size considerations:**
- Schema definitions count toward token budget (Google Vertex AI)
- Large schemas increase first-call latency (Cohere caches after initial requests)
- Keep total schema size < 2000 tokens for optimal performance

**Proto-to-JSON-Schema mapping guidance:**

```protobuf
// ✅ GOOD: Simple, portable schema
message CreateElementRequest {
  string type = 1;           // → type: string, enum: [...]
  string name = 2;           // → type: string, minLength: 1
  string description = 3;    // → type: string (optional)

  message Profile {          // → Nested 1 level (portable)
    string owner = 1;
    string criticality = 2;  // → enum: ["Low", "Medium", "High"]
  }
  Profile profile = 4;
}

// ⚠️ AVOID: Deep nesting, complex validation
message CreateComplexElement {
  oneof type_selector {      // → oneOf (limited vendor support)
    CapabilitySpec capability = 1;
    ComponentSpec component = 2;
  }

  message NestedLevel1 {
    message NestedLevel2 {
      message NestedLevel3 { // → 3+ levels (degrades Nova performance)
        // ...
      }
    }
  }
}
```

**Tool input examples validation:**

All tool input examples in documentation and tests MUST validate against the published schema. Anthropic Claude API rejects tool definitions with invalid examples (HTTP 400).

**Proto codegen enforcement:**

The `mcp-schema` buf plugin SHOULD:
- Warn on schemas exceeding 2000 tokens
- Error on nesting depth > 2 levels
- Validate that examples conform to schema
- Document vendor-specific feature usage

See also: `backlog/536-mcp-write-optimizations.md` §1.2 for vendor compatibility matrix.

---

## 7) Tool exposure policy

### 7.1 Default deny

MCP adapters follow a **default deny** policy for tool exposure:

- Only methods with explicit `(ameide.mcp.expose)` annotation are exposed
- Unannotated methods are NOT available via MCP
- This prevents accidental exposure of internal operations

### 7.2 Allowlist via proto annotations

The `(ameide.mcp.expose)` annotation serves as the allowlist:

```protobuf
// EXPOSED: Has annotation
rpc ListElements(...) returns (...) {
  option (ameide.mcp.expose) = { tool_name: "transformation.listElements" };
}

// NOT EXPOSED: No annotation
rpc InternalMigrate(...) returns (...);
```

### 7.3 AgentDefinition tool filtering

Platform agents are further constrained by AgentDefinition:

```yaml
apiVersion: ameide.io/v1
kind: AgentDefinition
metadata:
  name: architecture-reviewer
spec:
  tools:
    allow:
      - transformation.listElements
      - transformation.getView
    deny:
      - transformation.submit*  # Glob pattern
```

The effective tool set is: `proto-exposed ∩ agent-allowed`

### 7.4 Audit trail

All tool invocations are logged with:
- Caller identity (user or service account)
- Tool name and arguments
- Timestamp and trace ID
- Result status

---

## 8) Security requirements (Streamable HTTP)

### 8.1 Mandatory for Streamable HTTP transport

When exposing MCP via Streamable HTTP, implementations MUST enforce:

| Requirement | Description |
|-------------|-------------|
| **Origin validation** | Validate `Origin` header against allowlist |
| **HTTPS only** | TLS required for production deployments |
| **Authentication** | Bearer token or mTLS required |
| **Rate limiting** | Per-caller rate limits to prevent abuse |

### 8.2 Origin validation

```go
// Allowlist of permitted origins
var allowedOrigins = []string{
    "https://portal.ameide.io",
    "https://claude.ai",
    "vscode-webview://*",
}

func validateOrigin(r *http.Request) error {
    origin := r.Header.Get("Origin")
    if origin == "" {
        return nil // Non-browser client (CLI, server-to-server)
    }
    for _, allowed := range allowedOrigins {
        if matchOrigin(origin, allowed) {
            return nil
        }
    }
    return fmt.Errorf("origin %s not allowed", origin)
}
```

### 8.3 Localhost binding for development

For local development, the MCP adapter MUST:
- Bind to `127.0.0.1` only (not `0.0.0.0`)
- Reject requests with non-localhost Origin headers
- Log a warning if attempting to bind to all interfaces

```go
// Development mode: localhost only
if config.DevMode {
    listener, _ := net.Listen("tcp", "127.0.0.1:8080")
}
```

### 8.4 stdio transport security

stdio transport inherits the security context of the parent process:
- No network exposure
- Caller identity from process environment
- Suitable for local development and CLI tools

### 8.5 Security acceptance tests (non-optional)

Security hardening for Streamable HTTP MUST be validated via acceptance tests. These tests are **non-optional** — the MCP adapter cannot be promoted without passing all of them.

#### Test Suite: `mcp_adapter_security_test.go`

| Test ID | Test Name | Description | Pass Criteria |
|---------|-----------|-------------|---------------|
| SEC-01 | `TestOriginValidation_RejectsUnknownOrigin` | Request with unknown Origin header | HTTP 403, error message "origin not allowed" |
| SEC-02 | `TestOriginValidation_AcceptsAllowedOrigin` | Request with allowed Origin header | HTTP 200, normal response |
| SEC-03 | `TestOriginValidation_AcceptsNoOrigin` | Request with no Origin header (server-to-server) | HTTP 200, normal response |
| SEC-04 | `TestHTTPS_RejectsHTTPInProduction` | HTTP request when `PRODUCTION=true` | Connection refused or HTTP 301 redirect to HTTPS |
| SEC-05 | `TestAuth_RejectsMissingToken` | Request without Authorization header | HTTP 401, error "authentication required" |
| SEC-06 | `TestAuth_RejectsInvalidToken` | Request with malformed/expired token | HTTP 401, error "invalid token" |
| SEC-07 | `TestAuth_RejectsWrongAudience` | Token with incorrect audience claim | HTTP 401, error "invalid audience" |
| SEC-08 | `TestAuth_AcceptsValidToken` | Request with valid Bearer token | HTTP 200, normal response |
| SEC-09 | `TestRateLimit_RejectsExcessiveRequests` | Burst of requests exceeding rate limit | HTTP 429 after threshold |
| SEC-10 | `TestRateLimit_AllowsNormalTraffic` | Normal request rate | All HTTP 200 |
| SEC-11 | `TestLocalhost_BindsTo127Only` | Check listening address in dev mode | Only 127.0.0.1 bound, not 0.0.0.0 |
| SEC-12 | `TestLocalhost_RejectsNonLocalhostOrigin` | Non-localhost Origin in dev mode | HTTP 403 |

#### Example Test Implementation:

```go
func TestOriginValidation_RejectsUnknownOrigin(t *testing.T) {
    srv := newTestMCPServer(t, WithAllowedOrigins("https://portal.ameide.io"))

    req := httptest.NewRequest("POST", "/mcp", strings.NewReader(`{"jsonrpc":"2.0","method":"tools/list","id":1}`))
    req.Header.Set("Origin", "https://evil.com")
    req.Header.Set("Content-Type", "application/json")

    w := httptest.NewRecorder()
    srv.ServeHTTP(w, req)

    assert.Equal(t, http.StatusForbidden, w.Code)
    assert.Contains(t, w.Body.String(), "origin not allowed")
}

func TestAuth_RejectsWrongAudience(t *testing.T) {
    srv := newTestMCPServer(t, WithExpectedAudience("ameide-mcp-adapter"))

    // Token with wrong audience
    token := issueTestToken(t, jwt.MapClaims{
        "aud": "wrong-audience",
        "sub": "user-123",
    })

    req := httptest.NewRequest("POST", "/mcp", strings.NewReader(`{"jsonrpc":"2.0","method":"tools/call","id":1,"params":{"name":"test"}}`))
    req.Header.Set("Authorization", "Bearer "+token)
    req.Header.Set("Content-Type", "application/json")

    w := httptest.NewRecorder()
    srv.ServeHTTP(w, req)

    assert.Equal(t, http.StatusUnauthorized, w.Code)
    assert.Contains(t, w.Body.String(), "invalid audience")
}

func TestLocalhost_BindsTo127Only(t *testing.T) {
    cfg := &MCPConfig{DevMode: true, Port: 0}
    srv, err := NewMCPServer(cfg)
    require.NoError(t, err)

    addr := srv.Listener().Addr().String()
    assert.True(t, strings.HasPrefix(addr, "127.0.0.1:"), "expected 127.0.0.1 binding, got %s", addr)
}
```

#### CI Integration:

```yaml
# .github/workflows/mcp-adapter.yml
- name: Run MCP Security Tests
  run: |
    go test -v -run "^Test.*Security|^TestOrigin|^TestAuth|^TestRateLimit|^TestLocalhost" \
      ./packages/ameide_mcp_adapters/...
  env:
    MCP_TEST_MODE: "security"
```

---

## 9) Deployment patterns

### 9.1 Local stdio (development / Claude Code)

For local development, the MCP adapter runs as a subprocess:

```bash
# Add to Claude Code
claude mcp add --transport stdio transformation -- \
  ./transformation-mcp-adapter \
  --domain-addr localhost:50051 \
  --projection-addr localhost:50052

# Or with port-forward to cluster
claude mcp add --transport stdio transformation -- \
  ./transformation-mcp-adapter \
  --domain-addr transformation-v0-domain.ameide-local:50051 \
  --projection-addr transformation-v0-projection.ameide-local:50051
```

### 9.2 Remote Streamable HTTP (shared / production)

For shared environments, the MCP adapter exposes an HTTP endpoint:

```bash
# Add to Claude Code
claude mcp add --transport http transformation \
  https://mcp.transformation.ameide.local \
  --header "Authorization: Bearer ${AMEIDE_TOKEN}"
```

### 9.3 Kubernetes deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: transformation-mcp-adapter
spec:
  template:
    spec:
      containers:
        - name: mcp-adapter
          image: ameide/transformation-mcp-adapter:v1
          ports:
            - containerPort: 8080  # HTTP MCP
          env:
            - name: DOMAIN_ADDR
              value: transformation-v0-domain:50051
            - name: PROJECTION_ADDR
              value: transformation-v0-projection:50051
---
apiVersion: v1
kind: Service
metadata:
  name: transformation-mcp
spec:
  ports:
    - port: 80
      targetPort: 8080
```

---

## 10) Implementation guidelines

### 10.1 Adapter structure (Go)

```
packages/
  ameide_mcp_adapters/
    transformation/
      cmd/
        transformation-mcp-adapter/
          main.go
      internal/
        server/
          mcp_server.go       # MCP protocol handling
        handlers/
          commands.go         # Command tool handlers
          queries.go          # Query tool handlers
          resources.go        # Resource handlers
        clients/
          domain_client.go    # gRPC client to Domain
          projection_client.go # gRPC client to Projection
      manifest.yaml           # MCP manifest
```

### 10.2 Adapter responsibilities

The MCP adapter:
- Parses MCP tool calls and resource requests
- Translates JSON ↔ protobuf
- Routes commands to Domain, queries to Projection
- Passes through auth/tenant context
- Handles pagination token translation
- Returns structured errors

The MCP adapter does NOT:
- Own any state
- Perform business logic
- Cache query results (projection owns caching)
- Aggregate across capabilities

### 10.3 Error mapping

| gRPC Status | MCP Error |
|-------------|-----------|
| NOT_FOUND | Resource not found |
| INVALID_ARGUMENT | Invalid tool input |
| PERMISSION_DENIED | Unauthorized |
| UNAVAILABLE | Service unavailable |

---

## 11) Transformation reference implementation

The Transformation capability serves as the reference MCP implementation:

### 11.1 Tools exposed

**Commands:**
- `transformation.submitArchitectureIntent` — create/update/delete ArchiMate elements, relationships, views
- `transformation.submitScrumIntent` — manage backlog items, sprints, goals
- `transformation.createBaseline` — create promotable baseline
- `transformation.promoteBaseline` — promote baseline to approved

**Queries:**
- `transformation.getArchimateModel` — get model by ID
- `transformation.listElements` — list/filter elements
- `transformation.listRelationships` — list relationships for element
- `transformation.getView` — get view with layout
- `transformation.searchCapabilities` — search Strategy layer capabilities
- `transformation.getBaseline` — get baseline details
- `transformation.listDefinitions` — list process/agent definitions

### 11.2 Resources exposed

- `transformation://model/{id}` — ArchiMate model
- `transformation://element/{id}` — ArchiMate element
- `transformation://view/{id}` — ArchiMate view
- `transformation://baseline/{id}` — Baseline
- `transformation://definition/{id}` — Process/Agent definition

### 11.3 Example usage (Claude Code)

```
User: What capabilities does the platform have?

Claude: [invokes transformation.searchCapabilities with query="*" layer="strategy"]

User: Show me the Orders domain and its relationships

Claude: [invokes transformation.listElements with element_type="ApplicationComponent" name="orders"]
        [invokes transformation.listRelationships with element_id="orders-domain"]

User: Create a new capability called "Fulfillment"

Claude: [invokes transformation.submitArchitectureIntent with intent to create Capability element]
```

---

## 12) Acceptance criteria

1. MCP adapter is documented as **Integration primitive** (not a new primitive type).
2. CQRS routing is explicit: commands → Domain, queries → Projection.
3. One MCP adapter per capability (no platform gateway).
4. Tool/resource naming conventions are defined and followed.
5. Auth/tenant isolation uses passthrough (no MCP-specific auth layer).
6. Transformation reference implementation exposes ArchiMate model data.
7. **Transports**: stdio and Streamable HTTP are supported (WebSocket is optional/custom).
8. **JSON-RPC semantics**: Protocol uses `tools/list`, `tools/call`, `resources/read` method naming.
9. **Proto-first**: MCP tool schemas are generated from proto definitions with `(ameide.mcp.expose)` annotation.
10. **Default deny**: Only annotated methods are exposed; unannotated methods are hidden.
11. **Security**: Streamable HTTP requires Origin validation, HTTPS, and localhost binding for dev. All 12 security acceptance tests (SEC-01 through SEC-12) must pass.
12. Agentic access is defined as an access pattern distinct from AgentDefinition.
13. Capability tool surfaces are shared across agent types; MCP is the standard compatibility binding for external/devtool ecosystems.
14. AgentDefinition governs platform agents; user auth governs human-driven agents.
15. **OAuth 2.1**: MCP adapter serves OAuth discovery metadata (`/.well-known/oauth-authorization-server`) pointing to Keycloak.
16. **Bearer token validation**: MCP adapter validates JWT tokens and extracts user identity/tenant context.
17. **Keycloak clients**: Per-MCP-client Keycloak configurations exist (VS Code, Claude Code) with appropriate redirect URIs.
18. **Generated tool catalog**: Platform-wide MCP tool catalog is generated from proto and published at `.well-known/mcp-tools.json`.
19. **Tool calling lifecycle** (§3.3): Clarifies LLM produces intent, client executes, adapter translates, domain validates. Updated diagram shows vendor structured output layer.
20. **Portable schema constraints** (§6.7): JSON schemas follow portable subset (≤2 nesting levels, avoid `oneOf`/`anyOf`). Tool examples MUST validate against schema.
21. **Tool scaling guidance** (§13.6): Recommends ≤20 tools per adapter, warns at 50, requires review at 100. Documents vendor tool selection limitations.

---

## 13) Tool catalog and discovery

### 13.1 Platform-wide tool registry

While each capability owns its MCP adapter, a platform-wide **tool catalog** provides discoverability:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Tool Catalog (Registry)                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Capability MCP Adapters self-register on startup:                  │
│                                                                     │
│  ┌──────────────────────┐  ┌──────────────────────┐                │
│  │ transformation-mcp   │  │ commerce-mcp         │                │
│  │ adapter              │  │ adapter              │                │
│  │                      │  │                      │                │
│  │ • listElements       │  │ • getOrder           │                │
│  │ • submitIntent       │  │ • submitOrderIntent  │                │
│  │ • searchCapabilities │  │ • listProducts       │                │
│  └──────────┬───────────┘  └──────────┬───────────┘                │
│             │                         │                             │
│             └────────────┬────────────┘                             │
│                          │                                          │
│                          ▼                                          │
│                 ┌────────────────────┐                              │
│                 │  Tool Registry     │                              │
│                 │  (discovery API)   │                              │
│                 └────────────────────┘                              │
└─────────────────────────────────────────────────────────────────────┘
```

### 13.2 Registry endpoints

The tool registry provides discovery without being a routing layer:

| Endpoint | Purpose |
|----------|---------|
| `GET /tools` | List all registered tools across capabilities |
| `GET /tools?capability=transformation` | Filter by capability |
| `GET /tools/{tool_name}` | Get tool schema and endpoint |
| `GET /capabilities` | List capabilities with MCP adapters |

**Example response:**
```json
{
  "tools": [
    {
      "name": "transformation.listElements",
      "capability": "transformation",
      "endpoint": "https://mcp.transformation.ameide.io",
      "description": "List ArchiMate elements with optional filtering",
      "hints": {
        "readOnlyHint": true,
        "destructiveHint": false
      },
      "inputSchema": { ... }
    },
    {
      "name": "commerce.getOrder",
      "capability": "commerce",
      "endpoint": "https://mcp.commerce.ameide.io",
      "description": "Get order by ID",
      "hints": {
        "readOnlyHint": true,
        "destructiveHint": false
      },
      "inputSchema": { ... }
    }
  ]
}
```

### 13.3 Self-registration pattern

MCP adapters register on startup and deregister on shutdown:

```go
func (a *MCPAdapter) Start(ctx context.Context) error {
    // Register tools with catalog
    for _, tool := range a.tools {
        if err := a.catalog.Register(ctx, ToolRegistration{
            Name:        tool.Name,
            Capability:  a.capability,
            Endpoint:    a.externalURL,
            Description: tool.Description,
            Hints:       tool.Hints,
            InputSchema: tool.InputSchema,
        }); err != nil {
            return err
        }
    }
    // ...
}

func (a *MCPAdapter) Shutdown(ctx context.Context) error {
    return a.catalog.Deregister(ctx, a.capability)
}
```

### 13.4 Benefits

1. **Discoverability** — Agents can find tools without knowing capability endpoints
2. **Decentralization** — Registry is read-only; tool invocation goes directly to capability MCP adapter
3. **Schema consistency** — Central place to validate tool naming conventions
4. **Monitoring** — Track which capabilities have MCP adapters deployed

### 13.5 Non-goals

The registry does NOT:
- Route tool calls (agents call capability adapters directly)
- Own tool schemas (proto remains authoritative)
- Aggregate responses across capabilities
- Enforce authorization (each adapter validates tokens)

### 13.6 Tool scaling and selection performance

**Tool catalog scaling limitations:**

As the number of exposed tools grows, tool selection performance degrades across vendors. Research and vendor documentation shows:

- **Anthropic** explicitly documents tool search and programmatic tool calling to access thousands of tools without consuming context window
- Large tool lists increase prompt tokens and degrade tool selection accuracy
- Most vendors recommend ≤ 50-100 tools per call for optimal performance

**Platform mitigation strategies:**

1. **Per-use-case tool filtering**
   - Clients should request only relevant tools for the current task
   - Use capability-scoped adapters (one per capability, not platform-wide)
   - Implement tool discovery with filtering (`GET /tools?capability=transformation`)

2. **Proto-first schema management**
   - Keep tool schemas minimal (see §6.7 for portable schema constraints)
   - Use shared message types to reduce total schema size
   - Document when schemas exceed recommended token budgets

3. **High-level "meta-tools" for complex operations**
   - Prefer single `createCapabilityStructure` over exposing 5-10 low-level element/relationship tools
   - Reduces tool count and simplifies tool selection for agents
   - See `backlog/536-mcp-write-optimizations.md` §8 for template patterns

4. **Dynamic tool subsetting (future)**
   - Consider programmatic tool calling (per Anthropic pattern) if tool count exceeds 100
   - Implement tool search/retrieval as a meta-capability
   - Use context-aware tool filtering based on conversation state

**Current platform stance:**

- **Recommended**: ≤ 20 tools per capability adapter
- **Warning threshold**: 50 tools (document why count is high)
- **Hard limit**: 100 tools per adapter (require architectural review)

If a capability legitimately needs >50 tools, consider:
- Splitting into multiple logical capabilities
- Implementing tool search/discovery as a first-class operation
- Using high-level template tools to reduce surface area

---

## 14) Generated tool catalog artifact

### 14.1 Single source of truth for all MCP tools

A **generated tool catalog** artifact consolidates all proto-annotated MCP tools into a single, machine-readable JSON file. This artifact is:

- **Generated at build time** — from proto definitions with `(ameide.mcp.expose)` annotations
- **Version-controlled** — committed alongside proto changes for diff visibility
- **Published** — deployed to a well-known URL for agent discovery

### 14.2 Catalog schema

```json
{
  "$schema": "https://ameide.io/schemas/mcp-tool-catalog.v1.json",
  "version": "2025-12-16T10:30:00Z",
  "git_sha": "abc123",
  "capabilities": [
    {
      "name": "transformation",
      "endpoint": "https://mcp.transformation.ameide.io",
      "proto_package": "ameide_core_proto.transformation.core.v1",
      "tools": [
        {
          "name": "transformation.listElements",
          "description": "List ArchiMate elements with optional filtering",
          "rpc": "ArchiMateQueryService.ListArchimateElements",
          "cqrs_side": "query",
          "hints": {
            "readOnlyHint": true,
            "destructiveHint": false,
            "idempotentHint": true
          },
          "inputSchema": {
            "type": "object",
            "properties": {
              "model_id": { "type": "string", "description": "Model ID" },
              "layer": { "type": "string", "enum": ["strategy", "business", "application", "technology"] },
              "read_context": {
                "type": "object",
                "properties": {
                  "baseline_id": { "type": "string" },
                  "as_of": { "type": "string", "format": "date-time" }
                }
              }
            },
            "required": ["model_id"]
          },
          "outputSchema": { ... }
        },
        {
          "name": "transformation.submitArchitectureIntent",
          "description": "Submit an architecture domain intent",
          "rpc": "TransformationWriteService.SubmitIntent",
          "cqrs_side": "command",
          "hints": {
            "readOnlyHint": false,
            "destructiveHint": true,
            "idempotentHint": false
          },
          "inputSchema": { ... }
        }
      ],
      "resources": [
        {
          "pattern": "transformation://element/{element_id}",
          "description": "ArchiMate element",
          "rpc": "ArchiMateQueryService.GetElement"
        }
      ]
    },
    {
      "name": "commerce",
      "endpoint": "https://mcp.commerce.ameide.io",
      "proto_package": "ameide_core_proto.commerce.core.v1",
      "tools": [ ... ]
    }
  ]
}
```

### 14.3 Generation pipeline

```bash
# Generate tool catalog from proto
buf generate --template buf.gen.yaml --include-imports

# buf.gen.yaml plugin configuration
plugins:
  - plugin: go-grpc
    out: gen/go
  - plugin: mcp-catalog
    out: gen/mcp
    opt:
      - expose_annotation=ameide.mcp.expose
      - output_format=json
      - output_file=mcp-tool-catalog.json
```

**Build artifact location:**
```
packages/
  ameide_core_proto/
    gen/
      mcp/
        mcp-tool-catalog.json    ← Generated catalog
```

### 14.4 Published endpoints

| Endpoint | Purpose |
|----------|---------|
| `https://ameide.io/.well-known/mcp-tools.json` | Latest released tool catalog |
| `https://ameide.io/.well-known/mcp-tools.v1.json` | Versioned catalog (stable) |
| `https://ameide.io/.well-known/mcp-tools.dev.json` | Development/preview catalog |

### 14.5 Agent usage

Agents can discover all available tools without querying individual adapters:

```typescript
// Fetch platform-wide tool catalog
const catalog = await fetch('https://ameide.io/.well-known/mcp-tools.json').then(r => r.json());

// Find tools by capability
const transformationTools = catalog.capabilities.find(c => c.name === 'transformation').tools;

// Find all query tools
const queryTools = catalog.capabilities
  .flatMap(c => c.tools)
  .filter(t => t.cqrs_side === 'query');

// Check if a tool exists
const hasSemanticSearch = catalog.capabilities
  .some(c => c.tools.some(t => t.name === 'transformation.semanticSearch'));
```

### 14.6 Version tracking

The catalog includes version metadata for cache invalidation and debugging:

| Field | Description |
|-------|-------------|
| `version` | ISO 8601 timestamp of generation |
| `git_sha` | Git commit hash of proto sources |
| `proto_version` | Proto package version |

CI pipelines SHOULD fail if the generated catalog differs from the committed version (enforces regeneration on proto changes).

### 14.7 Acceptance criteria for catalog

1. Catalog is generated from proto sources only (no manual editing).
2. Catalog includes all tools with `(ameide.mcp.expose)` annotation.
3. Catalog schema matches published JSON Schema.
4. Catalog is version-controlled and CI-validated.
5. Catalog is published to well-known URL on release.

---

## 15) Future considerations

- **MCP Prompts**: Expose capability-specific prompts as slash commands (e.g., `/mcp__transformation__capability_review`)
- **Streaming**: Support streaming responses for large result sets
- **Subscriptions**: Real-time updates via MCP resource subscriptions (maps to fact stream)
- **Federated discovery**: Cross-cluster tool registry for multi-region deployments
