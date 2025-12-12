# 505 – Agent Developer Execution Environment (V2 Target Architecture)

**Status:** Target architecture (implementation in progress)
**Owner:** Platform / Agents
**Depends on:** 496 (EDA Principles), 505-agent-developer.md (implementation notes)

## 0. Executive summary

We standardize Ameide's "agentic coding" into a **Process primitive + Agent** system:

- **Agile/TOGAF ADM Process primitive(s) (Temporal-based)** own the **governance lifecycle** (sprint/ADM/PI state machine, timebox tracking, phase gates). It emits EDA events but does NOT embed agent logic.
- **Transformation (Domain primitive)** remains the **source of truth** for requirements and their state.
- **AmeidePO (Product Owner)** is an **Agent primitive (LangGraph-based)** deployed by the Agent operator. It makes **product decisions** (priority, acceptance, scope) and subscribes to Process EDA events.
- **AmeideSA (Solution Architect)** is an **Agent primitive (LangGraph-based)** deployed by the Agent operator. It makes **technical decisions** (approach, decomposition) and delegates to AmeideCoder via **A2A**.
- **AmeideCoder** is a **devcontainer service** deployed in the cluster that exposes a **standard A2A Server** endpoint. It owns the **code lifecycle** (checkout → edit → test → commit → PR).

**Key boundaries:**
- Process primitive(s) track state and emit events; they never make decisions.
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

3) Use **standard Agent-to-Agent interoperability** for PO↔Coder:
- **A2A protocol** for agent communication, including streaming progress updates.

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

- **Requirement**: A work item owned by Transformation (ID, description, acceptance criteria, state).
- **Dev Brief**: A normalized execution-ready description produced by AmeidePO (what/why/done, constraints).
- **A2A Task**: A stateful unit of work created/owned by the A2A server (AmeideCoder).
- **Artifacts**: PR URL, logs, test outputs, summaries, diffs — returned by AmeideCoder through A2A.

---

## 4. Target architecture

### 4.1 Components

```
┌────────────────────┐
│   Transformation   │ (Domain: requirements + state)
└─────────┬──────────┘
          │ EDA events (per 496)
          │ RequirementCreated / RequirementCompleted
          ▼
┌─────────────────────┐
│  Process primitive  │ (Agile/TOGAF ADM: sprint/ADM lifecycle)
│  Temporal workflow  │
└─────────┬───────────┘
          │ EDA events
          │ SprintStarted / PhaseGateReached
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
| Requirement system of record | ❌ | ✅ | reads/updates | reads | reads context |
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

### 5.1 Transformation ↔ Process ↔ AmeidePO: EDA (per backlog 496)

Domain-to-process and process-to-agent communication uses async events:

**Domain ↔ Process:**

| Event | Direction | Payload |
|-------|-----------|---------|
| `RequirementCreated` | Domain → Process | requirementId, description, criteria |
| `RequirementCompleted` | Process → Domain | requirementId, prUrl, summary |
| `RequirementFailed` | Process → Domain | requirementId, reason, logs |

**Process ↔ AmeidePO:**

| Event | Direction | Payload |
|-------|-----------|---------|
| `SprintStarted` | Process → PO | sprintId, goals[], capacity |
| `ItemReadyForDev` | Process → PO | sprintId, requirementId |
| `ItemCompleted` | PO → Process | sprintId, requirementId, prUrl |
| `PhaseGateReached` | Process → PO | cycleId, phase, deliverables[] |

AmeidePO subscribes to process events, makes product decisions, emits completion events.

### 5.2 AmeidePO ↔ AmeideSA: EDA / direct invocation

AmeidePO delegates technical work to AmeideSA:

| Event/Call | Direction | Payload |
|------------|-----------|---------|
| `TechnicalWorkRequested` | PO → SA | requirementId, devBrief, constraints |
| `TechnicalWorkCompleted` | SA → PO | requirementId, prUrl, summary, risks |
| `TechnicalWorkFailed` | SA → PO | requirementId, reason, logs |

AmeideSA receives work requests, makes technical decisions (approach, decomposition), and delegates to AmeideCoder.

### 5.3 AmeideSA ↔ AmeideCoder: A2A (standard)

AmeideCoder MUST implement an A2A Server:

- Agent discovery: `GET /.well-known/agent.json`
- Task initiation/continuation: `message/send`
- Streaming updates (SSE): `message/stream`
- Polling fallback: `tasks/get`
- Optional: `tasks/cancel`, `tasks/resubscribe`

AmeideSA is an A2A Client:

1) Discover coder via AgentCard (Agent.json)
2) Send Task message (Dev Brief + repo target)
3) Stream progress, collect artifacts
4) Review technical output, decide: accept / request changes / cancel
5) Report results back to AmeidePO

### 5.4 "Skill" definition: what AmeideCoder exposes

AmeideCoder exposes one primary skill:

- **Skill ID:** `develop_requirement`
- **Input:** requirementId + repo coordinates + constraints + Dev Brief
- **Output artifacts:** PR URL + summary + evidence (tests/lints/logs)

The A2A message payload format:

- `TextPart`: human-readable Dev Brief (what/why/done)
- `DataPart`: structured JSON:
  - `requirementId`
  - `repoUrl` or `repoId`
  - `baseRef`
  - `branchName` (or branch strategy)
  - `allowedPaths` / `denyPaths`
  - `acceptanceCriteria`
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

---

## 6. External DAGs vs Devcontainer agent loop (clear I/O)

### 6.1 AmeidePO LangGraph DAG (product orchestration)

| Node | Input | Output | Notes |
|------|-------|--------|------|
| PO1: Receive process event | SprintStarted / ItemReadyForDev | requirement context | from Process primitive |
| PO2: Load requirement | requirementId | requirement payload | from Transformation |
| PO3: Make product decision | requirement payload | priority + scope + constraints | product owner judgment |
| PO4: Normalize Dev Brief | requirement + decision | Dev Brief | done definition + constraints |
| PO5: Delegate to SA | Dev Brief + repo target | work request | triggers AmeideSA |
| PO6: Accept/reject | SA result (prUrl, summary) | decision | accept / request changes / escalate |
| PO7: Close loop | final decision | requirement update | update Transformation state + emit ItemCompleted |

**Guarantee:** PO DAG does not run CLI guardrails, repo commands, or A2A to Coder directly.

### 6.2 AmeideSA LangGraph DAG (technical orchestration)

| Node | Input | Output | Notes |
|------|-------|--------|------|
| SA1: Receive work request | Dev Brief from PO | technical context | from AmeidePO |
| SA2: Analyze approach | Dev Brief + repo state | technical plan | decomposition, risks |
| SA3: Delegate to coder (A2A) | technical plan + repo target | A2A TaskId | starts task via message/send or message/stream |
| SA4: Observe + review | SSE status + artifacts | decision | accept / request changes / cancel |
| SA5: Iterate | decision + feedback | follow-up A2A message | same taskId until done |
| SA6: Report to PO | final artifacts | work result | prUrl, summary, risks |

**Guarantee:** SA DAG does not run CLI guardrails or repo commands directly - only delegates to Coder via A2A.

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
- Emits EDA events (SprintStarted, PhaseGateReached, etc.)
- Does NOT embed agent logic - purely state machine

### 7.2 AmeidePO (Agent primitive)

Deployed as a normal Agent CR (managed by Agent operator):
- Loads its AgentDefinition from Transformation
- Runs LangGraph DAG for product decisions
- Subscribes to Process EDA events
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
- `GET /.well-known/agent.json`
- `POST /` (A2A JSON-RPC endpoint) OR `POST /a2a` depending on the service URL declared in Agent.json

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
  - requirementId (Transformation)
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
   - Emit EDA events (SprintStarted, PhaseGateReached, etc.)

2) **AmeideCoder A2A server**
   - Serve Agent.json
   - Implement message/send + message/stream + tasks/get
   - Implement task store (single-replica first; shared store later)

3) **AmeideSA A2A client**
   - Discover coder via Agent.json
   - Stream progress and collect artifacts
   - Make technical decisions (approach, decomposition)
   - Report results to AmeidePO

4) **AmeidePO product orchestration**
   - Subscribe to Process EDA events
   - Make product decisions (priority, acceptance, scope)
   - Delegate technical work to AmeideSA (not directly to Coder)
   - Accept/reject results and emit ItemCompleted

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
