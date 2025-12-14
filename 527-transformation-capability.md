# 527 — Transformation Capability (Define the capability, then realize it via primitives)

**Status:** Draft (authoritative once accepted)  
**Audience:** Architecture, platform engineering, operators/CLI, agent/runtime teams  
**Scope:** Define **Transformation** as a *business capability* implemented as Ameide primitives (Domain/Process/Projection/Integration/UISurface/Agent), with 496-native EDA contracts.

This backlog is the *capability definition* counterpart to the *method* in `524-transformation-capability-decomposition.md`.

## 0) Problem

Today “Transformation” exists as:

- a **legacy service** (`services/transformation`) implementing a partial `TransformationService` CRUD surface,
- an emerging **Scrum contract** (`ameide_core_proto.transformation.scrum.v1` intents/facts/query + DB tables + outbox migration),
- a **target architecture intent** across backlogs (Transformation stores definitions; Process operator fetches definitions; runtime is bus-only),

…but we lack a single authoritative backlog that says:

> “Transformation is a capability. These are its domains/processes/surfaces/integrations, and these are its EDA contracts.”

This causes ambiguity about:

- system-of-record boundaries (who is allowed to write what),
- what is design-time vs runtime,
- how capability architecture (like `523-commerce*`) is stored and linked to primitives.

## 1) Definition: what the Transformation capability is

Transformation is the platform’s **change-the-business capability**: it captures *what should be built*, *how it should be built*, and *how it is governed*.

It is not “a modeling UI” and not “a Temporal service”; it is a capability whose primitives provide:

- the **system of record** for transformation initiatives and methodology artifacts,
- the **definition registry** for design-time artifacts (process/agent/extension definitions and architecture artifacts),
- the **EDA-native contracts** that Process primitives and Agents can consume to execute work deterministically.

## 2) Core invariants (non-negotiable)

1. **Capability is realized by primitives.** Transformation is not a monolith; it is realized via Domain/Process/Projection/Integration/UISurface/Agent application components.
2. **Design-time vs runtime separation.**
   - Design-time artifacts are stored in Transformation (as domain data).
   - Runtime workflows execute in Process primitives.
3. **496-native EDA.**
   - commands/intents vs facts, outbox for writers, idempotent consumers, sagas in Process primitives.
4. **No runtime RPC coupling.**
   - Runtime Process workflows MUST NOT call Transformation synchronously to mutate/read domain state; they must use bus intents/facts.
   - Synchronous reads of definitions by operators are *control-plane only*.
5. **Tenant isolation and traceability metadata** on all messages.

## 3) Capability boundaries and nouns

Transformation capability owns (at minimum):

- **Transformation initiative** (portfolio/program object; “what we are doing and why”)
- **Workspace tree** (nodes/folders/scopes for artifacts)
- **Artifact registry** (elements, attachments, revisions/promotions)
- **Methodology profiles** (e.g., Scrum), implemented as bounded contexts
- **Definitions** used by the platform (ProcessDefinition, AgentDefinition, ExtensionDefinition, etc.)

Rules:

- Graph is a read-optimized projection/index; Transformation is canonical for transformation artifacts and definitions.
- Other capabilities may reference Transformation artifacts by stable IDs; they do not become writers of those artifacts.

## 4) Primitive decomposition (technology layer)

### 4.1 Domain primitives (system-of-record)

**A) `transformation-domain` (core)**

Responsibilities:

- Own `TransformationService` domain state (`ameide_core_proto.transformation.v1`) including:
  - transformations (initiative records),
  - workspace nodes,
  - milestones/metrics/alerts,
  - design-time definition storage (see §6).
- Emit domain facts for significant state changes (see §5).

**B) `transformation-scrum-domain` (methodology bounded context)**

Responsibilities:

- Own Scrum artifacts as a true domain (Product Backlog, PBI, Sprint, Increment, DoD, Goals).
- Accept change via `ScrumDomainIntent` (bus).
- Emit `ScrumDomainFact` via transactional outbox (bus).
- Provide `ScrumQueryService` as read-only RPC.

### 4.2 Process primitives (Temporal governance)

Responsibilities:

- Timeboxes, governance, approvals, and orchestration:
  - Scrum timebox governance (Sprint start/end windows, reminders, readiness cues) per `506-scrum-vertical-v2.md`
  - “requirement → running” orchestration (scaffold, verify, promote) for primitives and definitions
- Emit process facts (governance cues) distinct from domain facts.
- Never become a system-of-record for methodology artifacts.

### 4.3 Projection primitives (read models)

Responsibilities:

- Build read models for portals and agents:
  - portfolio dashboards,
  - “capability architecture” browsing,
  - methodology analytics (throughput, cycle time),
  - definition indexing and dependency views.
- Idempotent consumption (inbox or UPSERT) per `496-eda-principles.md`.

### 4.4 UISurface primitives (experiences)

Responsibilities:

- Transformation portal / design tooling UIs:
  - initiative views,
  - workspace browser,
  - modeling editors (BPMN, ArchiMate diagrams, Markdown),
  - definition management and promotion flows.
- UISurfaces are thin: they query projections and emit commands/intents.

### 4.5 Agent primitives (governed assistants)

Responsibilities:

- Read transformation context and propose changes.
- Emit commands/intents only through governed seams.
- Tooling grants and risk-tier enforcement are stored as definitions (AgentDefinitions).

### 4.6 Integration primitives (external boundaries)

Responsibilities:

- Backstage scaffolder, GitHub, CI, container registry, ticketing, etc.
- Strict idempotency on inbound webhooks and retries.

## 5) EDA contracts (message topology)

Transformation capability uses three message classes (496):

- **Domain intents** (imperative): request a state change.
- **Domain facts** (immutable): emitted after persistence.
- **Process facts** (governance): emitted by Process primitives for orchestration cues.

Concrete topic families already exist for the Scrum bounded context:

- `scrum.domain.intents.v1` → `ameide_core_proto.transformation.scrum.v1.ScrumDomainIntent`
- `scrum.domain.facts.v1` → `ameide_core_proto.transformation.scrum.v1.ScrumDomainFact`
- `scrum.process.facts.v1` → `ameide_core_proto.process.scrum.v1.ScrumProcessFact`

Action for this backlog: define the equivalent topic families for core Transformation domain changes (initiative/workspace/definition changes), using the same “aggregator envelope” pattern.

## 6) Definition Registry (design-time artifacts) inside Transformation

Target: Transformation becomes the canonical store for design-time artifacts:

- `ProcessDefinition` (design-time; referenced by Process CRs; fetched by Process operator as control-plane config)
- `AgentDefinition` (design-time; fetched by Agent operator; governs agent runtime)
- `ExtensionDefinition` / `PrimitiveImplementationDraft` (design-time; drives scaffolding/promotion processes)
- Capability architecture artifacts (ArchiMate models/views, Markdown, proto proposals), as transformation workspace elements

This implies a schema-backed model, not just markdown docs:

- `CapabilityDefinition` (capability identity, scope, nouns/topology)
- `CapabilityEDAContract` (topic families, envelopes, intent/fact catalogs)
- `CapabilityPrimitiveDecomposition` (primitives + responsibilities)

This backlog does not force the exact proto/package yet, but it makes the requirement explicit: the methodology in `524` must become domain data in Transformation.

## 7) Legacy mapping (current state)

### 7.1 `services/transformation` (legacy)

Current behavior:

- Implements partial `TransformationService` CRUD and agent definition upsert/get.
- Does not implement workspace/milestones/metrics/alerts.
- Does not implement EDA contracts for Scrum intents/facts or outbox publishing.

Role in target:

- Either becomes a façade while Domain primitives take over, or is evolved into a Domain-primitive-shaped implementation (outbox + bus + strict boundaries).

### 7.2 `primitives/domain/transformation` (scaffold)

Current behavior:

- Scaffold only, unimplemented handlers for `ScrumQueryService`.
- Has a generic outbox migration but is not wired to Transformation schema or Scrum tables.

Role in target:

- Becomes the canonical implementation of the Transformation bounded contexts (core + scrum), consistent with the primitives stack and 496.

## 8) Fit/Gap summary (what this backlog drives)

### Fits

- Protos exist for Transformation core entities and Scrum contracts.
- DB migrations exist for Scrum tables and a Scrum domain outbox.
- Operator backlogs already define the runtime seam: Process ↔ Transformation via bus.

### Gaps

- No canonical “Transformation capability” decomposition doc (this backlog fixes that).
- No core Transformation intent/fact topic families defined for initiative/workspace/definition lifecycle.
- No implemented Scrum domain runtime (intent ingestion + outbox publication + query RPC).
- Ambiguous writer boundaries between services (repository vs transformation vs future domain primitive).

## 9) Acceptance criteria

1. Transformation is described and operated as a **capability realized via primitives**, not as a single service.
2. There is an explicit **core Transformation EDA contract** (topic families + envelopes) mirroring the Scrum pattern.
3. There is a clear **migration stance**: what becomes canonical writer and what becomes projection/facade.
4. One end-to-end slice exists that proves the seam (example):
   - `ScrumDomainIntent` → domain persistence → outbox → `ScrumDomainFact` → Process reacts → emits `ScrumProcessFact` → UISurface reads via `ScrumQueryService` / projections.
