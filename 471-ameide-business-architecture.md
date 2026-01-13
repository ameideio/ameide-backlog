# 471 — Ameide Business Architecture

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
> **Core Invariants**: See [470-ameide-vision.md §0 "Ameide Core Invariants"](470-ameide-vision.md) for the canonical list (six primitives: Domain/Process/Agent/UISurface/Projection/Integration, Graph read-only, Transformation as domain, proto chain, tenant isolation, Backstage internal).
>
> **Deployment Implementation**:
> - [465-applicationset-architecture.md](465-applicationset-architecture.md) – Per-environment deployments
> - [443-tenancy-models.md](443-tenancy-models.md) – Multi-tenancy SKUs
> - [467-backstage.md](467-backstage.md) – Backstage implementation (§4 platform factory)

---

## Grounding & contract alignment

- **Primitive & tenancy model:** Applies the primitive-kind and tenant/organization invariants from `470-ameide-vision.md` to concrete business constructs (tenants, orgs, domains, processes, agents, workspaces, projections, integrations) that are later realized as primitives/CRDs in `472-ameide-information-application.md`, `473-ameide-technology.md`, `477-primitive-stack.md`, and `520-primitives-stack-v2.md`.  
- **Transformation/Scrum context:** Treats Transformation (including Agile/Scrum/TOGAF ADM) as “just another domain + processes + agents,” providing the business framing that the Scrum stack (`367-1-scrum-transformation.md`, `506-scrum-vertical-v2.md`, `508-scrum-protos.md`) and agent backlogs (`505-agent-developer-v2*.md`) specialize.  
- **Downstream consumers:** Service catalog, operator, and CLI backlogs (`475-ameide-domains.md`, `481-service-catalog.md`, `495-ameide-operators.md`, `496-eda-principles-v2.md`, `484a-484f`) rely on these business concepts when describing primitive portfolios and guardrails.

---

**Status:** Draft v1
**Audience:** Product, domain architects, transformation leads, principal engineers

This doc describes **how Ameide is used as a business platform**: how tenants, orgs, processes, agents and workspaces fit together, and how "run the business" and "change the business" are modelled.

It assumes the high‑level vision/principles are accepted and stops short of implementation details (those go into App/Info and Tech Architecture).

## Layer header (Business/Strategy)

- **Primary ArchiMate layer(s):** Strategy + Business (capabilities, value streams, business processes, roles, outcomes).
- **Primary element types used:** Capability, Value Stream, Business Process, Business Actor/Role, Work Package (Implementation & Migration terminology only).
- **Out-of-scope layers:** Application and Technology (except short “realization notes” that point to `472`/`473`).
- **Secondary layers referenced:** Application (only in clearly marked realization notes); Implementation & Migration (work packages/initiatives).
- **Allowed nouns:** capability, value stream, business process, role/actor, outcome/value, policy, initiative/work package.
- **Prohibited unless qualified:** process, service, domain, event (must be qualified per `470-ameide-vision.md` §0.2).

## Implementation Status (2025-02-14)

- ✅ Tenant → organization → membership behavior is live inside the platform domain service (`services/platform/src/tenants/service.ts:1-160`, `services/platform/src/organizations/service.ts:1-120`).
- ✅ Process primitives already execute in Temporal namespaces/task queues through the shared facade (`services/workflows/src/temporal/facade.ts:78-130`).
- ⚠️ Transformation design artifacts (ProcessDefinitions/AgentDefinitions) still live outside the Transformation domain, so agents can’t yet govern them centrally (`services/transformation/src/transformations/service.ts:1-75`).
- ✅ Backstage templates and CLI scaffolds use the Domain/Process/Agent primitive vocabulary (`service_catalog/domains/_primitive/template.yaml:1-40`, `service_catalog/processes/_primitive/template.yaml:1-40`, `service_catalog/agents/_primitive/template.yaml:1-32`).

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

* **Organization (Org)**

  * Business unit within a tenant — what end users identify as *their Ameide org*.
  * Holds teams, memberships, roles, domain/process configuration, and transformation backlog.
  * (Note: distinct from UI "workspaces" which are configurable page layouts) 

* **User**

  * Authenticated identity, via self‑serve signup or enterprise SSO/SCIM. 

* **Memberships, teams, roles**

  * Memberships bind users to orgs with roles (`admin`, `contributor`, `viewer`, etc). 
  * Teams group members for process responsibility (Sales team for L2O, Architecture team for Transformation).

This model is the anchor for all business flows (onboarding, invitations, SSO, billing). 

### 2.2 Design-time artifacts vs runtime primitives

Ameide separates **design-time artifacts** (what to build) from **runtime primitives** (what executes):

#### Design-Time (inside the Transformation Domain)

* **ProcessDefinition** – BPMN-compliant artifact produced by a custom React Flow modeller.
  * Defines process stages, tasks, gateways, and bindings to domains/agents.
  * Stored and versioned in the Transformation Domain with revisions & promotions (modelled via Transformation design tooling UIs).

* **AgentDefinition** – Declarative spec for an agent.
  * Tools (domain/process APIs), orchestration graph, scope, risk tier, policies.
  * Stored alongside ProcessDefinitions in the Transformation Domain (modelled via Transformation design tooling UIs).

#### Runtime primitives & CRDs

Runtime primitives are represented as Ameide custom resources (Domain/Process/Agent/UISurface/Projection/Integration CRDs) that encode the desired state for each primitive. Operators reconcile those CRDs into workloads and supporting infrastructure; GitOps applies only the CR YAML.

1. **Domain primitive**

   * Owns a bounded business context: data, rules, events.
   * Domain primitives also store any domain-level definitions (config, process/agent specs owned by that domain) according to their proto schema.
   * Examples:

     * Sales: Leads, Opportunities, Orders.
     * Transformation: Initiatives, Epics, Backlog Items, ProcessDefinitions, AgentDefinitions, all Transformation design tooling artifacts.
     * Identity/Onboarding: Tenants, Organizations, Users, Invitations.
   * Exposed via proto‑based APIs as part of the core contract.

2. **Process primitive**

   * Executes a **ProcessDefinition** loaded from the Transformation Domain.
   * Manages process instances, state, retries, compensation (backed by Temporal).
   * Process tasks call into Domain primitives and may invoke Agent primitives where non-deterministic work is acceptable.
   * Runtime spec lives in a `Process` CRD.

3. **Agent primitive**

   * Executes an **AgentDefinition** loaded from the Transformation Domain.
   * Encapsulates an LLM-backed "worker" with tools and policies: transformation assistant, coder/refactor agent, L2O coaching bot.
   * Always invoked via domain/process APIs; does **not** become a separate source of truth.
   * Runtime spec lives in an `Agent` CRD.

4. **UISurface primitive**

   * A user‑facing Next.js experience:

     * Process view (pipeline/board for L2O, stages for Onboarding).
     * Entity view (grid/form for Orders, Opportunities, Tenants).
   * Talks only to the platform via the TS SDK, never raw gRPC.
   * Runtime spec lives in a `UISurface` CRD.

Business architecture cares mostly about *how* tenants combine these building blocks; implementation details (services, transports) live in other docs. 

### 2.3 Implementation guardrails (proto-first + event-driven)

All primitives follow the proto-first chain described in [472 §2.7](472-ameide-information-application.md#27-contract--implementation--runtime-chain): Buf modules in `packages/ameide_core_proto` define the contracts, workspace SDKs (TS/Go/Py) consume them, and GitOps manifests record which image/proto versions run in each environment. Go-based Process/Domain primitives also standardize on Watermill's CQRS component for command/event routing (see [472 §3.3.1](472-ameide-information-application.md#331-event-driven-cqrs-runtime-watermill)). Business flows therefore assume commands mutate write models, events update read models, and read-only queries hit the projections maintained by those handlers.

**Command/event naming discipline**: APIs express business intent, not field changes. Commands are verbs like `PlaceOrder`, `ApproveQuote`; never generic CRUD like `UpdateOrder`, `SetStatus`. This ensures the domain owns invariants and callers don't bypass business logic. See [472 §2.8.6](472-ameide-information-application.md#286-cqrs-recipe-commands--queries--events) for the full pattern and [484 §4.0.2.2](484-ameide-cli.md) for CLI enforcement.

---

## 3. Transformation as a first‑class domain

### 3.1 Purpose of the Transformation domain

Instead of treating "implementation" as something done off‑platform, **Transformation** is itself a Domain primitive:

* Entities:

  * **Initiatives, Epics, Work Packages** – high‑level change containers.
    * Language note: “Work Package” here aligns to ArchiMate Implementation & Migration “Work Package” (a time/resource constrained series of actions).
  * **Backlog Items** – concrete change requests (e.g. "Add approvals to L2O stage 3").
  * **ProcessDefinitions** – BPMN-compliant process models (L2O variants, custom workflows).
  * **AgentDefinitions** – Agent specs with tools, policies, risk tiers.
  * **Design Artifacts** – ArchiMate, Markdown tied together via Transformation design tooling.
* Processes:

  * **Agile** (Scrum / Kanban) or **TOGAF ADM**‑inspired transformation cycles, modelled as ProcessDefinitions executed by Process primitives.
* Agents:

  * **Transformation Agent primitives** that:

    * Analyse current state (using Transformation design tooling + graph + runtime metrics).
    * Propose new/changed Domain primitives, ProcessDefinitions, AgentDefinitions, Workspaces.
    * Drive Backstage template runs to materialise those changes.
  * A more technical flavour is the `core-platform-coder` Agent primitive, which can refactor tests/code as part of implementation tasks.

This domain owns the **life cycle of platform change** for each tenant: from requirement to design to deployment.

### 3.2 Transformation design tooling as the modelling experience for Transformation

> **Core Definitions** (see [470-ameide-vision.md §0](470-ameide-vision.md)):
> - The **Transformation Domain** is the knowledge store (initiatives, backlog items, artifacts, promotion state).
> - **Transformation design tooling** is the modelling experience architects and agents use to edit those artifacts via the Transformation APIs.
> - **Graph** contains *projections* of these artifacts for cross-domain queries; the canonical records stay in Transformation.

Transformation design tooling provides a structured modelling experience for design-time artifacts:

* **ProcessDefinitions** – BPMN-compliant process models (the single source of truth for process logic).
* **AgentDefinitions** – Agent specs with tools, orchestration graphs, policies.
* ArchiMate 3.2 models/views of business/application/technology architecture.
* Markdown specs / decision records.

All artifacts are **stored by the Transformation Domain** (optionally event-sourced internally):

* **Event‑sourced pattern** – commands → revisions → snapshots → promotions (within Transformation Domain).
* **Multi‑tenant** – partitioned by tenant/org.
* **Governed** – promotion paths and validations (e.g. secret scan, BPMN lint) are central to Transformation.

From a business‑architecture perspective:

> "The **Transformation Domain** is the single source of truth for all design-time artifacts. **Transformation design tooling** is the set of modelling UIs (BPMN editor, diagram editor, Markdown editor) that call the Transformation Domain APIs—Transformation design tooling does not own storage. Process primitives and Agent primitives implement runtime behavior that *follows* definitions from the Transformation Domain; they do not store design-time artifacts themselves."

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

> **Important:** Backstage is an **internal factory** for Ameide engineers and transformation agents; **tenants never see Backstage**. Tenant-facing UX is delivered through UISurface primitives (Next.js apps), not through Backstage.

Backstage isn't a “capability factory”; it is an internal **platform factory** that accelerates delivery of application components (primitives) and their contracts, which in turn realize application services that support capabilities (see `470-ameide-vision.md` §0.2).

From a business angle:

> **Application realization note:** capabilities serve value streams realized by business processes; business processes use application services realized by primitives. This section describes the factory that scaffolds primitives and their contracts (not capabilities themselves).

* Each **Backstage template** scaffolds an **application component type** (primitive kind):

  * `Domain primitive template` – create a new DDD domain with proto APIs.
  * `Process primitive template` – create a runtime skeleton for Process primitives whose workflow code is derived from ProcessDefinitions in Transformation design tooling (via CLI/agents), but that does **not** load or interpret those definitions at runtime.
  * `Agent primitive template` – create a runtime skeleton for Agent primitives whose behavior follows AgentDefinitions, with translation from design-time specs to code handled by CLI/agents, not by operators.
  * `UISurface template` – spin up new UX for domain/process combinations.

* The **Transformation Domain** and its Agent primitives populate Backstage with:

  * Tenant‑specific config (L2O v2 for Tenant A).
  * Shared, standard product templates (canonical L2O/O2C, Onboarding flow, Scrum board).

* Approved changes become:

  * Git commits + GitOps changes in infra repos (Design ≠ Deploy ≠ Runtime).

Business‑wise, this gives you:

* A "**service catalog**" of reusable domain/process/agent/workspace building blocks.
* A standard place where **transformation Agent primitives** "press the buttons" to generate new features rather than talking to ad‑hoc APIs.

### 4.1 Tenant extensions

Ameide supports two coordinated extension tiers:

* **Tier 1 – Small extensions via WASM**
  * Tenant- and agent-authored `ExtensionDefinition` artifacts live in the Transformation domain.
  * At runtime they execute inside the shared `extensions-runtime` service in `ameide-{env}` as sandboxed WASM.
  * Used for small, deterministic custom logic (extra validations, pricing tweaks, routing rules) in processes, domains, or agents.
  * See [479-ameide-extensibility-wasm.md](479-ameide-extensibility-wasm.md).

* **Tier 2 – Large extensions via primitives**
  * Full Domain/Process/Agent primitives scaffolded via Backstage templates.
  * Deployed into tenant namespaces (`tenant-{id}-{env}-cust` and `tenant-{id}-{env}-base`) according to SKU.
  * Used for new domains, major process variants, or heavy integrations.
  * See [478-ameide-extensions.md](478-ameide-extensions.md).

The remainder of this subsection describes the Tier 2 primitive path. When tenants require custom primitives beyond what the standard product provides:

* Backstage templates calculate the correct **target namespace** based on tenant SKU.
* Custom code is always isolated in `tenant-{id}-{env}-cust` namespaces.
* The full E2E flow from requirement to running primitives is managed via the **PrimitiveImplementationDraft** pattern.

> **See [478-ameide-extensions.md](478-ameide-extensions.md)** for the complete tenant extension model, namespace strategy by SKU, and the step-by-step primitive creation workflow.

---

## 5. Key business capabilities

### 5.1 Run‑the‑business capabilities

These are processes and domains tenants use day‑to‑day:

* **Lead‑to‑Order (L2O) / Opportunity‑to‑Cash (O2C)**

  * Cross‑domain processes connecting:

    * Sales domain (Leads, Opportunities, Offers, Orders).
    * Product/Pricing domain.
    * Finance/Invoicing domain (later).
  * Implemented as Process primitives; business sees:

    * Pipelines, SLAs, KPIs, approvals.
    * Domain workspaces for data “behind” each stage.

* **Tenant Onboarding & Identity**

  * Now specified as a Process primitive over the Identity/Platform domain:

    * Self‑serve signup, invitation, SSO, billing, realm provisioning.

* **Support / Case Management** (future domain)

  * Tickets, SLAs, escalations.

* **Billing & Entitlements**

  * Subscription management, entitlements, seat limits, usage metering (tying into billing provider). 

### 5.2 Change‑the‑business capabilities

These are processes that **change** L2O, Onboarding, etc:

* **Transform with Ameide** (Transformation process)

  * Intake of requirements from business stakeholders.
  * Analysis of AS‑IS vs TO‑BE using Transformation design tooling and runtime data.
  * Design of new ProcessDefinitions, AgentDefinitions, Domain primitives, Workspaces.
  * Backstage template runs and rollout.
  * Continuous improvement loops (OKRs, KPIs).

**Workspace Governance Model:**

* **Standard workspaces** (tenant-wide defaults) are part of transformation/design and go through governance gates (draft → review → promoted).
* **Personal workspaces** (user layout customization) are a day-to-day productivity feature and do not require governance approval.

This enables flexibility without sacrificing control: governed defaults ensure consistency; user personalization enables individual productivity.

* **IPA → new model**

  * The old IPA domain (bundling workflows + agents + tools) becomes just **one pattern** of combining Domain + Process + Agent.
  * No IPA runtime surface for tenants; everything is described in terms of domains/processes/agents.

Agent primitives support both:

* **Run‑time Agent primitives** – e.g. an L2O assistant inside the sales workspace.
* **Transformation Agent primitives** – e.g. a backlog/architecture assistant helping define the next ProcessDefinition variant.

---

## 6. Business lifecycle (end‑to‑end tenant journey)

### 6.1 Phase 1 – Discover, trial, onboard

1. **Lead & trial**

   * Originates outside or inside Ameide (CRM, marketing forms).
   * When a prospect signs up, they enter the **Onboarding Process primitive**.
   * The identity/onboarding domain ensures org, memberships, and tenant realm are created correctly.

2. **Org & first admin**

   * A first admin user completes a 3‑step wizard (org name, slug, basic preferences).
   * An initial **Transformation initiative** is created automatically:

     * “Implement core L2O and Onboarding flows for Tenant X.”

3. **Default capabilities**

   * By default, a tenant gets:

     * Standard L2O/O2C Process primitives and domains in “vanilla” form.
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

   For small, per-tenant business rules we prefer Tier 1 WASM extensions (no new primitives); Tier 2 primitives are reserved for larger or integration-heavy changes (see 479).

6. **Governance & approvals**

   Before generation, approved designs pass through governance gates:

   * **Human review**

     * BPMN and ArchiMate artifacts reviewed by domain architects.
     * Risk/impact assessment using runtime metrics and Transformation design tooling knowledge.

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
       * Process primitives (new L2O BPMN + runtime config).
       * UISurface variants for Tenant X’s sales team.
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
     * Provide suggestions based on historical metrics and Transformation design tooling knowledge.

10. **Feedback loop**

    * Process performance is projected into graph/Transformation design tooling for analysis.
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

* Owns **L2O/O2C Process primitives** for the org.
* Journeys:

  * Monitor KPIs and bottlenecks.
  * Request process changes through Transformation:

    * Adds new steps, SLAs, or approvals.
  * Accept or reject agent‑proposed tweaks based on outcomes.

### 7.3 Transformation Lead / Architect

* Lives in the **Transformation workspace**:

  * Manages initiatives and backlog.
  * Works with Transformation design tooling artifacts (BPMN, ArchiMate diagrams, Markdown specs).
  * Collaborates with agents (incl. coder) on solution outlines.
* Pushes changes through Backstage‑backed governance.

### 7.4 Platform Administrator

* Maintains **tenant-level configuration**:

  * Identity & SSO, SCIM, domain verification, billing. 
  * Data residency & security policies.
* Approves transformation proposals that cross compliance boundaries.

### 7.5 Agent personas (internal)

* **Transformation Agent**

  * Reads from Transformation + Transformation design tooling, proposes new capabilities.
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
  * Backed by BPMN and Process primitives.

* **Workspace view**

  * Secondary: entity tables, forms, reports.
  * Backed by Domain primitives and proto APIs.

Business consequences:

* Training and documentation emphasise **“how our L2O works”** (the process) over “where to click to see orders”.
* Transformation changes are expressed as **process changes** first, with domain/UX changes derived.

---

## 9. Legacy alignment: IPA and current stack

From a business‑architecture point of view:

* The **IPA Architecture** was an earlier way of describing “composed automations” (workflows + agents + tools) as first‑class models and builder flows. 
* In the new framing:

  * An IPA = a **bundle of Domain primitives + Process primitives + Agent primitives** referenced together for a specific outcome.
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

* **Graph = design‑time knowledge projection**: Elements *represent* BPMN models, initiatives, backlog items, etc., **projected from the Transformation and other Domain primitives**.
* **Process primitive = runtime execution**: Implements Temporal workflows whose behavior is derived from ProcessDefinitions stored in the Transformation Domain, but the compilation/translation step from design‑time artifacts to code happens in CLI/agents and the primitive’s source, not in operators.
* **Graph is a read projection**: Operational data lives in domains; the graph receives projections for analysis, transformation, and agent context.

> **Important**: Transformation remains the source of truth for transformation artifacts; the graph stores references and read-optimised views, not the authoritative records.

This means: Transformation domain entities (initiatives, backlogs, artifacts) are projected into the graph as Elements, enabling cross‑methodology reporting and agent‑assisted design—but the canonical records remain in the Transformation Domain.

---

## 11. Terminology

| Legacy Term | New Term (471) |
|-------------|----------------|
| IPA (Intelligent Process Automation) | Domain primitive + Process primitive + Agent primitive bundle |
| Platform Workflows (305) | Process primitive (Temporal-backed runtime informed by ProcessDefinitions, not a BPMN interpreter) |
| AgentRuntime (310) | Agent primitive (runtime implementing behavior informed by AgentDefinitions) |
| DomainService | Domain primitive |

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

4. **Role of Transformation design tooling for non‑technical stakeholders**

   * How much of Transformation design tooling (BPMN, ArchiMate) is visible to business users vs architects only?

---

## 13. Related documents

This Business Architecture should be read together with:

* **Application & Information Architecture** – maps these concepts onto services, APIs, data models (proto‑based APIs & SDK).
* **Technology Architecture** – describes K8s, Temporal-backed Process primitives, Transformation Domain, observability, and Backstage templates.
* **Domain Specs** – L2O/O2C, Transformation, Onboarding/Identity.
* **Refactoring & Migration Plan** – evolution path to this structure.
