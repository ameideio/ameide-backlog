# 527 Transformation — UISurface Primitive Specification

**Status:** Draft (placeholder UISurface implemented; portal UX pending)  
**Parent:** [527-transformation-capability.md](527-transformation-capability.md)

This document specifies the **Transformation UISurface primitive** — the portal and design tooling experiences for initiatives, repository browsing, modeling, governance, and definition management.

---

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (UISurface), Application Interface (UI/API access), Application Services (read/query + write via commands/intents).
- **Out-of-scope layers:** Canonical state (domain) and orchestration (process).

## 1) UISurface responsibilities

- Provide the Transformation portal / design tooling UI:
  - initiative views and dashboards,
  - workspace browser,
  - modeling editors (typed ArchiMate views, BPMN, Markdown),
  - definition registry manager and promotion flows,
  - governance consoles (Scrum/TOGAF/PMI gates, approvals, evidence bundles),
  - activity feed and audit timeline views.
- Remain thin:
  - read via projection query services,
  - mutate via commands/intents only.

## 2) Full UX scope (future state inventory)

- Initiative dashboard (status, baselines/plateaus, governance state).
- Repository browser (workspace tree + artifacts + revisions + promotions).
- ArchiMate model browser (elements/relationships lists, filters, search).
- ArchiMate view editor (canvas layout, relationship routing, style, viewpoints).
- Relationship matrix / impact analysis (projection-driven).
- BPMN process browser/editor (ProcessDefinition + versions + links to artifacts).
- Definition registry manager (ProcessDefinitions/AgentDefinitions/Extensions; versioning; promotion).
- Governance console (TOGAF phase gates, PMI stage gates, approvals, audit trail).
- Chat + activity feed (agent conversation, generated proposals, applied changes).

## 2.1) Implementation progress (repo snapshot)

Delivered (placeholder + guardrails):

- UISurface placeholder exists at `primitives/uisurface/transformation` (static page server, tests, non-root container).
- Repo-mode verification is green: `bin/ameide primitive verify --kind uisurface --name transformation --mode repo`.

Not yet delivered (UISurface meaning):

- Portal UX (initiative dashboard, workspace browser, editors, governance consoles) is not implemented.
- Human-in-loop approval UX (review gates, evidence display, audit timeline with citations) is not implemented.

## 2.2) Clarification requests (next steps)

Confirm/decide:

- The v1 minimum UX slice (which screens/flows are required to prove the acceptance seam).
- Auth/session posture for the portal (tenant selection, roles, and how approvals are authorized).
- How the UI renders reproducible reads: which views must show `read_context` + citations by default.

## 3) Acceptance criteria

1. All reads go through `backlog/527-transformation-projection.md` query services.
2. All mutations go through domain write surfaces (RPC and/or domain intents).
3. UISurface never becomes a competing writer of canonical truth.
