# 4xx – Ameide Platform Vision / Rationale / Principles

**Status:** Draft v1
**Owner:** Architecture / Product
**Intended audience:** Founders, product, principal engineers, domain/solution architects

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
  * UAF for event-sourced artifacts (BPMN, ArchiMate, Markdown). 
  * BPMN + Camunda/Temporal for processes.
  * Existing onboarding & tenancy model. 

---

## 3. Vision

> **Ameide is a universal, cloud‑native business platform where domains, processes, agents, and workspaces are declared, generated, and operated as first‑class platform resources — and where transformation itself is modelled as just another domain.**

### 3.1 Core outcomes

1. **No more one‑off ERPs/CRMs**
   Customers assemble **domains** (Orders, Customers, Transformation, Identity, Billing…) and **process controllers** (L2O, O2C, Onboarding, Agile, TOGAF) instead of buying separate suites.

2. **Transformation is inside the product**
   The same platform that runs L2O/O2C also runs the **transformation processes** that define and evolve them, with artifacts managed via UAF.

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

   * **Design-time** (UAF artifacts, BPMN diagrams, ArchiMate models, Markdown specs). 
   * **Deployment-time** (Backstage templates, GitOps, Helm/Argo). 
   * **Runtime** (K8s services, Temporal/Camunda, agents runtime).

---

## 4. Conceptual model

At the highest level we standardise on four building blocks:

1. **DomainService** – a domain microservice:

   * Owns **proto APIs**, domain rules, persistence, and events.
   * Examples: Orders, Customers, Transformation, UAF, Onboarding, Identity.

2. **ProcessController** – orchestrates a business process (BPMN → Temporal/Camunda):

   * Defines the lifecycle (e.g. L2O, O2C, Tenant Onboarding, Scrum).
   * Binds BPMN service tasks to domain operations and optionally agent tools.

3. **AgentRuntime** – deploys an agent graph with tools and policies:

   * E.g. transformation requirement agent, coder/refactor agent, L2O assistant.
   * Accesses domain/process data via the SDK and public APIs.

4. **UIWorkspace** – UX layer:

   * Next.js workspaces that expose process views and domain workspaces using the TS SDK.

These are **logical roles**. Underneath, they’re implemented by concrete services described in the proto/API and North‑Star docs (graph/repository/platform/transformation/workflows/agents/chat/threads/www_*).

### 4.1 Transformation & UAF as first‑class domain

* **Transformation DomainService**

  * Stores: initiatives, epics, backlogs, architecture decisions, and **process/domain/agent/UI configurations**. 
  * Coordinates: Agile or TOGAF ADM-like processes as ProcessControllers.

* **Unified Artifact Framework (UAF)**

  * Treats BPMN, ArchiMate, Markdown as event‑sourced artifacts with revisions & promotions.
  * Connects design artifacts to running domains/processes/agents.

### 4.2 Backstage as “factory”

Backstage is the **factory** that turns Transformation/UAF decisions into running components:

* **Catalog**:

  * Indexes DomainService, ProcessController, AgentRuntime, UIWorkspace components and their sources (repos, Helm releases, proto APIs).
* **Templates**:

  * Scaffold new domain services, process controllers, agents, and workspaces using standard Ameide patterns (proto, SDK, Helm/Argo, CNPG, Temporal/Camunda).
* **Bridge**:

  * Listens to Transformation domain events and runs templates with specific parameters (e.g. “create L2O ProcessController variant for tenant X”).

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

3. **Deterministic domains, non-deterministic agents**

   * DomainServices + ProcessControllers are **deterministic**, versioned, and testable.
   * Agents are allowed to be non-deterministic, but:

     * They can only act through public domain/process APIs.
     * Their proposals become durable only once written into the Transformation domain or UAF.

4. **Process‑first modelling**

   * Processes like L2O, O2C, Onboarding, Scrum, TOGAF are represented as BPMN and executed via Temporal/Camunda.
   * Domain design starts by asking “which process stages does this domain support?”

5. **Graph and artifacts as analytic layers, not runtime truth**

   * **Graph/UAF** hold projections, designs, and histories for analysis, not operational state.
   * Each DomainService controls its own persistence; graph is read-only from the domain’s perspective.

### 5.2 Transformation & Governance Principles

6. **Transformation is a domain like any other**

   * It owns:

     * Transformation initiatives & roadmaps.
     * UAF artifacts (BPMN, ArchiMate, Markdown).
     * Backstage template configurations.
   * Change to the platform is expressed as **events and configs** in this domain, not ad-hoc pipelines.

7. **Design → Review → Generate → Deploy**

   * The canonical lifecycle is:

     1. Design change in UAF / Transformation (possibly with agent help).
     2. Human review / governance (change boards, risk tiering).
     3. Backstage template run → new domain/process/agent/UI.
     4. GitOps/Helm/Argo rollout.

8. **Self-development over time**

   * The long-term goal: most new capabilities are produced by agents **inside the Transformation domain**, guided by UAF artifacts and Backstage templates, with humans mostly approving and course-correcting rather than hand-writing code.

9. **IPA as a legacy concept**

   * IPA (Intelligent Process Automation) remains an **internal historical concept**, now decomposed into:

     * a bundle of DomainServices + ProcessControllers + AgentRuntimes. 
   * No new user-visible “IPA” product surface; everything is explained in terms of domains/processes/agents.

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

    * **Design**: BPMN/UAF/ArchiMate editing (North‑Star, UAF).
    * **APIs**: pure proto-based domain APIs (044). 
    * **Runtime**: K8s services, Temporal/Camunda, agents runtime.
    * **SDK / tools**: TS SDK, core-platform-coder, Backstage templates driving these layers.

14. **Observability, always-on**

    * Tracing, metrics, and logs are not optional; they are part of acceptance criteria for any new DomainService/ProcessController/AgentRuntime.
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

To ensure we don’t lose prior work:

* **IPA Architecture**

  * IPA remains as a conceptual ancestor of “Domain + Process + Agent bundles”, but new design and implementation use the DomainService / ProcessController / AgentRuntime pattern. 

* **Proto-based APIs**

  * The proto/REST/gRPC design and the API infrastructure backlog remain the foundation of DomainService patterns; new domains must follow them.

* **TypeScript SDK**

  * The SDK is the reference client for UIWorkspaces and many AgentRuntimes; all frontend workspaces must integrate via this path. 

* **North-Star & UAF**

  * North-Star defines the design/deploy/runtime separation and client-side modelling.
  * UAF defines artifacts, revisions, and promotions; it becomes the core of the Transformation domain’s knowledge layer.

* **core-platform-coder and internal agents**

  * These become specific AgentRuntime implementations we can generalise from (robust tool usage, scoring, state typing). 

* **Onboarding & identity**

  * The onboarding backlog is the canonical specification for tenancy, identity, and the first big process (tenant lifecycle) we must express as a ProcessController.

---

## 8. Next documents in the stack

This Vision / Rationale / Principles doc is intentionally high-level. The following documents refine it:

1. **Business Architecture** – how tenants, orgs, roles, and transformation journeys use the platform.
2. **Application & Information Architecture** – mapping DomainService/ProcessController/AgentRuntime/UIWorkspace to concrete services & data flows.
3. **Technology Architecture** – detailed runtime stack (K8s, Temporal/Camunda, Backstage, SDKs, infra operators).
4. **Domain Specifications** – L2O/O2C, Transformation/UAF, Onboarding/Identity, etc.
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

**Vision states** (§4.1):
> "Transformation DomainService stores: initiatives, epics, backlogs, architecture decisions, and process/domain/agent/UI configurations."

**Current reality**:
- ✅ `services/transformation/` exists with proto definitions (`ameide_core_proto/transformation/v1/`)
- ⚠️ Current implementation is monolithic; needs decomposition to align with Backstage-based "factory" model
- ⚠️ UAF artifact management partially implemented but not yet integrated with Backstage templates

**Alignment with 472 (Information Architecture)**:
- 472 §3.5 describes Transformation as owning: Initiatives, Backlog Items, UAF Artifacts, Governance
- 472 §4.2 describes Backstage templates for Domain/Process/Agent/UI generation

**Required actions**:
1. Refactor `transformation-service` to cleanly separate:
   - **Transformation DomainService**: initiatives, backlogs, governance state
   - **UAF subsystem**: artifact storage, revisions, promotions
   - **Backstage bridge**: template triggering from domain events
2. Define proto contracts for Backstage template parameters
3. Implement event-driven bridge: Transformation domain events → Backstage template runs

**Related backlogs**:
- [310-agents-v2.md](310-agents-v2.md) – AgentRuntime layer (n8n-aligned); fits into §4 conceptual model
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
- [310-agents-v2.md](310-agents-v2.md) – AgentRuntime layer (n8n-aligned)
- [305-workflow.md](305-workflow.md) – Workflow orchestration (Temporal)
