# 500 – Agent Operator

**Status:** Active (Phase 1 Complete)
**Audience:** Platform engineers implementing Agent operator
**Scope:** Implementation tracking, development phases, acceptance criteria

> **Related**:
> - [495-ameide-operators.md](495-ameide-operators.md) – CRD shapes & responsibilities
> - [497-operator-implementation-patterns.md](497-operator-implementation-patterns.md) – Go patterns & reference implementation

---

## 1. Overview

The Agent operator manages the lifecycle of **Agent primitives** – LLM-powered autonomous actors with tool access. Each `Agent` CR results in:

- AgentDefinition fetched from Transformation Domain
- ExternalSecrets for LLM API keys and tool credentials
- Tool grants ConfigMap (allowed tools per security policy)
- Agent runtime Deployment (LangGraph, custom framework, etc.)

**Key insight**: The operator wires secrets + policy; the Agent image implements prompt orchestration and tool execution.

---

## 2. CRD Summary

```yaml
apiVersion: ameide.io/v1
kind: Agent
metadata:
  name: core-platform-coder
spec:
  image: ghcr.io/ameide/agent-langgraph:2.1.0
  definitionRef:
    id: core-platform-coder-v4
    tenantId: t123
  model:
    provider: openai
    name: gpt-5.1-pro
  tools:
    allowed:
      - domain: transformation
      - process: l2o
      - primitive: primitive-cli
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

Backlog [505-agent-developer.md](505-agent-developer.md) introduces `runtime_type=langgraph`, `dag_ref`, and the `develop_in_container` tool binding for the `core-platform-coder` AgentDefinition. Implications for the operator:

* **Spec propagation.** When the Transformation domain resolves an AgentDefinition that includes `runtime_type=langgraph`, the operator must surface the referenced DAG module/image to the runtime Deployment (env vars or ConfigMap). This keeps the runtime controller generic while enabling LangGraph to build the DAG defined in Transformation.
* **Tool grants.** `develop_in_container` is treated like any other tool grant, but it is automatically `riskTier=high`. The existing policy validation (Phase 3) should reject Agent CRs that request `develop_in_container` without `spec.riskTier=high` or without the platform-approved devcontainer service configuration.
* **Runtime service wiring.** For LangGraph coder agents, `reconcileRuntime()` needs to include Service annotations or env vars that point at the devcontainer gRPC endpoint deployed via GitOps. Document this under Phase 3/4 artifacts so the ApplicationSet that deploys the devcontainer service stays in sync with the Agent operator rollout described in backlog 504.

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
    Allowed []ToolGrant `json:"allowed,omitempty"`
}

type ToolGrant struct {
    Domain    string `json:"domain,omitempty"`
    Process   string `json:"process,omitempty"`
    Primitive string `json:"primitive,omitempty"` // special tools like CLI
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
| **Transformation client** | gRPC client to fetch AgentDefinition | Can call `GetAgentDefinition` RPC |
| **Definition fetching** | `reconcileDefinition()` fetches by ID | Definition retrieved |
| **Definition as ConfigMap** | Store definition for agent runtime | Agent can read definition |
| **DefinitionResolved condition** | Set based on fetch success | Condition reflects definition state |

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
  - github-token (if tools.allowed includes github)
  - jira-token (if tools.allowed includes jira)
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
