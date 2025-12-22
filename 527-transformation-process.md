# 527 Transformation — Process Primitive Specification

**Status:** Draft (scaffold implemented; verification posture defined; workflow logic pending)  
**Parent:** [527-transformation-capability.md](527-transformation-capability.md)

This document specifies the **Transformation Process primitives** — Temporal-backed governance workflows that execute promoted, design-time **ProcessDefinitions** (BPMN 2.0 as source of truth) and the "design → scaffold → verify → promote → run" delivery loop.

**Extensions:**
- MCP write governance (agent-driven writes): `backlog/536-mcp-write-optimizations.md`

---

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (Process), Application Events (process facts), Application Services (workflow signals).
- **Out-of-scope layers:** Canonical state writing (domain responsibility) and portal read models (projection responsibility).

## 1) Process responsibilities

Scope identity (required everywhere): `{tenant_id, organization_id, repository_id}` (used for correlation, not as a read/write shortcut; no separate `graph_id`).

Process primitives implement **cross-domain orchestration**:

- Timeboxes, governance, approvals, and orchestration driven by **design-time ProcessDefinitions** stored and promoted in the Transformation Definition Registry (Domain).
- Deterministic execution (Temporal) with explicit workflow state transitions emitted as **process facts**.
- No system-of-record behavior: processes never own canonical business state.

**IT4IT alignment (default):** “IT4IT processes” are realized here as Process primitives inside the Transformation capability boundary:

- Process primitives execute **promoted ProcessDefinition versions** (design-time truth owned by Domain).
- They orchestrate the capability value streams (Initiate/Design/Realize/Govern) and produce auditable process facts as evidence.
- They never become writers of canonical domain truth (all mutations occur via domain intents/commands).

Core workflow families:

- **TOGAF/ADM governance** (phase progression, required deliverables, review/approval gates).
- **Scrum governance** (Sprint start/end, readiness cues) aligned with `backlog/506-scrum-vertical-v2.md`.
- **PMI governance** (stage gates, approvals, reporting cadence).
- **Delivery loop orchestration** (scaffold → generate → verify → promote → deploy) for primitives and definitions.

## 2) Contract boundaries (non-negotiable)

- Processes **emit process facts only**; they do not emit domain facts.
- All cross-domain writes occur via **domain intents / commands**; domains persist and emit facts.
- **No synchronous write coupling:** runtime workflows MUST NOT synchronously RPC into other primitives to perform writes; they drive change only via domain intents/commands (including an enqueue-style `SubmitIntent` surface, where the response does not carry “updated state” and does not become a hidden transactional dependency).
- **Reads are allowed but constrained:** runtime workflows may use projection-backed reads when necessary, and any “control-plane lookup” must be explicitly declared (timeout/retry posture) and never become a hidden dependency that recreates a distributed monolith.

### 2.1 Tool execution boundary (where CLI fits)

Workflow steps often require deterministic tool execution (scaffolding, codegen, verification, builds, publishing). The boundary is:

- The **Process primitive** owns sequencing/retries and emits process facts as evidence.
- An **Integration runner** performs external side effects (run `bin/ameide`, `buf`, tests/builds) and captures outputs as evidence.
- The **CLI is a tool**, not the orchestrator; it is invoked by workflow activities and by humans locally, but the process-of-record is always the Process primitive executing a stored ProcessDefinition.

**Execution substrate (normative; v1):** long-running tool/agent steps run on an event-driven WorkRequest seam:

- Process requests execution via a Domain intent that creates a Domain-owned `WorkRequest` (idempotent).
- Domain emits `WorkRequested` facts after persistence (outbox).
- KEDA ScaledJobs schedule devcontainer-derived Kubernetes Jobs based on Kafka consumer group lag on `WorkRequested` (Jobs consume and record results back to Domain idempotently).
- Process awaits the resulting domain facts, then emits `ToolRunRecorded` / `GateDecisionRecorded` / `ActivityTransitioned` process facts for the audit timeline.

Kafka note (normative):

- The WorkRequest queue uses dedicated Kafka topics per executor class (v1: `transformation.work.queue.toolrun.verify.v1`, `transformation.work.queue.toolrun.generate.v1`, `transformation.work.queue.agentwork.coder.v1`); KEDA scales by consumer group lag and the Job is the Kafka consumer.
- WorkRequest processors must disable auto-commit and only commit offsets after outcomes/evidence are durably recorded in Domain.

## 3) BPMN ProcessDefinitions (executable + scaffolding driver)

This capability treats **BPMN as the canonical authoring source** for governance/delivery workflows. To make BPMN simultaneously:

- **portable** (standards-compliant BPMN 2.0),
- **executable** (Temporal-backed workflows), and
- **scaffoldable** (deterministic generation targets),

we separate **source-of-truth** from **derived build inputs**:

- **Source (canonical):** BPMN 2.0 payload stored as an element-centric versioned payload (`ElementVersion`) and referenced by a Definition Registry entry (`ProcessDefinition`).
- **Derived (promotable):** a compiled **Workflow IR** and a **Scaffolding Plan** derived from the promoted BPMN version and stored/promoted in the Definition Registry.

### 3.1 Supported BPMN subset (v1)

v1 supports a deliberately small executable subset (expand later; any unsupported constructs MUST fail verification):

- `startEvent`, `endEvent`
- `sequenceFlow`
- `serviceTask` (machine step; emits intents/commands)
- `userTask` (human gate; maps to signals + approvals)
- `exclusiveGateway` (branching; decision reads must be declared)
- Collaboration constructs for boundaries: `collaboration`, `participant`, `messageFlow` (required for “scaffoldable” processes; see below)

**Waits/timers (v1 stance):**

- v1 does not require BPMN-native wait/timer constructs (e.g., `intermediateCatchEvent`, `timerEventDefinition`) to express “wait for an external event”.
- “Wait” semantics are expressed via the execution profile (`await_domain_facts`) so the workflow can deterministically advance when a correlated domain fact arrives.
- If/when BPMN-native waits/timers are added, they MUST map to the same execution profile concepts and produce the same step evidence (process facts).

### 3.2 Collaboration requirement (for “scaffoldable”)

If a ProcessDefinition is marked “deployable/scaffoldable”, it MUST be a **Collaboration** diagram:

- Participants (pools) define component ownership/boundaries for scaffolding.
- Message flows define the interaction contract (who sends what to whom).

Single-process diagrams may exist for documentation and learning, but they are not accepted as “scaffoldable” unless ownership and interaction bindings are fully specified by profile-approved metadata (see 3.3).

#### 3.2.1 MessageFlow mapping rule (deterministic scaffolding)

For deployable/scaffoldable workflows, every `messageFlow` MUST be representable deterministically in the promoted execution profile (see 3.4.1), without guessing topic names from BPMN:

- A `messageFlow` from participant **A → B** requires:
  - at least one activity in participant **A** that **sends a domain intent** (`send_domain_intent`), and
  - at least one activity in participant **B** that **awaits a domain fact** (`await_domain_facts`) correlated to that intent, and
  - correlation/causation propagation rules that allow projection to join step evidence to domain facts (see `backlog/527-transformation-proto.md` §3.1).

If a `messageFlow` exists but the execution profile cannot resolve the corresponding send/await bindings, promotion MUST fail.

#### 3.2.2 Participant → component identity rule (portable; no guessing)

To make scaffolding deterministic, each BPMN `participant` in a deployable/scaffoldable Collaboration MUST declare a **component reference** using portable metadata (standards-compliant carrier: `bpmn:documentation` or a linked binding element).

Minimum required fields (conceptual):

- `component_ref.kind`: `domain | process | projection | integration | agent | uisurface`
- `component_ref.name`: primitive name (e.g., `sales`, `transformation`)
- optional `component_ref.instance`: deployment instance identifier if needed (defaults to `default`)

Promotion MUST fail if any participant cannot be resolved to a component reference. This prevents divergent scaffolding outputs for the same diagram.

### 3.3 Binding metadata without breaking standards compliance

Profiles govern how binding metadata is provided.

- **Standards-compliant BPMN profile (default for portability):**
  - Binding metadata MUST be carried in either:
    - `bpmn:documentation` as structured YAML/JSON (with a strict marker/header), or
    - a separate linked Element (e.g., a `DOCUMENT_MD`/`DOCUMENT_JSON` ElementVersion) referenced from the BPMN element via a REFERENCE relationship.
  - BPMN XML must remain portable; no non-standard namespaces are required for correctness.

- **Extended BPMN profile (opt-in):**
  - BPMN extension elements/namespaces MAY be used for richer bindings (e.g., `ameide:*`), but downstream portability is explicitly reduced and must be surfaced in conformance results and exports.

### 3.4 Binding contract (what a deployable BPMN MUST define)

For each executable BPMN node that produces side effects, binding must resolve to:

- **Intent mapping:** exact domain intent type (proto message), topic family, and subject/aggregate scope.
- **Idempotency posture:** idempotency key strategy for retries (UI/process/operator).
- **Read dependencies (if any):** projection query(s) required for decisions and how they are bounded (timeouts, caching posture).
- **Evidence hooks:** what process facts are emitted for the transition and what evidence/attachments are required (if any).

#### 3.4.1 BPMN execution profile (promotable binding schema)

To keep BPMN portable and still generate code/contracts deterministically, bindings are normalized into a **promotable execution profile**:

- **Authoring inputs** (standards-compliant):
  - `bpmn:documentation` (structured YAML/JSON) and/or a linked binding element (REFERENCE relationship).
- **Canonical derived form (promoted):**
  - an `ExtensionDefinition` (or equivalent schema-backed definition) representing the resolved execution/binding profile for the ProcessDefinition version.

Execution profile requirements (conceptual shape; do not embed proto text):

- Profile identity:
  - `execution_profile_id` + `execution_profile_version`
  - `profile_kind = bpmn-execution-profile/v1`
- Target:
  - `{process_definition_id, process_definition_version}` (required)
- Participants (from Collaboration):
  - list of `participant_id` → `component_ref` mappings (required for deployable/scaffoldable)
- Activity bindings (keyed by BPMN element id):
  - `activity_id`
  - `emit_process_facts`: which step transitions MUST be emitted (see `backlog/527-transformation-proto.md` §3.1)
  - `send_domain_intent` (optional): intent type + topic family + subject scope + idempotency key strategy
  - `await_domain_facts` (optional): which facts (by type/subject) unblock the workflow and how correlation is computed
  - `run_work_request` (optional): request execution via a Domain-owned WorkRequest (tool run or agent work) and define how the workflow waits for completion:
    - `work_kind` ∈ `{tool_run, agent_work}`
    - `queue_ref` (logical; environment binds actual broker/subject)
    - `timeout_policy` / `retry_policy` (bounded; retries rely on idempotency)
    - correlation rules: how `work_request_id` / `correlation_id` is propagated into domain intents/facts
  - `allowed_reads` (optional): projection queries allowed for decisions (timeouts/retry posture)
  - `timeout_policy` / `retry_policy` (optional)

Defaults and invariants (v1):

- **Step events are mandatory** for deployable/scaffoldable workflows:
  - `emit_process_facts` MUST include `STARTED`, `COMPLETED`, and `FAILED` for every executable node.
  - Suppression is not allowed for deployable/scaffoldable workflows.
- Gateways are executable nodes too:
  - gateways MUST emit step transition evidence (represented as `ActivityTransitioned` with `activity_id = gateway_id`).
  - gateway decisions should be explainable either via:
    - `ActivityTransitioned` fields (selected path/branch summary), and/or
    - a dedicated read-evidence record (see `backlog/527-transformation-proto.md` §3.1 and `backlog/527-transformation-projection.md` §4.3).

Promotion rule:

- A ProcessDefinition version is only “deployable/scaffoldable” when its execution profile resolves with no missing mappings; otherwise promotion fails.

### 3.5 Execution model (compile-to-Temporal)

Process primitives are Temporal-backed. v1 execution posture is:

- **BPMN is not interpreted directly at runtime.**
- A promoted BPMN version is **compiled** to a deterministic **Workflow IR**.
- A generic Temporal workflow runner executes that IR (so the Process primitive stays stable while definitions evolve).

This keeps BPMN as the authoring and governance source of truth while making execution deterministic and testable.

### 3.6 Verification gates (deployable AND scaffoldable)

Promotion of a ProcessDefinition version MUST fail unless:

- the BPMN payload parses and conforms to the supported subset,
- the collaboration/participant/message-flow contract is present (if deployable/scaffoldable),
- bindings fully resolve (no “unknown intent”, “missing subject scope”, “undeclared read dependency”),
- compilation produces:
  - a `CompiledWorkflowDefinition` (Workflow IR), and
  - a `ScaffoldingPlanDefinition` (generation plan),
- both derived definitions are traceable back to `{process_definition_id, process_definition_version}` and scoped by `{tenant_id, organization_id, repository_id}`.

### 3.6.1 Single orchestrator rule (v1)

For compile-to-Temporal execution, v1 requires a single orchestrator:

- A deployable/scaffoldable Collaboration MUST contain exactly one executable orchestrator process body that compiles to Workflow IR.
- Other participants represent boundaries/contracts and may receive intents/produce facts, but do not define additional orchestrators in the same promoted definition.

This avoids accidental “multi-orchestrator” semantics and keeps compilation/execution deterministic.

### 3.6.2 Deterministic scaffolding mapping (what BPMN scaffolds)

The goal is “one authoring source drives execution AND scaffolding of supporting services” without letting BPMN invent schemas. The deterministic mapping is:

- Participants → components/primitives
  - For each participant `component_ref`, the `ScaffoldingPlanDefinition` MUST include a component entry.
  - If the referenced primitive does not exist yet, scaffolding tasks MUST include creating a repo-owned skeleton for that primitive kind/name (never overwriting implementation-owned code).
- Message flows → interaction contracts
  - Each message flow MUST correspond to bindings that reference an **existing proto symbol** (intent/fact type).
  - Scaffolding tasks MAY create:
    - publisher adapters in the orchestrator (to emit intents), and
    - subscriber/handler skeletons in the target domain primitive (to handle intents / emit facts),
    but MUST NOT invent new proto messages; missing protos are a Design-gate failure.
- Service tasks → runnable activities + adapters
  - If an activity has `send_domain_intent`, scaffolding tasks must ensure:
    - an intent publish path exists (adapter/SDK client),
    - idempotency key propagation rules are implemented,
    - and a corresponding handler exists in the target domain primitive (stubbed if absent).
- Allowed reads → explicit evidence
  - If an activity declares `allowed_reads`, the runner MUST record read evidence in the run timeline (either as `ReadPerformed` or as citations embedded in the activity transition evidence; see `backlog/527-transformation-proto.md` §3.1).

### 3.7 Notation-driven derivations (design-time content expansion)

Transformation treats all notations as profiles over Elements, but they can still be **actionable**: promotion gates may derive promotable outputs from promoted, normative artifacts (per profile).

Separation rule:

- **BPMN derives process realization artifacts** (compiled workflow IR + scaffolding plan) for Process primitives.
- **Capabilities and their contracts/decomposition** are architecture-level elements/definitions (commonly ArchiMate), not derived from BPMN.

Examples (illustrative):

- BPMN ProcessDefinition → `CompiledWorkflowDefinition` + `ScaffoldingPlanDefinition`
- ArchiMate model/view → `CapabilityEDAContract` + `CapabilityPrimitiveDecomposition` (and related derived plans)

These derivations are executed as Process gates (deterministic workflows), persisted by Domains, and evidenced by process facts. They must never introduce a parallel “ArchiMate tables as truth” storage model.

## 4) Signals (generic)

- `ApproveGate` / `RejectGate` — human approvals for governance gates (baseline promotion, definition promotion).
- `CancelWorkflow` — abort with audit trail.
- `OverridePolicy` — explicit break-glass for authorized roles (recorded as a process fact).

Role checks and evidence requirements for these signals are policy-driven and stored as design-time definitions (see `backlog/527-transformation-domain.md` §3.5).

## 5) Implementation notes

### Scaffold commands

```bash
ameide primitive scaffold --kind process --name togaf --include-gitops
ameide primitive scaffold --kind process --name pmi --include-gitops
```

### ProcessDefinitions

Store the canonical workflow specs in Transformation as BPMN 2.0 (element-centric versions) and bind Process CRs to **promoted** versions. Runtime execution uses the compiled Workflow IR derived from the promoted BPMN version.

**Metamodel profile gates:** when a workflow promotes a “normative” model/view, it MUST run the validator/exporter for the selected notation (metamodel) profile:

- Standards-compliant profiles (e.g., BPMN 2.0 / ArchiMate 3.2) block promotion if the element graph contains non-standard `type_key` namespaces or invalid relationships.
- Extended profiles can promote under their own constraints, but downstream automation must explicitly declare support for that profile.

Profile definitions (validators/exporters/constraints) are versioned/promotable design-time definitions (see `backlog/527-transformation-domain.md` §4.4).

## 5.1) Implementation progress (repo snapshot; checklists)

Delivered (scaffold + guardrails):

- [x] Process scaffold exists at `primitives/process/transformation` (worker + ingress + workflow shape, tests).
- [x] Process facts catalog stub exists (events directory present; initial catalog defined).
- [x] Gate: `bin/ameide primitive verify --kind process --name transformation --mode repo` passes (GitOps TODOs remain).

Not yet delivered (process meaning):

- [ ] BPMN → (bindings) → compiled Workflow IR → Temporal execution (end-to-end).
- [ ] BPMN-derived `ScaffoldingPlanDefinition` generation + “fail closed” verification gates.
- [ ] WorkRequest-based runner invocation from workflow activities (domain facts trigger ephemeral runner Jobs; outcomes recorded to Domain; process emits `ToolRunRecorded`).
- [ ] Deterministic governance workflows (Scrum/TOGAF/PMI) and continuous refinement loop.
- [ ] Full process-facts emission semantics per workflow node transition (ActivityTransitioned/GateDecisionRecorded/ToolRunRecorded/ReadPerformed) correlated to domain facts.

## 5.2) Clarification requests (next steps)

Confirm/decide:

- The v1 BPMN supported subset (confirm/expand) and the required binding schema for deployable/scaffoldable ProcessDefinitions.
- Whether collaboration diagrams are required for deployable/scaffoldable definitions (recommended) vs allowing non-collaboration diagrams with explicit ownership bindings.
- The canonical “continuous refinement loop” event catalog (process facts) and the minimum required fields for correlation/audit replay.
- Which gates are human-in-loop by default vs autonomous, and what signals are allowed at each gate (Approve/Reject/Override/Cancel).
- The canonical mapping from each signal to the next domain command/intent (including role checks and evidence requirements).

## 5) Acceptance criteria

1. Governance workflows are Temporal-backed and emit process facts for all transitions.
2. Processes never become canonical writers for Transformation repository elements/definitions.
3. Domain writes occur only via domain intents/commands (never direct DB writes; no “process-owned truth”).
4. At least one promoted BPMN ProcessDefinition version compiles to Workflow IR and executes deterministically in Temporal (runner + process facts).
5. At least one promoted BPMN ProcessDefinition version produces a deterministic scaffolding plan and fails verification when bindings are incomplete.
