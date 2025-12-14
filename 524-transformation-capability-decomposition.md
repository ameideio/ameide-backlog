# 524 — Transformation Method: Capability → EDA Contracts → Primitives

This document generalizes the methodology used for `523-commerce*` into a repeatable **Transformation Domain** workflow:

1. Define a **business capability** as value streams + nouns + invariants.
2. Define **EDA contracts** (commands/intents, domain facts, process facts, queries, integration ports) per `496-eda-principles.md`.
3. Decompose into **Ameide primitives (Application Components)** (Domain/Process/Projection/Integration/UISurface/Agent) per `520-primitives-stack-v2.md`.

The output is a set of artifacts that can be stored as Transformation workspace nodes (ArchiMate models/views, Markdown, BPMN) and then used by agents/CLI/operators to scaffold and deploy the runtime.

For an explicit IT4IT value-stream lens (R2D/R2F/D2C) over this same pipeline, see `525-it4it-value-stream-mapping.md`.

## Why this exists

Without a capability→process→contract→primitives pipeline, decompositions tend to drift toward “infra/onboarding” and miss day‑2 retail/ERP operations. Vendor suites avoid this by starting from end-to-end processes (value streams) and then deriving APIs, events, and deployment units.

## Inputs (Transformation intake)

Minimum inputs:

- Capability name + goal (e.g., “Commerce”).
- Tenancy model + identity boundaries (tenant/org).
- Primary value streams (Level 0/1).
- Topology modes (cloud / edge / offline) and what “degraded” means.
- Regulatory constraints (PCI/PII/retention).

Optional inputs:

- Existing systems to integrate with (ERP, WMS, PSP, DNS/CDN, etc.).
- Data migration constraints.
- SLAs/SLOs (latency, RPO/RTO, propagation windows).

## Outputs (Transformation artifacts)

Store these as versioned artifacts inside the Transformation Domain workspace:

1. **Capability brief** (scope, non-goals, glossary).
2. **Process catalog** (Level 0/1 value streams + “golden paths”).
3. **Identity & scope model** (canonical nouns: tenant/site/channel/location/etc.).
4. **EDA topology**:
   - topic families,
   - command/intent catalog,
   - domain fact catalog,
   - process fact catalog,
   - projection/query surfaces,
   - delivery guarantees and idempotency strategy.
5. **Proto proposal** (packages/topics/envelopes; initial v1 messages).
6. **Primitive decomposition** (which primitives exist, responsibilities, boundaries).
7. **Acceptance slices** (one end-to-end slice per value stream; test plan).

`523-commerce.md` + `523-commerce-process.md` + `523-commerce-proto.md` are an example of (2)+(3)+(4)+(5)+(6) for a single capability.

## The repeatable workflow (as a Transformation Process)

Implement this as a Transformation-owned ProcessDefinition (design-time) and a Process primitive (runtime) that orchestrates agent work.

### Step 0 — Glossary lock

Deliverable: a single page that locks the nouns for this capability.

Rule: do not overload “channel/store/site/location”; define the identity axes explicitly.

### Step 1 — Value streams (business process catalog)

Deliverable: Level 0/1 process catalog with 5–7 “first-class business processes”.

Each business process must declare:

- initiating actor/UISurface,
- inputs/outputs,
- invariants (what must be true),
- offline/edge behavior (what still works, what is gated),
- observability requirements.

### Step 2 — EDA contract (496-native)

Deliverable: message topology and catalogs.

Rules (from `496-eda-principles.md`):

- Commands/intents are imperative business verbs; avoid CRUD.
- Domain facts are immutable and past tense; emitted via transactional outbox.
- Consumers are idempotent (inbox or natural-key UPSERT).
- Cross-domain invariants are implemented as Temporal sagas (Process primitives).
- Tenant isolation + traceability metadata are required on every message.

### Step 3 — Proto shape (contract-first)

Deliverable: proto proposal doc + initial proto package plan.

Define the **proto types by role** (high-level implementation mapping):

- **Command services (Domain write APIs)**: RPCs for business intent; write side only.
- **Intent topics (optional)**: command envelopes for asynchronous command transport.
- **Domain fact streams**: immutable facts emitted by domains.
- **Process facts**: orchestration progress and governance signals emitted by processes.
- **Query services (Projection/Domain read APIs)**: read-only RPCs backed by projections/read models.
- **Integration services**: stable seams for external systems; requests/responses must be idempotent.

Mapping note (ArchiMate/EDA): facts are **Application Events**; intents/commands are **requests to invoke Application Services** (often carried asynchronously); queries are read-only Application Services.

### Step 4 — Primitive decomposition (Application realization)

Deliverable: a primitive inventory with boundaries and responsibility matrices.

Primitives are **Application Components** and they **use Technology Services** (brokers/DB/workflow/gateways) rather than being “technology primitives”.

For each primitive, define:

- owned commands/intents,
- emitted facts,
- consumed facts,
- storage (if any) and outbox/inbox requirements,
- operator-managed dependencies (DB, broker, secrets),
- deployment topology (cloud/edge/offline applicability).

### Step 5 — “One slice” implementation plan

Deliverable: one end-to-end vertical slice that forces the architecture to answer a real scenario.

Example patterns:

- Commerce: “BYOD domain connect” or “POS sale with WAN down but LAN up”.
- HR: “Employee onboarding” with approvals and notifications.

This slice produces the first executable tests and validates the contracts.

## Guardrails (how the system enforces the methodology)

- CI gates enforce proto/package/topic conventions (`509-proto-naming-conventions.md`).
- Plugins/codegen enforce required envelope metadata and idempotency scaffolds (`496-eda-principles.md`).
- Operators make runtime assumptions true (DB present, migrations, outbox dispatcher deployed, conditions surfaced) (`520-primitives-stack-v2.md`).

## Suggested “Capability skeleton” template

When creating a new capability backlog set:

- `<id>-<capability>.md` (overview, identity model, topology modes)
- `<id>-<capability>-process.md` (value streams + workflows)
- `<id>-<capability>-proto.md` (topic families, envelopes, catalogs)
- `<id>-<capability>-{domain,projection,integration,uisurface,agent}.md` (primitive responsibilities)
