# 500 – Agent Operator

**Status:** Active (Phase 1 Complete)
**Audience:** Platform engineers implementing Agent operator
**Scope:** Implementation tracking, development phases, acceptance criteria

**Authority & supersession**

- This backlog is **authoritative for the Agent operator control plane**: the `Agent` CRD shape, reconciliation flow, and how Agent runtimes (including AmeidePO/AmeideSA/AmeideCoder) are wired/deployed.  
- **Agent behavior and A2A contracts** live in `505-agent-developer-v2.md` and its implementation plan; this file must enforce the runtime roles and risk tiers those backlogs define.  
- **CLI/guardrail behavior** for AmeideCoder is documented in `504-agent-vertical-slice.md` and the 484a–484f series; this operator only wires those tools, it does not implement them.  
- If any older “coder agent” framing implies that LangGraph agents run Ameide CLI directly, prefer the v2 split: **PO/SA as A2A clients, AmeideCoder as `runtime_role=a2a_server` devcontainer**.

**Contract surfaces (owned elsewhere, referenced here)**

- A2A REST binding (`/.well-known/agent-card.json`, `/v1/message:send`, `/v1/message:stream`, `/v1/tasks/*`) and Agent Card schema are defined in the A2A proto/backlog and 505-v2.  
- Runtime roles (`product_owner`, `solution_architect`, `a2a_server`) are defined in 505-v2 and 477-primitive-stack; this operator enforces them via spec validation and, over time, explicit CRD fields such as `spec.runtimeRole` and `spec.a2a` so GitOps manifests can declare the intended role and A2A surface declaratively.  
- Shared operator condition types and vocabulary are defined in `operators/shared/api/v1/conditions.go` and described in 495/498/502.
- Capability tool identifiers (MCP-compatible): capabilities expose tools/resources under stable identifiers like `<capability>.<operation>` (e.g., `transformation.listElements`) and (optionally) bind them to MCP via Integration primitives; the tool catalog is proto-first and default-deny. See `backlog/534-mcp-protocol-adapter.md`.

## Grounding & cross-references

- **Architecture grounding:** Implements the Agent primitive described by the vision/primitive backlogs (`470-ameide-vision.md`, `471-ameide-business-architecture.md`, `472-ameide-information-application.md`, `473-ameide-technology.md`, `475-ameide-domains.md`, `477-primitive-stack.md`) and the operator/EDA guidelines in `495-ameide-operators.md`, `496-eda-principles.md`, and `497-operator-implementation-patterns.md`.  
- **Operator ecosystem:** Works alongside the Domain, Process, and UISurface operators tracked in `498-domain-operator.md`, `499-process-operator.md`, and `501-uisurface-operator.md`, and is packaged via the unified Helm chart in `503-operators-helm-chart.md`; all share the condition vocabulary and vertical-slice expectations set by `502-domain-vertical-slice.md`.  
- **Scrum stack alignment:** Hosts the AmeidePO/AmeideSA/AmeideCoder Agent primitives defined in `505-agent-developer-v2.md` and implemented via `505-agent-developer-v2-implementation.md`, wiring them into the Scrum stack that uses `506-scrum-vertical-v2.md` and `508-scrum-protos.md` as canonical domain/process contracts (see `507-scrum-agent-map.md`).  
- **CLI & A2A dependencies:** Exposes runtimes that consume the CLI/tooling slice from `504-agent-vertical-slice.md` and the A2A REST binding defined in the 505-v2/A2A backlogs; `ameide primitive` commands from the 484a–484f series are used inside Coder runtimes, not embedded into this operator.
- **Capability implementation DAG:** `backlog/533-capability-implementation-playbook.md` (context for end-to-end delivery workflows; this operator provides readiness conditions for agent nodes).

> **Related**:
> - [495-ameide-operators.md](495-ameide-operators.md) – CRD shapes & responsibilities
> - [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) – Go patterns & reference implementation

---

## 1. Overview

The Agent operator manages the lifecycle of **Agent primitives** – LLM-powered autonomous actors with tool access. Each `Agent` CR results in:

- A **definition reference** recorded for the runtime (ConfigMap + env vars; runtime fetches the definition)
- ExternalSecrets for LLM API keys and tool credentials
- Tool grants ConfigMap (allowed tools per security policy)
- Agent runtime Deployment (LangGraph, custom framework, etc.)

**Key insight**: The operator wires secrets + policy; the Agent image implements prompt orchestration and tool execution.

### 1.1 Condition semantics: “Ready means ready”

Target behavior is that `Ready` must be **impossible** unless all required runtime artifacts exist and are policy-compliant:

- `DefinitionResolved` (definitionRef recorded and available to the runtime)
- `SecretsReady` (all required ExternalSecrets synced; mountable)
- `ToolingReady` / `DependenciesReady` (tool grants + prompt/profile/config are reconciled and available to the workload)
- `PolicyCompliant` (runtimeRole/riskTier/model/tools validated)
- `WorkloadReady` (Deployment available)
- `RouteReady` (if the Agent exposes an ingress surface)

Rule:

- `Ready = AND(DefinitionResolved, SecretsReady, ToolingReady, PolicyCompliant, WorkloadReady, RouteReady*)`

`RouteReady` is optional only when the agent is not meant to be reachable (e.g., outbound-only agents).

> **505 alignment:** The Process + AmeidePO + AmeideSA + AmeideCoder architecture in [505-agent-developer-v2.md](505-agent-developer-v2.md) introduces an explicit split between A2A clients (PO/SA) and an A2A server (Coder). This backlog now tracks the operator work required to host all three Agent primitives plus the devcontainer runtime surfaces they depend on.

---

## 2. CRD Summary

```yaml
apiVersion: ameide.io/v1
kind: Agent
metadata:
  name: core-platform-coder
spec:
  image: ghcr.io/ameide/agent-runtime-langgraph:2.1.0
  definitionRef:
    id: core-platform-coder-v4
    tenantId: t123
  model:
    provider: openai
    name: gpt-5.1-pro
  tools:
    allow:
      - transformation.listElements
      - transformation.getView
      - transformation.submitScrumIntent
      - transformation.submitArchitectureIntent
      - ameide.develop_in_container
    deny:
      - transformation.submit* # example: read-only policy (not for coder)
  riskTier: high
  concurrency:
    maxParallelSessions: 20
  observability:
    logPrompts: redacted
    emitTokens: true
status:
  conditions:
    - type: Ready
    - type: DefinitionResolved
    - type: SecretsReady
    - type: RuntimeReady
    - type: PolicyCompliant
  observedGeneration: 3
```

### 2.1 Tool identifiers (proto-first, MCP-compatible)

This operator treats “tools” as **globally namespaced identifiers**:

- Capability tools: `<capability>.<operation>` (e.g., `transformation.listElements`, `sre.searchIncidents`)
- Platform/internal tools: `ameide.<operation>` (e.g., `ameide.develop_in_container`)

`spec.tools.allow`/`spec.tools.deny`:

- Support exact matches and glob patterns (e.g., `transformation.submit*`).
- Are validated against the **proto-exposed tool catalog** (default deny; only explicitly exposed tools exist), not against ad-hoc operator-defined semantics.
- Produce an effective allowlist for the agent runtime (written to the tool grants ConfigMap), optionally intersected with the resolved AgentDefinition allowlist.

---

## 3. Reconcile Flow

```
1. Fetch Agent CR
2. Handle deletion (finalizer cleanup – revoke tool grants)
3. Ensure finalizer present
4. Validate spec (definitionRef, model/provider allowed, riskTier + tools policy)
5. reconcileSecrets()  ← ExternalSecrets for LLM keys, tool creds
6. reconcileToolGrants() ← ConfigMap with allowed tools
7. reconcileRuntime()  ← Agent Deployment with secret mounts
8. Update status (conditions, observedGeneration)
```

### 3.1 LangGraph coder runtime (agent-developer linkage)

Backlog [505-agent-developer-v2.md](505-agent-developer-v2.md) cements the Process + AmeidePO + AmeideSA + AmeideCoder split. It introduces `runtime_type=langgraph`, `dag_ref`, the `develop_in_container` compatibility tool, and a new `runtime_role` concept (`product_owner`, `solution_architect`, `a2a_server`). Implications for the operator:

* **Spec propagation.** When the Transformation domain resolves an AgentDefinition that includes `runtime_type=langgraph`, the operator must surface the referenced DAG module/image to the runtime Deployment (env vars or ConfigMap). This keeps the runtime controller generic while enabling LangGraph to build the DAG defined in Transformation.
* **Tool grants.** `develop_in_container` is treated like any other tool grant, but it is automatically `riskTier=high`. The existing policy validation (Phase 3) should reject Agent CRs that request `develop_in_container` without `spec.riskTier=high` or without the platform-approved devcontainer service configuration.
* **Runtime service wiring.** For LangGraph coder agents, `reconcileRuntime()` needs to include Service annotations or env vars that point at the devcontainer gRPC endpoint deployed via GitOps. Document this under Phase 3/4 artifacts so the ApplicationSet that deploys the devcontainer service stays in sync with the Agent operator rollout described in backlog 504.
* **A2A roles.** AmeidePO and AmeideSA instances only require outbound HTTP (A2A client) plus access to Process/Transformation APIs. AmeideCoder instances must expose the standardized REST binding (`/v1/message:send`, `/v1/message:stream`, `/v1/tasks/*`) and publish an Agent Card at `/.well-known/agent-card.json`. Capture these requirements via annotations or an explicit `spec.runtimeRole`, and enforce that:
  * `runtimeRole=product_owner` / `solution_architect` **cannot** request high‑risk tools such as `develop_in_container` or direct repo CLI grants and typically run with `riskTier=medium`.  
  * `runtimeRole=a2a_server` (AmeideCoder devcontainer) **must** expose the A2A REST binding and run with `riskTier=high`, as it owns repo write/PR capabilities and internal CLI usage.
* **Process metadata.** AmeideSA AgentDefinitions reference Process lifecycle metadata (sprint/phase ids) when delegating to AmeideCoder. Propagate `process_scope`/`timebox_id` fields from the definition ConfigMap into the runtime environment so SA pods can correlate EDA events without hardcoding cluster config.

**Implementation status:** `internal/controller/agent_controller.go` now calls Transformation via the Go SDK, writes the resolved AgentDefinition (including `runtime_type`, `dag_ref`, `scope`, and `risk_tier`) into the definition ConfigMap as `definition.json`, and rejects LangGraph tool requests that violate the risk-tier guardrails (`sharedv1.ConditionDefinitionResolved`, `ConditionPolicyCompliant`). The operator Deployment exposes env vars for the Transformation address/token so GitOps overlays can point at the right service endpoint.

---

## 4. CRD Types (Go)

```go
type AgentSpec struct {
    Image         string             `json:"image"`
    DefinitionRef AgentDefinitionRef `json:"definitionRef"`
    Model         AgentModelConfig   `json:"model"`
    Tools         AgentToolsConfig   `json:"tools"`
    RiskTier      string             `json:"riskTier"` // low, medium, high
    Concurrency   AgentConcurrency   `json:"concurrency,omitempty"`
    Observability AgentObservability `json:"observability,omitempty"`
}

type AgentDefinitionRef struct {
    ID       string `json:"id"`
    TenantID string `json:"tenantId"`
}

type AgentModelConfig struct {
    Provider string `json:"provider"` // openai, anthropic, azure, etc.
    Name     string `json:"name"`     // gpt-4, claude-3, etc.
}

type AgentToolsConfig struct {
    Allow []string `json:"allow,omitempty"`
    Deny  []string `json:"deny,omitempty"`
}

type AgentStatus struct {
    Conditions         []metav1.Condition `json:"conditions,omitempty"`
    ObservedGeneration int64              `json:"observedGeneration,omitempty"`
}
```

---

## 5. Development Phases

### Phase 1: Core Reconciliation ✅ IMPLEMENTED

> **Implementation**: See [operators/agent-operator/](../operators/agent-operator/)

| Task | Description | Status |
|------|-------------|--------|
| **CRD types** | Define `Agent`, `AgentSpec`, `AgentStatus` | ✅ `api/v1/agent_types.go` |
| **Basic reconciler** | 7-step reconcile skeleton | ✅ `internal/controller/agent_controller.go` |
| **Finalizer handling** | Add/remove finalizer | ✅ Implemented |
| **Runtime Deployment** | Create Deployment for agent runtime | ✅ `internal/controller/reconcile_workload.go` |
| **Status conditions** | Set `Ready`, `RuntimeReady` | ✅ `internal/controller/conditions.go` |

**Helm chart**: Operators deployable via unified chart at `operators/helm/`

### Phase 2: Security & Secrets

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **ExternalSecret for LLM** | Create ExternalSecret for model provider API key | Secret synced from vault |
| **ExternalSecret for tools** | Create ExternalSecrets for tool credentials (GitHub, Jira, etc.) | Tool secrets available |
| **Secret mounting** | Mount secrets into agent Deployment | Container has `LLM_API_KEY` etc. |
| **SecretsReady condition** | Check all ExternalSecrets synced | Condition reflects secret status |
| **Secret cleanup** | Delete ExternalSecrets on Agent delete | No orphan secrets |

### Phase 3: Policy & Tool Grants

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **Policy validation** | Check riskTier + tools against security policy | `PolicyCompliant=False` if violation |
| **Tool grants ConfigMap** | Create ConfigMap listing allowed tools | ConfigMap contains tool list |
| **Model/provider validation** | Check model allowed for tenant/SKU | Error if model not permitted |
| **Risk tier enforcement** | Some tools only available for `riskTier=high` | Validation fails for low-risk agents with high-risk tools |
| **Tool grant revocation** | On delete, call tool grants service to revoke | Clean revocation in central registry |

### Phase 4: Transformation Integration

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **DefinitionRef recording** | ✅ Record `spec.definitionRef` for the runtime | Operator creates/updates a ConfigMap (e.g. `*-definition`) containing `definitionId`, `tenantId`, optional `version`, plus `definition.json` (serialized reference). |
| **DefinitionResolved condition** | ✅ Set once the reference is recorded | `DefinitionResolved=True` means “runtime has enough info to fetch/resolve the definition”, not “operator fetched it from Transformation”. |
| **Transformation prefetch (optional)** | ⏳ Fetch/validate AgentDefinition in-controller | When implemented, failures should set `DefinitionResolved=False` with a stable reason; until then, runtime owns fetch/validation. |

### Phase 5: Production Readiness

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **HPA based on queue depth** | Scale agents based on session queue | HPA triggers on custom metric |
| **Concurrency limits** | Configure max parallel sessions | Agent respects concurrency limits |
| **Prompt logging** | Configure redacted/full logging | Logs respect `observability.logPrompts` |
| **Token metrics** | Emit token usage metrics | Prometheus has token counts |
| **Audit events** | Emit K8s events for security-relevant actions | Events visible in `kubectl describe` |

### Phase 6: Testing & Validation

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| **Unit tests** | Fake clients for all external services | Tests pass without real services |
| **envtest tests** | Create Agent, verify Deployment | Integration tests pass |
| **Policy tests** | Test riskTier/tool policy combinations | Policy violations caught |
| **CLI integration** | `ameide primitive verify` checks Agent | CLI reports Agent health |

---

## 6. Key Implementation Details

### 6.1 Secrets Flow

```
Agent CR with model.provider=openai, riskTier=high
        ↓
reconcileSecrets() creates ExternalSecrets:
  - openai-api-key (from vault path per provider)
  - github-token (if `spec.tools.allow` includes `github.*` tool identifiers)
  - jira-token (if `spec.tools.allow` includes `jira.*` tool identifiers)
        ↓
External Secrets Operator syncs actual Secrets
        ↓
Agent Deployment mounts secrets as env vars
```

### 6.2 Policy Validation

```go
func (r *AgentReconciler) validatePolicy(agent *amv1.Agent) error {
    policy := r.SecurityPolicy // loaded from ConfigMap or CRD

    // Check model allowed for tenant
    if !policy.IsModelAllowed(agent.Spec.Model, agent.Spec.DefinitionRef.TenantID) {
        return fmt.Errorf("model %s not allowed for tenant", agent.Spec.Model.Name)
    }

    // Check tools vs riskTier
    for _, tool := range agent.Spec.Tools.Allowed {
        requiredTier := policy.ToolRiskTier(tool)
        if !isRiskTierSufficient(agent.Spec.RiskTier, requiredTier) {
            return fmt.Errorf("tool %v requires riskTier %s", tool, requiredTier)
        }
    }

    return nil
}
```

### 6.3 Reconciler Struct

```go
type AgentReconciler struct {
    client.Client
    Scheme               *runtime.Scheme
    Recorder             record.EventRecorder
    TransformationClient transformationv1.TransformationServiceClient
    SecurityPolicy       *SecurityPolicyConfig
    ToolGrantsService    toolsv1.ToolGrantsServiceClient // optional central service
}
```

### 6.4 Agent Deployment Template

```go
func mutateAgentDeployment(deploy *appsv1.Deployment, agent *amv1.Agent, secretNames map[string]string) {
    envVars := []corev1.EnvVar{
        {Name: "AGENT_DEFINITION_ID", Value: agent.Spec.DefinitionRef.ID},
        {Name: "TENANT_ID", Value: agent.Spec.DefinitionRef.TenantID},
        {Name: "MODEL_PROVIDER", Value: agent.Spec.Model.Provider},
        {Name: "MODEL_NAME", Value: agent.Spec.Model.Name},
        {Name: "RISK_TIER", Value: agent.Spec.RiskTier},
    }

    // Add secret refs for each credential
    for name, secretName := range secretNames {
        envVars = append(envVars, corev1.EnvVar{
            Name: strings.ToUpper(name) + "_API_KEY",
            ValueFrom: &corev1.EnvVarSource{
                SecretKeyRef: &corev1.SecretKeySelector{
                    LocalObjectReference: corev1.LocalObjectReference{Name: secretName},
                    Key:                  "api-key",
                },
            },
        })
    }

    deploy.Spec.Template.Spec.Containers[0].Env = envVars
}
```

### 6.5 Owned Resources

```go
ctrl.NewControllerManagedBy(mgr).
    For(&amv1.Agent{}).
    Owns(&appsv1.Deployment{}).
    Owns(&corev1.ConfigMap{}).        // tool grants
    Owns(&esv1.ExternalSecret{}).     // LLM + tool secrets
    Complete(r)
```

---

## 7. Non-Goals

| What | Why |
|------|-----|
| Prompt orchestration | Lives in Agent image |
| LLM API calls | Agent runtime handles |
| Tool implementations | Domain/Process services or MCP servers |
| Conversation storage | Chat/Threads domain responsibility |

---

## 8. Dependencies

| Dependency | Purpose |
|------------|---------|
| **Transformation Domain** | Source of AgentDefinitions |
| **External Secrets Operator** | Sync secrets from vault |
| **Security Policy** | ConfigMap or CRD defining allowed models/tools |
| **Tool Grants Service** | Optional central registry for tool access |
| **agent-developer (LangGraph coder)** | Defines LangGraph DAG metadata + devcontainer service contract consumed by the operator |

---

## 9. Open Questions

| Question | Status |
|----------|--------|
| Central vs K8s-side tool grants? | TBD – start with ConfigMap, evolve to service |
| Secret rotation handling? | TBD – ExternalSecret refresh interval |
| Multi-model agents? | Out of scope for v1 |

---

## 10. Cross-References

| Backlog | Relationship |
|---------|--------------|
| [446-namespace-isolation.md](446-namespace-isolation.md) | Operator deploys once per cluster |
| [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Cluster-scoped deployment via ApplicationSet |
| [503-operators-helm-chart.md](503-operators-helm-chart.md) | Helm chart for operator deployment |
| [495-ameide-operators.md](495-ameide-operators.md) | Agent operator responsibilities (§3) |
| [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) | Go patterns & adaptation (§10.9) |
| [477-primitive-stack.md](477-primitive-stack.md) | Agent in primitive architecture |
| [471-ameide-business-architecture.md](471-ameide-business-architecture.md) | AgentDefinition as design artifact |
| [505-agent-developer.md](505-agent-developer.md) | LangGraph coder runtime + `develop_in_container` tool requirements |
