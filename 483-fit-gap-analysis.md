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
| Base primitive definitions | ⏳ Not started | `_primitive/` base definitions pending |
| Backstage alignment | ✅ Complete | Templates live in `service_catalog/{domains,processes,agents}/_primitive/` |

### 6.6 Adding New Service (482)

| Area | Status | Notes |
|------|--------|-------|
| Checklist | ✅ Complete | Comprehensive checklist covering all cross-cutting concerns |
| Template scaffolding | ⏳ Not started | Backstage templates for service creation pending |
| Automation | ⏳ Not started | No automated service creation workflow yet |

---

## 7. Security Compliance Status (476)

### 7.1 RBAC Alignment (322)

**Target** (476 §4.2): RBAC with canonical roles (`admin`, `contributor`, `viewer`, `guest`, `service`)

| Requirement | Status | Notes |
|-------------|--------|-------|
| Canonical role catalog | ✅ MET | Keycloak, proto/SDK, database seeds wired |
| Ownership enforcement | ✅ MET | Consistent for transformations, repositories, teams |
| Realm-per-tenant | ⚠️ In progress | Moving from prefixed to realm-native roles |
| API route 403 enforcement | ❌ NOT MET | Routes do not enforce 403 for insufficient permissions |
| Server action auth checks | ❌ NOT MET | No authorization checks on server actions |
| DAL tenant scoping | ❌ NOT MET | No tenant + org + role checks in DAL |
| Audit logging (denied attempts) | ❌ NOT MET | Not implemented |

### 7.2 Authorization Alignment (329)

**Target** (476 §2.1): "DB-level RLS + JWT tenant claims are non-negotiable guardrails"

| Phase | Status | Notes |
|-------|--------|-------|
| Phase 1 (API Authorization Layer) | ✅ Complete | `withAuth` middleware applied to most routes |
| Phase 2 (Database Organization Scoping) | ❌ NOT STARTED | `organization_id` columns missing; data scoped by tenant only |
| Phase 3 (Defense in Depth) | ❌ NOT STARTED | RLS policies not created; audit logging not implemented |

**Required Actions**:
1. Create Flyway migrations for `organization_id` columns + FK constraints
2. Update proto definitions to include `organization_id`
3. Enable RLS policies with session variables
4. Implement audit logging service

### 7.3 Agent Governance Alignment (310)

**Target** (476 §8.1): AgentDefinitions have `scope` and `risk_tier`

| Requirement | Status | Notes |
|-------------|--------|-------|
| AgentInstance proto | ✅ Exists | Basic metadata (node_id, version, parameters, tools) |
| `scope` field | ❌ Missing | Cannot differentiate read-only vs read/write vs code-gen |
| `risk_tier` field | ❌ Missing | Cannot differentiate safe agents from dangerous ones |
| Tool grants enforcement | ⚠️ Schema only | `agents.tool_grants` schema exists; no grant/revoke APIs |
| Write intents logging | ❌ NOT MET | §8.3 logging not implemented |

### 7.4 Security Principle Compliance

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
| §8.1 AgentDefinition governance | ❌ NOT MET | scope/risk_tier fields missing |
| §10 Audit events | ❌ NOT MET | Audit logging not implemented |

---

## 8. Technology Gaps (473)

### 8.1 Infrastructure Gaps

| Gap | Description | Impact | Reference |
|-----|-------------|--------|-----------|
| Backstage templates | §4 assumes Backstage as primitive factory | Cannot use template-driven service creation | 477 |
| Transformation artifact storage | §2.6 assumes event-sourced ProcessDefinitions/AgentDefinitions | Implementation detail TBD | 471 §3.2 |
| Temporal namespace isolation | §3.2 mentions per-tenant namespaces | Current deployment is single namespace | 305 |
| AmeideTenant CRD | §3.1 suggests tenant CRD | 333 uses API-driven realm provisioning | 333 |
| ProcessDefinition→Temporal compiler | §3.2 assumes ProcessDefinition to Temporal translation | No compiler exists yet | 305, 471 |

### 8.2 Open Architectural Tensions

1. **Backstage vs GitOps static components**
   - 473 §4 envisions Backstage generating new primitives dynamically
   - 465 uses static `component.yaml` files discovered by ApplicationSets
   - **Resolution needed**: How do Backstage-generated services integrate with 465 GitOps model?

2. **AmeideTenant CRD vs API-driven**
   - 473 §3.1 suggests CRD-based tenant provisioning
   - 333 uses API-driven realm provisioning
   - **Clarification**: Both are valid; CRD is future target

---

## 9. Recent Progress (Dec 2025)

- **Tier 1 extensibility runtime shipped (480).** `extensions-runtime` now exists as a platform service in `ameide-{env}`, exposes `InvokeExtension` gRPC API, runs tenant WASM modules with Wasmtime sandboxing + MinIO-backed storage.

- **Unified ApplicationSet orchestration live (364).** GitOps repo drives namespaces, CRDs, operators, platform planes, and services via RollingSync ApplicationSets.

- **SDK + primitive alignment (388, 430).** Go/TS/Python SDKs consume the same `runtime.proto` and integration packs enforce env-var conventions.

---

## 10. Terminology Migration

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

## 11. Cross-References

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
