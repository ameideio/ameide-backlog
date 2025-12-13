Here’s **Document 4/6 – Technology Architecture**.

---

# 4 – Technology Architecture (Ameide Platform)

**Status:** Draft
**Audience:** Platform / Domain & Process Teams / DevEx / SRE
**Scope:** How Ameide is built and operated: runtimes, infra, integration points, and the technical guardrails that make the Ameide primitive kinds (Domain / Process / Agent / UISurface / Projection / Integration) real.

> **Core Invariants**: See [470-ameide-vision.md §0 "Ameide Core Invariants"](470-ameide-vision.md) for the canonical list (six primitives: Domain/Process/Agent/UISurface/Projection/Integration, Graph read-only, Transformation as domain, proto chain, tenant isolation, Backstage internal).
>
> **Cross-References (Deployment Architecture Suite)**:
> For GitOps deployment patterns, see:
>
> | Document | Purpose |
> |----------|---------|
> | [465-applicationset-architecture.md](465-applicationset-architecture.md) | How the two ApplicationSets work |
> | [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Rollout phases & sub-phase patterns |
> | [464-chart-folder-alignment.md](464-chart-folder-alignment.md) | Chart folder structure & domain alignment |
> | [426-keycloak-config-map.md](426-keycloak-config-map.md) | Secrets handling, OIDC client extraction |
> | [467-backstage.md](467-backstage.md) | Backstage implementation (§2.3 factory pattern) |

## Grounding & contract alignment

- **Primitive runtime layer:** Describes how the primitives from `470-ameide-vision.md` and `472-ameide-information-application.md` are realized technically (Kubernetes, operators, Temporal, Backstage, Buf/SDKs), forming the basis that operator/vertical-slice backlogs (`495-ameide-operators.md`, `497-operator-implementation-patterns.md`, `498-domain-operator.md`, `499-process-operator.md`, `500-agent-operator.md`, `501-uisurface-operator.md`, `503-operators-helm-chart.md`, `502-domain-vertical-slice.md`) implement.  
- **EDA and infra alignment:** Binds infra choices (CNPG, Envoy/Gateway, ExternalSecrets, Vault, Temporal) to the EDA and proto-chain rules further detailed in `472-ameide-information-application.md` and `496-eda-principles.md`.  
- **Scrum/agent relevance:** Provides the technical substrate (Temporal workers, agent runtimes, devcontainers, GitOps model) that the Scrum stack (`506-scrum-vertical-v2.md`, `508-scrum-protos.md`) and agent backlogs (`505-agent-developer-v2*.md`, `504-agent-vertical-slice.md`) rely on without redefining their contracts.

---

## Implementation Status (2025-02-14)

- ✅ Graph mutations are now blocked centrally so operators and services can trust the projection boundary (`services/graph/src/graph/service.ts:118-135`, `services/graph/src/graph/service.ts:4879-4905`).
- ✅ Backstage is delivered as a GitOps-managed chart with OIDC/HTTPRoute wiring, matching the “platform factory” deployment story (`gitops/ameide-gitops/sources/charts/platform/backstage/values.yaml:1-100`).
- ✅ LangGraph/devcontainer coder agent plumbing ships end-to-end: Transformation persists `runtime_type`/`dag_ref` (`db/flyway/sql/transformation/V2__agent_definitions.sql`, `services/transformation/src/transformations/service.ts:672-845`), the Agent operator fetches and mounts definition metadata into ConfigMaps for runtimes (`operators/agent-operator/internal/controller/agent_controller.go`), and GitOps assets deploy both the LangGraph runtime (`gitops/ameide-gitops/sources/values/_shared/platform/agents/core-platform-coder.yaml`) and devcontainer service (`gitops/ameide-gitops/sources/values/_shared/platform/platform-devcontainer-service.yaml`).
- ✅ LangGraph coder scaffolds are now repo-first: `primitives/agent/ameide-coder/**` carries the template instead of the Backstage Service Catalog, so teams can bootstrap runtimes directly from the monorepo while Backstage remains out of scope.
- ⚠️ Process and UISurface operators still reconcile only Deployments/Services without definition/Temporal/routing validation, so additional slices are required for parity (`operators/process-operator/internal/controller/process_controller.go:54-149`, `operators/uisurface-operator/internal/controller/uisurface_controller.go:55-169`).

## 1. Purpose & Relationship to Other Docs

* The **Vision** document says *what* Ameide aspires to be.
* The **Business Architecture** document describes *who* uses it and *for what* (L2O/O2C, transformation, etc.).
* The **Application & Information Architecture** describes *which logical building blocks* (Domain primitive, Process primitive, Agent, Transformation, Transformation design tooling) exist and how data flows between them.
* **This document** describes *how those building blocks are implemented and operated* using Kubernetes, Backstage, Temporal, proto-based APIs, and the Ameide SDKs.

The goal is to give implementers a **vendor-aligned blueprint** that can be executed with minimal invention.

---

## 2. Technology Principles

1. **Kubernetes-native declarative primitives**

  * We use Kubernetes for *both infrastructure and primitive lifecycles*. The six primitives are declared via their CRDs; GitOps applies those CRDs and Ameide operators reconcile them into workloads and supporting config.
   * We still reserve CRDs for coarse application constructs (primitives, tenants, infra). Business aggregates (Orders, Customers, etc.) remain inside domain services; we do **not** create one CRD per aggregate. ([Kubernetes][1])

2. **Proto‑first, SDK‑first APIs**

   * All Domain/Process/Agent services share a single proto source of truth (Buf BSR) and generated server + client stubs. 
   * The canonical way to talk to the platform is via the language SDKs (TS/Go/Python) and the shared `AmeideClient` contract, not by hand‑rolling gRPC clients. 

3. **Backstage as the “factory” for primitives**

  * Backstage’s **Software Catalog** and **Software Templates / Scaffolder** are used to create and manage Ameide primitives and their deployment artifacts. ([Backstage][2])
   * Ameide-specific templates encapsulate all wiring (proto skeleton, SDK, deployment, monitoring), so an agent or the Transformation Domain can “ask” for a new primitive and simply fill in code.

4. **Temporal for process orchestration; BPMN-compliant ProcessDefinitions for process intent**

   * The *runtime* for processes is Temporal (code‑first workflows). ([Temporal][3])
   * **ProcessDefinitions** are BPMN-compliant artifacts produced by a **custom React Flow modeller** (NOT Camunda/bpmn-js) and stored in the **Transformation Domain** (modelled via Transformation design tooling UIs). Mapping ProcessDefinitions to Temporal is handled by the Transformation pipeline.
   * Process primitives execute ProcessDefinitions at runtime, backed by Temporal workflows.

5. **Deterministic domains, non‑deterministic agents**

   * Domain primitives must be deterministic, side‑effect controlled, and replay‑safe (unit testable, idempotent).
   * Agents are explicitly non‑deterministic (LLMs, retrieval), wrapped behind tool interfaces. Process primitives orchestrate them while keeping all durable state in deterministic domains or workflows.

6. **Event‑sourced artifacts, not event‑sourced everything**

   * The **Transformation Domain** event‑sources *design artifacts* (BPMN, architecture diagrams, Markdown) so every transformation is traceable, but day‑to‑day transactional data (orders, invoices) uses traditional persistence.
   * **Note**: Transformation design tooling is the modelling UI that sends commands to Transformation; it doesn't own storage. 

7. **GitOps & immutable deployments**

  * Argo CD + Helm remain the deployment plane. Git stores primitive CRDs (Domain/Process/Agent/UISurface/Projection/Integration), infra CRDs, and operator manifests; operators own the low-level child resources, so Argo never `kubectl apply`s mutable children directly.

8. **Tiered extensibility (WASM for small, primitives for big)**

   * Small, deterministic tenant customizations run as WASM extensions in a shared `extensions-runtime` service; larger changes create dedicated Domain/Process/Agent primitives in tenant namespaces per 478.

9. **Proto-first contracts + event-driven plumbing**

   * Buf modules in `packages/ameide_core_proto` are the single source of truth. Workspace SDKs (TS/Go/Py) ingest generated stubs so services break immediately on schema changes; GitOps manifests annotate the image/proto versions per [472 §2.7](472-ameide-information-application.md#27-contract--implementation--runtime-chain).
   * Go-based Process/Domain primitives implement command/event handlers via Watermill’s CQRS component (CommandBus/EventBus/Processors), keeping write/read models in sync (see [472 §3.3.1](472-ameide-information-application.md#331-event-driven-cqrs-runtime-watermill)).

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
  * Application control plane: Domain, Process, Agent, UISurface, Projection, and Integration operators reconcile CRDs into workloads across namespaces/SKUs.
  * Tenant lifecycle: an **`AmeideTenant` CRD** to provision baseline infra per tenant (DB schema, Keycloak realm, namespaces).
* Operators stay focused on coarse-grained platform resources (infra, primitives, tenants). Business data and aggregates remain inside the services described by those primitives. ([Kubernetes][5])

---

### 3.2 Runtime Orchestration Layer

**Purpose:** Provide the primitives for long‑running business processes and background work.

**Temporal**

* **Temporal Cloud / self‑hosted Temporal** is the default workflow engine for Process primitives:

  * Code‑as‑workflow, deterministic replays, multi‑year executions. ([Temporal][3])
  * Best practices: workflows orchestrate, activities do work, external I/O is always in activities. ([Medium][6])
* Ameide runs dedicated **workers** per domain/process namespace (e.g. `sales-process`, `transformation-process`) and uses Temporal’s namespace access control to segregate tenants.

**ProcessDefinitions (BPMN-compliant)**

* ProcessDefinitions created via **custom React Flow modeller** are:

  * BPMN-compliant in semantics (stages, gateways, lanes) but NOT Camunda-stack.
  * Stored in the **Transformation Domain** (with event history), modelled via Transformation design tooling UIs.
* The default path compiles ProcessDefinitions to Temporal workflows executed by Process primitives.

**Background jobs & messaging**

* For most orchestration, **Temporal replaces bespoke message buses** (sagas, retries, compensation all live in workflows). ([Temporal][3])
* Lightweight, event-style notifications (e.g. for UI refresh) can use NATS/Kafka, but they are not a source of truth.

### 3.2.1 Broker Selection & Delivery Guarantees

Event-driven architectures require intentional broker selection. Ameide uses different technologies for different use cases:

| Use Case | Technology | Delivery Guarantee | Rationale |
|----------|------------|-------------------|-----------|
| **Domain events (outbox relay)** | Postgres + Watermill | At-least-once, ordered per aggregate | Transactional outbox ensures no dual writes; ordering by aggregate ID is sufficient |
| **Cross-domain integration** | NATS JetStream | At-least-once, durable | Low-latency, persistent streams; per-tenant subject partitioning |
| **Analytics & replay** | Kafka (or Redpanda) | At-least-once, replayable | High-throughput, long retention for graph projections and analytics |
| **UI real-time updates** | NATS (non-persistent) | Best-effort | Fire-and-forget for UI refresh; not source of truth |
| **Process orchestration** | Temporal | Exactly-once (within workflow) | Deterministic replay; no external broker needed |

**Delivery guarantee rules:**

1. **At-least-once is the default** – All event consumers must be idempotent (see [472 §3.3.2](472-ameide-information-application.md))
2. **Exactly-once is only for Temporal** – Never promise exactly-once delivery over external brokers
3. **Ordering is per-aggregate** – Partition by `(tenant_id, aggregate_id)` to ensure causal ordering within an aggregate
4. **Cross-aggregate ordering is not guaranteed** – Use Temporal sagas for cross-aggregate coordination

**Stream configuration patterns:**

```yaml
# NATS JetStream - Domain events
stream:
  name: domain-events-{domain}
  subjects: ["events.{domain}.>"]
  retention: limits          # 7-day rolling window
  max_age: 604800s
  storage: file              # Durable
  replicas: 3

# Kafka - Analytics replay
topic:
  name: analytics.{domain}
  partitions: 12             # By tenant_id hash
  retention.ms: 2592000000   # 30 days
  min.insync.replicas: 2
```

**Multi-tenant stream isolation:**

* Events always carry `tenant_id` in the payload
* NATS subjects include tenant prefix: `events.{tenant_id}.{domain}.{event_type}`
* Kafka partitions by `tenant_id` to ensure tenant-local ordering
* Consumers validate `tenant_id` against execution context (see [472 §3.3.7](472-ameide-information-application.md))

> **Invariant**: Domain primitives MUST NOT import broker clients directly. All event publishing goes through the outbox interface; the outbox dispatcher handles broker-specific publishing.

---

### 3.3 Service Layer: Domains, Processes, Agents, Transformation

In addition to the Domain/Process/Agent primitives described below, the platform runs a shared `extensions-runtime` service in `ameide-{env}` for Tier 1 WASM extensions (see §3.3.6 and 479/480). This runtime executes tenant-authored modules as data inside a platform-owned service, preserving the namespace invariants from 478.

**3.3.1 Domain primitives (domains)**

* Implemented as **stateless microservices** (Go or Python/TS) exposing gRPC/Connect services defined in the shared proto repo.
* Each Domain primitive has:

  * Its own schema in Postgres (with RLS by tenant).
  * A clear API surface (CRUD + domain commands) expressed in proto.
  * No direct UI logic; only data + domain rules.
* Domain primitives are deployed via standard Helm charts generated from Backstage templates (see §4). ([Backstage][7])

> **Event reliability**: Go-based Domain primitives must follow the EDA patterns in [472 §3.3](472-ameide-information-application.md): transactional outbox (§3.3.1.1), idempotent consumers (§3.3.2), schema versioning (§3.3.5), and multi-tenant isolation (§3.3.7).

**3.3.2 Process primitives (processes)**

* Each Process primitive executes a **ProcessDefinition** (design-time artifact) at runtime:

  * **ProcessDefinition** (stored in Transformation Domain): BPMN-compliant artifact from custom React Flow modeller, modelled via Transformation design tooling UIs.
  * **Process primitive** (runtime): Temporal workflow implementation (Go/TS/Python). ([Temporal][3])
  * Declarations of which Domain primitives and Agent primitives it interacts with.
* The technology pattern:

  * ProcessDefinitions are edited in-browser (custom React Flow modeller), stored in the Transformation Domain (via Transformation design tooling APIs).
  * Transformation Domain compiles ProcessDefinitions into Temporal workflow code or config.
  * Temporal workers execute the compiled workflows; Process primitives expose gRPC endpoints to start or query process instances.

**3.3.3 Agent primitives (agents)**

* Each Agent primitive executes an **AgentDefinition** (design-time artifact) at runtime:

  * **AgentDefinition** (stored in Transformation Domain): Declarative spec for tools, orchestration graph, scope, risk tier, policies (modelled via Transformation design tooling UIs).
  * **Agent primitive** (runtime): Inference runtime (Python/TS) that encapsulates:

    * LLM provider (OpenAI, others),
    * Tools that map to Domain primitive/Process primitive APIs via the Ameide SDKs,
    * LangGraph- or similar graph definition for multi-step reasoning.
* Agent primitives are declared as Backstage components and managed like any other service (templates + Helm), but they **do not own durable state**; they propose changes that must go through Process primitives/Domain primitives.

**3.3.4 Transformation & Transformation design tooling**

> **Core Definitions** (see [470-ameide-vision.md §0](470-ameide-vision.md)):
> - **Transformation Domain** = data + APIs + event-sourced artifact store (ProcessDefinitions, AgentDefinitions, etc.)
> - **Transformation design tooling** = set of frontends (BPMN editor, diagram editor, etc.) that call those APIs
> - There is no separate "Transformation design tooling service" in the runtime.

* The Transformation domain is implemented as:

  * A **Domain primitive** for transformation data (initiatives, stages, metrics, ProcessDefinitions, AgentDefinitions, and other design artifacts).
  * A **builder/compile service** (within Transformation Domain) that transforms design-time artifacts into runtime deployments (Temporal workflows, Agent primitive configs, Backstage template updates).
  * **Transformation design tooling UIs** that provide the modelling experience (BPMN editor, architecture diagrams, Markdown) by calling Transformation APIs.

**3.3.5 Primitive operators**

> **Repo layout**: Operator implementations live under `operators/*-operator` (e.g., `operators/domain-operator`). See [477-primitive-stack.md §2](477-primitive-stack.md) for the full folder structure.

* Six Ameide operators watch the primitive CRDs (one per primitive kind):

  * **Domain operator** (`operators/domain-operator`) – reconciles each Domain CRD into Deployments, Services, HPAs, ServiceMonitors, CNPG schemas/users, migration Jobs, and NetworkPolicies. Enforces namespace/SKU placement, image policies, resource classes, and standard sidecars/env vars.
  * **Process operator** (`operators/process-operator`) – reconciles Process CRDs into Temporal worker Deployments, ConfigMaps linking BPMN to code, ServiceMonitors, and optional CronJobs for compensations. Validates references to ProcessDefinitions and Temporal namespaces.
  * **Agent operator** (`operators/agent-operator`) – reconciles Agent CRDs into agent runtime Deployments, Secrets (LLM keys, tool credentials), allowlisted tool grants, and observability wiring.
  * **UISurface operator** (`operators/uisurface-operator`) – reconciles UISurface CRDs into Next.js Deployment/Service, HPA, Ingress/Gateway routes (domains/paths), env config pointing at Domain/Process/Agent endpoints, and auth scopes wired to Keycloak/IdP clients. This is a **thin infra operator**—it ensures a UISurface image is reachable and correctly wired to primitives, but does not understand UI components or form layouts.
  * **Projection operator** (`operators/projection-operator`) – reconciles Projection CRDs into read-model/query workloads and storage bindings (e.g., Postgres materialized views, event-driven projection tables, ClickHouse/analytics backends) and surfaces refresh/lag/health conditions. Projection operators are operational only: they never interpret business semantics beyond wiring, scheduling, and status.
  * **Integration operator** (`operators/integration-operator`) – reconciles Integration CRDs into “flows-as-code” runtimes (default: NiFi) plus day-2 lifecycle (sync from Git/registry, parameter contexts, schedules, drift detection) with secrets injected from Kubernetes. Integration operators are operational only: mapping/decision logic stays in human-owned integration code/flows.

* Operators publish status/conditions back to their CRDs (e.g., `Ready`, `Degraded`, `RolloutPaused`, `MigrationFailed`) so GitOps/UIs can reason in primitive-level objects.
* GitOps applies only the CR manifests plus the operator Deployments. All child resources are managed via reconciliation loops, so cross-cutting changes (sidecars, probes, annotations) happen centrally in operators.

**3.3.6 WasmExtensionRuntime (Tier 1)**

* Platform-owned service in `ameide-{env}` that executes ExtensionDefinitions (WASM modules) per tenant.
* Uses Wasmtime/WasmEdge with CPU/memory/time limits per invocation plus host-call policies.
* Loads modules from MinIO per the storage model in 479/480 and exposes an `InvokeExtension` RPC for primitives via the Ameide SDKs.
* All side effects go through Domain/Process/Agent APIs with the caller’s tenant/org/user context; the runtime does not expose direct DB, filesystem, or arbitrary network access.

---

### 3.4 Experience & UI Layer

**Next.js platform app**

* Primary user UI: Next.js app (`www_ameide_platform`) consuming Ameide SDK TS via the proto-based APIs.
* Microfrontends hosted under a shell; per‑domain/per‑process views are separate bundles but share the same Ameide SDK and auth/session model.

**Backstage instance**

* Separate Backstage deployment for:

  * Catalog of all Domain/Process/Agent components. ([Backstage][2])
* Software Templates for creating new primitives. ([Backstage][7])
  * Potential marketplace integration (3rd-party templates/agents).

**Design/Modeling tools**

* **Custom React Flow modeller** for ProcessDefinitions (BPMN-compliant semantics, NOT Camunda/bpmn-js).
* ArchiMate modelers (browser-based) for architecture artifacts.
* Both run inside Next.js or as separate webapps, communicating with the Transformation Domain APIs.

---

## 4. Backstage Implementation as the Ameide "Factory"

> **Important:** Backstage is an **internal factory** for Ameide engineers and transformation agents; **tenants never see Backstage**. Tenant-facing UX is delivered through UISurface primitives (Next.js apps), not through Backstage.

Backstage is where **primitives are born**.

### 4.1 Catalog model

Minimum entity types in the Backstage catalog:

* `Component`:

  * `type: domain-primitive` – Domain service with its proto package + DB schema.
  * `type: process-primitive` – Process service with BPMN + Temporal workers.
  * `type: agent` – LLM-based Agent primitive.
* `System`: groups components into solution domains (e.g. L2O).
* `Resource`: DB clusters, queues, Temporal namespaces associated with components. ([Backstage][2])

### 4.2 Software templates

We define a small set of **opinionated templates**: ([Backstage][7])

1. **Domain primitive template**

   * Generates:

     * Proto service skeleton + messages for the new domain.
     * Go or Python service skeleton (DDD-ish).
     * SQL migrations (Flyway-style) for initial tables.
     * Helm chart (Deployment, Service, VirtualService/HTTPRoute, Prometheus rules).
     * Backstage `catalog-info.yaml` with `type: domain-primitive`.
   * Uses the Buf-based proto pipeline + Ameide SDK import patterns.

2. **Process primitive template**

   * Generates:

     * Initial ProcessDefinition (BPMN-compliant, from React Flow modeller) + modeling project.
     * Temporal workflow worker skeleton in chosen language.
     * Transformation design tooling artifact registration & Transformation API integration.

3. **Agent primitive template**

   * Generates:

     * AgentDefinition file (tools, policies, orchestration graph) stored in Transformation Domain (via Transformation design tooling APIs).
     * Tool stubs for Domain primitive/Process primitive APIs using Ameide SDK TS/Go/Python.
     * Helm chart for Agent primitive inference runtime deployment (autoscaled).

These templates are *also* what agents pick from when “auto‑rolling” a new primitive: the agent writes the template input, not the raw K8s manifests.

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

> **Security**: For the full security model, threat analysis, and implementation gaps (RLS enforcement, `organization_id` columns, API authorization), see [476-ameide-security-trust.md](476-ameide-security-trust.md).

### 6.1 Tenancy model (technical)

* **Identity & access:** Keycloak realms and JWT `tenantId` / `orgId` claims determine tenant context in all services. 
* **Data isolation:**

  * Primary: RLS per tenant in shared schemas (SMB scale).
  * Promotion: heavy tenants can be moved to dedicated DB instances or clusters (managed by infra operators).
* **Execution isolation:**

  * Temporal uses namespaces per tenant tier or segment.
  * Backstage uses labels to group tenant-specific primitives if needed.

### 6.2 Environments

* Standard lanes: `dev`, `staging`, `prod`, optionally `preview/*`.
* Argo CD Apps-of-Apps structure; each environment is a separate Argo Application tree.
* Backstage templates and Ameide primitives always deploy via GitOps—no ad-hoc `kubectl apply`.

### 6.3 Tenant Extension Namespaces

For tenants with custom primitives, additional namespaces are provisioned based on SKU:

| SKU | Product Runtime | Custom Code |
|-----|-----------------|-------------|
| Shared | `ameide-{env}` (shared) | `tenant-{id}-{env}-cust` |
| Namespace | `tenant-{id}-{env}-base` | `tenant-{id}-{env}-cust` |
| Private | `tenant-{id}-{env}-base` | `tenant-{id}-{env}-cust` |

**Security invariant**: Custom code never runs in shared `ameide-*` namespaces. This is enforced at scaffolding time (Backstage templates) and deployment time (GitOps policies).

> **See [478-ameide-extensions.md](478-ameide-extensions.md)** for the complete namespace strategy, repository model, and E2E primitive creation flow.

Tier 1 WASM extensions run inside the shared `extensions-runtime` service in `ameide-{env}`. Modules are treated as tenant-scoped data executed under strict sandbox and host-call policies, so the “no tenant services in `ameide-*` namespaces” invariant still holds.

---

## 7. Observability, Quality & Tooling

* **Tracing & metrics:** OpenTelemetry across services; Temporal emits its own spans/metrics; combined in Grafana.
  * `extensions-runtime` emits per-extension latency/error/resource metrics and integrates with circuit breakers so Tier 1 hooks remain observable.
* **Logging:** Structured logs with correlation IDs (`trace_id`, `tenant_id`, `primitive_type`, `primitive_name`).
* **Testing & quality:**

  * Domain primitives: unit tests + integration tests (DB), contract tests via generated clients.
  * Process primitives: Temporal workflow unit tests + ProcessDefinition lint checks.
  * Agents: tool contract tests; sandbox execution before allowing them to write Backstage templates or code (reuse patterns from the core-platform coder review). 

---

## 8. Migration & Legacy Alignment (IPA → Controllers)

* The previous **IPA architecture** (IPAs as composed workflows/agents/tools) remains a conceptual ancestor; the new stack reimplements:

  * IPA composition → Process primitive + Transformation design tooling artifact.
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
   * Temporal is the only runtime for Process primitives; no Camunda integration planned.

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
| [305‑workflow](305-workflow.md) | Temporal/Process primitive runtime | See §10.1 |
| [310‑agents‑v2](310-agents-v2.md) | Agent primitive layer | See §10.2 |
| [388‑ameide‑sdks‑north‑star](388-ameide-sdks-north-star.md) | SDK publish strategy | See §10.3 |
| [478‑ameide‑extensions](478-ameide-extensions.md) | Namespace topology & extension deployment | See §10.4 |
| [479‑ameide‑extensibility‑wasm](479-ameide-extensibility-wasm.md) | Tier 1 + Tier 2 extension model | See §3.3.6 |
| [480‑ameide‑extensibility‑wasm-service](480-ameide-extensibility-wasm-service.md) | Runtime implementation | See §3.3.6 |

### 10.1 Workflow alignment (305)

* 305 describes Temporal-based workflow orchestration; aligns with §3.2 (Runtime Orchestration Layer).
* ProcessDefinitions (BPMN-compliant, from custom React Flow modeller) are compiled to Temporal workflows.
* Process primitives execute ProcessDefinitions at runtime, backed by Temporal.

### 10.2 Agents alignment (310)

* 310 describes Agent primitive implementation with n8n-style flows; 473 §3.3.3 describes LangGraph-based Agent primitives.
* AgentDefinitions (design-time specs stored in Transformation Domain, modelled via Transformation design tooling UIs) are executed by Agent primitives at runtime.
* **Clarification**: Both n8n-aligned (visual low-code) and LangGraph (code-first) patterns are valid Agent primitive implementations.

### 10.3 SDK alignment (388)

* 388 defines SDK publish strategy; 473 §5 depends on published SDKs for all primitive types.
* See [483-fit-gap-analysis.md](483-fit-gap-analysis.md) §1.3 for current SDK distribution status.

### 10.4 Extension alignment (478)

478 defines the namespace topology for tenant extensions:

* **Shared SKU**: Product in `ameide-{env}`, custom code in `tenant-{id}-{env}-cust`
* **Namespace SKU**: Product in `tenant-{id}-{env}-base`, custom code in `tenant-{id}-{env}-cust`
* **Private SKU**: Product in `tenant-{id}-{env}-base`, custom code in `tenant-{id}-{env}-cust`

**Key invariant**: Custom services always run in dedicated tenant namespaces, never in shared `ameide-*` namespaces.

The E2E flow in 478 §6 describes how primitives are scaffolded via Backstage and deployed via GitOps, extending the patterns in 473 §4.

---

## 11. Notable Gaps & Issues

### 11.1 Terminology alignment

| 473 Term | Related Terms | Notes |
|----------|---------------|-------|
| Domain primitive | DomainService (deprecated) | Consistent across 470-476 ✅ |
| Process primitive | Workflow (305) | Runtime that implements Temporal workflows informed by ProcessDefinitions (code, not dynamic BPMN execution) |
| ProcessDefinition | BPMN model (deprecated) | Design-time artifact in Transformation design tooling (from custom React Flow modeller) |
| Agent primitive | AgentRuntime (deprecated) | Runtime that implements agent behavior informed by AgentDefinitions |
| AgentDefinition | Agent config (deprecated) | Design-time artifact in Transformation design tooling |
| Deployment category (465) | "component domain" (deprecated) | 465 folder categories (apps, data, platform); not business domains |

**Terminology clarification**: "Domain" is reserved for business domains (bounded contexts). 465-style folder categories (apps, data, platform) are now called **deployment categories** to avoid overloading.

### 11.2 Open architectural questions

1. **Backstage vs GitOps static components**
   * 473 §4 envisions Backstage generating new primitives dynamically
   * 465 uses static `component.yaml` files discovered by ApplicationSets
   * **Question**: How do Backstage-generated services integrate with 465 GitOps model?

2. **AmeideTenant CRD vs API-driven**
   * 473 §3.1 suggests `AmeideTenant` CRD for tenant provisioning
   * 333 implements realm provisioning via Keycloak Admin API
   * **Clarification**: Both are valid; CRD is future target

3. **Agent primitive runtime choice**
   * 473 §3.3.3 describes LangGraph-based Agent primitives
   * 310 describes n8n-aligned visual workflows
   * **Resolution**: Both are valid Agent primitive implementations; AgentDefinitions abstract the runtime choice

> **Implementation Status**: See [483-fit-gap-analysis.md](483-fit-gap-analysis.md) §8 for current implementation gaps against this target architecture.

[1]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/?utm_source=chatgpt.com "Custom Resources"
[2]: https://backstage.io/docs/features/software-catalog/?utm_source=chatgpt.com "Backstage Software Catalog and Developer Platform"
[3]: https://docs.temporal.io/workflows?utm_source=chatgpt.com "Temporal Workflow | Temporal Platform Documentation"
[5]: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/?utm_source=chatgpt.com "Operator pattern"
[6]: https://medium.com/%40ajayshekar01/best-practices-for-building-temporal-workflows-a-practical-guide-with-examples-914fedd2819c?utm_source=chatgpt.com "Best Practices for Building Temporal Workflows"
[7]: https://backstage.io/docs/features/software-templates/?utm_source=chatgpt.com "Backstage Software Templates"
