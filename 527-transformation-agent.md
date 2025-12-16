# 527 Transformation — Agent Primitive Specification

**Status:** Draft  
**Parent:** [527-transformation-capability.md](527-transformation-capability.md)

This document specifies the **Transformation Agent primitives** — role-based assistants invoked by the portal and/or governance workflows.

---

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (Agent), Application Services (query + command/intents), Application Events (facts consumption).
- **Out-of-scope layers:** Canonical state writing (domain responsibility) and workflow execution durability (process responsibility).

## 1) Agent responsibilities

- Invoked interactively (chat in the UISurface) or non-interactively (by Process workflows).
- Read transformation context (projections, definitions) and propose/prepare changes.
- Emit commands/intents only through governed seams (no bypassing writer boundaries).
- Operate under definition-stored tool grants and risk tiers (AgentDefinitions), typically one per role (SA/TA/PM/etc.).

## 1.1) Tool access (MCP-compatible)

Transformation agents access the capability via the capability tool surface (queries → Projection; commands → Domain). MCP is an optional protocol binding for that surface, implemented as an Integration primitive per `backlog/534-mcp-protocol-adapter.md`.

Examples (illustrative; authoritative exposure comes from proto annotations):

- Queries (Projection): `transformation.listElements`, `transformation.getView`, `transformation.searchCapabilities`, `transformation.semanticSearch`
- Commands (Domain): `transformation.submitArchitectureIntent`, `transformation.submitScrumIntent`, `transformation.createBaseline`

Tool grants are expressed as tool identifiers (e.g., `<capability>.<operation>`) and enforced by AgentDefinitions/risk tiers; agents may invoke via SDK clients (preferred for in-platform agents) or via MCP adapters (external/devtool compatibility).

## 2) “Agent drives tenants through 524” (future behavior)

In the future state, a role-based agent guides a tenant through `backlog/524-transformation-capability-decomposition.md` inside a Transformation initiative:

- Creates/updates the required artifacts (glossary, value streams, EDA topology, proto proposal, decomposition, acceptance slice).
- Uses governance flows (Scrum/TOGAF/PMI) to enforce review/approval gates and promotions.
- Never bypasses writer boundaries: it proposes changes and emits intents; domains persist and emit facts; processes orchestrate.

## 3) Acceptance criteria

1. Agents are replay-safe and governed by stored AgentDefinitions (risk tiers + tool grants).
2. Agents read via projection query services; writes go through domain intents/commands only.
3. Agents can be invoked by Process workflows without introducing runtime RPC coupling.
