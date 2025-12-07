

# 1. Vision – Ameide Intelligent Business Platform

**File:** `docs/100-vision-ameide-platform.md`
**Audience:** founders, product, senior architects, GTM
**Scope:** *why* Ameide exists, *what* it is, and high‑level “shape” — no tech.


---

## 1.1 Purpose

Ameide exists to replace the fragmented landscape of ERP/CRM/HCM/MES “silos” with a **single, intelligent, cloud‑native business platform**.

Instead of buying and customizing monolithic apps, customers:

* articulate requirements in natural language,
* work in **process‑centric workspaces** (L2O, O2C, P2P, H2R…),
* and let Ameide **develop, deploy, and evolve** the underlying business logic and UIs.

Transformation is **first‑class**, not an afterthought.

---

## 1.2 Vision Statement

> **Ameide is an intelligent, multi‑tenant, cloud‑native business application platform** that:
>
> * models all business domains as **deterministic services** (domain controllers),
> * runs **end‑to‑end processes** as explicit BPMN workflows over those services,
> * and uses **agents** to continuously design, implement, and improve the platform itself.

## Core Insight

> **"By abstracting domain/process/agent primitives, we enable the Ameide Architect Agent to provision resources declaratively without heavy software engineering."**

The goal isn't K8s elegance - it's **agent-driven platform evolution**. The Architect Agent needs a stable abstraction layer to create/modify:
- Domain services (IDC)
- Business processes (IPC)
- AI agents (IAC)

Without this, every change requires traditional software engineering.

---

## 1.3 Core Value Propositions

1. **One platform, many “ERPs”**

   * All classic domains (Sales, Procurement, Finance, HR, Manufacturing…) live on one platform.
   * New verticals or customer‑specific domains are **new controllers + processes**, not new products.

2. **Process‑first, not module‑first**

   * Users work in **Lead‑to‑Order, Order‑to‑Cash, Procure‑to‑Pay** etc., not “Leads / Opportunities / Orders” silos.
   * Each process is a first‑class model (BPMN), with built‑in metrics, SLAs, and bottleneck analysis. 

3. **Deterministic core, intelligent edges**

   * All critical domain logic is deterministic and testable.
   * Agents propose designs, code, mappings, and optimizations; deterministic controllers enforce rules and persist state.

4. **Transformation as a domain**

   * Requirements, architecture models, backlog, and change workflows are themselves modeled as **domain + process**, with full audit via the Unified Artifact Framework. 

5. **Cloud‑native from day 1**

   * Ameide is not “ERP hosted on Kubernetes” — it **is** a K8s‑native platform with operators, CRDs, and GitOps at the core. 

---

## 1.4 Guiding Principles (business-level)

1. **Universal DDD** – every concept has one owner; no overlapping responsibility.
2. **Process at the center** – value streams are primary; modules are just one view.
3. **Agentic everywhere, but governed** – agents accelerate change; they do not bypass controls.
4. **Cloud‑native substrate** – elasticity, resilience, and automation assume Kubernetes, not bare VMs.
5. **Open contracts, not lock‑in** – proto APIs, BPMN, and artifact formats are open and inspectable.

---

# 2. Business Architecture – How Ameide Is Used & Developed

**File:** `docs/110-business-architecture-ameide.md`
**Audience:** business architects, solution architects, process owners, consulting partners
**Scope:** roles, capabilities, business processes (including transformation), how “ERP on Ameide” looks.

---

## 2.1 Key Roles

* **Business User**

  * Works in process workspaces (L2O, O2C, P2P, HR lifecycle).
  * Creates and updates operational data (orders, invoices, employees…).

* **Process Owner / Business Architect**

  * Owns a **value stream** (e.g. L2O).
  * Designs & curates BPMN models and KPIs.
  * Approves changes proposed by the platform or agents.

* **Solution Architect / Domain Architect**

  * Defines domain boundaries and responsibilities.
  * Decides when to introduce new **domain controllers** or processes.
  * Oversees tenant‑specific variants.

* **Platform Engineer**

  * Operates clusters, GitOps, operators.
  * Manages shared infra (Kafka, Temporal, gateways).

* **Transformation Lead / COE**

  * Owns the **Transformation domain** (requirements, backlog, architecture).
  * Prioritizes work across tenants.

---

## 2.2 Domain vs Process vs Agent (business view)

* **Domain (what the business *is*)**

  * Sales, Orders, Inventory, Invoices, HR, Projects, etc.
  * Each domain has:

    * a canonical data model (spec/status),
    * clear states (e.g. `Draft → Confirmed → Cancelled`).

* **Process (how the business *flows*)**

  * L2O, O2C, P2P, H2R, Transformation.
  * Each process is:

    * a BPMN model,
    * a set of **steps** mapped to domain operations,
    * measured (cycle time, throughput, conversion).

* **Agents (how the business *evolves and optimizes*)**

  * Assist with:

    * capturing requirements,
    * refactoring processes,
    * generating new domain controllers or UIs,
    * explaining system behavior.
  * **They never *own* business truth**; they propose, not decree.

---

## 2.3 Typical Business Journeys

### 2.3.1 Running a process (L2O)

1. Sales rep opens **L2O workspace**:

   * sees pipeline and active process instances.
2. Works through tasks (qualify lead, prepare quote, confirm order).
3. The process controller:

   * calls domain controllers (Sales, Orders),
   * tracks progress and SLAs.

The rep can also open “classic” lists (Leads, Opportunities, Orders) for comfort, but everything is powered by the same controllers.

---

### 2.3.2 Transforming the platform

1. Business user says: “We need a risk check before confirming high‑value orders.”
2. Transformation process kicks off:

   * Requirement is stored in **Transformation domain** (backlog item).
   * Agent proposes:

     * BPMN change for O2C or L2O,
     * new `Risk` domain controller or new primitive on an existing controller.
3. Process Owner reviews & approves:

   * BPMN diff,
   * risk rules,
   * rollout plan.
4. Platform Engineer / GitOps deploys:

   * new/updated domain controller spec,
   * new process spec.
5. Change goes live with:

   * updated workspaces,
   * metrics for new step.

Transformation itself is modeled as **Scrum / TOGAF‑like processes** running on the same substrate (Transformation IDC + IPCs).

---

## 2.4 Business-Level Policies for Agents

* **Deterministic core**

  * Only domain controllers may persist core business state.
  * All persisted transitions must be deterministic and testable.

* **Agents propose, processes decide**

  * Agents suggest changes to:

    * controller specs,
    * BPMN models,
    * UI layouts.
  * Changes are applied only via governed processes (review, approval, rollout).

* **Explainability**

  * Agents must produce human‑readable reasoning and diffs for proposals (code changes, BPMN edits, config changes). 

---

# 3. Information & Application Architecture

**File:** `docs/120-information-application-architecture-ameide.md`
**Audience:** application architects, senior engineers, platform architects
**Scope:** domains, processes, data, CRDs, and how they relate; where agents live and how they’re orchestrated.

---

## 3.1 Logical Components

1. **Intelligent Domain Controllers (IDCs)**

   * K8s CRDs + services that own domain models and persistence.

2. **Intelligent Process Controllers (IPCs)**

   * K8s CRDs + Temporal workers that implement BPMN processes over IDCs.

3. **Intelligent Agent Controllers (IACs)** — *new*

   * K8s CRDs + agent runtimes (LLM + tools) invoked by IPCs and some domain workflows.

4. **Unified Artifact Framework (UAF)**

   * Event‑sourced artifacts: BPMN, ArchiMate, Markdown; revisions + promotions.

5. **API & SDK Layer**

   * Proto‑based APIs, Connect/REST gateway, TS/Go/Python SDKs.

6. **UI Shell & Microfrontends**

   * Workspaces for domains and processes.

---

## 3.2 Intelligent Domain Controller (IDC) – Information Model

**CRD:** `IntelligentDomainController` (IDC)
**Owns:**

* **Data model**:

  * tables/collections, including `spec` and `status`,
  * `tenant_id` for multi‑tenancy,
  * invariants and constraints.

* **Proto API**:

  * one or more services (e.g. `OrdersService`),
  * methods annotated as **primitives** (used in BPMN/IPCs). 

* **Lifecycle & state machine**:

  * allowed states, transitions.

* **Event streams**:

  * topics and schemas for domain events.

* **Not agents**:

  * IDC operations must be deterministic: no LLM calls in core decision paths.
  * They may **consume agent suggestions** as data (e.g. “proposed GL mapping”), but must validate deterministically before changing state.

---

## 3.3 Intelligent Process Controller (IPC) – Information Model

**CRD:** `IntelligentProcessController` (IPC)
**Owns:**

* **Process definition**:

  * BPMN artifact ID + revision from UAF.

* **Binding to domain primitives**:

  * maps BPMN service tasks to IDC methods (primitives).

* **Binding to agents**:

  * maps selected BPMN tasks (e.g. “Propose schedule”) to **Agent tools** on IACs.

* **Process state**:

  * process instances, tasks, correlation IDs, SLAs (stored in Temporal + projections).

IPC is **where non‑determinism is orchestrated**:

* It can:

  * call deterministic IDCs,
  * invoke non‑deterministic agents (IACs),
  * decide how and when to accept agent outputs (e.g. human‑in‑the‑loop).

---

## 3.4 Intelligent Agent Controller (IAC) – Information Model

**CRD:** `IntelligentAgentController` (IAC)
**Owns:**

* **Agent graph definition** (like IPADomain, but scoped to agents/tools only).
* **LLM/runtime configuration**:

  * model, temperature, safety settings, rate limits.
* **Tool schema**:

  * which tools it can call:

    * read‑only queries to IDCs,
    * proposals for changes (not direct mutations),
    * code actions (via `core-platform-coder`). 
* **Scope & permissions**:

  * domains/processes/tenants where it’s allowed.

IACs are **non‑deterministic by design**:

* Their outputs are treated as:

  * suggestions,
  * candidate code/config,
  * draft BPMN/controller configs.

IPC and Transformation processes decide whether to:

* accept as‑is (for low‑risk changes),
* or route for human approval.

---

## 3.5 Data Flows

### 3.5.1 Runtime

* **Commands**:

  * UI → SDK → IDC/IPCs (synchronous RPC).

* **Events**:

  * IDCs publish domain events to Kafka (e.g. `OrderCreated`).
  * IPCs listen and correlate to process instances.

* **Agent calls**:

  * IPCs call IACs through well‑typed APIs (proto) and treat outputs as data.

### 3.5.2 Design & Transformation

* BPMN, ArchiMate, Markdown stored in UAF with `GenericCommand` histories and snapshots.

* Transformation IDC stores:

  * backlog items,
  * mappings from transformation tasks → IDC/IPC/IAC changes.

* Agents (via IACs) can:

  * propose new artifacts,
  * edit controller specs,
  * but must go through promotion + GitOps processes to affect runtime.

---

## 3.6 Agent Placement Decision

You asked:

> “Agents are possibly another crd operator and consumable from both processes and domains. Or domain components used by processes? Maybe best to keep domain deterministic and agents non deterministic and process orchestrating all.”

**Decision:**

* **Domain controllers remain deterministic.**

  * No LLM calls in core decision paths or internal invariants.
  * They can consume agent‑generated data but must re‑validate.

* **Agents are their own CRD/operator (IAC).**

  * IACs encapsulate agent graphs, tools, and LLM config.
  * They are invoked from:

    * IPCs (primary),
    * occasionally from IDCs *via explicit “assist” paths* (e.g. suggesting a default but not auto‑applying it).

* **Processes orchestrate all.**

  * IPCs coordinate deterministic domain changes and non‑deterministic agent calls.
  * This guarantees:

    * auditability,
    * testability,
    * clear separation of responsibilities.

---

# 4. Technology & Platform Architecture

**File:** `docs/130-technology-architecture-ameide.md`
**Audience:** platform engineers, infra/SRE, senior devs
**Scope:** K8s, operators, Temporal, Kafka, gateways, SDKs, CI/CD.

---

## 4.1 Runtime Stack

* **Kubernetes** (AKS/EKS/GKE/on‑prem)
* **Operators & CRDs**

  * `IntelligentDomainController` (IDC)
  * `IntelligentProcessController` (IPC)
  * `IntelligentAgentController` (IAC)
* **Compute**

  * IDCs: Go/Rust services implementing proto contracts. 
  * IPCs: Temporal workers (Go/Python).
  * IACs: agent runtimes (LangGraph/LLM services) with tool bridges.
* **Data**

  * Postgres (domain state, process projections).
  * Object store (BPMN, artifacts).
  * Event store / Kafka (events).
* **Control**

  * GitOps (ArgoCD/Flux) for CRDs & charts. 
* **Edge**

  * Connect‑based API gateway (gRPC/JSON over HTTP).
  * TS/Go/Python SDKs for clients.

---

## 4.2 Operators & Helm

* **Helm**

  * Install system components:

    * IDC Operator,
    * IPC Operator,
    * IAC Operator,
    * Kafka, Temporal, gateways, observability. 

* **Operators**

  * Reconcile CRDs → Deployments/Services/Jobs.
  * Manage spec/status, health, migrations, and event topics.

Each new IDC/IPC/IAC is introduced by:

1. Adding/Updating a CRD instance (YAML) in the GitOps repo.
2. ArgoCD applies it.
3. Relevant operator reconciles it.

Agents (through tools like core‑platform‑coder) can propose and update those YAMLs, but Git review can gate merges.

---

## 4.3 API & SDK

* **Proto & API services** per 044: auth, error mapping, unit of work, tracing. 

* **TS SDK** per 050:

  * Connect‑ES transport,
  * `AmeideClient` wrapper,
  * used by UI and some agents. 

* **Connect gateway**:

  * exposes gRPC + REST + streaming endpoints,
  * implements authn/z, rate limiting, correlation IDs.

---

## 4.4 Temporal & BPMN Execution

* BPMN stored and versioned via UAF & North‑Star design pipeline.
* IPC operator:

  * compiles BPMN model to Temporal workflows/activities,
  * deploys workers,
  * binds them to IDCs & IACs.

Camunda support remains optional/legacy; Temporal is the primary process engine.

---

## 4.5 Observability, Security, Compliance

* **Observability**

  * OpenTelemetry traces across UI → gateway → IDC/IPC/IAC → DB/Kafka.
  * Process KPIs (per BPMN model, per tenant).

* **Security**

  * OIDC (Keycloak) + RBAC.
  * Tenant isolation via:

    * JWT claims,
    * DB schema isolation,
    * per‑tenant settings in CRDs.

* **Compliance & Audit**

  * UAF command logs and revisions for all design artifacts.
  * Git history + CRDs for runtime configuration.
  * Process logs & events for runtime behavior.

---

## 4.6 Dev & CI/CD

* **Buf & proto pipelines** (lint, breaking‑change checks).
* **SDK pipelines** for TS/Go/Python releases.
* **Core‑platform‑coder** and similar agents embedded in CI:

  * automated test refactors,
  * code hygiene, under gatekeepers.

---

# 5. Strategic Rationale – Why IDC/IPC/IAC Abstraction

**File:** `docs/140-strategic-rationale-idc-ipc-iac.md`
**Audience:** architects, engineering leads, product
**Scope:** the "why" behind the abstraction layer

---

## 5.1 The Core Insight

> **By abstracting domain, process, and agent primitives into declarative CRDs, we enable the Ameide Architect Agent to provision and evolve the platform without traditional software engineering.**

This is not about Kubernetes elegance. It's about **agent-driven platform evolution**.

Without this abstraction:
* Every new domain requires Go/Python code, Helm charts, migrations, tests
* Every process change requires workflow code modifications
* Every agent addition requires deployment configuration
* **Result:** The Architect Agent cannot autonomously evolve the platform

With this abstraction:
* New domain = new `IntelligentDomainController` YAML
* New process = new `IntelligentProcessController` YAML + BPMN artifact
* New agent = new `IntelligentAgentController` YAML
* **Result:** The Architect Agent generates YAML, Git review gates it, ArgoCD deploys it

---

## 5.2 What the Architect Agent Can Do

With IDC/IPC/IAC primitives, the Architect Agent can:

| Action | Without Abstraction | With IDC/IPC/IAC |
|--------|---------------------|------------------|
| Add new domain (e.g., `Risk`) | Write Go service, proto, Helm chart, migrations | Generate `IntelligentDomainController` YAML |
| Add domain primitive | Modify service code, update proto | Add method to IDC spec, regenerate |
| Create new process | Write Temporal workflow code | Generate `IntelligentProcessController` YAML + BPMN |
| Bind process to domains | Hardcode in workflow activities | Declare bindings in IPC spec |
| Add AI capability | Deploy new service, configure routing | Generate `IntelligentAgentController` YAML |
| Modify agent tools | Code changes + redeploy | Update IAC tool schema |

**The abstraction transforms software engineering into configuration management.**

---

## 5.3 Governance Model

The Architect Agent proposes, humans (or automated policies) approve:

```
User Request
    ↓
Architect Agent analyzes requirements
    ↓
Agent generates IDC/IPC/IAC YAML + BPMN artifacts
    ↓
Git PR created (via core-platform-coder)
    ↓
Review gate (human or policy-based)
    ↓
Merge → ArgoCD sync
    ↓
Operators reconcile → Resources deployed
```

Low-risk changes (e.g., adding a field) can auto-merge.
High-risk changes (e.g., new domain) require human review.

---

## 5.4 Current State vs Target State

### Current Implementation

| Component | Current State |
|-----------|---------------|
| Domain services | Standard K8s Deployments (graph, threads, platform, workflows, agents) |
| Process orchestration | Temporal as external service + hardcoded workflows |
| Agent services | agents + agents-runtime Deployments |
| Provisioning | Manual Helm chart creation + GitOps |

### Target State

| Component | Target State |
|-----------|--------------|
| Domain services | IDC Operator reconciles `IntelligentDomainController` → Deployment + Service + Kafka topics |
| Process orchestration | IPC Operator compiles BPMN → Temporal workflows, binds to IDC primitives |
| Agent services | IAC Operator deploys agent runtimes with declared tools and permissions |
| Provisioning | Architect Agent generates CRD YAMLs → Git → ArgoCD → Operators |

---

# 6. Implementation Plan

**File:** `docs/150-implementation-plan-idc-ipc-iac.md`
**Audience:** engineering leads, platform team
**Scope:** phased delivery plan

---

## 6.1 Phase 0: Foundation (Current ✅)

Already in place:
- [x] ArgoCD + GitOps pipeline
- [x] Third-party operators (Strimzi, CNPG, Redis, ClickHouse, Keycloak)
- [x] Temporal deployed as service
- [x] agents + agents-runtime services
- [x] Proto-based APIs with buf pipelines
- [x] Multi-environment support (dev/staging/prod)

---

## 6.2 Phase 1: IAC Operator (Agents First)

**Why start here:** Agents already exist as services. IAC formalizes what we have.

### Deliverables

1. **Define `IntelligentAgentController` CRD**
   ```yaml
   apiVersion: ameide.io/v1
   kind: IntelligentAgentController
   metadata:
     name: architect-agent
     namespace: ameide-prod
   spec:
     displayName: "Ameide Architect"
     runtime:
       image: ghcr.io/ameideio/agents-runtime:v1
       model: claude-sonnet-4-20250514
       temperature: 0.2
       rateLimits:
         requestsPerMinute: 60
     tools:
       - name: read-domain-specs
         type: query
         targetKind: IntelligentDomainController
       - name: propose-idc-change
         type: proposal
         targetKind: IntelligentDomainController
       - name: generate-code
         type: action
         handler: core-platform-coder
     permissions:
       tenants: ["*"]
       domains: ["*"]
       processes: ["transformation/*"]
   ```

2. **Build IAC Operator** (Go, controller-runtime)
   - Reconcile IAC → Deployment + ConfigMap + ServiceAccount
   - Inject tool configurations into agents-runtime
   - Manage RBAC for tool permissions

3. **Migrate existing agents** to IAC CRDs
   - `general-agent` → IAC
   - `architect-agent` → IAC
   - `coder-agent` → IAC

### Success Criteria
- Agents deployed via `kubectl apply -f iac.yaml`
- Tool permissions enforced by operator
- Architect Agent can query its own IAC spec

---

## 6.3 Phase 2: IDC Operator (Domains)

**Why second:** Domain services are the foundation; IAC tools need them.

### Deliverables

1. **Define `IntelligentDomainController` CRD**
   ```yaml
   apiVersion: ameide.io/v1
   kind: IntelligentDomainController
   metadata:
     name: orders
     namespace: ameide-prod
   spec:
     displayName: "Orders Domain"
     dataModel:
       tables:
         - name: orders
           columns:
             - name: id
               type: uuid
               primaryKey: true
             - name: tenant_id
               type: uuid
               index: true
             - name: status
               type: string
               enum: [draft, confirmed, shipped, cancelled]
             - name: spec
               type: jsonb
             - name: status_data
               type: jsonb
     stateMachine:
       initialState: draft
       transitions:
         - from: draft
           to: confirmed
           primitive: ConfirmOrder
         - from: confirmed
           to: shipped
           primitive: ShipOrder
         - from: [draft, confirmed]
           to: cancelled
           primitive: CancelOrder
     api:
       package: ameide.orders.v1
       services:
         - name: OrdersService
           primitives:
             - name: CreateOrder
               input: CreateOrderRequest
               output: Order
             - name: ConfirmOrder
               input: ConfirmOrderRequest
               output: Order
     events:
       topic: orders.events
       schemas:
         - OrderCreated
         - OrderConfirmed
         - OrderShipped
   ```

2. **Build IDC Operator** (Go, controller-runtime)
   - Reconcile IDC → Deployment + Service + Migrations + Kafka topics
   - Generate proto stubs from spec (or validate against existing)
   - Run migrations on spec changes
   - Publish state machine to agents for constraint awareness

3. **Migrate existing domains** to IDC CRDs
   - Start with simpler domains (e.g., `platform`, `graph`)
   - Preserve existing proto contracts

### Success Criteria
- New domain deployed via `kubectl apply -f idc.yaml`
- State machine enforced at runtime
- Architect Agent can generate valid IDC specs

---

## 6.4 Phase 3: IPC Operator (Processes)

**Why third:** Processes bind IDCs and IACs together.

### Deliverables

1. **Define `IntelligentProcessController` CRD**
   ```yaml
   apiVersion: ameide.io/v1
   kind: IntelligentProcessController
   metadata:
     name: lead-to-order
     namespace: ameide-prod
   spec:
     displayName: "Lead to Order"
     process:
       bpmnArtifactId: "uaf://processes/l2o/main.bpmn"
       revision: "2024-01-15-v3"
     bindings:
       serviceTasks:
         - taskId: "create-lead"
           target:
             kind: IntelligentDomainController
             name: sales
             primitive: CreateLead
         - taskId: "create-order"
           target:
             kind: IntelligentDomainController
             name: orders
             primitive: CreateOrder
         - taskId: "propose-discount"
           target:
             kind: IntelligentAgentController
             name: pricing-agent
             tool: suggest-discount
     slas:
       - taskId: "approve-quote"
         maxDuration: 24h
         escalateTo: sales-manager
     metrics:
       - name: conversion_rate
         formula: "completed / started"
       - name: cycle_time
         formula: "avg(completed_at - started_at)"
   ```

2. **Build IPC Operator** (Go, controller-runtime)
   - Reconcile IPC → Temporal workflow definition + workers
   - Compile BPMN → Temporal workflow code (or interpret at runtime)
   - Bind service tasks to IDC primitives and IAC tools
   - Track SLAs and emit metrics

3. **BPMN → Temporal Compiler**
   - Parse BPMN from UAF
   - Generate Temporal workflow/activity stubs
   - Support human tasks, service tasks, gateways, events

### Success Criteria
- New process deployed via `kubectl apply -f ipc.yaml`
- BPMN changes automatically reflected in Temporal
- Architect Agent can generate valid IPC specs with bindings

---

## 6.5 Phase 4: Architect Agent Integration

**The payoff:** Architect Agent can now self-provision.

### Deliverables

1. **Architect Agent tools for IDC/IPC/IAC**
   - `list-controllers` - Query existing IDC/IPC/IAC specs
   - `validate-spec` - Check spec against schema and constraints
   - `generate-idc` - Create domain controller from requirements
   - `generate-ipc` - Create process controller from BPMN + bindings
   - `generate-iac` - Create agent controller with tools
   - `submit-pr` - Create Git PR with generated specs

2. **Transformation Process (IPC)**
   - BPMN process for platform changes
   - Steps: requirement → design → generate specs → review → deploy
   - Bindings to Architect Agent (IAC) for generation steps

3. **Self-improvement loop**
   - Architect Agent can propose changes to its own IAC spec
   - Meta-process for agent evolution with human gates

### Success Criteria
- User says "Add a Risk domain with approval workflow"
- Architect Agent generates IDC + IPC + BPMN
- PR created, reviewed, merged
- New capability live in minutes, not weeks

---

## 6.6 Effort Estimates

| Phase | Scope | Effort |
|-------|-------|--------|
| Phase 1: IAC Operator | CRD + operator + migration | 4-6 weeks |
| Phase 2: IDC Operator | CRD + operator + proto gen + migrations | 8-12 weeks |
| Phase 3: IPC Operator | CRD + operator + BPMN compiler | 8-12 weeks |
| Phase 4: Architect Integration | Tools + transformation process | 4-6 weeks |
| **Total** | | **24-36 weeks** |

### Risk Mitigation

- **Start with IAC** - Lowest risk, highest learning
- **Parallel proto work** - Keep existing services running during transition
- **Feature flags** - Operators can be enabled per-environment
- **Gradual migration** - Convert one domain/process at a time

---

## 6.7 Alternative Approaches Considered

### Option A: Application-Level Abstraction (No K8s Operators)

Store IDC/IPC/IAC specs in database, interpret at runtime.

**Pros:**
- No operator development
- Faster initial implementation

**Cons:**
- Loses GitOps benefits (audit, rollback, PR review)
- Platform service becomes single point of failure
- No K8s-native lifecycle management

**Verdict:** Rejected. GitOps is core to the vision.

### Option B: Code Generation Only

Architect Agent generates full service code, not just specs.

**Pros:**
- Maximum flexibility
- No operator overhead

**Cons:**
- Generated code must be maintained
- Drift between generated and hand-written code
- Higher bar for Architect Agent

**Verdict:** Hybrid approach - operators for common patterns, code gen for custom logic.

### Option C: Existing Low-Code Platforms

Use OutSystems, Mendix, or similar.

**Pros:**
- Mature tooling
- Visual designers

**Cons:**
- Vendor lock-in
- Not K8s-native
- Limited agent integration
- Not open-source

**Verdict:** Rejected. Ameide *is* the platform.

---

## 6.8 Next Steps

1. **Validate IAC CRD schema** with agents team
2. **Prototype IAC operator** in dev environment
3. **Define IDC schema** based on existing domain services
4. **Assess BPMN→Temporal** compilation options (generate vs interpret)
