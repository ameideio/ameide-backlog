# 4x2 – Intelligent Process Controller (IPC)

**Status:** Draft v1
**Audience:** Platform / domain engineers, process owners, architects
**Relates to:** IDC/IAC/UI controller docs, 471/472/473/475/476, 461-ipc-idc-iac, 479/480

---

## 1. Purpose

The **Intelligent Process Controller (IPC)** is the standard way Ameide:

* Executes **end‑to‑end processes** (L2O, O2C, Onboarding, Transformation, etc.).
* Orchestrates **deterministic domains** (IDCs), **non‑deterministic agents** (IACs), and **WASM extensions**.
* Provides a **stable, upgradeable abstraction** so we can manage hundreds of processes via CRDs + operators rather than hand‑rolled services. 

This doc defines the **target state** of IPCs: CRD shape, lifecycle, operator behaviour, and how they wire into Temporal, Transformation/UAF, AgentControllers, and UI.

---

## 2. Position in the architecture

### 2.1 Relations

At a glance:

* **Design‑time**

  * **ProcessDefinition** (BPMN‑compliant, from UAF modeller) lives in **Transformation DomainController** as an artifact. 
  * IPC spec binds that ProcessDefinition to domain primitives, agent tools, and optional WASM extensions. 

* **Runtime**

  * IPC CRD → reconciled to:

    * **Temporal workflow workers** (one per process type). 
    * Service/metrics, RBAC, NetworkPolicies (via IPC operator). 
  * IPC instances are Temporal workflows + Ameide projections (for SLA boards, dashboards). 

* **Other controllers**

  * **IDCs**: hold business data; IPCs call **primitives** on them. 
  * **IACs**: encapsulate agents; IPCs call tools exposed by IACs. 
  * **UI**: process boards and workspaces read IPC projections, not Temporal directly. 

### 2.2 Design‑ vs runtime split

* **Design‑time**

  * ProcessDefinitions + IPC specs live in Transformation (UAF artifacts + controller specs).
  * Agents (Architect/Transformation) can propose new/changed IPC specs as YAML. 

* **Runtime**

  * IPC operator compiles ProcessDefinition + IPC bindings into Temporal workflows/activities or runtime configs.
  * GitOps only applies IPC CRDs; everything else is reconciled. 

---

## 3. IPC CRD – Target Shape

We already have a draft in 461; this section **locks it as the target** (with some refinements for WASM + versioned interfaces). 

```yaml
apiVersion: ameide.io/v1
kind: IntelligentProcessController
metadata:
  name: lead-to-order
  namespace: ameide-prod   # or tenant-{id}-{env}-base for tenant-specific IPCs
spec:
  displayName: "Lead to Order"
  description: "Standard L2O from lead capture to order handover"

  process:
    definitionRef:
      artifactId: "uaf://processes/l2o/main.bpmn"
      revision: "2025-01-15-v3"      # UAF revision id
    interfaceVersion: "ameide.sales.l2o.v1"   # optional semantic version of this IPC

  bindings:
    serviceTasks:
      - taskId: "create-lead"
        target:
          kind: IntelligentDomainController
          name: sales
          primitive:
            service: ameide.sales.v1.SalesService
            method: CreateLead
            version: v1               # Buf package version; must exist in proto registry
      - taskId: "create-order"
        target:
          kind: IntelligentDomainController
          name: orders
          primitive:
            service: ameide.orders.v1.OrdersService
            method: CreateOrder
            version: v1
      - taskId: "propose-discount"
        target:
          kind: IntelligentAgentController
          name: pricing-agent
          tool: suggest-discount
      - taskId: "custom-routing"
        target:
          kind: WasmExtension
          extensionId: acme-routing-rule
          version: "2"

  slas:
    - taskId: "approve-quote"
      maxDuration: 24h
      escalateTo: "role:sales-manager"

  metrics:
    - name: conversion_rate
      formula: "completed / started"
    - name: cycle_time
      formula: "avg(completed_at - started_at)"

  tenancy:
    scope: "tenant"
    tenantSelector:
      skus: ["Shared", "Namespace", "Private"]

  security:
    allowedAgents: ["pricing-agent", "l2o-coach"]
    riskTier: "medium"   # influences agent & WASM policies
```

**Key decisions:**

* IPC **does not embed BPMN**; it references ProcessDefinitions in Transformation/UAF via `artifactId + revision`. 
* Bindings refer to **Buf‑versioned proto services** and agent tools, so we can evolve interfaces while keeping old IPCs pinned. 
* WASM hooks are expressed as a special target kind (`WasmExtension`) which the operator maps to the shared `extensions-runtime` service. 

---

## 4. ProcessController runtime model

### 4.1 Generated/managed runtime artefacts

For each IPC, the **IPC operator** produces/maintains: 

* **Temporal namespace/queue config** (or reuse shared ones per SKU).
* **Worker Deployment**:

  * Runs the Temporal workflow code for this process type.
  * Knows how to call:

    * IDC primitives via Ameide SDK.
    * IAC tools via AgentController APIs.
    * WASM hooks via `extensions-runtime`.
* **ConfigMap**:

  * Compiled mapping from BPMN ids → code handlers / declarative routing.
* **Service + ServiceMonitor**, HPAs, NetworkPolicies.

IPC authors do **not** write Temporal code directly if they don’t want to; the default path is:

> BPMN (UAF) → ProcessDefinition → IPC CRD → IPC operator → Temporal workers.

We keep the door open for “code‑first” power users by allowing them to point to existing workflow implementations in the IPC spec (e.g. `runtime.implementationRef`), but the *platform standard* is model‑driven.

### 4.2 Runtime responsibilities

An IPC instance (Temporal workflow):

* Manages **process state** (stage, variables, deadlines). 
* Orchestrates:

  * Sync calls to IDCs.
  * Async waits on domain events (via subscriptions/Temporal signals). 
  * Calls to IACs (agents) with explicit “accept/reject” decisions.
  * Optional calls to WASM extensions.
* Emits **process events** and **SLA metrics** which project into:

  * Ameide process instance tables.
  * Knowledge graph (read‑only) for analytics/agents. 

IPC **never** owns business data; all real state lives in domains or Transformation.

---

## 5. Lifecycle & upgrades at scale

IPC is the lever for “hundreds of processes but still sane ops”.

### 5.1 Creation

Pathways:

* **Human‑driven**: architect/engineer:

  1. Designs or modifies ProcessDefinition (BPMN) in UAF.
  2. Writes/edits IPC CRD (often via Backstage template).
  3. PR → review → GitOps → IPC operator reconciles.

* **Agent‑assisted**:

  * Architect/Transformation AgentController uses tools `generate-ipc` and `validate-spec` to propose IPC YAML in a PR. 

In both cases the unit is **one IPC CR** + referenced BPMN artifact.

### 5.2 Upgrade & versioning

Upgrades can include:

* New BPMN revision.
* New bindings (e.g. new domain primitive or agent tool).
* SLA/metrics tweaks.

Versioning rules:

* **ProcessDefinition id** + revision are immutable; new behaviour → new revision.
* IPC has its own `interfaceVersion` string and Git history.
* We strongly recommend **rolling upgrades**:

  * New IPC CR (or new version field).
  * Traffic pinning by environment/tenant or by *start time* (new instances use new version, old ones drain).

Operators must support:

* `spec.strategy` options for migration:

  * `OnNewInstances` (default).
  * `BlueGreen` (two IPCs with different names, UIs choose).
  * `Immediate` (only for non‑long‑running processes).

### 5.3 Decommissioning

* Mark IPC as `spec.deprecated: true`.
* Operator:

  * blocks new starts.
  * surfaces warnings in UIs/SDK metadata.
* Once all instances complete, IPC CR can be safely removed.

This pattern is essential when we have many tenant‑specific processes and need a cheap lifecycle story.

---

## 6. IPC & Other Layers

### 6.1 With DomainControllers (IDCs)

* IPCs **never bypass IDCs**:

  * All business changes go through domain APIs; no direct DB writes, no knowledge‑graph writes. 
* IPC binds **service tasks** to specific **primitives** (`service + method + version`).
* For large‑scale upgrades, we can:

  * introduce new primitives in domains (Buf v2),
  * gradually migrate IPCs to those primitives,
  * eventually remove old ones.

### 6.2 With AgentControllers (IACs)

* IPC is the **primary caller** of agents:

  * For each “agent task” in BPMN we map to an IAC tool.
* IPC decides:

  * When to call the agent.
  * How to interpret outputs (auto‑apply vs human‑in‑the‑loop).
  * What to persist (always via IDCs). 

Security:

* Which IACs an IPC may use is part of the IPC spec (`security.allowedAgents`) and enforced by:

  * IAC operator (tool grants).
  * IPC operator (validation).
  * Agent runtime (scope/risk enforcement). 

### 6.3 With WASM extensions (Tier 1)

* IPC uses WASM **only through** the shared `extensions-runtime`:

  * BPMN Extension Tasks map to `target.kind: WasmExtension`.
  * Operator injects the Wiring to call `InvokeExtension` with proper `ExecutionContext`. 

* Typical uses:

  * per‑tenant approval routing,
  * small risk/score rules,
  * custom SLAs.

* All side‑effects still go via:

  * IDCs, IACs, or other controllers (WASM has only host calls, no direct DB/net). 

### 6.4 With UI Controllers

* UI shells surface:

  * process boards,
  * task lists,
  * per‑instance timelines.
* They **never** orchestrate; they:

  * call IPC APIs (`Start`, `Signal`, `Query`), and
  * read projections / metrics for dashboards. 

The UI controller doc will define the MFE + Next.js patterns; here we just assert **IPC is the single orchestration point**.

---

## 7. Security, tenancy & isolation

IPC must obey the same principles as domains and agents. 

* **Tenancy**

  * Every IPC instance runs with `tenant_id` (and often `org_id`) context.
  * Temporal namespaces/queues may be per‑tenant tier; IPC workers enforce tenant context in every call.

* **AuthZ**

  * IPC APIs check roles/permissions before allowing:

    * starting new instances,
    * signalling tasks (completing user work),
    * querying sensitive state.

* **Network**

  * IPC workers are isolated by NetworkPolicies:

    * They can talk only to:

      * IDCs,
      * IACs,
      * `extensions-runtime`,
      * platform infra (Temporal, metrics, logging, auth).

* **Audit**

  * Each important state change:

    * who signalled, with what payload,
    * which agent/WASM extension contributed,
    * which domain primitives were called,
  * is logged and projected into audit/reporting.

---

## 8. Backstage, service catalog, and automation

IPC is a **first‑class component** in the internal platform catalog:

* Catalog `kind: Component` with `type: process-controller`. 
* Templates:

  * “New ProcessController” template asks for:

    * ProcessDefinition ref,
    * initial bindings to domains/agents,
    * SLAs/metrics defaults.
* Architect/Transformation agents use Backstage APIs to:

  * create/update IPC specs,
  * validate them,
  * open PRs.

Operators and GitOps only see IPC CRDs and template outputs; the rest is convention.

---

## 9. Examples

### 9.1 L2O – Lead‑to‑Order

* IPC spec as in the sample above (`lead-to-order`).
* ProcessDefinition from Transformation: L2O BPMN. 
* Bindings to:

  * `Sales`, `Product`, `Pricing`, `Orders` IDCs.
  * L2O‑specific AgentControllers for suggestions.
  * Optional tenant‑specific WASM rules.

### 9.2 Self‑serve Onboarding

* IPC `self-serve-onboarding`:

  * ProcessDefinition stored in Transformation (signup → org → SSO → billing). 
  * Binds to:

    * Platform/Identity IDCs (Tenants, Orgs, Users).
    * Billing and realm provisioning.
    * Optional agents for form helpers / risk checks.

---

## 10. Open questions

1. **Granularity of IPCs**

   * One IPC per macro process (L2O), or split into sub‑processes (Lead‑to‑Opportunity, Opportunity‑to‑Order)?

2. **Code‑first vs model‑first**

   * How much of Temporal can remain hand‑coded vs generated from BPMN?
   * What is the migration path for existing code‑first workflows?

3. **IPC per tenant vs multi‑tenant IPCs**

   * When do we clone an IPC for a tenant vs keep one shared IPC with tenancy‑aware logic and WASM hooks?

4. **Standard metrics & SLAs**

   * Can we define a base library of metrics per IPC type (e.g. conversion, cycle time) that templates always emit?

---

If you’d like, I can now do the same for **IAC (Intelligent Agent Controller)** or the **UI controller** spec, matching this shape and tying tightly into the same architecture.
