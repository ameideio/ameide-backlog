Here’s **Document 4/6 – Technology Architecture**.

---

# 4 – Technology Architecture (Ameide Platform)

**Status:** Draft
**Audience:** Platform / Domain & Process Teams / DevEx / SRE
**Scope:** How Ameide is built and operated: runtimes, infra, integration points, and the technical guardrails that make "domain / process / agent" controllers real.

> **Cross-References (Deployment Architecture Suite)**:
> For GitOps deployment patterns, see:
>
> | Document | Purpose |
> |----------|---------|
> | [465-applicationset-architecture.md](465-applicationset-architecture.md) | How the two ApplicationSets work |
> | [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Rollout phases & sub-phase patterns |
> | [464-chart-folder-alignment.md](464-chart-folder-alignment.md) | Chart folder structure & domain alignment |
> | [426-keycloak-config-map.md](426-keycloak-config-map.md) | Secrets handling, OIDC client extraction |
> | [477-backstage.md](477-backstage.md) | Backstage implementation (§2.3 factory pattern) |

---

## 1. Purpose & Relationship to Other Docs

* The **Vision** document says *what* Ameide aspires to be.
* The **Business Architecture** document describes *who* uses it and *for what* (L2O/O2C, transformation, etc.).
* The **Application & Information Architecture** describes *which logical building blocks* (DomainController, ProcessController, Agent, Transformation, UAF) exist and how data flows between them.
* **This document** describes *how those building blocks are implemented and operated* using Kubernetes, Backstage, Temporal, proto-based APIs, and the Ameide SDKs.

The goal is to give implementers a **vendor-aligned blueprint** that can be executed with minimal invention.

---

## 2. Technology Principles

1. **Kubernetes-native declarative controllers**

   * We use Kubernetes for *both infrastructure and controller lifecycles*. Domain/Process/Agent controllers are declared as `IntelligentDomainController`, `IntelligentProcessController`, and `IntelligentAgentController` custom resources (see 461). GitOps applies those CRs; Ameide operators reconcile them into Deployments, Services, HPAs, Temporal workers, and supporting config.
   * We still reserve CRDs for coarse application constructs (controllers, tenants, infra). Business aggregates (Orders, Customers, etc.) remain inside domain services; we do **not** create one CRD per aggregate. ([Kubernetes][1])

2. **Proto‑first, SDK‑first APIs**

   * All Domain/Process/Agent services share a single proto source of truth (Buf BSR) and generated server + client stubs. 
   * The canonical way to talk to the platform is via the language SDKs (TS/Go/Python) and the shared `AmeideClient` contract, not by hand‑rolling gRPC clients. 

3. **Backstage as the “factory” for controllers**

   * Backstage’s **Software Catalog** and **Software Templates / Scaffolder** are used to create and manage all Ameide services (domain/process/agent) and their Helm charts. ([Backstage][2])
   * Ameide-specific templates encapsulate all wiring (proto skeleton, SDK, deployment, monitoring), so an agent or the transformation domain can “ask” for a new controller and simply fill in code.

4. **Temporal for process orchestration; BPMN-compliant ProcessDefinitions for process intent**

   * The *runtime* for processes is Temporal (code‑first workflows). ([Temporal][3])
   * **ProcessDefinitions** are BPMN-compliant artifacts produced by a **custom React Flow modeller** (NOT Camunda/bpmn-js) and stored in the **Transformation DomainController** (modelled via UAF UIs). Mapping ProcessDefinitions to Temporal is handled by the Transformation pipeline.
   * ProcessControllers execute ProcessDefinitions at runtime, backed by Temporal workflows.

5. **Deterministic domains, non‑deterministic agents**

   * DomainControllers must be deterministic, side‑effect controlled, and replay‑safe (unit testable, idempotent).
   * Agents are explicitly non‑deterministic (LLMs, retrieval), wrapped behind tool interfaces. ProcessControllers orchestrate them while keeping all durable state in deterministic domains or workflows.

6. **Event‑sourced artifacts, not event‑sourced everything**

   * The **Transformation DomainController** event‑sources *design artifacts* (BPMN, architecture diagrams, Markdown) so every transformation is traceable, but day‑to‑day transactional data (orders, invoices) uses traditional persistence.
   * **Note**: UAF is the modelling UI that sends commands to Transformation; it doesn't own storage. 

7. **GitOps & immutable deployments**

   * Argo CD + Helm remain the deployment plane. Git stores controller CRs (IDC/IPC/IAC), infra CRs, and operator manifests; operators own the low-level Deployments/Services, so Argo never `kubectl apply`s mutable child resources directly.

8. **Tiered extensibility (WASM for small, controllers for big)**

   * Small, deterministic tenant customizations run as WASM extensions in a shared `extensions-runtime` service; larger changes create dedicated Domain/Process/Agent controllers in tenant namespaces per 478.

---

## 3. Layered Technology Stack

### 3.1 Cluster & Infrastructure Layer

**Purpose:** Provide secure, multi‑tenant Kubernetes-based infrastructure on which Ameide runs.

**Core components**

* **Kubernetes** (managed: AKS/EKS/GKE)
* **GitOps:** Argo CD + Helm for all app and infra releases
* **Databases:** Postgres via CNPG operator (tenant-aware schemas, RLS), ClickHouse for analytics
* **Object storage:** S3‑compatible (MinIO or cloud provider) for artifact snapshots (BPMN, docs, AI artifacts)
* **Secrets:** ExternalSecrets + Vault for app secrets, CNPG-managed credentials for DBs 
* **Identity:** Keycloak (realm-per-tenant for enterprise, shared realm for SMB), wired into Ameide via OIDC/JWT tenant claims 
* **Gateway/Ingress:** Envoy/Gateway API for TLS termination and routing

**Operator usage**

* Use Kubernetes “Operator pattern” for:

  * Databases (CNPG), Kafka (if used), Redis, ClickHouse.
  * Application control plane: `IntelligentDomainController`, `IntelligentProcessController`, `IntelligentAgentController` operators reconcile controller specs into workloads across namespaces/SKUs.
  * Tenant lifecycle: an **`AmeideTenant` CRD** to provision baseline infra per tenant (DB schema, Keycloak realm, namespaces).
* Operators stay focused on coarse-grained platform resources (infra, controllers, tenants). Business data and aggregates remain inside the services described by those controllers. ([Kubernetes][5])

---

### 3.2 Runtime Orchestration Layer

**Purpose:** Provide the primitives for long‑running business processes and background work.

**Temporal**

* **Temporal Cloud / self‑hosted Temporal** is the default workflow engine for ProcessControllers:

  * Code‑as‑workflow, deterministic replays, multi‑year executions. ([Temporal][3])
  * Best practices: workflows orchestrate, activities do work, external I/O is always in activities. ([Medium][6])
* Ameide runs dedicated **workers** per domain/process namespace (e.g. `sales-process`, `transformation-process`) and uses Temporal’s namespace access control to segregate tenants.

**ProcessDefinitions (BPMN-compliant)**

* ProcessDefinitions created via **custom React Flow modeller** are:

  * BPMN-compliant in semantics (stages, gateways, lanes) but NOT Camunda-stack.
  * Stored in the **Transformation DomainController** (with event history), modelled via UAF UIs.
* The default path compiles ProcessDefinitions to Temporal workflows executed by ProcessControllers.

**Background jobs & messaging**

* For most orchestration, **Temporal replaces bespoke message buses** (sagas, retries, compensation all live in workflows). ([Temporal][3])
* Lightweight, event-style notifications (e.g. for UI refresh) can use NATS/Kafka, but they are not a source of truth.

---

### 3.3 Service Layer: Domains, Processes, Agents, Transformation

In addition to the Domain/Process/Agent controllers described below, the platform runs a shared `extensions-runtime` service in `ameide-{env}` for Tier 1 WASM extensions (see §3.3.6 and 479/480). This runtime executes tenant-authored modules as data inside a platform-owned service, preserving the namespace invariants from 478.

**3.3.1 DomainControllers (domains)**

* Implemented as **stateless microservices** (Go or Python/TS) exposing gRPC/Connect services defined in the shared proto repo.
* Each DomainController has:

  * Its own schema in Postgres (with RLS by tenant).
  * A clear API surface (CRUD + domain commands) expressed in proto.
  * No direct UI logic; only data + domain rules.
* DomainControllers are deployed via standard Helm charts generated from Backstage templates (see §4). ([Backstage][7])

**3.3.2 ProcessControllers (processes)**

* Each ProcessController executes a **ProcessDefinition** (design-time artifact) at runtime:

  * **ProcessDefinition** (stored in Transformation DomainController): BPMN-compliant artifact from custom React Flow modeller, modelled via UAF UIs.
  * **ProcessController** (runtime): Temporal workflow implementation (Go/TS/Python). ([Temporal][3])
  * Declarations of which DomainControllers and AgentControllers it interacts with.
* The technology pattern:

  * ProcessDefinitions are edited in-browser (custom React Flow modeller), stored in the Transformation DomainController (via UAF APIs).
  * Transformation DomainController compiles ProcessDefinitions into Temporal workflow code or config.
  * Temporal workers execute the compiled workflows; ProcessControllers expose gRPC endpoints to start or query process instances.

**3.3.3 AgentControllers (agents)**

* Each AgentController executes an **AgentDefinition** (design-time artifact) at runtime:

  * **AgentDefinition** (stored in Transformation DomainController): Declarative spec for tools, orchestration graph, scope, risk tier, policies (modelled via UAF UIs).
  * **AgentController** (runtime): Inference runtime (Python/TS) that encapsulates:

    * LLM provider (OpenAI, others),
    * Tools that map to DomainController/ProcessController APIs via the Ameide SDKs,
    * LangGraph- or similar graph definition for multi-step reasoning.
* AgentControllers are declared as Backstage components and managed like any other service (templates + Helm), but they **do not own durable state**; they propose changes that must go through ProcessControllers/DomainControllers.

**3.3.4 Transformation & UAF**

> **Core Definitions** (see [470-ameide-vision.md §0](470-ameide-vision.md)):
> - **Transformation DomainController** = data + APIs + event-sourced artifact store (ProcessDefinitions, AgentDefinitions, etc.)
> - **UAF** = set of frontends (BPMN editor, diagram editor, etc.) that call those APIs
> - There is no separate "UAF service" in the runtime.

* The Transformation domain is implemented as:

  * A **DomainController** for transformation data (initiatives, stages, metrics, ProcessDefinitions, AgentDefinitions, and other design artifacts).
  * A **builder/compile service** (within Transformation DomainController) that transforms design-time artifacts into runtime deployments (Temporal workflows, AgentController configs, Backstage template updates).
  * **UAF UIs** that provide the modelling experience (BPMN editor, architecture diagrams, Markdown) by calling Transformation APIs.

**3.3.5 Controller Operators (IDC/IPC/IAC)**

* Three Ameide operators watch the controller CRDs defined in [461-ipc-idc-iac.md](461-ipc-idc-iac.md):

  * **IDC operator** – reconciles each `IntelligentDomainController` into Deployments, Services, HPAs, ServiceMonitors, CNPG schemas/users, migrations Jobs, and NetworkPolicies. Enforces namespace/SKU placement, image policies, resource classes, and standard sidecars/env vars.
  * **IPC operator** – reconciles `IntelligentProcessController` specs into Temporal worker Deployments, ConfigMaps linking BPMN to code, ServiceMonitors, and optional CronJobs for compensations. Validates references to ProcessDefinitions and Temporal namespaces.
  * **IAC operator** – reconciles `IntelligentAgentController` specs into agent runtime Deployments, Secrets (LLM keys, tool credentials), allowlisted tool grants, and observability wiring.

* Operators publish status/conditions back to their CRs (e.g., `Ready`, `Degraded`, `RolloutPaused`, `MigrationFailed`) so GitOps/UIs can reason in controller-level objects.
* GitOps applies only the CR manifests plus the operator Deployments. All child resources are managed via reconciliation loops, so cross-cutting changes (sidecars, probes, annotations) happen centrally in operators.

**3.3.6 WasmExtensionRuntime (Tier 1)**

* Platform-owned service in `ameide-{env}` that executes ExtensionDefinitions (WASM modules) per tenant.
* Uses Wasmtime/WasmEdge with CPU/memory/time limits per invocation plus host-call policies.
* Loads modules from MinIO per the storage model in 479/480 and exposes an `InvokeExtension` RPC for controllers via the Ameide SDKs.
* All side effects go through Domain/Process/Agent APIs with the caller’s tenant/org/user context; the runtime does not expose direct DB, filesystem, or arbitrary network access.

---

### 3.4 Experience & UI Layer

**Next.js platform app**

* Primary user UI: Next.js app (`www_ameide_platform`) consuming Ameide SDK TS via the proto-based APIs.
* Microfrontends hosted under a shell; per‑domain/per‑process views are separate bundles but share the same Ameide SDK and auth/session model.

**Backstage instance**

* Separate Backstage deployment for:

  * Catalog of all Domain/Process/Agent components. ([Backstage][2])
  * Software Templates for creating new controllers. ([Backstage][7])
  * Potential marketplace integration (3rd-party templates/agents).

**Design/Modeling tools**

* **Custom React Flow modeller** for ProcessDefinitions (BPMN-compliant semantics, NOT Camunda/bpmn-js).
* ArchiMate modelers (browser-based) for architecture artifacts.
* Both run inside Next.js or as separate webapps, communicating with the Transformation DomainController APIs.

---

## 4. Backstage Implementation as the Ameide “Factory”

Backstage is where **controllers are born**.

### 4.1 Catalog model

Minimum entity types in the Backstage catalog:

* `Component`:

  * `type: domain-controller` – Domain service with its proto package + DB schema.
  * `type: process-controller` – Process service with BPMN + Temporal workers.
  * `type: agent` – LLM-based AgentController.
* `System`: groups components into solution domains (e.g. L2O).
* `Resource`: DB clusters, queues, Temporal namespaces associated with components. ([Backstage][2])

### 4.2 Software templates

We define a small set of **opinionated templates**: ([Backstage][7])

1. **Domain Controller template**

   * Generates:

     * Proto service skeleton + messages for the new domain.
     * Go or Python service skeleton (DDD-ish).
     * SQL migrations (Flyway-style) for initial tables.
     * Helm chart (Deployment, Service, VirtualService/HTTPRoute, Prometheus rules).
     * Backstage `catalog-info.yaml` with `type: domain-controller`.
   * Uses the Buf-based proto pipeline + Ameide SDK import patterns.

2. **ProcessController template**

   * Generates:

     * Initial ProcessDefinition (BPMN-compliant, from React Flow modeller) + modeling project.
     * Temporal workflow worker skeleton in chosen language.
     * UAF artifact registration & Transformation API integration.

3. **AgentController template**

   * Generates:

     * AgentDefinition file (tools, policies, orchestration graph) stored in Transformation DomainController (via UAF APIs).
     * Tool stubs for DomainController/ProcessController APIs using Ameide SDK TS/Go/Python.
     * Helm chart for AgentController inference runtime deployment (autoscaled).

These templates are *also* what agents pick from when “auto‑rolling” a new controller: the agent writes the template input, not the raw K8s manifests.

---

## 5. API & Integration Technology

### 5.1 Proto contracts & transports

* All services use proto contracts stored in a Buf workspace and BSR; Buf linting and breaking-change rules protect compatibility. 
* Server side:

  * gRPC or Connect RPC (Go/Python/TS) with standardized interceptors for auth, tenant resolution, tracing, error mapping.
* Client side:

  * Ameide SDKs (TS/Go/Python) generated from the same protos; TS SDK uses `connect-es` + tree‑shakeable `AmeideClient`. 

### 5.2 Patterns & guardrails

* Central error mapping (domain ↔ HTTP/gRPC codes) and a `RequestContext` DTO as described in proto-based API spec. 
* UnitOfWork (transaction) pattern in domain services; no cross‑service distributed transactions—use Temporal sagas instead.
* Protos versioned via directories (`v1`, `v1alpha`) with Buf breaking-change CI.

---

## 6. Multi‑Tenancy & Environment Topology

### 6.1 Tenancy model (technical)

* **Identity & access:** Keycloak realms and JWT `tenantId` / `orgId` claims determine tenant context in all services. 
* **Data isolation:**

  * Primary: RLS per tenant in shared schemas (SMB scale).
  * Promotion: heavy tenants can be moved to dedicated DB instances or clusters (managed by infra operators).
* **Execution isolation:**

  * Temporal uses namespaces per tenant tier or segment.
  * Backstage uses labels to group tenant-specific controllers if needed.

### 6.2 Environments

* Standard lanes: `dev`, `staging`, `prod`, optionally `preview/*`.
* Argo CD Apps-of-Apps structure; each environment is a separate Argo Application tree.
* Backstage templates and Ameide controllers always deploy via GitOps—no ad-hoc `kubectl apply`.

### 6.3 Tenant Extension Namespaces

For tenants with custom controllers, additional namespaces are provisioned based on SKU:

| SKU | Product Runtime | Custom Code |
|-----|-----------------|-------------|
| Shared | `ameide-{env}` (shared) | `tenant-{id}-{env}-cust` |
| Namespace | `tenant-{id}-{env}-base` | `tenant-{id}-{env}-cust` |
| Private | `tenant-{id}-{env}-base` | `tenant-{id}-{env}-cust` |

**Security invariant**: Custom code never runs in shared `ameide-*` namespaces. This is enforced at scaffolding time (Backstage templates) and deployment time (GitOps policies).

> **See [478-ameide-extensions.md](478-ameide-extensions.md)** for the complete namespace strategy, repository model, and E2E controller creation flow.

Tier 1 WASM extensions run inside the shared `extensions-runtime` service in `ameide-{env}`. Modules are treated as tenant-scoped data executed under strict sandbox and host-call policies, so the “no tenant services in `ameide-*` namespaces” invariant still holds.

---

## 7. Observability, Quality & Tooling

* **Tracing & metrics:** OpenTelemetry across services; Temporal emits its own spans/metrics; combined in Grafana.
  * `extensions-runtime` emits per-extension latency/error/resource metrics and integrates with circuit breakers so Tier 1 hooks remain observable.
* **Logging:** Structured logs with correlation IDs (`trace_id`, `tenant_id`, `controller_type`, `controller_name`).
* **Testing & quality:**

  * Domain controllers: unit tests + integration tests (DB), contract tests via generated clients.
  * ProcessControllers: Temporal workflow unit tests + ProcessDefinition lint checks.
  * Agents: tool contract tests; sandbox execution before allowing them to write Backstage templates or code (reuse patterns from the core-platform coder review). 

---

## 8. Migration & Legacy Alignment (IPA → Controllers)

* The previous **IPA architecture** (IPAs as composed workflows/agents/tools) remains a conceptual ancestor; the new stack reimplements:

  * IPA composition → ProcessController + UAF artifact.
  * IPA builder → Transformation builder + Backstage templates + Temporal workers.
* Existing IPA-related packages inform code organization but are not exposed directly to tenants.

---

## 9. Open Questions & Options

1. **Crossplane or not?**

   * Crossplane could manage external cloud resources (DB instances, buckets) declaratively per tenant, but is optional; for now CNPG + manual IaC is sufficient.

2. **How far to push CRDs?**

   * Short term: `AmeideTenant` + infra operators only.
   * Medium term: consider CRDs for *catalog-ish* constructs (e.g. `TransformationPipeline`) if they provide clear operational benefits.

3. **ProcessDefinition modeller technology**

   * Custom React Flow modeller is the design-time tool for BPMN-compliant ProcessDefinitions.
   * Temporal is the only runtime for ProcessControllers; no Camunda integration planned.

4. **Backstage multi‑tenancy**

   * Initially: one Ameide Backstage (internal), exposing templates and catalog to Ameide engineers/agents.
   * Long term: evaluate customer-facing Backstage instances or “Backstage-as-a-service” for tenants.

---

## 10. Cross‑References

This Technology Architecture should be read with the following documents:

| Document | Relationship | Alignment |
|----------|-------------|-----------|
| [470‑ameide‑vision](470-ameide-vision.md) | Parent vision & principles | Strong ✅ |
| [471‑ameide‑business‑architecture](471-ameide-business-architecture.md) | Business use cases, personas | Strong ✅ |
| [475‑ameide‑domains](475-ameide-domains.md) | Domain portfolio & patterns | Strong ✅ |
| [465‑applicationset‑architecture](465-applicationset-architecture.md) | GitOps deployment model | Strong ✅ |
| [464‑chart‑folder‑alignment](464-chart-folder-alignment.md) | Chart structure & rollout phases | Strong ✅ |
| [333‑realms](333-realms.md) | Realm‑per‑tenant implementation | Strong ✅ |
| [305‑workflow](305-workflow.md) | Temporal/ProcessController runtime | See §10.1 |
| [310‑agents‑v2](310-agents-v2.md) | AgentController layer | See §10.2 |
| [388‑ameide‑sdks‑north‑star](388-ameide-sdks-north-star.md) | SDK publish strategy | See §10.3 |
| [478‑ameide‑extensions](478-ameide-extensions.md) | Namespace topology & extension deployment | See §10.4 |
| [479‑ameide‑extensibility‑wasm](479-ameide-extensibility-wasm.md) | Tier 1 + Tier 2 extension model | See §3.3.6 |
| [480‑ameide‑extensibility‑wasm-service](480-ameide-extensibility-wasm-service.md) | Runtime implementation | See §3.3.6 |

### 10.1 Workflow alignment (305)

* 305 describes Temporal-based workflow orchestration; aligns with §3.2 (Runtime Orchestration Layer).
* ProcessDefinitions (BPMN-compliant, from custom React Flow modeller) are compiled to Temporal workflows.
* ProcessControllers execute ProcessDefinitions at runtime, backed by Temporal.

### 10.2 Agents alignment (310)

* 310 describes AgentController implementation with n8n-style flows; 473 §3.3.3 describes LangGraph-based AgentControllers.
* AgentDefinitions (design-time specs stored in Transformation DomainController, modelled via UAF UIs) are executed by AgentControllers at runtime.
* **Clarification**: Both n8n-aligned (visual low-code) and LangGraph (code-first) patterns are valid AgentController implementations.

### 10.3 SDK alignment (388)

* 388 defines SDK publish strategy; 473 §5 depends on published SDKs for all controller types.
* **Gap**: Per 470 §9.2.3, TypeScript SDK is not yet published to npmjs. This blocks external consumers from using the TS SDK as 473 §5.1 requires.

### 10.4 Extension alignment (478)

478 defines the namespace topology for tenant extensions:

* **Shared SKU**: Product in `ameide-{env}`, custom code in `tenant-{id}-{env}-cust`
* **Namespace SKU**: Product in `tenant-{id}-{env}-base`, custom code in `tenant-{id}-{env}-cust`
* **Private SKU**: Product in `tenant-{id}-{env}-base`, custom code in `tenant-{id}-{env}-cust`

**Key invariant**: Custom services always run in dedicated tenant namespaces, never in shared `ameide-*` namespaces.

The E2E flow in 478 §6 describes how controllers are scaffolded via Backstage and deployed via GitOps, extending the patterns in 473 §4.

---

## 11. Notable Gaps & Issues

### 11.1 Terminology alignment

| 473 Term | Related Terms | Notes |
|----------|---------------|-------|
| DomainController | DomainService (deprecated) | Consistent across 470-476 ✅ |
| ProcessController | Workflow (305) | Runtime that executes ProcessDefinitions |
| ProcessDefinition | BPMN model (deprecated) | Design-time artifact in UAF (from custom React Flow modeller) |
| AgentController | AgentRuntime (deprecated) | Runtime that executes AgentDefinitions |
| AgentDefinition | Agent config (deprecated) | Design-time artifact in UAF |
| Component domain (465) | N/A | 465 "domain" = folder category (apps, data), not business domain |

**Clarification needed**: "Domain" is overloaded:
* **Business domain** (473 §3.3): bounded context (Orders, Product, Identity)
* **Component domain** (465): ArgoCD folder category (apps, data, platform)

Recommend using "component category" or "deployment domain" for 465-style usage.

### 11.2 Implementation gaps

| Gap | Description | Impact | Related Docs |
|-----|-------------|--------|--------------|
| Backstage not deployed | §4 assumes Backstage as controller factory | Cannot use template-driven service creation | 470 §9.2.2 |
| Transformation artifact storage | §2.6 assumes event-sourced ProcessDefinitions/AgentDefinitions in Transformation DomainController | Implementation detail; infra TBD | 471 §3.2 |
| Temporal namespace isolation | §3.2 mentions per-tenant namespaces | Current deployment is single namespace | 305 |
| AmeideTenant CRD | §3.1 suggests tenant CRD | 333 uses API-driven realm provisioning | 333 |
| ProcessDefinition→Temporal compiler | §3.2 assumes ProcessDefinition to Temporal translation | No compiler exists yet | 305, 471 |

### 11.3 Open architectural tensions

1. **Backstage vs GitOps static components**
   * 473 §4 envisions Backstage generating new controllers dynamically
   * 465 uses static `component.yaml` files discovered by ApplicationSets
   * **Resolution needed**: How do Backstage-generated services integrate with 465 GitOps model?

2. **AmeideTenant CRD vs API-driven**
   * 473 §3.1 suggests `AmeideTenant` CRD for tenant provisioning
   * 333 implements realm provisioning via Keycloak Admin API
   * **Decision needed**: Is CRD-based approach still desired, or has API-driven superseded it?

3. **AgentController runtime choice**
   * 473 §3.3.3 describes LangGraph-based AgentControllers
   * 310 describes n8n-aligned visual workflows
   * **Resolution**: Both are valid AgentController implementations; AgentDefinitions abstract the runtime choice

---

If you'd like, next we can do **Document 5/6 – Domains Architecture**, and apply this tech stack concretely to L2O and Transformation (including where UAF, ProcessDefinitions, and Temporal sit in those domains).

[1]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/?utm_source=chatgpt.com "Custom Resources"
[2]: https://backstage.io/docs/features/software-catalog/?utm_source=chatgpt.com "Backstage Software Catalog and Developer Platform"
[3]: https://docs.temporal.io/workflows?utm_source=chatgpt.com "Temporal Workflow | Temporal Platform Documentation"
[5]: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/?utm_source=chatgpt.com "Operator pattern"
[6]: https://medium.com/%40ajayshekar01/best-practices-for-building-temporal-workflows-a-practical-guide-with-examples-914fedd2819c?utm_source=chatgpt.com "Best Practices for Building Temporal Workflows"
[7]: https://backstage.io/docs/features/software-templates/?utm_source=chatgpt.com "Backstage Software Templates"
