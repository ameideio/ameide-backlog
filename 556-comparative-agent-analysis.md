# 556 — Comparative Agent Implementation Analysis

**Status:** Reviewed 2025-12-17 (revalidated vs current working tree; removed stale Transformation AgentDefinition claims; corrected LOC + backlog counts)
**Created:** 2025-12-16
**Purpose:** Comparative review of Agent primitive implementations across Sales, Commerce, Transformation, and SRE domains

## Layer header (Application: agent primitives)

- **Primary ArchiMate layer(s):** Application (Agent primitives are Application Components).
- **Primary element types used:** Application Component (Agent primitives), Application Services (consumed by agents), Application Events (EDA integration), Application Interfaces (MCP, A2A, gRPC).
- **Analysis scope:** Code implementation, proto contracts, agent architecture, tool systems, test coverage, documentation quality, multi-agent patterns.

> **Update (2026-01): 430v2 test contract**
>
> This comparative doc references legacy “integration harness” scripts (`run_integration_tests.sh`) as an implementation detail in some scaffolds. Treat `backlog/430-unified-test-infrastructure-v2-target.md` as the normative contract (native tooling; strict phases; no pack scripts as the canonical path).

---

## Executive Summary

This analysis reviews the implementation status of Agent primitives across four domains: **Sales** (sales-copilot), **Commerce** (commerce-assistant), **Transformation**, and **SRE**. The review covers code implementation, agent frameworks, proto/API contracts, test coverage, documentation quality, and integration patterns (MCP, A2A).

**Key Findings:**
- **Sales-Copilot**: Most complete agent (30% ready) - Python/LangGraph with comprehensive docs, functional skeleton, but no real agent logic
- **Commerce-Assistant**: Functional stub (20% ready) - Go/gRPC with domain-specific types, basic tests, minimal docs
- **Transformation**: Minimal stub (15% ready) - Go/gRPC with generic echo handler, no TLS, no docs, empty tests
- **SRE**: Minimal stub (15% ready) - Go/gRPC with generic echo handler, enforced TLS, minimal docs, basic test
- **Architecture Divergence**: Python/LangGraph for complex reasoning (Sales), Go/gRPC for fast orchestration (Commerce/Transform/SRE)

---

## Implementation Progress Ranking

### By Overall Completeness (Code + Proto + Tests + Docs + Architecture)

1. **Sales-Copilot: 30%** ⭐ Most Complete Architecture
   - ✅ Python/LangGraph framework with dev graph
   - ✅ FastAPI HTTP service wrapper
   - ✅ Comprehensive documentation (69-line README)
   - ✅ Unit tests (2 pytest tests)
   - ✅ Integration test infrastructure
   - ✅ Persistence-aware architecture
   - ✅ Comprehensive backlog doc (42 lines)
   - ❌ No proto service definition
   - ❌ No actual agent logic (stub only)
   - ❌ Empty tools directory

2. **Commerce-Assistant: 20%**
   - ✅ Go/gRPC handler with domain types
   - ✅ Proto service (CommerceAssistantAgentService)
   - ✅ Unit test (turn count tracking)
   - ✅ Thread-safe state management
   - ✅ Optional TLS/mTLS support
   - ✅ Domain-specific CommerceScope
   - ✅ Comprehensive backlog doc (65 lines)
   - ❌ Minimal README (4 lines)
   - ❌ No real commerce logic
   - ❌ No tool implementations

3. **Transformation: 15%**
   - ✅ Go/gRPC handler
   - ✅ Generic echo agent proto
   - ✅ Turn counting mechanism
   - ✅ Good backlog doc (87 lines)
   - ❌ No README (missing)
   - ❌ No TLS (insecure by default)
   - ❌ Empty smoke test
   - ❌ No real transformation logic

4. **SRE: 15%**
   - ✅ Go/gRPC handler
   - ✅ Generic echo agent proto
   - ✅ Enforced TLS (better than transformation)
   - ✅ Unit test (turn counting)
   - ✅ **Most comprehensive backlog doc** (521 lines)
   - ✅ Multi-agent pattern (SREAgent→SRECoder)
   - ✅ MCP adapter exists (sre-mcp-adapter)
   - ❌ Minimal README (4 lines)
   - ⚠️ No domain-specific proto service (uses generic EchoAgentService)
   - ❌ No real SRE logic

---

## Detailed Comparative Matrix

### 1. Code Implementation Status

| Domain | Primitive Location | Language | Framework | Total LOC | Key Features |
|--------|-------------------|----------|-----------|-----------|--------------|
| **Sales-Copilot** | `primitives/agent/sales-copilot/` | Python | LangGraph + FastAPI | 192 | - LangGraph DAG with dev graph<br>- FastAPI HTTP server<br>- Persistence hooks<br>- Prompt template system<br>- Empty tools registry |
| **Commerce-Assistant** | `primitives/agent/commerce-assistant/` | Go | gRPC | 182 | - Domain-specific proto (CommerceScope)<br>- Thread-safe turn counting<br>- TLS support<br>- Multi-domain scope awareness |
| **Transformation** | `primitives/agent/transformation/` | Go | gRPC | 110 | - Generic EchoAgentService<br>- Turn counting<br>- Generated service registration<br>- NO TLS (insecure) |
| **SRE** | `primitives/agent/sre/` | Go | gRPC | 143 | - Generic EchoAgentService<br>- Turn counting<br>- Enforced TLS<br>- Health checks |

### 2. Test Coverage Analysis

| Domain | Test Type | Test LOC | Test Quality | Assertions |
|--------|-----------|----------|--------------|------------|
| **Sales-Copilot** | Unit (pytest) | 18 | **Good** - Functional | - test_agent_returns_stub_output<br>- test_persistence_requires_thread_id<br>- Integration test shell script |
| **Commerce-Assistant** | Unit (Go) | 38 | **Fair** - Basic coverage | - TestInvokeMaintainsTurnCount<br>- Validates scope summary in output |
| **Transformation** | Smoke (Go) | 6 | **None** - Empty test | - Empty smoke test (no assertions) |
| **SRE** | Unit (Go) | 22 | **Fair** - Basic coverage | - TestInvoke_IncrementsTurn<br>- Validates turn counting |

### 3. Proto/API Contract Maturity

| Domain | Proto Service Defined | Proto Types | Request Fields | Response Fields | Streaming | API Maturity |
|--------|----------------------|-------------|----------------|-----------------|-----------|--------------|
| **Sales-Copilot** | ❌ None | Custom Python dataclass | N/A | N/A | No (HTTP) | **MISSING** - No proto service |
| **Commerce-Assistant** | ✅ CommerceAssistantAgentService | CommerceScope | thread_id, input, scope, hostname | thread_id, turn_count, output | No (unary) | **BETA** - Domain-specific types |
| **Transformation** | ✅ EchoAgentService | InvokeRequest/InvokeResponse | thread_id, input | thread_id, turn_count, output | No (unary) | **EARLY** - Generic echo pattern |
| **SRE** | ✅ EchoAgentService (generic) | Generic echo | thread_id, input | thread_id, turn_count, output | No (unary) | **EARLY** - Generic echo pattern |

**General Agent Framework Proto** (cross-domain):
- ✅ **AgentsService** (`agents/v1/agents.proto`) - Declarative catalog with versioning
- ✅ **AgentsRuntimeService** (`agents_runtime/v1/`) - Execution with streaming
- ✅ **InferenceService** (`inference/v1/`) - Alternative runtime (identical to AgentsRuntimeService)
- ✅ Chat/Threads integration with agent_id parameter

### 4. Documentation Quality

| Domain | README | Backlog Doc | Status Line | Implementation Checklist | Clarification Requests | Quality |
|--------|--------|-------------|-------------|-------------------------|----------------------|---------|
| **Sales-Copilot** | 69 lines | `540-sales-agent.md` (42 lines) | "Implemented (stub runtime)" | 6 items (1 complete) | 4 questions | **High** - Clear wiring guide |
| **Commerce-Assistant** | 4 lines | `523-commerce-agent.md` (65 lines) | "Scaffolded (read-only, tests)" | 3 items (2 complete) | 2 decisions | **Medium** - Good backlog, minimal README |
| **Transformation** | 0 lines | `527-transformation-agent.md` (87 lines) | "Draft (scaffold with tests)" | 3 items (2 complete) | 3 decisions | **Low** - No README at all |
| **SRE** | 4 lines | `526-sre-agent.md` (521 lines) | "Draft (production design)" | Multiple detailed | 5 decisions | **Excellent** - Most comprehensive |

### 5. Architecture & Patterns

#### Sales-Copilot (Python/LangGraph Pattern)

**Philosophy:** AI-native stateful reasoning with DAG-based orchestration

**Structure:**
```
primitives/agent/sales-copilot/
├── src/
│   ├── agent.py              # LangGraph AgentPrimitive class
│   ├── main.py               # FastAPI entrypoint (port 8080)
│   ├── dev_graph.py          # LangGraph DAG for visualization
│   ├── print_dag.py          # DAG rendering tool
│   ├── tools/                # Tool registry (empty)
│   └── prompts/
│       └── default.md        # Agent prompt with runtime rules
├── tests/
│   ├── test_agent.py         # 2 pytest tests
│   └── run_integration_tests.sh
├── langgraph.dev.json        # LangGraph dev configuration
├── pyproject.toml            # Python dependencies
├── Dockerfile.dev            # FastAPI + editable install
└── README.md                 # 69 lines - comprehensive
```

**Key Features:**
- **Framework:** LangGraph with async `run()` method
- **Runtime:** FastAPI HTTP server (not gRPC)
- **State:** Persistence-enabled flag with thread_id discipline
- **Output:** Returns stub: `"sales-copilot (stub): {prompt}"`
- **Dependencies:** langgraph, langchain-core, langchain-openai, langchain-anthropic, fastapi, uvicorn

**Completeness:** 30% - Proper structure, wrong (stub) implementation

#### Commerce-Assistant (Go/gRPC Pattern)

**Philosophy:** Fast, lightweight orchestration with domain awareness

**Structure:**
```
primitives/agent/commerce-assistant/
├── cmd/
│   └── main.go               # gRPC server bootstrap (63 lines)
├── internal/
│   ├── handlers/
│   │   ├── handlers.go        # Invoke implementation (55 lines)
│   │   └── handlers_test.go   # Unit tests (38 lines)
├── go.mod
├── Dockerfile                # Multi-stage build (distroless)
├── catalog-info.yaml
└── README.md                 # 4 lines (minimal)
```

**Key Features:**
- **Framework:** gRPC with generated proto
- **Proto Service:** `CommerceAssistantAgentService.Invoke()`
- **Domain Types:** CommerceScope (tenant_id, site_id, sales_channel_id, stock_location_id, store_site_id)
- **State:** In-memory turn count per thread_id (mutex-protected)
- **Output:** `"commerce-assistant(N): {input} (hostname={host} {scope_summary})"`
- **Security:** Optional TLS/mTLS with cert/key/CA loading

**Completeness:** 20% - Basic handler working, no domain integration

#### Transformation (Go/gRPC Minimal)

**Philosophy:** Generic agent pattern with minimal implementation

**Structure:**
```
primitives/agent/transformation/
├── cmd/
│   └── main.go               # gRPC server (44 lines)
├── internal/
│   ├── gen/
│   │   └── agent_services.generated.go  # Proto registration
│   ├── handlers/
│   │   └── handlers.go        # Invoke implementation (45 lines)
├── tests/
│   └── smoke_test.go         # Empty (6 lines)
└── README.md                 # MISSING (0 bytes)
```

**Key Features:**
- **Framework:** gRPC with generic `agent/v1` proto
- **Proto Service:** `EchoAgentService.Invoke()`
- **Output:** `"transformation(N): {input}"`
- **Security:** ⚠️ NO TLS (nosemgrep comment on insecure server)

**Completeness:** 15% - Bare minimum handler, insecure by default

#### SRE (Go/gRPC Minimal + Multi-Agent Design)

**Philosophy:** Generic handler with comprehensive architectural vision

**Structure:**
```
primitives/agent/sre/
├── cmd/
│   └── main.go               # gRPC server with TLS (63 lines)
├── internal/
│   ├── gen/
│   │   └── agent_services.generated.go  # Proto registration
│   ├── handlers/
│   │   ├── handlers.go        # Invoke (44 lines)
│   │   └── handlers_test.go   # Unit test (22 lines)
├── Dockerfile
└── README.md                 # 4 lines (minimal)
```

**Key Features:**
- **Framework:** gRPC with generic `agent/v1` proto
- **Proto Service:** `EchoAgentService.Invoke()` (identical to transformation)
- **Output:** `"sre(N): {input}"`
- **Security:** ✅ Enforced TLS (mandatory cert/key or GRPC_INSECURE flag)
- **Multi-Agent Pattern:** SREAgent (decision) → SRECoder (execution/A2A server)
- **MCP Integration:** `primitives/integration/sre-mcp-adapter` exists

**Completeness:** 15% - Same as transformation, but with proper TLS + architectural design

---

## Multi-Agent Patterns & Integration

### 1. Decision + Execution Separation Pattern (SRE)

**From:** `526-sre-agent.md` §3 + §5

```
┌─────────────┐                    ┌─────────────┐
│  SREAgent   │  ←──A2A──→         │  SRECoder   │
│  (Decision) │                    │ (Execution) │
└─────────────┘                    └─────────────┘
       ↓                                  ↓
 - Pattern search                   - kubectl/argocd
 - Risk assessment                  - Git commits
 - Proposal generation              - Cluster writes
 - NO cluster access                - Approval-gated
```

**Risk Tier Framework:**
- **Tier 1 (Low):** Read-only queries (fleet health, incidents, alerts)
- **Tier 2 (Medium):** SREAgent - observational writes (incident records, timeline notes)
- **Tier 3 (High):** SRECoder - infrastructure writes (kubectl, git push, argocd)

**A2A Contract Surface (REST binding):**
- `/.well-known/agent-card.json` — agent metadata
- `/v1/message:send` — sync task execution
- `/v1/message:stream` — async streaming tasks
- `/v1/tasks/*` — task lifecycle management

### 2. Role-Based Agent Pattern (Transformation)

**From:** `527-transformation-agent.md`

- Agents invoked by chat or processes
- One agent per role (SA/TA/PM/etc.)
- MCP-compatible tool surface

**Note:** This is currently a backlog design; the Transformation agent implementation is still the generic `EchoAgentService` and there is no Transformation-specific agent proto/types yet.

### 3. Copilot Pattern (Sales)

**From:** `540-sales-agent.md`

- Read-only projection-backed copilot
- Synthesizes assistance from `SalesQueryService`
- Proposes typed intents (not direct writes)
- Stub LangGraph implementation
- **Next Step:** projection-backed skills + intent proposal wiring

### 4. Diagnostics-Only Pattern (Commerce)

**From:** `523-commerce-agent.md`

- **Hard Rule:** Agent never mutates domain state
- Read-only setup/operations assistant
- Domain/DNS/cert/gateway diagnostics
- Guided steps (no state changes)
- Scaffolded with tests; tool design pending

---

## MCP (Model Context Protocol) Integration

### Current Status

**Documented in:** `534-mcp-protocol-adapter.md`

**Implemented MCP Adapters:**
- ✅ `primitives/integration/sre-mcp-adapter` (stdio + HTTP, hand-authored tools)
- ✅ `primitives/integration/transformation-mcp-adapter` (HTTP mux, shape-only)
- ✅ Shared runtime: `packages/ameide_mcp`

**ADR Decisions:**
| ADR | Decision | Impact |
|-----|----------|--------|
| ADR-01 | MCP adapter is Integration primitive, not gateway | One adapter per capability |
| ADR-02 | Proto-first tool schemas (generated, not hand-authored) | Avoids schema duplication |
| ADR-03 | OAuth 2.1 + Protected Resource Metadata | External auth standard |
| ADR-04 | CQRS routing: commands→Domain, queries→Projection | Clear responsibility split |
| ADR-07 | **Internal agents use SDK/gRPC; MCP for external only** | Preserves type safety in-cluster |

**Proto Integration Status:**
- ❌ No MCP protocol definitions in core protos
- ❌ No `(ameide.mcp.expose)` annotation system yet
- ✅ Tool execution events in StreamEvent (agents_runtime_types.proto)
- ✅ AgentToolGrant system supports extensibility

**Outstanding Gaps:**
- [ ] Proto-derived tool schemas (ADR-02 not implemented)
- [ ] Canonical tool naming scheme (proto-driven vs adapter-driven)
- [ ] MCP identity → Ameide tenant/user mapping

---

## Proto Framework Architecture

### General Agent Framework

**Production-Ready Proto Services:**

1. **AgentsService** (`agents/v1/agents.proto`)
   - Declarative agent catalog with versioning
   - NodeDefinition with version history, properties, UI schema
   - AgentInstance with routing rules, compiled artifacts
   - Catalog discovery: `ListNodes()`, `StreamCatalog()`
   - Lifecycle: `CreateAgent()`, `UpdateAgent()`, `PublishAgent()`

2. **AgentsRuntimeService** (`agents_runtime/v1/agents_runtime_service.proto`)
   - Execution contract: `Invoke()`, `Stream()`
   - InvocationContext with tenant, UI context (page, role, graph)
   - StreamEvent types: Status, TokenDelta, ToolCallStarted/Completed, Usage, Error
   - Resume capability with resume_token
   - Cost tracking with cost_usd

3. **InferenceService** (`inference/v1/inference_service.proto`)
   - Identical to AgentsRuntimeService
   - Alternative runtime (possibly for non-LangGraph agents)

4. **Chat/Threads Integration**
   - SendMessageRequest with `agent_id` parameter
   - Streaming response pattern (started, token, completed, error)

### Domain-Specific Proto Status

| Domain | Proto Service | Message Types | Maturity |
|--------|--------------|---------------|----------|
| **Sales** | ❌ None | N/A | **MISSING** |
| **Commerce** | ✅ CommerceAssistantAgentService | CommerceScope (tenant/site/channel context) | **BETA** |
| **Transformation** | ✅ EchoAgentService (generic) | InvokeRequest/InvokeResponse | **EARLY** |
| **SRE** | ✅ EchoAgentService (generic) | Generic echo | **EARLY** |

**Critical Gap:** Sales and SRE have agent implementations but no dedicated proto service definitions

---

## Key Gaps & Blockers

### Sales-Copilot
1. **No Proto Service:** Missing `SalesCopilotAgentService` definition
2. **No Real Agent Logic:** LangGraph nodes not implemented (only echo node)
3. **No Tool Implementations:** Empty `src/tools/` directory
4. **No Domain Integration:** Not connected to SalesQueryService/SalesCommandService
5. **Persistence Not Connected:** Backend not implemented
6. **Clarifications Needed:**
   - [ ] v0 skill set: read-only vs proposal-based?
   - [ ] Read transport: gRPC direct vs gateway?
   - [ ] Safety posture: direct commands vs proposal-only?
   - [ ] Thread state storage: Threads service vs local?

### Commerce-Assistant
1. **No Real Commerce Logic:** Echoes input, doesn't do commerce operations
2. **No Tool Implementations:** No cart/order/payment tools
3. **No Integration Tests:** Only unit test for turn counting
4. **Minimal Documentation:** 4-line README
5. **Tool Design Pending:** Read-only diagnostics not specified
6. **Clarifications Needed:**
   - [ ] Whether `commerce-mcp-adapter` ships in v1
   - [ ] Which read-only tools expose first

### Transformation
1. **No TLS:** Insecure by default (nosemgrep comment)
2. **No Real Logic:** Generic echo handler only
3. **No Tests:** Empty smoke test
4. **No README:** Missing documentation
5. **No Proto Service:** Uses generic EchoAgentService
6. **Role Definitions Pending:** SA/TA/PM taxonomy not specified
7. **Clarifications Needed:**
   - [ ] Initial role taxonomy and tool grants
   - [ ] What "small persisted agent state" means
   - [ ] Whether agents can initiate process workflows

### SRE
1. **No Proto Service:** Uses generic EchoAgentService (borrowed from transformation)
2. **No Real Logic:** Generic echo handler only
3. **Minimal Documentation:** 4-line README (backlog is excellent though)
4. **SRECoder Not Implemented:** Multi-agent pattern designed but not built
5. **Tool Grants Not Implemented:** Risk tier system designed but not coded
6. **Clarifications Needed:**
   - [ ] Canonical tool naming scheme
   - [ ] Tier 2 writes allowed without approval?
   - [ ] Evidence bundle structure
   - [ ] SRECoder: separate agent vs shared "coder" agent?

---

## Framework & Technology Stack Analysis

### Python vs Go Decision

**Python (LangGraph) - Sales-Copilot:**
- **Use Case:** Complex stateful reasoning, AI-native workflows
- **Framework:** LangGraph (DAG-based state management)
- **Protocol:** HTTP/FastAPI (not gRPC)
- **Strengths:** Rich AI ecosystem, stateful checkpointing, visual DAG tools
- **Weaknesses:** Slower execution, higher memory

**Go (gRPC) - Commerce/Transformation/SRE:**
- **Use Case:** Fast, lightweight orchestration
- **Framework:** gRPC with proto-generated types
- **Protocol:** gRPC (efficient binary)
- **Strengths:** Performance, low resource usage, type safety
- **Weaknesses:** Less AI tooling, manual state management

### Dependency Analysis

**Sales-Copilot Python Dependencies:**
```toml
langgraph = "*"
langchain-core = "*"
langchain-openai = "*"
langchain-anthropic = "*"
fastapi = "*"
uvicorn = "*"
pydantic = "*"
```

**Go Dependencies (all three):**
- `google.golang.org/grpc`
- `google.golang.org/protobuf`
- Generated proto packages
- TLS crypto libraries (commerce, sre)

---

## Recommendations

### Immediate Actions

1. **Define Missing Proto Services:**
   - Create `sales/agent/v1/sales_copilot_agent.proto`
   - Create `sre/agent/v1/sre_agent.proto`
   - Move from generic EchoAgentService to domain-specific services

2. **Fix Transformation Security:**
   - Remove nosemgrep comment
   - Enforce TLS like SRE
   - Add proper TLS configuration

3. **Standardize Documentation:**
   - Add README to transformation agent (currently missing)
   - Expand commerce and sre README to Sales level (69 lines)
   - Follow Sales wiring checklist pattern

4. **Implement Proto-First Tool Schemas (ADR-02):**
   - Define `(ameide.mcp.expose)` annotation
   - Generate tool catalogs from proto
   - Block hand-authored tool definitions

### Phase-Based Implementation

**Phase 1: Foundation (Proto + Security)**
- [ ] Sales: Define `SalesCopilotAgentService` proto
- [ ] SRE: Define `SREAgentService` proto with SREAgent/SRECoder split
- [ ] Transformation: Enforce TLS, add README
- [ ] All: Standardize thread_id discipline

**Phase 2: Agent Logic Implementation**
- [ ] Sales: Implement LangGraph nodes (replace echo)
- [ ] Sales: Connect to SalesQueryService/SalesCommandService
- [ ] Commerce: Implement commerce diagnostics tools
- [ ] Transformation: Implement role-based agent logic
- [ ] SRE: Implement pattern-first triage workflow (525)

**Phase 3: Tool Systems**
- [ ] Implement proto-first tool schemas (`(ameide.mcp.expose)`)
- [ ] Sales: Populate `src/tools/` directory
- [ ] Commerce: Define read-only diagnostic tools
- [ ] SRE: Implement risk-tiered tool grants

**Phase 4: Multi-Agent Patterns**
- [ ] SRE: Implement SREAgent→SRECoder A2A integration
- [ ] Implement AmeidePO/AmeideSA/AmeideCoder (505-v2)
- [ ] Test A2A streaming contracts
- [ ] Document multi-agent orchestration playbook

**Phase 5: Production Readiness**
- [ ] Integration tests for all agents
- [ ] Golden prompt → deterministic output tests
- [ ] GitOps deployment for all agent primitives
- [ ] Monitoring and observability integration

---

## Summary Table: Implementation Progress

| Metric | Sales-Copilot | Commerce | Transformation | SRE |
|--------|--------------|----------|----------------|-----|
| **Code Exists** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| **Proto Service** | ❌ None | ✅ CommerceAssistantAgentService | ⚠️ Generic EchoAgentService | ⚠️ Generic EchoAgentService |
| **Tests Quality** | ✅ Functional | ⚠️ Basic | ❌ Empty | ⚠️ Basic |
| **Framework** | ✅ LangGraph + FastAPI | ✅ gRPC | ✅ gRPC | ✅ gRPC |
| **TLS/Security** | N/A (HTTP) | ✅ Optional TLS/mTLS | ❌ Insecure | ✅ Enforced TLS |
| **Documentation** | ✅ Excellent (69 lines) | ⚠️ Minimal (4 lines) | ❌ None | ⚠️ Minimal (4 lines) |
| **Backlog Docs** | ✅ Good (42 lines) | ✅ Good (65 lines) | ✅ Good (87 lines) | ✅ **Excellent** (521 lines) |
| **MCP Adapter** | ❌ No | ❌ No | ✅ Yes (shape-only) | ✅ Yes (hand-authored) |
| **Multi-Agent** | ❌ No | ❌ No | ❌ No | ✅ Designed (not implemented) |
| **Real Logic** | ❌ Stub | ❌ Echo | ❌ Echo | ❌ Echo |
| **Overall Progress** | **30%** | **20%** | **15%** | **15%** |

---

## Critical Files Reference

### Agent Implementations
- Sales: [`primitives/agent/sales-copilot/`](../primitives/agent/sales-copilot/)
  - [`src/agent.py`](../primitives/agent/sales-copilot/src/agent.py) - LangGraph AgentPrimitive
  - [`src/main.py`](../primitives/agent/sales-copilot/src/main.py) - FastAPI entrypoint
  - [`tests/test_agent.py`](../primitives/agent/sales-copilot/tests/test_agent.py) - Unit tests
- Commerce: [`primitives/agent/commerce-assistant/`](../primitives/agent/commerce-assistant/)
  - [`internal/handlers/handlers.go`](../primitives/agent/commerce-assistant/internal/handlers/handlers.go) - Invoke handler
- Transformation: [`primitives/agent/transformation/`](../primitives/agent/transformation/)
  - [`internal/handlers/handlers.go`](../primitives/agent/transformation/internal/handlers/handlers.go) - Echo handler
- SRE: [`primitives/agent/sre/`](../primitives/agent/sre/)
  - [`internal/handlers/handlers.go`](../primitives/agent/sre/internal/handlers/handlers.go) - Echo handler

### Proto Definitions
- General Framework:
  - [`packages/ameide_core_proto/src/ameide_core_proto/agents/v1/agents.proto`](../packages/ameide_core_proto/src/ameide_core_proto/agents/v1/agents.proto) - Agent catalog
  - [`packages/ameide_core_proto/src/ameide_core_proto/agents_runtime/v1/agents_runtime_service.proto`](../packages/ameide_core_proto/src/ameide_core_proto/agents_runtime/v1/agents_runtime_service.proto) - Runtime execution
  - [`packages/ameide_core_proto/src/ameide_core_proto/agents_runtime/v1/agents_runtime_types.proto`](../packages/ameide_core_proto/src/ameide_core_proto/agents_runtime/v1/agents_runtime_types.proto) - Message types
- Commerce: [`packages/ameide_core_proto/src/ameide_core_proto/commerce/agent/v1/commerce_assistant_agent.proto`](../packages/ameide_core_proto/src/ameide_core_proto/commerce/agent/v1/commerce_assistant_agent.proto)
- Generic: [`packages/ameide_core_proto/src/ameide_core_proto/agent/v1/echo_agent.proto`](../packages/ameide_core_proto/src/ameide_core_proto/agent/v1/echo_agent.proto)

### Backlog Documentation
- Sales: [`backlog/540-sales-agent.md`](540-sales-agent.md)
- Commerce: [`backlog/523-commerce-agent.md`](523-commerce-agent.md)
- SRE: [`backlog/526-sre-agent.md`](526-sre-agent.md) **← Reference (521 lines)**
- Transformation: [`backlog/527-transformation-agent.md`](527-transformation-agent.md)

### Integration Patterns
- MCP: [`backlog/534-mcp-protocol-adapter.md`](534-mcp-protocol-adapter.md)
- A2A: [`backlog/505-agent-developer-v2.md`](505-agent-developer-v2.md)
- Agent Developer: [`backlog/505-agent-developer-v2-implementation.md`](505-agent-developer-v2-implementation.md)

### MCP Adapters
- SRE: [`primitives/integration/sre-mcp-adapter/`](../primitives/integration/sre-mcp-adapter/)
- Transformation: [`primitives/integration/transformation-mcp-adapter/`](../primitives/integration/transformation-mcp-adapter/)
- Shared Runtime: [`packages/ameide_mcp/`](../packages/ameide_mcp/)

---

## Related Backlogs

- `520-primitives-stack-v6.md` - Primitive stack architecture
- `550-comparative-domain.md` - Domain primitive analysis
- `551-comparative-integration.md` - Integration primitive analysis
- `553-comparative-uisurface-analysis.md` - UISurface primitive analysis
- `496-eda-principles-v6.md` - Integration contract rules for agents
- `500-agent-operator.md` - Agent operator and runtime roles
- `477-primitive-stack.md` - Original primitive stack definition

---

*This comparative analysis provides a comprehensive cross-domain view of Agent implementation maturity, highlighting the architectural divergence (Python/LangGraph vs Go/gRPC), proto contract gaps (Sales/SRE missing), and the path to production-ready agents across all four domains.*
