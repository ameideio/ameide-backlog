# 512 – Agent Primitive Scaffolding (Python, opinionated)

> **Status update (520/521/522):** This backlog specifies the Agent scaffold produced by the Ameide CLI (`ameide primitive scaffold`). The consolidated approach is a split: the CLI orchestrates scaffolding + external wiring (repo layout, GitOps), and `buf generate` + plugins handle internal deterministic generation (SDKs, generated-only glue). See `backlog/520-primitives-stack-v2.md`, `backlog/521c-internal-generation-improvements.md`, and `backlog/521d-external-generation-improvements.md`.

**Status:** Active reference (aligned with 520/521/522)  
**Audience:** AI agents, Python developers, CLI implementers  
**Scope:** Exact scaffold shape and patterns for **Agent** primitives. One opinionated Python pattern, aligned with `514-primitive-sdk-isolation.md` (SDK-only, self-contained primitives). No additional Agent-specific CLI parameters beyond the canonical `ameide primitive scaffold` flags.

---

## Primitive/operator alignment

- **Operator responsibilities (495, 500, 497):** The Agent operator reconciles `Agent` CRDs into Deployments, Services, secrets, and model configuration. It owns how agents are scheduled, how runtime secrets are mounted, and how readiness/health are reported, but it does not embed prompt logic, LangGraph DAGs, or SDK call patterns.  
- **Primitive responsibilities (this backlog, 400, 504, 505):** The Agent scaffold owns the **agent behavior**: prompt loading, tools, SDK-based calls into Domain/Process/UISurface primitives, and request/response handling via FastAPI. It runs inside the image referenced by the Agent CRD and is the only place where autonomy, risk tiers, and tool usage are implemented.  
- **Boundary:** The operator provides a stable **runtime envelope** for agents; Agent primitives provide the **agentic logic** inside that envelope. 512 ensures scaffolds never rely on operator internals and that operators never take on agent behavior, matching the contracts described in `495-ameide-operators.md`, `497-operator-implementation-patterns.md`, and `500-agent-operator.md`.

---

## General implementation guidelines (for CLI scaffolder)

- Use a single, opinionated scaffold per primitive kind; **do not add extra CLI parameters** beyond the existing `ameide primitive scaffold` flags.  
- All Agent scaffold instructions must be authored in **template files** (README/templates, comment templates), not hard‑coded as inline strings in the CLI implementation.  
- Scaffolded README and code comments must be **fully self‑contained** and **must not reference backlog IDs**; this backlog informs design, but generated projects must stand alone.  

MCP posture (for Agent scaffolds):

- Do not scaffold MCP servers inside Agent primitives. MCP is a capability-owned Application Interface implemented as an **Integration primitive** (protocol adapter).
- Agent primitives should prefer **Ameide SDK clients** (typed, stable) for internal calls; MCP exists as a compatibility layer for external ecosystems and developer tooling.
- If an agent must run outside the cluster (or be exercised from external tooling), prefer consuming the capability’s MCP adapter rather than embedding bespoke HTTP/JSON clients.

---

## Grounding & cross‑references

- **Agent stack:** `400-agentic-development.md`, `505-agent-developer-v2*.md`.  
- **Agent vertical slice:** `504-agent-vertical-slice.md`.  
- **CLI overview & scaffold:** `484-ameide-cli-overview.md`, `484a-ameide-cli-primitive-workflows.md`, `484f-ameide-cli-scaffold-implementation.md`.  
- **Primitive/operator contract:** `495-ameide-operators.md`, `497-operator-implementation-patterns.md`.  
- **Agent operator:** `500-agent-operator.md`.
- **Testing discipline:** `537-primitive-testing-discipline.md` (RED→GREEN TDD pattern, LangGraph invariants, CI enforcement).
- **Capability implementation DAG:** `backlog/533-capability-implementation-playbook.md` (Agent scaffold + implementation nodes).

---

## 1. Canonical scaffold command (Agent / Python)

Agents use a dedicated Python scaffold with a single recommended pattern:

```bash
ameide primitive scaffold \
  --kind agent \
  --name <name> \
  --include-gitops \
  --include-test-harness
```

This creates an Agent primitive under:

```text
primitives/agent/<name>/
gitops/primitives/agent/<name>/
```

---

## 2. Generated structure (Agent / Python)

```text
primitives/agent/<name>/
├── README.md                            # Wiring checklist, purpose, TODOs
├── catalog-info.yaml                    # Backstage component
├── pyproject.toml                       # Python project with FastAPI, LangGraph, SDK
├── Dockerfile.dev                       # Dev container (hot reload)
├── Dockerfile.release                   # Production build (multi-stage)
├── src/
│   ├── __init__.py                      # Package initializer
│   ├── main.py                          # FastAPI entrypoint (/healthz, /invoke)
│   ├── agent.py                         # AgentPrimitive skeleton (NotImplementedError)
│   ├── prompts/
│   │   └── default.md                   # Default prompt template
│   └── tools/
│       └── __init__.py                  # Tool registration stub
└── tests/
    └── test_agent.py                    # Failing test (RED)
```

GitOps with `--include-gitops`:

```text
gitops/primitives/agent/<name>/
├── values.yaml                          # Deployment, secrets, model config
├── component.yaml                       # Argo CD component descriptor
└── kustomization.yaml                   # Kustomize stub
```

---

## 3. Opinionated agent pattern

Scaffolded Agent primitives assume:

- A **single AgentPrimitive** entrypoint in `src/agent.py` with:
  - tools registration,
  - prompt loading from `src/prompts/default.md`,
  - a `run`/`invoke` method used by `src/main.py`.

- **FastAPI service** in `src/main.py`:
  - exposes `/healthz` and `/invoke` endpoints,
  - delegates to the AgentPrimitive.

- Integration with the Ameide system:
  - Agent uses SDK clients to talk to Domain/Process primitives (never imports proto packages directly for outbound calls).  
  - Agent runtime **does not** invoke `ameide` CLI commands or depend on devcontainer tooling during normal request handling; the CLI remains an out-of-band orchestrator used by humans and coding agents.

### 3.4 LangGraph-native DAG discipline (if using LangGraph)

If the Agent is implemented as a LangGraph DAG:

- **State updates only:** nodes compute state updates; avoid in-place mutation.
- **Reducers required:** every state field must have an explicit reducer (default `REPLACE`), and any field written by parallel branches must have a merge/append reducer.
- **Fan-out is explicit:** use `Send` for map-reduce style fan-out/fan-in; do not rely on “LLM decides to parallelize”.
- **Interrupt safety:** nodes may be re-run after interrupts; treat side-effects (A2A calls, domain intents, integrations) as idempotent operations with dedupe keys derived from `{thread_id, node_id, logical_task_key}`.
- **Streaming:** prefer `stream_mode="updates"` for machine-readable progress; keep token/message streaming opt-in.

---

## 4. Test and wiring semantics (Agent / Python)

Scaffolded tests:

- Live in `tests/test_agent.py`.  
- Are RED by default:
  - instantiate the agent,
  - call a minimal `invoke` path,
  - assert `NotImplementedError` or a TODO response.

Implementers (humans or coding agents) are expected to:

1. Implement the AgentPrimitive’s behavior in `src/agent.py` (tools, prompts, orchestration).  
2. Wire external secrets, models, and SDK clients in `Dockerfile.*` and GitOps.  
3. Replace scaffold test assertions with real expectations (GREEN).

---

## 5. Verification expectations

`ameide primitive verify --kind agent --name <name>` should verify:

- Presence of AgentPrimitive skeleton and FastAPI entrypoint.  
- Tests exist and initially fail.  
- GitOps manifests for the agent are valid (if generated).  
- Linting / packaging checks for `pyproject.toml`.

Behavioral details (delegation, workflows, prompts, and tool semantics) remain in vertical slices such as `504-agent-vertical-slice.md` / `505-agent-developer-v2*.md`. This backlog only constrains the **scaffold shape** for Agent primitives.

---

## 6. Implementation progress (CLI & scaffold)

This section describes the current implementation status of 512 in the CLI (`packages/ameide_core_cli`) and repo scaffolds. It is descriptive; the rest of 512 remains the target spec.

### 6.1 Scaffolder behavior for Agent primitives

**Status:** Largely implemented and template-driven, aligned with the Python/LangGraph/FastAPI shape described in 512.

- `ameide primitive scaffold --kind agent`:
  - Uses a dedicated `scaffoldAgentPrimitive` path in `primitive_scaffold.go`.
  - Creates `primitives/agent/<name>` with:
    - `README.md` (from templates),
    - `catalog-info.yaml`,
    - `pyproject.toml`,
    - `Dockerfile.dev` and `Dockerfile.release`,
    - `src/__init__.py`, `src/main.py`, `src/agent.py`,
    - `src/tools/__init__.py`,
    - `src/prompts/default.md`,
    - `tests/test_agent.py`.
  - Adds GitOps manifests under `gitops/primitives/agent/<name>` when `--include-gitops` is set.
  - Enforces the canonical language choice from 514:
    - `runScaffold` rejects `--lang` values other than `py`/`python` for Agent scaffolds, so Agents are always Python projects.

- Templates vs inline strings:
  - Agent scaffolds are driven by `templates/agent/*.tmpl` via `templates_agent.go`:
    - README, pyproject, package init, tools init, prompt content, and parts of the Dockerfiles come from templates.
  - This satisfies 512’s requirement that Agent scaffold instructions live in templates, not inline CLI strings.
  - `templates_agent_test.go` includes `TestAgentReadmeTemplateHasNoBacklogIds`, which renders the Agent README template and fails if any `NNN-*.md` backlog references appear, keeping scaffolded docs self-contained.

### 6.2 Runtime and SDK expectations

**Status:** Partially aligned with 514; SDK-only and “no CLI at runtime” are documented but not fully enforced.

- Scaffolded runtime (`src/main.py`, `src/agent.py`):
  - Exposes FastAPI `/healthz` and `/invoke` endpoints.
  - Defines an `AgentPrimitive` class stub with TODOs for behavior, tools, and prompts.
  - Does not embed `ameide` CLI calls into request handling — consistent with 514’s CLI-out-of-band rule.
- SDK usage:
  - The current scaffold hints at SDK usage conceptually, but:
    - It does **not** yet wire concrete Ameide SDK Python clients by default.
    - There is no Agent-specific import policy in `primitive verify` that enforces “no direct core-proto imports” for Python.

### 6.3 Verify behavior for Agent primitives

**Status:** Basic shape checks only; no 512-specific semantics yet.

- `primitive verify --kind agent --name <name>`:
  - Inherits generic checks:
    - Naming, security/SAST/secret scan, tests, shared `Imports` policy, and optional GitOps checks.
  - Does **not** currently:
    - Confirm presence of specific AgentPrimitive methods/tools beyond the scaffold files existing.
    - Enforce a full SDK wiring pattern (it only guards against obvious proto/core-proto and cross-primitive imports in Python runtime code).
    - Validate AgentDefinition wiring or risk-tier metadata; these remain in the 504/505 vertical slices and Backstage metadata.

### 6.4 Known gaps and next steps

- Scaffold:
  - Provide clearer SDK wiring in `src/agent.py` or `src/tools/__init__.py` (e.g., example Ameide SDK client usage).
  - Ensure README template references SDK usage and AgentDefinition wiring rather than CLI guardrail loops.
- Verify:
  - Introduce a Python import check similar to Domain’s Go import policy:
    - Fail when runtime Agent code imports `packages/ameide_core_proto` or other proto modules directly.
    - Optionally warn on imports of non-whitelisted workspace packages from primitives.
