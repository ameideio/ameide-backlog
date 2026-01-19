Below is **HR Capability #1** defined in **ArchiMate terms** following the **528 capability definition template** (Strategy → Business → Application → Technology → Implementation & Migration), with the capability expressed as **EDA contracts** realized by the **Ameide primitives** (Domain/Process/Projection/Integration/UISurface/Agent).  

---

## Market scan summary that drives the capability boundaries

Across “suite” HR/HCM platforms, the recurring pattern is:

* a **Core HR / employee record** foundation (system-of-record),
* surrounded by modules for **payroll**, **talent**, **workforce management (time/scheduling)**, **analytics**, and **employee experience / self-service**.

Examples (not exhaustive):

* **Workday HCM** groups its suite into areas including **Core HCM**, **Talent Management**, and **Workforce Management**, plus planning/analytics and experience/engagement. ([Workday][1])
* **SAP SuccessFactors** positions its suite around **core HR and payroll**, **talent management**, and **HR analytics/workforce planning**. ([SAP][2])
* **Oracle Fusion Cloud HCM** positions itself as a complete cloud solution that connects HR processes with a single experience/data model. ([Oracle][3])
* **Dayforce** emphasizes an all-in-one platform spanning **payroll, HR, benefits, talent, and workforce management**, and highlights a “single data model” concept. ([Dayforce][4])
* **UKG Pro** describes coverage across **HR, payroll, talent, and time** (and workforce management). ([UKG][5])
* **ADP Workforce Now** describes an all‑in‑one solution spanning **HR, time & payroll, and benefits**. ([ADP][6])
* Many orgs also add **HR service delivery / case management** platforms (e.g., ServiceNow) focusing on employee self‑service and HR case workflows. ([ServiceNow][7])
* Modern “unified admin” platforms (e.g., Rippling) extend HR into IT/spend/device management, reinforcing the **Joiner/Mover/Leaver** integration need. ([Rippling][8])
* Global payroll/EOR platforms (e.g., Deel) emphasize **global hiring/payroll/compliance** with integrations into core HRIS/HCM. ([Deel][9])

**Implication for our “first” HR capability:** start with the **foundational Core HR system-of-record** (workforce master data + lifecycle events), and treat payroll, benefits, talent, time, case management, etc. as **adjacent capabilities** integrated via explicit ports/contracts.

---

# HR Capability 1: Workforce Lifecycle Management

## Layer header

* **Primary ArchiMate layer:** Strategy (Capability + Value Stream) 
* **Secondary layers:** Business, Application, Technology, Implementation & Migration 
* **Realization principle:** capability is expressed as **EDA contracts (application services/interfaces/events)** and realized by **Application Components (Ameide primitives)**. 

---

## Strategy layer

### Capability

**Capability (Strategy):** *Workforce Lifecycle Management*
**Capability key (for envelopes/topic families):** `hr.workforce` (v1)

**Intent (what the org can do):**
Maintain an authoritative, auditable, compliant **workforce master record** and manage the **Join / Move / Leave** lifecycle such that downstream systems (payroll, benefits, IT identity, finance, analytics) can reliably react via **facts** and **queries**.

This matches the 528 definition of a capability as a business-level unit defined by value streams, nouns, invariants, and EDA contracts realized by primitives. 

### Outcomes / value

* Single, trusted **worker/employment record** for the tenant/legal entities
* Faster onboarding (fewer re-keys, fewer “missing data” stalls)
* Stronger compliance posture (effective dating, audit trail, privacy controls)
* Lower integration cost (contract-first facts + explicit integration ports)

### Value streams (Strategy: Value Stream)

1. **Join the organization** (candidate → worker → active employment)
2. **Change within the organization** (promotion/transfer/manager change/location change)
3. **Take leave** (leave start/end + status changes)
4. **Leave the organization** (termination/offboarding)
5. **Know who is in the workforce** (directory/org chart/eligibility lookups via read models)

---

## Business layer

### Business actors / roles

* **Business Role:** HR Administrator
* **Business Role:** HR Operations / Shared Services
* **Business Role:** Manager
* **Business Role:** Employee / Worker (self-service)
* **External Business Role (provider):** Payroll provider / Benefits provider / IdP / ATS

### Business processes (Business: Business Process)

These processes realize the value streams:

* **Business Process:** Hire worker (capture identity, contract, start date, position assignment)
* **Business Process:** Maintain worker profile (personal data, contact, emergency contacts, documents)
* **Business Process:** Manage org structure (org units, cost centers, positions, reporting lines)
* **Business Process:** Manage job change (transfer/promotion/comp change effective-dated)
* **Business Process:** Manage leave of absence (start/end, eligibility checks, status)
* **Business Process:** Terminate worker (reason/effective date, final state, eligibility end)
* **Business Process:** Worker / Manager self-service (view profile, request changes, approvals)

### Business policies / invariants (Business: Policy)

These become **hard invariants** in the Domain and **guardrails** in Processes:

* **Identity invariants**

  * A worker has a **stable WorkerId** (does not change when attributes change)
  * A person may have **multiple employments**, but employments cannot have invalid overlapping effective periods (per tenant + legal entity rules)
* **Lifecycle invariants**

  * Status changes are effective-dated and auditable (no silent “overwrite history”)
  * Termination ends eligibility for downstream provisioning/payroll/benefits via emitted facts
* **Data governance invariants**

  * PII is scoped by role and purpose; changes are audit-logged
  * “Single-writer”: Workforce master data is written by the Workforce Domain only

---

## Application layer

Per 528, this is where we define the **application services/interfaces/events (EDA contracts)** and the identity/scope model. 

### Identity & scope model (Application: Data Object)

**Identity axes carried across intents/facts/queries:**

* `tenant_id` (hard isolation boundary)
* `legal_entity_id`
* `worker_id`
* `employment_id`
* `position_id`
* `org_unit_id` / `cost_center_id`
* `effective_date` (or effective period)
* optional: `external_refs[]` (ATS id, payroll id, IdP id, etc.)

### Capability contract shape (Application: Application Event / Application Service)

We standardize topic families using the 528 envelope pattern: 

* **Topic family:** `hr.workforce.domain.intents.v1` → `HrWorkforceDomainIntent`
* **Topic family:** `hr.workforce.domain.facts.v1` → `HrWorkforceDomainFact`
* **Topic family:** `hr.workforce.process.facts.v1` → `HrWorkforceProcessFact`

Mapping note from 528 (important for modeling):

* facts = **Application Events**
* intents/commands = requests to invoke **Application Services**
* queries = read-only **Application Services** (often Projections) 

#### Application Services (write-side)

**Application Service:** `WorkforceCommandService` (canonical write service; realized by Domain primitive)

Representative operations (v1 set):

* `HireWorker`
* `UpdateWorkerPersonalData`
* `AssignPosition`
* `ChangeManager`
* `StartLeave`
* `EndLeave`
* `TerminateWorker`
* `RehireWorker`

**EDA mapping:**

* Synchronous RPC is allowed for interactive UI flows.
* Asynchronous “intent” ingestion is also allowed (same semantics), standardized through `HrWorkforceDomainIntent`.

#### Application Events (domain facts)

**Application Event:** `HrWorkforceDomainFact` (envelope)

Representative fact types:

* `WorkerHired`
* `WorkerProfileChanged`
* `EmploymentChanged` (position/manager/org unit/location/cost center; effective-dated)
* `LeaveStarted`
* `LeaveEnded`
* `WorkerTerminated`
* `WorkerRehired`
* `OrgUnitChanged` / `PositionChanged` (if owned here; see bounded context note below)

These facts are the **contract** for downstream payroll/benefits/IT provisioning/analytics, avoiding direct coupling to the Domain store.

#### Process facts (orchestration signals)

**Application Event:** `HrWorkforceProcessFact` (envelope; produced by Process primitive)

Representative process facts:

* `OnboardingStarted`
* `OnboardingBlocked` (with reason codes: missing I‑9, missing bank, missing approval, etc.)
* `IdentityProvisioned`
* `PayrollEnrollmentCompleted`
* `BenefitsEnrollmentRequested`
* `OffboardingStarted`
* `OffboardingCompleted`

> Note: Processes must not become a system-of-record for domain state; they orchestrate cross-domain invariants and emit process facts. 

#### Application Services (read-side)

**Application Service:** `WorkforceQueryService` (realized by Projection primitive)

Representative operations:

* `GetWorkerProfile(worker_id)`
* `SearchWorkers(filters, text_query)`
* `GetOrgChart(root_worker_id | org_unit_id)`
* `GetWorkforceSnapshot(as_of_date, filters)`
* `GetWorkerEligibility(worker_id, policy_context)` (read-model; not a policy engine)

### Integration ports (Application: Application Interface)

These are explicit seams; realized by **Integration primitives**. 

Minimum ports for v1:

* **Application Interface:** `IdentityProvisioningPort`

  * target examples: Okta / Entra ID / Google Workspace
  * triggered by: `WorkerHired`, `EmploymentChanged`, `WorkerTerminated`
* **Application Interface:** `PayrollSyncPort`

  * target examples: ADP / Dayforce / UKG / SAP/Oracle payroll modules
* **Application Interface:** `BenefitsEnrollmentPort`

  * target examples: benefits admin provider / broker platform
* **Application Interface:** `ATSInboundPort`

  * target examples: Greenhouse/Lever/SuccessFactors Recruiting/Workday Recruiting
  * creates/updates: `HireWorker` intent candidates (or pre-hire entity in later capability)
* **Application Interface:** `DocumentVerificationPort`

  * target examples: I‑9 / right-to-work / background check provider

### Read models (Application: Data Object) owned by Projection primitives

v1 projections (suggested):

* **Read Model:** Workforce Directory Index (searchable directory + facets)
* **Read Model:** Org Chart View (effective-dated reporting relationships)
* **Read Model:** Workforce Change Timeline (audit-friendly, human-readable)
* **Read Model:** Compliance Extract Views (minimal, policy-driven exports)

---

## Mapping to Ameide primitives (Application: Application Component realization)

Per 528, the capability’s contracts are realized by the six primitive kinds.  

| Ameide primitive (Application Component) | Suggested component name (v1)                         | What it owns for this capability                                                                    | Key contract surfaces                                                                |
| ---------------------------------------- | ----------------------------------------------------- | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| **Domain primitive**                     | `hr-workforce-domain`                                 | Single-writer for Worker/Employment/Assignments; enforces invariants; emits domain facts            | `WorkforceCommandService`; `HrWorkforceDomainIntent`; `HrWorkforceDomainFact`        |
| **Process primitive**                    | `hr-workforce-jml-process`                            | Join/Move/Leave orchestration (BPMN runtime: Zeebe or Flowable); retries/compensation; emits process facts | Consumes domain facts; emits `HrWorkforceProcessFact`                                |
| **Projection primitive**                 | `hr-workforce-directory-projection`                   | Search/directory/org chart/read-side queries; idempotent consumers                                  | `WorkforceQueryService`; consumes `HrWorkforceDomainFact` + `HrWorkforceProcessFact` |
| **Integration primitive**                | `hr-workforce-integrations` (split by provider later) | Provider connectors + webhook idempotency + outcome facts                                           | `IdentityProvisioningPort`, `PayrollSyncPort`, etc.                                  |
| **UISurface primitive**                  | `hr-workforce-portal`                                 | HR admin + manager + employee self-service; thin UI reads via queries, writes via commands/intents  | UI → Query + Command services                                                        |
| **Agent primitive**                      | `hr-workforce-assistant`                              | Governed assistant: reads via projections; writes via commands/intents with approvals               | MCP tools/resources (via Integration adapter)                                        |

---

## Technology layer

### Topology modes (Technology)

* **Cloud mode (primary):** full functionality (writes + projections + orchestration).
* **Degraded mode:** read-only workforce directory + cached org chart when downstream providers are unavailable; writes queue as intents for later (policy-driven).

### Platform constraints (Technology: Technology Service)

This capability assumes standard Ameide platform services (Kubernetes operators, broker, DB, workflow runtime, gateway, observability). The stack’s “three planes” (operators/control, proto+SDKs/behavior, CI gates/guardrails) is the build/operate contract. 

---

## Implementation & Migration layer

Work packages (Implementation & Migration: Work Package), sequenced to enable “iterate primitives one by one”:

1. **WP0 — Capability contract baseline**

   * Define proto module(s) and envelope messages for `hr.workforce.*.v1`
   * Define identity axes + required metadata
2. **WP1 — Domain primitive (authoritative writer)**

   * Implement `WorkforceCommandService`
   * Emit `HrWorkforceDomainFact` via outbox
3. **WP2 — Projection primitive (read models)**

   * Directory + org chart + change timeline
   * Implement `WorkforceQueryService`
4. **WP3 — Process primitive (JML orchestration)**

   * Onboarding/offboarding workflows
   * Emit `HrWorkforceProcessFact`
5. **WP4 — Integration primitives**

   * IdP provisioning + payroll sync as first external seams
6. **WP5 — UISurface primitive**

   * HR admin + employee self-service for the acceptance slice
7. **WP6 — Agent primitive + MCP exposure**

   * Add governed tools/resources, approvals, evidence trails

(These WPs align with the overall “capability → contracts → primitives” approach in 528. )

---

## Acceptance slice (minimum end-to-end)

**Slice:** “New hire appears in directory and is provisioned in IdP”

1. HR Admin uses UISurface → calls `WorkforceCommandService.HireWorker`
2. Domain writes Worker+Employment and emits `WorkerHired` fact
3. Projection consumes facts, updates Directory + Org Chart, Query returns the new worker
4. JML Process consumes `WorkerHired`, triggers `IdentityProvisioningPort`
5. Integration provisions identity; Process emits `IdentityProvisioned` fact
6. UISurface shows onboarding status; Agent can answer “Is Alice onboarded yet?” via Projection

---

## Application Interfaces for Agents (MCP) — v1 draft

528 requires an explicit MCP interface decision in every capability definition. 

### 9.1 What this capability publishes

* **MCP tools:** Yes (initially limited to read + low-risk writes)
* **MCP resources:** Yes (directory + worker profile)
* **MCP prompts:** No (v1)

### 9.2 Tool catalog (required)

| Tool                              | Kind    | Canonical Application Service             | Primitive  | Scopes / risk tier            | Approval            | Latency (P95)   | Failure modes                                            | Evidence / audit                          |
| --------------------------------- | ------- | ----------------------------------------- | ---------- | ----------------------------- | ------------------- | --------------- | -------------------------------------------------------- | ----------------------------------------- |
| `hr.workforce.search_workers`     | query   | `WorkforceQueryService.SearchWorkers`     | Projection | `hr.read` (tier 1)            | no                  | < 500ms         | `permission_denied`, `unavailable`, `partial_results`    | query audit log + access log              |
| `hr.workforce.get_worker_profile` | query   | `WorkforceQueryService.GetWorkerProfile`  | Projection | `hr.read_pii` (tier 2)        | no (but role-gated) | < 500ms         | `permission_denied`, `not_found`                         | query audit + PII access log              |
| `hr.workforce.initiate_hire`      | command | `WorkforceCommandService.HireWorker`      | Domain     | `hr.write` (tier 3)           | yes                 | < 2s (or async) | `invalid_argument`, `failed_precondition`, `unavailable` | `WorkerHired` fact + approval record      |
| `hr.workforce.terminate_worker`   | command | `WorkforceCommandService.TerminateWorker` | Domain     | `hr.write_sensitive` (tier 4) | yes                 | < 2s (or async) | `failed_precondition`, `permission_denied`               | `WorkerTerminated` fact + approval record |

### 9.3 Resource catalog

| Resource URI pattern                 | Backing query service                    | Primitive  | Latency (P95) | Notes                   |
| ------------------------------------ | ---------------------------------------- | ---------- | ------------- | ----------------------- |
| `hr-workforce://worker/{worker_id}`  | `WorkforceQueryService.GetWorkerProfile` | Projection | < 500ms       | role-gated PII          |
| `hr-workforce://directory?query=...` | `WorkforceQueryService.SearchWorkers`    | Projection | < 500ms       | supports filters/facets |

---

## What we will iterate next (primitive-by-primitive)

To align with your plan (“next steps we will iterate on the primitives one by one”), the next iteration can start with **Domain primitive decomposition**:

* define the **exact aggregates** and **invariants**
* define the **minimal v1 intent + fact taxonomy**
* define the **write API surface** (RPC + async intent ingestion)
* decide whether **Org Structure** is owned in the same bounded context or split into `hr-org-domain` (still inside this capability umbrella; 528 allows multiple bounded contexts per capability). 

[1]: https://www.workday.com/en-us/products/human-capital-management/overview.html?utm_source=chatgpt.com "Human Capital Management (HCM) Software | Workday US"
[2]: https://www.sap.com/products/hcm.html?utm_source=chatgpt.com "SAP SuccessFactors: Human Capital Management Software"
[3]: https://www.oracle.com/human-capital-management/?utm_source=chatgpt.com "Human Capital Management (HCM)"
[4]: https://www.dayforce.com/?utm_source=chatgpt.com "Dayforce - Global HCM Software | HR, Pay, Time, Talent ..."
[5]: https://www.ukg.com/products/ukg-pro?utm_source=chatgpt.com "UKG Pro® Human Capital Management (HCM) Software"
[6]: https://www.adp.com/what-we-offer/products/adp-workforce-now.aspx?utm_source=chatgpt.com "Workforce Now® All-In-One HR Software"
[7]: https://www.servicenow.com/products/hr-service-delivery.html?utm_source=chatgpt.com "HR Service Delivery - HR Management - ServiceNow"
[8]: https://www.rippling.com/?utm_source=chatgpt.com "Rippling: #1 Workforce Management System | HR, IT, Finance"
[9]: https://www.deel.com/en/?utm_source=chatgpt.com "Deel | Global Payroll, Compliance, HR Solutions | HRIS"
