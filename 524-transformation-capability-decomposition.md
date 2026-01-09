# 524 — Transformation Method: Capability → EDA Contracts → Primitives

This document generalizes the methodology used for `523-commerce*` into a repeatable **Transformation Domain** workflow:

1. Define a **business capability** as value streams + nouns + invariants.
2. Define **EDA contracts** (commands/intents, domain facts, process facts, queries, integration ports) per `496-eda-principles.md`.
3. Decompose into **Ameide primitives (Application Components)** (Domain/Process/Projection/Integration/UISurface/Agent) per `520-primitives-stack-v2.md`.

The output is a set of artifacts that can be stored as Transformation workspace nodes (ArchiMate models/views, Markdown, BPMN) and then used by agents/CLI/operators to scaffold and deploy the runtime.

For an explicit IT4IT value-stream lens (R2D/R2F/D2C) over this same pipeline, see `525-it4it-value-stream-mapping.md`.
For the agent-oriented, parallelizable implementation plan once these artifacts exist, see `backlog/533-capability-implementation-playbook.md`.

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

## Discovery discipline (524 drives decisions)

This method is the discovery driver. The goal is to avoid “architecture by assumption”.

Rules:

- Do not lock **proto package plan** (§ Step 3) until the **glossary** (§ Step 0) and **value streams** (§ Step 1) are locked.
- Do not lock **primitive boundaries** (§ Step 4) until the **EDA topology + catalogs** (§ Step 2) exist (topics, intents, facts, queries, integration seams).
- Treat any early package/topic/primitive claims as hypotheses until they are backed by artifacts produced in Steps 0–3.

Practical checklist before calling anything “foundational”:

- Glossary page exists and is referenced by every related backlog.
- Value-stream/process catalog exists with 5–7 first-class business processes and at least one “golden path”.
- EDA topology exists (topic families + intent/fact catalogs + idempotency/ordering rules).
- Proto proposal exists and follows `509-proto-naming-conventions.md` (packages + aggregator envelopes + topic mappings).
- One end-to-end slice is written down (Step 5) that forces the architecture to answer a real scenario.

Note: `527-transformation-capability.md` should be treated as the capability-definition output of this method when applied to the Transformation capability itself; keep it synchronized with these artifacts as they are produced.

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

### Step 6 — Scaffold, build, publish, deploy (delivery loop)

Deliverable: running primitives in-cluster + smoke probe pass for the slice.

This is the “turn artifacts into running reality” loop. It is repeated per primitive and per acceptance slice and should be executed via the same deterministic gates everywhere.

Required work (outer loop):

1. Scaffold primitive runtimes + GitOps wiring (implementation-owned) via the CLI (`ameide primitive scaffold`).
2. Update proto sources (if needed) and run `buf lint`, `buf breaking`, `buf generate` to refresh SDKs and generated glue.
3. Implement behavior in implementation-owned `_impl` surfaces until compilation and tests pass.
4. Build runtime binaries and run unit/integration tests (language-specific, but consistent gates).
5. Build and publish container images for:
   - primitive runtimes (Domain/Process/Projection/Integration/UISurface/Agent),
   - operators when required,
   via CI to GHCR (the only supported image publishing path).
6. Update GitOps components/values for the target environment and sync via ArgoCD.
7. Run in-cluster smoke probes (PostSync Jobs) and validate conditions/health endpoints.
8. Iterate until the slice passes end-to-end and produces the expected facts/read models.
9. Promote definitions/artifacts (baselines/plateaus) according to the chosen methodology governance flow.

For the repo-aligned end-to-end “proto shape → runtime → operator reconcile → ArgoCD sync → in-cluster probe” loop and concrete commands, use `backlog/520-primitives-stack-v2-tdd.md` plus the per-kind TDD trackers (`backlog/520-primitives-stack-v2-tdd-*.md`).

### Step 7 — Change management loop (continuous; TOGAF ADM Phase H)

Deliverable: the capability can evolve safely without drifting contracts or breaking tenants.

This step treats capability development as ongoing **Architecture Change Management**:

- Monitor drift signals (contract changes, schema changes, broken gates, SLO regressions) from facts, process facts, and CI/operator conditions.
- Propose changes as new work packages/slices and re-run Steps 0–6 with updated constraints and migration plans.
- Use governance flows (TOGAF/ADM, Scrum, PMI) to gate promotions and enforce coexistence windows/backfills/deprecations.
- Enable a tenant-scoped agent to drive users through this loop inside the transformation initiative, creating/updating the required artifacts as it goes.

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
