Here’s **Document 4/6 – Technology Architecture**.

---

# 4 – Technology Architecture (Ameide Platform)

**Status:** Draft
**Audience:** Platform / Domain & Process Teams / DevEx / SRE
**Scope:** How Ameide is built and operated: runtimes, infra, integration points, and the technical guardrails that make “domain / process / agent” controllers real.

---

## 1. Purpose & Relationship to Other Docs

* The **Vision** document says *what* Ameide aspires to be.
* The **Business Architecture** document describes *who* uses it and *for what* (L2O/O2C, transformation, etc.).
* The **Application & Information Architecture** describes *which logical building blocks* (DomainController, ProcessController, Agent, Transformation, UAF) exist and how data flows between them.
* **This document** describes *how those building blocks are implemented and operated* using Kubernetes, Backstage, Temporal, proto-based APIs, and the Ameide SDKs.

The goal is to give implementers a **vendor-aligned blueprint** that can be executed with minimal invention.

---

## 2. Technology Principles

1. **Kubernetes‑native, but domain‑agnostic**

   * We use Kubernetes for *infrastructure lifecycles* (Postgres, ingress, secrets, workers) and standard workloads, but **domain controllers themselves remain ordinary microservices**, not CRDs.
   * CRDs/operators are kept for infra or coarse tenant constructs (e.g. `AmeideTenant`, database clusters), following the Kubernetes guidance that custom resources are for extending the control plane, not for every business concept. ([Kubernetes][1])

2. **Proto‑first, SDK‑first APIs**

   * All Domain/Process/Agent services share a single proto source of truth (Buf BSR) and generated server + client stubs. 
   * The canonical way to talk to the platform is via the language SDKs (TS/Go/Python) and the shared `AmeideClient` contract, not by hand‑rolling gRPC clients. 

3. **Backstage as the “factory” for controllers**

   * Backstage’s **Software Catalog** and **Software Templates / Scaffolder** are used to create and manage all Ameide services (domain/process/agent) and their Helm charts. ([Backstage][2])
   * Ameide-specific templates encapsulate all wiring (proto skeleton, SDK, deployment, monitoring), so an agent or the transformation domain can “ask” for a new controller and simply fill in code.

4. **Temporal for process orchestration; BPMN for process intent**

   * The *runtime* for processes is Temporal (code‑first workflows). ([Temporal][3])
   * BPMN (Camunda tooling) is used as a **design artifact and contract** for ProcessControllers, not as the only runtime. Mapping BPMN to Temporal is handled by the transformation/UAF pipeline.
   * Where needed, Camunda 8 remains a first‑class target for BPMN deployment (e.g. customers heavily invested in Camunda). ([Camunda 8 Docs][4])

5. **Deterministic domains, non‑deterministic agents**

   * DomainControllers must be deterministic, side‑effect controlled, and replay‑safe (unit testable, idempotent).
   * Agents are explicitly non‑deterministic (LLMs, retrieval), wrapped behind tool interfaces. ProcessControllers orchestrate them while keeping all durable state in deterministic domains or workflows.

6. **Event‑sourced artifacts, not event‑sourced everything**

   * Unified Artifact Framework (UAF) event‑sources *design artifacts* (BPMN, ArchiMate, Markdown) so every transformation is traceable, but day‑to‑day transactional data (orders, invoices) uses more traditional persistence models. 

7. **GitOps & immutable deployments**

   * Argo CD + Helm are the deployment plane; operators/CRDs are used for infra (CNPG, Keycloak, etc.), not for application logic.

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
  * Potentially an **`AmeideTenant` CRD** to provision baseline infra per tenant (DB schema, Keycloak realm, namespaces).
* Avoid per-domain CRDs; we follow K8s guidance that operators manage applications and infra lifecycles, not business tables. ([Kubernetes][5])

---

### 3.2 Runtime Orchestration Layer

**Purpose:** Provide the primitives for long‑running business processes and background work.

**Temporal**

* **Temporal Cloud / self‑hosted Temporal** is the default workflow engine for ProcessControllers:

  * Code‑as‑workflow, deterministic replays, multi‑year executions. ([Temporal][3])
  * Best practices: workflows orchestrate, activities do work, external I/O is always in activities. ([Medium][6])
* Ameide runs dedicated **workers** per domain/process namespace (e.g. `sales-process`, `transformation-process`) and uses Temporal’s namespace access control to segregate tenants.

**BPMN / Camunda**

* BPMN models created via bpmn‑js + Camunda moddle are:

  * Stored as UAF artifacts (with an event history).
  * Optionally deployed to Camunda 8 clusters when required by customer architecture. ([Camunda 8 Docs][4])
* The default path translates BPMN to Temporal workflows (per the North‑Star design / UAF).

**Background jobs & messaging**

* For most orchestration, **Temporal replaces bespoke message buses** (sagas, retries, compensation all live in workflows). ([Temporal][3])
* Lightweight, event-style notifications (e.g. for UI refresh) can use NATS/Kafka, but they are not a source of truth.

---

### 3.3 Service Layer: Domains, Processes, Agents, Transformation

**3.3.1 DomainControllers (domains)**

* Implemented as **stateless microservices** (Go or Python/TS) exposing gRPC/Connect services defined in the shared proto repo.
* Each DomainController has:

  * Its own schema in Postgres (with RLS by tenant).
  * A clear API surface (CRUD + domain commands) expressed in proto.
  * No direct UI logic; only data + domain rules.
* DomainControllers are deployed via standard Helm charts generated from Backstage templates (see §4). ([Backstage][7])

**3.3.2 ProcessControllers (processes)**

* Each ProcessController is essentially a **bundle**:

  * A BPMN model (UAF artifact).
  * A Temporal workflow implementation (Go/TS/Python). ([Temporal][3])
  * Declarations of which DomainControllers and Agents it interacts with.
* The technology pattern:

  * BPMN is edited in-browser (bpmn‑js), stored via UAF.
  * Transformation domain compiles the BPMN + Ameide primitives into Temporal workflow code or config.
  * Temporal workers execute the compiled workflows; ProcessControllers expose gRPC endpoints to start or query process instances.

**3.3.3 Agents (AgentControllers)**

* Agents run in an **inference runtime** (Python/TS) that encapsulates:

  * LLM provider (OpenAI, others),
  * Tools that map to DomainController/ProcessController APIs via the Ameide SDKs,
  * LangGraph- or similar graph definition for multi-step reasoning.
* Agents are declared as Backstage components and managed like any other service (templates + Helm), but they **do not own durable state**; they propose changes that must go through processes/domains.

**3.3.4 Transformation & UAF**

* The Transformation domain is implemented as:

  * A DomainController for transformation data (initiatives, stages, metrics, IPAs/UAF artifacts).
  * A **builder/compile service** (inspired by the IPA builder) that transforms artifacts into runtime deployments (Temporal workflows, BPMN deployments, Backstage template updates).

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

* BPMN & possibly ArchiMate modelers are browser-based (bpmn‑js + properties panel, Archimate‑JS) running inside Next.js or as separate webapps, communicating with the UAF and Design APIs described in the North‑Star doc.

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

2. **Process Controller template**

   * Generates:

     * BPMN template file + modeling project.
     * Temporal workflow worker skeleton in chosen language.
     * UAF artifact registration & Design API integration.

3. **Agent template**

   * Generates:

     * Agent graph definition file (YAML/JSON).
     * Tool stubs for Domain/Process controllers using Ameide SDK TS/Go/Python.
     * Helm chart for inference runtime deployment (autoscaled).

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

---

## 7. Observability, Quality & Tooling

* **Tracing & metrics:** OpenTelemetry across services; Temporal and Camunda emit their own spans/metrics; combined in Grafana.
* **Logging:** Structured logs with correlation IDs (`trace_id`, `tenant_id`, `controller_type`, `controller_name`).
* **Testing & quality:**

  * Domain controllers: unit tests + integration tests (DB), contract tests via generated clients.
  * Process controllers: Temporal workflow unit tests + BPMN lint checks.
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

3. **Camunda vs Temporal balance**

   * For now, Temporal is the default runtime; Camunda 8 is a first‑class integration for customers already there, leveraging BPMN best practices from Camunda docs. ([Camunda 8 Docs][8])

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
| [310‑agents‑v2](310-agents-v2.md) | AgentRuntime layer | See §10.2 |
| [388‑ameide‑sdks‑north‑star](388-ameide-sdks-north-star.md) | SDK publish strategy | See §10.3 |

### 10.1 Workflow alignment (305)

* 305 describes Temporal-based workflow orchestration; aligns with §3.2 (Runtime Orchestration Layer).
* **Gap**: 305 uses code-first Temporal; 473 §2.4 also supports BPMN→Temporal translation via UAF. Ensure 305 can accept both code-first and BPMN-compiled workflows.

### 10.2 Agents alignment (310)

* 310 describes AgentRuntime with n8n-style flows; 473 §3.3.3 describes LangGraph-based agents.
* **Tension**: Two agent patterns may coexist (n8n for visual low-code, LangGraph for code-first). Backstage templates should support both.
* **Resolution needed**: Clarify which runtime is primary and how they interoperate.

### 10.3 SDK alignment (388)

* 388 defines SDK publish strategy; 473 §5 depends on published SDKs for all controller types.
* **Gap**: Per 470 §9.2.3, TypeScript SDK is not yet published to npmjs. This blocks external consumers from using the TS SDK as 473 §5.1 requires.

---

## 11. Notable Gaps & Issues

### 11.1 Terminology alignment

| 473 Term | Related Terms | Notes |
|----------|---------------|-------|
| DomainController | Domain Service, DomainService (470/471) | Consistent across docs ✅ |
| ProcessController | Workflow (305), BPMN Process | 305 uses "Workflow"; recommend aligning |
| AgentRuntime | Agent (310), AgentController | 310 uses "AgentRuntime"; consistent ✅ |
| Component domain (465) | N/A | 465 "domain" = folder category (apps, data), not business domain |

**Clarification needed**: "Domain" is overloaded:
* **Business domain** (473 §3.3): bounded context (Orders, Product, Identity)
* **Component domain** (465): ArgoCD folder category (apps, data, platform)

Recommend using "component category" or "deployment domain" for 465-style usage.

### 11.2 Implementation gaps

| Gap | Description | Impact | Related Docs |
|-----|-------------|--------|--------------|
| Backstage not deployed | §4 assumes Backstage as controller factory | Cannot use template-driven service creation | 470 §9.2.2 |
| UAF storage undefined | §2.6 assumes event-sourced artifacts | No infra backlog defines UAF persistence | 471 §3.2 |
| Temporal namespace isolation | §3.2 mentions per-tenant namespaces | Current deployment is single namespace | 305 |
| AmeideTenant CRD | §3.1 suggests tenant CRD | 333 uses API-driven realm provisioning | 333 |
| BPMN→Temporal compiler | §3.2 assumes BPMN to Temporal translation | No compiler exists yet | 305, 471 |

### 11.3 Open architectural tensions

1. **Backstage vs GitOps static components**
   * 473 §4 envisions Backstage generating new controllers dynamically
   * 465 uses static `component.yaml` files discovered by ApplicationSets
   * **Resolution needed**: How do Backstage-generated services integrate with 465 GitOps model?

2. **AmeideTenant CRD vs API-driven**
   * 473 §3.1 suggests `AmeideTenant` CRD for tenant provisioning
   * 333 implements realm provisioning via Keycloak Admin API
   * **Decision needed**: Is CRD-based approach still desired, or has API-driven superseded it?

3. **Agent runtime choice**
   * 473 §3.3.3 describes LangGraph-based agents
   * 310 describes n8n-aligned visual workflows
   * **Clarification needed**: Are both first-class, or is one primary?

4. **Camunda 8 integration scope**
   * 473 §2.4, §3.2 mention Camunda as optional runtime
   * No infrastructure or charts exist for Camunda deployment
   * **Decision needed**: Is Camunda out of scope for current infrastructure?

---

If you'd like, next we can do **Document 5/6 – Domains Architecture**, and apply this tech stack concretely to L2O and Transformation (including where UAF, BPMN and Temporal sit in those domains).

[1]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/?utm_source=chatgpt.com "Custom Resources"
[2]: https://backstage.io/docs/features/software-catalog/?utm_source=chatgpt.com "Backstage Software Catalog and Developer Platform"
[3]: https://docs.temporal.io/workflows?utm_source=chatgpt.com "Temporal Workflow | Temporal Platform Documentation"
[4]: https://docs.camunda.io/docs/components/modeler/bpmn/?utm_source=chatgpt.com "BPMN in Modeler | Camunda 8 Docs"
[5]: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/?utm_source=chatgpt.com "Operator pattern"
[6]: https://medium.com/%40ajayshekar01/best-practices-for-building-temporal-workflows-a-practical-guide-with-examples-914fedd2819c?utm_source=chatgpt.com "Best Practices for Building Temporal Workflows"
[7]: https://backstage.io/docs/features/software-templates/?utm_source=chatgpt.com "Backstage Software Templates"
[8]: https://docs.camunda.io/docs/components/best-practices/best-practices-overview/?utm_source=chatgpt.com "Best Practices - Camunda 8 Docs"
