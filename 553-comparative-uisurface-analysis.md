# 553 — Comparative UISurface Implementation Analysis

**Status:** Reviewed 2025-12-17 (accurate snapshot of current UISurface scaffolds + GitOps state)
**Created:** 2025-12-16
**Purpose:** Comparative review of UISurface primitive implementations across Sales, Commerce, Transformation, and SRE domains

## Layer header (Application: cross-cutting analysis)

- **Primary ArchiMate layer(s):** Application (UISurface primitives are Application Components).
- **Primary element types used:** Application Component (UISurface primitives), Application Services (consumed by UISurfaces), Application Events (EDA integration).
- **Analysis scope:** Code implementation, GitOps deployment, proto/API contracts, test coverage, documentation quality.

---

## Executive Summary

This analysis reviews the implementation status of UISurface primitives across four domains: **Sales**, **Commerce** (3 variants), **Transformation**, and **SRE**. The review covers code implementation, GitOps deployment configuration, proto/API contract maturity, and backlog documentation.

**Key Findings:**
- **Sales**: Most complete UISurface (60-70% ready) - scaffolded with comprehensive development guidance, functional tests, GitOps manifests present
- **Commerce**: Three variants (admin/pos/storefront) at 20-30% - scaffolded but identical implementations, not yet deployed to GitOps
- **Transformation**: 25-40% ready - functional placeholder with embedded documentation, not yet deployed to GitOps
- **SRE**: 0% - No UISurface primitive exists despite complete Domain/Process/Projection/Integration/Agent layers

---

## Implementation Progress Ranking

### By Overall Completeness (Code + GitOps + Tests + Docs)

1. **Sales: 60-70%** ⭐ Reference Implementation
   - ✅ Comprehensive README with development guidance
   - ✅ Functional integration tests
   - ✅ GitOps manifests present (component + values)
   - ✅ Buf generation configured
   - ✅ Backlog documentation
   - ⚠️ CQRS wiring to Domain/Projection pending
   - ⚠️ Real UI workflow not implemented

2. **Transformation: 25-40%**
   - ✅ Functional server (self-contained)
   - ✅ Backlog documentation clear
   - ✅ Proto services defined
   - ❌ GitOps not deployed
   - ❌ No buf generation config
   - ❌ Trivial test coverage
   - ⚠️ Portal UX (initiative dashboard, workspace browser, editors) not implemented

3. **Commerce (all 3 variants): 20-30%**
   - ✅ Primitives scaffolded
   - ✅ Excellent backlog documentation (topology matrix, context model)
   - ✅ Proto services with `uisurface_ref` integration
   - ❌ GitOps not deployed (none of 3 variants)
   - ❌ No buf generation configs
   - ❌ Minimal test coverage (file existence only)
   - ❌ Identical implementations (no differentiation)
   - ⚠️ Real UX, auth/session model, hostname resolution not implemented

4. **SRE: 0%**
   - ❌ No UISurface primitive exists
   - ❌ Architectural decision pending
   - ✅ All other primitive layers complete (Domain, Process, Projection, Integration, Agent)
   - ✅ Most comprehensive proto services (7 services)
   - ✅ Excellent capability documentation (value streams, business processes)
   - 🔍 Highest effort required (richest service surface + architectural decision needed)

---

## Detailed Comparative Matrix

### 1. Code Implementation Status

| Domain | Primitive Location | Status | Lines of Code | Key Features |
|--------|-------------------|--------|---------------|--------------|
| **Sales** | `primitives/uisurface/sales/` | Scaffolded | server.js: 34 lines<br>README: 1,016 bytes<br>tests: 1.8K functional | - Generated HTML reference<br>- Comprehensive README with dev checklist<br>- Functional integration tests<br>- Backstage catalog-info.yaml<br>- Clear scaffold philosophy |
| **Commerce-Admin** | `primitives/uisurface/commerce-admin/` | Placeholder | server.js: 36 lines<br>README: 69 bytes<br>tests: 363 bytes | - Generated HTML reference<br>- File existence tests only<br>- Identical to POS/Storefront |
| **Commerce-POS** | `primitives/uisurface/commerce-pos/` | Placeholder | server.js: 36 lines<br>README: 83 bytes<br>tests: 363 bytes | - Generated HTML reference<br>- File existence tests only<br>- Identical to Admin/Storefront |
| **Commerce-Storefront** | `primitives/uisurface/commerce-storefront/` | Placeholder | server.js: 36 lines<br>README: 72 bytes<br>tests: 363 bytes | - Generated HTML reference<br>- File existence tests only<br>- Identical to Admin/POS |
| **Transformation** | `primitives/uisurface/transformation/` | Functional Placeholder | server.js: 47 lines<br>README: 215 bytes<br>tests: 121 bytes | - Embedded HTML (self-contained)<br>- Architecture docs in HTML<br>- Trivial smoke test (1==1) |
| **SRE** | N/A | **Not Implemented** | N/A | - No primitive exists<br>- Architectural decision pending |

### 2. Test Coverage Analysis

| Domain | Test Type | Test Quality | Assertions |
|--------|-----------|--------------|------------|
| **Sales** | Integration | **High** - Functional testing | - Server startup validation<br>- HTTP response testing<br>- Fallback behavior verification<br>- Port discovery patterns |
| **Commerce (all 3)** | Artifact Verification | **Low** - File existence only | - Checks `internal/gen/index.generated.html` exists<br>- Basic HTML validation<br>- No functional testing |
| **Transformation** | Smoke Test | **Minimal** - Trivial assertion | - Single `1==1` assertion<br>- No server behavior testing |
| **SRE** | N/A | **None** | N/A |

### 3. GitOps Deployment Status

| Domain | Component Definition | Values Files | Deployed | Backlog Doc | Status |
|--------|---------------------|--------------|----------|-------------|---------|
| **Sales** | ✅ `environments/_shared/components/apps/primitives/uisurface-sales-v0/` | ✅ `sources/values/_shared/apps/uisurface-sales-v0.yaml` | ✅ Yes | `540-sales-uisurface.md` (40 lines) | **DEPLOYED** |
| **Commerce-Admin** | ❌ Not created | ❌ Not created | ❌ No | `523-commerce-uisurface.md` (150 lines) | Scaffolded, not deployed |
| **Commerce-POS** | ❌ Not created | ❌ Not created | ❌ No | Same as above | Scaffolded, not deployed |
| **Commerce-Storefront** | ❌ Not created | ❌ Not created | ❌ No | Same as above | Scaffolded, not deployed |
| **Transformation** | ❌ Not created | ❌ Not created | ❌ No | `527-transformation-uisurface.md` (65 lines) | Scaffolded, not deployed |
| **SRE** | ❌ Not created | ❌ Not created | ❌ No | `526-sre-capability.md` (lines 25-30) | Architectural decision pending |

**GitOps Configuration Pattern (Sales - Reference Implementation):**
```yaml
# uisurface-sales-v0.yaml structure
apiVersion: ameide.io/v1
kind: UISurface
metadata:
  name: sales-v0
  labels:
    ameide.io/primitive: uisurface
    ameide.io/uisurface: sales
spec:
  image: ghcr.io/ameideio/uisurface-sales:dev
  routing:
    host: uisurface-sales-v0.local
  rollout:
    replicas: 1
    strategy: RollingUpdate
```

### 4. Proto/API Contract Maturity

| Domain | Proto Services Defined | Buf Gen Config | Generation Target | API Maturity |
|--------|----------------------|----------------|-------------------|--------------|
| **Sales** | ✅ 2 services:<br>- SalesQueryService<br>- SalesCommandService | ✅ `buf.gen.uisurface-sales.local.yaml` | `primitives/uisurface/sales/internal/gen/` | **FAIR** - Services defined, UI scaffolded, CQRS wiring pending |
| **Commerce** | ✅ 2 services:<br>- CommerceQueryService<br>- CommerceStorefrontWriteService<br>*(includes `uisurface_ref` field)* | ❌ None | N/A | **EARLY** - Hostname routing designed, no generation, no UI implementation |
| **Transformation** | ✅ Knowledge CQRS services:<br>- TransformationKnowledgeCommandService<br>- TransformationKnowledgeQueryService | ❌ None | N/A | **EARLY** - No UISurface-specific proto; UI placeholder only |
| **SRE** | ✅ 7 services:<br>- IncidentService<br>- IncidentQueryService<br>- FleetHealthQueryService<br>- AlertQueryService<br>- RunbookQueryService<br>- SLOQueryService<br>- KnowledgeIndexQueryService | ❌ None | N/A | **RICH SERVICES, NO UI** - Most comprehensive service definitions, no UISurface primitive |

**Proto Generation Pattern (Current):**
```yaml
# buf.gen.uisurface-sales.local.yaml
plugins:
  - local: protoc-gen-ameide-uisurface-static
    out: ../../primitives/uisurface/sales/internal/gen
    opt:
      - output_file=index.generated.html
      - title=Hello UISurface v0
      - body=Hello UISurface v0
```
*Note: Currently generates only static marker HTML, not functional RPC bindings or client code.*

### 5. Backlog Documentation Quality

| Domain | Backlog File | Status Line | Implementation Checklist | Clarification Requests | Quality |
|--------|-------------|-------------|-------------------------|----------------------|---------|
| **Sales** | `540-sales-uisurface.md` | "Implemented (scaffolded server; CQRS wiring ready)" | 4 items (1 complete) | 3 questions | **Good** - Clear next steps |
| **Commerce** | `523-commerce-uisurface.md` | "Scaffolded (minimal Node placeholders + tests; real UX pending)" | 8 items (2 complete) | 3 decisions needed | **Excellent** - Comprehensive scope definition with topology matrix, context model, edge behavior |
| **Transformation** | `527-transformation-uisurface.md` | "Draft (placeholder UISurface implemented; portal UX pending)" | 2 items (1 complete) | 3 decisions needed | **Good** - Clear UX scope inventory |
| **SRE** | `526-sre-capability.md` | "In progress (proto + primitives implemented; doc still draft)" | 6 items (3 complete) | 5 decisions needed | **Excellent** - Comprehensive capability definition; explicitly calls out missing UISurface implementation |

---

## Architecture & Implementation Patterns

### Sales (Reference Pattern)
**Philosophy:** "Provides shape only so an AI agent can focus on meaning"
- Scaffold explicitly fails until business logic implemented
- RED→GREEN→REFACTOR test discipline
- Development checklist with verification commands
- `ameide primitive verify --kind uisurface --name sales --mode all --json`

**Structure:**
```
primitives/uisurface/sales/
├── README.md (1,016 bytes - comprehensive)
├── package.json (158 bytes)
├── server.js (34 lines - thin HTTP server)
├── catalog-info.yaml (Backstage integration)
├── internal/gen/
│   └── index.generated.html (14 lines - buf generated)
└── tests/
    ├── run_integration_tests.sh
    └── server.test.js (1.8K - functional assertions)
```

**Scaffolding Command Evidence:**
```bash
ameide primitive scaffold \
  --kind uisurface \
  --name sales \
  --include-gitops \
  --include-test-harness
```

### Commerce (3 Identical Variants)
**Philosophy:** Implicit placeholders with minimal documentation
- All three variants (admin/pos/storefront) have identical code
- No differentiation in functionality or tests
- Expect generated HTML, provide fallback text
- File existence tests instead of functional tests

**Gap:** No distinguishing features between admin/POS/storefront implementations despite different operational contexts:
- **Admin:** Should handle domain onboarding, replication health
- **POS:** Should handle edge/offline mode, degraded UX
- **Storefront:** Should handle hostname resolution, headless API

### Transformation (Self-Contained Approach)
**Philosophy:** Functional placeholder with embedded documentation
- No dependency on generation pipeline
- HTML hardcoded in server.js with architecture references
- Links to sibling primitives (Domain, Projection, Process, Integration)
- Runnable immediately without build step

**Advantage:** Demos work without generation infrastructure

### SRE (Not Yet Implemented)
**Decision Pending:** "Decide the canonical SRE UI surface: new `primitives/uisurface/sre-operations-console` vs extending an existing UI surface"

**Context:**
- All other primitive layers exist (Domain, Process, Projection, Integration, Agent)
- Most comprehensive service definitions (7 query/command services)
- Backlog documents operational value streams and a UISurface primitive section, but no UISurface implementation
- Architectural choice required before implementation

---

## Key Gaps & Blockers

### Sales
1. **CQRS Wiring:** UI actions not connected to SalesCommandService/SalesQueryService
2. **Auth Integration:** No tenant selection, user identity, or role-aware affordances
3. **Real UI Workflow:** Static scaffolding only, no interactive flows implemented

### Commerce (All 3 Variants)
1. **Differentiation:** Admin/POS/Storefront have identical code - no unique functionality
2. **GitOps Deployment:** Component definitions and values files don't exist yet
3. **Hostname Resolution:** Storefront host resolution (`ResolveHostname`) not integrated
4. **Auth/Session Model:** No current context selector (tenant/site/channel/location)
5. **Real UX:** BYOD domain onboarding, health dashboards, POS degraded-mode UI not implemented
6. **Test Coverage:** File existence checks instead of functional/integration tests

### Transformation
1. **GitOps Deployment:** Component definitions and values files don't exist yet
2. **Buf Generation:** No `buf.gen.uisurface-transformation.yaml` config
3. **Portal UX:** Initiative dashboard, workspace browser, modeling editors not implemented
4. **Test Coverage:** Trivial smoke test (1==1) instead of functional tests
5. **Human-in-Loop Approvals:** Review gates, evidence display, audit timeline UX pending

### SRE
1. **Architectural Decision:** Choose between new `sre-operations-console` vs extending existing UI
2. **UISurface Primitive:** Needs scaffolding (highest implementation effort)
3. **Buf Generation:** Would need `buf.gen.uisurface-sre.yaml` config
4. **UI Scope Definition:** Minimum screens for operations console undefined
5. **End-to-End Slice:** Need v1 scope definition (backlog reference: lines 29-34 in `526-sre-capability.md`)

---

## Patterns & Observations

### Generation Pipeline Dependency
- **Sales & Commerce:** Depend on `internal/gen/index.generated.html` from buf generation
- **Transformation:** Self-contained with embedded HTML (no generation dependency)
- **Pattern:** Sales/Commerce provide fallback when generation unavailable
- **Implication:** CI/CD must handle generation failures gracefully

### Test Strategy Variance
| Pattern | Used By | Characteristics |
|---------|---------|-----------------|
| Integration Testing | Sales | Server startup, HTTP behavior, fallback validation |
| Artifact Verification | Commerce (all) | File existence, basic HTML structure check |
| Trivial Smoke | Transformation | Passes without testing behavior (1==1) |

**Recommendation:** Standardize on Sales integration testing pattern

### Documentation Maturity
- **Sales:** Development-focused (checklist, verification commands, philosophy)
- **Commerce:** Architecture-focused (topology matrix, context model, capability matrix)
- **Transformation:** UX scope inventory (portal screens, governance consoles)
- **SRE:** Capability-focused (value streams, business processes, ArchiMate realization)

---

## Recommendations

### Immediate Actions

1. **SRE Architectural Decision:**
   - Resolve: "new `primitives/uisurface/sre-operations-console` vs extending existing UI"
   - Document minimum v1 UX scope (fleet health, incident list, runbook browser)
   - Create backlog sub-document: `526-sre-uisurface.md`

2. **Commerce Differentiation:**
   - Define unique functionality per variant:
     - **Admin:** Domain onboarding flows, replication health dashboards
     - **POS:** Edge/offline mode capability matrix, degraded UX
     - **Storefront:** Hostname resolution, headless API patterns
   - Update tests to verify variant-specific behavior

3. **Test Coverage Standardization:**
   - Adopt Sales integration testing pattern for all domains
   - Replace Commerce file-existence tests with functional server tests
   - Replace Transformation trivial smoke test with HTTP behavior tests

4. **GitOps Deployment Preparation:**
   - Create component definitions for commerce-admin, commerce-pos, commerce-storefront
   - Create component definition for transformation
   - Create values files following Sales pattern
   - Test ApplicationSet generation in dev environment

### Phase-Based Implementation

**Phase 1: Foundation (Complete Scaffolding)**
- [ ] SRE: Make architectural decision on UI surface approach
- [ ] SRE: Scaffold `primitives/uisurface/sre/` (or chosen variant)
- [ ] Commerce/Transformation: Create `buf.gen.uisurface-*.yaml` configs
- [ ] All: Standardize test coverage to Sales pattern

**Phase 2: GitOps Integration**
- [ ] Commerce (all 3): Create GitOps component definitions + values files
- [ ] Transformation: Create GitOps component definition + values file
- [ ] SRE: Create GitOps component definition + values file (once primitive exists)
- [ ] All: Deploy to dev environment and validate ArgoCD sync

**Phase 3: CQRS Wiring**
- [ ] Sales: Connect UI actions to SalesCommandService/SalesQueryService
- [ ] Commerce: Implement hostname resolution integration (Storefront)
- [ ] Transformation: Connect workspace browser to TransformationKnowledgeQueryService (projection) + TransformationKnowledgeCommandService (domain)
- [ ] SRE: Connect operations console to IncidentQueryService/FleetHealthQueryService

**Phase 4: Real UX Implementation**
- [ ] Sales: Implement minimal read-only UI (opportunities, quotes, approvals)
- [ ] Commerce: Implement BYOD domain onboarding (Admin), edge-mode UX (POS), host resolution (Storefront)
- [ ] Transformation: Implement initiative dashboard + workspace browser
- [ ] SRE: Implement fleet health dashboard + incident management UI

---

## Summary Table: Implementation Progress

| Metric | Sales | Commerce | Transformation | SRE |
|--------|-------|----------|----------------|-----|
| **Code Exists** | ✅ Yes | ✅ Yes (3 variants) | ✅ Yes | ❌ No |
| **GitOps Manifests** | ✅ Present | ❌ No | ❌ No | ❌ No |
| **Tests Quality** | ✅ Functional | ⚠️ File existence | ⚠️ Trivial | ❌ None |
| **Buf Gen Config** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **Proto Services** | ✅ 2 services | ✅ 2 services | ✅ Knowledge CQRS services | ✅ 7 services |
| **Backlog Docs** | ✅ Good | ✅ Excellent | ✅ Good | ✅ Excellent |
| **Real UX** | ❌ No | ❌ No | ❌ No | ❌ No |
| **CQRS Wiring** | ⚠️ Pending | ❌ No | ❌ No | ❌ No |
| **Overall Progress** | **60-70%** | **20-30%** | **25-40%** | **0%** |

*SRE backlog is excellent for capability definition and explicitly calls out the missing UISurface primitive

---

## Critical Files Reference

### Sales (Reference Implementation)
- Primitive: `primitives/uisurface/sales/`
- README: `primitives/uisurface/sales/README.md`
- Tests: `primitives/uisurface/sales/tests/server.test.js`
- GitOps Component: `gitops/ameide-gitops/environments/_shared/components/apps/primitives/uisurface-sales-v0/`
- GitOps Values: `gitops/ameide-gitops/sources/values/_shared/apps/uisurface-sales-v0.yaml`
- Backlog: `gitops/ameide-gitops/backlog/540-sales-uisurface.md`
- Buf Gen: `packages/ameide_core_proto/buf.gen.uisurface-sales.local.yaml`
- Proto: `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/sales_query_service.proto`

### Commerce
- Primitives:
  - `primitives/uisurface/commerce-admin/`
  - `primitives/uisurface/commerce-pos/`
  - `primitives/uisurface/commerce-storefront/`
- Backlog: `gitops/ameide-gitops/backlog/523-commerce-uisurface.md`
- Proto: `packages/ameide_core_proto/src/ameide_core_proto/commerce/v1/commerce_write_service.proto`

### Transformation
- Primitive: `primitives/uisurface/transformation/`
- Backlog: `gitops/ameide-gitops/backlog/527-transformation-uisurface.md`
- Proto: `packages/ameide_core_proto/src/ameide_core_proto/transformation/v1/transformation_service.proto`

### SRE
- Primitives (other layers):
  - `primitives/domain/sre/`
  - `primitives/process/sre/`
  - `primitives/projection/sre/`
  - `primitives/integration/sre/`
  - `primitives/agent/sre/`
- Backlog: `gitops/ameide-gitops/backlog/526-sre-capability.md`
- Proto: `packages/ameide_core_proto/src/ameide_core_proto/sre/core/v1/query.proto`

---

## Next Steps Decision Points

Before proceeding with implementation work, the following decisions are needed:

### SRE
1. UI surface architecture: new `sre-operations-console` vs extend existing?
2. v1 minimum UX scope: which screens are required?
3. Auth/session model for operations console
4. End-to-end slice definition for v1 (backlog reference: lines 29-34 in `526-sre-capability.md`)

### Commerce
1. Differentiation strategy for admin/POS/storefront implementations
2. Auth/session model and context selection UX (tenant/site/channel/location)
3. GitOps deployment priority order (which variant first?)
4. Whether storefront is truly headless (API-only) or also renders content in v1

### Transformation
1. v1 minimum UX slice (which portal screens are required to prove acceptance?)
2. Auth/session posture for portal (tenant selection, roles, approval authorization)
3. How UI renders reproducible reads (which views show read_context + citations)

### Sales
1. Target UI scope for v0 (seller pipeline only vs quoting/approval queue vs forecasting)
2. Canonical auth mechanism for UISurfaces (OIDC, JWT, service mesh)
3. Audit timeline and diff view UX requirements for governance

---

## Related Backlogs

- `520-primitives-stack-v2.md` - Primitive stack architecture and patterns
- `523-commerce-uisurface.md` - Commerce UISurface specification
- `526-sre-capability.md` - SRE capability definition (includes UISurface gap)
- `527-transformation-uisurface.md` - Transformation UISurface specification
- `540-sales-uisurface.md` - Sales UISurface specification

---

*This comparative analysis provides a comprehensive cross-domain view of UISurface implementation maturity and serves as the foundation for prioritizing implementation work across all four domains.*
