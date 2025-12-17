# 527 Transformation — Agent Primitive Specification

**Status:** Draft (agent scaffold implemented; role definitions/tools pending)  
**Parent:** [527-transformation-capability.md](527-transformation-capability.md)

This document specifies the **Transformation Agent primitives** — role-based assistants invoked by the portal and/or governance workflows.

---

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (Agent), Application Services (query + command/intents), Application Events (facts consumption).
- **Out-of-scope layers:** Canonical state writing (domain responsibility) and workflow execution durability (process responsibility).

## 1) Agent responsibilities

Scope identity (required everywhere): `{tenant_id, organization_id, repository_id}` (no separate `graph_id`).

- Invoked interactively (chat in the UISurface) or non-interactively (by Process workflows).
- Read transformation context (projections, definitions) and propose/prepare changes.
- Emit commands/intents only through governed seams (no bypassing writer boundaries).
- Operate under definition-stored tool grants and risk tiers (AgentDefinitions), typically one per role (SA/TA/PM/etc.).

### 1.0 Write posture (validate → propose → approve/apply)

Agents MUST follow an explicit lifecycle so they cannot become shadow writers:

- **Validate**: read via projections; determine scope (`{tenant_id, organization_id, repository_id}`) and profile constraints.
- **Propose**: create draft changes (draft versions/baselines/proposals) that are reviewable and traceable.
- **Approve/Reject**: approval is a process gate; approvals emit process facts as evidence.
- **Apply**: canonical state changes happen via domain promotion/commands; domains persist and emit domain facts.

**Metamodel profile awareness:** when operating on repository content, agents must respect the selected notation (metamodel) profile:

- For **standards-compliant** profiles (e.g., ArchiMate 3.2, BPMN 2.0), agents must not propose writes that introduce non-standard `type_key` namespaces (e.g., `ameide:*`) or invalid relationships.
- For **extended** profiles, agents may use extension namespaces, but must surface the downstream compatibility implications (consumers must opt in to that profile).

**BPMN binding awareness:** when operating on deployable ProcessDefinitions, agents must treat BPMN bindings as first-class correctness constraints:

- Proposals that make a process “deployable/scaffoldable” must include complete bindings (or a linked binding element) that pass the promotion linter/compiler.
- Agents must not introduce non-portable BPMN extension namespaces in standards-compliant profiles; use linked bindings or documentation-carried bindings instead.

## 1.1) Tool access (MCP-compatible)

Transformation agents access the capability via the capability tool surface (queries → Projection; commands → Domain). MCP is an optional protocol binding for that surface, implemented as an Integration primitive per `backlog/534-mcp-protocol-adapter.md`.

Examples (illustrative; authoritative exposure comes from proto annotations):

- Queries (Projection): `transformation.listElements`, `transformation.getView`, `transformation.searchCapabilities`, `transformation.semanticSearch`
- Commands (Domain): `transformation.submitRepositoryIntent`, `transformation.submitScrumIntent`, `transformation.createBaseline`

Tool grants are expressed as tool identifiers (e.g., `<capability>.<operation>`) and enforced by AgentDefinitions/risk tiers; agents may invoke via SDK clients (preferred for in-platform agents) or via MCP adapters (external/devtool compatibility).

## 2) “Agent drives tenants through 524” (future behavior)

In the future state, a role-based agent guides a tenant through `backlog/524-transformation-capability-decomposition.md` inside a Transformation initiative:

- Creates/updates the required repository elements (glossary, value streams, EDA topology, proto proposal, decomposition, acceptance slice).
- Uses governance flows (Scrum/TOGAF/PMI) to enforce review/approval gates and promotions.
- Never bypasses writer boundaries: it proposes changes and emits intents; domains persist and emit facts; processes orchestrate.

## 2.1) Implementation progress (repo snapshot; checklists)

Delivered (scaffold + guardrails):

- [x] Agent scaffold exists at `primitives/agent/transformation` (server skeleton + tests).
- [x] Gate: `bin/ameide primitive verify --kind agent --name transformation --mode repo` passes (GitOps TODOs remain).

Not yet delivered (agent meaning):

- [ ] Role-based AgentDefinitions (tool grants + risk tiers) stored/promoted in Transformation.
- [ ] Agent lifecycle (validate → propose → approve/apply) implemented end-to-end with audit evidence.

## 2.2) Clarification requests (next steps)

Confirm/decide:

- The initial role taxonomy (SA/TA/PM/etc.) and which tools each role is allowed to invoke by default.
- What “small persisted agent state” means operationally (what is allowed to be stored on the Agent primitive vs attached as Transformation elements/evidence).
- Whether agents can initiate process workflows directly, or must do so only via domain commands that trigger processes.

## 3) Acceptance criteria

1. Agents are replay-safe and governed by stored AgentDefinitions (risk tiers + tool grants).
2. Agents read via projection query services; writes go through domain intents/commands only.
3. Agents can be invoked by Process workflows without introducing runtime RPC coupling.
