Here we go — Business Architecture time 🔧🏗️

> **Cross-References (Vision Suite)**:
>
> | Document | Purpose |
> |----------|---------|
> | [470-ameide-vision.md](470-ameide-vision.md) | Vision, rationale, principles |
> | [472-ameide-information-application.md](472-ameide-information-application.md) | Application architecture |
> | [473-ameide-technology.md](473-ameide-technology.md) | Technology stack |
> | [475-ameide-domains.md](475-ameide-domains.md) | Domain portfolio |
> | [476-ameide-security-trust.md](476-ameide-security-trust.md) | Security architecture |
> | [478-ameide-extensions.md](478-ameide-extensions.md) | Tenant extension lifecycle |
> | [479-ameide-extensibility-wasm.md](479-ameide-extensibility-wasm.md) | Tier 1 WASM + Tier 2 extensibility model |
>
> **Deployment Implementation**:
> - [465-applicationset-architecture.md](465-applicationset-architecture.md) – Per-environment deployments
> - [443-tenancy-models.md](443-tenancy-models.md) – Multi-tenancy SKUs
> - [477-backstage.md](477-backstage.md) – Backstage implementation (§4 platform factory)

---

# 4xy – Ameide Business Architecture

**Status:** Draft v1
**Audience:** Product, domain architects, transformation leads, principal engineers

This doc describes **how Ameide is used as a business platform**: how tenants, orgs, processes, agents and workspaces fit together, and how "run the business" and "change the business" are modelled.

It assumes the high‑level vision/principles are accepted and stops short of implementation details (those go into App/Info and Tech Architecture).

---

## 1. Scope & intent

This document answers:

* What are the **core business concepts** (tenants, orgs, domains, processes, agents, workspaces)?
* How does a **tenant** go from first access to running a tailored L2O/O2C?
* How is **transformation** (requirements, roadmaps, architecture, design) modelled as just another domain?
* How do **agents** participate without becoming “magic black boxes”?
* How do we keep the experience **process‑first**, with traditional ERP workspaces as a supporting view?

It is **not** about container layout, CRDs, or code structure — those belong to App/Info and Tech Architecture.

---

## 2. Core business concepts

### 2.1 Tenants, organisations, users

At the outermost layer we keep the two‑level tenancy model already defined in onboarding/identity:

* **Tenant**

  * Infrastructure isolation unit (realm, DB schema/RLS partition, routing).
  * Has settings like region, data residency, billing tier. 

* **Organization**

  * “Workspace” within a tenant — what end users identify as *their Ameide org*.
  * Holds teams, memberships, roles, domain/process configuration, and transformation backlog. 

* **User**

  * Authenticated identity, via self‑serve signup or enterprise SSO/SCIM. 

* **Memberships, teams, roles**

  * Memberships bind users to orgs with roles (`admin`, `contributor`, `viewer`, etc). 
  * Teams group members for process responsibility (Sales team for L2O, Architecture team for Transformation).

This model is already implemented today and is the anchor for all business flows (onboarding, invitations, SSO, billing). 

### 2.2 Design-time artifacts vs runtime controllers

Ameide separates **design-time artifacts** (what to build) from **runtime controllers** (what executes):

#### Design-Time (in Transformation DomainController)

* **ProcessDefinition** – BPMN-compliant artifact produced by a custom React Flow modeller.
  * Defines process stages, tasks, gateways, and bindings to domains/agents.
  * Stored and versioned in Transformation DomainController with revisions & promotions (modelled via UAF UIs).

* **AgentDefinition** – Declarative spec for an agent.
  * Tools (domain/process APIs), orchestration graph, scope, risk tier, policies.
  * Stored alongside ProcessDefinitions in Transformation DomainController (modelled via UAF UIs).

#### Runtime Controllers

Runtime controllers are represented as Ameide custom resources (`IntelligentDomainController`, `IntelligentProcessController`, `IntelligentAgentController`) that encode the desired state for each controller. Operators reconcile those CRs into Deployments, Services, Temporal workers, and supporting infrastructure; GitOps applies only the CR YAML.

1. **DomainController (Domain)**

   * Owns a bounded business context: data, rules, events.
   * DomainControllers also store any domain-level definitions (config, process/agent specs owned by that domain) according to their proto schema.
   * Examples:

     * Sales: Leads, Opportunities, Orders.
     * Transformation: Initiatives, Epics, Backlog Items, ProcessDefinitions, AgentDefinitions, all UAF artifacts.
     * Identity/Onboarding: Tenants, Organizations, Users, Invitations.
   * Exposed via proto‑based APIs as part of the core contract.

2. **ProcessController (Process)**

   * Executes a **ProcessDefinition** loaded from Transformation DomainController.
   * Manages process instances, state, retries, compensation (backed by Temporal).
   * Process tasks call into DomainControllers and may invoke AgentControllers where non-deterministic work is acceptable.
   * Runtime spec lives in an `IntelligentProcessController` CR (IPC).

3. **AgentController (Agent)**

   * Executes an **AgentDefinition** loaded from Transformation DomainController.
   * Encapsulates an LLM-backed "worker" with tools and policies: transformation assistant, coder/refactor agent, L2O coaching bot.
   * Always invoked via domain/process APIs; does **not** become a separate source of truth.
   * Runtime spec lives in an `IntelligentAgentController` CR (IAC).

4. **UIWorkspace (Workspace)**

   * A user‑facing Next.js experience:

     * Process view (pipeline/board for L2O, stages for Onboarding).
     * Entity view (grid/form for Orders, Opportunities, Tenants).
   * Talks only to the platform via the TS SDK, never raw gRPC.

Business architecture cares mostly about *how* tenants combine these building blocks; implementation details (services, transports) live in other docs. 

---

## 3. Transformation as a first‑class domain

### 3.1 Purpose of the Transformation domain

Instead of treating "implementation" as something done off‑platform, **Transformation** is itself a DomainController:

* Entities:

  * **Initiatives, Epics, Work Packages** – high‑level change containers.
  * **Backlog Items** – concrete change requests (e.g. "Add approvals to L2O stage 3").
  * **ProcessDefinitions** – BPMN-compliant process models (L2O variants, custom workflows).
  * **AgentDefinitions** – Agent specs with tools, policies, risk tiers.
  * **Design Artifacts** – ArchiMate, Markdown tied together via UAF.
* Processes:

  * **Agile** (Scrum / Kanban) or **TOGAF ADM**‑inspired transformation cycles, modelled as ProcessDefinitions executed by ProcessControllers.
* Agents:

  * **Transformation AgentControllers** that:

    * Analyse current state (using UAF + graph + runtime metrics).
    * Propose new/changed DomainControllers, ProcessDefinitions, AgentDefinitions, Workspaces.
    * Drive Backstage template runs to materialise those changes.
  * A more technical flavour is the `core-platform-coder` AgentController, which can refactor tests/code as part of implementation tasks.

This domain owns the **life cycle of platform change** for each tenant: from requirement to design to deployment.

### 3.2 UAF as the modelling experience for Transformation

> **Core Definitions** (see [470-ameide-vision.md §0](470-ameide-vision.md)):
> - The **Transformation DomainController** is the knowledge store (initiatives, backlog items, artifacts, promotion state).
> - **UAF** is the modelling experience architects and agents use to edit those artifacts via the Transformation APIs.
> - **Graph** contains *projections* of these artifacts for cross-domain queries; the canonical records stay in Transformation.

The **Unified Artifact Framework** (UAF) provides a structured modelling experience for design-time artifacts:

* **ProcessDefinitions** – BPMN-compliant process models (the single source of truth for process logic).
* **AgentDefinitions** – Agent specs with tools, orchestration graphs, policies.
* ArchiMate models of application/business/technology architecture.
* Markdown specs / decision records.

All artifacts are **stored by the Transformation DomainController** (optionally event-sourced internally):

* **Event‑sourced pattern** – commands → revisions → snapshots → promotions (within Transformation DomainController).
* **Multi‑tenant** – partitioned by tenant/org.
* **Governed** – promotion paths and validations (e.g. secret scan, BPMN lint) are central to Transformation.

From a business‑architecture perspective:

> "The **Transformation DomainController** is the single source of truth for all design-time artifacts. **UAF** is the set of modelling UIs (BPMN editor, diagram editor, Markdown editor) that call the Transformation DomainController APIs—UAF does not own storage. ProcessControllers and AgentControllers execute definitions from the Transformation DomainController; they do not own the process/agent logic themselves."

### 3.3 Methodology selection & parallel initiatives

Each Transformation initiative selects a methodology profile at creation:

* **Scrum**: Sprint‑based delivery with backlogs, retrospectives, velocity tracking.
* **SAFe**: PI Planning, release trains, enterprise alignment ceremonies.
* **TOGAF ADM**: Architecture phases, deliverables, contracts, Architecture Board governance.

**Parallel initiatives**: Organizations can run multiple initiatives simultaneously with different methodologies. For example:

* Initiative A: Scrum for rapid L2O feature delivery.
* Initiative B: TOGAF ADM for enterprise architecture redesign.

Methodology profiles are tenant‑configurable, allowing organizations to:

* Customize ceremony cadences and sprint lengths.
* Add/remove governance gates per methodology.
* Define role mappings to their org structure (e.g. "Scrum Master" → internal role).

---

## 4. Backstage as the platform factory (business view)

Backstage isn't just an engineer portal; in this architecture it's the **factory for business capabilities**.

From a business angle:

* Each **Backstage template** corresponds to a *type of capability*:

  * `DomainController template` – create a new DDD domain with proto APIs.
  * `ProcessController template` – create a runtime that executes ProcessDefinitions from UAF.
  * `AgentController template` – create a runtime that executes AgentDefinitions from UAF.
  * `UIWorkspace template` – spin up new UX for domain/process combinations.

* The **Transformation DomainController** and its AgentControllers populate Backstage with:

  * Tenant‑specific config (L2O v2 for Tenant A).
  * Shared, standard product templates (canonical L2O/O2C, Onboarding flow, Scrum board).

* Approved changes become:

  * Git commits + GitOps changes in infra repos (Design ≠ Deploy ≠ Runtime).

Business‑wise, this gives you:

* A "**service catalog**" of reusable domain/process/agent/workspace building blocks.
* A standard place where **transformation AgentControllers** "press the buttons" to generate new features rather than talking to ad‑hoc APIs.

### 4.1 Tenant extensions

Ameide supports two coordinated extension tiers:

* **Tier 1 – Small extensions via WASM**
  * Tenant- and agent-authored `ExtensionDefinition` artifacts live in the Transformation domain.
  * At runtime they execute inside the shared `extensions-runtime` service in `ameide-{env}` as sandboxed WASM.
  * Used for small, deterministic custom logic (extra validations, pricing tweaks, routing rules) in processes, domains, or agents.
  * See [479-ameide-extensibility-wasm.md](479-ameide-extensibility-wasm.md).

* **Tier 2 – Large extensions via controllers**
  * Full Domain/Process/AgentControllers scaffolded via Backstage templates.
  * Deployed into tenant namespaces (`tenant-{id}-{env}-cust` and `tenant-{id}-{env}-base`) according to SKU.
  * Used for new domains, major process variants, or heavy integrations.
  * See [478-ameide-extensions.md](478-ameide-extensions.md).

The remainder of this subsection describes the Tier 2 controller path. When tenants require custom controllers beyond what the standard product provides:

* Backstage templates calculate the correct **target namespace** based on tenant SKU
* Custom code is always isolated in `tenant-{id}-{env}-cust` namespaces
* The full E2E flow from requirement to running controller is managed via the **ControllerImplementationDraft** pattern

> **See [478-ameide-extensions.md](478-ameide-extensions.md)** for the complete tenant extension model, namespace strategy by SKU, and the step-by-step controller creation workflow.

---

## 5. Key business capabilities

### 5.1 Run‑the‑business capabilities

These are processes and domains tenants use day‑to‑day:

* **Lead‑to‑Order (L2O) / Opportunity‑to‑Cash (O2C)**

  * Cross‑domain processes connecting:

    * Sales domain (Leads, Opportunities, Offers, Orders).
    * Product/Pricing domain.
    * Finance/Invoicing domain (later).
  * Implemented as ProcessControllers; business sees:

    * Pipelines, SLAs, KPIs, approvals.
    * Domain workspaces for data “behind” each stage.

* **Tenant Onboarding & Identity**

  * Now specified as a ProcessController over the Identity/Platform domain:

    * Self‑serve signup, invitation, SSO, billing, realm provisioning.

* **Support / Case Management** (future domain)

  * Tickets, SLAs, escalations.

* **Billing & Entitlements**

  * Subscription management, entitlements, seat limits, usage metering (tying into billing provider). 

### 5.2 Change‑the‑business capabilities

These are processes that **change** L2O, Onboarding, etc:

* **Transform with Ameide** (Transformation process)

  * Intake of requirements from business stakeholders.
  * Analysis of AS‑IS vs TO‑BE using UAF and runtime data.
  * Design of new ProcessDefinitions, AgentDefinitions, DomainControllers, Workspaces.
  * Backstage template runs and rollout.
  * Continuous improvement loops (OKRs, KPIs).

* **IPA → new model**

  * The old IPA domain (bundling workflows + agents + tools) becomes just **one pattern** of combining Domain + Process + Agent.
  * No IPA runtime surface for tenants; everything is described in terms of domains/processes/agents.

AgentControllers support both:

* **Run‑time AgentControllers** – e.g. an L2O assistant inside the sales workspace.
* **Transformation AgentControllers** – e.g. a backlog/architecture assistant helping define the next ProcessDefinition variant.

---

## 6. Business lifecycle (end‑to‑end tenant journey)

### 6.1 Phase 1 – Discover, trial, onboard

1. **Lead & trial**

   * Originates outside or inside Ameide (CRM, marketing forms).
   * When a prospect signs up, they enter the **Onboarding ProcessController**.
   * The identity/onboarding domain ensures org, memberships, and tenant realm are created correctly.

2. **Org & first admin**

   * A first admin user completes a 3‑step wizard (org name, slug, basic preferences).
   * An initial **Transformation initiative** is created automatically:

     * “Implement core L2O and Onboarding flows for Tenant X.”

3. **Default capabilities**

   * By default, a tenant gets:

     * Standard L2O/O2C ProcessControllers and domains in “vanilla” form.
     * Generic agents (help, documentation, explanation).
     * Basic workspaces for each process.

### 6.2 Phase 2 – Tailor L2O/O2C and workspaces

4. **Requirements articulation (business-side)**

   * Sales ops / business owners articulate requirements:

     * In chat with a **Transformation agent** (e.g. “we need a special approval step for deals > 100k”).
     * In structured forms or backlogs in the Transformation workspace.

5. **Design in Transformation domain**

   * Transformation domain (plus agent) turns these into:

     * Updated BPMN models (L2O_v2 with new stage).
     * Possibly new domain fields (e.g. “deal risk rating”).
     * Agent configuration for advising on the new process.

   For small, per-tenant business rules we prefer Tier 1 WASM extensions (no new controllers); Tier 2 controllers are reserved for larger or integration-heavy changes (see 479).

6. **Governance & approvals**

   Before generation, approved designs pass through governance gates:

   * **Human review**

     * BPMN and ArchiMate artifacts reviewed by domain architects.
     * Risk/impact assessment using runtime metrics and UAF knowledge.

   * **Architecture Board** (for complex transformations)

     * TOGAF ADM phases require Architecture Board sign‑off.
     * Enterprise‑wide changes escalate to governance committee.
     * Roles: Enterprise Architect, Sponsor, Domain Representatives.

   * **Compliance checks**

     * Secret scanning (no credentials in artifacts).
     * BPMN lint validation.
     * Test coverage requirements.
     * Policy alignment verification.

   * **Approval workflow**

     * Transformation agents propose changes.
     * Human reviewers approve/reject/request changes.
     * Approved changes become Git commits via Backstage.
     * GitOps promotion follows environment gates (dev → staging → prod).

   Agents are sandboxed: they can only call domain/process APIs via SDK, and their proposals require explicit human approval before affecting durable state.

7. **Generation via Backstage**

   * Approved designs are implemented by:

     * Running Backstage templates for:

       * Domain changes (Adding fields & APIs for the Sales domain).
       * ProcessControllers (new L2O BPMN + runtime config).
       * UIWorkspace variants for Tenant X’s sales team.
     * Optionally, using the coder agent for code/test adjustments within those services.

### 6.3 Phase 3 – Run, measure, iterate

8. **Go‑live**

   * The new variant is deployed to the tenant via GitOps (North‑Star ops model).

9. **Operations**

   * Users work primarily in:

     * L2O **process view** (kanban/stage board with KPIs).
     * L2O **workspace view** for detailed order/customer records.
   * Agents:

     * Offer contextual help (“why is this stuck in stage 3?”).
     * Provide suggestions based on historical metrics and UAF knowledge.

10. **Feedback loop**

    * Process performance is projected into graph/UAF for analysis.
    * Transformation domain picks up signals:

      * E.g. slow stage, high failure rate, low conversion.
    * New change requests start the cycle again.

---

## 7. Personas & their journeys

### 7.1 Sales Representative (run‑the‑business)

* Works in the **L2O workspace and board**:

  * Moves opportunities through process stages.
  * Gets inline recommendations from L2O agents (next best action, discount guardrails).
* Rarely interacts with Transformation directly; sees the platform as a “smart ERP/CRM”.

### 7.2 Process Owner / Sales Operations

* Owns **L2O/O2C ProcessControllers** for the org.
* Journeys:

  * Monitor KPIs and bottlenecks.
  * Request process changes through Transformation:

    * Adds new steps, SLAs, or approvals.
  * Accept or reject agent‑proposed tweaks based on outcomes.

### 7.3 Transformation Lead / Architect

* Lives in the **Transformation workspace**:

  * Manages initiatives and backlog.
  * Works with UAF artifacts (BPMN, ArchiMate diagrams, Markdown specs).
  * Collaborates with agents (incl. coder) on solution outlines.
* Pushes changes through Backstage‑backed governance.

### 7.4 Platform Administrator

* Maintains **tenant-level configuration**:

  * Identity & SSO, SCIM, domain verification, billing. 
  * Data residency & security policies.
* Approves transformation proposals that cross compliance boundaries.

### 7.5 Agent personas (internal)

* **Transformation Agent**

  * Reads from Transformation + UAF, proposes new capabilities.
* **Coder Agent**

  * Operates on code/test artifacts; invoked as part of transformation tasks. 

Business architecture needs to treat these as *tools in the transformation domain*, not independent systems.

---

## 8. Process‑first vs workspace‑first UX (business stance)

**Principle:** *Process first, workspaces second.*

For any major area like Sales or Transformation:

* **Process view**

  * Primary entry point: pipeline boards, process maps, dashboards.
  * Encodes stages, SLAs, transitions, and roles.
  * Backed by BPMN and ProcessControllers.

* **Workspace view**

  * Secondary: entity tables, forms, reports.
  * Backed by DomainControllers and proto APIs.

Business consequences:

* Training and documentation emphasise **“how our L2O works”** (the process) over “where to click to see orders”.
* Transformation changes are expressed as **process changes** first, with domain/UX changes derived.

---

## 9. Legacy alignment: IPA and current stack

From a business‑architecture point of view:

* The **IPA Architecture** was an earlier way of describing “composed automations” (workflows + agents + tools) as first‑class models and builder flows. 
* In the new framing:

  * An IPA = a **bundle of DomainControllers + ProcessControllers + AgentControllers** referenced together for a specific outcome.
  * The “IPA builder” idea becomes part of:

    * Transformation domain (design).
    * Backstage (generation).
    * GitOps (deployment).
* The underlying technical benefits of IPA — domain models, persistence, central builder — are preserved but **not exposed as a separate product concept**.

Business communication should therefore:

* Avoid introducing “IPA” to customers.
* Speak consistently about **domains, processes, agents, and workspaces**.

---

## 10. Cross‑references

This Business Architecture should be read with the following documents:

| Document | Relationship | Alignment |
|----------|-------------|-----------|
| [470‑ameide‑vision](470-ameide-vision.md) | Parent vision & principles | Strong ✅ |
| [473‑ameide‑technology](473-ameide-technology.md) | Technology implementation | Strong ✅ |
| [475‑ameide‑domains](475-ameide-domains.md) | Domain portfolio & patterns | Strong ✅ |
| [300‑ameide‑metamodel](300-ameide-metamodel.md) | Element graph foundation | See §10.1 |
| [319‑onboarding](319-onboarding.md) | Identity/Onboarding domain | Strong ✅ |
| [333‑realms](333-realms.md) | Multi‑tenant realm model | Strong ✅ |
| [367‑series](367-1-safe.md) | Transformation methodologies | Strong ✅ |
| [478‑ameide‑extensions](478-ameide-extensions.md) | Tenant extension lifecycle | Strong ✅ |

### 10.1 Metamodel alignment (300)

The Element graph (300) and Business Architecture (471) coexist:

* **Graph = design‑time knowledge projection**: Elements *represent* BPMN models, initiatives, backlog items, etc., **projected from the Transformation and other DomainControllers**.
* **ProcessController = runtime execution**: Compiled from ProcessDefinitions stored in the Transformation DomainController into executable Temporal workflows.
* **Graph is a read projection**: Operational data lives in domains; the graph receives projections for analysis, transformation, and agent context.

> **Important**: Transformation remains the source of truth for transformation artifacts; the graph stores references and read-optimised views, not the authoritative records.

This means: Transformation domain entities (initiatives, backlogs, artifacts) are projected into the graph as Elements, enabling cross‑methodology reporting and agent‑assisted design—but the canonical records remain in the Transformation DomainController.

---

## 11. Notable gaps & issues

### 11.1 Terminology alignment

| Legacy Term | New Term (471) | Status |
|-------------|----------------|--------|
| IPA (Intelligent Process Automation) | DomainController + ProcessController + AgentController bundle | Deprecated in customer comms |
| Platform Workflows (305) | ProcessController (executes ProcessDefinitions from UAF) | 305 aligned to 471 |
| AgentRuntime (310) | AgentController (executes AgentDefinitions from UAF) | Aligned to 471 |
| DomainService | DomainController | Aligned to 471 |

### 11.2 Implementation gaps

| Gap | Description | Related Docs |
|-----|-------------|--------------|
| Realm‑per‑tenant | Production runs single realm; hybrid model recommended | 333, 428 |
| Onboarding ProcessController | Code‑first, not BPMN; SSO/billing pending | 319, 428 |
| Event publishing | Domain events (outbox + bus) not yet live | 319, 473 |
| SCIM 2.0 | Enterprise identity provisioning pending | 428 |

### 11.3 Open architectural tensions

1. **Graph vs. runtime separation**: Ensure Transformation domain entities project to `graph.elements` while ProcessControllers remain runtime‑only.
2. **Agent guardrails**: Agents propose changes via domain/process APIs; proposals require human review before becoming durable state.
3. **Multi‑org users**: Users with roles in multiple organizations need documented org‑switching UX (login realm, org selector, session management).

---

## 12. Open business questions

These are deliberately left for follow‑up design sessions and may become ADRs:

1. **Commercial packaging**

   * Do we sell domains (Sales, Transformation, Onboarding) and processes (L2O, O2C) as separate SKUs?
   * How do we meter usage (seats vs active processes vs transformations)?

2. **Partner ecosystem**

   * How do partners distribute their own domains/processes/agents through a catalog (Backstage‑backed marketplace)?
   * What governance do we require for third‑party transformations?

3. **Boundaries for tenant‑specific vs product‑level change**

   * When does a tenant change graduate into a product feature?
   * How do we avoid fragmentation across L2O variants?

4. **Role of UAF for non‑technical stakeholders**

   * How much of UAF (BPMN, ArchiMate) is visible to business users vs architects only?

---

## 13. Next documents

This Business Architecture should be read together with:

* **Application & Information Architecture** – maps these concepts onto services, APIs, data models (heavy use of proto‑based APIs & SDK).
* **Technology Architecture** – describes K8s, Temporal-backed ProcessControllers, Transformation DomainController (optionally event-sourced), observability, and how Backstage templates actually run.
* **Domain Specs** – especially:

  * L2O/O2C (Sales).
  * Transformation.
  * Onboarding/Identity.
* **Refactoring & Migration Plan** – how we evolve from current IPA‑centric pieces to this structure.

---

If you’d like, next we can tackle the **Application & Information Architecture** doc and turn all of this into boxes/arrows: which concrete services implement each Domain/Process/Agent/Workspace, how UAF/graph fit, and how Backstage + SDK tie into it.
