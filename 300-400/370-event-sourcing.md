# backlog/370 – Event-Sourcing Exploration

**Status:** Draft  
**Owner:** Platform DX / ERP Architecture  
**Linked work:** backlog/367 (methodology/runtime), backlog/303 (metamodel), backlog/345 (CI/CD), backlog/362 (guardrails)

---

## Motivation

Teams asked whether shifting the platform to a fully event‑sourced architecture would improve scalability, auditability, and storage costs compared to the current Postgres-first design. This backlog captures the evaluation criteria, target shape, and prerequisites so we can make an informed call instead of drifting toward an incremental rewrite.

Key drivers:

- Immutable history / replay for governance-heavy workflows (Scrum, SAFe, TOGAF, release attestations).
- Native hooks for watchers (Temporal), automation plans, and analytics pipelines (CDC/warehouse).
- Potential to keep hot projections small while pushing long-term history to cheap object storage.

## Current Position

- The platform already standardizes on per-method Buf modules (for example, `ameide_core_proto.transformation.scrum.v1`, `ameide_core_proto.transformation.safe.v1`, `ameide_core_proto.transformation.togaf.v1`, `ameide_core_proto.governance.v1`) plus a shared Connect ingress, with package/topic naming governed by [509-proto-naming-conventions.md](../509-proto-naming-conventions.md).
- Postgres hosts shared schemas; we rely on tenant/org scoping, partitioning plans, CDC to warehouse, and Temporal for long-running processes.
- Auditability is handled via versioned tables and soon-to-be Attestation records (`governance.v1`), not append-only event logs.
- Infrastructure plan favors “shared cluster + partitioning/sharding for large tenants” over per-tenant databases to keep ops costs low.

Given this baseline, we need to weigh the benefits of adding event streams against the cost of rewriting all persistence logic.

---

## Objectives

1. **Define an event contract per domain** – Extend Buf modules with explicit event payloads (`ScrumBacklogItemCreated`, `SafePiObjectiveCommitted`, `TogafPhaseAdvanced`, `GovernanceAttestationRegistered`, etc.) and envelopes for streaming APIs.
2. **Decide on the event store** – Evaluate Kafka + compacted topics, Postgres logical decoding, or Postgres-compatible distributed SQL (Citus, Yugabyte, Cockroach) with CDC to an object store.
3. **Prototype projections** – Pick one domain (e.g., Scrum backlog) and implement: append-only journal, projection tables, replay tooling, Temporal watcher consuming events.
4. **Assess migration strategy** – Define how to backfill existing state into event streams (genesis events, synthetic snapshots, or dual-write via outbox) without downtime.
5. **Cost & operability model** – Compare storage/compute costs (event log + projections) vs. current Postgres-only approach, including operational overhead (schema evolution, replays, monitoring).

---

## Scope & Non-Scope

In Scope:
- Event model design for Scrum, SAFe, TOGAF, Governance, Automation.
- Outbox/Inbox pattern changes needed for reliable event emission.
- Projection services, replay tooling, and Temporal subscriber updates.
- Storage evaluation (Kafka/S3, EventStoreDB, Citus partitions, CDC pipelines).

Out of Scope (handled in sibling work):
- Replacing Postgres entirely (unless evaluation proves necessary).
- Immediate migration of all services—this backlog delivers the decision framework and a pilot.
- Business-domain redefinitions (handled in backlog/367).

---

## Proposed Architecture (Target Shape)

1. **Command layer** (unchanged): Connect RPCs (Scrum domain query/intent services, GovernanceService, WorkflowsService, etc.) validate commands and write to local ACID storage.
2. **Event emission**: Each command writes to an outbox (`*_events_outbox`) containing serialized Buf event payloads; a relay publishes to the event store (Kafka topic per domain).
3. **Event store**: Append-only topics stored in Kafka (short retention) + object storage (long retention). For low-traffic tenants we can leverage Postgres logical decoding instead.
4. **Projections**: Materialized Postgres tables (current backlog state, PI dashboards, release bundles) are fed by event consumers. Projections can be rebuilt by replaying topics from offset 0.
5. **Temporal integration**: Watchers listen to event streams (via Kafka consumer groups). Long-running workflows emit compensating events (`ImpedimentFiled`, `TimerExpired`) back into the log.
6. **Analytics**: CDC from the event store (or replay) populates the warehouse; no need for bespoke history tables.

Alternate path (if event sourcing is too heavy): keep Postgres as the source of truth, add CDC streams (Debezium) to mimic event feeds, and archive partitions for cost control. This backlog should compare both paths.

---

## Deliverables

1. **Event schema draft** – Buf PR adding event messages/envelopes per domain, plus streaming RPC definitions.
2. **Outbox pattern** – Shared library + migrations for event outbox tables, publisher service hooked into existing infra (NATS/Kafka + S3 for archiving).
3. **Pilot implementation** – Scrum backlog events with dual-write (commands + events), projection service, and Temporal watcher updated to consume events.
4. **Cost/scale analysis** – Documented comparison of storage, throughput, and ops impact (current Postgres vs. event-sourced vs. CDC-only).
5. **Decision memo** – Recommendation (adopt event sourcing, hybrid CDC, or stay Postgres-first) with phased rollout plan if applicable.

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Operational complexity** – managing Kafka/EventStore + projections + replays | Start with a single domain pilot, automate replay tooling, leverage managed Kafka/S3 where possible |
| **Data migration** – historic state lacks events | Generate genesis events from snapshots, dual-write (table + event) during migration window |
| **Cost creep** – storing all events forever | Implement retention + archival policies (compact topics, push old segments to S3, keep projections slim) |
| **Consistency gaps** – commands succeed but events fail | Use transactional outbox with idempotent publishers and retries; add monitoring for lag |
| **Schema evolution** – breaking changes to event payloads | Version events (`oneof + reserved fields`), document upgrade paths, enforce Buf breaking-change rules |

---

## Open Questions

1. Which event store aligns best with our stack (Kafka, EventStoreDB, Postgres logical decoding)?
2. Do we need per-tenant topics/partitions for isolation, or is tagging by tenant_id enough?
3. How will release governance (attestations) consume events vs. current direct DB reads?
4. Can existing CDC → warehouse pipelines be reused (Debezium) instead of full event sourcing?
5. What’s the impact on Temporal workers (rewriting to event-driven vs. RPC-triggered)?

---

## Next Steps

1. Draft event payloads + streaming RPCs in Buf modules (DX + domain leads).  
2. POC: implement outbox + Kafka publisher for Scrum backlog commands; build projection + replay job.  
3. Measure write/read performance, storage footprint, and ops burden vs. status quo.  
4. Produce decision memo (adopt, hybrid, or defer) and, if positive, create execution backlogs per domain.



Current State

Graph service handlers create/update rows directly in PostgreSQL; the same handler even inserts a matching row in graph.elements with raw SQL (services/graph/src/graph/service.ts (lines 1004-1034)), and the only persistence primitives defined in Flyway are normalized tables such as graph.repositories, graph.repository_nodes, graph.elements, etc., not append-only streams (db/flyway/sql/graph/V1__initial_schema.sql (lines 10-118)).
Platform service logic lives in the Connect handler and issues INSERT INTO platform.organizations (...) and membership writes inside the same transaction (services/platform/src/organizations/service.ts (lines 362-438)); its schema is likewise a conventional set of tables for organizations, teams, memberships, and invitations (db/flyway/sql/platform/V1__initial_schema.sql (lines 1-120)).
Threads service is a thin HTTP/2 server over a pg pool (services/threads/src/db.ts (lines 1-74)) and persists messages by inserting into "ThreadMessage"/"Thread" tables with JSON blobs (services/threads/src/threads/service.ts (lines 730-775)); the Flyway migration explicitly re-baselines the database to just those state tables and drops any prior “thread_events” artifacts (db/flyway/sql/threads/V2__threads_rebaseline.sql (lines 1-118)).
Workflows is documented as a Connect service backed by a Postgres schema named workflows_service (services/workflows/README.md (lines 1-15)), and its migrations define normalized definitions/versions tables plus run logs rather than streams (db/flyway/sql/workflows/V1__workflow_service_schema.sql (lines 1-55)).
Inference service persists LangGraph checkpoints through AsyncPostgresSaver (services/inference/app.py (lines 1-110)) and its README explicitly lists PostgreSQL as the state store (services/inference/README.md (lines 1-33)), with Flyway installing the LangGraph checkpoint tables (db/flyway/sql/langgraph/V1__create_checkpoint_tables.sql (lines 1-44)).
Despite this relational reality, the top-level docs position the platform as “event-sourced” (README.md (line 3)), and the services overview describes domain services as “backed by event-sourced persistence” wired via PostgresEventStore (services/README.md (lines 7-29), services/README.md (lines 87-92)).
Gaps Toward Event Sourcing

There is no actual event-store module in the workspace: pyproject.toml advertises a packages/ameide_core_storage workspace member (pyproject.toml (lines 20-43)), tests still import ameide_core_storage.eventstore.Event (tests/shared/event_test_utils.py (lines 1-36)), and the CLI has TODOs where it should call an event-store API (packages/ameide_core_cli/internal/commands/events.go (lines 38-90)), but the implementation directory is absent, leaving no shared append/stream/read abstractions.
Business rules and SQL live in the service handlers themselves (see organization creation logic above and thread message persistence in services/threads/src/threads/service.ts (lines 730-775)), so there are no aggregate classes that could be rehydrated from events or replay logic that could apply historical changes.
All Flyway migrations build the current-source-of-truth tables (graph elements, platform orgs, threads, workflows, LangGraph checkpoints) with no notion of append-only streams, version buckets, or projection tables; even the one outbox table under the graph schema (db/flyway/sql/graph/V1__initial_schema.sql (lines 73-95)) is populated only through a helper that enqueues JSON payloads (services/graph/src/graph/events/outbox.ts (lines 1-34)) with no corresponding dispatcher.
Documentation emphasizes an event-sourcing contract for packages (packages/README.md (lines 1-40)), yet services instantiate pg clients instead of injecting an EventStore, so packages cannot be reused across runtimes or tested independently as promised.
Operational tooling is missing: the CLI “events” command prints hard-coded mock events because there is no gRPC/HTTP endpoint to call (packages/ameide_core_cli/internal/commands/events.go (lines 38-124)), and without a real stream store there is nothing to drive projections, CDC, or downstream processors.
Migration Needs

Stand up a real event-store platform (decide on Postgres append-only tables, EventStoreDB, Kafka + compaction, etc.), expose it over gRPC/HTTP, and ship client SDKs for TypeScript, Python, and Go so services can append and read streams consistently (the contract sketched in services/README.md (lines 7-29) becomes actionable only once this exists).
Model aggregates and domain events per bounded context: e.g., Repository/Element/Relationship in graph (services/graph/src/graph/service.ts (lines 1004-1034)), Organization/Membership in platform (services/platform/src/organizations/service.ts (lines 362-438)), Thread in threads (services/threads/src/threads/service.ts (lines 730-775)), workflow definitions/runs (db/flyway/sql/workflows/V1__workflow_service_schema.sql (lines 1-55)), and agent conversations/checkpoints for inference (services/inference/app.py (lines 1-120)).
Refactor services so they load aggregates by replaying events, execute commands that emit new events, and persist via the shared store—pushing validation/business logic down into packages instead of inline SQL; this also means generating buffer schemas for the new command/event envelopes (extend packages/ameide_core_proto).
Build projection workers (could live in the services or as separate runtimes) that subscribe to event streams and hydrate the existing read models (graph.elements, platform.organizations, etc.), allowing you to keep the current query APIs while moving writes to the event store; the existing tables become projections instead of truth.
Plan data migration carefully: backfill historical events from current tables (e.g., traverse graph.elements to emit ElementCreated followed by ElementUpdated for each version, using metadata from graph.element_versions), support dual-write/dual-read during cutover, and add replay scripts plus verification to ensure projections match pre-migration state.
Deliver supporting tooling/tests: finish the CLI commands, create replay/repair jobs, update integration tests to seed aggregates via events instead of fixture rows, and instrument the new store with metrics/alerts (today’s tests/shared/event_test_utils.py (lines 1-80) can evolve into real fixtures once the package exists).
Risks & Open Questions

Multi-tenant concurrency: every schema today includes tenant_id columns (db/flyway/sql/graph/V1__initial_schema.sql (lines 10-70), db/flyway/sql/platform/V1__initial_schema.sql (lines 1-60)), so stream naming/versioning must enforce tenant isolation and optimistic concurrency per aggregate.
Storage growth & retention: translating tables like graph.element_versions and thread histories into event streams will increase volume; you’ll need retention, snapshotting, and archival policies before enabling replay at scale.
Cross-service consistency: inference already persists LangGraph checkpoints separately (services/inference/app.py (lines 1-110), db/flyway/sql/langgraph/V1__create_checkpoint_tables.sql (lines 1-44)); decide whether those checkpoints remain independent or are folded into the global event store to avoid divergent histories between agent state and thread persistence.
Operational maturity: nothing currently consumes the graph.repository_outbox, so introducing event sourcing also requires deploying background processors, monitoring, and remediation workflows for stuck projections or replay failures.
