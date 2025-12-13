# 523 Commerce — Proto proposal (communication between primitives)

This document proposes a proto shape for commerce that makes the communication topology explicit and 496-native:

- commands/intents vs facts,
- domain facts vs process facts,
- required metadata for traceability and tenant isolation,
- how Domain/Process/Projection/Integration/UISurface/Agent communicate.

It complements:

- `496-eda-principles.md` (EDA invariants)
- `509-proto-naming-conventions.md` (package/topic conventions)
- `523-commerce*.md` (commerce decomposition)

## 1) Message families and topics

Topic families (v1):

- `commerce.domain.intents.v1` → `ameide_core_proto.commerce.v1.CommerceDomainIntent`
- `commerce.domain.facts.v1` → `ameide_core_proto.commerce.v1.CommerceDomainFact`
- `commerce.process.facts.v1` → `ameide_core_proto.process.commerce.v1.CommerceProcessFact`
- (optional) `commerce.integration.facts.v1` → `ameide_core_proto.commerce.integration.v1.CommerceIntegrationFact`

Transport note:

- Domains may accept commands via RPC and publish facts via outbox→broker.
- Intents can be carried either via RPC or via the intent topic family; choose one as canonical per deployment and keep the other as an adapter.

## 2) Canonical envelope: metadata + commerce scope

Every intent/fact MUST carry:

- tenant isolation: `tenant_id` (required)
- traceability: `message_id`/`event_id`, `correlation_id`, `causation_id`, and timestamps (see `496-eda-principles.md` Principle 8)
- commerce scope: `site_id`, `sales_channel_id`, `stock_location_id`, `store_site_id` when applicable

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
  int32 schema_version = 6;
}
```

Notes:

- Use `message_id` as the idempotency key for commands/intents; facts should set `causation_id` to the triggering command/message ID.
- For ordering/partitioning, prefer `{tenant_id, aggregate_id}` keys (domain-specific).

## 3) Domain intents and facts (v1)

Domains should standardize on “aggregator messages” (509 pattern) to keep topic contracts stable:

```proto
message CommerceDomainIntent {
  CommerceMessageMeta meta = 1;
  CommerceScope scope = 2;
  oneof intent {
    CheckoutRequested checkout_requested = 10;
    CapturePaymentRequested capture_payment_requested = 11;
    ReturnOrderRequested return_order_requested = 12;
    // ...
  }
}

message CommerceDomainFact {
  CommerceMessageMeta meta = 1;
  CommerceScope scope = 2;
  CommerceAggregateRef aggregate = 3;
  oneof fact {
    OrderCreated order_created = 10;
    PaymentAuthorized payment_authorized = 11;
    ShiftOpened shift_opened = 12;
    // ...
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
    DomainMappingVerified domain_mapping_verified = 10;
    CheckoutSagaCompleted checkout_saga_completed = 11;
    StoreSiteProvisioned store_site_provisioned = 12;
    // ...
  }
}
```

Processes should:

- consume domain facts,
- call domain command RPCs (or emit domain intents),
- emit process facts for observability and governance.

## 5) Query APIs (read-only)

Queries should be explicit read services (Projection or Domain query surface) and MUST be read-only:

```proto
service CommerceQueryService {
  rpc ResolveHostname(ResolveHostnameRequest) returns (ResolveHostnameResponse);
  rpc SearchProducts(SearchProductsRequest) returns (SearchProductsResponse);
  rpc GetAvailability(GetAvailabilityRequest) returns (GetAvailabilityResponse);
}

message ResolveHostnameRequest { string hostname = 1; }
message ResolveHostnameResponse {
  CommerceScope scope = 1;  // must include tenant_id; typically includes site_id + default_sales_channel_id
  string uisurface_ref = 2;
  string canonical_host = 3;
  bool allowed = 4;
}
```

## 6) Integration ports (external seams)

Integration protos define stable seams; implementations may be NiFi-backed (flows as code) or custom runtimes:

```proto
package ameide_core_proto.commerce.integration.v1;

service CommercePaymentsIntegrationService {
  rpc Authorize(AuthorizePaymentRequest) returns (AuthorizePaymentResponse);
  rpc Capture(CapturePaymentRequest) returns (CapturePaymentResponse);
  rpc Refund(RefundPaymentRequest) returns (RefundPaymentResponse);
}
```

Inbound (webhooks) must be idempotent and should publish outcome facts (optional `commerce.integration.facts.v1`).

## 7) High-level implementation mapping (who talks to whom)

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

## 8) Next steps (when implementing)

- Pick the canonical command transport (RPC-first vs intent-topic-first) and document it.
- Define the initial v1 intent/fact catalogs for the 7 first-class business processes in `523-commerce-process.md`.
- Add CI gates/codegen to enforce:
  - required metadata fields present,
  - topic naming matches package major,
  - no CRUD RPC names for write services.

