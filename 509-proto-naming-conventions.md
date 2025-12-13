# 509 – Proto Naming & Package Conventions

**Status:** Active baseline for new proto work  
**Audience:** Platform engineers, domain teams, SDK owners, agents  
**Scope:** Naming, packaging, and topic conventions for all Ameide proto contracts

This document defines naming and packaging rules for proto contracts. For the repo workflow (Buf generation, skeleton generation, GitOps wiring, dev image publishing), use `520-primitives-stack-v2-tdd.md`.

## Grounding & contract alignment

- **Architecture grounding:** Extends the proto‑first, SDK‑first contract chain defined in `470-ameide-vision.md` and `472-ameide-information-application.md` into concrete naming rules, so packages across domains, Transformation, Process, Agents, and CLI stay coherent.  
- **EDA & runtime seams:** Aligns with the event‑driven rules in `496-eda-principles.md` and the Scrum runtime seam in `506-scrum-vertical-v2.md` / `508-scrum-protos.md`, ensuring domain/process/agent messages share a consistent envelope and naming style.  
- **Primitive & operator ecosystem:** Provides a shared naming baseline for proto references used by primitives and operators (`495-ameide-operators.md`, `498-domain-operator.md`, `499-process-operator.md`, `500-agent-operator.md`, `501-uisurface-operator.md`, `502-domain-vertical-slice.md`, `503-operators-helm-chart.md`, `504-agent-vertical-slice.md`).  
- **Scrum & agent stack:** Treats the Scrum packages in `508-scrum-protos.md` and the agent contracts in `505-agent-developer-v2.md` / `505-agent-developer-v2-implementation.md` as exemplars of these conventions; treat older packages that diverge as historical and track migrations explicitly.

> **Related proto/Buf backlogs**
> - [402-buf-first-generated-clients.md](402-buf-first-generated-clients.md) – Buf-first client strategy  
> - [403-ts-proto-first-migrations.md](403-ts-proto-first-migrations.md) – TS proto-first service migrations  
> - [404-go-buf-first-migration.md](404-go-buf-first-migration.md) – Go proto/Buf/SDK migration  
> - [407-single-proto-surface-sdk.md](407-single-proto-surface-sdk.md) – Single proto surface via SDKs  
> - [410-bsr-native.md](410-bsr-native.md) – BSR-native module & stub sync  
> - [484b-ameide-cli-proto-contract.md](484b-ameide-cli-proto-contract.md) – `ameide_core_proto.primitive.v1` contract for CLI  
> - [300-400/367-0-feedback-intake.md](300-400/367-0-feedback-intake.md), [300-400/367-1-scrum-transformation.md](300-400/367-1-scrum-transformation.md), [300-400/367-5-ameide-journey.md](300-400/367-5-ameide-journey.md), [300-400/370-event-sourcing.md](300-400/370-event-sourcing.md) – use per-method packages such as `ameide_core_proto.transformation.scrum.v1` consistently with these rules.

---

## 1. Package naming

All Ameide protos share a single root namespace:

```proto
package ameide_core_proto.<bounded_context>[.<subcontext>].v<major>;
```

### 1.1 Bounded‑context packages

Examples (non‑exhaustive):

- **Primitive meta (CLI / control plane):**  
  - `ameide_core_proto.primitive.v1` – generic primitive metadata (`PrimitiveInfo`, `DomainDetails`, `AgentDetails`, etc.).
- **Business domains:**  
  - `ameide_core_proto.platform.v1` – tenants, orgs, users, roles.  
  - `ameide_core_proto.sales.v1`, `ameide_core_proto.billing.v1`, etc.
- **Transformation & methodologies:**  
  - `ameide_core_proto.transformation.core.v1` – ProcessDefinitions, AgentDefinitions, methodology profiles.  
  - `ameide_core_proto.transformation.scrum.v1` – Scrum artifacts/intents/facts/query (see `508-scrum-protos.md`).  
  - Future: `ameide_core_proto.transformation.safe.v1`, `...transformation.togaf.v1` using the same pattern.
- **Process facts (governance):**  
  - `ameide_core_proto.process.scrum.v1` – Scrum process facts (`ScrumProcessFact` and friends) backing `scrum.process.facts.v1`.
- **Cross‑cutting:**  
  - `ameide_core_proto.governance.v1`, `...observability.v1`, etc.

**Rules:**

- One package per bounded context + major version (`…v1`, `…v2`, …).  
- No “bare” packages like `scrum.v1`, `process.facts.v1`, or `governance.v1` outside the `ameide_core_proto` root.  
- Subcontexts are only used when the top‑level context would become too broad (`transformation.scrum`, `process.scrum`, etc.).

---

## 2. Directory layout

Under `packages/ameide_core_proto/src/ameide_core_proto/...`, files are grouped by package and concern, but share the same `package` declaration:

```text
packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/
  transformation-scrum-common.proto      # envelope/meta/identity
  transformation-scrum-artifacts.proto   # aggregates (ProductBacklog, Sprint, Increment, …)
  transformation-scrum-intents.proto     # ScrumDomainIntent + per-intent payloads
  transformation-scrum-facts.proto       # ScrumDomainFact + per-fact payloads
  transformation-scrum-query.proto       # ScrumQueryService + request/response

packages/ameide_core_proto/src/ameide_core_proto/process/scrum/v1/
  process-scrum-facts.proto              # ScrumProcessFact + process facts
```

Conventions:

- All files in a directory share the same `package` and language‑specific options (`go_package`, TS/JS options, etc.).  
- Use hyphenated file names scoped by context (`transformation-scrum-*`, `process-scrum-*`, `primitive-*`) instead of generic names like `messages.proto` or `events.proto`.

---

## 3. Message naming: commands vs events

We distinguish three classes of messages (see `506-scrum-vertical-v2.md §3`):

1. **Domain intents** – commands over the bus (“please change state”).  
2. **Domain facts** – immutable facts about state changes (“state changed”).  
3. **Process facts** – immutable governance/timebox facts (timers, windows, work‑available cues).

### 3.1 Suffix rules

- **Domain intents:** `…Requested`
  - Examples: `CreateProductBacklogItemRequested`, `CommitSprintBacklogRequested`, `RecordProductBacklogItemDoneRequested`, `RecordIncrementRequested`.
- **Domain facts:** past‑tense or “Defined/Created/Recorded/Updated`
  - Examples: `ProductBacklogItemCreated`, `ProductBacklogItemRefined`, `SprintBacklogCommitted`, `ProductBacklogItemDoneRecorded`, `IncrementUpdated`.
- **Process facts:** explicit governance/timebox vocabulary
  - Examples: `SprintPlanningWindowOpened`, `SprintBacklogReadyForExecution`, `SprintBacklogItemReadyForWork`, `SprintStartingSoon`, `SprintTimeboxReachedEnd`.

Avoid:

- Generic verbs like `AdvanceLifecycleState`, `SetStatus`, `UpdateX`.  
- Requirement/WorkItem‑centric names (`CompleteRequirement`, `ItemReadyForDev`) in new contracts; keep them only as legacy mapping in historical docs (e.g., `506-scrum-vertical.md`).

### 3.2 Aggregator messages

Use a single envelope per class, per context:

```proto
// Scrum domain intents
message ScrumDomainIntent {
  ScrumMessageMeta meta = 1;
  oneof intent {
    SetProductGoalRequested set_product_goal_requested = 10;
    CreateProductBacklogItemRequested create_product_backlog_item_requested = 11;
    // …
  }
}

// Scrum domain facts
message ScrumDomainFact {
  ScrumMessageMeta meta = 1;
  ScrumAggregateRef aggregate = 2;
  oneof fact {
    ProductGoalSet product_goal_set = 10;
    ProductBacklogItemCreated product_backlog_item_created = 11;
    // …
  }
}

// Scrum process facts
message ScrumProcessFact {
  ScrumMessageMeta meta = 1;
  oneof fact {
    SprintBacklogReadyForExecution sprint_backlog_ready_for_execution = 10;
    SprintBacklogItemReadyForWork sprint_backlog_item_ready_for_work = 11;
    // …
  }
}
	```

Other domains follow the same pattern (`<Context>Command`, `<Context>Event` or `<Context>DomainFact`, `<Context>ProcessFact`), with intents carrying request payloads and facts carrying state snapshots or diffs.

---

## 4. Envelope & identity types

To keep EDA semantics consistent (per `496-eda-principles.md`), all event‑carrying packages share a common envelope shape inside their own context:

- **Message envelope:** `ScrumMessageMeta` (for Scrum); other contexts define analogous types (`OrdersMessageMeta`, etc.) and retain the same semantics.
- **Subject/routing key:** `ScrumSubject` – `product_id`, `sprint_id`, `product_backlog_item_id`, etc.  
- **Actor attribution:** `ScrumActor` – `ScrumAccountability` + `actor_id`.  
- **Aggregate identity:** `ScrumAggregateRef` – `type`, `id`, `version` (monotonic).

Envelope fields:

- `message_id` – globally unique idempotency key.  
- `schema_version` – semantic version of payload + envelope.  
- `occurred_at` – producer‑asserted timestamp.  
- `producer` – service/workload identity.  
- `tenant_id` – multi‑tenant routing + isolation.  
- `correlation_id` / `causation_id` – tracing and event chaining.  
- `subject` – domain‑specific routing keys.  
- `actor` – optional attribution for who caused the change.

For **domain facts**, keep `ScrumAggregateRef.version` monotonic per aggregate; downstream consumers (Temporal, analytics, projections) rely on this for idempotency and ordering.

---

## 5. Topic naming

Topic (or subject/stream) names are logical; encode class and version and map 1:1 to the aggregator message types:

- `scrum.domain.intents.v1` → payload `ameide_core_proto.transformation.scrum.v1.ScrumDomainIntent`.  
- `scrum.domain.facts.v1` → payload `ameide_core_proto.transformation.scrum.v1.ScrumDomainFact`.  
- `scrum.process.facts.v1` → payload `ameide_core_proto.process.scrum.v1.ScrumProcessFact`.

Pattern:

```text
<methodology>.<class>.<kind>.v<major>
  where class ∈ { domain, process }
        kind  ∈ { intents, facts }
```

### 5.1 Projections and integrations

These conventions apply across the six primitives; they do **not** introduce a second behavior DSL.

- **Projection messages:** use a `*Projection` suffix (e.g., `SalesOrderProjection`) and keep them in the bounded context package that owns the semantics (typically the originating domain or process context).
- **Projection services:** use `*QueryService` or `*ProjectionQueryService` names and keep them read-only (queries only).
- **Integration services:** use `*IntegrationService` names for stable external seams; do not encode environment-specific endpoints or secrets in proto (bind them at runtime).

Rules:

- Match topic version to proto package major (`*.v1` ↔ `...v1`).  
- Do not invent synonyms (`scrum.events.v1`, `scrum.stream.v1`) for the same payload; if you need multiple consumer groups, use broker‑specific constructs (consumer groups, partitions), not separate semantic topics.

---

## 6. Versioning rules

Versioning happens at the package/topic level, not on individual messages:

- Proto packages are suffixed with `v<major>` (`...v1`, `...v2`, …).  
- Topics follow the same major version suffix.  
- Within a given package:
  - Never change field numbers or remove/rename fields.  
  - Add new fields with new numbers and sensible defaults.  
  - Use `reserved` for removed fields.

Breaking changes require:

- A new proto package (e.g., `ameide_core_proto.transformation.scrum.v2`).  
- New logical topics (`scrum.domain.intents.v2`, `scrum.domain.facts.v2`, `scrum.process.facts.v2`).  
- Migration backlogs that describe how v1/v2 coexist and how consumers upgrade.

---

## 7. Legacy & compatibility

Several older backlogs and packages pre‑date these conventions (e.g., `506-scrum-vertical.md`’s `scrum.intents.v1` / `scrum.facts.v1` naming, Requirement‑centric events, or neutral “WorkItem” abstractions). Apply the rules above to new work:

- Use `ameide_core_proto.<context>[.<sub>].v<major>` for new packages.  
- Use the suffixes and aggregator patterns in §3 for new commands/events.  
- Use the `<methodology>.<class>.<kind>.v<major>` pattern for new topics and map them to the aggregator messages.

Legacy packages and names:

- Keep them in the repo for migration and compatibility.  
- Label them as historical in their backlogs (see the contract‑alignment/supersedence boxes added to 505‑legacy and 506‑legacy).  
- Do not use them as the basis for new implementations; target the v1 packages that conform to this document.

As new domains/methodologies are added (e.g., SAFe, TOGAF), their proto backlogs reference this file and mirror the Scrum packages (`transformation.safe.v1`, `process.safe.v1`, etc.) instead of inventing a new naming scheme.
