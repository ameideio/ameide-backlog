# 526 — SRE Capability (Site Reliability Engineering)

**Status:** Draft (normative once accepted)
**Audience:** Architecture, Platform SRE, platform engineering, operators/CLI, agent/runtime teams
**Scope:** Define **SRE** as a *business capability* implemented as Ameide primitives (Domain/Process/Projection/Integration/UISurface/Agent), with 496-native EDA contracts.

This backlog formalizes the operational workflows described in `525-backlog-first-triage-workflow.md` as a first-class Ameide capability.

**Use with:**
- Capability definition: `backlog/528-capability-definition.md`
- Capability method: `backlog/524-transformation-capability-decomposition.md`
- Capability worksheet: `backlog/530-ameide-capability-design-worksheet.md`
- EDA contract rules: `backlog/496-eda-principles.md`, `backlog/496-eda-protobuf-ameide.md`
- Primitives pipeline: `backlog/470-ameide-vision.md`, `backlog/477-primitive-stack.md`, `backlog/520-primitives-stack-v2.md`

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

> "SRE is a capability. These are its domains/processes/surfaces/integrations, and these are its EDA contracts."

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
- **Triage → Investigate**: Alert triggers incident → backlog lookup (per 525) → targeted investigation → root cause identification.
- **Remediate → Verify**: Apply fix (GitOps commit) → verify health restored → confirm ArgoCD sync.
- **Document → Learn**: Update backlog/runbook → close incident → feed knowledge base.
- **Define → Measure**: Define SLOs → track SLIs → alert on burn rate → report on reliability.

---

## 2) Core invariants (non-negotiable)

1. **Capability is realized by primitives.** SRE is not a monolith; it is realized via Domain/Process/Projection/Integration/UISurface/Agent application components.

2. **Backlog-first triage (per 525).** Search existing backlog guidance BEFORE deep cluster triage.

3. **No band-aids by default.** Prefer root-cause fixes that improve reproducibility and idempotency.

4. **Time-bounded operations.** No command timeout higher than 5 minutes.

5. **GitOps discipline.** Changes land as commits (backlog + gitops repo), then Argo reconciles.

6. **496-native EDA.** Commands/intents vs facts, outbox for writers, idempotent consumers.

7. **Tenant isolation and traceability metadata** on all messages.

---

## 3) Capability boundaries and nouns

SRE capability owns:

- **Incident** — operational event requiring investigation/remediation
- **Alert** — monitoring signal that may trigger an incident
- **Runbook** — documented procedure for operational tasks
- **SLO** — Service Level Objective definition
- **SLI** — Service Level Indicator measurement
- **HealthCheck** — resource health assessment
- **FleetState** — aggregate GitOps/cluster state

### Identity axes

- `tenant_id`, `environment_id`, `cluster_id`, `namespace_id`
- `application_id`, `incident_id`, `alert_id`, `runbook_id`

---

## 4) Application realization (primitive decomposition)

### 4.1 Domain primitives

**`sre-domain`**: Own incidents, alerts, runbooks, SLOs/SLIs, health checks, fleet state. Emit domain facts via transactional outbox.

See: [526-sre-domain.md](526-sre-domain.md)

### 4.2 Process primitives

- **IncidentTriageProcess**: Implements 525 workflow (Detect → BacklogLookup → Triage → Remediate → Verify → Document)
- **ChangeVerificationProcess**: Post-deployment health verification
- **SLOBurnAlertProcess**: Error budget monitoring and escalation

See: [526-sre-process.md](526-sre-process.md)

### 4.3 Projection primitives

- Fleet health dashboard, incident timeline, SLO burn-down, alert correlation, runbook catalog

See: [526-sre-projection.md](526-sre-projection.md)

### 4.4 Integration primitives

- ArgoCD adapter, Prometheus/AlertManager adapter, ticketing (Jira), paging (PagerDuty), backlog adapter

See: [526-sre-integration.md](526-sre-integration.md)

### 4.5 Agent primitives

**SREAgent**: Implements 525 backlog-first triage workflow with human approval gates.

See: [526-sre-agent.md](526-sre-agent.md)

---

## 5) EDA contracts

### Topic families

- `sre.domain.intents.v1` → `SreDomainIntent`
- `sre.domain.facts.v1` → `SreDomainFact`
- `sre.process.facts.v1` → `SreProcessFact`

### Domain intents

- Incident: `CreateIncidentRequested`, `UpdateIncidentSeverityRequested`, `AssignIncidentRequested`, `ResolveIncidentRequested`, `CloseIncidentRequested`
- Alert: `IngestAlertRequested`, `AcknowledgeAlertRequested`, `CorrelateAlertsToIncidentRequested`
- Runbook: `CreateRunbookRequested`, `ExecuteRunbookRequested`
- SLO: `CreateSLORequested`, `RecordSLIMeasurementRequested`
- Health: `RecordHealthCheckRequested`, `RecordFleetStateRequested`

### Domain facts

- Incident: `IncidentCreated`, `IncidentResolved`, `IncidentClosed`
- Alert: `AlertIngested`, `AlertAcknowledged`, `AlertsCorrelatedToIncident`
- Runbook: `RunbookCreated`, `RunbookExecutionCompleted`
- SLO: `SLOCreated`, `SLIMeasurementRecorded`, `SLOBudgetExhausted`
- Health: `HealthCheckRecorded`, `FleetStateRecorded`

### Process facts

- `IncidentTriageStarted`, `BacklogLookupCompleted`, `TriagePhaseCompleted`
- `RemediationProposed`, `RemediationApplied`, `VerificationCompleted`
- `IncidentTriageCompleted`

See: [526-sre-proto.md](526-sre-proto.md)

---

## 6) Technology requirements

- Kubernetes + GitOps + operators
- Postgres/CNPG — Domain state + outbox
- Broker (NATS/Kafka) — facts topic families
- Temporal — Process primitive orchestration
- Prometheus + AlertManager — integration source
- ArgoCD — integration source

---

## 7) Implementation phases

1. **Contract foundation** — Define proto surfaces, scaffold primitives
2. **Domain implementation** — Implement sre-domain with aggregates
3. **Integration adapters** — ArgoCD, AlertManager, backlog adapters
4. **Process workflows** — IncidentTriageProcess (formalizes 525)
5. **Agent + UISurface** — SREAgent, operations console
6. **Hardening** — Multi-environment, SLO reporting

---

## 8) Cross-references

| Document | Relationship |
|----------|--------------|
| [525-backlog-first-triage-workflow.md](525-backlog-first-triage-workflow.md) | Source workflow formalized by this capability |
| [450-argocd-service-issues-inventory.md](450-argocd-service-issues-inventory.md) | Current incident tracking (to be migrated) |
| [519-gitops-fleet-policy-hardening.md](519-gitops-fleet-policy-hardening.md) | Fleet policy constraints |
| [475-ameide-domains.md](475-ameide-domains.md) | Domain portfolio (to add SRE cluster) |
| [528-capability-definition.md](528-capability-definition.md) | Capability framework |

---

## 9) Acceptance criteria

1. SRE is described as a **capability realized via primitives**
2. Explicit **SRE EDA contract** (topic families + envelopes) following 496 pattern
3. 525 triage workflow formalized as **Process primitive**
4. End-to-end slice: Alert → Incident → Backlog lookup → Triage → Remediation → Verification → Documentation
5. Domain portfolio (475) includes SRE as a first-class domain cluster
