# 470.B – Ameide Code-First / AI-First ERP vs. SAP S/4HANA Metadata- & Config-Driven ERP

**Status:** Draft  
**Owner:** Architecture / Product  
**Companion:** [470-ameide-vision.md](470-ameide-vision.md)
**Related:** [520-primitives-stack-v2.md](520-primitives-stack-v2.md) (normative primitives stack v2)

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

> A **code-first, proto-centric system** where business logic lives in Go/Temporal/TS in a single repo, with **six primitives** modeled as Kubernetes CRDs: **Domain / Process / Agent / UISurface / Projection / Integration**; an **AI agent** is the primary “developer” that edits code and specs.

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
     * `Projection` – read-optimized consumption/analytics
     * `Integration` – flows-as-code external integration

	   * No CRDs for “table”, “field”, “form”, or “UI annotation”.
	   * CRDs are **operational only** (operators reconcile lifecycle/config); behavior remains **code + proto**, with `buf generate` as the canonical generator runner and CI “regen-diff” guardrails.

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
  * manipulates primitive CRDs (Domain/Process/Agent/UISurface/Projection/Integration).
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
  * reads primitive CRDs and module manifests.

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

#### CQRS Recipe for SAP Architects

For architects migrating from S/4HANA, here's how the CQRS pattern (see [472 §2.8.6](472-ameide-information-application.md)) maps:

| SAP Pattern | Ameide Pattern | Notes |
|-------------|----------------|-------|
| BAPIs with `COMMIT WORK` | Command RPCs (`CreateFoo`, `UpdateFoo`) | Explicit operations vs BAPI conventions |
| SELECT on DDIC tables / CDS views | Query RPCs (`GetFoo`, `ListFoos`) | Proto request/response vs ABAP OpenSQL |
| RAP Business Events | Event messages in `events/v1` | Published via Watermill vs RAP event framework |
| OData `POST/PATCH/DELETE` on A_* services | Command RPCs via SDK | Unified SDK vs separate OData layer |
| OData `GET` / `$filter` / `$expand` | Query RPCs via SDK | Same SDK; filter params in request message |
| IDocs (outbound) | Event messages with `BUSINESS` kind | Async integration; proto replaces IDoc segments |

**Migration recipe:**

1. **DDIC tables → Entities**: Convert `VBAK`/`VBAP` fields to `SalesOrder` / `SalesOrderItem` proto messages
2. **BAPIs → Commands**: Convert `BAPI_SALESORDER_CREATEFROMDAT2` to `CreateSalesOrder` RPC
3. **CDS views → Query RPCs**: Convert `C_SalesOrderItemFS` to `ListSalesOrderItems` RPC with filter/pagination
4. **Business Events → Events**: Convert `SalesOrder.Created.v1` RAP event to `SalesOrderCreated` event message
5. **IDocs → Events**: Convert `ORDERS05` outbound to `SalesOrderCreated` event with `BUSINESS` kind
6. **CDS projections → Projections**: Convert `C_SalesOrderTP` consumption view to `SalesOrderProjection` message

**Process boundary alignment:**

| SAP Concept | Ameide Concept |
|-------------|----------------|
| Transaction boundary (LUW) | Command RPC scope |
| Application Jobs / BTC | Process primitive (Temporal workflow) |
| BADI / Enhancement Spot | Tier 1 WASM hook or Tier 2 custom primitive |
| BRF+ / Decision Table | Agent primitive with deterministic rules |

---

### 6.8 Security Model Comparison (S/4HANA → Ameide)

> **Security Metamodel**: Ameide provides an ERP-class security metamodel with D365/S4-familiar vocabulary. For full details, see [476-ameide-security-trust.md §7](476-ameide-security-trust.md). This section maps SAP security terminology so architects can reason about equivalents.

#### Security Metamodel Concepts

| S/4HANA Concept | Ameide Concept | Implementation |
|-----------------|----------------|----------------|
| **Authorization Object** | `AuthorizationDescriptor` | Proto method + required dimensions + scope + risk tier |
| **Auth Object Fields** | `AuthorizationDimension` | Tenant/org/company/plant columns with RLS policies |
| **PFCG Role** | `OrganizationRole` + `Duty` refs | Keycloak realm role with duties in `permissions` map |
| **Org Level Values** | Dimension claims in JWT | `ext.company_codes`, `ext.plants` in token |
| **SU53 (Auth Trace)** | Interceptor audit logging | OTel traces with descriptor checks |
| **GRC SoD Matrix** | `SoDRule` | Conflicting descriptors checked at role assignment time |

**Key differences from S/4:**

1. **No DDIC layer** – S/4 has authorization object definitions in DDIC. Ameide keeps these as design-time artifacts in the Transformation Domain.

2. **Explicit dimension columns** – S/4 org levels are configured per role. Ameide dimensions are explicit columns (`company_code`, `plant_id`) on domain tables with RLS enforced at the database layer.

3. **Real-time SoD enforcement** – S/4 SoD is typically evaluated in GRC batch jobs. Ameide evaluates SoD rules at role assignment time, blocking conflicts before they occur.

4. **Tenant isolation as hard invariant** – S/4 relies on client configuration. Ameide enforces `tenant_id` + `organization_id` via RLS + JWT claims as non-negotiable guardrails.

5. **Agent governance** – S/4 has no AI agent concept. Ameide adds `scope` and `risk_tier` to AgentDefinitions, ensuring AI workers operate within explicit capability bounds.

#### Authorization Objects & Roles

| SAP Concept | Ameide Equivalent | Notes |
|-------------|-------------------|-------|
| **Authorization objects** | Proto service methods + RLS predicates | Auth object fields → RPC parameters + session variables |
| **Authorization fields** | `tenant_id`, `organization_id`, role claims | JWT claims validated at gateway and in RLS |
| **Roles (PFCG)** | Keycloak roles scoped to tenant/org | `admin`, `contributor`, `viewer`, `guest`, `service` |
| **Role profiles** | Role bundles in Keycloak | Composite roles for persona-based access |
| **Activity codes** (01-27) | CRUD permissions on proto services | Mapped to Create/Read/Update/Delete RPCs |

#### Data Authorization

| SAP Concept | Ameide Equivalent | Notes |
|-------------|-------------------|-------|
| **AUTHORITY-CHECK** | Service interceptor + RLS | Proto interceptors check claims; RLS enforces at DB |
| **Org-level auth (BUKRS, WERKS)** | `organization_id` + RLS predicates | Row-Level Security with session variables |
| **DCL (Data Control Language)** | RLS policies per domain | Postgres RLS with tenant/org predicates |
| **Access controls (CDS)** | Proto field visibility + projection design | Sensitive fields excluded or masked |

#### Fiori / UI Security

| SAP Concept | Ameide Equivalent | Notes |
|-------------|-------------------|-------|
| **Catalogs & groups** | UISurface CRD `auth_scopes` | Declares which roles see which surfaces |
| **Tile visibility** | Code-first UI role checks | React components render based on SDK role checks |
| **App-to-app navigation** | Process primitive flow with task permissions | Workflow steps gated by role membership |

#### Key-User & Developer Extensibility

| SAP Concept | Ameide Equivalent | Notes |
|-------------|-------------------|-------|
| **Key-user extensions** (custom fields, logic) | Tier 1 WASM hooks | Sandboxed; scoped host calls only |
| **Developer extensions** (RAP, CAP) | Tier 2 custom primitives | Full Domain/Process/Agent in tenant namespace |
| **Clean core boundary** | Namespace isolation (`ameide-*` vs `tenant-*`) | Platform code never modified by tenants |
| **Execution context** (user, system, elevated) | SDK context with tenant/org/role claims | All SDK calls carry execution context |

#### Clean Core Security Model

| SAP Clean Core Principle | Ameide Implementation |
|--------------------------|----------------------|
| No modification of standard | Tier 2 primitives in tenant namespace; platform is immutable |
| Key-user tools for safe extension | Tier 1 WASM with host-call policies (476 §8.4) |
| Released APIs only | Integration interfaces (`INTEGRATION` kind) are stable contracts |
| Side-by-side extensions | Tenant namespace isolation; NetworkPolicies |

**Key architectural difference**: SAP enforces security via authorization objects checked at runtime (`AUTHORITY-CHECK`) plus DCL in CDS. Ameide enforces via:
1. **JWT claims** (tenant, org, roles) validated at API gateway
2. **Proto service interceptors** checking permissions per RPC
3. **RLS policies** at database level (defense in depth)
4. **Namespace isolation** for tenant code (K8s NetworkPolicies)

---

## 7. Summary

Compared to **SAP S/4HANA’s metadata- and configuration-driven architecture**, Ameide:

* Chooses **code-first, AI-first** over config-first and key-user tooling.
* Replaces DDIC + CDS + IMG + BAdI frameworks with:

  * a **single codebase**,
  * **proto contracts**,
  * **primitive CRDs** (Domain/Process/Agent/UISurface/Projection/Integration),
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
