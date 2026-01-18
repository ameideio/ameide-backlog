# 4xx – Ameide Platform Vision / Rationale / Principles

**Status:** Draft v1
**Owner:** Architecture / Product
**Intended audience:** Founders, product, principal engineers, domain/solution architects

## Layer header (cross-layer anchor)

- **Primary ArchiMate layer(s):** Cross-layer vocabulary + invariants (Motivation/Strategy/Business/Application/Technology/Implementation & Migration).
- **Primary element types used:** Capability, Value Stream, Business Process, Application Service/Interface/Event, Application Component, Technology Service/Node/System Software, Work Package/Deliverable/Plateau/Gap.
- **Out-of-scope layers:** none (this doc is the root anchor).
- **Secondary layers referenced:** all (this doc is the root anchor).
- **Allowed nouns:** capability/value stream/business process; application component/service/interface/event; technology service/node; work package/deliverable.
- **Prohibited unless qualified:** process, service, domain, event (must be qualified per §0.2).

> **Cross-References (Vision Suite)**:
>
> | Document | Purpose |
> |----------|---------|
> | [471-ameide-business-architecture.md](471-ameide-business-architecture.md) | Business concepts (tenants, orgs, processes) |
> | [472-ameide-information-application.md](472-ameide-information-application.md) | Application & information architecture |
> | [473-ameide-technology.md](473-ameide-technology.md) | Technology stack and GitOps patterns |
> | [475-ameide-domains.md](475-ameide-domains.md) | Domain portfolio and structure |
> | [476-ameide-security-trust.md](476-ameide-security-trust.md) | Security principles and threat model |
> | [478-ameide-extensions.md](478-ameide-extensions.md) | Tenant extension model & namespace strategy |
> | [474-ameide-refactoring.md](474-ameide-refactoring.md) | Migration plan into the six primitives + CRDs |
>
> **Deployment Implementation**:
> - [465-applicationset-architecture.md](465-applicationset-architecture.md) – GitOps deployment model
> - [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) – Rollout phases
> - [442-environment-isolation.md](442-environment-isolation.md) – Environment isolation
> - [446-namespace-isolation.md](446-namespace-isolation.md) – Namespace patterns

> **Comparative Briefs (Ameide vs incumbent ERPs)**:
> - [470a-ameide-vision-vs-d365.md](470a-ameide-vision-vs-d365.md) – Code-first Ameide platform compared to Microsoft D365FO's metadata/AOT paradigm.
> - [470b-ameide-vision-vs-saps4.md](470b-ameide-vision-vs-saps4.md) – Ameide positioning versus SAP S/4HANA's DDIC/CDS/Fiori/Customizing stack.
> - [470c-ameide-vision-vs-oracle.md](470c-ameide-vision-vs-oracle.md) – Ameide positioning versus Oracle Fusion Cloud ERP's SaaS + configuration/extensibility model.
> - [470d-ameide-vision-vs-odoo.md](470d-ameide-vision-vs-odoo.md) – Ameide positioning versus Odoo's modular, Python/ORM + configuration/customization model.
>
> These appendices translate the high-level Ameide principles in this document into head-to-head comparisons with the dominant metadata/configuration ERPs so product, field, and architecture teams can articulate what stays the same (coherence, discoverability) and what changes (code-first, AI operated, CRD-based infra) when pitching Ameide.

---

## Grounding & contract alignment

- **Primitive model:** Establishes the canonical six primitives (Domain, Process, Agent, UISurface, Projection, Integration) and Graph/Transformation invariants that all later primitive/operator/Scrum backlogs (471–477, 495–505, 506–508, 520+) must follow.  
- **EDA/security spine:** Defines the EDA invariants and security assumptions that are expanded in `472-ameide-information-application.md`, `473-ameide-technology.md`, `476-ameide-security-trust.md`, and codified for inter-primitive integration in `496-eda-principles-v6.md`.  
- **Scrum & agents:** Provides the platform-level constraints (Transformation-as-domain, Graph read-only, primitive CRDs only) that the Scrum stack (`367-1-scrum-transformation.md`, `506-scrum-vertical-v2.md`, `508-scrum-protos.md`) and agent backlogs (`505-agent-developer-v2*.md`) refine rather than override.

---

## Implementation Status (2025-02-14)

- ✅ Graph handlers now reject every mutation RPC so the Knowledge Graph remains a projection-only surface (`services/graph/src/graph/service.ts:118-135`, `services/graph/src/graph/service.ts:4879-4905`).
- ✅ The proto→SDK→service chain is enforced through the shared Buf module (`packages/ameide_core_proto/buf.yaml:1-20`) and consumed by workspace services.
- ⚠️ Transformation services still only persist `Transformation` aggregates; ProcessDefinitions/AgentDefinitions are not yet stored in the Transformation domain (`services/transformation/src/transformations/service.ts:1-75`).

## 0. Core Definitions: primitives, CRDs, and Graph

> **Domain primitives** own the authoritative data and rules for a bounded business context (Orders, Customers, Transformation, etc.) and expose proto-first APIs plus migrations/tests as normal code.
>
> **Process primitives** orchestrate multiple Domains and Agents into outcomes such as L2O or O2C by executing BPMN-compliant definitions.
>
> **Agent primitives** wrap non-deterministic workers (LLMs + typed tools) that make proposals but never store durable state themselves.
>
> **UISurface primitives** are user-facing entry points (process views or domain workspaces) implemented as code-first Next.js apps that rely on generated SDKs.
>
> **Projection primitives** are read-optimized query services and materialized views for consumption, analytics, and integration; they are derived from facts and are never sources of truth.
>
> **Integration primitives** are “flows-as-code” runtimes that connect Ameide to external systems; they are contract-first (proto ports) and operator-managed day-2 (sync, parameters, rollout, drift).
>
> **Domain / Process / Agent / UISurface / Projection / Integration CRDs** are the application-level CRDs. Each declares *how* the corresponding primitive is run (image, config, bindings, risk tier) so GitOps + operators can reconcile code into deployments. These CRDs remain a **thin operational metadata layer**: they do not encode business semantics, policy logic, prompts, or provider decisions.
>
> **Transformation design tooling** is the modelling UX (BPMN editor, ArchiMate/diagram editor, Markdown/agent specs) that reads/writes artifacts in the Transformation Domain; it has no independent persistence.
>
> **Graph** is a read-only knowledge layer that projects selected data from any primitive into a graph database so we can traverse relationships without heavy cross-domain joins. All writes go through primitives; Graph is never a source of truth.

---

## 0.1 Glossary

| Term | Definition |
|------|------------|
| **Domain primitive** | Code package (Go/TS/etc.) that owns a bounded context’s data, invariants, migrations, and proto-first APIs. |
| **ProcessDefinition** | BPMN-compliant artifact stored in the Transformation Domain that specifies orchestration intent. |
| **Process primitive** | Runtime implementation (Temporal-backed workflow workers) that executes a ProcessDefinition and calls Domain + Agent primitives. |
| **AgentDefinition** | Declarative spec (tools, orchestration graph, scope, risk tier) stored in the Transformation Domain for a given Agent primitive. |
| **Agent primitive** | Runtime chassis that loads an AgentDefinition and runs LLM/tool loops through Ameide SDKs. |
| **UISurface primitive** | User-facing workspace or process view built in Next.js that consumes Ameide SDKs; no metadata form engine. |
| **Projection primitive** | Read-optimized projection/query service and/or materialized view derived from facts; optimized for consumption/analytics; never a source of truth. |
| **Integration primitive** | Flow runtime that connects Ameide to external systems using proto-declared ports/contracts; operator-managed for lifecycle and drift. |
| **Domain CRD** | Declarative runtime config for one Domain primitive (image, config, DB bindings, resources, observability). |
| **Process CRD** | Declarative runtime config for one Process primitive (image, process type, Temporal namespace/bindings, timeouts). |
| **Agent CRD** | Declarative runtime config for one Agent primitive (image, model/provider config, tool grants, risk tier). |
| **UISurface CRD** | Declarative runtime config for one UISurface primitive (image, routing, auth scopes, feature flags). |
| **Projection CRD** | Declarative runtime config for one Projection primitive (image, storage bindings, refresh policies, query exposure). |
| **Integration CRD** | Declarative runtime config for one Integration primitive (image/runtime type, flow sync refs, endpoint bindings, schedules). |
| **Transformation design tooling** | Ameide’s modelling UX (BPMN editor, ArchiMate/diagram editor, Markdown/agent specs) that stores artifacts in the Transformation Domain; the tooling itself is stateless. |
| **Graph** | Read-only knowledge projection that ingests selected data from primitives into a graph database for traversal; never a source of truth. |

## 0.2 ArchiMate alignment (language + layering + relationships)

This section is the canonical language anchor for `470+` docs (see also `529-archimate-alignment-470plus.md`). Ameide uses **ArchiMate 3.2** vocabulary to keep “business vs application vs technology” unambiguous.

ArchiMate is not just prose discipline: **ArchiMate 3.2 notation is Ameide’s official design-time language for architecture and capability models**, produced/edited via Transformation design tooling and persisted as Transformation-domain artifacts (see `471-ameide-business-architecture.md`, `472-ameide-information-application.md`, and the capability worksheet `530-ameide-capability-design-worksheet.md`).

Design-time language rule of thumb:

- **ArchiMate (3.2)**: capability/value-stream/application/technology models and views (design-time architecture artifacts).
- **BPMN (2.0, BPMN-compliant)**: ProcessDefinitions (design-time orchestration intent).
- **Markdown**: narrative specs/decision records that glue together the above.
- **Protobuf (Buf modules)**: the application boundary for runtime contracts (services/messages/envelopes).

### Layer model (what belongs where)

- **Motivation (optional):** principles, drivers, constraints.
- **Strategy:** capabilities, value streams, courses of action, resources.
- **Business:** business processes, policies, roles.
- **Application layer:** application services/interfaces/events, data objects, application components.
- **Technology layer:** runtime topology (Kubernetes/operators/brokers/DBs/gateways), nodes, technology services.
- **Implementation & Migration:** work packages, deliverables, rollout phases, fit/gap.

### Crosswalk (ArchiMate → Ameide)

| ArchiMate concept | Ameide concept |
|---|---|
| Capability (Strategy) | Capability docs (e.g., `523`, `527`, `528`) |
| Value Stream (Strategy) | capability “golden paths” + process/value-stream maps |
| Business Process (Business) | business process narratives; realized in runtime by Process primitives |
| Application Component (Application) | primitives: Domain / Process / Projection / Integration / UISurface / Agent |
| Application Service (Application) | proto RPC/query services (behavior exposed) |
| Application Interface (Application) | gRPC/HTTP endpoints + topic families/subjects |
| Application Event (Application) | facts (domain facts and process facts; state change) |
| Technology Service / Node / System Software (Technology) | K8s, gateways, brokers, DBs, workflow engines, operators |
| Work Package / Deliverable (Implementation & Migration) | phased rollout plans, migrations, templates |

### Canonical relationship chain (required verbs)

1. Capability **serves** Value Stream.
2. Value Stream **is realized by** Business Processes.
3. Business Processes **use (are served by)** Application Services.
4. Application Services **are realized by** Application Components (primitives).
5. Application Components **use** Technology Services.
6. Implementation & Migration elements describe *change work* and trace to the above; they are not runtime architecture elements.

### EDA taxonomy mapping (intent/fact/query)

- Fact (domain fact / process fact) → **Application Event** (state change; name as facts).
- Intent / command → request to invoke an **Application Service** (often asynchronously over an Application Interface).
- Query → read-only **Application Service** (often realized by a Projection primitive).

### Disambiguation rules (overloaded terms)

| Overloaded word | Required form |
|---|---|
| process | Business Process / Process primitive |
| service | Business Service / Application Service / Technology Service |
| domain | business domain / bounded context / Domain primitive |
| event | Application Event (fact) / message (transport) |

> **Ameide Core Invariants (Canonical Reference)**
>
> All vision suite documents (470-480) must respect these invariants. Link here rather than restating.
>
> 1. **Six primitives (thin CRD layer).** Domain / Process / Agent / UISurface / Projection / Integration are the only application-level CRD types; no app-level CRDs for low-level concepts (tables, fields, forms, etc.).
>
> 2. **Graph is read-only.** Knowledge Graph is a projection layer for cross-domain queries. All writes go through Domain primitives. Graph is never a source of truth.
>
> 3. **Transformation is a domain; Transformation design tooling is UI.** The Transformation Domain primitive owns all design-time artifacts (ProcessDefinitions, AgentDefinitions, BPMN, etc.). Transformation design tooling is purely a UI layer (BPMN editor, diagram editor, Markdown editor) that calls Transformation Domain APIs—it has no independent persistence.
>
> 4. **Proto → SDK → Runtime chain.** Contracts live in `packages/ameide_core_proto` (Buf-managed). Generated SDKs flow to services. Docker images are tagged/labelled with proto version. GitOps manifests reference exact image + proto metadata. See [472 §2.7](472-ameide-information-application.md) for details.
>
> 5. **Tenant isolation via `tenant_id`.** Every business record carries `tenant_id`. Organization-level isolation via `organization_id` + RLS is the target model.
>
> 6. **CRD naming is Domain / Process / Agent / UISurface / Projection / Integration.** Earlier documents (e.g., 461) used `IntelligentDomainController (IDC)`, `IntelligentProcessController (IPC)`, `IntelligentAgentController (IAC)`. These names are **deprecated**.
>
> 7. **Backstage is internal only.** Backstage is the factory for Ameide engineers and transformation agents. Tenants never see Backstage; tenant UX is delivered through UISurface primitives.

> **Event-Driven Architecture (EDA) Invariants**
>
> All Domain/Process/Agent primitives must follow these EDA principles. See [472 §3.3](472-ameide-information-application.md) for implementation details.
>
> 8. **Commands express business intent, not CRUD.** APIs use business verbs (`PlaceOrder`, `ApproveQuote`) not generic operations (`UpdateOrder`, `SetStatus`). See [472 §2.8.6](472-ameide-information-application.md).
>
> 9. **Events are immutable facts.** Every state-changing command emits at least one domain fact as a past-tense fact (`OrderPlaced`, `QuoteApproved`). This aligns with ArchiMate’s Application Event meaning: a state change represented as a fact.
>
> 10. **Transactional outbox required.** Domain primitives MUST write events to an outbox table in the same transaction as the aggregate update. Never call `publisher.Publish()` directly from domain code. See [472 §3.3.1.1](472-ameide-information-application.md).
>
> 11. **Consumers must be idempotent.** All event handlers must handle duplicates gracefully via inbox pattern or natural key idempotency. Assume at-least-once delivery. See [472 §3.3.2](472-ameide-information-application.md).
>
> 12. **Domain primitives don't import broker clients.** All event publishing goes through outbox interfaces. Domain logic is isolated from Watermill, NATS, Kafka, or other broker implementations. See [473 §3.2.1](473-ameide-technology.md).
>
> 13. **Events carry tenant context.** Every event message includes `tenant_id`. Consumers validate tenant context matches before processing. See [472 §3.3.7](472-ameide-information-application.md).

> **Security Spine**
>
> Security is a cross-cutting concern addressed in [476-ameide-security-trust.md](476-ameide-security-trust.md). Key areas:
>
> - **Tenant/org isolation (RLS)**: Target model requires `tenant_id` + `organization_id` on all business tables with Row-Level Security. See [476 §3](476-ameide-security-trust.md) and [329-authz](329-authz.md).
> - **Agent governance**: AgentDefinitions carry `scope`, `risk_tier`, and `tool_grants` to constrain what agents can do. See [476 §8](476-ameide-security-trust.md) and [310](310-agent-runtime.md).
> - **Extension isolation**: Tier 1 WASM runs in sandboxed `extensions-runtime`; Tier 2 primitives run in isolated `tenant-{id}-{env}-cust` namespaces. See [476 §6](476-ameide-security-trust.md) and [478](478-ameide-extensions.md)/[479](479-ameide-extensibility-wasm.md)/[480](480-ameide-extensibility-wasm-service.md).

> **Terminology Migration (Legacy → Current)**
>
> The 47x vision suite establishes canonical vocabulary. Older documents (461, 465, earlier drafts) use terms that are now **deprecated**. This table provides the mapping:
>
> | Legacy Term | Current Term | Notes |
> |-------------|--------------|-------|
> | IDC (IntelligentDomainController) | **Domain CRD** | 461 naming; deprecated |
> | IPC (IntelligentProcessController) | **Process CRD** | 461 naming; deprecated |
> | IAC (IntelligentAgentController) | **Agent CRD** | 461 naming; deprecated |
> | "component domain" (465) | Chart folder | 465 used "domain" for Helm chart directories; unrelated to Domain primitives |
> | "Transformation tooling service" | **Transformation design tooling** (UI only) | Earlier docs implied a separate runtime service; there is none—tooling is stateless UI calling Transformation Domain APIs |
> | "Agent domain" | **Transformation Domain** | Agent configs (AgentDefinitions) live in Transformation Domain, not a separate "Agent domain" |
> | IPA (Intelligent Process Automation) | Domain + Process + Agent primitive bundle | Marketing term; deprecated in technical docs |
>
> **Action**: When encountering legacy terms in code, comments, or older backlogs, use the current vocabulary. Do not re-introduce "Transformation tooling service" as a runtime—it does not exist.

---

## 1. Purpose

This document defines **why Ameide exists**, **what we are building**, and the **principles** that constrain future design decisions.

It sits above business, application, and technology architecture docs and provides a common language for:

* How we think about **domains, processes, agents, and UIs**
* How **transformation itself** is modelled as a first-class domain
* How Backstage, Transformation design tooling, BPMN, and the current proto/SDK stack come together

---

## 2. Problem & Opportunity

### 2.1 The problem with today’s ERP / CRM / HCM landscape

Typical enterprise stacks:

* Split work across many siloed systems: ERP, CRM, HCM, MES, project tools, etc.
* Treat **“implementation / transformation” as a consulting project**, not a product capability.
* Force customers into:

  * brittle customisations
  * long upgrade cycles
  * opaque “what is actually running” state.

Even in Ameide today, powerful building blocks (onboarding, proto APIs, Transformation design tooling, agents, workflows) are still experienced as **bespoke services**, not a coherent business platform.   

### 2.2 Opportunity

We can build a **cloud-native business platform** where:

* “ERP/CRM/HCM/MES” collapse into **domains + processes + agents + UIs** on a single substrate.
* **Transformation** (roadmaps, designs, deployments) is modelled as a domain **inside the platform**, not as a separate consulting lifecycle. 
* Customers express requirements in natural language; **agents + templates + GitOps** turn them into running capabilities.
* We reuse and extend existing strengths:

  * Proto-based APIs and clean architecture.
  * Production-ready TS SDK for web UIs.
  * Transformation design tooling as modelling UIs for artifacts (BPMN, architecture diagrams, Markdown specs), with all artifacts persisted by the Transformation Domain.
  * BPMN-compliant ProcessDefinitions executed by Temporal-backed Process primitives.
  * Existing onboarding & tenancy model. 

---

## 3. Vision

> **Ameide is a universal, cloud‑native business platform where domains, processes, agents, and workspaces are declared, generated, and operated as first‑class platform resources — and where transformation itself is modelled as just another domain.**

### 3.1 Core outcomes

1. **No more one‑off ERPs/CRMs**
   Customers assemble **domains** (Orders, Customers, Transformation, Identity, Billing…) and **process primitives** (L2O, O2C, Onboarding, Agile, TOGAF) instead of buying separate suites.

2. **Transformation is inside the product**
   The same platform that runs L2O/O2C also runs the **transformation processes** that define and evolve them, with artifacts managed by the **Transformation Domain** and modelled through Transformation design tooling UIs.

3. **Process‑first, workspace‑second**
   Users primarily work in **process views** (e.g. L2O stages), with classic entity workspaces (Orders, Customers) as supporting tools, not the main entry point. 

4. **Agentic everywhere, but controlled**
   Agents assist in:

   * Understanding current architecture and process performance.
   * Suggesting new variants or domains.
   * Wiring Backstage templates and config.
     But deterministic domains + processes remain the source of truth; agents are **non-deterministic helpers**, never the authority.

5. **Design ≠ Deploy ≠ Runtime**
   We maintain a strict separation between:

   * **Design-time**: ProcessDefinitions, AgentDefinitions, ExtensionDefinitions, and other Transformation design tooling artifacts stored in Transformation.
   * **Deployment-time**: Backstage templates, GitOps, and `Domain` / `Process` / `Agent` / `UISurface` / `Projection` / `Integration` CRs committed to Git.
   * **Runtime**: Operators reconcile those CRDs into Deployments, Services, Temporal workers, and other workloads on Kubernetes.

---

## 4. Conceptual model

At the highest level we standardise on **six primitives**, with a clear **design-time vs runtime** split:

### 4.0 Design-Time Artifacts (owned by the Transformation Domain)

* **ProcessDefinition** – BPMN-compliant artifact produced via Transformation design tooling.
  * Defines process stages, tasks, gateways, and bindings to Domain/Agent primitives.
  * Stored and versioned in the Transformation Domain with revisions & promotions.

* **AgentDefinition** – Declarative spec for an agent.
  * Tools (Domain/Process APIs), orchestration graph, scope, risk tier, policies.
  * Stored alongside ProcessDefinitions in the Transformation Domain.

* **ExtensionDefinition & other artifacts** – Markdown specs, architecture diagrams, Backstage template configs, etc., all persisted by the Transformation Domain and addressed like any other domain records.

### 4.1 Runtime primitives & CRDs

Every running primitive is described by exactly one CRD so Git remains the source of desired state and operators can reconcile code into Deployments/Services/Temporal workers/agent runtimes.

1. **Domain primitive**

   * Owns proto-first APIs, domain rules, persistence, and events.
   * Examples: Orders, Customers, Transformation, design tooling, Onboarding, Identity.
   * Deterministic, versioned, testable.
   * **Runtime representation**: `Domain` CRD declares image, config, DB bindings, resources, observability. Operators reconcile it into Deployments, Services, CNPG schemas, HPAs, ServiceMonitors, etc.

2. **Process primitive**

   * Loads a specific ProcessDefinition version from the Transformation Domain.
   * Manages process instances, retries, compensation, and lifecycle metrics.
   * Backed by Temporal; binds BPMN tasks to Domain and Agent primitives.
   * **Runtime representation**: `Process` CRD declares image, Temporal namespace/bindings, rollout strategy, queues/topics, and timeout policy.

3. **Agent primitive**

   * Loads an AgentDefinition and runs the LLM/tool loop with typed tools via SDKs.
   * Enforces scope/risk policies and produces deterministic proposals the Domains/Processes persist.
   * **Runtime representation**: `Agent` CRD declares image, model/provider config, allowed tools, and risk tier plus observability hooks.

4. **UISurface primitive**

   * Next.js workspaces or process views that expose domain workspaces and process timelines using the TS SDK.
   * **Runtime representation**: `UISurface` CRD declares image, routing, auth scopes, feature flags, and dependencies on Domain/Process/Agent primitives.

5. **Projection primitive**

   * Read-optimized query services and materialized views derived from Domain facts/events and/or transactional data.
   * Used for fast UI queries, analytics, cross-domain views, and integration read models; never a source of truth.
   * **Runtime representation**: `Projection` CRD declares image, storage bindings, ingestion bindings, refresh/backfill policy, and query exposure.

6. **Integration primitive**

   * “Flows-as-code” runtimes that connect Ameide to external systems using proto-declared ports/contracts.
   * Operator-managed day-2 operations (sync, schedules, drift detection, retries) and secrets/config injection (no secrets in proto).
   * **Runtime representation**: `Integration` CRD declares runtime type, flow sync refs, endpoint bindings, schedules, and rollout strategy.

These are **logical roles** implemented by concrete services described in the proto/API and North‑Star docs (graph/repository/platform/transformation/workflows/agents/chat/threads/www_*).

### 4.2 Transformation domain & design tooling

* **Transformation Domain**

  * Stores: initiatives, epics, backlogs, architecture decisions, ProcessDefinitions, AgentDefinitions, ExtensionDefinitions, Backstage template configs.
  * Coordinates Agile/TOGAF-style transformation processes via Process primitives.
  * Single source of truth for all design-time artifacts—there is no separate “design tooling service” in the runtime.

* **Transformation design tooling**

  * A frontend modelling layer (BPMN editor, diagram editor, Markdown + agent editors) that calls Transformation Domain APIs.
  * Sends commands to Transformation; owns zero storage and simply presents versioning/promotion UX backed by Transformation data.

### 4.3 Backstage as "factory"

Backstage is the **factory** that turns Transformation Domain decisions into running components:

* **Catalog**:

  * Indexes Domain, Process, Agent, UISurface, Projection, and Integration primitives plus their sources (repos, Helm releases, proto APIs).
* **Templates**:

  * Scaffold new primitives using standard Ameide patterns (proto, SDK, GitOps/operators) and emit the corresponding CRDs (Domain/Process/Agent/UISurface/Projection/Integration) that GitOps/Argo will apply.
* **Bridge**:

  * Listens to Transformation domain facts (Application Events) and runs templates with specific parameters (e.g. "create L2O Process primitive variant for tenant X").

> **Extension Model**: For tenant-specific primitives and custom code isolation, see [478-ameide-extensions.md](478-ameide-extensions.md) which defines the namespace strategy by SKU and the E2E flow for primitive creation.

### 4.4 Proto-first contracts (Buf + SDK chain)

All primitives follow a single, enforced chain for contracts and runtime artifacts:

1. **Schema layer (Buf)** – `packages/ameide_core_proto` hosts the Buf module for every Domain/Process/Agent surface. Buf breaking-change checks run in CI (see [365-buf-sdks-v2.md](365-buf-sdks-v2.md)). Generated outputs land inside the SDK packages (TS/Go/Python) so services never import Buf registry stubs directly.
2. **Implementation layer (workspace SDKs)** – Services consume AmeideClient + proto types from the workspace SDKs per [402](402-buf-first-generated-clients.md), with language-specific rollout plans in [403](403-python-buf-first-migration-plan.md) and [404](404-go-buf-first-migration.md). Proto diffs therefore break dev/CI immediately.
3. **Runtime layer (GitOps)** – Docker images are built from those workspace SDKs and referenced by GitOps manifests (Argo CD). Metadata/annotations capture the proto + image versions (see [472 §2.7](472-ameide-information-application.md)).
4. **Publish track (outer loop)** – SDKs/configs still ship as product artifacts for external consumers per [388-ameide-sdks-north-star.md](388-ameide-sdks-north-star.md), but internal Rings 1/2 never wait on publishes.

This keeps “proto → SDK → deployment” consistent across every primitive and is the basis for tenant-specific extensions and catalog metadata referenced throughout the 47x suite.

### 4.5 Event-driven orchestration (Watermill CQRS)

Where Process/Domain primitives run inside Go services, we rely on Watermill’s CQRS component to implement command and fact/event handlers with minimal boilerplate. Process primitives send commands (`CommandBus`) and publish facts (`EventBus`) as plain Go structs; Watermill handles serialization, topic selection, and pub/sub integration (Kafka, RabbitMQ, Postgres, JetStream, etc.). Downstream read models subscribe via `EventProcessor` handlers. See [472 §3.3.1](472-ameide-information-application.md#331-event-driven-cqrs-runtime-watermill) for details and references. Other languages follow the same CQRS semantics even when they don’t use Watermill directly.

---

## 5. Principles

### 5.1 Domain & Data Principles

1. **Universal DDD, proto-first**

   * Every domain we expose is:

     * Modelled explicitly (bounded context, entities, aggregates).
     * Exposed via **proto contracts** from a single source of truth (`ameide_core_proto`).
   * Public access is through SDKs (TS/Go/Python), not ad-hoc stubs. 

2. **Multi-tenancy as a first-class concern**

   * Tenant is always explicit: in tokens, DB schemas/RLS, and APIs.
   * Onboarding/Identity design (realm-per-tenant for enterprise, shared realms for SMB) is canonical and reused by all domains. 

3. **Deterministic primitives, non-deterministic agents**

   * Domain primitives + Process primitives are **deterministic**, versioned, and testable.
   * Agent primitives are allowed to be non-deterministic, but:

     * They can only act through public domain/process APIs.
     * Their proposals become durable only once written into the Transformation Domain.

4. **Process‑first modelling**

   * Processes like L2O, O2C, Onboarding, Scrum, TOGAF are defined as ProcessDefinitions (BPMN-compliant) in Transformation Domain (modelled via Transformation design tooling UIs).
   * Process primitives execute these definitions via Temporal.
   * Domain design starts by asking "which process stages does this domain support?"

5. **Graph as read-only projection, not runtime truth**

   * **Graph** holds read-only projections for analysis; designs and histories live in Domain primitives (primarily Transformation) and can be projected into Graph as needed.
   * Graph is never a source of truth; all writes go through primitives.
   * Each Domain primitive controls its own persistence; Graph merely indexes selected data for cross-domain queries.

### 5.2 Transformation & Governance Principles

6. **Transformation is a domain like any other**

   * The Transformation Domain owns:

     * Transformation initiatives & roadmaps.
     * **ProcessDefinitions** and **AgentDefinitions** as first-class Transformation design tooling artifacts.
     * Transformation design tooling artifacts (BPMN, ArchiMate, Markdown).
     * Backstage template configurations.
   * Change to the platform is expressed as **events and configs** in this domain, not ad-hoc pipelines.

7. **Design → Review → Generate → Deploy**

   * The canonical lifecycle is:

     1. Design change in Transformation design tooling / Transformation (possibly with Agent primitive assistance).
     2. Human review / governance (change boards, risk tiering).
     3. Backstage template run → new Domain primitive/Process primitive/Agent primitive/UISurface.
     4. GitOps/Helm/Argo rollout.

8. **Self-development over time**

   * The long-term goal: most new capabilities are produced by Agent primitives **inside the Transformation domain**, guided by ProcessDefinitions/AgentDefinitions in Transformation design tooling and Backstage templates, with humans mostly approving and course-correcting rather than hand-writing code.

9. **IPA as a legacy concept**

   * IPA (Intelligent Process Automation) remains an **internal historical concept**, now decomposed into:

     * a bundle of Domain primitives + Process primitives + Agent primitives.
   * No new user-visible "IPA" product surface; everything is explained in terms of domains/processes/agents.

### 5.3 Engineering & Platform Principles

10. **Kubernetes & GitOps native**

    * All long-running components run on K8s with ArgoCD/Helm GitOps, ExternalSecrets/Vault, CNPG Postgres, and hardened charts. 

11. **North‑Star: client-side first, immutable artifacts, stateless edges**

    * Follow the North‑Star blueprint:

      * Keep modelling/editor stacks as client-heavy (IndexedDB/command stacks, BPMN editors).
      * Treat design artifacts as immutable, content-addressed blobs with revision history (via Transformation design tooling). 

12. **Single proto & SDK contract layer**

    * All services use `ameide_core_proto` and generated SDKs; no direct wire-level improvisation.
    * Frontends always go through the TS SDK; agents and workers use Go/Python SDKs.
    * **Command/event naming discipline**: APIs express business intent (`PlaceOrder`, `ApproveQuote`), not field changes (`UpdateOrder`, `SetStatus`). See [472 §2.8.6](472-ameide-information-application.md) for the CQRS pattern.

13. **Clean separation of concerns**

    * **Design**: ProcessDefinitions/AgentDefinitions in custom React Flow modeller, stored in Transformation Domain.
    * **APIs**: pure proto-based domain APIs (044).
    * **Runtime**: Domain primitives, Process primitives (Temporal-backed), Agent primitives on K8s.
    * **SDK / tools**: TS SDK, core-platform-coder, Backstage templates driving these layers.

14. **Observability, always-on**

    * Tracing, metrics, and logs are not optional; they are part of acceptance criteria for any new Domain primitive/Process primitive/Agent primitive.
    * North‑Star and onboarding specs define core telemetry attributes (tenant, organization, onboarding id, workflow id).

15. **Security and tenancy over convenience**

    * Secrets authority model: CNPG for DB creds, Vault + ExternalSecrets for app secrets. 
    * Realm-per-tenant for enterprise, shared realm for SMB with strong tenant isolation. 

### 5.4 UX & Experience Principles

16. **Process-first user experience**

    * Primary entrypoints are **process views** (L2O board, Onboarding funnel, Transformation roadmap). Workspaces for entities (Orders, Leads, Tenants) are secondary navigation. 

17. **Agentic assistance, not magic**

    * Agents:

      * Explain what they are doing.
      * Propose changes in understandable terms (BPMN diff, domain model changes, Backstage template params).
      * Do not bypass review steps or governance.

18. **Symmetry between “run the business” and “change the business”**

    * The same patterns (domains, processes, agents, workspaces) are used both for:

      * Operational processes (L2O, O2C, Onboarding).
      * Transformation processes (Agile, TOGAF, architecture review).

---

## 6. Strategic Themes & KPIs

We measure progress towards this vision with a small set of cross-cutting KPIs (detailed in other docs, but referenced here for alignment):

1. **Time from requirement to running capability**

   * From “tenant expresses new L2O variant requirement” to “variant running in production” (target: < days → eventually hours).

2. **Percentage of changes executed through Transformation + Backstage**

   * Target: majority of new domains/processes/agents/workspaces originate as:

     * Transformation records + Transformation design tooling artifacts.
     * Backstage template runs, not bespoke scripting.

3. **Process-first adoption**

   * Share of daily active users that enter via process views vs entity tables.

4. **Agent-assisted vs manually implemented changes**

   * How many changes to domains/processes/agents are **agent-proposed and human-approved** vs purely manual.

---

## 7. Legacy alignment & reuse

To ensure we don't lose prior work:

* **IPA Architecture**

  * IPA remains as a conceptual ancestor of "Domain + Process + Agent bundles", but new design and implementation use the Domain primitive / Process primitive / Agent primitive pattern.

* **Proto-based APIs**

  * The proto/REST/gRPC design and the API infrastructure backlog remain the foundation of Domain primitive patterns; new domains must follow them.

* **TypeScript SDK**

  * The SDK is the reference client for UISurfaces and many Agent primitives; all frontend workspaces must integrate via this path.

* **North-Star & Transformation design tooling**

  * North-Star defines the design/deploy/runtime separation and client-side modelling.
  * Transformation design tooling defines ProcessDefinitions, AgentDefinitions, and other artifacts with revisions and promotions; it becomes the core of the Transformation domain's knowledge layer.

* **core-platform-coder and internal agents**

  * These become specific Agent primitive implementations we can generalise from (robust tool usage, scoring, state typing).

* **Onboarding & identity**

  * The onboarding backlog is the canonical specification for tenancy, identity, and the first big process (tenant lifecycle) we must express as a ProcessDefinition + Process primitive.

---

## 8. Next documents in the stack

This Vision / Rationale / Principles doc is intentionally high-level. The following documents refine it:

1. **Business Architecture** – how tenants, orgs, roles, and transformation journeys use the platform.
2. **Application & Information Architecture** – mapping Domain primitive/Process primitive/Agent primitive/UISurface to concrete services & data flows; ProcessDefinitions and AgentDefinitions as design-time artifacts.
3. **Technology Architecture** – detailed runtime stack (K8s, Temporal-backed Process primitives, Backstage, SDKs, infra operators).
4. **Domain Specifications** – L2O/O2C, Transformation, Onboarding/Identity, etc.
5. **Refactoring & Migration Plan** – how we move from the existing microservices + IPA builder to this model.

Those docs should all **inherit and reference these principles**; any deviations should be called out explicitly and treated as architecture decisions.

---
## 9. Cross-References

> **Implementation Status**: For current implementation status, gaps, and alignment tracking, see [483-fit-gap-analysis.md](483-fit-gap-analysis.md).

### 9.1 Vision Suite

- [470-ameide-vision.md](470-ameide-vision.md) – This document
- [471-ameide-business-architecture.md](471-ameide-business-architecture.md) – Business use cases, personas, journeys
- [472-ameide-information-application.md](472-ameide-information-application.md) – Application & information architecture
- [473-ameide-technology.md](473-ameide-technology.md) – Technology stack blueprint
- [475-ameide-domains.md](475-ameide-domains.md) – Domain architecture
- [476-ameide-security-trust.md](476-ameide-security-trust.md) – Security architecture

### 9.2 Core Principles

- [000-ameide-principles.md](000-ameide-principles.md) – Foundation principles

### 9.3 Multi-Tenancy & Identity

- [333-realms.md](333-realms.md) – Realm-per-tenant target architecture
- [331-tenant-resolution.md](331-tenant-resolution.md) – JWT-based tenant resolution
- [443-tenancy-models.md](443-tenancy-models.md) – SKU matrix (Shared/Namespace/Private)
- [329-authz.md](329-authz.md) – Authorization

### 9.4 Secrets & Infrastructure

- [451-secrets-management.md](451-secrets-management.md) – E2E secrets flow
- [462-secrets-origin-classification.md](462-secrets-origin-classification.md) – Authority model

### 9.5 SDKs & APIs

- [388-ameide-sdks-north-star.md](388-ameide-sdks-north-star.md) – SDK publish strategy
- [393-ameide-sdk-import-policy.md](393-ameide-sdk-import-policy.md) – Import guardrails

### 9.6 Agents & Transformation

- [310-agents-v2.md](310-agents-v2.md) – Agent primitive layer
- [305-workflow.md](305-workflow.md) – Workflow orchestration (Temporal-backed Process primitives)
