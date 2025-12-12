# 505 – Agent Developer Execution Environment (V2 Target Architecture)

**Status:** Target architecture (implementation in progress)
**Owner:** Platform / Agents
**Depends on:** 496 (EDA Principles), 505-agent-developer.md (implementation notes)

## 0. Executive summary

We standardize Ameide’s “agentic coding” into a **two-agent system**:

- **Transformation (Domain)** remains the **source of truth** for requirements and their state.
- **AmeidePO (Product Owner Agent)** is a **LangGraph-based Agent primitive** deployed by the Agent operator. It owns the **product development DAG** (triage → delegate → review → iterate → complete).
- **AmeideCoder (Devcontainer Coder Agent)** is a **devcontainer-based coding runtime** deployed in the cluster that exposes a **standard A2A Server** endpoint. It owns the **code lifecycle** (checkout → edit → test → commit → PR).

**Key boundary:**  
AmeidePO never shells into repos, never runs Ameide CLI, never runs Codex/Claude CLIs.  
AmeideCoder does all code execution, and uses **Ameide CLI as an internal tool** (guardrails + repo intelligence).

---

## 1. Goals

1) Make “agent writes code safely in the official devcontainer and opens a PR” a **repeatable platform capability**, not a one-off script.

2) Ensure we have a **clear separation of responsibilities**:
- **AmeidePO** = orchestration + product decisions
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
┌─────────────────────┐   A2A (standard)   ┌─────────────────────────┐
│      AmeidePO       │ ──────────────────▶│      AmeideCoder        │
│  Agent primitive    │ ◀──────────────────│ Devcontainer A2A Server │
│   LangGraph DAG     │    (streaming)     │  Code lifecycle + tools │
└─────────────────────┘                    └─────────────────────────┘
```


### 4.2 Runtime placement

- **AmeidePO** runs in the **Agent Runtime Plane** (Agent primitive) managed by the **Agent operator**.
- **AmeideCoder** runs as a **Devcontainer Coder Service** in the cluster:
  - A2A Server endpoint (HTTP/S)
  - Workspace volume (persistent or ephemeral)
  - Tooling installed: `git`, build/test, Ameide CLI, and one or more “code editor backends” (Claude Code CLI / Codex CLI)

### 4.3 Responsibility split

| Area | Transformation | AmeidePO | AmeideCoder |
|------|----------------|----------|-------------|
| Requirement system of record | ✅ | reads/updates | reads context |
| Decide “what to build next” | ❌ | ✅ | ❌ |
| Own product DAG | ❌ | ✅ | ❌ |
| Checkout/edit/test/push | ❌ | ❌ | ✅ |
| Run Ameide CLI | ❌ | ❌ | ✅ (internal tool) |
| Talk to Codex/Claude CLI | ❌ | ❌ | ✅ |
| Produce PR + evidence | ❌ | reviews | ✅ |

---

## 5. Contracts and interfaces

### 5.1 Transformation ↔ AmeidePO: EDA (per backlog 496)

Domain-to-agent communication uses async events, not direct invocation:

| Event | Direction | Payload |
|-------|-----------|---------|
| `RequirementCreated` | Domain → PO | requirementId, description, criteria |
| `RequirementCompleted` | PO → Domain | requirementId, prUrl, summary |
| `RequirementFailed` | PO → Domain | requirementId, reason, logs |

AmeidePO subscribes to domain events, performs work, emits completion events.

### 5.2 AmeidePO ↔ AmeideCoder: A2A (standard)

AmeideCoder MUST implement an A2A Server:

- Agent discovery: `GET /.well-known/agent.json`
- Task initiation/continuation: `message/send`
- Streaming updates (SSE): `message/stream`
- Polling fallback: `tasks/get`
- Optional: `tasks/cancel`, `tasks/resubscribe`

AmeidePO is an A2A Client:

1) Discover coder via AgentCard (Agent.json)
2) Send Task message (Dev Brief + repo target)
3) Stream progress, collect artifacts
4) Decide: accept / request changes / cancel
5) Update Transformation requirement state with results

### 5.3 "Skill" definition: what AmeideCoder exposes

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

### 5.4 Coder task lifecycle (A2A Task state machine)

AmeideCoder tasks MUST follow a clear lifecycle:

- `submitted` → `working`
- `input-required` (only if coder needs clarification)
- terminal: `completed` / `failed` / `canceled`

PO continues a task by sending a follow-up A2A message with the same task ID until a terminal state is reached.

### 5.5 Artifacts contract (minimum set)

AmeideCoder MUST return, on completion:

- `artifact.pr_url` (required if success)
- `artifact.summary` (required)
- `artifact.tests` (required: pass/fail + command + key logs)
- `artifact.changed_files` (optional but strongly preferred)
- `artifact.risks` (optional: e.g., migrations, breaking changes)

---

## 6. External DAG vs Devcontainer agent loop (clear I/O)

### 6.1 AmeidePO LangGraph DAG (external orchestration)

| Node | Input | Output | Notes |
|------|-------|--------|------|
| PO1: Load requirement | requirementId | requirement payload | from Transformation |
| PO2: Normalize Dev Brief | requirement payload | Dev Brief | done definition + constraints |
| PO3: Delegate to coder (A2A) | Dev Brief + repo target | A2A TaskId | starts task via message/send or message/stream |
| PO4: Observe + review | SSE status + artifacts | decision | accept / request changes / cancel |
| PO5: Iterate | decision + feedback | follow-up A2A message | same taskId until done |
| PO6: Close loop | final artifacts | requirement update | update Transformation state + attach PR URL |

**Guarantee:** PO DAG does not run CLI guardrails or repo commands.

### 6.2 AmeideCoder devcontainer execution loop (internal)

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

### 7.1 AmeidePO (Agent primitive)

Deployed as a normal Agent CR (managed by Agent operator):
- Loads its AgentDefinition from Transformation
- Runs LangGraph DAG
- Has **A2A client capability** (HTTP out) to call AmeideCoder

### 7.2 AmeideCoder (Devcontainer coder service)

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

- AmeidePO: MEDIUM (no repo shell, no git push)
- AmeideCoder: HIGH (repo write + PR creation)

### 8.2 Policy enforcement points

AmeideCoder enforces:
- repo allowlist
- allowedPaths / denyPaths (optional but strongly recommended)
- timeouts per phase
- tool allowlist (e.g., “Claude allowed, but no kubectl”)

Cluster enforces:
- NetworkPolicy: PO may call Coder; Coder cannot laterally access internal services except what is explicitly allowed.
- No cluster credentials inside coder workspace.

---

## 9. Observability

- Correlation IDs:
  - requirementId (Transformation)
  - A2A taskId (AmeideCoder)
- Logs and metrics emitted by AmeideCoder:
  - duration per phase (prep/edit/test/pr)
  - success rate
  - failure reasons
- Streaming:
  - status updates and log artifacts streamed to PO via SSE.

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
- AmeidePO remains the orchestration agent.

---

## 11. Implementation milestones (v2)

1) **AmeideCoder A2A server**
   - Serve Agent.json
   - Implement message/send + message/stream + tasks/get
   - Implement task store (single-replica first; shared store later)

2) **AmeidePO A2A client**
   - Discover coder via Agent.json
   - Stream progress and collect artifacts
   - Loop until acceptance criteria met

3) **Remove coding tools from PO**
   - No CLI guardrails in PO DAG
   - PO only delegates/reviews

4) **Security hardening**
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

---

## 13. Related documents

| Document | Purpose |
|----------|---------|
| [505-agent-developer.md](505-agent-developer.md) | Implementation status, A2A protocol examples, ASCII diagrams |
| [496 EDA Principles](496-eda-principles.md) | Event-driven architecture patterns for domain↔agent |
