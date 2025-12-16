# 526 SRE — Proto Contract Specification

**Status:** Implemented (v1 proto pack) / In progress (capability adoption)
**Parent:** [526-sre-capability.md](526-sre-capability.md)

This document specifies the **proto contracts** for the SRE capability following `496-eda-principles.md` and `509-proto-naming-conventions.md`.

> **No embedded proto text.** Following the lesson from 508 (Scrum protos), this document is a **file index + invariants** specification. Full proto definitions live in the canonical file locations and are the source of truth. Embedding proto text in backlog docs causes drift.

---

## Implementation progress (repo)

- [x] Canonical SRE protos exist under `packages/ameide_core_proto/src/ameide_core_proto/sre/` and pass `buf lint` + `buf breaking`.
- [x] Envelopes are annotated with the shared eventing options spine in `packages/ameide_core_proto/src/ameide_core_proto/common/v1/eventing.proto` (stable type, schema subject, stream ref, partition/idempotency key hints).
- [x] Envelope metadata includes W3C trace context fields (`traceparent`, `tracestate`) in `SreMessageMeta`.
- [x] SDK artifacts are generated via `packages/ameide_core_proto/buf.gen.*` templates; `buf.gen.sdk-go*.yaml` maps `google/api/field_behavior.proto` to `google.golang.org/genproto/googleapis/api/annotations` to avoid duplicate proto registration at runtime.
- [ ] Proto-driven MCP tool exposure annotations are not implemented yet (no canonical `(ameide.mcp.expose)` option, no tool catalog generator).

## Clarifications requested (next steps)

- [ ] Confirm the stable type naming convention for envelopes (e.g., `ameide.sre.domain.intent.v1`) and whether it must match package+message exactly.
- [ ] Confirm the canonical `schema_version` format and update policy (SemVer? capability-local? global?), and how breaking changes are coordinated with projections/processes.
- [ ] Confirm whether `traceparent` is required-by-convention on every emitted event (even if proto doesn’t mark it REQUIRED) and how “missing trace context” is handled.
- [ ] Decide whether Ameide will eventually converge on a single shared broker metadata message (vs per-capability `SreMessageMeta`/`SalesMessageMeta`) and what the migration path would be.

## 1) Package structure (canonical path per 496/509)

```
packages/ameide_core_proto/src/ameide_core_proto/
└── sre/
    ├── core/
    │   └── v1/
    │       ├── sre.proto           # Common types (enums, ResourceRef)
    │       ├── incident.proto      # Incident aggregate messages
    │       ├── alert.proto         # Alert aggregate messages
    │       ├── runbook.proto       # Runbook aggregate messages
    │       ├── slo.proto           # SLO/SLI aggregate messages
    │       ├── health.proto        # HealthCheck, FleetState messages
    │       ├── service.proto       # Service catalog messages
    │       ├── postmortem.proto    # Postmortem aggregate messages
    │       ├── intents.proto       # SreDomainIntent envelope
    │       ├── facts.proto         # SreDomainFact envelope
    │       └── query.proto         # Query service definitions
    └── process/
        └── v1/
            └── process_facts.proto # SreProcessFact envelope
```

**Note:** Path follows `packages/ameide_core_proto/src/ameide_core_proto/**` as required by 496.

---

## 2) Topic families

| Topic | Message Type | Purpose |
|-------|--------------|---------|
| `sre.domain.intents.v1` | `SreDomainIntent` | Commands requesting state changes |
| `sre.domain.facts.v1` | `SreDomainFact` | Immutable events after persistence |
| `sre.process.facts.v1` | `SreProcessFact` | Workflow state transition events |

---

## 3) Envelope metadata requirements (per 496)

All SRE envelopes (`SreDomainIntent`, `SreDomainFact`, `SreProcessFact`) must include:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `message_id` | string | Yes | Unique message identifier |
| `schema_version` | string | Yes | Proto schema version (e.g., "v1.0.0") |
| `tenant_id` | string | Yes | Tenant isolation boundary |
| `occurred_at` | Timestamp | Yes | When the event occurred |
| `producer` | string | Yes | Producing service/component |
| `correlation_id` | string | Yes | Request trace correlation |
| `causation_id` | string | No | ID of the causing message |
| `traceparent` | string | Yes | W3C trace context header value (distributed tracing) |
| `tracestate` | string | No | W3C trace state header value |

**Critical:** `schema_version` is required per 496 for envelope compatibility and tooling.

**Eventing options:** Envelope messages must be annotated with `(ameide_core_proto.common.v1.eventing)` for stable type ids, schema subjects, stream refs, and partition/idempotency key field hints (per `backlog/520-primitives-stack-v2.md`).

---

## 4) Subject fields

Domain intents/facts include `SreSubject`:

| Field | Type | Description |
|-------|------|-------------|
| `aggregate_type` | string | e.g., "Incident", "Alert", "Runbook" |
| `aggregate_id` | string | Aggregate instance ID |
| `aggregate_version` | int64 | Version for idempotent consumption |

Process facts include `SreProcessSubject`:

| Field | Type | Description |
|-------|------|-------------|
| `workflow_type` | string | e.g., "IncidentTriage" |
| `workflow_id` | string | Workflow instance ID |
| `incident_id` | string | Associated incident |

---

## 5) File index: common types (sre.proto)

**Enums:**
- `Severity` — CRITICAL, HIGH, MEDIUM, LOW
- `IncidentStatus` — OPEN, ACKNOWLEDGED, INVESTIGATING, MITIGATING, RESOLVED, CLOSED
- `AlertStatus` — FIRING, RESOLVED, SILENCED, ACKNOWLEDGED
- `HealthStatus` — HEALTHY, DEGRADED, UNHEALTHY, UNKNOWN
- `ServiceTier` — CRITICAL, STANDARD, NON_CRITICAL
- `PostmortemStatus` — DRAFT, IN_REVIEW, PUBLISHED

**Messages:**
- `ResourceRef` — type, id, namespace, cluster, environment
- `SreMessageMeta` — envelope metadata (with `schema_version`)
- `SreSubject` — aggregate identity + version

---

## 6) File index: domain intents (intents.proto)

**Envelope:** `SreDomainIntent` with `meta`, `subject`, and `oneof intent`

**Intent catalog:**

| Intent | Field # | Aggregate |
|--------|---------|-----------|
| `CreateIncidentRequested` | 10 | Incident |
| `ReclassifyIncidentSeverityRequested` | 11 | Incident |
| `AssignIncidentRequested` | 12 | Incident |
| `AddIncidentTimelineEntryRequested` | 13 | Incident |
| `ResolveIncidentRequested` | 14 | Incident |
| `CloseIncidentRequested` | 15 | Incident |
| `AcknowledgeIncidentRequested` | 16 | Incident |
| `TransitionIncidentStatusRequested` | 17 | Incident |
| `IngestAlertRequested` | 20 | Alert |
| `AcknowledgeAlertRequested` | 21 | Alert |
| `CorrelateAlertsToIncidentRequested` | 22 | Alert |
| `CreateRunbookRequested` | 30 | Runbook |
| `ExecuteRunbookRequested` | 31 | Runbook |
| `UpdateRunbookRequested` | 32 | Runbook |
| `CreateSLORequested` | 40 | SLO |
| `RecordSLIWindowRequested` | 41 | SLO (aggregated windows) |
| `UpdateSLORequested` | 42 | SLO |
| `RecordHealthCheckRequested` | 50 | HealthCheck |
| `RecordFleetStateRequested` | 51 | FleetState |
| `CreateServiceRequested` | 60 | Service |
| `UpdateServiceRequested` | 61 | Service |
| `DeleteServiceRequested` | 62 | Service |
| `CreatePostmortemRequested` | 70 | Postmortem |
| `UpdatePostmortemRequested` | 71 | Postmortem |
| `PublishPostmortemRequested` | 72 | Postmortem |

---

## 7) File index: domain facts (facts.proto)

**Envelope:** `SreDomainFact` with `meta`, `subject`, and `oneof fact`

**Fact catalog:**

| Fact | Field # | Aggregate |
|------|---------|-----------|
| `IncidentCreated` | 10 | Incident |
| `IncidentSeverityChanged` | 11 | Incident |
| `IncidentAssigned` | 12 | Incident |
| `IncidentTimelineEntryAdded` | 13 | Incident |
| `IncidentResolved` | 14 | Incident |
| `IncidentClosed` | 15 | Incident |
| `IncidentAcknowledged` | 16 | Incident |
| `IncidentStatusChanged` | 17 | Incident |
| `AlertIngested` | 20 | Alert |
| `AlertAcknowledged` | 21 | Alert |
| `AlertsCorrelatedToIncident` | 22 | Alert |
| `RunbookCreated` | 30 | Runbook |
| `RunbookExecutionCompleted` | 31 | Runbook |
| `RunbookUpdated` | 32 | Runbook |
| `SLOCreated` | 40 | SLO |
| `SLIWindowRecorded` | 41 | SLO |
| `SLOBudgetExhausted` | 42 | SLO |
| `SLOUpdated` | 43 | SLO |
| `HealthCheckRecorded` | 50 | HealthCheck |
| `FleetStateRecorded` | 51 | FleetState |
| `ServiceCreated` | 60 | Service |
| `ServiceUpdated` | 61 | Service |
| `ServiceDeleted` | 62 | Service |
| `PostmortemCreated` | 70 | Postmortem |
| `PostmortemUpdated` | 71 | Postmortem |
| `PostmortemPublished` | 72 | Postmortem |

---

## 8) File index: process facts (process_facts.proto)

**Envelope:** `SreProcessFact` with `meta`, `subject`, and `oneof fact`

**Process fact catalog:**

| Fact | Field # | Workflow |
|------|---------|----------|
| `IncidentTriageStarted` | 10 | IncidentTriage |
| `PatternLookupCompleted` | 11 | IncidentTriage |
| `TriagePhaseCompleted` | 12 | IncidentTriage |
| `RemediationProposed` | 13 | IncidentTriage |
| `RemediationApplied` | 14 | IncidentTriage |
| `VerificationStarted` | 15 | IncidentTriage |
| `VerificationCompleted` | 16 | IncidentTriage |
| `DocumentationRequested` | 17 | IncidentTriage |
| `IncidentTriageCompleted` | 18 | IncidentTriage |

---

## 9) Invariants

1. **All envelopes must include `schema_version`** — required for tooling and compatibility
2. **Naming convention:** Intents use `...Requested` suffix; facts use past tense
3. **Field numbers are stable** — never reuse or renumber once published
4. **Aggregate version required** — enables idempotent downstream consumption
5. **Tenant isolation** — `tenant_id` required on all messages
6. **Proto path structure** — files under `packages/ameide_core_proto/src/ameide_core_proto/sre/`

---

## 10) Acceptance criteria

1. All proto files pass `buf lint` with STANDARD profile
2. All envelopes include `schema_version` in metadata
3. Proto files located at canonical path per 496/509
4. Topic families follow 496 naming conventions
5. Intent names use `...Requested` suffix consistently
6. Fact names use past tense consistently
7. Aggregate version included for idempotent consumption
8. No embedded proto text in backlog docs (file index only)
