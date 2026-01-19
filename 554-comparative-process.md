# 554 — Comparative Process Primitives Analysis

**Status:** Reviewed 2025-12-17 (analysis complete; codebase metrics corrected)
**Audience:** Architecture, platform engineering, operators/CLI, workflow teams
**Scope:** Comparative analysis of process primitive implementations across Sales, Commerce, Transformation, and SRE to identify maturity levels, workflow patterns, gaps, and recommended next steps.

> **Update (v6 posture, 2026-01):** this document analyzes a **Temporal-first** era of Process primitives.  
> Current posture for **business BPMN** workflows is Zeebe (default) or Flowable (supported profile) per `backlog/520-primitives-stack-v6.md` and `backlog/511-process-primitive-scaffolding-v3.md`.  
> Temporal remains platform-only (non-BPMN platform workflows). Treat Temporal-specific “best practices” here as historical.

## Codebase Metrics (December 2025)

| Process | Workflows (LOC) | Activities (LOC) | Ingress Router (LOC) | Tests (LOC) | Proto Facts |
|---------|-----------------|------------------|----------------------|-------------|-------------|
| **Sales** | 306 | 118 | 111 | 77 | 6 events (78 LOC) |
| **SRE** | 209 | 50 | 76 | 69 | 16 events (148 LOC) |
| **Commerce** | 22 (placeholder) | 0 | 25 (scaffold) | 63 | 0 (no process proto) |
| **Transformation** | 4 (constant only) | 0 | 4 (empty struct) | 63 | 0 events |

**Total Implementation Lines:**
- **Sales:** 535 LOC (production-ready)
- **SRE:** 404 LOC (production-ready)
- **Commerce:** 47 LOC (scaffold)
- **Transformation:** 8 LOC (placeholder)

**Use with:**
- Process primitives: `backlog/540-sales-process.md`, `backlog/523-commerce-process.md`, `backlog/527-transformation-process.md`, `backlog/526-sre-process.md`
- EDA contract rules: `backlog/496-eda-principles-v2.md`, `backlog/496-eda-protobuf-ameide.md`
- Proto conventions: `backlog/509-proto-naming-conventions.md`
- Primitives stack: `backlog/520-primitives-stack-v2.md`
- Capability definitions: `backlog/540-sales-capability.md`, `backlog/526-sre-capability.md`, `backlog/523-commerce.md`, `backlog/527-transformation-capability.md`

> **Update (2026-01): 430v2 test contract**
>
> This comparative doc references legacy “integration harness” scripts (`run_integration_tests.sh`) as an implementation detail in some scaffolds. Treat `backlog/430-unified-test-infrastructure-v2-target.md` as the normative contract (native tooling; strict phases; no pack scripts as the canonical path).

## Layer header (Application Architecture + Technology)

- **Primary ArchiMate layer(s):** Application (Application Component, Service, Event) + Technology (Workflow orchestration, Temporal infrastructure)
- **Primary element types used:** Application Component, Application Service, Data Object, Technology Artifact, Business Process
- **Prohibited unless qualified:** Generic use of "orchestration" without specifying the orchestration runtime (Zeebe/Flowable for BPMN; Temporal for platform workflows)

## 0) Problem

Four process primitives (Sales, Commerce, Transformation, SRE) exist at varying levels of maturity with different implementation approaches. We lack a consolidated view of:

- Comparative implementation completeness (Temporal workflows, activities, proto contracts, GitOps)
- Workflow orchestration pattern consistency (signal-driven, saga patterns, idempotency)
- Common gaps across all processes (deployment configs, test coverage, compensation logic)
- Strategic recommendations for bringing all processes to production readiness with Temporal

## 1) Executive Summary

### Overall Maturity Ranking

1. **SRE Process** - 85% Complete (Production-Ready Workflows)
2. **Sales Process** - 80% Complete (Production-Ready, Deployed)
3. **Commerce Process** - 30% Complete (Scaffold with Minimal Logic)
4. **Transformation Process** - 10% Complete (Placeholder Only)

### Implementation Strategy Variance

**Two Distinct Approaches:**

1. **Temporal-First (Sales, SRE):**
   - Start with Temporal workflows and activities
   - Implement signal-driven orchestration patterns
   - Full worker + ingress dual-binary architecture
   - Result: Production-ready workflow orchestration

2. **Deferred Implementation (Commerce, Transformation):**
   - Define process contracts but defer implementation
   - Minimal or no Temporal workflow logic
   - Placeholder scaffolds awaiting completion
   - Result: Structural foundation only

## 2) Implementation Completeness Matrix

| Aspect | Sales | SRE | Commerce | Transformation |
|--------|-------|-----|----------|-----------------|
| **Workflow Count** | 2 | 1 | 0 (placeholder only) | 0 |
| **Workflow Complexity** | High (dual workflows) | Medium (incident triage) | None | None |
| **Activities** | 3 | 1 | 0 | 0 |
| **Ingress Router** | Full (SignalWithStart) | Full (SignalWithStart) | Scaffold | None |
| **Worker Bootstrap** | Full | Full | Scaffold | Placeholder |
| **Process Facts** | 6 event types | 16 event types | 0 | 0 |
| **Proto Contract** | 60% (events only) | 65% (events only) | 0% (no process contract) | 0% (no process contract) |
| **Event Handling** | Full | Full | Defined only | Not implemented |
| **Test Coverage** | Basic | Intermediate | None | None |
| **Idempotency** | Implemented (version tracking) | Implemented (version tracking) | Not started | Not started |
| **GitOps Deployment** | ✓ Active | ✗ Missing | ✗ Missing | ✗ Missing |
| **Documentation** | Comprehensive README | Minimal README | Minimal README | Missing README |
| **Dockerfile** | Multi-stage (worker+ingress) | Multi-stage (worker+ingress) | Multi-stage (worker+ingress) | Single binary (no Temporal SDK) |

## 3) Process-by-Process Analysis

### 3.1 Sales Process - Production-Ready Deployed (80%)

**Status:** Production-ready with GitOps manifests present; dual workflow implementation

**Strengths:**
- **Two production workflows implemented:**
  1. `QuoteApprovalWorkflow` - 24h default timeout, signal-driven lifecycle management
  2. `ErpHandoffWorkflow` - Quote acceptance → ERP order creation orchestration
- **Three activities:** PublishProcessFact, FetchOpportunityIDForQuote, CreateErpOrder
- **Full ingress router:** Deterministic workflow ID derivation, SignalWithStart pattern for idempotency
- **Dual-binary architecture:** Worker and ingress built separately for scaling
- **Process fact emission:** 6 event types covering quote approval and ERP handoff flows
- **Version tracking:** lastSeenVersion in workflow state for idempotency
- **Active deployment:** GitOps config with Temporal connection

**Implementation Details:**
- **Location:** `primitives/process/sales/`
- **Proto Files:** `packages/ameide_core_proto/src/ameide_core_proto/sales/process/v1/process_facts.proto`
- **Workflows:** `internal/workflows/workflow.go` - QuoteApprovalWorkflow + ErpHandoffWorkflow
- **Activities:** `internal/activities/activities.go` - 3 implemented activities
- **Ingress Router:** `internal/ingress/router.go` - Full SignalWithStart implementation
- **Worker:** `cmd/worker/main.go` - Temporal worker with workflow registration
- **GitOps:** `gitops/ameide-gitops/sources/values/_shared/apps/process-sales-v0.yaml`
- **Topics:** `sales.process.facts.v1`

**Workflow Patterns:**
- **Deterministic Workflow IDs:**
  - `sales.quote_approval.{tenantID}.{quoteID}`
  - `sales.erp_handoff.{tenantID}.{quoteID}`
- **Signal-based orchestration:** Domain facts trigger workflow state transitions
- **Activity retry policy:** Max 5 attempts, exponential backoff (2s → 30s)
- **Timeout management:** Timer-based approval timeout with escalation
- **Cross-domain coordination:** Integration service calls for ERP order creation

**Process Facts (6 events):**
1. **Quote Approval Flow:**
   - QuoteApprovalRequested
   - QuoteApprovalTimedOut
   - QuoteApprovalEscalated

2. **ERP Handoff Flow:**
   - OpportunityHandoffToErpRequested
   - OpportunityHandoffToErpCompleted
   - OpportunityHandoffToErpFailed

**Gaps:**
- No process query API (by design; use Temporal visibility + projection timelines for read/query)
- No compensation/rollback workflows defined
- No process instance cancellation support
- Limited test coverage (only ping_test.go)
- No smoke tests defined

**Key Patterns:**
- At-least-once delivery with SignalWithStart idempotency
- Workflow state tracks aggregate versions
- Process emits facts, never writes to domain DB
- Integration via activities (domain RPC, integration RPC)

---

### 3.2 SRE Process - Production-Ready Not Deployed (85%)

**Status:** Production-ready implementation; most comprehensive process facts; not deployed

**Strengths:**
- **One sophisticated workflow:** `IncidentTriageWorkflow` with multi-phase orchestration
- **16 process fact types:** Most comprehensive event catalog across all domains
- **Full ingress router:** Signal-based routing with validation
- **Incident lifecycle management:** Created → Resolved → Closed state tracking
- **W3C trace context support:** Full distributed tracing integration
- **Comprehensive tests:** Router unit tests with fake Temporal clients
- **12-hour workflow timeout:** Appropriate for incident response SLAs

**Implementation Details:**
- **Location:** `primitives/process/sre/`
- **Proto Files:** `packages/ameide_core_proto/src/ameide_core_proto/sre/process/v1/process_facts.proto`
- **Workflow:** `internal/workflows/workflow.go` - IncidentTriageWorkflow
- **Activity:** `internal/activities/activities.go` - PublishProcessFact
- **Ingress Router:** `internal/ingress/router.go` - Full validation and routing
- **Worker:** `cmd/worker/main.go` - Temporal worker with workflow registration
- **Topics:** `sre.process.facts.v1`
- **Tests:** `internal/ingress/router_test.go` - 2 comprehensive unit tests

**Workflow Patterns:**
- **Workflow ID:** `sre.incident_triage.{tenantID}.{incidentID}`
- **Signal-driven:** SRE domain facts trigger triage phases
- **Activity timeout:** 30s start-to-close, 5 max retries
- **State tracking:** IncidentTriageState with IncidentID, Phase, UpdatedAt
- **Outcome tracking:** "resolved" or "closed" workflow completion

**Process Facts (16 events - most comprehensive):**
1. **Incident Triage Workflow:**
   - IncidentTriageStarted
   - PatternLookupCompleted (matched backlog items)
   - TriagePhaseCompleted
   - RemediationProposed (proposal + risk tier)
   - RemediationApplied (evidence ref)
   - VerificationStarted
   - VerificationCompleted (health status)
   - DocumentationRequested (suggested backlog item)
   - IncidentTriageCompleted (outcome)

2. **Additional Workflows:**
   - ChangeVerificationStarted/Completed (GitOps commit SHA tracking)
   - SLOBurnEvaluationCompleted (budget, burn rate)
   - SLOBurnAlertRaised (alert + incident correlation)
   - AlertCorrelationCompleted
   - RunbookExecutionProcessStarted/Completed

**Gaps:**
- No GitOps deployment config (implementation complete, not deployed)
- No process query API (by design; use Temporal visibility + projection timelines for read/query)
- No compensation workflows
- No runbook orchestration implementation (facts defined, no workflow)
- Limited documentation

**Key Patterns:**
- Multi-phase workflow with phase transitions
- Evidence-based remediation tracking
- Pattern matching against backlog items
- SLO budget burn rate monitoring
- Alert correlation and runbook execution scaffolding

---

### 3.3 Commerce Process - Minimal Scaffold (30%)

**Status:** Scaffold with placeholder workflow; no commerce-specific process proto contract; not deployed

**Strengths:**
- **Worker + ingress architecture:** Dual-binary structure in place
- **Temporal SDK integration:** Dependencies properly configured
- **Process fact topic defined:** `commerce.process.facts.v1`

**Implementation Details:**
- **Location:** `primitives/process/commerce/`
- **Workflow:** `internal/workflows/workflow.go` - DomainOnboardingWorkflow (PLACEHOLDER)
- **Ingress Router:** `internal/ingress/router.go` - Skeleton with TODO comments (not yet wired)
- **Worker:** `cmd/worker/main.go` - Scaffold with placeholder registration
- **Ingress:** `cmd/ingress/main.go` - Scaffold (connects to Temporal; TODO broker subscription)
- **Topics:** `commerce.process.facts.v1` (constant only; no proto schema yet)

**Workflow Patterns:**
- **Placeholder workflow:** DomainOnboardingWorkflow logs start/completion, sleeps 1s, no logic
- **Intent:** Orchestrate DNS verification + cert issuance + mapping activation
- **Parameters:** tenantID, hostname
- **No workflow ID derivation implemented**

**Process Facts:**
- None defined (no commerce process facts proto contract exists)

**Gaps:**
- **No real workflow implementation** (placeholder only)
- No activities defined
- No ingress router logic (complete skeleton)
- No workflow ID derivation
- No process state tracking
- No process fact schema (no events contract vs Sales/SRE)
- Stub-only tests (smoke test + harness script; no workflow assertions)
- Missing explicit workflow context (no workflow_id/workflow_type in subject)
- No GitOps deployment config

**Key Patterns (Planned but Not Implemented):**
- BYOD domain onboarding orchestration
- DNS verification coordination
- Certificate issuance tracking
- Hostname mapping activation

---

### 3.4 Transformation Process - Placeholder Only (10%)

**Status:** Placeholder; gRPC Ping server + worker/ingress stubs; no process facts contract; no Temporal SDK integration (yet)

**Strengths:**
- **gRPC server implemented:** Health checks and reflection working
- **Basic infrastructure:** Dockerfile and build process ready
- **Temporal reachability check:** Main binary requires `TEMPORAL_ADDRESS` to be reachable at startup

**Implementation Details:**
- **Location:** `primitives/process/transformation/`
- **Proto Files:** Shared `process/v1` ping service only (no transformation process facts proto)
- **Workflow:** `internal/workflows/workflow.go` - Constant only (`RefinementWorkflowName`)
- **Worker:** `cmd/worker/main.go` - PLACEHOLDER (single log statement)
- **Ingress:** `cmd/ingress/main.go` - PLACEHOLDER (single log statement)
- **gRPC Server:** `cmd/main.go` - Ping + health checks; requires Temporal reachability
- **No Temporal SDK:** Missing from `go.mod` dependencies (worker/ingress are placeholders)

**Workflow Patterns:**
- **Workflow name constant defined:** `transformation.refinement.v0`
- **No workflow implementation**
- **No worker bootstrap logic**
- **No ingress routing logic**

**Process Facts:**
- None defined (no process contract exists)

**Gaps:**
- **No Temporal SDK dependency** (critical gap)
- No process fact definitions
- No workflow implementations
- No activities
- No ingress router
- No worker logic
- No process state
- Stub-only tests (smoke test + harness script; no workflow assertions)
- No documentation (no README, no catalog-info.yaml)
- No GitOps deployment config
- No event ingestion path (no broker subscription; no SignalWithStart routing)

---

## 4) Architectural Patterns (Process Primitives)

### 4.1 Worker + Ingress Dual-Binary Pattern

**Used by:** Sales, SRE, Commerce (scaffold)

**Pattern:**
```
primitives/process/{domain}/
├── cmd/
│   ├── worker/main.go          # Temporal worker: polls task queue, executes workflows
│   └── ingress/main.go         # Event router: consumes domain facts, signals workflows
├── internal/
│   ├── workflows/
│   │   └── workflow.go         # Deterministic workflow definitions
│   ├── activities/
│   │   └── activities.go       # Non-deterministic side effects
│   ├── ingress/
│   │   └── router.go           # SignalWithStart routing logic
│   └── process/
│       └── state.go            # Workflow state structures
```

**Dockerfile:**
- Build both binaries: `process-{domain}-worker`, `process-{domain}-ingress`
- Base: `golang:1.25` → `gcr.io/distroless/base-debian12`
- Entry point: Worker binary (ingress deployed separately)

**Configuration:**
- `TEMPORAL_NAMESPACE`: Isolation namespace (default: `default`)
- `TEMPORAL_TASK_QUEUE`: Worker polling queue (e.g., `process-sales-v0`)
- `TEMPORAL_ADDRESS`: Frontend service (e.g., `temporal-frontend:7233`)
- `DOMAIN_ADDRESS`: Domain service gRPC endpoint
- `INTEGRATION_ADDRESS`: Integration service gRPC endpoint (if needed)

### 4.2 Signal-Driven Workflow Orchestration

**Used by:** Sales, SRE

**Pattern:**
1. **Ingress receives domain fact** (from Kafka/event stream)
2. **Derive deterministic workflow ID** from (tenant_id, aggregate_id)
3. **SignalWithStart workflow:**
   - If workflow exists: deliver signal
   - If workflow not running: start workflow + deliver signal
4. **Workflow processes signal:**
   - Update internal state
   - Track lastSeenAggregateVersion for idempotency
   - Execute activities (side effects)
   - Emit process facts
5. **Activity publishes process fact** to Kafka

**Idempotency Guarantees:**
- At-least-once delivery: same domain fact may arrive multiple times
- Workflow state tracks `lastSeenVersion`
- Duplicate signals ignored if version ≤ lastSeenVersion
- Workflow ID determinism ensures single workflow instance per aggregate

**Example (Sales QuoteApprovalWorkflow):**
```
Domain Fact: QuoteSubmittedForApproval (v3)
  ↓
Ingress derives workflow ID: sales.quote_approval.{tenant}.{quote_id}
  ↓
SignalWithStart(workflow_id, "domain_fact", fact)
  ↓
Workflow checks: lastSeenVersion < 3?
  Yes → Process signal, update state, emit QuoteApprovalRequested
  No → Ignore duplicate
```

### 4.3 Activity Patterns

**Activities are non-deterministic side effects:**
- Publishing process facts to Kafka
- Calling domain service gRPC endpoints
- Calling integration service gRPC endpoints
- External API calls

**Activity Options:**
- `StartToCloseTimeout`: 30s (SRE), 60s (Sales)
- `RetryPolicy`: Max 5 attempts, exponential backoff (2s → 30s)

**Activities in Sales:**
1. `PublishProcessFact` - Marshals and publishes to Kafka
2. `FetchOpportunityIDForQuote` - Calls SalesCommandService
3. `CreateErpOrder` - Calls SalesIntegrationService

**Activities in SRE:**
1. `PublishProcessFact` - Publishes SRE process facts

### 4.4 Process Fact Emission Pattern

All process primitives follow:
```
1. Workflow receives domain fact signal
2. Workflow updates internal state (deterministic)
3. Workflow invokes activity to publish process fact
4. Activity marshals protobuf message
5. Activity publishes to Kafka topic: {domain}.process.facts.v1
6. Activity returns to workflow
7. Workflow continues or completes
```

**Topic Naming:**
- `sales.process.facts.v1`
- `sre.process.facts.v1`
- `commerce.process.facts.v1`

**Partition Keys:**
- `tenant_id` + `workflow_id`

**Metadata:**
- `message_id` (UUID, idempotency)
- `correlation_id` (request tracing)
- `causation_id` (event causality)
- `occurred_at` (timestamp)
- `producer` (source process)
- `schema_version` (version compatibility)
- `traceparent` / `tracestate` (W3C Trace Context)

### 4.5 Standard File Structure

**Production-ready processes (Sales, SRE):**
```
primitives/process/{domain}/
├── cmd/
│   ├── worker/main.go                  # Temporal worker entry point
│   └── ingress/main.go                 # Event ingress router
├── internal/
│   ├── workflows/
│   │   └── workflow.go                 # Workflow definitions
│   ├── activities/
│   │   └── activities.go               # Activity handlers
│   ├── ingress/
│   │   └── router.go                   # Fact routing logic
│   ├── process/
│   │   └── state.go                    # Workflow state structs
│   ├── handlers/
│   │   └── handlers.go                 # gRPC service handlers (Ping, etc.)
│   └── gen/
│       └── process_services.generated.go  # Generated service registration
├── tests/
│   └── run_integration_tests.sh        # Integration test harness
├── events/
│   └── catalog.go                       # Event topic constants
├── Dockerfile                           # Multi-stage build (worker+ingress)
├── catalog-info.yaml                    # Backstage metadata
├── README.md                            # Documentation
├── go.mod
└── go.sum
```

**Scaffold processes (Commerce):**
- Same structure, minimal implementation

**Placeholder processes (Transformation):**
- Single binary gRPC server pattern, no worker/ingress

## 5) Proto Contract Comparison

### 5.1 Process Fact API Surface

| Process | Process Facts | Workflow Types | Service API | Query Capability | Idempotency |
|---------|--------------|----------------|-------------|------------------|-------------|
| **SRE** | 16 events | 1 workflow | None | None | ✓ (message_id) |
| **Sales** | 6 events | 2 workflows | None | None | ✓ (message_id) |
| **Commerce** | 0 | 1 placeholder | None | None | N/A |
| **Transformation** | 0 | 0 | Ping only | None | N/A |

### 5.2 Workflow Context Maturity

| Process | Workflow ID | Workflow Type | Subject Context | Partition Keys |
|---------|------------|---------------|-----------------|----------------|
| **SRE** | ✓ Explicit | ✓ Explicit | ✓ SreProcessSubject | tenant_id + workflow_id |
| **Sales** | ✓ Explicit | ✓ Explicit | ✓ SalesProcessSubject | tenant_id + workflow_id |
| **Commerce** | ✗ Missing | ✗ Missing | ✗ Implicit (scope-based) | tenant_id + scope |
| **Transformation** | N/A | N/A | N/A | N/A |

### 5.3 Event Catalog Completeness

**SRE (16 events - most comprehensive):**
- Incident triage: 9 events covering full lifecycle
- Change verification: 2 events
- SLO burn evaluation: 2 events
- Alert correlation: 1 event
- Runbook execution: 2 events (scaffolded)

**Sales (6 events):**
- Quote approval: 3 events (request, timeout, escalation)
- ERP handoff: 3 events (request, complete, fail)

**Commerce (0 events):**
- No process fact definitions (no commerce process proto)

**Transformation (0 events):**
- No process fact definitions

### 5.4 Metadata Pattern Compliance

| Process | Message ID | Correlation ID | Causation ID | Trace Context | Producer |
|---------|-----------|----------------|--------------|---------------|----------|
| **SRE** | ✓ | ✓ | ✓ | ✓ (traceparent/state) | ✓ |
| **Sales** | ✓ | ✓ | ✓ | ✓ (traceparent/state) | ✓ |
| **Commerce** | Partial | Partial | Partial | Partial | Partial |
| **Transformation** | N/A | N/A | N/A | N/A | N/A |

## 6) GitOps Deployment Status

### 6.1 Deployed Processes

**Sales Process:**
- **File:** `gitops/ameide-gitops/sources/values/_shared/apps/process-sales-v0.yaml`
- **Image:** `ghcr.io/ameideio/process-sales@sha256:<digest>`
- **Temporal Connection:**
  - Namespace: `default`
  - Task Queue: `process-sales-v0`
  - Address: `temporal-frontend:7233`
- **Replicas:** 1
- **Strategy:** RollingUpdate
- **Log Level:** `info`
- **Resources:** Not specified (operator defaults)
- **Smoke Tests:** None defined

### 6.2 Not Deployed (Code Complete or Partial)

**SRE Process:**
- Implementation: Complete (85%)
- Temporal workflows: Fully implemented
- GitOps config: Missing
- Recommended: Deploy immediately

**Commerce Process:**
- Implementation: Scaffold (30%)
- Temporal workflows: Placeholder only
- GitOps config: Missing
- Recommended: Complete implementation before deployment

**Transformation Process:**
- Implementation: Minimal (10%)
- Temporal workflows: None
- GitOps config: Missing
- Recommended: converge on the standard runner pattern (worker + ingress; SignalWithStart; see 511)

### 6.3 Process CRD Capabilities (Available but Underutilized)

The `ameide.io/v1` Process CRD supports:
```yaml
spec:
  temporal:                              # ✓ Used in deployed processes
    namespace: <namespace>
    taskQueue: <task-queue>
    address: <temporal-frontend-address>
  resources:                             # ✗ Not used in any deployment
    requests: { cpu, memory }
    limits: { cpu, memory }
  workerConfig:                          # ✗ Not used
    maxConcurrentWorkflows: <int>
    maxConcurrentActivities: <int>
  observability:                         # Partially used (logLevel only)
    logLevel: <level>
    metrics: <prometheus-config>
    tracing: <otel-config>
  env:                                   # ✗ Not used
```

Current deployments use only: `temporal`, `logLevel`

## 7) Common Gaps (All Processes)

### 7.1 Universal Gaps

1. **No Process Query API:** Cannot query workflow status, list instances, or inspect state via gRPC
2. **No Compensation Workflows:** No rollback/undo logic defined for any process
3. **No Process Cancellation:** No API to cancel or terminate running workflows
4. **Resource Limits:** No CPU/memory requests/limits defined in GitOps
5. **Observability:** Only logLevel configured; no Prometheus metrics or OpenTelemetry tracing
6. **Worker Configuration:** No concurrency limits (maxConcurrentWorkflows/Activities)

### 7.2 Specific Gaps by Process

**Sales:**
- No smoke tests defined (unlike ping-v0 reference)
- Limited test coverage (only ping_test.go)
- No SLA/deadline management queries
- No escalation policy configuration

**SRE:**
- No GitOps deployment (ready but not deployed)
- Runbook execution workflows scaffolded but not implemented
- No compensation for failed remediations

**Commerce:**
- Minimal implementation (placeholder workflow only)
- No workflow ID derivation logic
- No process state tracking
- Single event type (needs 8-10 more for BYOD flow)

**Transformation:**
- No Temporal SDK dependency
- No process fact contract
- Missing standard runner posture (worker + ingress; SignalWithStart)
- No worker/ingress implementation

## 8) Recommended Next Steps

### 8.1 For Sales Process (Improve Deployed)
1. **Add smoke tests:** Follow ping-v0 pattern with gRPC health checks
2. **Expand test coverage:** Add workflow and activity unit tests
3. **Add operational visibility:** standardize Temporal Search Attributes + projection timelines (avoid process-level read APIs)
4. **Define compensation workflows:** Rollback for failed ERP handoffs
5. **Resource tuning:** Add CPU/memory limits based on load testing
6. **Worker config:** Define maxConcurrentWorkflows/Activities

### 8.2 For SRE Process (Deploy Immediately)
1. **Create GitOps config:** Deploy to production (implementation complete)
2. **Add smoke tests:** gRPC health check harness
3. **Implement runbook workflows:** Complete RunbookExecutionProcess (scaffolded)
4. **Add operational visibility:** standardize Temporal Search Attributes + projection timelines (avoid process-level read APIs)
5. **Expand test coverage:** Add workflow tests
6. **Add compensation:** Define remediation rollback workflows

### 8.3 For Commerce Process (Complete Implementation)
1. **Implement DomainOnboardingWorkflow:** Replace placeholder with real logic
2. **Add activities:** DNS verification, cert issuance, mapping activation
3. **Complete ingress router:** Implement HandleFact with workflow ID derivation
4. **Expand process facts:** Add 8-10 event types for full BYOD flow
5. **Add workflow context:** Include workflow_id and workflow_type in subject
6. **Test coverage:** Add workflow and router tests
7. **GitOps deployment:** Only after implementation complete

### 8.4 For Transformation Process (Architectural Decision)
1. **Converge on event-driven runner:** Add Temporal SDK, implement worker + ingress, route facts via SignalWithStart (see 511)
2. **Define process fact contract:** emit orchestration timeline facts (for projections/UI)
3. **Implement workflows:** milestone progression, phase gates (Temporal Updates for user gates)
4. **GitOps deployment:** wire Temporal connection and deploy runner image

### 8.5 For All Processes
1. **Operational visibility:** use Temporal visibility/Search Attributes plus projection query APIs; do not introduce process-level read models
2. **Resource Governance:** Define CPU/memory requests/limits for all deployments
3. **Observability:** Add Prometheus metrics, OpenTelemetry tracing, alerting rules
4. **Worker Configuration:** Define concurrency limits for optimal throughput
5. **Compensation Framework:** Define standard patterns for rollback/undo workflows
6. **Smoke Test Standard:** Create gRPC health check smoke tests for all deployed processes
7. **Documentation:** Complete README files with workflow diagrams and event catalogs

## 9) Critical Files Reference

### Sales Process
- **Proto:** `packages/ameide_core_proto/src/ameide_core_proto/sales/process/v1/process_facts.proto`
- **Workflows:** `primitives/process/sales/internal/workflows/workflow.go`
- **Activities:** `primitives/process/sales/internal/activities/activities.go`
- **Ingress:** `primitives/process/sales/internal/ingress/router.go`
- **Worker:** `primitives/process/sales/cmd/worker/main.go`
- **GitOps:** `gitops/ameide-gitops/sources/values/_shared/apps/process-sales-v0.yaml`

### SRE Process
- **Proto:** `packages/ameide_core_proto/src/ameide_core_proto/sre/process/v1/process_facts.proto`
- **Workflow:** `primitives/process/sre/internal/workflows/workflow.go`
- **Activity:** `primitives/process/sre/internal/activities/activities.go`
- **Ingress:** `primitives/process/sre/internal/ingress/router.go`
- **Worker:** `primitives/process/sre/cmd/worker/main.go`
- **Tests:** `primitives/process/sre/internal/ingress/router_test.go`

### Commerce Process
- **Proto:** `packages/ameide_core_proto/src/ameide_core_proto/process/commerce/v1/commerce_process_facts.proto`
- **Workflow:** `primitives/process/commerce/internal/workflows/workflow.go` (placeholder)
- **Ingress:** `primitives/process/commerce/internal/ingress/router.go` (scaffold)
- **Worker:** `primitives/process/commerce/cmd/worker/main.go` (scaffold)

### Transformation Process
- **No Process Proto** (domain service only in `transformation/v1/transformation_service.proto`)
- **Workflow Constant:** `primitives/process/transformation/internal/workflows/workflow.go` (name only)
- **Worker:** `primitives/process/transformation/cmd/worker/main.go` (placeholder)
- **gRPC Server:** `primitives/process/transformation/cmd/main.go`

## 10) Conclusion

The four process primitives demonstrate distinct maturity levels:
- **Sales & SRE** are production-ready with complete Temporal workflow implementations
- **Commerce** has solid scaffolding but needs workflow logic completion
- **Transformation** needs convergence on the standard runner posture (worker + ingress; SignalWithStart)

All production-ready processes follow consistent patterns (signal-driven workflows, dual-binary architecture, process fact emission), but vary in deployment status and event catalog completeness.

### Key Insights: Temporal-First Success Pattern

The analysis reveals that **Temporal-first processes (Sales, SRE) achieved higher maturity** compared to scaffolds:
- **Signal-driven orchestration** with SignalWithStart provides robust idempotency
- **Dual-binary architecture** (worker + ingress) enables independent scaling
- **Process fact emission** maintains event-driven principles while orchestrating cross-domain flows
- **Version tracking in workflow state** solves at-least-once delivery challenges

### Primary Gaps Across All Processes

1. **No standardized operational visibility** - Use Temporal visibility + projection timelines (avoid process-level read APIs)
2. **No compensation workflows** - No rollback/undo logic defined
3. **Minimal test coverage** - Only basic tests; no workflow integration tests
4. **Resource limits missing** - All deployments use operator defaults
5. **SRE not deployed** - Production-ready code awaiting GitOps config

### Strategic Recommendation: Converge on Temporal Pattern

- **Deploy SRE immediately** (85% complete, tested, ready)
- **Complete Commerce workflows** before deployment (30% → 80%)
- **Bring Transformation to the standard runner posture** (worker + ingress; SignalWithStart)
- **Standardize operational visibility** via Temporal visibility + projection timelines
- **Add compensation frameworks** for all workflows with external side effects
