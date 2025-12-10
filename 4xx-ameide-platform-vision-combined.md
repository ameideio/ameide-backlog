
<!-- ═══════════════════════════════════════════════════════════════════════════ -->
<!-- FILE: 470-ameide-vision.md -->
<!-- ═══════════════════════════════════════════════════════════════════════════ -->

# 4xx – Ameide Platform Vision / Rationale / Principles

**Status:** Draft v1
**Owner:** Architecture / Product
**Intended audience:** Founders, product, principal engineers, domain/solution architects

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
> | [474-ameide-refactoring.md](474-ameide-refactoring.md) | Migration plan into Domain/Process/Agent/UISurface primitives + CRDs |
>
> **Deployment Implementation**:
> - [465-applicationset-architecture.md](465-applicationset-architecture.md) – GitOps deployment model
> - [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) – Rollout phases
> - [442-environment-isolation.md](442-environment-isolation.md) – Environment isolation
> - [446-namespace-isolation.md](446-namespace-isolation.md) – Namespace patterns

> **Comparative Briefs (Ameide vs incumbent ERPs)**:
> - [470a-ameide-vision-vs-d365.md](470a-ameide-vision-vs-d365.md) – Code-first Ameide platform compared to Microsoft D365FO's metadata/AOT paradigm.
> - [470b-ameide-vision-vs-saps4.md](470b-ameide-vision-vs-saps4.md) – Ameide positioning versus SAP S/4HANA's DDIC/CDS/Fiori/Customizing stack.
>
> These appendices translate the high-level Ameide principles in this document into head-to-head comparisons with the dominant metadata/configuration ERPs so product, field, and architecture teams can articulate what stays the same (coherence, discoverability) and what changes (code-first, AI operated, CRD-based infra) when pitching Ameide.

---

## 0. Core Definitions: Domain / Process / Agent / UISurface primitives, CRDs, and Graph

> **Domain primitives** own the authoritative data and rules for a bounded business context (Orders, Customers, Transformation, etc.) and expose proto-first APIs plus migrations/tests as normal code.
>
> **Process primitives** orchestrate multiple Domains and Agents into outcomes such as L2O or O2C by executing BPMN-compliant definitions.
>
> **Agent primitives** wrap non-deterministic workers (LLMs + typed tools) that make proposals but never store durable state themselves.
>
> **UISurface primitives** are user-facing entry points (process views or domain workspaces) implemented as code-first Next.js apps that rely on generated SDKs.
>
> **Domain / Process / Agent / UISurface CRDs** are the only application-level CRDs. Each one declares *how* the corresponding primitive is run (image, config, bindings, risk tier) so GitOps + operators can reconcile code into deployments.
>
> **Transformation design tooling** is the modelling UX (BPMN editor, diagram editor, Markdown/agent specs) that reads/writes artifacts in the Transformation Domain; it has no independent persistence.
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
| **Domain CRD** | Declarative runtime config for one Domain primitive (image, config, DB bindings, resources, observability). |
| **Process CRD** | Declarative runtime config for one Process primitive (image, process type, Temporal namespace/bindings, timeouts). |
| **Agent CRD** | Declarative runtime config for one Agent primitive (image, model/provider config, tool grants, risk tier). |
| **UISurface CRD** | Declarative runtime config for one UISurface primitive (image, routing, auth scopes, feature flags). |
| **Transformation design tooling** | Ameide’s modelling UX (BPMN editor, diagram editor, Markdown/agent specs) that stores artifacts in the Transformation Domain; the tooling itself is stateless. |
| **Graph** | Read-only knowledge projection that ingests selected data from primitives into a graph database for traversal; never a source of truth. |

> **Ameide Core Invariants (Canonical Reference)**
>
> All vision suite documents (470-480) must respect these invariants. Link here rather than restating.
>
> 1. **Four primitives only.** Domain / Process / Agent / UISurface are the only application-level CRDs. No other app-level CRD types.
>
> 2. **Graph is read-only.** Knowledge Graph is a projection layer for cross-domain queries. All writes go through Domain primitives. Graph is never a source of truth.
>
> 3. **Transformation is a domain; Transformation design tooling is UI.** The Transformation Domain primitive owns all design-time artifacts (ProcessDefinitions, AgentDefinitions, BPMN, etc.). Transformation design tooling is purely a UI layer (BPMN editor, diagram editor, Markdown editor) that calls Transformation Domain APIs—it has no independent persistence.
>
> 4. **Proto → SDK → Runtime chain.** Contracts live in `packages/ameide_core_proto` (Buf-managed). Generated SDKs flow to services. Docker images are tagged/labelled with proto version. GitOps manifests reference exact image + proto metadata. See [472 §2.7](472-ameide-information-application.md) for details.
>
> 5. **Tenant isolation via `tenant_id`.** Every business record carries `tenant_id`. Organization-level isolation via `organization_id` + RLS is the target model.
>
> 6. **CRD naming is Domain / Process / Agent / UISurface.** Earlier documents (e.g., 461) used `IntelligentDomainController (IDC)`, `IntelligentProcessController (IPC)`, `IntelligentAgentController (IAC)`. These names are **deprecated**.
>
> 7. **Backstage is internal only.** Backstage is the factory for Ameide engineers and transformation agents. Tenants never see Backstage; tenant UX is delivered through UISurface primitives.

> **Security Spine**
>
> Security is a cross-cutting concern addressed in [476-ameide-security-trust.md](476-ameide-security-trust.md). Key areas:
>
> - **Tenant/org isolation (RLS)**: Target model requires `tenant_id` + `organization_id` on all business tables with Row-Level Security. See [476 §3](476-ameide-security-trust.md) and [329-authz](329-authz.md).
> - **Agent governance**: AgentDefinitions carry `scope`, `risk_tier`, and `tool_grants` to constrain what agents can do. See [476 §7](476-ameide-security-trust.md) and [310](310-agent-runtime.md).
> - **Extension isolation**: Tier 1 WASM runs in sandboxed `extensions-runtime`; Tier 2 primitives run in isolated `tenant-{id}-{env}-cust` namespaces. See [476 §6](476-ameide-security-trust.md) and [478](478-ameide-extensions.md)/[479](479-ameide-extensibility-wasm.md)/[480](480-ameide-extensibility-wasm-service.md).

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
   * **Deployment-time**: Backstage templates, GitOps, and `Domain` / `Process` / `Agent` / `UISurface` CRs committed to Git.
   * **Runtime**: Operators reconcile those CRDs into Deployments, Services, Temporal workers, and other workloads on Kubernetes.

---

## 4. Conceptual model

At the highest level we standardise on four building blocks, with a clear **design-time vs runtime** split:

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

  * Indexes Domain, Process, Agent, and UISurface primitives plus their sources (repos, Helm releases, proto APIs).
* **Templates**:

  * Scaffold new Domain/Process/Agent/UISurface primitives using standard Ameide patterns (proto, SDK, Helm/Argo, CNPG, Temporal) and emit the corresponding CRDs that GitOps/Argo will apply.
* **Bridge**:

  * Listens to Transformation domain events and runs templates with specific parameters (e.g. "create L2O Process primitive variant for tenant X").

> **Extension Model**: For tenant-specific primitives and custom code isolation, see [478-ameide-extensions.md](478-ameide-extensions.md) which defines the namespace strategy by SKU and the E2E flow for primitive creation.

### 4.4 Proto-first contracts (Buf + SDK chain)

All primitives follow a single, enforced chain for contracts and runtime artifacts:

1. **Schema layer (Buf)** – `packages/ameide_core_proto` hosts the Buf module for every Domain/Process/Agent surface. Buf breaking-change checks run in CI (see [365-buf-sdks-v2.md](365-buf-sdks-v2.md)). Generated outputs land inside the SDK packages (TS/Go/Python) so services never import Buf registry stubs directly.
2. **Implementation layer (workspace SDKs)** – Services consume AmeideClient + proto types from the workspace SDKs per [402](402-buf-first-generated-clients.md), with language-specific rollout plans in [403](403-python-buf-first-migration-plan.md) and [404](404-go-buf-first-migration.md). Proto diffs therefore break dev/CI immediately.
3. **Runtime layer (GitOps)** – Docker images are built from those workspace SDKs and referenced by GitOps manifests (Argo CD). Metadata/annotations capture the proto + image versions (see [472 §2.7](472-ameide-information-application.md)).
4. **Publish track (outer loop)** – SDKs/configs still ship as product artifacts for external consumers per [388-ameide-sdks-north-star.md](388-ameide-sdks-north-star.md), but internal Rings 1/2 never wait on publishes.

This keeps “proto → SDK → deployment” consistent across every primitive and is the basis for tenant-specific extensions and catalog metadata referenced throughout the 47x suite.

### 4.5 Event-driven orchestration (Watermill CQRS)

Where Process/Domain primitives run inside Go services, we rely on Watermill’s CQRS component to implement command and event handlers with minimal boilerplate. Process primitives send commands (`CommandBus`) and publish events (`EventBus`) as plain Go structs; Watermill handles serialization, topic selection, and pub/sub integration (Kafka, RabbitMQ, Postgres, JetStream, etc.). Downstream read models subscribe via `EventProcessor` handlers. See [472 §3.3.1](472-ameide-information-application.md#331-event-driven-cqrs-runtime-watermill) for details and references. Other languages follow the same CQRS semantics even when they don’t use Watermill directly.

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


<!-- ═══════════════════════════════════════════════════════════════════════════ -->
<!-- FILE: 470a-ameide-vision-vs-d365.md -->
<!-- ═══════════════════════════════════════════════════════════════════════════ -->

# 470.A – Ameide Code‑First / AI‑First ERP vs. D365 AOT Metadata‑Driven ERP

**Status:** Draft  
**Owner:** Architecture / Product  
**Companion:** [470-ameide-vision.md](470-ameide-vision.md)

## 1. Purpose

This document defines the objectives of the **Ameide architecture** relative to the **Dynamics 365 Finance & Operations (D365FO) metadata / AOT–driven model**.

We explicitly **do not** aim to recreate the D365 AOT stack in Kubernetes. Instead, we:

* Keep some *systemic* benefits AOT gives (coherence, discoverability, extensibility),
* While **intentionally replacing** the metadata‑first paradigm with a **code‑first, AI‑operated design**.

---

## 2. Two paradigms in one sentence each

* **D365 AOT model:**
  The application is a **metadata tree** (AOT) of tables, EDTs, enums, forms, menu items, security, etc. – a compiler turns that into runtime artifacts. Humans primarily edit metadata in Visual Studio.

* **Ameide model:**
  The application is **code (Go / TS / Temporal) plus proto contracts**, living in a **single codebase**, with **Domain / Process / Agent / UISurface** as operational primitives (CRDs). An **AI agent**, not humans in a designer, is the main developer.

---

## 3. Ameide Approach – High‑Level Objectives

1. **Code‑first, AI‑friendly representation**

   * Business logic and structure are primarily expressed as **normal code** and **protos**, not heavyweight metadata trees.
   * We optimize for **LLMs understanding and editing code** over humans dragging boxes in a designer.

2. **Thin, operational metadata layer**

   * **CRDs** exist only for *infra‑relevant* units:

     * `Domain`
     * `Process`
     * `Agent`
     * `UISurface`
   * We do **not** create CRDs for low‑level concepts like EDTs, fields, or individual forms.

3. **Central, coherent system graph derived from code**

   * A **single repository** + proto descriptors + CRDs form a **global model** of the system.
   * Tools and AI can ask: “what domains exist?”, “which processes touch SalesOrder?”, “which agents can issue command X?”

4. **Extension model is code, not metadata**

   * ISV / partner extensions are **modules of code** (plus a thin manifest), developed by **chatting with AI**.
   * We do **not** ask partners to understand or modify a giant metadata tree; they focus on code and well‑defined extension points.

5. **Preserve ERP‑class coherence without AOT overhead**

   * We still care about:

     * cross‑reference analysis,
     * upgrade safety,
     * security and data classification,
     * discoverable APIs and entities,
   * but we achieve it with **code analysis + proto + minimal manifests**, not with a dedicated AOT storage and designer.

---

## 4. D365 AOT vs Ameide – Objective‑Level Comparison

### 4.1 Source of truth

**D365 AOT**

* **Metadata‑first.** AOT is the canonical definition of tables, forms, security, labels, etc.
* Build pipeline compiles AOT/X++ into IL and packages.
* If it’s not in AOT, it’s not really part of the app.

**Ameide**

* **Code‑first, metadata‑guided.**
* The canonical definition of behavior is **Go / TS / Temporal code + proto contracts**.
* CRDs and design artifacts **operate** this code (configure, deploy, orchestrate), they don’t replace it.
* We still maintain a **derived “system graph”** for tooling, but that graph is *computed from* code/protos/CRDs instead of being hand‑maintained metadata.

**Objective:**
Maximize expressiveness and AI‑operability by treating **code as the primary model**, and metadata as a thin operational and analytical layer.

---

### 4.2 Object model granularity

**D365 AOT**

* Very fine‑grained application object model:

  * Tables, fields, relations, EDTs, enums, queries, views, forms, menu items, workspaces, security roles/duties/privileges, label files, etc.
* All of these are first‑class metadata objects.

**Ameide**

* **Infra‑level model is intentionally coarse:**

  * `Domain` (bounded context with data + rules)
  * `Process` (orchestration / saga)
  * `Agent` (AI worker)
  * `UISurface` (UI entry point)
* Lower‑level concepts (e.g. “EDT”, “form”, “query”) are:

  * **code**, patterns, and conventions,
  * optionally described via **proto options**, type wrappers, or lightweight manifests,
  * but **not** CRDs and not infra objects.

**Objective:**
Keep infra simple and composable; avoid running “infrastructure for an EDT”. Treat low‑level semantics as **code patterns** AI can understand, not as separate runtime entities.

---

### 4.3 Development model

**D365 AOT**

* Designed around **human developers**:

  * Application Explorer, designers for forms, security, and data entities.
  * Metadata exists partly to provide rich design‑time UX.

**Ameide**

* Designed around **AI as the primary developer**:

  * AI navigates **code and proto**, not heavyweight designers.
  * Humans interact mostly through:

    * conceptual docs and requirements,
    * reviewing diffs,
    * guiding the AI.

**Objective:**
Treat “metadata‑for‑Visual Studio” as unnecessary overhead. Instead, shape the system around what LLMs understand best today: **code + simple, structured metadata**.

---

### 4.4 Extensibility & ISV model

**D365 AOT**

* Extensions are mainly **metadata extensions**:

  * extension classes, event handlers, extensible enums, etc.
* ISVs work in the same AOT tree, within a constrained extensibility model.

**Ameide**

* Extensions are **code modules + manifests**, developed by chatting with AI:

  * Add new Go packages / Temporal workflows / UISurfaces.
  * Register against explicit extension points (interfaces, events, hooks).
* A **small manifest** (not a whole AOT layer) declares:

  * module name, version, dependencies,
  * which Domains/Processes/UISurfaces it touches.

**Objective:**
Make extensibility **AI‑operated and code‑centric**, but still track dependency and impact information via a thin manifest so we can reason about upgrades and conflicts.

---

### 4.5 Runtime representation & introspection

**D365 AOT**

* Model store is used both at design‑time and runtime.
* Rich introspection: tools can walk the tree of objects, search, do cross‑reference analysis, generate security and impact reports.

**Ameide**

* Runtime consists of:

  * **Pods/Deployments** (Domains/Processes/Agents/UISurfaces),
  * **Databases** (Postgres),
  * **Workflows** (Temporal), etc.
* Introspection is built by combining:

  * **proto descriptors**,
  * **code index** (AST / symbols),
  * **CRDs** and module manifests.

**Objective:**
Provide **equivalent or better "what's in the system?" answers** via the **read-only Knowledge Graph** (projecting from code+proto+CRDs), without forcing all structure through a dedicated AOT store.

---

## 5. What We Intentionally *Don’t* Replicate from D365 AOT

1. **Full metadata‑first object tree**

   * No attempt to replicate every AOT concept (EDT, Form, Query, LabelFile, etc.) as first‑class metadata entities.

2. **Visual designers**

   * No heavy VS‑style designers; we rely on:

     * conventional patterns in React/Next.js,
     * code and schema the AI can operate on.

3. **Metadata‑driven UI layout**

   * We don’t have a general “form engine” that renders UI purely from metadata.
   * We accept **hand/AI‑written UISurfaces** as code, with optional hints in metadata.

4. **Model store as only source of truth**

   * We do not enforce that *everything* goes through a metadata store before existing.
   * Code is allowed to be primary, as long as it remains analyzable and mapped into the system graph.

---

## 6. What We *Do* Preserve in Spirit

Even with the differences above, we aim to preserve several **AOT benefits** in a new form:

1. **Coherent system graph (read-only projection)**

   * One place (Knowledge Graph) to ask:

     * "Which domains exist?"
     * "Which processes depend on Domain X?"
     * "Which modules extend Order lifecycle?"
   * Implemented via analysis of **code + proto + CRDs + manifests**; Graph is never a write surface.

2. **Upgrade & impact analysis**

   * When changing a proto or Domain, we want to know:

     * which modules and UISurfaces are impacted,
     * which extension points are affected.
   * Achieved with static analysis and module manifests rather than metadata trees.

3. **Security and data classification visibility**

   * Even though security and data semantics are mainly expressed in code/proto,
   * We standardize annotations (e.g. proto options, tags) so tools can:

     * find PII,
     * list privileged operations,
     * answer “who can do what where?” questions.

4. **Extensibility discipline**

   * Even if extensions are code, we define:

     * explicit, stable extension points,
     * linting/validation that enforces "don't patch core internals directly".

### 6.5 Security Mapping: D365FO to Ameide

Ameide provides an ERP-class security metamodel that maps directly to D365FO concepts. For full details, see [476-ameide-security-trust.md §7](476-ameide-security-trust.md).

| D365FO Concept | Ameide Concept | Implementation |
|----------------|----------------|----------------|
| **Security Role** | `OrganizationRole` + Duty refs | Keycloak realm role with duties in `permissions` map |
| **Duty** | `Duty` | Aggregate of AuthorizationDescriptors stored in Transformation Domain |
| **Privilege** | `AuthorizationDescriptor` | Proto method + required dimensions + scope + risk tier |
| **Menu Item / Form** | UISurface route + action | Next.js route protected via `withAuth()` middleware |
| **Security Policy (XDS)** | `AuthorizationDimension` + RLS | Dimension config generates Postgres RLS policies |
| **Data Classification** | `(pii)`, `(encrypted)` proto options | Field-level annotations in `common/v1/annotations.proto` |
| **Segregation of Duties** | `SoDRule` | Conflicting descriptors checked at role assignment time |

**Key differences from D365:**

1. **Code-first definitions** – Ameide uses proto options and code conventions, not AOT metadata trees. Security artifacts are design-time artifacts in the Transformation Domain.

2. **Explicit dimension columns** – D365 XDS policies apply virtual filters at runtime. Ameide dimensions are explicit columns (`company_code`, `plant_id`) on domain tables with RLS enforced at the database layer.

3. **Real-time SoD enforcement** – D365 SoD is typically evaluated in batch via GRC reporting. Ameide evaluates SoD rules at role assignment time, blocking conflicts before they occur.

4. **Tenant isolation as hard invariant** – D365 relies on legal entity configuration. Ameide enforces `tenant_id` + `organization_id` via RLS + JWT claims as non-negotiable guardrails.

5. **Agent governance** – D365 has no AI agent concept. Ameide adds `scope` and `risk_tier` to AgentDefinitions, ensuring AI workers operate within explicit capability bounds.

**Example mapping:**

| D365 Concept | Ameide Equivalent |
|--------------|-------------------|
| `SalesOrderMaintainPurchOrder` (privilege) | `PURCHASE_ORDER_MAINTAIN` AuthorizationDescriptor |
| `Maintain purchase orders` (duty) | `DUTY_PURCHASE_ORDER_MAINTAIN` duty |
| `Purchasing agent` (role) | `purchasing-agent` OrganizationRole |
| `CustInvoiceJour` XDS policy | `company_code` AuthorizationDimension |
| `SoD rule: create vs approve` | `SOD_PO_001` SoDRule |

---

## 7. Non‑Goals

To avoid confusion, these are **explicit non‑goals** of the Ameide approach:

* Recreating the **D365 AOT designers** or development UX.
* Providing **metadata for every low‑level object** (EDT, form control, etc.).
* Forcing partners to learn a bespoke metadata language or tree.
* Making design‑time metadata the single, authoritative source of truth for everything.

Our focus is:

> **ERP‑class capabilities, with AI‑operable code as the core, and just enough metadata to keep the system coherent and introspectable.**

---

## 7.5 Execution Snapshot (Dec 2025)

- **Runtime proof:** `extensions-runtime` (Backlog 480) now operates as the Tier 1 plugin host, exposing a single proto-first `InvokeExtension` API and executing tenant WASM modules behind Ameide-owned host adapters. This is the tangible replacement for D365’s Metadata AOT “class extensions”—it keeps Domain/Process/Agent primitives deterministic while letting partners ship helper logic in code.
- **Deployment discipline:** The GitOps layout from Backlog 364 is live: ApplicationSets roll out namespaces → CRDs → operators → apps via RollingSync waves, and Helmfiles are render-only. That mirrors the “design vs deploy vs runtime” separation this comparison leans on, avoiding the AOT-style mingling of metadata, build, and runtime.
- **Tooling coherence:** SDKs, proto packages, and integration packs now lint on a shared `SERVICEPREFIX_FIELD` env-var convention (Backlog 430) and reuse the same `runtime.proto` definitions. This ensures the “single source of truth” graph is code+proto, not metadata trees, and the AI/dev loop stays consistent across primitives and UIs.
- **Tight tenancy rules:** Even though Tier 1 extensions run in `ameide-{env}`, tenancy + auth are enforced by ExecutionContext tokens issued by primitives, satisfying §3’s “Agentic but controlled” principle and contrasting with D365’s shared metadata store that often needed manual guardrails.

These concrete deliverables differentiate Ameide from D365 not just philosophically but operationally: we now have shipping services, GitOps automation, and SDK contracts that embody the code-first, AI-first stance outlined above.

### 7.6 Proto-first contract chain (Buf vs AOT)

D365’s metadata tree is a runtime source of truth; Ameide keeps the contract in code via Buf:

1. `packages/ameide_core_proto` houses the Buf module for every Domain/Process/Agent (see [365](365-buf-sdks-v2.md)).
2. Generated stubs flow into the TS/Go/Python SDK packages; services consume only those workspace SDKs per [402](402-buf-first-generated-clients.md), [403](403-python-buf-first-migration-plan.md), and [404](404-go-buf-first-migration.md).
3. GitOps manifests declare which image (and therefore proto version) runs where, so the chain **proto → implementation → runtime** is explicit and CI-enforced (documented in [472 §2.7](472-ameide-information-application.md)).
4. SDK publishes (per [388](388-ameide-sdks-north-star.md)) are for external consumers; internal loops never wait on registry updates.

This gives us the coherence people expect from AOT while keeping Buf-first contracts and GitOps-controlled runtime state.

### 7.7 Proto Semantic Layers → D365 Concept Mapping

Proto carries three semantics in Ameide (see [472 §2.8](472-ameide-information-application.md) for full details). Here's how they map to D365 concepts:

| Ameide Proto Semantic | D365 AOT Equivalent | Notes |
|-----------------------|---------------------|-------|
| **Entity** (`message` in domain pkg) | Tables, Data Entities | Business objects; we use `AGGREGATE` vs `PROJECTION` tags instead of D365's table vs data entity split |
| **Projection Entity** (`PROJECTION`) | Data Entities, OData entities | Read-optimized views for integration; like D365's consumption-layer entities |
| **Interface** (`service`, `DOMAIN`) | X++ Service classes | Internal domain operations; what UIs and agents call |
| **Interface** (`service`, `INTEGRATION`) | OData endpoints, Data Entities (CRUD) | Stable external contracts; like D365's released integration surfaces |
| **Event** (`BUSINESS`) | Business Events | Facts for external subscribers; stable integration contracts |
| **Event** (`DOMAIN_INTERNAL`) | Internal telemetry / class events | Implementation-detail events; not for external consumption |

**Key differences:**

1. **No EDT/field metadata** – D365 has EDTs, base enums, and field-level metadata. Ameide keeps these in proto field definitions + validation in code.

2. **No form metadata** – D365 Forms are AOT objects. Ameide UISurfaces are code-first React/Next.js apps consuming proto-generated types.

3. **Single contract surface** – D365 has separate OData/$metadata, X++ contracts, and DMF packages. Ameide unifies on proto → SDK for all consumers.

**Example mapping:**

| D365 Concept | Ameide Equivalent |
|--------------|-------------------|
| `CustTable` (table) | `Customer` message in `ameide.sales.v1` |
| `CustTableEntity` (data entity) | `CustomerProjection` message with `PROJECTION` tag |
| `SalesOrderService` (X++ service) | `SalesDomainService` proto service |
| `SalesOrderCreated` (business event) | `SalesOrderCreated` message in `ameide.sales.events.v1` |

---

## 8. Summary

Compared to D365’s AOT‑driven, metadata‑first ERP development:

* Ameide is **code‑first, AI‑first**.
* We keep only a **thin operational metadata layer** (CRDs for Domains/Processes/Agents/UISurfaces + small manifests).
* We deliberately move concepts like EDTs, forms, and extensions **into code and proto**, where AI can work most effectively.
* We rebuild the **AOT benefits** (coherence, introspection, extensibility discipline) via:

  * a central repo,
  * proto descriptors,
  * CRDs,
  * and analyzers over that combined graph.

This gives us an architecture that respects what made AOT powerful, while aligning with a future where **AI, not Visual Studio, is the primary development environment.**

---

## 9. Follow-Up Backlog Updates

- **478-ameide-extensions.md** – Needs an explicit pointer to the shipping `extensions-runtime` Tier 1 host (Backlog 480) so the extension doc references a concrete runtime rather than abstract hooks.
- **473-ameide-technology.md / 364-argo-configuration-v5-decomposition.md** – Update the technology/GitOps docs to highlight that ApplicationSet RollingSync sequencing is now live, replacing the legacy Helmfile orchestration this comparison critiques.
- **474-ameide-implementation.md** – Should outline how we bootstrap the “code-first vs metadata-first” posture in customer projects (Backstage templates, proto-first SDKs, env-var lint) so field teams have a prescriptive rollout tied to the differences documented here.


<!-- ═══════════════════════════════════════════════════════════════════════════ -->
<!-- FILE: 470b-ameide-vision-vs-saps4.md -->
<!-- ═══════════════════════════════════════════════════════════════════════════ -->

Here’s a version of the same “vision” but framed explicitly **vs SAP S/4HANA** instead of D365.

---

# 470.B – Ameide Code-First / AI-First ERP vs. SAP S/4HANA Metadata- & Config-Driven ERP

**Status:** Draft  
**Owner:** Architecture / Product  
**Companion:** [470-ameide-vision.md](470-ameide-vision.md)

## 1. Purpose

This document describes the objectives of the **Ameide architecture** relative to **SAP S/4HANA’s metadata- and configuration-driven model** (ABAP Data Dictionary, CDS, Fiori annotations, Customizing/IMG, extensibility frameworks).

We are **not** trying to rebuild S/4HANA’s DDIC + CDS + Customizing stack inside Kubernetes. Instead we want:

* ERP-class capabilities and coherence,
* but with a **code-first, AI-operated** core,
* and a **thin, infra-oriented metadata layer**, not a huge metadata/configuration universe.

---

## 2. Two paradigms in one sentence each

**SAP S/4HANA:**

> A mix of **metadata repositories and configuration frameworks** (ABAP Data Dictionary, CDS views with annotations, Fiori elements, Customizing/IMG, key-user and developer extensibility) plus ABAP code; humans primarily adapt the system via configuration and metadata, and extend it via ABAP and BAdIs.

**Ameide:**

> A **code-first, proto-centric system** where business logic lives in Go/Temporal/TS in a single repo, with **Domain / Process / Agent / UISurface** as coarse-grained runtime primitives modeled as Kubernetes CRDs; an **AI agent** is the primary “developer” that edits code and specs.

---

## 3. Ameide Approach – High-Level Objectives (recap)

1. **Code-first, AI-friendly representation**

   * Business logic and structure are primarily **code + proto contracts**, not dense metadata/config layers.
   * We optimize for **LLMs navigating and editing code**, not humans working in SE11/SE80 or Fiori config apps.

2. **Thin operational metadata layer (CRDs only where infra matters)**

   * We introduce CRDs only for **infra-relevant** units:

     * `Domain` – bounded context with data + rules
     * `Process` – long-running orchestration
     * `Agent` – AI worker
     * `UISurface` – UI shell / entry point

   * No CRDs for “table”, “field”, “form”, or “UI annotation”.

3. **Central, coherent system graph derived from code**

   * A **single codebase + proto descriptors + CRDs** is the system’s “model store”.
   * Tools (and AI) query this graph to understand domains, processes, agents, surfaces, and their relationships.

4. **Extensions are code modules, not configuration trees**

   * Partners ship **code + a small manifest**, not IMG projects, Customizing variants, or DDIC artifacts.
   * They work by **chatting with AI**, which edits code and specs against well-defined extension points.

5. **Preserve ERP-class coherence, without replicating S/4’s config/meta bulk**

   * We still care about:

     * cross-reference and impact analysis,
     * upgrade-safe extensions (“clean core” in spirit),
     * security + data classification,
   * but we achieve it by analyzing **code+proto+CRDs**, not via a huge Customizing layer or DDIC.

---

## 4. Quick overview: S/4HANA’s metadata & config stack

To clarify what we’re deliberately *not* recreating, here’s a rough picture of the S/4 world.

### 4.1 ABAP Data Dictionary (DDIC)

* Central repository of metadata: tables, views, structures, **domains, data elements, search helps**, etc.
* Defines data types, lengths, check tables, foreign-key relations, technical settings.
* All ABAP programs use the DDIC to interact with the DB; it enforces global integrity.

### 4.2 CDS views, annotations, and Fiori Elements

* **Core Data Services (CDS)** views define semantically rich data models (associations, aggregations, etc.).
* **UI annotations** on CDS/OData specify:

  * which fields appear in lists/forms,
  * UI facets, value helps, filters, semantics.
* SAP Fiori Elements uses these annotations to **auto-generate UI**.

S/4 uses CDS+annotations as a **metadata-driven UI and API layer** on top of DDIC.

### 4.3 Customizing / IMG / Business Configuration

* Huge configuration tree via **Implementation Guide (IMG/SPRO)** for finance, logistics, etc.
* Customizing data stored in control tables; grouped into BC Sets, transported between systems.
* This configuration drives system behaviour without changing code.

### 4.4 Extensibility frameworks

* **In-app / key-user extensibility**:

  * Custom fields & logic, custom CDS views, custom business objects, etc. via Fiori apps.
* **Developer extensibility / ABAP**:

  * BAdIs, enhancement framework, side-by-side or “clean core”-compliant ABAP code.

So S/4 is *heavily* metadata/config driven, with ABAP as the programmable escape valve.

---

## 5. Ameide vs S/4HANA – Objective-Level Comparison

### 5.1 Source of truth: metadata/config vs code + proto

**S/4HANA**

* System behaviour emerges from:

  * DDIC (tables, domains, data elements, views),
  * CDS views and annotations (OData, UI semantics),
  * Customizing (IMG, configuration tables),
  * plus ABAP code and BAdI implementations.

* There are multiple overlapping repositories of truth:

  * Data model in DDIC,
  * UI semantics in CDS annotations,
  * behaviour in ABAP,
  * configuration in IMG tables.

**Ameide**

* **Code + proto are the primary source of truth**:

  * Proto packages define entities, commands, queries, events.
  * Go/Temporal/TS implement behaviour.
* CRDs (`Domain`, `Process`, `Agent`, `UISurface`) describe **how to run** that code in the cluster, not business semantics.
* Business configuration is expressed mainly as:

  * structured config objects in code or proto,
  * not as a separate, massive Customizing tree.

**Objective:**
Avoid multiple divergent metadata/config layers. Keep a *single* modelable graph based on **code + proto + a thin set of CRDs**, so AI and tools can reason about the whole system without reconciling DDIC+CDS+IMG.

---

### 5.2 Object model granularity

**S/4HANA**

* Very fine-grained metadata in DDIC:

  * domains, data elements, tables, fields, search helps, views.
* CDS layer adds a second modeling space (entities, associations, annotations).
* Fiori, analytics, and extensibility layers reference these meta-objects.

**Ameide**

* Infra-level object model is **intentionally coarse**:

  * A `Domain` might own dozens of entities, but infra doesn’t know them individually.
* “EDT-like” concepts, forms, and queries live as:

  * code patterns and proto message types,
  * optional annotations/options in proto,
  * not as separate runtime objects.

**Objective:**
Keep infra simple; avoid an explosion of runtime metadata entities. Low-level semantics are **code patterns** (which AI finds natural) rather than first-class runtime objects.

---

### 5.3 Development model: key-user / config vs AI developer

**S/4HANA**

* Heavy emphasis on **configuration and key-user tools**:

  * Custom Fields & Logic, Custom CDS Views, Custom Business Objects, etc.
* ABAP developers extend via released APIs and BAdIs, often guided by “clean core” principles.
* Tools are designed for **humans** to click/configure in Fiori or SE11/SE80.

**Ameide**

* Designed for **AI as the primary developer**:

  * AI reads/writes Go/Temporal/TS and proto,
  * manipulates Domain/Process/Agent/UISurface CRDs.
* Humans primarily:

  * express requirements,
  * review diffs,
  * approve changes / rollouts.

**Objective:**
Treat S/4’s rich key-user tooling as human-heavy and not needed in this context. Instead, use **plain code + simple metadata** as the substrate for LLM-driven development and refactoring.

---

### 5.4 UI & semantics: CDS + Fiori elements vs UISurfaces as code

**S/4HANA**

* CDS views with UI annotations define **UI-relevant metadata directly in the data model**; Fiori Elements generates list reports, object pages, etc. automatically.

**Ameide**

* UISurfaces are **Next.js apps / micro-portals**:

  * They consume typed SDKs based on proto,
  * but we don’t have a full metadata-driven form engine like Fiori Elements.
* Optional: we may add “form/board/workspace patterns” in code, but still code-centric.

**Objective:**
Rely less on metadata-driven UI generation and more on **conventional, AI-crafted React**. Keep UI semantics as close to standard web development as possible so both AI and web developers are productive.

---

### 5.5 Extensibility & “clean core”

**S/4HANA**

* Extensibility strategy:

  * Key-user in-app extensibility (fields, logic, CDS, BOs),
  * developer extensibility via BAdIs, ABAP on S/4HANA, and side-by-side BTP apps,
  * all under a **clean core** philosophy (stay within released interfaces, avoid core modifications).

**Ameide**

* Extensibility = **code modules + manifest**:

  * A module declares:

    * which Domains/Processes/UISurfaces it extends,
    * which extension points it hooks into.
  * AI helps partners write those modules.

* “Clean core” is enforced by:

  * explicit extension interfaces,
  * linting and static analysis (no imports into forbidden packages),
  * impact analysis based on the module manifest and code graph.

**Objective:**
Keep the **spirit** of S/4’s clean core (upgrade-safe, interface-based extensions) while making the mechanics code-centric and AI-operated rather than config/metadata-heavy.

---

### 5.6 Business configuration

**S/4HANA**

* Business behaviour is heavily influenced by **Customizing / IMG**:

  * chart of accounts, document types, number ranges, posting rules, etc. managed via transactions & Fiori apps, stored in config tables and BC Sets.

**Ameide**

* Business configuration is:

  * structured **config data** owned by Domains (e.g., config tables in Postgres, feature flags),
  * exposed via typed proto messages and admin UISurfaces,
  * versioned and moved via normal app mechanisms (migrations, seed data, domain services),
  * not a separate IMG universe.

**Objective:**
Avoid a separate “config world” with its own UI, transport, and logic. Treat configuration as part of the **domain model** and code, which AI can also understand and evolve.

---

### 5.7 Runtime introspection

**S/4HANA**

* You can introspect:

  * DDIC (tables, elements, domains),
  * CDS views & annotations,
  * Customizing tables,
  * registered BAdIs and implementations.

* SAP tooling (SE11/SE80, Fiori apps, Extensibility Explorer, etc.) gives a rich view of metadata and extensibility options.

**Ameide**

* We provide introspection via the **read-only Knowledge Graph** that:

  * projects from proto descriptors,
  * indexes Go/Temporal/TS code,
  * reads Domain/Process/Agent/UISurface CRDs and module manifests.

* Queries like:

  * "Show me all processes involving Invoices."
  * "Which extensions hook into Domain 'Sales'?"
  * "Where is PII stored?"

  are answered by querying this graph projection, not by traversing DDIC/IMG/CDS. **Graph is never a write surface; all mutations go through Domain primitives.**

**Objective:**
Provide S/4-like or better "what's in my system?" answers, but over **modern code + proto + k8s resources** rather than multiple legacy metadata/config repositories.

---

## 6. Non-Goals vs S/4HANA

Ameide explicitly does **not** aim to:

* Recreate the **ABAP Data Dictionary** as a separate metadata system.
* Provide a **CDS + Fiori Elements** style metadata-driven UI generator.
* Build an **IMG / Customizing** universe with BC Sets and dedicated config transports.
* Replicate key-user tools for non-technical users to mold the system.

Instead:

* We assume **developers + AI** as the primary shaping forces.
* We use standard web & cloud-native primitives (Go, TS, Kubernetes, Postgres, Temporal).
* We keep metadata **thin and infra-oriented**, not a parallel modeling universe.

---

## 6.5 Execution Snapshot (Dec 2025)

- **Code-first extensibility proved out.** The `extensions-runtime` service (Backlog 480) now executes tenant-provided WASM helpers with Wasmtime sandboxes, host-call adapters built on Ameide SDKs, and MinIO-backed module distribution. This is the Ameide alternative to S/4’s Customizing + key-user extensibility layers: partners ship compiled modules and manifests, not IMG transports or DDIC artifacts.
- **GitOps + waves replace transport landscapes.** ApplicationSet-based deployments (Backlog 364) roll namespaces, CRDs, operators, and services through RollingSync waves, guarded by health checks. That gives us the equivalent of S/4’s DEV→QAS→PRD pipelines but grounded in declarative GitOps rather than transport requests and manual approvals.
- **Unified modeling surface.** Proto descriptors + CRDs + code scanning feed a central graph, and integration packs enforce naming/contract linting (Backlog 430). There is no split between DDIC/CDS/UI annotations/config tables; everything the AI and operators need lives in the repo and is validated together.
- **Tenancy + clean core by construction.** Risk-tiered execution contexts ensure extensions use existing Domain/Process APIs with primitive-issued tokens—no “t-code” style ad-hoc DB access. This keeps the clean-core promise S/4 emphasizes, but via mechanically enforced SDKs instead of policy guidelines.

These shipping capabilities show the Ameide vision is already manifesting in code, not just in architectural slides: extensions, deployments, and introspection now operate exactly as the code-first, AI-operated model prescribes.

### 6.6 Proto-first contract chain (Buf vs DDIC/CDS)

S/4 splits behavior across DDIC/CDS/UI annotations/config. Ameide tracks everything via Buf and GitOps:

1. `packages/ameide_core_proto` is the single Buf module; Buf breaking checks run in CI ([365](365-buf-sdks-v2.md)).
2. Workspace SDKs ingest those stubs and every service uses AmeideClient per [402](402-buf-first-generated-clients.md), [403](403-python-buf-first-migration-plan.md), [404](404-go-buf-first-migration.md).
3. GitOps manifests record which images (and proto versions) run in each environment, as formalized in [472 §2.7](472-ameide-information-application.md).
4. SDK publishing (outer loop) is governed by [388](388-ameide-sdks-north-star.md) for external consumers; internal rings never depend on those registries.

So instead of DDIC/CDS being the runtime schema, Buf is the contract, SDKs implement it, and Argo CD ensures the exact versions are live—while staying code-first.

### 6.7 Proto Semantic Layers → SAP Concept Mapping

Proto carries three semantics in Ameide (see [472 §2.8](472-ameide-information-application.md) for full details). Here's how they map to SAP S/4HANA concepts:

| Ameide Proto Semantic | SAP S/4 Equivalent | Notes |
|-----------------------|-------------------|-------|
| **Entity** (`message` in domain pkg) | DDIC tables, CDS entities | Business objects; we use `AGGREGATE` vs `PROJECTION` tags instead of DDIC table vs CDS view split |
| **Projection Entity** (`PROJECTION`) | CDS consumption views, OData entities | Read-optimized views for Fiori/integration; like S/4's "C_*" consumption views |
| **Interface** (`service`, `DOMAIN`) | BAPIs, ABAP OO methods | Internal domain operations; what UIs and agents call |
| **Interface** (`service`, `INTEGRATION`) | Released OData services, A2X services | Stable external contracts; like S/4's released integration APIs |
| **Event** (`BUSINESS`) | SAP Business Events, Event Mesh messages | Facts for external subscribers; stable integration contracts |
| **Event** (`DOMAIN_INTERNAL`) | Internal ABAP events, class events | Implementation-detail events; not for external consumption |

**Key differences:**

1. **No DDIC metadata layer** – S/4 has domains, data elements, structures, search helps. Ameide keeps these in proto field definitions + validation in code.

2. **No CDS annotations for UI** – S/4 uses `@UI.*` annotations in CDS for Fiori Elements. Ameide UISurfaces are code-first React/Next.js apps.

3. **No IDoc/message types** – S/4 has IDocs for B2B integration. Ameide uses proto events with `BUSINESS` tag for the same purpose.

4. **Single contract surface** – S/4 has DDIC, CDS, BAPI, OData, IDoc as separate layers. Ameide unifies on proto → SDK.

**Example mapping:**

| SAP Concept | Ameide Equivalent |
|-------------|-------------------|
| `VBAK` (DDIC table) | `SalesOrder` message in `ameide.sales.v1` |
| `C_SalesOrderTP` (CDS consumption view) | `SalesOrderListItem` message with `PROJECTION` tag |
| `I_SalesOrder` (CDS interface view) | Part of `SalesOrder` aggregate |
| `BAPI_SALESORDER_CREATEFROMDAT2` | `CreateSalesOrder` rpc in `SalesDomainService` |
| `A_SalesOrder` (released OData) | `SalesIntegrationService` proto service |
| `SalesOrder.Created.v1` (business event) | `SalesOrderCreated` message in `ameide.sales.events.v1` |
| `ORDERS05` (IDoc) | `SalesOrderCreated` with `BUSINESS` event kind |

---

## 7. Summary

Compared to **SAP S/4HANA’s metadata- and configuration-driven architecture**, Ameide:

* Chooses **code-first, AI-first** over config-first and key-user tooling.
* Replaces DDIC + CDS + IMG + BAdI frameworks with:

  * a **single codebase**,
  * **proto contracts**,
  * **Domain/Process/Agent/UISurface CRDs**,
  * and **extension modules + manifests**.
* Keeps the *goals* of ERP metadata systems (coherence, extensibility, clean core, introspection), but:

  * implements them via **code analysis, proto descriptors, and k8s primitives**,
  * in a shape that’s friendlier for LLMs and cloud-native operations.

In short:
**Ameide is what you get if you take the core *intent* of S/4HANA (structured ERP, clean core, extensible) but redesign it under the assumption that “the main developer is an AI that loves code, not a human that loves configuration screens.”**

---

## 8. Follow-Up Backlog Updates

- **478-ameide-extensions.md** – Document the Tier 1/Tier 2 split explicitly (Transformation-owned ExtensionDefinitions promoted into the `extensions-runtime` host) so the extensions backlog reflects the concrete runtime now in place.
- **473-ameide-technology.md** – Capture the ApplicationSet/RollingSync orchestration (364) and Telepresence overlays so the technology doc mirrors the GitOps/transport story used for this comparison.
- **474-ameide-implementation.md** – Needs a narrative showing how we stand up Ameide in greenfield customers: proto scaffolding, AI-driven development loops, and GitOps promotion. Without it, field teams still fall back to S/4-style “blueprints + customizing”, which defeats the purpose of this document.


<!-- ═══════════════════════════════════════════════════════════════════════════ -->
<!-- FILE: 471-ameide-business-architecture.md -->
<!-- ═══════════════════════════════════════════════════════════════════════════ -->

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
> **Core Invariants**: See [470-ameide-vision.md §0 "Ameide Core Invariants"](470-ameide-vision.md) for the canonical list (four primitives, Graph read-only, Transformation as domain, proto chain, tenant isolation, Backstage internal).
>
> **Deployment Implementation**:
> - [465-applicationset-architecture.md](465-applicationset-architecture.md) – Per-environment deployments
> - [443-tenancy-models.md](443-tenancy-models.md) – Multi-tenancy SKUs
> - [467-backstage.md](467-backstage.md) – Backstage implementation (§4 platform factory)

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

Runtime Domain/Process/Agent/UISurface primitives are represented as Ameide custom resources (Domain/Process/Agent/UISurface CRDs) that encode the desired state for each primitive. Operators reconcile those CRDs into Deployments, Services, Temporal workers, and supporting infrastructure; GitOps applies only the CR YAML.

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

All primitives follow the proto-first chain described in [472 §2.7](472-ameide-information-application.md#27-contract--implementation--runtime-chain): Buf modules in `packages/ameide_core_proto` define the contracts, workspace SDKs (TS/Go/Py) consume them, and GitOps manifests record which image/proto versions run in each environment. Go-based Process/Domain primitives also standardize on Watermill’s CQRS component for command/event routing (see [472 §3.3.1](472-ameide-information-application.md#331-event-driven-cqrs-runtime-watermill)). Business flows therefore assume commands mutate write models, events update read models, and read-only queries hit the projections maintained by those handlers.

---

## 3. Transformation as a first‑class domain

### 3.1 Purpose of the Transformation domain

Instead of treating "implementation" as something done off‑platform, **Transformation** is itself a Domain primitive:

* Entities:

  * **Initiatives, Epics, Work Packages** – high‑level change containers.
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
* ArchiMate models of application/business/technology architecture.
* Markdown specs / decision records.

All artifacts are **stored by the Transformation Domain** (optionally event-sourced internally):

* **Event‑sourced pattern** – commands → revisions → snapshots → promotions (within Transformation Domain).
* **Multi‑tenant** – partitioned by tenant/org.
* **Governed** – promotion paths and validations (e.g. secret scan, BPMN lint) are central to Transformation.

From a business‑architecture perspective:

> "The **Transformation Domain** is the single source of truth for all design-time artifacts. **Transformation design tooling** is the set of modelling UIs (BPMN editor, diagram editor, Markdown editor) that call the Transformation Domain APIs—Transformation design tooling does not own storage. Process primitives and Agent primitives execute definitions from the Transformation Domain; they do not own the process/agent logic themselves."

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

Backstage isn't just an engineer portal; in this architecture it's the **factory for business capabilities**.

From a business angle:

* Each **Backstage template** corresponds to a *type of capability*:

  * `Domain primitive template` – create a new DDD domain with proto APIs.
  * `Process primitive template` – create a runtime that executes ProcessDefinitions from Transformation design tooling.
  * `Agent primitive template` – create a runtime that executes AgentDefinitions from Transformation design tooling.
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
* **Process primitive = runtime execution**: Compiled from ProcessDefinitions stored in the Transformation Domain into executable Temporal workflows.
* **Graph is a read projection**: Operational data lives in domains; the graph receives projections for analysis, transformation, and agent context.

> **Important**: Transformation remains the source of truth for transformation artifacts; the graph stores references and read-optimised views, not the authoritative records.

This means: Transformation domain entities (initiatives, backlogs, artifacts) are projected into the graph as Elements, enabling cross‑methodology reporting and agent‑assisted design—but the canonical records remain in the Transformation Domain.

---

## 11. Terminology

| Legacy Term | New Term (471) |
|-------------|----------------|
| IPA (Intelligent Process Automation) | Domain primitive + Process primitive + Agent primitive bundle |
| Platform Workflows (305) | Process primitive (executes ProcessDefinitions) |
| AgentRuntime (310) | Agent primitive (executes AgentDefinitions) |
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


<!-- ═══════════════════════════════════════════════════════════════════════════ -->
<!-- FILE: 472-ameide-information-application.md -->
<!-- ═══════════════════════════════════════════════════════════════════════════ -->

Here's a first cut of the **Application & Information Architecture** doc, aligned with the vision + business docs we already sketched and wired to vendor concepts (Backstage, BPMN-compliant definitions via custom React Flow modeller, Temporal, K8s, Buf, etc.).

---

# 3 – Application & Information Architecture (Draft)

**Audience:** Platform & domain engineers, solution architects, "platform-facing" agents
**Scope:** How Ameide is structured as software (domains, processes, agents, UI) and how information flows and is stored across the platform.

> **Core Invariants**: See [470-ameide-vision.md §0 "Ameide Core Invariants"](470-ameide-vision.md) for the canonical list (four primitives, Graph read-only, Transformation as domain, proto chain, tenant isolation, Backstage internal).

---

## 1. Objectives & Constraints

### 1.1 What this document does

* Defines the **core application building blocks**:

  * Domain primitives
  * Process primitives
  * Agent primitives
  * UISurfaces / process views
* Describes how **information is modeled**:

  * Per-domain SQL stores
  * Process definitions (BPMN) stored in **Transformation Domain**
  * Transformation artifacts (managed via Transformation design tooling UIs, stored by Transformation Domain)
  * Cross-domain knowledge graph (read-only projection)
* Shows how **Backstage** is used as the “declarative front-door” to design domains/processes/agents.
* Aligns with:

  * Proto-based APIs & SDK strategy
  * Unified Artifact Framework for transformation artifacts
  * Temporal-backed Process primitive execution and SaaS-like ops model.

### 1.2 Constraints & design principles (from previous docs)

We carry forward the earlier principles and make them concrete here:

1. **Process-first**
   *Primary abstraction is an end-to-end process (L2O, O2C, T2C…), not a module (Sales, Procurement). Manual workspaces are second-class views over these processes.*

2. **Universal DDD**
   *All behavior lives in bounded contexts (domains), each with its own model, storage and APIs.*

3. **Transformation as a domain**
   *Transformation (requirements, Transformation design tooling artifacts, governance) is modeled like O2C: domain + processes + agents, not a side-console.*

4. **Agentic from any angle**
   *Agents can read from knowledge, call domains, and be invoked from processes in a controlled, typed way.*

5. **Proto-first contracts**
   *Every domain/process/agent exposes a proto-based API, with Buf/BSR governance and SDKs.*

6. **Kubernetes-style declarative state**
   *We declare desired state (e.g. “invoice posted”) and primitives + workflows converge actual state, similar to K8s reconciliation loops and CRDs.*

---

## 2. Core Application Building Blocks

### 2.1 Domain primitives (Domain layer)

**Responsibility:** Encapsulate business concepts, invariants and persistence for a single bounded context.

*Examples:*

* *Sales* domain (Leads, Opportunities, Quotes)
* *Billing* domain (Invoices, Payments, Dunning)
* *Transformation* domain (Initiatives, Backlog Items, Transformation design tooling artifacts)

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
* **Runtime representation** – each Domain primitive exists as a `Domain` CRD. The Domain operator reconciles the CRD into Deployments, Services, HPAs, DB schemas, and ServiceMonitors, enforcing standard Ameide policies.

### 2.2 Process primitives (Process layer)

**Responsibility:** Orchestrate **cross-domain** flows such as L2O, O2C, Procure-to-Pay, or Transformation workflows (Scrum/Togaf ADM).

*Design-time:* **ProcessDefinitions** are BPMN-compliant artifacts produced by a **custom React Flow modeller** and stored in the **Transformation Domain** (modelled via Transformation design tooling UIs). At runtime they are executed by **Process primitives** backed by Temporal workflows.

**Key concepts**

* **ProcessDefinition** (design-time, stored in the Transformation Domain)

  * BPMN-compliant artifact + metadata (tenant, version, process type like L2O/O2C/T2C).
  * Defines process stages, tasks, gateways, and bindings to domains/agents.
  * Stored and versioned in the **Transformation Domain** with revisions & promotions (modelled via Transformation design tooling UIs).
* **Process primitive** (runtime)

  * Logical unit that "executes" a ProcessDefinition:

    * Loads ProcessDefinition from the Transformation Domain.
    * Maps BPMN tasks to Domain primitive API calls and/or Agent primitive tools.
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
* **Runtime representation** – Process primitives are declared via `Process` CRDs that reference ProcessDefinition versions, Temporal namespaces, rollout policies, and dependent Domain/Agent primitives. The Process operator reconciles those CRDs into worker Deployments, Services, and monitoring assets.

### 2.3 Agents (Agent layer)

**Responsibility:** Non-deterministic, AI-powered components that:

* Read and summarize data (domain stores, graph, logs)
* Propose changes (requirements, configurations, data corrections)
* Generate new domain/process/agent definitions via Backstage templates

*Design-time:* **AgentDefinitions** are declarative specs stored in the **Transformation Domain** (modelled via Transformation design tooling UIs). At runtime they are executed by **Agent primitives**.

**Key concepts**

* **AgentDefinition** (design-time, stored in Transformation Domain)

  * Declarative spec for an agent: tools, orchestration graph, scope, risk tier, policies.
  * Stored alongside ProcessDefinitions in the Transformation Domain.
  * Subject to governance (review, promotion) before runtime use.
* **Agent primitive** (runtime)

  * Loads an AgentDefinition from the Transformation Domain and runs the LLM/tool loop.
  * Enforces scope/risk policies at execution time.
  * Always invoked via domain/process APIs; never becomes source of truth.

**Characteristics**

* **Explicit tool contracts** – each AgentDefinition specifies:

  * Which domain/process APIs the Agent primitive can call.
  * What Transformation design tooling / transformation artifacts it can read or modify.
* **Runtime**

  * LangGraph / OpenAI / other inference runtimes orchestrate tools via the Agent primitive.
* **Determinism boundary**

  * Agent primitives never become the authoritative "source of truth" for domain data; they make proposals or trigger domain commands.

**Information model**

* **AgentDefinitions** (stored per tenant in Transformation Domain, modelled via Transformation design tooling UIs):

  * Prompt & behavior description
  * Allowed tools & scopes (e.g. "Transformation agent can only touch transformation artifacts and Backstage templates")
  * Risk tier and policy bindings
* **Interaction history**

  * Persisted in Chat/Threads services for audit and context.
* **Runtime representation** – Agent primitives are instantiated from `Agent` CRDs that reference AgentDefinitions, risk tier, runtime chassis, and tool grants. The Agent operator manages Deployments, Secrets, and observability for each agent runtime.

### 2.4 UISurfaces & process views

We keep two complementary UI modes:

1. **Traditional “ERP-style” workspaces**

   * List/detail pages over domain entities (Customers, Opportunities, Orders, Invoices).
   * Implemented as microfrontends per domain, using the TS SDK to call proto-based APIs.

2. **Process views** (process-first principle)

   * A timeline / swimlane view of L2O/O2C/T2C, driven by process instance state.
   * Users see “where they are” in L2O, not “which form they’re on”.

Information-wise:

* UI does **not** own data; it binds:

  * Domain state → Domain primitives (via SDKs)
  * Process state → Process primitives (via SDKs)
  * Knowledge/analytics → Graph and Transformation design tooling projections

### 2.5 Extensions (Tier 1 WASM)

*Design-time*: `ExtensionDefinition` artifacts are stored in the Transformation Domain alongside ProcessDefinitions and AgentDefinitions via Transformation design tooling UIs.

*Runtime*: A shared `extensions-runtime` service in `ameide-{env}` executes WebAssembly modules for three kinds of hooks:

* `process_hook` – BPMN Extension Tasks in ProcessDefinitions.
* `domain_hook` – explicit extension points in Domain primitives.
* `agent_tool` – deterministic helpers for Agent primitives.

Extensions are sandboxed, multi-tenant, and never own durable state; all data access goes back through domain/process/agent APIs with the same tenant/org/user context as the caller. See [479-ameide-extensibility-wasm.md](479-ameide-extensibility-wasm.md) and [480-ameide-extensibility-wasm-service.md](480-ameide-extensibility-wasm-service.md).

### 2.6 Primitive CRDs & operators

At runtime Ameide treats every Domain/Process/Agent/UISurface primitive as a declarative Kubernetes custom resource:

* `Domain` CRD – domain runtime desired state (image, config, DB bindings, resources)
* `Process` CRD – process runtime desired state (image, Temporal namespace/bindings, rollout policy)
* `Agent` CRD – agent runtime desired state (runtime chassis, model/provider config, tool grants, risk tier)
* `UISurface` CRD – UI runtime desired state (image, routing, auth scopes, feature flags)

ArgoCD applies these CRDs checked into Git; dedicated Ameide operators reconcile them into Deployments, Services, HPAs, Temporal workers, CNPG schemas, ServiceMonitors, and other low-level objects. This keeps Git as the single source of truth while giving SREs a first-class object model for primitives. See [474-ameide-refactoring.md](474-ameide-refactoring.md) for how the platform converges on this primitive + CRD model.

### 2.7 Contract → Implementation → Runtime chain

Every primitive travels the same three-layer chain. We make the links explicit, versioned, and machine-checkable so that a change in the proto contract propagates deterministically into Go code and ultimately into the GitOps manifests that Argo CD applies.

#### Proto (contract)

* Protos live in dedicated Buf modules per bounded context under `packages/ameide_core_proto/` (mirrored into Go modules such as `github.com/ameide/apis/domains/sales`).
* CI runs Buf breaking-change checks against the main branch before publishing a new module tag (e.g. `apis/domains/sales v1.5.0`).[^buf]
* Generated Go (and TS/Python) SDKs are published as versioned modules so downstream services can pin to an exact contract.

#### Implementation (Go service)

* Each Domain/Process/Agent implementation is its own Go module (or sub-module) that **imports the generated proto module** in `go.mod`:

  ```go
  module github.com/ameide/sales-domain

  require github.com/ameide/apis/domains/sales v1.5.0
  ```

* CI enforces:

  * `go mod tidy` cleanliness (no transitive drift).
  * Tests against the imported proto version.
  * Docker images tagged with `git sha` + semantic version (e.g. `ghcr.io/ameide/sales-domain:1.12.3`).
  * Optional OCI labels capturing proto module + version:

    ```bash
    docker build \
      --label 'ameide.io/proto-module=github.com/ameide/apis/domains/sales' \
      --label 'ameide.io/proto-version=v1.5.0' \
      -t ghcr.io/ameide/sales-domain:1.12.3 .
    ```

#### Runtime (GitOps + Argo CD)

* GitOps manifests in the platform repo (and tenant repos) reference the exact image tag and annotate the relationship:

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: sales-domain
    annotations:
      ameide.io/proto-module: github.com/ameide/apis/domains/sales
      ameide.io/proto-version: v1.5.0
      ameide.io/image-digest: sha256:…
  spec:
    template:
      spec:
        containers:
        - name: sales-domain
          image: ghcr.io/ameide/sales-domain:1.12.3
  ```

* Argo CD best practices already recommend separating application source repos from environment/GitOps repos; we follow that model so manifests reflect the latest *published* image, not whatever is on a developer branch.[^argo]
* Rolling out a new proto or service version therefore becomes:

  1. Merge proto PR → Buf module tag.
  2. Update Go module import + release image.
  3. Update GitOps manifest (fast-forward PR) to new image tag.

#### Service descriptor & catalog cross-link

To make the chain easy to query, each primitive carries a Backstage `catalog-info.yaml` (or `service.yaml`) describing:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: sales-domain
  annotations:
    ameide.io/proto-module: github.com/ameide/apis/domains/sales
    ameide.io/proto-version: v1.5.0
    ameide.io/gitops-path: gitops/platform/domains/sales-domain
spec:
  type: domain
  lifecycle: production
  owner: platform/sales
```

Backstage ingests these descriptors so humans and agents can answer “which proto version is live in prod?” without scraping clusters.[^backstage] Automation (e.g. the `PrimitiveImplementationDraft` flow in [478-ameide-extensions.md](478-ameide-extensions.md)) uses the same metadata to open the right PRs in source and GitOps repos. `478` goes deeper on folder layout and template scaffolding; this section documents why the linkage is required.

[^buf]: Buf modules and breaking change detection – [buf.build/docs/breaking](https://buf.build/docs/breaking/?utm_source=chatgpt.com)
[^argo]: Argo CD best practices for separating source vs config repos – [argo-cd.readthedocs.io](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/?utm_source=chatgpt.com)
[^backstage]: Backstage Software Catalog descriptors – [backstage.io/docs/features/software-catalog](https://backstage.io/docs/features/software-catalog/?utm_source=chatgpt.com)

---

### 2.8 Proto Semantic Layers (Entities / Interfaces / Events)

Proto serves as Ameide's single schema language, but carries **three distinct semantics**:

| Semantic | Purpose | Proto construct | Package convention |
|----------|---------|-----------------|-------------------|
| **Entity** | Data that lives in the DB | `message` | `ameide.{domain}.v1` |
| **Interface** | Operations callers invoke | `service` | `ameide.{domain}.v1` |
| **Event** | Facts that happened | `message` | `ameide.{domain}.events.v1` |

These are differentiated via **conventions + lightweight options**, not a separate metadata system. This keeps us aligned with the "code-first, thin metadata" stance (470 §5).

#### 2.8.1 Entities

Entities are the "business objects" a Domain primitive owns. They map conceptually to tables or aggregates, but we don't force 1:1 with DB tables in proto (DB schema stays an implementation detail).

**Conventions:**

* Live in the domain package for that bounded context
* Can be tagged with `(ameide.entity_kind)` for tooling: `AGGREGATE`, `PROJECTION`, `LOOKUP`

```proto
package ameide.sales.v1;

message Opportunity {
  string id = 1;
  string tenant_id = 2;
  string organization_id = 3;
  string customer_id = 4;
  string name = 5;
  string stage = 6;
}

// Projection entity (like a read model / view)
message OpportunityListItem {
  option (ameide.entity_kind) = PROJECTION;
  string id = 1;
  string customer_name = 2;
  string stage = 3;
}
```

#### 2.8.2 Interfaces

Interfaces are the `service` definitions – what UIs, agents, or other domains actually call. We typically split into:

* **Domain APIs** – internal, rich operations
* **Integration APIs** – external, stable contracts

```proto
service SalesDomainService {
  option (ameide.interface_kind) = DOMAIN;

  // Commands
  rpc CreateOpportunity(CreateOpportunityRequest) returns (Opportunity);
  rpc UpdateOpportunityStage(UpdateOpportunityStageRequest) returns (Opportunity);

  // Queries
  rpc GetOpportunity(GetOpportunityRequest) returns (Opportunity);
  rpc ListOpportunities(ListOpportunitiesRequest) returns (ListOpportunitiesResponse);
}

service SalesIntegrationService {
  option (ameide.interface_kind) = INTEGRATION;

  rpc UpsertOpportunityProjection(OpportunityProjection) returns (UpsertResult);
  rpc StreamOpportunityChanges(ChangesRequest) returns (stream OpportunityChangeEvent);
}
```

#### 2.8.3 Events

Events are messages representing facts that already happened. They live in a separate `events` package and are tagged by kind:

* `BUSINESS` – external integration events (stable contracts)
* `DOMAIN_INTERNAL` – internal domain events (implementation detail)

```proto
package ameide.sales.events.v1;

message OpportunityWon {
  option (ameide.event_kind) = BUSINESS;
  string id = 1;
  string tenant_id = 2;
  string customer_id = 3;
  double amount = 4;
  string currency = 5;
  google.protobuf.Timestamp won_at = 6;
}

message OpportunityStageChanged {
  option (ameide.event_kind) = DOMAIN_INTERNAL;
  string id = 1;
  string from_stage = 2;
  string to_stage = 3;
}
```

Events are published by Domain/Process primitives via Watermill (§3.3.1) and consumed by other domains or projections.

#### 2.8.4 Typical Domain Layout

```text
ameide/
  sales/
    v1/
      sales_entities.proto      # Opportunity, Customer, Quote, ...
      sales_service.proto       # SalesDomainService
      sales_integration.proto   # SalesIntegrationService (optional)
    events/
      v1/
        sales_events.proto      # OpportunityWon, InvoicePosted, ...
```

#### 2.8.5 Design Heuristic

When deciding where something belongs:

> * *Do I need to store this as domain state?* → **Entity** (`message` in domain package)
> * *Is this an operation a caller invokes?* → **Interface** (`service`)
> * *Is this a fact that already happened?* → **Event** (`message` in `events/v1`)

For mappings to D365 and SAP concepts, see [470a-ameide-vision-vs-d365.md](470a-ameide-vision-vs-d365.md) and [470b-ameide-vision-vs-saps4.md](470b-ameide-vision-vs-saps4.md).

---

## 3. Information Architecture

### 3.1 Multi-tenant Data Layout (conceptual)

At the information level the platform is **logically multi-tenant**, regardless of how physical databases are configured in the tech layer.

> **Security**: For the full security model and implementation gaps (RLS, `organization_id` columns, API authorization), see [476-ameide-security-trust.md](476-ameide-security-trust.md).

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

   * Transformation design tooling content: BPMN, ArchiMate, Markdown docs, etc.
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

This mirrors Temporal's distinction between workflow definitions and workflow executions: ProcessDefinitions are design-time artifacts; Process primitives execute them at runtime.

We will:

* Store **ProcessDefinitions** in **Transformation design tooling / Transformation domain** as versioned artifacts.
* Store **execution projections** in Ameide's own DB for:

  * process analytics
  * SLA monitoring
  * process-first UI views.

### 3.3.1 Event-driven CQRS runtime (Watermill)

Many Process primitives (especially those written in Go) are implemented with Watermill’s CQRS component so they can emit and consume commands/events as plain Go structs while relying on Watermill for serialization, routing, and pub/sub plumbing.

* **Commands** – intent to change state (e.g. `RegisterUser`, `CreateInvoice`). Process primitives send commands via `CommandBus`, which Watermill serializes and routes through the configured pub/sub backend (Kafka, RabbitMQ, Postgres, etc.). Command handlers mutate the write model and/or emit domain events.
* **Events** – facts that happened (e.g. `UserRegistered`). Domain primitives publish events through `EventBus`; read models and integrations subscribe via `EventProcessor` handlers to update projections or trigger side effects.
* **Queries** – read models (Postgres, Elastic, etc.) stay optimized for queries and are updated by event handlers.

Watermill’s CQRS layer gives us:

- Type-safe command/event handlers (`func(ctx context.Context, cmd *RegisterUser) error`) without hand-written topic or encoding code.
- Pluggable pub/sub backends, consistent middleware (logging, metrics, retries).
- A router abstraction that makes it easy to compose Process primitives from command and event handlers.

When describing Process/Domain implementations, reference Watermill CQRS so teams standardize on the same event-driven plumbing; see [`github.com/ThreeDotsLabs/watermill`](https://github.com/ThreeDotsLabs/watermill?utm_source=chatgpt.com) and the [Watermill CQRS docs](https://watermill.io/docs/cqrs/?utm_source=chatgpt.com). For lighter weight use cases (e.g., TypeScript agents) we still follow the same CQRS concepts even if Watermill is not involved.

### 3.4 Knowledge Graph (Read-side only)

> **Core Definitions** (see [470-ameide-vision.md §0](470-ameide-vision.md)):
> - **Graph** is a read-only knowledge layer that projects selected data from any primitive into a graph database.
> - All writes go through primitives; Graph is never a source of truth.

The Knowledge Graph provides a **cross-domain read-only model**:

* Entities: Customer, Product, Opportunity, ProcessInstance, Initiative, System, API, etc.
* Relationships: `CUSTOMER->HAS_OPPORTUNITY`, `OPPORTUNITY->FLOWS_THROUGH_L2O`, `SERVICE->IMPLEMENTS_API`.

This is *not* an authoritative store:

* Write path: Domain/Process primitives project to graph as part of their post-commit flow.
* Read path: agents and analytics use it to understand topology and behavior (e.g. "show me all L2O paths where margin < 10%").

**Design artifacts** (BPMN, architecture diagrams, Markdown) are persisted by the **Transformation Domain**, using an artifact + revision pattern. Graph can project these artifacts or references to them for cross-domain queries, but the canonical records remain in the Transformation Domain.

### 3.5 Transformation / Transformation design tooling Data

> **Core Definitions** (see [470-ameide-vision.md §0](470-ameide-vision.md)):
> - **Transformation Domain** owns all design-time artifacts (ProcessDefinitions, AgentDefinitions, BPMN, diagrams, Markdown).
> - **Transformation design tooling** is the set of modelling UIs (BPMN editor, diagram editor, Markdown editor) that call Transformation Domain APIs—Transformation design tooling has no independent storage.

Transformation is a **Domain primitive like any other**, owning:

* **Initiatives & Backlog Items**

  * Standard domain entities (SQL) representing epics, features, tasks.
* **Design-time artifacts** (stored in Transformation Domain, modelled via Transformation design tooling UIs)

  * **ProcessDefinitions** – BPMN-compliant process models (produced by custom React Flow modeller).
  * **AgentDefinitions** – Declarative agent specs (tools, policies, risk tiers).
  * **ExtensionDefinitions** – WASM extension specs for process/domain/agent hooks.
  * ArchiMate, Markdown, etc., optionally event-sourced internally with commands and snapshots.
* **Governance**

  * Promotion status of artifacts (draft, in review, promoted).
  * Links from artifacts to runtime primitives (e.g. "ProcessDefinition L2O_v3 is executed by Process primitive l2o-process-svc").

> **Important**: There is no separate "Transformation design tooling service" in the runtime. The event-sourced artifact store is part of the Transformation Domain. Transformation design tooling is the *client* (modelling experience) that sends commands to Transformation.

Transformation Agent primitives operate primarily on this domain:

* Read backlog + design artifacts (ProcessDefinitions, AgentDefinitions) from Transformation Domain.
* Generate or modify Backstage templates and Domain primitive/Process primitive/Agent primitive configurations.
* Propose changes that are then applied via standard deployment workflows.

---

## 4. Backstage as the Application Catalog & Template Engine

> **Important:** Backstage is an **internal factory** for Ameide engineers and transformation agents; **tenants never see Backstage**. Tenant-facing UX is delivered through UISurface primitives (Next.js apps), not through Backstage.

We standardize on **Backstage** as the catalog and templating layer for:

* Domain primitives
* Process primitives
* Agents
* UI components / microfrontends

This aligns with Backstage’s ecosystem modeling and software templates features.

### 4.1 Catalog modeling

**Backstage entities** (examples):

* `Domain` (custom kind)

  * Represents a bounded context such as *Sales*, *Billing*, *Transformation*.
* `Component` (service)

  * Domain primitive service (e.g. `sales-domain-svc`).
  * Process primitive service (e.g. `l2o-process-svc`).
* `API`

  * Proto-defined API surface of a domain or process.
* `Resource`

  * External dependencies (DB cluster, Kafka topics, Camunda cluster, Temporal cluster).
* `Template`

  * Domain/Process/Agent/UI templates used by the scaffolder.

**Mapping to Ameide concepts**

| Ameide concept     | Backstage kind                     |
| ------------------ | ---------------------------------- |
| Domain primitive   | `Component` (+ custom `Domain`)    |
| Process primitive  | `Component` + BPMN artifact link   |
| Agent              | `Component` or custom `Agent` kind |
| UI workspace       | `Component` (frontend)             |
| Domain API         | `API`                              |
| Process API        | `API` (process control / queries)  |
| Transformation design tooling artifact       | `Resource` or custom `Artifact`    |

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

2. **Process primitive Template**

   * Inputs: process type (L2O, O2C, custom), participating domains, SLA target.
   * Generates:

     * Initial ProcessDefinition (BPMN-compliant, from React Flow modeller) with lane structure (Sales, Billing, Logistics, etc.).
     * Temporal workflow skeletons (service tasks bound to Domain primitive APIs).
     * Process primitive service skeleton.
     * Backstage `Component` entity + links to domain APIs.

3. **Agent primitive Template**

   * Inputs: agent purpose, allowed domains/processes, risk tier.
   * Generates:

     * AgentDefinition (prompt, tools, scopes, policies) stored in Transformation Domain.
     * Proto definition for the agent configuration & invocation.
     * Agent primitive service skeleton.
     * Backstage `Component`/`Agent` entity with RBAC metadata for who can use this template.

4. **UI Template**

   * Inputs: target domain/process, view style (workspace vs process view).
   * Generates:

     * Next.js microfrontend skeleton with Ameide TS SDK integration.
     * Backstage `Component` entry of type `frontend`.

Backstage templates remain **the control plane** for agents too: a transformation agent manipulates template parameters and triggers scaffolding rather than directly creating low-level services.

> Templates are strictly **Tier 2** tooling for full primitives. Tier 1 WASM extensions follow the `ExtensionDefinition` + shared runtime path from 479/480 and do not create new services via Backstage.

Every primitive-oriented template emits both a repository skeleton (code, proto, CI) and the corresponding Domain/Process/Agent/UISurface CR so GitOps/Argo can apply the runtime object. Human engineers and agents therefore work with the same declarative surface (the CR specs) regardless of who authored the primitive.

---

## 5. Example: L2O Application & Information Flow

To make this less abstract, here’s how **Lead-to-Opportunity (L2O)** looks in this architecture.

### 5.1 Involved building blocks

* Domain primitives:

  * `SalesDomain` (Lead, Contact, Opportunity)
  * `ProductDomain` (Products, Price Lists)
  * `BillingDomain` (for credit check / pre-invoice)
* Process primitive:

  * `L2OProcess` – BPMN definition + process service.
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

   * `L2OProcess primitive` is started with `leadId` (Temporal).
   * Process instance state is stored in Temporal; Ameide stores a projection (e.g. "L2O phase = Qualification").

3. **Quote created**

   * `L2OProcess primitive` reaches "Prepare Quote" task.
   * For manual: Sales user opens `SalesWorkspace`, sees tasks from `L2OProcessView` and creates a Quote via `SalesDomain primitive` APIs.
   * For automated: an Agent primitive or automated task calls `SalesDomain primitive::CreateQuote` using its proto API.

4. **Approval**

   * `L2OProcess primitive` orchestrates approval steps, maybe invoking:

     * `TransformationAgent primitive` to suggest better pricing.
     * `SalesCoachAgent primitive` to highlight risk.
   * Agent primitives read from Graph & domain APIs but write back via domain commands only.

5. **Handover to O2C**

   * Once L2O is "Won", `L2OProcess primitive`:

     * Calls `BillingDomain primitive::CreateProformaInvoice`.
     * Emits `OpportunityWon` → `O2CProcess primitive` starter.

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
* Transformation design tooling artifacts describing L2O design (as the process improves).

---

## 6. Alignment with Existing Specs

This section is just to show that we’re not inventing yet another stack, but reusing patterns from existing Ameide backlogs.

1. **IPA Architecture (043)**

   * IPA = "composition of workflows + agents + tools from DB, compiled to runtimes".
   * New view:

     * *Domain primitives* ≈ "domain and storage" part of IPA.
     * *Process primitives* (executing ProcessDefinitions) ≈ "workflows" part.
     * *Agent primitives* (executing AgentDefinitions) ≈ "agents/tools" part.
   * The layered approach (Domain → Storage → Builder → Runtime) remains valid, but the *unit* moves from "IPA" to explicit ProcessDefinitions and AgentDefinitions (design-time) executed by primitives (runtime).

2. **Proto-based APIs (044)**

   * All domain/process/agent services are still proto-first, with gRPC + REST surfaces and consistent error handling.

3. **North-Star (064)**

   * Design vs Deploy vs Runtime separation is kept:

     * Design: Backstage + Transformation design tooling + transformation domain.
     * Deploy: primitives & process engines.
     * Runtime: domain & process services.

4. **Transformation design tooling (067)**

   * The Transformation design tooling *pattern* (artifacts, revisions, promotions) is implemented inside the **Transformation Domain**.
   * Transformation design tooling as a UI surfaces those artifacts; the primitive remains the source of truth.
   * Instead of a generic "IPA designer", the Transformation Domain (with Transformation design tooling UIs) is used to generate Backstage templates & primitive config.

5. **BPMN-compliant definitions (063)**

   * BPMN semantics (stages, gateways, lanes) remain valid; we use a **custom React Flow modeller** (not Camunda/bpmn-js) to produce ProcessDefinitions stored in the Transformation Domain (via Transformation design tooling UIs) and executed by Temporal-backed Process primitives.

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
| [305‑workflow](305-workflow.md) | Process primitive implementation | See §8.2 |
| [310‑agents‑v2](310-agents-v2.md) | Agent primitive implementation | See §8.3 |
| [370‑event‑sourcing](370-event-sourcing.md) | Event sourcing exploration | See §8.4 |
| [478‑ameide‑extensions](478-ameide-extensions.md) | Extension namespace topology | See §8.5 |
| [479‑ameide‑extensibility‑wasm](479-ameide-extensibility-wasm.md) | Tier 1 + Tier 2 extensibility model | See §2.5, §8.5 |
| [480‑ameide‑extensibility‑wasm‑service](480-ameide-extensibility-wasm-service.md) | Shared runtime implementation | See §2.5 |

### 8.1 Metamodel alignment (300)

The Element graph (300) and Application Architecture (472) coexist:

* **Graph = read-side projection**: Cross-domain knowledge for agents, analytics, and transformation. Not authoritative for operational data.
* **Domain stores = write-side authority**: Each Domain primitive owns its data; graph receives projections.
* **Design artifacts are projected**: BPMN, ArchiMate, Markdown stored in **Transformation Domain** and **projected** as versioned nodes in the graph.

> **Important**: Transformation remains the source of truth for transformation artifacts; the graph stores references and read-optimised views, not the authoritative records.

This means: Transformation domain entities project to `graph.elements`, but the canonical records stay in the Transformation Domain. Process primitive runtime state stays in Temporal.

### 8.2 Workflow alignment (305)

305 describes platform workflow infrastructure implementing **Process primitives** from this document:

* "Platform Workflows" = implementation detail of Process primitives (§2.2)
* Temporal is the runtime; customers see "process stages" and "process boards"
* ProcessDefinitions (BPMN-compliant, from custom React Flow modeller) stored in Transformation Domain (via Transformation design tooling UIs); compiled to Temporal workflows

### 8.3 Agent alignment (310)

310 describes Agent primitive implementation:

* AgentDefinitions stored in Transformation Domain (modelled via Transformation design tooling UIs); Agent primitives execute them
* Key constraint: Agent primitives invoked via domain/process APIs; never become source of truth

### 8.4 Event sourcing alignment (370)

472 §3.5 describes Transformation Domain with "optionally event-sourced" artifacts:

* Event contracts per domain, outbox + relay, projections from streams
* The event-sourced pattern is an internal implementation detail of the Transformation Domain, not a separate service

### 8.5 Extension alignment (478)

478 describes how tenant-specific primitives are cataloged and deployed:

* **Platform repo** contains `service_catalog/` with base primitive structure.
* **Tenant repos** (`tenant-{id}-controllers`, `tenant-{id}-gitops`) hold custom code and manifests.
* Backstage catalog uses **Locations** pointing at tenant repos, so tenant primitives appear in the internal portal.
* Namespace topology by SKU determines where primitives deploy (§4 of 478).

The catalog entity mappings in §4.1 extend to tenant primitives:

| Ameide Concept | Backstage Kind | Repository |
|----------------|----------------|------------|
| Platform Domain primitive | `Component` | Platform repo |
| Tenant Domain primitive | `Component` | `tenant-{id}-controllers` |
| Platform Process primitive | `Component` | Platform repo |
| Tenant Process primitive | `Component` | `tenant-{id}-controllers` |
| Platform Agent primitive | `Component` | Platform repo |
| Tenant Agent primitive | `Component` | `tenant-{id}-controllers` |

Only the **Transformation Domain** manipulates primitive definitions (ProcessDefinitions, AgentDefinitions, ExtensionDefinitions, PrimitiveImplementationDrafts). Backstage templates and GitOps keep platform vs tenant concerns isolated while letting us enforce guardrails (namespaces, secrets, rollout waves).

---

## 9. Terminology

| Legacy Term | 472 Term |
|-------------|----------|
| IPA (Intelligent Process Automation) | Domain primitive + Process primitive + Agent primitive bundle |
| Platform Workflows (305) | Process primitive (executes ProcessDefinitions) |
| AgentRuntime (310) | Agent primitive (executes AgentDefinitions) |
| "Graph service" | Knowledge Graph (read projection) |
| DomainService | Domain primitive |
| BPMN definition | ProcessDefinition (from custom React Flow modeller) |


<!-- ═══════════════════════════════════════════════════════════════════════════ -->
<!-- FILE: 473-ameide-technology.md -->
<!-- ═══════════════════════════════════════════════════════════════════════════ -->

Here’s **Document 4/6 – Technology Architecture**.

---

# 4 – Technology Architecture (Ameide Platform)

**Status:** Draft
**Audience:** Platform / Domain & Process Teams / DevEx / SRE
**Scope:** How Ameide is built and operated: runtimes, infra, integration points, and the technical guardrails that make "Domain / Process / Agent / UISurface" primitives real.

> **Core Invariants**: See [470-ameide-vision.md §0 "Ameide Core Invariants"](470-ameide-vision.md) for the canonical list (four primitives, Graph read-only, Transformation as domain, proto chain, tenant isolation, Backstage internal).
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

---

## 1. Purpose & Relationship to Other Docs

* The **Vision** document says *what* Ameide aspires to be.
* The **Business Architecture** document describes *who* uses it and *for what* (L2O/O2C, transformation, etc.).
* The **Application & Information Architecture** describes *which logical building blocks* (Domain primitive, Process primitive, Agent, Transformation, Transformation design tooling) exist and how data flows between them.
* **This document** describes *how those building blocks are implemented and operated* using Kubernetes, Backstage, Temporal, proto-based APIs, and the Ameide SDKs.

The goal is to give implementers a **vendor-aligned blueprint** that can be executed with minimal invention.

---

## 2. Technology Principles

1. **Kubernetes-native declarative primitives**

   * We use Kubernetes for *both infrastructure and primitive lifecycles*. Domain/Process/Agent/UISurface primitives are declared via their CRDs; GitOps applies those CRDs and Ameide operators reconcile them into Deployments, Services, HPAs, Temporal workers, and supporting config.
   * We still reserve CRDs for coarse application constructs (primitives, tenants, infra). Business aggregates (Orders, Customers, etc.) remain inside domain services; we do **not** create one CRD per aggregate. ([Kubernetes][1])

2. **Proto‑first, SDK‑first APIs**

   * All Domain/Process/Agent services share a single proto source of truth (Buf BSR) and generated server + client stubs. 
   * The canonical way to talk to the platform is via the language SDKs (TS/Go/Python) and the shared `AmeideClient` contract, not by hand‑rolling gRPC clients. 

3. **Backstage as the “factory” for primitives**

   * Backstage’s **Software Catalog** and **Software Templates / Scaffolder** are used to create and manage all Ameide Domain/Process/Agent/UISurface primitives and their Helm charts. ([Backstage][2])
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

   * Argo CD + Helm remain the deployment plane. Git stores Domain/Process/Agent/UISurface CRDs, infra CRDs, and operator manifests; operators own the low-level Deployments/Services, so Argo never `kubectl apply`s mutable child resources directly.

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
  * Application control plane: Domain, Process, Agent, and UISurface operators reconcile CRDs into workloads across namespaces/SKUs.
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

* Three Ameide operators watch the primitive CRDs:

  * **Domain operator** – reconciles each Domain CRD into Deployments, Services, HPAs, ServiceMonitors, CNPG schemas/users, migration Jobs, and NetworkPolicies. Enforces namespace/SKU placement, image policies, resource classes, and standard sidecars/env vars.
  * **Process operator** – reconciles Process CRDs into Temporal worker Deployments, ConfigMaps linking BPMN to code, ServiceMonitors, and optional CronJobs for compensations. Validates references to ProcessDefinitions and Temporal namespaces.
  * **Agent operator** – reconciles Agent CRDs into agent runtime Deployments, Secrets (LLM keys, tool credentials), allowlisted tool grants, and observability wiring.

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
* **Gap**: Per 470 §9.2.3, TypeScript SDK is not yet published to npmjs. This blocks external consumers from using the TS SDK as 473 §5.1 requires.

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
| Process primitive | Workflow (305) | Runtime that executes ProcessDefinitions |
| ProcessDefinition | BPMN model (deprecated) | Design-time artifact in Transformation design tooling (from custom React Flow modeller) |
| Agent primitive | AgentRuntime (deprecated) | Runtime that executes AgentDefinitions |
| AgentDefinition | Agent config (deprecated) | Design-time artifact in Transformation design tooling |
| Component domain (465) | N/A | 465 "domain" = folder category (apps, data), not business domain |

**Clarification needed**: "Domain" is overloaded:
* **Business domain** (473 §3.3): bounded context (Orders, Product, Identity)
* **Component domain** (465): ArgoCD folder category (apps, data, platform)

Recommend using "component category" or "deployment domain" for 465-style usage.

### 11.2 Implementation gaps

| Gap | Description | Impact | Related Docs |
|-----|-------------|--------|--------------|
| Backstage not deployed | §4 assumes Backstage as primitive factory | Cannot use template-driven service creation | 470 §9.2.2 |
| Transformation artifact storage | §2.6 assumes event-sourced ProcessDefinitions/AgentDefinitions in Transformation Domain | Implementation detail; infra TBD | 471 §3.2 |
| Temporal namespace isolation | §3.2 mentions per-tenant namespaces | Current deployment is single namespace | 305 |
| AmeideTenant CRD | §3.1 suggests tenant CRD | 333 uses API-driven realm provisioning | 333 |
| ProcessDefinition→Temporal compiler | §3.2 assumes ProcessDefinition to Temporal translation | No compiler exists yet | 305, 471 |

### 11.3 Open architectural tensions

1. **Backstage vs GitOps static components**
   * 473 §4 envisions Backstage generating new primitives dynamically
   * 465 uses static `component.yaml` files discovered by ApplicationSets
   * **Resolution needed**: How do Backstage-generated services integrate with 465 GitOps model?

2. **AmeideTenant CRD vs API-driven**
   * 473 §3.1 suggests `AmeideTenant` CRD for tenant provisioning
   * 333 implements realm provisioning via Keycloak Admin API
   * **Decision needed**: Is CRD-based approach still desired, or has API-driven superseded it?

3. **Agent primitive runtime choice**
   * 473 §3.3.3 describes LangGraph-based Agent primitives
   * 310 describes n8n-aligned visual workflows
   * **Resolution**: Both are valid Agent primitive implementations; AgentDefinitions abstract the runtime choice

---

If you'd like, next we can do **Document 5/6 – Domains Architecture**, and apply this tech stack concretely to L2O and Transformation (including where Transformation design tooling, ProcessDefinitions, and Temporal sit in those domains).

[1]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/?utm_source=chatgpt.com "Custom Resources"
[2]: https://backstage.io/docs/features/software-catalog/?utm_source=chatgpt.com "Backstage Software Catalog and Developer Platform"
[3]: https://docs.temporal.io/workflows?utm_source=chatgpt.com "Temporal Workflow | Temporal Platform Documentation"
[5]: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/?utm_source=chatgpt.com "Operator pattern"
[6]: https://medium.com/%40ajayshekar01/best-practices-for-building-temporal-workflows-a-practical-guide-with-examples-914fedd2819c?utm_source=chatgpt.com "Best Practices for Building Temporal Workflows"
[7]: https://backstage.io/docs/features/software-templates/?utm_source=chatgpt.com "Backstage Software Templates"


<!-- ═══════════════════════════════════════════════════════════════════════════ -->
<!-- FILE: 474-ameide-refactoring.md -->
<!-- ═══════════════════════════════════════════════════════════════════════════ -->

# 474 – Refactoring & Migration Plan (Domain / Process / Agent / UISurface)

**Status:** Draft v1
**Audience:** Platform & product engineering, architecture, internal agents

This guide describes **how we move from the current service topology and legacy concepts to the Domain / Process / Agent / UISurface primitives and their CRDs**, while staying **code-first and AI-first**. It complements the vision and architecture docs in the 470–480 range.

> **Core Invariants**: See [470-ameide-vision.md §0 "Ameide Core Invariants"](470-ameide-vision.md) for the canonical list (four primitives, Graph read-only, Transformation as domain, proto chain, tenant isolation, Backstage internal).
>
> **Cross-References**:
> - [471 §4](471-ameide-business-architecture.md) – Backstage as the platform factory (business view)
> - [472 §2](472-ameide-information-application.md) – Primitive definitions (Domain/Process/Agent/UISurface)
> - [472 §2.7](472-ameide-information-application.md) – Contract → Implementation → Runtime chain
> - [473 §4](473-ameide-technology.md) – Backstage implementation as factory
> - [475 §4](475-ameide-domains.md) – Domain portfolio and E2E processes
> - [478](478-ameide-extensions.md) / [479](479-ameide-extensibility-wasm.md) / [480](480-ameide-extensibility-wasm-service.md) – Tier 1/Tier 2 extensibility model

---

## 1. Objectives

### 1.1 What we’re aiming for

1. **Code-first, AI-first**

   * The **canonical representation** of the system is **plain code**: modules, functions, types, tests.
   * AI and humans both operate on this code directly (with help from design artifacts), not on a giant metadata layer.
   * Metadata exists only where it helps AI and operations; it never replaces code.

2. **Four primitives only**

   We converge on **four application primitives**:

   * **Domain** – owns business state and rules for a bounded context.
   * **Process** – orchestrates multiple Domains and Agents into an end-to-end outcome.
   * **Agent** – non-deterministic worker using typed tools.
   * **UISurface** – user-facing entry point (workspace or process view).

   Each primitive has two faces:

   * A **Primitive** – the code: packages, modules, tests.
   * A **CRD** – a declarative runtime object on Kubernetes (Domain CRD, Process CRD, Agent CRD, UISurface CRD).

3. **CRDs for runtime wiring, not for modeling the business**

   * App-level CRDs exist **only** for the four primitives and a small number of infra objects (tenant, DB, etc.).
   * We **never** introduce CRDs for tables, fields, forms, or other fine-grained constructs.

4. **Not a traditional metadata-driven platform**

   * We **deliberately avoid** the pattern where a giant metadata system is the “real” model and code is just glue written by human developers.
   * Here, **code *is* the model**. Design-time artifacts and CRDs are hints that keep AI and ops sane, not a meta-runtime.

5. **Backward-compatible, incremental migration**

   * We keep systems running while we gradually re-shape them.
   * Existing services are wrapped and renamed where possible, not rewritten from scratch on day one.

---

## 2. Target model (quick recap)

This section restates the target model in the new vocabulary so the rest of the guide can be very literal. For full definitions, see [472 §2](472-ameide-information-application.md) and [475 §4](475-ameide-domains.md).

### 2.1 Primitives

**Domain primitive**

* Code that owns:

  * A bounded business concept (e.g. Orders, Transformation, Product).
  * Its persistence schema and migrations.
  * Its invariants and events.
* Exposes a stable API (proto or equivalent) for other primitives and surfaces.

**Process primitive**

* Code that orchestrates:

  * Calls to Domain primitives.
  * Interactions with Agent primitives.
  * Lifecycle of a business instance (e.g. “this opportunity from lead to order”).

**Agent primitive**

* Code that:

  * Accepts typed requests.
  * Uses tools that correspond to Domain and Process APIs.
  * Emits typed proposals or decisions.

**UISurface primitive**

* Code for user-facing apps:

  * Workspaces (entity-centric).
  * Process views (stage/board/timeline).
* Uses generated SDKs / typed clients for Domain, Process, Agent primitives.

### 2.2 CRDs

For each primitive, there is **at most one CRD type**. For operator design, see [473 §3](473-ameide-technology.md).

* **Domain CRD** – describes how a Domain primitive is run:

  * image, config, resources, DB bindings, observability.
* **Process CRD** – describes how a Process primitive is run:

  * image, process type, queue/topic bindings, timeouts.
* **Agent CRD** – describes how an Agent primitive is run:

  * image, model config, tool grants, risk tier.
* **UISurface CRD** – describes how a UISurface primitive is run:

  * image, routing, auth scopes, feature flags.

Operators reconcile these CRDs into Deployments/Services/etc., but **the CRDs never describe tables, fields or layout**.

### 2.3 Contract → Implementation → Runtime chain

Every primitive participates in the contract chain described in [472 §2.7](472-ameide-information-application.md):

1. **Proto module** (`packages/ameide_core_proto`) defines the API contract.
2. **Generated SDKs** (Go/TS/Python) are produced from proto via Buf.
3. **Service implementation** pins to proto version via `go.mod` / `package.json`.
4. **Docker image** tagged and labelled with proto version metadata.
5. **GitOps manifest** references exact image tag + proto annotations.
6. **Backstage catalog** links service to proto module/version.

Refactors must preserve this chain: proto version bumps trigger SDK regen, image rebuilds, and GitOps manifest updates.

---

## 3. Current state (very high-level)

Today we have:

* A set of **domain-ish services**:

  * own their own schema and APIs (e.g. platform, identity, transformation, graph, various business services).
* A set of **workflow-ish services**:

  * implement long-running flows in code, but are not yet framed as Process primitives.
* A set of **agent-ish services**:

  * provide LLM-driven helpers using internal tool stacks.
* **UI apps**:

  * the main platform UI and adjunct tools.
* A number of **legacy concepts** (IPA, “platform workflows”, legacy controller naming) still present in code and docs.

The goal is to **relabel and reshape** these into the four primitives + CRDs without pausing the platform.

---

## 4. Refactoring principles

These rules should be applied ruthlessly across code, docs and tooling.

### 4.1 Naming

1. **Use `{Domain/Process/Agent/UISurface} primitive` for code**

   * Example: “Sales Domain primitive”, “Onboarding Process primitive”, “Transformation Agent primitive”.

2. **Use `{Domain/Process/Agent/UISurface} CRD` for runtime objects**

   * Example: “Sales Domain CRD”, “L2O Process CRD”, “Transformation Agent CRD”, “Sales UISurface CRD”.

3. **Do not use old controller names in new work**

   * Avoid `DomainController`, `ProcessController`, `AgentController`.
   * Avoid `Intelligent*`, `IDC`, `IPC`, `IAC`.
   * When touching old code, move it incrementally to the new naming.

### 4.2 Code-first, AI-first

4. **Plain code is the source of truth**

   * Each primitive is a **module or package** with:

     * Types/entities.
     * Public API.
     * Tests.
   * Design-time diagrams, processes, and specs are commentary around the code, not executable metadata.

5. **AI operates on the same code as humans**

   * Refactoring and extension work is expressed as:

     * “Change this Domain primitive in these ways.”
     * “Introduce a new Process primitive between Sales and Orders.”
   * We avoid any separate DSL or meta language that only AI understands.

### 4.3 Minimal metadata

6. **Use CRDs only for runtime configuration**

   * CRDs should contain:

     * References to the code artifact (image/tag, or build ref).
     * Operational settings (resources, SLOs, rollout strategy).
   * CRDs should **not** try to encode:

     * Entity schemas.
     * Field properties.
     * UI layouts.
     * Business rules.

7. **Design artifacts are guidance, not runtime**

   * BPMN diagrams, architecture diagrams, written specs:

     * Live in the Transformation area (or equivalent).
     * Help humans and agents understand the system.
     * Never act as the runtime model of truth.

### 4.4 Gradual, reversible changes

8. **Wrap before rewrite**

   * First give each area a clear primitive name (Domain, Process, Agent, UISurface).
   * Introduce CRDs that point at the existing deployments.
   * Only then start splitting or reshaping code.

9. **Keep behavior identical per step**

   * A refactor step is “done” only when:

     * External APIs are unchanged.
     * Observability and SLA coverage are at least as good as before.

---

## 5. Mapping old to new (conceptual)

We avoid naming legacy types explicitly, but we still need a mental map:

* **Any service that owns business state + rules + schema**
  → becomes a **Domain primitive**, with a **Domain CRD** for runtime.

* **Any service that orchestrates cross-domain flows over time**
  → becomes a **Process primitive**, with a **Process CRD** for runtime.

* **Any service that wraps an LLM with tools and policies**
  → becomes an **Agent primitive**, with an **Agent CRD** for runtime.

* **Any UI app that presents workspaces or process views**
  → becomes a **UISurface primitive**, with a **UISurface CRD** for runtime.

The rest of this guide assumes we can identify current services that match these shapes and rename them accordingly.

---

## 6. Step-by-step refactoring plan

> **Extensibility context**: Phases 0–2 establish the primitive foundation. Tenant-specific extensions (Tier 1 WASM, Tier 2 primitives) layer on top of this foundation—see [478](478-ameide-extensions.md), [479](479-ameide-extensibility-wasm.md), [480](480-ameide-extensibility-wasm-service.md) for how extensions interact with the primitives being refactored here.

### 6.1 Phase 0 – Foundations

**Goal:** Create the new vocabulary and minimal tooling without moving any behavior.

1. **Introduce primitive types in the codebase**

   * Create top-level packages/modules for:

     * `domain/`
     * `process/`
     * `agent/`
     * `uisurface/`
   * Move *only* type definitions and interfaces at first; keep implementations where they are.

2. **Introduce the four CRD kinds**

   * Define:

     * `Domain` CRD.
     * `Process` CRD.
     * `Agent` CRD.
     * `UISurface` CRD.
   * For now, fields are minimal: name, image, version, environment, references to infra.

3. **Backfill CRDs for existing deployments**

   * For each running service:

     * Create the appropriate `{Primitive} CRD` pointing to the existing image/tag.
     * Deploy CRDs side-by-side with current manifests.
   * Operators should reconcile CRDs **without** changing behavior yet (or run in “observe only” mode).

**Exit criteria:**

* Four CRD types live in the cluster and are applied through Git.
* Each important service is represented by exactly one primitive CRD.

---

### 6.2 Phase 1 – Code alignment: Domain primitives

**Goal:** Make domain services explicit Domain primitives.

1. **Create Domain primitive modules**

   * For each domain-ish service:

     * Add a `domain/<name>/` module.
     * Move or alias:

       * Entities and value objects.
       * Business rule functions.
       * Public API types.

2. **Tie Domain CRD to Domain primitive**

   * Ensure each Domain CRD has:

     * A `spec.primitive` field referencing the Domain primitive (e.g. by package name or logical id).
     * A reference to its database or schema.

3. **Standardise patterns across Domains**

   * Consistent:

     * Logging, tracing.
     * Error semantics.
     * Tenant/org scoping.

4. **Update docs & ADRs**

   * All references to “domain services” or legacy naming become **Domain primitive + Domain CRD** language.

**Exit criteria:**

* Each core business domain has:

  * A `domain/<name>` module.
  * A Domain CRD that is the authoritative deployment description.

---

### 6.3 Phase 2 – Code alignment: Process primitives

**Goal:** Elevate orchestration logic into explicit Process primitives.

1. **Identify candidate processes**

   * L2O, O2C, onboarding, transformation cycles, etc.
   * For each, list the domain interactions.

2. **Create Process primitive modules**

   * Add `process/<name>/` modules.
   * Move orchestration code here:

     * Sequences/calls across domains.
     * State models and transitions (if any, outside the engine).

3. **Tie Process CRDs to Process primitives**

   * For each Process primitive:

     * Create a Process CRD specifying:

       * Which Process primitive it runs.
       * How it is scheduled (queues/topics).
       * Timeouts, retry policies.

4. **Align UI**

   * Update UISurfaces to treat process instances as first-class:

     * Process boards, timelines, SLA views.

**Exit criteria:**

* Key flows run through named Process primitives, configured via Process CRDs.
* Orchestration logic is no longer hidden in ad-hoc application services.

---

### 6.4 Phase 3 – Code alignment: Agent primitives

**Goal:** Treat LLM workers as first-class Agent primitives.

1. **Create Agent primitive modules**

   * Add `agent/<name>/` modules.
   * Centralise:

     * Prompts and system instructions.
     * Tool definitions (typed wrappers over Domain/Process APIs).
     * Policies and risk tiers.

2. **Tie Agent CRDs to Agent primitives**

   * Agent CRD contains:

     * Reference to the Agent primitive.
     * Model + provider config.
     * Allowed tools (by Domain/Process).

3. **Refactor call sites**

   * Replace ad-hoc HTTP/gRPC calls to “some agent service” with:

     * A typed client for the specific Agent primitive.
   * This ensures:

     * Clear boundaries between deterministic code and AI-driven behavior.

**Exit criteria:**

* All AI helpers are catalogued as Agent primitives, with Agent CRDs and clear tool grants.

---

### 6.5 Phase 4 – Code alignment: UISurfaces

**Goal:** Make the UI surfaces explicit UISurface primitives.

1. **Identify UISurfaces**

   * Main platform app, plus any separate portals.
   * Group them by:

     * Process views.
     * Domain workspaces.

2. **Create UISurface primitive modules**

   * Add `uisurface/<name>/` modules.
   * Centralise:

     * Navigation structure.
     * Data hooks (SDK usage).
     * Authorization model.

3. **Introduce UISurface CRDs**

   * For each UISurface primitive:

     * Create UISurface CRD with:

       * Routing (paths, domains).
       * Required scopes/permissions.
       * References to the Domain/Process/Agent primitives it depends on.

4. **Align user journeys**

   * Ensure the “process-first, workspaces-second” experience is visible in the code structure:

     * Process views live under Process-oriented UISurfaces.
     * Entity tables/forms live under domain-oriented UISurfaces.

**Exit criteria:**

* Each major UI is represented by a UISurface primitive and CRD.
* Platform navigation is obviously aligned with Domain and Process primitives.

---

### 6.6 Phase 5 – Clean-up & de-legacy

After the primitives exist, we can remove older concepts.

1. **Remove legacy names from the codebase**

   * Delete or rename:

     * IPA terminology.
     * Old “Workflow” or “AgentRuntime” types that duplicate Process/Agent.
   * Update package paths and imports.

2. **Update documentation**

   * Replace:

     * Legacy diagrams and labels with Domain/Process/Agent/UISurface primitives.
   * Ensure 470–475 and related docs reference the new names consistently.

3. **Simplify CRDs**

   * Once operators are stable, review CRD schemas:

     * Remove fields that leak implementation detail.
     * Keep only what truly belongs in runtime configuration.

4. **Tighten boundaries**

   * Enforce:

     * “Domains don’t orchestrate long-running flows” (that’s Process).
     * “Agents never own durable state” (state belongs to Domain/Process).
     * “UISurfaces never bypass primitives to hit storage directly.”

---

## 7. Refactoring workstreams

To keep the plan manageable, organize work as parallel streams:

1. **Domain stream**

   * Enumerate domains.
   * Create Domain primitives and CRDs.
   * Standardize patterns.

2. **Process stream**

   * Prioritize key processes (Onboarding, L2O, O2C, Transformation).
   * Create Process primitives and CRDs.
   * Align UISurfaces and metrics.

3. **Agent stream**

   * Catalog existing agents.
   * Create Agent primitives and CRDs.
   * Harden policies and tooling.

4. **UISurface stream**

   * Catalog UI apps.
   * Create UISurface primitives and CRDs.
   * Align journeys with primitives.

5. **Ops & CRD stream**

   * Build operators and pipelines around the new CRDs.
   * Ensure Git is the source of truth for all CRDs.

6. **Documentation & language stream**

   * Sweep docs, diagrams, and onboarding materials.
   * Enforce the “four primitives + CRD” vocabulary everywhere.

---

## 8. Risks & mitigations

* **Risk:** Fragmented terminology persists in the codebase and docs.  
  **Mitigation:** Introduce a linter/checklist for naming, and make PR review enforce primitive + CRD vocabulary.

* **Risk:** Over-eager metadata creep into CRDs.  
  **Mitigation:** Aggressively review CRD schemas; any attempt to encode tables/fields/forms into CRDs must be treated as an architecture decision.

* **Risk:** AI tooling implicitly re-creates a metadata universe.  
  **Mitigation:** Keep AI agents working on **code + small CRDs**, not on a separate model. If an internal DSL appears, document why and how it stays small.

* **Risk:** Refactors stall due to fear of behavior changes.  
  **Mitigation:** Always start by **relabeling and wrapping** existing behavior as primitives, then only change internals with strong tests and metrics.

---

## 9. Success criteria

We consider this refactor “done enough” when:

1. **Every major service is a primitive**

   * Each has a clear classification: Domain, Process, Agent, or UISurface.
   * Each is configured via exactly one `{Primitive} CRD`.

2. **Code, docs, and CRDs share the same language**

   * No stray references to legacy controller names.
   * Internal onboarding materials explain the platform purely in terms of the four primitives.

3. **AI agents can evolve the system via primitives and CRDs**

   * Agents can:

     * Propose changes to Domain/Process/Agent/UISurface primitives in code.
     * Propose changes to corresponding CRDs for deployment.
   * There is no separate metadata layer they have to learn.

4. **We clearly differentiate from traditional metadata-driven systems**

   * Internally and externally we can say:

     * “Our model is code, not a giant metadata repository. Metadata and CRDs help us operate it, but they are not the application.”

---


<!-- ═══════════════════════════════════════════════════════════════════════════ -->
<!-- FILE: 475-ameide-domains.md -->
<!-- ═══════════════════════════════════════════════════════════════════════════ -->

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
> | [478-ameide-extensions.md](478-ameide-extensions.md) | Tenant primitive patterns |
>
> **Core Invariants**: See [470-ameide-vision.md §0 "Ameide Core Invariants"](470-ameide-vision.md) for the canonical list (four primitives, Graph read-only, Transformation as domain, proto chain, tenant isolation, Backstage internal).
>
> **Deployment Implementation**:
> - [461-ipc-idc-iac.md](461-ipc-idc-iac.md) – CRD operator patterns *(note: 461 uses deprecated IDC/IPC/IAC naming; use Domain/Process/Agent CRD per 470 glossary)*
> - [465-applicationset-architecture.md](465-applicationset-architecture.md) – GitOps deployment model
> - [464-chart-folder-alignment.md](464-chart-folder-alignment.md) – Chart folder structure by domain

> **Alignment Note**: This document describes domain architecture patterns
> that align with the design-time/runtime split defined in [471‑ameide‑business‑architecture](471-ameide-business-architecture.md):
>
> - **ProcessDefinitions** (design-time): BPMN-compliant artifacts from custom React Flow modeller, stored in **Transformation Domain primitive** (modelled via Transformation design tooling UIs)
> - **Process primitives** (runtime): Execute ProcessDefinitions, backed by Temporal
> - **AgentDefinitions** (design-time): Declarative agent specs stored in **Transformation Domain primitive** (modelled via Transformation design tooling UIs)
> - **Agent primitives** (runtime): Execute AgentDefinitions
>
> **Core Definitions** (see [470-ameide-vision.md §0](470-ameide-vision.md)):
> - **Transformation design tooling** is the set of modelling UIs—it does not own storage. All Transformation design tooling-originated artifacts are persisted by the **Transformation Domain primitive**.
> - **Graph** is a read-only knowledge projection; all writes go through primitives.
>
> For cross-references and gaps analysis, see §10 and §11.

---

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

---

## 4. Primitive Types per Domain

We use three runtime primitive types plus two design-time artifact types:

**Design-time (stored in Transformation Domain primitive, modelled via Transformation design tooling UIs)**:
* **ProcessDefinition** – BPMN-compliant artifact (from custom React Flow modeller) defining a process.
* **AgentDefinition** – Declarative spec for an agent (tools, policies, orchestration graph).

**Runtime (primitives)**:
* **Domain primitive** – owns data+rules for a particular domain.
* **Process primitive** – executes ProcessDefinitions; orchestrates multiple Domain primitives and Agent primitives to implement an E2E process. Backed by Temporal.
* **Agent primitive** – executes AgentDefinitions; non-deterministic LLM/tool automation that reads/writes via domain/process APIs.

**Extension hooks (Tier 1)**

Domain primitives and Process primitives may expose explicit extension points (e.g. `BeforePostInvoice`, BPMN Extension Tasks) which are satisfied by `ExtensionDefinition` artifacts executed via the shared `extensions-runtime` service. This gives tenants a light-weight way to customize behaviour without introducing new primitives for every rule change. See [479-ameide-extensibility-wasm.md](479-ameide-extensibility-wasm.md).

All primitives expose proto-based APIs and are consumed via the Ameide SDKs; this is uniform across all domains. At runtime each Domain/Process/Agent primitive is represented as a declarative CRD (Domain/Process/Agent) reconciled by Ameide operators (see [474-ameide-refactoring.md](474-ameide-refactoring.md) for the migration into this model).

### 4.1 Example: Platform & Identity

* **Domain primitives**

  * `TenantsController`, `OrganizationsController`, `UsersController`, `MembershipsController`, `RolesController`, `SubscriptionsController`. 
* **Process primitives**

  * `SelfServeOnboardingProcess`, `EnterpriseOnboardingProcess`, `SubscriptionLifecycleProcess`.

### 4.2 Transformation & Transformation design tooling

* **Domain primitives**

  * `TransformationController` (initiatives, metrics, stages).
  * `BacklogController` (epics, stories, tasks).
  * `ArtifactController` (Transformation design tooling artifacts & revisions, including ProcessDefinitions and AgentDefinitions).
* **Design-time artifacts owned by this domain**

  * ProcessDefinitions – BPMN-compliant process models.
  * AgentDefinitions – Declarative agent specs.
* **Process primitives**

  * `AgileDeliveryProcess`, `TOGAFADMProcess`, `ChangeRequestProcess`.

### 4.3 Product & Pricing Domains

**Domain primitives**

* `ProductController`

  * Product master, variants, attributes, product hierarchies.
* `PricingController`

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

  * `LeadsController`, `AccountsController`, `OpportunitiesController`, `ProductController`, `PricingController`, `ContractsController`, plus Identity.
* Orchestrates:

  * Lead capture → qualification → offer/quote → negotiation → contract → **hand‑over to O2C** via `OrdersController`.
* Agent primitives:

  * L2O‑specialised advisors, executing AgentDefinitions stored in Transformation Domain primitive (e.g. `tags: ["process:L2O", "domain:pricing"]`).

#### O2C (Order‑to‑Cash) Process primitive

* Uses domain primitives:

  * `OrdersController`, `FulfillmentController`, `InvoicingController`, `PaymentsController`, `ProductController`, `PricingController`.
* Orchestrates:

  * Order validation → fulfillment → billing → payments/collections.
* Again, O2C is “just another process” that composes existing domains.

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

Backstage templates (for Domain/Process/Agent/UISurface primitives) are **internal implementation details**; they live in the Application/Tech layer and are used by engineers and transformation agents to spin up new primitives, not by business users directly.

### 4.7 Tenant-Specific Controllers

Whenever possible, small, local customizations should attach to Tier 1 WASM extension hooks on existing primitives. The model below describes **Tier 2 primitive-based extensions** for requirements that justify new services—see [479-ameide-extensibility-wasm.md](479-ameide-extensibility-wasm.md) for how both tiers relate.

When tenants require custom primitives beyond the standard product, Backstage templates (or agents) create Domain/Process/Agent/UISurface CRDs plus repositories and GitOps manages those CRDs per environment:

* **Platform primitives** run in `ameide-{env}` (Shared SKU) or `tenant-{id}-{env}-base` (Namespace/Private SKU)
* **Tenant custom primitives** always run in `tenant-{id}-{env}-cust` namespaces
* Custom code is scaffolded via Backstage templates with tenant-specific parameters
* Code lives in `tenant-{id}-controllers` repos, manifests in `tenant-{id}-gitops` repos

> **See [478-ameide-extensions.md](478-ameide-extensions.md)** for the complete tenant extension model, namespace strategy by SKU, and the E2E flow from requirement to running primitives.

---

## 5. End‑to‑End Example: Generic E2E Process Pattern

To show that this is universal, take a generic **E2E Process X** (e.g. some future `Idea‑to‑Launch` or `Case‑to‑Resolution` process):

1. **Transformation domain (design-time)**

   * Stakeholders describe the process using Transformation design tooling artifacts (ProcessDefinitions, docs, constraints) under the Transformation domain.
   * ProcessDefinitions are created using the custom React Flow modeller.
   * AgentDefinitions are created for any agent assistance needed.

2. **Design → Runtime deployment**

   * Transformation processes (Agile/TOGAF) lead to formal ProcessDefinitions + requirements.
   * A Process primitive `ProcessX` is deployed (executes ProcessDefinitions via Temporal).

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

    * shows *process‑centric views* (e.g. “L2O pipeline”, “O2C flow board”), and
    * *domain‑centric workspaces* (e.g. “Product catalog”, “Pricing rules”, “Orders”).

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
| Process primitive | Workflow (305) | Runtime that executes ProcessDefinitions |
| ProcessDefinition | BPMN model (deprecated) | Design-time artifact in Transformation Domain primitive |
| Agent primitive | AgentRuntime (deprecated) | Runtime that executes AgentDefinitions |
| AgentDefinition | Agent config (deprecated) | Design-time artifact in Transformation Domain primitive |

**Clarification**: AgentDefinitions are design-time artifacts owned by **Transformation Domain primitive** (§3.2), modelled via Transformation design tooling UIs. There is no separate "Transformation design tooling service"—Transformation design tooling is the UI layer that calls Transformation APIs. Agent primitives are runtime components that execute AgentDefinitions.


<!-- ═══════════════════════════════════════════════════════════════════════════ -->
<!-- FILE: 476-ameide-security-trust.md -->
<!-- ═══════════════════════════════════════════════════════════════════════════ -->

Totally agree — we can't keep evolving Ameide without locking down the security story in parallel.

> **Cross-References (Vision Suite)**:
>
> | Document | Purpose |
> |----------|---------|
> | [470-ameide-vision.md](470-ameide-vision.md) | Vision and principles |
> | [471-ameide-business-architecture.md](471-ameide-business-architecture.md) | Business concepts |
> | [472-ameide-information-application.md](472-ameide-information-application.md) | Application architecture |
> | [473-ameide-technology.md](473-ameide-technology.md) | Technology stack |
> | [475-ameide-domains.md](475-ameide-domains.md) | Domain portfolio |
> | [478-ameide-extensions.md](478-ameide-extensions.md) | Extension security model |
> | [479-ameide-extensibility-wasm.md](479-ameide-extensibility-wasm.md) | Tier 1 WASM + Tier 2 extension model |
> | [480-ameide-extensibility-wasm-service.md](480-ameide-extensibility-wasm-service.md) | Shared WASM runtime implementation |
>
> **Core Invariants**: See [470-ameide-vision.md §0 "Ameide Core Invariants"](470-ameide-vision.md) for the canonical list (four primitives, Graph read-only, Transformation as domain, proto chain, tenant isolation, Backstage internal).
>
> **Secrets & Security Implementation**:
> - [462-secrets-origin-classification.md](462-secrets-origin-classification.md) – Secret origin taxonomy
> - [451-secrets-management.md](451-secrets-management.md) – Secrets flow (Azure KV → Vault → K8s)
> - [426-keycloak-config-map.md](426-keycloak-config-map.md) – OIDC client patterns
> - [340-api-routes.md](340-api-routes.md) – API authentication
> - [341-middleware-auth.md](341-middleware-auth.md) – Middleware AuthN/AuthZ
> - [342-token.md](342-token.md) – Token handling

Here's a first **Security & Trust Architecture** pass that plugs into the 6-doc set you already have (Vision, Business, App/Info, Tech, Domains, Refactoring) and reuses as much of the existing backlog work as possible.

---

## 1. Purpose & Scope

This layer answers:

> “Given Ameide is a multi‑tenant, agentic, process‑driven business platform, how do we **stop tenants, agents, and components from hurting each other**?”

Scope:

* Cross‑cutting security principles and threat model
* How security attaches to:

  * Tenancy / identity / onboarding
  * Domains & Process primitives
  * Agents
  * Backstage + primitive templates
  * APIs & SDKs
  * Infra (secrets, network, cluster)
* What we should do *next* (concrete backlog slices)

It doesn’t replace the existing onboarding/security work — it builds on it. 

---

## 2. Security Principles (Ameide‑specific)

Let’s make a very opinionated short list:

1. **Tenant isolation first, everything else second**

   * Every request, event, and row is scoped by `tenant_id` (and often `organization_id`).
   * DB‑level RLS + JWT tenant claims are *non‑negotiable* guardrails. 

2. **Deterministic vs non‑deterministic boundary**

   * **Domain primitives & Process primitives**: deterministic, auditable, replayable, no LLM calls inside.
   * **Agents**: non‑deterministic; they *propose* actions and edit artifacts, but must go through deterministic primitives to persist state.

3. **Proto‑first, zero trust between services**

   * Every RPC is proto‑based and goes through:

     * AuthN (JWT/mTLS)
     * AuthZ (role/tenant checks)
     * Input validation
   * No “trusted internal” bypasses; even internal calls use the same RPC contracts.
   * Proto definitions live in `packages/ameide_core_proto` (Buf-managed) and propagate through workspace SDKs into GitOps manifests per [472 §2.7](472-ameide-information-application.md#27-contract--implementation--runtime-chain); Go services additionally inherit Watermill’s middleware (logging/metrics/retries) when handling commands/events (see [472 §3.3.1](472-ameide-information-application.md#331-event-driven-cqrs-runtime-watermill)).

4. **Immutable artifacts, append‑only history**

   * Transformation design tooling, BPMN, ArchiMate, Markdown, Backstage templates: always versioned, never overwritten.
   * Security decisions (approvals, waivers, promotions) are events in that history.

5. **Defence in depth, but with clear responsibility**

   * Infra (cluster, network, secrets)
   * Platform (identity, RBAC, tenancy)
   * Domains & Processes
   * Agents
   * UI & UX
     Each layer has its own “owner” and tests — no single magic firewall.

6. **Secure‑by‑template, not secure‑by‑discipline**

   * If a Backstage template can generate an insecure service, that’s a bug.
   * Base templates *must* wire RLS, AuthN/ AuthZ interceptors, metrics, and secrets usage correctly.

---

## 3. Threat Model & Trust Zones

### 3.1 Threats (simplified)

* **Tenant escape**

  * Data leakage across tenants (DB, cache, events, logs)
* **Agent abuse / prompt injection**

  * LLM‑based agents executing dangerous code or leaking secrets across tenants. 
* **Supply chain / code‑gen risk**

  * Agents (like core‑platform‑coder) generating insecure code or misconfiguring infra. 
* **Identity & onboarding attacks**

  * Account takeovers, invitation abuse, SSO misconfig, SCIM mis‑provisioning. 
* **API misuse**

  * Over-privileged service accounts, missing rate limits, poisoning inputs into BPMN/Transformation design tooling/Backstage.
* **Tier 1 WASM extensions**

  * Sandboxed tenant logic running inside the shared `extensions-runtime` service; risks include sandbox escape, over-permissive host calls, and resource exhaustion.
* **Controller spec tampering**

  * Malicious or accidental edits to `IntelligentDomain primitive`, `IntelligentProcess primitive`, or `IntelligentAgent primitive` CRs (wrong namespace, over-privileged images, disabled probes) or compromised operators that reconcile them.

### 3.2 Trust zones

Roughly:

1. **Public edge** (Next.js UIs, public docs, marketing)
2. **Tenant application plane** (Platform UI, Domain & Process APIs)
3. **Agent & Transformation plane** (Agents, Transformation design tooling, Backstage, IPA legacy, shared `extensions-runtime`)
4. **Control & infra plane** (Argo, Vault, Keycloak, CNPG, K8s)

Each zone gets progressively fewer humans and more automation; crossing zones always goes through authenticated RPCs or well‑defined APIs.

---

## 4. Identity, Tenancy & Access Control

We already have a solid foundation in the onboarding backlog — let’s elevate it to “security baseline” instead of just “feature behaviour.” 

### 4.1 Tenancy model

From the consolidated onboarding spec:

* **Two‑level tenancy**:

  * `tenant` (infra isolation)
  * `organization` (user workspace within tenant)
* **Realm‑per‑tenant (enterprise) + shared realm (SMB)** Keycloak model to balance isolation vs scalability. 

Security consequences:

* All JWTs carry **`tenantId` claim**; services **must fail‑secure** when it is missing. 
* All platform tables (and later, domain schemas) enforce **RLS by tenant**; no ad‑hoc tenant filtering in application code.
* Multi‑cluster / shard decisions are expressed as `tenant.keycloak_shard_id`, `tenant.database_plan`, etc., and enforced by operators, not random code. 

### 4.2 AuthN/AuthZ

* **Keycloak** for OIDC; realm‑per‑tenant for big customers, shared for SMB.
* **RBAC** with canonical roles (`admin`, `contributor`, `viewer`, `guest`, `service`). Seeded at org creation.
* **SCIM + SSO + JIT** with clear precedence:

  * SCIM > JIT > manual invites, as already defined. 
* **Service‑to‑service**:

  * mTLS or signed JWTs between system components; in TS/Go/Python SDKs, this is hidden behind `AmeideClient` configuration. 

---

## 5. Data Security & Multi‑Tenancy in Domains

Security for **Domain primitives** is mostly about data isolation and integrity.

### 5.1 Data classification

* **Transactional business data**

  * Lives in domain DBs (Orders, Product, Pricing, etc.).
  * Multi‑tenant via RLS, plus optional “noisy” tenant isolation (separate DB / schema per plan).
* **Knowledge graph projections**

  * Copy‑on‑read from domains to Graph/Repository for analysis; still tagged by `tenant_id`.
* **Artifacts (BPMN, diagrams, Markdown, agent specs)**

  * Stored by the **Transformation (and other) Domain primitives**, often edited via Transformation design tooling UIs.
  * Event-sourced (internally by Transformation), versioned, immutable; enriched with metadata and promotion state. 
  * `ExtensionDefinition` sources and compiled WASM blobs live in tenant-tagged MinIO prefixes with the same governance and promotion metadata as other Transformation design tooling artifacts; modules are treated strictly as data owned by Transformation.

Every Domain primitive must:

* Enforce **tenant + org + role checks** in its service methods (via shared interceptors).
* Use dedicated **DB roles/credentials** provided by CNPG + ExternalSecrets; no shared "superuser" in app code.
* Project to Graph only via **explicit configuration** — no "random joins across tenants".

> **Note**: The Knowledge Graph holds read-only projections of selected domain/process/transformation data; RLS and tenant isolation still apply, but Graph is never the authoritative system of record (see [470-ameide-vision.md §0](470-ameide-vision.md)).

### 5.2 Encryption & secrecy

* At rest: DB, object storage, and logs encrypted (KMS‑backed).
* In transit: TLS everywhere (including internal service mesh/gateway).
* Secrets: Vault + ExternalSecrets as already standardized; CNPG remains the owner of DB credentials. 

---

## 6. APIs & SDKs: Security as a Contract

The proto/API backlogs already define the right building blocks; we just declare them “mandatory”.

**Baseline:**

* All proto services use:

  * Unified **AuthProvider** (JWT/OIDC & API key)
  * **ErrorMapper** (translates domain errors to safe HTTP/gRPC codes)
  * **RequestContext** DTO with user/tenant/trace IDs. 
* Interceptors for:

  * AuthN/AuthZ
  * Request validation
  * Rate limiting
  * Tracing & correlation IDs.

**SDK side (TS/Go/Python):**

* `AmeideClient` automatically attaches tokens, tenant headers, retries, and tracing metadata.
* For browser clients, CORS is locked down to platform UIs; no generic cross‑origin wildcards.

### 6.1 Custom Code Isolation Invariant

Tenant extensions follow a strict carve-out:

> * No tenant-owned **services** may run in shared `ameide-*` namespaces; all tenant primitives live in `tenant-{id}-{env}-cust` (and `tenant-{id}-{env}-base` for Namespace/Private SKUs).
> * The only tenant-specific logic that executes in `ameide-*` is Tier 1 WASM modules, treated as data and run inside the platform-owned `extensions-runtime` service under strict sandbox and host-call policies (see 479, 480).

This is enforced at multiple layers:

| Layer | Enforcement |
|-------|-------------|
| **Transformation** | Validates tenant SKU and computes target namespace before scaffolding |
| **Backstage Templates** | Namespace calculation logic enforces isolation rules |
| **GitOps Policy** | ArgoCD ApplicationSets validate namespace targeting |
| **Wasm runtime** | Enforces limits + allowlists per extension and treats modules as untrusted data |
| **NetworkPolicy** | Denies ingress from `tenant-*-cust` to `ameide-*` internal services |

Namespace topology by SKU:

| SKU | Product Runtime | Custom Code |
|-----|-----------------|-------------|
| Shared | `ameide-{env}` | `tenant-{id}-{env}-cust` |
| Namespace | `tenant-{id}-{env}-base` | `tenant-{id}-{env}-cust` |
| Private | `tenant-{id}-{env}-base` | `tenant-{id}-{env}-cust` |

> **See [478-ameide-extensions.md](478-ameide-extensions.md)** for the complete namespace strategy and [479](479-ameide-extensibility-wasm.md)/[480](480-ameide-extensibility-wasm-service.md) for the Tier 1 runtime model.

---

## 7. Security Metamodel: ERP-Class Authorization Model

This section provides ERP architects with a familiar vocabulary that maps Ameide's security model to D365FO and SAP S/4HANA concepts. For detailed ERP-specific comparisons, see [470a-ameide-vision-vs-d365.md §6.5](470a-ameide-vision-vs-d365.md) and [470b-ameide-vision-vs-saps4.md §6.5](470b-ameide-vision-vs-saps4.md).

### 7.1 Core Concepts

Ameide defines four authorization artifact types that map directly to traditional ERP security constructs:

| Ameide Concept | Purpose | D365FO Equivalent | S/4HANA Equivalent |
|----------------|---------|-------------------|-------------------|
| **AuthorizationDescriptor** | Binds operation + dimensions + scope + risk tier | Privilege | Authorization Object |
| **Duty** | Aggregates descriptors into functional groupings | Duty | PFCG Role (partial) |
| **AuthorizationDimension** | Defines RLS constraint pattern (column + claim + policy) | XDS Security Policy | Org Level |
| **SoDRule** | Defines conflicting descriptor combinations | Segregation of Duties Rule | GRC SoD Matrix |

These concepts are **design-time artifacts** stored in the **Transformation Domain** alongside ProcessDefinitions and AgentDefinitions. They map onto existing Ameide runtime constructs:

| Design-Time Artifact | Runtime Implementation |
|---------------------|----------------------|
| AuthorizationDescriptor | `AuthRequirement` in `common/v1/annotations.proto` (method-level) |
| AuthorizationDimension | RLS policies + JWT claims (`tenant_id`, `organization_id`, `ext.*`) |
| Duty | `OrganizationRole.permissions` map in `platform/v1/roles.proto` |
| SoDRule | Violation checks at role assignment time (conceptual) |

### 7.2 AuthorizationDescriptors

An **AuthorizationDescriptor** bundles:

* **Target**: Proto service + method, or process stage + action
* **Required dimensions**: Which authorization dimensions apply (e.g., `company_code`, `plant`)
* **Scope**: READ / WRITE / DELETE / ADMIN
* **Risk tier**: LOW / MEDIUM / HIGH / CRITICAL

```yaml
# Conceptual example (design-time artifact in Transformation Domain)
code: INVOICE_APPROVE
name: Approve Invoice for Payment
target:
  proto_service: ameide_core_proto.finance.v1.InvoiceService
  proto_method: ApproveInvoice
required_dimensions: [company_code]
scope: WRITE
risk_tier: HIGH
erp_hints:
  d365_privilege: InvoiceApprove
  d365_duty: InvoicePaymentProcessing
  s4_auth_object: F_BKPF_BED
  s4_activity: "02"
```

The `erp_hints` field provides explicit cross-references to D365/S4 concepts for migration documentation and architect familiarity.

### 7.3 AuthorizationDimensions

An **AuthorizationDimension** defines how a particular scope (company, plant, etc.) is enforced at the database level:

| Dimension | Type | Column | RLS Expression | JWT Claim |
|-----------|------|--------|----------------|-----------|
| `tenant` | TENANT | `tenant_id` | `tenant_id = current_setting('app.tenant_id')` | `tenant_id` (always) |
| `organization` | ORGANIZATION | `organization_id` | `organization_id = current_setting('app.organization_id')` | `organization_id` |
| `company_code` | COMPANY | `company_code` | `company_code = ANY(string_to_array(current_setting('app.company_codes'), ','))` | `ext.company_codes` |
| `plant` | PLANT | `plant_id` | `plant_id = ANY(string_to_array(current_setting('app.plants'), ','))` | `ext.plants` |
| `sales_org` | SALES_ORG | `sales_org` | `sales_org = ANY(...)` | `ext.sales_orgs` |

**Key differences from D365/S4:**

* **Explicit columns**: Dimensions are explicit columns on domain tables, not DDIC virtual fields or CDS annotations
* **RLS-enforced**: Postgres RLS policies enforce dimensions at the database layer, not application logic
* **JWT-carried**: Dimension values are carried in JWT claims, validated at API entry and applied to DB sessions

### 7.4 Duties

A **Duty** aggregates multiple AuthorizationDescriptors into a functional grouping:

```yaml
# Conceptual example
code: DUTY_INVOICE_PROCESSING
name: Invoice Processing
description: Full access to invoice lifecycle
authorization_descriptors:
  - INVOICE_CREATE
  - INVOICE_UPDATE
  - INVOICE_APPROVE
  - INVOICE_POST
functional_area: FINANCE
risk_tier: HIGH  # Inherited from highest-risk descriptor
```

Duties provide the same abstraction layer as D365 Duties, allowing:

* **Role composition**: Roles reference duties, not individual descriptors
* **Functional grouping**: Related operations are managed together
* **Risk aggregation**: Duty risk tier is derived from its most sensitive descriptor

### 7.5 Separation of Duties (SoD) Rules

An **SoDRule** defines conflicting privilege combinations that should never be held by the same user:

```yaml
# Conceptual example
code: SOD_PAY_001
name: Payment Creation vs Approval
description: Users who create payments should not approve them
conflicting_descriptors_a:
  - PAYMENT_CREATE
conflicting_descriptors_b:
  - PAYMENT_APPROVE
severity: CRITICAL
enforcement: BLOCK  # or WARN, MITIGATE
compensating_controls: "Requires manager override with documented justification"
control_reference: SOX-404-3.1
```

**Enforcement options:**

| Enforcement | Behavior |
|-------------|----------|
| WARN | Log violation, allow assignment |
| BLOCK | Prevent role assignment containing conflict |
| MITIGATE | Require compensating control acknowledgment |

SoD rules are evaluated **at role assignment time**, not just in periodic batch reports (as in S/4 GRC). This provides real-time enforcement rather than after-the-fact detection.

### 7.6 Enforcement Points

Authorization is enforced at multiple layers in the Ameide stack:

| Layer | Mechanism | Current State |
|-------|-----------|---------------|
| **Proto Method** | `AuthRequirement` option in `common/v1/annotations.proto` | Exists ✅ |
| **SDK Interceptor** | `AmeideClient` validates roles/permissions before RPC | Exists ✅ |
| **API Middleware** | `withAuth()` validates session + membership + role | Exists ✅ |
| **Database** | RLS policies enforce dimensions per session | Design complete, implementation gap (§13.2) |
| **Role Assignment** | SoD violation check at duty/role grant | Conceptual (future) |
| **UI** | Effective authorizations determine action visibility | Via SDK + role checks |

### 7.7 Ameide Advantages vs D365/S4

Ameide's security model provides several advantages over traditional ERP security:

| Aspect | D365/S4 Approach | Ameide Approach |
|--------|------------------|-----------------|
| **Tenant isolation** | Configuration + middleware | **Hard invariant** (RLS + JWT claims, non-negotiable) |
| **Extension isolation** | Trust extensions in core namespace | **Namespace isolation** (custom code never in `ameide-*`) |
| **Agent governance** | N/A (no AI agents) | **Explicit scope/risk_tier** on AgentDefinitions |
| **Code-gen security** | N/A | **ControllerImplementationDraft pattern** (agents propose, GitOps applies) |
| **SoD enforcement** | Batch reports (GRC) | **Real-time** at role assignment |
| **Security as code** | Metadata trees (AOT/DDIC) | **Proto + RLS** (version-controlled, testable) |
| **Dimension enforcement** | Application logic (XDS filters) | **Database-level RLS** (cannot be bypassed) |

---

## 8. Agents & AI Safety

Agents are where things can go very wrong, so the architecture must be explicit:

### 8.1 AgentDefinition governance

> **Core Definitions** (see [470-ameide-vision.md §0](470-ameide-vision.md)):
> - **AgentDefinitions** are stored in the **Transformation Domain primitive** (modelled via Transformation design tooling UIs).
> - There is no separate "Transformation design tooling service" in the runtime—Transformation design tooling is the UI layer that calls Transformation APIs.

* **AgentDefinitions** (declarative specs for tools, orchestration graphs, policies) live in the **Transformation Domain primitive** and are versioned like ProcessDefinitions.
* Each AgentDefinition has:

  * `domains: [...]` and `processes: [...]` tags (for routing & UI only)
  * `scope`: read‑only vs read/write vs code‑gen
  * `risk_tier`: e.g. `low`, `medium`, `high`
* "Dangerous" capabilities (file system access, k8s access, direct SQL) are only exposed to **high‑tier, non‑default AgentDefinitions**, and always wrapped by deterministic tools.

### 8.2 Deterministic boundaries

* Domain primitives and Process primitives **never call raw LLM APIs**; they invoke Agent primitives which:

  * Execute AgentDefinitions loaded from the **Transformation Domain primitive**
  * Log prompts & results
  * Enforce rate limits and timeouts
  * Apply basic content filters (PII, secret‑like patterns).

### 8.3 Agent‑generated code & configs

The `core-platform-coder` review already flags security concerns: CLI fragility, prompt size, and raw shell argument handling.

We generalize:

* Any Agent primitive that writes code, Helm values, or Backstage configs must:

  * Propose changes as **artifacts in Transformation design tooling or Git**, not apply them directly.
  * Pass through:

    * Static checks (lint, tests, security scanning)
    * Human or policy‑based approval
    * GitOps pipeline for deployment.
* Transformation Domain primitive logs "write intents" with full diff and user/tenant context.

#### ControllerImplementationDraft Pattern

Per [478-ameide-extensions.md](478-ameide-extensions.md) §6, agents creating new primitives use a controlled workflow:

1. **Agent never gets Git or K8s credentials** directly
2. Agent uses `GetControllerDraftFiles(draft_id)` to read current state
3. Agent uses `UpdateControllerDraftFiles(draft_id, patches)` to propose changes
4. A **deterministic worker** applies patches, runs checks, and commits
5. `RequestPromotion(draft_id, env)` triggers CI and GitOps deployment

This pattern ensures all agent-generated code passes through the same governance as human-written code.

### 8.4 WASM extensions & host calls

Host calls from Tier 1 WASM modules inherit the caller’s `ExecutionContext` (tenant/org/user/risk tier) and must route through existing Ameide APIs via allowlisted SDK adapters. Risk tier plus `extensions-runtime` policy determines which host calls are exposed (`low` often has none, `medium` read-only, `high` curated write helpers). This is the enforcement point for the carve-out described in §6.1 and the reason Tier 1 logic can safely run in `ameide-*`.

---

## 9. Backstage, Controllers & GitOps

Backstage is now our internal factory — which means it’s also a high‑risk surface.

### 9.1 Backstage security

* Only **Ameide engineers + transformation agents** have access, via SSO and RBAC. No tenant users.
* Templates are treated as code:

  * Stored in Git
  * Reviewed, tested
  * Versioned and rolled out via Argo.

### 9.2 Secure‑by‑template primitive generation

Each template (Domain primitive, Process primitive, Agent) bakes in:

* Proper `AmeideClient` usage with AuthN/AuthZ interceptors.
* Per‑service DB role & ExternalSecret wiring; no root credentials. 
* Default network policies (deny‑all except required dependencies).
* OTel tracing and structured logging with tenant and user correlation fields.
* Rate limits and timeouts based on proto budgets (e.g. p95 <100ms for simple operations). 

---

## 10. Observability, Audit & Transformation design tooling

Security without observability is vibes. We already have strong foundations:

* **Onboarding**:

  * Structured audit events for invites, SSO, SCIM, subscription changes. 
* **Transformation Domain primitive / Transformation design tooling**:

  * Event‑sourced commands + snapshots (within Transformation), promotion flows, and secret scan validator.
  * Secret scanning and promotion gates run inside the **Transformation Domain primitive's** artifact lifecycle; Transformation design tooling UI just triggers those flows. 
* **North‑Star modeling**:

  * Immutable designs and deployments with audit logs per revision. 

We extend:

* Every Domain/Process/Agent primitive emits:

  * audit events with `tenant_id`, `user_id`, `resource_type`, `resource_id`.
* Transformation Domain primitive's secret scanner runs on artifacts (BPMN, diagrams, Markdown, Backstage templates) and **blocks promotion** if secrets are detected, with override via signed waiver. 
* GitOps + operators record **Domain/Process/Agent CRD spec changes** (who/what changed the CR, diffs, resulting operator actions) in audit streams so we can trace primitive configuration drift or tampering.

---

## 11. Immediate Next Steps (Security Backlog Slice)

Here’s how I’d start making this real without boiling the ocean:

1. **Codify security principles & zones**

   * Add a short “Security & Trust” section into the Vision + Tech docs, basically the principles from sections 2–3.

2. **Lock down Domain primitive base template**

   * In Backstage:

     * Inject `AmeideClient` with auth interceptors.
     * Enforce `tenant_id` in all queries (DB and Graph).
     * Wire in OTel tracing and error mapping by default. 

3. **AgentDefinition governance v1**

   * Add `scope` + `risk_tier` to AgentDefinitions in Transformation Domain primitive (modelled via Transformation design tooling UIs).
   * Wrap any code‑gen / infra‑gen in Transformation artifacts and GitOps flows, not direct writes.

4. **Transformation secret‑scan + promotion gate**

   * Implement the MVP secret scanner + promotion endpoint inside the Transformation Domain primitive and wire it into Transformation workflows. 

5. **API security posture check**

   * Verify at least one “golden path” Domain primitive + Process primitive (e.g. L2O) uses:

     * proto‑based APIs
     * SDK interceptors
     * RLS + tenant claims end‑to‑end.

From there, we can expand into more formal things (STRIDE‑style threat modeling, formal policies, pen‑tests), but these five steps will already embed security into the Ameide story instead of bolting it on later.

If you want, next I can propose a very small **"Security RFC 000‑Ameide"** skeleton you can drop into the repo and iterate on with the team.

---

## 12. Cross‑References

This Security & Trust Architecture should be read with the following documents:

| Document | Relationship | Alignment |
|----------|-------------|-----------|
| [470‑ameide‑vision](470-ameide-vision.md) | Parent vision & principles | Strong ✅ |
| [471‑ameide‑business‑architecture](471-ameide-business-architecture.md) | Business use cases, personas | Strong ✅ |
| [472‑ameide‑information‑application](472-ameide-information-application.md) | Application & information architecture | Strong ✅ |
| [473‑ameide‑technology](473-ameide-technology.md) | Technology implementation | Strong ✅ |
| [475‑ameide‑domains](475-ameide-domains.md) | Domain portfolio & patterns | Strong ✅ |
| [322‑rbac](322-rbac.md) | RBAC implementation & testing | See §12.1 |
| [329‑authz](329-authz.md) | Authorization implementation | See §12.2 |
| [333‑realms](333-realms.md) | Realm‑per‑tenant model | Strong ✅ |
| [451‑secrets‑management](451-secrets-management.md) | Secrets authority model | Strong ✅ |
| [462‑secrets‑origin‑classification](462-secrets-origin-classification.md) | Secrets classification | Strong ✅ |
| [310‑agents‑v2](310-agents-v2.md) | Agent domain implementation | See §12.3 |
| [388‑ameide‑sdks‑north‑star](388-ameide-sdks-north-star.md) | SDK publish strategy | See §12.4 |
| [478‑ameide‑extensions](478-ameide-extensions.md) | Tier 2 primitive security model | See §12.5 |
| [479‑ameide‑extensibility‑wasm](479-ameide-extensibility-wasm.md) | Tier 1 + Tier 2 extensibility framing | See §12.5 |
| [480‑ameide‑extensibility‑wasm‑service](480-ameide-extensibility-wasm-service.md) | Shared WASM runtime | See §12.5 |
| [470a‑ameide‑vision‑vs‑d365](470a-ameide-vision-vs-d365.md) | D365 security mapping | See §7 |
| [470b‑ameide‑vision‑vs‑saps4](470b-ameide-vision-vs-saps4.md) | S/4HANA security mapping | See §7 |

### 12.1 RBAC alignment (322)

* 476 §4.2 requires RBAC with canonical roles (`admin`, `contributor`, `viewer`, `guest`, `service`)
* 322 confirms: ✅ Canonical role catalog wired end-to-end (Keycloak, proto/SDK, database seeds)
* 322 confirms: ✅ Ownership enforcement consistent for transformations, repositories, teams
* 322 confirms: ✅ Realm-per-tenant architecture update in progress (moving from prefixed to realm-native roles)

**Gaps identified in 322 that impact 476 requirements:**
- ❌ API routes do NOT enforce 403 for insufficient permissions (476 §6 requires interceptors)
- ❌ Server actions have no authorization checks
- ❌ DAL with tenant scoping not implemented (476 §5.1 requires tenant + org + role checks)
- ❌ RLS policies not enabled (476 §2.1 requires RLS as "non-negotiable")
- ❌ Audit logging for denied attempts not implemented (476 §10 requires audit events)

**322 next activities align with 476:**
1. Land owner/role checks for org membership mutations → supports §5.1
2. Wire 401/403 telemetry into dashboards → supports §10
3. Fold API integration suites into CI gating → supports §6

### 12.2 Authorization alignment (329)

* 476 §2.1 requires: "DB‑level RLS + JWT tenant claims are non‑negotiable guardrails"
* 476 §4.1 requires: "All platform tables enforce RLS by tenant"
* 476 §5.1 requires: "Enforce tenant + org + role checks in service methods"

**Current status per 329-authz:**
- ✅ **Phase 1 (API Authorization Layer)**: `withAuth` middleware exists, applied to most routes
- ❌ **Phase 2 (Database Organization Scoping)**: NOT STARTED
  * `organization_id` columns missing on business tables (repositories, transformations, elements)
  * Data scoped only by `tenant_id`, not `organization_id`
- ❌ **Phase 3 (Defense in Depth)**: NOT STARTED
  * RLS policies not created
  * Audit logging service not implemented

**Impact on 476 compliance:**
- §2.1 principle (tenant isolation first): ❌ NOT MET at DB level
- §5.1 requirement (tenant + org + role checks): ⚠️ API-level only, not DB-enforced

**Required actions per 329:**
1. Create Flyway migrations for `organization_id` columns + FK constraints
2. Update proto definitions to include `organization_id`
3. Enable RLS policies with session variables
4. Implement audit logging service

### 12.3 Agents alignment (310)

* 476 §8.1 requires AgentDefinitions to have `scope` (read-only vs read/write vs code-gen) and `risk_tier` (low, medium, high)
* 476 §8.2 requires deterministic boundaries (Domain primitives/Process primitives invoke Agent primitives, never raw LLM APIs)
* 476 §8.3 requires agent-generated code to propose via Transformation design tooling/Git artifacts, not direct writes

**Terminology alignment:**
* AgentDefinitions = design-time specs stored in **Transformation Domain primitive** (modelled via Transformation design tooling UIs) (§8.1)
* Agent primitives = runtime that executes AgentDefinitions (§8.2)
* Note: There is no separate "Transformation design tooling service"—Transformation design tooling is the UI layer that calls Transformation APIs

**Current status per 310-agents-v2:**
- ✅ AgentInstance proto exists with basic metadata (node_id, version, parameters, tools)
- ❌ **Gap**: No `scope` field on AgentDefinition/AgentInstance message
- ❌ **Gap**: No `risk_tier` field on AgentDefinition/AgentInstance message
- ⚠️ **Gap**: `agents.tool_grants` schema exists but service does NOT expose grant/revoke APIs
- ❌ **Gap**: No enforcement of grants at publish time
- ❌ **Gap**: "Write intents" logging (§8.3) not implemented

**Impact on 476 compliance:**
- §8.1 (AgentDefinition governance): ❌ NOT MET - cannot differentiate safe agents from dangerous ones
- §8.3 (agent-generated code governance): ❌ NOT MET - awaiting Transformation design tooling promotion flow

### 12.4 SDK alignment (388)

* 476 §6 requires SDKs to "automatically attach tokens, tenant headers, retries, and tracing metadata"
* 476 §6 requires `AmeideClient` as unified client contract

**Current status per 388-ameide-sdks-north-star:**
| SDK | Status | Gap |
|-----|--------|-----|
| Go SDK | ✅ Published | `v0.1.0` on GitHub + GHCR |
| Python SDK | ⚠️ Published | PyPI has `2.10.1`, but GHCR lacks SemVer aliases |
| **TypeScript SDK** | ❌ **NOT PUBLISHED** | 404 on npmjs; GHCR has only `dev`/hash tags |
| Configuration | ⚠️ Dev-only | GHCR has only `dev`/hash tags |

**Impact on 476 compliance:**
- §6 (AmeideClient requirement): ⚠️ PARTIAL - TS consumers cannot use secure SDK patterns until TS SDK is published
- "Open by design" principle: ❌ Compromised for external TS consumers

### 12.5 Extension alignment (478/479/480)

478 defines the security model for Tier 2 primitive-based extensions, while 479 pairs that with Tier 1 WASM hooks and 480 implements the shared runtime:

* **§6.1 Custom Code Isolation Invariant**: Aligns with 478 §8.1 (No custom code in shared namespaces)
* **§8.3 ControllerImplementationDraft Pattern**: Operationalizes 478 §6 (E2E flow)
* **Namespace topology**: 478 §4 provides the SKU-based namespace strategy enforced by §6.1

**Key security invariants from 478**:

| Invariant | 476 Enforcement |
|-----------|-----------------|
| No custom services in `ameide-*` | §6.1 namespace isolation |
| Tier 1 WASM runs as data in platform runtime | §6.1 carve-out + §7.4 host-call policies |
| Agents don't touch Git directly | §8.3 draft pattern |
| All calls via tenant-aware auth | §6 API security contract |
| NetworkPolicies isolate tenant code | §6.1 + §9.2 |

---

## 13. Notable Gaps & Issues

### 13.1 Terminology alignment

| 476 Term | Related Terms | Notes |
|----------|---------------|-------|
| Trust zones (§3.2) | N/A | 476-specific concept |
| Tenant application plane | Domain layer (475) | Consistent ✅ |
| Agent & Transformation plane | Transformation domain (475) | Consistent ✅ |
| Control & infra plane | Foundation layer (465) | Consistent ✅ |

### 13.2 Implementation gaps

| Gap | Description | Impact | Related Docs |
|-----|-------------|--------|--------------|
| **RLS not enabled** | §2.1, §4.1, §5.1 require DB-level RLS | Tenant isolation principle NOT MET | [329-authz](329-authz.md) ⚠️ CRITICAL |
| **Org-level scoping missing** | §5.1 requires tenant + org + role checks | Cross-org data leakage possible within tenant | [329-authz](329-authz.md) Phase 2 |
| **API routes lack 403 enforcement** | §6 requires AuthZ interceptors | Insufficient permission not rejected | [322-rbac](322-rbac.md) |
| **AgentDefinition scope/risk_tier** | §8.1 requires capability classification | No enforcement of dangerous capability restrictions | [310-agents-v2](310-agents-v2.md) |
| **Write intents logging** | §8.3 requires agent diff logging | No audit trail for agent-generated changes | — |
| **Transformation secret scanner** | §10 describes secret scan + promotion gate in Transformation Domain primitive | Not yet implemented | Transformation spec |
| **TS SDK not published** | §6 requires AmeideClient | External TS consumers cannot use secure patterns | [388-ameide-sdks](388-ameide-sdks-north-star.md) |
| **Backstage not deployed** | §9.1 requires SSO + RBAC for Backstage | Cannot use secure-by-template pattern | [473](473-ameide-technology.md) §11.2 |
| **Audit logging** | §9 requires denied attempt logging | Cannot detect authorization attacks | [322-rbac](322-rbac.md), [329-authz](329-authz.md) |

### 13.3 Open architectural tensions

1. **RLS vs application-level filtering**
   * §2.1, §5.1 require DB-level RLS as "non-negotiable"
   * Current implementation uses application-level tenant filtering only (per 322-rbac)
   * **Resolution required**: Implement RLS per 329-authz Phase 2/3

2. **Agent primitive deterministic boundary enforcement**
   * §8.2 requires Domain primitives/Process primitives to invoke Agent primitives (never raw LLM APIs)
   * Current implementation may have direct LLM calls in some services
   * **Audit needed**: Verify deterministic boundary compliance

3. **Secure-by-template dependency on Backstage**
   * §9.2 assumes Backstage templates for primitive generation
   * Backstage not yet deployed per 473 §11.2
   * **Alternative needed**: Document how to achieve secure defaults without Backstage

4. **AgentDefinition scope/risk_tier vs tool grants**
   * 476 §8.1 describes `scope` and `risk_tier` as AgentDefinition-level properties
   * 310 has `tool_grants` schema but no enforcement
   * **Clarification needed**: How do scope/risk_tier interact with tool grants?

### 13.4 Security compliance status

| Principle | Status | Notes |
|-----------|--------|-------|
| §2.1 Tenant isolation (RLS) | ❌ NOT MET | RLS not enabled; 329-authz Phase 2 pending |
| §2.2 Deterministic boundary | ⚠️ PARTIAL | Architecture defined; enforcement not audited |
| §2.3 Zero trust between services | ✅ MET | Proto-based APIs with auth interceptors |
| §2.4 Immutable artifacts | ⚠️ PARTIAL | Transformation design tooling design exists; promotion gate not implemented |
| §2.5 Defence in depth | ⚠️ PARTIAL | Layers defined; not all owners assigned |
| §2.6 Secure-by-template | ❌ NOT MET | Backstage templates not created |
| §4.2 RBAC canonical roles | ✅ MET | Per 322-rbac, roles wired end-to-end |
| §6 SDK security contract | ⚠️ PARTIAL | Go/Python SDKs published; TS SDK missing |
| §8.1 AgentDefinition governance | ❌ NOT MET | scope/risk_tier fields missing in AgentDefinitions |
| §10 Audit events | ❌ NOT MET | Audit logging not implemented |

---

## 14. Recommended Priority Actions

Based on gap analysis, the following actions are recommended in priority order:

### Priority 1: CRITICAL (Blocks security compliance)

1. **Complete 329-authz Phase 2** (4 weeks)
   * Add `organization_id` to business tables
   * Enable RLS policies
   * Prerequisite for §2.1, §5.1 compliance

2. **Add API route 403 enforcement** (1 week)
   * Per 322-rbac gaps: API routes do not enforce 403 for insufficient permissions
   * Add `requirePermission()` middleware to all API routes
   * Add authorization checks to all server actions

### Priority 2: HIGH (Enables agent governance)

3. **AgentDefinition governance** (2 weeks)
   * Add `scope` and `risk_tier` to AgentDefinition in Transformation Domain primitive per §8.1
   * Implement tool grant enforcement at publish time
   * Wire "write intents" logging per §8.3

4. **Publish TypeScript SDK** (1 week)
   * Per 388 gaps: TS SDK not on npmjs
   * Publish `@ameideio/ameide-sdk-ts` with SemVer tags
   * Enables §6 compliance for external consumers

### Priority 3: MEDIUM (Completes security story)

5. **Transformation secret scanner** (2 weeks)
   * Implement secret scan validator inside Transformation Domain primitive per §9
   * Wire into promotion workflow

6. **Audit logging service** (2 weeks)
   * Per 322-rbac and 329-authz gaps
   * Wire 401/403 telemetry into dashboards

7. **Backstage security baseline** (2 weeks)
   * Deploy Backstage with SSO + RBAC per §9.1
   * Create secure-by-default templates per §9.2


<!-- ═══════════════════════════════════════════════════════════════════════════ -->
<!-- FILE: 467-backstage.md -->
<!-- ═══════════════════════════════════════════════════════════════════════════ -->

# 477 – Backstage Platform Factory

**Status:** In Progress
**Priority:** High
**Complexity:** Large
**Created:** 2025-12-07
**Updated:** 2025-12-07

> **Cross-References (Vision Suite)**:
>
> | Document | Relationship |
> |----------|-------------|
> | [470-ameide-vision.md](470-ameide-vision.md) | §5.3, §8 – Backstage template runs as "transformation records" |
> | [471-ameide-business-architecture.md](471-ameide-business-architecture.md) | §4 – Backstage as platform factory (business view) |
> | [472-ameide-information-application.md](472-ameide-information-application.md) | §4 – Catalog modeling, entity mappings |
> | [473-ameide-technology.md](473-ameide-technology.md) | §2.3 – Backstage as "factory" for primitives |
> | [475-ameide-domains.md](475-ameide-domains.md) | §2 Principle 5, §6 – Internal factory, not tenant UI |
> | [476-ameide-security-trust.md](476-ameide-security-trust.md) | §5.5 – Backstage security baseline |
> | [478-ameide-extensions.md](478-ameide-extensions.md) | Extension scaffolding workflow |
>
> **Deployment Architecture Suite**:
>
> | Document | Relationship |
> |----------|-------------|
> | [465-applicationset-architecture.md](465-applicationset-architecture.md) | How Backstage app is generated via ApplicationSet |
> | [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Rollout phase 356 (sub-phase `*56` post-bootstrap service) |
> | [464-chart-folder-alignment.md](464-chart-folder-alignment.md) | Chart at sources/charts/platform/backstage |
> | [426-keycloak-config-map.md](426-keycloak-config-map.md) | OIDC client pattern for Backstage |

---

## 1. Executive Summary

Backstage is designated as the Ameide "platform factory" – the internal developer portal that provides:

1. **Software Catalog** – Registry of all Domain/Process/Agent/UISurface primitives and their APIs
2. **Software Templates** – Scaffolder templates for creating new primitives
3. **TechDocs** – Technical documentation aggregation
4. **Integration Hub** – GitHub, Keycloak, ArgoCD integrations
5. **Primitive CR authoring** – Templates emit Domain/Process/Agent/UISurface CRDs so GitOps/Argo manage declarative primitive specs instead of raw deployments.

Per architecture doc 473 §4: "Backstage's Software Catalog and Software Templates / Scaffolder are used to create and manage all Ameide services (domain/process/agent) and their Helm charts."

---

## 2. Goals

1. Deploy Backstage as a platform service at phase 356 (sub-phase `*55`)
2. Integrate with Keycloak for OIDC authentication
3. Provision PostgreSQL database via existing CNPG cluster
4. Configure HTTPRoute for external access via Gateway API
5. Establish foundation for Software Templates

---

## 3. Non-Goals (Future Work)

- Custom Backstage Docker image with advanced plugins
- Full Software Template suite (Domain primitive, Process primitive, Agent primitive, UISurface)
- Catalog population with all existing services
- TechDocs publishing pipeline
- Keycloak organization/group sync
- ArgoCD plugin integration

---

## 4. Implementation Plan

### Phase 1: Database Setup

**Task:** Add `backstage` database to CNPG cluster

**File:** `sources/values/_shared/data/platform-postgres-clusters.yaml`

```yaml
# Add to managed.roles:
- name: backstage
  ensure: present
  login: true
  createdb: true  # Required for backstage_plugin_* databases
  passwordSecret:
    name: backstage-db-credentials

# Add to credentials.appUsers:
- name: backstage
  database: backstage
  username: backstage
  secretName: backstage-db-credentials
  template: backstage

# Add to databases:
- name: backstage
  owner: backstage
  schemas:
    - name: public
      owner: backstage
```

**Dependencies:** CNPG operator (phase 020), postgres-clusters (phase 250)

---

### Phase 2: Helm Chart

**Task:** Create custom Backstage chart

**Location:** `sources/charts/platform/backstage/`

**Structure:**
```
sources/charts/platform/backstage/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── service.yaml
    ├── httproute.yaml
    ├── configmap.yaml
    └── serviceaccount.yaml
```

**Key Templates:**

| Template | Purpose | Pattern Reference |
|----------|---------|-------------------|
| deployment.yaml | Backstage container | `www-ameide-platform/templates/deployment.yaml` |
| httproute.yaml | Gateway API routing | `www-ameide-platform/templates/httproute.yaml` |
| configmap.yaml | app-config.yaml | Custom for Backstage |

**Note:** Database credentials are managed by CNPG (per [412-cnpg-owned-postgres-greds.md](412-cnpg-owned-postgres-greds.md)), not ExternalSecrets. The deployment uses `envFrom` to inject credentials from the CNPG-generated secret.

**Backstage Configuration (app-config.yaml):**

```yaml
app:
  title: Ameide Platform Factory
  baseUrl: https://backstage.dev.ameide.io

backend:
  baseUrl: https://backstage.dev.ameide.io
  listen:
    port: 7007
  database:
    client: pg
    connection:
      host: ${POSTGRES_HOST}
      port: ${POSTGRES_PORT}
      user: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}
      database: backstage

auth:
  environment: production
  providers:
    oidc:
      production:
        metadataUrl: ${KEYCLOAK_ISSUER}/.well-known/openid-configuration
        clientId: backstage
        clientSecret: ${BACKSTAGE_OIDC_CLIENT_SECRET}
        prompt: auto
        signIn:
          resolvers:
            - resolver: emailMatchingUserEntityProfileEmail

catalog:
  rules:
    - allow: [Component, System, API, Resource, Location, Template, Group, User]
  locations:
    - type: url
      target: https://github.com/ameideio/ameide-gitops/blob/main/sources/backstage/catalog/all.yaml
      rules:
        - allow: [Location]
```

---

### Phase 3: Component Definition

**Task:** Register Backstage as ArgoCD component

**File:** `environments/_shared/components/platform/developer/backstage/component.yaml`

```yaml
name: platform-backstage
project: ameide
domain: platform
dependencyPhase: "platform"
componentType: "workload"
rolloutPhase: "356"
autoSync: true
chart:
  repoURL: https://github.com/ameideio/ameide-gitops.git
  path: sources/charts/platform/backstage
  version: main
syncOptions:
  - CreateNamespace=true
  - RespectIgnoreDifferences=true
  - SkipDryRunOnMissingResource=true
syncPolicy:
  automated:
    prune: true
    selfHeal: true
orphanedResources: {}
```

---

### Phase 4: Values Files

**Files to create:**

| File | Purpose |
|------|---------|
| `sources/values/_shared/platform/platform-backstage.yaml` | Shared configuration |
| `sources/values/dev/platform/platform-backstage.yaml` | Dev environment |
| `sources/values/staging/platform/platform-backstage.yaml` | Staging environment |
| `sources/values/production/platform/platform-backstage.yaml` | Production environment |

**Shared Values:**

```yaml
tier: platform
domain: platform
exposure: external

replicaCount: 1

image:
  repository: ghcr.io/backstage/backstage
  tag: "1.35.0"
  pullPolicy: IfNotPresent

imagePullSecrets:
  - name: ghcr-pull

service:
  type: ClusterIP
  port: 7007

resources:
  requests:
    cpu: 200m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi

postgresql:
  host: postgres-ameide-rw
  port: 5432
  database: backstage

httproute:
  enabled: true
  gateway: ameide
  sectionName: https

externalSecrets:
  enabled: true
  storeRef:
    kind: SecretStore
    name: ameide-vault
  refreshInterval: 1h
```

---

### Phase 5: Keycloak Integration

**Task:** Add Backstage OIDC client to Keycloak realm

**Location:** Keycloak realm configuration

```json
{
  "clientId": "backstage",
  "name": "Backstage Developer Portal",
  "description": "OIDC client for Backstage platform factory",
  "enabled": true,
  "publicClient": false,
  "standardFlowEnabled": true,
  "implicitFlowEnabled": false,
  "directAccessGrantsEnabled": false,
  "protocol": "openid-connect",
  "redirectUris": [
    "https://backstage.dev.ameide.io/api/auth/oidc/handler/frame",
    "https://backstage.staging.ameide.io/api/auth/oidc/handler/frame",
    "https://backstage.ameide.io/api/auth/oidc/handler/frame"
  ],
  "webOrigins": [
    "https://backstage.dev.ameide.io",
    "https://backstage.staging.ameide.io",
    "https://backstage.ameide.io"
  ],
  "defaultClientScopes": ["profile", "email", "roles"]
}
```

**Vault Secret:** Store client secret at `backstage-oidc-client-secret`

---

## 5. Technical Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Chart type | Custom wrapper | Matches existing Ameide patterns; easier integration |
| Authentication | Native OIDC via Janus IDP | Vendor-aligned with Red Hat Developer Hub; OIDC pre-configured |
| Database | Shared CNPG cluster | Follows existing pattern; no separate PostgreSQL |
| Rollout phase | 356 (platform, sub-phase `*55`) | Post-runtime bootstrap after Keycloak (350) + client-patcher (355) |
| Port | 7007 | Backstage default port |
| Docker image | `quay.io/janus-idp/backstage-showcase` | Production-ready OIDC support; aligns with RHDH vendor docs |

### 5.0.1 Vendor Alignment Decision (2025-12-07)

**Problem**: Stock `ghcr.io/backstage/backstage` image doesn't include OIDC provider module in backend.

**Vendor guidance** (Red Hat Developer Hub docs):
1. Backstage requires `@backstage/plugin-auth-backend-module-oidc-provider` registered in backend
2. Frontend needs SignInPage configured for OIDC
3. `auth.session.secret` is required for cookie signing

**Solution**: Use **Janus IDP** (`quay.io/janus-idp/backstage-showcase`) which:
- Ships with OIDC provider pre-configured
- Has SignInPage wired for OIDC
- Follows Red Hat Developer Hub patterns

**References**:
- [Red Hat Developer Hub Authentication](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.5/html/authentication_in_red_hat_developer_hub/)
- [Backstage OIDC Provider](https://backstage.io/docs/auth/oidc/)

---

## 5.1 Third-Party Chart Vendoring Strategy

Per [464-chart-folder-alignment.md](464-chart-folder-alignment.md) §1, third-party charts are vendored for reproducibility and auditability.

### Decision: Custom Wrapper vs Vendored Upstream

| Option | Description | When to Use |
|--------|-------------|-------------|
| **Custom wrapper** | Write our own chart with Deployment, Service, HTTPRoute | Simple services; need full control over templates |
| **Vendored upstream** | Copy official chart to `third_party/` and configure via values | Complex services with many features (subcharts, CRDs) |
| **Wrapper + dependency** | Our chart declares upstream as dependency in Chart.yaml | Need upstream complexity + custom templates |

**Backstage choice**: **Custom wrapper**

**Rationale**:
- Backstage official chart has heavy dependencies (Redis, bundled PostgreSQL, nginx)
- We already have CNPG and Gateway API infrastructure
- Database credentials use CNPG-managed secrets (per 412), not ExternalSecrets
- Custom wrapper integrates cleanly with existing patterns
- Simpler upgrade path (just update image tag)

### If Vendoring Were Needed

For future components requiring vendored upstream charts:

```bash
# Standard vendoring pattern
sources/charts/third_party/{registry}/{chart-name}/{version}/
├── Chart.yaml
├── values.yaml
├── templates/
└── charts/            # Subcharts (if any)
```

Example vendors in use:
```
third_party/
├── backstage/           # (if we vendored Backstage)
├── cnpg/cloudnative-pg/0.26.1/
├── grafana/loki/6.46.0/
├── temporal/temporal/0.70.0/
├── strimzi/strimzi-kafka-operator/0.48.0/
└── prometheus-community/kube-prometheus-stack/79.1.1/
```

---

## 5.2 Authentication Integration (OIDC + client-patcher)

Backstage uses the standard Ameide OIDC pattern with Keycloak. Per [426-keycloak-config-map.md](426-keycloak-config-map.md) §3.2:

### OIDC Client Configuration

The `backstage` client is defined in the Keycloak realm config and extracted via the `client-patcher` Job:

```yaml
# Per-env platform-keycloak-realm.yaml
clientPatcher:
  secretExtraction:
    clients:
      - clientId: backstage
        realm: ameide
        vaultPath: secret/backstage-oidc-client-secret
```

### Authentication Flow

```
┌─────────────┐     ┌─────────────┐     ┌───────────────┐     ┌───────────┐
│  Backstage  │────▶│  Keycloak   │────▶│ client-patcher│────▶│   Vault   │
│   (OIDC)    │     │   Realm     │     │     Job       │     │           │
└─────────────┘     └─────────────┘     └───────────────┘     └───────────┘
       │                                                             │
       │  ┌─────────────────────────────────────────────────────────┐
       └──│  ExternalSecret syncs secret from Vault to K8s          │
          └─────────────────────────────────────────────────────────┘
```

### Phase Ordering

| Phase | Component | Description |
|-------|-----------|-------------|
| 350 | Keycloak | Creates realm with `backstage` client |
| 355 | client-patcher | Extracts client secret to Vault |
| 356 | Backstage | Starts with working OIDC (ExternalSecret syncs immediately) |

**Note**: Phase 356 follows the sub-phase convention (see 447 "Sub-phase Convention" and "Platform (300-399)" band details).

### RBAC Integration

Backstage roles should map to Keycloak realm roles:

| Backstage Role | Keycloak Role | Permissions |
|----------------|---------------|-------------|
| `backstage-admin` | `platform-admin` | Full access to catalog, templates |
| `backstage-editor` | `developer` | Create from templates, edit catalog |
| `backstage-viewer` | `viewer` | Read-only catalog access |

### 5.2.1 OIDC Client Reconciliation Gap (2025-12-07)

**Status**: ⚠️ Blocking Issue Identified

**Problem**: The `backstage` OIDC client is defined in Git but does not exist in Keycloak.

**Root Cause Analysis**:

1. `KeycloakRealmImport` is **create-only** (vendor behavior per [426-keycloak-config-map.md](426-keycloak-config-map.md) §5)
2. The `ameide` realm was created **before** the backstage client was added to the config
3. `KeycloakRealmImport` won't update an existing realm - it's ephemeral (PostSync hook, deleted after success)
4. `client-patcher` only **extracts** secrets from existing clients, doesn't create missing ones

**Current Flow (Broken)**:

```
Git (backstage client) ──X──▶ Keycloak (client missing)
                                      │
                               client-patcher
                               "Client not found - skipping"
                                      │
                                      X (no secret in Vault)
                                      │
                               ExternalSecret FAILS
                                      │
                               Backstage FAILS (missing secret)
```

**Layer Analysis**:

| Layer | Status | Notes |
|-------|--------|-------|
| Client definition in Git | ✅ Done | `platform-keycloak-realm.yaml` has backstage client |
| Sync Git → Keycloak | ❌ Missing | KeycloakRealmImport is create-only |
| Secret extraction (Keycloak → Vault) | ⚠️ Partial | Works but skips missing clients |
| Secret sync (Vault → K8s) | ✅ Ready | ExternalSecret configured |
| Backstage consuming secret | ✅ Ready | Deployment env vars configured |

**This is a General Pattern Issue**:

This affects **all** OIDC clients added after initial realm creation:
- `backstage` (current blocker)
- `k8s-dashboard` (may have same issue)
- Any future OIDC-enabled service

**Solution**: See [485-keycloak-oidc-client-reconciliation.md](485-keycloak-oidc-client-reconciliation.md) for the declarative fix.

**Vendor References**:
- [Red Hat Developer Hub Authentication](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.5/html/authentication_in_red_hat_developer_hub/)
- [Backstage OIDC Provider](https://backstage.io/docs/auth/oidc/)
- [ArgoCD Keycloak Integration](https://argo-cd.readthedocs.io/en/release-2.13/operator-manual/user-management/keycloak/)

---

## 5.3 Telemetry & Observability Integration

Per [473-ameide-technology.md](473-ameide-technology.md) §7, all platform services must emit tracing, metrics, and structured logs.

### OpenTelemetry Configuration

```yaml
# In app-config.yaml (ConfigMap)
backend:
  # ... existing config ...

# OpenTelemetry auto-instrumentation
telemetry:
  enabled: true
  endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT}
  serviceName: backstage
  metricsDisabled: false
```

### Deployment Environment Variables

```yaml
# In deployment.yaml
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://otel-collector.ameide-{{ .Values.environment }}:4318"
  - name: OTEL_SERVICE_NAME
    value: "backstage"
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "deployment.environment={{ .Values.environment }},service.namespace=ameide-{{ .Values.environment }}"
```

### Prometheus Metrics

Backstage exposes metrics at `/metrics`. Create ServiceMonitor:

```yaml
# templates/servicemonitor.yaml
{{- if .Values.metrics.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "backstage.fullname" . }}
  labels:
    {{- include "backstage.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "backstage.selectorLabels" . | nindent 6 }}
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
{{- end }}
```

### Structured Logging

Backstage should emit JSON logs with correlation IDs:

```yaml
# In app-config.yaml
backend:
  logger:
    format: json
    level: info
  # Include trace context
  plugins:
    - package: '@backstage/plugin-logging'
```

Log fields required per platform standard:
- `trace_id`, `span_id` (from OTel)
- `tenant_id` (from request context)
- `primitive_type: platform`
- `primitive_name: backstage`

---

## 5.4 Helm Testing Requirements

Per [458-helm-chart-ci-testing.md](458-helm-chart-ci-testing.md), all custom charts must pass CI validation.

### Required Test Coverage

#### 1. Helm Lint (Automatic)

The workflow at `.github/workflows/helm-test.yaml` automatically lints `sources/charts/platform/backstage/`.

#### 2. Unit Tests (Required)

Create `tests/` directory with helm-unittest specs:

```
sources/charts/platform/backstage/
├── Chart.yaml
├── values.yaml
├── templates/
│   └── ...
└── tests/
    ├── deployment_test.yaml
    ├── configmap_test.yaml
    ├── externalsecrets_test.yaml
    └── httproute_test.yaml
```

**Example unit test** (`tests/deployment_test.yaml`):

```yaml
suite: deployment tests
templates:
  - templates/deployment.yaml
tests:
  - it: should render deployment
    asserts:
      - isKind:
          of: Deployment
      - equal:
          path: metadata.name
          value: RELEASE-NAME-backstage

  - it: should use correct image
    set:
      image:
        repository: ghcr.io/backstage/backstage
        tag: "1.35.0"
    asserts:
      - equal:
          path: spec.template.spec.containers[0].image
          value: "ghcr.io/backstage/backstage:1.35.0"

  - it: should include OTEL environment variables when telemetry enabled
    set:
      telemetry:
        enabled: true
        endpoint: "http://otel-collector:4318"
    asserts:
      - contains:
          path: spec.template.spec.containers[0].env
          content:
            name: OTEL_EXPORTER_OTLP_ENDPOINT
            value: "http://otel-collector:4318"
```

#### 3. Template Validation (Automatic)

The workflow templates each chart with real values and validates against Kubernetes schemas via kubeconform.

### CI Integration

Add Backstage to the workflow's chart discovery:

```yaml
# Already covered by existing pattern - platform charts at sources/charts/platform/
# are automatically discovered and tested
```

---

## 6. Dependencies

| Dependency | Phase | Status |
|------------|-------|--------|
| CNPG Operator | 020 | ✅ Deployed |
| ExternalSecrets Operator | 020 | ✅ Deployed |
| Gateway API | 340 | ✅ Deployed |
| Keycloak | 350 | ✅ Deployed |
| PostgreSQL Clusters | 250 | ✅ Deployed |
| Vault | - | ✅ External |

---

## 6.1 GitOps Alignment (Deployment Architecture)

Backstage follows the established Ameide GitOps patterns documented in [465-applicationset-architecture.md](465-applicationset-architecture.md).

### Dual ApplicationSet Model

Backstage is an **environment-scoped workload** (not a cluster-scoped operator), deployed via the `ameide` ApplicationSet:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ArgoCD ApplicationSets                          │
├─────────────────────────────────────────────┬───────────────────────────┤
│              ameide.yaml                    │      cluster.yaml         │
│         (Environment-Scoped)                │    (Cluster-Scoped)       │
├─────────────────────────────────────────────┼───────────────────────────┤
│  ✅ Backstage belongs here                  │  ❌ NOT here              │
│  3 apps: dev-*, staging-*, production-*     │  1 app: cluster-*         │
│  Phase 356 (sub-phase *55)                  │  Phases 010-030 only      │
└─────────────────────────────────────────────┴───────────────────────────┘
```

### File Location Conventions

Per [464-chart-folder-alignment.md](464-chart-folder-alignment.md), Backstage files follow the `platform` domain pattern:

| What | Convention | Backstage File |
|------|------------|----------------|
| Component | `components/{domain}/{subdomain}/{name}/component.yaml` | `platform/developer/backstage/component.yaml` |
| Chart | `charts/{domain}/{name}/` | `charts/platform/backstage/` |
| Shared values | `values/_shared/{domain}/{name}.yaml` | `values/_shared/platform/platform-backstage.yaml` |
| Env override | `values/{env}/{domain}/{name}.yaml` | `values/dev/platform/platform-backstage.yaml` |

### Rollout Phase Placement

Per [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md), Backstage sits at phase **356** in the platform band (300-399).

**Sub-phase convention** (from 447 "Sub-phase Convention"):
| Sub-phase | Purpose |
|-----------|---------|
| `*50` | Runtimes/workloads (e.g., Keycloak at 350) |
| `*55` | Post-runtime bootstraps (e.g., keycloak-realm at 355) |
| `*56+` | Services dependent on bootstraps (Backstage at 356) |

**Dependency chain**:

```
Phase 020  │ Operators (CNPG, ExternalSecrets)
Phase 130  │ ExternalSecrets SecretStores
Phase 250  │ PostgreSQL Clusters (creates backstage DB)
Phase 340  │ Gateway API (HTTPRoutes depend on this)
Phase 350  │ Keycloak (creates `backstage` client)
Phase 355  │ client-patcher (extracts secret to Vault) + keycloak-realm
Phase 356  │ ◀── Backstage (sub-phase *55: post-runtime bootstrap)
Phase 650  │ Apps (downstream services that consume Backstage)
```

### Values Resolution Order

When ArgoCD deploys `dev-platform-backstage`:

```yaml
valueFiles:
  - $values/sources/values/dev/globals.yaml                    # 1. Environment globals
  - $values/sources/values/_shared/platform/platform-backstage.yaml  # 2. Shared values
  - $values/sources/values/dev/platform/platform-backstage.yaml      # 3. Environment override
```

### Secrets Pattern

Backstage uses the standard ExternalSecrets + Vault pattern per [426-keycloak-config-map.md](426-keycloak-config-map.md):

```
Vault (external)
    └── kv/ameide/dev/backstage-oidc-client-secret
                 ▼
ExternalSecrets SecretStore (phase 130)
                 ▼
ExternalSecret → K8s Secret (phase 356, via Backstage chart)
                 ▼
Backstage deployment mounts secret as env var
```

### Application Generation

The `ameide` ApplicationSet matrix generates 3 Backstage Applications:

```
List elements × Git files = Applications

dev         × platform/developer/backstage = dev-platform-backstage
staging     × platform/developer/backstage = staging-platform-backstage
production  × platform/developer/backstage = production-platform-backstage
```

---

## 7. Rollout Plan

### 7.1 Development Environment

1. ✅ Add `backstage` database to CNPG cluster values
2. ✅ Deploy postgres-clusters update
3. ✅ Create Backstage Helm chart
4. ✅ Add Keycloak OIDC client to realm config
5. ✅ Configure client-patcher to extract secret to Vault
6. ✅ Create component definition
7. ✅ Create dev values file
8. ✅ Deploy to dev via ArgoCD
9. ⏳ Verify authentication flow (after keycloak-realm sync)
10. ✅ Verify database connectivity

### 7.2 Staging & Production

1. ✅ Create staging/production values files
2. ✅ Update hostnames
3. ✅ Deploy via ArgoCD
4. ⏳ Smoke test each environment

---

## 7.3 Implementation Progress

**Implementation Date:** 2025-12-07

### MVP Scope

The initial deployment is an MVP focused on:
- ✅ Database connectivity (CNPG-managed)
- ✅ HTTPRoute exposure via Gateway API
- ✅ ArgoCD deployment across all environments
- ✅ Authentication (OIDC via Keycloak - implemented 2025-12-07)
- ⏳ Telemetry integration (planned)
- ⏳ Helm unit tests (planned)

### Phase 1: Database Setup ✅

Added backstage role and database to CNPG cluster:

```yaml
# sources/values/_shared/data/platform-postgres-clusters.yaml
managed:
  roles:
    - name: backstage
      ensure: present
      login: true
      createdb: true  # Required for backstage_plugin_* databases
      passwordSecret:
        name: backstage-db-credentials

credentials:
  appUsers:
    - name: backstage
      database: backstage
      username: backstage
      secretName: backstage-db-credentials
      template: backstage

databases:
  - name: backstage
    owner: backstage
    schemas:
      - name: public
        owner: backstage
```

**Key Discovery:** Backstage requires `CREATEDB` privilege to create plugin databases (e.g., `backstage_plugin_app`, `backstage_plugin_auth`, `backstage_plugin_catalog`).

### Phase 2: Helm Chart ✅

Created custom Backstage chart at `sources/charts/platform/backstage/`:

| File | Purpose |
|------|---------|
| `Chart.yaml` | Chart metadata (appVersion: 1.35.0) |
| `values.yaml` | Default values |
| `templates/_helpers.tpl` | Template helpers |
| `templates/deployment.yaml` | Backstage container with CNPG envFrom |
| `templates/service.yaml` | ClusterIP service on port 7007 |
| `templates/httproute.yaml` | Gateway API routing |
| `templates/configmap.yaml` | app-config.yaml with techdocs section |
| `templates/serviceaccount.yaml` | ServiceAccount |

**Key Decision:** Uses CNPG-managed credentials directly (via `envFrom` secretRef) instead of ExternalSecrets. This aligns with [412-cnpg-owned-postgres-greds.md](412-cnpg-owned-postgres-greds.md) pattern.

Added `backstage` template to CNPG app-secrets:

```yaml
# sources/charts/foundation/operators-config/postgres_clusters/templates/app-secrets.yaml
{{- else if eq $template "backstage" }}
  POSTGRES_HOST: {{ $host | quote }}
  POSTGRES_PORT: {{ printf "%d" (int $port) | quote }}
  POSTGRES_USER: {{ $username | quote }}
  POSTGRES_PASSWORD: {{ $password | quote }}
  POSTGRES_URL: {{ printf "postgresql://%s:%s@%s:%d/%s" ... | quote }}
{{- end }}
```

### Phase 3: Component Definition ✅

Created ArgoCD component at `environments/_shared/components/platform/developer/backstage/component.yaml`:

```yaml
name: platform-backstage
project: ameide
domain: platform
dependencyPhase: "platform"
componentType: "workload"
rolloutPhase: "356"
autoSync: true
```

### Phase 4: Values Files ✅

Created all environment values files:

| File | Hostname |
|------|----------|
| `sources/values/_shared/platform/platform-backstage.yaml` | (shared config) |
| `sources/values/dev/platform/platform-backstage.yaml` | `backstage.dev.ameide.io` |
| `sources/values/staging/platform/platform-backstage.yaml` | `backstage.staging.ameide.io` |
| `sources/values/production/platform/platform-backstage.yaml` | `backstage.ameide.io` |

### Deployment Status

| Environment | ArgoCD App | Status |
|-------------|------------|--------|
| dev | dev-platform-backstage | ✅ Synced, Healthy |
| staging | staging-platform-backstage | ✅ Synced, Healthy |
| production | production-platform-backstage | ⏳ Synced, Progressing (namespace pending) |

### Issues Resolved During Implementation

1. **Permission denied to create database** - Added `createdb: true` to backstage role in CNPG config
2. **YAML parse error in app-secrets.yaml** - Removed inline Helm comments that caused parsing issues
3. **Missing techdocs config** - Added required `techdocs` section to ConfigMap app-config.yaml

---

## 8. Files Summary

### New Files Created

```
# Component registration ✅
environments/_shared/components/platform/developer/backstage/component.yaml

# Helm chart ✅
sources/charts/platform/backstage/Chart.yaml
sources/charts/platform/backstage/values.yaml
sources/charts/platform/backstage/templates/_helpers.tpl
sources/charts/platform/backstage/templates/deployment.yaml
sources/charts/platform/backstage/templates/service.yaml
sources/charts/platform/backstage/templates/httproute.yaml
sources/charts/platform/backstage/templates/configmap.yaml
sources/charts/platform/backstage/templates/serviceaccount.yaml

# Values files ✅
sources/values/_shared/platform/platform-backstage.yaml
sources/values/dev/platform/platform-backstage.yaml
sources/values/staging/platform/platform-backstage.yaml
sources/values/production/platform/platform-backstage.yaml
```

### Planned Files (Future)

```
# Telemetry (§5.3)
sources/charts/platform/backstage/templates/servicemonitor.yaml

# Helm unit tests (§5.4)
sources/charts/platform/backstage/tests/deployment_test.yaml
sources/charts/platform/backstage/tests/configmap_test.yaml
sources/charts/platform/backstage/tests/httproute_test.yaml
sources/charts/platform/backstage/tests/servicemonitor_test.yaml

# Backstage catalog (future)
sources/backstage/catalog/all.yaml
sources/backstage/catalog/domains/
sources/backstage/catalog/systems/
```

### Modified Files

```
# Database provisioning ✅
sources/values/_shared/data/platform-postgres-clusters.yaml

# CNPG app-secrets template ✅
sources/charts/foundation/operators-config/postgres_clusters/templates/app-secrets.yaml

# Keycloak realm config ✅ (OIDC client added 2025-12-07)
sources/values/_shared/platform/platform-keycloak-realm.yaml    # backstage client definition

# client-patcher configuration ✅ (2025-12-07)
sources/values/dev/platform/platform-keycloak-realm.yaml        # secretExtraction for backstage
sources/values/staging/platform/platform-keycloak-realm.yaml
sources/values/production/platform/platform-keycloak-realm.yaml

# ExternalSecret template ✅ (2025-12-07)
sources/charts/foundation/operators-config/keycloak_realm/templates/externalsecret-oidc-clients.yaml
```

---

## 9. Success Criteria

### Deployment (MVP)
- [ ] Backstage accessible at `backstage.dev.ameide.io`
- [ ] Health endpoint returns 200 at `/healthz`
- [x] ArgoCD shows `Synced` and `Healthy` (dev, staging)

### Authentication (§5.2) - Implemented 2025-12-07
- [x] `backstage` client added to Keycloak `ameide` realm config
- [x] client-patcher configured to extract secret to Vault (all environments)
- [x] ExternalSecret template updated to include backstage-client-secret
- [x] Backstage configmap includes auth section with OIDC provider
- [x] Deployment mounts OIDC secret as AUTH_OIDC_CLIENT_SECRET
- [ ] Keycloak SSO login working (pending keycloak-realm sync)

### Database
- [x] `backstage` database exists in CNPG cluster
- [x] `backstage` role has CREATEDB privilege for plugin databases
- [x] CNPG-managed credentials available via Secret
- [x] Backstage connects without errors in logs
- [x] Migrations complete successfully on startup

### Telemetry (§5.3) - Future
- [ ] Traces visible in Tempo/Grafana
- [ ] Metrics scraped by Prometheus (ServiceMonitor)
- [ ] Logs in JSON format with trace_id correlation

### CI (§5.4) - Future
- [ ] `helm lint` passes for Backstage chart
- [ ] `helm unittest` passes all tests
- [ ] Template renders for all environments (dev/staging/prod)

---

## 10. Future Backlog Items

### 10.1 Software Templates (Design-Time Integration)

Per [472-ameide-information-application.md](472-ameide-information-application.md) §4, Backstage templates scaffold the primitive lifecycle:

| Template | Artifact Type | Output |
|----------|---------------|--------|
| `Domain primitive` | Runtime | Proto skeleton, Go/TS service, Domain CRD manifest, ArgoCD component |
| `Process primitive` | Runtime | Temporal workflow, ProcessDefinition loader, Process CRD manifest |
| `AgentController` | Runtime | AgentDefinition executor, tool registry, IAC manifest |
| `ProcessDefinition` | Design-time (UAF) | BPMN-compliant artifact from React Flow modeller |
| `AgentDefinition` | Design-time (UAF) | Declarative agent spec (tools, scope, risk tier, policies) |

**Note**: ProcessDefinitions and AgentDefinitions are **design-time artifacts** stored in Transformation Domain. Backstage templates can scaffold the runtime primitives that **execute** these definitions, but the definitions themselves come from the custom React Flow modeller (and other UAF UIs).

#### Tenant Extension Templates

Per [478-ameide-extensions.md](478-ameide-extensions.md) §4-5, templates include **namespace calculation logic** that determines target deployment based on:

* `tenant_id` - identifies the tenant
* `sku` - Shared / Namespace / Private
* `code_origin` - platform vs tenant

**Namespace calculation**:

```
if origin == "platform":
  if sku == "Shared": return "ameide-{env}"
  else: return "tenant-{id}-{env}-base"
else:  # tenant/agent origin
  return "tenant-{id}-{env}-cust"
```

**Security invariant**: Custom code never targets `ameide-*` namespaces. See [478-ameide-extensions.md](478-ameide-extensions.md) §7 for enforcement details.

### 10.2 Catalog Population

Populate Software Catalog with existing Ameide services per the entity mapping in [472](472-ameide-information-application.md) §4.1:

| Ameide Concept | Backstage Kind |
|----------------|----------------|
| Domain primitive | `Component` (+ custom `Domain`) |
| Process primitive | `Component` (+ custom `Process`) |
| Agent primitive | `Component` (+ custom `Agent`) |
| Proto service | `API` (grpc type) |
| ProcessDefinition | `Resource` (custom kind, links to UAF) |
| AgentDefinition | `Resource` (custom kind, links to UAF) |

### 10.3 Backlog Summary

| Item | Description |
|------|-------------|
| 479-backstage-templates | Domain/Process/Agent primitive scaffold templates |
| 480-backstage-catalog | Populate catalog with existing Ameide services and APIs |
| 481-backstage-techdocs | TechDocs publishing pipeline (from backlog markdown) |
| 482-backstage-argocd | ArgoCD plugin for deployment visibility |
| 483-backstage-uaf-integration | Link ProcessDefinition/AgentDefinition catalog entries to UAF |

**Note**: 478-ameide-extensions.md now documents the tenant extension model that will inform template development.

---

## 11. References

### External Documentation
- [Backstage Documentation](https://backstage.io/docs)
- [Backstage Helm Charts](https://github.com/backstage/charts)
- [Backstage OIDC Authentication](https://backstage.io/docs/auth/oidc/)
- [OpenTelemetry Node.js](https://opentelemetry.io/docs/languages/js/)

### Vision & Architecture Suite
| Document | Section | Relationship |
|----------|---------|--------------|
| [470-ameide-vision.md](470-ameide-vision.md) | §4.3, §5.3 | Backstage as "factory", template runs |
| [471-ameide-business-architecture.md](471-ameide-business-architecture.md) | §4 | Platform factory business view |
| [472-ameide-information-application.md](472-ameide-information-application.md) | §4 | Catalog modeling, entity mappings |
| [473-ameide-technology.md](473-ameide-technology.md) | §2.3, §4 | Technology blueprint, templates |
| [475-ameide-domains.md](475-ameide-domains.md) | §2, §6 | Internal factory principle |
| [476-ameide-security-trust.md](476-ameide-security-trust.md) | §8 | Security baseline |
| [478-ameide-extensions.md](478-ameide-extensions.md) | §4-6 | Tenant extension scaffolding, namespace strategy |

### Deployment Architecture Suite
| Document | Section | Relationship |
|----------|---------|--------------|
| [465-applicationset-architecture.md](465-applicationset-architecture.md) | Full | Dual ApplicationSet model |
| [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Sub-phase Convention, Platform band | Rollout phase 356 (sub-phase `*56`) |
| [464-chart-folder-alignment.md](464-chart-folder-alignment.md) | §1, GAPs | Chart conventions |
| [426-keycloak-config-map.md](426-keycloak-config-map.md) | §3.2 | client-patcher pattern |
| [458-helm-chart-ci-testing.md](458-helm-chart-ci-testing.md) | Full | Helm unit tests, CI workflow |
| [462-secrets-origin-classification.md](462-secrets-origin-classification.md) | §4 | Secret authority model |

### Operational Patterns
| Document | Topic | Relationship |
|----------|-------|--------------|
| [450-argocd-service-issues-inventory.md](450-argocd-service-issues-inventory.md) | OIDC secrets | client-patcher resolution |
| [412-cnpg-owned-postgres-greds.md](412-cnpg-owned-postgres-greds.md) | DB credentials | CNPG credential pattern |
| [436-envoy-gateway-observability.md](436-envoy-gateway-observability.md) | Telemetry | ServiceMonitor patterns |


<!-- ═══════════════════════════════════════════════════════════════════════════ -->
<!-- FILE: 478-ameide-extensions.md -->
<!-- ═══════════════════════════════════════════════════════════════════════════ -->

# 478 – Ameide Extensions & Tenant Primitive Model

**Status:** Draft v1
**Owner:** Architecture / Platform
**Audience:** Platform engineers, domain architects, solution architects

> **Cross-References (Vision Suite)**:
>
> | Document | Purpose |
> |----------|---------|
> | [470-ameide-vision.md](470-ameide-vision.md) | Vision, rationale, principles |
> | [471-ameide-business-architecture.md](471-ameide-business-architecture.md) | Business concepts, tenant journey |
> | [472-ameide-information-application.md](472-ameide-information-application.md) | Application architecture, Backstage catalog |
> | [473-ameide-technology.md](473-ameide-technology.md) | Technology stack, deployment |
> | [475-ameide-domains.md](475-ameide-domains.md) | Domain portfolio, primitive patterns |
> | [476-ameide-security-trust.md](476-ameide-security-trust.md) | Security principles, agent governance |
> | [467-backstage.md](467-backstage.md) | Backstage implementation |
> | [479-ameide-extensibility-wasm.md](479-ameide-extensibility-wasm.md) | Tier 1 + Tier 2 extensibility framing |
> | [480-ameide-extensibility-wasm-service.md](480-ameide-extensibility-wasm-service.md) | Tier 1 runtime implementation |
>
> **Related Implementation**:
> - [443-tenancy-models.md](443-tenancy-models.md) – SKU definitions (Shared/Namespace/Private)
> - [465-applicationset-architecture.md](465-applicationset-architecture.md) – GitOps deployment model
> - [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) – Rollout phases

---

## 1. Purpose

This document describes **how tenant-specific primitives are created, deployed, and isolated** within the Ameide platform. It operationalizes the architecture principles from 470-477 into concrete workflows.

**Scope note:** This document covers **Tier 2 primitive-based extensions** (Domain/Process/Agent primitives owned by tenants). Tier 1 WASM extensions stay inside the shared runtime described in [479](479-ameide-extensibility-wasm.md)/[480](480-ameide-extensibility-wasm-service.md); they only appear here when referencing shared namespace/security invariants.

Key questions answered:

* How does Backstage fit into the extension model?
* Where does platform code vs tenant code live?
* How are namespaces structured by SKU?
* What is the E2E flow from requirement to running a new primitive?
* What security invariants must hold?

---

## 2. What Backstage Is in Ameide

### 2.1 Positioning

Backstage is the **internal factory + map** for primitives:

* Domain primitives
* Process primitives
* Agent primitives

It is **never tenant-facing**. Tenants only see the Ameide platform UI.

Backstage pulls truth from:

* `service_catalog/` in the platform repo (which primitives exist, how they're wired)
* Tenant extension repos (for custom primitives)
* Runtime systems (ArgoCD, K8s, observability)

> **Think of it as**: "where platform engineers go to create & reason about primitives", not "where customers click buttons".
>
> Tier 1 WASM extensions do **not** use Backstage; they are stored as `ExtensionDefinition` artifacts and executed by the shared runtime (479/480).
>
> Tier 2 templates emit primitive repositories **and** the Domain/Process/Agent/UISurface CRDs that GitOps applies to realise runtime primitives.

### 2.2 Backstage Capabilities Used

| Backstage Feature | Ameide Usage |
|-------------------|--------------|
| Software Catalog | Registry of all Domain/Process/Agent primitives and APIs |
| Software Templates | Scaffolder templates for creating new primitives |
| TechDocs | Technical documentation aggregation |
| Integrations | GitHub, Keycloak, ArgoCD plugins |

See [467-backstage.md](467-backstage.md) for implementation details.

---

## 3. Repository Strategy

### 3.1 Platform Repository

Contains:

* `service_catalog/` with primitive structure and base implementations
* Backstage app + plugins
* **Backstage templates**:
  * Domain primitive template
  * Process primitive template
  * Agent primitive template
* Shared libraries / SDKs, infra charts, etc.

**Only Ameide-owned code lives here. No tenant custom code.**

### 3.2 Tenant-Side Repositories

Per tenant that has customisation:

| Repository | Purpose |
|------------|---------|
| `tenant-{id}-controllers` | Tenant-specific or tenant-scoped primitive code |
| `tenant-{id}-gitops` | Helm/Argo manifests for those primitives |

Backstage catalog has **Locations** pointing at these repos, so tenant primitives show up in the internal portal, but code and manifests stay out of the platform repo.

### 3.3 Repository Ownership Matrix

| Code Origin | Repository | Who Can Commit |
|-------------|------------|----------------|
| Platform | `ameide-platform` | Ameide engineers only |
| Tenant (custom) | `tenant-{id}-controllers` | Tenant engineers + approved agents |
| GitOps (platform) | `ameide-gitops` | Ameide engineers + CI |
| GitOps (tenant) | `tenant-{id}-gitops` | Tenant engineers + approved agents |

---

## 4. Namespace Strategy by SKU

For each environment (`dev`, `staging`, `prod`), namespaces are structured based on SKU.

### 4.1 Shared Platform Plane

* `ameide-{env}`
  * Multi-tenant platform primitives (shared product)
  * RLS + tenantId/orgId for data isolation
  * **Code must come from platform repos only**
  * **No tenant-specific/custom code ever**
  * Tier 1 WASM modules run here only as data executed by the platform-owned runtime (see 479/480); they are not services.

### 4.2 Tenant Planes by SKU

#### Shared SKU

* Shared product runs in: `ameide-{env}`
* If the tenant has custom code, they get:
  * `tenant-{id}-{env}-cust`
    * Tenant/agent-generated primitives
    * Calls **into** `ameide-{env}` only via well-defined APIs (HTTP/gRPC) with `tenantId` in auth
    * No direct access to shared DBs or internal services
  * No `tenant-{id}-{env}-base` (reserved for dedicated namespace/cluster SKUs)

#### Namespace SKU

On a shared cluster with dedicated namespace:

* `tenant-{id}-{env}-base`
  * Primary product runtime **dedicated to that tenant** (their "base" namespace)
  * Platform-managed primitives specialised for that tenant
* Optionally `tenant-{id}-{env}-cust`
  * Tenant/agent-owned code, kept separate from the "product" base

#### Private SKU

On a dedicated cluster:

* `tenant-{id}-{env}-base`
  * Primary product runtime on dedicated infrastructure
* Optionally `tenant-{id}-{env}-cust`
  * Custom extensions

### 4.3 Namespace Naming Convention

```
ameide-{env}                    # Shared platform (Shared SKU product)
tenant-{id}-{env}-base          # Dedicated product (Namespace/Private SKU)
tenant-{id}-{env}-cust          # Custom extensions (all SKUs)
```

Where:
* `{env}` = `dev` | `staging` | `prod`
* `{id}` = tenant identifier (e.g., `acme`, `contoso`)

### 4.4 Namespace Decision Matrix

| SKU | Product Runtime | Custom Code |
|-----|-----------------|-------------|
| Shared | `ameide-{env}` | `tenant-{id}-{env}-cust` |
| Namespace | `tenant-{id}-{env}-base` | `tenant-{id}-{env}-cust` |
| Private | `tenant-{id}-{env}-base` | `tenant-{id}-{env}-cust` |

**Invariant**: Custom services always run in a dedicated tenant namespace, never in the shared `ameide-*` namespaces. Tier 1 WASM extensions are the single, governed exception and execute as data within the shared runtime per [479](479-ameide-extensibility-wasm.md)/[480](480-ameide-extensibility-wasm-service.md).

---

## 5. Backstage Templates

### 5.1 Golden Templates

We maintain 3–4 **golden templates** in the platform repo:

| Template | Purpose |
|----------|---------|
| `Domain primitive` | Create a new domain with proto APIs, DB schema, Helm chart |
| `Process primitive` | Create a process runtime with Temporal workers |
| `Agent primitive` | Create an agent runtime with tool registry |
| `UIWorkspace` (optional) | Create a Next.js microfrontend |

### 5.2 Template Inputs

Each template asks for:

| Input | Description |
|-------|-------------|
| `tenant_id` | Optional if platform-only |
| `sku` | Shared / Namespace / Private |
| `primitive_name` | Name of the primitive |
| `primitive_type` | Domain / Process / Agent |
| `target_repo` | Platform vs tenant repo |

### 5.3 Template Outputs

The template computes the **target namespace** per env based on:

* `tenant_id`
* SKU (Shared / Namespace / Private)
* Origin (`platform` vs `tenant`)

Then scaffolds:

* Primitive code skeleton (with SDK, RLS hooks, telemetry, auth middleware)
* Dockerfile / CI skeleton
* Helm chart / K8s manifests
* `catalog-info.yaml` for Backstage

The **Backstage Scaffolder** always publishes into the **right repo** (platform or tenant) and generates manifests pointing at the **right namespace** (`ameide-*`, `tenant-{id}-*-base`, or `tenant-{id}-*-cust`), according to the rules in §4.

### 5.4 Namespace Calculation Logic

```
function calculateNamespace(tenant_id, sku, origin, env):
  if origin == "platform":
    if sku == "Shared":
      return "ameide-{env}"
    else:
      return "tenant-{tenant_id}-{env}-base"
  else:  # origin == "tenant" or "agent"
    return "tenant-{tenant_id}-{env}-cust"
```

---

## 6. E2E Flow: From Idea to Running Primitive

### Step 0 – Requirement Appears

* A human or AI agent sees a need: "Tenant X needs capability Y."
* This is captured through the **Transformation domain** (your "change-the-business" API).

### Step 1 – Design First (Config/Definitions)

* The agent/human works in **Transformation**:
  * Defines / updates `ProcessDefinition` (L2O variant, workflows, SLAs)
  * Defines / updates `AgentDefinition` (tools, prompts, policies)
  * Attaches them to `tenantId`/`orgId`
* If the requirement can be met by **definitions + existing primitives**, it ends here: **no new code, no Backstage**.

### Step 2 – Decide If New Code Is Needed

If existing primitives and definitions can't implement the behaviour:

* Transformation (or a human) decides: "we need a new [Domain/Process/Agent] primitive."
* Transformation validates:
  * Tenant SKU
  * Allowed isolation mode
  * Whether custom code is permitted at all for this tenant

### Step 3 – Create a PrimitiveImplementationDraft

Agent calls a high-level API:

```
CreatePrimitiveImplementationDraft(
  tenant_id,
  primitive_type,
  primitive_id,
  template_id,
  code_origin   = "platform" | "tenant",
  target_repo   = "...",
  design_refs   = [ProcessDefinition/AgentDefinition IDs]
)
```

Transformation:

* Looks up tenant SKU & execution plan
* Calculates proper namespaces per env:
  * Shared + platform code → `ameide-{env}`
  * Any custom/tenant code → `tenant-{id}-{env}-cust`
  * Namespace/private base → `tenant-{id}-{env}-base` (for platform-owned dedicated primitives)
* Creates a `PrimitiveImplementationDraft` artifact with all that info

### Step 4 – Deterministic Worker Calls Backstage

A backend worker (Process primitive) reacts to the draft and:

* Calls Backstage Scaffolder with the right template and values
* Backstage:
  * Generates code & manifests
  * Publishes to the **target repo** (platform or tenant)
  * Opens a branch/PR

The draft is updated with **links to the repo/PR** and Backstage task logs.

### Step 5 – Agent "Writes Code" via the Draft, Not Git

The agent never gets Git or K8s credentials.

Instead, it uses tools like:

| Tool | Purpose |
|------|---------|
| `GetPrimitiveDraftFiles(draft_id)` | Snapshot of files from the feature branch |
| `UpdatePrimitiveDraftFiles(draft_id, patches)` | Apply patches via deterministic worker |

The deterministic worker:

* Applies patches
* Runs lint/tests/security checks
* Commits & pushes to the feature branch

All this is:

* Audited (who/what changed what, for which tenant)
* Logged as **write-intent** events in Transformation

### Step 6 – "Put This in Test" → Promote to Dev

Agent (or human) calls:

```
RequestPromotion(draft_id, env="dev")
```

Transformation worker:

1. Runs CI on the branch (tests, lint, SAST/secret scan, etc.)
2. If green, updates **`tenant-{id}-gitops`** (or platform GitOps repo if platform-owned code) with:
  * Argo `Application`/`HelmRelease` pointing to the primitive
   * Appropriate namespace:
     * Platform code → `ameide-dev`
     * Tenant custom code → `tenant-{id}-dev-cust`
     * Dedicated base → `tenant-{id}-dev-base`
3. Opens a GitOps PR, which may auto-merge for `dev` under policy

ArgoCD's ApplicationSet for that env syncs:

* Controller is deployed into the **dedicated namespace** dictated by the rules
* No custom images in `ameide-dev` ever

### Step 7 – Observe & Iterate

* Agent / human uses Backstage + observability to see:
  * Health, logs, metrics
  * Catalog relations (which definitions use this primitive, which tenants are affected)
* If changes needed, repeat Step 5–6

### Step 8 – Promote to Staging / Prod

Same API, stricter policy:

```
RequestPromotion(draft_id, env="staging")
RequestPromotion(draft_id, env="prod")
```

* Reuses the **same built artifact** (image/tag) that ran in dev
* Only changes GitOps config to:
  * `ameide-staging` vs `ameide-prod`, or
  * `tenant-{id}-staging-{base/cust}` vs `tenant-{id}-prod-{base/cust}`
* Requires approvals appropriate to risk tier and tenant

Backstage & ArgoCD show the rollout status per environment.

---

## 7. Security Invariants

No matter how fancy the flow gets, these hold:

### 7.1 No Custom Code in Shared Namespaces

* Any primitive whose origin is `tenant` (or "untrusted") **cannot** target `ameide-*`
* Validated in Transformation before scaffolding, and in GitOps/Argo policy

### 7.2 Dedicated Namespaces for All Custom Services

* Shared SKU → custom code in `tenant-{id}-{env}-cust`
* Namespace/Private → product runtime in `tenant-{id}-{env}-base`, optional custom in `*-cust`

### 7.3 Only Deterministic Services Touch Git, Backstage, Argo

* Agents only call high-level APIs in Transformation
* Everything side-effectful (scaffold, commit, deploy) happens in predictable primitives

### 7.4 All Calls from Tenant Code via APIs with Tenant-Aware Auth

* No direct DB or internal network access into shared plane
* NetworkPolicies and mTLS + JWT enforce "API/tenantId only"

### 7.5 Alignment with 476 Security Principles

| 476 Principle | 478 Implementation |
|---------------|-------------------|
| §2.1 Tenant isolation first | §7.1, §7.2 namespace isolation |
| §2.2 Deterministic boundary | §7.3 agents don't touch Git directly |
| §2.3 Zero trust between services | §7.4 API-only access with auth |
| §7.3 Agent-generated code governance | §6 Step 5 via draft APIs, not direct Git |

---

## 8. PrimitiveImplementationDraft Artifact

### 8.1 Definition

A `PrimitiveImplementationDraft` is a first-class artifact in the **Transformation Domain** that tracks the lifecycle of creating a new primitive.

### 8.2 Schema

```protobuf
message PrimitiveImplementationDraft {
  string id = 1;
  string tenant_id = 2;
  string primitive_type = 3;  // "domain" | "process" | "agent"
  string primitive_id = 4;
  string template_id = 5;
  string code_origin = 6;      // "platform" | "tenant"
  string target_repo = 7;
  repeated string design_refs = 8;  // ProcessDefinition/AgentDefinition IDs

  // Computed by Transformation
  map<string, string> target_namespaces = 9;  // env -> namespace
  string sku = 10;

  // State
  DraftState state = 11;
  string branch_name = 12;
  string pr_url = 13;
  repeated PromotionRecord promotions = 14;
}

enum DraftState {
  DRAFT_STATE_UNSPECIFIED = 0;
  PENDING = 1;
  SCAFFOLDED = 2;
  IN_REVIEW = 3;
  PROMOTED_DEV = 4;
  PROMOTED_STAGING = 5;
  PROMOTED_PROD = 6;
  REJECTED = 7;
}
```

### 8.3 APIs

| API | Purpose |
|-----|---------|
| `CreatePrimitiveImplementationDraft` | Create a new draft |
| `GetPrimitiveDraftFiles` | Get current files from feature branch |
| `UpdatePrimitiveDraftFiles` | Apply patches via deterministic worker |
| `RequestPromotion` | Promote to an environment |
| `GetDraftStatus` | Get current state and audit trail |

---

## 9. NetworkPolicy Patterns

### 9.1 Shared Namespace (`ameide-{env}`)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-tenant-custom-ingress
  namespace: ameide-dev
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    # Allow from same namespace
    - from:
        - podSelector: {}
    # Allow from gateway
    - from:
        - namespaceSelector:
            matchLabels:
              name: envoy-gateway-system
    # Deny from tenant-*-cust namespaces (implicit)
```

### 9.2 Tenant Custom Namespace (`tenant-{id}-{env}-cust`)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-platform-api
  namespace: tenant-acme-dev-cust
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    # Allow to platform API gateway only
    - to:
        - namespaceSelector:
            matchLabels:
              name: envoy-gateway-system
      ports:
        - protocol: TCP
          port: 443
    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

---

## 10. Cross-References

### 10.1 Vision Suite Alignment

| Document | Section | Relationship |
|----------|---------|--------------|
| [470-ameide-vision](470-ameide-vision.md) | §4.3 | Backstage as factory – 478 operationalizes |
| [471-ameide-business-architecture](471-ameide-business-architecture.md) | §4, §6 | Business lifecycle – 478 details primitive creation |
| [472-ameide-information-application](472-ameide-information-application.md) | §4 | Backstage catalog – 478 adds tenant repos |
| [473-ameide-technology](473-ameide-technology.md) | §6 | Multi-tenancy – 478 defines namespace topology |
| [475-ameide-domains](475-ameide-domains.md) | §4.6 | Agent primitives – 478 adds tenant isolation |
| [476-ameide-security-trust](476-ameide-security-trust.md) | §7 | Agent governance – 478 adds draft pattern |
| [467-backstage](467-backstage.md) | §10 | Templates – 478 adds namespace calculation |

### 10.2 Implementation Alignment

| Document | Relationship |
|----------|--------------|
| [443-tenancy-models](443-tenancy-models.md) | SKU definitions used in §4 |
| [465-applicationset-architecture](465-applicationset-architecture.md) | GitOps model extended for tenant repos |
| [310-agents-v2](310-agents-v2.md) | Agent primitive runtime for draft APIs |

---

## 11. Open Questions

1. **Tenant repo provisioning**
   * How is `tenant-{id}-controllers` and `tenant-{id}-gitops` created initially?
   * Backstage template? Manual? Transformation API?

2. **Image registry isolation**
   * Do tenant custom images go to a tenant-specific registry path?
   * `ghcr.io/ameideio/tenant-{id}/controller-name:tag`?

3. **Cost allocation**
   * How are compute costs for `tenant-{id}-*-cust` namespaces attributed?

4. **Draft expiry**
   * How long do unpromoted drafts persist before cleanup?

---

## 12. Implementation Status

| Component | Status | Notes |
|-----------|--------|-------|
| Namespace convention | ✅ Defined | §4 |
| Backstage templates | ⏳ Planned | [467-backstage.md](467-backstage.md) §10.1 |
| PrimitiveImplementationDraft | ⏳ Not implemented | §8 schema defined |
| Draft APIs | ⏳ Not implemented | §8.3 |
| NetworkPolicies | ⏳ Not implemented | §9 patterns defined |
| Tenant repo provisioning | ❓ Open question | §11.1 |


<!-- ═══════════════════════════════════════════════════════════════════════════ -->
<!-- FILE: 479-ameide-extensibility-wasm.md -->
<!-- ═══════════════════════════════════════════════════════════════════════════ -->

Here’s a complete rewritten spec you can drop in as something like
`4xz-ameide-extensibility-wasm.md`.

---

# 479 – Ameide Extensibility (Tier 1 WASM + Tier 2 Primitives)

**Status:** Draft v1
**Owner:** Architecture / Platform
**Audience:** Platform engineers, domain architects, solution architects, security
**Depends on:** 470 (Vision), 471 (Business), 472 (App/Info), 473 (Tech), 478 (Extensions), 476 (Security & Trust)     

---

## 1. Purpose

This document defines Ameide’s **extensibility model**, with two coordinated tiers:

1. **Tier 1 – Small extensions via WASM**
   Agent‑ and human‑authored **WebAssembly (WASM) extensions**, executed by a **shared platform runtime** in `ameide-{env}`, wired into:

   * **Processes (BPMN)** – “extension tasks” in ProcessDefinitions. 
   * **Domains** – explicit extension points (CoC‑style hooks) in Domain primitives. 
   * **Agents** – deterministic tools for Agent primitives.

2. **Tier 2 – Large extensions via primitives (existing 478 model)**
   Full **Domain/Process/Agent primitives** running in **tenant namespaces** (`tenant-{id}-{env}-base` / `tenant-{id}-{env}-cust`) created via **Backstage templates + GitOps**. 

Goals:

* Give tenants and agents a **lightweight way** to add custom business logic without spinning up new services for every small change.
* Keep **transformation** as the owner of design‑time artifacts (ProcessDefinitions, AgentDefinitions, ExtensionDefinitions). 
* Align mentally with **SAP** (in‑app vs side‑by‑side) and **Dynamics X++ Chain of Command** patterns.
* Preserve and extend the **security and tenancy invariants** defined in 470/476/478.  

---

## 2. Positioning & Context

### 2.1 Relationship to 470–473 & 478

This spec sits directly below the Vision + Architecture suite:

* 470 – establishes **Design ≠ Deploy ≠ Runtime** and Transformation as a domain. 
* 471 – defines **business concepts** (tenants, orgs, processes, agents, workspaces). 
* 472 – defines **Domain/Process/Agent primitives**, Transformation design tooling and Transformation Domain primitive. 
* 473 – defines **K8s/GitOps/Backstage/Temporal** tech stack. 
* 478 – defines **Tier 2 tenant extensions**: repo strategy, namespace topology, Backstage templates, PrimitiveImplementationDraft. 

This document **does not replace 478**. Instead:

* Tier 2 (478) remains the model for **new primitives & services**.
* This doc introduces Tier 1 as a **lighter path** for small, often agent‑generated extensions.

### 2.2 Two-tier extensibility at a glance

**Tier 1 – WASM extensions (“small customs”)**

* Unit: `ExtensionDefinition` (design‑time artifact, stored in Transformation). 
* Runtime: `WasmExtensionRuntime` shared service in `ameide-{env}`.
* Invoked from:

  * BPMN Extension Tasks in ProcessDefinitions,
  * Domain extension points,
  * Agent deterministic tools.
* Best for:

  * small validation rules,
  * pricing/discount tweaks,
  * routing/approval decisions,
  * deterministic helper functions for agents.

**Tier 2 – Primitives (“big customs”)**

* Unit: Domain/Process/Agent primitive created via **Backstage**, deployed via GitOps.
* Runtime: services in:

  * `ameide-{env}` for platform product, and
  * `tenant-{id}-{env}-base` / `tenant-{id}-{env}-cust` for tenant code, per 478. 
* Best for:

  * new domains or major process variants,
  * integrations,
  * multi‑endpoint services with their own APIs & storage.
* Runtime representation: `Domain`, `Process`, and `Agent` CRDs reconciled by Ameide operators across platform and tenant namespaces.

Backstage is now clearly **Tier 2‑only** for new primitives; it does not sit in the hot path for small Tier 1 WASM extensions.

### 2.3 Alignment with proto-first & CQRS patterns

Tier 1 and Tier 2 extensions ride on the same substrate described elsewhere:

* Contracts live in `packages/ameide_core_proto` (Buf-managed) and flow into workspace SDKs + GitOps manifests per [472 §2.7](472-ameide-information-application.md#27-contract--implementation--runtime-chain). Extension runtime APIs (`InvokeExtension`, host-call adapters) follow the same Buf/SDK conventions documented in [480](480-ameide-extensibility-wasm-service.md).
* Process/Domain primitives that wrap Tier 1 hooks or Tier 2 tenant code inherit the Watermill CQRS plumbing (commands/events) described in [472 §3.3.1](472-ameide-information-application.md#331-event-driven-cqrs-runtime-watermill) so the event stream stays uniform.
* SDK publishing for extension tooling follows the outer-loop policies in [388](388-ameide-sdks-north-star.md) but internal dev/CI images always consume workspace SDKs (no registry dependency), consistent with [402](402-buf-first-generated-clients.md)/[403](403-python-buf-first-migration-plan.md)/[404](404-go-buf-first-migration.md).

Readers should treat this spec as “how Tier 1 WASM and Tier 2 primitives fit into that existing chain,” not as a new architectural layer.

---

## 3. Conceptual Model for Tier 1 (WASM)

### 3.1 ExtensionDefinition

A **design‑time artifact** managed by the Transformation Domain primitive (Transformation design tooling), analogous to ProcessDefinitions and AgentDefinitions. 

```text
ExtensionDefinition
  id: string
  tenant_id: string
  kind: "process_hook" | "domain_hook" | "agent_tool"
  abi: "json_in_json_out_v1"
  wasm_blob_ref: string        # pointer to object storage
  version: int
  limits:
    cpu_millis: int
    memory_mb: int
    timeout_ms: int
  risk_tier: "low" | "medium" | "high"
  metadata:
    description: string
    created_by: "user:<id>" | "agent:<id>"
    tags: [string]
```

**Kinds:**

1. `process_hook` – invoked by **Process primitive** as an Extension Task in BPMN.
2. `domain_hook` – invoked by **Domain primitive** at explicit extension points (CoC‑style).
3. `agent_tool` – invoked by **Agent primitive** as a deterministic helper.

**ABI**: For v1, all kinds share:

```text
fn run(input_json: bytes) -> output_json: bytes
```

### 3.2 WasmExtensionRuntime (shared platform service)

A **multi‑tenant platform service** per environment:

* Namespace: `ameide-{env}` (platform plane). 
* Owned: **entirely by Ameide**; code lives in platform repo (not tenant repos). 
* Responsibilities:

  1. Resolve `(tenant_id, extension_id, version)` to a WASM module (via `wasm_blob_ref`) stored in the shared **MinIO object storage service** (`data-minio` ApplicationSet component).
  2. Execute `run(input_json)` in a WASM sandbox with:

     * CPU/mem/time limits,
     * No direct FS/network/env access,
     * A small host API (logging + optional host calls).
  3. Enforce host‑call policies:

     * All host calls use **existing Ameide APIs** with **user/tenant context** (see §6.3).

**RPC surface** (internal‑only, used by primitives):

```protobuf
service WasmExtensionRuntime {
  rpc InvokeExtension(InvokeExtensionRequest)
      returns (InvokeExtensionResponse);
}

message ExecutionContext {
  string tenant_id      = 1;
  string org_id         = 2;
  string user_id        = 3;  // "system" or agent id allowed
  string actor_type     = 4;  // USER | AGENT | SYSTEM
  string session_id     = 5;
  string correlation_id = 6;
  string env            = 7;  // dev / staging / prod
  string risk_tier      = 8;  // LOW | MEDIUM | HIGH
}

message InvokeExtensionRequest {
  string extension_id   = 1;
  string version        = 2;  // or "latest"
  ExecutionContext ctx  = 3;
  bytes  input_json     = 4;
}

message InvokeExtensionResponse {
  bytes  output_json    = 1;
  string error_code     = 2;
  string error_msg      = 3;
}
```

### 3.3 Where it sits in the existing architecture

* **Design‑time**: Transformation Domain primitive + Transformation design tooling manage ExtensionDefinitions, versions, and promotions with the same revision/promotion pattern as ProcessDefinitions and AgentDefinitions. 
* **Deploy‑time**: WasmExtensionRuntime is deployed as a **standard platform Helm chart** in `ameide-{env}`, via the same GitOps & ApplicationSet patterns as other platform primitives. 
* **Runtime**:

  * Process primitives, Domain primitives, Agent primitives call `InvokeExtension(...)` with a populated `ExecutionContext`.
  * All durable state, auth, and tenancy enforcement remain in primitives and domains; WASM is “pure function + host calls”.

### 3.4 Deployment approach & operator options

We do **not** need a special Wasm operator for the Tier 1 runtime. The recommended path is to treat `wasm-extension-runtime` like any other platform service:

1. Build a container that embeds Wasmtime/WasmEdge, loads tenant modules from object storage, and exposes `InvokeExtension` over gRPC/HTTP.
2. Package that binary as a **Helm chart** (e.g. `charts/wasm-extension-runtime`) with a Deployment, ClusterIP Service, config for storage buckets and host-call allowlists, and standard resource limits.
3. Add an ArgoCD `Application`/`ApplicationSet` entry so each `ameide-{env}` namespace gets the runtime just like our other primitives. Sync/rollback flows stay identical to the rest of the platform plane.

This keeps all security invariants: code is platform-owned, modules are data, and nothing special runs on the worker nodes.

If we ever need to run Wasm workloads as **pods** (not just within our service), we can install node/runtime operators via ArgoCD-managed Helm charts:

* **KWasm operator** – installs a containerd shim and `RuntimeClass` for WasmEdge/Wasmtime so pods can set `runtimeClassName: wasmedge`. Still alpha in places and only solves node-level execution.
* **SpinKube** – bundles the Fermyon Spin operator + shim; introduces a `SpinApp` CRD and a specific component model that differs from our JSON-in/JSON-out hooks.
* **wasmCloud platform chart** – deploys wasmCloud’s lattice/operator; powerful but opinionated, with its own capability contracts and NATS dependencies.

Those frameworks are useful if we adopt node-level Wasm generally, but they are **optional** for Tier 1. We stay leanest and most controllable by simply shipping our runtime service via Helm + ArgoCD.

### 3.5 Storage substrate (MinIO)

`wasm_blob_ref` objects live in Ameide’s existing **MinIO installation** (`data-minio`, see backlog/380). The storage model is:

* Buckets/prefixes partitioned by tenant (`s3://extensions/<tenant>/<extension>/<version>.wasm`), owned by Transformation but deployed via the shared MinIO chart.
* Access controlled via MinIO service users and policies managed by the same provisioning jobs that serve other workloads. The runtime receives credentials through ExternalSecrets so primitives never handle MinIO keys directly.
* Promotion between environments copies (or re-encrypts) blobs into the destination environment’s MinIO namespace; the `wasm_blob_ref` encodes bucket, key, and checksum so the runtime can verify integrity before caching modules locally.

Using MinIO keeps Tier 1 aligned with the rest of the platform’s artifact storage strategy and avoids inventing a bespoke blob service.

---

## 4. Hook Points

### 4.1 Process hooks (BPMN extension tasks)

**Design‑time**

* Transformation design tooling’s BPMN modeller (React Flow) gains an **“Extension Task”** type. 
* Each Extension Task references:

  * `extension_id`
  * Optional `version` (or `latest`)
  * Input mapping (process variables → JSON)
  * Output mapping (JSON → process variables)

**Runtime**

When a Process primitive (Temporal‑backed) executes an Extension Task:

1. It gathers input variables according to the mapping.
2. Builds `ExecutionContext` from process context (tenant, org, user, actor type, correlation id).
3. Calls `WasmExtensionRuntime.InvokeExtension()`.
4. Writes the result into process variables and continues.

Usage examples:

* Custom **discount/routing rule** per tenant.
* Extra **approval logic** before moving to a next stage.

### 4.2 Domain hooks (CoC‑style)

Each Domain primitive can define explicit extension points, e.g.:

```text
BeforeCreateOrder
AfterCreateOrder
BeforePostInvoice
AfterPostInvoice
```

**Design‑time**

* For each extension point, add a config object in domain config (stored in the domain’s persistence):

```protobuf
message DomainExtensionConfig {
  string extension_id = 1;  // optional
  string version      = 2;
}
```

* Transformation Domain primitive exposes APIs & UIs to manage these configs per tenant.

**Runtime**

At a hook:

1. Domain primitive builds a small JSON view of the current state (e.g. order draft).
2. Builds `ExecutionContext` from the current call:

   * `tenant_id`, `org_id`, `user_id` from JWT / session. 
3. Calls `InvokeExtension(...)`.
4. Interprets the result (e.g. `ok` / `validationMessages` / `derivedFields`) and proceeds or fails the command.

This gives a **Chain‑of‑Command‑like experience** without loading tenant code into the domain process.

### 4.3 Agent tools

An AgentDefinition may reference an extension as a **deterministic tool**, e.g.:

* “Calculate risk score for an opportunity.”
* “Enforce policy for discount suggestion.”

Agent primitive:

1. Calls `InvokeExtension(...)` with an `ExecutionContext` representing the user or agent. 
2. Uses the result as deterministic input to the LLM chain (logged, auditable, replayable).

---

## 5. SAP & Dynamics Analogies

### 5.1 SAP

* **SAP in‑app extensibility / BAdIs**:

  * Small code snippets at pre‑defined hook points inside S/4HANA.
  * **Analogy:** Ameide `process_hook` & `domain_hook` ExtensionDefinitions.
* **SAP side‑by‑side (BTP)**:

  * Full apps running separately, calling S/4 via APIs/events.
  * **Analogy:** Ameide Tier 2 primitives in tenant namespaces (478).

So:

* Tier 1 WASM ≈ **SAP in‑app extension** equivalents, but sandboxed and controlled via Transformation.
* Tier 2 primitives ≈ **SAP BTP side‑by‑side apps**.

### 5.2 Dynamics 365 X++ Chain of Command (CoC)

* CoC = wrap a method with pre/`next()`/post logic inside the same app.
* **Analogy:** Ameide `domain_hook` extensions:

  * Instead of overriding methods in‑process, Domain primitives call a WASM hook before/after key operations.
* Difference:

  * Dynamics runs extensions inside the same CLR sandbox as MS code.
  * Ameide runs extensions in a **separate WASM runtime** with narrow host APIs and strong auth/tenancy controls.

---

## 6. Security & Governance

Tier 1 must fit into 476’s security & trust principles and 478’s extension invariants, with a carefully defined exception for WASM running in `ameide-*`.  

### 6.1 Threat model (incremental risk)

New risks introduced by WASM:

1. **Privilege escalation / data overreach** – extension tries to access data from other tenants/orgs.
2. **Data exfiltration** – extension attempts to leak data via host calls.
3. **Resource abuse** – expensive computations starving the runtime.
4. **Supply chain** – tampered modules or toolchains.
5. **Agent mis‑use** – agent generates unsafe or non‑compliant logic.

### 6.2 Invariants and carve‑outs

From 478:

> Custom code never runs in shared `ameide-*` namespaces; custom services run in `tenant-{id}-{env}-cust` namespaces. 

We **preserve this for Tier 2 primitives**. For Tier 1 we introduce a **narrow carve‑out**:

> **WASM modules are treated as data executed by a sandboxed, platform‑owned runtime in `ameide-{env}`, not as independently deployed services.**

Invariants:

1. **No tenant services in `ameide-*`**

   * Only the **WasmExtensionRuntime** runs there, fully platform‑owned.

2. **WASM sandbox**

   * No direct network, filesystem, or environment access.
   * Limited instruction count / fuel, memory, and wall‑clock time per invocation.

3. **No raw credential access**

   * WASM modules see only:

     * Input JSON,
     * Host functions (strictly controlled),
     * No tokens, secrets, or DB handles.

4. **All side‑effects via existing APIs + user/tenant context**

   * Host calls back into Ameide must:

     * Use existing proto/SDK APIs. 
     * Use the **same context** (tenant, org, user/agent) as the calling primitive.

5. **Deterministic boundary**

   * Primitives remain the sole owners of durable state & authorization; WASM is not allowed to write directly to storage.

### 6.3 Host calls, API, and user context

Inside WASM we expose:

```ts
// conceptual host API
declare function host_log(level: string, msg: string): void;

declare function host_call(
  service: string,
  method: string,
  requestJson: string
): string; // throws on error
```

**Implementation details (in the runtime, not in WASM):**

1. `ExecutionContext` is passed from the primitive to the runtime.
2. When WASM calls `host_call(...)`, the runtime:

   * Checks **allowlists** per `extension_id` and `risk_tier`:

     * `LOW`: no host calls.
     * `MEDIUM`: can call certain read‑only APIs.
     * `HIGH`: may call a wider set, but only under strict governance.
   * Builds a **short‑lived internal token** that encodes:

     * `tenant_id`, `org_id`, `user_id`, `actor_type`, scopes.
   * Calls the target Domain/Process/Agent primitive via the existing SDKs. 
3. Domain/Process/Agent primitives perform **normal auth & RLS**:

   * If context lacks rights, the call fails.

**WASM never sees or creates tokens itself**, and cannot change `tenant_id` / `org_id` / `user_id`.

### 6.4 Risk tiers & promotion rules

* `low` – pure functions (mapping, formatting, simple scoring):

  * No host calls.
  * Auto‑promotion possible for dev with tests.

* `medium` – read‑only enrichment, validations:

  * Limited host calls.
  * Requires human review for staging/prod.

* `high` – pricing, approvals, compliance:

  * Host calls allowed to a curated subset of APIs.
  * Requires multi‑party approval and stronger testing before prod.

Risk tier is a property of the ExtensionDefinition; Transformation Domain primitive enforces **environment‑specific promotion rules**, similar to how ProcessDefinitions/AgentDefinitions are governed. 

### 6.5 Observability & incident response

* Each invocation produces:

  * `tenant_id`, `org_id`, `user_id`, `actor_type`,
  * `extension_id`, version,
  * Hook kind (`process`, `domain`, `agent`),
  * Latency, resource usage,
  * Outcome (OK, business error, technical error).
* SRE:

  * Per‑extension SLOs and error budgets.
  * Circuit breakers to disable or roll back a specific extension version if it misbehaves.

---

## 7. Lifecycle of an Agent‑Developed Extension

This section focuses on the **Tier 1 path** for an extension primarily written by an agent, with humans in the loop.

### 7.1 Actors

* **Tenant stakeholder** – expresses requirement in natural language.
* **Transformation Agent primitive** – helps design the extension and wiring. 
* **Transformation Domain primitive** – stores ExtensionDefinitions, governs promotion. 
* **Compile Worker** – deterministic service that compiles source → WASM and runs validations.
* **WasmExtensionRuntime** – executes modules at runtime.
* **Process/Domain/Agent primitives** – call extensions through the runtime.

### 7.2 Stages

#### Stage 0 – Requirement capture

Example requirement:

> “For ACME, deals over 50k with low margin must go to an extra approval step, and we need a custom risk score combining margin and churn risk.”

Captured as:

* Backlog item and/or initiative in Transformation domain (Change‑the‑business process). 

#### Stage 1 – Design in Transformation

The Transformation agent + humans:

1. Identify the hook points:

   * BPMN: insert Extension Task in L2O ProcessDefinition before approval. 
   * Domain: optional `BeforeCreateOrder` hook.
2. Create an `ExtensionDefinition` draft:

   * `kind = "process_hook"`,
   * Input/output JSON schema,
   * Initial risk tier (likely `medium` or `high`),
   * Description.

All stored in Transformation Domain primitive as draft revision.

#### Stage 2 – Agent writes source code

The agent uses its toolset to produce a minimal function in an allowed language (Rust, AssemblyScript, TinyGo) with the `run(input_json) -> output_json` ABI.

It attaches this **source code** to the `ExtensionDefinition` draft via Transformation APIs (not direct Git).

#### Stage 3 – Compile & validate

Compile Worker:

1. Fetches the draft.
2. Compiles source → WASM using a pinned toolchain.
3. Runs:

   * ABI checks (exported `run` function),
   * Static policy checks,
   * Unit tests with synthetic + example inputs.
  4. If successful:

     * Stores the WASM blob in a tenant-scoped MinIO bucket/prefix (`s3://extensions/<tenant>/<extension>/<version>.wasm`).
     * Links `wasm_blob_ref` + new version to the ExtensionDefinition.

On failure, the draft remains in a failed state with detailed compile/test errors.

#### Stage 4 – Review & promotion

Depending on `risk_tier`:

* `low`:

  * Auto‑promote to `dev` environment after compile/tests.
* `medium`/`high`:

  * Human reviewers (process owner, architect) inspect:

    * Spec & code summary,
    * Test coverage & sample outputs,
    * Risk assessment.
  * Approve or request changes.

Promotion steps:

* `draft → in_review → promoted_dev → promoted_staging → promoted_prod`

Each promotion updates the runtime configuration for that env.

#### Stage 5 – Deploy to runtime

On promotion to an env:

1. Transformation emits `ExtensionVersionPromoted(...)` event.
2. Platform GitOps pipeline updates WasmExtensionRuntime Helm values for that env:

   * Known extensions (IDs/versions),
   * Resource limits,
   * Risk tier & host‑call policies.
3. ArgoCD syncs; runtime config reloads.

No new primitives/services are deployed; the runtime simply becomes aware of the new extension/version.

#### Stage 6 – Invocation in business flow

* **Process**: L2O Process primitive hits the Extension Task, calls runtime, uses result to drive routing/approval.
* **Domain**: OrdersDomain primitive `BeforeCreateOrder` calls runtime to run custom validations.
* **Agent**: L2O Agent primitive calls runtime to compute a deterministic risk score.

Everything is observable and tied back to:

* `tenant_id`, `org_id`, `user_id`,
* `extension_id`, version,
* Process/domain/agent context.

#### Stage 7 – Evolution & rollback

To change behaviour:

1. Agent/human creates new revision of ExtensionDefinition with updated source.
2. Repeat compile → review → promotion.
3. Older versions remain available; config supports:

   * rolling back per env, and/or
   * pinning a specific process to a specific version if needed.

---

## 8. Tier 2 Primitives: Confirmed Status

Tier 2 remains exactly as specified in 478:

* **Namespace model**:

  * `ameide-{env}` – shared product plane.
  * `tenant-{id}-{env}-base` – dedicated product plane for Namespace/Private.
  * `tenant-{id}-{env}-cust` – custom code for all SKUs. 
* **Repositories**:

  * Platform code in `ameide-platform` + `ameide-gitops`.
  * Tenant code in `tenant-{id}-controllers` + `tenant-{id}-gitops`. 
* **Backstage templates** and `PrimitiveImplementationDraft` remain the path for new primitives and services. 

Tier 1 (this doc) is additive and does **not** change any Tier 2 rules.

---

## 9. Implementation Backlog (High‑level)

Short version of the work needed to make Tier 1 real.

### Epic EXT‑WASM‑01 – Core runtime & artifact model

* Define `ExtensionDefinition` in Transformation protos + persistence.
* Implement Compile Worker (source → WASM, tests, validations).
* Implement WasmExtensionRuntime service (shared chart, RPC interface, sandbox, host API).
* Wire extension metrics into existing observability stack.

### Epic EXT‑WASM‑02 – Hooking into processes & domains

* Extend BPMN/Transformation design tooling modeller with “Extension Task” type.
* Teach Process primitive to invoke runtime at Extension Tasks.
* Add extension point pattern to at least one reference Domain primitive (e.g. Orders).
* Provide SDK helpers for primitives to call WasmExtensionRuntime with ExecutionContext.

### Epic EXT‑WASM‑03 – Agent lifecycle

* Transformation APIs for managing ExtensionDefinition drafts, source upload, promotion.
* Agent tools for:

  * Proposing extension specs,
  * Generating code,
  * Triggering compile & tests,
  * Requesting promotion.
* Basic UI for human review & approval.

### Epic EXT‑WASM‑04 – Security & governance

* Implement sandboxing and host‑call policy enforcement.
* Integrate risk tiers into Transformation governance flows.
* Add guardrails for agent behaviour (max risk tier per agent, env limitations).
* Define SRE playbooks & circuit breakers for extensions.

### Epic EXT‑WASM‑05 – SAP/Dynamics parity examples

* Create reference examples:

  * SAP “in‑app”‑style pricing BAdI mapped to process/domain hooks.
  * Dynamics CoC “before/after validate” mapped to a domain hook.
* Document these for sales/solution teams.

---

## 10. Performance Envelope & Guardrails

Tier 1 will only feel “slow” if we design it poorly. This section captures the latency expectations and the rules that keep the runtime snappy.

### 10.1 What the extra hop costs

**Primitive → WasmExtensionRuntime → WASM module** adds the following work:

* Proto marshal/unmarshal – microseconds on each side.
* In‑cluster gRPC hop over warm HTTP/2 connections – typically **1–5 ms p50–p95**, occasionally ~10 ms in a noisy cluster.
* WASM function execution for a “small rule” – microseconds to sub‑millisecond.
* JSON encode/decode inside the runtime – microseconds to sub‑millisecond.

So the incremental cost of an extension call is usually **~1–5 ms**, which sits in the same ballpark as any other intra‑cluster primitive/domain call. For contrast:

* Domain primitive → Postgres + RLS + index scan: **5–30 ms**.
* Process primitive → Domain primitive + Temporal activity hop: **dozens of ms**.

### 10.2 Why that cost is usually acceptable

* Process primitives are **Temporal workflows** that complete over seconds to hours; inserting a 5 ms policy hook at a stage boundary is negligible compared to human latency.
* Domain primitives already pay DB and network costs. Budgeting **50–150 ms** for a domain command leaves plenty of headroom for one extension call, especially if the WASM result saves follow‑up hops.
* The platform prioritises **clarity, determinism, and tenant isolation** (“Design ≠ Deploy ≠ Runtime”) over shaving every millisecond, so the shared runtime hop is an intentional, governable tradeoff.

### 10.3 When it will slow things down

We still need to treat Tier 1 like an RPC hop, not an in‑process helper. Problems arise when:

1. **Hot, chatty UI validations** – calling an extension per keystroke or per line item causes dozens of gRPC → WASM → host round trips, dragging down perceived latency.
2. **Loops over large collections** – iterating 500 invoice lines and invoking the runtime for each one burns ~2.5 s just on extension hops, before any real work is done.
3. **Chatty host calls from inside WASM** – making sequential `host_call`s (“get customer”, “get last 10 orders”, “get discounts”) multiplies DB and network latency and defeats the purpose of keeping logic small.

### 10.4 Design rules that keep it fast

1. **Coarse‑grained contracts** – primitives gather all required inputs (full order header, lines, risk flags, etc.) and call the runtime **once per business decision**, not per field or row.
2. **Minimise host calls** – defaults: `low` tier = none, `medium` = 0–1 read‑only call, `high` = tightly curated list for approvals/pricing. Let primitives orchestrate IO; WASM stays mostly pure logic.
3. **Cache and preload modules** – WasmExtensionRuntime keeps modules, host connections, and worker pools warm so only the gRPC hop and function execution remain on the hot path.
4. **Co‑locate when needed** – latency‑sensitive primitives can run near the runtime (node affinity or Unix Domain Socket mode) without changing the security model.
5. **Promote hot paths to Tier 2** – if a “custom rule” grows into logic that runs per keystroke, needs multiple DB calls, or demands independent scaling/SLOs, it should graduate into a dedicated primitive (Tier 2) instead of stressing Tier 1.

We will back these rules with linting, modelling guardrails, and latency histograms per `extension_id` so teams see when an extension is drifting into “chatty helper” territory.

---

That’s the full rewritten specification with your updated assumptions:

* **Tier 2**: tenant-scope namespaces and Backstage primitive model from 478 remain untouched.
* **Tier 1**: a shared platform WASM runtime in `ameide-{env}` for small extensions, wired into processes, domains, and agents, with the latency guardrails captured in §10.
* **Host calls** from WASM always go through existing APIs using the same user/tenant context and authorization model as the rest of the platform.

---

## 11. Cross-References

| Document | Relationship |
|----------|-------------|
| [472-ameide-information-application.md](472-ameide-information-application.md) | Defines `ExtensionDefinition` as a first-class artifact (§2.5) |
| [473-ameide-technology.md](473-ameide-technology.md) | Introduces `extensions-runtime` as a platform service (§3.3.5) |
| [476-ameide-security-trust.md](476-ameide-security-trust.md) | Captures the sandbox carve-out and host-call policies (§6–§7) |
| [480-ameide-extensibility-wasm-service.md](480-ameide-extensibility-wasm-service.md) | Implements the shared runtime described in this doc |


<!-- ═══════════════════════════════════════════════════════════════════════════ -->
<!-- FILE: 480-ameide-extensibility-wasm-service.md -->
<!-- ═══════════════════════════════════════════════════════════════════════════ -->

# 480 – Ameide Extensibility WASM Service

**Status:** Draft  
**Owner:** Architecture / Platform  
**Depends on:** 479 (Extensibility model), 478 (Service catalog), 476 (Security), 472 (Controller contracts), 362 (Unified Secret Guardrails v2), 365 (Buf SDKs v2), 408 (Workspace-first Ring 2), 430 (Unified Test Infrastructure), 434 (Unified Environment Naming), 441 (Networking), 447 (ArgoCD rollout phases), 465 (ApplicationSet architecture)

---

## 1. Purpose

Establish the shared `extensions-runtime` service that runs Tier 1 WASM extensions inside `ameide-{env}`. The service must:

1. Expose the `WasmExtensionRuntime.InvokeExtension` RPC for Process/Domain/Agent primitives.
2. Resolve extension artifacts, execute them in a sandbox, and enforce risk-tier controls.
3. Ship as a standard platform workload (Helm + ArgoCD) with dev/release Dockerfiles similar to other services.

---

## 2. Scope

- **In scope**
  - New service under `services/extensions-runtime` (Go preferred) with README, Tilt target, and pairing Dockerfiles.
  - Runtime proto (`packages/ameide_core_proto/ameide/extensions/v1/runtime.proto`) plus SDK bindings.
  - Module storage/caching, Wasmtime-based sandbox executor, and host-call bridge leveraging shared MinIO buckets per 479/362 guardrails.
  - Config hydration from Transformation promotion events, observability, and guardrails (limits, SLO telemetry).
  - Helm chart + GitOps wiring for each platform environment (per the dual ApplicationSet model). Although tenant-authored WASM runs in `ameide-{env}`, modules are treated purely as data executed by a sandboxed, Ameide-owned runtime; all host calls remain scoped by the caller’s tenant/org/user and reuse existing primitive APIs, keeping the 478 namespace invariant intact.

- **Out of scope**
  - Transformation UI/UX for authoring ExtensionDefinitions (handled by EXT-WASM-03).
  - Backstage templates for tenant-tier services (covered by 478).

### 2.1 Links into the architecture suite

This service is the concrete realization of the Tier 1 WASM model from [479-ameide-extensibility-wasm.md](479-ameide-extensibility-wasm.md). It plugs into:

* Transformation / Transformation design tooling as the executor for `ExtensionDefinition` artifacts (472 §3.5).
* The Service Layer as a platform primitive in `ameide-{env}` (473 §3).
* The Security & Trust invariants for sandboxed tenant logic (476 §6–§7).
* The tenant extension story alongside Tier 2 primitives (478).
* The operator-managed primitive plane: Domain/Process/Agent CRDs call `InvokeExtension` via SDK helpers, regardless of whether the primitive is platform or tenant-owned.

---

## 3. High-Level Plan

1. **Proto & SDK Foundations**
   - Author `runtime.proto` per §3.2 of 479 (ExecutionContext, InvokeExtension, responses) and keep it inside the shared Buf module (365).
   - Regenerate `ameide_sdk_go` / `ameide_sdk_ts` / `ameide_sdk_py` clients with helper functions to build contexts derived from the primitive’s authenticated `RequestContext`. Services consume the new RPC only via SDKs (no direct proto imports) to stay aligned with the Buf SDKs v2 policy (365). Go SDK now exposes `InvokeExtensionJSON` + `ExecutionContext` options so primitives avoid hand-written RPC glue.

2. **Service Scaffold**
   - Create `services/extensions-runtime/` with README, basic main.go, health endpoints, configuration stubs.
   - Add `Dockerfile.dev` / `Dockerfile.release` mirroring primitive skeleton patterns (Go multi-stage) and copying workspace SDK packages per the Ring 1 / Ring 2 rules (408).

3. **Module Management**
   - Implement `ModuleStore` that pulls WASM blobs via `wasm_blob_ref`, verifies integrity, and caches by `(extension_id, version)` with eviction policies. Objects live in the GitOps-managed MinIO deployment (`data-minio`); respect the shared ExternalSecrets defined in backlog/362 and the storage pattern documented in 479 §3.5. Reference `380-minio-config-overview.md` for bucket tenancy so the runtime reuses the existing MinIO service rather than provisioning bespoke storage.
   - Support refreshing config from promotion events (watch storage, SSE, or queue).

4. **Sandbox Execution**
   - Integrate Wasmtime (or WasmEdge) with per-invocation CPU/memory/time enforcement.
   - Implement host functions (`host_log`, `host_call`) with deterministic behavior.

5. **Host Bridge & Policy**
   - Enforce risk-tier allowlists for host calls.
   - Emit short-lived internal tokens bound to the incoming ExecutionContext (mirroring primitive auth claims), route to allowed Ameide services via existing SDKs, and block disallowed calls. Registry-driven host-call adapters (e.g., `transformation.get_transformation`) replace the interim HTTP shim and reuse Ameide SDK clients.

6. **Observability & Controls**
   - Instrument metrics per invocation: latency, resource usage, error type, tenant/extension labels.
   - Add circuit-breaker hooks for SRE to disable an extension version or risk tier.
   - Follow backlog/434/441 labeling conventions (`ameide.io/environment`, `ameide.io/tier`, `ameide.io/tenant-kind`, etc.) so dashboards, policies, and ApplicationSets can slice data consistently across environments/tenants.

7. **Deployment Assets**
   - **Service activities (this repo):** Add Helm chart scaffolding, ensure `Tiltfile` exposes a target for local development (workspace-first build), and wire GitHub workflows/cd-service-images entries so Docker images publish automatically. These assets belong in this repository because they impact developer workflows and guardrails (408).
   - **GitOps activities (platform repo):** Register ApplicationSet entries so each `ameide-{env}` namespace deploys the runtime. Follow backlog/465 for file placement and backlog/447 for rollout phases (Platform band, e.g., rollout-phase `350`), ensuring RollingSync steps pick it up correctly. Keep GitOps-specific changes in the GitOps repo to preserve separation of duties.
   - **Image promotion helper (ameide-core):** `scripts/dev/vendor_service_images.sh` re-tags any existing `ghcr.io/ameideio/<service>:<tag>` digest to aliases such as `:main`. Use it from this repo to unblock deployments when a pod references a tag that hasn’t been published yet (e.g., `extensions-runtime:main`).

---

## 4. Implementation Status — Service Repo (Dec 2025)

| Area | Status | Notes |
|------|--------|-------|
| **Proto & SDKs** | ✅ Complete | `packages/ameide_core_proto/src/ameide_core_proto/extensions/v1/runtime.proto` landed; Go/TS/Python SDKs expose `WasmExtensionRuntimeService`, and `scripts/dev/check_sdk_alignment.py` passes locally. Go SDK now includes `InvokeExtensionJSON` + ExecutionContext helpers. |
| **Service scaffold** | ✅ Complete | `services/extensions-runtime/` contains config package, Wasmtime executor, MinIO-backed ModuleStore, gRPC server wiring, README, and dual Dockerfiles (workspace-first). |
| **Testing** | ✅ Mock mode, ⚠ cluster | Mock + cluster-aware integration suite (`__tests__/integration`) follows the unified runner (430). Cluster env vars still need documentation/pipeline wiring before running against `ameide-dev`. |
| **Workspace-first build/test guardrails** | ✅ Verified | `Dockerfile.dev` / `.release` copy workspace SDK packages (408/405). Local `go test ./...` and mock-mode integration tests pass. |
| **SDK alignment (365 v2)** | ✅ | SDK regeneration landed (Go/TS/Py). CI now fails only if host-call integration/GitOps wiring is missing. |
| **Docker build** | ✅ | `docker build` succeeds for both dev/release images; GitHub workflow `extensions-runtime.yml` runs unit + integration tests and both Dockerfiles on PRs. CGO is enabled for Wasmtime. |
| **Observability/telemetry** | ✅ baseline | OTEL wiring mirrors inference-gateway pattern (contextual logging, tenant interceptor, OTLP exporter). Metrics/diagnostics still pending. |
| **Host-call bridge & Wasmtime** | ⚠ Initial adapters | Wasmtime executor runs modules with `alloc/dealloc/run` exports and `host_log/host_call` ABI. Registry-driven host bridge instantiates adapters via the Ameide SDK (first adapter: `transformation.get_transformation`). Need token minting + additional adapters. |
| **MinIO module management** | ✅ | LRU cache + singleflight loader backed by MinIO client; config struct aligned with 362. Need config reload events from Transformation. |
| **Tilt target & Helm scaffolding** | ⏳ Not started | Need Tilt resource + chart skeleton in this repo so dev workflows can build/test locally; blocks `cd-service-images` wiring. |
| **Release packaging (cd-service-images)** | ⚠ In progress | `.github/workflows/cd-service-images.yml` now includes `extensions-runtime`, but we still need to verify GitHub environments/secrets for promotion and document the workflow alongside other services. |
| **Primitive integration** | ⚠ Helper implemented | A mock-mode integration test spins up Wasmtime + host-call adapter, proving primitives can invoke extensions through the helper. Need to wire a real Domain/Process/Agent primitive path and add negative sandbox tests. |

### 4.1 Implementation Status — GitOps Repo

| Area | Status | Notes |
|------|--------|-------|
| **ApplicationSets & rollout phase** | ⏳ Not started | Must add Helm chart + ApplicationSet entries per environment, apply rollout phase ≈350, and ensure RollingSync picks it up alongside other platform services. |
| **Secrets & hydration** | ⏳ Not started | Need ExternalSecret references for MinIO credentials + host-call adapters, plus a config sync path for Transformation promotions. |
| **Observability/SLO wiring** | ⏳ Not started | Dashboards, runtime labels, and alert policies must be registered in GitOps alongside deployment manifests. |

Next mileposts:
1. **Service repo:** Finish HostBridge policy work (token minting, additional adapters), wire a real primitive path via the new SDK helper, add Tilt target + Helm scaffolding, and extend negative sandbox tests.
2. **GitOps repo:** Create Helm/ApplicationSet assets with rollout metadata, wire ExternalSecrets + config hydration jobs, and document runtime SLO dashboards/disable switches.

---

## 5. Acceptance Criteria

- Controllers can call `InvokeExtension` via generated SDKs and receive sandboxed results.
- The runtime enforces limits declared in `ExtensionDefinition.limits` and risk-tier host-call policies.
- Dev/release images build reproducibly in CI (Tilt + production).
- Helm chart deploys successfully to a test `ameide-dev` namespace with ArgoCD.
- Basic observability dashboards/slo alerts exist for invocation latency/error rates.
- Integration tests follow the dual-mode (mock/cluster) contract in backlog 430 (runner script, fixtures, fail-fast env validation).
- Runtime manifests include the required namespace/pod labels so networking guardrails from backlog 441 apply automatically.
- Benchmark shows `InvokeExtension` adds <10 ms p95 latency for a pure function with small payloads (baseline on `ameide-dev`), and negative sandbox tests prove denied operations (network/FS/CPU abuse) are blocked.
- Operators can disable or degrade individual extensions via config/feature flags when risk thresholds are exceeded (ties into 476 incident response expectations).
- Sandbox enforcement, host-call allowlists, and circuit-breaker controls collectively satisfy the Tier 1 carve-out described in [476-ameide-security-trust.md](476-ameide-security-trust.md) §6–§7.

---

## 6. Open Questions / Follow-ups

1. Preferred storage backend for WASM blobs (S3-compatible bucket vs database) and caching invalidation mechanism.
2. How Transformation publishes promotion events (Webhook? Kafka?) for the runtime to consume.
3. Whether we need a lightweight CLI for simulating extensions locally for agent developers.


<!-- ═══════════════════════════════════════════════════════════════════════════ -->
<!-- FILE: 481-service-catalog.md -->
<!-- ═══════════════════════════════════════════════════════════════════════════ -->

# 478 – Service Catalog

**Status:** In Progress
**Priority:** High
**Complexity:** Large
**Created:** 2025-12-07
**Updated:** 2025-12-07

> **Cross-References (Vision Suite)**:
>
> | Document | Relationship |
> |----------|-------------|
> | [470-ameide-vision.md](470-ameide-vision.md) | §5 – Domain/Process/Agent/UISurface primitive model |
> | [472-ameide-information-application.md](472-ameide-information-application.md) | §2 – Domain/Process/Agent primitive definitions |
> | [473-ameide-technology.md](473-ameide-technology.md) | §2.3 – Backstage as factory for primitives |
> | [475-ameide-domains.md](475-ameide-domains.md) | §3 – Domain portfolio |
> | [467-backstage.md](467-backstage.md) | §10 – Software Templates for primitives |

---

## 1. Executive Summary

This document defines the **Service Catalog** architecture: a flat structure for organizing Ameide primitives aligned with Backstage, Buf, and Temporal patterns.

**Key concepts:**
- **service_catalog/** – Single folder containing all primitive types
- **_controller/** – Base definitions (contracts + base types) under each primitive type (folder name unchanged for now)
- **{name}/** – Concrete implementations alongside `_controller`

This structure enables Backstage to scaffold new primitives via Software Templates while keeping definitions and implementations co-located. The architectural rationale for the proto → implementation → runtime chain lives in [472-ameide-information-application.md §2.7](472-ameide-information-application.md); this document focuses on how that chain maps to source folders and GitOps repos. Remember that proto contracts live in `packages/ameide_core_proto/` (the service catalog merely references those modules), so the catalog is an implementation detail rather than a new architecture center of gravity.

---

## 2. Folder Structure

```
ameide-core/
├── service_catalog/
│   ├── domains/                        # Domain primitives
│   │   ├── _controller/                # Base contract + helpers
│   │   ├── transformation/             # Transformation Domain primitive
│   │   ├── platform/                   # Platform Domain primitive
│   │   ├── sales/                      # Sales Domain primitive
│   │   └── ...
│   ├── processes/                      # Process primitives
│   │   ├── _controller/                # Base contract + helpers
│   │   ├── togaf_adm/                  # TOGAF ADM Process primitive
│   │   ├── agile/                      # Agile/Scrum Process primitive
│   │   ├── l2o/                        # Lead-to-Opportunity
│   │   └── ...
│   └── agents/                         # Agent primitives
│       ├── _controller/                # Base contract + helpers
│       ├── architect/                  # Architect Agent primitive
│       ├── sales_clerk/                # Sales Clerk Agent primitive
│       └── ...
│
├── services/                           # LEGACY + non-primitive workloads
│   ├── www_ameide/                     # Next.js UI (NOT a primitive)
│   ├── www_ameide_platform/            # Platform UI (NOT a primitive)
│   └── ...
│
└── packages/                           # Shared SDKs (unchanged)
    └── ameide_core_proto/              # Existing proto structure stays as-is
```

---

## 3. Base Definitions (_controller/)

**Purpose:** Contracts and thin base abstractions for each primitive type (folder name `_controller` retained for now).

### 3.1 What goes here (future)

- Buf proto contracts (`ameide.primitives.domain.v1`, etc.)
- Base interfaces/helpers (`Domain primitiveBase`, `Process primitiveBase`, `Agent primitiveBase`)
- Common middleware (auth, tenant context, logging)

### 3.2 What does NOT go here

- Heavy business/runtime logic
- Domain-specific messages (Order, Invoice, etc.)
- Design-time artifacts (ProcessDefinitions, AgentDefinitions)

### 3.3 Future proto structure

```
service_catalog/
├── domains/_controller/
│   └── v1/
│       └── primitive.proto        # GetPrimitiveInfo, HealthCheck, ListCapabilities
├── processes/_controller/
│   └── v1/
│       └── primitive.proto        # StartProcess, AbortProcess, QueryProcess
└── agents/_controller/
    └── v1/
        └── primitive.proto        # InvokeAgent, StreamAgentResponse
```

---

## 4. Primitive Implementations

**Purpose:** Concrete primitive implementations scaffolded via Backstage.

### 4.1 Primitive types

| Type | Responsibility | Runtime |
|------|---------------|---------|
| **Domain primitive** | Own entities, CRUD, business rules | gRPC service + Postgres |
| **Process primitive** | Execute ProcessDefinitions | Temporal workflows |
| **Agent primitive** | Execute AgentDefinitions | LangGraph/LLM runtime |

### 4.2 Future primitive folder structure

Each primitive folder will contain:

```
service_catalog/domains/{name}/
├── src/                            # Service source code
├── proto/                          # Domain-specific protos (optional)
├── Dockerfile.dev
├── Dockerfile.release
├── catalog-info.yaml               # Backstage Component registration
├── Chart.yaml                      # Helm chart (or reference to charts/)
└── README.md
```

---

## 5. Proto Strategy

### 5.1 Current state (unchanged)

- `packages/ameide_core_proto/` – Central platform protos
- `packages/ameide_sdk_*` – Generated SDKs (TS, Go, Python)

### 5.2 Open question: Tenant-specific protos

Primitive-specific protos may diverge per tenant. If tenants customize their Domain primitives (e.g., custom Sales fields), they get different API surfaces.

**Options to explore:**

| Option | Description | Trade-offs |
|--------|-------------|------------|
| All protos stay central | Primitives extend base types, no tenant variation | Simple but inflexible |
| Primitive protos in service_catalog/ | Each primitive owns its proto, SDK generated per primitive | More flexible but complex generation |
| Tenant-specific SDKs | BSR modules per tenant, tenant-specific SDK packages | Maximum flexibility but operational complexity |

This decision is deferred until we understand tenant customization requirements better.

---

## 6. Backstage Integration

### 6.1 Software Templates

Backstage templates scaffold into `service_catalog/`. Templates are implemented in `_controller/` directories:

```
service_catalog/
├── domains/_controller/
│   ├── template.yaml              # Backstage template definition
│   └── skeleton/                  # Scaffolding files
│       ├── catalog-info.yaml      # Component registration (Jinja)
│       ├── README.md              # Documentation (Jinja)
│       ├── Dockerfile.dev         # Dev container (Jinja)
│       ├── Dockerfile.release     # Production container (Jinja)
│       ├── src/.gitkeep
│       └── migrations/.gitkeep
├── processes/_controller/
│   ├── template.yaml
│   └── skeleton/
│       ├── catalog-info.yaml
│       ├── README.md
│       ├── Dockerfile.dev
│       ├── Dockerfile.release
│       └── src/{workflows,activities}/.gitkeep
└── agents/_controller/
    ├── template.yaml
    └── skeleton/
        ├── catalog-info.yaml
        ├── README.md
        ├── Dockerfile.dev
        ├── Dockerfile.release
        ├── pyproject.toml
        └── src/{tools,prompts}/.gitkeep
```

#### Template parameters

| Template | Key Parameters |
|----------|----------------|
| **Domain primitive** | `domain_id`, `owner`, `language` (ts/go), `database_plan` (shared/dedicated) |
| **Process primitive** | `process_id`, `owner`, `language` (ts/go), `temporal_task_queue`, `process_definition_ref` |
| **Agent primitive** | `agent_id`, `owner`, `default_model`, `agent_definition_ref` |

#### Template actions

All templates use the same action sequence:
1. `fetch:template` – Copy skeleton with Jinja substitution
2. `publish:github:pull-request` – Create PR to `ameide-core`
3. `catalog:register` – Register Component in Backstage catalog

### 6.2 Catalog entities

Each primitive registers as a Backstage Component with appropriate type:

| Primitive Type | Backstage Component Type |
|----------------|--------------------------|
| Domain primitive | `domain-primitive` (future; currently `domain-controller`) |
| Process primitive | `process-primitive` (future; currently `process-controller`) |
| Agent primitive | `agent-primitive` (future; currently `agent-controller`) |

---

## 7. Relationship to Existing Concepts

| Concept | Location | Notes |
|---------|----------|-------|
| ProcessDefinitions | Transformation Domain primitive | Design-time artifacts (BPMN from React Flow) |
| AgentDefinitions | Transformation Domain primitive | Design-time artifacts (tools, policies) |
| UAF | UI layer | Calls Transformation APIs, no own storage |
| Graph | Read-only projection | Fed by primitive events |
| Domain-specific protos | TBD | Central vs per-primitive decision pending |

---

## 8. Relationship to services/

The existing `services/` folder contains legacy services that predate this architecture:

| Current Service | Future Location | Notes |
|-----------------|-----------------|-------|
| `services/platform/` | `service_catalog/domains/platform/` | Migrate when ready |
| `services/agents/` | `service_catalog/domains/agents/` | Agents control plane |
| `services/workflows_runtime/` | Process primitive infrastructure | Temporal worker |
| `services/www_ameide/` | Stays in `services/` | UI, not a primitive |
| `services/www_ameide_platform/` | Stays in `services/` | UI, not a primitive |

**Non-primitives stay in `services/`:**
- Next.js UIs
- Backstage itself
- Infrastructure helpers
- Utility services

---

## 9. Migration Plan

### Phase 1: Foundation (complete)
- [x] Create folder structure with `.gitkeep` files
- [x] Create this spec document
- [x] Implement Backstage templates for all primitive types:
  - [x] Domain primitive template (`service_catalog/domains/_controller/`)
  - [x] Process primitive template (`service_catalog/processes/_controller/`)
  - [x] Agent primitive template (`service_catalog/agents/_controller/`)

### Phase 2: Base contracts
- [ ] Define base proto contracts in `service_catalog/{domains,processes,agents}/_controller/`
- [ ] Update Buf workspace
- [ ] Generate base SDK types

### Phase 3: First primitive
- [ ] Implement Transformation Domain primitive in `service_catalog/domains/transformation/`
- [ ] Create Backstage template for Domain primitive
- [ ] Validate GitOps deployment pattern

### Phase 4: Migrate existing
- [ ] Migrate `services/platform/` → `service_catalog/domains/platform/`
- [ ] Migrate `services/agents/` → `service_catalog/domains/agents/`
- [ ] Migrate `services/workflows_runtime/` → Process primitive infrastructure

### Phase 5: Expand
- [ ] Create Process primitive for L2O/O2C
- [ ] Create Agent primitive templates
- [ ] Deprecate legacy `services/` paths

---

## 10. Open Questions

1. **Tenant-specific protos/SDKs**: If tenants customize primitives, do they need tenant-specific SDKs? How does this interact with BSR?

2. **Proto location**: Should primitive-specific protos live in `service_catalog/` or stay centralized in `packages/ameide_core_proto/`?

3. **SDK generation**: Per-primitive SDK generation vs monolithic SDK with all primitives?

4. **Backstage template versioning**: How do we version primitive templates as the base contracts evolve?

---

## 11. Non-Goals

- Not redefining UAF/Transformation/Graph concepts (see [472](472-ameide-information-application.md))
- Not moving design-time artifacts out of Transformation Domain primitive
- Not killing `services/` for non-primitive workloads (UIs, Backstage, infra)
- Not implementing primitives in this phase (scaffolding only)

---

## 12. Decision Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2025-12-07 | Flat structure: `service_catalog/{domains,processes,agents}/` with `_controller/` for base definitions | Simpler hierarchy; co-locates definitions with implementations |
| 2025-12-07 | Keep existing `packages/ameide_core_proto/` unchanged | Minimize disruption; proto strategy TBD |
| 2025-12-07 | Non-primitives stay in `services/` | UIs and infrastructure don't fit primitive model |
| 2025-12-07 | Backstage templates implemented | Domain primitive, Process primitive, Agent primitive templates with skeleton/ directories |
| 2025-12-07 | Agent primitive is Python-only | LangGraph requirement; Domain primitive/Process primitive support TS or Go |


<!-- ═══════════════════════════════════════════════════════════════════════════ -->
<!-- FILE: 482-adding-new-service.md -->
<!-- ═══════════════════════════════════════════════════════════════════════════ -->

# 481 – Adding a New Service (Reference Checklist)

**Status:** Draft  
**Owner:** Architecture / Platform Enablement  
**Purpose:** Provide a single checklist and pointer map for spinning up a new Ameide service (platform service or primitive) with all cross-cutting requirements (security, testing, GitOps, documentation) enforced from day one.

---

## 1. When to use this backlog

Use this checklist whenever you create a net-new service under `services/` or `service_catalog/`. It consolidates the required work from related backlogs so teams don’t miss key steps (Dockerfiles, AppSets, secrets, tests, documentation, etc.).

---

## 2. Topic Checklist & Source Backlogs

| Topic | Questions to answer | Primary references |
|-------|--------------------|--------------------|
| **Architecture & positioning** | Is this a primitive (Tier 2) or a platform service? How does it map to the extensibility model? | `481-service-catalog.md`, `479-ameide-extensibility-wasm.md`, `480-ameide-extensibility-wasm-service.md` |
| **Scaffolding & repo layout** | Which folders/files must exist (README, Dockerfiles, Tilt targets, catalog info)? | `service_catalog/*/_controller/skeleton/README.md`, `service_catalog/*/_controller/skeleton/Dockerfile.*`, `services/README.md` |
| **Dockerfiles & build pipelines** | Do we have dev/release Dockerfiles aligned with repo conventions? Are we leveraging multi-stage builds? | `service_catalog/.../Dockerfile.dev`, `Dockerfile.release`, `services/platform/Dockerfile.dev` (patterns) |
| **Secrets & configuration** | What secrets/config does the service need? Are ExternalSecrets defined (zero inline secrets per guardrail)? Does Vault bootstrap cover it? | `362-unified-secret-guardrails-v2.md`, `348-envs-secrets-harmonization.md` (historical), `services/<service>/README.md` examples |
| **Storage / dependencies** | Does the service rely on shared infrastructure (MinIO, Postgres, Temporal, etc.)? Where are the dependencies documented? | `380-minio-config-overview.md`, `442-environment-isolation.md`, service-specific backlogs (e.g., `359-resilient-deploy-sequencing.md` for sequencing) |
| **GitOps & deployment** | Which ApplicationSet/Helm chart entries are required? What sync wave & labels apply? | `364-argo-configuration-v3.md`, `375-rolling-sync-wave-redesign.md`, `367-bootstrap-v2.md`, `387/447-argocd-waves*.md`, `465-applicationset-architecture.md`, environment naming per `434-unified-environment-naming.md` |
| **SDK & workspace alignment** | Are protos consumed exclusively via Ameide SDKs, have Go/TS/Py packages been regenerated (`scripts/dev/check_sdk_alignment.py`), and do Dockerfiles follow the Ring 1/Ring 2 workspace-first rules? | `365-buf-sdks-v2.md`, `408-workspace-first-ring-2.md`, `405-docker-files.md` |
| **Networking & tenancy labels** | Are namespace/pod labels and NetworkPolicies configured correctly (tier, tenant, environment)? | `441-networking.md`, `434-unified-environment-naming.md`, `442-environment-isolation.md`, `459-httproute-ownership.md` |
| **Testing (mock/cluster)** | Do integration packs follow the unified test contract (single implementation, mock/cluster modes) and exercise critical flows (e.g., primitive → `extensions-runtime` → host-call)? | `430-unified-test-infrastructure.md`, `480-ameide-extensibility-wasm-service.md`, `services/_templates/README.md`, `tools/integration-runner/README.md` |
| **Build/test verification & CI logs** | Have we run unit + integration suites locally, verified `docker build` (dev + release) succeeds, and confirmed GitHub workflows (`extensions-runtime.yml`, `cd-service-images.yml`) include the new service? Did the latest `gh run view <id>` logs pass SDK alignment and publishing checks? | `430-unified-test-infrastructure.md`, `395-sdk-build-docker-tilt-north-star.md`, `405-docker-files.md`, `408-workspace-first-ring-2.md`, `.github/workflows/*.yml` |
| **Observability & SLOs** | What metrics/logging/tracing does the service emit? Are dashboards updated? | `platform observability backlogs`, `services/README.md`, service-specific SLO docs |
| **Security & governance** | Does the service follow risk-tier rules, host-call policies, and secrets governance? | `476-security-and-trust.md`, `362-unified-secret-guardrails-v2.md`, `479-ameide-extensibility-wasm.md` (for Tier 1 runtime specifics) |
| **Documentation & ownership** | README, runbooks, rotation steps, owners? Has Backstage catalog info been created? | `service_catalog/.../README.md`, `backlog/481-service-catalog.md` (Backstage integration), `services/README.md` |
| **Tooling & developer experience** | Tilt target, devcontainer wiring, scripts? | `429-devcontainer-bootstrap.md`, `367-bootstrap-v2.md`, `tools/*` docs |
| **CI/CD & release automation** | Are GitHub workflows/cd-service pipelines building, testing, and packaging Docker images? | `395-sdk-build-docker-tilt-north-star.md`, `405-docker-files.md`, repo-specific workflows |

---

## 3. Detailed Steps

1. **Decide service type**
   - Primitive (Domain/Process/Agent/UISurface) → use Backstage template (see `service_catalog/…/_controller/template.yaml`).
   - Platform/legacy service → follow `services/README.md` contract.

2. **Scaffold files**
   - Create README (describe purpose, APIs, dependencies, rotation steps).
   - Add `Dockerfile.dev` & `Dockerfile.release` using skeleton patterns.
   - Add Tilt target (if applicable) and ensure `pnpm/golang` scripts exist.

3. **Define interfaces**
   - Proto definitions (if gRPC) under `packages/ameide_core_proto/...`.
   - Regenerate SDKs (`pnpm proto:generate`, Go tooling, Python packaging) and reference them. Run `scripts/dev/check_sdk_alignment.py` to confirm Go/TS/Py remain in sync. Services must consume APIs through the Ameide SDK packages as mandated by 365; no runtime imports of `packages/ameide_core_proto`. When services need primitive helpers (e.g., invoking `extensions-runtime`), add them to the SDKs instead of duplicating RPC glue.
   - Mirror the `extensions-runtime` workflow pattern: every service-specific CI job must install the Buf CLI and run the `scripts/ci/generate_*_stubs.sh` helpers before compiling or testing so workspace SDK stubs are refreshed from the BSR. Docker builds never invoke `buf generate`; they rely solely on the committed SDK artifacts.

4. **Secrets & config**
   - Identify required secrets, add ExternalSecrets referencing Vault keys (see backlog 362 for guardrails—no inline secrets or local fallbacks).
   - Update `foundation-vault-bootstrap` fixtures if new keys are needed.

5. **Storage/backing services**
   - Document dependencies (Postgres, Redis, MinIO). Reuse shared services where possible (e.g., MinIO per `380-minio-config-overview.md`).
   - Ensure required buckets or databases are provisioned via GitOps jobs, not manual steps.

6. **Testing**
   - Set up `__mocks__/` and `__tests__/integration/` per backlog 430 (single implementation, mock + cluster modes).
   - Ensure `run_integration_tests.sh` uses the standard tooling and fails fast on missing envs. Include end-to-end scenarios when the service depends on shared components (e.g., primitive → `extensions-runtime` → host-call) so regression tests cover cross-service seams.

7. **GitOps deployment**
   - Create Helm chart (or reuse existing) under `charts/`.
   - Add component definitions/values under `gitops/ameide-gitops` following the dual ApplicationSet structure in backlog 465.
   - Label the component with the correct rollout phase (per backlog 447) so RollingSync orders it properly, and ensure environment naming matches backlog 434.

8. **Observability & health**
   - Add health endpoints (HTTP/grpc health) per `services/README.md`.
   - Wire metrics/logging/tracing (OTel exporters, etc.) and register dashboards/alerts.
   - Apply namespace/pod labels and NetworkPolicy hooks documented in backlog 441/442 (tier, tenant, environment).
   - Verify Dockerfiles and build scripts respect Ring 1/Ring 2 rules (workspace SDK copies only, no registry fetches) and that SDK publish pipelines remain separate (408).

9. **Documentation & runbooks**
   - Ensure README covers rotation steps, dependencies, integration tests, and owners.
   - Update catalog info (Backstage `catalog-info.yaml`) if applicable.

10. **Security/governance checks**
    - Validate host-call policies (if applicable), risk tier, and auth patterns.
    - Run `scripts/validate-hardened-charts.sh` (manual while guardrail CI is deferred).

11. **CI/CD wiring**
    - Add or update GitHub workflows (and `cd-service-images` entries where applicable) so the service runs unit/integration tests and builds both dev/release Docker images on PRs/main. Confirm the workspace-first Dockerfiles pass `policy/check_docker_sources.sh` (408/405) and the unit/integration suites are referenced in the workflow.
    - After wiring, re-run the workflows (e.g., `gh run list`, `gh run view <run-id>`) to ensure `CD / Packages` succeeds (no missing SDK surfaces per 365/393) and `CD / Service Images` publishes the dev/release images. Capture run IDs in the backlog for traceability, especially when shared components (e.g., `extensions-runtime`) introduce new SDK helpers or Docker targets that need to ship together.

---

## 4. Suggested References by Topic

| Topic | File(s) |
|-------|---------|
| MinIO details | `backlog/380-minio-config-overview.md`, `services/agents/README.md` (artifact usage) |
| GitOps/AppSet placement | `backlog/465-applicationset-architecture.md`, `backlog/447-waves-v3-cluster-scoped-operators.md`, `backlog/375-rolling-sync-wave-redesign.md`, `backlog/367-bootstrap-v2.md` |
| Secret guardrails | `backlog/362-unified-secret-guardrails-v2.md`, `backlog/348-envs-secrets-harmonization.md` |
| SDK + Ring alignment | `backlog/365-buf-sdks-v2.md`, `backlog/408-workspace-first-ring-2.md`, `backlog/405-docker-files.md` |
| Test infrastructure | `backlog/430-unified-test-infrastructure.md`, `tools/integration-runner/README.md`, `services/_templates/README.md` |
| Environment naming & networking | `backlog/434-unified-environment-naming.md`, `backlog/441-networking.md`, `backlog/442-environment-isolation.md` |
| Extensibility/Tier1 runtime | `backlog/479-ameide-extensibility-wasm.md`, `backlog/480-ameide-extensibility-wasm-service.md` |
| Service catalog patterns | `backlog/481-service-catalog.md`, `service_catalog/domains/_controller/skeleton/*` |
| Bootstrap/devcontainer | `backlog/429-devcontainer-bootstrap.md`, `backlog/367-bootstrap-v2.md`, `435-remote-first-development.md` |

---

## 5. Open Questions

1. Should this checklist be automated (lint script or Backstage action) so new services must acknowledge each step?
2. Where should we store reusable Helm chart boilerplate for platform services (vs primitive scaffolds)?
3. Do we need a “service inception” PR template referencing these checkpoints?


<!-- ═══════════════════════════════════════════════════════════════════════════ -->
<!-- FILE: 483-fit-gap-analysis.md -->
<!-- ═══════════════════════════════════════════════════════════════════════════ -->

# 483 – Fit/Gap Analysis (Implementation Status)

**Status:** Living document
**Purpose:** Consolidate implementation status, gaps, and alignment notes from the vision suite (470-482) into a single tracking document.

> **Note**: The vision documents (470-476) describe **target state architecture**. This document tracks **current implementation status** against that target.

---

## 1. Critical Gaps

### 1.1 Authorization & Organization Isolation

**Target** (470 §5.1.2, 476 §3):
> "Tenant is always explicit: in tokens, DB schemas/RLS, and APIs."

**Current Status**:
- ❌ Phase 2 NOT STARTED: Database lacks `organization_id` columns on business tables
- ❌ Row-Level Security (RLS) policies not enabled
- ⚠️ API routes protected at membership level, but data isolation is tenant-only, not org-level

**Impact**: Users in Org A can potentially see Org B's data within the same tenant.

**Required Actions**:
1. Complete 329-authz Phase 2: add `organization_id` to repositories, transformations, elements tables
2. Enable RLS policies per Phase 3

**References**: [329-authz.md](329-authz.md), [476-ameide-security-trust.md](476-ameide-security-trust.md)

---

### 1.2 Transformation Domain Refactor

**Target** (470 §4.2, 472 §3.5):
> "Transformation Domain stores: initiatives, epics, backlogs, architecture decisions. Owns ProcessDefinitions and AgentDefinitions as first-class artifacts."

**Current Status**:
- ✅ `services/transformation/` exists with proto definitions
- ⚠️ Current implementation is monolithic; needs decomposition
- ⚠️ ProcessDefinitions and AgentDefinitions not yet first-class artifact types
- ⚠️ Transformation design tooling artifact management not yet integrated with Backstage templates

**Required Actions**:
1. Refactor `transformation-service` to cleanly separate:
   - Transformation Domain: initiatives, backlogs, governance state
   - Transformation design tooling subsystem: ProcessDefinitions, AgentDefinitions, artifact storage
   - Backstage bridge: template triggering from domain events
2. Define proto contracts for Backstage template parameters

**References**: [472-ameide-information-application.md](472-ameide-information-application.md) §3.5, [310-agents-v2.md](310-agents-v2.md)

---

### 1.3 SDK Distribution

**Target** (470 §5.3.12):
> "All services use `ameide_core_proto` and generated SDKs... Frontends always go through the TS SDK."

**Current Status**:
| SDK | Status | Gap |
|-----|--------|-----|
| Go SDK | ✅ Published | `v0.1.0` on GitHub + GHCR |
| Python SDK | ✅ Published | `2.10.1` on PyPI |
| **TypeScript SDK** | ❌ NOT PUBLISHED | 404 on npmjs; GHCR has only `dev`/hash tags |
| Configuration package | ⚠️ Partial | Only `dev`/hash tags on GHCR |

**Required Actions**:
1. Publish `@ameideio/ameide-sdk-ts` to npmjs with SemVer tags
2. Add GHCR SemVer aliases (`:0.1.0`, `:latest`) for TS SDK images
3. Wire `cd-packages.yml` workflow per 388 §CI/CD Blueprint

**References**: [388-ameide-sdks-north-star.md](388-ameide-sdks-north-star.md), [390-ameide-sdk-versioning.md](390-ameide-sdk-versioning.md)

---

## 2. High Priority Gaps

### 2.1 ProcessDefinition Execution Model

**Target** (472 §2.2, 475 §5):
> Process primitives execute ProcessDefinitions (BPMN-compliant, from custom React Flow modeller) via Temporal.

**Current Status**:
- ✅ Temporal infrastructure exists (305)
- ⚠️ ProcessDefinition → Temporal compiler does not exist
- ⚠️ Current Process primitives are code-first, not definition-driven

**Required Actions**:
1. Build ProcessDefinition → Temporal workflow compiler, OR
2. Clarify that current Process primitives are code-first (and update docs accordingly)

**References**: [305-workflow.md](305-workflow.md), [475-ameide-domains.md](475-ameide-domains.md) §5

---

### 2.2 Event Publishing

**Target** (472 §3.3.1):
> Domain events via outbox pattern + message bus for cross-domain communication.

**Current Status**:
- ⚠️ Outbox pattern exists but not consumed
- ⚠️ Services use direct Postgres, not append-only streams

**References**: [370-event-sourcing.md](370-event-sourcing.md)

---

### 2.3 Backstage Integration

**Target** (471 §4, 472 §4, 473 §4):
> Backstage as internal factory for scaffolding primitives via templates.

**Current Status**:
- ✅ Backstage MVP deployed (477)
- ⚠️ Templates not yet implemented
- ⚠️ Catalog modeling defined but not wired

**References**: [467-backstage.md](467-backstage.md)

---

## 3. Medium Priority Gaps

### 3.1 Domain Portfolio

**Target** (475 §3):
> Platform/Identity, Transformation, Commercial, Execution, Operations, People, Finance domain clusters.

**Current Status**:
- ✅ Platform/Identity partially implemented (319, 333)
- ✅ Transformation domain exists
- ⚠️ Business domain clusters (Commercial, Execution, etc.) not implemented

**References**: [475-ameide-domains.md](475-ameide-domains.md) §3

---

### 3.2 CRD Operators

**Target** (461, 474):
> Domain/Process/Agent CRDs reconciled by Ameide operators.

**Current Status**:
- ⚠️ CRD operators (Phase 1-3) not yet built
- ✅ Services deploy as standard K8s deployments

**References**: [461-ipc-idc-iac.md](461-ipc-idc-iac.md), [474-ameide-refactoring.md](474-ameide-refactoring.md)

---

### 3.3 Realm-per-Tenant

**Target** (333):
> Enterprise tenants get dedicated Keycloak realms; SMB shares a realm.

**Current Status**:
- ⚠️ Production runs single realm
- ✅ Hybrid model documented

**References**: [333-realms.md](333-realms.md), [428-sso-provider-support.md](428-sso-provider-support.md)

---

## 4. Operational Gaps

| Area | Gap | Reference |
|------|-----|-----------|
| Observability coverage | Framework exists but service-level coverage uneven | [334-logging-tracing.md](334-logging-tracing.md) |
| SCIM 2.0 | Enterprise identity provisioning pending | [428-sso-provider-support.md](428-sso-provider-support.md) |
| Temporal namespace isolation | Single namespace; per-tenant namespaces not yet implemented | [473-ameide-technology.md](473-ameide-technology.md) §3.2 |

---

## 5. Strong Alignment (No Gaps)

| Vision Principle | Supporting Backlogs | Status |
|-----------------|---------------------|--------|
| Multi-tenancy as first-class | [443-tenancy-models.md](443-tenancy-models.md) | ✅ 3 SKUs defined (Shared/Namespace/Private) |
| Realm-per-tenant target | [333-realms.md](333-realms.md) | ✅ Target architecture documented |
| Tenant explicit in tokens | [331-tenant-resolution.md](331-tenant-resolution.md) | ✅ JWT-based resolution |
| K8s & GitOps native | [364-argo-configuration.md](364-argo-configuration.md) | ✅ RollingSync ApplicationSets live |
| Secrets authority model | [451-secrets-management.md](451-secrets-management.md) | ✅ Classification complete |
| Proto-first contracts | [388-ameide-sdks-north-star.md](388-ameide-sdks-north-star.md) | ✅ Strategy documented |
| Tier 1 extensibility | [480-ameide-extensibility-wasm-service.md](480-ameide-extensibility-wasm-service.md) | ✅ extensions-runtime shipped |

---

## 6. Implementation Suite Status (477-482)

### 6.1 Backstage (477)

| Area | Status | Notes |
|------|--------|-------|
| Backstage deployment | ✅ MVP deployed | Running in `ameide-{env}` |
| Software templates | ⏳ Not started | Templates for Domain/Process/Agent/UISurface primitives pending |
| Catalog integration | ⚠️ Partial | Catalog modeling defined; wiring to primitives pending |
| TechDocs | ⏳ Not started | Documentation publishing not configured |

### 6.2 Tenant Extensions (478)

| Area | Status | Notes |
|------|--------|-------|
| Namespace strategy | ✅ Documented | Shared/Namespace/Private SKU topology defined |
| Repository model | ✅ Documented | `tenant-{id}-controllers` + `tenant-{id}-gitops` pattern |
| Backstage templates | ⏳ Not started | Template scaffolding for tenant primitives pending |
| PrimitiveImplementationDraft | ⏳ Not started | Transformation artifact type not yet implemented |

### 6.3 WASM Extensibility Vision (479)

| Area | Status | Notes |
|------|--------|-------|
| Tier 1 model | ✅ Documented | WASM extensions as data in shared runtime |
| Tier 2 model | ✅ Documented | Custom primitives in tenant namespaces |
| ExtensionDefinition | ⚠️ Partial | Proto exists; Transformation storage pending |
| Risk tier enforcement | ⏳ Not started | Policy engine for host-call restrictions pending |

### 6.4 Extensions Runtime (480)

| Area | Status | Notes |
|------|--------|-------|
| Proto & SDKs | ✅ Complete | `extensions/v1/runtime.proto` landed; Go/TS/Python SDKs expose `WasmExtensionRuntimeService` |
| Service scaffold | ✅ Complete | `services/extensions-runtime/` with Wasmtime executor, MinIO-backed ModuleStore |
| Testing | ✅ Mock mode, ⚠️ cluster | Mock + cluster-aware integration suite; cluster env vars need pipeline wiring |
| Docker build | ✅ Complete | Dev/release images build; CGO enabled for Wasmtime |
| Host-call bridge | ⚠️ Initial adapters | Registry-driven host bridge with first adapter; need token minting + additional adapters |
| Tilt target | ⏳ Not started | Need Tilt resource + chart skeleton |
| GitOps deployment | ⏳ Not started | ApplicationSet entries, secrets, observability pending |

### 6.5 Service Catalog (481)

| Area | Status | Notes |
|------|--------|-------|
| Folder structure | ✅ Documented | `service_catalog/domains/`, `processes/`, `agents/` pattern |
| Base controllers | ⏳ Not started | `_controller/` base definitions pending |
| Backstage alignment | ⚠️ Partial | Structure supports templates; templates not implemented |

### 6.6 Adding New Service (482)

| Area | Status | Notes |
|------|--------|-------|
| Checklist | ✅ Complete | Comprehensive checklist covering all cross-cutting concerns |
| Template scaffolding | ⏳ Not started | Backstage templates for service creation pending |
| Automation | ⏳ Not started | No automated service creation workflow yet |

---

## 7. Recent Progress (Dec 2025)

- **Tier 1 extensibility runtime shipped (480).** `extensions-runtime` now exists as a platform service in `ameide-{env}`, exposes `InvokeExtension` gRPC API, runs tenant WASM modules with Wasmtime sandboxing + MinIO-backed storage.

- **Unified ApplicationSet orchestration live (364).** GitOps repo drives namespaces, CRDs, operators, platform planes, and services via RollingSync ApplicationSets.

- **SDK + primitive alignment (388, 430).** Go/TS/Python SDKs consume the same `runtime.proto` and integration packs enforce env-var conventions.

---

## 7. Terminology Migration

| Legacy Term | New Term | Status |
|-------------|----------|--------|
| IPA (Intelligent Process Automation) | Domain + Process + Agent primitive bundle | Deprecated in customer comms |
| IDC/IPC/IAC | Domain/Process/Agent CRD | Use new naming per 470 |
| Platform Workflows (305) | Process primitive | Aligned |
| AgentRuntime (310) | Agent primitive | Aligned |
| DomainService | Domain primitive | Aligned |
| BPMN model | ProcessDefinition | Aligned |
| Agent config | AgentDefinition | Aligned |
| "Graph service" | Knowledge Graph (read projection) | Clarified |

---

## 8. Cross-References

**Vision Suite** (target state):
- [470-ameide-vision.md](470-ameide-vision.md) – Vision and principles
- [471-ameide-business-architecture.md](471-ameide-business-architecture.md) – Business architecture
- [472-ameide-information-application.md](472-ameide-information-application.md) – Application architecture
- [473-ameide-technology.md](473-ameide-technology.md) – Technology stack
- [475-ameide-domains.md](475-ameide-domains.md) – Domain architecture
- [476-ameide-security-trust.md](476-ameide-security-trust.md) – Security architecture

**Implementation Backlogs**:
- [474-ameide-refactoring.md](474-ameide-refactoring.md) – Migration plan
- [467-backstage.md](467-backstage.md) – Backstage implementation
- [478-ameide-extensions.md](478-ameide-extensions.md) – Tenant extensions
- [479-ameide-extensibility-wasm.md](479-ameide-extensibility-wasm.md) – WASM extensibility vision
- [480-ameide-extensibility-wasm-service.md](480-ameide-extensibility-wasm-service.md) – extensions-runtime implementation
- [481-service-catalog.md](481-service-catalog.md) – Service catalog structure
- [482-adding-new-service.md](482-adding-new-service.md) – New service checklist

