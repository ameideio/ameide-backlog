# 527 Transformation — Process Primitive Specification

**Status:** Draft  
**Parent:** [527-transformation-capability.md](527-transformation-capability.md)

This document specifies the **Transformation Process primitives** — Temporal-backed governance workflows that execute methodology profiles (Scrum/TOGAF/PMI) and the "requirement → running" delivery loop.

**Extensions:**
- MCP write governance (agent-driven writes): `backlog/536-mcp-write-optimizations.md`

---

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (Process), Application Events (process facts), Application Services (workflow signals).
- **Out-of-scope layers:** Canonical state writing (domain responsibility) and portal read models (projection responsibility).

## 1) Process responsibilities

Process primitives implement **cross-domain orchestration**:

- Timeboxes, governance, approvals, and orchestration driven by **design-time ProcessDefinitions** stored in Transformation.
- Deterministic execution (Temporal) with explicit workflow state transitions emitted as **process facts**.
- No system-of-record behavior: processes never own canonical business state.

Core workflow families:

- **TOGAF/ADM governance** (phase progression, required deliverables, review/approval gates).
- **Scrum governance** (Sprint start/end, readiness cues) aligned with `backlog/506-scrum-vertical-v2.md`.
- **PMI governance** (stage gates, approvals, reporting cadence).
- **Delivery loop orchestration** (scaffold → verify → promote) for primitives and definitions.

## 2) Contract boundaries (non-negotiable)

- Processes **emit process facts only**; they do not emit domain facts.
- All cross-domain writes occur via **domain intents / commands**; domains persist and emit facts.
- Runtime workflows do not synchronously RPC into Transformation for reads/writes; workflows react to facts and use projection-backed reads when necessary.

## 3) Signals (generic)

- `ApproveGate` / `RejectGate` — human approvals for governance gates (baseline promotion, definition promotion).
- `CancelWorkflow` — abort with audit trail.
- `OverridePolicy` — explicit break-glass for authorized roles (recorded as a process fact).

## 4) Implementation notes

### Scaffold commands

```bash
ameide primitive scaffold --kind process --name togaf --include-gitops
ameide primitive scaffold --kind process --name pmi --include-gitops
```

### ProcessDefinitions

Store the canonical workflow specs in Transformation (BPMN-compliant ProcessDefinitions + versions) and bind Process CRs to promoted versions.

## 5) Acceptance criteria

1. Governance workflows are Temporal-backed and emit process facts for all transitions.
2. Processes never become canonical writers for Transformation artifacts.
3. Domain writes occur only via domain intents/commands (never direct DB writes; no “process-owned truth”).

