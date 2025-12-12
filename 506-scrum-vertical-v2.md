# Scrum governance contract target state

This is a clean, **Scrum-Guide-aligned** contract for structuring Scrum as:

* **Scrum Domain (Transformation)** = *system of record* for Scrum artifacts + commitments (data and rules), **reactive only** (no timers/orchestration).
* **Scrum Process (Temporal)** = *timebox governor* (timers, reminders, orchestration), **event-driven only** (no direct domain writes/calls).
* **Domain ↔ Process integrate only through messages on the event bus** (domain “facts”, domain “intents”, and process “facts”).  

The Scrum wording used below follows the **official Scrum Guide**: Scrum Team accountabilities, Scrum events, Scrum artifacts, and their commitments.

---

## Grounding & cross-references

- **Role in architecture:** Canonical Scrum runtime seam between Transformation (Scrum domain) and Process primitives; defines envelopes, topics, and Temporal workflow structure.  
- **Domain side:** Builds on `300-400/367-1-scrum-transformation.md` and the `transformation-scrum-*` protos in `508-scrum-protos.md`; Transformation persists Scrum artifacts and emits the domain facts referenced here.  
- **Process side:** Consumed by Process primitives described in `499-process-operator.md` and positioned in the primitive stack by `477-primitive-stack.md`; Temporal workflows must obey the intent/fact separation and idempotency rules in this backlog and the EDA rules in `496-eda-principles.md`.  
- **Agents & tooling:** `505-agent-developer-v2.md` and `505-agent-developer-v2-implementation.md` describe how AmeidePO/AmeideSA/AmeideCoder react to process/domain facts from this seam; `495-ameide-operators.md` and `502-domain-vertical-slice.md` provide shared operator/condition vocabulary; `507-scrum-agent-map.md` locates this contract at Stage 2 in the Scrum stack.

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

## 10. Supersedence note

This target-state contract is meant to be the clean reference point; older/implementation-specific naming, neutralized models, or mixed responsibility boundaries should be considered subordinate to:

* Stage separation and async handoffs 
* “Domain stores Scrum artifacts; process emits governance events; no direct mutation” 

(You can still keep older docs for migration history, but treat this contract as the normative spec.)
