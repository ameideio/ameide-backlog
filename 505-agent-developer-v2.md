# 505 – Agent Developer Execution Environment (V2 Target Architecture)

**Status:** Target architecture (implementation in progress)
**Owner:** Platform / Agents
**Depends on:** 367-1 (Transformation Scrum profile), 506 (Scrum event model), 496 (EDA Principles), 505-agent-developer.md (implementation notes)

> See [507-scrum-agent-map.md](507-scrum-agent-map.md) for a visual/textual map tying Transformation, Process, and Agent layers together.

**Authority & supersession**

- This backlog is **authoritative for the Agent Developer architecture**: AmeidePO/AmeideSA/AmeideCoder roles, A2A contracts, and how agents consume/produce messages.  
- **Scrum data model and domain facts/intents** are owned by `300-400/367-1-scrum-transformation.md`, `506-scrum-vertical-v2.md`, and `508-scrum-protos.md`. This file must conform to those contracts (topics, event names, envelope fields).  
- **Operator responsibilities and condition vocabulary** are owned by `495-ameide-operators.md`, `499-process-operator.md`, `500-agent-operator.md`, and `502-domain-vertical-slice.md`.  
- The earlier `505-agent-developer.md` file is **historical / implementation tracking**; if it contradicts this v2 architecture or the 506-v2/508 event model, **this file wins**.

**Contract surfaces (owned elsewhere, referenced here)**

- **Domain intents** (commands) on `scrum.domain.intents.v1` and **domain facts** on `scrum.domain.facts.v1` (defined in 506-v2/508).  
- **Process facts** on `scrum.process.facts.v1` (defined in 506-v2).  
- **A2A REST binding + Agent Card** (`/.well-known/agent-card.json`, `/v1/message:send`, `/v1/message:stream`, `/v1/tasks/*`) shared with other agents and owned by the A2A proto/backlog series.  
- **Coder-internal CLI/tooling contracts** are defined in `504-agent-vertical-slice.md` and the 484a–484f CLI backlogs; AmeideCoder treats those as internal tools.

## Grounding & cross-references

- **Architecture grounding:** Built on the primitive stack and information model in `470-ameide-vision.md`, `471-ameide-business-architecture.md`, `472-ameide-information-application.md`, `473-ameide-technology.md`, `475-ameide-domains.md`, `476-ameide-security-trust.md`, `477-primitive-stack.md`, and the EDA rules in `496-eda-principles.md`.  
- **Scrum domain & process:** Consumes and produces Scrum intents/facts defined by `300-400/367-1-scrum-transformation.md`, `506-scrum-vertical-v2.md`, and `508-scrum-protos.md`; Process primitives and the Process operator that emit `scrum.process.facts.v1` are described in `499-process-operator.md`.  
- **Primitives & operators:** Relies on primitive/operator patterns from `495-ameide-operators.md`, `498-domain-operator.md`, `499-process-operator.md`, `500-agent-operator.md`, `501-uisurface-operator.md`, `502-domain-vertical-slice.md`, and `503-operators-helm-chart.md` for how AmeidePO/AmeideSA/AmeideCoder are deployed and surfaced.  
- **Implementation & CLI:** `505-agent-developer-v2-implementation.md` turns this architecture into concrete workstreams; `504-agent-vertical-slice.md` plus the 484a–484f CLI backlogs define the CLI/guardrail behavior that AmeideCoder uses internally; `507-scrum-agent-map.md` situates this architecture in the Stage 0–3 Scrum stack.

## 0. Executive summary

We standardize Ameide's "agentic coding" into a **Process primitive + Agent** system:

- **Agile/TOGAF ADM Process primitive(s) (Temporal-based)** own the **governance lifecycle** (sprint/ADM/PI state machine, timebox tracking, phase gates). They consume **Scrum domain facts** from `scrum.domain.facts.v1`, emit **process facts** on `scrum.process.facts.v1`, and issue **Scrum domain intents** on `scrum.domain.intents.v1`, but do NOT embed agent logic.
- **Transformation (Domain primitive)** remains the **source of truth** for Scrum artifacts (Product Backlog, Product Backlog Items, Sprints, Sprint Backlogs, Increments).
- **AmeidePO (Product Owner)** is an **Agent primitive (LangGraph-based)** deployed by the Agent operator. It makes **product decisions** (priority, acceptance, scope) and subscribes primarily to **process facts** on `scrum.process.facts.v1` (e.g., `SprintBacklogReadyForExecution`, `SprintBacklogItemReadyForWork`).
- **AmeideSA (Solution Architect)** is an **Agent primitive (LangGraph-based)** deployed by the Agent operator. It makes **technical decisions** (approach, decomposition) and delegates to AmeideCoder via **A2A**.
- **AmeideCoder** is a **devcontainer service** deployed in the cluster that exposes a **standard A2A Server** endpoint. It owns the **code lifecycle** (checkout → edit → test → commit → PR).

**Key boundaries:**
- Process primitive(s) track state and emit **process facts** / **domain intents**; they never make product or technical decisions.
- AmeidePO and AmeideSA never shell into repos, never run Ameide CLI, never run Codex/Claude CLIs.
- AmeideCoder does all code execution, using **Ameide CLI as an internal tool** (guardrails + repo intelligence).

---

## 1. Goals

1) Make “agent writes code safely in the official devcontainer and opens a PR” a **repeatable platform capability**, not a one-off script.

2) Ensure we have a **clear separation of responsibilities**:
- **Agile/Scrum/TOGAF ADM Process primitive(s)** = governance lifecycle state machine (no decisions)
- **AmeidePO** = product decisions (priority, acceptance, scope)
- **AmeideSA** = technical decisions (approach, decomposition, delegation)
- **AmeideCoder** = coding execution + evidence generation

3) Use **standard Agent-to-Agent interoperability** for **AmeideSA ↔ AmeideCoder**:
- **A2A protocol** for technical delegation, including streaming progress updates.

4) Keep operators “wiring only”:
- Operators deploy and secure runtimes (images, secrets, policies).
- Business logic lives in domain services and agent runtimes.

---

## 2. Non-goals

- Multi-tenant, customer-facing “self-serve AI dev env UI” (separate initiative).
- Auto-scaling ephemeral devcontainers “one pod per task” (future step).
- Replacing Git provider workflows / CI pipelines (we integrate, we do not replace).
- Encoding business rules for domains into the operator layer.

---

## 3. Glossary

- **Product Backlog Item (PBI)**: A Scrum work item owned by Transformation (ID, description, attributes such as estimate/value) that lives in the Product Backlog; when it meets the Definition of Done it can be included in an Increment. In legacy wording this was sometimes called a “Requirement”; that term is deprecated in the canonical contract.
- **Dev Brief**: A normalized execution-ready description produced by AmeidePO (what/why/done, constraints).
- **A2A Task**: A stateful unit of work created/owned by the A2A server (AmeideCoder).
- **Artifacts**: PR URL, logs, test outputs, summaries, diffs — returned by AmeideCoder through A2A.
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
          │ EDA / direct invocation
          ▼
┌─────────────────────┐   A2A (standard)   ┌─────────────────────────┐
│      AmeideSA       │ ──────────────────▶│      AmeideCoder        │
│  Agent primitive    │ ◀──────────────────│ Devcontainer A2A Server │
│   LangGraph DAG     │    (streaming)     │  Code lifecycle + tools │
└─────────────────────┘                    └─────────────────────────┘
```


### 4.2 Runtime placement

- **Process primitive(s)** run as **Temporal workers** (Process CRD) managed by the **Process operator**.
- **AmeidePO** runs in the **Agent Runtime Plane** (Agent primitive) managed by the **Agent operator**.
- **AmeideSA** runs in the **Agent Runtime Plane** (Agent primitive) managed by the **Agent operator**.
- **AmeideCoder** runs as a **Devcontainer Coder Service** in the cluster:
  - A2A Server endpoint (HTTP/S)
  - Workspace volume (persistent or ephemeral)
  - Tooling installed: `git`, build/test, Ameide CLI, and one or more "code editor backends" (Claude Code CLI / Codex CLI)

### 4.3 Responsibility split

| Area | Process primitive(s) | Transformation | AmeidePO | AmeideSA | AmeideCoder |
|------|---------------------|----------------|----------|----------|-------------|
| Governance lifecycle (sprint/ADM) | ✅ | ❌ | subscribes | ❌ | ❌ |
| Scrum artifact system of record (Product Backlog, PBIs, Sprints, Increments) | ❌ | ✅ | reads/updates | reads | reads context |
| Decide "what to build next" | ❌ | ❌ | ✅ | ❌ | ❌ |
| Own product DAG | ❌ | ❌ | ✅ | ❌ | ❌ |
| Technical approach/decomposition | ❌ | ❌ | ❌ | ✅ | ❌ |
| Delegate to coder (A2A) | ❌ | ❌ | ❌ | ✅ | ❌ |
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

### 5.2 AmeidePO ↔ AmeideSA: EDA / direct invocation

AmeidePO delegates technical work to AmeideSA:

| Event/Call | Direction | Payload |
|------------|-----------|---------|
| `TechnicalWorkRequested` | PO → SA | productBacklogItemId, devBrief, constraints |
| `TechnicalWorkCompleted` | SA → PO | productBacklogItemId, prUrl, summary, risks |
| `TechnicalWorkFailed` | SA → PO | productBacklogItemId, reason, logs |

AmeideSA receives work requests, makes technical decisions (approach, decomposition), and delegates to AmeideCoder.

### 5.3 AmeideSA ↔ AmeideCoder: A2A (standard)

AmeideCoder MUST implement an A2A Server that conforms to the current A2A REST binding:

- **Agent discovery:** `GET /.well-known/agent-card.json` returns the Agent Card. For legacy clients we mirror identical content at `/.well-known/agent.json`.
- **Task initiation/continuation:** `POST /v1/message:send`
- **Streaming updates (SSE):** `POST /v1/message:stream`
- **Polling fallback:** `GET /v1/tasks/{taskId}`
- **Optional controls:** `POST /v1/tasks/{taskId}:cancel`, `POST /v1/tasks/{taskId}:subscribe`

AmeideSA is an A2A Client:

1) Discover coder via AgentCard (Agent.json)
2) Send Task message (Dev Brief + repo target)
3) Stream progress, collect artifacts
4) Review technical output, decide: accept / request changes / cancel
5) Report results back to AmeidePO

### 5.4 "Skill" definition: what AmeideCoder exposes

AmeideCoder exposes one primary skill:

- **Skill ID:** `develop_product_backlog_item`
- **Input:** productBacklogItemId + repo coordinates + constraints + Dev Brief
- **Output artifacts:** PR URL + summary + evidence (tests/lints/logs)

The A2A message payload format:

- `TextPart`: human-readable Dev Brief (what/why/done)
- `DataPart`: structured JSON:
  - `productBacklogItemId`
  - `repoUrl` or `repoId`
  - `baseRef`
  - `branchName` (or branch strategy)
  - `allowedPaths` / `denyPaths`
  - `definitionOfDoneRef` (or equivalent Definition of Done context)
  - `policy`: allowed tools, timeouts, repo allowlist key

### 5.5 Coder task lifecycle (A2A Task state machine)

AmeideCoder tasks MUST follow a clear lifecycle:

- `submitted` → `working`
- `input-required` (only if coder needs clarification)
- terminal: `completed` / `failed` / `canceled`

SA continues a task by sending a follow-up A2A message with the same task ID until a terminal state is reached.

### 5.6 Artifacts contract (minimum set)

AmeideCoder MUST return, on completion:

- `artifact.pr_url` (required if success)
- `artifact.summary` (required)
- `artifact.tests` (required: pass/fail + command + key logs)
- `artifact.changed_files` (optional but strongly preferred)
- `artifact.risks` (optional: e.g., migrations, breaking changes)

### 5.7 Normative decisions

- **Agent discovery** is canonical at `/.well-known/agent-card.json` (with `/.well-known/agent.json` mirrored for backwards compatibility).
- **A2A transport** uses the REST binding first (`/v1/message:send`, `/v1/message:stream`, `/v1/tasks/...`). JSON-RPC bindings can be added later but are not required for v2.
- **Proto-first implementation:** The REST handlers wrap a gRPC/Connect service whose proto lives in the Ameide SDKs so internal systems can stay proto-driven even while honoring the public REST binding.
- **Scrum artifact state** changes only through Scrum domain intents/facts defined in `508-scrum-protos.md` (e.g., `CreateProductBacklogItemRequested`, `CommitSprintBacklogRequested`, `RecordProductBacklogItemDoneRequested`, `RecordIncrementRequested`, and their corresponding facts). All downstream systems rely on these emitted domain events; there are no “accept/reject requirement” RPCs in the core Scrum contract.
- **Repo access** outside the devcontainer is prohibited. AmeideSA always requests any repo digest or evidence from AmeideCoder over A2A instead of touching git directly.

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

**Guarantee:** PO DAG does not run CLI guardrails, repo commands, or A2A to Coder directly.

### 6.2 AmeideSA LangGraph DAG (technical orchestration)

| Node | Input | Output | Notes |
|------|-------|--------|------|
| SA1: Receive work request | Dev Brief from PO | technical context | from AmeidePO |
| SA2: Request repo digest (A2A) | Dev Brief + repo target | repo digest (read-only) | `POST /v1/message:send` with `intent=repo_digest`; AmeideCoder returns summary + key files |
| SA3: Analyze approach | Dev Brief + repo digest | technical plan | decomposition, risks, guardrail plan |
| SA4: Delegate implementation (A2A) | technical plan + repo target | A2A taskId | starts task via `POST /v1/message:send` (`skill=develop_product_backlog_item`) |
| SA5: Observe + review | SSE status + artifacts | decision | accept / request changes / cancel |
| SA6: Iterate | decision + feedback | follow-up A2A message | same taskId until done |
| SA7: Report to PO | final artifacts | work result | prUrl, summary, risks |

**Guarantee:** SA DAG does not run CLI guardrails or repo commands directly; even read-only repo context is obtained through AmeideCoder's A2A APIs.

### 6.3 AmeideCoder devcontainer execution loop (internal)

| Phase | Input | Output | Internal tooling |
|------|-------|--------|------------------|
| C1: Authorize | A2A request | accepted/rejected | auth + repo allowlist |
| C2: Workspace prep | repoUrl + baseRef | worktree + branch | git clone/checkout |
| C3: Guardrails | repo + Dev Brief | plan + risks | **Ameide CLI** (internal) |
| C4: Edit implementation | plan | code changes | Claude/Codex CLI + file ops |
| C5: Verify | code changes | pass/fail evidence | build/test/lint + Ameide verify |
| C6: Package result | evidence | commit + PR | git + provider API |
| C7: Respond | PR + evidence | A2A artifacts | stream + final completion |

---

## 7. Deployment model (cluster)

### 7.1 Process primitive(s) (Agile/TOGAF ADM)

Deployed as Process CRs (managed by Process operator):
- Loads ProcessDefinition from Transformation
- Runs Temporal workflows for governance lifecycle
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
- Has **A2A client capability** (HTTP out) to call AmeideCoder

### 7.4 AmeideCoder (Devcontainer coder service)

Deployed as a platform component (Helm/GitOps):
- Deployment/Service
- Workspace PV (or per-task ephemeral)
- ExternalSecret for:
  - Git token (scoped)
  - A2A auth keys / JWT verification keys
  - Optional: provider keys for Claude/Codex CLIs

AmeideCoder exposes:
- `GET /.well-known/agent-card.json` (mirrors to `/.well-known/agent.json` for legacy clients)
- `POST /v1/message:send`
- `POST /v1/message:stream` (SSE)
- `GET /v1/tasks/{taskId}`
- `POST /v1/tasks/{taskId}:cancel`
- `POST /v1/tasks/{taskId}:subscribe`

---

## 8. Security posture

### 8.1 Risk tiers

- Process primitive(s): LOW (pure state machine, no external calls)
- AmeidePO: MEDIUM (no repo shell, no git push, no A2A to Coder)
- AmeideSA: MEDIUM (A2A client only, no direct repo shell)
- AmeideCoder: HIGH (repo write + PR creation)

### 8.2 Policy enforcement points

AmeideCoder enforces:
- repo allowlist
- allowedPaths / denyPaths (optional but strongly recommended)
- timeouts per phase
- tool allowlist (e.g., "Claude allowed, but no kubectl")

Cluster enforces:
- NetworkPolicy: SA may call Coder; Coder cannot laterally access internal services except what is explicitly allowed.
- No cluster credentials inside coder workspace.

---

## 9. Observability

- Correlation IDs:
  - productBacklogItemId (Transformation)
  - sprintId / cycleId (Process primitive)
  - A2A taskId (AmeideCoder)
- Logs and metrics emitted by Process primitive:
  - sprint/phase state transitions
  - timebox tracking
- Logs and metrics emitted by AmeideCoder:
  - duration per phase (prep/edit/test/pr)
  - success rate
  - failure reasons
- Streaming:
  - status updates and log artifacts streamed to SA via SSE.
  - SA aggregates and reports to PO.

---

## 10. Legacy bridge (explicitly deprecated)

### 10.1 The old pattern (bridge only)

Older flows used:
- A LangGraph “coder” agent calling a `develop_in_container` tool
- Which invoked a devcontainer service endpoint (`/v1/develop`) to run shell commands

This is now considered a **compatibility bridge** only.

### 10.2 Migration goal

End-state:
- The devcontainer exposes **A2A server** directly (AmeideCoder).
- `develop_in_container` becomes internal-only or disappears.
- AmeideSA is the A2A client that delegates to AmeideCoder.
- AmeidePO focuses on product decisions and delegates to AmeideSA.
- Process primitive(s) own governance lifecycle (sprint/ADM).

---

## 11. Implementation milestones (v2)

1) **Process primitive(s) for governance**
   - Define ProcessDefinitions for Scrum Sprint, TOGAF ADM
   - Deploy Process CRs via Process operator
   - Emit **process facts** on `scrum.process.facts.v1` (e.g., `SprintBacklogReadyForExecution`, `SprintBacklogItemReadyForWork`) based on the Scrum domain facts they consume from `scrum.domain.facts.v1`

2) **AmeideCoder A2A server**
   - Serve Agent Card at `/.well-known/agent-card.json` (mirror to `agent.json`)
   - Implement `/v1/message:send`, `/v1/message:stream`, `/v1/tasks/*`
   - Implement task store (single-replica first; shared store later)

3) **AmeideSA A2A client**
   - Discover coder via Agent Card
   - Stream progress and collect artifacts
   - Make technical decisions (approach, decomposition)
   - Report results to AmeidePO

4) **AmeidePO product orchestration**
   - Subscribe to **process facts** on `scrum.process.facts.v1`
   - Make product decisions (priority, acceptance, scope)
   - Delegate technical work to AmeideSA (not directly to Coder)
   - Accept/reject results and emit Scrum domain intents such as `CommitSprintBacklogRequested`, `RefineProductBacklogItemRequested`, `RecordProductBacklogItemDoneRequested`, and `RecordIncrementRequested`; Process workflows complete when they observe the corresponding `ProductBacklogItemDoneRecorded` / `IncrementUpdated` / `SprintEnded` domain facts (no separate “item completed” process fact in the v2 design)

5) **Remove coding tools from PO and SA**
   - No CLI guardrails in PO or SA DAGs
   - PO only makes product decisions
   - SA only makes technical decisions and delegates via A2A

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
- PO ↔ SA communication: EDA events vs direct invocation vs message queue?
- SA decomposition: how fine-grained should task breakdown be before A2A to Coder?

---

## 13. Related documents

| Document | Purpose |
|----------|---------|
| [505-agent-developer.md](505-agent-developer.md) | Implementation status, A2A protocol examples, ASCII diagrams |
| [496 EDA Principles](496-eda-principles.md) | Event-driven architecture patterns for domain↔agent |
