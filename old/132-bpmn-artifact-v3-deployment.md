# Deployment, Ops & Security

## 1) Components
- **Web App**: bpmn-js editor + Rules Gate + gRPC-Web client.
- **Envoy**: terminates gRPC-Web; routes to internal gRPC.
- **Command Service**: validation, projection, event append, snapshotting.
- **Moddle Sidecar**: `bpmn-moddle` service (stateless).
- **XSD Validator**: Python `lxml` service (stateless).
- **Event Store**: append-only (e.g., Postgres streams / Kafka+DB).
- **Read Models**: PostgreSQL + Neo4j/AGE.
- **Agent Runtime**: submits proposals (HITL by policy).

## 2) Envoy (edge) — essentials
- HTTP/2, gRPC-Web filter enabled.
- mTLS to backend gRPC.
- Headers propagated: `x-tenant-id`, `x-actor-id`, `x-request-id`, `x-run-id`.

## 3) Scalability
- Scale **Command Service** horizontally; enforce idempotency via `transaction_id`.
- Make **Moddle/XSD** sidecars autoscalable; cold-start tolerant.
- Snapshots reduce replay time; tune thresholds per diagram size.

## 4) Observability
- **Structured logs** with `diagram_id`, `tenant_id`, `version`, `transaction_id`.
- **Metrics**: accept latency (p50/p95), XSD error rate, lint warning density, event append rate, snapshot size.
- **Tracing**: proposal → validation → append → ack (W3C tracecontext).

## 5) Security & Tenancy
- **AuthN**: OIDC/JWT at edge; short-lived tokens.
- **AuthZ**: RBAC/ABAC evaluated in Command Service.
- **Row-level** or stream-level isolation per `tenant_id`.
- **Input validation**: schema-checked proposals; hard caps on batch size, XML size, waypoint counts.

## 6) Backups & Recovery
- Event store is primary; schedule **logical backups** of streams.
- Periodic **snapshot archival** (compressed XML).
- Disaster recovery: rebuild read models from streams + latest snapshot.

## 7) Compatibility & Migration
- Version events; support **projection version** flag.
- Migrate layout independently of semantics.
- Keep **id_map** behavior stable across releases.

## 8) SLAs / SLOs (suggested)
- Accept latency p95 ≤ 300 ms for small changes.
- XSD validation budget ≤ 80 ms average (<100 KB XML).
- Snapshot write ≤ 500 ms p95 (async).

## 9) Test Strategy (architectural)
- **Contract tests**: Propose → Applied/Rejected (codes, id_map, version).
- **Projection fidelity**: round-trip events → XML → XSD valid.
- **Rules Gate**: canExecute denies without permit; allows with permit.
- **Throughput**: sustained append & snapshot under load.

## 10) Rollout Plan (phases)
1) **Gate & Transport**: rules gate + Propose/Ack path live.
2) **Validation & Store**: XSD+lint pipeline, event append, snapshots.
3) **Apply Strategy**: programmatic modeling path stable; one undo step/tx.
4) **Read Models**: SQL + Graph consumers; analytics endpoints.
5) **Agents (HITL)**: proposal submission, reviewer approval, audit logs.

## 11) Operational Runbook (abridged)
- **Hot issue**: `VERSION_MISMATCH` spikes → investigate concurrent edits; adjust UX to propose on dragend.
- **XSD error bursts**: check model templates, engine-specific profiles.
- **Slow accept**: profile moddle sidecar and XSD service; scale; enable snapshot in ack for large diffs.