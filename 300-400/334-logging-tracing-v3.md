# Observability Contract – v3 (OpenTelemetry + Grafana Loki/Tempo)

Status as of **2026-01-20**.

This document is **authoritative** and supersedes:

- `backlog/300-400/334-logging-tracing-v2.md` (**v2**, which had precedence over v1)
- `backlog/300-400/334-logging-tracing.md` (**v1**)

## 1) Goal

Make “find everything for this dev session” a first-class workflow:

- one dev session id is the join key across **requests → spans → logs**
- engineers and agents query **Loki + Tempo** (not `kubectl logs`) for the normal inner loop

## 2) Telemetry contract (normative)

### 2.1 Canonical keys

Propagation:

- W3C Baggage key: `ameide.session_id`

Tracing:

- Span attribute (request-scoped): `ameide.session_id`

Logging:

- Log field (canonical): `ameide_session_id`
- Log field (compat): `ameide.session_id` (optional but recommended during transition)
- Trace correlation fields: `trace_id`, `span_id`

### 2.2 Semantics (why this matters)

- **Baggage is propagation**, not storage: it must be **copied** onto spans/logs by service code if we want reliable searching/correlation.
- `ameide.session_id` is **request-scoped**, so it must be a **span attribute**, not a resource attribute.

## 3) Protos in this model

Proto-first still applies — but protos are for **payload contracts**, while propagation remains in **OTel headers**.

Requirements:

- Any service-level “request context” proto should include a `session_id` field (if it carries other request identifiers like `tenant_id`, `user_id`, `request_id`).
- Event envelopes (EDA) should also carry `session_id` for offline correlation.

Mapping:

- proto field `session_id` ↔ OTel baggage `ameide.session_id` ↔ span attribute `ameide.session_id` ↔ log field `ameide_session_id`.

## 4) Runtime requirements (normative)

For every inbound request/RPC:

1. Extract OTel context (tracecontext + baggage).
2. If baggage contains `ameide.session_id`:
   - set span attribute `ameide.session_id` on the active server span
   - ensure all logs emitted in that request context include:
     - `ameide_session_id` (and optionally `ameide.session_id`)
     - `trace_id`, `span_id`

For every outbound request/RPC:

- inject/propagate tracecontext + baggage (including `ameide.session_id`)

### 4.1 “No hand-rolled telemetry” rule

Services and SDKs must not re-implement this contract ad-hoc. Use the shared helpers:

- TypeScript: `packages/ameide_telemetry_ts/` (`@ameideio/ameide-telemetry-ts`)
- Python: `packages/ameide_telemetry_py/` (`ameide-telemetry-py`, import `ameide_telemetry`)
- Go: `packages/ameide_telemetry_go/` (`github.com/ameideio/ameide/packages/ameide_telemetry_go`)

## 5) Loki (logs)

Rules:

- Keep Loki labels **low-cardinality** (service/env/namespace/etc).
- Do not put session ids or trace ids into Loki labels.
- Always include `ameide_session_id`, `trace_id`, `span_id` in the JSON log payload.
- Target state: also attach `ameide_session_id`, `trace_id`, `span_id` as Loki **structured metadata** (when ingestion supports it).

Query model (CLI + Grafana):

- Full-stack by dev session: `{...} | json | ameide_session_id="<session>"`

## 6) Tempo (traces)

Search must target **span attributes**:

- TraceQL: `{ span.ameide.session_id="<session>" }`

Optionally narrow by service:

- `{ span.ameide.session_id="<session>" && resource.service.name="platform-service" }`

## 7) CLI dev loop

The CLI is a pure consumer of Loki/Tempo:

- `ameide dev logs`:
  - backfill via Loki `query_range`
  - follow via Loki `tail` (WebSocket)
- `ameide dev traces`:
  - Tempo search via `/api/search?q=<TraceQL>`

## 8) Definition of done (per service)

- `ameide.session_id` propagated via baggage.
- `ameide.session_id` copied onto spans as a span attribute.
- Logs include `ameide_session_id`, `trace_id`, `span_id` for all request-scoped log events.
