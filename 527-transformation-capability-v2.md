# 527 — Transformation Capability (v2): Zeebe-executed BPMN process primitives

This document replaces the “Temporal-backed Process primitive” execution posture described in `backlog/527-transformation-capability.md`.

## Summary

Transformation remains “design → realize → govern”, implemented via Ameide primitives over the Enterprise Knowledge substrate (Elements + Versions + Relationships).

The execution posture changes:

- **BPMN-authored processes execute on Camunda 8 / Zeebe.**
- The **Process primitive** remains a first-class concept, but for BPMN processes it is:
  - the BPMN definition (design-time truth),
  - deployed to Zeebe (runtime truth),
  - executed via Zeebe workers implemented by Ameide primitives (agents/integrations/domains).
- **Temporal is platform-only** (internal platform concerns) and is **not part of Ameide business capabilities**.

## Design-time vs runtime (unchanged principle; different runtime)

- **Design-time truth** lives in Transformation as versioned elements/definitions (BPMN, ArchiMate, docs).
- **Runtime orchestration** for Transformation governance BPMN executes on Zeebe.

Domains remain the system of record; process engines do not become canonical truth stores.

## BPMN as executable + scaffolding driver (still true, clarified)

BPMN remains the canonical authoring format for Transformation governance processes.

What changes is “how it becomes executable”:

- **Before:** BPMN → compiled IR/workflow code → Temporal execution.
- **Now:** BPMN → verified profile + worker coverage → **Zeebe deployment** → Zeebe execution.

## Side effects: implemented by primitives, not embedded in the process engine

Side effects are performed by Zeebe workers:

- Agent primitives implement “agentic work” as workers (coding, analysis, drafting).
- Integration primitives implement tool execution as workers (CI, GitHub, scanners, runners).
- Domain primitives are invoked by workers for canonical writes; domains emit facts.

The process engine coordinates; primitives do the work; domains persist truth.

## Verification and “diagram must not lie”

Transformation governance requires design-time verification gates that enforce:

- BPMN profile conformance (only what we support operationally).
- Zeebe deployability (engine/runtime constraints satisfied).
- Worker coverage (every executable side-effect step has an owning worker implementation).

If any required contract is missing, promotion/deployment must fail. No best-effort fallbacks.
