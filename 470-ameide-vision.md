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
> | [461-ipc-idc-iac.md](461-ipc-idc-iac.md) | Declarative controller model (IDC/IPC/IAC CRDs) |
>
> **Deployment Implementation**:
> - [465-applicationset-architecture.md](465-applicationset-architecture.md) – GitOps deployment model
> - [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) – Rollout phases
> - [442-environment-isolation.md](442-environment-isolation.md) – Environment isolation
> - [446-namespace-isolation.md](446-namespace-isolation.md) – Namespace patterns

> **Comparative Briefs (Ameide vs incumbent ERPs)**:
> - [470-ameide-visio-vs-d365.md](470-ameide-visio-vs-d365.md) – Code-first Ameide platform compared to Microsoft D365FO’s metadata/AOT paradigm.
> - [470-ameide-vision-vs-saps4.md](470-ameide-vision-vs-saps4.md) – Ameide positioning versus SAP S/4HANA’s DDIC/CDS/Fiori/Customizing stack.
>
> These appendices translate the high-level Ameide principles in this document into head-to-head comparisons with the dominant metadata/configuration ERPs so product, field, and architecture teams can articulate what stays the same (coherence, discoverability) and what changes (code-first, AI operated, CRD-based infra) when pitching Ameide.

---

## 0. Core Definitions: DomainControllers, UAF, and Graph

> **DomainControllers** own all authoritative definitions and runtime data for their bounded context, persisted according to their proto contracts.
>
> **UAF** (Unified Artifact Framework) is the set of modelling UIs (BPMN, diagrams, Markdown, agent specs) that talk to DomainControllers via APIs; UAF does not persist anything. All artifacts created through UAF are stored by the **Transformation DomainController** (or other relevant DomainControllers).
>
> **Graph** is a read-only knowledge layer that projects selected data from any controller into a graph database so we can traverse relationships without heavy cross-domain joins. All writes go through controllers; Graph is never a source of truth.

---

## 0.1 Glossary

| Term | Definition |
|------|------------|
| **Domain** | Business bounded context (e.g. Orders, Customers, Transformation) |
| **DomainController** | Logical service implementing a domain (APIs, rules, persistence, events); at runtime expressed as an `IntelligentDomainController` (IDC) custom resource |
| **ProcessDefinition** | BPMN-compliant artifact defining a business process (design-time, stored in Transformation DomainController, modelled via UAF UIs) |
| **ProcessController** | Logical runtime that executes ProcessDefinitions, calling DomainControllers and AgentControllers; materialised as an `IntelligentProcessController` (IPC) custom resource |
| **AgentDefinition** | Declarative spec for an agent graph + tools + policies (design-time, stored in Transformation DomainController, modelled via UAF UIs) |
| **AgentController** | Logical runtime that executes AgentDefinitions via LLM/tool loops, always via SDKs; represented by an `IntelligentAgentController` (IAC) custom resource |
| **UIWorkspace** | Next.js frontend that talks to controllers via the TS SDK |
| **UAF** | Unified Artifact Framework - modelling UIs (BPMN editor, diagrams, Markdown) that talk to DomainControllers via APIs; UAF does not persist anything itself. All UAF-originated artifacts are persisted by the **Transformation DomainController** |
| **Graph** | Read-only knowledge projection that ingests selected data from any controller into a graph database for relationship traversal without cross-DB joins; never a source of truth |
| **IntelligentDomainController (IDC)** | Kubernetes custom resource (see 461) describing the desired state for a DomainController; reconciled by the IDC operator into Deployments, Services, HPAs, DB schemas, etc. |
| **IntelligentProcessController (IPC)** | Custom resource describing a ProcessController (image, Temporal namespace, bindings); reconciled by the IPC operator. |
| **IntelligentAgentController (IAC)** | Custom resource describing an AgentController (runtime, AgentDefinition reference, risk profile); reconciled by the IAC operator. |

---

## 1. Purpose

This document defines **why Ameide exists**, **what we are building**, and the **principles** that constrain future design decisions.

It sits above business, application, and technology architecture docs and provides a common language for:

* How we think about **domains, processes, agents, and UIs**
* How **transformation itself** is modelled as a first-class domain
* How Backstage, UAF, BPMN, and the current proto/SDK stack come together

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

Even in Ameide today, powerful building blocks (onboarding, proto APIs, UAF, agents, workflows) are still experienced as **bespoke services**, not a coherent business platform.   

### 2.2 Opportunity

We can build a **cloud-native business platform** where:

* “ERP/CRM/HCM/MES” collapse into **domains + processes + agents + UIs** on a single substrate.
* **Transformation** (roadmaps, designs, deployments) is modelled as a domain **inside the platform**, not as a separate consulting lifecycle. 
* Customers express requirements in natural language; **agents + templates + GitOps** turn them into running capabilities.
* We reuse and extend existing strengths:

  * Proto-based APIs and clean architecture.
  * Production-ready TS SDK for web UIs.
  * UAF as modelling UIs for artifacts (BPMN, architecture diagrams, Markdown specs), with all artifacts persisted by the Transformation DomainController.
  * BPMN-compliant ProcessDefinitions executed by Temporal-backed ProcessControllers.
  * Existing onboarding & tenancy model. 

---

## 3. Vision

> **Ameide is a universal, cloud‑native business platform where domains, processes, agents, and workspaces are declared, generated, and operated as first‑class platform resources — and where transformation itself is modelled as just another domain.**

### 3.1 Core outcomes

1. **No more one‑off ERPs/CRMs**
   Customers assemble **domains** (Orders, Customers, Transformation, Identity, Billing…) and **process controllers** (L2O, O2C, Onboarding, Agile, TOGAF) instead of buying separate suites.

2. **Transformation is inside the product**
   The same platform that runs L2O/O2C also runs the **transformation processes** that define and evolve them, with artifacts managed by the **Transformation DomainController** and modelled through UAF UIs.

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

   * **Design-time**: ProcessDefinitions, AgentDefinitions, ExtensionDefinitions, and other UAF artifacts stored in Transformation.
   * **Deployment-time**: Backstage templates, GitOps, and controller manifests (IDC/IPC/IAC CRs) committed to Git.
   * **Runtime**: Operators reconcile IDC/IPC/IAC custom resources into Deployments, Services, Temporal workers, and other workloads on Kubernetes.

---

## 4. Conceptual model

At the highest level we standardise on four building blocks, with a clear **design-time vs runtime** split:

### 4.0 Design-Time Artifacts (in Transformation DomainController)

* **ProcessDefinition** – BPMN-compliant artifact produced by a custom React Flow modeller.
  * Defines process stages, tasks, gateways, and bindings to domains/agents.
  * Stored and versioned in Transformation DomainController with revisions & promotions (modelled via UAF UIs).

* **AgentDefinition** – Declarative spec for an agent.
  * Tools (domain/process APIs), orchestration graph, scope, risk tier, policies.
  * Stored alongside ProcessDefinitions in Transformation DomainController (modelled via UAF UIs).

### 4.1 Runtime Controllers

At runtime every controller is captured as a declarative custom resource (IDC/IPC/IAC). Ameide operators reconcile those CRs into Deployments/Services/Temporal workers/agent runtimes so Git remains the single source of desired state.

1. **DomainController** – a domain microservice:

   * Owns **proto APIs**, domain rules, persistence, and events.
   * Examples: Orders, Customers, Transformation, UAF, Onboarding, Identity.
   * Deterministic, versioned, testable.
   * **Runtime representation**: `IntelligentDomainController` CR (see [461](461-ipc-idc-iac.md)) reconciled into Deployments, Services, DB schemas, HPAs, ServiceMonitors, etc.

2. **ProcessController** – executes ProcessDefinitions:

   * Loads a specific ProcessDefinition version from Transformation DomainController.
   * Manages process instances, state, retries, compensation.
   * Backed by Temporal; binds BPMN tasks to DomainController/AgentController calls.
   * **Runtime representation**: `IntelligentProcessController` CR defining Temporal workers, namespace bindings, rollout strategy.

3. **AgentController** – executes AgentDefinitions:

   * Loads an AgentDefinition and runs the LLM/tool loop.
   * Accesses domain/process data via the SDK and public APIs.
   * Enforces scope/risk policies at execution time.
   * E.g. transformation requirement agent, coder/refactor agent, L2O assistant.
   * **Runtime representation**: `IntelligentAgentController` CR describing runtime image, tool grants, risk tier, rollout.

4. **UIWorkspace** – UX layer:

   * Next.js workspaces that expose process views and domain workspaces using the TS SDK.

These are **logical roles**. Underneath, they're implemented by concrete services described in the proto/API and North‑Star docs (graph/repository/platform/transformation/workflows/agents/chat/threads/www_*).

### 4.2 Transformation & UAF as first‑class domain

* **Transformation DomainController**

  * Stores: initiatives, epics, backlogs, architecture decisions.
  * Owns **ProcessDefinitions** and **AgentDefinitions** as first-class artifacts (optionally using an event-sourced pattern internally).
  * Coordinates: Agile or TOGAF ADM-like processes as ProcessControllers.
  * Single source of truth for all design-time artifacts—there is no separate "UAF service" in the runtime.

* **Unified Artifact Framework (UAF)**

  * A *frontend modelling layer* (BPMN editor, diagram editor, Markdown editor) that calls the Transformation DomainController APIs.
  * UAF UIs send commands to Transformation; they don't own storage.
  * Provides versioning and promotion UX but delegates persistence to the Transformation DomainController.

### 4.3 Backstage as "factory"

Backstage is the **factory** that turns Transformation DomainController decisions into running components:

* **Catalog**:

  * Indexes DomainController, ProcessController, AgentController, UIWorkspace components and their sources (repos, Helm releases, proto APIs).
* **Templates**:

  * Scaffold new domain controllers, process controllers, agent controllers, and workspaces using standard Ameide patterns (proto, SDK, Helm/Argo, CNPG, Temporal) and emit the corresponding IDC/IPC/IAC manifests that GitOps/Argo will apply.
* **Bridge**:

  * Listens to Transformation domain events and runs templates with specific parameters (e.g. "create L2O ProcessController variant for tenant X").

> **Extension Model**: For tenant-specific controllers and custom code isolation, see [478-ameide-extensions.md](478-ameide-extensions.md) which defines the namespace strategy by SKU and the E2E flow for controller creation.

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

3. **Deterministic controllers, non-deterministic agents**

   * DomainControllers + ProcessControllers are **deterministic**, versioned, and testable.
   * AgentControllers are allowed to be non-deterministic, but:

     * They can only act through public domain/process APIs.
     * Their proposals become durable only once written into the Transformation DomainController.

4. **Process‑first modelling**

   * Processes like L2O, O2C, Onboarding, Scrum, TOGAF are defined as ProcessDefinitions (BPMN-compliant) in Transformation DomainController (modelled via UAF UIs).
   * ProcessControllers execute these definitions via Temporal.
   * Domain design starts by asking "which process stages does this domain support?"

5. **Graph as read-only projection, not runtime truth**

   * **Graph** holds read-only projections for analysis; designs and histories live in DomainControllers (primarily Transformation) and can be projected into Graph as needed.
   * Graph is never a source of truth; all writes go through controllers.
   * Each DomainController controls its own persistence; Graph merely indexes selected data for cross-domain queries.

### 5.2 Transformation & Governance Principles

6. **Transformation is a domain like any other**

   * The Transformation DomainController owns:

     * Transformation initiatives & roadmaps.
     * **ProcessDefinitions** and **AgentDefinitions** as first-class UAF artifacts.
     * UAF artifacts (BPMN, ArchiMate, Markdown).
     * Backstage template configurations.
   * Change to the platform is expressed as **events and configs** in this domain, not ad-hoc pipelines.

7. **Design → Review → Generate → Deploy**

   * The canonical lifecycle is:

     1. Design change in UAF / Transformation (possibly with AgentController assistance).
     2. Human review / governance (change boards, risk tiering).
     3. Backstage template run → new DomainController/ProcessController/AgentController/UIWorkspace.
     4. GitOps/Helm/Argo rollout.

8. **Self-development over time**

   * The long-term goal: most new capabilities are produced by AgentControllers **inside the Transformation domain**, guided by ProcessDefinitions/AgentDefinitions in UAF and Backstage templates, with humans mostly approving and course-correcting rather than hand-writing code.

9. **IPA as a legacy concept**

   * IPA (Intelligent Process Automation) remains an **internal historical concept**, now decomposed into:

     * a bundle of DomainControllers + ProcessControllers + AgentControllers.
   * No new user-visible "IPA" product surface; everything is explained in terms of domains/processes/agents.

### 5.3 Engineering & Platform Principles

10. **Kubernetes & GitOps native**

    * All long-running components run on K8s with ArgoCD/Helm GitOps, ExternalSecrets/Vault, CNPG Postgres, and hardened charts. 

11. **North‑Star: client-side first, immutable artifacts, stateless edges**

    * Follow the North‑Star blueprint:

      * Keep modelling/editor stacks as client-heavy (IndexedDB/command stacks, BPMN editors).
      * Treat design artifacts as immutable, content-addressed blobs with revision history (via UAF). 

12. **Single proto & SDK contract layer**

    * All services use `ameide_core_proto` and generated SDKs; no direct wire-level improvisation.
    * Frontends always go through the TS SDK; agents and workers use Go/Python SDKs. 

13. **Clean separation of concerns**

    * **Design**: ProcessDefinitions/AgentDefinitions in custom React Flow modeller, stored in Transformation DomainController.
    * **APIs**: pure proto-based domain APIs (044).
    * **Runtime**: DomainControllers, ProcessControllers (Temporal-backed), AgentControllers on K8s.
    * **SDK / tools**: TS SDK, core-platform-coder, Backstage templates driving these layers.

14. **Observability, always-on**

    * Tracing, metrics, and logs are not optional; they are part of acceptance criteria for any new DomainController/ProcessController/AgentController.
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

     * Transformation records + UAF artifacts.
     * Backstage template runs, not bespoke scripting.

3. **Process-first adoption**

   * Share of daily active users that enter via process views vs entity tables.

4. **Agent-assisted vs manually implemented changes**

   * How many changes to domains/processes/agents are **agent-proposed and human-approved** vs purely manual.

---

## 7. Legacy alignment & reuse

To ensure we don't lose prior work:

* **IPA Architecture**

  * IPA remains as a conceptual ancestor of "Domain + Process + Agent bundles", but new design and implementation use the DomainController / ProcessController / AgentController pattern.

* **Proto-based APIs**

  * The proto/REST/gRPC design and the API infrastructure backlog remain the foundation of DomainController patterns; new domains must follow them.

* **TypeScript SDK**

  * The SDK is the reference client for UIWorkspaces and many AgentControllers; all frontend workspaces must integrate via this path.

* **North-Star & UAF**

  * North-Star defines the design/deploy/runtime separation and client-side modelling.
  * UAF defines ProcessDefinitions, AgentDefinitions, and other artifacts with revisions and promotions; it becomes the core of the Transformation domain's knowledge layer.

* **core-platform-coder and internal agents**

  * These become specific AgentController implementations we can generalise from (robust tool usage, scoring, state typing).

* **Onboarding & identity**

  * The onboarding backlog is the canonical specification for tenancy, identity, and the first big process (tenant lifecycle) we must express as a ProcessDefinition + ProcessController.

---

## 8. Next documents in the stack

This Vision / Rationale / Principles doc is intentionally high-level. The following documents refine it:

1. **Business Architecture** – how tenants, orgs, roles, and transformation journeys use the platform.
2. **Application & Information Architecture** – mapping DomainController/ProcessController/AgentController/UIWorkspace to concrete services & data flows; ProcessDefinitions and AgentDefinitions as design-time artifacts.
3. **Technology Architecture** – detailed runtime stack (K8s, Temporal-backed ProcessControllers, Backstage, SDKs, infra operators).
4. **Domain Specifications** – L2O/O2C, Transformation, Onboarding/Identity, etc.
5. **Refactoring & Migration Plan** – how we move from the existing microservices + IPA builder to this model.

Those docs should all **inherit and reference these principles**; any deviations should be called out explicitly and treated as architecture decisions.

---

## 9. Cross-References & Alignment Status

This section maps vision principles to existing backlogs and flags gaps requiring attention.

### 9.1 Strong Alignment (No Gaps)

| Vision Principle | Supporting Backlogs | Status |
|-----------------|---------------------|--------|
| Multi-tenancy as first-class (§5.1.2) | [443-tenancy-models.md](443-tenancy-models.md) | ✅ 3 SKUs defined (Shared/Namespace/Private) |
| Realm-per-tenant for enterprise | [333-realms.md](333-realms.md) | ✅ Target architecture documented |
| Tenant explicit in tokens & APIs | [331-tenant-resolution.md](331-tenant-resolution.md) | ✅ JWT-based resolution implemented |
| K8s & GitOps native (§5.3.10) | [364-argo-configuration.md](364-argo-configuration.md), [387-argocd-waves-v2.md](387-argocd-waves-v2.md) | ✅ Implemented |
| Secrets authority model (§5.3.15) | [451-secrets-management.md](451-secrets-management.md), [462-secrets-origin-classification.md](462-secrets-origin-classification.md) | ✅ Classification complete |
| CNPG for DB creds | [412-cnpg-owned-postgres-creds.md](412-cnpg-owned-postgres-creds.md) | ✅ Documented |
| Proto-first contracts (§5.1.1) | [388-ameide-sdks-north-star.md](388-ameide-sdks-north-star.md), [393-ameide-sdk-import-policy.md](393-ameide-sdk-import-policy.md) | ✅ Strategy documented |
| IPA decomposition (§5.2.9) | [472-ameide-information-application.md](472-ameide-information-application.md) §6 | ✅ Mapping documented |

### 9.2 Gaps & Issues Requiring Attention

#### 9.2.1 Authorization & Organization Isolation ⚠️ CRITICAL

**Vision states** (§5.1.2):
> "Tenant is always explicit: in tokens, **DB schemas/RLS**, and APIs."

**Current reality per [329-authz.md](329-authz.md)**:
- ❌ **Phase 2 NOT STARTED**: Database lacks `organization_id` columns on business tables
- ❌ Row-Level Security (RLS) policies not enabled
- ⚠️ API routes protected at membership level, but data isolation is **tenant-only, not org-level**

**Impact**: Users in Org A can potentially see Org B's data within the same tenant. The vision's "strong tenant isolation" principle is only partially met.

**Required actions**:
1. Complete 329-authz Phase 2: add `organization_id` to repositories, transformations, elements tables
2. Enable RLS policies per Phase 3
3. Clarify: Is org-within-tenant isolation required for v1, or is tenant-level sufficient initially?

---

#### 9.2.2 Transformation Domain Implementation ⚠️ REFACTOR NEEDED

**Vision states** (§4.2):
> "Transformation DomainController stores: initiatives, epics, backlogs, architecture decisions. Owns ProcessDefinitions and AgentDefinitions as first-class artifacts."

**Current reality**:
- ✅ `services/transformation/` exists with proto definitions (`ameide_core_proto/transformation/v1/`)
- ⚠️ Current implementation is monolithic; needs decomposition to align with Backstage-based "factory" model
- ⚠️ UAF artifact management partially implemented but not yet integrated with Backstage templates
- ⚠️ ProcessDefinitions and AgentDefinitions not yet first-class artifact types

**Alignment with 472 (Information Architecture)**:
- 472 §3.5 describes Transformation as owning: Initiatives, Backlog Items, UAF Artifacts, Governance
- 472 §4.2 describes Backstage templates for DomainController/ProcessController/AgentController/UIWorkspace generation

**Required actions**:
1. Refactor `transformation-service` to cleanly separate:
   - **Transformation DomainController**: initiatives, backlogs, governance state
   - **UAF subsystem**: ProcessDefinitions, AgentDefinitions, artifact storage, revisions, promotions
   - **Backstage bridge**: template triggering from domain events
2. Define proto contracts for Backstage template parameters
3. Implement event-driven bridge: Transformation domain events → Backstage template runs

**Related backlogs**:
- [310-agents-v2.md](310-agents-v2.md) – AgentController layer (n8n-aligned); fits into §4 conceptual model
- [472-ameide-information-application.md](472-ameide-information-application.md) §4 – Backstage catalog modeling

---

#### 9.2.3 SDK Distribution ⚠️ PUBLICATION GAP

**Vision states** (§5.3.12):
> "All services use `ameide_core_proto` and generated SDKs... Frontends always go through the TS SDK; agents and workers use Go/Python SDKs."

**Current reality per [388-ameide-sdks-north-star.md](388-ameide-sdks-north-star.md) (status section)**:
| SDK | Status | Gap |
|-----|--------|-----|
| Go SDK | ✅ Published | `v0.1.0` on GitHub + GHCR |
| Python SDK | ✅ Published | `2.10.1` on PyPI |
| **TypeScript SDK** | ❌ **NOT PUBLISHED** | 404 on npmjs; GHCR has only `dev`/hash tags |
| Configuration package | ⚠️ Partial | Only `dev`/hash tags on GHCR |

**Impact**: The "open by design – APIs & SDKs everywhere" principle (§5.2 of [000-ameide-principles.md](000-ameide-principles.md)) is compromised for external TS consumers.

**Required actions**:
1. Publish `@ameideio/ameide-sdk-ts` to npmjs with SemVer tags
2. Add GHCR SemVer aliases (`:0.1.0`, `:latest`) for TS SDK images
3. Wire `cd-packages.yml` workflow per 388 §CI/CD Blueprint

**Related backlogs**:
- [390-ameide-sdk-versioning.md](390-ameide-sdk-versioning.md) – SemVer policy
- [391-resilient-cd-packages-workflow.md](391-resilient-cd-packages-workflow.md) – CI wiring

---

### 9.3 Minor Gaps (Operational)

| Area | Gap | Reference |
|------|-----|-----------|
| Observability coverage | Framework exists but service-level coverage uneven | [334-logging-tracing.md](334-logging-tracing.md) |
| 474 Implementation Plan | Empty placeholder | [474-ameide-implementation.md](474-ameide-implementation.md) |

---

### 9.4 Cross-Reference Index

**Vision Suite**:
- [470-ameide-vision.md](470-ameide-vision.md) – This document
- [471-ameide-business-architecture.md](471-ameide-business-architecture.md) – Business use cases, personas, journeys
- [472-ameide-information-application.md](472-ameide-information-application.md) – Application & information architecture
- [473-ameide-technology.md](473-ameide-technology.md) – Technology stack blueprint

**Core Principles**:
- [000-ameide-principles.md](000-ameide-principles.md) – Foundation principles (well-aligned)

**Multi-Tenancy & Identity**:
- [333-realms.md](333-realms.md) – Realm-per-tenant target architecture
- [331-tenant-resolution.md](331-tenant-resolution.md) – JWT-based tenant resolution
- [443-tenancy-models.md](443-tenancy-models.md) – SKU matrix (Shared/Namespace/Private)
- [329-authz.md](329-authz.md) – Authorization (⚠️ Phase 2 not started)

**Secrets & Infrastructure**:
- [451-secrets-management.md](451-secrets-management.md) – E2E secrets flow
- [462-secrets-origin-classification.md](462-secrets-origin-classification.md) – Authority model
- [418-secrets-strategy-map.md](418-secrets-strategy-map.md) – Current strategy

**SDKs & APIs**:
- [388-ameide-sdks-north-star.md](388-ameide-sdks-north-star.md) – SDK publish strategy (⚠️ TS gap)
- [393-ameide-sdk-import-policy.md](393-ameide-sdk-import-policy.md) – Import guardrails

**Agents & Transformation**:
- [310-agents-v2.md](310-agents-v2.md) – AgentController layer (n8n-aligned)
- [305-workflow.md](305-workflow.md) – Workflow orchestration (Temporal-backed ProcessControllers)
