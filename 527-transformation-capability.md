# 527 — Transformation Capability (Define the capability, then realize it via primitives)

> **DEPRECATED (2026-01-12):** This document assumes “Temporal-backed Process primitives” for BPMN-authored orchestration.  
> Current direction: BPMN-authored Process primitives execute on **Camunda 8 / Zeebe**, with side effects implemented by primitives as workers; Temporal is platform-only and not part of Ameide business capabilities.  
> See `backlog/527-transformation-capability-v2.md`.

**Status:** Draft (scaffolds implemented; spec acceptance pending)  
**Audience:** Architecture, platform engineering, operators/CLI, agent/runtime teams  
**Scope:** Define **Transformation** as a *business capability* implemented as Ameide primitives (Domain/Process/Projection/Integration/UISurface/Agent), with 496-native EDA contracts.

> Update (2026-01): the execution substrate sections in this doc (WorkRequests executed via KEDA-scaled Kubernetes Jobs; Telepresence-era dev workflows) are superseded as the platform default by:
> - `backlog/651-agentic-coding-ameide-coding-agent.md` (Coder-backed tasks + Camunda orchestration)
> - `backlog/653-agentic-coding-test-automation.md` (no Telepresence; preview env truth)
> - `backlog/654-agentic-coding-cli-surface.md` (CLI front doors and profiles)

This backlog is the capability definition counterpart to the method in `backlog/524-transformation-capability-decomposition.md`.

**Use with:**
- Capability definition template: `backlog/528-capability-definition.md`
- Capability worksheet: `backlog/530-ameide-capability-design-worksheet.md`
- Capability implementation DAG: `backlog/533-capability-implementation-playbook.md`
- ArchiMate vocabulary/verbs: `backlog/529-archimate-alignment-470plus.md`
- Integration/EDA contract rules: `backlog/496-eda-principles-v6.md` (Kubernetes standard), plus legacy notes in `backlog/496-eda-protobuf-ameide.md`
- Proto + semantic identity conventions: `backlog/509-proto-naming-conventions-v6.md`
- Primitives stack guardrails: `backlog/520-primitives-stack-v6.md`
- Legacy mapping (historical docs → target): `backlog/527-transformation-crossreference-303-ontology.md`

## Layer header (Strategy + Business, with Application realization)

- **Primary ArchiMate layer(s):** Strategy + Business (Capability, Value Streams, Business Processes) with Application realization via primitives.
- **Secondary layers referenced:** Technology (constraints only), Implementation & Migration (work packages/phases).
- **Primary element types used:** Capability, Value Stream, Business Process, Application Component, Application Service/Interface/Event, Work Package/Gap.
- **Prohibited unless qualified:** process, service, domain, event (qualify per `backlog/529-archimate-alignment-470plus.md`).

## Future state (executive summary)

Transformation is Ameide’s change-the-business capability. In the future state it provides:

- A tiny, ontology-agnostic Enterprise Repository substrate as the system of record.
  - **TBD (v6 memory model):** whether this is represented as “elements/versions” over Git files or a separate canonical write model. Until decided, treat “memory” as a projection concern.
- **Graph + vector search are read projections**, not separate canonical stores:
  - graph traversal/impact analysis is a projection over element links,
  - semantic search is a projection over element-version content embeddings (e.g., pgvector).
- Multi-methodology **experiences** (Scrum, TOGAF/ADM, PMI) delivered as **dedicated UISurfaces + workflows** over the same substrate (no methodology-specific canonical data model).
- Optional, when needed for configurability: a **Definition Registry** for promotable workflow/UI/validation definitions stored as domain data (still subject to the same EDA + promotion rules).
- v6 integration contracts (hybrid: RPC commands + event facts) so Process primitives and Agents can execute work deterministically without coupling to any single transport posture.

**Repository identity (fixed):**

- Scope identity is `{tenant_id, organization_id, repository_id}`.
- `repository_id` is the only repository identifier exposed in contracts and APIs (no separate `graph_id`).

## Layered big picture (ArchiMate-aligned; agentic delivery/coding)

This capability is Strategy/Business-led and realized via Application/Technology layers. For agentic delivery/coding, the topics stack as:

- **Strategy/Business (what/why + governance):** delivery roles (Product Owner, Solution Architect), approval gates, promotion policy, and evidence requirements; profiles (Scrum/TOGAF/PMI) map their accountabilities onto the same role taxonomy.
- **Application (how it is executed):** primitives as components/services/events — Domain as system-of-record + facts, Process as orchestration + gate decisions + process facts, Agent as role-based assistants (LangGraph DAGs), Integration as runners/connectors returning evidence, UISurface as thin UI over projections.
- **Technology (where it runs):** execution substrates (devcontainer, CI/in-cluster runners), fixed toolchain versions, secrets/network isolation, and observability; these are swappable but must always produce auditable evidence.

## 0) Problem

Today “Transformation” exists as:

- a legacy `TransformationService` CRUD surface (proto exists; legacy `services/transformation` implementation has been deleted),
- an emerging Scrum contract (`ameide_core_proto.transformation.scrum.v1` protos exist), but under the `303/527` target posture Scrum is modeled as an **ontology over elements** (type_key namespace + UISurface/workflows/projections), not as separate canonical tables,
- a target architecture intent across backlogs (Transformation stores definitions; Process operator fetches definitions; runtime is bus-only),

…but we lack a single authoritative backlog that says:

> “Transformation is a capability. These are its domains/processes/surfaces/integrations, and these are its EDA contracts.”

## 0.1) Implementation progress (repo snapshot; checklists)

This section is the current repo snapshot of **what is implemented vs what is still only specified**.

### Lock-ins (canonical target-state decisions)

- [x] Canonical storage is **elements-only** (no canonical “artifact” noun; “artifact” is UX vocabulary only).
- [x] Scope identity is required everywhere: `{tenant_id, organization_id, repository_id}`.
- [x] `repository_id` is the only repository identifier in contracts/APIs (no separate `graph_id`).
- [x] Canonical store is **relational Postgres**; graph traversal and vector search are **projection concerns**.
- [x] Legacy backends are removed; all flows must go through primitives (see “Deletion”).
- [x] Work execution scales only on dedicated work-queue topics (`transformation.work.queue.toolrun.verify.v1`, `transformation.work.queue.toolrun.generate.v1`, `transformation.work.queue.agentwork.coder.v1`; recommended new for the UI harness verification suite: `transformation.work.queue.toolrun.verify.ui_harness.v1`), not on canonical domain fact streams.

### Transformation Knowledge (Domain + Projection + UISurface wiring)

- [x] Domain primitive implemented: `primitives/domain/transformation`.
  - [x] Postgres migrations: outbox + enterprise repository substrate.
  - [x] Commands implemented for repositories/nodes/assignments/elements/relationships.
  - [x] Transactional outbox emits `transformation.knowledge.domain.facts.v1` after persistence.
  - [x] Gate: `go run ./packages/ameide_core_cli/cmd/ameide primitive verify --kind domain --name transformation --mode repo` passes.
- [x] Projection primitive implemented: `primitives/projection/transformation`.
  - [x] Idempotent ingestion via `projection_inbox(tenant_id,message_id)`.
  - [x] Materialized read model tables for repositories/nodes/assignments/elements/relationships/versions.
  - [x] Query services implemented (MVP browse/list/get).
  - [x] Gate: `go run ./packages/ameide_core_cli/cmd/ameide primitive verify --kind projection --name transformation --mode repo` passes.
- [x] Ingestion loop exists (bridge mode; Kafka is the normative transport):
  - [x] `primitives/projection/transformation/cmd/relay` tails the Domain outbox and applies facts to the projection store with durable offsets.
  - [ ] Replace bridge mode with Kafka consumers per topic family once the broker is the system-of-record transport for runtime seams.
- [x] WWW portal wired to primitives (MVP “repository editing/viewing” slice):
  - [x] `services/www_ameide_platform` reads via `TransformationKnowledgeQueryService` and writes via `TransformationKnowledgeCommandService`.
  - [x] Existing ArchiMate editor is wired end-to-end via primitives (create/edit views; create elements/relationships; persist layout).
  - [x] MVP-excluded “Transformations/initiatives” UI/routes removed.
  - [x] Gate: `pnpm -C services/www_ameide_platform run typecheck` passes.

### Primitives scaffolds (beyond MVP slice)

- [x] Process primitive scaffold exists: `primitives/process/transformation` (BPMN/DefinitionRegistry execution pending).
  - [x] Initial workflow logic exists (v0 hard-coded workflows): R2R governance (Scrum + TOGAF ADM) over WorkRequests + versioned anchor relationships.
  - [x] Gate: `go run ./packages/ameide_core_cli/cmd/ameide primitive verify --kind process --name transformation --mode repo` passes (GitOps TODOs remain).
- [x] Agent primitive scaffold exists: `primitives/agent/transformation` (role definitions/tool grants pending).
  - [x] Gate: `go run ./packages/ameide_core_cli/cmd/ameide primitive verify --kind agent --name transformation --mode repo` passes (GitOps TODOs remain).
- [x] UISurface primitive scaffold exists: `primitives/uisurface/transformation` (placeholder; MVP portal is `services/www_ameide_platform`).
  - [x] Gate: `go run ./packages/ameide_core_cli/cmd/ameide primitive verify --kind uisurface --name transformation --mode repo` passes (GitOps TODOs remain).
- [x] Integration primitive scaffold exists: `primitives/integration/transformation-mcp-adapter` (connectors/runners pending).

### Deletion (remove old/new soup)

- [x] Deleted legacy services:
  - [x] `services/graph`
  - [x] `services/repository`
  - [x] `services/transformation`
- [x] Workspace tooling updated (remove deleted workspaces + build wiring).

### Still pending (capability meaning beyond MVP)

- [ ] Definition Registry (schema-backed definitions + versions/promotions + operator consumption).
- [ ] Governance objects (initiatives, baselines/promotions, approvals, evidence bundles) and their UIs/projections.
- [ ] NotationProfileDefinition / conformance + exporters as data-driven definitions (standards-compliant vs extended).
- [ ] Stored ProcessDefinitions (BPMN) compiled to `CompiledWorkflowDefinition` + `ScaffoldingPlanDefinition`, executed by the Process primitive.
- [ ] Integration “runner” primitives for tool execution + evidence capture (CLI as tool, not orchestrator).

## 1) Definition

Transformation is the platform’s **change-the-business capability**: it captures what should be built, how it should be built, and how it is governed.

It is not “a modeling UI” and not “a Temporal service”; it is a capability realized by primitives that provide:

- the Enterprise Repository (typed design-time system of record),
- the Definition Registry (design-time definitions as domain data),
- EDA-native contracts that processes and agents consume to execute work deterministically.

### 1.0.1 IT4IT alignment (default operating model)

Transformation is **IT4IT-aligned by default**: it is Ameide’s “IT value-stream capability”.

- **IT4IT systems of record** map to Transformation **Domain primitives** (Enterprise Repository / Enterprise Knowledge substrate, baselines/promotions/approvals, Definition Registry).
- **IT4IT processes** map to Transformation **Process primitives** (Temporal-backed workflows that execute “Initiate/Design/Realize/Govern” and emit process facts as evidence).
  - **ProcessDefinitions are design-time truth** stored/promoted in the Definition Registry (Domain).
  - **Process execution is runtime orchestration** performed by Process primitives; process state is not a system of record.
- **IT4IT visibility** maps to Transformation **Projection primitives** (query services + audit/evidence read models).

IT4IT is treated as the *operating model* of the capability, not as a required notation. (Optional export/mapping can exist, but canonical storage remains element-centric.)

### 1.0.2 Transformation as IT value-stream execution (hard boundaries)

- **System of record:** Transformation Domain (Enterprise Repository + Definition Registry + baselines/promotions/approvals).
- **Process of record:** Transformation Process (executes promoted ProcessDefinitions; emits process facts; never writes canonical domain state).
- **Read model of record:** Transformation Projection (query services for UI/agents; audit trail; evidence views).
- **Contract spine:** topic families + envelope invariants (`tenant_id`/`organization_id`/`repository_id`, traceability, monotonic versions).
- **Tool/agent access:** Integrations (incl. MCP) are compatibility interfaces; authorization/approvals/evidence are enforced by Domain/Projection.

### 1.0.3 BPMN as executable + scaffolding driver (one source of truth)

Transformation treats **BPMN 2.0 ProcessDefinitions** as the canonical *authoring source* for workflow/governance logic, while keeping execution and generation deterministic:

- **BPMN is the source of truth** (portable under standards-compliant profiles).
- Promotion gates derive two promotable, deterministic build inputs:
  - `CompiledWorkflowDefinition` — Workflow IR executed by Temporal-backed Process primitives.
  - `ScaffoldingPlanDefinition` — diff-friendly plan used by scaffolders/operators (ports, workers, adapters, required contracts).
- **Standards-compliant vs extended profiles** govern *where bindings live*:
  - standards-compliant: bindings via `bpmn:documentation` (structured) and/or linked elements (REFERENCE)
  - extended: bindings may use extensions/namespaces but portability must be explicitly labeled

This preserves “one source” as the canonical truth while preventing runtime ambiguity (“BPMN XML is not the runtime program”).

### 1.0.4 Any notation as a derivation source (not just “diagrams”)

The same “authoring source → derived, promotable definitions” pattern applies to **any notation profile** represented in the Enterprise Knowledge substrate — but with an important separation:

- **Capability semantics live as architecture-level elements/definitions** (capabilities, value streams, nouns, contracts, decomposition). These are typically authored in architecture-oriented notations (e.g., ArchiMate) and stored as Elements.
- **Workflow/execution semantics live as ProcessDefinitions** (BPMN) and drive Process primitives.

- A model/view/document can be marked **normative** (profile-governed) and treated as a design-time source for:
  - `CapabilityDefinition`
  - `CapabilityEDAContract` (topics, envelopes, intent/fact catalogs)
  - `CapabilityPrimitiveDecomposition`
  - derived generation plans that drive scaffolding/verification (when applicable to that notation/profile)
- Promotion gates can derive/update these definitions deterministically from **promoted** source versions, with full traceability back to `{element_id, version_id}`.

This keeps the Enterprise Knowledge substrate element-centric while letting any notation drive downstream execution and delivery without turning any notation into a parallel canonical storage model.

### 1.0.5 From design-time artifacts to IT4IT steps to primitives (the execution chain)

Transformation is “definition-driven”: **design-time artifacts** (stored as Elements and/or Definition Registry entries) drive **IT4IT-aligned process steps** (stored ProcessDefinitions), which in turn drive the creation and evolution of **Ameide primitives**.

The chain is:

1. **Design-time artifacts + notations** (canonical, versioned/promotable)
   - Capability semantics: architecture-level elements/definitions (often ArchiMate, but any notation/profile can contribute)
     - `CapabilityDefinition`
     - `CapabilityEDAContract`
     - `CapabilityPrimitiveDecomposition`
   - Workflow semantics: ProcessDefinitions (BPMN)
     - BPMN is the authoring source; promotion yields `CompiledWorkflowDefinition` + `ScaffoldingPlanDefinition`.

2. **IT4IT process steps (Process primitive executes)**
   - Initiate/Design/Realize/Govern are realized as stored ProcessDefinitions and executed by the Process primitive (process-of-record).
   - Each step has explicit inputs/outputs and emits process facts as evidence.

3. **Tool runners (Integration executes; CLI is a tool)**
   - Some steps require deterministic tool execution (existing CLI scaffolding, `buf generate`, `ameide primitive verify`, tests/build/publish).
   - Tool/agent execution is requested via a **WorkRequest** recorded in Domain (see “Execution substrate” below). In v1, WorkRequests are executed via **KEDA ScaledJob → Kubernetes Job per execution intent**, using a dedicated **executor** image (not the workbench/devcontainer): Domain emits `WorkExecutionRequested` execution intents after persistence onto the queue topics; Jobs consume those intents, execute, and record evidence back into Domain idempotently; Domain emits facts for the audit trail; Activities wait for completion (poll/heartbeat) and return results to workflows; Process emits process facts for the audit timeline. For parity/debugging, a separate long-lived workbench runs the devcontainer toolchain image (human attach/exec only; never consumes the queue).
   - Outputs are captured as evidence bundles (attachments + structured summaries) and referenced from promotions/baselines and process facts.

4. **Ameide primitives (realization outputs)**
   - `CapabilityPrimitiveDecomposition` + scaffolding plans drive repo-owned skeletons for Domain/Process/Projection/Integration/UISurface/Agent primitives.
   - Contracts (proto) drive generated SDKs and service stubs; verification gates ensure determinism and prevent drift.

This is how we avoid a “mega CLI”: the **Process primitive is the orchestrator**, Integrations run tools, Domains persist truth + emit facts, and Projections provide the read path.

## 1.1) Strategy: outcomes and value streams (starter set)

Primary outcomes (future state):

- Tenants can run transformation initiatives using a repeatable methodology.
- Design-time truth (ArchiMate/BPMN/definitions) is governable, versioned, promotable, and queryable.
- Runtime execution is deterministic: Process primitives and Agents drive work from contracts and facts, not ad-hoc RPC calls.

Value streams (refine via `backlog/524-transformation-capability-decomposition.md` Step 1):

- **Initiate**: create initiative → lock glossary/identity axes → establish repository structure.
- **Design**: model capability/value streams → derive contracts/topology → propose proto packages/topics → promote baselines.
- **Realize**: decompose to primitives → scaffold → verify → promote definitions.
- **Govern**: run methodology gates (ADM phases, Scrum timeboxes, PMI stage gates) → manage drift and standards compliance (when required).

## 2) Core invariants (non-negotiable)

1. Capability is realized by primitives (no monolith).
2. Design-time vs runtime separation (design-time elements/definitions stored in Transformation; runtime workflows execute in Process primitives).
3. 496-native EDA (intents vs facts, outbox for writers, idempotent consumers, sagas in Process primitives).
4. No runtime state coupling (workflows never rely on synchronous cross-primitive state coupling; side effects happen in Activities with explicit idempotency).
5. Tenant isolation and traceability metadata on all messages.
6. Typed design-time store (ArchiMate/BPMN/definitions as domain data; file formats are attachments only).
7. Multi-methodology by profile (Scrum/TOGAF/PMI as profiles).
8. Profiles govern; they don’t redefine the repository (no forked canonical models).
9. Metamodel conformance is profile-driven: “standards-compliant” vs “extended” notation profiles constrain `type_key` and exports; canonical storage remains element-centric.

## 2.1 Execution substrate (normative): WorkRequest → evidence → timelines

To make “ephemeral jobs from a queue” safe and interoperable without turning KEDA (or a CLI wrapper) into the orchestrator, v1 standardizes a single seam:

1. **Process decides** a tool/agent step is needed (policy + orchestration).
2. **Process requests execution** via a **Domain intent** that creates a `WorkRequest` record (idempotent).
3. **Domain persists** the WorkRequest and emits a **domain fact** (`WorkRequested`) after commit (outbox) on `transformation.work.domain.facts.v1`.
4. **Execution backend runs**: Domain also emits an execution intent (`WorkExecutionRequested`) after commit onto a dedicated execution queue topic; KEDA ScaledJob schedules in-cluster ephemeral Jobs based on Kafka consumer group lag on the execution queue topics (KEDA schedules; the Job consumes):
   - it executes exactly what was requested (no extra policy),
   - writes artifacts to object storage,
   - records outcomes back into Domain via a Domain intent (idempotent).
5. **Domain emits** `WorkCompleted`/`WorkFailed` (and links evidence bundles).
6. **Activity waits** for completion (poll/timeout/heartbeat) and returns a result; **Process emits** `ToolRunRecorded` / `GateDecisionRecorded` / `ActivityTransitioned` as process facts, and continues.
7. **Projection materializes** the joined run timeline (process facts + correlated domain facts) with citations.

### Kafka transport (normative)

Kafka is the event broker for runtime seams. In the execution substrate:

- `WorkExecutionRequested` is published to **dedicated Kafka execution queue topics** so KEDA can scale on lag without consuming domain fact streams.
- KEDA scales **by consumer group lag**; it does not “hand a specific message to a pod”. The Job/pod is the Kafka consumer.
- A WorkRequest processor MUST treat Kafka as **at-least-once** delivery:
  - disable auto-commit,
  - record outcomes/evidence durably in Domain,
  - only then commit offsets.

Hard rules (v1):

- Queue-triggered execution MUST consume **execution intents** (`WorkExecutionRequested`) explicitly requested by Process/Domain, not raw external webhooks/events and not domain fact streams.
- Job TTL/cleanup is hygiene only; audit correctness comes from Domain-persisted evidence + process facts + projection citations.
- A debug/admin “workbench” pod MUST exist in `local` and `dev` for human attach/exec and reproduction; it MUST NOT process WorkRequests and MUST NOT be deployed in `staging`/`production`. Workbench is an admin/debug surface and must not be conflated with external parallel dev “agent slots” (`agent-01`, `agent-02`, `agent-03`) described in `backlog/581-parallel-ai-devcontainers.md`.

Queue topics (v1; dedicated per class of work to avoid scaling on fact streams):

- `transformation.work.queue.toolrun.verify.v1`
- `transformation.work.queue.toolrun.generate.v1`
- `transformation.work.queue.agentwork.coder.v1`
- recommended (new): `transformation.work.queue.toolrun.verify.ui_harness.v1` for the non-agentic UI harness verification suite (cluster-only; stable URLs; Gateway API overlays).

Implementation status (GitOps; `ameide-gitops`):

- KEDA is installed cluster-scoped (see `backlog/585-keda.md`).
- Work-queue topics are provisioned in `local` and `dev` via `data-kafka-workrequests-topics` (disabled elsewhere).
- A debug/admin workbench + ExternalSecrets contract is deployed in `local` and `dev` via `workrequests-runner` (disabled elsewhere).
- KEDA `ScaledJob` resources are scaffolded but intentionally disabled until a real WorkRequest consumer exists (to prevent runaway failing Jobs).

## 3) Capability boundaries and nouns

Transformation capability owns (at minimum):

- Transformation initiative
- Enterprise Repository workspace tree
- Enterprise Knowledge graph (element-centric substrate): elements, relationships, versions, assignments
- Notation (metamodel) profiles:
  - standards-compliant profiles (e.g., ArchiMate 3.2, BPMN 2.0) that constrain `type_key` to the standard namespaces and enable standard exports
  - extended profiles (Ameide/vendor extensions) that allow additional `type_key` namespaces (e.g., `ameide:*`) and define their own export targets
  - delivered as profile definitions (at minimum: ArchiMate, BPMN, documents/markdown)
- Methodology overlays (Scrum, TOGAF/ADM, PMI) as UI + validation + workflow definitions over the same element graph
- Platform definitions (ProcessDefinition, AgentDefinition, ExtensionDefinition, etc.)

Rule of thumb:

- Transformation (Enterprise Knowledge + governance) is canonical; projections are read-optimized indexes/materializations.
- Other capabilities may reference Transformation repository elements/definitions by stable IDs; they do not become writers of those elements.

**Profile taxonomy (avoid ambiguity):**

- **Notation (metamodel) profiles** govern *what kinds of elements/relationships are valid* and how diagrams/models are validated/exported (ArchiMate/BPMN/etc.).
- **Methodology overlays** govern *how work is executed* (ProcessDefinitions + gates) and *what “compliance” means* (checks/validators + UI views), without forking the canonical repository model.
- **Kubernetes CRDs/operators** govern runtime wiring only; they MUST NOT encode methodology/business semantics. Methodology meaning is stored as domain data (Definitions + Elements) and enforced via validations + gates.

## 3.1) Cluster E2E harness (non-agentic; stable URLs; no GitOps churn)

Transformation’s v1 posture separates **agentic development** from **non-agentic UI harness verification**:

- **Dev WorkRequests (agent-capable):** interactive, human-in-the-loop development in an in-cluster devcontainer toolchain, producing repo changes + repo-mode verification evidence.
- **UI harness WorkRequests (non-agentic):** fully programmatic “build → run shadow → route overlay → run Playwright → collect artifacts → cleanup” against **stable environment URLs** (e.g., `https://platform.local.ameide.io`), without per-run GitOps/Argo changes.

To keep stable URLs unchanged while routing only test traffic to the modified service versions, the platform uses the gateway as the “intercept plane”:

- Create ephemeral **shadow** Deployments/Services for the changed services (no changes to baseline Deployments).
- Create ephemeral **Gateway API** overlay routes (e.g., `HTTPRoute`) matching a run-scoped header (recommended: `X-Ameide-Run-Key=<nonce>`) that routes only that run’s requests to the shadow Services.
- A verification suite (recommended: `transformation.verify.ui_harness.gateway_overlay.v1`) drives the harness behavior; WorkRequests keep `action_kind=verify` and select the harness via `verification_suite_ref`. All other traffic continues to hit baseline services.
- RBAC posture: the UI harness executor must use least-privilege identities and bind-only delegation to predeclared roles (avoid ad-hoc Role creation that grants new privileges).

Telepresence remains a developer ergonomics tool for interactive remote-first workflows; it is not required for the v1 in-cluster E2E harness.

## 4) Application realization (primitives)

- Domain primitives: `backlog/527-transformation-domain.md`
- Process primitives: `backlog/527-transformation-process.md`
- Projection primitives: `backlog/527-transformation-projection.md`
- UISurface primitives: `backlog/527-transformation-uisurface.md`
- Agent primitives: `backlog/527-transformation-agent.md`
- Integration primitives: `backlog/527-transformation-integration.md`
- Proto contracts: `backlog/527-transformation-proto.md`
- Methodology dictionary: `backlog/527-transformation-methodology-dictionary.md`
- Scenarios (flow-only): `backlog/527-transformation-scenario-scrum.md`, `backlog/527-transformation-scenario-togaf-adm.md`
- Overlay implementation trackers: `backlog/527-transformation-implementation-migration-scrum.md`, `backlog/527-transformation-implementation-migration-togaf-adm.md`
- Implementation & migration: `backlog/527-transformation-implementation-migration.md`

## 4.1) Application Interfaces for Agents (MCP)

Transformation is the reference implementation for MCP-based agentic access.

References:
- MCP adapter pattern: `backlog/534-mcp-protocol-adapter.md`
- Read optimizations (semantic search): `backlog/535-mcp-read-optimizations.md`
- Write optimizations (agent-friendly commands): `backlog/536-mcp-write-optimizations.md`

**Published interface**
- MCP tools: yes
- MCP resources: yes
- MCP prompts: optional (defer until prompts are versioned and promotable)

**Integration primitive**
- `transformation-mcp-adapter` (Integration primitive) translates MCP (stdio + Streamable HTTP) into proto-first Application Services.
- Tool schemas are generated from proto annotations (default deny; explicit allowlist).

Starter tool map (illustrative; authoritative source is generated from proto):

| Tool | Kind | Application Service (canonical) | Primitive | Approval | Evidence / audit |
|------|------|----------------------------------|----------|----------|------------------|
| `transformation.listElements` | query | `EnterpriseKnowledgeQueryService.ListElements` | Projection | no | audit log + query tracing |
| `transformation.getView` | query | `EnterpriseKnowledgeQueryService.GetElement` (view kind) | Projection | no | audit log + query tracing |
| `transformation.semanticSearch` | query | `SemanticQueryService.Search` | Projection | no | audit log + query tracing |
| `transformation.submitRepositoryIntent` | command | `EnterpriseKnowledgeWriteService.SubmitIntent` | Domain | yes | domain facts + audit trail projection |
| `transformation.createBaseline` | command | `BaselineWriteService.CreateBaselineDraft` | Domain | yes | baseline facts + audit trail projection |

Starter resources (illustrative):

| Resource URI pattern | Backing query service | Primitive |
|----------------------|-----------------------|----------|
| `transformation://model/{id}` | `EnterpriseKnowledgeQueryService.GetElement` | Projection |
| `transformation://element/{id}` | `EnterpriseKnowledgeQueryService.GetElement` | Projection |
| `transformation://view/{id}` | `EnterpriseKnowledgeQueryService.GetElement` | Projection |

Edge constraints (Transformation):

- Writes require human approval by default; approvals are recorded as domain/process facts and visible in the audit timeline.
- Evidence is emitted as domain facts (after persistence) and surfaced via projection-backed audit trail.

**Proto contract references:**
- Domain facts (Enterprise Knowledge substrate): `transformation.knowledge.domain.facts.v1`
- Process facts: `transformation.process.facts.v1`
- RPC services/packages: `ameide_core_proto.transformation.knowledge.v1`, `ameide_core_proto.process.transformation.v1`
- Full contract catalog: [527-transformation-proto.md](527-transformation-proto.md)

**MCP Security Spine (invariants):**

| Invariant | Requirement |
|-----------|-------------|
| **OAuth 2.1 RS** | Streamable HTTP MCP adapter MUST be an OAuth 2.1 Resource Server. Keycloak is the Authorization Server. |
| **Origin validation** | Adapter MUST validate `Origin` header for browser-based MCP clients (CORS policy). |
| **Protocol ≠ privilege** | MCP is a protocol binding, not a privilege boundary. Authorization decisions are enforced by Domain/Projection services, not the adapter. |
| **Tenant isolation** | All tool calls MUST pass through tenant context from access token. Adapter never allows cross-tenant access. |
| **Scopes** | Required scopes: `mcp:tools`, `mcp:resources`, `openid`, `profile`. Additional capability-specific scopes as needed. |

See [534-mcp-protocol-adapter.md](534-mcp-protocol-adapter.md) §5 for full OAuth 2.1 specification.

**Internal vs external agent transport:**

| Agent Type | Transport | Rationale |
|------------|-----------|-----------|
| Platform agents (AmeidePO, AmeideSA, AmeideCoder) | SDK/gRPC | Typed, efficient, no JSON-RPC overhead |
| External AI tools (Claude Code, Cursor, Copilot) | MCP (Streamable HTTP) | Compatibility binding for devtools |

MCP is a **compatibility interface**, not the canonical tool runtime. Tool definitions live in proto with `(ameide.mcp.expose)` annotations; MCP schemas are generated, never hand-authored.

**Event-driven first handover (agents + tools):**

- The canonical integration posture for **handover** (Product Owner→Solution Architect→Executor) is **messages/events**, not direct RPC chaining. Process primitives and LangGraph DAGs remain the orchestration layers; eventing is the integration boundary.
- For agentic coding specifically, align to `backlog/505-agent-developer-v2.md`: work delegation is bus-native; interactive transports (e.g., A2A) are optional bindings over the same semantics, not parallel state machines.

**Agentic coding guardrails: enforcement plane vs evidence plane:**

- **Git provider (e.g., GitHub) is the hard enforcement plane (“what is possible”):** protected branches, rulesets, push rulesets (path restrictions), merge queue; agent identities must be unable to push/merge outside their declared namespace even if they branch-switch locally.
- **Transformation is the policy + orchestration + evidence plane (“what is allowed”):** AgentDefinitions (tool grants + risk tiers), approval policies, and required evidence bundles; Process gates consume runner evidence and record decisions as facts for audit/replay.

**Interoperability posture (CloudEvents):**

- For inter-primitive (microservice-to-microservice) traffic in Kubernetes, CloudEvents + Protobuf is the canonical contract per `backlog/496-eda-principles-v6.md`.
- Integration adapters MUST map external messages into the same CloudEvents envelope used in-cluster; any “binding definition” is a versioned mapping document, not a separate semantic standard.

**Definitions to prevent drift (v1):**

- `CloudEventsBindingDefinition`: the canonical→CloudEvents mapping version (attributes + required extensions) used by Integration adapters.
- `GitProviderPolicyDefinition`: branch namespace patterns, protected branch posture, push ruleset path blocks, required checks; used by Process gates and recorded in evidence snapshots.
- `AgentPermissionMatrixDefinition`: who can do what (push, open PR, merge, approve, bypass) across agent roles and human roles; used to enforce separation-of-duties in workflows.

**Agentic coding guardrails + execution environments:**

- Guardrails must be expressed as evidence-producing gates (verify/test/lint/codegen drift) so promotions can be policy-driven and auditable (see `backlog/527-transformation-integration.md` runner evidence contract).
- Execution environments should be explicit and swappable:
  - interactive devcontainer (human/agent in loop; fixed tooling versions, e.g. Codex CLI version lock in `backlog/433-codex-cli-057.md`; developer workflow in `backlog/435-remote-first-development.md`; parallel agent slots in `backlog/581-parallel-ai-devcontainers.md`),
  - KEDA-scheduled in-cluster Jobs (executor image; deterministic tool runs, structured evidence),
  - local dev (human) as a convenience, never a promotion authority without evidence capture.

## 4.2) Clarification requests (next steps)

Confirm/decide:

- Which Transformation proto surface is canonical going forward: legacy `ameide_core_proto.transformation.v1` RPC façade vs bus-native intents/facts packages (and the planned deprecation window).
- Which single end-to-end “acceptance slice” is the v1 target (Scrum governance? baseline promotion? Architecture revisions?) and what facts/process facts prove it.
- Whether proposal/draft-as-default writes are required for all agent-driven mutations (and which exceptions exist), and where the approval gates live (Domain vs Process).
- The minimum “agent-grade memory read surface” requirements for v1 (read_context + citations + audit replay), and which query services own them.

## 5) Acceptance criteria

1. Transformation is described and operated as a capability realized via primitives, not as a single service.
2. Core Transformation EDA contracts exist (topic families + envelopes) aligned with the Scrum pattern.
3. Migration stance is explicit: what becomes canonical writer and what becomes projection/facade.
4. At least one end-to-end slice proves the seam for **both** governance workflows (Scrum + TOGAF ADM) over the same element substrate (Domain intents → domain facts/outbox → Process reacts → process facts → Projection reads).
5. Verification is **workflow-driven background work**: Process requests `action_kind=verify` WorkRequests and gates on their outcomes; UI harness verification is selected by `verification_suite_ref` and runs against stable URLs via Gateway API overlays (430-aligned), not by manual UI actions.
