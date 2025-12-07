Got it — let's re-cut the **Domains Architecture** layer with those corrections baked in.

> **Alignment Note**: This document describes domain architecture patterns
> that align with **ProcessControllers** as defined in [471‑ameide‑business‑architecture](471-ameide-business-architecture.md).
>
> - ProcessControllers here are business-level abstractions (BPMN + runtime)
> - Platform Workflows (305) implement ProcessControllers as Temporal infrastructure
> - Customers experience these as "process stages" and "process boards"
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

3. **Transformation is “just another domain”**

   * Transformation (requirements, backlogs, UAF artifacts, architecture) is treated as its own domain cluster, not a magical meta‑layer.

4. **Agents are first‑class and cross‑cutting**

   * Agents form their **own domain** (Agent domain) and are *not* owned by individual business domains.
   * Agents can be *specialised* for certain domains/processes via tags/labels, but technically they are global resources that any process or domain can call.

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

* Transformations, initiatives, backlogs, architecture models, process designs, and the Unified Artifact Framework (BPMN, ArchiMate, Markdown).
* This is where LLM‑assisted “change the system” conversations and designs live, but technically it’s *just another domain*.

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

### 3.9 Agent Domain

* Agent definitions, tools, runtimes, policies.
* First‑party in the sense that “agents are a core domain”, not sidecar.
* Agents are cross‑domain: they may be *tagged* as `domains: ["product", "pricing"]` or `processes: ["L2O"]`, but remain globally addressable.

---

## 4. Controller Types per Domain

We stick to the three controller types:

* **DomainController** – owns data+rules for a particular domain (e.g. Product, Orders).
* **ProcessController** – orchestrates multiple domains and agents to implement an E2E process (e.g. L2O, O2C, P2P).
* **Agent** – non‑deterministic automation that reads/writes via domain/process APIs.

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
  * `ArtifactController` (UAF artifacts & revisions).
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
* Agents:

  * L2O‑specialised advisors, but implemented in the **Agent domain** and tagged accordingly (e.g. `tags: ["process:L2O", "domain:pricing"]`).

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

No special handling needed: each gets one or more ProcessControllers orchestrating the relevant DomainControllers and Agents.

### 4.6 Agent Domain

* **DomainControllers**

  * `AgentsController`

    * Stores agent graphs, tool definitions, constraints, allowed capabilities.
* **Agents themselves**

  * LLM or tool‑based entities in the inference runtime, using the Ameide SDK to talk to domains and processes.
* **Association model**

  * Agents may declare:

    * `domains`: e.g. `["product", "pricing", "orders"]`
    * `processes`: e.g. `["L2O", "O2C"]`
  * This is **metadata**, not a strict ownership; you can always call an agent from any process as long as your call is allowed by policy.

Backstage templates (for domain/process/agent controllers) are **internal implementation details**; they live in the Application/Tech layer and are used by engineers and agents to spin up new controllers, not by business users directly.

---

## 5. End‑to‑End Example: Generic E2E Process Pattern

To show that this is universal, take a generic **E2E Process X** (e.g. some future `Idea‑to‑Launch` or `Case‑to‑Resolution` process):

1. **Transformation domain**

   * Stakeholders describe the process using UAF artifacts (BPMN, docs, constraints) under the Transformation domain.

2. **Design → ProcessController**

   * Transformation processes (Agile/TOGAF) lead to a formal BPMN + requirements.
   * A ProcessController `ProcessXController` is defined (BPMN + Temporal/Camunda code).

3. **Domains reused**

   * `ProcessXController` calls whatever domains it needs: Product, Pricing, Orders, Workforce, Finance, etc.
   * No new domains unless really necessary; we prefer reusing.

4. **Agents assist, but don’t own data**

   * Agents help draft BPMN, propose domain changes, generate code for new DomainControllers, etc., but actual state lives in the relevant domains (via APIs).

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

  * Transformation/UAF is about *design and governance*, not runtime transactional state.

* **Do not use Backstage as business UI**

  * Keep the platform’s UX clean, domain‑specific, and tailored to end‑users.

---

## 8. Alignment with Previous Ideas (IPA, UAF, North‑Star)

* **IPA**

  * Old “IPA” concept (workflows+agents+tools as one unit) is now split into:

    * DomainControllers + ProcessControllers + Agents, and
    * design/governance via UAF & Transformation domain.

* **UAF**

  * Unified Artifact Framework is clearly **in the Transformation domain** and used to design processes and architectures across *all* domains, not just L2O/O2C. 

* **North‑Star & BPMN/Camunda/Temporal**

  * The modeling and deployment blueprint stays the same: BPMN in the browser → Design/Deploy APIs → Temporal and/or Camunda runtime. Controllers simply plug into that pipeline.

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

* 310 describes AgentRuntime with n8n-style flows.
* 475 §3.9 defines Agent Domain as first-class, cross-cutting domain.
* **Alignment note**: 310's approach (agents as visual workflows) fits the "agents are cross-domain" principle. Ensure 310 agents are registered via `AgentsController` per §4.6.

### 10.3 IDC/IPC/IAC alignment (461)

The domain portfolio (§3) maps to declarative CRDs defined in 461:

* **DomainController** → `IntelligentDomainController` (IDC) CRD
* **ProcessController** → `IntelligentProcessController` (IPC) CRD
* **AgentRuntime** → `IntelligentAgentController` (IAC) CRD

461 describes K8s operators that would enable agent-driven domain provisioning. Current services (`graph`, `platform`, `threads`, `workflows`, `agents`) are standard deployments. Migration to operator-managed CRDs is future work per 461 implementation plan.

**Gap**: 461 operators (Phase 1-3) are not yet built. Until then, 475 domains are realized as standard K8s deployments rather than declarative CRDs.

### 10.4 Element Graph alignment (300)

The Element graph (300) and Domains Architecture (475) coexist:

* **Graph = design-time knowledge**: Elements store BPMN models, domain schemas, and UAF artifacts as versioned nodes.
* **DomainController = runtime execution**: Owns operational data and enforces state machines.
* **Graph is a read projection**: Operational data lives in domains; the graph receives projections for analysis, transformation, and agent context (per §2 Principle 6).

**Gap**: 300's Element graph is partially implemented. Ontology-specific adapters (ArchiMate, BPMN, Document) route through legacy artifacts; converters to Element service remain TODO per 300 §Implementation Progress.

---

## 11. Notable Gaps & Issues

### 11.1 Terminology alignment

| 475 Term | Related Terms | Notes |
|----------|---------------|-------|
| Domain Cluster | N/A | 475-specific grouping concept |
| DomainController | Domain Service (470/471/473) | Consistent ✅ |
| ProcessController | Workflow (305) | 305 uses "Workflow"; recommend aligning |
| Agent Domain | AgentRuntime (310, 473) | 475 treats agents as a domain; 310/473 treat as runtime |

**Clarification needed**: 475 frames agents as a "domain" with `AgentsController`. This is conceptually different from 310/473 where agents are runtime components. Both views can coexist:
* **Agent Domain** (475): metadata about agents (definitions, tools, policies)
* **AgentRuntime** (310/473): execution layer for agent inference

### 11.2 Implementation gaps

| Gap | Description | Impact | Related Docs |
|-----|-------------|--------|--------------|
| Domain catalogue not implemented | §3 lists 9 domain clusters | Only Platform/Identity partially implemented | 319, 333 |
| Product/Pricing domains | §3.3 describes as separate domains | Currently bundled or missing | — |
| Agent Domain | §3.9 describes `AgentsController` | No proto or service exists | 310 |
| Knowledge Graph domain | §3.8 describes unified graph | Partial implementation (graph service) | 300-ameide-metamodel |
| E2E process library | §4.4 describes L2O, O2C | ProcessControllers not BPMN-backed yet | 305, 471 |

### 11.3 Open architectural tensions

1. **Domain granularity**
   * §9 Q1 asks: "How fine-grained do we go for Product vs Catalog vs Configuration?"
   * **Impact**: Affects service count, API surface, data ownership
   * **Decision needed**: Define domain boundary criteria

2. **Agent domain vs runtime**
   * §3.9 treats Agent as a domain with `AgentsController`
   * 310 treats agents as visual workflow nodes
   * 473 treats agents as inference runtime
   * **Resolution needed**: Clarify relationship between Agent domain (metadata) and agent execution (runtime)

3. **E2E process ownership**
   * §4.4 positions L2O/O2C as "examples, not special cases"
   * Current implementation has L2O-specific code
   * **Refactor needed**: Abstract L2O patterns into reusable ProcessController templates

4. **Knowledge Graph write access**
   * §7 anti-pattern: "Do not make the Graph a write-source"
   * Current graph service may allow direct writes
   * **Audit needed**: Ensure graph is projection-only per principle

5. **BPMN execution model**
   * §5 describes ProcessControllers as BPMN-driven via Temporal/Camunda
   * 305 uses Temporal but BPMN→Temporal compiler (461 Phase 3) doesn't exist
   * **Resolution needed**: Either build compiler or clarify that current ProcessControllers are code-first

6. **Backstage integration**
   * §2 Principle 5 and §6 reference Backstage as platform factory
   * No Backstage implementation backlog exists
   * **Decision needed**: Create Backstage integration backlog or document alternative

### 11.4 Terminology migration

| Legacy Term | New Term (475) | Migration Status |
|-------------|----------------|------------------|
| IPA (Intelligent Process Automation) | Domain + ProcessController + Agent bundle | Deprecated in customer comms; `ipa/v1/` proto namespace remains |
| Platform Workflows (305) | ProcessController | 305 aligned |
| AgentRuntime (310) | AgentsController (domain view) | 310 aligned; Backstage-based controller planned |

---

If you like this rewrite, we can next adjust the **Refactoring / Migration Plan** document (6/6) so it maps the *current* services into this updated, universal domain view (including separating Product/Pricing domains explicitly and repositioning Backstage and Agents the way we just defined).
