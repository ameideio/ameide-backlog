# 504 – Agent Vertical Slice: Domain + CLI + Operator

**Status:** Implemented (operator + prompt assets landed, CLI verify/scaffold guardrails plus repo-root/gitops-root plumbing shipped)  
**Audience:** Platform engineers, AI agents implementing Agent primitives  
**Scope:** Deliver an Agent primitive end-to-end (operator, CLI, GitOps, sample CR, prompt workflow)

**Authority & supersession**

- This backlog is **authoritative for the Agent vertical slice implementation**: Agent CRD/operator wiring, CLI guardrails, and demo assets for a generic Agent primitive.  
- For the **AmeidePO/AmeideSA/AmeideCoder split and runtime roles**, defer to `505-agent-developer-v2.md` and `477-primitive-stack.md`; this file describes the toolchain that AmeideCoder (`runtime_role=a2a_server`) uses internally.  
- Operator responsibilities shared across primitives are described in `495-ameide-operators.md` and the Domain slice (`502-domain-vertical-slice.md`); this backlog is expected to stay in lockstep with those documents’ condition vocabulary and status snapshot tables.  
- Any historical patterns that imply PO/SA run CLI commands directly are superseded by the v2 architecture: **only AmeideCoder’s devcontainer runtime runs these CLI workflows**.

**Contract surfaces (owned elsewhere, referenced here)**

- Agent runtime roles and A2A contracts are defined in `505-agent-developer-v2.md` and enforced by the Agent operator backlog (`500-agent-operator.md`).  
- CLI proto contracts and JSON output shapes are defined in `484b-ameide-cli-proto-contract.md`.  
- Shared condition vocabulary used by `primitive verify`/`primitive describe` is defined in `502-domain-vertical-slice.md` and `495-ameide-operators.md`.  
- Stage 0/1/2/3 mapping for Scrum and the placement of AmeidePO/AmeideSA/AmeideCoder in that stack are described in `507-scrum-agent-map.md`; this backlog focuses only on the Agent slice from that map.

## Grounding & cross-references

- **Architecture grounding:** Follows the primitive stack and operator patterns from `470-ameide-vision.md`, `471-ameide-business-architecture.md`, `472-ameide-information-application.md`, `473-ameide-technology.md`, `475-ameide-domains.md`, `477-primitive-stack.md`, `495-ameide-operators.md`, and the vertical-slice pattern in `502-domain-vertical-slice.md`.  
- **Runtime relationships:** Assumes Agent CRDs and operator behavior from `500-agent-operator.md` and Helm/GitOps deployment via `503-operators-helm-chart.md`; UISurface integration patterns (for any frontends that drive agents) follow `501-uisurface-operator.md`.  
- **Scrum stack alignment:** Provides the tooling slice that AmeideCoder (runtime_role=`a2a_server`) uses inside the Scrum/Process stack defined in `505-agent-developer-v2.md`, `505-agent-developer-v2-implementation.md`, `506-scrum-vertical-v2.md`, and `508-scrum-protos.md`, as mapped in `507-scrum-agent-map.md`.  
- **CLI & A2A dependencies:** Depends on CLI contract backlogs `484a-ameide-cli-primitive-workflows.md` … `484f` and the A2A binding described in `505-agent-developer-v2.md`; agents treat the CLI workflows here as internal tools, not customer-facing UIs.

> **Related**
> - [495-ameide-operators.md](495-ameide-operators.md) – shared CRD principles
> - [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) – controller-runtime patterns
> - [502-domain-vertical-slice.md](502-domain-vertical-slice.md) – reference format
> - [505-agent-developer-v2.md](505-agent-developer-v2.md) – Process + AmeidePO/AmeideSA/AmeideCoder architecture
> - [507-scrum-agent-map.md](507-scrum-agent-map.md) – cross-layer map (Stage 0–3) this slice plugs into
> - [484a-ameide-cli-primitive-workflows.md](484a-ameide-cli-primitive-workflows.md) – CLI guardrails
> - [484b-ameide-cli-proto-contract.md](484b-ameide-cli-proto-contract.md) – proto schemas

---

## 0. Status snapshot (repo state)

| Track/Phase | State | Notes |
|-------------|-------|-------|
| **Operator A (CRD Types)** | ✅ | Types/conditions defined in `operators/agent-operator/api/v1/agent_types.go`. |
| **Operator B (Controller skeleton)** | ✅ | `internal/controller/agent_controller.go` implements the reconcile loop/finalizer. |
| **Operator C/C2 (runtime + secrets/tool grants)** | ⚠️ Partial | Controller only creates a Deployment. Service/HPA, ExternalSecrets, and prompt/tool ConfigMaps are stubbed; spec fields like `resources`, `observability.logPrompts`, `security.serviceAccountName`, and rollout strategy are ignored. |
| **Operator D (Status & policy)** | ⚠️ Partial | Conditions update, but policy validation is limited to a single risk-tier rule and Ready is computed even when runtime artifacts are missing. |
| **Operator E (Helm chart)** | ✅ | Helm + GitOps charts now ship the regenerated Agent CRD/RBAC (`operators/helm/crds/ameide.io_agents.yaml`, `gitops/.../platform/ameide-operators/crds/ameide.io_agents.yaml`). |
| **Operator F/G (GitOps + sample)** | ⚠️ Blocked | Sample CR + ApplicationSet wiring exist under `gitops/ameide-gitops/...`, yet they cannot apply until the Helm CRD is refreshed and RBAC allows ConfigMap writes. |
| **CLI H (Proto)** | ✅ | `packages/ameide_core_proto/.../primitive_types.proto` includes `AgentDetails` + `AgentToolGrant`. |
| **CLI I (Commands)** | ✅ | `describe`/`verify` surface agent data, accept `--repo-root/--gitops-root/--mode`, and `verify --mode all` runs naming/security/EDA/test heuristics. |
| **CLI J (Prompt)** | ✅ | `primitive_prompt.go` + `prompts/agent/default.md` render onboarding text aligned with the new `verify --mode all --repo-root --gitops-root` guardrail workflow. |
| **CLI K (Scaffold)** | ✅ | `primitive_scaffold.go` scaffolds Domain and Agent (Go) services with README/tests/GitOps stubs. |
| **CLI L (Tests)** | ⚠️ Partial | Describe/prompt conversion tests exist, but no agent-specific verify/scaffold coverage. |
| **Prompt Profiles (P)** | ⚠️ Partial | Default profile exists; additional profiles + profile selection logic remain TODO. |

> Update this table alongside 502 to keep intent vs. implementation in sync each sprint.

> **V2 alignment:** In the Process + AmeidePO + AmeideSA + AmeideCoder architecture (505-agent-developer-v2.md), everything in this backlog describes the **internal toolchain of AmeideCoder**. PO/SA agents never invoke these CLI commands directly; they rely on AmeideCoder's A2A server to run the OBSERVE→REASON→ACT→VERIFY loop documented here.

---

## 1. Why an Agent Vertical Slice?

Agent primitives are the foundation for AI-driven workflows. An Agent CR describes:
- Which LLM/runtime image to run
- Allowed tools (Domain/Process/UISurface access)
- Risk tier & policy (prompt logging, concurrency)
- Prompt templates / system instructions

This slice ensures we can:
1. Declare an Agent via GitOps
2. Reconcile it into a running container (with secrets, tool grants)
3. Read status via the CLI (`describe`, `verify`, `drift`, `plan`, `impact`, `prompt`, `scaffold`) to keep agents on the OBSERVE→ACT loop
4. Provide starting prompts so new agents can bootstrap themselves following the guardrail workflow

Deliverable: End-to-end demo – create an Agent CR, operator provisions secrets + deployment, CLI shows status, prompt file boots an agent with zero prior context.

---

## 2. Scope & Phases

The slice mirrors the dual-track pattern from 502 (Operator + CLI). Phases labeled A–G for operator, H–L for CLI, P for Prompt Profiles.

### Operator Track

| Phase | Focus | Files | Acceptance Criteria |
|-------|-------|-------|---------------------|
| **A** | CRD Types | `operators/agent-operator/api/v1/*.go` | `make manifests` yields CRD |
| **B** | Reconciler Skeleton | `internal/controller/agent_controller.go` | Controller runs, logs reconcile |
| **C** | Runtime (Deployment/Service) | `reconcile_runtime.go` | Agent Pod + Service created |
| **C2** | Secrets/Tool Grants | `reconcile_secrets.go` | API keys pulled via ExternalSecrets; tool config ConfigMap |
| **D** | Status/Conditions + Policy | `conditions.go`, `policy.go` | Conditions cover Ready, DefinitionResolved, SecretsReady, ToolingReady, PolicyCompliant; risk tier/model/tool policy enforced |
| **E** | Helm Chart | `operators/helm/` | `helm template` renders agent operator pieces |
| **F** | GitOps | `gitops/.../agent-operator` | ArgoCD applies operator cluster-wide |
| **G** | Sample CR | `gitops/.../primitives/agent/...` | Sample agent reconciles end-to-end |

### CLI Track

| Phase | Focus | Files | Acceptance Criteria |
|-------|-------|-------|---------------------|
| **H** | Proto Messages | `ameide_core_proto/primitive/v1/*.proto` | Agent-specific details + statuses |
| **I** | CLI Commands | `packages/ameide_core_cli/...` | describe/verify/drift/plan/impact handle Agents |
| **J** | Prompt Command | `primitive_prompt.go`, `prompts/agent/default.md` | CLI emits onboarding prompt |
| **K** | Scaffold Enhancements | `primitive_scaffold.go` | Agent skeleton (Go + prompts) |
| **L** | Tests | `*_test.go` | `go test ./packages/ameide_core_cli/...` covers Agent cases |
| **P** | Prompt Profiles | `prompts/agent/*.md` | Start-without-context template available |

The CLI phases inherit the guardrails from the 484 suite and the Vision invariants (470 §0):
- **TDD + one-shot scaffolding (484a §2, 484e §2):** `primitive scaffold` for Agents must be conservative, generate failing tests that call real handlers, never overwrite existing files, and leave business logic/prompt meaning to agents.
- **Repo-root vs GitOps-root handling (484c §1-4):** All repo-mutating commands accept `--repo-root`, and `primitive verify` now understands `--repo-root`, `--gitops-root`, and `--mode` to mix repo + cluster checks. `describe` still defaults to the current directory and remains on the backlog.
- **Verify security/EDA hooks (470 §8-13, 484a §3):** `primitive verify --mode all` now layers RPC naming, security heuristics, go test execution, and EDA checks atop cluster health. Follow-on work will replace the heuristics with the Buf/Govuln/gitleaks/SAST toolchain.

### Sample Assets

- **Operator/Helm example:** `operators/helm/examples/agent-sample.yaml` shows a fully-populated Agent CR (`definitionRef`, tool grants, observability, risk tier) that can be rendered via `helm template`.
- **GitOps sample:** `gitops/ameide-gitops/environments/_shared/primitives/agent/core-platform-coder.yaml` represents the same Agent primitive checked into the GitOps repo so ApplicationSets can promote it through dev/staging/prod without ad-hoc manifests.
- **ApplicationSet wiring:** `environments/_shared/components/apps/primitives/core-platform-coder/component.yaml` feeds the sample CR (via `_shared/platform/agents/core-platform-coder.yaml`) into the global ApplicationSet, so every environment deploys the Agent once operators are present.
- **Demo script:** `scripts/demo-agent-slice.sh` uses the Helm sample, waits for `Ready`/`PolicyCompliant`, then runs `describe`/`verify`/`prompt` to document the OBSERVE→ACT guardrail loop.

---

## 3. Agent CRD Overview

```yaml
apiVersion: ameide.io/v1
kind: Agent
metadata:
  name: core-platform-coder
spec:
  image: ghcr.io/ameide/agent-langchain:1.0.0
  definitionRef:
    id: core-platform-coder-v4      # Stored in Transformation Domain
    tenantId: t123                 # Transformation design tooling writes this
  # runtimeRole was introduced with the v2 agent architecture:
  # - product_owner
  # - solution_architect
  # - a2a_server (AmeideCoder devcontainer)
  runtimeRole: a2a_server
  tools:
    domains: ["transformation", "platform"]
    processes: ["l2o"]
    custom:
      - name: github
        secretRef: platform/github-token
  model:
    provider: openai
    name: gpt-4o
    temperature: 0.3
  riskTier: medium
  observability:
    logPrompts: redacted
    emitTokens: true
status:
  conditions:
    - type: Ready
    - type: DefinitionResolved
    - type: SecretsReady
    - type: ToolingReady
    - type: PolicyCompliant
```

Key responsibilities:
- Resolve the `definitionRef` from the Transformation Domain (via design tooling APIs)
- Mount LLM secrets (ExternalSecrets)
- Generate prompt config (ConfigMap) referencing prompt profiles + requirement context
- Generate tool grants (ConfigMap/Secret)
- Enforce risk tier + tool/model policy before permitting runtime changes
- Run Deployment with concurrency controls (HPA optional)
- Report conditions for CLI consumption (`Ready`, `DefinitionResolved`, `SecretsReady`, `ToolingReady`, `PolicyCompliant`, `Degraded`)

> **V2 alignment:** In the Process + AmeidePO + AmeideSA + AmeideCoder architecture (505-agent-developer-v2.md), CRDs with `runtimeRole=a2a_server` represent AmeideCoder instances that expose the A2A REST binding (`/v1/message:send`, `/v1/message:stream`, `/v1/tasks/*`). PO/SA agents are separate Agent CRs with `runtimeRole=product_owner` / `solution_architect` and outbound-only HTTP; they never run the CLI directly and instead delegate to AmeideCoder.

---

## 4. Prompt Profiles

**Goal**: Provide agents zero-context starting points. Each profile lives under `prompts/agent/<profile>.md`.

Structure:
1. **Requirement** – Temporary hard-coded scenario (e.g., “Implement Domain scaffolder extensions”)
2. **Workflow** – Step-by-step instructions referencing CLI commands (describe, drift, plan, impact, scaffold, verify, prompt)
3. **Documentation references** – Inline summary (no dependency on backlog files)
4. **Output expectations** – e.g., “Provide RED→GREEN→REFACTOR updates; cite CLI results”

CLI wiring: `ameide primitive prompt --profile default [--requirement ...]` reads the template and prints it. The workflow now references `ameide primitive verify --mode all --repo-root … --gitops-root …` so both repo and cluster guardrails stay green.

Future: integrate with transformation domain to fetch dynamic requirements.

### 4.1 Local Telepresence/Swap Flow (Optional)

During development, you can “be” the agent by swapping its pod with your local environment (Telepresence/ksync or `kubectl exec` + VS Code remote session):
- Deploy the Agent CR in a dev cluster.
- Use Telepresence (or similar) to intercept the agent Deployment, running your local CLI + repo inside the cluster namespace.
- Load the prompt profile via `ameide primitive prompt` and follow the OBSERVE→REASON→ACT→VERIFY flow (describe/drift/plan/scaffold/verify).
- This ensures the same guardrails apply whether a human/AI is controlling the pod locally or it’s running autonomously in-cluster.

This pattern becomes even more valuable when the Agent primitive is fully autonomous: the same prompt/CLI workflow can bootstrap or debug live agents.

---

## 5. Implementation Plan

1. **CRD & Operator (Phases A–D)**  
   - Start from operator/shared scaffolding (497 patterns).  
   - Define `AgentSpec` with `definitionRef` pointing to Transformation Domain IDs (no inline definitions).  
   - Implement `reconcileDefinition`, `reconcileSecrets`, `reconcileTooling`, `reconcileDeployment`.  
   - Enforce risk tier/model/tool policy (`PolicyCompliant` condition) before rolling out.  
   - Conditions: `Ready`, `DefinitionResolved`, `SecretsReady`, `ToolingReady`, `PolicyCompliant`, `Degraded`.

2. **Helm/GitOps/Sample (Phases E–G)**  
   - Extend existing Helm chart with agent operator deployment.  
   - Add GitOps component at rolloutPhase 020 (cluster).  
   - Provide sample CR in `operators/helm/examples/agent-sample.yaml` and GitOps repo.

3. **CLI Support (Phases H–L, P)**  
   - Proto: extend `PrimitiveInfo` with `AgentDetails` (tool counts, prompt profile, model).  
   - CLI: ensure `describe`, `verify`, `drift`, `plan`, `impact`, `scaffold`, `prompt` handle Agents.  
   - Prompt command: load template file, allow overrides.  
   - Scaffold: optionally generate agent runtime skeleton (Go entrypoint + prompt files).  
   - Tests covering new flows.

4. **End-to-End Demo (Phase M equivalent)**  
   - Apply sample Agent CR  
   - Verify secrets + tooling + deployment  
   - Run CLI commands (describe → drift → plan → scaffold → verify)  
   - Use prompt profile to bootstrap an agent and demonstrate log capture

---

## 6. Acceptance Criteria Summary

| Track | Criteria |
|-------|----------|
| Operator | Agent CR applied → operator resolves `definitionRef`, provisions secrets/ConfigMaps/Deployment, enforces policy. `status.conditions` show Ready + DefinitionResolved + SecretsReady + ToolingReady + PolicyCompliant. |
| Helm/GitOps | Agent operator deployable via existing ApplicationSet. Sample CR in GitOps dev environment. |
| CLI | Describe/verify/drift/plan/impact surface Agent data; `prompt` loads template; scaffold **still Domain/Go only (Agent support TODO).** |
| Prompt Profiles | `prompts/agent/default.md` (hard-coded requirement) exists and matches CLI output. |
| Tests | `go test ./packages/ameide_core_cli/...` and operator unit tests pass. |
| Demo | Documented workflow replicating OBSERVE → ACT loop for Agent requirement. |

---

## 8. End-to-End Demo Instructions

Use `scripts/demo-agent-slice.sh` to exercise the entire slice against a cluster with the agent-operator deployed:

```bash
./scripts/demo-agent-slice.sh --namespace primitives-dev
```

The script:
1. Applies `operators/helm/examples/agent-sample.yaml` (same spec synced via GitOps).
2. Waits for `definitionResolved`, `secrets/tooling ready`, `policyCompliant`, and `Ready`.
3. Runs `ameide primitive describe/verify/prompt` so the OBSERVE → REASON → ACT → VERIFY guardrail output is captured.
4. Cleans up the Agent CR unless `--keep` is provided.

Run with `--sample <path>` if you need to point at an environment-specific manifest.

---

## 7. Next Steps

1. Implement CRD + operator phases (reuse domain operator structure but focus on secrets/tooling instead of DB).  
2. Extend Helm chart and GitOps repo, publish sample Agent CR.  
3. Update CLI proto + commands, scaffolding, prompt profile loader.  
4. Add prompt template(s) under `prompts/agent/`.  
5. Record/demo the end-to-end flow in docs.

This document will evolve similarly to 502 as we move from draft → implemented.

---

## 9. Known Gaps (keep docs + implementation aligned)

- **Describe still assumes repo cwd** – `primitive describe` has not picked up the new `--repo-root/--gitops-root` plumbing, so agents must invoke it from the core repo. Align it with `verify`.
- **Verify heuristics vs toolchain** – The new naming/security/EDA/test checks rely on lightweight heuristics. Swap them for the full Buf/gitleaks/govuln/Semgrep stack so agents get authoritative signals.
