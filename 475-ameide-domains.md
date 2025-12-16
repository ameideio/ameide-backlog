> **Terminology**: In all forward-looking docs, **IDC = Domain CRD**, **IPC = Process CRD**, **IAC = Agent CRD**. New work MUST use the latter names. See [470 §0 Terminology Migration](470-ameide-vision.md) for full mapping.

---

> **Cross-References (Vision Suite)**:
>
> | Document | Purpose |
> |----------|---------|
> | [470-ameide-vision.md](470-ameide-vision.md) | Vision and principles |
> | [471-ameide-business-architecture.md](471-ameide-business-architecture.md) | Business concepts |
> | [472-ameide-information-application.md](472-ameide-information-application.md) | Application architecture |
> | [473-ameide-technology.md](473-ameide-technology.md) | Technology stack |
> | [476-ameide-security-trust.md](476-ameide-security-trust.md) | Security architecture |
> | [478-ameide-extensions.md](478-ameide-extensions.md) | Tenant primitive patterns |
>
> **Core Invariants**: See [470-ameide-vision.md §0 "Ameide Core Invariants"](470-ameide-vision.md) for the canonical list (six primitives: Domain/Process/Agent/UISurface/Projection/Integration, Graph read-only, Transformation as domain, proto chain, tenant isolation, Backstage internal).
>
> **Deployment Implementation**:
> - [461-ipc-idc-iac.md](461-ipc-idc-iac.md) – CRD operator patterns *(note: 461 uses deprecated IDC/IPC/IAC naming; use Domain/Process/Agent CRD per 470 glossary)*
> - [465-applicationset-architecture.md](465-applicationset-architecture.md) – GitOps deployment model
> - [464-chart-folder-alignment.md](464-chart-folder-alignment.md) – Chart folder structure by domain

> **Alignment Note**: This document describes domain architecture patterns
> that align with the design-time/runtime split defined in [471‑ameide‑business‑architecture](471-ameide-business-architecture.md):
>
> - **ProcessDefinitions** (design-time): BPMN-compliant artifacts from custom React Flow modeller, stored in **Transformation Domain primitive** (modelled via Transformation design tooling UIs)
> - **Process primitives** (runtime): Implement Temporal workflows whose behavior is derived from ProcessDefinitions, but the translation from design-time artifacts to code happens in CLI/agents and the primitive’s source, not in operators or dynamic runtime compilation
> - **AgentDefinitions** (design-time): Declarative agent specs stored in **Transformation Domain primitive** (modelled via Transformation design tooling UIs)
> - **Agent primitives** (runtime): Implement agent behavior that follows AgentDefinitions; they do not persist or compile design-time artifacts themselves
>
> **Core Definitions** (see [470-ameide-vision.md §0](470-ameide-vision.md)):
> - **Transformation design tooling** is the set of modelling UIs—it does not own storage. All Transformation design tooling-originated artifacts are persisted by the **Transformation Domain primitive**.
> - **Graph** is a read-only knowledge projection; all writes go through primitives.
>
> For cross-references and gaps analysis, see §10 and §11.

## Grounding & contract alignment

- **Domain & process portfolio:** Specializes the primitive model from `470-ameide-vision.md`, `471-ameide-business-architecture.md`, and `472-ameide-information-application.md` into a concrete domain/process portfolio, with Transformation modeled explicitly as a Domain primitive owning design-time artifacts (ProcessDefinitions/AgentDefinitions).  
- **Primitive patterns:** Provides the domain/process architecture patterns that operator and vertical-slice backlogs (`495-ameide-operators.md`, `498-domain-operator.md`, `499-process-operator.md`, `502-domain-vertical-slice.md`, `477-primitive-stack.md`, `481-service-catalog.md`) follow when structuring primitives and their GitOps artifacts.  
- **Scrum & transformation:** Establishes how Transformation and Graph behave as domains, which is a prerequisite for the Scrum-specific modeling in `300-400/367-1-scrum-transformation.md`, `506-scrum-vertical-v2.md`, and the Scrum protos in `508-scrum-protos.md`.

---

## Implementation Status (2025-02-14)

- ✅ Knowledge Graph writes are now blocked so this document’s read-only projection rule is enforceable in production (`services/graph/src/graph/service.ts:118-135`, `services/graph/src/graph/service.ts:4879-4905`).
- ⚠️ The domain portfolio is still represented by placeholder manifests (e.g., the Orders Domain CR just runs `nginx`), so no business domains have been migrated into primitives yet (`gitops/ameide-gitops/environments/_shared/primitives/domain/orders.yaml:1-22`).
- ⚠️ ProcessDefinitions/AgentDefinitions live in the workflows runtime instead of the Transformation domain, which keeps Process primitives from declaring real definition refs (`services/workflows/src/repositories/definitions-graph.ts:1-80`, `operators/process-operator/internal/controller/process_controller.go:54-149`).

# 475 – Domains Architecture (Universal Business Platform)

**Status:** Draft
**Audience:** Domain & process architects, product owners, platform engineers
**Scope:** How Ameide defines and structures domains and processes so that *any* business scenario can be modeled and implemented using Domain primitives, Process primitives, and Agents. This document describes **application architecture patterns**, not functional scope.

---

## 1. Purpose

This document explains:

* **How domains are represented technically** via Domain primitives, Process primitives and Agents.
* **How end‑to‑end processes are built** by composing domains.
* **Where Transformation and the Knowledge Graph fit** into the domain picture.

It sits between:

* **Business Architecture** (who uses what, for which value streams), and
* **Application/Technology Architecture** (K8s, Temporal, Backstage, SDKs, etc.).

> **Note**: This document describes **architecture patterns**. Functional scope (which specific domains or processes to implement) is a separate product decision.

---

## 2. Domain Principles

1. **Universal architecture, specific domains**

   * The platform architecture supports *any* business process.
   * It does **not** do this via one giant meta‑model but via *many small, well‑defined domains* each owned by a Domain primitive.

2. **Processes first, domains reusable**

   * End‑to‑end processes are modeled as **Process primitives** orchestrating multiple domains.
   * Domains are reusable building blocks across many processes.

3. **Transformation is "just another domain" – and owns agent/process definitions**

   * Transformation (requirements, backlogs, Transformation design tooling artifacts, architecture) is treated as its own domain cluster, not a magical meta‑layer.
   * **ProcessDefinitions** and **AgentDefinitions** are design-time artifacts owned by Transformation Domain primitive (modelled via Transformation design tooling UIs).

4. **Agent primitives are cross‑cutting runtimes**

   * Agent primitives execute AgentDefinitions loaded from Transformation Domain primitive.
   * AgentDefinitions can be *specialised* for certain domains/processes via tags/labels (e.g. `domains: ["product", "pricing"]`).
   * Agent primitives are global runtimes that any process or domain can call, but they don't own the agent logic – Transformation Domain primitive does.

5. **Backstage is an internal factory, not a tenant UI**

   * Backstage is used **internally by Ameide (and agents)** as the “software factory” for primitives and infrastructure templates.
   * Tenant users never see Backstage; they only see the platform’s custom UI (Next.js) built on top of the domain/process APIs.

6. **Knowledge graph is for understanding, not source of truth**

   * Operational data lives in domains; selected projections flow into the Graph/Repository domain to support analysis, transformation, and agents.
7. **Proto-first + event-driven by default**

   * Domains/Processes/Agents follow the Buf → SDK → GitOps chain in [472 §2.7](472-ameide-information-application.md#27-contract--implementation--runtime-chain), and Go implementations standardize on Watermill CQRS for command/event routing (see [472 §3.3.1](472-ameide-information-application.md#331-event-driven-cqrs-runtime-watermill)).

---

## 3. Domain Portfolio

This section describes the target domain portfolio. All domains follow the same architectural pattern.

### 3.1 Platform & Identity Domain

* Tenants, orgs, users, roles, teams, onboarding, SSO, SCIM, subscriptions.
* Responsible for tenancy, access and entitlements; every other domain depends on it.

### 3.2 Transformation Domain

* Transformations, initiatives, backlogs, architecture models, and the Unified Artifact Framework.
* **Owns design-time artifacts**:
  * **ProcessDefinitions** – BPMN-compliant process models (from custom React Flow modeller).
  * **AgentDefinitions** – Declarative agent specs (tools, policies, orchestration graphs).
  * ArchiMate models, Markdown docs, and other design artifacts.
* This is where LLM‑assisted "change the system" conversations and designs live, but technically it's *just another domain*.

### 3.3 Commercial & Customer Domain Cluster

* **Customer & Account** (Customer master, org hierarchies)
* **Contact & Party**
* **Sales** (Leads, Opportunities, Activities)
* **Product** (Catalog, variants, hierarchies)
* **Pricing** (Price lists, discounts, conditions)
* **Contracting** (Quotes, Contracts, Terms)

L2O is one end‑to‑end process that spans several of these domains.

### 3.4 Execution & Revenue Domain Cluster

* **Orders** (Sales Orders, Order Lines)
* **Fulfillment** (Shipments, deliveries, service delivery)
* **Billing & Invoicing**
* **Payments & Collections**
* **Returns & Disputes**

O2C is one example process orchestrating these domains.

### 3.5 Operations & Supply Chain Domains

* **Procurement** (P2P – Purchase requisitions, Purchase Orders, Vendors)
* **Inventory & Warehousing**
* **Production / Manufacturing**
* **Asset Management**
* **Service Management / Field Service**

### 3.6 People & Organization Domains

* **Workforce** (Employees, contractors)
* **Compensation & Benefits**
* **Time & Attendance**
* **Talent & Performance**

### 3.7 Finance & Controlling Domains

* **General Ledger, Journals**
* **Accounts Payable & Receivable**
* **Cost Centers, Profit Centers, Allocations**
* **Budgeting & Forecasting**

### 3.8 Knowledge Graph (Read-Only Projection)

* Unified graph of entities and relationships and artifact references, used for analysis, impact assessment and agent context.
* **Read-only projection**: Graph receives data from Domain primitives/Process primitives/Transformation—it is never a source of truth.
* Domain primitives remain the sources of truth; the Knowledge Graph only stores projections and references.

> **Note on Agents**: AgentDefinitions are design-time artifacts owned by the **Transformation Domain primitive** (§3.2), modelled via Transformation design tooling UIs. Agent primitives are runtime components that execute AgentDefinitions. There is no separate "Agent Domain" – definitions are stored in Transformation, and Agent primitives are cross-cutting runtimes.

### 3.9 Platform Operations / SRE Domain Cluster

* **Incident Management** (incidents, alerts, on-call escalation, timelines)
* **Runbook Registry** (documented procedures, automation playbooks)
* **SLO/SLI Framework** (service level objectives, error budgets, burn rates)
* **Fleet Health** (GitOps state, cluster health, application status)
* **Change Verification** (deployment health, sync status, smoke tests)

This domain cluster provides the **operate-the-platform capability** — ensuring reliability, observability, and operational excellence. It is distinct from Transformation (which owns design-time artifacts and governance) and Platform & Identity (which owns tenancy and access).

Key responsibilities:
* System-of-record for operational events (incidents, alerts, health checks)
* Formalization of operational workflows (e.g., backlog-first triage per `525-backlog-first-triage-workflow.md`)
* Integration with external systems (ArgoCD, Prometheus/AlertManager, PagerDuty)
* SRE agent assistance for incident investigation and remediation

See [526-sre-capability.md](526-sre-capability.md) for the full capability definition.

---

## 4. Primitive Types per Domain

We use three runtime primitive types plus two design-time artifact types:

**Design-time (stored in Transformation Domain primitive, modelled via Transformation design tooling UIs)**:
* **ProcessDefinition** – BPMN-compliant artifact (from custom React Flow modeller) defining a process.
* **AgentDefinition** – Declarative spec for an agent (tools, policies, orchestration graph).

**Runtime (primitives)**:
* **Domain primitive** – owns data+rules for a particular domain.
* **Process primitive** – implements Temporal workflows whose behavior is derived from ProcessDefinitions authored in the Transformation Domain; orients those workflows around cross-domain orchestration, but does **not** load or interpret ProcessDefinitions dynamically at runtime. Backed by Temporal.
* **Agent primitive** – implements agent behavior that follows AgentDefinitions; non-deterministic LLM/tool automation that reads/writes via domain/process APIs, without persisting or compiling design-time specs itself.

**Extension hooks (Tier 1)**

Domain primitives and Process primitives may expose explicit extension points (e.g. `BeforePostInvoice`, BPMN Extension Tasks) which are satisfied by `ExtensionDefinition` artifacts executed via the shared `extensions-runtime` service. This gives tenants a light-weight way to customize behaviour without introducing new primitives for every rule change. See [479-ameide-extensibility-wasm.md](479-ameide-extensibility-wasm.md).

All primitives expose proto-based APIs and are consumed via the Ameide SDKs; this is uniform across all domains. At runtime each Domain/Process/Agent primitive is represented as a declarative CRD (Domain/Process/Agent) reconciled by Ameide operators (see [474-ameide-refactoring.md](474-ameide-refactoring.md) for the migration into this model).

### 4.1 Example: Platform & Identity

* **Domain primitives**

  * `TenantsDomain`, `OrganizationsDomain`, `UsersDomain`, `MembershipsDomain`, `RolesDomain`, `SubscriptionsDomain`.
* **Process primitives**

  * `SelfServeOnboardingProcess`, `EnterpriseOnboardingProcess`, `SubscriptionLifecycleProcess`.

### 4.2 Transformation & Transformation design tooling

* **Domain primitives**

  * `TransformationDomain` (initiatives, metrics, stages).
  * `BacklogDomain` (epics, stories, tasks).
  * `ArtifactDomain` (Transformation design tooling artifacts & revisions, including ProcessDefinitions and AgentDefinitions).
* **Design-time artifacts owned by this domain**

  * ProcessDefinitions – BPMN-compliant process models.
  * AgentDefinitions – Declarative agent specs.
* **Process primitives**

  * `AgileDeliveryProcess`, `TOGAFADMProcess`, `ChangeRequestProcess`.

### 4.3 Product & Pricing Domains

**Domain primitives**

* `ProductDomain`

  * Product master, variants, attributes, product hierarchies.
* `PricingDomain`

  * Price lists, discount rules, regional pricing, customer segments.

These are *generic* domains:

* L2O uses them for quoting and contracting.
* O2C uses them for orders and invoices.
* P2P may use them for purchase catalogs.

**Process primitives**

Examples:

* `PriceListUpdateProcess`

  * Handles mass price changes, approvals, coordination with L2O/O2C.
* `NewProductIntroductionProcess`

  * Coordinates Product, Inventory, Pricing, Marketing domains.

### 4.4 L2O and O2C as Examples (Not the whole world)

Rather than being “special”, L2O and O2C are **canonical examples** of how to wire the universal building blocks.

#### L2O (Lead‑to‑Order) Process primitive

* Uses domain primitives:

  * `LeadsDomain`, `AccountsDomain`, `OpportunitiesDomain`, `ProductDomain`, `PricingDomain`, `ContractsDomain`, plus Identity.
* Orchestrates:

  * Lead capture → qualification → offer/quote → negotiation → contract → **hand‑over to O2C** via `OrdersDomain`.
* Agent primitives:

  * L2O‑specialised advisors, executing AgentDefinitions stored in Transformation Domain primitive (e.g. `tags: ["process:L2O", "domain:pricing"]`).

#### O2C (Order‑to‑Cash) Process primitive

* Uses domain primitives:

  * `OrdersDomain`, `FulfillmentDomain`, `InvoicingDomain`, `PaymentsDomain`, `ProductDomain`, `PricingDomain`.
* Orchestrates:

  * Order validation → fulfillment → billing → payments/collections.
* Again, O2C is "just another process" that composes existing domains.

### 4.5 Other E2E Processes

The same pattern holds for:

* P2P (Procure‑to‑Pay) combining Procurement, Vendor, Inventory, Finance domains.
* H2R (Hire‑to‑Retire) combining Workforce, Payroll, Org structure domains.
* RtR (Record‑to‑Report) across Finance and Reporting.

No special handling needed: each gets one or more Process primitives orchestrating the relevant Domain primitives and Agent primitives.

### 4.6 Agent primitives (Cross-cutting)

Agent primitives are **not** a separate domain. They are cross-cutting runtimes that:

* **Execute AgentDefinitions** stored in Transformation Domain primitive (§3.2, §4.2).
* **Use tags/labels** for specialization:

  * `domains`: e.g. `["product", "pricing", "orders"]`
  * `processes`: e.g. `["L2O", "O2C"]`
* This is **metadata**, not strict ownership; any process can invoke any Agent primitive as long as policy allows.

Backstage templates (for primitives) are **internal implementation details**; they live in the Application/Tech layer and are used by engineers and transformation agents to spin up new primitives, not by business users directly.

### 4.7 Tenant-Specific Primitives

Whenever possible, small, local customizations should attach to Tier 1 WASM extension hooks on existing primitives. The model below describes **Tier 2 primitive-based extensions** for requirements that justify new services—see [479-ameide-extensibility-wasm.md](479-ameide-extensibility-wasm.md) for how both tiers relate.

When tenants require custom primitives beyond the standard product, Backstage templates (or agents) create primitive CRDs (Domain/Process/Agent/UISurface/Projection/Integration) plus repositories and GitOps manages those CRDs per environment:

* **Platform primitives** run in `ameide-{env}` (Shared SKU) or `tenant-{id}-{env}-base` (Namespace/Private SKU)
* **Tenant custom primitives** always run in `tenant-{id}-{env}-cust` namespaces
* Custom code is scaffolded via Backstage templates with tenant-specific parameters
* Code lives in `tenant-{id}-primitives` repos, manifests in `tenant-{id}-gitops` repos

> **See [478-ameide-extensions.md](478-ameide-extensions.md)** for the complete tenant extension model, namespace strategy by SKU, and the E2E flow from requirement to running primitives.

### 4.8 Platform Operations / SRE Primitives

**Domain primitives**

* `SREDomain` (incidents, alerts, runbooks, SLOs/SLIs, health checks, fleet state)

**Process primitives**

* `IncidentTriageProcess` — formalizes the 525 backlog-first triage workflow
* `ChangeVerificationProcess` — post-deployment health verification
* `SLOBurnAlertProcess` — error budget monitoring and escalation

**Projection primitives**

* `FleetHealthProjection` — real-time fleet status dashboard
* `IncidentTimelineProjection` — historical incident data and MTTR metrics
* `SLOBurndownProjection` — error budget tracking and trends

**Integration primitives**

* `ArgoCDIntegration` — sync ArgoCD application state
* `AlertManagerIntegration` — receive alerts from Prometheus
* `TicketingIntegration` — create tickets in Jira/GitHub
* `PagingIntegration` — escalate to PagerDuty/OpsGenie

**Agent primitives**

* `SREAgent` — implements 525 backlog-first triage workflow, assists with incident investigation and remediation

See [526-sre-capability.md](526-sre-capability.md) for the full capability definition.

---

## 5. End‑to‑End Example: Generic E2E Process Pattern

To show that this is universal, take a generic **E2E Process X** (e.g. some future `Idea‑to‑Launch` or `Case‑to‑Resolution` process):

1. **Transformation domain (design-time)**

   * Stakeholders describe the process using Transformation design tooling artifacts (ProcessDefinitions, docs, constraints) under the Transformation domain.
   * ProcessDefinitions are created using the custom React Flow modeller.
   * AgentDefinitions are created for any agent assistance needed.

2. **Design → Runtime deployment**

   * Transformation processes (Agile/TOGAF) lead to formal ProcessDefinitions + requirements.
   * CLI/agents use those ProcessDefinitions and requirements to generate or evolve the Temporal workflow code for a Process primitive `ProcessX`.
   * A Process primitive `ProcessX` is built and deployed; at runtime it executes its own workflow code informed by the ProcessDefinitions, but does **not** interpret BPMN/process definitions dynamically.

3. **Domains reused**

   * `ProcessX` calls whatever Domain primitives it needs: Product, Pricing, Orders, Workforce, Finance, etc.
   * No new domains unless really necessary; we prefer reusing.

4. **Agent primitives assist, but don't own data**

   * Agent primitives help draft ProcessDefinitions, propose domain changes, generate code for new Domain primitives, etc., but actual state lives in the relevant domains (via APIs).
   * Agent primitives execute AgentDefinitions from Transformation design tooling.

5. **Graph & Transformation design tooling for insight**

   * The Knowledge Graph observes domain/process events and Transformation design tooling artifacts, enabling “what if” analysis, impact analysis, and cross‑domain reporting.

L2O/O2C are just named instances of this pattern.

---

## 6. UI & Experience (Domain View vs Process View)

* **Custom Platform UI (Next.js)**

  * All tenant‑facing UX is implemented in a custom Next.js platform app, which:

    * shows *process‑centric views* (e.g. "L2O pipeline", "O2C flow board"), and
    * *domain‑centric workspaces* (e.g. "Product catalog", "Pricing rules", "Orders").

**Workspace Composition Model:**

Workspaces are composed of pages that host **canvases**; canvases render **widgets**; layouts are configurable per tenant and per user. Governed templates are promoted through the Transformation domain using the same lifecycle as ProcessDefinitions and AgentDefinitions.

End users experience workspaces as:
* **Process-centric views** (case management, approval inboxes, process monitoring)
* **Domain workspaces** (record pages, forms, tables, timelines)
* **Dashboard canvases** (metrics, charts, highlights, activity feeds)

All composed from the same widget registry with tenant and user layout flexibility.

* **Backstage is back‑office only**

  * Backstage is used by Ameide engineers, power users and agents to:

    * register new Domain primitives / Process primitives / Agents,
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

  * Graph is read‑optimised; all write operations go through Domain primitives.

* **Do not tie an agent exclusively to one domain**

  * Agents should be reusable; use tags/labels to indicate specialty, not hard coupling.

* **Do not make Transformation meta‑own everything**

  * Transformation Domain primitive is about *design and governance*, not runtime transactional state.

* **Do not use Backstage as business UI**

  * Keep the platform’s UX clean, domain‑specific, and tailored to end‑users.

---

## 8. Alignment with Previous Ideas (IPA, Transformation design tooling, North‑Star)

* **IPA**

  * Old “IPA” concept (workflows+agents+tools as one unit) is now split into:

    * Domain primitives + Process primitives + Agents, and
    * design/governance via Transformation design tooling & Transformation domain.

* **Transformation design tooling**

  * The Transformation design tooling *pattern* (artifacts, revisions, promotions) is implemented inside the **Transformation Domain primitive**.
  * Transformation design tooling as a UI surfaces those artifacts; the primitive remains the source of truth.
  * Used to design processes and architectures across *all* domains, not just L2O/O2C.

* **North‑Star & ProcessDefinitions/Temporal**

  * The modeling and deployment blueprint: ProcessDefinitions (BPMN-compliant, from custom React Flow modeller) → Transformation Domain primitive storage (via Transformation design tooling UIs) → Temporal-backed Process primitives. Controllers simply plug into that pipeline.

---

## 9. Open Questions

1. **Domain boundary criteria**

   * What criteria determine when to split vs merge domains (e.g. "Product" vs "Catalog" vs "Configuration")?

2. **Agent capability taxonomy**

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
| [305‑workflow](305-workflow.md) | Process primitive runtime | Strong ✅ |
| [310‑agents‑v2](310-agents-v2.md) | Agent domain implementation | See §10.2 |
| [367 series](367-0-feedback-intake.md) | Transformation methodology | Strong ✅ |
| [319‑onboarding](319-onboarding.md) | Platform & Identity domain | See §10.1 |
| [333‑realms](333-realms.md) | Realm‑per‑tenant for Identity | Strong ✅ |
| [478‑ameide‑extensions](478-ameide-extensions.md) | Tenant primitive patterns | See §10.5 |

### 10.1 Onboarding alignment (319)

* 319 describes the onboarding flow; aligns with §3.1 (Platform & Identity Domain).
* Onboarding is modeled as a Process primitive per §4.1.

### 10.2 Agents alignment (310)

* 310 describes Agent primitive implementation with n8n-style flows.
* 475 §4.6 defines Agent primitives as cross-cutting runtimes (not a separate domain).
* AgentDefinitions (design-time specs) are owned by Transformation Domain primitive (§3.2, §4.2).
* **Alignment note**: 310's approach (agents as visual workflows) fits the "Agent primitives execute AgentDefinitions" pattern.

### 10.3 CRD alignment (461)

The domain portfolio (§3) maps to declarative CRDs:

* **Domain primitive** → Domain CRD
* **Process primitive** → Process CRD
* **Agent primitive** → Agent CRD

> **Note**: 461 uses deprecated IDC/IPC/IAC naming; use Domain/Process/Agent CRD per 470 glossary.

### 10.4 Element Graph alignment (300)

The Element graph (300) and Domains Architecture (475) coexist:

* **Graph = design-time knowledge projection**: Elements *represent* BPMN models, domain schemas, and design artifacts **projected from the Transformation and other Domain primitives**.
* **Domain primitive = runtime execution**: Owns operational data and enforces state machines.
* **Graph is a read projection**: Operational data lives in domains; the graph receives projections for analysis, transformation, and agent context (per §2 Principle 6).

> **Important**: Transformation remains the source of truth for design artifacts; the graph stores references and read-optimised views, not the authoritative records.

### 10.5 Extension alignment (478)

478 describes how tenant-specific primitives fit into the domain portfolio:

* **Platform primitives** (§3, §4) run in shared or dedicated namespaces depending on SKU
* **Tenant custom primitives** (478 §4.7) always run in isolated `tenant-{id}-{env}-cust` namespaces
* The E2E flow (478 §6) describes how custom primitives are scaffolded via Backstage and deployed via GitOps
* **PrimitiveImplementationDraft** (478 §8) is a first-class artifact in Transformation for tracking primitive creation

This extends the domain portfolio in §3 with tenant-specific variants while maintaining the security invariants in §7.

---

## 11. Terminology

| 475 Term | Related Terms | Notes |
|----------|---------------|-------|
| Domain primitive | DomainService (deprecated) | Consistent across 470-476 |
| Process primitive | Workflow (305) | Runtime that implements Temporal workflows informed by ProcessDefinitions (code, not dynamic BPMN execution) |
| ProcessDefinition | BPMN model (deprecated) | Design-time artifact in Transformation Domain primitive |
| Agent primitive | AgentRuntime (deprecated) | Runtime that implements agent behavior informed by AgentDefinitions |
| AgentDefinition | Agent config (deprecated) | Design-time artifact in Transformation Domain primitive |

**Clarification**: AgentDefinitions are design-time artifacts owned by **Transformation Domain primitive** (§3.2), modelled via Transformation design tooling UIs. There is no separate "Transformation design tooling service"—Transformation design tooling is the UI layer that calls Transformation APIs. Agent primitives are runtime components that execute AgentDefinitions.
