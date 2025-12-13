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
> **Core Invariants**: See [470-ameide-vision.md §0 "Ameide Core Invariants"](470-ameide-vision.md) for the canonical list (six primitives: Domain/Process/Agent/UISurface/Projection/Integration, Graph read-only, Transformation as domain, proto chain, tenant isolation, Backstage internal).
>
> **Secrets & Security Implementation**:
> - [462-secrets-origin-classification.md](462-secrets-origin-classification.md) – Secret origin taxonomy
> - [451-secrets-management.md](451-secrets-management.md) – Secrets flow (Azure KV → Vault → K8s)
> - [426-keycloak-config-map.md](426-keycloak-config-map.md) – OIDC client patterns
> - [340-api-routes.md](340-api-routes.md) – API authentication
> - [341-middleware-auth.md](341-middleware-auth.md) – Middleware AuthN/AuthZ
> - [342-token.md](342-token.md) – Token handling

## Grounding & contract alignment

- **Security spine for primitives:** Applies the vision-level invariants from `470-ameide-vision.md` and the domain/process/agent/UISurface patterns from `471-ameide-business-architecture.md`, `472-ameide-information-application.md`, and `475-ameide-domains.md` to define how security attaches to primitives and their CRDs (tenancy, identity, agents, extensions).  
- **Operator & EDA alignment:** Provides the security assumptions that operator and EDA backlogs (`473-ameide-technology.md`, `495-ameide-operators.md`, `496-eda-principles.md`, `497-operator-implementation-patterns.md`, `498-domain-operator.md`, `499-process-operator.md`, `500-agent-operator.md`, `501-uisurface-operator.md`, `503-operators-helm-chart.md`) must respect when wiring workloads and events.  
- **Scrum/agent constraints:** Supplies the agent governance, risk tiers, and isolation model that the Scrum/agent stack (`505-agent-developer-v2*.md`, `504-agent-vertical-slice.md`, `506-scrum-vertical-v2.md`, `508-scrum-protos.md`) relies on when defining PO/SA/Coder behavior and process/event contracts.

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
* 322 defines canonical role catalog (Keycloak, proto/SDK, database seeds)
* 322 defines ownership enforcement patterns for transformations, repositories, teams

**Target activities per 322:**
1. Owner/role checks for org membership mutations → supports §5.1
2. 401/403 telemetry in dashboards → supports §10
3. API integration suites in CI gating → supports §6

### 12.2 Authorization alignment (329)

* 476 §2.1 requires: "DB‑level RLS + JWT tenant claims are non‑negotiable guardrails"
* 476 §4.1 requires: "All platform tables enforce RLS by tenant"
* 476 §5.1 requires: "Enforce tenant + org + role checks in service methods"

**329 defines three phases:**
- Phase 1: API Authorization Layer (`withAuth` middleware)
- Phase 2: Database Organization Scoping (`organization_id` columns, FK constraints)
- Phase 3: Defense in Depth (RLS policies, audit logging)

### 12.3 Agents alignment (310)

* 476 §8.1 requires AgentDefinitions to have `scope` (read-only vs read/write vs code-gen) and `risk_tier` (low, medium, high)
* 476 §8.2 requires deterministic boundaries (Domain primitives/Process primitives invoke Agent primitives, never raw LLM APIs)
* 476 §8.3 requires agent-generated code to propose via Transformation design tooling/Git artifacts, not direct writes

**Terminology alignment:**
* AgentDefinitions = design-time specs stored in **Transformation Domain primitive** (modelled via Transformation design tooling UIs) (§8.1)
* Agent primitives = runtime that executes AgentDefinitions (§8.2)
* Note: There is no separate "Transformation design tooling service"—Transformation design tooling is the UI layer that calls Transformation APIs

### 12.4 SDK alignment (388)

* 476 §6 requires SDKs to "automatically attach tokens, tenant headers, retries, and tracing metadata"
* 476 §6 requires `AmeideClient` as unified client contract
* 388 defines SDK publish strategy for Go, Python, and TypeScript SDKs

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

### 13.2 Open architectural questions

1. **RLS vs application-level filtering**
   * §2.1, §5.1 require DB-level RLS as "non-negotiable"
   * Current architecture uses application-level tenant filtering
   * **Target**: Implement RLS per 329-authz Phase 2/3

2. **Agent primitive deterministic boundary enforcement**
   * §8.2 requires Domain primitives/Process primitives to invoke Agent primitives (never raw LLM APIs)
   * **Question**: How to audit/enforce deterministic boundary compliance?

3. **Secure-by-template dependency on Backstage**
   * §9.2 assumes Backstage templates for primitive generation
   * **Question**: What alternatives exist for achieving secure defaults?

4. **AgentDefinition scope/risk_tier vs tool grants**
   * 476 §8.1 describes `scope` and `risk_tier` as AgentDefinition-level properties
   * **Clarification needed**: How do scope/risk_tier interact with tool grants?

> **Implementation Status**: See [483-fit-gap-analysis.md](483-fit-gap-analysis.md) §7 for current security compliance status against this target architecture.
