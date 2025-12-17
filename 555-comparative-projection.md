# 555 Comparative Projection Primitive Analysis

**Status:** Reviewed 2025-12-17 (analysis complete; maturity table metrics corrected)
**Type:** Cross-cutting Architecture Analysis
**Scope:** Sales, Commerce, Transformation, SRE projection primitives

## Executive Summary

Comparative analysis of projection primitive implementation maturity across four domains reveals **convergence toward the Sales CQRS pattern**. The most significant development is **Transformation projection's architectural shift from a proxy facade to a full PostgreSQL-backed CQRS read model** with event consumption, placing it alongside Sales as a production-ready implementation.

## Maturity Overview (Updated)

| Domain | Overall Score | Storage | Read Models | Query RPCs | Event Consumption | Tests | GitOps |
|--------|--------------|---------|-------------|------------|-------------------|-------|--------|
| **Sales** | ★★★★★ (5/5) | PostgreSQL | 5 tables | 6/6 (100%) | ✅ Inbox pattern | ✅ 284 LOC | ✅ Yes |
| **Transformation** | ★★★★ (4/5) | PostgreSQL | 6 tables | 7/11 (63.6%) | ✅ Inbox pattern | ✅ 89 LOC | ❌ No |
| **Commerce** | ★★★ (3/5) | PostgreSQL | 2 tables | 3/3 (100%) | ❌ None | ⚠️ 6 LOC stub | ❌ No |
| **SRE** | ★½ (1.5/5) | None | N/A | 1/7 (14.3%) | N/A | ⚠️ 67 LOC | ❌ No |

## Architectural Patterns

### Pattern 1: Traditional CQRS Read Model (Sales, Transformation)
**Event Sourcing → Denormalized PostgreSQL → Query Service**

Event consumption with idempotency guarantees drives materialized read models optimized for query patterns. **Transformation now follows this pattern** with full PostgreSQL-backed read models.

### Pattern 2: Domain-Specific Read Model (Commerce)
**Direct State → PostgreSQL → Query Service**

Specialized projection for hostname resolution and domain management, unclear event consumption.

### Pattern 3: Partial Domain Forwarding (SRE)
**Query Service → Domain Service (with filtering)**

Mostly stub implementation with only SearchIncidents functional (in-memory filtering of domain data).

---

## Domain-Specific Findings

### 1. Sales Projection ⭐ **GOLD STANDARD**

**Location:** `/workspaces/ameide-core/primitives/projection/sales`

**Architecture:**
- **Entry Points:** Dual executable pattern
  - `cmd/main.go` - gRPC query service (port 50051)
  - `cmd/ingress/main.go` - Event consumer (stdin-based)
- **Storage:** PostgreSQL with embedded migrations
- **Event Model:** Two-stream consumption (domain facts + process facts)

**Strengths:**

#### Database Implementation (Most Mature)
- **Migration System:** Custom embedded runner with version tracking
  - File: `migrations/V1__sales_projection.sql` (79 LOC)
  - Automatic schema creation with parameterization
  - Transaction-based with SERIALIZABLE isolation
  - Idempotent via `IF NOT EXISTS` clauses
  - Version tracking in `schema_migrations` table

- **Read Models:** 5 tables optimized for query patterns
  1. `projection_inbox` - Idempotency tracking (tenant_id, message_id)
  2. `opportunity_views` - Denormalized opportunity snapshots
     - Fields: stage, name, owner, expected_amount (Money type), close_date
     - Indexes: `(tenant_id, stage)`, `(tenant_id, owner_user_id)`
  3. `quote_views` - Denormalized quote snapshots
     - Fields: status, opportunity_id, total (Money), expires_date
     - Indexes: `(tenant_id, status)`, `(tenant_id, opportunity_id)`
  4. `quote_line_items` - Nested line item data
     - Fields: sku, quantity, unit_price, total_price (currency triples)
     - Foreign key: `(tenant_id, quote_id, line_no)`
  5. `quote_approval_work_items` - Approval workflow tracking
     - Fields: approval_policy_ref, assignee, submitted_at
     - Index: `(tenant_id, submitted_at DESC)` for FIFO ordering

- **Schema Strategy:**
  - Multi-tenant via `tenant_id` in every table (composite primary keys)
  - Denormalized data matching query patterns (no joins)
  - Wide, flat schema for performance
  - Money type stored as currency triples (currency, units, nanos)

#### Event Consumption (Best Practice)
- **Idempotency:** Inbox pattern with `message_id` tracking
  - `ON CONFLICT DO NOTHING` for exactly-once processing
  - Transaction spans inbox insert + read model mutations

- **Event Types Consumed:**
  - **Domain Facts:** OpportunityCreated, StageChanged, AmountAdjusted, ClosedWon/Lost, QuoteCreated, QuoteSubmitted, QuoteApproved/Rejected, QuoteSent, QuoteAccepted/Declined
  - **Process Facts:** QuoteApprovalRequested, QuoteApprovalTimedOut

- **Application Logic:**
  ```
  1. Extract tenant_id and message_id from event
  2. Begin transaction (READ COMMITTED)
  3. Insert to inbox (idempotent)
  4. If already processed, commit and return
  5. Switch on event type and apply mutations
  6. Commit transaction
  ```

#### Query API (6 RPCs - All Implemented)
1. **GetOpportunityView** - Single opportunity with financial details
2. **ListOpportunities** - Paginated with filters (stage, owner), cursor-based tokens
3. **GetQuoteView** - Single quote with nested line items
4. **ListQuotes** - Paginated with filters (status, opportunity_id)
5. **GetPipelineSummary** - Aggregated metrics (count, total amount by stage/currency)
6. **ListApprovalWorkItems** - Work item queue ordered by FIFO

**Pagination Strategy:**
- Cursor-based with ID pagination for opportunities/quotes
- Timestamp+ID for approval items (FIFO guarantee)
- Default page size: 50, max: 200
- Encoded as base64 JSON

**Test Coverage (Excellent):**
- **308 LOC of test code** for ~1,100 LOC implementation (28% ratio)
- **Unit Tests:** 8 test functions with fake store mocks
  - TestListOpportunities, TestGetPipelineSummary, TestListApprovalWorkItems
  - TestGetOpportunityView, TestListQuotes, TestGetQuoteView
- **Integration Tests:** 3 test functions with sqlmock
  - TestGetOpportunityView_NotFound (error handling)
  - TestListOpportunities_PaginatesByID (pagination logic)
  - TestGetQuoteView_ReturnsLineItems (complex multi-query assertions)
- **Test Infrastructure:**
  - Fake store implementing `Store` interface
  - `testContext()` helper for RequestContext generation
  - `RequireIntegrationMode()` for mock/cluster mode support
  - Integration test runner: `run_integration_tests.sh`

**Code Quality:**
- Clean separation: handlers → store adapter
- Interface-based design enabling mocks
- Raw SQL with prepared statements (no ORM)
- Schema name validated via regex: `^[a-zA-Z_][a-zA-Z0-9_]*$`

**Gaps:**
- None identified - production-ready

**Critical Files:**
- Proto: `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/sales_query_service.proto`
- Migration: `primitives/projection/sales/internal/adapters/postgres/migrations/V1__sales_projection.sql`
- Store: `primitives/projection/sales/internal/adapters/postgres/store.go` (959 LOC)
- Handlers: `primitives/projection/sales/internal/handlers/handlers.go` (118 LOC)
- Event Consumer: `primitives/projection/sales/cmd/ingress/main.go`
- GitOps: `gitops/ameide-gitops/sources/values/_shared/apps/projection-sales-v0.yaml`

---

### 2. Commerce Projection

**Location:** `/workspaces/ameide-core/primitives/projection/commerce`

**Architecture:**
- **Entry Point:** `cmd/main.go` only (no separate event consumer)
- **Storage:** PostgreSQL (no embedded migrations)
- **Specialization:** Hostname resolution and domain management

**Strengths:**

#### Database Implementation
- **Tables:** 2 specialized tables
  1. `storefront_hostname_mappings` - Active hostname → commerce scope mappings
     - Scope: tenant, site, channel, stock_location, store_site
     - Meta: uisurface_ref, status (active/inactive)
     - Error tracking: `last_error` (JSON-serialized protobuf)
  2. `storefront_hostname_claims` - Domain verification claims
     - Same scope fields
     - Verification: hostname_kind, status, instructions (JSON)
     - Error tracking: `last_error` (JSON-serialized protobuf)

- **Schema Strategy:**
  - Multi-tenant via `tenant_id`
  - Protobuf JSON serialization for complex nested structures
  - Schema name validated via regex (same pattern as Sales)

- **Connection:** Direct DB access in handlers (no adapter layer)
  - Connection pool: 10 max connections, 10 idle

#### Query API (3 RPCs - All Implemented)
1. **ResolveHostname** - Public lookup (no tenant scoping)
   - Returns CommerceScope + status
   - Gates access via ACTIVE status check
2. **GetHostnameClaim** - Get domain verification claim for onboarding
   - Requires tenant_id in scope
   - Returns claim with verification instructions (protobuf JSON)
3. **GetHostnameMapping** - Get active hostname mapping with error tracking
   - Requires tenant_id in scope
   - Returns mapping with error info (protobuf JSON)

**Protobuf Serialization Pattern:**
```
SQL query returns JSON column → Unmarshal via protojson → Protobuf message
Examples: DomainVerificationInstructions, DomainOnboardingError
```

**Security:**
- Advanced gRPC TLS configuration (optional)
- Client certificate authentication support
- Server-side certificate handling via env vars

**Test Coverage (Minimal):**
- **6 LOC** - Empty smoke test stub only
- No unit tests for schema validation, hostname normalization, DB error handling
- No sqlmock tests
- Integration test runner exists but runs no tests

**Code Quality:**
- Monolithic handler with embedded DB logic (219 LOC)
- No separation of concerns (handlers directly execute SQL)
- Direct SQL queries with parameterization (SQL injection safe)

**Gaps:**
- ❌ No test coverage (0% meaningful)
- ❌ No migration system (schema managed externally)
- ❌ No GitOps deployment manifest
- ❌ Event consumption unclear (how do mappings get populated?)
- ⚠️ No adapter layer (handlers mix business logic + SQL)

**Critical Files:**
- Proto: `packages/ameide_core_proto/src/ameide_core_proto/commerce/v1/commerce_query.proto`
- Handlers: `primitives/projection/commerce/internal/handlers/handlers.go` (219 LOC)
- Tests: `primitives/projection/commerce/tests/smoke_test.go` (empty)

---

### 3. Transformation Projection ⚡ **MAJOR ARCHITECTURAL SHIFT**

**Location:** `/workspaces/ameide-core/primitives/projection/transformation`

**Architecture (Updated 2025-12-17):**
- **Entry Points:** Dual executable pattern (now matches Sales)
  - `cmd/main.go` - gRPC query service (port 50051) with PostgreSQL
  - `cmd/ingress/main.go` - Event consumer (stdin-based)
- **Storage:** PostgreSQL with embedded migrations
- **Pattern:** Full CQRS read model (migrated from proxy)

**Previous Architecture (Obsolete):**
```
gRPC Request → Handler → Domain Client → Response (pass-through)
```

**New Architecture (Current):**
```
Domain Events → Ingress Handler → Inbox Dedup → PostgreSQL Read Models → Query Service
```

**Strengths:**

#### Database Implementation (Production-Ready)
- **Migration System:** Custom embedded runner with version tracking
  - File: `migrations/V1__transformation_projection.sql` (100 LOC)
  - Automatic schema creation with parameterization
  - Transaction-based with SERIALIZABLE isolation
  - Schema templating: `{{.Schema}}` placeholder support

- **Read Models:** 6 tables for ArchiMate repository
  1. `projection_inbox` - Idempotency tracking `(tenant_id, message_id)`
  2. `archimate_model_views` - Model metadata (name, description)
  3. `archimate_element_views` - Elements with JSONB properties, version tracking
  4. `archimate_relationship_views` - Source/target element relationships
  5. `archimate_view_views` - View definitions with head/published revision
  6. `archimate_view_revision_views` - Full revision snapshots as JSONB

- **Scoping:** All tables include `organization_id` and `repository_id`
  - Multi-tenant isolation: `(tenant_id, organization_id, repository_id, entity_id)` PKs
  - Strategic indexes for pagination: `(tenant, org, repo, created_at, id)`

#### Event Consumption (Matches Sales Pattern)
- **Idempotency:** Inbox pattern with `message_id` tracking
  - `ON CONFLICT DO NOTHING` for exactly-once processing
  - Transaction spans inbox insert + read model mutations

- **Event Types Supported (13 total):**
  | Category | Events |
  |----------|--------|
  | **Create** | ArchimateModelCreated, ArchimateElementCreated, ArchimateRelationshipCreated, ArchimateViewCreated, ArchimateViewRevisionRecorded |
  | **Update** | ArchimateModelUpdated, ArchimateElementUpdated, ArchimateRelationshipUpdated, ArchimateViewUpdated |
  | **Delete** | ArchimateModelDeleted, ArchimateElementDeleted, ArchimateRelationshipDeleted, ArchimateViewDeleted |

- **Application Logic:**
  ```
  1. Extract tenant_id and message_id from fact.meta
  2. Begin transaction
  3. Insert to inbox (idempotent - ON CONFLICT DO NOTHING)
  4. If rows_affected == 0, commit and return (already processed)
  5. Switch on event type and apply mutations (UPSERT/DELETE)
  6. Commit transaction
  ```

**Query API (7 RPCs - All Implemented):**
1. **GetArchimateModel** - Single model retrieval
2. **ListArchimateModels** - Paginated listing with cursor tokens (NEW)
3. **ListArchimateElements** - Elements by model
4. **ListArchimateRelationships** - Relationships by model
5. **GetArchimateView** - Single view retrieval
6. **ListArchimateViews** - Paginated listing with cursor tokens (NEW)
7. **GetArchimateViewRevision** - Specific revision snapshot

**Pagination Strategy:**
- Cursor-based with `(created_at, id)` keyset pagination
- Default page size: 50, max: 200
- Encoded as base64 JSON tokens

**Test Coverage (Good):**
- **84 LOC** test code in `tests/archimate_projection_apply_test.go`
- Tests idempotent event application using sqlmock
- Validates inbox deduplication prevents re-processing
- Verifies transactional consistency

**Code Quality:**
- Clean separation: handlers → store adapter (matches Sales pattern)
- Interface-based design enabling mocks
- Raw SQL with prepared statements (no ORM)
- Schema name validated via regex: `^[a-zA-Z_][a-zA-Z0-9_]*$`

**Gaps:**
- ❌ No GitOps deployment manifest
- ⚠️ Test coverage below Sales (84 LOC vs 308 LOC)
- ⚠️ Missing integration test runner script

**Critical Files:**
- Proto: `packages/ameide_core_proto/src/ameide_core_proto/transformation/architecture/v1/transformation_architecture_query.proto`
- Migration: `primitives/projection/transformation/internal/adapters/postgres/migrations/V1__transformation_projection.sql`
- Store: `primitives/projection/transformation/internal/adapters/postgres/store.go` (447 LOC)
- Event Handler: `primitives/projection/transformation/internal/adapters/postgres/apply_architecture_fact.go` (500+ LOC)
- Handlers: `primitives/projection/transformation/internal/handlers/handlers.go` (48 LOC)
- Event Consumer: `primitives/projection/transformation/cmd/ingress/main.go`
- Tests: `primitives/projection/transformation/tests/archimate_projection_apply_test.go` (84 LOC)

---

### 4. SRE Projection

**Location:** `/workspaces/ameide-core/primitives/projection/sre`

**Architecture:**
- **Entry Point:** `cmd/main.go` only
- **Storage:** None - mostly stub implementations
- **Pattern:** Partial domain forwarding with stubs

**Strengths:**

#### Security (Best in Class)
- **Mandatory TLS:** Requires TLS certs OR explicit `GRPC_INSECURE=true` flag
- Server credentials function enforces security policy
- Optional mTLS client authentication
- Domain connection supports TLS with CA file

#### Partial Implementation
- **Implemented:** SearchIncidents (filters open incidents in-memory)
- **Stubbed:** 14 other query methods (return empty responses)

**Query APIs (15 RPCs across 6 services):**

**IncidentQueryService (2 RPCs):**
1. ✅ **SearchIncidents** - Text search over open incidents (IMPLEMENTED)
   - Connects to `srecorev1.IncidentServiceClient`
   - Filters by query in title/description
2. ❌ **GetIncidentTimeline** - Empty stub

**FleetHealthQueryService (1 RPC):**
1. ❌ **GetFleetHealth** - Empty stub

**AlertQueryService (1 RPC):**
1. ❌ **ListActiveAlerts** - Empty stub

**RunbookQueryService (1 RPC):**
1. ❌ **SearchRunbooks** - Empty stub

**SLOQueryService (1 RPC):**
1. ❌ **GetSLO** - Empty stub

**KnowledgeIndexQueryService (1 RPC):**
1. ❌ **SearchPatterns** - Empty stub

**Storage Pattern:**
```
No persistent store - all data from domain incident service
Implemented: SearchIncidents filters in-memory incident list
Stubbed: Fleet health, alerts, runbooks, SLOs, patterns
```

**Test Coverage (Partial):**
- **67 LOC** of test code for 71 LOC implementation (94% ratio, but implementation is stubs)
- **Unit Test:** 1 test function
  - `TestSearchIncidents_FiltersByQuery` - Tests actual search logic with fake domain
- **Test Infrastructure:**
  - `fakeIncidentDomain` mock implementing `IncidentServiceClient`
  - No tests for 14 stub methods

**Code Quality:**
- Clean separation: handlers → domain client
- TLS enforcement via `serverCreds()` function
- Minimal implementation: 72 LOC handlers

**Gaps:**
- ❌ **Incomplete:** 1 of 15 RPCs implemented (6.7%)
- ❌ No persistent read models (queries to domain only)
- ❌ Missing knowledge base/runbook storage
- ❌ Missing SLO evaluation logic
- ❌ No fleet health aggregation
- ❌ No GitOps deployment manifest
- ❌ No integration test runner script

**Critical Files:**
- Proto: `packages/ameide_core_proto/src/ameide_core_proto/sre/core/v1/query.proto`
- Handlers: `primitives/projection/sre/internal/handlers/handlers.go` (72 LOC)
- Tests: `primitives/projection/sre/internal/handlers/handlers_test.go` (67 LOC)

---

## Comparative Matrices

### Storage & Read Models

| Domain | Database | Migration System | Tables | Event Consumption | Idempotency |
|--------|----------|------------------|--------|-------------------|-------------|
| Sales | PostgreSQL | ✅ Embedded | 5 | ✅ Two-stream | ✅ Inbox pattern |
| Transformation | PostgreSQL | ✅ Embedded | 6 | ✅ Architecture facts | ✅ Inbox pattern |
| Commerce | PostgreSQL | ❌ External | 2 | ❌ Unclear | ❌ Manual UPSERTs |
| SRE | None | N/A | 0 | N/A | N/A |

### Query API Completeness

| Domain | Total RPCs | Implemented | Stubbed | Implementation Rate |
|--------|-----------|-------------|---------|---------------------|
| Sales | 6 | 6 | 0 | 100% ✅ |
| Transformation | 7 | 7 | 0 | 100% ✅ |
| Commerce | 3 | 3 | 0 | 100% ✅ |
| SRE | 15 | 1 | 14 | 6.7% ❌ |

### Test Coverage

| Domain | Test Files | Test LOC | Implementation LOC | Test Ratio | Coverage |
|--------|-----------|----------|-------------------|------------|----------|
| Sales | 8 | 308 | 1,100 | 28% | ⭐⭐⭐⭐⭐ |
| Transformation | 1 | 84 | 1,000+ | 8% | ⭐⭐⭐ |
| SRE | 1 | 67 | 71 | 94%* | ⭐⭐ (stubs) |
| Commerce | 1 | 6 | 219 | 2.7% | ⭐ (stub only) |

*SRE ratio inflated due to stub implementation

### Architectural Complexity

| Domain | Code LOC | Pattern | Separation | Testability |
|--------|---------|---------|------------|-------------|
| Sales | ~1,100 | CQRS Read Model | ✅ Handler + Adapter | ✅ Interface mocks |
| Transformation | ~1,000 | CQRS Read Model | ✅ Handler + Adapter | ✅ sqlmock |
| Commerce | ~220 | Domain-Specific | ❌ Monolithic | ⚠️ Direct DB |
| SRE | ~72 | Partial Proxy | ✅ Handler + Client | ✅ Client mocks |

### Deployment Readiness

| Domain | GitOps Manifest | Docker | Security | Production Ready |
|--------|----------------|--------|----------|------------------|
| Sales | ✅ Yes | Multi-stage | Optional TLS | ✅ YES |
| Transformation | ❌ No | Multi-stage | None | ⚠️ GitOps needed |
| Commerce | ❌ No | Multi-stage | Optional TLS/mTLS | ⚠️ Tests needed |
| SRE | ❌ No | Multi-stage | Mandatory TLS | ❌ Incomplete |

### Database Patterns (Sales, Transformation & Commerce)

| Aspect | Sales | Transformation | Commerce |
|--------|-------|----------------|----------|
| **ORM** | None (raw SQL) | None (raw SQL) | None (raw SQL) |
| **SQL Safety** | Prepared statements | Prepared statements | Prepared statements |
| **Schema Validation** | Regex | Regex | Regex |
| **Connection Pool** | 10 max / 10 idle | 5 max / 5 idle | 10 max / 10 idle |
| **Transaction Isolation** | READ COMMITTED | Default | Default |
| **Multi-Tenancy** | tenant_id in all | tenant+org+repo in all | tenant_id in all |
| **Denormalization** | 5 read model tables | 6 read model tables | 2 specialized tables |
| **Complex Types** | Money triples | JSONB properties | Protobuf JSON |
| **Indexes** | Strategic | Pagination-optimized | Not visible |

---

## Key Findings

### Architectural Convergence (Updated)

**Three patterns now established** (down from four - Transformation migrated to CQRS):

1. **Traditional CQRS (Sales, Transformation):**
   - Event consumption → Materialized views → Query optimization
   - Best for complex domains with rich query patterns
   - **Transformation now follows this pattern** with full PostgreSQL-backed read models

2. **Domain-Specific (Commerce):**
   - Direct state storage → Query API
   - Best for specialized concerns (hostname resolution)

3. **Partial Stub (SRE):**
   - Work in progress → Mostly unimplemented
   - Pattern unclear (needs design decision)

### Maturity Gaps (Updated)

1. **Test Coverage Gap** (2 of 4 domains):
   - Sales: Comprehensive tests (308 LOC) ✅
   - Transformation: Good tests (84 LOC) ✅
   - Commerce: Empty smoke test (6 LOC) ❌
   - SRE: Single test for only implemented method (67 LOC) ⚠️

2. **GitOps Deployment** (3 of 4 domains):
   - Only Sales has deployment manifest
   - Commerce, Transformation, SRE cannot be deployed

3. **SRE Incomplete** (14 of 15 RPCs stubbed):
   - No persistent storage
   - No event consumption
   - No aggregation logic for fleet health, SLOs, alerts
   - No knowledge base implementation

4. **Event Consumption Unclear** (Commerce):
   - Has PostgreSQL tables but no visible event consumer
   - How do hostname mappings get populated?
   - Missing inbox pattern for idempotency

5. **CI/CD Integration** (ALL domains):
   - Integration test scripts exist but **not invoked by CI**
   - No automated testing in `.github/workflows/ci-integration-packs.yml`
   - Workflow focuses on services, not primitives

### Best Practices (from Sales)

1. **Migration System:**
   - Embedded SQL files with version tracking
   - Transaction-based with SERIALIZABLE isolation
   - Idempotent via `IF NOT EXISTS`

2. **Idempotency:**
   - Inbox pattern with `(tenant_id, message_id)` primary key
   - `ON CONFLICT DO NOTHING` for exactly-once processing
   - Transaction spans inbox + mutations

3. **Test Architecture:**
   - Interface-based store design
   - Fake store for unit tests
   - sqlmock for integration tests
   - Integration mode support (mock/cluster)

4. **Query Optimization:**
   - Denormalized read models matching query patterns
   - Strategic indexes on filter dimensions
   - Cursor-based pagination for performance

5. **Clean Architecture:**
   - Handler layer (gRPC serialization)
   - Adapter layer (data access)
   - Clear interfaces enabling mocking

---

## Recommendations

### P0 (Immediate)

1. **Complete SRE Projection**
   - Implement 14 stubbed query methods or remove if not needed
   - Add persistent storage for fleet health, alerts, SLOs
   - Implement knowledge base and runbook indexing
   - Decision: Follow Sales pattern (CQRS) or Transformation pattern (proxy)?

2. **Add Tests for Commerce & Transformation**
   - Commerce: sqlmock tests for hostname resolution, schema validation, error cases
   - Transformation: Mock upstream client, test error propagation
   - Target: 20%+ test coverage minimum

3. **Add GitOps Manifests**
   - Commerce: Create `projection-commerce-v0.yaml` (use Sales as template)
   - Transformation: Create `projection-transformation-v0.yaml`
   - SRE: Create `projection-sre-v0.yaml`

4. **Clarify Commerce Event Consumption**
   - Document how hostname mappings are populated
   - Add event consumer if missing
   - Implement inbox pattern for idempotency

### P1 (Short-term)

5. **Add Projections to CI/CD**
   - Update `.github/workflows/ci-integration-packs.yml`
   - Include `primitives/projection/**/__tests__/integration/**`
   - Execute `run_integration_tests.sh` for each domain

6. **Migrate Commerce to Adapter Pattern**
   - Separate handlers from data access (like Sales)
   - Create `internal/adapters/postgres/` layer
   - Add migration system for schema versioning
   - Improve testability via interfaces

7. **Standardize Pagination**
   - Implement cursor-based pagination in Commerce (if needed)
   - Document pagination patterns
   - Add pagination tests

8. **Add Migration Systems**
   - Commerce: Implement embedded migrations (use Sales pattern)
   - Consider: Separate migration CLI vs embedded?

### P2 (Long-term)

9. **Clarify Projection Semantics**
   - ~~Transformation: Rename to "query-facade" or clarify projection role~~ ✅ Now a proper CQRS projection
   - SRE: Choose pattern (CQRS vs proxy vs remove)
   - Document when to use each pattern

10. **Event Consumption Best Practices**
    - Document inbox pattern usage (Sales example)
    - Create shared inbox table schema
    - Standardize event consumer entry points

11. **Monitoring & Observability**
    - Add event lag metrics (inbox vs latest event)
    - Add read model staleness metrics
    - Alert on projection failures

12. **Performance Optimization**
    - Add connection pooling metrics
    - Optimize indexes based on query patterns
    - Consider materialized views for expensive aggregations

---

## File Reference Index

### Proto Definitions
- **Sales:** `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/sales_query_service.proto`
- **Commerce:** `packages/ameide_core_proto/src/ameide_core_proto/commerce/v1/commerce_query.proto`
- **Transformation:** `packages/ameide_core_proto/src/ameide_core_proto/transformation/architecture/v1/transformation_architecture_query.proto`
- **SRE:** `packages/ameide_core_proto/src/ameide_core_proto/sre/core/v1/query.proto`

### Projection Primitives
- Sales: `primitives/projection/sales/`
- Commerce: `primitives/projection/commerce/`
- Transformation: `primitives/projection/transformation/`
- SRE: `primitives/projection/sre/`

### Database Migrations
- Sales: `primitives/projection/sales/internal/adapters/postgres/migrations/V1__sales_projection.sql`
- Transformation: `primitives/projection/transformation/internal/adapters/postgres/migrations/V1__transformation_projection.sql`
- Commerce: None (external management)
- SRE: N/A

### Test Files
- Sales: `primitives/projection/sales/internal/tests/*_test.go` (8 files, 308 LOC)
- Transformation: `primitives/projection/transformation/tests/archimate_projection_apply_test.go` (84 LOC)
- Commerce: `primitives/projection/commerce/tests/smoke_test.go` (6 LOC)
- SRE: `primitives/projection/sre/internal/handlers/handlers_test.go` (67 LOC)

### Integration Test Runners
- Sales: `primitives/projection/sales/tests/run_integration_tests.sh`
- Commerce: `primitives/projection/commerce/tests/run_integration_tests.sh`
- Transformation: `primitives/projection/transformation/tests/run_integration_tests.sh`
- SRE: **Missing**

### GitOps Manifests
- Sales: `gitops/ameide-gitops/sources/values/_shared/apps/projection-sales-v0.yaml`
- Commerce: **Missing**
- Transformation: **Missing**
- SRE: **Missing**

### CI/CD Configuration
- `.github/workflows/ci-integration-packs.yml` (does NOT include primitives)
- `.github/workflows/ci-core-quality.yml` (packages/services only)

---

## Related Backlog Items

- **551**: Comparative integration primitive analysis
- **550**: Comparative domain primitive analysis (if exists)
- **496**: EDA/Idempotency patterns
- **537**: Primitive testing discipline
- **540-sales-projection.md**: Sales projection component

---

## Conclusion (Updated 2025-12-17)

**Convergence toward CQRS pattern** with Transformation joining Sales:

- **Sales**: Production-ready CQRS read model with idempotent event consumption ✅
- **Transformation**: Now production-ready CQRS with 6 tables, inbox pattern, 13 event types ✅ (MAJOR UPGRADE)
- **Commerce**: Specialized domain projection with unclear event story ⚠️
- **SRE**: Scaffold with 93% stub rate (needs completion or removal) ❌

**Two dominant patterns established:**
1. **Complex domains → CQRS pattern** (Sales, Transformation) - event sourcing + read models
2. **Specialized concerns → Domain-specific pattern** (Commerce) - direct state + query API

**Production readiness status:**
| Domain | Test Coverage | GitOps | Event Consumption | Migration | Architecture | Status |
|--------|--------------|--------|-------------------|-----------|--------------|--------|
| Sales | ✅ 28% | ✅ | ✅ | ✅ | ✅ | **READY** |
| Transformation | ✅ 8% | ❌ | ✅ | ✅ | ✅ | **NEAR-READY** |
| Commerce | ❌ 0% | ❌ | ❌ | ❌ | ❌ | NEEDS WORK |
| SRE | ⚠️ 94%* | ❌ | ❌ | N/A | ⚠️ | INCOMPLETE |

**Next steps:**
1. Add GitOps manifest for Transformation projection
2. Decide SRE projection pattern (CQRS vs proxy vs remove)
3. Add tests and event consumption for Commerce
