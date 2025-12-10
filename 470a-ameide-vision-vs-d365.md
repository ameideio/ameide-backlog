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

#### CQRS Recipe for D365 Architects

For architects migrating from D365, here's how the CQRS pattern (see [472 §2.8.6](472-ameide-information-application.md)) maps:

| D365 Pattern | Ameide Pattern | Notes |
|--------------|----------------|-------|
| X++ `insert()`, `update()`, `delete()` | Command RPCs (`CreateFoo`, `UpdateFoo`, `DeleteFoo`) | Explicit operations vs implicit table methods |
| X++ `select` / Query framework | Query RPCs (`GetFoo`, `ListFoos`) | Proto request/response vs X++ query |
| Business Events (`BusinessEventsContract`) | Event messages in `events/v1` | Published via Watermill vs D365 event framework |
| Data Entity OData `POST/PATCH/DELETE` | Command RPCs via SDK | Unified gRPC/REST gateway vs separate OData surface |
| Data Entity OData `GET` | Query RPCs via SDK | Same SDK for queries and commands |

**Migration recipe:**

1. **Tables → Entities**: Convert `SalesTable` fields to `SalesOrder` proto message
2. **X++ Service operations → Commands**: Convert `SalesOrderService.create()` to `CreateSalesOrder` RPC
3. **Query objects → Query RPCs**: Convert `SalesOrderQuery` to `ListSalesOrders` RPC with filter/pagination
4. **Business Events → Events**: Convert `SalesOrderConfirmedBusinessEvent` to `SalesOrderConfirmed` event message
5. **Data Entities → Projections**: Convert `SalesOrderEntity` to `SalesOrderProjection` message (read-optimized)

---

### 7.8 Security Model Comparison (D365 → Ameide)

> **Full security model**: See [476-ameide-security-trust.md](476-ameide-security-trust.md) for Ameide's complete security architecture. This section maps D365 security terminology so architects can reason about equivalents.

#### Role & Permission Model

| D365 Concept | Ameide Equivalent | Notes |
|--------------|-------------------|-------|
| **Security roles** | Tenant/org roles + domain RBAC | Keycloak roles (`admin`, `contributor`, `viewer`) scoped to tenant/org |
| **Duties** | Domain-level permission groups | Logical groupings enforced in proto service methods |
| **Privileges** | RPC-level permissions | Each proto RPC can require specific permissions |
| **Process cycles** (duties) | Process primitive task permissions | Workflow steps carry permission requirements |

#### Data Security

| D365 Concept | Ameide Equivalent | Notes |
|--------------|-------------------|-------|
| **Table permissions** (CRUD) | Domain API + RLS policies | Proto service controls access; RLS enforces at DB level |
| **Field security** | Proto field visibility + service logic | Sensitive fields excluded from projections or masked in code |
| **Record-level security** | RLS predicates (`tenant_id`, `organization_id`) | Row-Level Security with JWT session variables |
| **Organization hierarchies** | Org membership + role inheritance | `organization_id` scoping with parent org visibility |

#### UI & Menu Security

| D365 Concept | Ameide Equivalent | Notes |
|--------------|-------------------|-------|
| **Security keys** (menu items) | UISurface CRD `auth_scopes` | Declares which roles can access the surface |
| **Menu item permissions** | Process/task-level permissions | Workflow tasks gated by role membership |
| **Form control visibility** | Code-first UI with role checks | React components conditionally render based on SDK role checks |

#### Extensibility Security

| D365 Concept | Ameide Equivalent | Notes |
|--------------|-------------------|-------|
| **CoC (Chain of Command)** | Tier 1 WASM hooks | Scoped host calls; sandboxed execution |
| **Overlayering** (deprecated) | Tier 2 custom primitives | Tenant namespace isolation; never modifies platform code |
| **ISV solutions** | Tier 2 primitives + Backstage templates | Full Domain/Process/Agent/UISurface in tenant namespace |

**Key architectural difference**: D365 enforces security via AOT metadata (security roles, duties, privileges stored in model). Ameide enforces via:
1. **JWT claims** (tenant, org, roles) validated at API gateway
2. **Proto service interceptors** checking permissions per RPC
3. **RLS policies** at database level (defense in depth)
4. **UISurface CRD auth_scopes** gating UI access

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
