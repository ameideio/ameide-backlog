# 552 — Comparative Proto Contracts Analysis

**Status:** Reviewed 2025-12-17 (revalidated vs current proto sources; corrected counts + fixed stale examples)
**Audience:** Architecture, platform engineering, API designers, domain teams, SDK maintainers
**Scope:** Comparative analysis of protobuf contract definitions across Sales, Commerce, Transformation, and SRE domains to identify maturity levels, consistency patterns, code generation readiness, and API design quality.

**Use with:**
- Comparative implementations: `backlog/550-comparative-domain.md`
- Domain-specific proto: `backlog/540-sales-proto.md`, `backlog/523-commerce.md`, `backlog/527-transformation-capability.md`, `backlog/526-sre-capability.md`
- EDA principles: `backlog/496-eda-principles.md`, `backlog/496-eda-protobuf-ameide.md`
- Proto conventions: `backlog/509-proto-naming-conventions.md`
- Buf tooling: `backlog/365-buf-sdks-v2.md`

## Layer header (Application + Technology)

- **Primary ArchiMate layer(s):** Application (Application Service, Application Interface, Data Object) + Technology (SDK artifacts, code generation)
- **Primary element types used:** Application Service, Application Interface, Data Object, Technology Artifact
- **Prohibited unless qualified:** Mixing domain and integration concerns in single proto file; skipping event sourcing metadata

## 0) Problem

Four domain proto contracts (Sales, Commerce, Transformation, SRE) have been designed at different times with varying levels of sophistication. We lack a consolidated view of:

- **Proto contract completeness** (service definitions, event types, data models)
- **Event sourcing maturity** (metadata patterns, intent/fact symmetry, Confluent integration)
- **API design consistency** (CQRS separation, pagination, error handling, versioning)
- **Code generation readiness** (buf configs, SDK generation, multi-language support)
- **Strategic recommendations** for standardizing proto contracts across all domains

This analysis complements `550-comparative-domain.md` (which focuses on implementations) by examining the **contract layer** (proto definitions, API design, code generation infrastructure).

## 1) Executive Summary

### Overall Proto Maturity Ranking (Revised December 2025)

1. **Transformation** (ADVANCED / IN-FLUX) - 2,077 LOC across 12 files, 3 services / 40 RPCs, Core + Knowledge + Scrum contexts, complete message metadata across emitted facts
2. **SRE** (PRODUCTION-READY) - 7 services (17 RPCs), rich observable systems coverage, complete message metadata; intent/fact symmetry is not perfect (25 intents vs 26 facts)
3. **Sales** (PRODUCTION-READY) - Clean CQRS: 2 services / 24 RPCs (command includes 15 writes + 3 strong reads; query has 6 RPCs), symmetric 15 intent/fact, 5 buf configs
4. **Commerce** (FOCUSED/PRODUCTION) - 5 services / 16 RPCs (domain + integrations + agent), symmetric 4 intent/fact, domain-specific error codes (10 values incl UNSPECIFIED)

### Proto Complexity Comparison (Revised)

| Domain | Proto Files | Lines of Code | Services | Intents | Facts | Bounded Contexts |
|--------|------------|---------------|----------|---------|-------|------------------|
| **Transformation** | 12 | 2,077 | 3 | 13 (Scrum only; Knowledge writes are RPC) | 30 (Knowledge 16 + Scrum 14) | 3 (Core, Knowledge, Scrum) |
| **SRE** | 13 | 1,178 | 7 | 25 | 26 | 1 (Observable Systems) |
| **Sales** | 9 | 897 | 2 | 15 | 15 | 1 (Sales Lifecycle) |
| **Commerce** | 9 | 573 | 5 | 4 | 4 | 1 (Storefront BYOD) |

### Key Findings (Revised)

1. **Event Sourcing Maturity**: ✅ All four domains have message metadata patterns with `message_id`, correlation/causation IDs, and trace context; note Transformation uses multiple meta types (`TransformationMessageMeta`, `TransformationKnowledgeMessageMeta`, `ScrumMessageMeta`)
2. **CQRS Separation**: Sales is explicit (command + query); Transformation Knowledge is explicit (command + query); SRE splits projection queries into separate services but `IncidentService` mixes read/write; Commerce splits domain write vs projection read
3. **Code Generation**: Sales has 5 buf configs (domain, integration, process, projection, uisurface); Transformation has 1 (domain); Commerce and SRE have none
4. **API Surface**: Transformation has the largest current RPC surface (40 RPCs across Knowledge command/query + Scrum query); Sales has 24 total RPCs; Commerce has 16 total RPCs; SRE has 17 RPCs across 7 services
5. **Buf Configuration Gap**: Commerce and SRE need domain buf configs for consistent code generation

## 2) Proto Files Inventory

### Sales Domain (9 files, 897 LOC)

**Location:** `packages/ameide_core_proto/src/ameide_core_proto/sales/`

**File Structure:**
```
sales/
├── core/v1/
│   ├── sales.proto                      # Core types (Lead, Opportunity, Quote)
│   ├── sales_command_service.proto      # 18 RPCs (15 writes + 3 strong reads)
│   ├── sales_query_service.proto        # Read API (6 RPCs)
│   ├── intents.proto                    # Domain intents (15 variants)
│   ├── facts.proto                      # Domain facts (15 variants)
│   ├── opportunity.proto                # Opportunity aggregate
│   └── quote.proto                      # Quote aggregate
├── integration/v1/
│   └── integration_facts.proto          # Inbound events from other systems
└── process/v1/
    └── process_facts.proto              # Process orchestration events
```

**Buf Configs (5 files):**
- `buf.gen.domain-sales.local.yaml` - Domain primitive code generation
- `buf.gen.integration-sales.local.yaml` - Integration primitive code generation
- `buf.gen.process-sales.local.yaml` - Process primitive code generation
- `buf.gen.projection-sales.local.yaml` - Projection primitive code generation
- `buf.gen.uisurface-sales.local.yaml` - UI Surface primitive code generation

**Maturity:** PRODUCTION-READY

---

### Commerce Domain (9 files, 573 LOC)

**Location:** `packages/ameide_core_proto/src/ameide_core_proto/commerce/`

**File Structure:**
```
commerce/
├── v1/
│   ├── commerce_write_service.proto     # Write API (8 RPCs)
│   ├── commerce_query.proto             # Read API (3 RPCs)
│   ├── commerce_domain_messages.proto   # Domain events
│   ├── commerce_envelope.proto          # Message metadata
│   ├── commerce_storefront_domains.proto # Hostname claim/mapping
│   └── [supporting types]
├── integration/v1/
│   ├── commerce_hardware_integration.proto
│   ├── commerce_payments_integration.proto
│   └── commerce_replication.proto
└── agent/v1/
    └── commerce_assistant_agent.proto   # AI assistant service
```

**Buf Configs:** None (GAP)

**Maturity:** FOCUSED/PRODUCTION (narrow scope)

---

### Transformation Domain (12 files, 2,077 LOC)

**Location:** `packages/ameide_core_proto/src/ameide_core_proto/transformation/`

**File Structure:**
```
transformation/
├── core/v1/
│   └── transformation_core_common.proto           # Legacy/core meta (TransformationMessageMeta)
├── knowledge/v1/
│   ├── transformation_knowledge.proto             # Knowledge graph types (Element, Relationship, Node, etc.)
│   ├── transformation_knowledge_common.proto      # TransformationKnowledgeMessageMeta
│   ├── transformation_knowledge_facts.proto       # Facts (16 variants)
│   ├── transformation_knowledge_command_service.proto # Command service (15 RPCs)
│   ├── transformation_knowledge_query_service.proto   # Query service (15 RPCs)
│   └── view_layout.proto
└── scrum/v1/
    ├── transformation_scrum_artifacts.proto
    ├── transformation_scrum_common.proto    # ScrumMessageMeta
    ├── transformation_scrum_facts.proto     # Facts (14 variants)
    ├── transformation_scrum_intents.proto   # Intents (13 variants)
    └── transformation_scrum_query.proto     # 10 RPCs
```

**Buf Configs (1 file):**
- `buf.gen.domain-transformation.local.yaml` - Domain primitive with managed mode

**Maturity:** ADVANCED/SPECIALIZED

---

### SRE Domain (13 files, 1,178 LOC)

**Location:** `packages/ameide_core_proto/src/ameide_core_proto/sre/`

**File Structure:**
```
sre/
├── core/v1/
│   ├── sre.proto                        # Core SRE types
│   ├── alert.proto                      # Alert types
│   ├── facts.proto                      # Domain facts (26 variants)
│   ├── health.proto                     # Health check types
│   ├── incident.proto                   # Incident aggregate
│   ├── intents.proto                    # Domain intents (25 variants)
│   ├── integration_facts.proto          # Inbound events
│   ├── postmortem.proto                 # Postmortem types
│   ├── query.proto                      # Query services (7 services)
│   ├── runbook.proto                    # Runbook types
│   ├── service.proto                    # Service catalog types
│   └── slo.proto                        # SLO/SLI types
└── process/v1/
    └── process_facts.proto              # Process orchestration events
```

**Buf Configs:** None (GAP)

**Maturity:** PRODUCTION-READY

---

## 3) Service Architecture Comparison

### 3.1 Sales Domain - Clean CQRS

**SalesCommandService** (Write boundary + minimal strong reads, 18 RPCs)
```protobuf
service SalesCommandService {
  // Leads (3 operations)
  rpc CreateLead(CreateLeadRequest) returns (CreateLeadResponse);
  rpc QualifyLead(QualifyLeadRequest) returns (QualifyLeadResponse);
  rpc DisqualifyLead(DisqualifyLeadRequest) returns (DisqualifyLeadResponse);

  // Opportunities (5 operations)
  rpc CreateOpportunity(CreateOpportunityRequest) returns (CreateOpportunityResponse);
  rpc MoveOpportunityStage(MoveOpportunityStageRequest) returns (MoveOpportunityStageResponse);
  rpc AdjustOpportunityAmount(AdjustOpportunityAmountRequest) returns (AdjustOpportunityAmountResponse);
  rpc CloseOpportunityWon(CloseOpportunityWonRequest) returns (CloseOpportunityWonResponse);
  rpc CloseOpportunityLost(CloseOpportunityLostRequest) returns (CloseOpportunityLostResponse);

  // Quotes (7 operations)
  rpc CreateQuote(CreateQuoteRequest) returns (CreateQuoteResponse);
  rpc SubmitQuoteForApproval(SubmitQuoteForApprovalRequest) returns (SubmitQuoteForApprovalResponse);
  rpc ApproveQuote(ApproveQuoteRequest) returns (ApproveQuoteResponse);
  rpc RejectQuote(RejectQuoteRequest) returns (RejectQuoteResponse);
  rpc SendQuoteToCustomer(SendQuoteToCustomerRequest) returns (SendQuoteToCustomerResponse);
  rpc MarkQuoteAccepted(MarkQuoteAcceptedRequest) returns (MarkQuoteAcceptedResponse);
  rpc MarkQuoteDeclined(MarkQuoteDeclinedRequest) returns (MarkQuoteDeclinedResponse);

  // Minimal strong reads (get-by-id; projections provide browse/search)
  rpc GetLead(GetLeadRequest) returns (GetLeadResponse);
  rpc GetOpportunity(GetOpportunityRequest) returns (GetOpportunityResponse);
  rpc GetQuote(GetQuoteRequest) returns (GetQuoteResponse);
}
```

**SalesQueryService** (Read API, 6 RPCs)
```protobuf
service SalesQueryService {
  rpc GetOpportunityView(GetOpportunityViewRequest) returns (GetOpportunityViewResponse);
  rpc ListOpportunities(ListOpportunitiesRequest) returns (ListOpportunitiesResponse);
  rpc GetQuoteView(GetQuoteViewRequest) returns (GetQuoteViewResponse);
  rpc ListQuotes(ListQuotesRequest) returns (ListQuotesResponse);
  rpc GetPipelineSummary(GetPipelineSummaryRequest) returns (GetPipelineSummaryResponse);
  rpc ListApprovalWorkItems(ListApprovalWorkItemsRequest) returns (ListApprovalWorkItemsResponse);
}
```

**Pattern:** Explicit CQRS with separate command and query services

---

### 3.2 SRE Domain - Specialized Query Services

**IncidentService** (10 RPCs; mixed write + minimal strong reads)
```protobuf
service IncidentService {
  rpc CreateIncident(CreateIncidentRequest) returns (CreateIncidentResponse);
  rpc ReclassifyIncidentSeverity(ReclassifyIncidentSeverityRequest) returns (ReclassifyIncidentSeverityResponse);
  rpc AssignIncident(AssignIncidentRequest) returns (AssignIncidentResponse);
  rpc AddIncidentTimelineEntry(AddIncidentTimelineEntryRequest) returns (AddIncidentTimelineEntryResponse);
  rpc AcknowledgeIncident(AcknowledgeIncidentRequest) returns (AcknowledgeIncidentResponse);
  rpc TransitionIncidentStatus(TransitionIncidentStatusRequest) returns (TransitionIncidentStatusResponse);
  rpc ResolveIncident(ResolveIncidentRequest) returns (ResolveIncidentResponse);
  rpc CloseIncident(CloseIncidentRequest) returns (CloseIncidentResponse);

  rpc GetIncident(GetIncidentRequest) returns (GetIncidentResponse);
  rpc ListOpenIncidents(ListOpenIncidentsRequest) returns (ListOpenIncidentsResponse);
}
```

**Projection query services** (6 services / 7 RPCs total)
- `IncidentQueryService` (SearchIncidents, GetIncidentTimeline)
- `FleetHealthQueryService` (GetFleetHealth)
- `AlertQueryService` (ListActiveAlerts)
- `RunbookQueryService` (SearchRunbooks)
- `SLOQueryService` (GetSLO)
- `KnowledgeIndexQueryService` (SearchPatterns)

**Pattern:** Domain service keeps minimal reads; specialized query services provide projection reads by concern

---

### 3.3 Transformation Domain - Knowledge + Scrum

**Knowledge CQRS services**
- `TransformationKnowledgeCommandService` (15 RPCs)
- `TransformationKnowledgeQueryService` (15 RPCs)

**Scrum query service**
- `ScrumQueryService` (10 RPCs)

**Pattern:** Knowledge uses explicit command/query separation; Scrum uses bus intents/facts plus a read query service

---

### 3.4 Commerce Domain - Integration-Heavy

**CommerceStorefrontWriteService** (8 RPCs)
```protobuf
service CommerceStorefrontWriteService {
  rpc RequestHostnameClaim(RequestHostnameClaimRequest) returns (RequestHostnameClaimResponse);
  rpc VerifyHostnameClaim(VerifyHostnameClaimRequest) returns (VerifyHostnameClaimResponse);
  rpc FailHostnameClaimVerification(FailHostnameClaimVerificationRequest) returns (FailHostnameClaimVerificationResponse);

  rpc RequestHostnameMapping(RequestHostnameMappingRequest) returns (RequestHostnameMappingResponse);
  rpc ActivateHostnameMapping(ActivateHostnameMappingRequest) returns (ActivateHostnameMappingResponse);
  rpc FailHostnameMappingProvisioning(FailHostnameMappingProvisioningRequest) returns (FailHostnameMappingProvisioningResponse);

  rpc RevokeHostnameClaim(RevokeHostnameClaimRequest) returns (google.protobuf.Empty);
  rpc RevokeHostnameMapping(RevokeHostnameMappingRequest) returns (google.protobuf.Empty);
}
```

**CommerceQueryService** (3 RPCs)
```protobuf
service CommerceQueryService {
  rpc ResolveHostname(...) returns (...);
  rpc GetHostnameClaim(...) returns (...);
  rpc GetHostnameMapping(...) returns (...);
}
```

**Integration Services:**
- `CommercePaymentsIntegrationService` (Payment processing)
- `CommerceHardwareIntegrationService` (Hardware devices)
- `commerce_replication.proto` defines messages only (no service)
- `CommerceAssistantAgentService` (AI assistant)

**Pattern:** Write/query separation + specialized integration services

---

## 4) Event Sourcing Maturity Analysis

### 4.1 Event Metadata Comparison (Revised)

| Domain | Message ID | Correlation ID | Causation ID | Trace Context | Producer | Schema Version | Tenant ID |
|--------|-----------|----------------|--------------|---------------|----------|----------------|-----------|
| **Sales** | ✓ | ✓ | ✓ | ✓ (traceparent/state) | ✓ | ✓ | ✓ |
| **SRE** | ✓ | ✓ | ✓ | ✓ (traceparent/state) | ✓ | ✓ | ✓ |
| **Commerce** | ✓ | ✓ | ✓ | ✓ (traceparent/state) | ✓ | ✓ | ✓ (via scope) |
| **Transformation** | ✓ | ✓ | ✓ | ✓ (traceparent/state) | ✓ | ✓ | ✓ |

**Status:** ✅ ALL four domains now have complete event sourcing metadata.

### 4.2 Message Metadata Structures

**Sales Domain (SalesMessageMeta)**
```protobuf
message SalesMessageMeta {
  string message_id = 1;        // REQUIRED
  string correlation_id = 2;
  string causation_id = 3;
  google.protobuf.Timestamp occurred_at = 4;  // REQUIRED
  string producer = 5;          // REQUIRED
  string schema_version = 6;    // REQUIRED
  string traceparent = 20;      // W3C Trace Context
  string tracestate = 21;       // W3C Trace Context
  string tenant_id = 50;        // REQUIRED
}
```

**SRE Domain (SreMessageMeta)** - Identical structure to Sales with actor field

**Commerce Domain (CommerceMessageMeta)** - Similar structure with event_type/source/subject extensions

**Transformation Domain (TransformationMessageMeta)** - Present in `transformation/core/v1`:
```protobuf
message TransformationMessageMeta {
  string message_id = 1;        // REQUIRED
  string correlation_id = 2;
  string causation_id = 3;
  google.protobuf.Timestamp occurred_at = 4;  // REQUIRED
  string producer = 5;          // REQUIRED
  string schema_version = 6;
  string traceparent = 20;      // W3C Trace Context
  string tracestate = 21;
  string tenant_id = 50;        // REQUIRED
  TransformationSubject subject = 51;
  TransformationActor actor = 52;
}
```

**Transformation Knowledge Domain (TransformationKnowledgeMessageMeta)** - Used by emitted knowledge facts:

- See `packages/ameide_core_proto/src/ameide_core_proto/transformation/knowledge/v1/transformation_knowledge_common.proto`

**Status:** All domains implement consistent event sourcing metadata patterns.

---

### 4.3 Intent/Fact Symmetry

**Sales Domain - Symmetry (15:15)**

Intents:
- `SalesDomainIntent` (oneof with 15 intent variants)

Facts:
- `SalesDomainFact` (oneof with 15 fact variants)

**Note:** SalesCommandService also includes 3 strong reads (GetLead/GetOpportunity/GetQuote) that do not map to bus intents/facts.

**SRE Domain - Near Symmetry (25:26)**

Intents:
- `SreDomainIntent` (oneof with 25 intent variants)

Facts:
- `SreDomainFact` (oneof with 26 fact variants)

**Why asymmetric:** Some facts are emitted without a direct external “intent” equivalent (e.g., budget exhaustion).

**Commerce Domain - Perfect Symmetry (4:4)**

Intents:
- `CommerceDomainIntent` (oneof with 4 intent types)

Facts:
- `CommerceDomainFact` (oneof with 4 fact types)

**Transformation Domain - Mixed (by context)**

- **Knowledge:** facts only (16 variants); write surface is RPC-based (`TransformationKnowledgeCommandService`), not a bus intent envelope
- **Scrum:** 13 intent variants, 14 fact variants

**Pattern:** Intent/fact symmetry is strong in Sales/Commerce, but not universal across all domains/contexts.

---

### 4.4 Confluent Schema Registry Integration

**All domains use `buf/confluent/v1` subject annotations for schema naming:**

```protobuf
import "buf/confluent/v1/extensions.proto";

message SalesDomainIntent {
  option (buf.confluent.v1.subject) = {
    instance_name: "default"
    name: "sales.domain.intents.v1-value"
  };
}
```

**Stream References:**
- Sales: `sales.domain.intents.v1`, `sales.domain.facts.v1` (plus `ameide_core_proto.common.v1.eventing` stream_ref fields)
- Commerce: `commerce.domain.intents.v1`, `commerce.domain.facts.v1`
- Transformation: `transformation.knowledge.domain.facts.v1` (facts), `scrum.domain.intents.v1` (intents), `scrum.domain.facts.v1` (facts)
- SRE: `sre.domain.intents.v1`, `sre.domain.facts.v1`

**Partition / idempotency keys:**
- Sales and SRE explicitly declare these via `ameide_core_proto.common.v1.eventing` options.
- Commerce, Transformation Knowledge, and Scrum currently do not declare partition/idempotency rules via the shared eventing option (but their meta includes `message_id` and tenant scoping).

**Status:** Excellent consistency across all domains.

---

## 5) Data Model Sophistication

### 5.1 Enum Type Comparison

| Domain | Key Enums (examples) | Notes |
|--------|-----------------------|-------|
| **SRE** | `Severity`, `IncidentStatus`, `AlertStatus`, `HealthStatus`, `ServiceTier`, `PostmortemStatus` | Broadest enum surface across multiple subdomains |
| **Sales** | `LeadStatus`, `OpportunityStage`, `QuoteStatus` | Tight, workflow-centric enum set |
| **Transformation** | `ElementKind` (Knowledge) | Large enum set driving UI/editor mapping for repository elements |
| **Commerce** | `HostnameClaimStatus`, `HostnameMappingStatus`, `DomainVerificationMethod`, `DomainOnboardingErrorCode` | Status + operational error codes for onboarding |

---

### 5.2 Complex Type Patterns

**Sales Domain - Business Domain Types**
```protobuf
message Money {
  string currency = 1;  // ISO 4217 (e.g. "USD")
  int64 units = 2;
  int32 nanos = 3;
}

message LineItem {
  string sku = 1;
  string description = 2;
  int32 quantity = 3;
  Money unit_price = 4;
  Money total_price = 5;
}

message ActorRef {
  string user_id = 1;  // REQUIRED
  string display_name = 2;
}
```

**Transformation Knowledge Domain - Extensible Repository Types**
```protobuf
message Element {
  string id = 1;
  string tenant_id = 2;
  string organization_id = 3;
  string repository_id = 4;
  ElementKind kind = 5;
  string type_key = 7;
  google.protobuf.Any body = 8;          // Polymorphic payload
  map<string, string> metadata = 9;      // Extensible attributes
}
```

**SRE Domain - Observable Systems Types**
```protobuf
message Incident {
  string incident_id = 1;
  Severity severity = 2;
  IncidentStatus status = 3;
  repeated TimelineEntry timeline = 4;  // Immutable audit trail
  ResourceRef affected_resource = 5;
  SLOImpact slo_impact = 6;
}

message ResourceRef {
  string type = 1;        // service, cluster, namespace
  string id = 2;
  string namespace = 3;
  string cluster = 4;
  string environment = 5;
}
```

**Commerce Domain - Domain-Specific Types**
```protobuf
message StorefrontHostnameClaim {
  CommerceScope scope = 1;
  string hostname = 2;
  HostnameClaimStatus status = 4;
  DomainVerificationInstructions instructions = 5;
  DomainOnboardingError last_error = 6;
}
```

**Patterns:**
- Sales: Strong business domain types (Money, LineItem, Actor)
- Transformation: Extensibility via google.protobuf.Struct/Any
- SRE: Rich observable systems types (Incident, Alert, SLO, Timeline)
- Commerce: JSONB patterns for semi-structured data

---

### 5.3 Reference Type Patterns

**Sales Domain - Explicit References**
```protobuf
message AccountRef {
  string account_id = 1;
  string account_name = 2;
}

message ContactRef {
  string contact_id = 1;
  string contact_name = 2;
  string contact_email = 3;
}
```

**SRE Domain - Generic Resource References**
```protobuf
message ResourceRef {
  string type = 1;
  string id = 2;
  string namespace = 3;
  string cluster = 4;
  string environment = 5;
}
```

**Transformation Domain - Hierarchical References**
```protobuf
message ResourceMetadata {
  string id = 1;
  string name = 2;
  string namespace = 3;
  map<string, string> labels = 4;
  map<string, string> annotations = 5;
}
```

**Pattern Recommendation:** SRE's generic ResourceRef is most reusable; Sales' explicit refs provide better type safety.

---

## 6) Buf Configuration Analysis

### 6.1 Sales Domain - Most Complete (5 configs)

**buf.gen.domain-sales.local.yaml**
```yaml
version: v2
clean: true
inputs:
  - directory: src
plugins:
  - local: protoc-gen-ameide-register-go
    out: ../../primitives/domain/sales/internal/gen
    opt:
      - package=gen
      - output_file=domain_services.generated.go
      - register_func=RegisterDomainServices
      - interface_name=DomainServices
      - go_import_prefix=github.com/ameideio/ameide-sdk-go/gen/go
```

**buf.gen.integration-sales.local.yaml** - Integration primitive code generation
**buf.gen.process-sales.local.yaml** - Process primitive code generation
**buf.gen.projection-sales.local.yaml** - Projection primitive code generation
**buf.gen.uisurface-sales.local.yaml** - UI Surface primitive code generation

**Status:** Production-ready multi-primitive code generation

---

### 6.2 Transformation Domain - Managed Mode (1 config)

**buf.gen.domain-transformation.local.yaml**
```yaml
version: v2
managed:
  enabled: true
  override:
    - file_option: go_package_prefix
      value: github.com/ameideio/ameide-sdk-go/gen/go
clean: true
inputs:
  - directory: src
plugins:
  - local: protoc-gen-ameide-register-go
    out: ../../primitives/domain/transformation/internal/gen
    opt:
      - package=gen
      - output_file=domain_services.generated.go
      - register_func=RegisterDomainServices
      - interface_name=DomainServices
```

**Status:** Domain primitive only; needs process/projection/uisurface configs

---

### 6.3 Commerce & SRE Domains - No Buf Configs (GAP)

**Missing:**
- `buf.gen.domain-commerce.local.yaml`
- `buf.gen.domain-sre.local.yaml`
- Plus process/projection/integration/uisurface configs

**Impact:**
- No consistent code generation for primitives
- No protoc-gen-ameide-register-go integration
- Manual SDK generation required

**Recommendation:** Add buf configs following transformation's managed mode pattern.

---

### 6.4 Buf Configuration Standardization Proposal

**Template: buf.gen.domain-{domain}.local.yaml**
```yaml
version: v2
managed:
  enabled: true
  override:
    - file_option: go_package_prefix
      value: github.com/ameideio/ameide-core/primitives/domain/{domain}/internal/gen
plugins:
  - local: protoc-gen-ameide-register-go
    out: ../../primitives/domain/{domain}/internal/gen
    opt:
      - paths=source_relative
  - local: protoc-gen-go
    out: ../../primitives/domain/{domain}/internal/gen
    opt:
      - paths=source_relative
  - local: protoc-gen-go-grpc
    out: ../../primitives/domain/{domain}/internal/gen
    opt:
      - paths=source_relative
```

**Apply to:** Commerce, SRE (immediate); Process/Projection for all domains (medium-term)

---

## 7) API Design Quality Assessment

### 7.1 Pagination Patterns

**Sales Domain - Page size/token fields**
```protobuf
message ListOpportunitiesRequest {
  ameide_core_proto.common.v1.RequestContext context = 1;
  OpportunityStage stage = 2;
  string owner_user_id = 3;
  int32 page_size = 10;
  string page_token = 11;
}

message ListOpportunitiesResponse {
  repeated OpportunityView opportunities = 1;
  string next_page_token = 2;
}
```

**SRE Domain - Shared PaginationRequest/PaginationResponse**
```protobuf
message SearchIncidentsRequest {
  ameide_core_proto.common.v1.RequestContext context = 1;
  string query = 2;
  ameide_core_proto.common.v1.PaginationRequest pagination = 3;
}

message SearchIncidentsResponse {
  repeated Incident incidents = 1;
  ameide_core_proto.common.v1.PaginationResponse pagination = 2;
}
```

**Commerce Domain**
- No list/browse RPCs in `CommerceQueryService` (ResolveHostname/GetHostnameClaim/GetHostnameMapping), so pagination patterns are currently not exercised in the domain query surface.

**Transformation Knowledge Domain - Shared PaginationRequest/PaginationResponse**
```protobuf
message ListRepositoriesRequest {
  ameide_core_proto.common.v1.RequestContext context = 1;
  string tenant_id = 2;
  ameide_core_proto.common.v1.PaginationRequest pagination = 3;
}
```

**Status:** Pagination patterns are not fully standardized (Sales uses page_size/page_token fields; SRE/Transformation Knowledge use shared pagination messages).

---

### 7.2 Error Handling Patterns

**Commerce Domain - Domain-specific error codes (10 values incl UNSPECIFIED)**
```protobuf
enum DomainOnboardingErrorCode {
  DOMAIN_ONBOARDING_ERROR_CODE_UNSPECIFIED = 0;
  DOMAIN_ONBOARDING_ERROR_CODE_DNS_TARGET_INCORRECT = 1;
  DOMAIN_ONBOARDING_ERROR_CODE_TXT_MISSING = 2;
  DOMAIN_ONBOARDING_ERROR_CODE_TXT_MISMATCH = 3;
  DOMAIN_ONBOARDING_ERROR_CODE_PROPAGATION_PENDING = 4;
  DOMAIN_ONBOARDING_ERROR_CODE_CAA_BLOCKED = 10;
  DOMAIN_ONBOARDING_ERROR_CODE_CERT_ISSUE_RATE_LIMIT = 11;
  DOMAIN_ONBOARDING_ERROR_CODE_CERT_ISSUE_FAILED = 12;
  DOMAIN_ONBOARDING_ERROR_CODE_HOSTNAME_ALREADY_CLAIMED = 20;
  DOMAIN_ONBOARDING_ERROR_CODE_CDN_INTERFERENCE = 21;
}
```

**Sales/SRE/Transformation Domains - Standard gRPC Status Codes**
- Use `google.rpc.Status` for errors
- No domain-specific error enums

**Pattern:** Commerce has most sophisticated domain error handling; others rely on standard gRPC patterns.

---

### 7.3 Field Mask Support

**Transformation Knowledge Domain - Field Mask for Partial Updates**
```protobuf
import "google/protobuf/field_mask.proto";

message EditRepositoryRequest {
  ameide_core_proto.common.v1.RequestContext context = 1;
  Repository repository = 2;
  google.protobuf.FieldMask update_mask = 3;
}
```

**Transformation Scrum Domain - Field Mask Support**
- `RefineProductBacklogItemRequested` includes `google.protobuf.FieldMask update_mask`

**Sales/Commerce/SRE Domains**
- Full replacement updates only
- No partial update capability

**Recommendation:** Add field mask support to Sales/Commerce/SRE for partial updates.

---

## 8) Critical Gaps Summary (Revised)

### 8.1 By Domain

**Sales Domain**
- ✅ Strong CQRS contract (command + query services) and complete metadata in bus envelopes
- ✅ Most complete buf config coverage (5 files)
- Gaps:
  - Pagination style differs from SRE/Transformation Knowledge (Sales uses `page_size`/`page_token` fields vs shared pagination messages)
  - No field mask support in Sales command RPCs (edits are modeled as intent-style commands)

**Commerce Domain**
1. Missing `buf.gen.domain-commerce.local.yaml`
2. Query surface does not currently expose list/browse RPCs (so pagination patterns are not exercised yet)
3. Limited to hostname management (lacks broader commerce scope)

**Transformation Domain**
- ✅ Knowledge context provides explicit CQRS (command + query services) and uses `TransformationKnowledgeMessageMeta` on emitted facts
- ✅ Largest current RPC surface (40 RPCs across 3 services)
- Gaps:
  - Multiple metadata types across contexts (`TransformationMessageMeta`, `TransformationKnowledgeMessageMeta`, `ScrumMessageMeta`)
  - Knowledge writes are RPC-based (no bus intent envelope), while Scrum uses bus intents/facts; eventing conventions differ by context
  - Missing buf configs for non-domain primitives (process/projection/integration/uisurface) if generation glue is desired

**SRE Domain**
1. Missing `buf.gen.domain-sre.local.yaml`
2. Missing buf configs for process/projection/integration primitives

---

### 8.2 Cross-Domain Gaps (Revised)

1. **Buf Configuration Inconsistency**: Only 2/4 domains have buf configs (Sales has 5, Transformation has 1)
2. **Event Metadata Standardization**: Metadata fields are broadly aligned, but types and conventions vary (notably across Transformation contexts)
3. **Pagination Consistency**: Pagination is not standardized (Sales uses `page_size/page_token`; SRE/Transformation Knowledge use `PaginationRequest/Response`)
4. **Field Mask Support**: Only Transformation Knowledge + Scrum use field masks for partial updates
5. **Error Handling**: Inconsistent approach (Commerce uses error enums; others rely on gRPC Status)

---

## 9) Recommendations

### 9.1 Immediate Actions (Priority 1)

1. **Add Missing Buf Configs**
   - Create `buf.gen.domain-commerce.local.yaml` (follow transformation managed mode)
   - Create `buf.gen.domain-sre.local.yaml` (follow transformation managed mode)
   - Target: `packages/ameide_core_proto/`

2. **Standardize Pagination Conventions**
   - Decide between `page_size/page_token` fields (Sales pattern) vs shared `PaginationRequest/Response` (SRE/Transformation Knowledge pattern)
   - Apply consistently for new list/search RPCs

3. **Clarify Intent/Fact Strategy Per Context**
   - Decide whether “knowledge writes” should also have a bus intent envelope (like Sales/SRE/Scrum), or remain RPC-only with facts

---

### 9.2 Medium-Term Enhancements (Priority 2)

1. **Standardize CQRS Separation Where Needed**
   - Keep explicit CQRS for Knowledge; evaluate whether SRE’s `IncidentService` should remain mixed read/write or be split

2. **Add Field Mask Support**
   - Sales: Add `google.protobuf.FieldMask` to update operations
   - Commerce: Add field mask support for partial updates
   - SRE: Add field mask support for incident updates

3. **Multi-Primitive Buf Configs**
   - Add process/projection/integration/uisurface buf configs for all domains
   - Follow Sales 4-config pattern

4. **Error Handling Standardization**
   - Decide: Domain-specific error enums (Commerce pattern) vs gRPC Status codes (Sales/SRE/Transformation pattern)
   - Apply consistently across all domains

---

### 9.3 Long-Term Strategic (Priority 3)

1. **Proto Style Guide**
   - Document pagination patterns
   - Document error handling patterns
   - Document event sourcing metadata requirements
   - Document CQRS separation guidelines

2. **SDK Generation Pipeline**
   - Automated SDK generation for Go, TypeScript, Python
   - Version management and publishing
   - Breaking change detection

3. **Proto Linting and Validation**
   - Enforce event sourcing metadata in all domains
   - Enforce pagination in all list operations
   - Enforce consistent naming conventions
   - Enforce CQRS separation

4. **Cross-Domain Proto Sharing**
   - Extract common types (Money, ActorRef, ResourceRef) to shared package
   - Reduce duplication across domains
   - Maintain backward compatibility

---

## 10) Critical Files Reference

### Proto Contract Files

**Sales:**
- Command Service: `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/sales_command_service.proto`
- Query Service: `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/sales_query_service.proto`
- Intents: `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/intents.proto`
- Facts: `packages/ameide_core_proto/src/ameide_core_proto/sales/core/v1/facts.proto`

**Commerce:**
- Write Service: `packages/ameide_core_proto/src/ameide_core_proto/commerce/v1/commerce_write_service.proto`
- Query Service: `packages/ameide_core_proto/src/ameide_core_proto/commerce/v1/commerce_query.proto`
- Domain Messages: `packages/ameide_core_proto/src/ameide_core_proto/commerce/v1/commerce_domain_messages.proto`

**Transformation:**
- Knowledge Command Service: `packages/ameide_core_proto/src/ameide_core_proto/transformation/knowledge/v1/transformation_knowledge_command_service.proto`
- Knowledge Query Service: `packages/ameide_core_proto/src/ameide_core_proto/transformation/knowledge/v1/transformation_knowledge_query_service.proto`
- Knowledge Facts: `packages/ameide_core_proto/src/ameide_core_proto/transformation/knowledge/v1/transformation_knowledge_facts.proto`
- Scrum Intents: `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/transformation_scrum_intents.proto`

**SRE:**
- Incident + Query Services: `packages/ameide_core_proto/src/ameide_core_proto/sre/core/v1/query.proto`
- Intents: `packages/ameide_core_proto/src/ameide_core_proto/sre/core/v1/intents.proto`
- Facts: `packages/ameide_core_proto/src/ameide_core_proto/sre/core/v1/facts.proto`
- Query Services: `packages/ameide_core_proto/src/ameide_core_proto/sre/core/v1/query.proto`

### Buf Configuration Files

**Sales (5 files - most complete):**
- Domain: `packages/ameide_core_proto/buf.gen.domain-sales.local.yaml`
- Integration: `packages/ameide_core_proto/buf.gen.integration-sales.local.yaml`
- Process: `packages/ameide_core_proto/buf.gen.process-sales.local.yaml`
- Projection: `packages/ameide_core_proto/buf.gen.projection-sales.local.yaml`
- UISurface: `packages/ameide_core_proto/buf.gen.uisurface-sales.local.yaml`

**Transformation (1 file):**
- Domain: `packages/ameide_core_proto/buf.gen.domain-transformation.local.yaml`

**Commerce & SRE:**
- Missing (GAP - needs domain configs)

---

## 11) Conclusion (Revised December 2025)

The four domain proto contracts demonstrate varying levels of API design maturity:

- **Transformation** currently leads in RPC surface (40 RPCs across Knowledge command/query + Scrum query) and uses field masks in Knowledge/Scrum, but mixes conventions across contexts
- **SRE** has the most mature multi-service query surface (7 services, 17 RPCs) and complete message metadata, but intent/fact symmetry is not perfect (25 intents vs 26 facts)
- **Sales** leads in codegen readiness (5 buf configs), clean CQRS separation (2 services / 24 RPCs), and symmetric 15 intent/fact design
- **Commerce** remains focused (573 LOC, BYOD storefront) with domain-specific error codes (10 values) but no buf configs and a narrow query surface

Primary remaining gaps:
1. **Buf configuration** for Commerce and SRE (blocks consistent codegen glue)
2. **Pagination standardization** (Sales differs from SRE/Transformation Knowledge)
3. **Consistency within Transformation** (RPC vs bus conventions, metadata types, and shared eventing options)

**This analysis complements `550-comparative-domain.md`** (which focuses on implementation maturity) by providing a contract-layer view of API design, event sourcing, and code generation readiness across all four domains.
