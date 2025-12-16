# 550 — Comparative Domain Primitives Analysis

**Status:** Analysis Complete
**Audience:** Architecture, platform engineering, operators/CLI, domain teams
**Scope:** Comparative analysis of domain primitive implementations across Sales, Commerce, Transformation, and SRE to identify maturity levels, common patterns, gaps, and recommended next steps.

**Use with:**
- Domain primitives: `backlog/540-sales-domain.md`, `backlog/523-commerce-domain.md`, `backlog/527-transformation-domain.md`, `backlog/526-sre-domain.md`
- EDA contract rules: `backlog/496-eda-principles.md`, `backlog/496-eda-protobuf-ameide.md`
- Proto conventions: `backlog/509-proto-naming-conventions.md`
- Primitives stack: `backlog/520-primitives-stack-v2.md`
- Capability definitions: `backlog/540-sales-capability.md`, `backlog/526-sre-capability.md`, `backlog/523-commerce.md`, `backlog/527-transformation-capability.md`

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

### Overall Maturity Ranking

1. **SRE Domain** - 98% Complete (Production-Ready)
2. **Transformation Domain** - 90% Complete (Production-Ready, Most Complex)
3. **Commerce Domain** - 75% Complete (Production-Ready, Focused Scope)
4. **Sales Domain** - 40% Complete (Scaffold/Template)

### Implementation Strategy Variance

**Two Distinct Approaches:**

1. **Top-Down (Sales, Transformation):**
   - Start with full proto contract + GitOps deployment
   - Add ecosystem primitives (Process, Integration, Projection, UI, Agent)
   - Defer handler implementation ("shape only")
   - Result: Rich API surface, minimal business logic

2. **Bottom-Up (Commerce, SRE):**
   - Start with production handler implementation
   - Focus on core domain logic first
   - No GitOps deployment initially
   - Result: Production code, limited deployment/ecosystem

## 2) Implementation Completeness Matrix

| Aspect | Sales | Commerce | Transformation | SRE |
|--------|-------|----------|----------------|-----|
| **Proto Contract** | 95% (14 write + 6 read RPCs) | 60% (8 write + 3 read RPCs) | 85% (10 combined RPCs) | 98% (8 write + 8 read RPCs) |
| **Handler Logic** | 0% (Scaffold only) | 100% (Production code) | 100% (Production code) | 100% (Production code) |
| **Database Schema** | 135 lines (2 migrations) | 76 lines (2 migrations) | 1,179 lines (6 migrations) | 80 lines (2 migrations) |
| **Aggregates** | 3 types (Leads, Opportunities, Quotes) | 2 types (Claims, Mappings) | 5+ types (Models, BPMN, Baselines, etc.) | 1 type (Incidents) |
| **GitOps Deployment** | ✓ Active | ✗ Missing | ✓ Active + Smoke Test | ✗ Missing |
| **Ecosystem Primitives** | 5 (Process, Integration, Projection, UI, Agent) | 0 | 1 (Smoke Test) | 0 |
| **Test Coverage** | Scaffold (RED tests) | Basic unit tests | Smoke tests | Unit tests |
| **Event Dispatcher** | Skeleton | Functional | Skeleton | Functional |
| **Documentation** | Detailed README with checklist | Simple README | Detailed README | Minimal README |

## 3) Domain-by-Domain Analysis

### 3.1 SRE Domain - Most Mature (98%)

**Status:** Production-ready, single-writer incident management system

**Strengths:**
- Most comprehensive proto contract: 7 separate query services (Incident, FleetHealth, Alert, Runbook, SLO, KnowledgeIndex)
- Complete event sourcing: Full message metadata with correlation/causation IDs, trace context
- Clean state machine: OPEN → ACKNOWLEDGED → INVESTIGATING → MITIGATING → RESOLVED → CLOSED
- Production handlers: 8 fully implemented RPC methods with validation, state transitions, timeline tracking
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
- No ecosystem primitives (Process, Integration, Projection, UI, Agent)
- Limited test coverage

**Key Patterns:**
- Single-writer constraint with version monotonicity
- Timeline entries as immutable audit trail
- Terminal state validation (cannot transition from RESOLVED/CLOSED)

### 3.2 Transformation Domain - Most Complex (90%)

**Status:** Production-ready, architecture repository with multiple capabilities

**Strengths:**
- Largest schema: 6 migrations totaling 1,179 lines (most comprehensive)
- Complex domain model: ArchiMate, BPMN, Baselines, Process Definitions, Agent Definitions
- Sophisticated handlers: 1,826 lines supporting 15+ intent types
- Version/revision tracking: head_version, published_version, version history snapshots
- Lifecycle management: DRAFT → IN_REVIEW → APPROVED → RETIRED
- Agent integration: LangGraph agent definitions with tools/permissions
- Active deployment: GitOps config + smoke test harness

**Implementation Details:**
- **Location:** `primitives/domain/transformation/`
- **Proto Files:** 16 files in `packages/ameide_core_proto/src/ameide_core_proto/transformation/`
- **Database:** 6 migrations - initial schema, agent defs, scrum, architecture repo, outbox metadata, aggregate versions
- **Handler:** `cmd/main.go` - 15+ intent handlers
- **GitOps:** `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0.yaml` + smoke test config

**Gaps:**
- No event sourcing metadata in proto contract (no message_id, correlation IDs)
- Read/write combined in single service (not strict CQRS)
- Database config missing in GitOps (spec.db not populated)
- Resource limits not defined
- Limited ecosystem primitives (only smoke test)

**Key Patterns:**
- Aggregate versioning with FOR UPDATE locks (optimistic concurrency)
- Baseline item references with kind mapping
- BPMN payload attachments
- Workspace hierarchy with graph elements
- Field mask support for partial updates

### 3.3 Commerce Domain - Focused Production (75%)

**Status:** Production-ready BYOD storefront hostname claim + mapping writer

**Strengths:**
- Production handlers: Fully implemented business logic (NOT scaffold)
- Comprehensive validation: Input validation, hostname normalization, error handling
- 17 specific error codes: DNS, certificate, rate-limiting scenarios (DomainOnboardingErrorCode)
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
- No ecosystem primitives
- Narrow scope compared to typical commerce platform

**Key Patterns:**
- JSONB columns for DNS instructions and error details
- Tenant-scoped schema operations (prevents cross-tenant leakage)
- Fact emission on state changes (CommerceDomainFact)
- Transactional outbox with metadata extraction

### 3.4 Sales Domain - Well-Designed Scaffold (40%)

**Status:** Shape-only scaffold; intentionally failing handlers awaiting AI agent implementation

**Strengths:**
- Excellent proto contract: 14 write RPCs + 6 read RPCs (most comprehensive command surface)
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
- **Handler:** `internal/handlers/handlers.go` - Intentional RED failures
- **GitOps:** `gitops/ameide-gitops/sources/values/_shared/apps/domain-sales-v0.yaml`
- **Ecosystem:** Agent (`gitops/ameide-gitops/sources/values/_shared/apps/agent-sales-copilot-v0.yaml`), Process, Integration, Projection, UI Surface

**Gaps:**
- **Zero handler implementation** (intentionally scaffolded for AI completion)
- Database config missing in GitOps (spec.db not populated)
- Resource limits not defined
- Integration tests skeleton only (run_integration_tests.sh exists)
- README states: "shape only, AI agent can focus on meaning"

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
- Domain inbox: `migrations/V1__domain_outbox.sql`

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
**Weakest Implementation:** Transformation (missing from proto contract)

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

**Status:**
- Commerce: Functional LogPublisher
- SRE: Functional LogPublisher
- Sales: Skeleton only
- Transformation: Skeleton only

## 5) Proto Contract Comparison

### 5.1 API Surface

| Domain | Write RPCs | Read RPCs | Query Services | State Machines | Pagination |
|--------|-----------|-----------|----------------|----------------|------------|
| **SRE** | 8 | 8 | 7 specialized | Yes (Incidents) | Yes |
| **Sales** | 14 | 6 | 1 | Yes (Opp/Quote) | Yes |
| **Transformation** | 10 (combined) | - | 0 (combined) | No | Yes |
| **Commerce** | 8 | 3 | 1 | Yes (Claims) | No |

### 5.2 Event Sourcing Maturity

| Domain | Message ID | Correlation ID | Causation ID | Trace Context | Idempotency |
|--------|-----------|----------------|--------------|---------------|-------------|
| **SRE** | ✓ | ✓ | ✓ | ✓ (traceparent/state) | ✓ |
| **Sales** | ✓ | ✓ | ✓ | ✓ (traceparent/state) | ✓ |
| **Commerce** | ✓ | ✓ | ✓ | ✓ (traceparent/state) | ✓ |
| **Transformation** | ✗ | ✗ | ✗ | ✗ | Partial |

### 5.3 Comparative Strengths

| Domain | Best Feature | Competitive Advantage |
|--------|-------------|----------------------|
| **SRE** | 7 specialized query services | Most mature CQRS/ES implementation |
| **Transformation** | 6 migrations, complex schema | Most sophisticated domain model |
| **Sales** | 14 write RPCs + full ecosystem | Best API design + deployment readiness |
| **Commerce** | 17 error codes, production logic | Most production-ready business logic |

## 6) GitOps Deployment Status

### 6.1 Deployed Domains

**Sales:**
- File: `gitops/ameide-gitops/sources/values/_shared/apps/domain-sales-v0.yaml`
- Image: `ghcr.io/ameideio/sales-domain:dev`
- Replicas: 1
- Config: Minimal (image + log level only)
- Gaps: No database config, no resource limits

**Transformation:**
- File: `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0.yaml`
- Image: `ghcr.io/ameideio/transformation-domain:dev`
- Replicas: 1
- Smoke Test: `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0-smoke.yaml` with gRPC health check
- Config: Minimal (image + log level only)
- Gaps: No database config, no resource limits

### 6.2 Not Deployed (Code Only)

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

1. **GitOps Database Config:** No domain uses `spec.db` (CNPG cluster ref, schema, migration job)
2. **Resource Limits:** No domains define CPU/memory requests/limits
3. **Security Context:** No `serviceAccountName` or `networkProfile` specified
4. **Environment Variables:** No custom env vars configured
5. **Observability:** Only `logLevel` configured, no metrics/traces/alerts
6. **Dispatcher Integration:** All have skeleton dispatchers, none wired to Watermill/Kafka

## 8) Recommended Next Steps

### 8.1 For Sales Domain (Highest ROI)
1. **Implement handlers:** Convert scaffold to production logic (follow Commerce/SRE patterns)
2. **Add database config:** Populate `spec.db` in GitOps with CNPG cluster ref
3. **Wire dispatcher:** Connect to Kafka via Watermill publisher
4. **Add integration tests:** Implement test harness (mock + cluster modes)
5. **Resource tuning:** Define CPU/memory based on load testing

### 8.2 For Commerce & SRE Domains
1. **Create GitOps configs:** Add deployment manifests
2. **Define ecosystem:** Add Process/Integration/Projection primitives if needed
3. **Add database config:** Wire CNPG integration
4. **Expand test coverage:** Add integration and E2E tests

### 8.3 For Transformation Domain
1. **Add event metadata:** Enhance proto contract with message_id, correlation_id, causation_id
2. **Separate CQRS services:** Split combined service into command + query
3. **Add database config:** Wire CNPG integration
4. **Wire dispatcher:** Connect to Kafka

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
- **Proto:** `packages/ameide_core_proto/src/ameide_core_proto/transformation/v1/transformation_service.proto`
- **Handler:** `primitives/domain/transformation/cmd/main.go`
- **GitOps:** `gitops/ameide-gitops/sources/values/_shared/apps/domain-transformation-v0.yaml`
- **Migrations:** `primitives/domain/transformation/migrations/`

### SRE Domain
- **Proto:** `packages/ameide_core_proto/src/ameide_core_proto/sre/core/v1/sre_service.proto`
- **Handler:** `primitives/domain/sre/internal/handlers/handlers.go`
- **Migrations:** `primitives/domain/sre/migrations/`

## 10) Conclusion

The four domain primitives demonstrate varying levels of maturity:
- **SRE** leads in CQRS/ES implementation quality with complete event sourcing
- **Transformation** has the most complex domain model with sophisticated versioning
- **Sales** has the best API design and ecosystem readiness but needs handler implementation
- **Commerce** has the most production-ready business logic but narrow functional scope

All follow consistent architectural patterns (transactional outbox, port/adapter, event sourcing), but vary in deployment readiness and feature completeness. The primary gap across all domains is GitOps database configuration and resource governance.

### Key Insight: Strategic Convergence Needed

The analysis reveals a need for convergence:
- **Bring bottom-up domains (Commerce, SRE) to GitOps deployment** with full ecosystem primitives
- **Complete top-down domains (Sales) with production handler implementation**
- **Standardize all domains** on CNPG database configuration, resource limits, security contexts
- **Activate event dispatchers** across all domains for true event-driven operation
