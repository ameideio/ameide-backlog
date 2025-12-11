

## agent-developer – Agent Developer Execution Environment (LangGraph + DevContainer + Claude SDK)

**Status:** Active (guardrail plumbing landed; LangGraph/devcontainer runtime in progress)
**Owner:** Platform / Agents
**Depends on:**

* 471/472/473/475/476 (Vision, App, Tech, Domains, Security)
* Agent primitive / AgentDefinitions (Transformation)
* Existing `core-platform-coder` / codex-cli work (if any)
* ArgoCD / GitOps / ApplicationSet patterns
* Historical context: [000-200/030-coding-agent.md](000-200/030-coding-agent.md) and [000-200/041-coder.md](000-200/041-coder.md) capture the earlier PoC implementations and are now marked as superseded in favour of this backlog.

---

### 0. Status snapshot (repo evidence)

| Workstream | State | Notes & repo pointers |
|------------|-------|-----------------------|
| **Agent primitive plumbing** | ✅ | Agent CRD/operator Helm/GitOps bundles were refreshed (see `operators/helm/crds/ameide.io_agents.yaml`, `gitops/ameide-gitops/sources/charts/platform/ameide-operators/crds/ameide.io_agents.yaml`) so `definitionRef`/tooling fields in this backlog are available cluster-wide. Demo assets live under `gitops/.../primitives/agent/core-platform-coder.yaml` + `scripts/demo-agent-slice.sh`. |
| **CLI guardrails + prompts** | ✅ | `packages/ameide_core_cli/internal/commands/primitive.go` ships `--repo-root/--gitops-root/--mode` plumbing plus naming/security/EDA/test heuristics for `verify --mode all`, and `primitive_prompt.go` + `prompts/agent/default.md` now instruct agents to run the full guardrail loop via the devcontainer tool. |
| **LangGraph coder DAG** | ⚠️ Partial | `services/agent_runtime_langgraph/src/agent_runtime_langgraph/coder/agent.py` backs the runtime; the new devtools (`coder/print_dag.py`, `coder/dev_graph.py`) and `langgraph.dev.json` allow local DAG inspection (`uv run … python -m agent_runtime_langgraph.coder.print_dag`) and `langgraph dev` launches. The graph, however, is still a single develop_in_container call—explicit nodes for describe/drift/plan/impact/scaffold/verify remain TODO. |
| **Devcontainer gRPC service** | ✅ | `services/devcontainer_service` exposes `POST /v1/develop` (packaged inside `ghcr.io/ameide/devcontainer-service`), and `gitops/.../platform/platform-devcontainer-service.yaml` wires the Deployment/Service + ExternalSecret (`platform-devcontainer-agent-token`) via the new `platform-devcontainer-service` component. |
| **develop_in_container tool** | ✅ | `services/agent_runtime_langgraph/src/agent_runtime_langgraph/tools/develop_in_container.py` registers the tool, enforces repo/command arguments, streams logs back to LangGraph, and the default agent prompt now calls it whenever a CLI command needs to run inside the devcontainer. |
| **Workflow/Temporal metadata** | ⚠️ Partial | **Future automation (Stage‑2)** – backlog/300-400/367-2-agent-orchestration-coding-agent.md tracked the idea of auto-planning/auto-running Coding Agent work via the existing Workflow + Temporal stack. That orchestration layer still lacks methodology metadata (`timebox_ids`, repo adapter policies) and isn’t part of the LangGraph/devcontainer slice; keep the row as a reminder that the governance/attestation automation remains open downstream. |

> Keep this table in sync with backlog/504 and 502 snapshots so platform/agent teams can see exactly what’s implemented. The workflow/Temporal row references the historical Stage‑2 automation plan (`367-2-agent-orchestration-coding-agent.md`); that document remains as a governance/auto-plan blueprint, not as a dependency for the LangGraph/devcontainer runtime described here.

---

### 1. Purpose

Provide a **first‑class “agent developer” runtime** inside Ameide:

* Every new **change request** can be executed by a **LangGraph‑based coding DAG** (“coder agent”) running in the cluster.
* The DAG orchestrates steps like: understand requirement → plan changes → edit code in a dev container → run tests → open PR.
* Actual code edits happen via **Claude Code SDK/CLI** running inside a **.devcontainer image** (`ameideio/ameide`) exposed to the cluster via **gRPC**.
* The existing **Agent primitive** stays high‑level; we just add a `type=langgraph` + DAG + tool config.
* A dedicated **CRD + Agent runtime** turns this into a **primitive in Ameide**, deployable and operable at scale via ArgoCD.

Goal: make “an agent that writes code in the official devcontainer, then opens a PR” a **repeatable, observable, safe pattern**, not a one‑off script.

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

* **Vertical slice contract.** The coder runtime extends the `core-platform-coder` scenario delivered in [504-agent-vertical-slice.md](504-agent-vertical-slice.md §2-5); every LangGraph DAG invocation must run the same OBSERVE→REASON→ACT→VERIFY loop by invoking the CLI commands from [484a-ameide-cli-primitive-workflows.md](484a-ameide-cli-primitive-workflows.md) (`describe`, `drift`, `plan`, `impact`, `scaffold`, `verify`, `prompt`). The DAG should treat these commands/tool calls as explicit nodes so that prompts/logs stay identical to the human/agent workflows already documented.
* **Sample assets + GitOps.** When we add the devcontainer service and LangGraph runtime, update the sample manifests referenced in 504 (e.g. `operators/helm/examples/agent-sample.yaml`, `gitops/.../primitives/agent/core-platform-coder.yaml`, prompt profiles under `prompts/agent/default.md`) so the guardrail demo script keeps working with the new `runtime_type=langgraph` AgentDefinition and the `develop_in_container` tool binding.
* **Dual-track reporting.** Continue the dual-track format from [502-domain-vertical-slice.md](502-domain-vertical-slice.md) by documenting operator changes (new CRD fields, ApplicationSet wiring for the devcontainer service) and CLI changes (new prompt profile hints, scaffold updates) in the same structure once the LangGraph DAG is implemented, so downstream teams can trace status exactly like they do for backlog 504.
* **Primitive-first scaffolds.** The CLI must scaffold agents into `primitives/{kind}/{name}` (e.g. `primitives/agent/core-platform-coder`). Service catalog entries stay human-authored while we pivot the tooling toward agentic coding, so Backstage metadata can lag without blocking the primitive workflow.

---

### 3. High‑Level Architecture

Text diagram:

```text
                ┌───────────────────────────────────────┐
                │            ArgoCD / GitOps            │
                │  (deploy Agent runtimes & dev svc)  │
                └───────────────────────────────────────┘

                    (platform namespaces: ameide-{env})

┌───────────────────────────────────────────────────────────────────────┐
│                       Agent Runtime Plane                            │
│                                                                       │
│  ┌─────────────────────────┐       gRPC        ┌───────────────────┐  │
│  │  Agent runtime (coder)│──────────────────▶│ devcontainer svc  │  │
│  │  (LangGraph runtime)    │                  │ (.devcontainer     │  │
│  │  - loads AgentDefinition│                  │  ameideio/ameide)  │  │
│  │  - runs LangGraph DAG   │                  │  + Claude SDK/CLI  │  │
│  │  - calls tools (incl.   │◀─────────────────┤  exposed via gRPC  │  │
│  │    develop_in_container)│   responses       └───────────────────┘  │
│  └─────────────────────────┘                                         │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘

Data/control flow for a dev task:

1. Transformation domain creates a BacklogItem/Initiative.
2. A Process primitive (e.g. “Transform with Ameide”) triggers the coder Agent runtime.
3. The Agent runtime loads an AgentDefinition (runtime_type=langgraph, dag_ref=..., tools=[develop_in_container,...]).
4. LangGraph DAG orchestrates steps; whenever it needs to actually change code, it calls `develop_in_container`.
5. `develop_in_container` tool calls devcontainer gRPC (repo, branch, instructions).
6. devcontainer gRPC wrapper drives Claude SDK/CLI inside the .devcontainer.
7. At the end, DAG opens a PR / posts status back to Transformation domain.
```

Key points:

* **AgentDefinition** stays the source of truth for “how this coder agent behaves” (DAG, tools, risk tier).
* **Agent runtime** is just one more Agent runtime (like other Agent runtimes), but specialised for coding tasks.
* **devcontainer service** is the only thing that actually runs Claude Code / codex CLI with filesystem access to repos.

---

### 4. Design Details

#### 4.1 AgentDefinition extensions

Extend the agent primitive (in Transformation) with fields like:

```yaml
# conceptual
AgentDefinition:
  id: "core-platform-coder"
  runtime_type: "langgraph"
  dag_ref:
    image: "ghcr.io/ameideio/agent-runtime-langgraph:main"  # for codegen
    module: "ameide.agents.coder.graph"
    entrypoint: "build_graph"
  tools:
    - id: "develop_in_container"
      type: "internal"
      config_ref: "develop-in-container-config/v1"
    - id: "read_repo_state"
    - id: "create_pr"
  scope: "WRITE_CODE"
  risk_tier: "HIGH"
```

Transformation remains the **design-time owner**; promotion & risk governance live there.

**Implementation status:** The AgentDefinition schema + migration now include `runtime_type`, `dag_ref`, `scope`, `risk_tier`, and tool grants (`db/flyway/sql/transformation/V2__agent_definitions.sql`, `services/transformation/src/transformations/service.ts:672-845`). The Agent operator fetches these definitions through the Go SDK and mounts the serialized JSON alongside the basic ID/tenant metadata for runtimes to consume (`operators/agent-operator/internal/controller/agent_controller.go`).

#### 4.2 Agent runtime (coder)

A standard Agent runtime service:

* Image: e.g. `agent-runtime-langgraph` (Python).
* Responsibilities:

  * Load AgentDefinition by id from Transformation.
  * Instantiate LangGraph DAG via `dag_ref`.
  * Expose an `InvokeAgent`/`RunTask` RPC (already in Agent runtime contract) that:

    * Takes a “dev task” (link to backlog item / requirement artifact).
    * Runs the DAG, which calls tools.
    * Returns status + artifacts (PR URL, summary, logs).

No special semantics beyond “runtime_type=langgraph”.

**Implementation status:** `services/agent_runtime_langgraph/src/agent_runtime_langgraph/coder/agent.py`
and `services/agent_runtime_langgraph/src/agent_runtime_langgraph/app.py` register the
`core-platform-coder` runtime as a dedicated LangGraph service so the Agent runtime can
invoke the `develop_in_container` tool directly. Dev helpers (`coder/print_dag.py`,
`coder/dev_graph.py`, `services/agent_runtime_langgraph/langgraph.dev.json`) now let us
render the DAG locally and run it via `langgraph dev`, but the graph still executes the
guardrail workflow as a single develop_in_container call; future iterations must split
describe → drift → plan → scaffold → verify into explicit LangGraph nodes and load DAG
metadata directly from Transformation AgentDefinitions.

#### 4.3 devcontainer service + Claude SDK gRPC

Service runs **.devcontainer `ameideio/ameide`** (the same environment humans would use) as a long‑lived DevPod, but:

* Adds a small **gRPC server** in front of Claude Code SDK/CLI with an API like:

```protobuf
service DevContainerCoding {
  rpc DevelopInContainer(DevelopRequest) returns (DevelopResponse);
}

message DevelopRequest {
  string repo_url;
  string branch_name;        // or "create from main"
  string task_id;            // backlink to Ameide backlog item
  string instruction;        // natural language + structured hints
  repeated string paths;     // optional files/folders to focus on
}

message DevelopResponse {
  string new_branch;
  string pr_url;            // optional
  string log_url;           // build/test logs
  bool   success;
  string summary;
}
```

Implementation inside the devcontainer:

* Clone repo (if not already).
* Create or reuse a “work” branch.
* Run Claude Code SDK/CLI with the instruction, letting it edit files.
* Run tests / formatting commands.
* Commit and optionally push + open PR.
* Return summary & URLs.

Security:

* Service must be configured **per repo / tenant**; no free‑for‑all shell.
* No cluster credentials inside the devcontainer; everything goes via Git hosting and CI/CD.

**Implementation status:** The platform devcontainer Deployment/Service plus ExternalSecret wiring live under `gitops/ameide-gitops/sources/values/_shared/platform/platform-devcontainer-service.yaml`, and the `core-platform-coder` Agent manifest (`gitops/ameide-gitops/sources/values/_shared/platform/agents/core-platform-coder.yaml`) binds to the `develop_in_container` grant so LangGraph runtimes can reach the gRPC endpoint.

#### 4.4 `develop_in_container` tool

From LangGraph’s perspective, this is just a tool that:

* Takes a structured instruction (plus repo + branch context)
* Calls the devcontainer gRPC
* Returns structured result (PR URL, summary, logs)

Conceptual type:

```ts
type DevelopInContainerArgs = {
  repoId: string;
  branchStrategy: "feature-per-task" | "reuse";
  paths?: string[];
  instruction: string;   // generated by agent from requirement
};

type DevelopInContainerResult = {
  prUrl?: string;
  branchName: string;
  summary: string;
  success: boolean;
  logsUrl?: string;
};
```

Mapping to gRPC is hidden behind the Agent runtime’s tool adapter layer.

#### 4.5 Implementation status (2025‑02)

* **Tool runtime.** `services/agent_runtime_langgraph/src/agent_runtime_langgraph/tools/develop_in_container.py`
  registers the LangChain tool. It reads the guardrail command list that LangGraph hands it, then:
  * Calls the remote devcontainer service via HTTP/JSON when `DEVELOP_IN_CONTAINER_ENDPOINT` +
    `DEVELOP_IN_CONTAINER_TOKEN` are present (timeouts configurable via `DEVELOP_IN_CONTAINER_TIMEOUT`).
  * Falls back to local execution (`subprocess_shell` inside the inference pod) so unit tests can run
    without the cluster service. Logs/stdout/stderr for every guardrail command are returned to the DAG.
* **Dependency wiring.** `ToolDependencies` / `_tool_dependencies` now include the devcontainer endpoint
  fields, so every tool builder gets the same configuration (`services/inference/src/inference_service/tools/base.py`,
  `services/inference/src/inference_service/context.py`).
* **Coder agent.** `services/agent_runtime_langgraph/src/agent_runtime_langgraph/coder/agent.py` renders the guardrail
  prompt via `ameide primitive prompt --json` (with an inline fallback), extracts `workflowCommands`, and invokes
  `develop_in_container` with repo/branch/task metadata derived from the Transformation context. The agent emits
  a summary with PR URL/branch/log links supplied by the tool.
* **CLI contract.** `packages/ameide_core_cli/internal/commands/primitive_prompt.go` now exposes `--json`
  output containing the rendered prompt + `workflowCommands`, plus `--proto-path`/`--gitops-root` flags so
  LangGraph can pass the paths it resolved from Transformation. Tests (`primitive_prompt_test.go`) verify
  the JSON payload and fallback behavior.
* **Sample manifests + GitOps.** The `core-platform-coder` Agent CRs under `operators/helm/examples/…` and
  `gitops/ameide-gitops/**/core-platform-coder.yaml` now grant the `develop_in_container` tool (riskTier=high) and
  reference the devcontainer endpoint secret (`platform/devcontainer-agent-token`). The guardrail demo script
  keeps working once the devcontainer Deployment + ExternalSecret are deployed.
* **Devcontainer service.** `services/devcontainer_service` packages the HTTP/JSON wrapper (`POST /v1/develop`)
  inside the official devcontainer image (`ghcr.io/ameide/devcontainer-service`). It clones repositories on-demand
  via `DEVCONTAINER_GIT_TOKEN`, executes each guardrail command via `/bin/sh`, and returns logs/branch summaries.
* **Cluster rollout.** `gitops/ameide-gitops/environments/_shared/components/platform/developer/devcontainer-service/component.yaml`
  (plus `platform-devcontainer-service.yaml`) deploy the Deployment/Service/workspace volume, and the
  `platform-devcontainer-agent-token-sync` ExternalSecret materializes the PAT used by both the service and the
  sample AgentDefinitions.
* **Service catalog template – GAP.** `service_catalog/agents/` still only exposes the `_primitive/template.yaml`
  placeholder; no `langgraph-scribe` (or equivalent) Backstage entry exists yet, so teams cannot scaffold the
  coder runtime from the Service Catalog.

---

### 5. Implementation Plan (Epics & Tasks)

#### Epic 1 – Spec & contract (agent-developer‑SPEC)

1.1. Extend AgentDefinition schema in Transformation:

* Add `runtime_type`, `dag_ref`, `tools`, `scope`, `risk_tier` fields for LangGraph agents.
  1.2. Define “coder agent” contract:
* What inputs it expects (link to backlog item / requirement).
* What outputs it guarantees (PR URL, summary, status).
  1.3. Document security expectations:
* Coder agents are always `risk_tier=HIGH`.
* Only Transformation / Platform can invoke them directly.

#### Epic 2 – LangGraph Agent runtime runtime (agent-developer‑RUNTIME)

2.1. Build `agent-runtime-langgraph` image:

* Base Python image with LangGraph + Ameide SDK + gRPC server.
  2.2. Implement Agent runtime server:
* `InvokeAgent` API that:

  * Loads AgentDefinition from Transformation.
  * Builds the LangGraph DAG from `dag_ref`.
  * Executes the DAG with given task context.
    2.3. Backstage / service catalog:
* Add a template/skeleton for LangGraph Agent runtimes (or reuse generic Agent runtime template with `runtime_type=langgraph`).
  2.4. ArgoCD:
* Helm chart + ApplicationSet entry to deploy Agent runtime in `ameide-{env}`.

#### Epic 3 – Devcontainer coding service (agent-developer‑DEVCONTAINER)

3.1. Decide DevPod vs plain Deployment:

* For now: one long‑lived pod per env, running `.devcontainer ameideio/ameide`.
  3.2. Wrap Claude Code SDK/CLI in gRPC:
* Implement `DevContainerCoding` service:

  * Shells out to CLI with correct environment.
  * Manages repo checkout and branch creation.
  * Opens PR via Git provider API.
    3.3. Configuration & secrets:
* Per‑env config: which repos are allowed, Git tokens, etc.
* Use ExternalSecrets for tokens.
  3.4. Observability:
* Metrics: number of tasks, success rate, duration.
* Logs correlated to Ameide task IDs.

#### Epic 4 – `develop_in_container` tool wiring (agent-developer‑TOOL)

4.1. Tool adapter in Agent runtime runtime:

* Implement LangGraph tool that maps to `DevContainerCoding.DevelopInContainer`.
  4.2. Update AgentDefinition tooling registry:
* Allow “develop_in_container” as a known tool with config reference.
  4.3. Example DAG:
* Implement a reference LangGraph DAG for `core-platform-coder`:

  * Plan → call `develop_in_container` → evaluate → maybe iterate → output PR summary.

#### Epic 5 – Integration with Transformation & Processes (agent-developer‑INTEGRATION)

5.1. Extend Transformation domain:

* Add a “DevTask” / “ControllerImplementationDraft” binding so each coding run is tied to a backlog item.
  5.2. ProcessController hook:
* In the Transformation process (Scrum/TOGAF), add a step that:

  * Calls the coder Agent runtime for certain tasks.
  * Stores PR URL & status back in the Transformation domain.
    5.3. UI wiring:
* In the Transformation workspace, show:

  * “Run via coder agent” button on tasks.
  * Status of agent runs and PRs.

#### Epic 6 – Security & guardrails (agent-developer‑SECURITY)

6.1. Network boundaries:

* NetworkPolicy so:

  * Agent runtime can talk to devcontainer service.
  * devcontainer service cannot talk to random cluster services.
    6.2. Repo whitelisting:
* Only allow specific repos (platform, selected tenant repos) per env.
  6.3. Audit logging:
* Log every `develop_in_container` invocation with:

  * tenant/org/user (if relevant),
  * task/backlog id,
  * repo + branch,
  * result.
    6.4. SRE controls:
* Feature flag / kill‑switch for:

  * the coder Agent runtime,
  * specific tools,
  * specific repos.

---

### 6. Open Questions

You can leave these in the backlog item as discussion points:

1. **Persistent vs ephemeral devcontainers**

   * v1 assumes a long‑lived devcontainer; later we can switch to “one ephemeral pod per task” using a CRD or Job.
2. **Multi‑repo work**

   * Do we need cross‑repo changes in a single run, or is “one repo per task” enough?
3. **LangGraph DAG source of truth**

   * Do we store DAG code in:

     * platform repo (versioned), and just reference it from AgentDefinition,
     * or inline small DAG definitions inside Transformation artifacts?
4. **Claude SDK vs Codex CLI vs A2A**

   * For now, treat “develop_in_container” as an abstraction; implementation could swap between Claude SDK, codex CLI, or an A2A protocol.
