# 526 SRE — Proto Contract Specification

**Status:** Draft
**Parent:** [526-sre-capability.md](526-sre-capability.md)

This document specifies the **proto contracts** for the SRE capability following `496-eda-principles.md` and `509-proto-naming-conventions.md`.

> **No embedded proto text.** Following the lesson from 508 (Scrum protos), this document is a **file index + invariants** specification. Full proto definitions live in the canonical file locations and are the source of truth. Embedding proto text in backlog docs causes drift.

---

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

**Critical:** `schema_version` is required per 496 for envelope compatibility and tooling.

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
| `UpdateIncidentSeverityRequested` | 11 | Incident |
| `AssignIncidentRequested` | 12 | Incident |
| `AddIncidentTimelineEntryRequested` | 13 | Incident |
| `ResolveIncidentRequested` | 14 | Incident |
| `CloseIncidentRequested` | 15 | Incident |
| `IngestAlertRequested` | 20 | Alert |
| `AcknowledgeAlertRequested` | 21 | Alert |
| `CorrelateAlertsToIncidentRequested` | 22 | Alert |
| `CreateRunbookRequested` | 30 | Runbook |
| `ExecuteRunbookRequested` | 31 | Runbook |
| `CreateSLORequested` | 40 | SLO |
| `RecordSLIWindowRequested` | 41 | SLO (aggregated windows) |
| `RecordHealthCheckRequested` | 50 | HealthCheck |
| `RecordFleetStateRequested` | 51 | FleetState |

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
| `AlertIngested` | 20 | Alert |
| `AlertAcknowledged` | 21 | Alert |
| `AlertsCorrelatedToIncident` | 22 | Alert |
| `RunbookCreated` | 30 | Runbook |
| `RunbookExecutionCompleted` | 31 | Runbook |
| `SLOCreated` | 40 | SLO |
| `SLIWindowRecorded` | 41 | SLO |
| `SLOBudgetExhausted` | 42 | SLO |
| `HealthCheckRecorded` | 50 | HealthCheck |
| `FleetStateRecorded` | 51 | FleetState |

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
