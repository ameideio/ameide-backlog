# 496 – Event-Driven Architecture (EDA) Principles

**Status:** Active
**Audience:** Platform engineering, domain teams, AI agents, architects
**Scope:** Canonical EDA principles for all Ameide primitives (Domain, Process, Projection, Integration, Agent, UISurface)

> **Core Invariants**: EDA invariants §8-13 are defined in [470-ameide-vision.md](470-ameide-vision.md). This document provides the complete reference with implementation guidance.

## Grounding & contract alignment

- **EDA spine for primitives:** Expands the high-level EDA invariants from `470-ameide-vision.md` and the application architecture from `472-ameide-information-application.md` into concrete messaging rules that all primitives and operators must follow (commands vs events, transactional outbox, idempotent consumers).  
- **Operator/CLI integration:** Provides the event-handling expectations and patterns that operator/vertical-slice backlogs (`473-ameide-technology.md`, `495-ameide-operators.md`, `497-operator-implementation-patterns.md`, `498-domain-operator.md`, `499-process-operator.md`, `502-domain-vertical-slice.md`) and CLI tooling (`484a-484f`) use when validating primitive behavior.  
- **Scrum/process catalog:** Defines the Scrum-aware process event catalog and domain/process topic separation (`scrum.domain.intents.v1`, `scrum.domain.facts.v1`, `scrum.process.facts.v1`) that the Scrum stack (`506-scrum-vertical-v2.md`, `508-scrum-protos.md`) and agent architecture (`505-agent-developer-v2*.md`, `507-scrum-agent-map.md`) build on.

---

## 1. Purpose

This document defines the **event-driven architecture principles** that all Ameide primitives must follow. These principles ensure:

- Reliable event delivery without data loss
- Idempotent processing for at-least-once semantics
- Clean separation between domain logic and infrastructure
- Multi-tenant isolation in event flows
- Observable, traceable event processing

Agent-to-agent (A2A) communication per [505-agent-developer-v2.md](505-agent-developer-v2.md) is deliberately **out of scope** for this document; this backlog codifies broker-based EDA traffic between primitives.

---

## 2. The 10 EDA Principles

### Principle 1: Events are Immutable Facts, Commands Express Intent

**Rule**: An **event** is an immutable fact about something that happened in the past. A **command** is a request for something to happen.

| Concept | Definition | Example |
|---------|------------|---------|
| **Command** | Request expressing business intent | `PlaceOrder`, `ApproveQuote`, `CancelSubscription` |
| **Event** | Past-tense fact that occurred | `OrderPlaced`, `QuoteApproved`, `SubscriptionCancelled` |

**Ameide terminology**: In Ameide topic-based contracts, a bus-carried **command** is a **domain intent**, and a bus-carried **event** is a **domain fact** (see `496-eda-protobuf-ameide.md` for topic/envelope rules).

#### Semantics ≠ transport

- All broker messages are “events” in the technical sense; Ameide still distinguishes **intents** (requests) from **facts** (statements) because ownership and retry/idempotency responsibilities differ.
- **Commands/intents** MAY be delivered via RPC or via an intent topic; choose one canonical ingress per domain.
- **Facts/events** are published (outbox → broker) and consumed via pub/sub; they are never requests.

#### Classify by state ownership (not producer)

- If a message requests changing **domain-owned state**, it is a **domain command/intent** whether it came from a UI, agent, integration, or process.
- A **domain fact/event** can only be emitted by the domain that owns that aggregate state, after commit.
- A **process fact/event** describes orchestration state (governance/timeboxes/work-available) and is emitted by the process runtime.

#### UI and human work

- UISurfaces are clients: they read projections and observe facts; they issue commands/intents back to the owning domain/process.
- A process reaching a human-review step should emit a **process fact** like `HumanReviewRequested` / `ApprovalStepActivated` and wait; the human response is a command/intent back (e.g. `ApproveInvoiceRequested`, `CompleteHumanReviewRequested`).
- UI telemetry (“ButtonClicked”) is analytics/observability and is not a domain integration contract.

#### Coupling anti-pattern to avoid

- Avoid “events as passive-aggressive commands”: publishing a fact while implicitly requiring a specific consumer to act. If you need one responsible handler, use a command/intent.

#### Execution queues (single-responsibility) vs fact topics (pub/sub)

- **Fact topics are pub/sub**: many consumers may react; facts are emitted only by the owning domain/process after persistence (outbox).
- **Execution queues are point-to-point** (single-responsibility consumption): their payloads MUST be **intents** (requests), never facts.
- Any executor (in-cluster or external) MUST report outcomes via the owning domain/process write surface; the owner emits facts after persistence for audit and orchestration continuation.
- **At-least-once is the default reality:** broker delivery is at-least-once, and Temporal Activities may run more than once (retries/crashes). Therefore:
  - all commands/intents MUST carry an idempotency key (`message_id` / `request_id`), and
  - domain/process write surfaces MUST be idempotent on that key.
- **Kafka→Temporal signal ingestion requires dedupe:** if facts are routed into long-lived workflows via `SignalWithStart`, workflows MUST dedupe signals by `message_id` (especially across `Continue-As-New` boundaries).

#### Temporal complements outbox (standard orchestration + fact emission pattern)

Temporal makes **orchestration** durable; the transactional outbox makes **fact emission** durable.

- **Outbox:** domain state change ↔ domain fact publish (DB commit + outbox row in one transaction; dispatcher publishes).
- **Temporal:** workflow state machine is durable; Activities are *at-least-once in practice* and must be idempotent.

Therefore:

- Workflows/Activities MUST NOT publish domain facts directly; they submit domain intents/commands and domains emit facts after persistence (outbox).
- If Kafka facts are routed into Temporal via ingress (`SignalWithStart`), workflows MUST dedupe by `message_id` (and plan `Continue-As-New`).

Canonical flow:

```text
Workflow (Temporal) -> Activity -> Domain command (idempotent) -> DB TX (state + outbox rows)
  -> Outbox dispatcher -> Kafka facts (pub/sub) + Kafka execution intents (point-to-point, optional)
  -> Ingress -> SignalWithStart -> Workflow (dedupe by message_id; Continue-As-New as needed)
```

#### Idempotency keys: what to dedupe on

- `meta.message_id`: the dedupe key for broker-delivered messages and for workflow signals derived from them.
- `meta.correlation_id`: trace linkage only; MUST NOT be used as a dedupe key.
- `meta.causation_id`: “what caused this” linkage; not a dedupe key.

**Implementation**:

```protobuf
// Commands (requests) - imperative verbs
service OrdersService {
  rpc PlaceOrder(PlaceOrderRequest) returns (Order);
  rpc CancelOrder(CancelOrderRequest) returns (Order);
  rpc ShipOrder(ShipOrderRequest) returns (Order);
}

// Events (facts) - past tense
message OrderPlaced {
  string order_id = 1;
  string tenant_id = 2;
  google.protobuf.Timestamp placed_at = 3;
}

message OrderCancelled {
  string order_id = 1;
  string tenant_id = 2;
  string reason = 3;
  google.protobuf.Timestamp cancelled_at = 4;
}
```

**Anti-patterns to avoid**:

| Bad (CRUD) | Good (Business Intent) |
|------------|------------------------|
| `UpdateOrder` | `CancelOrder`, `ShipOrder`, `AdjustOrderLines` |
| `SetStatus` | `ApproveQuote`, `RejectQuote`, `SubmitForReview` |
| `ModifyCustomer` | `ChangeCustomerAddress`, `UpgradeCustomerTier` |
| `PatchInvoice` | `VoidInvoice`, `ApplyPayment`, `AddLineItem` |

> **Reference**: [470 §8](470-ameide-vision.md), [472 §2.8.6](472-ameide-information-application.md)

---

### Principle 2: Schema-First, Type-Safe Messages (Proto + Buf)

**Rule**: All events and commands are defined in Protobuf. No ad-hoc JSON for events.

**Implementation**:

```
packages/ameide_core_proto/
├── src/
│   └── ameide_core_proto/
│       ├── orders/
│       │   └── v1/
│       │       ├── orders_service.proto   # RPC ingress (commands/queries)
│       │       ├── orders_intents.proto   # Bus ingress (domain intents), optional
│       │       ├── orders_facts.proto     # Bus egress (domain facts)
│       │       └── orders_query.proto     # Query surface, optional
│       └── sales/
│           └── v1/
│               ├── sales_service.proto
│               ├── sales_intents.proto
│               └── sales_facts.proto
```

**Event proto conventions**:

```protobuf
// packages/ameide_core_proto/src/ameide_core_proto/orders/v1/orders_facts.proto

syntax = "proto3";
package ameide_core_proto.orders.v1;

import "google/protobuf/timestamp.proto";

// Message metadata - required on all broker-delivered messages
message MessageMetadata {
  string message_id = 1;         // UUID, idempotency key
  string tenant_id = 2;          // Required for isolation
  string correlation_id = 3;     // Links related events
  string causation_id = 4;       // What caused this event
  string traceparent = 5;        // W3C Trace Context (canonical)
  string tracestate = 6;         // W3C Trace Context (optional)
  google.protobuf.Timestamp occurred_at = 7;
}

// In Ameide contracts, topics are stable and typically map 1:1 to an "aggregator" message.
message OrdersDomainFact {
  MessageMetadata meta = 1;
  oneof fact {
    OrderPlaced order_placed = 10;
  }
}

message OrderPlaced {
  string order_id = 1;
  string customer_id = 2;
  repeated OrderLine lines = 3;
  Money total = 4;
}
```

**Schema evolution rules** (enforced by Buf):

1. Never remove or rename fields
2. Never change field numbers
3. Add new fields with new numbers
4. Use `reserved` for removed fields
5. Version packages when breaking changes are unavoidable (`v1` → `v2`)

> **Reference**: [472 §2.7](472-ameide-information-application.md), [365-buf-sdks-v2](365-buf-sdks-v2.md)

---

### Principle 3: Single Transaction for State + Events (Transactional Outbox)

**Rule**: Aggregate updates and event publishing MUST happen in a single database transaction. Never call `publisher.Publish()` directly from domain code.

**The problem (dual write)**:

```go
// BAD - Dual write: DB can succeed, publish can fail (or vice versa)
func (s *Service) PlaceOrder(ctx context.Context, req *pb.PlaceOrderRequest) error {
    order := domain.NewOrder(req)
    if err := s.repo.Save(ctx, order); err != nil {
        return err
    }
    // If this fails, DB has the order but no event was published!
    if err := s.publisher.Publish(ctx, "OrderPlaced", order); err != nil {
        log.Error("failed to publish", err) // Data inconsistency!
    }
    return nil
}
```

**The solution (transactional outbox)**:

```go
// GOOD - Single transaction: aggregate + outbox are atomic
func (s *Service) PlaceOrder(ctx context.Context, req *pb.PlaceOrderRequest) (*pb.Order, error) {
    tx, err := s.db.BeginTx(ctx, nil)
    if err != nil { return nil, err }
    defer tx.Rollback()

    // 1. Create and persist aggregate
    order := domain.NewOrder(req)
    if err := s.repo.Save(ctx, tx, order); err != nil {
        return nil, err
    }

    // 2. Write fact to outbox (same transaction)
    // Note: The broker destination is the topic family (not a per-event-type string).
    fact := &orders.OrdersDomainFact{
        Meta: &orders.MessageMetadata{
            MessageId:     uuid.New().String(),
            TenantId:      order.TenantId,
            CorrelationId: extractCorrelationId(ctx),
            Traceparent:   extractTraceparent(ctx),
            Tracestate:    extractTracestate(ctx),
            OccurredAt:    timestamppb.Now(),
        },
        Fact: &orders.OrdersDomainFact_OrderPlaced{OrderPlaced: &orders.OrderPlaced{
            OrderId:    order.Id,
            CustomerId: order.CustomerId,
            Total:      order.Total,
        }},
    }
    if err := s.outbox.Insert(ctx, tx, "orders.domain.facts.v1", "OrderPlaced", proto.Marshal(fact)); err != nil {
        return nil, err
    }

    // 3. Commit - both succeed or both fail
    if err := tx.Commit(); err != nil {
        return nil, err
    }

    return order.ToProto(), nil
}
```

**Outbox table schema**:

```sql
CREATE TABLE outbox_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    message_id UUID NOT NULL,
    topic TEXT NOT NULL,
    message_type TEXT NOT NULL,
    payload_bytes BYTEA NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    published_at TIMESTAMPTZ,
    retry_count INT DEFAULT 0,

    -- Index for dispatcher polling
    INDEX idx_outbox_pending (published_at) WHERE published_at IS NULL
);
```

**Outbox dispatcher** (separate process):

```go
// Dispatcher polls outbox and publishes to broker
func (d *Dispatcher) Run(ctx context.Context) error {
    for {
        // Fetch MUST be safe for horizontal scaling (leasing or row locks such as FOR UPDATE SKIP LOCKED).
        records, err := d.outbox.FetchPending(ctx, 100)
        if err != nil { return err }

        for _, r := range records {
            if err := d.publisher.Publish(ctx, r.Topic, r.PayloadBytes); err != nil {
                // Retry with backoff; after max retries, move to a DLQ with the last error reason.
                d.outbox.IncrementRetry(ctx, r.ID)
                continue
            }
            d.outbox.MarkPublished(ctx, r.ID)
        }

        time.Sleep(100 * time.Millisecond)
    }
}
```

> **Reference**: [470 §10](470-ameide-vision.md), [472 §3.3.1.1](472-ameide-information-application.md)

---

### Principle 4: Idempotent Consumers + Inbox Pattern

**Rule**: All event consumers MUST handle duplicates gracefully. Assume at-least-once delivery.

**The problem**: Brokers guarantee at-least-once, not exactly-once. The same event may be delivered multiple times.

**Solution 1: Inbox pattern** (explicit deduplication):

```go
func (h *Handler) HandleOrderPlaced(ctx context.Context, event *events.OrderPlaced) error {
    // Check if we've already processed this event
    processed, err := h.inbox.IsProcessed(ctx, "orders_projection_v1", event.Metadata.MessageId)
    if err != nil { return err }
    if processed {
        return nil // Already handled, skip
    }

    tx, err := h.db.BeginTx(ctx, nil)
    if err != nil { return err }
    defer tx.Rollback()

    // Process the event
    if err := h.processOrder(ctx, tx, event); err != nil {
        return err
    }

    // Mark as processed (same transaction)
    if err := h.inbox.MarkProcessed(ctx, tx, "orders_projection_v1", event.Metadata.MessageId); err != nil {
        return err
    }

    return tx.Commit()
}
```

**Inbox table schema**:

```sql
CREATE TABLE inbox_events (
    consumer_name TEXT NOT NULL,
    message_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    topic TEXT NOT NULL,
    message_type TEXT NOT NULL,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Prevent duplicates
    UNIQUE (consumer_name, message_id)
);
```

**Solution 2: Natural key idempotency** (when inbox is overkill):

```go
// For simple projections, use UPSERT with natural keys
func (h *Handler) HandleOrderPlaced(ctx context.Context, event *events.OrderPlaced) error {
    // UPSERT is naturally idempotent - same event produces same result
    return h.db.Exec(ctx, `
        INSERT INTO order_summaries (order_id, tenant_id, customer_id, total, status)
        VALUES ($1, $2, $3, $4, 'placed')
        ON CONFLICT (order_id) DO UPDATE SET
            customer_id = EXCLUDED.customer_id,
            total = EXCLUDED.total
    `, event.OrderId, event.Metadata.TenantId, event.CustomerId, event.Total)
}
```

> **Reference**: [470 §11](470-ameide-vision.md), [472 §3.3.2](472-ameide-information-application.md)

---

### Principle 5: Design for Eventual Consistency

**Rule**: Cross-domain invariants are enforced via processes (Temporal), not distributed transactions.

**Anti-pattern**: Distributed transaction across services

```go
// BAD - Distributed transaction (2PC)
func (s *Service) PlaceOrder(ctx context.Context, req *pb.PlaceOrderRequest) error {
    tx := s.distributedTx.Begin()

    // These are separate services - 2PC is fragile!
    if err := s.inventory.Reserve(tx, req.Items); err != nil {
        tx.Rollback()
        return err
    }
    if err := s.orders.Create(tx, req); err != nil {
        tx.Rollback()
        return err
    }
    if err := s.payments.Authorize(tx, req.Payment); err != nil {
        tx.Rollback()
        return err
    }

    return tx.Commit() // Single point of failure
}
```

**Correct pattern**: Saga via Temporal Process primitive

```go
// GOOD - Saga with compensation
func OrderPlacementWorkflow(ctx workflow.Context, req *pb.PlaceOrderRequest) error {
    var compensations []func(context.Context) error

    // Step 1: Reserve inventory
    if err := workflow.ExecuteActivity(ctx, ReserveInventory, req.Items).Get(ctx, nil); err != nil {
        return err
    }
    compensations = append(compensations, func(ctx context.Context) error {
        return ReleaseInventory(ctx, req.Items)
    })

    // Step 2: Create order
    if err := workflow.ExecuteActivity(ctx, CreateOrder, req).Get(ctx, nil); err != nil {
        runCompensations(ctx, compensations)
        return err
    }
    compensations = append(compensations, func(ctx context.Context) error {
        return CancelOrder(ctx, req.OrderId)
    })

    // Step 3: Authorize payment
    if err := workflow.ExecuteActivity(ctx, AuthorizePayment, req.Payment).Get(ctx, nil); err != nil {
        runCompensations(ctx, compensations)
        return err
    }

    return nil
}
```

**Key insight**: Each Domain primitive is the **single writer** for its aggregates. Cross-domain coordination happens in Process primitives.

> **Reference**: [473 §3.2](473-ameide-technology.md), [305-workflow](305-workflow.md)

---

### Principle 6: Clean Architecture Around Events

**Rule**: Domain logic MUST NOT import broker clients directly. All event publishing goes through interfaces.

**Directory structure** (hexagonal/ports-and-adapters):

```
primitives/domain/{name}/
├── cmd/main.go                      # Infrastructure wiring
├── internal/
│   ├── domain/                      # Pure business logic (no broker imports!)
│   │   ├── aggregates/
│   │   │   └── order.go             # Order aggregate
│   │   ├── events/
│   │   │   └── order_events.go      # Domain event definitions
│   │   └── commands/
│   │       └── place_order.go       # Command handlers
│   ├── ports/                       # Interfaces (contracts)
│   │   ├── repository.go            # type OrderRepository interface
│   │   ├── outbox.go                # type EventOutbox interface
│   │   └── clock.go                 # type Clock interface
│   └── adapters/                    # Infrastructure implementations
│       ├── postgres/
│       │   ├── order_repository.go  # PostgresOrderRepository
│       │   └── outbox.go            # PostgresEventOutbox
│       └── watermill/
│           └── publisher.go         # WatermillPublisher (used by dispatcher only)
```

**Port interface** (domain defines contract):

```go
// internal/ports/outbox.go
package ports

import "context"

// EventOutbox is the contract for event persistence
// Domain code depends on this interface, not on Watermill/NATS/Kafka
type EventOutbox interface {
    // Insert persists a broker-ready record as part of the domain transaction.
    // Routing uses topic families; message_type is for observability/debugging.
    Insert(ctx context.Context, tx Transaction, topic string, messageType string, messageId string, tenantId string, payloadBytes []byte) error
}

// The domain NEVER imports:
// - github.com/ThreeDotsLabs/watermill
// - github.com/nats-io/nats.go
// - github.com/segmentio/kafka-go
```

**Adapter implementation** (infrastructure implements contract):

```go
// internal/adapters/postgres/outbox.go
package postgres

import (
    "context"
    "database/sql"

    "github.com/ameideio/ameide/primitives/domain/orders/internal/ports"
)

type PostgresEventOutbox struct {
    db *sql.DB
}

func (o *PostgresEventOutbox) Insert(ctx context.Context, tx ports.Transaction, topic string, messageType string, messageId string, tenantId string, payloadBytes []byte) error {
    _, err := tx.ExecContext(ctx, `
        INSERT INTO outbox_events (tenant_id, message_id, topic, message_type, payload_bytes, created_at)
        VALUES ($1, $2, $3, $4, $5, NOW())
    `, tenantId, messageId, topic, messageType, payloadBytes)
    return err
}
```

> **Reference**: [470 §12](470-ameide-vision.md), [473 §3.2.1](473-ameide-technology.md)

---

### Principle 7: Intentional Broker Selection & Delivery Guarantees

**Rule**: Choose the right broker for each use case. Document delivery guarantees explicitly.

**Broker selection matrix**:

| Use Case | Technology | Delivery | Ordering | Retention |
|----------|------------|----------|----------|-----------|
| **Domain events (outbox relay)** | Postgres + Watermill | At-least-once | Per aggregate | Until published |
| **Cross-domain integration** | NATS JetStream | At-least-once | Per subject | 7 days |
| **Analytics & replay** | Kafka / Redpanda | At-least-once | Per partition | 30+ days |
| **UI real-time updates** | NATS (ephemeral) | Best-effort | None | None |
| **Process orchestration** | Temporal | Exactly-once* | Workflow-local | Workflow history |

*Exactly-once within workflow execution, not across external systems.

**Delivery guarantee rules**:

1. **At-least-once is the default** – Never promise exactly-once over external brokers
2. **Ordering is per-aggregate** – Partition by `(tenant_id, aggregate_id)`
3. **Cross-aggregate ordering is NOT guaranteed** – Use Temporal for coordination
4. **Replay is not free** – Only Kafka/analytics streams support replay

**Stream configuration examples**:

```yaml
# NATS JetStream - Domain events
stream:
  name: domain-events-orders
  subjects: ["orders.domain.facts.v1"]
  retention: limits
  max_age: 604800s           # 7-day rolling window
  storage: file
  replicas: 3

# Kafka - Analytics (long retention, replayable)
topic:
  name: analytics.orders
  partitions: 12             # By tenant_id hash
  retention.ms: 2592000000   # 30 days
  min.insync.replicas: 2
```

> **Reference**: [473 §3.2.1](473-ameide-technology.md)

---

### Principle 8: Observability as First-Class EDA Concern

**Rule**: Every event must be traceable from command to final consumer state change.

**Required message metadata**:

```protobuf
message MessageMetadata {
  string message_id = 1;         // Unique ID for deduplication
  string tenant_id = 2;          // Multi-tenant isolation
  string correlation_id = 3;     // Links all events in a flow
  string causation_id = 4;       // Direct parent event/command
  string traceparent = 5;        // W3C Trace Context (canonical)
  string tracestate = 6;         // W3C Trace Context (optional)
  google.protobuf.Timestamp occurred_at = 7;
}
```

**Metrics per event type** (Prometheus):

```go
var (
    eventsPublished = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "events_published_total",
            Help: "Total events published to outbox",
        },
        []string{"domain", "topic", "message_type", "tenant_id"},
    )

    eventsProcessed = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "events_processed_total",
            Help: "Total events processed by consumers",
        },
        []string{"domain", "topic", "message_type", "outcome"}, // outcome: success, failure, duplicate
    )

    eventProcessingLatency = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "event_processing_duration_seconds",
            Help:    "Event processing latency",
            Buckets: prometheus.DefBuckets,
        },
        []string{"domain", "topic", "message_type"},
    )

    consumerLag = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "consumer_lag_seconds",
            Help: "Consumer lag behind latest event",
        },
        []string{"domain", "consumer_group"},
    )

    dlqDepth = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "dlq_depth",
            Help: "Dead letter queue depth",
        },
        []string{"domain"},
    )
)
```

**Structured logging**:

```go
log.Info("event processed",
    "message_id", event.Metadata.MessageId,
    "message_type", "OrderPlaced",
    "topic", "orders.domain.facts.v1",
    "tenant_id", event.Metadata.TenantId,
    "correlation_id", event.Metadata.CorrelationId,
    "traceparent", event.Metadata.Traceparent,
    "processing_time_ms", duration.Milliseconds(),
    "outcome", "success",
)
```

**OpenTelemetry span linking** (across async boundaries):

```go
func (h *Handler) HandleOrderPlaced(ctx context.Context, event *events.OrderPlaced) error {
    // Link to producer span
    producerSpanCtx := trace.SpanContextFromContext(
        otel.GetTextMapPropagator().Extract(ctx, carrier),
    )

    ctx, span := tracer.Start(ctx, "HandleOrderPlaced",
        trace.WithLinks(trace.Link{SpanContext: producerSpanCtx}),
        trace.WithAttributes(
            attribute.String("message.id", event.Metadata.MessageId),
            attribute.String("message.type", "OrderPlaced"),
            attribute.String("tenant.id", event.Metadata.TenantId),
        ),
    )
    defer span.End()

    // Process event...
}
```

> **Reference**: [472 §3.3.4](472-ameide-information-application.md), [472 §3.3.6](472-ameide-information-application.md)

---

### Principle 9: 12-Factor & Cloud-Native Basics

**Rule**: Event-driven services follow 12-factor app principles.

| Factor | EDA Application |
|--------|-----------------|
| **Codebase** | One repo per primitive, proto contracts shared |
| **Dependencies** | Explicit (go.mod, package.json), no implicit broker clients |
| **Config** | Broker URLs, credentials via env vars, not hardcoded |
| **Backing services** | Postgres, NATS, Kafka are attached resources |
| **Build, release, run** | Immutable images tagged with proto version |
| **Processes** | Stateless event handlers, state in DB + event log |

---

## 11. Process primitive event catalog (per 505/506-v2/508)

Process primitives own the sprint/ADM lifecycle defined in [505-agent-developer-v2.md](505-agent-developer-v2.md) and follow the **Scrum intent/fact split** described in [506-scrum-vertical-v2.md](506-scrum-vertical-v2.md) and the `transformation_scrum_*` protos in [508-scrum-protos.md](508-scrum-protos.md):

- **Scrum domain facts** (e.g., `SprintCreated`, `SprintStarted`, `SprintBacklogCommitted`, `ProductBacklogItemDoneRecorded`, `IncrementUpdated`) are emitted by the Transformation domain on `scrum.domain.facts.v1`.  
- **Scrum domain intents** (e.g., `StartSprintRequested`, `EndSprintRequested`, `CommitSprintBacklogRequested`, `RecordProductBacklogItemDoneRequested`, `RecordIncrementRequested`) are emitted by Process or agents on `scrum.domain.intents.v1`.  
- **Process facts** (governance/orchestration signals) are emitted by Process on `scrum.process.facts.v1` and consumed by AmeidePO/SA/observers.

Canonical **Scrum process facts** (see 506-v2 §6 for the authoritative list and field definitions):

| Event | Emitted by | Consumers | Payload (summary) |
|-------|------------|-----------|-------------------|
| `SprintBacklogReadyForExecution` | Process primitive | AmeidePO | `sprintId`, `productId` |
| `SprintBacklogItemReadyForWork` | Process primitive | AmeidePO, SA, downstream automation | `sprintId`, `productBacklogItemId`, optional `devBriefRef` |
| `SprintStartingSoon` / `SprintEndingSoon` | Process primitive | AmeidePO, reporting services | `sprintId`, `productId`, warning thresholds |
| `SprintTimeboxReachedEnd` | Process primitive | AmeidePO, observability, escalation handlers | `sprintId`, `productId`, `timeboxId`, `deadline` |
| `SLAWarning` / `SLAExceeded` | Process primitive | Observability, escalation handlers | `sprintId`, `productBacklogItemId?`, `timeboxId`, `breachKind`, `deadline` |

Implementation guidance:

- Publish these **process facts** on the `scrum.process.facts.v1` topic family so AmeidePO and other consumers can subscribe without special wiring.  
- Do **not** re-emit Scrum domain facts such as `SprintStarted` or `ProductBacklogItemDoneRecorded` on the process topic; downstream systems that need raw domain state listen to `scrum.domain.facts.v1` instead.  
- Include `process_scope` / `timebox_id` and the relevant aggregate identifiers (`sprintId`, `productBacklogItemId`) on every event so AmeideSA/AmeidePO DAGs can correlate Tasks → EDA events deterministically.
| **Port binding** | gRPC/Connect services self-contained |
| **Concurrency** | Scale out via consumer groups, not threads |
| **Disposability** | Fast startup/shutdown, graceful drain on SIGTERM |
| **Dev/prod parity** | Same broker in dev (Tilt) as prod |
| **Logs** | Structured JSON to stdout, correlation IDs |
| **Admin processes** | Migrations, outbox dispatcher as separate processes |

**Stateless handlers**:

```go
// GOOD - Handler is stateless, all state in DB
type OrderHandler struct {
    repo   ports.OrderRepository  // Injected
    outbox ports.EventOutbox      // Injected
}

// BAD - Handler caches state in memory
type OrderHandler struct {
    cache map[string]*Order  // State leaks between invocations!
}
```

> **Reference**: [12factor.net](https://12factor.net)

---

### Principle 10: Multi-Tenant Event Isolation

**Rule**: Every message MUST carry `tenant_id`. Consumers MUST establish tenant execution context from the message and enforce data isolation.

**Event structure**:

```protobuf
message OrderPlaced {
  MessageMetadata metadata = 1;  // Contains tenant_id
  string order_id = 2;
  // ...
}
```

**Consumer validation**:

```go
func (h *Handler) HandleOrderPlaced(ctx context.Context, event *events.OrderPlaced) error {
    // Async consumers often do not have a request-scoped tenant context.
    // Set tenant context from the message before touching tenant-scoped state.
    ctx = tenant.WithContext(ctx, event.Metadata.TenantId)

    // Optional stricter mode: consumer instance is configured for a single tenant.
    if h.config.TenantId != "" && event.Metadata.TenantId != h.config.TenantId {
        return fmt.Errorf("tenant mismatch: message=%s configured=%s",
            event.Metadata.TenantId, h.config.TenantId)
    }

    // Safe to process
    return h.processOrder(ctx, event)
}
```

**Stream partitioning**:

```go
// Topic family is stable (routing); tenant is for isolation and partitioning.
topic := "orders.domain.facts.v1"

// Kafka partitions by (tenant_id, aggregate_id) for per-aggregate ordering.
partitionKey := fmt.Sprintf("%s:%s", tenantId, aggregateId)
```

**Database isolation**:

```sql
-- RLS policy ensures tenant can only see their events
CREATE POLICY tenant_isolation ON order_events
    FOR ALL
    USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

> **Reference**: [470 §13](470-ameide-vision.md), [472 §3.3.7](472-ameide-information-application.md)

---

## 3. CLI Enforcement

The `ameide primitive verify` command enforces these principles:

| Check | Principle | Severity |
|-------|-----------|----------|
| **RPC naming** | §1 Commands express intent | WARN |
| **Forbidden prefixes** | §1 No CRUD verbs | WARN |
| **Event emission** | §1 Commands emit events | WARN |
| **Outbox wiring** | §3 Transactional outbox | WARN |
| **Idempotency guard** | §4 Inbox pattern | WARN |
| **Broker imports** | §6 Clean boundaries | WARN |
| **Tenant validation** | §10 Tenant isolation | WARN |
| **Schema versioning** | §2 Buf compatibility | WARN |

```bash
$ ameide primitive verify domain/orders --json
{
  "summary": "fail",
  "checks": {
    "rpc_naming": {"status": "PASS"},
    "event_emission": {"status": "PASS", "coverage": "8/8"},
    "outbox_wiring": {"status": "PASS"},
    "idempotency_guard": {
      "status": "WARN",
      "issues": [
        {"handler": "HandleOrderShipped", "file": "handlers/shipping.go", "line": 42}
      ]
    }
  }
}
```

> **Reference**: [484-ameide-cli §4.0.2](484-ameide-cli.md)

---

## 4. Quick Reference Checklist

For every Domain primitive:

- [ ] **Commands use business verbs**, not CRUD (`PlaceOrder` not `UpdateOrder`)
- [ ] **Events are past-tense facts** (`OrderPlaced` not `OrderPlacing`)
- [ ] **All intents/facts defined in proto** (stable topic families with aggregator messages; no ad-hoc `events/v1` side packages)
- [ ] **Outbox pattern for publishing** (no direct `publisher.Publish()`)
- [ ] **Consumers are idempotent** (inbox pattern or natural key upsert)
- [ ] **No broker imports in domain** (only in adapters)
- [ ] **Events carry `tenant_id`** (validated by consumers)
- [ ] **Trace propagation** (`correlation_id` plus W3C Trace Context: `traceparent` / `tracestate`; derive `trace_id`/`span_id` for logs)
- [ ] **Metrics per event type** (`events_processed_total`, `consumer_lag`)
- [ ] **DLQ for failed events** (with alerting)

---

## 5. Cross-References

| Document | Relationship |
|----------|--------------|
| [470-ameide-vision.md](470-ameide-vision.md) | Core invariants §8-13 |
| [472-ameide-information-application.md](472-ameide-information-application.md) | CQRS implementation details (§2.8.6, §3.3) |
| [473-ameide-technology.md](473-ameide-technology.md) | Broker selection (§3.2.1) |
| [484-ameide-cli.md](484-ameide-cli.md) | CLI enforcement (§4.0.2) |
| [365-buf-sdks-v2.md](365-buf-sdks-v2.md) | Proto/SDK pipeline |
| [305-workflow.md](305-workflow.md) | Temporal saga patterns |

---

## 6. External References

- [Event-Driven Architecture in Go](https://blog.devops.dev/event-driven-architecture-in-go-golang-ab46f23bf9a8)
- [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)
- [Outbox, Inbox and Delivery Guarantees Explained](https://event-driven.io/en/outbox_inbox_patterns_and_delivery_guarantees_explained/)
- [Inbox Pattern for Idempotency](https://dev.to/actor-dev/inbox-pattern-51af)
- [12-Factor App](https://12factor.net)
- [Pattern: Event-driven architecture](https://microservices.io/patterns/data/event-driven-architecture.html)
