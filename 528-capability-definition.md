# 528 — Capability (Definition, boundaries, and how it maps to primitives)

**Status:** Draft (normative once accepted)  
**Audience:** Architecture, Transformation, platform engineering, operators/CLI, agents  
**Scope:** Define what a **Capability** is in Ameide, and how it is expressed as **EDA contracts** (application services/interfaces/events) realized by **primitives** (application components).

This backlog defines the **concept**. A concrete example is `527-transformation-capability.md` (Transformation capability) and `523-commerce*.md` (Commerce capability).

Language note: this doc follows the ArchiMate alignment anchor in `470-ameide-vision.md` §0.2 and the alignment rules in `529-archimate-alignment-470plus.md`.

Practical note: use `530-ameide-capability-design-worksheet.md` as the copy/paste worksheet when drafting a new capability backlog.
Implementation note: use `backlog/533-capability-implementation-playbook.md` as the end-to-end, parallelizable implementation DAG once the capability spec exists.

## Layer header (Strategy/Business)

- **Primary ArchiMate layer(s):** Strategy (Capability, Value Stream).
- **Secondary layers referenced:** Business (business processes, outcomes, ownership); Application (contracts + realization); Technology (topology constraints only); Implementation & Migration (work packages/deliverables/phases).
- **Primary element types used:** Capability, Value Stream, Business Process, plus Application Services/Interfaces/Events realized by Application Components; Work Package/Deliverable terms for Implementation & Migration.
- **Out-of-scope layers:** none (this doc is a template and touches multiple layers, but keeps them separated).
- **Allowed nouns:** capability, value stream, business process, outcome/value, policy, identity axis, contract surface.
- **Prohibited unless qualified:** process, service, domain, event (must be qualified per `470-ameide-vision.md` §0.2).

## 0) Definition: what a Capability is (in Ameide)

A **Capability** is a Strategy-layer “ability” the organization possesses. In Ameide terms, it is the *business-level unit of intent and ownership* that serves value delivery:

- a coherent set of **value streams** (and the business processes that realize them),
- a coherent set of **nouns** (bounded concepts/identity axes),
- a coherent set of **invariants** (what must always be true),
- a coherent set of **EDA contracts** (intents/facts/queries/integration seams) as application services/interfaces/events,
- realized by **Application Components** (Ameide primitives: Domain/Process/Projection/Integration/UISurface/Agent) that expose the contract surface and run on platform **Technology Services**.

Capabilities are what the platform is “selling” and what transformation work is “building”.

## 1) Why Capability exists (and why “primitives alone” isn’t enough)

Primitives are a technology taxonomy. Capabilities prevent architecture drift by anchoring:

- **business completeness** (end-to-end flows, not just infra slices),
- **ownership and boundaries** (who is authoritative writer; who is consumer),
- **contracts first** (proto and message topology before implementation),
- **topology modes** (cloud/edge/offline) and what “degraded” means,
- **acceptance slices** (testable outcomes per value stream).

## 2) Capability vs Domain (bounded context)

- A **Capability** is a *business-level umbrella* that can contain multiple bounded contexts.
- A **Domain primitive** is an implementation unit (a single-writer bounded context) that owns specific aggregates and emits facts.

Rule of thumb:

- Capability design should start with **value streams and nouns**, then derive bounded contexts (Domains), then define application services (EDA contracts) mapped to primitives (Processes/Projections/Integrations/Surfaces).

## 3) Capability vs “Service”

A capability is not “a service”:

- a capability can be implemented by many services/primitives,
- a capability can have multiple UISurfaces (admin/POS/storefront/etc),
- a capability can have multiple deployable topologies (cloud/edge).

Service naming is implementation detail; capability naming is the stable product + governance surface.

## 4) Capability deliverables (what must be defined)

For a capability to be considered “architected” (not just brainstormed), it must define:

1. **Strategy/Business**
   - capability brief (goal/non-goals),
   - outcomes/value propositions,
   - value streams and the business processes that realize them,
   - canonical nouns + policies/invariants.
2. **Application**
   - application services/interfaces/events (EDA contracts),
   - identity & scope model (axes used across intents/facts/queries),
   - data objects and read models (projections + query surfaces),
   - integration ports (external seams).
   - application interfaces for agents (MCP tools/resources/prompts; see §9).
3. **Technology**
   - topology modes (cloud/edge/offline) and platform constraints (brokers/DB/workflow engine/gateways),
   - placement assumptions and degraded-mode expectations.
4. **Implementation & Migration**
   - phases/plateaus, work packages/deliverables, and explicit gaps.
5. **Acceptance slices**
   - at least one end-to-end slice proving the seams between primitives and the contracts.

`524-transformation-capability-decomposition.md` is the repeatable method for producing these deliverables.

## 5) How a Capability maps to primitives (standard pattern)

### 5.1 Domain primitives (authoritative state)

Domains are the **single-writer** for their aggregates.

They must:

- accept change via **commands** (RPC) and/or **domain intents** (bus),
- emit **domain facts** via transactional outbox,
- avoid “CRUD as API”; prefer business verbs and invariant enforcement.

### 5.2 Process primitives (cross-domain invariants)

Processes implement:

- sagas and cross-domain workflows (Temporal),
- retries/compensation,
- timeboxes/governance and orchestration signals (process facts).

They must not become a system-of-record for domain state.

### 5.3 Projection primitives (read models)

Projections build:

- browse/report/search read models,
- lookup caches used by gateways/surfaces (e.g., hostname resolution),
- operational dashboards.

They are idempotent consumers (inbox or natural-key UPSERT).

### 5.4 Integration primitives (external seams)

Integrations implement:

- provider connectors (payments, tax, ERP, DNS, CI, etc),
- inbound webhook idempotency and dedupe,
- publication of integration outcomes as facts/signals.

### 5.5 UISurface primitives (experiences)

UISurfaces are thin:

- read via queries/projections,
- mutate via commands/intents,
- may consume best-effort UI realtime, but correctness stays with facts/read models.

### 5.6 Agent primitives (governed assistants)

Agents are:

- consumers of facts/read models,
- producers of intents/commands (often gated by human approval),
- governed by definition-stored tool grants and scopes.

## 6) Capability “contract shape” in proto (what we standardize)

A capability should standardize:

- **Topic families** per capability or bounded context, using aggregator envelopes:
  - `<cap>.domain.intents.v1` → `<Cap>DomainIntent`
  - `<cap>.domain.facts.v1` → `<Cap>DomainFact`
  - `<cap>.process.facts.v1` → `<Cap>ProcessFact`
- **Required envelope metadata** per `496-eda-principles.md` (tenant isolation + traceability).

Mapping note:

- facts are **Application Events**,
- intents/commands are requests to invoke **Application Services** (often asynchronously),
- queries are read-only **Application Services** (often realized by Projections).

A capability may also standardize:

- query services (read-only),
- integration service ports,
- shared scope messages (identity axes).

## 7) Where Capability “lives” in Ameide (Transformation + Graph)

Target positioning:

- The **Capability definition** (brief/process map/EDA contract/decomposition) is stored as **Transformation-domain artifacts** (versioned, promotable), expressed in official design-time languages (ArchiMate models/views, BPMN where relevant, plus Markdown narrative).
- The **Graph** can project capability elements (e.g., `archimate::capability`) for cross-domain impact analysis, but it is not the canonical writer for capability definitions.

This aligns with “Transformation is the change-the-business capability” described in `527-transformation-capability.md`.

## 8) Acceptance criteria for this backlog

1. “Capability” is a stable concept across backlogs: business intent → EDA contracts → primitives.
2. New capability backlogs follow the deliverables in §4 and the method in `524-transformation-capability-decomposition.md`.
3. We do not introduce a new primitive kind called “Capability”; capability is a business-level grouping whose implementation is primitives + contracts.

---

## 9) Application Interfaces for Agents (MCP)

This section is required in every capability backlog. It makes **agentic access** an explicit Application Interface decision (not an accidental side effect).

**Normative references:**
- MCP adapter pattern: `backlog/534-mcp-protocol-adapter.md`
- Read-side optimizations (semantic search): `backlog/535-mcp-read-optimizations.md`
- Write-side optimizations (agent-friendly commands): `backlog/536-mcp-write-optimizations.md`

### 9.1 What the capability publishes

Declare whether the capability publishes:

- **MCP tools:** yes/no (list)
- **MCP resources:** yes/no (list)
- **MCP prompts:** optional; only if you can keep them versioned/owned properly (list)

### 9.2 Tool catalog (required)

For each exposed MCP tool, record:

- **Tool name:** `<capability>.<operation>`
- **MCP kind:** `query` or `command`
- **Canonical Application Service:** the ArchiMate Application Service name (usually the gRPC service + method)
- **Owning primitive:** Domain (writes) or Projection (reads)
- **Auth/risk:** required scopes/roles + risk tier; whether human approval is required
- **Latency:** expected P50/P95 and whether the tool is safe for interactive use
- **Failure modes:** expected error classes (permission denied, invalid argument, unavailable, partial results)
- **Evidence/audit:** what facts/events/logs are emitted (domain facts, process facts, audit trail projection)

Recommended table (copy/paste):

| Tool | Kind | Application Service (canonical) | Primitive | Scopes / risk tier | Approval | Latency (P95) | Failure modes | Evidence / audit |
|------|------|----------------------------------|----------|--------------------|----------|---------------|---------------|------------------|
| `<cap>.<op>` | query/command | `<Service>.<Method>` | Domain/Projection | `<scopes> (tier N)` | yes/no | `<ms/s>` | `<codes>` | `<facts/logs>` |

### 9.3 Resource catalog (required if any resources exist)

Resources are read-only and resolve via Projection query services.

| Resource URI pattern | Backing query service | Primitive | Latency (P95) | Notes |
|----------------------|-----------------------|----------|---------------|------|
| `<cap>://<type>/{id}` | `<Service>.<Method>` | Projection | `<ms/s>` | `<size/caching>` |

### 9.4 Prompts (optional; discouraged unless well-governed)

If publishing prompts:

- Treat prompts as versioned, capability-owned artifacts with explicit ownership and promotion rules.
- Prefer “prompt IDs” that refer to versioned artifacts rather than embedding prompt text in multiple places.

### 9.5 Edge constraints (required)

Declare:

- **Write approvals:** which tools require human approval, and where approvals are recorded
- **Evidence bundles:** what gets produced as evidence (facts, baseline promotions, audit timeline links)
- **Safety posture:** default deny tool exposure; explicit allowlist; Origin validation for Streamable HTTP

### 9.6 Non-negotiables (proto-first)

- **Protocol adapters are Integration primitives.** MCP servers belong in Integration primitives, not Domains/Agents.
- **Adapters do not own semantics.** They translate to proto-first Application Services and do not become a parallel “business API”.
- **Tool schemas are derived from proto.** Never hand-author a parallel JSON schema contract; generate MCP schemas from proto annotations and keep exposure default-deny.
