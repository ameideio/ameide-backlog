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
- Log field (compat, temporary): `ameide.session_id` (allowed during v3; will be removed in v4)
- Trace correlation fields: `trace_id`, `span_id`

### 2.1.1 Session ID safety + format

- `ameide.session_id` MUST be opaque and MUST NOT contain PII (no user ids/emails/org names/tokens).
- `ameide.session_id` SHOULD be a UUID/ULID (or similar opaque identifier) and SHOULD be short (recommend ≤ 64 chars).
- Tooling SHOULD treat it as a “dev run” identifier and rotate it between independent dev runs.

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

All runtimes MUST configure a composite propagator that includes **W3C Trace Context** and **W3C Baggage** for all supported transports (HTTP and RPC).

For every inbound request/RPC:

1. Extract OTel context (tracecontext + baggage).
2. If baggage contains `ameide.session_id`:
   - set span attribute `ameide.session_id` on the active server span
   - ensure all logs emitted in that request context include:
     - `ameide_session_id` (and optionally `ameide.session_id`)
     - `trace_id`, `span_id`

For every outbound request/RPC:

- inject/propagate tracecontext + baggage (including `ameide.session_id`)

For async messaging / event buses:

- producers MUST propagate `ameide.session_id` in message headers/metadata AND include `session_id` in the envelope payload (when an envelope exists).
- consumers MUST copy the session id (header/envelope) onto spans/logs exactly as for HTTP/RPC.

### 4.1 “No hand-rolled telemetry” rule

Services and SDKs must not re-implement this contract ad-hoc. Use the shared helpers:

- TypeScript: `packages/ameide_telemetry_ts/` (`@ameideio/ameide-telemetry-ts`)
- Python: `packages/ameide_telemetry_py/` (`ameide-telemetry-py`, import `ameide_telemetry`)
- Go: `packages/ameide_telemetry_go/` (`github.com/ameideio/ameide/packages/ameide_telemetry_go`)

Helper libraries MUST provide (per language/runtime):

1. inbound middleware/interceptor (extract + set span attribute + attach log fields)
2. outbound middleware/interceptor (inject)
3. logger hook/adapter (adds `ameide_session_id`, `trace_id`, `span_id` from active context/span)

Each runtime SHOULD have “contract tests” to prevent drift:

- given an inbound request with baggage, the server span has `ameide.session_id`
- a log emitted under that request context includes `ameide_session_id`, `trace_id`, `span_id`

### 4.2 Enforcement (repo gate)

Phase 0 of `ameide test` is the repo gate front door.

Enforced: `ObservabilityContract334v3` runs in the repo gate and fails if a service is missing the required helper + propagator install + session correlation (baggage → span attribute + log fields).

Implementation lives in `packages/ameide_coding_helpers/verify/` (go test gate).

## 5) Loki (logs)

Rules:

- Keep Loki labels **low-cardinality** (service/env/namespace/etc).
- Do not put session ids or trace ids into Loki labels (avoid high-cardinality labels).
- All services MUST emit structured logs (JSON) for request-scoped events.
- JSON logs MUST include a timestamp, a level, and a message field (key names may vary), plus `ameide_session_id`, `trace_id`, `span_id`.
- Target state: also attach `ameide_session_id`, `trace_id`, `span_id` as Loki **structured metadata** (when ingestion supports it).

GitOps status (dev/staging/prod):

- Loki has `allow_structured_metadata: true` enabled.
- Alloy log tailing extracts `ameide_session_id`, `trace_id`, `span_id` from JSON logs and attaches them as structured metadata.
- OTLP logs (e.g., Envoy Gateway access logs) are received by the OTel Collector and exported to Loki via Loki OTLP HTTP ingestion.

Query model (CLI + Grafana):

- Minimum (JSON payload): `{...} | json | ameide_session_id="<session>"`
- Target (structured metadata): `{...} | ameide_session_id="<session>"`

Operational note:

- If any environments still rely on Promtail stages for parsing/enrichment, plan a migration to Grafana Alloy or an OpenTelemetry Collector-based logs pipeline.

## 6) Tempo (traces)

Search must target **span attributes**:

- TraceQL: `{ span.ameide.session_id="<session>" }`

Optionally narrow by service:

- `{ span.ameide.session_id="<session>" && resource.service.name="platform-service" }`

Operational notes:

- CLI search uses Tempo `/api/search?q=<TraceQL>`; `q` MUST be URL-encoded TraceQL.
- If traces are not found for a session id, the first thing to verify is: services are copying baggage → span attributes (session id must exist as `span.ameide.session_id`).

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
