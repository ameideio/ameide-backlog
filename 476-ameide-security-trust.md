Totally agree — we can’t keep evolving Ameide without locking down the security story in parallel.

Here’s a first **Security & Trust Architecture** pass that plugs into the 6-doc set you already have (Vision, Business, App/Info, Tech, Domains, Refactoring) and reuses as much of the existing backlog work as possible.

---

## 1. Purpose & Scope

This layer answers:

> “Given Ameide is a multi‑tenant, agentic, process‑driven business platform, how do we **stop tenants, agents, and components from hurting each other**?”

Scope:

* Cross‑cutting security principles and threat model
* How security attaches to:

  * Tenancy / identity / onboarding
  * Domains & ProcessControllers
  * Agents
  * Backstage + controller templates
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

   * **DomainControllers & ProcessControllers**: deterministic, auditable, replayable, no LLM calls inside.
   * **Agents**: non‑deterministic; they *propose* actions and edit artifacts, but must go through deterministic controllers to persist state.

3. **Proto‑first, zero trust between services**

   * Every RPC is proto‑based and goes through:

     * AuthN (JWT/mTLS)
     * AuthZ (role/tenant checks)
     * Input validation
   * No “trusted internal” bypasses; even internal calls use the same RPC contracts.

4. **Immutable artifacts, append‑only history**

   * UAF, BPMN, ArchiMate, Markdown, Backstage templates: always versioned, never overwritten.
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

  * Over‑privileged service accounts, missing rate limits, poisoning inputs into BPMN/UAF/Backstage.

### 3.2 Trust zones

Roughly:

1. **Public edge** (Next.js UIs, public docs, marketing)
2. **Tenant application plane** (Platform UI, Domain & Process APIs)
3. **Agent & Transformation plane** (Agents, UAF, Backstage, IPA legacy)
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

Security for **DomainControllers** is mostly about data isolation and integrity.

### 5.1 Data classification

* **Transactional business data**

  * Lives in domain DBs (Orders, Product, Pricing, etc.).
  * Multi‑tenant via RLS, plus optional “noisy” tenant isolation (separate DB / schema per plan).
* **Knowledge graph projections**

  * Copy‑on‑read from domains to Graph/Repository for analysis; still tagged by `tenant_id`.
* **Artifacts (UAF, BPMN, ArchiMate, Markdown)**

  * Event‑sourced, versioned, immutable; enriched with metadata and promotion state. 

Every DomainController must:

* Enforce **tenant + org + role checks** in its service methods (via shared interceptors).
* Use dedicated **DB roles/credentials** provided by CNPG + ExternalSecrets; no shared “superuser” in app code. 
* Project to Graph/UAF only via **explicit configuration** — no “random joins across tenants”.

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

---

## 7. Agents & AI Safety

Agents are where things can go very wrong, so the architecture must be explicit:

### 7.1 Agent domain constraints

* **Agent definitions** (graphs, tools, permissions) live in the Agent domain and are versioned.
* Each agent has:

  * `domains: [...]` and `processes: [...]` tags (for routing & UI only)
  * `scope`: read‑only vs read/write vs code‑gen
  * `risk_tier`: e.g. `low`, `medium`, `high`
* “Dangerous” capabilities (file system access, k8s access, direct SQL) are only exposed to **high‑tier, non‑default agents**, and always wrapped by deterministic tools.

### 7.2 Deterministic boundaries

* DomainControllers and ProcessControllers **never call raw LLM APIs**; they call Agents through the Agent domain, which:

  * Logs prompts & results
  * Enforces rate limits and timeouts
  * Applies basic content filters (PII, secret‑like patterns).

### 7.3 Agent‑generated code & configs

The `core-platform-coder` review already flags security concerns: CLI fragility, prompt size, and raw shell argument handling. 

We generalize:

* Any agent that writes code, Helm values, or Backstage configs must:

  * Propose changes as **artifacts in UAF or Git**, not apply them directly.
  * Pass through:

    * Static checks (lint, tests, security scanning)
    * Human or policy‑based approval
    * GitOps pipeline for deployment.
* The Agent domain logs “write intents” with full diff and user/tenant context.

---

## 8. Backstage, Controllers & GitOps

Backstage is now our internal factory — which means it’s also a high‑risk surface.

### 8.1 Backstage security

* Only **Ameide engineers + transformation agents** have access, via SSO and RBAC. No tenant users.
* Templates are treated as code:

  * Stored in Git
  * Reviewed, tested
  * Versioned and rolled out via Argo.

### 8.2 Secure‑by‑template controller generation

Each template (DomainController, ProcessController, Agent) bakes in:

* Proper `AmeideClient` usage with AuthN/AuthZ interceptors.
* Per‑service DB role & ExternalSecret wiring; no root credentials. 
* Default network policies (deny‑all except required dependencies).
* OTel tracing and structured logging with tenant and user correlation fields.
* Rate limits and timeouts based on proto budgets (e.g. p95 <100ms for simple operations). 

---

## 9. Observability, Audit & UAF

Security without observability is vibes. We already have strong foundations:

* **Onboarding**:

  * Structured audit events for invites, SSO, SCIM, subscription changes. 
* **UAF**:

  * Event‑sourced commands + snapshots, promotion flows, and secret scan validator. 
* **North‑Star modeling**:

  * Immutable designs and deployments with audit logs per revision. 

We extend:

* Every Domain/Process/Agent controller emits:

  * audit events with `tenant_id`, `user_id`, `resource_type`, `resource_id`.
* UAF secret scanner runs on artifacts (BPMN, ArchiMate, Markdown, Backstage templates) and **blocks promotion** if secrets are detected, with override via signed waiver. 

---

## 10. Immediate Next Steps (Security Backlog Slice)

Here’s how I’d start making this real without boiling the ocean:

1. **Codify security principles & zones**

   * Add a short “Security & Trust” section into the Vision + Tech docs, basically the principles from sections 2–3.

2. **Lock down DomainController base template**

   * In Backstage:

     * Inject `AmeideClient` with auth interceptors.
     * Enforce `tenant_id` in all queries (DB and Graph).
     * Wire in OTel tracing and error mapping by default. 

3. **Agent domain hardening v1**

   * Add `scope` + `risk_tier` to agent definitions.
   * Wrap any code‑gen / infra‑gen in UAF artifacts and GitOps flows, not direct writes.

4. **UAF secret‑scan + promotion gate**

   * Implement the MVP secret scanner + promotion endpoint described in UAF spec and wire it into Transformation workflows. 

5. **API security posture check**

   * Verify at least one “golden path” DomainController + ProcessController (e.g. L2O) uses:

     * proto‑based APIs
     * SDK interceptors
     * RLS + tenant claims end‑to‑end.

From there, we can expand into more formal things (STRIDE‑style threat modeling, formal policies, pen‑tests), but these five steps will already embed security into the Ameide story instead of bolting it on later.

If you want, next I can propose a very small **"Security RFC 000‑Ameide"** skeleton you can drop into the repo and iterate on with the team.

---

## 11. Cross‑References

This Security & Trust Architecture should be read with the following documents:

| Document | Relationship | Alignment |
|----------|-------------|-----------|
| [470‑ameide‑vision](470-ameide-vision.md) | Parent vision & principles | Strong ✅ |
| [471‑ameide‑business‑architecture](471-ameide-business-architecture.md) | Business use cases, personas | Strong ✅ |
| [472‑ameide‑information‑application](472-ameide-information-application.md) | Application & information architecture | Strong ✅ |
| [473‑ameide‑technology](473-ameide-technology.md) | Technology implementation | Strong ✅ |
| [475‑ameide‑domains](475-ameide-domains.md) | Domain portfolio & patterns | Strong ✅ |
| [322‑rbac](322-rbac.md) | RBAC implementation & testing | See §11.1 |
| [329‑authz](329-authz.md) | Authorization implementation | See §11.2 |
| [333‑realms](333-realms.md) | Realm‑per‑tenant model | Strong ✅ |
| [451‑secrets‑management](451-secrets-management.md) | Secrets authority model | Strong ✅ |
| [462‑secrets‑origin‑classification](462-secrets-origin-classification.md) | Secrets classification | Strong ✅ |
| [310‑agents‑v2](310-agents-v2.md) | Agent domain implementation | See §11.3 |
| [388‑ameide‑sdks‑north‑star](388-ameide-sdks-north-star.md) | SDK publish strategy | See §11.4 |

### 11.1 RBAC alignment (322)

* 476 §4.2 requires RBAC with canonical roles (`admin`, `contributor`, `viewer`, `guest`, `service`)
* 322 confirms: ✅ Canonical role catalog wired end-to-end (Keycloak, proto/SDK, database seeds)
* 322 confirms: ✅ Ownership enforcement consistent for transformations, repositories, teams
* 322 confirms: ✅ Realm-per-tenant architecture update in progress (moving from prefixed to realm-native roles)

**Gaps identified in 322 that impact 476 requirements:**
- ❌ API routes do NOT enforce 403 for insufficient permissions (476 §6 requires interceptors)
- ❌ Server actions have no authorization checks
- ❌ DAL with tenant scoping not implemented (476 §5.1 requires tenant + org + role checks)
- ❌ RLS policies not enabled (476 §2.1 requires RLS as "non-negotiable")
- ❌ Audit logging for denied attempts not implemented (476 §9 requires audit events)

**322 next activities align with 476:**
1. Land owner/role checks for org membership mutations → supports §5.1
2. Wire 401/403 telemetry into dashboards → supports §9
3. Fold API integration suites into CI gating → supports §6

### 11.2 Authorization alignment (329)

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

### 11.3 Agents alignment (310)

* 476 §7.1 requires agents to have `scope` (read-only vs read/write vs code-gen) and `risk_tier` (low, medium, high)
* 476 §7.2 requires deterministic boundaries (controllers never call raw LLM APIs)
* 476 §7.3 requires agent-generated code to propose via UAF/Git artifacts, not direct writes

**Current status per 310-agents-v2:**
- ✅ AgentInstance proto exists with basic metadata (node_id, version, parameters, tools)
- ❌ **Gap**: No `scope` field on AgentInstance message
- ❌ **Gap**: No `risk_tier` field on AgentInstance message
- ⚠️ **Gap**: `agents.tool_grants` schema exists but service does NOT expose grant/revoke APIs
- ❌ **Gap**: No enforcement of grants at publish time
- ❌ **Gap**: "Write intents" logging (§7.3) not implemented

**Impact on 476 compliance:**
- §7.1 (agent domain constraints): ❌ NOT MET - cannot differentiate safe agents from dangerous ones
- §7.3 (agent-generated code governance): ❌ NOT MET - awaiting UAF promotion flow

### 11.4 SDK alignment (388)

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

---

## 12. Notable Gaps & Issues

### 12.1 Terminology alignment

| 476 Term | Related Terms | Notes |
|----------|---------------|-------|
| Trust zones (§3.2) | N/A | 476-specific concept |
| Tenant application plane | Domain layer (475) | Consistent ✅ |
| Agent & Transformation plane | Transformation domain (475) | Consistent ✅ |
| Control & infra plane | Foundation layer (465) | Consistent ✅ |

### 12.2 Implementation gaps

| Gap | Description | Impact | Related Docs |
|-----|-------------|--------|--------------|
| **RLS not enabled** | §2.1, §4.1, §5.1 require DB-level RLS | Tenant isolation principle NOT MET | [329-authz](329-authz.md) ⚠️ CRITICAL |
| **Org-level scoping missing** | §5.1 requires tenant + org + role checks | Cross-org data leakage possible within tenant | [329-authz](329-authz.md) Phase 2 |
| **API routes lack 403 enforcement** | §6 requires AuthZ interceptors | Insufficient permission not rejected | [322-rbac](322-rbac.md) |
| **Agent scope/risk_tier** | §7.1 requires capability classification | No enforcement of dangerous capability restrictions | [310-agents-v2](310-agents-v2.md) |
| **Write intents logging** | §7.3 requires agent diff logging | No audit trail for agent-generated changes | — |
| **UAF secret scanner** | §9 describes secret scan + promotion gate | Not yet implemented | UAF spec |
| **TS SDK not published** | §6 requires AmeideClient | External TS consumers cannot use secure patterns | [388-ameide-sdks](388-ameide-sdks-north-star.md) |
| **Backstage not deployed** | §8.1 requires SSO + RBAC for Backstage | Cannot use secure-by-template pattern | [473](473-ameide-technology.md) §11.2 |
| **Audit logging** | §9 requires denied attempt logging | Cannot detect authorization attacks | [322-rbac](322-rbac.md), [329-authz](329-authz.md) |

### 12.3 Open architectural tensions

1. **RLS vs application-level filtering**
   * §2.1, §5.1 require DB-level RLS as "non-negotiable"
   * Current implementation uses application-level tenant filtering only (per 322-rbac)
   * **Resolution required**: Implement RLS per 329-authz Phase 2/3

2. **Agent deterministic boundary enforcement**
   * §7.2 requires agents to "never call raw LLM APIs" from controllers
   * Current implementation may have direct LLM calls in some services
   * **Audit needed**: Verify deterministic boundary compliance

3. **Secure-by-template dependency on Backstage**
   * §8.2 assumes Backstage templates for controller generation
   * Backstage not yet deployed per 473 §11.2
   * **Alternative needed**: Document how to achieve secure defaults without Backstage

4. **Agent scope/risk_tier vs tool grants**
   * 476 §7.1 describes `scope` and `risk_tier` as agent-level properties
   * 310 has `tool_grants` schema but no enforcement
   * **Clarification needed**: How do scope/risk_tier interact with tool grants?

### 12.4 Security compliance status

| Principle | Status | Notes |
|-----------|--------|-------|
| §2.1 Tenant isolation (RLS) | ❌ NOT MET | RLS not enabled; 329-authz Phase 2 pending |
| §2.2 Deterministic boundary | ⚠️ PARTIAL | Architecture defined; enforcement not audited |
| §2.3 Zero trust between services | ✅ MET | Proto-based APIs with auth interceptors |
| §2.4 Immutable artifacts | ⚠️ PARTIAL | UAF design exists; promotion gate not implemented |
| §2.5 Defence in depth | ⚠️ PARTIAL | Layers defined; not all owners assigned |
| §2.6 Secure-by-template | ❌ NOT MET | Backstage templates not created |
| §4.2 RBAC canonical roles | ✅ MET | Per 322-rbac, roles wired end-to-end |
| §6 SDK security contract | ⚠️ PARTIAL | Go/Python SDKs published; TS SDK missing |
| §7.1 Agent constraints | ❌ NOT MET | scope/risk_tier fields missing |
| §9 Audit events | ❌ NOT MET | Audit logging not implemented |

---

## 13. Recommended Priority Actions

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

3. **Agent domain hardening** (2 weeks)
   * Add `scope` and `risk_tier` to AgentInstance proto per §7.1
   * Implement tool grant enforcement at publish time
   * Wire "write intents" logging per §7.3

4. **Publish TypeScript SDK** (1 week)
   * Per 388 gaps: TS SDK not on npmjs
   * Publish `@ameideio/ameide-sdk-ts` with SemVer tags
   * Enables §6 compliance for external consumers

### Priority 3: MEDIUM (Completes security story)

5. **UAF secret scanner** (2 weeks)
   * Implement secret scan validator per §9
   * Wire into promotion workflow

6. **Audit logging service** (2 weeks)
   * Per 322-rbac and 329-authz gaps
   * Wire 401/403 telemetry into dashboards

7. **Backstage security baseline** (2 weeks)
   * Deploy Backstage with SSO + RBAC per §8.1
   * Create secure-by-default templates per §8.2
