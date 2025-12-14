# 496 – Event-Driven Architecture (EDA) Principles

**Status:** Active
**Audience:** Platform engineering, domain teams, AI agents, architects
**Scope:** Canonical EDA principles for all Ameide primitives (Domain, Process, Agent, UISurface)

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

Agent-to-agent (A2A) communication per [505-agent-developer-v2.md](505-agent-developer-v2.md) is deliberately **out of scope** for this document; A2A is a peer-to-peer contract between AmeideSA and AmeideCoder, whereas this backlog codifies domain ↔ process ↔ agent EDA traffic.

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
│       │       ├── orders.proto       # Commands & queries
│       │       └── events.proto       # Domain events
│       └── sales/
│           └── v1/
│               ├── sales.proto
│               └── events.proto
```

**Event proto conventions**:

```protobuf
// packages/ameide_core_proto/src/ameide_core_proto/orders/v1/events.proto

syntax = "proto3";
package ameide_core_proto.orders.v1.events;

import "google/protobuf/timestamp.proto";

// Event metadata - required on all events
message EventMetadata {
  string event_id = 1;           // UUID, idempotency key (aka message_id)
  string tenant_id = 2;          // Required for isolation
  string correlation_id = 3;     // Links related events
  string causation_id = 4;       // What caused this event
  google.protobuf.Timestamp occurred_at = 5;
  int32 schema_version = 6;      // For evolution
}

message OrderPlaced {
  EventMetadata metadata = 1;
  string order_id = 2;
  string customer_id = 3;
  repeated OrderLine lines = 4;
  Money total = 5;
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

    // 2. Write event to outbox (same transaction)
    event := &events.OrderPlaced{
        Metadata: &events.EventMetadata{
            EventId:       uuid.New().String(),
            TenantId:      order.TenantId,
            CorrelationId: extractCorrelationId(ctx),
            OccurredAt:    timestamppb.Now(),
        },
        OrderId:    order.Id,
        CustomerId: order.CustomerId,
        Total:      order.Total,
    }
    if err := s.outbox.Insert(ctx, tx, "orders.events.v1.OrderPlaced", event); err != nil {
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
    aggregate_type VARCHAR(255) NOT NULL,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL,
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
        events, err := d.outbox.FetchPending(ctx, 100)
        if err != nil { return err }

        for _, e := range events {
            if err := d.publisher.Publish(ctx, e.EventType, e.Payload); err != nil {
                d.outbox.IncrementRetry(ctx, e.ID)
                continue
            }
            d.outbox.MarkPublished(ctx, e.ID)
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
    processed, err := h.inbox.IsProcessed(ctx, event.Metadata.EventId)
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
    if err := h.inbox.MarkProcessed(ctx, tx, event.Metadata.EventId); err != nil {
        return err
    }

    return tx.Commit()
}
```

**Inbox table schema**:

```sql
CREATE TABLE inbox_events (
    event_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Prevent duplicates
    UNIQUE (event_id)
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
    Insert(ctx context.Context, tx Transaction, eventType string, payload proto.Message) error
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
    "encoding/json"

    "github.com/ameideio/ameide/primitives/domain/orders/internal/ports"
)

type PostgresEventOutbox struct {
    db *sql.DB
}

func (o *PostgresEventOutbox) Insert(ctx context.Context, tx ports.Transaction, eventType string, payload proto.Message) error {
    data, err := protojson.Marshal(payload)
    if err != nil { return err }

    _, err = tx.ExecContext(ctx, `
        INSERT INTO outbox_events (event_type, payload, created_at)
        VALUES ($1, $2, NOW())
    `, eventType, data)
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
  subjects: ["events.orders.>"]
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

**Required event metadata**:

```protobuf
message EventMetadata {
  string event_id = 1;           // Unique ID for deduplication (aka message_id)
  string tenant_id = 2;          // Multi-tenant isolation
  string correlation_id = 3;     // Links all events in a flow
  string causation_id = 4;       // Direct parent event/command
  string trace_id = 5;           // OpenTelemetry trace
  string span_id = 6;            // OpenTelemetry span
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
        []string{"domain", "event_type", "tenant_id"},
    )

    eventsProcessed = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "events_processed_total",
            Help: "Total events processed by consumers",
        },
        []string{"domain", "event_type", "outcome"}, // outcome: success, failure, duplicate
    )

    eventProcessingLatency = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "event_processing_duration_seconds",
            Help:    "Event processing latency",
            Buckets: prometheus.DefBuckets,
        },
        []string{"domain", "event_type"},
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
    "event_id", event.Metadata.EventId,
    "event_type", "OrderPlaced",
    "tenant_id", event.Metadata.TenantId,
    "correlation_id", event.Metadata.CorrelationId,
    "trace_id", event.Metadata.TraceId,
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
            attribute.String("event.id", event.Metadata.EventId),
            attribute.String("event.type", "OrderPlaced"),
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

Process primitives own the sprint/ADM lifecycle defined in [505-agent-developer-v2.md](505-agent-developer-v2.md) and follow the **Scrum intent/fact split** described in [506-scrum-vertical-v2.md](506-scrum-vertical-v2.md) and the `transformation-scrum-*` protos in [508-scrum-protos.md](508-scrum-protos.md):

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

**Rule**: Every event MUST carry `tenant_id`. Consumers MUST validate tenant context before processing.

**Event structure**:

```protobuf
message OrderPlaced {
  EventMetadata metadata = 1;  // Contains tenant_id
  string order_id = 2;
  // ...
}
```

**Consumer validation**:

```go
func (h *Handler) HandleOrderPlaced(ctx context.Context, event *events.OrderPlaced) error {
    // Extract tenant from execution context (e.g., JWT, middleware)
    ctxTenantId := tenant.FromContext(ctx)

    // Validate event tenant matches context
    if event.Metadata.TenantId != ctxTenantId {
        return fmt.Errorf("tenant mismatch: event=%s context=%s",
            event.Metadata.TenantId, ctxTenantId)
    }

    // Safe to process
    return h.processOrder(ctx, event)
}
```

**Stream partitioning**:

```go
// NATS subjects include tenant
subject := fmt.Sprintf("events.%s.orders.OrderPlaced", tenantId)

// Kafka partitions by tenant
partition := hash(tenantId) % numPartitions
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
- [ ] **All events defined in proto** (`events/v1/*.proto`)
- [ ] **Outbox pattern for publishing** (no direct `publisher.Publish()`)
- [ ] **Consumers are idempotent** (inbox pattern or natural key upsert)
- [ ] **No broker imports in domain** (only in adapters)
- [ ] **Events carry `tenant_id`** (validated by consumers)
- [ ] **Correlation IDs for tracing** (`correlation_id`, `trace_id`)
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
