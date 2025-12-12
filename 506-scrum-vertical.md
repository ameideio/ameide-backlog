> **Contract alignment**
> - **Canonical contract:** `transformation-scrum-*` protos and event contracts defined in `506-scrum-vertical-v2.md` and `508-scrum-protos.md` (Scrum nouns only; no neutral `WorkItem`/`Timebox`/`Goal` wire model).  
> - **Canonical integration:** intents + facts via bus; Process (Temporal) owns timeboxes and process facts; Transformation (Scrum domain) is reactive and never called synchronously at runtime.  
> - **Non-Scrum extensions / legacy design:** the Requirement-centric aggregates, DoR gates, and `ItemReadyForDev`/`PhaseGateReached` process facts below are historical and retained only as migration background. All new work must follow `506-scrum-vertical-v2.md` and `508-scrum-protos.md`.

Here’s a clean way to structure **Scrum** with the constraints you set:

* **Transformation (domain) primitive** is the **system of record for Scrum data** (Product Backlog, Product Backlog Items, Sprints, Sprint Backlogs, Increments, commitments). It is *reactive only* (no timers, no orchestration).
* **Process primitive (Temporal)** is the **governor/orchestrator** (timeboxes, reminders, governance cues). It *triggers* by emitting messages, but it **never writes domain state directly**.
* **Domain ↔ Process interact only via messages on the bus**: **Scrum domain intents** on `scrum.domain.intents.v1`, **Scrum domain facts** on `scrum.domain.facts.v1`, and **process facts** on `scrum.process.facts.v1`. No synchronous calls at runtime.

To make this unambiguous and implementable, you need 3 things:

1. a **data model** (Scrum aggregates + invariants) owned by Transformation
2. an **event model** split into *Scrum domain intents* vs *Scrum domain facts* vs *process facts*
3. a **Temporal workflow structure** + an ingress/router that maps bus events to workflow signals

Below is a concrete blueprint that works well in practice.

**Authority & supersession**

- This backlog is now a **historical view of the original Scrum runtime seam**: it documents the Requirement-centric envelopes, topics, and Temporal workflow structure that have since been superseded by `506-scrum-vertical-v2.md` and `508-scrum-protos.md`, which are authoritative for any new implementation.  
- `300-400/367-1-scrum-transformation.md` is authoritative for the underlying Scrum **data model**; this file owns how that model is exposed on the bus.  
- `505-agent-developer-v2.md` is authoritative for **PO/SA/Coder behavior** and A2A; when it references Scrum events, it must match the contracts here.  
- If any backlog (including this one, `367-0-feedback-intake.md`, or `505-agent-developer.md`) contradicts the canonical contracts in `506-scrum-vertical-v2.md`, `508-scrum-protos.md`, or `505-agent-developer-v2.md`, **those newer files win**.

> **Cross references:**  
> - Transformation data/governance decisions live in [367-1-scrum-transformation.md](300-400/367-1-scrum-transformation.md).  
> - The multi-agent execution that consumes these events is documented in [505-agent-developer-v2.md](505-agent-developer-v2.md) and its implementation plan.  
> - A context map across all layers sits in [507-scrum-agent-map.md](507-scrum-agent-map.md).

## Grounding & cross-references

- **Status:** Historical Scrum seam capturing the original Requirement-centric aggregates and process facts; retained for migration and reasoning about legacy events, but superseded by `506-scrum-vertical-v2.md` and `508-scrum-protos.md` for all new runtime work.  
- **Architecture grounding:** Shares the same Domain/Process separation and event-only integration model described in `470-ameide-vision.md`, `472-ameide-information-application.md`, `475-ameide-domains.md`, and `496-eda-principles.md`, but uses pre-Scrum-cleanup terminology.  
- **Legacy dependents:** Earlier implementation notes in `505-agent-developer.md` and any stages that refer to `RequirementCreated/Completed/Failed`, `ItemReadyForDev`, or `PhaseGateReached` should be read through the mapping table in §1.1.1 of this file and migrated to the canonical Scrum nouns in 506‑v2/508.  
- **Navigation:** `507-scrum-agent-map.md` positions this backlog as a historical reference for Stage 1/2; `505-agent-developer-v2.md` and `505-agent-developer-v2-implementation.md` show how the new architecture uses the canonical contracts instead.

---

# 1) Data model (Transformation / Scrum domain)

Model Scrum as a small set of **event-sourced aggregates**. Keep it minimal and “exact”, but not overfit to ceremonies.

## 1.1 Core aggregates (historical view – superseded by 506-scrum-vertical-v2.md and 508-scrum-protos.md)

> **Supersedence note:** The aggregate and event names below use an earlier “Requirement”-centric framing. For any new implementation, prefer the canonical Scrum aggregates and messages in `506-scrum-vertical-v2.md` and `508-scrum-protos.md` (Product Backlog, Product Backlog Item, Sprint, Sprint Backlog, Increment, Product Goal, Sprint Goal, Definition of Done). The content in this section is retained only as migration background.

### 1.1.1 Legacy → canonical Scrum mapping

When reading the legacy names in this file, translate them to the canonical Scrum contract as follows:

- Legacy `BacklogItem` / `Requirement` → **Product Backlog Item** (`ProductBacklogItem` in 508).  
- Legacy `RequirementCreated` / `RequirementReady` / `RequirementCommittedToSprint` / `RequirementDevCompleted` / `RequirementCompleted` / `RequirementFailed` → **Scrum domain facts** such as `ProductBacklogItemCreated` / `ProductBacklogItemRefined` / `SprintBacklogCommitted` / `ProductBacklogItemDoneRecorded` / `IncrementUpdated` (see 508 for exact shapes).  
- Legacy process facts `ItemReadyForDev` / `PhaseGateReached` → **Scrum process facts** such as `SprintBacklogReadyForExecution`, `SprintBacklogItemReadyForWork`, and timebox/threshold signals defined in 506‑v2 §6.  
- Legacy domain intents `CreateRequirementRequested` / `CompleteRequirementRequested` / `FailRequirementRequested` → **Scrum domain intents** such as `CreateProductBacklogItemRequested`, `RefineProductBacklogItemRequested`, `RecordProductBacklogItemDoneRequested`, and `RecordIncrementRequested`.

These mappings are informative only; all new schema and runtime work must use the canonical names and protos.

### A) `Sprint` (timebox)

**Purpose:** captures the Scrum timebox as a first-class domain object.

**Fields**

* `tenant_id`
* `sprint_id`
* `team_id` (optional if single-team)
* `name`
* `goal` (optional but useful)
* `starts_at`, `ends_at` (timebox)
* `status`: `PLANNED | ACTIVE | CLOSED | CANCELED`
* `capacity` / `velocity_target` (optional)
* `version` (event-sourcing aggregate version)

**Invariants**

* `starts_at < ends_at`
* can only move `PLANNED -> ACTIVE -> CLOSED` (or `CANCELED`)
* cannot commit items after `CLOSED` (or allow with explicit “carryover” semantics)

---

### B) `BacklogItem` / `Requirement`

**Purpose:** the backlog item that flows through “ready → committed → done”.

**Fields**

* `tenant_id`
* `requirement_id`
* `title`, `description`
* `acceptance_criteria[]`
* `priority` (or `ordering_key`)
* `dor_state`: `NOT_READY | READY` (+ optional checklist)
* `status` (recommended minimal):

  * `DRAFT`
  * `READY` (DoR satisfied)
  * `COMMITTED` (in a sprint)
  * `IN_DEV`
  * `IN_REVIEW`
  * `DONE`
  * `FAILED`
* `committed_sprint_id?`
* `evidence_set?` (see below)
* `version`

**Invariants**

* `DONE` requires minimum evidence (DoD gate)
* only one `committed_sprint_id` at a time (or explicit “carryover” transition)

---

### C) `EvidenceSet` (embedded or separate aggregate)

**Purpose:** makes completion auditable and machine-verifiable.

**Minimum fields**

* `a2a_task_id` (or list if multiple)
* `pr_url?`
* `commit_sha?`
* `build_log_ref?`
* `test_report_ref?`
* `artifact_refs[]` (optional)
* `risk_summary` (short string, required for DONE)
* `produced_at`

You can embed it inside Requirement events, or model it as its own aggregate with references. For V2, embedding is usually simpler.

---

## 1.2 Optional (only if needed)

* `SprintCommitment` as its own object if you need richer semantics (scope changes, commitment history).
* `CeremonyRecord` (planning/review/retro). Personally: **don’t** model ceremonies as aggregates unless you need audit for them. Use **process events** instead.

---

# 2) Event model (the key to “domain/process only via events”)

You need to separate **two kinds of messages**:

* **Intent messages** (“please do X”) — consumed by the domain (or process) and validated
* **Fact events** (“X happened”) — emitted by the domain (or process) as a result

Even though both travel on the bus, don’t call both “events” in code. You’ll avoid endless confusion.

## 2.1 Shared envelope fields (required everywhere)

Make these required on *every* message (intent or fact):

* `message_id` (unique)
* `tenant_id`
* `correlation_id` (end-to-end trace)
* `causation_id` (the triggering message_id)
* `occurred_at`
* `producer` (service/workload identity)
* **routing keys** (at minimum):

  * `sprint_id` for sprint-scoped messages
  * `requirement_id` for item-scoped messages
* **aggregate_version** when the producer is a domain aggregate (facts)

> **Critical:** include `aggregate_version` on domain facts. It makes Temporal signal handling idempotent and deterministic.

---

## 2.2 Topics/streams (recommended – superseded)

> **Supersedence note:** Topic names and event families here reflect the earlier v1 contracts. New work should follow the `scrum.domain.intents.v1` / `scrum.domain.facts.v1` / `scrum.process.facts.v1` naming and message structures defined in `506-scrum-vertical-v2.md` and `508-scrum-protos.md`. The outline below is kept only to explain the evolution of the seam.

Use 3 logical channels (can be Kafka topics, NATS subjects, etc.):

1. `scrum.domain.intents.v1`

   * **Process → Domain** (and PO/SA → Domain)
2. `scrum.domain.facts.v1`

   * **Domain → everyone**
3. `scrum.process.facts.v1`

   * **Process → Agents/Observers** (gates, dispatch, timers, SLAs)

This preserves your separation:

* domain doesn’t “trigger”; it emits facts when asked
* process “triggers” by emitting intents and process facts

---

## 2.3 Domain intents (consumed by Transformation)

These are the “commands via bus”.

Minimal set:

### Sprint-related intents

* `CreateSprintRequested`
* `StartSprintRequested`
* `CloseSprintRequested`
* `CancelSprintRequested` (optional)

### Requirement-related intents (historical; see 506-scrum-vertical-v2.md / 508-scrum-protos.md for canonical Scrum intents)

* `CreateRequirementRequested` (if not created elsewhere)
* `MarkRequirementReadyRequested` (DoR)
* `CommitRequirementToSprintRequested`
* `RecordDevStartedRequested` (usually from SA/Coder boundary)
* `RecordDevCompletedRequested` (with evidence refs)
* `CompleteRequirementRequested` (usually from PO)
* `FailRequirementRequested` (usually from PO)

> In the canonical Scrum contract these map to `CreateProductBacklogItemRequested`, refinement/order intents, and `RecordProductBacklogItemDoneRequested` / `RecordIncrementRequested`. The names above are retained only to describe the earlier design.

---

## 2.4 Domain facts (emitted by Transformation)

These represent the authoritative Scrum state changes.

### Sprint facts

* `SprintCreated`
* `SprintStarted`
* `SprintClosed`
* `SprintCanceled`

### Requirement facts

* `RequirementCreated`
* `RequirementReady`
* `RequirementCommittedToSprint`
* `RequirementDevStarted`
* `RequirementDevCompleted` (includes evidence refs, a2a_task_id, pr_url if available)
* `RequirementCompleted` (includes final EvidenceSet)
* `RequirementFailed` (includes reason + any evidence)

Each must include:

* `aggregate_id` (sprint_id or requirement_id)
* `aggregate_version`

---

## 2.5 Process facts (emitted by Temporal process)

These are governance/orchestration “signals to act”, not domain state.

Recommended minimal set:

* `ItemReadyForDev`
  Emitted when process observes gating conditions (e.g., sprint active + requirement committed + DoR ready).

  * includes: `requirement_id`, `sprint_id`, `dev_brief_ref` or inline summary
* `PhaseGateReached`
  Emitted when process determines a gate is satisfied and the next actor should act.

  * `gate_name` examples:

    * `dev_dispatch`
    * `dev_complete`
    * `po_acceptance_due`
    * `sprint_close_due`
  * include: `supporting_fact_ids[]` (which domain facts caused the gate)
* `SLAWarning` / `SLAExceeded` (optional but very helpful)
* `SprintEndingSoon` (optional)

> This is the trick: agents don’t need to interpret raw domain state; they can react to **process facts** which encode “the governor says go”.

---

# 3) Process primitive (Temporal): structure + implementation pattern

Temporal workflows can’t “subscribe to Kafka” directly in a deterministic way. So you implement a **Process Event Ingress** that:

* consumes bus messages (domain facts + relevant process facts)
* routes them into Temporal via `SignalWithStart`
* uses workflow IDs derived from domain keys

## 3.1 Components in the process primitive

### A) `process-ingress` (non-deterministic service)

Responsibilities:

* subscribe to `scrum.facts.v1`
* optionally subscribe to `process.facts.v1` (if you want feedback loops)
* map messages → workflow signal calls
* guarantee *at-least-once* delivery to Temporal (workflows dedupe)

Routing rules:

* sprint-scoped facts → `SprintWorkflow(tenant,sprint_id)`
* requirement-scoped facts → `RequirementWorkflow(tenant,requirement_id)`
* if an event includes both, route to both (carefully)

### A.1 Deployment wiring (Process operator)

- The **Process operator** (backlog `499-process-operator.md`) runs the control plane: it fetches `ProcessDefinition` data from Transformation via gRPC/Connect and injects it into worker Deployments as ConfigMaps/env vars.  
- Temporal workflows themselves **must not** call Transformation directly; they only emit **domain intents** on `scrum.intents.v1` and consume **domain facts** from `scrum.facts.v1`. All runtime interaction with the Scrum data model occurs through the bus contracts defined in this file.

### B) Temporal workflows (deterministic)

You generally want **two levels**:

#### 1) `SprintWorkflow(tenant_id, sprint_id)`

Owns timebox governance:

* schedules `starts_at` / `ends_at` timers
* emits `StartSprintRequested` / `CloseSprintRequested` intents at the right time
* tracks sprint active/closed based on **domain facts**
* can emit process facts like `SprintEndingSoon`

#### 2) `RequirementWorkflow(tenant_id, requirement_id)`

Owns requirement governance gates:

* waits for required domain facts:

  * `RequirementReady`
  * `RequirementCommittedToSprint`
  * `SprintStarted` (for that sprint)
* once gating satisfied → emits `ItemReadyForDev` process fact
* waits for `RequirementDevCompleted` → emits `PhaseGateReached(dev_complete)` process fact (for PO)
* waits for `RequirementCompleted` / `RequirementFailed` → completes workflow
* handles deadlines (e.g., sprint ends first) → emits escalation events

> Keep the workflow state small and derived from facts; don’t mirror the whole domain.

---

## 3.2 Idempotency strategy (this matters)

Because bus delivery is at-least-once, workflows must dedupe.

**Best practice:**

* store `last_seen_requirement_version`
* store `last_seen_sprint_version`
* ignore any fact with `aggregate_version <= last_seen`

Additionally:

* track whether you already emitted `ItemReadyForDev` for a given `(requirement_id, sprint_id)` to avoid duplicate dispatch.

This is why `aggregate_version` on domain facts is non-negotiable.

---

# 4) Concrete gating logic: “What does the process actually do?”

Scrum-wise, the process primitive is not “doing Scrum ceremonies”; it is implementing **governance gates** over Scrum state.

Here are the **minimal gates** for your V2 environment:

## Gate G0: Sprint activation gate

**Triggering condition**

* `SprintCreated` exists AND
* `now >= sprint.starts_at` AND
* sprint not started yet

**Action**

* emit intent: `StartSprintRequested(sprint_id)`
* wait for fact: `SprintStarted`

---

## Gate G1: Item ready-for-dev gate

**Triggering condition**

* `SprintStarted(sprint_id)` observed AND
* `RequirementCommittedToSprint(requirement_id, sprint_id)` observed AND
* `RequirementReady(requirement_id)` observed AND
* requirement not yet dispatched

**Action**

* emit process fact: `ItemReadyForDev(requirement_id, sprint_id, dev_brief_ref, …)` (historical name; superseded by `SprintBacklogItemReadyForWork` in 506-scrum-vertical-v2.md)

---

## Gate G2: Dev complete gate

**Triggering condition**

* `RequirementDevCompleted(requirement_id)` observed

**Action**

* emit process fact: `PhaseGateReached(gate_name="dev_complete", requirement_id, supporting_fact_ids=[…])` (historical; in the canonical contract PO reacts to process facts such as `SprintBacklogItemReadyForWork` and emits Scrum domain intents like `RecordProductBacklogItemDoneRequested` / `RecordIncrementRequested`.)

---

## Gate G3: Sprint closure gate

**Triggering condition**

* `now >= sprint.ends_at` AND sprint is active

**Action**

* emit intent: `CloseSprintRequested(sprint_id)`
* wait for fact: `SprintClosed`

Optionally:

* if there are incomplete committed requirements, emit `PhaseGateReached("carryover_decision_due")`

---

# 5) End-to-end message flow example

### Scenario: requirement gets done inside a sprint

1. Domain fact: `SprintCreated(sprint_id, starts_at, ends_at)`
2. Process ingress signals `SprintWorkflow(sprint_id)`
3. Timer fires → Process intent: `StartSprintRequested(sprint_id)`
4. Domain fact: `SprintStarted(sprint_id)`
5. Domain fact: `RequirementReady(requirement_id)`
6. Domain fact: `RequirementCommittedToSprint(requirement_id, sprint_id)`
7. Process sees all → Process fact: `ItemReadyForDev(requirement_id, sprint_id, dev_brief_ref)`
8. SA agent reacts → calls Coder A2A; then emits domain intent: `RecordDevCompletedRequested(requirement_id, evidence_refs, pr_url, a2a_task_id)`
9. Domain fact: `RequirementDevCompleted(requirement_id, …)`
10. Process emits: `PhaseGateReached(dev_complete)`
11. PO reacts → domain intent: `CompleteRequirementRequested(requirement_id, final_evidence_set)` (historical alias for `RecordProductBacklogItemDoneRequested` / `RecordIncrementRequested`)
12. Domain fact: `RequirementCompleted(requirement_id, evidence_set)` (historical alias for `ProductBacklogItemDoneRecorded` / `IncrementUpdated`)
13. Process ends `RequirementWorkflow`

No direct calls. No domain timers. Pure event interaction.

---

# 6) What to implement first (P0 “Process primitive” execution plan)

If your focus is “how exactly the process primitive should be implemented”, here’s the minimal slicing that avoids rework:

## Step 1 — Lock the contract

* finalize:

  * `scrum.facts.v1` (domain facts) with `aggregate_version`
  * `scrum.intents.v1` (domain intents)
  * `process.facts.v1` (ItemReadyForDev, PhaseGateReached)
* enforce required envelope fields (tenant/correlation/causation)

## Step 2 — Build `process-ingress`

* subscribe to `scrum.facts.v1`
* implement routing to Temporal:

  * `SignalWithStart` SprintWorkflow by sprint_id
  * `SignalWithStart` RequirementWorkflow by requirement_id
* add metrics: lag, signal failures, redelivery counts

## Step 3 — Implement SprintWorkflow timers + intents

* on `SprintCreated` store starts/ends
* set timer for start → emit `StartSprintRequested`
* set timer for end → emit `CloseSprintRequested`

## Step 4 — Implement RequirementWorkflow gates + process facts

* collect the three prerequisite facts (sprint started + committed + ready)
* emit exactly once `ItemReadyForDev`
* emit `PhaseGateReached(dev_complete)` after `RequirementDevCompleted`

## Step 5 — Tests (the real “definition of done”)

* deterministic Temporal tests:

  * feed facts in different orders (out-of-order delivery) → still works
  * duplicate facts → no duplicate process facts
* integration harness:

  * fake bus + Temporal test env + assertion on published messages

---

# 7) One design decision you should make explicitly

You need to pick **who emits `SprintStarted`**:

### Option A (recommended): Domain emits Scrum facts

* Process emits `StartSprintRequested` intent
* Domain emits `SprintStarted` fact
  ✅ “Scrum state is exact and authoritative in Transformation”
  ✅ Process stays purely orchestration
  ⚠️ Slightly more plumbing (intent + confirmation fact)

### Option B: Process emits SprintStarted as a process fact and domain records it

* Process emits `SprintStarted` (process)
* Domain consumes and updates sprint state
  ✅ Simpler early
  ⚠️ Domain is no longer the sole origin of Scrum facts (it’s recording someone else’s “facts”)

Given your statement “Transformation holds scrum related data (to be modelled exactly)”, **Option A** is the cleaner seam. For the current Scrum stack (505/507/367‑1/499), **Option A is the chosen design**; Option B is documented here only as a rejected alternative and must not be implemented in new code.

---

If you want, I can take your existing event names (`SprintStarted`, `ItemReadyForDev`, `PhaseGateReached`) and propose an exact **proto-level contract** (fields + required/optional + versions) and a **Temporal workflow pseudocode** for both workflows (including signal handlers and dedupe logic).
