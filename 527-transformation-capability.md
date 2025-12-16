# 527 — Transformation Capability (Define the capability, then realize it via primitives)

**Status:** Draft (authoritative once accepted)  
**Audience:** Architecture, platform engineering, operators/CLI, agent/runtime teams  
**Scope:** Define **Transformation** as a *business capability* implemented as Ameide primitives (Domain/Process/Projection/Integration/UISurface/Agent), with 496-native EDA contracts.

This backlog is the capability definition counterpart to the method in `backlog/524-transformation-capability-decomposition.md`.

**Use with:**
- Capability definition template: `backlog/528-capability-definition.md`
- Capability worksheet: `backlog/530-ameide-capability-design-worksheet.md`
- Capability implementation DAG: `backlog/533-capability-implementation-playbook.md`
- ArchiMate vocabulary/verbs: `backlog/529-archimate-alignment-470plus.md`
- EDA contract rules: `backlog/496-eda-principles.md`, `backlog/496-eda-protobuf-ameide.md`
- Proto conventions: `backlog/509-proto-naming-conventions.md`
- Primitives stack guardrails: `backlog/520-primitives-stack-v2.md`

## Layer header (Strategy + Business, with Application realization)

- **Primary ArchiMate layer(s):** Strategy + Business (Capability, Value Streams, Business Processes) with Application realization via primitives.
- **Secondary layers referenced:** Technology (constraints only), Implementation & Migration (work packages/phases).
- **Primary element types used:** Capability, Value Stream, Business Process, Application Component, Application Service/Interface/Event, Work Package/Gap.
- **Prohibited unless qualified:** process, service, domain, event (qualify per `backlog/529-archimate-alignment-470plus.md`).

## Future state (executive summary)

Transformation is Ameide’s change-the-business capability. In the future state it provides:

- A typed **Enterprise Repository** as the system of record for transformation initiatives and design-time artifacts.
- Multi-methodology **profiles** (Scrum, TOGAF/ADM, PMI) as bounded contexts that govern how the repository is used.
- A **Definition Registry** for platform definitions (ProcessDefinition, AgentDefinition, ExtensionDefinition, etc.) stored as domain data and consumed by operators and runtimes.
- 496-native EDA contracts so Process primitives and Agents can execute work deterministically without runtime RPC coupling.

## 0) Problem

Today “Transformation” exists as:

- a legacy service (`services/transformation`) implementing a partial `TransformationService` CRUD surface,
- an emerging Scrum contract (`ameide_core_proto.transformation.scrum.v1` intents/facts/query + DB tables + outbox migration),
- a target architecture intent across backlogs (Transformation stores definitions; Process operator fetches definitions; runtime is bus-only),

…but we lack a single authoritative backlog that says:

> “Transformation is a capability. These are its domains/processes/surfaces/integrations, and these are its EDA contracts.”

## 1) Definition

Transformation is the platform’s **change-the-business capability**: it captures what should be built, how it should be built, and how it is governed.

It is not “a modeling UI” and not “a Temporal service”; it is a capability realized by primitives that provide:

- the Enterprise Repository (typed design-time system of record),
- the Definition Registry (design-time definitions as domain data),
- EDA-native contracts that processes and agents consume to execute work deterministically.

## 1.1) Strategy: outcomes and value streams (starter set)

Primary outcomes (future state):

- Tenants can run transformation initiatives using a repeatable methodology.
- Design-time truth (ArchiMate/BPMN/definitions) is governable, versioned, promotable, and queryable.
- Runtime execution is deterministic: Process primitives and Agents drive work from contracts and facts, not ad-hoc RPC calls.

Value streams (refine via `backlog/524-transformation-capability-decomposition.md` Step 1):

- **Initiate**: create initiative → lock glossary/identity axes → establish repository structure.
- **Design**: model capability/value streams → derive contracts/topology → propose proto packages/topics → promote baselines.
- **Realize**: decompose to primitives → scaffold → verify → promote definitions.
- **Govern**: run methodology gates (ADM phases, Scrum timeboxes, PMI stage gates) → manage drift and compliance.

## 2) Core invariants (non-negotiable)

1. Capability is realized by primitives (no monolith).
2. Design-time vs runtime separation (design-time artifacts stored in Transformation; runtime workflows execute in Process primitives).
3. 496-native EDA (intents vs facts, outbox for writers, idempotent consumers, sagas in Process primitives).
4. No runtime RPC coupling (runtime workflows use bus intents/facts; operator reads are control-plane only).
5. Tenant isolation and traceability metadata on all messages.
6. Typed design-time store (ArchiMate/BPMN/definitions as domain data; file formats are attachments only).
7. Multi-methodology by profile (Scrum/TOGAF/PMI as profiles).
8. Profiles govern; they don’t redefine the repository (no forked canonical models).

## 3) Capability boundaries and nouns

Transformation capability owns (at minimum):

- Transformation initiative
- Enterprise Repository workspace tree
- Artifact registry (typed objects, attachments, revisions/promotions)
- Typed ArchiMate models and views (layout is first-class)
- Typed BPMN process models (ProcessDefinitions as first-class design-time artifacts)
- Methodology profiles (Scrum, TOGAF/ADM, PMI) as bounded contexts
- Platform definitions (ProcessDefinition, AgentDefinition, ExtensionDefinition, etc.)

Rule of thumb:

- Transformation is canonical; Graph is a read-optimized projection/index.
- Other capabilities may reference Transformation artifacts by stable IDs; they do not become writers of those artifacts.

## 4) Application realization (primitives)

- Domain primitives: `backlog/527-transformation-domain.md`
- Process primitives: `backlog/527-transformation-process.md`
- Projection primitives: `backlog/527-transformation-projection.md`
- UISurface primitives: `backlog/527-transformation-uisurface.md`
- Agent primitives: `backlog/527-transformation-agent.md`
- Integration primitives: `backlog/527-transformation-integration.md`
- Proto contracts: `backlog/527-transformation-proto.md`
- Implementation & migration: `backlog/527-transformation-implementation-migration.md`

## 4.1) Application Interfaces for Agents (MCP)

Transformation is the reference implementation for MCP-based agentic access.

References:
- MCP adapter pattern: `backlog/534-mcp-protocol-adapter.md`
- Read optimizations (semantic search): `backlog/535-mcp-read-optimizations.md`
- Write optimizations (agent-friendly commands): `backlog/536-mcp-write-optimizations.md`

**Published interface**
- MCP tools: yes
- MCP resources: yes
- MCP prompts: optional (defer until prompts are versioned and promotable)

**Integration primitive**
- `transformation-mcp-adapter` (Integration primitive) translates MCP (stdio + Streamable HTTP) into proto-first Application Services.
- Tool schemas are generated from proto annotations (default deny; explicit allowlist).

Starter tool map (illustrative; authoritative source is generated from proto):

| Tool | Kind | Application Service (canonical) | Primitive | Approval | Evidence / audit |
|------|------|----------------------------------|----------|----------|------------------|
| `transformation.listElements` | query | `ArchiMateQueryService.ListElements` | Projection | no | audit log + query tracing |
| `transformation.getView` | query | `ArchiMateViewQueryService.GetView` | Projection | no | audit log + query tracing |
| `transformation.semanticSearch` | query | `SemanticQueryService.Search` | Projection | no | audit log + query tracing |
| `transformation.submitArchitectureIntent` | command | `TransformationArchitectureWriteService.SubmitIntent` | Domain | yes | `transformation.domain.facts.v1` + audit trail projection |
| `transformation.createBaseline` | command | `BaselineWriteService.CreateBaselineDraft` | Domain | yes | baseline facts + audit trail projection |

Starter resources (illustrative):

| Resource URI pattern | Backing query service | Primitive |
|----------------------|-----------------------|----------|
| `transformation://model/{id}` | `ArchiMateQueryService.GetModel` | Projection |
| `transformation://element/{id}` | `ArchiMateQueryService.GetElement` | Projection |
| `transformation://view/{id}` | `ArchiMateViewQueryService.GetView` | Projection |

Edge constraints (Transformation):

- Writes require human approval by default; approvals are recorded as domain/process facts and visible in the audit timeline.
- Evidence is emitted as domain facts (after persistence) and surfaced via projection-backed audit trail.

**Proto contract references:**
- Topic families: `transformation.domain.intents.v1`, `transformation.domain.facts.v1`
- Proto package: `ameide_core_proto.transformation.core.v1`
- Full contract catalog: [527-transformation-proto.md](527-transformation-proto.md)

**MCP Security Spine (invariants):**

| Invariant | Requirement |
|-----------|-------------|
| **OAuth 2.1 RS** | Streamable HTTP MCP adapter MUST be an OAuth 2.1 Resource Server. Keycloak is the Authorization Server. |
| **Origin validation** | Adapter MUST validate `Origin` header for browser-based MCP clients (CORS policy). |
| **Protocol ≠ privilege** | MCP is a protocol binding, not a privilege boundary. Authorization decisions are enforced by Domain/Projection services, not the adapter. |
| **Tenant isolation** | All tool calls MUST pass through tenant context from access token. Adapter never allows cross-tenant access. |
| **Scopes** | Required scopes: `mcp:tools`, `mcp:resources`, `openid`, `profile`. Additional capability-specific scopes as needed. |

See [534-mcp-protocol-adapter.md](534-mcp-protocol-adapter.md) §5 for full OAuth 2.1 specification.

**Internal vs external agent transport:**

| Agent Type | Transport | Rationale |
|------------|-----------|-----------|
| Platform agents (AmeidePO, AmeideSA, AmeideCoder) | SDK/gRPC | Typed, efficient, no JSON-RPC overhead |
| External AI tools (Claude Code, Cursor, Copilot) | MCP (Streamable HTTP) | Compatibility binding for devtools |

MCP is a **compatibility interface**, not the canonical tool runtime. Tool definitions live in proto with `(ameide.mcp.expose)` annotations; MCP schemas are generated, never hand-authored.

## 5) Acceptance criteria

1. Transformation is described and operated as a capability realized via primitives, not as a single service.
2. Core Transformation EDA contracts exist (topic families + envelopes) aligned with the Scrum pattern.
3. Migration stance is explicit: what becomes canonical writer and what becomes projection/facade.
4. At least one end-to-end slice proves the seam (example: `ScrumDomainIntent` → outbox → `ScrumDomainFact` → Process reacts → `ScrumProcessFact` → UISurface reads via projection query services).
