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

- Scope identity (required everywhere): `{tenant_id, organization_id, repository_id}` (no separate `graph_id`).

- Provide the Transformation portal / design tooling UI:
  - initiative views and dashboards,
  - workspace browser,
  - modeling editors (generic canvas + metamodel profiles for ArchiMate/BPMN/Docs),
  - definition registry manager and promotion flows,
  - governance consoles (Scrum/TOGAF/PMI gates, approvals, evidence bundles),
  - activity feed and audit timeline views.
- Remain thin:
  - read via projection query services,
  - mutate via commands/intents only.

### 1.1 Metamodel profiles (standards-compliant vs extended)

The UI treats modeling as a **generic editor** over Elements, configured by a **notation (metamodel) profile** selected in diagram/model properties:

- **Standards-compliant profile** (e.g., ArchiMate 3.2, BPMN 2.0):
  - constrain creation to the standard `type_key` namespace (`archimate:*`, `bpmn:*`)
  - enforce a standard relationship validity matrix (warn/block)
  - provide standard export targets (e.g., ArchiMate Exchange, BPMN XML) from the **published/baselined** version
  - when the notation is BPMN and a ProcessDefinition is marked deployable/scaffoldable, provide a binding editor that stores portable binding metadata (documentation-carried or linked binding element) and surfaces binding completeness as a promotion gate requirement
- **Extended profile** (Ameide/vendor extensions):
  - allow additional namespaces (e.g., `ameide:*`)
  - define custom constraints and export targets (downstream must explicitly declare support)
  - allow richer BPMN binding metadata via extensions, but surface portability implications in conformance results

Profile definitions and conformance/export status are data-driven:

- Profile definitions are versioned/promotable and referenced from model/view metadata (see `backlog/527-transformation-domain.md` §4.4).
- Conformance results, exports, and gate evidence are projection-backed views with `read_context` + citations (see `backlog/527-transformation-projection.md` §4.2.1).

## 2) Full UX scope (future state inventory)

- Initiative dashboard (status, baselines/plateaus, governance state).
- Repository browser (workspace tree + elements + versions + promotions).
- ArchiMate model browser (elements/relationships lists, filters, search).
- ArchiMate view editor (canvas layout, relationship routing, style, viewpoints).
- Relationship matrix / impact analysis (projection-driven).
- BPMN process browser/editor (ProcessDefinition + versions + links to elements).
- BPMN binding editor (portable bindings for standards-compliant BPMN; richer bindings for extended profiles).
- Definition registry manager (ProcessDefinitions/AgentDefinitions/Extensions; versioning; promotion).
- Governance console (TOGAF phase gates, PMI stage gates, approvals, audit trail).
- Chat + activity feed (agent conversation, generated proposals, applied changes).

## 2.1) Implementation progress (repo snapshot)

Delivered (MVP slice + guardrails):

- [x] MVP portal is implemented in `services/www_ameide_platform` (repository browsing + ArchiMate view editing) and wired to primitives.
- [x] UISurface placeholder exists at `primitives/uisurface/transformation` (static page server, tests).
- [x] Gate: `bin/ameide primitive verify --kind uisurface --name transformation --mode repo` passes (GitOps TODOs remain).

Not yet delivered (UISurface meaning):

- [ ] Workspace tree browser + assignments UI (beyond “view list”).
- [ ] Governance UX (initiatives/baselines/promotions/approvals/evidence).
- [ ] Audit timeline views with `read_context` + citations.

## 2.2) Clarification requests (next steps)

Confirm/decide:

- The v1 minimum UX slice (which screens/flows are required to prove the acceptance seam).
- Auth/session posture for the portal (tenant selection, roles, and how approvals are authorized).
- How the UI renders reproducible reads: which views must show `read_context` + citations by default.

Cross-reference:

- Approval policy is stored as a schema-backed definition and enforced by Domain/Process gates; UISurface must not hard-code who can approve what (see `backlog/527-transformation-domain.md` §3.5).

## 3) Acceptance criteria

1. All reads go through `backlog/527-transformation-projection.md` query services.
2. All mutations go through domain write surfaces (RPC and/or domain intents).
3. UISurface never becomes a competing writer of canonical truth.
