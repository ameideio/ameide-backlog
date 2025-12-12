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

### A) Historical details removed

Earlier revisions of this file defined Requirement-centric aggregates (`Sprint`, `BacklogItem/Requirement`, `EvidenceSet`), status pipelines (`DRAFT → READY → … → DONE`), DoR gates, and concrete intent/fact names such as `CreateRequirementRequested`, `RequirementCompleted`, `ItemReadyForDev`, and `PhaseGateReached`. Those details are now fully superseded by the Scrum-only contracts in `506-scrum-vertical-v2.md` and `508-scrum-protos.md` and have been removed to avoid accidental reimplementation. Use this file only for the mapping above when reasoning about legacy events; implement all new work directly against the canonical Scrum aggregates, intents, and facts.
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

The earlier versions of this file also contained detailed routing rules, Temporal workflow pseudocode, gate definitions (G0–G3), and execution plans based on the legacy `Requirement*`, `ItemReadyForDev`, and `PhaseGateReached` events. Those sections have been removed in favour of the canonical governance model in `506-scrum-vertical-v2.md`. For concrete workflow structure, event routing, gating logic, and implementation guidance, refer directly to `506-scrum-vertical-v2.md` and the Process operator backlog (`499-process-operator.md`).
