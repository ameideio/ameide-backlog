Here’s a concrete “requirement → release” flow for **“add a field + some logic + an agentic decision in the Sales process”**, expressed in the **Transformation capability** terms (Domain/Process/Projection/Integration/Agent/UISurface), with **technical steps** and **I/O per phase**.

## First: what is orchestrating what?

Transformation is meant to act like an **IT4IT-aligned value-stream capability**:

* **System of record** = **Transformation Domain** (design-time truth + baselines/promotions/approvals + Definition Registry).  
* **Process of record** = **Transformation Process** (Temporal workflows executing **promoted ProcessDefinitions**, emitting **process facts**, not owning canonical state). 
* **Read model of record** = **Transformation Projection** (idempotent consumer of domain facts + process facts; query services for UI/agents). 
* **External orchestration hooks** = **Transformation Integration** (git provider/registry/CI status connectors + transport bindings; idempotent; emits commands/intents only, never facts). 
* **Agentic work** = **Transformation Agents** (read via projection; write via governed commands/intents; tool grants + risk tiers stored as AgentDefinitions). 
* **Human UX** = **Transformation UISurface** (thin: reads via projection, writes via commands/intents only). 
* **Contract spine** = proto topic families + envelope invariants (tenant/org/repo + traceability). 

This framing matters because it makes the CLI a **tool invoked by the process**, not “the process”.

### Clarification: “Sales BPMN” vs “Delivery BPMN”

This scenario intentionally involves **two different BPMN concerns**:

1. **Sales business workflow BPMN** (optional, capability-owned):
   - A BPMN model that describes how Sales work flows (business semantics).
   - This may be **model-only** (design artifact) or **executable** (compiled to a Sales Process primitive), depending on what you choose for the Sales capability.
2. **IT4IT delivery workflow BPMN** (Transformation-owned):
   - A ProcessDefinition such as “Requirement → Release” that orchestrates the delivery loop (scaffold → generate → verify → promote → deploy).
   - This is executed by the **Transformation Process primitive** and emits **process facts** as orchestration evidence.

If you want “each BPMN step produces an event”, the default interpretation is: **each executable BPMN activity emits process facts** (see `backlog/527-transformation-proto.md` §3.1) and projection materializes run timelines (see `backlog/527-transformation-projection.md` §4.3).

---

## Practical example: add a field + agentic decision in Sales

### The change

A product owner asks:

> “In the Sales process, add a new field `deal_risk_tier` and apply logic + an agentic decision: if risk is high, route to manager approval; otherwise auto-continue.”

This typically touches:

* **Sales contracts (proto)**: add the new field + add/adjust an intent/fact so downstream systems can rely on it.
* **Sales domain primitive**: store the field and emit the fact via outbox (single-writer behavior).
* **Sales process primitive**: orchestration logic (route decision).
* **Sales agent primitive** (or shared agent): compute “risk tier” + rationale.
* **Sales projection primitive**: show the field and decision rationale in read models.
* **Sales UI surface**: display the field / approval state.

Now the Transformation capability orchestrates the requirement to release.

---

# Phase 1 — Initiate (intake + scope locking)

### Input

* Free-text requirement + “done when…” acceptance criteria
* Target: Sales capability
* Risk posture: “contains agentic decision” (so it will require stronger evidence/approval)

### Actions (what happens)

1. **Transformation UISurface** (or a Transformation agent in the portal) creates a “change item” inside a Transformation initiative (or Scrum profile item).

   * The key is: it’s now tracked with the canonical `{tenant_id, organization_id, repository_id}` scope.  

2. **Transformation Process** starts a “Requirement → Release” workflow instance (Temporal) for that change.

   * This is explicitly in-scope: “delivery loop orchestration (scaffold → generate → verify → promote → deploy)”.
   * Each workflow activity transition must emit process facts (see `backlog/527-transformation-proto.md` §3.1).

### Output

* A workflow instance exists with a correlation id
* A first “requirements captured” **process fact** is emitted (evidence that orchestration began)
  (Process emits **process facts only**, not domain facts.) 

### Expectations for agentic development

* Agent is not coding yet; it’s producing an **impact assessment** and a **checklist** (more below). This aligns with “agents propose/prepare changes” rather than bypass boundaries. 

### Phase 1 checklist (per change)

- [ ] Scope is explicit: `{tenant_id, organization_id, repository_id}`.
- [ ] Change item exists (initiative/work item) and is linkable to repository elements/versions.
- [ ] Process instance started (Requirement → Release) and first process fact emitted.

---

# Phase 2 — Design (models + contracts become explicit)

This is where you prevent “drift-by-implementation”.

### Input

* The change item from Phase 1

### Actions (what happens)

3. **Update design-time process model (business workflow, optional)**: if Sales uses BPMN as a design artifact, update the Sales workflow model to include:

   * a task “Assess risk”
   * a decision “requires manager approval?”
   * an explicit message boundary (what intent triggers it, what fact results)

   Notes:

   - If this Sales BPMN is intended to be **executable**, it must have a promoted execution profile and compile target (see `backlog/527-transformation-process.md` §3.4.1).
   - If it is **model-only**, it still lives as design-time truth (Elements/versions) and can be referenced by contracts/definitions as justification.

4. **Update delivery ProcessDefinition (IT4IT)**: ensure the “Requirement → Release” ProcessDefinition has an explicit step for:

   - contract validation (buf lint/breaking + codegen freshness),
   - verification (`ameide primitive verify` + tests),
   - and promotion gates (Approve/Reject/Override/Cancel).

   This ProcessDefinition must emit step transition evidence as process facts (see `backlog/527-transformation-proto.md` §3.1).

5. **Make protos central** (contract-first): update Sales protos to carry:

   * the new field (`deal_risk_tier`)
   * the intent/fact boundary for the decision (e.g., `AssessDealRiskRequested` intent + `DealRiskAssessed` fact, or extend an existing fact)
   * keep envelope invariants (tenant/org/repo + traceability) consistent. 

6. **Define the agent’s scope and authority** (design-time definition):

   * Update or create an **AgentDefinition** for the “SalesRiskAgent” role:

     * allowed tools (queries vs commands)
     * risk tier (e.g., read-only vs “may propose writes”)
   * This belongs in the **Definition Registry**, alongside ProcessDefinitions and ExtensionDefinitions. 

7. **If Sales BPMN is executable: define the BPMN execution profile**

   - Bind each executable BPMN activity to:
     - emitted step process facts,
     - any required domain intents (send),
     - any awaited domain facts (wait),
     - correlation/causation propagation rules.
   - Store the resolved profile as a promotable definition (recommended as an `ExtensionDefinition`), keeping the BPMN XML portable (see `backlog/527-transformation-process.md` §3.4.1 and `backlog/527-transformation-domain.md` §4.1).

### Output

* BPMN change recorded as a new versioned definition ready for promotion
* Proto diff (contract diff) ready for implementation
* AgentDefinition diff (tool grants / risk tier) ready for promotion
* (If executable) BPMN execution profile definition ready for promotion

### Gate (recommended)

8. **Design gate**: the workflow asks for approval:

* `ApproveGate` / `RejectGate` are explicit workflow signals used for gates (baseline promotion, definition promotion). 

### Phase 2 checklist (per change)

- [ ] Protos updated first (IDL-first); buf lint/breaking are clean for the proposed change.
- [ ] Design-time model updates captured as elements/versions (any notation allowed; BPMN is the workflow-specific authoring case).
- [ ] AgentDefinition updates (tool grants + risk tier) captured as a promotable definition (when agentic behavior is introduced).
- [ ] If BPMN is deployable/scaffoldable: collaboration present and bindings resolve; promotion would produce compiled workflow IR + scaffolding plan.
- [ ] Design gate completed (Approve/Reject recorded as process facts).

---

# Phase 3 — Realize (code changes, scaffolding, verification)

This is where the CLI becomes “a tool inside the process”.

### Input

* Promoted (or at least accepted) contract/design diffs:

  * updated proto
  * updated BPMN ProcessDefinition
  * updated AgentDefinition

### Actions (what happens)

7. **Scaffolding / codegen is triggered (WorkRequest → ephemeral execution → evidence)**
   The Transformation Process requests execution via the **WorkRequest seam** (Domain-owned), then Integration executes:

   * Process issues a Domain intent to create a `WorkRequest` (`work_kind=tool_run`, `action_kind=scaffold|generate`, repo coordinates, plan refs, idempotency key).
   * Domain persists the WorkRequest and emits `WorkRequested` as a domain fact (outbox).
   * KEDA ScaledJobs consume `WorkRequested`, schedule a devcontainer-derived Kubernetes Job, check out the repo, run the scaffolder/codegen, and store artifacts.
   * The Job records outcomes back into Domain idempotently (evidence bundle + terminal status).
   * Process awaits the resulting domain facts and emits `ToolRunRecorded` process facts referencing the evidence (see `backlog/527-transformation-proto.md` §3.1 and `backlog/527-transformation-integration.md` §1.0.5).

8. **Agentic coding step (implementation)**
   A coding agent (or a human) applies the checklist and implements:

**Sales proto**

* Add field
* Add/extend intent/fact message(s)

**Sales domain primitive**

* Accept new field on the write boundary
* Persist it
* Emit the fact via outbox (domain facts after persistence) 
* Keep idempotency constraints if command-driven (e.g., `client_request_id` behavior). 

**Sales process primitive**

* Implement routing logic:

  * deterministic “simple rules” first (if possible)
  * agentic decision invoked in a governed way:

    * the agent reads via projection
    * the agent produces a proposal/decision
    * the process then triggers a domain intent/command to apply it (domain persists; domain emits facts)
* Keep the boundary: processes orchestrate, they don’t become the system of record.  

**Sales agent primitive**

* Implement “Risk assessment”:

  * reads context via projection queries
  * emits commands/intents only (no bypassing writer boundaries) 
  * operate under stored tool grants + risk tier (AgentDefinitions) 

**Sales projection primitive**

* Ingest the new facts/fields
* Serve it via query APIs (UI and agents read via projection). 

**Sales UISurface**

* Display the field + approval state
* Stay thin: read via projection, mutate via commands/intents only. 

9. **Verification is run (as a gate, not advice)**
   Verification is also requested and executed via the WorkRequest seam so it is replay-safe and auditable:

* This is part of the “scaffold → verify → promote” delivery loop. 
* Verification should validate:

  * contracts/codegen freshness
  * security posture
  * naming/verb discipline
  * minimal tests
  * capability-local GitOps presence (repo-owned deploy shape; env wiring is CI-owned)
  * etc.

**Key: verification output becomes the “next-step instruction list” for the agent.**

Cross-reference: verification expectations should align with `backlog/521f-external-verification-baseline.md`.

### Output

* A PR with:

  * proto changes
  * updated generated code
  * Sales primitive code updates (domain/process/agent/projection/ui)
  * tests updated/added
  * capability-local GitOps updates as required by policy
* A verification report + evidence bundle (logs, links, artifact references)

### Phase 3 checklist (per change)

- [ ] Scaffolding run via runner (or locally) is idempotent and produces repo-owned skeleton changes only.
- [ ] `buf generate` executed and regen-diff is empty.
- [ ] Implementations honor boundaries:
  - [ ] Domain persists + emits facts after persistence (outbox).
  - [ ] Process orchestrates; emits process facts only.
  - [ ] Projection ingests facts; UI/agents read via projection.
  - [ ] Agents write only via commands/intents and remain governed.
- [ ] Verification is green and captured as evidence (`ToolRunRecorded` / logs/reports attached).

---

# Phase 4 — Govern + Promote (baseline + definition promotion)

### Input

* PR contents + verification evidence

### Actions (what happens)

10. **Promotion gate**

* Workflow requests approval (`ApproveGate`) for:

  * promoting the ProcessDefinition version
  * promoting the AgentDefinition version
  * promoting the executable BPMN execution profile (if applicable)
  * promoting the “release baseline” (if you use baselines to represent “what’s shippable”) 

11. **Domain records the promoted versions**

* This is the point where the system-of-record asserts “this is the approved definition / baseline”.
* Domain owns baselines/promotions/approvals as canonical state. 

### Output

* Promoted definition versions (ProcessDefinition / AgentDefinition) stored and referenceable
* Promotion facts emitted; projections reflect them in audit trail

### Phase 4 checklist (per change)

- [ ] Approval decisions are recorded as process facts with traceability metadata.
- [ ] Promotions update canonical pointers (baseline/definition promotions are domain-owned).
- [ ] Evidence bundles are linked (verify output + CI links + conformance outputs when required).

---

# Phase 5 — Release + Deploy (operation)

### Input

* Approved PR + promoted definitions/baseline

### Actions (what happens)

12. Merge + CI build + deploy

* Integration triggers CI and deploy mechanics (gitops/cluster/etc.). Integration’s job is to connect tools and remain audit-friendly (evidence bundles).  

13. Projection provides operational visibility

* Projection consumes the domain/process facts and provides the UI/agent read model for “what shipped, what’s live, what’s approved.” 

### Output

* Sales process change deployed
* The portal can show:

  * the new field in read models
  * the decision path taken (manager approval vs auto)
  * the evidence chain (verify logs, CI run link, promotion record)
  * the per-step run timeline (“each step produces an event”) from process facts joined to correlated domain facts (see `backlog/527-transformation-projection.md` §4.3)

### Phase 5 checklist (per change)

- [ ] Build + deploy is reproducible (GitOps + smoke tests) and captured as evidence.
- [ ] Projection provides “what shipped/what’s approved/what’s running” visibility.
- [ ] Process run timeline is available (step events as process facts, correlated to domain facts).

---

## The checklist you want agents to follow (what to auto-generate)

This is the “simple + opinionated” part: every change request like this generates the same checklist, filled with specifics.

**Design checklist**

* [ ] BPMN ProcessDefinition updated + versioned (Assess risk + decision point)
* [ ] Proto contract updated (new field + intent/fact boundary)
* [ ] AgentDefinition updated (tool grants + risk tier)

**Implementation checklist**

* [ ] Sales Domain: persist field; emit fact after persistence; idempotency respected 
* [ ] Sales Process: orchestrates only; does not become system of record 
* [ ] Sales Agent: reads via projection; writes only via intents/commands; governed by tool grants 
* [ ] Sales Projection: ingests new facts; serves read surfaces 
* [ ] UI: reads via projection; writes via commands/intents 

**Verification + promotion checklist**

* [ ] Verify green (policy gates)
* [ ] ApproveGate completed for definition promotion and release promotion 
* [ ] Evidence bundle attached (verify output + CI links)

---

## Non-happy paths (and what the orchestration should do)

These are the common “rework loops” your process must make cheap:

1. **Proto change is accidentally breaking**

* Process fails at contract gate (buf/breaking).
* Action: revert to additive field, introduce new message, or new RPC—but don’t “work around” by skipping checks.
* Outcome: redo Phase 2 (Design) → Phase 3 (Realize).

2. **Agentic decision introduces unsafe or non-repeatable behavior**

* Gate: risk tier rejects auto-apply; requires human approval (ApproveGate). 
* Action: constrain tool grants / add “propose-only” path / add stronger tests; then re-verify.

3. **Verification failures**

* Should be treated as “definition of done not met”, not a suggestion list.
* Process loops Phase 3 until green, then returns to promotion gate.

4. **“Policy vs implementation drift”**

* If the guardrail is wrong (e.g., brittle expectations), the right fix is: correct the guardrail to match policy intent, not force teams into cargo-cult compliance.
* In this architecture, that kind of correction belongs as an **ExtensionDefinition / tooling evolution**, i.e., part of the system of record for how scaffolding and verification should work. 

---

## The “simple, predictable, opinionated” takeaway

If Transformation is doing its job, then for changes like this the developer experience collapses to:

* **One tracked change** (initiative/work item)
* **One contract-first diff** (proto)
* **One implementation diff** (primitives)
* **One definition of done** (“verify green + approved promotion + deployed”)
* **One evidence trail** (process facts + domain facts + CI logs)

…and the CLI is just a runner the **Process + Integration** use to enforce that bar, not a bag of flags.

If you want, I can turn this into:

* an explicit **BPMN-like workflow** (tasks + gates + retries) for the “Sales change” lifecycle, **and**
* a concrete template for the generated **AGENTS.md** (the exact checklist file content your scaffolder should emit for Sales changes).
