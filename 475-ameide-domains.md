Got it — let's re-cut the **Domains Architecture** layer with those corrections baked in.

> **Cross-References (Vision Suite)**:
>
> | Document | Purpose |
> |----------|---------|
> | [470-ameide-vision.md](470-ameide-vision.md) | Vision and principles |
> | [471-ameide-business-architecture.md](471-ameide-business-architecture.md) | Business concepts |
> | [472-ameide-information-application.md](472-ameide-information-application.md) | Application architecture |
> | [473-ameide-technology.md](473-ameide-technology.md) | Technology stack |
> | [476-ameide-security-trust.md](476-ameide-security-trust.md) | Security architecture |
>
> **Deployment Implementation**:
> - [461-ipc-idc-iac.md](461-ipc-idc-iac.md) – Infrastructure as Code patterns
> - [465-applicationset-architecture.md](465-applicationset-architecture.md) – GitOps deployment model
> - [464-chart-folder-alignment.md](464-chart-folder-alignment.md) – Chart folder structure by domain

> **Alignment Note**: This document describes domain architecture patterns
> that align with the design-time/runtime split defined in [471‑ameide‑business‑architecture](471-ameide-business-architecture.md):
>
> - **ProcessDefinitions** (design-time): BPMN-compliant artifacts from custom React Flow modeller, stored in **Transformation DomainController** (modelled via UAF UIs)
> - **ProcessControllers** (runtime): Execute ProcessDefinitions, backed by Temporal
> - **AgentDefinitions** (design-time): Declarative agent specs stored in **Transformation DomainController** (modelled via UAF UIs)
> - **AgentControllers** (runtime): Execute AgentDefinitions
>
> **Core Definitions** (see [470-ameide-vision.md §0](470-ameide-vision.md)):
> - **UAF** is the set of modelling UIs—it does not own storage. All UAF-originated artifacts are persisted by the **Transformation DomainController**.
> - **Graph** is a read-only knowledge projection; all writes go through controllers.
>
> For cross-references and gaps analysis, see §10 and §11.

---

# 475 – Domains Architecture (Universal Business Platform)

**Status:** Draft
**Audience:** Domain & process architects, product owners, platform engineers
**Scope:** How Ameide defines and structures domains and processes so that *any* ERP/CRM/HCM/general‑business scenario (not just L2O/O2C) can be modeled and implemented using DomainControllers, ProcessControllers, and Agents.

> L2O/O2C are **examples**, not special cases. Product, Pricing, HR, Finance, etc. are domains like any other.

---

## 1. Purpose

This document explains:

* **Which domains exist conceptually in Ameide** (now and in the future).
* **How those domains are represented technically** via DomainControllers, ProcessControllers and Agents.
* **How “universal” end‑to‑end processes** (like L2O, O2C, P2P, H2R, RtR) are built by composing domains.
* **Where transformation/UAF and the knowledge graph fit** into the domain picture.

It sits between:

* **Business Architecture** (who uses what, for which value streams), and
* **Application/Technology Architecture** (K8s, Temporal, Backstage, SDKs, etc.).

---

## 2. Domain Principles

1. **Universal coverage, specific domains**

   * The platform aspires to cover *all* business processes that today live in ERP, CRM, HCM, MES, etc.
   * It does **not** do this via one giant meta‑model but via *many small, well‑defined domains* (Product, Pricing, Orders, Workforce, etc.) each owned by a DomainController.

2. **Processes first, domains reusable**

   * End‑to‑end processes (L2O, O2C, P2P, H2R…) are modeled as **ProcessControllers** orchestrating multiple domains.
   * Domains are reusable building blocks across many processes (e.g. `Product` is used by L2O, O2C, P2P, M2O, etc.).

3. **Transformation is "just another domain" – and owns agent/process definitions**

   * Transformation (requirements, backlogs, UAF artifacts, architecture) is treated as its own domain cluster, not a magical meta‑layer.
   * **ProcessDefinitions** and **AgentDefinitions** are design-time artifacts owned by Transformation DomainController (modelled via UAF UIs).

4. **AgentControllers are cross‑cutting runtimes**

   * AgentControllers execute AgentDefinitions loaded from Transformation DomainController.
   * AgentDefinitions can be *specialised* for certain domains/processes via tags/labels (e.g. `domains: ["product", "pricing"]`).
   * AgentControllers are global runtimes that any process or domain can call, but they don't own the agent logic – Transformation DomainController does.

5. **Backstage is an internal factory, not a tenant UI**

   * Backstage is used **internally by Ameide (and agents)** as the “software factory” for controllers and infrastructure templates.
   * Tenant users never see Backstage; they only see the platform’s custom UI (Next.js) built on top of the domain/process APIs.

6. **Knowledge graph is for understanding, not source of truth**

   * Operational data lives in domains; selected projections flow into the Graph/Repository domain to support analysis, transformation, and agents.

---

## 3. Domain Portfolio (Top‑Level)

This is the “universe” we expect to cover long‑term. Not all are implemented on day one, but all follow the same pattern.

### 3.1 Platform & Identity Domain

* Tenants, orgs, users, roles, teams, onboarding, SSO, SCIM, subscriptions. 
* Responsible for tenancy, access and entitlements; every other domain depends on it.

### 3.2 Transformation & UAF Domain

* Transformations, initiatives, backlogs, architecture models, and the Unified Artifact Framework.
* **Owns design-time artifacts**:
  * **ProcessDefinitions** – BPMN-compliant process models (from custom React Flow modeller).
  * **AgentDefinitions** – Declarative agent specs (tools, policies, orchestration graphs).
  * ArchiMate models, Markdown docs, and other design artifacts.
* This is where LLM‑assisted "change the system" conversations and designs live, but technically it's *just another domain*.

### 3.3 Commercial & Customer Domain Cluster

Covers what most people today call CRM/part of ERP:

* **Customer & Account** (Customer master, org hierarchies)
* **Contact & Party**
* **Sales** (Leads, Opportunities, Activities)
* **Product** (Catalog, variants, hierarchies)
* **Pricing** (Price lists, discounts, conditions)
* **Contracting** (Quotes, Contracts, Terms)

L2O is one **end‑to‑end process** that spans several of these domains; they remain reusable beyond L2O.

### 3.4 Execution & Revenue Domain Cluster

What classical ERP calls O2C/Revenue:

* **Orders** (Sales Orders, Order Lines)
* **Fulfillment** (Shipments, deliveries, service delivery)
* **Billing & Invoicing**
* **Payments & Collections**
* **Returns & Disputes**

O2C is one example process orchestrating these domains.

### 3.5 Operations & Supply Chain Domains

Future but architecturally identical:

* **Procurement** (P2P – Purchase requisitions, Purchase Orders, Vendors)
* **Inventory & Warehousing**
* **Production / Manufacturing**
* **Asset Management**
* **Service Management / Field Service**

### 3.6 People & Organization Domains

For HCM-style processes:

* **Workforce** (Employees, contractors)
* **Compensation & Benefits**
* **Time & Attendance**
* **Talent & Performance**

### 3.7 Finance & Controlling Domains

* **General Ledger, Journals**
* **Accounts Payable & Receivable**
* **Cost Centers, Profit Centers, Allocations**
* **Budgeting & Forecasting**

### 3.8 Knowledge Graph & Reporting Domain

* Unified graph of entities and relationships and artifact references, used for analysis, impact assessment and agent context.
* **Read-only projection**: Graph receives data from DomainControllers/ProcessControllers/Transformation—it is never a source of truth.
* Transformation, Product, Orders, etc. remain the sources of truth; the Knowledge Graph only stores projections and references.

> **Note on Agents**: AgentDefinitions are design-time artifacts owned by the **Transformation DomainController** (§3.2), modelled via UAF UIs. AgentControllers are runtime components that execute AgentDefinitions. There is no separate "Agent Domain" – definitions are stored in Transformation, and AgentControllers are cross-cutting runtimes.

---

## 4. Controller Types per Domain

We use three controller types (runtime) plus two design-time artifact types:

**Design-time (stored in Transformation DomainController, modelled via UAF UIs)**:
* **ProcessDefinition** – BPMN-compliant artifact (from custom React Flow modeller) defining a process.
* **AgentDefinition** – Declarative spec for an agent (tools, policies, orchestration graph).

**Runtime (controllers)**:
* **DomainController** – owns data+rules for a particular domain (e.g. Product, Orders).
* **ProcessController** – executes ProcessDefinitions; orchestrates multiple DomainControllers and AgentControllers to implement an E2E process (e.g. L2O, O2C, P2P). Backed by Temporal.
* **AgentController** – executes AgentDefinitions; non‑deterministic LLM/tool automation that reads/writes via domain/process APIs.

All controllers expose proto‑based APIs and are consumed via the Ameide SDKs; this is uniform across all domains.

### 4.1 Example: Platform & Identity

* **DomainControllers**

  * `TenantsController`, `OrganizationsController`, `UsersController`, `MembershipsController`, `RolesController`, `SubscriptionsController`. 
* **ProcessControllers**

  * `SelfServeOnboardingProcess`, `EnterpriseOnboardingProcess`, `SubscriptionLifecycleProcess`.

### 4.2 Transformation & UAF

* **DomainControllers**

  * `TransformationController` (initiatives, metrics, stages).
  * `BacklogController` (epics, stories, tasks).
  * `ArtifactController` (UAF artifacts & revisions, including ProcessDefinitions and AgentDefinitions).
* **Design-time artifacts owned by this domain**

  * ProcessDefinitions – BPMN-compliant process models.
  * AgentDefinitions – Declarative agent specs.
* **ProcessControllers**

  * `AgileDeliveryProcess`, `TOGAFADMProcess`, `ChangeRequestProcess`.

### 4.3 Product & Pricing Domains

**DomainControllers**

* `ProductController`

  * Product master, variants, attributes, product hierarchies.
* `PricingController`

  * Price lists, discount rules, regional pricing, customer segments.

These are *generic* domains:

* L2O uses them for quoting and contracting.
* O2C uses them for orders and invoices.
* P2P may use them for purchase catalogs.

**ProcessControllers**

Examples:

* `PriceListUpdateProcess`

  * Handles mass price changes, approvals, coordination with L2O/O2C.
* `NewProductIntroductionProcess`

  * Coordinates Product, Inventory, Pricing, Marketing domains.

### 4.4 L2O and O2C as Examples (Not the whole world)

Rather than being “special”, L2O and O2C are **canonical examples** of how to wire the universal building blocks.

#### L2O (Lead‑to‑Order) ProcessController

* Uses domain controllers:

  * `LeadsController`, `AccountsController`, `OpportunitiesController`, `ProductController`, `PricingController`, `ContractsController`, plus Identity.
* Orchestrates:

  * Lead capture → qualification → offer/quote → negotiation → contract → **hand‑over to O2C** via `OrdersController`.
* AgentControllers:

  * L2O‑specialised advisors, executing AgentDefinitions stored in Transformation DomainController (e.g. `tags: ["process:L2O", "domain:pricing"]`).

#### O2C (Order‑to‑Cash) ProcessController

* Uses domain controllers:

  * `OrdersController`, `FulfillmentController`, `InvoicingController`, `PaymentsController`, `ProductController`, `PricingController`.
* Orchestrates:

  * Order validation → fulfillment → billing → payments/collections.
* Again, O2C is “just another process” that composes existing domains.

### 4.5 Other E2E Processes

The same pattern holds for:

* P2P (Procure‑to‑Pay) combining Procurement, Vendor, Inventory, Finance domains.
* H2R (Hire‑to‑Retire) combining Workforce, Payroll, Org structure domains.
* RtR (Record‑to‑Report) across Finance and Reporting.

No special handling needed: each gets one or more ProcessControllers orchestrating the relevant DomainControllers and AgentControllers.

### 4.6 AgentControllers (Cross-cutting)

AgentControllers are **not** a separate domain. They are cross-cutting runtimes that:

* **Execute AgentDefinitions** stored in Transformation DomainController (§3.2, §4.2).
* **Use tags/labels** for specialization:

  * `domains`: e.g. `["product", "pricing", "orders"]`
  * `processes`: e.g. `["L2O", "O2C"]`
* This is **metadata**, not strict ownership; any process can invoke any AgentController as long as policy allows.

Backstage templates (for DomainController/ProcessController/AgentController) are **internal implementation details**; they live in the Application/Tech layer and are used by engineers and transformation agents to spin up new controllers, not by business users directly.

---

## 5. End‑to‑End Example: Generic E2E Process Pattern

To show that this is universal, take a generic **E2E Process X** (e.g. some future `Idea‑to‑Launch` or `Case‑to‑Resolution` process):

1. **Transformation domain (design-time)**

   * Stakeholders describe the process using UAF artifacts (ProcessDefinitions, docs, constraints) under the Transformation domain.
   * ProcessDefinitions are created using the custom React Flow modeller.
   * AgentDefinitions are created for any agent assistance needed.

2. **Design → Runtime deployment**

   * Transformation processes (Agile/TOGAF) lead to formal ProcessDefinitions + requirements.
   * A ProcessController `ProcessXController` is deployed (executes ProcessDefinitions via Temporal).

3. **Domains reused**

   * `ProcessXController` calls whatever DomainControllers it needs: Product, Pricing, Orders, Workforce, Finance, etc.
   * No new domains unless really necessary; we prefer reusing.

4. **AgentControllers assist, but don't own data**

   * AgentControllers help draft ProcessDefinitions, propose domain changes, generate code for new DomainControllers, etc., but actual state lives in the relevant domains (via APIs).
   * AgentControllers execute AgentDefinitions from UAF.

5. **Graph & UAF for insight**

   * The Knowledge Graph observes domain/process events and UAF artifacts, enabling “what if” analysis, impact analysis, and cross‑domain reporting.

L2O/O2C are just named instances of this pattern.

---

## 6. UI & Experience (Domain View vs Process View)

* **Custom Platform UI (Next.js)**

  * All tenant‑facing UX is implemented in a custom Next.js platform app, which:

    * shows *process‑centric views* (e.g. “L2O pipeline”, “O2C flow board”), and
    * *domain‑centric workspaces* (e.g. “Product catalog”, “Pricing rules”, “Orders”).

* **Backstage is back‑office only**

  * Backstage is used by Ameide engineers, power users and agents to:

    * register new DomainControllers / ProcessControllers / Agents,
    * manage templates and service metadata.
  * It is *not* a place where business end‑users “do their work”.

* **Agents in the UI**

  * End‑users experience agents embedded in domain/process workspaces (“suggest pricing strategy”, “generate new L2O variant”), but those interactions always route through the Agent domain and domain APIs underneath.

---

## 7. Domain Boundaries & Anti‑Patterns

To keep things from becoming “yet another big monolith model”:

* **Do not put tenant/account logic into business domains**

  * Only Platform & Identity handle tenants, orgs, membership; all other domains reference a `tenant_id`/`org_id`. 

* **Do not make the Graph a write‑source**

  * Graph is read‑optimised; all write operations go through DomainControllers.

* **Do not tie an agent exclusively to one domain**

  * Agents should be reusable; use tags/labels to indicate specialty, not hard coupling.

* **Do not make Transformation meta‑own everything**

  * Transformation DomainController is about *design and governance*, not runtime transactional state.

* **Do not use Backstage as business UI**

  * Keep the platform’s UX clean, domain‑specific, and tailored to end‑users.

---

## 8. Alignment with Previous Ideas (IPA, UAF, North‑Star)

* **IPA**

  * Old “IPA” concept (workflows+agents+tools as one unit) is now split into:

    * DomainControllers + ProcessControllers + Agents, and
    * design/governance via UAF & Transformation domain.

* **UAF**

  * The UAF *pattern* (artifacts, revisions, promotions) is implemented inside the **Transformation DomainController**.
  * UAF as a UI surfaces those artifacts; the controller remains the source of truth.
  * Used to design processes and architectures across *all* domains, not just L2O/O2C.

* **North‑Star & ProcessDefinitions/Temporal**

  * The modeling and deployment blueprint: ProcessDefinitions (BPMN-compliant, from custom React Flow modeller) → Transformation DomainController storage (via UAF UIs) → Temporal-backed ProcessControllers. Controllers simply plug into that pipeline.

---

## 9. Open Questions

1. **Domain catalogue granularity**

   * How fine‑grained do we go for “Product” vs “Catalog” vs “Configuration”?
2. **Default E2E library**

   * Do we ship a standard set of canonical E2E processes (L2O, O2C, P2P, H2R, RtR) out‑of‑the‑box as Ameide “first‑party modules”?
3. **Agent capability taxonomy**

   * How rich should the tagging/classification scheme be for agents (domains, processes, risk tier, data access level)?

---

## 10. Cross‑References

This Domains Architecture should be read with the following documents:

| Document | Relationship | Alignment |
|----------|-------------|-----------|
| [470‑ameide‑vision](470-ameide-vision.md) | Parent vision & principles | Strong ✅ |
| [471‑ameide‑business‑architecture](471-ameide-business-architecture.md) | Business use cases, personas | Strong ✅ |
| [461‑ipc‑idc‑iac](461-ipc-idc-iac.md) | Technology realization (CRDs) | See §10.3 |
| [300‑ameide‑metamodel](300-ameide-metamodel.md) | Element graph foundation | See §10.4 |
| [305‑workflow](305-workflow.md) | ProcessController runtime | Strong ✅ |
| [310‑agents‑v2](310-agents-v2.md) | Agent domain implementation | See §10.2 |
| [367 series](367-0-feedback-intake.md) | Transformation methodology | Strong ✅ |
| [319‑onboarding](319-onboarding.md) | Platform & Identity domain | See §10.1 |
| [333‑realms](333-realms.md) | Realm‑per‑tenant for Identity | Strong ✅ |

### 10.1 Onboarding alignment (319)

* 319 describes current onboarding flow; aligns with §3.1 (Platform & Identity Domain).
* **Gap**: 319 is code-first orchestrator; 475 §4.1 describes `SelfServeOnboardingProcess` as a ProcessController (BPMN-backed).
* **Resolution**: Onboarding should be refactored as a ProcessController per 471 §11.2.

### 10.2 Agents alignment (310)

* 310 describes AgentController implementation with n8n-style flows.
* 475 §4.6 defines AgentControllers as cross-cutting runtimes (not a separate domain).
* AgentDefinitions (design-time specs) are owned by Transformation DomainController (§3.2, §4.2).
* **Alignment note**: 310's approach (agents as visual workflows) fits the "AgentControllers execute AgentDefinitions" pattern.

### 10.3 IDC/IPC/IAC alignment (461)

The domain portfolio (§3) maps to declarative CRDs defined in 461:

* **DomainController** → `IntelligentDomainController` (IDC) CRD
* **ProcessController** → `IntelligentProcessController` (IPC) CRD
* **AgentController** → `IntelligentAgentController` (IAC) CRD

461 describes K8s operators that would enable agent-driven domain provisioning. Current services (`graph`, `platform`, `threads`, `workflows`, `agents`) are standard deployments. Migration to operator-managed CRDs is future work per 461 implementation plan.

**Gap**: 461 operators (Phase 1-3) are not yet built. Until then, 475 domains are realized as standard K8s deployments rather than declarative CRDs.

### 10.4 Element Graph alignment (300)

The Element graph (300) and Domains Architecture (475) coexist:

* **Graph = design-time knowledge projection**: Elements *represent* BPMN models, domain schemas, and design artifacts **projected from the Transformation and other DomainControllers**.
* **DomainController = runtime execution**: Owns operational data and enforces state machines.
* **Graph is a read projection**: Operational data lives in domains; the graph receives projections for analysis, transformation, and agent context (per §2 Principle 6).

> **Important**: Transformation remains the source of truth for design artifacts; the graph stores references and read-optimised views, not the authoritative records.

**Gap**: 300's Element graph is partially implemented. Ontology-specific adapters (ArchiMate, BPMN, Document) route through legacy artifacts; converters to Element service remain TODO per 300 §Implementation Progress.

---

## 11. Notable Gaps & Issues

### 11.1 Terminology alignment

| 475 Term | Related Terms | Notes |
|----------|---------------|-------|
| Domain Cluster | N/A | 475-specific grouping concept |
| DomainController | DomainService (deprecated) | Consistent across 470-476 ✅ |
| ProcessController | Workflow (305) | Runtime that executes ProcessDefinitions |
| ProcessDefinition | BPMN model (deprecated) | Design-time artifact in Transformation DomainController |
| AgentController | AgentRuntime (deprecated) | Runtime that executes AgentDefinitions |
| AgentDefinition | Agent config (deprecated) | Design-time artifact in Transformation DomainController |

**Clarification**: AgentDefinitions are design-time artifacts owned by **Transformation DomainController** (§3.2), modelled via UAF UIs. There is no separate "UAF service"—UAF is the UI layer that calls Transformation APIs. AgentControllers are runtime components that execute AgentDefinitions.

### 11.2 Implementation gaps

| Gap | Description | Impact | Related Docs |
|-----|-------------|--------|--------------|
| Domain catalogue not implemented | §3 lists 9 domain clusters | Only Platform/Identity partially implemented | 319, 333 |
| Product/Pricing domains | §3.3 describes as separate domains | Currently bundled or missing | — |
| AgentDefinitions in Transformation | §3.2, §4.2 describe AgentDefinitions in Transformation DomainController | Transformation artifact storage not fully implemented | 310 |
| Knowledge Graph domain | §3.8 describes unified graph | Partial implementation (graph service) | 300-ameide-metamodel |
| E2E process library | §4.4 describes L2O, O2C | ProcessControllers not BPMN-backed yet | 305, 471 |

### 11.3 Open architectural tensions

1. **Domain granularity**
   * §9 Q1 asks: "How fine-grained do we go for Product vs Catalog vs Configuration?"
   * **Impact**: Affects service count, API surface, data ownership
   * **Decision needed**: Define domain boundary criteria

2. **AgentDefinition ownership**
   * §3.2, §4.2 now clarify: AgentDefinitions are design-time artifacts in **Transformation DomainController** (modelled via UAF UIs)
   * AgentControllers are cross-cutting runtimes (§4.6)
   * **Resolved**: No separate "Agent Domain" or "UAF service" – definitions in Transformation DomainController, execution in AgentControllers

3. **E2E process ownership**
   * §4.4 positions L2O/O2C as "examples, not special cases"
   * Current implementation has L2O-specific code
   * **Refactor needed**: Abstract L2O patterns into reusable ProcessController templates

4. **Knowledge Graph write access**
   * §7 anti-pattern: "Do not make the Graph a write-source"
   * Current graph service may allow direct writes
   * **Audit needed**: Ensure graph is projection-only per principle

5. **ProcessDefinition execution model**
   * §5 describes ProcessControllers executing ProcessDefinitions (from custom React Flow modeller) via Temporal
   * 305 uses Temporal but ProcessDefinition→Temporal compiler doesn't exist yet
   * **Resolution needed**: Either build compiler or clarify that current ProcessControllers are code-first

6. **Backstage integration** ✅ RESOLVED
   * §2 Principle 5 and §6 reference Backstage as platform factory
   * Implementation backlog created: [477-backstage.md](477-backstage.md)
   * §6.1 documents GitOps alignment, §10 covers future templates

### 11.4 Terminology migration

| Legacy Term | New Term (475) | Migration Status |
|-------------|----------------|------------------|
| IPA (Intelligent Process Automation) | DomainController + ProcessController + AgentController bundle | Deprecated in customer comms; `ipa/v1/` proto namespace remains |
| Platform Workflows (305) | ProcessController (executes ProcessDefinitions from Transformation) | 305 aligned |
| AgentRuntime (310) | AgentController (executes AgentDefinitions from Transformation) | 310 aligned |
| DomainService | DomainController | Aligned across 470-476 |
| BPMN model | ProcessDefinition (from custom React Flow modeller) | Aligned across 470-476 |
| Agent config | AgentDefinition (in Transformation DomainController) | Aligned across 470-476 |

---

If you like this rewrite, we can next adjust the **Refactoring / Migration Plan** document (6/6) so it maps the *current* services into this updated, universal domain view (including separating Product/Pricing domains explicitly and repositioning Backstage and Agents the way we just defined).
