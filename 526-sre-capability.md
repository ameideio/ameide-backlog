# 526 — SRE Capability (Site Reliability Engineering)

**Status:** In progress (proto + primitives implemented; doc still draft)
**Audience:** Architecture, Platform SRE, platform engineering, operators/CLI, agent/runtime teams
**Scope:** Define **SRE** as a *business capability* with value streams and business processes, supported by Application Services, which are realized by Ameide primitives (Domain/Process/Projection/Integration/UISurface/Agent).

This backlog formalizes the operational workflows described in `525-backlog-first-triage-workflow.md` as a first-class Ameide capability.

**Use with:**
- Capability definition: `backlog/528-capability-definition.md`
- Capability method: `backlog/524-transformation-capability-decomposition.md`
- Capability worksheet: `backlog/530-ameide-capability-design-worksheet.md`
- Integration/EDA contract rules: `backlog/496-eda-principles-v6.md`, `backlog/496-eda-protobuf-ameide.md`
- Primitives pipeline: `backlog/470-ameide-vision.md`, `backlog/477-primitive-stack.md`, `backlog/520-primitives-stack-v6.md`

## Implementation progress (repo)

- [x] SRE proto pack exists and compiles: `packages/ameide_core_proto/src/ameide_core_proto/sre/` (intents/facts/process facts + query surfaces).
- [x] SRE primitives exist and pass repo verification: `bin/ameide primitive verify --kind {domain,process,projection,integration,agent} --name sre --mode repo`.
- [x] SRE MCP adapter exists and passes MCP adapter scaffold checks: `bin/ameide primitive verify --kind integration --name sre-mcp-adapter --mode repo`.
- [x] Incident MVP is implemented as a canonical Domain writer (`primitives/domain/sre`) with outbox + W3C trace context fields persisted/emitted.
- [ ] Projection read models are mostly stubbed (minimal incident search; most query endpoints return empty responses).
- [ ] Process workflows beyond “IncidentTriage started/completed” are not implemented yet (full 525 phase model + approvals + verification pending).
- [ ] Real integrations are not implemented yet (AlertManager/ArgoCD/Kubernetes/paging/ticketing adapters are scaffold-only).
- [ ] No SRE UISurface exists yet (no `primitives/uisurface/sre-*` implementation for an operations console).

## Clarifications requested (next steps)

- [ ] Define the v1 “end-to-end slice” that makes SRE “real” (e.g., Alert ingestion → incident creation → triage workflow start → human approval gate → remediation execution → verification → close).
- [ ] Decide the canonical SRE UI surface: new `primitives/uisurface/sre-operations-console` vs extending an existing UI surface (and what the minimum screens are).
- [ ] Prioritize external adapters for v1 (AlertManager/ArgoCD/Kubernetes first?) and define the minimum contract each adapter must satisfy (idempotency, dedupe, intent-only).
- [ ] Confirm whether on-call scheduling and stakeholder comms/status pages are in-scope for SRE v1 (or explicitly deferred as integrations).
- [ ] Confirm the “definition of done” for “Process primitive implemented” (workflow visibility, approval signals, time-bounded ops, replay/idempotency guarantees).

## Layer header (Strategy + Business + Application realization)

- **Primary ArchiMate layer(s):** Strategy (Capability, Value Stream) + Business (Business Processes, Roles) + Application (realization via Application Components + Services/Interfaces/Events).
- **Secondary layers referenced:** Technology (topology constraints), Implementation & Migration (work packages/phases).
- **Primary element types used:** Capability, Value Stream, Business Process, Application Component, Application Service/Interface/Event, Technology Service, Work Package/Gap.
- **Prohibited unless qualified:** process, service, domain, event (qualify per `529-archimate-alignment-470plus.md`).

---

## 0) Problem

Today "SRE" exists as:

- A **way of working** document (`525-backlog-first-triage-workflow.md`) defining triage workflow and operational discipline.
- Ad-hoc **incident inventories** (`450-argocd-service-issues-inventory.md`) tracking issues as markdown.
- **Policy documents** (`519-gitops-fleet-policy-hardening.md`) defining GitOps hardening approaches.
- Manual **runbook execution** without formal state tracking or automation.

...but we lack a single authoritative backlog that says:

> "SRE is a capability. These are its value streams/business processes, the Application Services that support them, and the primitives that realize those services."

---

## 1) Definition: what the SRE capability is

SRE is the platform's **operate-the-platform capability**: it ensures reliability, observability, and operational excellence for the Ameide platform and tenant workloads.

It provides:

- The **Incident Management System** (system of record) for incidents, alerts, and operational events.
- The **Runbook Registry** for documented procedures and automation playbooks.
- The **SLO/SLI Framework** for defining and tracking service level objectives.
- The **Fleet Health Model** for GitOps state, cluster health, and application status.
- The **EDA-native contracts** that Process primitives and Agents can consume to execute operational workflows deterministically.

---

## 1.1) Strategy: outcomes and value streams (SRE)

Primary outcomes (future state):

- Platform operators can detect, triage, and remediate incidents using a repeatable, backlog-first methodology (per 525).
- "Operational truth" (incidents, runbooks, SLOs) is governable, versioned, queryable, and integrated with the knowledge base.
- Runtime execution is deterministic: Process primitives and Agents drive incident response from contracts and facts, not ad-hoc manual procedures.
- Fleet health is observable and actionable across all environments (local/dev/staging/production).

Value streams:

- **Detect → Alert**: Monitoring systems detect anomalies → create alerts → correlate into potential incidents.
- **Triage → Investigate**: Alert triggers incident → pattern lookup (via projection) → targeted investigation → root cause identification.
- **Remediate → Verify**: Apply fix (GitOps commit) → verify health restored → confirm ArgoCD sync.
- **Document → Learn**: Update backlog/runbook → close incident → feed knowledge base.
- **Define → Measure**: Define SLOs → track SLIs → alert on burn rate → report on reliability.

---

## 1.2) ArchiMate realization chain

```
┌─────────────────────────────────────────────────────────────────────────┐
│ STRATEGY LAYER                                                          │
│                                                                         │
│  ┌───────────────────┐                                                  │
│  │   SRE Capability  │                                                  │
│  └─────────┬─────────┘                                                  │
│            │ realized by                                                │
│            ▼                                                            │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │              Value Streams (Detect→Alert, Triage→Investigate,     │  │
│  │              Remediate→Verify, Document→Learn, Define→Measure)    │  │
│  └─────────────────────────────────┬─────────────────────────────────┘  │
└────────────────────────────────────┼────────────────────────────────────┘
                                     │ realized by
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ BUSINESS LAYER                                                          │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                     Business Processes                            │  │
│  │  (IncidentTriageProcess, ChangeVerificationProcess,               │  │
│  │   SLOBurnAlertProcess, RunbookExecutionProcess)                   │  │
│  └─────────────────────────────────┬─────────────────────────────────┘  │
└────────────────────────────────────┼────────────────────────────────────┘
                                     │ supported by
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ APPLICATION LAYER                                                       │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │              Application Services / Interfaces / Events           │  │
│  │  (IncidentService, AlertService, RunbookService, SLOService,      │  │
│  │   FleetHealthQueryService, KnowledgeIndexQueryService,            │  │
│  │   SreDomainIntents, SreDomainFacts, SreProcessFacts)              │  │
│  └─────────────────────────────────┬─────────────────────────────────┘  │
│                                    │ realized by                        │
│                                    ▼                                    │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │              Application Components (Primitives)                  │  │
│  │  (sre-domain, sre-process, sre-projection, sre-integration,       │  │
│  │   SREAgent, SRECoder, operations-console UISurface)               │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key ArchiMate relationships:**
- Capability **is realized by** Value Streams
- Value Streams **are realized by** Business Processes
- Business Processes **are supported by** Application Services/Interfaces/Events
- Application Services **are realized by** Application Components (primitives)

---

## 2) Core invariants (non-negotiable)

1. **ArchiMate realization chain.** Capability → Value Streams → Business Processes → Application Services → Primitives. Primitives are Application Component realizations, not direct capability decomposition.

2. **Backlog-first triage (per 525).** Search existing knowledge patterns BEFORE deep cluster triage.

3. **No runtime Transformation domain coupling.** Runtime processes/agents query projection-backed knowledge index, never the Transformation domain directly.

4. **Domain is single-writer for facts.** Integrations emit intents/commands; only Domain emits facts via transactional outbox.

5. **No band-aids by default.** Prefer root-cause fixes that improve reproducibility and idempotency.

6. **Time-bounded operations.** No command timeout higher than 5 minutes.

7. **GitOps discipline.** Changes land as commits (backlog + gitops repo), then Argo reconciles.

8. **496-native EDA.** Commands/intents vs facts, outbox for writers, idempotent consumers, schema_version in envelopes.

9. **Tenant isolation and traceability metadata** on all messages.

---

## 3) Capability boundaries and nouns

SRE capability owns:

- **Incident** — operational event requiring investigation/remediation
- **Alert** — monitoring signal that may trigger an incident
- **Runbook** — documented procedure for operational tasks
- **SLO** — Service Level Objective definition
- **SLI** — Service Level Indicator (aggregated measurements, not raw telemetry)
- **HealthCheck** — resource health assessment
- **FleetState** — aggregate GitOps/cluster state
- **Service** — service catalog entry with ownership, on-call schedule, dependencies
- **Postmortem** — structured learning artifact (timeline, root cause, action items)

### Identity axes

- `tenant_id`, `environment_id`, `cluster_id`, `namespace_id`
- `application_id`, `incident_id`, `alert_id`, `runbook_id`, `service_id`, `postmortem_id`

---

## 4) Application Services (supported by primitives)

### 4.1 Domain Services (strongly consistent, minimal query surface)

| Service | Operations | Notes |
|---------|------------|-------|
| `IncidentService` | CreateIncident, ReclassifyIncidentSeverity, AssignIncident, AddIncidentTimelineEntry, AcknowledgeIncident, TransitionIncidentStatus, ResolveIncident, CloseIncident, GetIncident, ListOpenIncidents | Bounded list queries only |
| `AlertService` | IngestAlert, Acknowledge, CorrelateToIncident, GetAlert | No search (projection) |
| `RunbookService` | CreateRunbook, UpdateRunbook, GetRunbook | No search (projection) |
| `SLOService` | CreateSLO, UpdateSLO, GetSLO, ListSLOs | Budget state only |
| `HealthService` | RecordHealthCheck, RecordFleetState, GetCurrentFleetState | Current state only |
| `ServiceCatalogService` | CreateService, UpdateService, DeleteService, GetService | Service ownership |
| `PostmortemService` | CreatePostmortem, UpdatePostmortem, PublishPostmortem, GetPostmortem | Learning artifacts |

### 4.2 Projection Query Services (read-optimized, searchable)

| Service | Operations | Notes |
|---------|------------|-------|
| `FleetHealthQueryService` | GetFleetHealth, ListEnvironmentHealth, GetApplicationHealth | Dashboard reads |
| `IncidentQueryService` | SearchIncidents, GetIncidentTimeline, GetMTTRMetrics | Full search + history |
| `AlertQueryService` | ListActiveAlerts, ListAlertGroups, SearchAlerts | Correlation + search |
| `SLOQueryService` | GetSLOBurndown, GetSLITimeSeries | Time-series charts |
| `RunbookQueryService` | SearchRunbooks, ListRunbooksByTag | Full-text search |
| `ServiceCatalogQueryService` | SearchServices, GetServiceDependencyGraph, GetOnCallForResource | On-call routing |
| `PostmortemQueryService` | SearchPostmortems, ListPostmortemsByRootCause, GetPostmortemsByService | Learning search |
| `KnowledgeIndexQueryService` | SearchPatterns, GetSimilarIncidents, GetSimilarPostmortems, SearchRunbooksBySemantic | Vector-backed semantic search |

### 4.3 Event Contracts

| Topic | Message Type | Purpose |
|-------|--------------|---------|
| `sre.domain.intents.v1` | `SreDomainIntent` | Commands requesting state changes |
| `sre.domain.facts.v1` | `SreDomainFact` | Immutable events after persistence |
| `sre.process.facts.v1` | `SreProcessFact` | Workflow state transition events |

### 4.4 Application Interfaces for Agents (MCP)

SRE is intended to be agent-operable (browser chat, external tooling, and platform agents). MCP is treated as a protocol adapter surface owned by an Integration primitive.

References:
- MCP adapter pattern: `backlog/534-mcp-protocol-adapter.md`
- Read optimizations: `backlog/535-mcp-read-optimizations.md`
- Write optimizations: `backlog/536-mcp-write-optimizations.md`

**Published interface**
- MCP tools: yes (read-heavy; bounded writes)
- MCP resources: yes (read-only)
- MCP prompts: optional (defer unless versioned/promotable)

Starter tool map (illustrative; authoritative source is generated from proto):

| Tool | Kind | Application Service (canonical) | Primitive | Approval | Evidence / audit |
|------|------|----------------------------------|----------|----------|------------------|
| `sre.getFleetHealth` | query | `FleetHealthQueryService.GetFleetHealth` | Projection | no | audit log + query tracing |
| `sre.searchIncidents` | query | `IncidentQueryService.SearchIncidents` | Projection | no | audit log + query tracing |
| `sre.searchPatterns` | query | `KnowledgeIndexQueryService.SearchPatterns` | Projection | no | audit log + query tracing |
| `sre.createIncident` | command | `IncidentService.CreateIncident` | Domain | no (observational record) | `sre.domain.facts.v1` + audit trail projection |
| `sre.resolveIncident` | command | `IncidentService.ResolveIncident` | Domain | yes | `sre.domain.facts.v1` + audit trail projection |

Edge constraints (SRE):

- High-risk infrastructure operations (cluster writes, GitOps commits) are not exposed as direct domain writes by default; they are gated by human approval and executed via controlled workflows/agents.
- All domain writes emit domain facts via outbox; audit/read models are projection-backed.

---

## 5) Application Component realization (primitives)

### 5.1 Domain primitive

**`sre-domain`**: Single-writer bounded context for incidents, alerts, runbooks, SLOs, health checks, fleet state, services, and postmortems. Emits domain facts via transactional outbox. Exposes minimal query APIs (get-by-id, list-by-parent, current state).

See: [526-sre-domain.md](526-sre-domain.md)

### 5.2 Process primitives

- **IncidentTriageProcess**: Implements 525 workflow (Detect → PatternLookup → Triage → Remediate → Verify → Document)
- **ChangeVerificationProcess**: Post-deployment health verification
- **SLOBurnAlertProcess**: Error budget monitoring and escalation

See: [526-sre-process.md](526-sre-process.md)

### 5.3 Projection primitives

- **FleetHealthProjection**: Real-time fleet status for dashboards
- **IncidentTimelineProjection**: Historical incident data, MTTR metrics, searchable
- **SLOBurndownProjection**: Error budget tracking, SLI time-series
- **AlertCorrelationProjection**: Grouped alerts, correlation keys
- **RunbookCatalogProjection**: Searchable runbook catalog
- **ServiceCatalogProjection**: Service ownership, on-call routing, dependency graph
- **PostmortemSearchProjection**: Searchable postmortem archive
- **KnowledgeIndexProjection**: Vector-backed semantic search for patterns, incidents, runbooks, postmortems

See: [526-sre-projection.md](526-sre-projection.md)

### 5.4 Integration primitives

- **KubernetesIntegration**: Direct K8s API access for resource state, events, logs (multi-cluster)
- **ArgoCDIntegration**: Polls ArgoCD, emits `RecordFleetStateRequested` intents
- **ObservabilityIntegration**: Unified OTel/Prometheus/AlertManager/Grafana/Loki/Tempo access
- **TicketingIntegration**: Creates tickets in Jira/GitHub on incident
- **PagingIntegration**: Escalates to PagerDuty/OpsGenie

See: [526-sre-integration.md](526-sre-integration.md)

### 5.5 Agent primitives

- **SREAgent**: Decision agent implementing 525 backlog-first triage. Queries projection services (never domain). Delegates execution to SRECoder.
- **SRECoder**: Execution agent (A2A Server) for kubectl, argocd, git operations.

See: [526-sre-agent.md](526-sre-agent.md)

### 5.6 UISurface primitives

- **OperationsConsole**: Fleet health dashboard, incident management UI, runbook browser

---

## 6) EDA contracts summary

### Domain intents (commands)

- Incident: `CreateIncidentRequested`, `ReclassifyIncidentSeverityRequested`, `AssignIncidentRequested`, `AddIncidentTimelineEntryRequested`, `AcknowledgeIncidentRequested`, `TransitionIncidentStatusRequested`, `ResolveIncidentRequested`, `CloseIncidentRequested`
- Alert: `IngestAlertRequested`, `AcknowledgeAlertRequested`, `CorrelateAlertsToIncidentRequested`
- Runbook: `CreateRunbookRequested`, `ExecuteRunbookRequested`, `UpdateRunbookRequested`
- SLO: `CreateSLORequested`, `RecordSLIWindowRequested` (aggregated windows, not raw), `UpdateSLORequested`
- Health: `RecordHealthCheckRequested`, `RecordFleetStateRequested`
- Service: `CreateServiceRequested`, `UpdateServiceRequested`, `DeleteServiceRequested`
- Postmortem: `CreatePostmortemRequested`, `UpdatePostmortemRequested`, `PublishPostmortemRequested`

### Domain facts (emitted by domain only)

- Incident: `IncidentCreated`, `IncidentSeverityChanged`, `IncidentAssigned`, `IncidentTimelineEntryAdded`, `IncidentAcknowledged`, `IncidentStatusChanged`, `IncidentResolved`, `IncidentClosed`
- Alert: `AlertIngested`, `AlertAcknowledged`, `AlertsCorrelatedToIncident`
- Runbook: `RunbookCreated`, `RunbookUpdated`, `RunbookExecutionCompleted`
- SLO: `SLOCreated`, `SLIWindowRecorded`, `SLOBudgetExhausted`, `SLOUpdated`
- Health: `HealthCheckRecorded`, `FleetStateRecorded`
- Service: `ServiceCreated`, `ServiceUpdated`, `ServiceDeleted`
- Postmortem: `PostmortemCreated`, `PostmortemUpdated`, `PostmortemPublished`

### Process facts

- `IncidentTriageStarted`, `PatternLookupCompleted`, `TriagePhaseCompleted`
- `RemediationProposed`, `RemediationApplied`, `VerificationCompleted`
- `IncidentTriageCompleted`

See: [526-sre-proto.md](526-sre-proto.md)

---

## 7) Technology requirements

- Kubernetes + GitOps + operators — multi-cluster access via KubernetesIntegration
- Postgres/CNPG — Domain state + outbox
- Postgres/CNPG + pgvector — Projection DBs (including semantic index)
- Broker (NATS/Kafka) — facts topic families
- Temporal — Process primitive orchestration
- Prometheus + AlertManager + Loki + Tempo + Grafana — ObservabilityIntegration sources
- ArgoCD — integration source

---

## 8) Implementation phases

1. **Contract foundation** — Define proto surfaces (file index + invariants), scaffold primitives
2. **Domain implementation** — Implement sre-domain with aggregates, outbox, minimal query APIs
3. **Projection implementation** — Fleet health, incident timeline, knowledge index (with vector support)
4. **Integration adapters** — ArgoCD, AlertManager (emitting intents, not facts)
5. **Process workflows** — IncidentTriageProcess (queries projections, not domain)
6. **Agent + UISurface** — SREAgent (queries projections), SRECoder, operations console
7. **Hardening** — Multi-environment, SLO reporting, semantic search tuning

---

## 9) Cross-references

| Document | Relationship |
|----------|--------------|
| [525-backlog-first-triage-workflow.md](525-backlog-first-triage-workflow.md) | Source workflow formalized by this capability |
| [450-argocd-service-issues-inventory.md](450-argocd-service-issues-inventory.md) | Current incident tracking (to be migrated) |
| [519-gitops-fleet-policy-hardening.md](519-gitops-fleet-policy-hardening.md) | Fleet policy constraints |
| [475-ameide-domains.md](475-ameide-domains.md) | Domain portfolio (includes SRE cluster) |
| [528-capability-definition.md](528-capability-definition.md) | Capability framework |
| [496-eda-principles-v6.md](496-eda-principles-v6.md) | Integration/EDA contract rules |
| [527-transformation-capability.md](527-transformation-capability.md) | Transformation capability (no runtime coupling) |

---

## 10) Acceptance criteria

1. SRE described with correct **ArchiMate realization chain** (Capability → Value Streams → Business Processes → Application Services → Primitives)
2. **No runtime Transformation domain coupling**: processes/agents query projection-backed knowledge index
3. **Integrations emit intents only**: domain is single-writer for facts
4. Explicit **SRE EDA contract** (topic families + envelopes) following 496 pattern with `schema_version`
5. 525 triage workflow formalized as **Process primitive** that queries projections
6. **Domain query APIs are minimal** (get-by-id, current state); **search is projection-backed**
7. **KnowledgeIndexProjection** provides vector-backed semantic search for patterns, postmortems
8. **Service catalog owned by SRE**: supports on-call routing via `GetOnCallForResource`
9. **Postmortem as governed artifact**: structured learning loop indexed for pattern matching
10. **Kubernetes + Observability integrations**: direct access to K8s API, Prometheus, Loki, Tempo, Grafana
11. End-to-end slice: Alert → Incident → Pattern lookup (projection) → Triage → Remediation → Verification → Documentation → Postmortem
