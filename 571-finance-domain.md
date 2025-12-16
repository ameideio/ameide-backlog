## HR Domain primitive

### ArchiMate element and intent

* **ArchiMate type:** **Application Component**
* **Primitive kind:** **Domain**
* **Bounded context:** `hr` (single-writer)
* **Intent:** provide **transactional domain services** (write APIs) and publish **domain facts/events** as the authoritative HR system-of-record. 
* **Normative behavior:** as a Domain primitive, it is the **single-writer** for its aggregates; it accepts change via **commands (RPC)** and/or **domain intents (bus)**, emits **domain facts via a transactional outbox**, and **does not expose CRUD** (verbs + invariants instead). 

---

### Naming and proto boundary

* **Proto package root:** `package ameide_core_proto.hr.v1;` (bounded context `hr`, version `v1`) per the repo convention. 
* **EDA contract families (envelopes):**

  * `hr.domain.intents.v1` → `HrDomainIntent`
  * `hr.domain.facts.v1` → `HrDomainFact` 

This keeps “one bounded context” literal: **all authoritative HR write semantics** and **HR domain facts** are owned by this one Domain primitive.

---

## Domain ownership

### Owned nouns and aggregates

Because the bounded context is *just* `hr`, the domain primitive is the **single writer** for these canonical aggregates (IDs are stable; changes are effective-dated):

1. **Worker**

   * identity + profile (PII, contact points, demographics where applicable)
2. **Employment**

   * employment relationship (tenant/legal entity), status, start/end, worker type
3. **Assignment**

   * position, manager, org unit, cost center, location; **effective-dated** history
4. **Org unit**

   * org tree (departments/teams) + effective changes
5. **Position**

   * position definition + lifecycle (open/filled/closed)
6. **Leave**

   * leave-of-absence case (start/end/status) as a **domain state** (not workflow)

> Notes on scope:
>
> * Payroll *calculation* is not owned here; this domain owns the authoritative **inputs** and emits facts for downstream payroll systems.
> * Long-running onboarding/offboarding orchestration is *not* owned here (that’s a Process primitive later), but the HR Domain still owns the worker/employment state transitions those workflows rely on. 

---

### Identity and scope axes

These axes must appear consistently across write requests and facts:

* `tenant_id` (hard isolation)
* `legal_entity_id`
* `worker_id`
* `employment_id`
* `assignment_id` (or implied by `(employment_id, effective_period)` depending on modeling)
* `org_unit_id`
* `position_id`
* `effective_date` / `effective_period`
* optional `external_ref[]` (ATS id, payroll id, IdP id, etc.)

---

## Domain invariants

These are enforced inside the Domain primitive (not in UI or integrations):

### Identity invariants

* `worker_id` is stable and never reused.
* A worker can have multiple employments, but employments must not violate tenant/legal-entity rules.

### Temporal invariants

* Effective-dated records **cannot overlap** for the same “slot” (e.g., two managers effective at the same time for one employment).
* No “silent overwrite”: historical corrections are modeled as new effective-dated changes, not in-place mutation.

### Lifecycle invariants

* Employment status transitions follow an explicit state machine:

  * example: `Draft → Active → (Leave) → Active → Terminated` (rehire is explicit)
* Termination sets end dates and triggers eligibility implications via facts.

### Governance invariants

* Single-writer: **no other component** writes HR domain state.
* PII access is role-gated; all changes are audit-logged.

---

## Contract surface

Per Ameide, the Domain primitive exposes **Application Services** (write interfaces) and emits **Application Events** (domain facts).  

### Application Service: HrCommandService

A **verb-first** write service (examples; v1 set):

* `HireWorker`
* `UpdateWorkerProfile`
* `CreateEmployment`
* `ActivateEmployment`
* `ChangeAssignment` (position/org unit/manager/cost center/location, effective-dated)
* `StartLeave`
* `EndLeave`
* `TerminateEmployment`
* `RehireWorker`
* `DefineOrgUnit`
* `MoveOrgUnit`
* `CloseOrgUnit`
* `DefinePosition`
* `FillPosition`
* `ClosePosition`

**Design rule:** no `UpdateWorker` / generic patch endpoints as the primary public API; business verbs encode invariants. 

### Application Service shape: RPC and/or bus intents

This Domain primitive supports both invocation modes:

* **RPC commands** for interactive UI flows
* **Asynchronous domain intents** for integrations and batch pipelines

This is required capability→primitive mapping for Domains. 

### Application Event: HrDomainFact

A single envelope message published to the domain fact family.

Representative fact types (v1):

* `WorkerHired`
* `WorkerProfileChanged`
* `EmploymentCreated`
* `EmploymentActivated`
* `AssignmentChanged`
* `OrgUnitDefined`
* `OrgUnitMoved`
* `PositionDefined`
* `PositionFilled`
* `LeaveStarted`
* `LeaveEnded`
* `EmploymentTerminated`
* `WorkerRehired`

---

## Event metadata and delivery guarantees

### Required fact metadata

Every fact must carry shared metadata hints for routing, schema, and idempotency:

* `type` (stable external identifier)
* `schema_subject`
* `stream_ref`
* `partition_key_fields[]`
* `idempotency_key_fields[]` 

Hard rule: **no per-environment broker URLs or literal topic names in proto**; binding is done via CRDs/runtime config. 

### CloudEvents compatibility

Facts emitted across boundaries must be CloudEvents-compatible, including mapping stable type identifiers and carrying trace context. 

### Transactional outbox

* The Domain primitive must emit facts via a **transactional outbox** coupled to the domain write transaction. 
* Domain conformance explicitly requires a testable outbox story and facts annotated with the shared event/fact option. 

---

## Runtime and operator contract

This Domain primitive follows the v2 primitive runtime pattern:

* **Generated surfaces:** gRPC stubs + server skeletons (compile-time drift detection), plus HTTP `/healthz`, `/readyz`, `/metrics`, and outbox implementation when facts exist. 
* **Operator responsibilities:** reconcile Deployment/Service/HTTPRoute, run migrations as Jobs, inject DB/broker secrets and runtime env, surface readiness and outbox/EDA status. 
* **Standard narrow waist:** the runtime must expose `/healthz`, `/readyz`, `/metrics`; operator sets standard ports and identity env vars. 

---

## What “done” means for the HR Domain primitive v1

A Domain primitive is considered defined (and ready for implementation) when we have:

1. **Proto package + topic families** fixed (`ameide_core_proto.hr.v1`, `hr.domain.intents.v1`, `hr.domain.facts.v1`).  
2. **Command verbs** and **fact taxonomy** fixed (above) with stable fact `type` identifiers. 
3. **Invariants** explicitly listed and enforced by the Domain (not by UI).
4. **Outbox** present and tested; facts annotated and emitted reliably. 
5. **Operator/runtime contract** adhered to (health/readiness/metrics, migration job).  

---

If you want, the next iteration step can be **making the aggregate boundaries explicit** (exact command preconditions, the effective-dated model, and the minimal v1 schema for Worker/Employment/Assignment/OrgUnit/Position/Leave) while keeping everything inside this single `hr` bounded context.
