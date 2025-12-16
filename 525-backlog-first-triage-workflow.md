# Backlog 525 — Backlog-First Triage Workflow (Way of Working)

> **Capability formalization:** This workflow is formalized as a first-class capability in [526-sre-capability.md](526-sre-capability.md). The SRE capability provides:
> - Domain primitives for incidents, alerts, runbooks, SLOs
> - Process primitives that implement this workflow (IncidentTriageProcess)
> - Agent primitives (SREAgent) that automate backlog-first triage
> - Integration primitives for ArgoCD, AlertManager, and ticketing systems
>
> This document remains the authoritative "way of working" specification; 526 provides the system-of-record and automation layer.

## Goal

When an issue is spotted (ArgoCD unhealthy, sync failure, CrashLoop, policy drift), we should **avoid re-learning** problems we already solved. The backlog is the source of truth for incident patterns, runbooks, and accepted tradeoffs.

This document defines the **required order of operations** and the **tooling constraints** for investigation and remediation work.

## Non-Negotiables

1. **Backlog-first**: search and follow existing backlog guidance *before* deep cluster triage.
2. **No band-aids by default**: prefer root-cause fixes that improve reproducibility and idempotency.
3. **Backlog stays current**: update backlog entries for any new incident or meaningful variant of a known incident.
4. **Time-bounded commands**: do not set command timeouts higher than **5 minutes**.
5. **GitOps discipline**: changes land as commits (backlog + gitops repo), then Argo reconciles.
6. **Don’t delete what you don’t manage**: never delete repo files or cluster resources as “cleanup” unless they are explicitly owned by this repo/change and the deletion is part of the requested work.

## Workflow: Spot → Verify → Triage → Fix → Verify → Document

### 0) Capture the symptom (30–60 seconds)

Record, verbatim:
- Environment (`local`, `dev`, `staging`, `production`)
- Argo Application name(s)
- Unhealthy resource(s) (`kind/namespace/name`)
- The first error message(s) (Argo condition or resource health message)

This short symptom string becomes the search key for backlog lookup.

### 1) Backlog verification (required, before triage)

Search the backlog using the symptom string:
- `rg -n "<app>|<resource>|<error substring>" backlog`

Then check the canonical backlog anchors:
- `backlog/450-argocd-service-issues-inventory.md` (incident inventory + known mitigations)
- `backlog/519-gitops-fleet-policy-hardening.md` (policy decisions, accepted exceptions, follow-ups)
- Any relevant runbook documents (e.g., operator-specific recovery or known failure modes)

**Outcome A — Known issue found**
- Follow the documented remediation steps first.
- If the current symptoms differ materially (new failure mode, new dependency, new topology), add an addendum to the existing backlog entry *or* create a new backlog item and cross-link.

**Outcome B — No known issue found**
- Proceed to triage, but keep it minimal and structured (next section).

### 2) Triage (only after backlog verification)

Triage should:
- Identify the *single* resource blocking health (avoid multi-hour “read everything” work).
- Collect only the minimum supporting data required to propose a deterministic fix:
  - `kubectl describe` + last logs + relevant events
  - Argo `Application.status.resources` health messages

If the API server is slow/unreliable:
- Prefer shorter, scoped requests over long global queries.
- Use Kubernetes request timeouts (e.g. `kubectl ... --request-timeout=30s`).

### 3) Define remediation approach (no band-aids)

Propose remediation that improves at least one of:
- **Reproducibility** (same inputs → same outputs)
- **Idempotency** (safe to re-apply indefinitely)
- **Self-healing** (Argo can converge without manual steps)
- **Policy-shaped behavior** (fleet guardrails are explicit)

If a temporary mitigation is unavoidable:
- Mark it as **temporary** in the backlog, explain why, and create a follow-up backlog item to remove it.

### 4) Update backlog (required, before code changes)

Before implementing changes:
- Update the relevant existing backlog item(s), or create a new one.
- Add:
  - Symptom, scope, and severity
  - Root cause (or best current hypothesis)
  - Fix strategy and verification steps
  - Links to affected manifests/charts/values

### 5) Implement + verify

Implementation should be small and reversible:
- Prefer refactors that reduce special-casing and drift.
- Prefer vendor-supported knobs over hacks, unless explicitly documented as local-only capability-gated behavior.

Verification must include:
- Argo Applications `Synced/Healthy`
- The specific resource becomes Ready/Available and stays stable

### 6) Commit and push (backlog-first)

Backlog is a git submodule. Order matters:
1. Commit and push **backlog repo** changes in `backlog/`.
2. Commit and push **gitops repo** submodule pointer update + any manifest changes.

## Operational Constraint: Timeouts

Do not set any command timeout higher than **5 minutes**.

Preferred approach:
- Break investigations into smaller commands (narrowed selectors, single namespace, single app).
- Use `kubectl --request-timeout=...` to bound Kubernetes API calls.
- Avoid long-running “watch everything” loops; use targeted queries and iterate.

## Templates

### Incident note (to add to `backlog/450-argocd-service-issues-inventory.md` or a dedicated backlog item)

- **Date/Env**:
- **Argo app**:
- **Unhealthy resource**:
- **Symptom**:
- **Root cause**:
- **Fix**:
- **Verification**:
- **Follow-ups**:

### Policy note (to add to `backlog/519-gitops-fleet-policy-hardening.md`)

- **Policy**:
- **Rationale**:
- **Scope**:
- **Exception** (if any):
- **Enforcement/Guardrail**:
