# Scrum governance contract target state

This is a clean, **Scrum-Guide-aligned** contract for structuring Scrum as:

* **Scrum Domain (Transformation)** = *system of record* for Scrum artifacts + commitments (data and rules), **reactive only** (no timers/orchestration).
* **Scrum Process (Temporal)** = *timebox governor* (timers, reminders, orchestration), **event-driven only** (no direct domain writes/calls).
* **Domain ↔ Process integrate only through messages on the event bus** (domain “facts”, domain “intents”, and process “facts”).  

The Scrum wording used below follows the **official Scrum Guide**: Scrum Team accountabilities, Scrum events, Scrum artifacts, and their commitments.

---

## Grounding & cross-references

- **Role in architecture:** Canonical Scrum runtime seam between Transformation (Scrum domain) and Process primitives; defines envelopes, topics, and Temporal workflow structure.  
- **Domain side:** Builds on `300-400/367-1-scrum-transformation.md` and the `transformation_scrum_*` protos in `508-scrum-protos.md`; Transformation persists Scrum artifacts and emits the domain facts referenced here.  
- **Process side:** Consumed by Process primitives described in `499-process-operator.md` and positioned in the primitive stack by `477-primitive-stack.md`; Temporal workflows must obey the intent/fact separation and idempotency rules in this backlog and the EDA rules in `496-eda-principles.md`.  
- **Agents & tooling:** `505-agent-developer-v2.md` and `505-agent-developer-v2-implementation.md` describe how AmeidePO/AmeideSA/AmeideCoder react to process/domain facts from this seam; `495-ameide-operators.md` and `502-domain-vertical-slice.md` provide shared operator/condition vocabulary; `507-scrum-agent-map.md` locates this contract at Stage 2 in the Scrum stack.
- **IT4IT mapping lens:** How this seam fits “requirement → running” streams (R2D/R2F/D2C): `525-it4it-value-stream-mapping.md`
- **Capability implementation DAG:** `backlog/533-capability-implementation-playbook.md` (generic DAG; 506 is a concrete instantiation of the end-to-end seam proof).

---

## Vendor best-practice alignment (Scrum, Temporal, Proto/EDA)

This contract is intentionally aligned with three “vendors of truth”:

1. **Scrum Guide (domain semantics)**  
   - The **Transformation Scrum subdomain** models *only* Scrum artifacts and commitments using Scrum terms (Product Backlog / Product Goal, Sprint Backlog / Sprint Goal, Increment / Definition of Done).  
   - No “requirements”, “phase gates”, or “Definition of Ready” appear as Scrum domain concepts; if they exist, they live as **policy or platform extensions**, not state in this contract.

2. **Temporal (process/orchestration semantics)**  
   - All non-deterministic concerns (bus consumption, network I/O, clocks) live in an **ingress router + Activities**; Temporal workflows remain **deterministic and signal-driven**.  
   - If `ContinueAsNew` is used for “entity workflows”, deduplication is handled explicitly in workflow state (using `aggregate_version` and “already emitted” markers), because Temporal’s built-in dedupe does **not** span runs.

3. **Proto / EDA conventions (contract semantics)**  
   - One root proto namespace with **versioned packages** (see `508-scrum-protos.md` and `509-proto-naming-conventions.md`); **topic major = package major**.  
   - Exactly **three message classes** across this seam: domain intents, domain facts, process facts, all sharing a common envelope.  
   - Domain facts carry monotonic `aggregate_version` per aggregate (for Temporal idempotency).  
   - Bus topics use **aggregator messages** (e.g., `ScrumDomainIntent`, `ScrumDomainFact`, `ScrumProcessFact`) with a `oneof` payload to keep topics stable while individual event types evolve.

The rest of this backlog translates those vendor-aligned constraints into a concrete domain/process split and a bottom‑up implementation plan.

---

## 1. Canonical Scrum vocabulary we will model

### 1.1 Scrum Team accountabilities

The Scrum Team has three accountabilities:

* **Product Owner**
* **Scrum Master**
* **Developers**

> In the contract, these are **actor accountabilities** used for authorization/audit (who can trigger which intents), not “workflow steps”.

---

### 1.2 Scrum events

Scrum events are:

* The **Sprint** (container event)
* **Sprint Planning**
* **Daily Scrum**
* **Sprint Review**
* **Sprint Retrospective**

> The **domain** stores the *outcomes* of these events (e.g., Sprint Goal, Sprint Backlog selection). The **process** governs their *timeboxes and scheduling*.

---

### 1.3 Scrum artifacts and commitments

Scrum artifacts:

* **Product Backlog** (commitment: **Product Goal**)
* **Sprint Backlog** (commitment: **Sprint Goal**)
* **Increment** (commitment: **Definition of Done**)

Also: a **Product Backlog Item** that meets the **Definition of Done** becomes part of an **Increment**.

---

## 2. Data contract owned by the Scrum Domain

### 2.1 Design principles

1. **The domain is authoritative for Scrum artifacts and commitments.** (All persistent Scrum state lives here.) 
2. **The domain never “ticks”.** No timeouts, cron, or Temporal logic inside the domain.
3. **All changes happen by processing a “domain intent” message**, and the domain emits **domain facts** after validation and persistence.
4. **The domain does not model “how the work is executed”** (tools, agents, CI). Those are platform concerns that can attach “evidence” as an extension, but Scrum state remains Scrum state.

---

### 2.2 Core aggregates

Below are the minimum aggregates to represent Scrum **exactly** using official terms.

#### A. Product Backlog

**Artifact:** Product Backlog
**Commitment:** Product Goal

Required fields:

* `product_id`
* `product_goal` (see below)
* `ordering_policy` (how items are ordered: by value, risk, etc. — implementation-specific)
* `ordered_item_ids[]` (the order is material)

Invariants:

* There is exactly **one Product Backlog per product**.
* A Product Goal is **the commitment** for the Product Backlog and is the planning target for the Scrum Team.

#### B. Product Goal

Store as a value object or separate aggregate.

Required fields:

* `product_goal_id`
* `product_id`
* `description`
* `status` (e.g., `ACTIVE | ACHIEVED | ABANDONED`)
* `effective_from`, `effective_to?`

Invariants:

* At most one `ACTIVE` Product Goal per product at a time.

#### C. Product Backlog Item

**Artifact element:** Product Backlog item (item of Product Backlog)

Required fields:

* `product_backlog_item_id`
* `product_id`
* `title`
* `description`
* `acceptance_criteria[]` (optional but common)
* `size` (optional; sizing is a Developers responsibility in Scrum)
* `status` *(domain needs a status; Scrum Guide does not prescribe one — keep it minimal)*:

  * `IN_PRODUCT_BACKLOG`
  * `SELECTED_FOR_SPRINT`
  * `DONE` (means “meets Definition of Done”)
* `selected_sprint_id?`
* `done_definition_of_done_id?`
* `done_at?`

Invariants:

* If `status = DONE`, the item **must reference the Definition of Done used** and must be eligible to be included in an Increment.

#### D. Sprint

**Event container:** Sprint

Required fields:

* `sprint_id`
* `product_id`
* `start_at`
* `end_at`
* `status`: `PLANNED | STARTED | ENDED | CANCELED`
* `cancellation_reason?`

Scrum constraints to enforce:

* Sprint length is fixed once started (you can enforce that as “no end_at extension after STARTED”).
* Sprint cancellation is permitted; in Scrum it’s a Product Owner accountability to cancel when the Sprint Goal becomes obsolete (capture cancellation intent/audit).

#### E. Sprint Backlog

**Artifact:** Sprint Backlog
**Commitment:** Sprint Goal

In Scrum terms, the Sprint Backlog consists of:

* the **Sprint Goal**
* the set of **Product Backlog Items selected for the Sprint**
* an actionable **plan** for delivering the Increment

Model it as a child object of Sprint:

Required fields:

* `sprint_id`
* `sprint_goal` (see below)
* `selected_item_ids[]`
* `plan` (optional structure; can be free text or structured “tasks”)

Invariants:

* A Sprint Backlog must exist for a started Sprint (because Developers need a plan).
* The Sprint Goal must be finalized within Sprint Planning (domain can enforce “SprintBacklogCommitted must include Sprint Goal”).

#### F. Sprint Goal

Store as value object.

Required fields:

* `sprint_goal_id`
* `sprint_id`
* `description`

Invariants:

* Exactly one Sprint Goal per Sprint.

#### G. Definition of Done

**Commitment:** Definition of Done (for the Increment)

Required fields:

* `definition_of_done_id`
* `product_id`
* `text` (or checklist structure)
* `version`
* `effective_from`

Invariants:

* If the DoD is an organizational standard, treat it as the minimum baseline; otherwise the Scrum Team defines it (store provenance).

#### H. Increment

**Artifact:** Increment
**Commitment:** Definition of Done

Important Scrum nuance: multiple Increments may be created within a Sprint.

Model option that stays faithful and still implementable:

* `increment_id` (unique)
* `product_id`
* `sprint_id`
* `definition_of_done_id`
* `included_item_ids[]` (items that meet DoD and are integrated)
* `created_at`
* `evidence_refs[]` *(platform extension — see below)*

Invariants:

* Work cannot be considered part of an Increment unless it meets the Definition of Done (domain enforces this by only adding DONE items to Increment).

---

### 2.3 Optional extension: “Evidence” (platform extension, not Scrum)

Scrum doesn’t define “evidence artifacts”, but platforms do. Keep it as an **extension attached to Increment or Product Backlog Item DONE**:

* `evidence_ref` could link to build/test logs, PR URLs, etc.
* This never changes Scrum semantics; it only supports auditability and automation.

(If you later want to drive automation/agents, this is where you anchor “how we know it meets DoD”.)

---

## 3. Message contract

### 3.1 Three message classes

1. **Domain intents**: “Please change Scrum state” (validated by domain).
2. **Domain facts**: “Scrum state changed” (emitted by domain, immutable).
3. **Process facts**: “A timebox/governance condition occurred” (emitted by process, immutable).

This preserves the separation: domain is reactive; process triggers by publishing intents and its own governance facts.  

---

### 3.2 Required message envelope

Every message **MUST** include:

* `message_id` (globally unique)
* `schema_version`
* `occurred_at` (timestamp)
* `producer` (service identity)
* `tenant_id` (if multi-tenant)
* `correlation_id` (end-to-end trace)
* `causation_id` (triggering message_id)
* `subject` (routing keys):

  * `product_id` (always)
  * `sprint_id` (if applicable)
  * `product_backlog_item_id` (if applicable)

For **domain facts** specifically:

* `aggregate_type`
* `aggregate_id`
* `aggregate_version` (**MUST** be monotonic per aggregate) — critical for Temporal idempotency/deduplication.  

---

### 3.3 Recommended topics/streams

* `scrum.domain.intents.v1`
* `scrum.domain.facts.v1`
* `scrum.process.facts.v1` 

---

## 4. Domain intents

These are the minimal bus-published intents the domain should support.

### 4.1 Product Backlog / Product Goal

* `SetProductGoalRequested`

  * `product_id`, `description`, `effective_from`

* `CreateProductBacklogItemRequested`

  * `product_id`, `title`, `description`, `acceptance_criteria?`, `initial_order_hint?`

* `RefineProductBacklogItemRequested`

  * `product_backlog_item_id`, `patch` (fields to update)

* `OrderProductBacklogRequested`

  * `product_id`, `ordered_item_ids[]` *(explicit full order is simplest and auditable)*

### 4.2 Sprint + Sprint Backlog (Sprint Planning outcomes)

* `CreateSprintRequested`

  * `product_id`, `start_at`, `end_at`

* `StartSprintRequested`

  * `sprint_id`

* `CancelSprintRequested`

  * `sprint_id`, `reason`

* `EndSprintRequested`

  * `sprint_id`

* `CommitSprintBacklogRequested` *(this is the durable outcome of Sprint Planning)*

  * `sprint_id`
  * `sprint_goal.description`
  * `selected_item_ids[]`
  * `plan?` (free text or structure)

* `ChangeSprintBacklogRequested` *(scope/plan adjustments during the Sprint)*

  * `sprint_id`
  * `add_item_ids[]?`, `remove_item_ids[]?`, `plan_patch?`

### 4.3 Definition of Done + Increment

* `DefineDefinitionOfDoneRequested`

  * `product_id`, `text`, `version`

* `RecordProductBacklogItemDoneRequested`

  * `product_backlog_item_id`
  * `sprint_id`
  * `definition_of_done_id`
  * `evidence_refs[]?` *(extension)*

---

## 5. Domain facts

For each accepted intent, the domain emits facts that reflect the authoritative new state.

### 5.1 Product Backlog / Product Goal

* `ProductGoalSet`
* `ProductBacklogItemCreated`
* `ProductBacklogItemRefined`
* `ProductBacklogReordered`

### 5.2 Sprint / Sprint Backlog

* `SprintCreated`
* `SprintStarted`
* `SprintCanceled`
* `SprintEnded`
* `SprintBacklogCommitted`
* `SprintBacklogChanged`

### 5.3 Definition of Done / Increment

* `DefinitionOfDoneDefined`
* `ProductBacklogItemDoneRecorded`
* `IncrementUpdated` *(or `IncrementCreated` + `IncrementUpdated` if you want an explicit first creation event)*

All domain facts carry `aggregate_version`.

---

## 6. Process facts

These are emitted by the Temporal process primitive to govern timeboxes and to notify downstream automation/agents/humans.

### 6.1 Scrum event scheduling (pure governance)

* `SprintPlanningWindowOpened`
* `DailyScrumWindowOpened` *(one per working day of the Sprint; you can also emit “window closed” if useful)*
* `SprintReviewWindowOpened`
* `SprintRetrospectiveWindowOpened`

These are **not** domain state changes; they are timebox notifications so actors can act within timeboxes.

### 6.2 Sprint timebox governance

* `SprintStartingSoon`
* `SprintEndingSoon`
* `SprintTimeboxReachedEnd` *(governance fact that the timebox ended; process will also publish `EndSprintRequested` into domain intents)*

### 6.3 Platform extension: dispatchable work cues

Scrum does not require the system to “push” items to Developers, but automation often needs a **clear trigger** that *work is available*. Keep these explicitly labeled as platform extensions:

* `SprintBacklogReadyForExecution`

  * emitted when **both**:

    * domain fact `SprintStarted` exists, and
    * domain fact `SprintBacklogCommitted` exists

* `SprintBacklogItemReadyForWork`

  * emitted per item when it is in the Sprint Backlog and the Sprint is started

This keeps Scrum semantics intact while enabling orchestration.

---

## 7. Process primitive structure in Temporal

### 7.1 Components

1. **Bus consumer / ingress router** (non-deterministic)

   * subscribes to `scrum.domain.facts.v1`
   * routes facts into Temporal via `SignalWithStart`
   * uses deterministic workflow IDs (e.g., `tenant/product/sprint/...`)

2. **Temporal workflows** (deterministic)

   * keep only *derived* state needed for gating/timers
   * dedupe using `aggregate_version` and/or processed message IDs 

---

### 7.2 Workflow hierarchy

This is where **nested workflows** are genuinely valuable.

#### A. `SprintWorkflow(product_id, sprint_id)`

Responsibilities:

* Schedule Sprint start/end timers from `SprintCreated.start_at/end_at`
* At `start_at`, publish `StartSprintRequested` (domain intent)
* After observing domain fact `SprintStarted`, publish `SprintPlanningWindowOpened`
* After observing `SprintBacklogCommitted`, publish `SprintBacklogReadyForExecution`
* Emit `DailyScrumWindowOpened` on each working day during the Sprint
* Near `end_at`, publish `SprintEndingSoon`
* At `end_at`, publish `EndSprintRequested` (domain intent) and emit `SprintTimeboxReachedEnd`
* After observing `SprintEnded`, emit `SprintReviewWindowOpened` and `SprintRetrospectiveWindowOpened` (ordering/timeboxing decisions are policy)

#### B. `SprintBacklogItemWorkflow(product_id, sprint_id, product_backlog_item_id)` (child workflow)

Start/stop logic:

* Created when `SprintBacklogCommitted` includes the item
* Emits `SprintBacklogItemReadyForWork` once Sprint is started
* Completes when the domain emits `ProductBacklogItemDoneRecorded` for that item in that sprint
* If Sprint ends first, emits a governance fact like `SprintBacklogItemNotDoneBySprintEnd` (optional)

This structure makes scaling and idempotency easier than trying to keep all item-level state in the SprintWorkflow.

---

### 7.3 Idempotency rules

Because bus delivery is typically **at-least-once**, the process must be idempotent:

* Store `last_seen_aggregate_version` per relevant aggregate.
* Ignore any domain fact where `aggregate_version <= last_seen`. 
* Track “already emitted” flags for process facts that must be exactly-once (e.g., `SprintBacklogReadyForExecution`, `SprintBacklogItemReadyForWork`).

---

## 8. Minimal end-to-end sequence

1. Domain emits `SprintCreated` (facts).
2. Process starts `SprintWorkflow` and schedules timers.
3. At `start_at`, process emits domain intent `StartSprintRequested`.
4. Domain emits `SprintStarted`.
5. Process emits `SprintPlanningWindowOpened`.
6. Sprint Planning outcome recorded by publishing `CommitSprintBacklogRequested`.
7. Domain emits `SprintBacklogCommitted` (contains Sprint Goal + selected PBIs).
8. Process:

   * emits `SprintBacklogReadyForExecution`
   * starts child workflows for each selected PBI, each emits `SprintBacklogItemReadyForWork`
9. As PBIs meet DoD, publish `RecordProductBacklogItemDoneRequested`.
10. Domain emits `ProductBacklogItemDoneRecorded` and `IncrementUpdated`.
11. At `end_at`, process emits `EndSprintRequested`.
12. Domain emits `SprintEnded`.
13. Process emits `SprintReviewWindowOpened` and `SprintRetrospectiveWindowOpened`.

---

## 9. What this contract deliberately avoids

* **No “Definition of Ready” as a first-class Scrum construct.** Teams can implement refinement policies, but Scrum Guide compliance centers on **Definition of Done**.
* **No “phase gates”** as Scrum concepts. If you want gates, keep them as **process facts** (governance) or platform extensions, not domain state.
* **No coupling to agent execution** in the Scrum contract. Agents (if any) are simply actors that can publish intents and consume facts (and may be considered “Developers” depending on policy).

---

## 10. Legacy docs and migration

This target-state contract is the **normative reference**; older/implementation-specific naming, neutralized models, or mixed responsibility boundaries should be considered subordinate to:

* Stage separation and async handoffs. 
* “Domain stores Scrum artifacts; process emits governance events; no direct mutation.” 

You can keep older docs for migration history, but apply the following rules when they conflict with this backlog or `508/509`:

### 10.1 Deprecate legacy naming

- Treat legacy topic names such as `scrum.intents.v1` / `scrum.facts.v1` as **historical only**; new work must use `scrum.domain.intents.v1`, `scrum.domain.facts.v1`, `scrum.process.facts.v1` as defined here and in `509-proto-naming-conventions.md`.  
- Where older docs or code use requirement/workitem-centric event names, plan to migrate them to the Scrum‑centric names over time or keep them behind an explicit translation layer.

### 10.2 Replace Requirement/WorkItem language with Scrum artifacts

- In older backlogs, names like `RequirementCompleted` or `ItemReadyForDev` should either:  
  - be **renamed** to use Scrum artifacts (Product Backlog Item, Sprint Backlog, Increment), or  
  - be clearly marked as **legacy mapping only**, not targets for new contracts.  
- `509-proto-naming-conventions.md` applies to new schemas: avoid requirement‑centric language when naming new intents/facts.

### 10.3 Keep DoR and phase gates out of the Scrum contract

- If an older backlog models “Definition of Ready” or explicit phase gates as domain state, treat that as **non-Scrum policy** to be moved either into:  
  - configuration / working agreements (outside this contract), or  
  - **process facts** (governance extensions) emitted by the Process primitive.  
- New domain contracts MUST NOT introduce DoR or phase gates as Scrum artifacts; they can appear only as platform policy or process extensions around the core Scrum model.

---

## 11. Implementation plan (bottom‑up)

This section sketches a **bottom‑up path** to implementing the contract above without re‑inventing semantics or coupling. It is intentionally layered so Transformation, Process, operators, and agents can progress in parallel once the contract is frozen.

### 11.1 Dependency ladder

1. **Layer 0 – Contract & codegen (blocks everything else)**  
   - Protos: `transformation_scrum_*` under `ameide_core_proto.transformation.scrum.v1` (see `508-scrum-protos.md`) plus a `scrum.process.facts.v1` package.  
   - Message classes and topics: this file defines the three classes (domain intents, domain facts, process facts), required envelope (`ScrumMessageMeta` equivalent), `ScrumAggregateRef`/`aggregate_version`, and topics: `scrum.domain.intents.v1`, `scrum.domain.facts.v1`, `scrum.process.facts.v1`.  
   - Codegen: SDKs for Go/TS/Python used by Transformation, Process workers, and agents/portal. No bespoke JSON at runtime.  
   _Implementation checklist (Layer 0):_  
   - [x] `transformation_scrum_*` proto files live under `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/` (see `transformation_scrum_common.proto`, `-artifacts.proto`, `-intents.proto`, `-facts.proto`, `-query.proto`).  
   - [x] `process_scrum_facts.proto` lives under `packages/ameide_core_proto/src/ameide_core_proto/process/scrum/v1/` and defines `ScrumProcessFact`.  
   - [x] `pnpm -F @ameide/core-proto build` generates and commits Go/TS/Python outputs into `packages/ameide_core_proto/gen/ts`, `packages/ameide_sdk_ts/src/_proto`, `packages/ameide_sdk_go/internal/proto` (+ the public re-export surface under `packages/ameide_sdk_go/proto`), and `packages/ameide_sdk_python/src/ameide_sdk/_proto`.
   - [x] Language SDKs expose Scrum types via their public proto surfaces (TS: `packages/ameide_core_proto/src/index.ts` and `packages/ameide_sdk_ts/src/proto/index.ts` → `@ameide/core-proto` / `@ameideio/ameide-sdk-ts/proto.js`; Go: `github.com/ameideio/ameide-sdk-go/proto/ameide_core_proto/transformation/scrum/v1`; Python: `ameide_sdk.proto.ameide_core_proto.transformation.scrum.v1` and `ameide_sdk.proto.ameide_core_proto.process.scrum.v1`).
   - [ ] Runtime primitives that consume this Scrum contract (Domain, Process, Agent, UISurface) call it **only via SDKs**, never by importing `@ameide/core-proto` or `packages/ameide_core_proto` directly in their runtime code (see `393-ameide-sdk-import-policy.md` and `514-primitive-sdk-isolation.md`).  

2. **Layer 1 – Transformation Scrum domain (source of truth)**  
   - Implements the aggregates and invariants in §2 using normal domain code + Postgres.  
   - Consumes `scrum.domain.intents.v1`, validates, persists, and emits `scrum.domain.facts.v1` with monotonic `aggregate_version` (via transactional outbox).  
   - Exposes a read‑only `ScrumQueryService` for UI/agents and debugging. No Temporal, no timers.
   _Implementation checklist (Layer 1):_  
   - [ ] Transformation DB schema extended with Scrum tables and outbox aligned to `ameide_core_proto.transformation.scrum.v1` (e.g. `db/flyway/sql/transformation/V3__scrum_domain.sql` defining `transformation.scrum_*` tables and `transformation.scrum_domain_outbox`).  
   - [ ] Intent handlers in the Transformation domain primitive (`primitives/domain/transformation`) persist aggregates and write `ScrumDomainFact` rows with monotonic `aggregate_version` using `ScrumAggregateRef` and `ScrumDomainFactSchema`.  
   - [ ] An outbox dispatcher (following `496-eda-principles.md` and the Graph outbox pattern in `services/graph/src/graph/events/outbox.ts`) publishes serialized `ScrumDomainFact` envelopes to `scrum.domain.facts.v1`.  
   - [ ] `ScrumQueryService` RPC handlers are implemented in the Transformation domain primitive (`primitives/domain/transformation`) and exposed via the platform gateway (see `transformation_scrum_query.proto` and `367-1-scrum-transformation.md` for routing expectations).  

3. **Layer 2 – Process primitive(s) (Temporal governor)**  
   - Ingress router subscribes to `scrum.domain.facts.v1` and signals workflows via `SignalWithStart` using deterministic workflow IDs.  
   - Workflows implement the structure in §7, keep only derived state, and dedupe by `aggregate_version` + “already emitted” flags for process facts.  
   - All writes back to the domain are published as intents on `scrum.domain.intents.v1`; all additional emissions are process facts on `scrum.process.facts.v1`.
   _Implementation checklist (Layer 2):_  
   - [ ] Ingress router process subscribes to `scrum.domain.facts.v1` and calls `SignalWithStart` on Temporal workflows using deterministic IDs (e.g. `product/{product_id}/sprint/{sprint_id}`), as described in §7 and `496-eda-principles.md`.  
   - [ ] `SprintWorkflow` and `SprintBacklogItemWorkflow` implemented per §7 inside a dedicated Process primitive (e.g. `primitives/process/scrum`), owning timers, signals, and child workflow orchestration but not domain persistence.  
   - [ ] Ingress router explicitly pins Temporal `WorkflowIDReusePolicy` for `SignalWithStart` (do not rely on defaults); add a test that asserts the selected policy for the primary workflows.  
   - [ ] Idempotency state (`last_seen_aggregate_version`, “already emitted” flags for each process fact) is persisted in workflow state and/or a dedicated process-side table to handle replays and `ContinueAsNew`.  
   - [ ] `ScrumProcessFact` messages (from `ameide_core_proto.process.scrum.v1`) are emitted on `scrum.process.facts.v1` using the same envelope conventions (meta + subject) and are published via a process outbox/dispatcher analogous to the domain outbox.  

4. **Layer 3 – Operators & GitOps**  
   - Process operator (499) fetches `ProcessDefinition` from the Definition Registry (Transformation), creates ConfigMaps, and wires Temporal workers (env vars, task queue, namespaces).  
   - Domain operator (498) and `502-domain-vertical-slice.md` provide the pattern for implementing the Scrum domain as a Domain primitive; operators deploy once per cluster, CRDs per environment.
   _Implementation checklist (Layer 3):_  
   - [ ] Process CRD (see `499-process-operator.md`) describes the Scrum governance workflows and references the Temporal task queue / namespace to use.  
   - [ ] Process operator implementation fetches `ProcessDefinition` from the Definition Registry in Transformation (per `367-1-scrum-transformation.md`), stores it in a ConfigMap/Secret, and mounts it into worker pods.  
   - [ ] Temporal worker Deployments/StatefulSets are wired via env/ConfigMaps for namespace, task queue, and broker settings, and are registered in the GitOps manifests under `gitops/ameide-gitops`.  
   - [ ] Domain/operator CRDs and reconcilers deploy the Transformation domain (including the Scrum profile) following the domain primitive pattern in `502-domain-vertical-slice.md` and `495-ameide-operators.md`.  

5. **Layer 4 – Agents, UI, and automation**  
   - AmeidePO/AmeideSA/AmeideCoder stack (`505-agent-developer-v2*.md`) sits *on top* of this seam: PO consumes process facts and issues domain intents; SA delegates to the executor via the canonical work handover seam (event-driven; optional A2A transport binding); the executor may use the Ameide CLI as an orchestrator (wrapping `buf generate`, tests, and repo wiring), but the canonical drift gate remains CI regen-diff + compile/tests.
   - Portal surfaces (backlog views, Sprint views, run dashboards in 367‑5) read from `ScrumQueryService` and process facts; they do not define new Scrum state.
   _Implementation checklist (Layer 4):_  
   - [ ] AmeidePO/AmeideSA/AmeideCoder flows (see `505-agent-developer-v2*.md` and agent DAGs) updated to use `ScrumDomainIntent` / `ScrumDomainFact` / `ScrumProcessFact` only (no legacy `Requirement*` lifecycle), with work handover messages carrying Scrum IDs (optional A2A binding must map to the same work IDs).
   - [ ] Portal Scrum views (backlog, Sprint, Increment dashboards in `services/www_ameide_platform`) backed by `ScrumQueryService` and `scrum.process.facts.v1` projections instead of ad‑hoc tables or legacy transformation endpoints.  
   - [ ] CLI/agent tooling wired so Coder uses the Ameide CLI against the Scrum seam (publishing intents, consuming facts) rather than bespoke RPCs; CLI remains an orchestrator (no bespoke codegen), and guardrails are enforced via CI regen-diff + tests (see `520-primitives-stack-v2.md`, `backlog/521c-internal-generation-improvements.md`, `backlog/521d-external-generation-improvements.md`).  
   - [ ] Any non‑Scrum extensions (DoR, acceptance workflows, board columns) clearly labelled in profile/backlog docs and implemented as policy/process extensions around the core Scrum contract, not as new domain state in Transformation.  

_Progress notes for this ladder (non‑normative, snapshot only):_

- Layer 0 – Contract & codegen: implemented in `packages/ameide_core_proto` (protos added under `src/ameide_core_proto/transformation/scrum/v1/` and `src/ameide_core_proto/process/scrum/v1/`, with Buf wired to generate Go/TS/Python SDKs via the `buf.gen*.yaml` templates and `ameide_sdk_*` packages).

### 11.2 Minimum working slice (A + B)

To avoid over‑design, implement a **small, complete loop** first.

**Slice A – Sprint timebox + Sprint Backlog committed + “work available”**

- **Domain side (Transformation)** supports at least:
  - Intents: `CreateSprintRequested`, `StartSprintRequested`, `EndSprintRequested`, `CommitSprintBacklogRequested`.  
  - Facts: `SprintCreated`, `SprintStarted`, `SprintEnded`, `SprintBacklogCommitted` (including Sprint Goal + selected PBIs).  
- **Process side (Temporal)** implements:
  - `SprintWorkflow(product_id, sprint_id)`:
    - schedule start/end timers from `SprintCreated.start_at`/`end_at`;  
    - at `start_at`, publish `StartSprintRequested`;  
    - after `SprintStarted`, emit `SprintPlanningWindowOpened`;  
    - after `SprintBacklogCommitted`, emit `SprintBacklogReadyForExecution`;  
    - near `end_at`, emit `SprintEndingSoon`;  
    - at `end_at`, publish `EndSprintRequested` and emit `SprintTimeboxReachedEnd`;  
    - after `SprintEnded`, emit `SprintReviewWindowOpened` and `SprintRetrospectiveWindowOpened`.  
  - `SprintBacklogItemWorkflow(product_id, sprint_id, product_backlog_item_id)`:
    - start when `SprintBacklogCommitted` includes the PBI;  
    - emit `SprintBacklogItemReadyForWork` once `SprintStarted` observed;  
    - optionally emit a governance fact such as `SprintBacklogItemNotDoneBySprintEnd` if the Sprint ends first.

**Slice B – Done + Increment update**

- **Domain side** additionally supports:
  - Intent: `RecordProductBacklogItemDoneRequested` (and, if desired, `RecordIncrementRequested`);  
  - Facts: `ProductBacklogItemDoneRecorded` and `IncrementUpdated` (or `IncrementCreated` + `IncrementUpdated`).  
- **Process side (child workflow)**:
  - `SprintBacklogItemWorkflow` completes when `ProductBacklogItemDoneRecorded` is observed for that PBI in that Sprint.  

When Slice A + B work end‑to‑end (intents in → facts out → process facts emitted → child workflow completes), you have a complete Scrum‑aligned “work available → Done → Increment updated” loop with a clean seam.

### 11.3 Recommended implementation order

1. **Step A – Freeze the contracts (Proto‑first, Layer 0)**  
   - Land the Scrum Transformation proto files (`transformation_scrum_*`) under `ameide_core_proto.transformation.scrum.v1` as in `508-scrum-protos.md`.  
   - Add the *process* side proto: `process_scrum_facts.proto` under `ameide_core_proto.process.scrum.v1` so that process facts are first‑class and code‑generated.  
   - Lock the topic names to exactly `scrum.domain.intents.v1`, `scrum.domain.facts.v1`, `scrum.process.facts.v1` and generate Go/TS/Python SDKs (no bespoke JSON at runtime).  

2. **Step B – Implement the Transformation Scrum subdomain (Layer 1, Slice A+B)**  
   - Use the Ameide CLI as an **orchestrator** to create/maintain the implementation-owned Transformation Domain primitive skeleton at `primitives/domain/transformation` (and optional GitOps wiring), following `510-domain-primitive-scaffolding.md` and `514-primitive-sdk-isolation.md` rather than hand‑rolling a bespoke service skeleton.  
   - Run `buf generate` to refresh SDKs and generated-only glue/tests for the Scrum protos; treat regen-diff + compile/tests as the canonical drift gate (the CLI may wrap these locally, but CI is authoritative).  
   - Treat `primitives/domain/transformation` as the **Scrum system-of-record endpoint** for `ScrumQueryService`, while the legacy `services/transformation` Node service either acts as a façade or is retired over time.  
   - Add Postgres schema and handlers in the Transformation domain that:  
     - consume domain intents (starting with Slice A+B) from `scrum.domain.intents.v1`;  
     - enforce the aggregates and invariants in §2 (Scrum artifacts + commitments only);  
     - emit corresponding domain facts on `scrum.domain.facts.v1` with monotonic `aggregate_version` and the shared envelope fields;  
     - keep Increment membership restricted to work that is `DONE` and meets Definition of Done, with “evidence” stored as a platform extension;  
     - avoid DoR/phase‑gate concepts in the domain contract, even if internal policies exist elsewhere;  
     - expose a read‑only `ScrumQueryService` for PBIs, Sprints, Sprint Backlogs, and Increments.

3. **Step C – Implement the Process primitive as “governance over facts” (Layer 2, Slice A)**  
   - Use the Ameide CLI as an **orchestrator** to create/maintain a Process primitive module (worker + ingress + GitOps wiring), keeping process code aligned with the shared EDA and operator patterns.  
   - Run `buf generate` to refresh any generated-only workflow/ingress glue and structural tests/harness for the process seam; implement missing behavior in implementation-owned files until compilation and tests pass.  
   - Build a non‑deterministic ingress router that subscribes to `scrum.domain.facts.v1` and routes into Temporal via `SignalWithStart` using deterministic workflow IDs.  
   - Pin `WorkflowIDReusePolicy` explicitly in the ingress `SignalWithStart` call and test the chosen policy (do not rely on defaults).  
   - Implement the workflow hierarchy from §7 (`SprintWorkflow`, `SprintBacklogItemWorkflow` as child), keeping only derived state needed for timers and governance decisions.  
   - Apply the idempotency rules from §7.3: track `last_seen_aggregate_version`, ignore facts where `aggregate_version <= last_seen`, and track “already emitted” flags for each process fact.  
   - Emit the process facts from §6 for event scheduling windows, Sprint timebox governance, and the explicit platform extensions (`SprintBacklogReadyForExecution`, `SprintBacklogItemReadyForWork`).  

4. **Step D – Wire deployment via the Process operator (Layer 3)**  
   - Complete Phase 2/3 of `499-process-operator.md` so the operator: fetches the Process CR, fetches `ProcessDefinition` from Transformation, stores it in a ConfigMap, wires Temporal workers (env vars, task queue, namespaces), and updates status conditions.  
   - Keep this as **control plane** responsibility only: at runtime, Process ↔ Transformation interaction is exclusively via intents/facts on the bus.

5. **Step E – Only then integrate agents and UI (Layer 4)**  
   - Make AmeidePO/AmeideSA/AmeideCoder consume process facts and publish domain intents only, as described in `505-agent-developer-v2*.md`; the Process layer itself stays “decisionless” and purely governance‑oriented.  
   - Back Ameide‑on‑Ameide Scrum views in `367-5-ameide-journey.md` with `ScrumQueryService` + process facts, without introducing new Scrum state in the UI.  
   - Use the minimal path from §8/§11.2 (`SprintCreated → StartSprintRequested → … → SprintEnded`) as the first E2E slice for agents, with “developers did work” modelled as `RecordProductBacklogItemDoneRequested`.

This sequencing stays aligned with the Scrum Guide, Temporal’s deterministic‑workflow guidance, and the proto/EDA naming conventions, while getting you to a working, testable vertical slice before layering on agents and UI.

---

## 12. “Are we aligned?” checklist

Use this as a quick litmus test when designing new flows or reviewing implementations against this contract and its companion docs (`508-scrum-protos.md`, `509-proto-naming-conventions.md`).

### 12.1 Scrum domain contract

- [ ] Domain stores **Product Backlog / Product Backlog Item / Sprint / Sprint Backlog / Increment** (+ commitments Product Goal, Sprint Goal, Definition of Done).  
- [ ] Domain contracts contain **no Definition of Ready, no phases/gates, no “requirements”** as first‑class Scrum concepts.  
- [ ] “Evidence” (builds, tests, PRs, etc.) is modelled only as a **platform extension** attached to Done items / Increments, not as a Scrum artifact.

### 12.2 EDA seam (topics + contracts)

- [ ] Topics are exactly `scrum.domain.intents.v1`, `scrum.domain.facts.v1`, `scrum.process.facts.v1`.  
- [ ] Domain facts use monotonic `aggregate_version` for idempotency and dedupe.  
- [ ] Bus contracts use aggregator messages with `oneof` payloads (`ScrumDomainIntent`, `ScrumDomainFact`, `ScrumProcessFact`) so topics stay stable while event types evolve.

### 12.3 Temporal process

- [ ] Non-deterministic bus consumption and network I/O live **outside** workflows; workflows are **signal-driven and deterministic**.  
- [ ] Nested workflows are used where natural (e.g., `SprintWorkflow` + `SprintBacklogItemWorkflow`), rather than a single giant workflow.  
- [ ] There is an **explicit dedupe plan**, especially if `ContinueAsNew` is introduced: workflows track `last_seen_aggregate_version` and “already emitted” flags across runs.
