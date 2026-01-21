# 505 – Agent Developer Execution Environment (V2 Target Architecture)

**Status:** Target architecture (implementation in progress)
**Owner:** Platform / Agents
**Depends on:** 367-1 (Transformation Scrum profile), 506 (Scrum event model), 496 (EDA Principles), 505-agent-developer.md (implementation notes)

> See [507-scrum-agent-map.md](507-scrum-agent-map.md) for a visual/textual map tying Transformation, Process, and Agent layers together.

**Authority & supersession**

- This backlog is **authoritative for the Agent Developer architecture**: role taxonomy, inter-role handover contracts, and how agents consume/produce messages.
- **Scrum data model and domain facts/intents** are owned by `300-400/367-1-scrum-transformation.md`, `506-scrum-vertical-v2.md`, and `508-scrum-protos.md`. This file must conform to those contracts (topics, event names, envelope fields).  
- **Operator responsibilities and condition vocabulary** are owned by `495-ameide-operators.md`, `499-process-operator.md`, `500-agent-operator.md`, and `502-domain-vertical-slice.md`.  
- The earlier `505-agent-developer.md` file is **historical / implementation tracking**; if it contradicts this v2 architecture or the 506-v2/508 event model, **this file wins**.

**Contract surfaces (owned elsewhere, referenced here)**

- **Domain intents** (commands) on `scrum.domain.intents.v1` and **domain facts** on `scrum.domain.facts.v1` (defined in 506-v2/508).  
- **Process facts** on `scrum.process.facts.v1` (defined in 506-v2).  
- **Agent work handover** (canonical): event-driven work delegation between roles (defined in this backlog; must follow 496 envelope invariants).
- **A2A REST binding + Agent Card** (optional transport binding): `/.well-known/agent-card.json`, `/v1/message:send`, `/v1/message:stream`, `/v1/tasks/*` shared with other agents and owned by the A2A proto/backlog series; when used, it MUST map to the canonical work handover semantics rather than introducing a parallel state machine.
- **Coder-internal CLI/tooling contracts** are defined in `504-agent-vertical-slice.md` and the 484a–484f CLI backlogs; AmeideCoder treats those as internal tools.
- **MCP (agentic access) compatibility layer:** MCP servers are Integration primitives that expose capability-owned tools/resources; agents do not require MCP internally and should prefer Ameide SDK clients for typed calls. See `backlog/534-mcp-protocol-adapter.md`.

### Protocols (clarification)

- **MCP (capability tools/resources):** Agent → Capability (Domain/Projection). Tool identifiers are capability-scoped (`<capability>.<operation>`) and can be invoked via SDK clients (preferred in-platform) or via MCP protocol adapters (external/devtool compatibility).
- **Agent work handover (canonical, event-driven):** Role → Role delegation (Product Owner/Solution Architect/Executor), suitable for integrating external tools without coupling to a specific HTTP/SDK boundary.
- **A2A (optional transport binding):** Agent → Agent delegation transport (often interactive/streaming). If present, it publishes/consumes the same canonical work handover messages and returns the same evidence artifacts.

## Grounding & cross-references

- **Architecture grounding:** Built on the primitive stack and information model in `470-ameide-vision.md`, `471-ameide-business-architecture.md`, `472-ameide-information-application.md`, `473-ameide-technology.md`, `475-ameide-domains.md`, `476-ameide-security-trust.md`, `477-primitive-stack.md`, and the integration rules in `496-eda-principles-v6.md`.  
- **Scrum domain & process:** Consumes and produces Scrum intents/facts defined by `300-400/367-1-scrum-transformation.md`, `506-scrum-vertical-v2.md`, and `508-scrum-protos.md`; Process primitives and the Process operator that emit `scrum.process.facts.v1` are described in `499-process-operator.md`.  
- **Primitives & operators:** Relies on primitive/operator patterns from `495-ameide-operators.md`, `498-domain-operator.md`, `499-process-operator.md`, `500-agent-operator.md`, `501-uisurface-operator.md`, `502-domain-vertical-slice.md`, and `503-operators-helm-chart.md` for how AmeidePO/AmeideSA/AmeideCoder are deployed and surfaced.  
- **Implementation & CLI:** `505-agent-developer-v2-implementation.md` turns this architecture into concrete workstreams; `504-agent-vertical-slice.md` plus the 484a–484f CLI backlogs define the CLI/guardrail behavior that AmeideCoder uses internally; `507-scrum-agent-map.md` situates this architecture in the Stage 0–3 Scrum stack.
- **Capability implementation DAG:** `backlog/533-capability-implementation-playbook.md` (generic, parallelizable playbook; this backlog plugs in as the Agent node discipline).

## 0. Executive summary

We standardize Ameide's "agentic coding" into a **Process primitive + Agents + event-driven handover** system:

- Note (naming): role naming is being standardized in `backlog/717-ameide-agents-v6.md`. Treat “Product Owner / Solution Architect / Executor” as legacy labels when reading this doc; the boundaries and safety invariants remain applicable.

- **Agile/TOGAF ADM Process primitive(s) (Temporal-based)** own the **governance lifecycle** (sprint/ADM/PI state machine, timebox tracking, phase gates). They consume **Scrum domain facts** from `scrum.domain.facts.v1`, emit **process facts** on `scrum.process.facts.v1`, and issue **Scrum domain intents** on `scrum.domain.intents.v1`, but do NOT embed agent logic.
- **Transformation (Domain primitive)** remains the **source of truth** for Scrum artifacts (Product Backlog, Product Backlog Items, Sprints, Sprint Backlogs, Increments).
- **Agents implement role-specific reasoning** (LangGraph DAGs) and communicate work handovers via events:
  - **Product Owner** (AmeidePO) makes product/scope decisions and produces Dev Briefs.
  - **Solution Architect** (AmeideSA) makes technical decisions and produces execution-ready plans.
  - **Executor** (AmeideCoder) performs coding/tool runs in a constrained execution environment and returns evidence.

**Key boundaries:**
- Process primitive(s) track state and emit **process facts** / **domain intents**; they never make product or technical decisions.
- AmeidePO and AmeideSA never shell into repos, never run Ameide CLI, never run Codex/Claude CLIs.
- AmeideCoder does all code execution, using **Ameide CLI as an internal tool** (guardrails + repo intelligence), and returns structured evidence suitable for promotion gates.

---

## 1. Goals

1) Make “agent writes code safely in the official devcontainer and opens a PR” a **repeatable platform capability**, not a one-off script.

2) Ensure we have a **clear separation of responsibilities**:
- **Agile/Scrum/TOGAF ADM Process primitive(s)** = governance lifecycle state machine (no decisions)
- **Product Owner** (AmeidePO) = product decisions (priority, acceptance, scope)
- **Solution Architect** (AmeideSA) = technical decisions (approach, decomposition, delegation)
- **Executor** (AmeideCoder) = coding execution + evidence generation

### 1.1 Role taxonomy (generic; methodology mappings live in profiles)

These are **methodology-agnostic delivery roles**. Scrum/TOGAF/PMI profiles map their accountabilities to these roles; the platform enforces tool grants + risk tiers per role via AgentDefinitions.

| Role | Primary decisions | Canonical outputs | Repo execution | Default write posture |
|------|-------------------|------------------|---------------|-----------------------|
| Product Owner | What/why/priority/scope | Dev Brief + acceptance constraints | ❌ | Propose via intents/drafts only |
| Solution Architect | How/plan/risks | Technical plan + guardrail plan | ❌ | Propose via intents/drafts only |
| Executor | How to implement safely | PR + evidence bundle (tests/lint/verify) | ✅ (constrained) | Implements in repo; domain writes only via governed intents |

3) Prefer **event-driven handover** between roles (bus-first):
- Delegation is expressed as canonical work messages (see §5.2–§5.3), so external tools can integrate without coupling to an HTTP-specific agent runtime.
- If an interactive transport is required, A2A can be used as a transport binding that maps to the same canonical work messages.

4) Keep operators “wiring only”:
- Operators deploy and secure runtimes (images, secrets, policies).
- Business logic lives in domain services and agent runtimes.

---

## 2. Non-goals

- Multi-tenant, customer-facing “self-serve AI dev env UI” (separate initiative).
- Auto-scaling ephemeral devcontainers “one pod per task” (superseded by the 527 WorkRequest substrate; in-cluster execution is KEDA ScaledJob scheduling executor Jobs).
- Replacing Git provider workflows / CI pipelines (we integrate, we do not replace).
- Encoding business rules for domains into the operator layer.

---

## 3. Glossary

- **Product Backlog Item (PBI)**: A Scrum work item owned by Transformation (ID, description, attributes such as estimate/value) that lives in the Product Backlog; when it meets the Definition of Done it can be included in an Increment. In legacy wording this was sometimes called a “Requirement”; that term is deprecated in the canonical contract.
- **Dev Brief**: A normalized execution-ready description produced by AmeidePO (what/why/done, constraints).
- **Work Task**: A unit of delegated work (Product Owner→Solution Architect or Solution Architect→Executor) represented canonically as event-driven handover messages; may optionally be exposed/streamed via A2A.
- **Artifacts / evidence**: PR URL, logs, test outputs, summaries, diffs, and structured verification results — returned by Executor and attached to promotions/approvals as evidence.
- **`thread_id` (agent persisted identity)**: Required identity for any Agent that enables persistence/checkpointing; mapped to LangGraph `configurable.thread_id`. Recommended scheme: `<tenant>/<capability>/<work_item_id>`, with a separate `run_id` for concurrent executions under the same business identity. Agent state stores “resume essentials” and keeps large outputs as out-of-band artifact references.

---

## 4. Target architecture

### 4.1 Components

```
┌────────────────────┐
│   Transformation   │ (Domain: Scrum artifacts + state)
└─────────┬──────────┘
          │ Scrum domain facts (scrum.domain.facts.v1, per 496/506-v2/508)
          │ ProductBacklogItemCreated / ProductBacklogItemDoneRecorded / SprintCreated/Started / SprintBacklogCommitted / IncrementUpdated
          ▼
┌─────────────────────┐
│  Process primitive  │ (Agile/TOGAF ADM: sprint/ADM lifecycle)
│  Temporal workflow  │
└─────────┬───────────┘
          │ Process facts (scrum.process.facts.v1)
          │ SprintBacklogReadyForExecution / SprintBacklogItemReadyForWork / SprintTimeboxReachedEnd
          ▼
┌─────────────────────┐
│      AmeidePO       │ (Product Owner: priority, acceptance, scope)
│  Agent primitive    │
│   LangGraph DAG     │
└─────────┬───────────┘
          │ work handover (canonical events; optional A2A transport)
          ▼
┌─────────────────────┐                   ┌─────────────────────────┐
│      AmeideSA       │ ────────────────▶ │      AmeideCoder        │
│  Agent primitive    │   work intents     │  Executor runtime       │
│   LangGraph DAG     │ ◀───────────────  │  (repo tools + evidence) │
└─────────────────────┘    work facts      └─────────────────────────┘
```


### 4.2 Runtime placement

- **Process primitive(s)** run as **Temporal workers** (Process CRD) managed by the **Process operator**.
- **AmeidePO** runs in the **Agent Runtime Plane** (Agent primitive) managed by the **Agent operator**.
- **AmeideSA** runs in the **Agent Runtime Plane** (Agent primitive) managed by the **Agent operator**.
- **AmeideCoder** runs as an **Executor runtime** (workspace/task provider such as Coder/Che, or a CI-like runner):
  - Workspace volume (persistent or ephemeral)
  - Tooling installed: `git`, build/test, `bin/ameide`, `buf`, and one or more "code editor backends" (Codex CLI / Claude Code CLI)
  - Optional A2A Server endpoint (HTTP/S) for interactive delegation/streaming (transport binding only)

### 4.3 Responsibility split

| Area | Process primitive(s) | Transformation | AmeidePO | AmeideSA | AmeideCoder |
|------|---------------------|----------------|----------|----------|-------------|
| Governance lifecycle (sprint/ADM) | ✅ | ❌ | subscribes | ❌ | ❌ |
| Scrum artifact system of record (Product Backlog, PBIs, Sprints, Increments) | ❌ | ✅ | reads/updates | reads | reads context |
| Decide "what to build next" | ❌ | ❌ | ✅ | ❌ | ❌ |
| Own product DAG | ❌ | ❌ | ✅ | ❌ | ❌ |
| Technical approach/decomposition | ❌ | ❌ | ❌ | ✅ | ❌ |
| Delegate to executor (work handover) | ❌ | ❌ | ❌ | ✅ | ❌ |
| Checkout/edit/test/push | ❌ | ❌ | ❌ | ❌ | ✅ |
| Run Ameide CLI | ❌ | ❌ | ❌ | ❌ | ✅ (internal tool) |
| Talk to Codex/Claude CLI | ❌ | ❌ | ❌ | ❌ | ✅ |
| Produce PR + evidence | ❌ | ❌ | accepts/rejects | reviews | ✅ |

---

## 5. Contracts and interfaces

### 5.1 Transformation ↔ Process ↔ AmeidePO: Commands + Events (per backlog 496/506-v2/508)

Transformation remains the **system of record** for Scrum artifacts (Product Backlog, Product Backlog Items, Sprints, Sprint Backlogs, Increments). Everyone else issues commands to Transformation via **Scrum domain intents**; the domain emits immutable **Scrum domain facts** after state changes. Process primitives consume those domain facts and emit process facts; AmeidePO primarily consumes process facts.

**Scrum domain facts (Transformation → Process/PO):**

Emitted on `scrum.domain.facts.v1`:

| Event | Direction | Payload (core fields only) |
|-------|-----------|---------------------------|
| `ProductBacklogItemCreated` | Transformation → Process | productBacklogItemId, productId, description, estimate/value |
| `ProductBacklogItemRefined` | Transformation → Process | productBacklogItemId, updated attributes |
| `ProductBacklogReordered` | Transformation → Process | productId, orderedProductBacklogItemIds[] |
| `SprintCreated` | Transformation → Process | sprintId, productId, startAt, endAt |
| `SprintStarted` | Transformation → Process | sprintId, productId |
| `SprintCanceled` | Transformation → Process | sprintId, reason |
| `SprintEnded` | Transformation → Process | sprintId |
| `SprintBacklogCommitted` | Transformation → Process | sprintId, sprintBacklog (Sprint Goal + selected PBIs) |
| `SprintBacklogChanged` | Transformation → Process | sprintId, sprintBacklog |
| `ProductBacklogItemDoneRecorded` | Transformation → Process | productBacklogItemId, sprintId?, definitionOfDoneId, doneAt |
| `IncrementCreated` / `IncrementUpdated` | Transformation → Process | incrementId, productId, sprintId?, definitionOfDoneId, includedProductBacklogItemIds[] |

**Commands into Transformation (Scrum domain intents):**

Published on `scrum.domain.intents.v1`:

| Command | Caller | Payload (core fields only) |
|---------|--------|----------------------------|
| `CreateProductBacklogItemRequested` | AmeidePO / other | productId, description, estimate/value, orderRank? |
| `RefineProductBacklogItemRequested` | AmeidePO | productBacklogItemId, patch + updateMask |
| `OrderProductBacklogRequested` | AmeidePO | productId, orderedProductBacklogItemIds[] |
| `CreateSprintRequested` | Process / AmeidePO | productId, startAt, endAt |
| `StartSprintRequested` / `CancelSprintRequested` / `EndSprintRequested` | Process | sprintId (+ reason for cancel) |
| `CommitSprintBacklogRequested` | AmeidePO | sprintId, Sprint Goal, selectedProductBacklogItemIds[], plan? |
| `ChangeSprintBacklogRequested` | AmeidePO | sprintId, add/remove PBI ids, plan? |
| `RecordProductBacklogItemDoneRequested` | AmeidePO / Process | productBacklogItemId, sprintId?, definitionOfDoneId, doneAt? |
| `RecordIncrementRequested` | AmeidePO / Process | incrementId?, productId, sprintId?, definitionOfDoneId, includedProductBacklogItemIds[] |

Process primitives never emit domain-scoped facts; they only publish intents and consume domain facts.

**Process ↔ AmeidePO (process facts on `scrum.process.facts.v1`):**

| Event | Direction | Payload |
|-------|-----------|---------|
| `SprintBacklogReadyForExecution` | Process → PO | sprintId, productId |
| `SprintBacklogItemReadyForWork` | Process → PO | sprintId, productBacklogItemId, optional devBriefRef |
| `SprintStartingSoon` / `SprintEndingSoon` | Process → PO | sprintId, productId, thresholds |
| `SprintTimeboxReachedEnd` | Process → PO | sprintId, productId |

AmeidePO subscribes to process facts, makes product decisions, and issues Scrum domain intents (e.g., `CommitSprintBacklogRequested`, `RefineProductBacklogItemRequested`, `RecordProductBacklogItemDoneRequested`, `RecordIncrementRequested`) once work is accepted. Process workflows complete when they observe the corresponding domain facts (`ProductBacklogItemDoneRecorded`, `IncrementUpdated`, `SprintEnded`).

### 5.2 Product Owner ↔ Solution Architect: work handover (canonical, event-driven)

Product Owner delegates technical work to Solution Architect via a bus-native work contract.

**Topics (target):**

- `agent.work.intents.v1` (requests)
- `agent.work.facts.v1` (status + outcomes)

**Minimal work kinds (v1):**

- `PLAN_TECHNICAL_WORK` (Product Owner → Solution Architect)
- `DELIVER_IMPLEMENTATION` (Solution Architect → Executor)
- `REPO_DIGEST` (Solution Architect → Executor; read-only)

Product Owner → Solution Architect handover (conceptual event set):

| Message | Direction | Payload (core fields only) |
|---------|-----------|----------------------------|
| `WorkRequested` | Product Owner → Solution Architect | `work_id`, `work_kind=PLAN_TECHNICAL_WORK`, `product_backlog_item_id`, `dev_brief_ref`, constraints |
| `WorkCompleted` | Solution Architect → Product Owner | `work_id`, outcome refs (plan ref), risks |
| `WorkFailed` | Solution Architect → Product Owner | `work_id`, reason, evidence refs |

Solution Architect MUST treat these messages as the source of truth for delegation state; any interactive transport (A2A) is an optional binding that must map to the same messages and outcomes.

### 5.3 Solution Architect ↔ Executor: work handover (canonical, event-driven; optional A2A binding)

Solution Architect delegates repo execution to Executor via the same work contract.

Solution Architect → Executor handover (conceptual event set):

| Message | Direction | Payload (core fields only) |
|---------|-----------|----------------------------|
| `WorkRequested` | Solution Architect → Executor | `work_id`, `work_kind=DELIVER_IMPLEMENTATION`, repo coordinates, `allowed_paths`/`deny_paths`, Dev Brief + plan refs |
| `WorkProgressed` | Executor → Solution Architect | `work_id`, phase (`checkout`/`edit`/`verify`/`pr`), short status |
| `WorkCompleted` | Executor → Solution Architect | `work_id`, `artifact.pr_url`, summary, evidence bundle refs |
| `WorkFailed` | Executor → Solution Architect | `work_id`, reason, evidence bundle refs |

**Optional binding (interactive/streaming): A2A**

If A2A is enabled for interactive tooling, it is a transport binding for the same semantics:

- A2A `taskId` MUST map 1:1 to `work_id`.
- A2A streaming events MUST be derivable from `WorkProgressed` / `WorkCompleted` / `WorkFailed`.
- A2A artifacts MUST reference the same evidence bundles referenced in facts.

### 5.4 Executor capability (“deliver implementation”)

Executor exposes one primary capability:

- **Capability ID:** `deliver_implementation`
- **Input:** productBacklogItemId + repo coordinates + constraints + Dev Brief
- **Output artifacts:** PR URL + summary + evidence (tests/lints/logs)

If an A2A binding is used, the task payload should include:

- `TextPart`: human-readable Dev Brief (what/why/done)
- `DataPart`: structured JSON:
  - `productBacklogItemId`
  - `repoUrl` or `repoId`
  - `baseRef`
  - `branchName` (or branch strategy)
  - `allowedPaths` / `denyPaths`
  - `definitionOfDoneRef` (or equivalent Definition of Done context)
  - `policy`: allowed tools, timeouts, repo allowlist key

### 5.5 Work lifecycle (canonical; transport-independent)

Work MUST follow a clear lifecycle:

- `submitted` → `working`
- `input-required` (only if coder needs clarification)
- terminal: `completed` / `failed` / `canceled`

If A2A is used, follow-up messages MUST target the same `work_id` and only refine inputs (never mutate outcomes retroactively).

### 5.6 Artifacts contract (minimum set)

AmeideCoder MUST return, on completion:

- `artifact.pr_url` (required if success)
- `artifact.summary` (required)
- `artifact.tests` (required: pass/fail + command + key logs)
- `artifact.changed_files` (optional but strongly preferred)
- `artifact.risks` (optional: e.g., migrations, breaking changes)

### 5.7 Normative decisions

- **Event-driven first:** role handover is bus-native (see §5.2–§5.3); this is the integration boundary for external tools.
- **Optional A2A binding:** when enabled, it is a transport binding for the same work semantics (taskId = work_id), not a parallel workflow system.
- **Scrum artifact state** changes only through Scrum domain intents/facts defined in `508-scrum-protos.md` (e.g., `CreateProductBacklogItemRequested`, `CommitSprintBacklogRequested`, `RecordProductBacklogItemDoneRequested`, `RecordIncrementRequested`, and their corresponding facts). All downstream systems rely on these emitted domain events; there are no “accept/reject requirement” RPCs in the core Scrum contract.
- **Repo access** outside the executor environment is prohibited. Solution Architect obtains repo digest/evidence by delegating work, not by running git directly.

---

## 6. External DAGs vs Devcontainer agent loop (clear I/O)

### 6.1 AmeidePO LangGraph DAG (product orchestration)

| Node | Input | Output | Notes |
|------|-------|--------|------|
| PO1: Receive process event | SprintBacklogReadyForExecution / SprintBacklogItemReadyForWork | PBI context | from Process primitive (`scrum.process.facts.v1`) |
| PO2: Load PBI | productBacklogItemId | PBI payload | from Transformation |
| PO3: Make product decision | PBI payload | priority + scope + constraints | product owner judgment |
| PO4: Normalize Dev Brief | PBI + decision | Dev Brief | Definition of Done context + constraints |
| PO5: Delegate to SA | Dev Brief + repo target | work request | triggers AmeideSA |
| PO6: Accept/reject | SA result (prUrl, summary) | decision | accept / request changes / escalate |
| PO7: Close loop | final decision | Scrum domain intents | call `RecordProductBacklogItemDoneRequested` and, if applicable, `RecordIncrementRequested` in Transformation; Process workflows observe the resulting `ProductBacklogItemDoneRecorded` / `IncrementUpdated` domain facts and complete without a separate “item completed” process event. |

**Guarantee:** PO DAG does not run CLI guardrails or repo commands, and does not delegate to the executor directly (handover always goes through the Solution Architect role).

### 6.2 AmeideSA LangGraph DAG (technical orchestration)

| Node | Input | Output | Notes |
|------|-------|--------|------|
| SA1: Receive work request | Dev Brief from PO | technical context | from AmeidePO |
| SA2: Request repo digest (work intent) | Dev Brief + repo target | repo digest (read-only) | publish `WorkRequested(work_kind=REPO_DIGEST)`; executor returns summary + key files |
| SA3: Analyze approach | Dev Brief + repo digest | technical plan | decomposition, risks, guardrail plan |
| SA4: Delegate implementation (work intent) | technical plan + repo target | work_id | publish `WorkRequested(work_kind=DELIVER_IMPLEMENTATION)` |
| SA5: Observe + review | work facts + artifacts | decision | accept / request changes / cancel |
| SA6: Iterate | decision + feedback | follow-up work intent | same `work_id` until done |
| SA7: Report to PO | final artifacts | work result | prUrl, summary, risks |

**Guarantee:** SA DAG does not run CLI guardrails or repo commands directly; even read-only repo context is obtained through delegated work results.

### 6.3 AmeideCoder devcontainer execution loop (internal)

| Phase | Input | Output | Internal tooling |
|------|-------|--------|------------------|
| C1: Authorize | work intent | accepted/rejected | auth + repo allowlist |
| C2: Workspace prep | repoUrl + baseRef | worktree + branch | git clone/checkout |
| C3: Guardrails | repo + Dev Brief | plan + risks | **Ameide CLI** (internal) |
| C4: Edit implementation | plan | code changes | Claude/Codex CLI + file ops |
| C5: Verify | code changes | pass/fail evidence | build/test/lint + `ameide test` |
| C6: Package result | evidence | commit + PR | git + provider API |
| C7: Respond | PR + evidence | work facts (+ optional A2A artifacts) | publish outcomes; stream if enabled |

---

## 7. Deployment model (cluster)

### 7.1 Process primitive(s) (Agile/TOGAF ADM)

Deployed as Process CRs (managed by Process operator):
- Loads the published ProcessDefinition (BPMN) as a governed artifact (v6 posture: tenant Enterprise Repository under `processes/<module>/<process_key>/v<major>/process.bpmn`, mediated by the owning Domain where required)
- Runs Zeebe process instances for governance lifecycle (not Temporal)
- Emits **process facts** on `scrum.process.facts.v1` (e.g., `SprintBacklogReadyForExecution`, `SprintBacklogItemReadyForWork`, `SprintTimeboxReachedEnd`, `SLAWarning`) based on the domain facts it consumes from `scrum.domain.facts.v1`
- Does NOT embed agent logic - purely state machine

### 7.2 AmeidePO (Agent primitive)

Deployed as a normal Agent CR (managed by Agent operator):
- Loads its AgentDefinition from Transformation
- Runs LangGraph DAG for product decisions
- Subscribes to **process facts** on `scrum.process.facts.v1` (and can optionally read domain facts from `scrum.domain.facts.v1` if needed)
- Invokes AmeideSA for technical work (does NOT call AmeideCoder directly)

### 7.3 AmeideSA (Agent primitive)

Deployed as a normal Agent CR (managed by Agent operator):
- Loads its AgentDefinition from Transformation
- Runs LangGraph DAG for technical orchestration
- Publishes/consumes work handover messages (bus) for delegation
- Optional: A2A client (HTTP out) for interactive delegation/streaming when enabled

### 7.4 AmeideCoder (Executor runtime)

Deployed as a platform component (Helm/GitOps):
- Deployment/Service
- Workspace PV (or per-task ephemeral)
- ExternalSecret for:
  - Git token (scoped)
  - bus credentials (publish/consume work intents/facts)
  - optional A2A auth keys / JWT verification keys (if A2A transport is enabled)
  - Optional: provider keys for Claude/Codex CLIs

Executor consumes work intents, produces work facts, and emits evidence artifacts suitable for promotion gates.

Optional (interactive/streaming binding): expose an A2A server surface that maps to the same `work_id` lifecycle.

**Update (2026-01):** In `ameide-gitops`, the devcontainer service is currently represented by an optional component stub (under `environments/_optional/**`) and a disabled placeholder values file (`sources/values/_shared/platform/platform-devcontainer-service.yaml`). Treat AmeideCoder as a planned executor runtime until that component is enabled with a digest-pinned image and explicit NetworkPolicies.

---

## 8. Security posture

### 8.1 Risk tiers

- Process primitive(s): LOW (pure state machine, no external calls)
- AmeidePO: MEDIUM (no repo shell, no git push, no executor credentials)
- AmeideSA: MEDIUM (no direct repo shell; delegates via work handover)
- AmeideCoder: HIGH (repo write + PR creation)

### 8.2 Policy enforcement points

AmeideCoder enforces:
- repo allowlist
- allowedPaths / denyPaths (optional but strongly recommended)
- timeouts per phase
- tool allowlist (e.g., "Claude allowed, but no kubectl")
- guardrail execution with evidence capture:
  - `bin/ameide primitive verify` (per `backlog/484a-ameide-cli-primitive-workflows.md`)
  - repo workflow constraints (dev→PR→main posture per `backlog/400-agentic-development.md` and repo CI policy)
  - pinned toolchain where applicable (e.g., Codex CLI pin in `backlog/433-codex-cli-057.md`)

Cluster enforces:
- NetworkPolicy: only allow explicitly declared delegation/integration paths (e.g., Solution Architect → Executor if an A2A binding is enabled); Executor cannot laterally access internal services except what is explicitly allowlisted.
- No broad cluster credentials by default. If the executor needs cluster access for a specific operation, use least-privilege identities and bind-only delegation to predeclared roles (do not let executor provisioning mint new RBAC privileges).

---

## 9. Observability

- Correlation IDs:
  - productBacklogItemId (Transformation)
  - sprintId / cycleId (Process primitive)
  - work_id (work handover)
- Logs and metrics emitted by Process primitive:
  - sprint/phase state transitions
  - timebox tracking
- Logs and metrics emitted by AmeideCoder:
  - duration per phase (prep/edit/test/pr)
  - success rate
  - failure reasons
- Streaming:
  - if enabled, interactive transport streams must be derivable from work facts and evidence artifacts (no “secret extra state”).

---

## 10. Legacy bridge (explicitly deprecated)

### 10.1 The old pattern (bridge only)

Older flows used:
- A LangGraph “coder” agent calling a `develop_in_container` tool
- Which invoked an executor endpoint (historically documented as a devcontainer service endpoint like `/v1/develop`) to run shell commands

This is now considered a **compatibility bridge** only.

### 10.2 Migration goal

End-state:
- Work handover is bus-native (`agent.work.*`); external tools integrate by publishing intents and consuming facts.
- `develop_in_container` becomes internal-only or disappears.
- Optional A2A surfaces (when present) are adapters over the same work handover.
- AmeidePO focuses on product decisions and delegates to AmeideSA.
- Process primitive(s) own governance lifecycle (sprint/ADM).

---

## 11. Implementation milestones (v2)

1) **Process primitive(s) for governance**
   - Define ProcessDefinitions for Scrum Sprint, TOGAF ADM
   - Deploy Process CRs via Process operator
   - Emit **process facts** on `scrum.process.facts.v1` (e.g., `SprintBacklogReadyForExecution`, `SprintBacklogItemReadyForWork`) based on the Scrum domain facts they consume from `scrum.domain.facts.v1`

2) **Work handover topics + semantics**
   - Publish `agent.work.intents.v1` / `agent.work.facts.v1` contracts (envelope invariants, required fields, evidence references)
   - Implement idempotent consumption and evidence attachment for executor outcomes

3) **Executor runtime**
   - Consume work intents, perform repo execution with guardrails, publish work facts + evidence
   - Optional: add A2A transport binding for interactive streaming

4) **AmeidePO product orchestration**
   - Subscribe to **process facts** on `scrum.process.facts.v1`
   - Make product decisions (priority, acceptance, scope)
   - Delegate technical work to AmeideSA (not directly to Coder)
   - Accept/reject results and emit Scrum domain intents such as `CommitSprintBacklogRequested`, `RefineProductBacklogItemRequested`, `RecordProductBacklogItemDoneRequested`, and `RecordIncrementRequested`; Process workflows complete when they observe the corresponding `ProductBacklogItemDoneRecorded` / `IncrementUpdated` / `SprintEnded` domain facts (no separate “item completed” process fact in the v2 design)

5) **Remove coding tools from PO and SA**
   - No CLI guardrails in PO or SA DAGs
   - PO only makes product decisions
   - SA only makes technical decisions and delegates via work handover

6) **Security hardening**
   - repo allowlist
   - NetworkPolicies
   - scoped git credentials
   - audit logging

---

## 12. Open questions

- Task storage / horizontal scaling: when do we need a shared task store?
- Workspace model: persistent pool vs one workspace per task
- Code backend: standardize on Claude Code CLI vs Codex CLI, or keep pluggable
- Artifact retention: what do we store long-term vs ephemeral logs
- Process primitive granularity: one Process per methodology or one per active sprint/cycle?
- PO ↔ SA communication: keep it bus-native (work handover) vs allow direct invocation in-cluster?
- SA decomposition: how fine-grained should task breakdown be before delegating to the executor?

---

## 13. Related documents

| Document | Purpose |
|----------|---------|
| [505-agent-developer.md](505-agent-developer.md) | Historical notes (may reference deprecated A2A-first posture) |
| [496 Integration Principles](496-eda-principles-v6.md) | Hybrid integration patterns for domain↔agent |
