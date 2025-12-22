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
- Operate under definition-stored tool grants and risk tiers (AgentDefinitions), using a **generic role taxonomy** (Product Owner/Solution Architect/Executor) with profile-specific specializations.

### 1.0.1 Execution substrate (ephemeral agent work; WorkRequests)

Agent work that has meaningful side effects (repo changes, external writes, long-running tool use) MUST run on the same execution substrate as tool runners:

- Process (or an authorized actor) creates a Domain-owned `WorkRequest` for `work_kind = agent_work`.
- Domain emits `WorkRequested` facts onto the Kafka work-queue topic family (`transformation.work.domain.facts.v1`); KEDA ScaledJobs scale by consumer group lag and schedule devcontainer-derived Kubernetes Jobs that run the agent/runtime with role-scoped credentials.
- Outcomes are recorded back into Domain idempotently with linked evidence bundles; Process emits process facts for the run timeline.

This keeps “agent invoked by Process” event-driven and audit-grade without adding hidden RPC coupling.

### 1.0 Role taxonomy (generic, real-world aligned)

Transformation’s agent roles should not be Scrum-only. Scrum/TOGAF/PMI profiles map their accountabilities onto these generic delivery roles.

| Role | Primary responsibility | Default inputs | Default outputs | Repo execution | Notes |
|------|------------------------|----------------|-----------------|---------------|------|
| Product Owner | decide what/why/priority/scope | process facts + projection reads | intents, Dev Briefs, acceptance constraints | ❌ | maps to “product leadership” roles; Scrum PO is one mapping |
| Solution Architect | decide how/plan/risks | Dev Briefs + repo digest + projection reads | technical plans, proposals, gate requirements | ❌ | maps to solution/enterprise/technical architect roles |
| Executor | execute in constrained environment | repo coordinates + plan refs | PRs + evidence bundles | ✅ (constrained) | can be implemented as an executor runtime or an integration runner (tool-runner) |

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

### 1.2 Work handover (event-driven first)

Inter-role delegation (Product Owner→Solution Architect, Solution Architect→Executor) should be **event-driven first** so external tools can integrate without coupling to a specific in-cluster HTTP interface.

Canonical posture (cross-reference): `backlog/505-agent-developer-v2.md` defines the bus-native work handover semantics and how optional interactive transports (A2A) map onto them.

### 1.3 Role enforcement (defense in depth; tri-axis)

Role safety requires three independent enforcement axes:

1. **Semantic permissions**: AgentDefinitions (tool grants + risk tiers).
2. **Cluster permissions**: Kubernetes identity (ServiceAccount/RBAC/namespace isolation).
3. **External permissions**: provider identities (e.g., GitHub App/token) + server-side rulesets/branch protections.

No single layer is sufficient on its own; the system is designed so branch switching locally (or Job rescheduling) cannot increase authority.

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

- [ ] Role-based AgentDefinitions (tool grants + risk tiers) stored/promoted in Transformation, aligned to the generic role taxonomy above.
- [ ] Agent lifecycle (validate → propose → approve/apply) implemented end-to-end with audit evidence.

## 2.2) Clarification requests (next steps)

Confirm/decide:

- Which profile-specific specializations ship first (e.g., “Transformation Product Owner”, “Transformation Solution Architect”) and which tools each is allowed to invoke by default.
- What “small persisted agent state” means operationally (what is allowed to be stored on the Agent primitive vs attached as Transformation elements/evidence).
- Whether agents can initiate process workflows directly, or must do so only via domain commands that trigger processes.

## 3) Acceptance criteria

1. Agents are replay-safe and governed by stored AgentDefinitions (risk tiers + tool grants).
2. Agents read via projection query services; writes go through domain intents/commands only.
3. Agents can be invoked by Process workflows without introducing runtime RPC coupling.
