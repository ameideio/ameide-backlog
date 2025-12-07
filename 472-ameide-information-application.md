Here's a first cut of the **Application & Information Architecture** doc, aligned with the vision + business docs we already sketched and wired to vendor concepts (Backstage, BPMN-compliant definitions via custom React Flow modeller, Temporal, K8s, Buf, etc.).

---

# 3 – Application & Information Architecture (Draft)

**Audience:** Platform & domain engineers, solution architects, “platform-facing” agents
**Scope:** How Ameide is structured as software (domains, processes, agents, UI) and how information flows and is stored across the platform.

---

## 1. Objectives & Constraints

### 1.1 What this document does

* Defines the **core application building blocks**:

  * Domain controllers
  * Process controllers
  * Agents
  * UI workspaces / process views
* Describes how **information is modeled**:

  * Per-domain SQL stores
  * Process definitions (BPMN) stored in **Transformation DomainController**
  * Transformation artifacts (managed via UAF UIs, stored by Transformation DomainController)
  * Cross-domain knowledge graph (read-only projection)
* Shows how **Backstage** is used as the “declarative front-door” to design domains/processes/agents.
* Aligns with:

  * Proto-based APIs & SDK strategy
  * Unified Artifact Framework for transformation artifacts
  * Temporal-backed ProcessController execution and SaaS-like ops model.

### 1.2 Constraints & design principles (from previous docs)

We carry forward the earlier principles and make them concrete here:

1. **Process-first**
   *Primary abstraction is an end-to-end process (L2O, O2C, T2C…), not a module (Sales, Procurement). Manual workspaces are second-class views over these processes.*

2. **Universal DDD**
   *All behavior lives in bounded contexts (domains), each with its own model, storage and APIs.*

3. **Transformation as a domain**
   *Transformation (requirements, UAF artifacts, governance) is modeled like O2C: domain + processes + agents, not a side-console.*

4. **Agentic from any angle**
   *Agents can read from knowledge, call domains, and be invoked from processes in a controlled, typed way.*

5. **Proto-first contracts**
   *Every domain/process/agent exposes a proto-based API, with Buf/BSR governance and SDKs.*

6. **Kubernetes-style declarative state**
   *We declare desired state (e.g. “invoice posted”) and controllers + workflows converge actual state, similar to K8s reconciliation loops and CRDs.*

---

## 2. Core Application Building Blocks

### 2.1 Domain Controllers (Domain layer)

**Responsibility:** Encapsulate business concepts, invariants and persistence for a single bounded context.

*Examples:*

* *Sales* domain (Leads, Opportunities, Quotes)
* *Billing* domain (Invoices, Payments, Dunning)
* *Transformation* domain (Initiatives, Backlog Items, UAF Artifacts)

**Characteristics**

* **Deterministic, low-entropy behavior**

  * Pure business rules, minimal AI inside the domain (agents live next to it).
* **Own their data**

  * Dedicated logical schemas / DBs (multi-tenant) and migrations.
* **Expose proto-based APIs**

  * CRUD + business methods encoded in `ameide.<domain>.v1` protos, versioned by Buf.
* **Emit domain events**

  * Changes are published to a streaming layer (K8s events or Kafka, detail in tech doc).

**Information model**

* **Authoritative** for its aggregate roots (Customer, Opportunity, Invoice, Initiative, etc.).
* **Tenant-aware** – every record tagged with `tenant_id`.
* **Projectable** – each domain can opt-in to project parts of its model into the cross-domain Graph for analysis and agent reasoning (see §3.4).

### 2.2 Process Controllers (Process layer)

**Responsibility:** Orchestrate **cross-domain** flows such as L2O, O2C, Procure-to-Pay, or Transformation workflows (Scrum/Togaf ADM).

*Design-time:* **ProcessDefinitions** are BPMN-compliant artifacts produced by a **custom React Flow modeller** and stored in the **Transformation DomainController** (modelled via UAF UIs). At runtime they are executed by **ProcessControllers** backed by Temporal workflows.

**Key concepts**

* **ProcessDefinition** (design-time, stored in Transformation DomainController)

  * BPMN-compliant artifact + metadata (tenant, version, process type like L2O/O2C/T2C).
  * Defines process stages, tasks, gateways, and bindings to domains/agents.
  * Stored and versioned in the **Transformation DomainController** with revisions & promotions (modelled via UAF UIs).
* **ProcessController** (runtime)

  * Logical unit that "executes" a ProcessDefinition:

    * Loads ProcessDefinition from the Transformation DomainController.
    * Maps BPMN tasks to DomainController API calls and/or AgentController tools.
    * Handles business-level errors and compensations.
  * Backed by Temporal workflows.
* **Execution**

  * Long-running process instances in Temporal representing specific business cases.

**Why BPMN-compliant**

* Standard semantics for business users (stages, gateways, lanes).
* Custom React Flow modeller provides visual editing; definitions are BPMN-compliant but NOT Camunda-stack.

**Information model**

* **Process instance state** (per tenant):

  * Process variables (e.g. `leadId`, `quoteId`, `orderId`)
  * Execution state (activity id, token position)
  * Audit trail (who approved, when)
* Stored in **Temporal** plus Ameide projections (for consolidated reporting).

### 2.3 Agents (Agent layer)

**Responsibility:** Non-deterministic, AI-powered components that:

* Read and summarize data (domain stores, graph, logs)
* Propose changes (requirements, configurations, data corrections)
* Generate new domain/process/agent definitions via Backstage templates

*Design-time:* **AgentDefinitions** are declarative specs stored in the **Transformation DomainController** (modelled via UAF UIs). At runtime they are executed by **AgentControllers**.

**Key concepts**

* **AgentDefinition** (design-time, stored in Transformation DomainController)

  * Declarative spec for an agent: tools, orchestration graph, scope, risk tier, policies.
  * Stored alongside ProcessDefinitions in the Transformation DomainController.
  * Subject to governance (review, promotion) before runtime use.
* **AgentController** (runtime)

  * Loads an AgentDefinition from the Transformation DomainController and runs the LLM/tool loop.
  * Enforces scope/risk policies at execution time.
  * Always invoked via domain/process APIs; never becomes source of truth.

**Characteristics**

* **Explicit tool contracts** – each AgentDefinition specifies:

  * Which domain/process APIs the AgentController can call.
  * What UAF / transformation artifacts it can read or modify.
* **Runtime**

  * LangGraph / OpenAI / other inference runtimes orchestrate tools via the AgentController.
* **Determinism boundary**

  * AgentControllers never become the authoritative "source of truth" for domain data; they make proposals or trigger domain commands.

**Information model**

* **AgentDefinitions** (stored per tenant in UAF):

  * Prompt & behavior description
  * Allowed tools & scopes (e.g. "Transformation agent can only touch UAF artifacts and Backstage templates")
  * Risk tier and policy bindings
* **Interaction history**

  * Persisted in Chat/Threads services for audit and context.

### 2.4 UI Workspaces & Process Views

We keep two complementary UI modes:

1. **Traditional “ERP-style” workspaces**

   * List/detail pages over domain entities (Customers, Opportunities, Orders, Invoices).
   * Implemented as microfrontends per domain, using the TS SDK to call proto-based APIs.

2. **Process views** (process-first principle)

   * A timeline / swimlane view of L2O/O2C/T2C, driven by process instance state.
   * Users see “where they are” in L2O, not “which form they’re on”.

Information-wise:

* UI does **not** own data; it binds:

  * Domain state → domain controllers
  * Process state → process controllers
  * Knowledge/analytics → Graph and UAF projections

---

## 3. Information Architecture

### 3.1 Multi-tenant Data Layout (conceptual)

At the information level the platform is **logically multi-tenant**, regardless of how physical databases are configured in the tech layer.

Core concepts:

* **Tenant** (Platform domain)

  * Owns org structure, users, environments, configuration.
* **Tenant isolation**

  * Every domain, process, agent, artifact row carries `tenant_id`.
  * Cross-tenant access only allowed in platform/operations contexts.

We keep **three main categories of data**:

1. **Operational domain data**

   * e.g. leads, quotes, orders, invoices, backlog items, initiatives.
   * Authoritative per domain.

2. **Process execution data**

   * e.g. L2O instances, O2C instances, transformation sprints.
   * Lives in process engines and is projected into Ameide for analytics.

3. **Design / knowledge artifacts**

   * UAF content: BPMN, ArchiMate, Markdown docs, etc.
   * Projection graph: cross-domain graph of entities and relations.
   * Backstage catalog entities: domain/process/agent templates and instances.

### 3.2 Domain Data

Each domain specifies:

* **Canonical aggregates** (e.g. Opportunity, Quote, Order, Invoice, Initiative, BacklogItem).
* **Invariants & state machines** embedded in its APIs (e.g. Invoice can only move from `Draft` → `Posted` via a validated posting command).
* **Tenant-scoped reporting views** (read models) that can be:

  * Direct SQL views
  * Materialized views
  * Projections into Graph

Older IPA architecture already codifies the pattern of “pure domain + persistence + builder/runtime”; we reuse the same layering for domains, but replace “IPA” as an aggregate with explicit Domain/Process definitions.

### 3.3 Process Data

For each process type (L2O, O2C, T2C, etc.) we define:

* **ProcessDefinition** = BPMN-compliant artifact (from custom React Flow modeller) + metadata
* **Process variables schema** = typed variables bound to domain identifiers
* **SLA and policy metadata** (e.g. L2O should close within 30 days)

This mirrors Temporal's distinction between workflow definitions and workflow executions: ProcessDefinitions are design-time artifacts; ProcessControllers execute them at runtime.

We will:

* Store **ProcessDefinitions** in **UAF / Transformation domain** as versioned artifacts.
* Store **execution projections** in Ameide's own DB for:

  * process analytics
  * SLA monitoring
  * process-first UI views.

### 3.4 Knowledge Graph (Read-side only)

The Graph service provides a **cross-domain read model**:

* Entities: Customer, Product, Opportunity, ProcessInstance, Initiative, System, API, etc.
* Relationships: `CUSTOMER->HAS_OPPORTUNITY`, `OPPORTUNITY->FLOWS_THROUGH_L2O`, `SERVICE->IMPLEMENTS_API`.

This is *not* an authoritative store:

* Write path: domains/process controllers project to graph as part of their post-commit flow.
* Read path: agents and analytics use it to understand topology and behavior (e.g. “show me all L2O paths where margin < 10%”).

UAF builds on this philosophy for design artifacts: BPMN / ArchiMate / Markdown are **artifacts**, versioned with commands and snapshots; we treat them as a specialization of the same “artifact + revision” pattern.

### 3.5 Transformation / UAF Data

Transformation is a **domain like any other**, but its data is special:

* **Initiatives & Backlog Items**

  * Standard domain entities (SQL) representing epics, features, tasks.
* **Design-time artifacts (UAF)**

  * **ProcessDefinitions** – BPMN-compliant process models (produced by custom React Flow modeller).
  * **AgentDefinitions** – Declarative agent specs (tools, policies, risk tiers).
  * ArchiMate, Markdown, etc., stored as event-sourced artifacts with commands and snapshots.
* **Governance**

  * Promotion status of artifacts (draft, in review, promoted).
  * Links from artifacts to runtime controllers (e.g. "ProcessDefinition L2O_v3 is executed by ProcessController l2o-process-svc").

Transformation AgentControllers operate primarily on this domain:

* Read backlog + UAF artifacts (ProcessDefinitions, AgentDefinitions).
* Generate or modify Backstage templates and DomainController/ProcessController/AgentController configurations.
* Propose changes that are then applied via standard deployment workflows.

---

## 4. Backstage as the Application Catalog & Template Engine

We standardize on **Backstage** as the catalog and templating layer for:

* Domain controllers
* Process controllers
* Agents
* UI components / microfrontends

This aligns with Backstage’s ecosystem modeling and software templates features.

### 4.1 Catalog modeling

**Backstage entities** (examples):

* `Domain` (custom kind)

  * Represents a bounded context such as *Sales*, *Billing*, *Transformation*.
* `Component` (service)

  * Domain controller service (e.g. `sales-domain-svc`).
  * Process controller service (e.g. `l2o-process-svc`).
* `API`

  * Proto-defined API surface of a domain or process.
* `Resource`

  * External dependencies (DB cluster, Kafka topics, Camunda cluster, Temporal cluster).
* `Template`

  * Domain/Process/Agent/UI templates used by the scaffolder.

**Mapping to Ameide concepts**

| Ameide concept     | Backstage kind                     |
| ------------------ | ---------------------------------- |
| Domain controller  | `Component` (+ custom `Domain`)    |
| Process controller | `Component` + BPMN artifact link   |
| Agent              | `Component` or custom `Agent` kind |
| UI workspace       | `Component` (frontend)             |
| Domain API         | `API`                              |
| Process API        | `API` (process control / queries)  |
| UAF artifact       | `Resource` or custom `Artifact`    |

This keeps all “things that exist” in the platform visible and discoverable via the catalog.

### 4.2 Templates for Domain / Process / Agent / UI

We provide **four base templates** (each parameterized per tenant):

1. **Domain Template**

   * Inputs: domain name, initial aggregates, tenant scope, persistence plan.
   * Generates:

     * Proto definitions & service skeletons (aligned with proto-based API guidelines).
     * Base repository & migrations.
     * Domain events schema.
     * Backstage `Component` + `Domain` entities.
   * Vendor-aligned with Backstage’s Software Templates “skeleton + publish” model.

2. **ProcessController Template**

   * Inputs: process type (L2O, O2C, custom), participating domains, SLA target.
   * Generates:

     * Initial ProcessDefinition (BPMN-compliant, from React Flow modeller) with lane structure (Sales, Billing, Logistics, etc.).
     * Temporal workflow skeletons (service tasks bound to DomainController APIs).
     * ProcessController service skeleton.
     * Backstage `Component` entity + links to domain APIs.

3. **AgentController Template**

   * Inputs: agent purpose, allowed domains/processes, risk tier.
   * Generates:

     * AgentDefinition (prompt, tools, scopes, policies) stored in UAF.
     * Proto definition for the agent configuration & invocation.
     * AgentController service skeleton.
     * Backstage `Component`/`Agent` entity with RBAC metadata for who can use this template.

4. **UI Template**

   * Inputs: target domain/process, view style (workspace vs process view).
   * Generates:

     * Next.js microfrontend skeleton with Ameide TS SDK integration.
     * Backstage `Component` entry of type `frontend`.

Backstage templates remain **the control plane** for agents too: a transformation agent manipulates template parameters and triggers scaffolding rather than directly creating low-level services.

---

## 5. Example: L2O Application & Information Flow

To make this less abstract, here’s how **Lead-to-Opportunity (L2O)** looks in this architecture.

### 5.1 Involved building blocks

* Domain controllers:

  * `SalesDomain` (Lead, Contact, Opportunity)
  * `ProductDomain` (Products, Price Lists)
  * `BillingDomain` (for credit check / pre-invoice)
* Process controller:

  * `L2OProcess` – BPMN definition + controller service.
* Agents:

  * `SalesCoachAgent` (helps sales with next actions)
  * `TransformationAgent` (helps the org refine L2O version)
* UI:

  * `SalesWorkspace` (classic ERP-style)
  * `L2OProcessView` (timeline / swimlane)

### 5.2 Data lifecycle

1. **Lead created**

   * `SalesDomain` persists `Lead` row with `tenant_id`.
   * Domain emits `LeadCreated` event → Graph projects `CustomerCandidate` node.

2. **L2O instance started**

   * `L2OProcessController` is started with `leadId` (Temporal).
   * Process instance state is stored in Temporal; Ameide stores a projection (e.g. "L2O phase = Qualification").

3. **Quote created**

   * `L2OProcessController` reaches "Prepare Quote" task.
   * For manual: Sales user opens `SalesWorkspace`, sees tasks from `L2OProcessView` and creates a Quote via `SalesDomainController` APIs.
   * For automated: an AgentController or automated task calls `SalesDomainController::CreateQuote` using its proto API.

4. **Approval**

   * `L2OProcessController` orchestrates approval steps, maybe invoking:

     * `TransformationAgentController` to suggest better pricing.
     * `SalesCoachAgentController` to highlight risk.
   * AgentControllers read from Graph & domain APIs but write back via domain commands only.

5. **Handover to O2C**

   * Once L2O is "Won", `L2OProcessController`:

     * Calls `BillingDomainController::CreateProformaInvoice`.
     * Emits `OpportunityWon` → `O2CProcessController` starter.

Information picture:

* **Domain tables**

  * `sales_leads`, `sales_opportunities`, `sales_quotes`, `billing_preinvoices`.
* **Process tables / engine**

  * Temporal internal tables for instance execution.
  * Ameide `process_instances` projection for analytics and UI.
* **Graph**

  * Nodes for Customer, Lead, Opportunity, Quote, L2OInstance.
  * Edges for flows (Lead → L2O → Opportunity → O2C).

Agents “see” the world via:

* Graph queries (structure + history).
* Domain APIs (truth).
* UAF artifacts describing L2O design (as the process improves).

---

## 6. Alignment with Existing Specs

This section is just to show that we’re not inventing yet another stack, but reusing patterns from existing Ameide backlogs.

1. **IPA Architecture (043)**

   * IPA = "composition of workflows + agents + tools from DB, compiled to runtimes".
   * New view:

     * *DomainControllers* ≈ "domain and storage" part of IPA.
     * *ProcessControllers* (executing ProcessDefinitions) ≈ "workflows" part.
     * *AgentControllers* (executing AgentDefinitions) ≈ "agents/tools" part.
   * The layered approach (Domain → Storage → Builder → Runtime) remains valid, but the *unit* moves from "IPA" to explicit ProcessDefinitions and AgentDefinitions (design-time) executed by controllers (runtime).

2. **Proto-based APIs (044)**

   * All domain/process/agent services are still proto-first, with gRPC + REST surfaces and consistent error handling.

3. **North-Star (064)**

   * Design vs Deploy vs Runtime separation is kept:

     * Design: Backstage + UAF + transformation domain.
     * Deploy: controllers & process engines.
     * Runtime: domain & process services.

4. **UAF (067)**

   * UAF becomes the storage & governance mechanism for

     * BPMN process designs
     * ArchiMate views, Markdown docs
   * Instead of a generic “IPA designer”, UAF is owned by the **Transformation domain** and used to generate Backstage templates & controller config.

5. **BPMN-compliant definitions (063)**

   * BPMN semantics (stages, gateways, lanes) remain valid; we use a **custom React Flow modeller** (not Camunda/bpmn-js) to produce ProcessDefinitions stored in UAF and executed by Temporal-backed ProcessControllers.

6. **TypeScript SDK (050)**

   * The SDK acts as the canonical way UI and other services call Domain/Process/Agent APIs, keeping the application boundary clean.

---

## 8. Cross-References

This Application & Information Architecture should be read with:

| Document | Relationship | Alignment |
|----------|-------------|-----------|
| [470‑ameide‑vision](470-ameide-vision.md) | Parent vision & principles | Strong ✅ |
| [471‑ameide‑business‑architecture](471-ameide-business-architecture.md) | Business context & journeys | Strong ✅ |
| [473‑ameide‑technology](473-ameide-technology.md) | Technology implementation | Strong ✅ |
| [475‑ameide‑domains](475-ameide-domains.md) | Domain portfolio & patterns | Strong ✅ |
| [300‑ameide‑metamodel](300-ameide-metamodel.md) | Element graph foundation | See §8.1 |
| [305‑workflow](305-workflow.md) | ProcessController implementation | See §8.2 |
| [310‑agents‑v2](310-agents-v2.md) | AgentController implementation | See §8.3 |
| [370‑event‑sourcing](370-event-sourcing.md) | Event sourcing exploration | See §8.4 |

### 8.1 Metamodel alignment (300)

The Element graph (300) and Application Architecture (472) coexist:

* **Graph = read-side projection**: Cross-domain knowledge for agents, analytics, and transformation. Not authoritative for operational data.
* **Domain stores = write-side authority**: Each DomainController owns its data; graph receives projections.
* **UAF artifacts are Elements**: BPMN, ArchiMate, Markdown stored as versioned nodes in the graph.

This means: Transformation domain entities project to `graph.elements`, but ProcessController runtime state stays in Temporal.

### 8.2 Workflow alignment (305)

305 describes platform workflow infrastructure implementing **ProcessControllers** from this document:

* "Platform Workflows" = implementation detail of ProcessControllers (§2.2)
* Temporal is the runtime; customers see "process stages" and "process boards"
* ProcessDefinitions (BPMN-compliant, from custom React Flow modeller) stored via UAF; compiled to Temporal workflows

### 8.3 Agent alignment (310)

310 describes AgentController implementation:

* Current: agents control plane (Go) + agents_runtime (Python) = AgentController implementation
* AgentDefinitions stored in UAF; AgentControllers execute them
* Key constraint: AgentControllers invoked via domain/process APIs; never become source of truth

### 8.4 Event sourcing status (370)

472 §3.3 describes UAF as "event-sourced artifacts". Current reality per 370:

* **Aspirational, not implemented**: Services use direct Postgres, not append-only streams
* **Outbox pattern exists** but not consumed
* **Target path**: Event contracts per domain, outbox + relay, projections from streams

This should be treated as target architecture; current implementation uses traditional persistence.

---

## 9. Notable Gaps & Issues

### 9.1 Terminology alignment

| Legacy Term | 472 Term | Status |
|-------------|----------|--------|
| IPA (Intelligent Process Automation) | DomainController + ProcessController + AgentController bundle | Deprecated; §6 documents mapping |
| Platform Workflows (305) | ProcessController (executes ProcessDefinitions from UAF) | 305 aligned |
| AgentRuntime (310) | AgentController (executes AgentDefinitions from UAF) | Aligned |
| "Graph service" | Knowledge Graph (read projection) | Clarified in §3.4 |
| DomainService | DomainController | Aligned |
| BPMN definition | ProcessDefinition (from custom React Flow modeller) | Aligned |

### 9.2 Implementation gaps

| Gap | Description | Related Docs |
|-----|-------------|--------------|
| Authorization & org isolation | DB lacks `organization_id` columns; RLS not enabled | [329-authz](329-authz.md) ⚠️ Phase 2 not started |
| Transformation domain refactor | Current `services/transformation/` is monolithic; needs decomposition to DomainController + UAF + Backstage bridge | 470 §9.2.2 |
| SDK distribution | TypeScript SDK not published to npmjs; only `dev`/hash tags on GHCR | [388-ameide-sdks-north-star](388-ameide-sdks-north-star.md) ⚠️ |
| Event publishing | Domain events (outbox + bus) not yet live | [370-event-sourcing](370-event-sourcing.md) |
| Backstage integration | Templates & catalog modeling defined; implementation planned | §4 of this doc, [477-backstage.md](477-backstage.md) |

### 9.3 Open architectural tensions

1. **Graph vs. runtime separation**:
   - 472 §3.4 says graph is read-only projection
   - 300 says everything *could* become an Element
   - 305 explicitly excludes platform workflows from graph
   - **Resolution**: Transformation artifacts (ProcessDefinitions, AgentDefinitions) → graph; runtime state → Temporal

2. **Design-time vs runtime clarity**:
   - ProcessDefinitions and AgentDefinitions are design-time artifacts in UAF
   - ProcessControllers and AgentControllers are runtimes that execute them
   - **Clarification**: Temporal backs ProcessControllers; LangGraph/OpenAI backs AgentControllers

3. **Agent complexity gap**:
   - 472 §2.3 describes AgentDefinition/AgentController split
   - 310 describes sophisticated n8n-aligned node registry with catalog streaming
   - **Resolution**: 472 is conceptual model; 310 is current AgentController implementation detail

---

## 10. Next Steps

This doc deliberately stops at the **logical** application & information level.

Next documents will:

1. **Technology Architecture**

   * Map DomainControllers/ProcessControllers/AgentControllers to K8s workloads, CRDs and operators.
   * Detail how Temporal clusters, databases and messaging layers are provisioned and wired.

2. **Domain Architecture (L2O, O2C, Transformation/UAF, etc.)**

   * Provide detailed models for each domain using this framework.

3. **Refactoring Plan**

   * Show how current services (Graph, Transformation, Workflows, Agents) migrate to the Domain/Process/Agent controller model over time.

If you're happy with this level of granularity, we can move on to the **Technology Architecture** doc next.
