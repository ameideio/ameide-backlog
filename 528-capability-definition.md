# 528 — Capability (Definition, boundaries, and how it maps to primitives)

**Status:** Draft (normative once accepted)  
**Audience:** Architecture, Transformation, platform engineering, operators/CLI, agents  
**Scope:** Define what a **Capability** is in Ameide, and how it is expressed as **EDA contracts** and decomposed into **primitives**.

This backlog defines the **concept**. A concrete example is `527-transformation-capability.md` (Transformation capability) and `523-commerce*.md` (Commerce capability).

## 0) Definition: what a Capability is (in Ameide)

A **Capability** is a *business-level unit of intent and ownership*:

- a coherent set of **value streams** (business processes),
- a coherent set of **nouns** (bounded concepts/identity axes),
- a coherent set of **invariants** (what must always be true),
- a coherent set of **EDA contracts** (intents/facts/queries/integration seams),
- implemented as a set of **technology primitives** (Domain/Process/Projection/Integration/UISurface/Agent).

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

- Capability decomposition should start with **value streams and nouns**, then derive bounded contexts (Domains), then Processes/Projections/Integrations/Surfaces.

## 3) Capability vs “Service”

A capability is not “a service”:

- a capability can be implemented by many services/primitives,
- a capability can have multiple UISurfaces (admin/POS/storefront/etc),
- a capability can have multiple deployable topologies (cloud/edge).

Service naming is implementation detail; capability naming is the stable product + governance surface.

## 4) Capability deliverables (what must be defined)

For a capability to be considered “architected” (not just brainstormed), it must define:

1. **Capability brief**
   - goal and non-goals,
   - glossary / canonical nouns,
   - topology modes (cloud/edge/offline).
2. **Process/value-stream map**
   - Level 0/1 value streams and 5–7 “first-class processes”,
   - at least one “golden path” scenario per process.
3. **Identity & scope model**
   - required scope axes used across intents/facts/queries (e.g., tenant/site/channel/location).
4. **EDA contract (496-native)**
   - topic families (domain intents, domain facts, process facts),
   - required envelopes/metadata (tenant isolation, traceability),
   - idempotency strategy and delivery expectations,
   - query surfaces and read models.
5. **Primitive decomposition**
   - inventory of primitives and responsibilities:
     - owned commands/intents,
     - emitted facts,
     - consumed facts,
     - storage and outbox/inbox,
     - topology placement (cloud/edge).
6. **Acceptance slices**
   - one end-to-end slice proving the seam between primitives and the contracts.

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

A capability may also standardize:

- query services (read-only),
- integration service ports,
- shared scope messages (identity axes).

## 7) Where Capability “lives” in Ameide (Transformation + Graph)

Target positioning:

- The **Capability definition** (brief/process map/EDA contract/decomposition) is stored as **Transformation-domain artifacts** (versioned, promotable).
- The **Graph** can project capability elements (e.g., `archimate::capability`) for cross-domain impact analysis, but it is not the canonical writer for capability definitions.

This aligns with “Transformation is the change-the-business capability” described in `527-transformation-capability.md`.

## 8) Acceptance criteria for this backlog

1. “Capability” is a stable concept across backlogs: business intent → EDA contracts → primitives.
2. New capability backlogs follow the deliverables in §4 and the method in `524-transformation-capability-decomposition.md`.
3. We do not introduce a new primitive kind called “Capability”; capability is a business-level grouping whose implementation is primitives + contracts.

