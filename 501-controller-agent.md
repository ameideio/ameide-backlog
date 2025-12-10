# 48x – Intelligent Agent Controllers (IAC)

**Status:** Draft v1  
**Owner:** Platform / Architecture  
**Audience:** Platform engineers, domain/process teams, SRE, “architect/agent” implementers  
**Depends on:** 470 (Vision), 471 (Business Architecture), 472 (Application & Information), 473 (Technology), 475 (Domains), 476 (Security & Trust), 478 (Extensions), 479‑480 (Tier 1 WASM), 461 (IDC/IPC/IAC control plane)

---

## 1. Purpose

The **Intelligent Agent Controller (IAC)** is the declarative control-plane representation of Ameide agent runtimes. It bridges the gap between:

* **AgentController** (logical/runtime) – executes an AgentDefinition via LLM/tool loops, always invoked through proto/SDK contracts.
* **IntelligentAgentController** (IAC CRD) – a Kubernetes custom resource (`kind: IntelligentAgentController`) that declares how that agent runtime should exist (image, runtime chassis, tool grants, risk tier, tenancy, observability). An Ameide operator reconciles the CR into Deployments, secrets, ServiceMonitors, and policy objects.

This document captures the target state for IACs: CRD schema, operator behaviour, interactions with Transformation/UAF, IDC/IPC, Tier 1 WASM, and Backstage/GitOps. Migration strategy is out of scope.

---

## 2. Position in the architecture

### 2.1 Relationships

* **Design-time** – AgentDefinitions live in the Transformation DomainController (modelled via UAF). They specify tools, orchestration graphs, policies, and risk tiers.
* **Runtime (IAC CRD)** – references the AgentDefinition revision and adds operational configuration (runtime image, resource class, secret bindings, telemetry, rollout phase). The IAC operator reconciles this CR.
* **IDCs/DomainControllers** – expose deterministic APIs + “primitives”. IAC tools call those APIs via the shared SDK. Tool grants are recorded in the IAC spec so security can reason about which domains an agent can touch.
* **IPCs/ProcessControllers** – orchestrate IACs by calling `InvokeAgentTool` entries defined in IPC bindings. IPC operator validates that the referenced IAC/tool exists and is in `Ready` status before compiling BPMN → Temporal.
* **Tier 1 WASM** – IACs may invoke deterministic helpers hosted in the shared `extensions-runtime` (e.g. to apply guarded post-processing). Those WASM calls are declared in the IAC spec as additional “deterministic tools” with their own limits.

### 2.2 Backstage & GitOps

Backstage templates (Domain/Process/Agent) now emit IAC manifests alongside repo skeletons. GitOps applies only the CR; operators manage Deployments/Services/Secrets. The Backstage catalog uses the IAC metadata (`ameide.io/controller-tier=agent`) to show ownership, tenant scope, and links to AgentDefinitions.

---

## 3. IAC CRD – target shape

```yaml
apiVersion: ameide.io/v1
kind: IntelligentAgentController
metadata:
  name: pricing-agent
  namespace: ameide-prod         # or tenant-{id}-{env}-cust for tenant agents
  labels:
    ameide.io/domain: sales
    ameide.io/controller-tier: agent
spec:
  displayName: "Pricing Strategy Agent"
  controllerType: platform | tenant
  sku:
    tier: shared | namespace | private

  runtime:
    chassis: langgraph | n8n | custom
    image: ghcr.io/ameideio/pricing-agent:v0.15.2
    resources:
      class: default-medium
    replicas:
      min: 2
      max: 6
    loggingLevel: info

  definitionRef:
    artifactId: "uaf://agents/pricing-strategy"
    revision: "2025-01-05-v4"
    interfaceVersion: "ameide.agents.pricing.v1"

  risk:
    tier: low | medium | high
    piiHandling: redact | allow | deny
    maxConcurrentRuns: 3
    deterministicToolsOnly: false

  toolGrants:
    - name: suggest-discount
      target:
        kind: IntelligentDomainController
        name: orders
        primitive:
          service: ameide.orders.v1.OrdersService
          method: ProposeDiscount
      mode: write
    - name: explain-margin
      target:
        kind: IntelligentProcessController
        name: lead-to-order
        action: fetch-context
      mode: read
    - name: run-deterministic-hook
      target:
        kind: WasmExtension
        extensionId: pricing-safe-rounding
        version: "latest"
      mode: deterministic

  secrets:
    openAiAPIKey: secret://platform/openai
    vectorStore: secret://platform/qdrant

  observability:
    sloClass: silver
    traces: enabled
    logSampleRate: 0.1

  tenancy:
    allowedTenants: ["*"]
    namespacePolicy: auto

status:
  phase: Ready | Progressing | Degraded
  conditions:
    - type: DeploymentAvailable
      status: "True"
      reason: MinimumReplicasAvailable
    - type: ToolGrantsValidated
      status: "True"
      reason: GrantsSynced
  endpoints:
    grpc: pricing-agent.ameide-prod.svc.cluster.local:8080
    http: https://agents.prod.ameide.io/pricing
  metrics:
    avgLatencyMs: 820
    successRate: 0.97
```

### 3.1 Metadata, labels, and rollout

Agent controllers follow the same metadata contract introduced in 500/461 so automation can treat every controller uniformly:

* `metadata.labels.ameide.io/controller-tier=agent`
* `metadata.labels.ameide.io/domain=<owning bounded context>`
* `metadata.labels.ameide.io/sku=<shared|namespace|private>`
* `metadata.annotations.argocd.argoproj.io/rollout-phase=650`

The default rollout phase (650) keeps IACs behind platform/domain controllers while still allowing tenant overrides to run additional smokes later in the ladder. Operators must also emit `ToolGrantsValidated` and `SecretsReady` conditions in `status.conditions`; the Lua health checks used by ArgoCD read these flags before marking the Application Healthy.

We do **not** re-list the label contract here—see §4.3 of [500-controller-domain](500-controller-domain.md)—because that backlog is the canonical source. Instead, IAC docs are responsible for the _additional_ agent-specific requirements:

* The shared `status.phase` enum (`Ready`, `Progressing`, `Degraded`) applies exactly as defined in 500 so controller-contract tests can treat IDCs/IACs/IPCs uniformly.
* Operators must publish the mandatory IDC conditions (`DeploymentAvailable`, `MigrationsApplied`, `DataPlaneHealthy`) **and** flip on `RoutesPublished` whenever HTTP/GRPC routes exist, then layer the agent-only extensions `ToolGrantsValidated` and `SecretsReady` on top. Depending on whether the controller owns Gateway routes, ArgoCD and the Architect Agent will inspect five or six conditions, but the names and semantics stay identical to the contract defined in backlog 500.

### 3.2 Agent-specific condition extensions

In addition to the baseline table from §4.4 of [500-controller-domain](500-controller-domain.md), the IAC operator **must** surface the following conditions in `status.conditions`:

| Condition | Purpose | Set to `True` when… |
|-----------|---------|---------------------|
| `ToolGrantsValidated` | Confirms the operator reconciled every declared grant, checked target existence, and registered the grant with the central tool registry. | Each `spec.toolGrants[]` entry resolved to a live IDC/IPC/Wasm target, the SDK allowlist was updated, and no policy violations remain. |
| `SecretsReady` | Shows that ExternalSecrets/Vault data landed and the runtime has the redacted/rotated credentials it needs. | All references under `spec.secrets` are projected into the pod (env/volume) and the operator has verified checksums against the Vault metadata. |

Only when **all** baseline and agent-specific conditions are `True` may the operator set `status.phase=Ready`. Keeping the requirements explicit here prevents drift between controllers and keeps Argo’s Lua health script declarative (the Lua merely looks for condition names instead of controller-specific logic).

Key points:

* CRDs reference **AgentDefinition artifacts** rather than embedding prompt/code. UAF remains the source of truth for agent logic; CRs focus on runtime posture.
* Tool grants reference `IntelligentDomainController`, `IntelligentProcessController`, or `WasmExtension` targets and specify read/write modes.
* Risk tier drives runtime policies (max concurrency, sandboxing, manual approvals) enforced by the operator.

---

## 4. IAC operator responsibilities

1. **Provision runtime workloads**
   * Create Deployment, Service, HPA, ServiceMonitor, PodDisruptionBudget, NetworkPolicies.
   * Inject standard sidecars (OTel collector, structured logging).
   * Apply rollout phases per 447 and reconcile image digests.

2. **Manage secrets & credentials**
   * Pull LLM/API keys from ExternalSecrets + Vault according to `spec.secrets`.
   * Mount vector stores / file systems via CSI drivers when declared.
   * Ensure no secrets are baked into the image (validation step before rollout).

3. **Register tool grants**
   * Validate that referenced IDC/IPC targets exist and expose the declared primitives.
   * Publish grants to the centralized `tool_registry` service so IPCs/UIs know which capabilities the agent exposes.
   * Block rollout if grants violate policies (e.g., medium-risk agent requesting write access to high-risk domains).

4. **Enforce risk tier policies**
   * Rate-limit or serialize executions based on `spec.risk.maxConcurrentRuns`.
   * Toggle deterministic-only mode (no arbitrary HTTP, no shell) for low/medium tiers.
   * Configure guardrails for external calls (timeout, cost ceilings, PII scrubbing).

5. **Wire deterministic helpers (Tier 1 WASM)**
   * When a tool targets `kind: WasmExtension`, automatically fetch allowed `extensionId/version` pairs from Transformation events and configure the runtime to call the shared `extensions-runtime` service with the agent’s execution context.

6. **Emit status & telemetry**
   * Update `status.conditions` for deployment health, tool-grant sync, and secret hydration.
   * Expose per-agent metrics (token usage, success/error counts, latency) and forward them to observability stacks with `ameide.io/controller-tier=agent` labels.

7. **Own Gateway API exposure**
   * Agents that present gRPC/Connect or HTTP endpoints must ship their `HTTPRoute`/`GRPCRoute` resources in the IAC release (reusing the same contract as 500/417/459). Platform gateways provide listeners/TLS only; every controller owns its hostnames, oauth2-proxy frontends, and health probes so Argo can treat traffic exposure as part of the controller spec instead of an `extraHttpRoutes` side-file.
   * The operator also emits the shared `RoutesPublished` condition (§4.3 of 500) whenever those routes exist. Keep it `False` until each HTTPRoute/GRPCRoute reports `Accepted=True` and `Programmed=True` against `gateway/ameide` so Argo health blocks rollouts when oauth2-proxy chains or hostnames drift.

---

## 5. Security, tenancy & compliance

IACs embody the constraints from 476:

* **Isolation** – platform-owned agents run in `ameide-{env}`; tenant-specific agents deploy to `tenant-{id}-{env}-cust`. The operator enforces namespace selection using SKU + controllerType. No tenant agent may target shared namespaces.
* **Tool grant enforcement** – RBAC + policy engines ensure an agent can call only the declared IDC/IPC primitives. Requests outside the allowlist are denied at the SDK/interceptor layer and logged.
* **Secrets & PII** – risk tier defines whether raw data can leave the tenant boundary. Operators apply redact/allow/deny policies; for `redact`, the runtime replaces sensitive fields before sending them to LLMs.
* **Auditability** – every tool invocation logs tenant/org/user context plus prompt/output digests. GitOps changes to IAC specs are captured as audit events.

---

## 6. Lifecycle & upgrades

### 6.1 Creation & promotion

1. Architect or Agent engineer defines/updates the AgentDefinition in Transformation.
2. Backstage template (or an agent) generates the IAC spec skeleton referencing that artifact.
3. PR adds the CR to GitOps along with runtime repo updates.
4. CI validates lint checks (risk tier, tool grants, secret references).
5. ArgoCD syncs; operator reconciles; rollout phases per 447 apply.

### 6.2 Change vectors

* **Prompt/logic changes** – new AgentDefinition revision; update `spec.definitionRef.revision`.
* **Tool grants** – modify `spec.toolGrants`; operator revalidates + updates registry.
* **Runtime image/SDK** – change `spec.runtime.image`; operator rolls out Deployment with health checks.
* **Risk posture** – raising `risk.tier` forces manual approval and different concurrency limits.

### 6.3 Downgrades/disablement

* `spec.suspend: true` (future extension) allows operators to pause an agent (Deployment scaled to zero, tool grants revoked) without deleting the CR.

---

## 7. Interaction with other controllers

* **IDCs** – IAC tools call IDC primitives strictly via SDKs with tenant-aware context. IDC specs list which primitives are “agent-safe”; IAC linting ensures grants reference only those methods.
* **IPCs** – Process controllers bind BPMN service tasks to agent tools. IPC operator uses the IAC status to decide if a process variant can be promoted.
* **Tier 1 WASM** – Agents can call deterministic WASM extensions for math/rules. Those hooks remain platform-owned; IAC specs simply reference the `extensionId` and version.
* **Transformation/UAF** – IACs push execution metrics back to Transformation so architects and agents can compare behaviour across revisions.

---

## 8. Example: Tenant-specific transformation agent

```yaml
apiVersion: ameide.io/v1
kind: IntelligentAgentController
metadata:
  name: tenant42-transform-agent
  namespace: tenant-42-prod-cust
spec:
  controllerType: tenant
  sku:
    tier: namespace
  runtime:
    chassis: langgraph
    image: ghcr.io/tenant42/transform-agent:v0.3.1
    resources:
      class: tenant-small
  definitionRef:
    artifactId: "uaf://agents/tenant42/transform"
    revision: "tenant42-transform-2025-02-01"
  risk:
    tier: medium
    maxConcurrentRuns: 2
  toolGrants:
    - name: propose-domain-change
      target:
        kind: IntelligentDomainController
        name: tenant42-orders
        primitive:
          service: tenant42.orders.v1.OrdersService
          method: ProposeChange
      mode: write
  secrets:
    openAiAPIKey: secret://tenant42/openai
  tenancy:
    allowedTenants: ["tenant42"]
```

This illustrates how tenant agents stay inside tenant namespaces and reference tenant-owned IDCs while still following the same CRD/operator model.

---

## 9. Open questions

1. **Tool grant DSL** – do we need richer semantics (filters, rate limits) per grant, or is simple read/write/deterministic enough?
2. **LLM provider mixes** – should the spec allow multiple providers with weighted routing (OpenAI + Azure + self-hosted) or rely on runtime config?
3. **Suspend vs delete semantics** – do we encode `spec.suspend` now or rely on scaling to zero via annotations?
4. **Tenant override safety** – how do we guarantee tenant authz when tenants fork agent repos but reuse platform IAC operator? Additional signature checks may be necessary.
5. **Cost telemetry** – do we standardize a `status.costs` block so finance can reason about per-agent token spend?
