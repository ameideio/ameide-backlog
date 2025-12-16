# 523 Commerce — Proto proposal (communication between primitives)

**Status:** Implemented (v1 commerce protos + write service + agent surface)

This document proposes a proto shape for commerce that makes the communication topology explicit, 496-native, and ArchiMate-aligned (Application Services/Interfaces/Events realized by Ameide primitives as Application Components).

- commands/intents vs facts,
- domain facts vs process facts,
- required metadata for traceability and tenant isolation,
- how Domain/Process/Projection/Integration/UISurface/Agent communicate.

It complements:

- `496-eda-principles.md` (EDA invariants)
- `509-proto-naming-conventions.md` (package/topic conventions)
- `523-commerce*.md` (commerce decomposition)
- `525-it4it-value-stream-mapping.md` (IT4IT value-stream lens)

Phase 1 focus (v1): **public storefront domains + BYOD onboarding**.

Implementation status:

- Implemented in `packages/ameide_core_proto/src/ameide_core_proto/commerce/v1/` (including `commerce_write_service.proto`), `packages/ameide_core_proto/src/ameide_core_proto/commerce/agent/v1/`, `packages/ameide_core_proto/src/ameide_core_proto/commerce/integration/v1/`, and `packages/ameide_core_proto/src/ameide_core_proto/process/commerce/v1/`.

## Implementation progress (current)

Delivered (v1 BYOD slice):

- [x] Commerce Domain intents/facts and Commerce Process facts carry the shared `common.v1.eventing` options spine (stable type + stream ref + schema subject + routing/idempotency hints).
- [x] Domain write service RPCs use business verbs (no CRUD-style update/set/patch prefixes).
- [x] Proto files for BYOD storefront domains exist (envelope, domains, query, write service, agent surface, process facts).

Build-out checklist (contracts hardening):

- [ ] Standardize RequestContext propagation across all RPCs (Domain/Process/Integration/Projection) and document header-vs-message rules.
- [ ] Freeze and publish the stable fact type identifiers (and their CloudEvents mapping) as the compatibility contract.
- [ ] Add a single “ownership table” mapping business workflows → Domain/Process responsibility and the facts/intents involved.

## Clarification requests (next steps)

Confirm/decide:

- [ ] Whether RPC correlation/trace propagation is standardized via gRPC metadata/headers or must be explicitly present in request messages (especially Integration RPCs).
- [ ] Whether Integration requests should also carry `common.v1.RequestContext` (vs only `scope + idempotency_key`) for uniform observability.
- [ ] Whether CloudEvents mapping is normative (and which fields are the canonical stable identifiers across refactors).

## Layer header (Application contracts)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Service (RPC/query), Application Interface (topic families/subjects), Application Event (facts), Data Object (proto messages/envelopes).
- **Out-of-scope layers:** Business/Strategy definitions (see `523-commerce.md`); Technology runtime selection (see `473-ameide-technology.md`).

EDA mapping note (per `470-ameide-vision.md` §0.2):

- facts (domain facts + process facts) → Application Events (state changes)
- intents/commands → requests to invoke Application Services (carried via RPC and/or an intent topic family)
- queries → read-only Application Services (often realized by Projection primitives)

## 1) Message families and topics

Topic families (v1):

- `commerce.domain.intents.v1` → `ameide_core_proto.commerce.v1.CommerceDomainIntent`
- `commerce.domain.facts.v1` → `ameide_core_proto.commerce.v1.CommerceDomainFact`
- `commerce.process.facts.v1` → `ameide_core_proto.process.commerce.v1.CommerceProcessFact`
- (optional) `commerce.integration.facts.v1` → `ameide_core_proto.commerce.integration.v1.CommerceIntegrationFact`

Transport note:

- Domains may accept commands via RPC and publish facts via outbox→broker.
- Intents can be carried either via RPC or via the intent topic family; choose one as canonical per deployment and keep the other as an adapter.

## 2) Canonical envelope: commerce scope + CloudEvents/TraceContext alignment

Commerce should be able to bridge cleanly across brokers and HTTP without bespoke adapters. The envelope below is intentionally aligned so it can be mapped losslessly to:

- CloudEvents attributes (`id`, `type`, `source`, `subject`, `specversion`)
- W3C Trace Context headers (`traceparent`, `tracestate`)

Every intent/fact MUST carry:

- tenant isolation: `tenant_id` (required)
- traceability: `message_id`, `correlation_id`, `causation_id`, and timestamps (see `496-eda-principles.md` Principle 8)
- commerce scope: `site_id`, `sales_channel_id`, `stock_location_id`, `store_site_id` when applicable
- event metadata that can map to CloudEvents + W3C trace context

Proposal:

```proto
syntax = "proto3";
package ameide_core_proto.commerce.v1;

import "google/protobuf/timestamp.proto";

message CommerceScope {
  string tenant_id = 1;
  string site_id = 2;           // web presence (domains/base URLs)
  string sales_channel_id = 3;  // selling context (currency/tax/pricing/promos)
  string stock_location_id = 4; // physical inventory/fulfillment node
  string store_site_id = 5;     // edge deployment unit (store cluster)
}

message CommerceMessageMeta {
  string message_id = 1;                // idempotency key
  string correlation_id = 2;            // end-to-end request chain
  string causation_id = 3;              // prior message that caused this
  google.protobuf.Timestamp occurred_at = 4;
  string producer = 5;                  // workload identity
  string schema_version = 6;            // semantic marker (e.g. "1.0.0")

  // CloudEvents alignment (lossless mapping target for cross-broker/HTTP).
  string event_type = 22;     // CloudEvents "type"
  string event_source = 23;   // CloudEvents "source" (service/tenant URN)
  string event_subject = 24;  // CloudEvents "subject" (order_id, hostname, etc.)

  // W3C Trace Context (propagate across HTTP/brokers).
  string traceparent = 20;
  string tracestate = 21;
}
```

Notes:

- Use `message_id` as the idempotency key for commands/intents; facts should set `causation_id` to the triggering command/message ID.
- For ordering/partitioning, prefer `{tenant_id, aggregate_id}` keys (domain-specific).
- For CloudEvents mapping: `message_id → ce-id`, `event_type → ce-type`, `event_source → ce-source`, `event_subject → ce-subject`, and `occurred_at → ce-time`.

## 3) Domain intents and facts (v1)

Domains should standardize on “aggregator messages” (509 pattern) to keep topic contracts stable:

```proto
message CommerceDomainIntent {
  CommerceMessageMeta meta = 1;
  CommerceScope scope = 2;
  oneof intent {
    RequestStorefrontHostnameClaim request_storefront_hostname_claim = 10;
    RequestStorefrontHostnameMapping request_storefront_hostname_mapping = 11;
    RevokeStorefrontHostnameClaim revoke_storefront_hostname_claim = 12;
    RevokeStorefrontHostnameMapping revoke_storefront_hostname_mapping = 13;
  }
}

message CommerceDomainFact {
  CommerceMessageMeta meta = 1;
  CommerceScope scope = 2;
  CommerceAggregateRef aggregate = 3;
  oneof fact {
    StorefrontHostnameClaimCreated storefront_hostname_claim_created = 10;
    StorefrontHostnameClaimUpdated storefront_hostname_claim_updated = 11;
    StorefrontHostnameMappingCreated storefront_hostname_mapping_created = 12;
    StorefrontHostnameMappingUpdated storefront_hostname_mapping_updated = 13;
  }
}

message CommerceAggregateRef {
  string type = 1;     // e.g., "order", "payment_intent", "shift"
  string id = 2;       // aggregate ID
  int64 version = 3;   // monotonic per aggregate
}
```

Guidance:

- Intents are imperative business verbs (no CRUD).
- Facts are immutable, past-tense, and emitted via transactional outbox.
- `aggregate.version` is required for downstream idempotency/ordering.

## 4) Process facts (v1)

Processes emit process facts that describe orchestration state, distinct from domain facts:

```proto
package ameide_core_proto.process.commerce.v1;

message CommerceProcessFact {
  ameide_core_proto.commerce.v1.CommerceMessageMeta meta = 1;
  ameide_core_proto.commerce.v1.CommerceScope scope = 2;
  oneof fact {
    DomainMappingStatusChanged domain_mapping_status_changed = 10;
  }
}
```

Processes should:

- consume domain facts,
- call domain command RPCs (or emit domain intents),
- emit process facts for observability and governance.

## 5) BYOD domains (Phase 1): explicit onboarding objects + state machine

BYOD onboarding is a product surface and must be representable as structured data, not just “some facts happened”.

### Canonical types

```proto
enum HostnameKind {
  HOSTNAME_KIND_UNSPECIFIED = 0;
  HOSTNAME_KIND_APEX = 1;      // example.com
  HOSTNAME_KIND_SUBDOMAIN = 2; // shop.example.com
  HOSTNAME_KIND_WILDCARD = 3;  // *.example.com (v1 may choose to disallow)
}

enum DnsRecordType {
  DNS_RECORD_TYPE_UNSPECIFIED = 0;
  DNS_RECORD_TYPE_A = 1;
  DNS_RECORD_TYPE_AAAA = 2;
  DNS_RECORD_TYPE_CNAME = 3;
  DNS_RECORD_TYPE_TXT = 4;
}

message DnsRecordRequirement {
  DnsRecordType type = 1;
  string name = 2;     // e.g. "_storefront-verify.shop.example.com"
  string value = 3;    // token, CNAME target, IP, etc.
  bool required = 4;
  bool removable_after_validation = 5;
}

enum DomainVerificationMethod {
  DOMAIN_VERIFICATION_METHOD_UNSPECIFIED = 0;
  DOMAIN_VERIFICATION_METHOD_TXT = 1;
  DOMAIN_VERIFICATION_METHOD_HTTP = 2;
  DOMAIN_VERIFICATION_METHOD_CNAME = 3;
}

message DomainVerificationInstructions {
  DomainVerificationMethod method = 1;
  HostnameKind hostname_kind = 2;
  repeated DnsRecordRequirement records = 3;
  google.protobuf.Timestamp issued_at = 4;
  google.protobuf.Timestamp expires_at = 5;
}

enum DomainOnboardingErrorCode {
  DOMAIN_ONBOARDING_ERROR_CODE_UNSPECIFIED = 0;

  // DNS/verification
  DOMAIN_ONBOARDING_ERROR_CODE_DNS_TARGET_INCORRECT = 1;
  DOMAIN_ONBOARDING_ERROR_CODE_TXT_MISSING = 2;
  DOMAIN_ONBOARDING_ERROR_CODE_TXT_MISMATCH = 3;
  DOMAIN_ONBOARDING_ERROR_CODE_PROPAGATION_PENDING = 4;

  // TLS/certs
  DOMAIN_ONBOARDING_ERROR_CODE_CAA_BLOCKED = 10;
  DOMAIN_ONBOARDING_ERROR_CODE_CERT_ISSUE_RATE_LIMIT = 11;
  DOMAIN_ONBOARDING_ERROR_CODE_CERT_ISSUE_FAILED = 12;

  // Platform constraints / common “battle scars”
  DOMAIN_ONBOARDING_ERROR_CODE_HOSTNAME_ALREADY_CLAIMED = 20;
  DOMAIN_ONBOARDING_ERROR_CODE_CDN_INTERFERENCE = 21; // proxy/CDN obscures validation
}

message DomainOnboardingError {
  DomainOnboardingErrorCode code = 1;
  string detail = 2; // human-readable for UI/support
  google.protobuf.Timestamp observed_at = 3;
  google.protobuf.Duration suggested_retry_after = 4;
}
```

### Claim and mapping objects

```proto
enum HostnameClaimStatus {
  HOSTNAME_CLAIM_STATUS_UNSPECIFIED = 0;
  HOSTNAME_CLAIM_STATUS_PENDING = 1;
  HOSTNAME_CLAIM_STATUS_VERIFIED = 2;
  HOSTNAME_CLAIM_STATUS_FAILED = 3;
  HOSTNAME_CLAIM_STATUS_REVOKED = 4;
}

enum HostnameMappingStatus {
  HOSTNAME_MAPPING_STATUS_UNSPECIFIED = 0;
  HOSTNAME_MAPPING_STATUS_REQUESTED = 1;
  HOSTNAME_MAPPING_STATUS_CERT_PENDING = 2;
  HOSTNAME_MAPPING_STATUS_ROUTE_PENDING = 3;
  HOSTNAME_MAPPING_STATUS_ACTIVE = 4;
  HOSTNAME_MAPPING_STATUS_FAILED = 5;
  HOSTNAME_MAPPING_STATUS_REVOKED = 6;
}

message StorefrontHostnameClaim {
  CommerceScope scope = 1;   // must include tenant_id + site_id
  string hostname = 2;       // lowercased FQDN
  HostnameKind hostname_kind = 3;
  HostnameClaimStatus status = 4;
  DomainVerificationInstructions instructions = 5;
  DomainOnboardingError last_error = 6;
}

message StorefrontHostnameMapping {
  CommerceScope scope = 1;   // must include tenant_id + site_id
  string hostname = 2;
  string uisurface_ref = 3;          // routing target
  HostnameMappingStatus status = 4;
  DomainOnboardingError last_error = 5;

  // Debug-only / logical selector (do not encode namespaces)
  string exposure_class = 10; // e.g., "shared-public-gateway", "per-tenant-gateway"
}
```

Implementation notes:

- Hostname must be lowercased and validated; reject invalid/uppercase early (CloudFront-like constraints are common).
- Treat propagation delay as normal; model retry/backoff in the error (`suggested_retry_after`).
- Wildcards are special and often require additional validation semantics; v1 can defer wildcard support.

### Process facts should be structured

Prefer structured fields in process facts (not just a named event):

```proto
message DomainMappingStatusChanged {
  string hostname = 1;
  HostnameClaimStatus claim_status = 2;
  HostnameMappingStatus mapping_status = 3;
  DomainOnboardingError last_error = 4;
}
```

## 6) Query APIs (read-only)

Queries should be explicit read services (Projection or Domain query surface) and MUST be read-only:

```proto
service CommerceQueryService {
  rpc ResolveHostname(ResolveHostnameRequest) returns (ResolveHostnameResponse);
  rpc GetHostnameClaim(GetHostnameClaimRequest) returns (GetHostnameClaimResponse);
  rpc GetHostnameMapping(GetHostnameMappingRequest) returns (GetHostnameMappingResponse);
}

message ResolveHostnameRequest { string hostname = 1; }
message ResolveHostnameResponse {
  CommerceScope scope = 1;  // must include tenant_id; typically includes site_id + default_sales_channel_id
  string uisurface_ref = 2;
  string canonical_host = 3;
  bool allowed = 4;
}

message GetHostnameClaimRequest {
  CommerceScope scope = 1; // tenant_id + site_id
  string hostname = 2;
}

message GetHostnameMappingRequest {
  CommerceScope scope = 1; // tenant_id + site_id
  string hostname = 2;
}

message GetHostnameClaimResponse { StorefrontHostnameClaim claim = 1; }
message GetHostnameMappingResponse { StorefrontHostnameMapping mapping = 1; }

// Future query APIs (Phase 2+): SearchProducts, GetAvailability, OrderHistory, etc.
```

## 7) Integration ports (external seams)

Integration protos define stable seams; implementations may be NiFi-backed (flows as code) or custom runtimes:

```proto
package ameide_core_proto.commerce.integration.v1;

message Money {
  string currency = 1; // ISO 4217
  int64 units = 2;
  int32 nanos = 3;
}

enum PaymentIntentStatus {
  PAYMENT_INTENT_STATUS_UNSPECIFIED = 0;
  PAYMENT_INTENT_STATUS_REQUIRES_PAYMENT_METHOD = 1;
  PAYMENT_INTENT_STATUS_REQUIRES_CONFIRMATION = 2;
  PAYMENT_INTENT_STATUS_REQUIRES_ACTION = 3;
  PAYMENT_INTENT_STATUS_PROCESSING = 4;
  PAYMENT_INTENT_STATUS_REQUIRES_CAPTURE = 5;
  PAYMENT_INTENT_STATUS_SUCCEEDED = 6;
  PAYMENT_INTENT_STATUS_CANCELED = 7;
  PAYMENT_INTENT_STATUS_FAILED = 8;
  PAYMENT_INTENT_STATUS_EXPIRED = 9;
}

message PaymentNextAction {
  oneof action {
    string redirect_url = 1;
    string sdk_payload_json = 2; // opaque provider hints
  }
}

service CommercePaymentsIntegrationService {
  rpc Authorize(AuthorizeRequest) returns (AuthorizeResponse);
  rpc Capture(CaptureRequest) returns (CaptureResponse);
  rpc Refund(RefundRequest) returns (RefundResponse);
}

message AuthorizeRequest {
  ameide_core_proto.commerce.v1.CommerceScope scope = 1;
  string idempotency_key = 2; // REQUIRED
  string order_id = 3;
  Money amount = 4;
  string payment_method_token = 5; // tokenized reference only (never PAN)
  bool capture_immediately = 6;     // allow auth-only flows
}

message AuthorizeResponse {
  PaymentIntentStatus status = 1;
  string provider_intent_id = 2;
  PaymentNextAction next_action = 3;
  string decline_code = 4;
}

message CaptureRequest {
  ameide_core_proto.commerce.v1.CommerceScope scope = 1;
  string idempotency_key = 2; // REQUIRED
  string provider_intent_id = 3;
  Money amount_to_capture = 4; // allow partial capture
}

message CaptureResponse {
  PaymentIntentStatus status = 1;
  string provider_charge_id = 2;
}

message RefundRequest {
  ameide_core_proto.commerce.v1.CommerceScope scope = 1;
  string idempotency_key = 2; // REQUIRED
  string provider_charge_id = 3;
  Money amount_to_refund = 4; // allow partial refund
}

message RefundResponse {
  PaymentIntentStatus status = 1;
  string provider_refund_id = 2;
}
```

Inbound (webhooks) must be idempotent and should publish outcome facts (optional `commerce.integration.facts.v1`).

### Replication job model (edge)

```proto
enum ReplicationJobKind {
  REPLICATION_JOB_KIND_UNSPECIFIED = 0;
  REPLICATION_JOB_KIND_DOWNSYNC_MASTERDATA = 1;
  REPLICATION_JOB_KIND_UPSYNC_TRANSACTIONS = 2;
  REPLICATION_JOB_KIND_BACKFILL = 3;
  REPLICATION_JOB_KIND_RECOVERY = 4;
}

message ReplicationCheckpoint {
  int64 replication_counter = 1;
  google.protobuf.Timestamp as_of = 2;
}

message RunReplicationJobRequest {
  ameide_core_proto.commerce.v1.CommerceScope scope = 1; // must include store_site_id
  string idempotency_key = 2;
  ReplicationJobKind kind = 3;
  string binding_ref = 4; // logical binding name (no URLs/topics)
  ReplicationCheckpoint since = 5;
}
```

### Hardware gateway seam (edge)

```proto
enum PeripheralKind {
  PERIPHERAL_KIND_UNSPECIFIED = 0;
  PERIPHERAL_KIND_RECEIPT_PRINTER = 1;
  PERIPHERAL_KIND_CASH_DRAWER = 2;
  PERIPHERAL_KIND_PAYMENT_TERMINAL = 3;
  PERIPHERAL_KIND_BARCODE_SCANNER = 4;
}

message RegisterDeviceRef {
  ameide_core_proto.commerce.v1.CommerceScope scope = 1; // includes store_site_id
  string register_id = 2;
  string device_id = 3;
}

message PingHardwareStationRequest { RegisterDeviceRef device = 1; }

service CommerceHardwareIntegrationService {
  rpc PingHardwareStation(PingHardwareStationRequest) returns (google.protobuf.Empty);
}
```

## 8) High-level implementation mapping (who talks to whom)

- **UISurface → Domain**
  - emits commands/intents (RPC or intent topic)
  - reads via query services/projections

- **Domain → Broker**
  - emits domain facts via transactional outbox dispatcher

- **Process ↔ Domain**
  - consumes domain facts, orchestrates saga
  - calls domain command RPCs / emits intents
  - emits process facts

- **Projection**
  - consumes facts and builds read models (idempotent upsert/inbox)

- **Integration**
  - boundary adapters for payments/hardware/ERP/etc.
  - inbound is at-least-once + idempotent; outbound driven by process/domain workflows

- **Agent**
  - consumes facts for awareness/diagnostics
  - proposes or emits commands/intents through the same surfaces (no side-channel writes)

## 9) Next steps (when implementing)

- Pick the canonical command transport (RPC-first vs intent-topic-first) and document it.
- Define the initial v1 intent/fact catalogs for the 7 first-class business processes in `523-commerce-process.md`.
- Add CI gates/codegen to enforce:
  - required metadata fields present,
  - topic naming matches package major,
  - no CRUD RPC names for write services.

---

## MCP exposure (proto-first)

If Commerce publishes MCP tools/resources, MCP schemas/manifests are derived from these proto contracts (service/method annotations) and implemented via a capability-owned Integration primitive (`commerce-mcp-adapter`).

Rules:

- Do not hand-author a parallel JSON tool schema; generate MCP schemas from proto (default deny; explicit allowlist).
- MCP is a protocol binding; semantics remain owned by the underlying Domain/Projection Application Services.

See `backlog/534-mcp-protocol-adapter.md` for the adapter pattern and generation posture.
