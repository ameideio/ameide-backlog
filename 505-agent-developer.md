
> **Contract alignment**
> - **Canonical contract:** Scrum data and events follow `transformation_scrum_*` protos (`ameide_core_proto.transformation.scrum.v1`) and the event seams in `506-scrum-vertical-v2.md` / `508-scrum-protos.md`; this file’s Requirement-centric event names are legacy only.  
> - **Canonical integration:** Process primitives interact with Transformation only via intents/facts on the bus; any direct RPC coupling or agent↔domain shortcuts described here are historical and superseded by 505-v2/506-v2/367-1.  
> - **Non-Scrum extensions:** any acceptance workflows, impediment models, or board columns in this file must be treated as optional extensions or migration notes, not canonical Scrum artifacts.


## agent-developer – Agent Developer Execution Environment (Two-Agent Architecture with A2A)

**Authority & supersession**

- The **canonical architecture** for Agent Developer now lives in `505-agent-developer-v2.md` (Process + AmeidePO + AmeideSA + AmeideCoder). That file, together with `506-scrum-vertical-v2.md`, `508-scrum-protos.md`, and `300-400/367-1-scrum-transformation.md`, is authoritative for agent roles, A2A contracts, and the domain↔process event model.  
- This file is **historical and implementation-tracking only**: it preserves the earlier two-agent (PO + Coder) A2A design and current implementation status, to support migration and evidence. It must not be treated as normative when it conflicts with 505‑v2 or 506.  
- Any event names, topic layouts, or domain/process coupling described here are superseded by the `scrum.domain.intents.v1` / `scrum.domain.facts.v1` / `scrum.process.facts.v1` contracts in `506-scrum-vertical-v2.md` / `508-scrum-protos.md`; any places where AmeidePO or AmeideCoder appear to cross those boundaries should be read as legacy behavior to be removed during migration.

> **Target architecture lives in [505-agent-developer-v2.md](505-agent-developer-v2.md).** This file now tracks implementation progress, migration notes, and historical two-agent context while we land the Process + AmeidePO + AmeideSA + AmeideCoder model.

**Status:** Active (guardrail plumbing landed; A2A integration in progress)
**Owner:** Platform / Agents
**Depends on:**

* 471/472/473/475/476 (Vision, App, Tech, Domains, Security)
* 496 (EDA Principles) - Domain↔Agent contracts
* Agent primitive / AgentDefinitions (Transformation)
* ArgoCD / GitOps / ApplicationSet patterns
* Historical context: [000-200/030-coding-agent.md](000-200/030-coding-agent.md) and [000-200/041-coder.md](000-200/041-coder.md) capture the earlier PoC implementations and are now marked as superseded in favour of this backlog.

## Grounding & cross-references

- **Status & purpose:** Historical record of the original two-agent (PO + Coder) A2A model and Requirement-centric contracts; retained to explain existing assets and migration steps, but superseded by `505-agent-developer-v2.md`, `506-scrum-vertical-v2.md`, and `508-scrum-protos.md` for any new design or implementation.  
- **Architecture grounding:** Built on the same primitive and vision backlogs as v2 (`470-ameide-vision.md`, `471-ameide-business-architecture.md`, `472-ameide-information-application.md`, `473-ameide-technology.md`, `475-ameide-domains.md`, `477-primitive-stack.md`, and `496-eda-principles.md`), but before the strict Scrum-aligned contract was introduced.  
- **Legacy consumers:** Older operator/CLI demos and A2A experiments that still reference Requirement events or direct domain RPCs should treat this file as migration guidance only and target the v2 contracts when updated. `504-agent-vertical-slice.md` and `500-agent-operator.md` now follow the v2 architecture.  
- **Navigation:** `507-scrum-agent-map.md` and `505-agent-developer-v2-implementation.md` show how the new multi-agent stack replaces this design while preserving evidence from its implementation history.

---

### 0. Status snapshot (repo evidence)

| Workstream | State | Notes & repo pointers |
|------------|-------|-----------------------|
| **Agent primitive plumbing** | ✅ | Agent CRD/operator Helm/GitOps bundles were refreshed (see `operators/helm/crds/ameide.io_agents.yaml`, `gitops/ameide-gitops/sources/charts/platform/ameide-operators/crds/ameide.io_agents.yaml`). Demo assets live under `gitops/.../primitives/agent/core-platform-coder.yaml` + `scripts/demo-agent-slice.sh`. |
| **CLI guardrails (AmeideCoder tool)** | ✅ | `packages/ameide_core_cli/internal/commands/primitive.go` ships guardrail commands (describe, drift, plan, verify, etc.). These are tools for AmeideCoder, NOT workflow for AmeidePO. |
| **AmeidePO LangGraph DAG** | ⚠️ Partial | `primitives/agent/ameide-coder/` needs refactoring to become AmeidePO. Current graph still has coding tools - must be removed. Product management DAG nodes (analyze, delegate, review, loop) TODO. |
| **A2A Protocol** | ❌ Not started | A2A server (AmeideCoder) and client (AmeidePO) need implementation. AgentCard discovery, JSON-RPC endpoint, SSE streaming all TODO. |
| **AmeideCoder (Devcontainer)** | ⚠️ Partial | `services/devcontainer_service` provides base runtime (`POST /v1/develop`). Needs A2A server endpoint and full tool palette (CLI is one tool among many). **Note:** GitOps deployment of the devcontainer service is still a placeholder/optional component in `ameide-gitops` (not enabled by default). |
| **EDA Integration** | ⚠️ Partial | EDA infrastructure exists (backlog 496). Event schemas for agent coordination (RequirementCreated, RequirementCompleted) need definition. |
| **Workflow/Temporal metadata** | ⚠️ Deferred | Historical Stage-2 automation (`367-2-agent-orchestration-coding-agent.md`) remains as governance blueprint. Not part of A2A architecture. |

> **V2 alignment note:** Each workstream above remains active only insofar as it unlocks the Process + PO + SA + Coder flow defined in [505-agent-developer-v2.md](505-agent-developer-v2.md). Treat the two-agent descriptions below as implementation breadcrumbs while we update every runtime, CRD, and CLI reference.

> **Architecture shift:** Canonical target architecture is the **Process + AmeidePO + AmeideSA + AmeideCoder** model in [505-agent-developer-v2.md](505-agent-developer-v2.md). The remainder of this document tracks migration progress and retains the earlier **two-agent A2A model** (AmeidePO + AmeideCoder) for compatibility and evidence until every implementation artifact is updated.

---

### 1. Purpose

Provide a **two-agent architecture** for autonomous code development inside Ameide:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Transformation │    │    AmeidePO     │    │   AmeideCoder   │
│     (Domain)    │    │ (Agent Primitive│    │     (Agent)     │
│                 │    │   LangGraph)    │    │   Devcontainer  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
        │                      │                      │
        │ EDA (496)            │ A2A Protocol         │ Tools
        │ Events               │ Continuous Loop      │ (CLI, Claude, Git...)
        ▼                      ▼                      ▼
```

**Three Components:**

| Component | Type | Runtime | Responsibilities |
|-----------|------|---------|------------------|
| **Transformation** | Domain | gRPC service | Backlog, requirements, events |
| **AmeidePO** | Agent primitive | LangGraph DAG | Product management, orchestration |
| **AmeideCoder** | Agent | Devcontainer | Code execution, generic coding |

**Two Contract Types:**

| Contract | Protocol | Pattern |
|----------|----------|---------|
| Domain ↔ Agents | EDA (496) | Async events, decoupled |
| Agent ↔ Agent | A2A | Continuous loop, not one-off |

**Key Principles:**

* **AmeidePO** (Product Owner) handles **product management only** - no coding tools
* **AmeideCoder** is a **generic coding agent** with many tools (Ameide CLI is one of them)
* Development is a **continuous A2A loop** between PO and Coder until PO is satisfied
* Domain communication follows **EDA principles** (backlog 496)

Goal: make "an agent that manages product development and delegates coding to a specialist coder agent" a **repeatable, observable, safe pattern**.

---

### 2. Scope

**In scope**

* Extend **AgentDefinition** / agent primitive to support **LangGraph‑based agents**:

  * `runtime_type: "langgraph"` (vs e.g. “simple_llm”, “tool_only”).
  * `dag_ref` (Python module + entrypoint or equivalent).
  * `tool_bindings` (which tools this agent can call, including `develop_in_container`).
* Define & implement an **Agent primitive runtime** for “coder agents”:

  * Container image: **langgraph/agent-runtime** (or equivalent custom image).
  * ArgoCD‑managed Deployment/Service.
* Stand up a **devcontainer service**:

  * Runs `.devcontainer` (`ameideio/ameide`) under DevPod/whatever you pick.
  * Exposes `Claude SDK / CLI` as **gRPC**: “develop code in this repo/branch”.
* Implement a **`develop_in_container` tool**:

  * From the LangGraph DAG’s point of view, it’s just a tool.
  * Under the hood, it’s a gRPC client to the devcontainer service, which in turn shells out to Claude Code / codex CLI.
* Optional for v1: a **CRD** to declaratively define “coding agents” and connect them to repos.

**Out of scope (for this item)**

* Scalability/auto-provisioning of devcontainers (assume 1 persistent pool for now).
* Multi-tenant “self-serve AI dev env” UI for customers.
* Full ephemeral “one container per task” scheduling (can be a follow-up).

#### 2.1 Alignment with 504/484 guardrails

* **Vertical slice contract.** The coder runtime extends the `core-platform-coder` scenario delivered in [504-agent-vertical-slice.md](504-agent-vertical-slice.md §2-5); every LangGraph DAG invocation must run the same OBSERVE→REASON→ACT→VERIFY loop by invoking the CLI workflows from [484a-ameide-cli-primitive-workflows.md](484a-ameide-cli-primitive-workflows.md) (`describe`, `drift`, `plan`, `impact`, `verify`, `prompt`). Proto-driven wiring/codegen is executed via `buf generate` (the CLI may wrap it, but must not become a parallel generator system).
* **Sample assets + GitOps.** When we add the devcontainer service and LangGraph runtime, update the sample manifests referenced in 504 (e.g. `operators/helm/examples/agent-sample.yaml`, `gitops/.../primitives/agent/core-platform-coder.yaml`, prompt profiles under `prompts/agent/default.md`) so the guardrail demo script keeps working with the new `runtime_type=langgraph` AgentDefinition and the `develop_in_container` tool binding.
* **Dual-track reporting.** Continue the dual-track format from [502-domain-vertical-slice.md](502-domain-vertical-slice.md) by documenting operator changes (new CRD fields, ApplicationSet wiring for the devcontainer service) and CLI changes (new prompt profile hints, verify/codegen gate updates) in the same structure once the LangGraph DAG is implemented, so downstream teams can trace status exactly like they do for backlog 504.
* **Primitive-first bootstraps.** Agent repo skeletons live under `primitives/{kind}/{name}` (e.g. `primitives/agent/core-platform-coder`). Any bootstrap scaffolding should be Backstage-template-driven, and any proto-derived wiring/tests should come from `buf generate` outputs under generated roots; keep the CLI as an orchestrator/observer so Backstage metadata can lag without blocking the primitive workflow.

---

### 3. High‑Level Architecture

#### 3.1 Two-Agent Model

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                              ArgoCD / GitOps                                │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        ▼                             ▼                             ▼
┌───────────────────┐    ┌───────────────────────┐    ┌───────────────────────┐
│  TRANSFORMATION   │    │       AmeidePO        │    │     AmeideCoder       │
│     (Domain)      │    │   (LangGraph DAG)     │    │    (Devcontainer)     │
│  ─────────────────│    │  ─────────────────────│    │  ─────────────────────│
│                   │    │                       │    │                       │
│  • Backlog items  │    │  • Product management │    │  • Generic coder      │
│  • Requirements   │    │  • Reads requirements │    │  • Many tools:        │
│  • AgentDefs      │    │  • Prioritizes work   │    │    - Ameide CLI       │
│                   │    │  • Delegates to Coder │    │    - Claude/Codex     │
│                   │    │  • Reviews results    │    │    - Git CLI          │
│  EDA Events ◀────▶│    │  • Loops until done   │    │    - Build/Test       │
│  (backlog 496)    │    │  • Emits domain events│    │    - Linters          │
│                   │    │                       │    │    - Any other...     │
└───────────────────┘    └───────────────────────┘    └───────────────────────┘
                                    │                          │
                                    │◀─────── A2A ────────────▶│
                                    │   Continuous Loop        │
                                    │   (not one-off)          │
```

#### 3.2 A2A Development Loop

```text
     AmeidePO                              AmeideCoder
    (LangGraph)                           (Devcontainer)
         │                                      │
    ┌────┴────┐                                 │
    │ PLAN    │──── A2A: plan task ────────────▶│
    │ phase   │◀─── A2A: plan result ───────────│
    └────┬────┘                                 │
         │ (PO reviews)                         │
    ┌────┴────┐                                 │
    │IMPLEMENT│──── A2A: implement ────────────▶│
    │ phase   │◀─── A2A: code done ─────────────│
    └────┬────┘                                 │
         │ (PO verifies)                        │
    ┌────┴────┐                                 │
    │ VERIFY  │──── A2A: fix issues ───────────▶│  ◀─── LOOP
    │ phase   │◀─── A2A: fixed ─────────────────│       until
    └────┬────┘                                 │       satisfied
         │                                      │
         ▼                                      │
  EDA Event: RequirementCompleted              │
  (PR URL, summary) → Transformation           │
```

#### 3.3 Agent Responsibilities

| AmeidePO (LangGraph DAG) | AmeideCoder (Devcontainer) |
|--------------------------|---------------------------|
| Product management | Code execution |
| Reads requirements | Generic coding agent |
| Prioritizes work | Has MANY tools available |
| Delegates via A2A | Ameide CLI = one tool |
| Reviews results | Claude/Codex = one tool |
| Loops until satisfied | Git, build, lint = tools |
| Emits domain events | Agent decides how to use |
| **DAG = orchestration** | **Agent = autonomous executor** |
| **NO coding tools** | **ALL coding tools** |

Key points:

* **AmeidePO** is a product management DAG - no coding activities
* **AmeideCoder** is a generic autonomous coder - uses tools as needed
* **Ameide CLI** is a tool for the coder, not a workflow for the DAG
* **A2A loop** continues until PO is satisfied (not single handoff)
* **EDA events** connect to domain (backlog 496)

---

### 4. Design Details

#### 4.1 Two-Agent Architecture

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TWO AGENT DEFINITIONS                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   AmeidePO (Product Owner)              AmeideCoder (Coder)                 │
│   ────────────────────────              ────────────────────                │
│   • runtime_type: langgraph             • runtime_type: devcontainer        │
│   • scope: PRODUCT_MANAGEMENT           • scope: CODE_EXECUTION             │
│   • risk_tier: MEDIUM                   • risk_tier: HIGH                   │
│   • tools: NONE (product mgmt only)     • tools: ALL coding tools           │
│   • a2a_client: true                    • a2a_server: true                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**AmeidePO AgentDefinition:**

```yaml
AgentDefinition:
  id: "ameide-product-owner"
  runtime_type: "langgraph"
  dag_ref:
    # GitOps-managed environments deploy digest-pinned refs; see backlog/602-image-pull-policy.md.
    image: "ghcr.io/ameideio/agent-runtime-langgraph@sha256:<digest>"
    module: "ameide.agents.product_owner.graph"
    entrypoint: "build_graph"
  tools: []  # NO coding tools - product management only
  a2a_client:
    target_agent: "ameide-coder"
  scope: "PRODUCT_MANAGEMENT"
  risk_tier: "MEDIUM"
```

**AmeideCoder AgentDefinition:**

```yaml
AgentDefinition:
  id: "ameide-coder"
  runtime_type: "devcontainer"
  # GitOps-managed environments deploy digest-pinned refs; see backlog/602-image-pull-policy.md.
  image: "ghcr.io/ameideio/devcontainer-service@sha256:<digest>"
  tools:  # Generic coder with MANY tools
    - id: "ameide_cli"
    - id: "claude_code"
    - id: "git_cli"
    - id: "build_test"
    - id: "linters"
    # ... any other tools the coder needs
  a2a_server:
    endpoint: "/.well-known/agent-card.json"
  scope: "CODE_EXECUTION"
  risk_tier: "HIGH"
```

**Implementation status:** The AgentDefinition schema supports `runtime_type`, `dag_ref`, `scope`, `risk_tier`, and tool grants. A2A client/server fields are new and need schema extension.

#### 4.2 A2A Protocol Integration

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                            A2A PROTOCOL                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Transport: JSON-RPC 2.0 over HTTP(S)                                       │
│  Discovery: AgentCard at /.well-known/agent-card.json                       │
│  Methods:  message/send, tasks/get, tasks/sendSubscribe (SSE streaming)     │
│  Headers:  A2A-Version: 1.0                                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**AmeideCoder AgentCard (/.well-known/agent-card.json):**

```json
{
  "name": "ameide-coder",
  "description": "Generic coding agent with Ameide CLI guardrails + devcontainer",
  "version": "1.0.0",
  "url": "https://coder.ameide.internal/a2a",
  "protocolVersion": "1.0",
  "capabilities": { "streaming": true, "pushNotifications": false },
  "defaultInputModes": ["text/plain"],
  "defaultOutputModes": ["text/plain", "application/json"],
  "skills": [
    {
      "id": "implementRequirement",
      "name": "Implement requirement",
      "description": "Implement/modify code using available tools autonomously",
      "inputSchema": {
        "type": "object",
        "properties": {
          "requirementId": { "type": "string" },
          "repoUrl": { "type": "string" },
          "instruction": { "type": "string" }
        },
        "required": ["requirementId", "repoUrl", "instruction"]
      }
    }
  ],
  "auth": { "type": "oauth2", "tokenUrl": "...", "scopes": ["ameide.coder.run"] }
}
```

**A2A Task Request (PO → Coder):**

```json
{
  "jsonrpc": "2.0",
  "id": "req-123",
  "method": "message/send",
  "params": {
    "task": {
      "id": "task-uuid",
      "status": { "state": "submitted" },
      "metadata": {
        "skill": "implementRequirement",
        "requirementId": "REQ-123",
        "repoUrl": "git@github.com:ameideio/ameide.git"
      },
      "history": [
        { "role": "user", "parts": [{ "kind": "text", "text": "Implement..." }] }
      ]
    }
  }
}
```

**A2A Task Response (Coder → PO):**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "task": {
      "id": "task-uuid",
      "status": { "state": "completed" },
      "artifacts": [
        { "id": "pr", "kind": "link", "parts": [{ "text": "https://github.com/.../pull/123" }] },
        { "id": "summary", "kind": "text", "parts": [{ "text": "Implemented validation..." }] }
      ]
    }
  }
}
```

**Implementation status:** A2A protocol support is new and needs implementation in both agents.

#### 4.3 AmeidePO (LangGraph DAG)

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                     AmeidePO LangGraph DAG                                  │
│                   (Product Management ONLY)                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐      │
│  │  START  │───▶│ Analyze Req │───▶│ Delegate to │───▶│   Review    │      │
│  └─────────┘    │ (internal)  │    │ Coder (A2A) │    │   Result    │      │
│                 └─────────────┘    └─────────────┘    └──────┬──────┘      │
│                                                              │             │
│                                     ┌────────────────────────┤             │
│                                     │                        │             │
│                              ┌──────▼──────┐          ┌──────▼──────┐      │
│                              │  Not Done   │          │    Done     │      │
│                              │ (loop A2A)  │          │ (EDA event) │      │
│                              └──────┬──────┘          └─────────────┘      │
│                                     │                                      │
│                                     └─────▶ Back to Delegate               │
│                                                                             │
│  NO CODING TOOLS IN THIS DAG                                               │
│  • Does NOT run: describe, drift, plan, scaffold, verify                   │
│  • Only: analyzes, delegates, reviews, loops, emits events                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Implementation status:** `primitives/agent/ameide-coder/src/agent.py` needs refactoring to become AmeidePO - removing coding tools and adding A2A client.

#### 4.4 AmeideCoder (Devcontainer)

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AmeideCoder (Devcontainer)                             │
│                       Generic Autonomous Coder                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  A2A Server (receives tasks from AmeidePO)                                  │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        TOOL PALETTE                                 │   │
│  │                                                                     │   │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐       │   │
│  │  │ Ameide    │  │ Claude    │  │ Git CLI   │  │ Build/    │       │   │
│  │  │ CLI       │  │ Code      │  │           │  │ Test      │       │   │
│  │  └───────────┘  └───────────┘  └───────────┘  └───────────┘       │   │
│  │                                                                     │   │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐       │   │
│  │  │ Linters   │  │ File Ops  │  │ Search    │  │ Any       │       │   │
│  │  │           │  │           │  │           │  │ Other...  │       │   │
│  │  └───────────┘  └───────────┘  └───────────┘  └───────────┘       │   │
│  │                                                                     │   │
│  │  Agent decides HOW to use tools autonomously                        │   │
│  │  Ameide CLI is ONE tool among many (not a workflow)                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│       │                                                                     │
│       ▼                                                                     │
│  A2A Response (PR URL, summary, artifacts)                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

The coder is a **generic autonomous agent** that:
* Receives task via A2A
* Decides which tools to use
* Ameide CLI provides guardrails but is NOT prescriptive
* Returns results via A2A

**Implementation status:** `services/devcontainer_service` provides the base runtime. Needs A2A server endpoint and tool palette wiring.

#### 4.5 EDA Integration (Domain ↔ Agents)

Per backlog 496, communication between Transformation domain and agents uses EDA:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       EDA (backlog 496)                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Transformation Domain                   AmeidePO                           │
│         │                                    │                              │
│         │ Event: RequirementCreated          │                              │
│         │────────────────────────────────────▶                              │
│         │                                    │                              │
│         │      (PO orchestrates via A2A)     │                              │
│         │                                    │                              │
│         │ Event: RequirementCompleted        │                              │
│         │◀────────────────────────────────────                              │
│         │ (PR URL, summary, status)          │                              │
│                                                                             │
│  • Async, decoupled                                                         │
│  • Domain emits events, agents subscribe                                    │
│  • Agents emit events, domain subscribes                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Implementation status:** EDA infrastructure from 496 applies here. Event schemas for agent coordination need definition.

---

### 5. Implementation Plan (Epics & Tasks)

#### Epic 1 – AgentDefinition Schema for Two-Agent Model (agent-developer‑SCHEMA)

1.1. Extend AgentDefinition for AmeidePO:
* Add `a2a_client` field with `target_agent` reference
* Set `scope: PRODUCT_MANAGEMENT`, `risk_tier: MEDIUM`
* Empty `tools[]` - no coding tools

1.2. Extend AgentDefinition for AmeideCoder:
* Add `a2a_server` field with endpoint configuration
* Set `scope: CODE_EXECUTION`, `risk_tier: HIGH`
* Full tool palette: `ameide_cli`, `claude_code`, `git_cli`, `build_test`, `linters`

1.3. Schema migration:
* Add A2A fields to `V3__agent_a2a_support.sql`
* Update Agent operator to parse A2A configuration

#### Epic 2 – A2A Protocol Implementation (agent-developer‑A2A)

2.1. A2A Server (AmeideCoder):
* Implement `/.well-known/agent-card.json` endpoint
* Implement `/a2a` JSON-RPC endpoint (message/send, tasks/get)
* Implement SSE streaming for tasks/sendSubscribe
* Task lifecycle: submitted → working → completed/failed

2.2. A2A Client (AmeidePO):
* Integrate A2A Python SDK into LangGraph runtime
* AgentCard discovery from `a2a_client.target_agent`
* Task submission and response handling
* SSE streaming for real-time updates

2.3. A2A Authentication:
* OAuth2 token exchange between agents
* Scopes: `ameide.coder.run`, `ameide.po.delegate`

#### Epic 3 – AmeidePO LangGraph DAG (agent-developer‑PO)

3.1. Refactor existing coder agent:
* Rename `core-platform-coder` → `ameide-product-owner`
* Remove all coding tools (describe, drift, plan, etc.)
* Add A2A client tool for coder delegation

3.2. Implement PO DAG nodes:
* `analyze_requirement` - parse incoming requirement
* `delegate_to_coder` - send A2A task
* `review_result` - evaluate coder output
* `loop_or_complete` - decide if more work needed
* `emit_eda_event` - notify Transformation domain

3.3. Product management focus:
* NO coding activities in this DAG
* Decision making, delegation, quality review only

#### Epic 4 – AmeideCoder Devcontainer (agent-developer‑CODER)

4.1. Tool palette implementation:
* `ameide_cli` - CLI guardrails (describe, drift, plan, verify, etc.)
* `claude_code` - Claude Code SDK/CLI for file editing
* `git_cli` - clone, branch, commit, push, PR
* `build_test` - run build and test commands
* `linters` - code quality checks
* Extensible for additional tools

4.2. Autonomous execution:
* Agent decides which tools to use and when
* Ameide CLI provides guardrails, not prescriptive workflow
* Returns artifacts via A2A (PR URL, summary, logs)

4.3. Runtime environment:
* Based on `services/devcontainer_service`
* `.devcontainer ameideio/ameide` base image
* A2A server embedded in service

#### Epic 5 – EDA Integration (agent-developer‑EDA)

5.1. Event schemas (per backlog 496):
* `RequirementCreated` - domain → AmeidePO
* `RequirementCompleted` - AmeidePO → domain
* `RequirementFailed` - AmeidePO → domain
* `DevelopmentStarted` - AmeidePO → domain (optional)

5.2. Event handlers:
* AmeidePO subscribes to `RequirementCreated`
* Transformation subscribes to completion/failure events
* Update backlog item status from events

5.3. Decouple from direct invocation:
* Remove `InvokeAgent` direct calls
* All coordination via EDA events

#### Epic 6 – Security & Observability (agent-developer‑SECURITY)

6.1. Network boundaries:
* NetworkPolicy: AmeidePO ↔ AmeideCoder (A2A only)
* AmeideCoder cannot access cluster services
* Repo whitelisting per environment

6.2. A2A Security:
* mTLS between agents
* OAuth2 token validation
* Audit logging for all A2A calls

6.3. Observability:
* Trace A2A task lifecycle (submitted → completed)
* Metrics: task count, duration, success rate
* Logs correlated to requirement ID

6.4. SRE controls:
* Feature flags per agent
* Kill-switch for A2A communication
* Rate limiting on task submission

---

### 6. Open Questions

1. **A2A Loop Termination**

   * How does AmeidePO decide when to stop the A2A loop?
   * Options: max iterations, quality threshold, explicit "done" signal from coder
   * What happens if the loop times out?

2. **Tool Palette Extensibility**

   * How do we add new tools to AmeideCoder without redeploying?
   * Dynamic tool loading vs static tool palette?
   * Tool versioning and compatibility?

3. **Multi-Requirement Coordination**

   * Can AmeidePO handle multiple requirements in parallel?
   * How do we manage shared state across concurrent A2A tasks?
   * Ordering and dependency resolution?

4. **Persistent vs Ephemeral Devcontainers**

   * v1 assumes a long-lived devcontainer for AmeideCoder
   * Future: "one ephemeral pod per A2A task" using a CRD or Job
   * Tradeoff: startup latency vs isolation

5. **A2A Protocol Extensions**

   * Do we need Ameide-specific A2A extensions?
   * How do we handle streaming artifacts (logs, progress)?
   * Should we support push notifications for long-running tasks?

6. **EDA Event Granularity**

   * How detailed should progress events be?
   * Options: coarse (started/completed) vs fine (per-tool, per-file)
   * Impact on domain storage and UI updates
