# 526 SRE — Proto Contract Specification

**Status:** Draft
**Parent:** [526-sre-capability.md](526-sre-capability.md)

This document specifies the **proto contracts** for the SRE capability following `496-eda-principles.md` and `509-proto-naming-conventions.md`.

---

## 1) Package structure

```
packages/ameide_core_proto/
└── sre/
    ├── core/
    │   └── v1/
    │       ├── sre.proto           # Common types
    │       ├── incident.proto      # Incident aggregate
    │       ├── alert.proto         # Alert aggregate
    │       ├── runbook.proto       # Runbook aggregate
    │       ├── slo.proto           # SLO/SLI aggregates
    │       ├── health.proto        # HealthCheck, FleetState
    │       ├── intents.proto       # SreDomainIntent envelope
    │       ├── facts.proto         # SreDomainFact envelope
    │       └── query.proto         # Query services
    └── process/
        └── v1/
            └── process_facts.proto # SreProcessFact envelope
```

---

## 2) Topic families

| Topic | Message Type | Purpose |
|-------|--------------|---------|
| `sre.domain.intents.v1` | `SreDomainIntent` | Commands requesting state changes |
| `sre.domain.facts.v1` | `SreDomainFact` | Immutable events after persistence |
| `sre.process.facts.v1` | `SreProcessFact` | Workflow state transition events |

---

## 3) Common types (sre.proto)

```protobuf
syntax = "proto3";
package ameide_core_proto.sre.core.v1;

enum Severity {
  SEVERITY_UNSPECIFIED = 0;
  SEVERITY_CRITICAL = 1;
  SEVERITY_HIGH = 2;
  SEVERITY_MEDIUM = 3;
  SEVERITY_LOW = 4;
}

enum IncidentStatus {
  INCIDENT_STATUS_UNSPECIFIED = 0;
  INCIDENT_STATUS_OPEN = 1;
  INCIDENT_STATUS_ACKNOWLEDGED = 2;
  INCIDENT_STATUS_INVESTIGATING = 3;
  INCIDENT_STATUS_MITIGATING = 4;
  INCIDENT_STATUS_RESOLVED = 5;
  INCIDENT_STATUS_CLOSED = 6;
}

enum AlertStatus {
  ALERT_STATUS_UNSPECIFIED = 0;
  ALERT_STATUS_FIRING = 1;
  ALERT_STATUS_RESOLVED = 2;
  ALERT_STATUS_SILENCED = 3;
  ALERT_STATUS_ACKNOWLEDGED = 4;
}

enum HealthStatus {
  HEALTH_STATUS_UNSPECIFIED = 0;
  HEALTH_STATUS_HEALTHY = 1;
  HEALTH_STATUS_DEGRADED = 2;
  HEALTH_STATUS_UNHEALTHY = 3;
  HEALTH_STATUS_UNKNOWN = 4;
}

message ResourceRef {
  string type = 1;
  string id = 2;
  string namespace = 3;
  string cluster = 4;
  string environment = 5;
}

message SreMessageMeta {
  string message_id = 1;
  string tenant_id = 2;
  google.protobuf.Timestamp occurred_at = 3;
  string producer = 4;
  string correlation_id = 5;
  string causation_id = 6;
}

message SreSubject {
  string aggregate_type = 1;
  string aggregate_id = 2;
  int64 aggregate_version = 3;
}
```

---

## 4) Domain intents (intents.proto)

```protobuf
message SreDomainIntent {
  SreMessageMeta meta = 1;
  SreSubject subject = 2;

  oneof intent {
    // Incident
    CreateIncidentRequested create_incident = 10;
    UpdateIncidentSeverityRequested update_incident_severity = 11;
    AssignIncidentRequested assign_incident = 12;
    AddIncidentTimelineEntryRequested add_timeline_entry = 13;
    ResolveIncidentRequested resolve_incident = 14;
    CloseIncidentRequested close_incident = 15;

    // Alert
    IngestAlertRequested ingest_alert = 20;
    AcknowledgeAlertRequested acknowledge_alert = 21;
    CorrelateAlertsToIncidentRequested correlate_alerts = 22;

    // Runbook
    CreateRunbookRequested create_runbook = 30;
    ExecuteRunbookRequested execute_runbook = 31;

    // SLO
    CreateSLORequested create_slo = 40;
    RecordSLIMeasurementRequested record_sli_measurement = 41;

    // Health
    RecordHealthCheckRequested record_health_check = 50;
    RecordFleetStateRequested record_fleet_state = 51;
  }
}
```

---

## 5) Domain facts (facts.proto)

```protobuf
message SreDomainFact {
  SreMessageMeta meta = 1;
  SreSubject subject = 2;

  oneof fact {
    // Incident
    IncidentCreated incident_created = 10;
    IncidentSeverityChanged incident_severity_changed = 11;
    IncidentAssigned incident_assigned = 12;
    IncidentTimelineEntryAdded incident_timeline_entry_added = 13;
    IncidentResolved incident_resolved = 14;
    IncidentClosed incident_closed = 15;

    // Alert
    AlertIngested alert_ingested = 20;
    AlertAcknowledged alert_acknowledged = 21;
    AlertsCorrelatedToIncident alerts_correlated = 22;

    // Runbook
    RunbookCreated runbook_created = 30;
    RunbookExecutionCompleted runbook_execution_completed = 31;

    // SLO
    SLOCreated slo_created = 40;
    SLIMeasurementRecorded sli_measurement_recorded = 41;
    SLOBudgetExhausted slo_budget_exhausted = 42;

    // Health
    HealthCheckRecorded health_check_recorded = 50;
    FleetStateRecorded fleet_state_recorded = 51;
  }
}
```

---

## 6) Process facts (process_facts.proto)

```protobuf
message SreProcessFact {
  SreMessageMeta meta = 1;
  SreProcessSubject subject = 2;

  oneof fact {
    IncidentTriageStarted incident_triage_started = 10;
    BacklogLookupCompleted backlog_lookup_completed = 11;
    TriagePhaseCompleted triage_phase_completed = 12;
    RemediationProposed remediation_proposed = 13;
    RemediationApplied remediation_applied = 14;
    VerificationStarted verification_started = 15;
    VerificationCompleted verification_completed = 16;
    DocumentationRequested documentation_requested = 17;
    IncidentTriageCompleted incident_triage_completed = 18;
  }
}

message SreProcessSubject {
  string workflow_type = 1;
  string workflow_id = 2;
  string incident_id = 3;
}
```

---

## 7) Acceptance criteria

1. All proto files pass `buf lint` with STANDARD profile
2. All messages have required validation constraints
3. Topic families follow 496 naming conventions
4. Envelope metadata includes all required fields
5. Aggregate version included for idempotent consumption
