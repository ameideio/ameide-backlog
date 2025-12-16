# 552 — Comparative Proto Contracts Analysis

**Status:** Analysis Complete
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

### Overall Proto Maturity Ranking

1. **Sales** (PRODUCTION-READY) - Clean CQRS, 4 buf configs, symmetric intent/fact, comprehensive command surface
2. **SRE** (PRODUCTION-READY) - 7 query services, complete event sourcing metadata, most mature CQRS
3. **Transformation** (ADVANCED/SPECIALIZED) - Most complex (1,873 LOC), 3 bounded contexts, extensible data model
4. **Commerce** (FOCUSED/PRODUCTION) - Narrow but complete (604 LOC), 5 services, integration-heavy

### Proto Complexity Comparison

| Domain | Proto Files | Lines of Code | Services | Intents | Facts | Bounded Contexts |
|--------|------------|---------------|----------|---------|-------|------------------|
| **Transformation** | 16 | 1,873 | 4 | 29 | 29 | 3 (Core, Architecture, Scrum) |
| **SRE** | 13 | 1,178 | 7 | 18 | 18 | 1 (Observable Systems) |
| **Sales** | 9 | 897 | 2 | 14 | 14 | 1 (Sales Lifecycle) |
| **Commerce** | 9 | 604 | 5 | 4 | 4 | 1 (Storefront BYOD) |

### Key Findings

1. **Event Sourcing Maturity**: SRE and Sales have complete event sourcing metadata (message_id, correlation_id, causation_id, traceparent); Transformation lacks these critical fields
2. **CQRS Separation**: Sales has explicit command/query separation; SRE has 7 specialized query services; Transformation and Commerce use combined services
3. **Code Generation**: Sales has 4 buf configs (domain, process, projection, uisurface); Transformation has 1 (domain); Commerce and SRE have none
4. **API Surface**: Sales has most comprehensive write API (14 RPCs); SRE has most comprehensive read API (7 query services)
5. **Buf Configuration Gap**: Commerce and SRE need domain buf configs for consistent code generation

## 2) Proto Files Inventory

### Sales Domain (9 files, 897 LOC)

**Location:** `packages/ameide_core_proto/src/ameide_core_proto/sales/`

**File Structure:**
```
sales/
├── core/v1/
│   ├── sales.proto                      # Core types (Lead, Opportunity, Quote)
│   ├── sales_command_service.proto      # Write API (14 RPCs)
│   ├── sales_query_service.proto        # Read API (6 RPCs)
│   ├── intents.proto                    # Commands (14 intent types)
│   ├── facts.proto                      # Events (14 fact types)
│   ├── opportunity.proto                # Opportunity aggregate
│   └── quote.proto                      # Quote aggregate
├── integration/v1/
│   └── integration_facts.proto          # Inbound events from other systems
└── process/v1/
    └── process_facts.proto              # Process orchestration events
```

**Buf Configs (4 files):**
- `buf.gen.domain-sales.local.yaml` - Domain primitive code generation
- `buf.gen.process-sales.local.yaml` - Process primitive code generation
- `buf.gen.projection-sales.local.yaml` - Projection primitive code generation
- `buf.gen.uisurface-sales.local.yaml` - UI Surface primitive code generation

**Maturity:** PRODUCTION-READY

---

### Commerce Domain (9 files, 604 LOC)

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

### Transformation Domain (16 files, 1,873 LOC)

**Location:** `packages/ameide_core_proto/src/ameide_core_proto/transformation/`

**File Structure:**
```
transformation/
├── v1/
│   ├── transformation.proto             # Core transformation types
│   └── transformation_service.proto     # Combined API (11 RPCs)
├── core/v1/
│   └── transformation_core_common.proto # Shared types
├── architecture/v1/
│   ├── transformation_architecture_facts.proto
│   ├── transformation_architecture_intents.proto
│   └── transformation_architecture_query.proto
├── repository/v1/
│   ├── transformation_repository_archimate.proto
│   ├── transformation_repository_baselines.proto
│   ├── transformation_repository_bpmn.proto
│   ├── transformation_repository_definitions.proto
│   └── transformation_repository_evidence.proto
└── scrum/v1/
    ├── transformation_scrum_artifacts.proto
    ├── transformation_scrum_common.proto
    ├── transformation_scrum_facts.proto
    ├── transformation_scrum_intents.proto
    └── transformation_scrum_query.proto
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
│   ├── facts.proto                      # Events (18 fact types)
│   ├── health.proto                     # Health check types
│   ├── incident.proto                   # Incident aggregate
│   ├── intents.proto                    # Commands (18 intent types)
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

**SalesCommandService** (Write API, 14 RPCs)
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

**Write Service** (Combined, 8 RPCs)
```protobuf
service IncidentService {
  rpc CreateIncident(...) returns (...);
  rpc ClassifyIncident(...) returns (...);
  rpc AssignIncident(...) returns (...);
  rpc AddIncidentTimelineEntry(...) returns (...);
  rpc TransitionIncidentStatus(...) returns (...);
  rpc ResolveIncident(...) returns (...);
  rpc CloseIncident(...) returns (...);
  rpc AcknowledgeIncident(...) returns (...);
}
```

**Read Services** (7 specialized query services)
```protobuf
service IncidentQueryService {
  rpc GetIncident(...) returns (...);
  rpc ListIncidents(...) returns (...);
}

service FleetHealthQueryService {
  rpc GetFleetHealth(...) returns (...);
  rpc ListFleetClusters(...) returns (...);
}

service AlertQueryService {
  rpc GetAlert(...) returns (...);
  rpc ListAlerts(...) returns (...);
}

service RunbookQueryService {
  rpc GetRunbook(...) returns (...);
  rpc ListRunbooks(...) returns (...);
}

service SLOQueryService {
  rpc GetSLO(...) returns (...);
  rpc ListSLOs(...) returns (...);
}

service KnowledgeIndexQueryService {
  rpc SearchKnowledgeBase(...) returns (...);
}
```

**Pattern:** Most mature CQRS with 7 specialized query services by concern

---

### 3.3 Transformation Domain - Combined Service

**TransformationService** (Combined, 11 RPCs)
```protobuf
service TransformationService {
  // Read operations
  rpc ListTransformations(...) returns (...);
  rpc GetTransformation(...) returns (...);
  rpc GetWorkspace(...) returns (...);
  rpc ListElements(...) returns (...);
  rpc GetMetricsSnapshot(...) returns (...);
  rpc ListAlerts(...) returns (...);

  // Write operations
  rpc CreateTransformation(...) returns (...);
  rpc ReviseTransformation(...) returns (...);
  rpc CreateMilestone(...) returns (...);
  rpc UpdateMilestoneProgress(...) returns (...);
}
```

**Additional Services:**
- `TransformationArchitectureWriteService` (1 RPC: SubmitIntent)
- `TransformationArchitectureQueryService` (Architecture queries)
- `ScrumQueryService` (Scrum artifact queries)

**Pattern:** Combined read/write in main service, separate query services for specialized contexts

---

### 3.4 Commerce Domain - Integration-Heavy

**CommerceStorefrontWriteService** (8 RPCs)
```protobuf
service CommerceStorefrontWriteService {
  rpc RequestHostnameClaim(...) returns (...);
  rpc VerifyHostnameOwnership(...) returns (...);
  rpc CreateHostnameMapping(...) returns (...);
  rpc ActivateHostnameMapping(...) returns (...);
  rpc RevokeHostnameClaim(...) returns (...);
  rpc RevokeHostnameMapping(...) returns (...);
  // ... additional operations
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
- `CommerceAssistantAgentService` (AI assistant)

**Pattern:** Write/query separation + specialized integration services

---

## 4) Event Sourcing Maturity Analysis

### 4.1 Event Metadata Comparison

| Domain | Message ID | Correlation ID | Causation ID | Trace Context | Producer | Schema Version |
|--------|-----------|----------------|--------------|---------------|----------|----------------|
| **Sales** | ✓ | ✓ | ✓ | ✓ (traceparent/state) | ✓ | ✓ |
| **SRE** | ✓ | ✓ | ✓ | ✓ (traceparent/state) | ✓ | ✓ |
| **Commerce** | ✓ | ✓ | ✓ | ✓ (traceparent/state) | ✓ | ✓ |
| **Transformation** | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |

**Critical Gap:** Transformation domain proto lacks event sourcing metadata in message definitions.

### 4.2 Message Metadata Structures

**Sales Domain (SalesMessageMeta)**
```protobuf
message SalesMessageMeta {
  string message_id = 1;        // REQUIRED, UUID, idempotency key
  string correlation_id = 2;    // Request tracing
  string causation_id = 3;      // Event causality chain
  google.protobuf.Timestamp occurred_at = 4;  // REQUIRED
  string producer = 5;          // REQUIRED, source service identifier
  string schema_version = 6;    // REQUIRED, version compatibility
  string traceparent = 7;       // W3C Trace Context
  string tracestate = 8;        // W3C Trace Context
  string tenant_id = 9;         // REQUIRED, multi-tenancy
}
```

**SRE Domain (SreMessageMeta)** - Identical structure to Sales

**Commerce Domain (CommerceMessageMeta)** - Identical structure to Sales/SRE

**Transformation Domain** - Missing structured metadata message

**Recommendation:** Add `TransformationMessageMeta` following Sales/SRE/Commerce pattern.

---

### 4.3 Intent/Fact Symmetry

**Sales Domain - Perfect Symmetry (14:14)**

Intents:
- `SalesDomainIntent` (oneof with 14 intent types)

Facts:
- `SalesDomainFact` (oneof with 14 fact types)

**Mapping:**
- `CreateLeadRequested` → `LeadCreated`
- `QualifyLeadRequested` → `LeadQualified`
- `DisqualifyLeadRequested` → `LeadDisqualified`
- (11 more mappings...)

**SRE Domain - Perfect Symmetry (18:18)**

Intents:
- `SreDomainIntent` (oneof with 18 intent types)

Facts:
- `SreDomainFact` (oneof with 18 fact types)

**Mapping:**
- `CreateIncidentRequested` → `IncidentCreated`
- `ClassifyIncidentRequested` → `IncidentClassified`
- (16 more mappings...)

**Commerce Domain - Perfect Symmetry (4:4)**

Intents:
- `CommerceDomainIntent` (oneof with 4 intent types)

Facts:
- `CommerceDomainFact` (oneof with 4 fact types)

**Transformation Domain - Perfect Symmetry (29:29)**

Architecture Intents/Facts:
- 12 architecture intent types
- 12 architecture fact types

Scrum Intents/Facts:
- 17 scrum intent types
- 17 scrum fact types

**Pattern:** All domains maintain 1:1 intent-to-fact symmetry (excellent design).

---

### 4.4 Confluent Schema Registry Integration

**All domains use buf/confluent/v1 extensions:**

```protobuf
import "buf/confluent/v1/extensions.proto";

option (buf.confluent.v1.file_reference) = {
  schema_reference: {
    name: "sales.domain.intents.v1"
    subject_name_strategy: TOPIC_NAME_STRATEGY
  }
};
```

**Stream References:**
- Sales: `sales.domain.intents.v1`, `sales.domain.facts.v1`
- Commerce: `commerce.domain.intents.v1`, `commerce.domain.facts.v1`
- Transformation: `transformation.architecture.intents.v1`, `transformation.architecture.facts.v1`, `transformation.scrum.intents.v1`, `transformation.scrum.facts.v1`
- SRE: `sre.domain.intents.v1`, `sre.domain.facts.v1`

**Partition Keys:** All domains use `meta.tenant_id` + aggregate identity
**Idempotency Keys:** All domains use `meta.message_id`

**Status:** Excellent consistency across all domains.

---

## 5) Data Model Sophistication

### 5.1 Enum Type Comparison

| Domain | Key Enums | States/Values |
|--------|----------|---------------|
| **SRE** | Severity (4), IncidentStatus (6), AlertStatus (4), HealthStatus (4), ServiceTier (4), PostmortemStatus (3) | 6 enums, 25 total values |
| **Sales** | LeadStatus (4), OpportunityStage (6), QuoteStatus (8) | 3 enums, 18 total values |
| **Transformation** | TransformationStage (6), TransformationHealth (4), TransformationCadence (3) | 3 enums, 13 total values |
| **Commerce** | HostnameClaimStatus (~4), HostnameMappingStatus (~4), DomainVerificationMethod (~3) | ~4 enums, ~11 total values |

**Winner:** SRE (most comprehensive enum system for observable systems domain)

---

### 5.2 Complex Type Patterns

**Sales Domain - Business Domain Types**
```protobuf
message Money {
  string currency_code = 1;  // ISO 4217
  int64 amount_cents = 2;    // Cents/smallest unit
}

message LineItem {
  string product_id = 1;
  string product_name = 2;
  int32 quantity = 3;
  Money unit_price = 4;
  Money line_total = 5;
}

message ActorRef {
  string user_id = 1;
  string display_name = 2;
}
```

**Transformation Domain - Extensible Repository Types**
```protobuf
message Transformation {
  ResourceMetadata metadata = 1;
  TransformationSpec spec = 2;
  google.protobuf.Struct attributes = 3;  // Extensible
  repeated Milestone milestones = 4;
  repeated MetricSnapshot metrics = 5;
}

message ArchiMateModel {
  string model_id = 1;
  google.protobuf.Any payload = 2;  // Polymorphic
  repeated ArchiMateElement elements = 3;
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
  string claim_id = 1;
  string hostname = 2;
  HostnameClaimStatus status = 3;
  DomainVerificationMethod verification_method = 4;
  google.protobuf.Struct dns_instructions = 5;  // JSONB
  google.protobuf.Struct error_details = 6;     // JSONB
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

### 6.1 Sales Domain - Most Complete (4 configs)

**buf.gen.domain-sales.local.yaml**
```yaml
version: v2
plugins:
  - local: protoc-gen-ameide-register-go
    out: ../../primitives/domain/sales/internal/gen
    opt:
      - paths=source_relative
```

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
      value: github.com/ameideio/ameide-core/primitives/domain/transformation/internal/gen
plugins:
  - local: protoc-gen-ameide-register-go
    out: ../../primitives/domain/transformation/internal/gen
    opt:
      - paths=source_relative
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

**Sales Domain - Consistent Pagination**
```protobuf
message ListOpportunitiesRequest {
  string tenant_id = 1;
  int32 page_size = 2;
  string page_token = 3;
  OpportunityFilter filter = 4;
}

message ListOpportunitiesResponse {
  repeated OpportunityView opportunities = 1;
  string next_page_token = 2;
  int32 total_count = 3;
}
```

**SRE Domain - Consistent Pagination**
```protobuf
message ListIncidentsRequest {
  string tenant_id = 1;
  int32 page_size = 2;
  string page_token = 3;
  IncidentFilter filter = 4;
}

message ListIncidentsResponse {
  repeated Incident incidents = 1;
  string next_page_token = 2;
}
```

**Commerce Domain - No Pagination (GAP)**
```protobuf
message ListHostnameClaimsRequest {
  string tenant_id = 1;
  // Missing: page_size, page_token
}

message ListHostnameClaimsResponse {
  repeated HostnameClaim claims = 1;
  // Missing: next_page_token
}
```

**Transformation Domain - Has Pagination**
```protobuf
message ListElementsRequest {
  string transformation_id = 1;
  int32 page_size = 2;
  string page_token = 3;
}
```

**Status:** Sales and SRE have consistent pagination; Commerce needs pagination; Transformation has partial pagination.

---

### 7.2 Error Handling Patterns

**Commerce Domain - Most Comprehensive (17 error codes)**
```protobuf
enum DomainOnboardingErrorCode {
  DOMAIN_ONBOARDING_ERROR_CODE_UNSPECIFIED = 0;
  DOMAIN_ONBOARDING_ERROR_CODE_DNS_VERIFICATION_FAILED = 1;
  DOMAIN_ONBOARDING_ERROR_CODE_CERTIFICATE_PROVISIONING_FAILED = 2;
  DOMAIN_ONBOARDING_ERROR_CODE_RATE_LIMIT_EXCEEDED = 3;
  DOMAIN_ONBOARDING_ERROR_CODE_HOSTNAME_IN_USE = 4;
  DOMAIN_ONBOARDING_ERROR_CODE_INVALID_HOSTNAME = 5;
  // ... 12 more error codes
}
```

**Sales/SRE/Transformation Domains - Standard gRPC Status Codes**
- Use `google.rpc.Status` for errors
- No domain-specific error enums

**Pattern:** Commerce has most sophisticated domain error handling; others rely on standard gRPC patterns.

---

### 7.3 Field Mask Support

**Transformation Domain - Field Mask for Partial Updates**
```protobuf
import "google/protobuf/field_mask.proto";

message ReviseTransformationRequest {
  string transformation_id = 1;
  Transformation transformation = 2;
  google.protobuf.FieldMask update_mask = 3;  // Partial update support
}
```

**Sales/Commerce/SRE Domains - No Field Mask Support**
- Full replacement updates only
- No partial update capability

**Recommendation:** Add field mask support to Sales/Commerce/SRE for partial updates.

---

## 8) Critical Gaps Summary

### 8.1 By Domain

**Sales Domain**
- ✅ No critical proto gaps (production-ready)
- Enhancement: Add field mask support for partial updates

**Commerce Domain**
1. Missing `buf.gen.domain-commerce.local.yaml`
2. No pagination in list operations
3. Limited to hostname management (lacks broader commerce scope)

**Transformation Domain**
1. Missing event sourcing metadata (message_id, correlation_id, causation_id)
2. No explicit CQRS separation (combined read/write service)
3. Missing buf configs for process/projection/integration/uisurface primitives

**SRE Domain**
1. Missing `buf.gen.domain-sre.local.yaml`
2. Missing buf configs for process/projection/integration primitives

---

### 8.2 Cross-Domain Gaps

1. **Buf Configuration Inconsistency**: Only 2/4 domains have buf configs
2. **Event Metadata Standardization**: Transformation lacks event sourcing metadata
3. **Pagination Consistency**: Commerce lacks pagination patterns
4. **Field Mask Support**: Only Transformation supports partial updates
5. **Error Handling**: Inconsistent approach (Commerce uses enums, others use gRPC Status)

---

## 9) Recommendations

### 9.1 Immediate Actions (Priority 1)

1. **Add Missing Buf Configs**
   - Create `buf.gen.domain-commerce.local.yaml` (follow transformation managed mode)
   - Create `buf.gen.domain-sre.local.yaml` (follow transformation managed mode)
   - Target: `packages/ameide_core_proto/`

2. **Add Event Sourcing Metadata to Transformation**
   - Create `TransformationMessageMeta` message following Sales/SRE/Commerce pattern
   - Add to all intent and fact messages
   - File: `packages/ameide_core_proto/src/ameide_core_proto/transformation/core/v1/transformation_core_common.proto`

3. **Add Pagination to Commerce**
   - Add `page_size`, `page_token` to list requests
   - Add `next_page_token` to list responses
   - Follow Sales/SRE pagination pattern

---

### 9.2 Medium-Term Enhancements (Priority 2)

1. **Standardize CQRS Separation**
   - Transformation: Split combined service into command + query services
   - Follow Sales pattern (explicit command/query separation) or SRE pattern (specialized query services)

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
- Main Service: `packages/ameide_core_proto/src/ameide_core_proto/transformation/v1/transformation_service.proto`
- Architecture Intents: `packages/ameide_core_proto/src/ameide_core_proto/transformation/architecture/v1/transformation_architecture_intents.proto`
- Scrum Intents: `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/transformation_scrum_intents.proto`

**SRE:**
- Incident Service: `packages/ameide_core_proto/src/ameide_core_proto/sre/core/v1/sre_service.proto`
- Intents: `packages/ameide_core_proto/src/ameide_core_proto/sre/core/v1/intents.proto`
- Facts: `packages/ameide_core_proto/src/ameide_core_proto/sre/core/v1/facts.proto`
- Query Services: `packages/ameide_core_proto/src/ameide_core_proto/sre/core/v1/query.proto`

### Buf Configuration Files

**Sales:**
- Domain: `packages/ameide_core_proto/buf.gen.domain-sales.local.yaml`
- Process: `packages/ameide_core_proto/buf.gen.process-sales.local.yaml`
- Projection: `packages/ameide_core_proto/buf.gen.projection-sales.local.yaml`
- UISurface: `packages/ameide_core_proto/buf.gen.uisurface-sales.local.yaml`

**Transformation:**
- Domain: `packages/ameide_core_proto/buf.gen.domain-transformation.local.yaml`

**Commerce & SRE:**
- Missing (GAP)

---

## 11) Conclusion

The four domain proto contracts demonstrate varying levels of API design maturity:

- **Sales** leads in code generation readiness (4 buf configs), clean CQRS separation, and comprehensive command surface (14 RPCs)
- **SRE** has the most mature CQRS implementation (7 specialized query services) and complete event sourcing metadata
- **Transformation** has the most sophisticated data model (1,873 LOC, 3 bounded contexts, extensibility patterns) but lacks event sourcing metadata
- **Commerce** has the narrowest but most focused scope (604 LOC, BYOD storefront) with excellent error handling (17 error codes)

All domains maintain excellent intent/fact symmetry and Confluent schema registry integration. The primary gaps are:

1. **Buf configuration** for Commerce and SRE (blocks consistent code generation)
2. **Event sourcing metadata** for Transformation (blocks proper event tracing)
3. **Pagination** for Commerce (limits scalability)
4. **CQRS standardization** across Transformation and Commerce

### Key Insight: Proto Layer is Production-Ready for Sales/SRE, Needs Standardization for Commerce/Transformation

The analysis reveals Sales and SRE have production-ready proto contracts with complete event sourcing, CQRS patterns, and code generation infrastructure. Commerce and Transformation need buf configs, event metadata standardization, and API consistency improvements to match Sales/SRE maturity.

**This analysis complements `550-comparative-domain.md`** (which focuses on implementation maturity) by providing a contract-layer view of API design, event sourcing, and code generation readiness across all four domains.
