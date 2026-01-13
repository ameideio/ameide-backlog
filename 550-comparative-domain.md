# 550 — Comparative Domain Primitives Analysis

**Status:** Reviewed 2025-12-17 (revalidated vs current repo state)
**Audience:** Architecture, platform engineering, operators/CLI, domain teams
**Scope:** Comparative analysis of domain primitive implementations across Sales, Commerce, Transformation, and SRE to identify maturity levels, common patterns, gaps, and recommended next steps.

**Use with:**
- Domain primitives: `backlog/540-sales-domain.md`, `backlog/523-commerce-domain.md`, `backlog/527-transformation-domain.md`, `backlog/526-sre-domain.md`
- EDA contract rules: `backlog/496-eda-principles-v2.md`, `backlog/496-eda-protobuf-ameide.md`
- Proto conventions: `backlog/509-proto-naming-conventions.md`
- Primitives stack: `backlog/520-primitives-stack-v2.md`
- Capability definitions: `backlog/540-sales-capability.md`, `backlog/526-sre-capability.md`, `backlog/523-commerce.md`, `backlog/527-transformation-capability.md`

> **Update (2026-01): 430v2 test contract**
>
> This comparative doc references legacy “integration harness” scripts (`run_integration_tests.sh`) as an implementation detail in some scaffolds. Treat `backlog/430-unified-test-infrastructure-v2-target.md` as the normative contract (native tooling; strict phases; no pack scripts as the canonical path).

## Verification Notes (2025-12-17)

- Recomputed proto and migration counts/LOC from `packages/ameide_core_proto/` and `primitives/domain/*/migrations/`.
- Verified RPC coverage by matching proto `rpc` names to `func (h *Handler) <RPC>(...)` for each domain’s **registered** gRPC services.
- Confirmed tests pass: `go test ./primitives/domain/{sales,commerce,transformation,sre}/...`.

## Layer header (Application Architecture + Technology)

- **Primary ArchiMate layer(s):** Application (Application Component, Service, Interface, Event) + Technology (Implementation artifacts, deployment)
- **Primary element types used:** Application Component, Application Service, Data Object, Technology Artifact
- **Prohibited unless qualified:** Generic use of "system" or "platform" without primitive type qualification

## 0) Problem

Four domain primitives (Sales, Commerce, Transformation, SRE) exist at varying levels of maturity with different implementation approaches. We lack a consolidated view of:

- Comparative implementation completeness (proto contracts, handlers, database schemas, GitOps)
- Architectural pattern consistency (transactional outbox, event sourcing metadata, CQRS separation)
- Common gaps across all domains (database config, resource limits, security, observability)
- Strategic recommendations for bringing all domains to production readiness

## 1) Executive Summary

### Maturity Signals (Current Snapshot)

- **GitOps manifests present for domain primitives:** Sales, Transformation
- **Code-only (no GitOps manifest):** Commerce, SRE
- **Largest domain schema (by SQL LOC):** Transformation (2 migrations, 209 LOC SQL)
- **Most complete “primitive ecosystem” coverage:** Sales and Commerce (Domain + Integration + Process + Projection + UISurface + Agent present; Commerce UISurface has 3 variants)

### Implementation Strategy Variance

**Two Distinct Axes (Observed):**

1. **Deployment-first vs code-first**
   - **Deployment-first:** Sales, Transformation (GitOps manifests exist)
   - **Code-first:** Commerce, SRE (no GitOps manifests yet)

2. **CQRS separation**
   - **Strict CQRS binaries:** Sales, Commerce, Transformation (domain write; projection read)
   - **Mixed read+write in domain binary:** SRE (IncidentService includes reads)

## 2) Implementation Completeness Matrix

| Aspect | Sales | Commerce | Transformation | SRE |
|--------|-------|----------|----------------|-----|
| **Domain RPCs served** | 18 (SalesCommandService) | 8 (CommerceStorefrontWriteService) | 15 (TransformationKnowledgeCommandService) | 10 (IncidentService) |
| **Handler coverage** | All 18 RPCs implemented | All 8 RPCs implemented | All registered RPCs implemented | All 10 RPCs implemented |
| **Database Schema** | 135 LOC SQL (2 migrations) | 76 LOC SQL (2 migrations) | 209 LOC SQL (2 migrations) | 80 LOC SQL (2 migrations) |
| **Aggregates** | 3 types (Leads, Opportunities, Quotes) | 2 types (Claims, Mappings) | 5 types (Repositories, Nodes, Elements, Relationships, Assignments) | 1 type (Incidents) |
| **GitOps Deployment** | ✓ Manifests present | ✗ Missing | ✓ Manifests present | ✗ Missing |
| **Ecosystem Primitives** | Integration + Process + Projection + UISurface + Agent | Integration + Process + Projection + 3× UISurface + Agent | Process + Projection + UISurface + Agent (+ MCP adapter integration) | Integration + Process + Projection + Agent (no UISurface) |
| **Tests (go test)** | PASS | PASS | PASS | PASS |
| **Event Dispatcher** | Log-only publisher (no broker wiring) | Log-only publisher (no broker wiring) | Log-only publisher (no broker wiring) | Log-only publisher (no broker wiring) |
| **Documentation** | 58-line README | 18-line README | 58-line README | 4-line README |

## 3) Domain-by-Domain Analysis

### 3.1 SRE Domain - Incident Management (Code-only)

**Status:** Implemented incident management domain (IncidentService) with separate projection/query layer; not GitOps deployed

**Strengths:**
- Rich proto surface includes IncidentService plus 6 projection query services (IncidentQuery, FleetHealth, Alert, Runbook, SLO, KnowledgeIndex)
- Complete event sourcing: Full message metadata with correlation/causation IDs, trace context
- Clean state machine: OPEN → ACKNOWLEDGED → INVESTIGATING → MITIGATING → RESOLVED → CLOSED
- IncidentService implements 10 RPCs (including Get/List reads) with validation, state transitions, and timeline tracking
- Rich domain model: Severity levels, timeline entries, postmortem workflow, SLO tracking
- Transactional outbox: Idempotent writes with domain inbox deduplication

**Implementation Details:**
- **Location:** `primitives/domain/sre/`
- **Proto Files:** 13 files in `packages/ameide_core_proto/src/ameide_core_proto/sre/`
- **Database:** 2 migrations (80 lines) - `incidents` + `incident_timeline_entries` + `domain_inbox`
- **Handler:** `internal/handlers/handlers.go` - Full incident lifecycle
- **Topic:** `sre.domain.facts.v1`

**Gaps:**
- No GitOps deployment config (code exists but not deployed)
- No UISurface primitive
- Dispatcher is log-only (not wired to Kafka/Watermill)

**Key Patterns:**
- Single-writer constraint with version monotonicity
- Timeline entries as immutable audit trail
- Terminal state validation (cannot transition from RESOLVED/CLOSED)

### 3.2 Transformation Domain - Knowledge Repository (GitOps manifests present)

**Status:** Implemented knowledge repository write domain (TransformationKnowledgeCommandService); reads are served by the Transformation projection

**Strengths:**
- Clean write boundary: 15 explicit write RPCs (repositories, nodes, elements, relationships, assignments)
- Transactional inbox/outbox pattern: idempotent processing via `domain_inbox` + facts written to `domain_outbox`
- Enterprise repository schema: repositories + tree nodes + elements + relationships + assignments (element-centric canonical store)
- Emits strongly typed facts: `TransformationKnowledgeDomainFact` (topic `transformation.knowledge.domain.facts.v1`)

**Implementation Details:**
- **Location:** `primitives/domain/transformation/`
- **Proto Files:** 12 files in `packages/ameide_core_proto/src/ameide_core_proto/transformation/` (domain registers Knowledge command only)
- **Database:** 2 migrations (209 lines) - `V1__domain_outbox.sql`, `V2__enterprise_repository.sql`
- **Handler:** `internal/handlers/handlers.go`
- **GitOps:** `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0.yaml` (+ smoke values file present)

**Gaps:**
- No domain query service registered in the domain binary (reads are projection responsibility)
- Database config missing in GitOps (spec.db not populated)
- Resource limits not defined
- Dispatcher is log-only (not wired to Kafka/Watermill)

**Key Patterns:**
- FOR UPDATE locking + monotonic `version` increments for optimistic concurrency
- Node hierarchy stored with `path` and `order` for deterministic tree queries
- FieldMask support for partial updates in edit operations

### 3.3 Commerce Domain - Storefront Domains (Code-only)

**Status:** Production-ready BYOD storefront hostname claim + mapping writer

**Strengths:**
- Production handlers: Fully implemented business logic (NOT scaffold)
- Comprehensive validation: Input validation, hostname normalization, error handling
- 10 defined error codes (including UNSPECIFIED): DNS/cert/rate-limit/onboarding scenarios (DomainOnboardingErrorCode)
- Complete state machines: HostnameClaimStatus + HostnameMappingStatus
- Event dispatcher: Functional LogPublisher ready for Watermill integration
- Idempotency: Domain inbox with ON CONFLICT deduplication
- Helper methods: Hostname classification (APEX, SUBDOMAIN, WILDCARD)

**Implementation Details:**
- **Location:** `primitives/domain/commerce/`
- **Proto Files:** 9 files in `packages/ameide_core_proto/src/ameide_core_proto/commerce/`
- **Database:** 2 migrations (76 lines) - `storefront_hostname_claims` + `storefront_hostname_mappings`
- **Handler:** `internal/handlers/handlers.go` - 8 RPC methods
- **Topic:** `commerce.domain.facts.v1`

**Gaps:**
- No GitOps deployment config (not actively deployed)
- Limited to hostname management (lacks customer, product, cart, order, payment domains)
- No pagination in read operations
- Dispatcher is log-only (not wired to Kafka/Watermill)
- Narrow scope compared to typical commerce platform

**Key Patterns:**
- JSONB columns for DNS instructions and error details
- Tenant-scoped schema operations (prevents cross-tenant leakage)
- Fact emission on state changes (CommerceDomainFact)
- Transactional outbox with metadata extraction

### 3.4 Sales Domain - CQRS Command Service (GitOps manifests present)

**Status:** Implemented SalesCommandService (18 RPCs) with full primitive ecosystem; GitOps deployed

**Strengths:**
- Strong CQRS contract: 18 write RPCs (SalesCommandService) + 6 read RPCs (SalesQueryService)
- Complete state machines: LeadStatus, OpportunityStage (6 stages), QuoteStatus (8 states)
- Approval workflow: SubmitQuoteForApproval → ApproveQuote / RejectQuote flow
- Rich read models: Pipeline summary, approval work items, filtered lists
- Strong typing: Money type with ISO 4217 currency, ActorRef, AccountRef, ContactRef
- Full ecosystem: Process, Integration, Projection, UI Surface, Agent (5 primitives)
- Active deployment: GitOps config with event dispatcher skeleton
- Development guide: Detailed README with implementation checklist

**Implementation Details:**
- **Location:** `primitives/domain/sales/`
- **Proto Files:** 9 files in `packages/ameide_core_proto/src/ameide_core_proto/sales/`
- **Database:** 2 migrations (135 lines) - `leads`, `opportunities`, `quotes`, `quote_line_items` with indexes
- **Handler:** `internal/handlers/handlers.go` - SalesCommandService RPCs implemented with inbox/outbox patterns
- **GitOps:** `gitops/ameide-gitops/sources/values/_shared/apps/domain-sales-v0.yaml`
- **Ecosystem:** Agent (`gitops/ameide-gitops/sources/values/_shared/apps/agent-sales-copilot-v0.yaml`), Process, Integration, Projection, UI Surface

**Gaps:**
- Database config missing in GitOps (spec.db not populated)
- Resource limits not defined
- Dispatcher is log-only (not wired to Kafka/Watermill)

**Key Patterns (Ready to Implement):**
- Port/adapter outbox interface
- SDK clients placeholder for outbound calls
- Test harness mode detection (mock vs cluster)
- Transactional outbox V1 template

## 4) Architectural Patterns (Common Across All)

### 4.1 Transactional Outbox Pattern

All domains implement identical outbox flow:
1. Begin TX
2. Insert `domain_inbox` (deduplication on `tenant_id` + `message_id`)
3. If duplicate, return cached idempotent response
4. Execute business logic (INSERT/UPDATE aggregates)
5. Create domain fact
6. Insert to `domain_outbox` with metadata
7. Commit TX

**Files:**
- Outbox port: `internal/ports/outbox.go`
- Postgres adapter: `internal/adapters/postgres/outbox.go`
- Domain inbox/outbox schema: `primitives/domain/sales/migrations/V1__domain_outbox.sql`, `primitives/domain/commerce/migrations/V1__domain_outbox.sql`, `primitives/domain/sre/migrations/V1__domain_outbox.sql`, `primitives/domain/transformation/migrations/V4__architecture_repository.sql`

### 4.2 Event Metadata Structure

All events include:
- `message_id` (UUID, idempotency key)
- `tenant_id` (multi-tenancy isolation)
- `correlation_id` (request tracing)
- `causation_id` (event causality chain)
- `occurred_at` (event timestamp)
- `producer` (source service identifier)
- `schema_version` (version compatibility)
- `traceparent` / `tracestate` (W3C Trace Context)

**Best Implementation:** SRE (SreMessageMeta with all fields)
**Also present in Transformation:** TransformationMessageMeta and ScrumMessageMeta include `message_id`, `correlation_id`, `causation_id`, and trace context

### 4.3 Standard File Structure

All domains follow:
```
primitives/domain/{name}/
├── cmd/
│   ├── main.go                    # gRPC server entry point
│   └── dispatcher/
│       └── main.go                # Event dispatcher worker
├── internal/
│   ├── handlers/
│   │   └── handlers.go            # Business logic (command handlers)
│   ├── adapters/
│   │   ├── postgres/
│   │   │   └── outbox.go          # Persistence adapter
│   │   └── sdk/
│   │       └── clients.go         # Outbound SDK calls
│   └── ports/
│       └── outbox.go              # Outbox interface
├── migrations/
│   ├── V1__domain_outbox.sql      # Outbox + inbox tables
│   └── V2__*.sql                  # Domain-specific schema
├── tests/
│   └── run_integration_tests.sh   # Integration test harness
├── Dockerfile                     # Two-stage distroless build
├── catalog-info.yaml              # Backstage metadata
├── README.md                      # Development guide
├── go.mod
└── go.sum
```

### 4.4 Dispatcher Pattern

All domains have skeleton dispatcher:
- Read from `domain_outbox` WHERE `published_at IS NULL`
- Publish to event stream (Watermill/Kafka)
- Mark as published (`UPDATE domain_outbox SET published_at = NOW()`)

**Status:** All four domain dispatchers currently use a log-only publisher (no broker wiring)

## 5) Proto Contract Comparison

### 5.1 API Surface

| Domain | Domain (write) RPCs | Read/query RPCs (elsewhere) | Query Services | State Machines | Pagination |
|--------|-----------|-----------|----------------|----------------|------------|
| **SRE** | 10 (IncidentService, mixed read+write) | 7 (projection query RPCs) | 6 query services | Yes (Incidents) | Yes |
| **Sales** | 18 (SalesCommandService) | 6 (SalesQueryService via projection) | 1 | Yes (Opp/Quote) | Yes |
| **Transformation** | 15 (TransformationKnowledgeCommandService) | 15 (TransformationKnowledgeQueryService via projection) | 1 | Yes (versioned resources) | Yes |
| **Commerce** | 8 (CommerceStorefrontWriteService) | 3 (CommerceQueryService via projection) | 1 | Yes (Claims/Mappings) | No |

### 5.2 Event Sourcing Maturity

| Domain | Message ID | Correlation ID | Causation ID | Trace Context | Idempotency |
|--------|-----------|----------------|--------------|---------------|-------------|
| **SRE** | ✓ | ✓ | ✓ | ✓ (traceparent/state) | ✓ |
| **Sales** | ✓ | ✓ | ✓ | ✓ (traceparent/state) | ✓ |
| **Commerce** | ✓ | ✓ | ✓ | ✓ (traceparent/state) | ✓ |
| **Transformation** | ✓ | ✓ | ✓ | ✓ (traceparent/state) | ✓ |

### 5.3 Comparative Strengths

| Domain | Best Feature | Competitive Advantage |
|--------|-------------|----------------------|
| **SRE** | 6 specialized query services | Most mature CQRS/ES implementation |
| **Transformation** | Enterprise repository primitives | Rich knowledge repo model with inbox/outbox + versioned resources |
| **Sales** | 18 write RPCs + full ecosystem | Best deployment + ecosystem coverage |
| **Commerce** | 10 error codes, production logic | Most production-ready business logic |

## 6) GitOps Manifest Status

### 6.1 Manifests Present

**Sales:**
- File: `gitops/ameide-gitops/sources/values/_shared/apps/domain-sales-v0.yaml`
- Image: `ghcr.io/ameideio/sales-domain@sha256:<digest>`
- Replicas: 1
- Config: Minimal (image + log level only)
- Gaps: No database config, no resource limits

**Transformation:**
- File: `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0.yaml`
- Image: `ghcr.io/ameideio/transformation-domain@sha256:<digest>`
- Replicas: 1
- Smoke Test values: `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0-smoke.yaml` (gRPC health check harness)
- Config: Minimal (image + log level only)
- Gaps: No database config, no resource limits

### 6.2 Missing Manifests (Code Only)

- **Commerce:** Implementation exists, no GitOps config
- **SRE:** Implementation exists, no GitOps config

### 6.3 Domain CRD Capabilities (Available but Unused)

The `ameide.io/v1` Domain CRD supports:
```yaml
spec:
  db:                                    # ✗ Not used in any deployment
    clusterRef: <cnpg-cluster-ref>
    schema: <schema-name>
    migrationJobImage: <flyway-image>
  resources:                             # ✗ Not used in any deployment
    requests: { cpu, memory }
    limits: { cpu, memory }
  security:                              # ✗ Not used in any deployment
    serviceAccountName: <sa-name>
    networkProfile: <network-policy-ref>
  env:                                   # ✗ Not used in any deployment
```

All current deployments use only: `image`, `replicas`, `strategy`, `logLevel`

## 7) Common Gaps (All Domains)

1. **GitOps Database Config:** Deployed domains (Sales/Transformation) do not use `spec.db` (CNPG cluster ref, schema, migration job)
2. **Resource Limits:** No domains define CPU/memory requests/limits
3. **Security Context:** No `serviceAccountName` or `networkProfile` specified
4. **Environment Variables:** No custom env vars configured
5. **Observability:** Only `logLevel` configured, no metrics/traces/alerts
6. **Dispatcher Integration:** All dispatchers are log-only (none wired to Watermill/Kafka)

## 8) Recommended Next Steps

### 8.1 For Sales Domain (Highest ROI)
1. **Harden invariants:** Add targeted tests around critical command behaviors (idempotency, state transitions, invariants)
2. **Add database config:** Populate `spec.db` in GitOps with CNPG cluster ref
3. **Wire dispatcher:** Connect to Kafka via Watermill publisher
4. **Add integration tests:** Implement test harness (mock + cluster modes)
5. **Resource tuning:** Define CPU/memory based on load testing

### 8.2 For Commerce & SRE Domains
1. **Create GitOps configs:** Add deployment manifests
2. **Wire database config:** Use `spec.db` and migrations in-cluster (CNPG + migration job)
3. **Decide UISurface posture:** SRE has no UISurface; Commerce has 3 variants but no GitOps deployment
4. **Expand test coverage:** Add integration and E2E tests

### 8.3 For Transformation Domain
1. **Lock in CQRS boundary:** Keep domain as write-only; serve reads from the projection (avoid duplicated query implementations)
2. **Close the projection loop:** Ensure all knowledge query RPCs compile/are implemented in projection handlers and exercised by tests
3. **Add database config:** Wire CNPG integration in GitOps `spec.db` (schema + migration job)
4. **Wire dispatcher/relay:** Decide the canonical outbox consumption path (Kafka/Watermill vs brokerless relay) and make it consistent

### 8.4 For All Domains
1. **Database operator integration:** Implement CNPG cluster ref + Flyway migration jobs
2. **Resource governance:** Define requests/limits for cost optimization
3. **Security hardening:** Add service accounts, network policies, RBAC
4. **Observability:** Add Prometheus metrics, OpenTelemetry traces, alerting rules
5. **Dispatcher activation:** Wire all dispatchers to production event streams

## 9) Critical Files Reference

### Sales Domain
- **Proto:** `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/sales_command_service.proto`
- **Handler:** `primitives/domain/sales/internal/handlers/handlers.go`
- **GitOps:** `gitops/ameide-gitops/sources/values/_shared/apps/domain-sales-v0.yaml`
- **Migrations:** `primitives/domain/sales/migrations/`

### Commerce Domain
- **Proto:** `packages/ameide_core_proto/src/ameide_core_proto/commerce/v1/commerce_write_service.proto`
- **Handler:** `primitives/domain/commerce/internal/handlers/handlers.go`
- **Migrations:** `primitives/domain/commerce/migrations/`

### Transformation Domain
- **Proto:** `packages/ameide_core_proto/src/ameide_core_proto/transformation/knowledge/v1/transformation_knowledge_command_service.proto`
- **Handler:** `primitives/domain/transformation/internal/handlers/handlers.go`
- **GitOps:** `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0.yaml`
- **Migrations:** `primitives/domain/transformation/migrations/`

### SRE Domain
- **Proto:** `packages/ameide_core_proto/src/ameide_core_proto/sre/core/v1/query.proto`
- **Handler:** `primitives/domain/sre/internal/handlers/handlers.go`
- **Migrations:** `primitives/domain/sre/migrations/`

## 10) Conclusion

The four domain primitives demonstrate varying levels of maturity:
- **SRE** leads in CQRS/ES implementation quality with complete event sourcing
- **Transformation** has the richest knowledge repository patterns (tree + element graph + assignments) with inbox/outbox + versioning
- **Sales** has strong CQRS design, GitOps deployment, and full primitive ecosystem coverage
- **Commerce** has the most production-ready business logic but narrow functional scope

All follow consistent architectural patterns (transactional outbox, port/adapter, event sourcing), but vary in deployment readiness and feature completeness. The primary gap across all domains is GitOps database configuration and resource governance.

### Key Insight: Strategic Convergence Needed

The analysis reveals a need for convergence:
- **Bring code-only domains (Commerce, SRE) to GitOps deployment**, and decide on missing/variant primitives (e.g., SRE UISurface; Commerce’s 3 UISurface variants)
- **Standardize all domains** on CNPG database configuration, resource limits, security contexts
- **Activate event dispatchers** across all domains for true event-driven operation
