# 530 — Ameide Capability Design Worksheet (ArchiMate-aligned)

**Status:** Draft (worksheet)  
**Audience:** Architecture, Transformation, domain/solution architects, platform engineering, agents  
**Purpose:** Provide a repeatable, ArchiMate-consistent worksheet for defining a new **Capability** (Strategy layer) and its realization via Ameide’s contract-first primitives pipeline (Application/Technology layers).

**Design-time language note:** ArchiMate 3.2 notation is Ameide’s official design-time language for capability/architecture models, and those models live inside the **Transformation capability** (persisted by the Transformation Domain, edited via Transformation design tooling). BPMN-compliant ProcessDefinitions and Markdown specs are companion design-time artifacts in the same store.
**Use with:**
- Capability definition: `backlog/528-capability-definition.md`
- ArchiMate vocabulary/verbs: `backlog/529-archimate-alignment-470plus.md`
- Capability method: `backlog/524-transformation-capability-decomposition.md`
- EDA contract rules: `backlog/496-eda-principles.md`, `backlog/496-eda-protobuf-ameide.md`
- Primitives pipeline: `backlog/470-ameide-vision.md`, `backlog/477-primitive-stack.md`, `backlog/520-primitives-stack-v2.md`

## Layer header (cross-layer worksheet)

- **Primary ArchiMate layer(s):** Strategy (Capability, Value Stream).
- **Secondary layers referenced:** Motivation; Business; Application; Technology; Implementation & Migration.
- **Primary element types used:** Capability, Value Stream, Business Process, Application Service/Interface/Event, Application Component, Technology Service, Work Package/Deliverable/Plateau/Gap.
- **Prohibited unless qualified:** process, service, domain, event (qualify per `backlog/470-ameide-vision.md` §0.2).

---

## Motivation (Motivation layer)

**What it answers:** *Why are we doing this and what rules constrain us?*

- **Principles / constraints:** Ameide’s core invariants (tenant isolation, contract-first chain, Graph read-only, Transformation inside the product, EDA outbox, etc.).
- Use this layer to capture **non-negotiables** and link to the canonical invariant source rather than re-stating it (`backlog/470-ameide-vision.md`).

---

## Strategy (Strategy layer)

**What it answers:** *What ability are we building and what value stream delivers it?*

- **Capability:** the Strategy-layer “ability” you’re delivering (e.g., Sales, Finance, Commerce, Transformation).
- **Value Stream:** the staged flow that delivers value to stakeholders/end users.
- In Ameide, a capability definition is expected to produce: value streams, nouns/invariants, and a contract-first application surface that is realized by primitives (`backlog/528-capability-definition.md`, `backlog/524-transformation-capability-decomposition.md`).

**Relationship rule (Strategy):**

- Capability **serves** the Value Stream (`backlog/529-archimate-alignment-470plus.md`).

---

## Business (Business layer)

**What it answers:** *What does the organization do (independent of software) to run the value stream?*

* **Business Processes**: the human/org workflows for the value stream stages (e.g., “Qualify lead”, “Approve credit”, “Close month”).
* **Business roles/actors**: salesperson, finance controller, approver, customer, etc.
* **Business objects**: Invoice, Payment, Opportunity, Ledger Entry (semantic meaning).

**Important Ameide disambiguation to enforce in every capability doc:**

* “Business Process” ≠ “Process primitive” (Ameide’s runtime orchestration component).

---

## Application (Application layer)

**What it answers:** *What application services/events/interfaces implement the business behavior—and what components realize them?*

### A) Application Components (Ameide primitives)

Treat Ameide primitives as **Application Components**:

* Domain / Process / Projection / Integration / UISurface / Agent

This is the “implementation structure” you repeatedly refer to across the architecture docs.

### B) Application Services & Interfaces (proto-first boundaries)

In Ameide, **proto contracts are the application boundary** (services + messages + event envelopes).
If backlog prose contradicts `.proto`, the `.proto` wins (`backlog/496-eda-protobuf-ameide.md`).
Map them like this:

* **Application Service** = exposed behavior (RPC + read-only query APIs).
* **Application Interface** = point of access (gRPC/HTTP endpoints; topic-family subscription surfaces).
* **Data Objects** = protobuf messages/entities/envelopes.

### C) Application Events (EDA facts)

* **Domain facts / Process facts** map to **Application Events** (facts represent state change).
* **Commands/intents** are best described as **requests to invoke Application Services**, even if transported asynchronously over a topic family (`backlog/496-eda-protobuf-ameide.md`).

### D) CQRS rule (Ameide’s default blueprint)

- **Writes:** Domain commands (RPC) and/or Domain intents (bus) handled by Domain primitives; Process primitives orchestrate and emit process facts but do not become the system-of-record for domain state.
- **Reads:** via query services realized by Projections/read models; Graph is a read-only projection (`backlog/470-ameide-vision.md`).

### E) Classification + routing rules (avoid “intent vs event” drift)

Use these rules while filling the contract inventory:

- **Classify by state ownership, not by producer.** A UISurface/Agent can emit a *Domain intent*; a Process primitive can emit a *Process fact*; classification is about whose state changes or is being asserted (`backlog/496-eda-principles.md`).
- **Stable topic families, not event-per-topic, by default.** Prefer one topic family per semantic class and one aggregator message per topic family (intents/facts/process facts) to minimize routing drift (`backlog/496-eda-protobuf-ameide.md`).
- **“Facts are events; intents are requests.”** Transport may be evented, but semantics remain: facts assert state change, intents request behavior.

### F) Buf/BSR-native contract inventory (worksheet)

Fill these tables and treat them as the authoritative “application boundary index” for the capability.

#### F.1 Protobuf module(s)

For each Buf module that contains the capability’s contracts:

- Module name (BSR): `buf.build/<org>/<module>`
- Local module dir: (e.g., `packages/<name>/src`)
- Lint profile: `STANDARD` (no permanent exceptions)
- Breaking policy: `FILE` (minimum); record the chosen category and rationale if stricter/looser
- Dependencies: (e.g., `googleapis`, `protovalidate`, `confluent` extensions)
- Generation templates: list `buf.gen.*.yaml` files that must run for local dev

#### F.2 Application Services (RPC + query services)

For each service:

- Service: `<package>.<ServiceName>`
- Proto file path: `.../*.proto`
- Owning primitive: Domain/Process/Projection/Integration/UISurface/Agent
- Write vs read: `write` (command) / `read` (query)
- Auth boundary: anonymous / user / service / mixed
- Idempotency key strategy (write services): where does `message_id` come from and how is it enforced?

#### F.3 Topic families (interfaces) + aggregator messages + CSR subjects

For each topic family used by the capability:

- Topic family: `<capability>.<class>.<kind>.vN` (examples: `<cap>.domain.intents.v1`, `<cap>.domain.facts.v1`, `<cap>.process.facts.v1`)
- Purpose: what semantic class flows here (intents vs facts vs process facts)
- Producing primitive(s): which component(s) publish here
- Consuming primitive(s): which components subscribe here
- Aggregator message: `<X>DomainIntent` / `<X>DomainFact` / `<X>ProcessFact` (or equivalent)
- Partition keys: tenant + aggregate identity (and any additional routing keys)
- Envelope metadata (required): `message_id`, `schema_version`, `occurred_at`, `producer`, `tenant_id` (if multi-tenant), `correlation_id`, `causation_id`, and W3C Trace Context (`traceparent`, `tracestate`) when available

**CSR binding (Buf/BSR):**

- CSR subject (TopicNameStrategy style): `<topic>-value`
- Subject annotation location:
  - Proto file: `.../*.proto`
  - Message name: `<AggregatorMessage>`
  - Must declare `option (buf.confluent.v1.subject) = { instance_name: "...", name: "<topic>-value" };`

#### F.4 Versioning vocabulary (be explicit)

Use these terms consistently in capability docs and PRs:

- **Package version**: the `vN` in `package ...vN;` (semantic major for schemas)
- **Module version**: the BSR module version/label (distribution unit)
- **Subject version**: the schema history version within a CSR subject

Record the rules for this capability:

- When do we bump package major (`v1` → `v2`)?
- What Buf breaking category is enforced in CI?
- What CSR compatibility mode is expected by consumers (if applicable)?

---

## Technology (Technology layer)

**What it answers:** *What platform services and runtime nodes host those application components?*

- **Technology Services:** Kubernetes + GitOps + operators, workflow runtime (Temporal), broker (NATS/Kafka), DB (Postgres/CNPG), gateway/IAM, observability (`backlog/473-ameide-technology.md`).
- Separation rule: domain logic does not import broker clients; outbox + adapters enforce that (`backlog/470-ameide-vision.md`, `backlog/496-eda-protobuf-ameide.md`).

---

## Implementation & Migration (Implementation & Migration layer)

**What it answers:** *How do we deliver this capability safely over time?*

- **Work Packages / Deliverables / Plateaus / Gaps** (phased rollouts, fit-gap, migration steps).
- This is where “v1 slice”, “dual-running”, “backfill projections”, “deprecate v0 topics” belongs (`backlog/483-fit-gap-analysis.md` is the living gap tracker).

### Delivery loop checklist (generic)

Capture the “minimum repeatable loop” for delivering the capability, without embedding capability-specific semantics:

- Scaffold the external structure (repo wiring, GitOps manifests, skeleton code) via the Ameide CLI.
- Generate deterministic internal artifacts (SDKs, generated-only glue) via `buf generate`.
- Enforce contract quality gates:
  - `buf lint` (STANDARD)
  - `buf breaking` (chosen policy)
- Plan migration explicitly (if changing existing subjects/topics):
  - dual-publish or dual-consume window
  - projection backfill strategy + cutover criterion
  - deprecation milestone for old topics/packages

---

# The Ameide “chain” you should reuse for every new capability

Use this exact phrasing structure as your blueprint:

1. **Capability (Strategy)** → *serves* → **Value Stream (Strategy)**
2. **Value Stream** → *realized by* → **Business Processes (Business)**
3. **Business Processes** → *use / are served by* → **Application Services (Application)**
4. **Application Services** → *realized by* → **Application Components (Ameide primitives)**
5. **Application Components** → *use* → **Technology Services (Technology)**
6. Delivery evolves via **Work Packages/Plateaus** (Implementation & Migration)

This is the canonical chain in `backlog/529-archimate-alignment-470plus.md` and prevents the common layering error: describing a capability directly as a “primitive decomposition” without the Strategy/Business/Application boundary.

---

# A practical “capability blueprint” template (copy/paste for Sales, Finance, etc.)

### 1) Strategy

* Capability statement (one sentence)
* Value stream stages (5–10 stages, stakeholder-defined)
* Key policies/invariants (what must always be true)

### 2) Business

* Business processes per stage
* Roles/actors + responsibilities
* Business objects + outcomes

### 3) Application

* **Domain primitives** (bounded contexts; single-writer aggregates)
* **Process primitives** (orchestration/workflows; timers/timeboxes)
* **UISurface** (write surfaces; read via projections)
* **Projection** (read models, search views, reporting views)
* **Integration** (external system boundaries; inbound/outbound)
* **Agent** (assist/propose; durable only via domain writes)

Plus the contract inventory:

* Application Services (RPC + query services) + authoritative `.proto` file list (do not paste full contracts)
* Application Interfaces (gRPC/HTTP bindings; topic families)
* Application Events (domain facts + process facts)
* Data Objects (proto schemas + envelope metadata)
* Tenant/partitioning rules (keys + required envelope metadata)

### 4) Technology

* Required tech services (DB, broker, workflow, gateway, observability)
* Runtime topology constraints (cloud/edge/offline)
* SLO/SLI expectations (latency, freshness of projections, DLQ depth)

### 5) Implementation & Migration

* v1 slice (minimum value stream stage subset)
* coexistence/migration plan (topics, schema versions, backfills)
* deprecation plan
