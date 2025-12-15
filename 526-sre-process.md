# 526 SRE — Process Primitive Specification

**Status:** Draft
**Parent:** [526-sre-capability.md](526-sre-capability.md)

This document specifies the **sre-process** primitive — Temporal-backed workflows for incident triage, change verification, and SLO monitoring.

---

## 1) Process responsibilities

The `sre-process` primitive implements **cross-domain workflows**:

- **IncidentTriageProcess** — the 525 backlog-first triage workflow
- **ChangeVerificationProcess** — post-deployment health verification
- **SLOBurnAlertProcess** — error budget monitoring and escalation
- **AlertCorrelationProcess** — grouping related alerts into incidents

Process primitives:

- Orchestrate activities across domain boundaries
- Emit **process facts** for workflow state changes
- Never become the system-of-record for domain state
- Are backed by Temporal for durability and visibility

---

## 2) IncidentTriageProcess (525 Workflow)

Formalizes [525-backlog-first-triage-workflow.md](525-backlog-first-triage-workflow.md).

### Workflow phases

```
START → BACKLOG_LOOKUP → TRIAGE → REMEDIATION → VERIFICATION → DOCUMENTATION → END
```

### Phase 1: PATTERN_LOOKUP

Search existing patterns BEFORE deep cluster triage (per 525 non-negotiable).

> **Critical:** Query projection-backed services only, never Transformation domain directly.

- Query `KnowledgeIndexQueryService` for matching patterns (backlog items, past incidents)
- Query `IncidentQueryService` for similar past incidents
- Query `RunbookQueryService` for applicable procedures

All queries go through **projection services**, not domain internals.

**Process fact:** `PatternLookupCompleted`

### Phase 2: TRIAGE

Targeted investigation based on backlog lookup results.

- If known pattern: follow documented steps
- If no match: gather targeted diagnostics

**Constraints:** No command timeout > 5 minutes

**Process fact:** `TriagePhaseCompleted`

### Phase 3: REMEDIATION

Propose and apply fix following GitOps discipline.

- Generate fix proposal
- Request human approval
- Commit to gitops repo
- Wait for ArgoCD sync

**Process facts:** `RemediationProposed`, `RemediationApplied`

### Phase 4: VERIFICATION

Verify health restored after remediation.

- Poll ArgoCD for sync status
- Check resource health conditions
- Verify alerts clear

**Process facts:** `VerificationStarted`, `VerificationCompleted`

### Phase 5: DOCUMENTATION

Update backlog/runbook with learnings.

- Generate incident summary
- Create/update backlog item
- Close incident

**Process fact:** `DocumentationRequested`

---

## 3) ChangeVerificationProcess

**Trigger:** GitOps commit detected

**Phases:**
1. SYNC_WAIT — Wait for ArgoCD sync
2. HEALTH_CHECK — Poll application health
3. SMOKE_TEST — Run smoke tests (optional)
4. CONFIRM — Mark verification complete

---

## 4) SLOBurnAlertProcess

**Trigger:** Scheduled or SLI threshold exceeded

**Phases:**
1. EVALUATE — Calculate burn rate and remaining budget
2. ALERT_IF_NEEDED — Create alert if burn rate high
3. ESCALATE_IF_NEEDED — Create incident if budget near exhaustion

---

## 5) Signals

- `ApproveRemediation` — human approves proposed fix
- `RejectRemediation` — human rejects proposed fix
- `EscalateIncident` — force escalation
- `CancelTriage` — abort workflow

---

## 6) Implementation notes

### Scaffold command

```bash
ameide primitive scaffold --kind process --name sre --include-gitops
```

### Activity timeouts

All activities respect 5-minute timeout constraint per 525.

---

## 7) Acceptance criteria

1. IncidentTriageProcess implements full 525 workflow
2. Process emits facts for all state transitions
3. Human approval gates work via Temporal signals
4. Timeouts respect 525 constraints
5. Process never writes domain state directly
6. **Pattern lookup queries projection services** (never Transformation domain)
