# 527 Transformation - E2E Execution Sequence (Kanban-ready; Process vs Events vs Infra)

**Status:** Draft (normative intent; implementation exists for Transformation work execution)  
**Parent:** `backlog/527-transformation-capability.md`  
**Related:** `backlog/520-primitives-stack-v2.md`, `backlog/496-eda-principles.md`, `backlog/509-proto-naming-conventions.md`, `backlog/527-transformation-process.md`, `backlog/527-transformation-domain.md`, `backlog/527-transformation-integration.md`

This backlog rewrites the end-to-end "work execution" story as a clean separation:

- **Process** decides what happens next (orchestration only; Temporal workflows are deterministic).
- **Events** carry **intents** (requests) and **facts** (audit/outcomes) with strict 496 semantics.
- **Infra** decides how work runs (KEDA/Jobs/worker pools/external executors), without leaking into process semantics.
- **UI** shows a projection-driven timeline/kanban (facts + process facts + evidence refs), not infra internals.

Terminology (canonical in `backlog/520-primitives-stack-v2.md`):

- **Activity-inline execution:** work runs inside the Process primitive's Temporal Activity worker.
- **Activity-delegated execution:** a Process Activity initiates work but another runtime executes it; delegation is via an **intent**, completion is observed via **facts**.

---

## Sequence V3 - Value-stream view (R2D first)

V3 restates the same execution seam as Sequence A, but in value-stream terms so it can align to IT4IT-style naming (and your spreadsheet-style “phase / subphase” breakdown).

**Kanban mapping (recommended):**

- **A (Requirements management)** → `intake` / `triage` / `awaiting_approval` (often no Temporal yet).
- **B (Design/build/test/package)** → `execution` (Temporal workflow + WorkRequests).
- **C (Acceptance / preview)** → `execution` or `release` (depends on whether preview is a release gate).
- **D (Release)** → `release` → `done|failed`.

### Requirement to Deploy (R2D) — “Ship”

| Process | Phase nr | Phase | SubPhase | Description | Implemented as | Input | Output | Proto (write surface) | Events (facts vs intents) | Temporal Activity | UI/Kanban |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| R2D | A | Requirements management | A1 Requirement development | Stabilize/clarify the requirement and supporting artifacts (markdown/BPMN/etc) with chat + element-editor assistance. | Platform app element editor + chat + LangGraph agent (Agent primitive). Temporal is optional until you want a durable run/timeline. | User chat + element edits. | Updated requirement Elements/ElementVersions and links (the “source of truth” artifacts). | `ameide_core_proto.transformation.knowledge.v1.TransformationKnowledgeCommandService` (`CreateElement`/`EditElement`/`CreateRelationship`/etc). | **Facts:** `ameide_core_proto.transformation.knowledge.v1.TransformationKnowledgeDomainFact` on `transformation.knowledge.domain.facts.v1`.<br>**No intents:** no `WorkExecutionRequested` queues unless you explicitly request delegated work. | None required. | Typically `intake` → `triage` (optionally show as “drafting” without a Process run). |
| R2D | A | Requirements management | A2 Approval gate (optional) | Decide that the requirement slice is “ready to deploy” (DoR) and freeze a promotable set of versions. | Process gate (Temporal Update) if you started a run; otherwise a UI action that creates/submits a baseline. | Requirement element refs. | Baseline submitted/approved (and any required decision evidence). | `ameide_core_proto.transformation.governance.v1.TransformationGovernanceCommandService` (`CreateBaseline`/`SubmitBaseline`/`ApproveBaseline`/etc). | **Facts:** `ameide_core_proto.transformation.governance.v1.TransformationGovernanceDomainFact` on `transformation.governance.domain.facts.v1`.<br>**Process facts (if Temporal):** `ameide_core_proto.process.transformation.v1.TransformationProcessFact` on `transformation.process.facts.v1` (`Awaiting(kind=approval)`, `GateDecisionRecorded`). | `EmitPhaseEntered(awaiting_approval)` + `AwaitDecision` (Update/Signal). | `awaiting_approval` (card blocks until approved). |
| R2D | B | Design, build, test, package | B1 Code | Produce the code change that fulfills the requirement slice (repo change + PR). | Temporal workflow orchestrates; code runs as delegated `WorkRequest` (agentic) or tool-run steps. | Approved baseline + repo checkout ref (commit SHA). | `WorkRequest` created; PR link or patch recorded as evidence. | `ameide_core_proto.transformation.work.v1.TransformationWorkCommandService.RequestWork` with `spec.work_kind=WORK_KIND_AGENT_WORK`, `spec.action_kind=ACTION_KIND_PUBLISH`. | **Intent:** `ameide_core_proto.transformation.work.v1.WorkExecutionRequested` to `transformation.work.queue.agentwork.*.v1`.<br>**Facts:** `ameide_core_proto.transformation.work.v1.TransformationWorkDomainFact` on `transformation.work.domain.facts.v1` (`WorkRequested/Started/Completed/Failed`).<br>**Process facts:** `ToolRunRecorded`/`ActivityTransitioned`/`PhaseEntered`. | `RequestWork` (side-effect boundary) + “await WorkCompleted/WorkFailed facts” (via SignalWithStart ingress). | `execution` (card shows PR + evidence refs). |
| R2D | B | Design, build, test, package | B2 Iterative tests | Run fast feedback loops (unit/integration suites) until green; may repeat with more code changes. | Temporal workflow loop + delegated tool-run WorkRequests. | Repo checkout ref (commit SHA), suite selection. | Verified evidence + pass/fail outcomes; optional additional code WorkRequests. | `ameide_core_proto.transformation.work.v1.TransformationWorkCommandService.RequestWork` with `spec.work_kind=WORK_KIND_TOOL_RUN`, `spec.action_kind=ACTION_KIND_VERIFY`. | **Intent:** `WorkExecutionRequested` to `transformation.work.queue.toolrun.verify.*.v1`.<br>**Facts:** `TransformationWorkDomainFact` on `transformation.work.domain.facts.v1` + `TransformationProcessFact.ToolRunRecorded`. | `RequestWork` + loop on work outcomes. | `execution` (card shows test evidence + failures driving next steps). |
| R2D | B | Design, build, test, package | B3 Final test | Run the required “final gate” suite(s) (e2e/smoke/security) and lock evidence for release decisioning. | Temporal workflow + delegated tool-run WorkRequests (may be a dedicated queue/class). | Repo checkout ref (candidate commit SHA), final suite refs. | Final verification evidence bundle + decision input. | Same as B2 (verify WorkRequest), possibly with a stricter `verification_suite_ref`/tool args. | Same as B2, plus explicit `Awaiting(kind=external_work, ref=work_request_id)` in process facts while waiting. | `RequestWork` + wait on completion facts. | `execution` (card shows “final gate” status). |
| R2D | C | Acceptance | Preview environment | Validate in a preview environment before release (optional posture). | Argo CD PR generator (infra) + Process waits on evidence. | PR / candidate commit + config refs. | Preview URL + acceptance decision evidence. | Integration-specific (Git provider + GitOps repo PR + ArgoCD), plus optional Process gate. | **Facts:** process facts to record phase/gate; domain facts for any persisted acceptance artifacts/evidence. (Preview infra details must not leak into process semantics.) | `EmitPhaseEntered(acceptance)` + `Awaiting(kind=dependency, ref=preview_env)` (if modeled). | `execution` or `release` (team choice); UI shows preview link + accept/reject. |
| R2D | D | Release |  | Promote/publish and roll out (digest-pinned), then close the run. | Temporal workflow orchestrates release gates; actual rollout is GitOps promotion + Argo sync health. | Approved changes + artifacts/digests + policy checks. | Promotion PRs + rollout evidence + terminal status. | Integration + governance write surfaces; process emits terminal facts. | **Facts:** process facts `PhaseEntered(release)` + `RunCompleted/RunFailed`; domain facts for any recorded release/promotions. | `EmitPhaseEntered(release)` + request publish/deploy WorkRequests (if used). | `release` → `done|failed` (UI shows promoted digests + rollout summary). |

### Other value streams (stubs)

These reuse the same execution seam (Workflow → Activity → Domain write → domain facts + process facts → UI/projection). The naming differs; the contracts and invariants do not.

- **Request to Fulfill (R2F) — “Deliver”**: customer fulfillment flows (often more integration-heavy than agent-work heavy).
- **Detect to Correct (D2C) — “Run”**: operational incident/change flows (signals + runbooks + verification WorkRequests).
- **Strategy to Portfolio (S2P) — “Plan”**: initiative/roadmap shaping flows (mostly Domain writes + approvals; fewer WorkRequests).

## Infra contract (agentic coding via Codex)

- The AgentWork "coder" executor expects authenticated Codex CLI via an `auth.json` file.
- In-cluster, infra provides this by mounting `Secret/codex-auth` (created by ExternalSecret) and setting `CODEX_HOME=/codex-home`.
  - An initContainer copies `/codex-secret/auth.json` -> `/codex-home/auth.json` (no secret bytes ever travel on the broker).
- If `codex-auth` is missing, the executor MUST fail fast with a clear error and record a `WorkFailed` outcome (Domain emits facts via outbox).

---

## Sequence A - Kanban progression (detailed)

Sequence A is expressed as a **Kanban progression** (phase-first progress), aligned with `backlog/520-primitives-stack-v2.md`:

- A projection can render a board/timeline without workflow-specific UI logic.
- Progress facts are **phase-first by default**; step-level progress is opt-in.
- The board scope is initiative + repository by default (initiative is the change-governance container; repository is the architecture context).
  - Repo roll-up boards are allowed, but the primary workspace view is `{repository_id, initiative_id}`.
- Progress/process facts that drive the board carry:
  - `process_instance_id` (Temporal Workflow Id),
  - `process_run_id` (Temporal Run Id),
  - stable ordering (`run_epoch`, `seq`) across `Continue-As-New`.
- Include `process_definition_id` in run start and progress facts (required for one Kanban contract everywhere; see `backlog/620-kanban-fullstack-implementation-plan.md` and `backlog/509-proto-naming-conventions.md`).

### Kanban columns (Transformation run board)

These columns are a recommended baseline for Transformation; they map to a `phase_key` (or equivalent) carried in process progress facts.

1. `intake` - question/need captured; no durable run required yet.
2. `triage` - context gathering + drafting of promotable artifacts.
3. `awaiting_approval` - a human/agent decision is required (BPMN user task).
4. `execution` - work is being executed (tool-run, agent-work, verification loops).
5. `release` - publish + GitOps promotion + rollout verification.
6. `done` / `failed` - terminal states.

The delegated execution seam is always the same:

- Workflow decides and stays deterministic.
- Workflow calls an Activity (the only side-effect boundary).
- Activity submits a Domain command/intent (e.g., `RequestWork`, idempotent).
- Domain persists and emits:
  - **Facts** (audit/outcomes) via outbox to the spine, and
  - **Execution intents** to point-to-point execution queues when delegation is required.
- Executor consumes execution intent, runs work, then calls the owning Domain write surface to record started/outcome (idempotent); Domain emits completion facts.
- Process ingress observes facts and signals the workflow (`SignalWithStart`), and the UI shows the timeline via projections.

**Hard rule:** executors (in-cluster or external) do not publish facts; they only call the owning Domain write surface; the Domain emits facts (outbox).

---

### intake -> triage (user chat + triage start)

This stage is interactive and mostly lives in the Agent primitive (LangGraph runtime). Temporal is optional until you want a durable run/timeline.

| Task | Process (Temporal) | Events (intent vs fact) | Infra (mechanics) | UI (what the user can see) |
| --- | --- | --- | --- | --- |
| Chat turn: architecture question -> answer | Optional: start a "triage run" workflow to capture a durable timeline; otherwise no workflow yet | none required | UISurface calls Agent primitive API (LangGraph) | Chat transcript + agent answer streamed; optional "run created" badge |
| Enter `triage` column | Workflow emits a phase/progress fact (idempotent Activity) | Fact: process progress `PhaseEntered(phase_key=triage)` | none | Kanban card appears in `triage` (links to artifacts/evidence) |
| Triage reads context | Workflow may allow read-only projection reads (declared, time-bounded) but does not mutate state | none | Agent reads Projection query surfaces via SDK | UI can show "sources used" (projection queries / evidence refs), not raw infra calls |
| Draft promotable artifacts | Workflow (if running) calls an Activity that requests a Domain write (proposal/draft) | Intent: Domain command to create/update draft artifacts<br>Facts: domain facts via outbox | Domain persists and emits facts via outbox | Draft artifact appears with version + status=draft |
| Deep analysis needed (repo/tool) | Workflow calls delegated Activity -> Domain `RequestWork(work_kind=AGENT_WORK|TOOL_RUN)` | Intent: execution intent on `transformation.work.queue.*.v1`<br>Facts: Work lifecycle on `transformation.work.domain.facts.v1` | Executor runs agent/tool work; records outcomes to Domain | UI shows "analysis requested/in progress/completed" + evidence links |

---

### triage -> awaiting_approval -> execution (stabilize + iterate)

This stage turns drafts into an approved slice, then runs execution loops (generate/implement/verify) until gates are satisfied.

| Task | Process (Temporal) | Events (intent vs fact) | Infra (mechanics) | UI (what the user can see) |
| --- | --- | --- | --- | --- |
| Enter `awaiting_approval` column | Workflow reaches a gate and emits progress fact; waits | Fact: `PhaseEntered(phase_key=awaiting_approval)` + optional `Awaiting(kind=approval, ref=...)` | none | UI shows "approval required" with context + decision actions |
| Approve / reject | Prefer Temporal Update (UpdateId=idempotency key) for BPMN user task approvals | Intent: approval decision submitted via Process write surface<br>Fact: gate decision recorded | none | UI shows decision recorded; card moves based on outcome |
| Enter `execution` column | Workflow emits progress fact (idempotent Activity) | Fact: `PhaseEntered(phase_key=execution)` | none | Kanban card moves to `execution` |
| Request scaffold/codegen | Workflow Activity calls Domain `RequestWork(work_kind=TOOL_RUN, action_kind=SCAFFOLD|GENERATE)` | Intent: execution intent to tool-run queue<br>Facts: Work lifecycle | Executor runs `ameide`/`buf generate`; records evidence | UI shows codegen status + evidence links |
| Request coding change (agentic) | Workflow Activity calls Domain `RequestWork(work_kind=AGENT_WORK, action_kind=PUBLISH)` | Intent: execution intent to agent-work queue<br>Facts: Work lifecycle | Executor runs coding agent with `CODEX_HOME` from `Secret/codex-auth`; records evidence (PR/patch/summary) | UI shows run status; on success shows PR link; on fail shows error + evidence |
| Request verification | Workflow Activity calls Domain `RequestWork(work_kind=TOOL_RUN, action_kind=VERIFY)` | Intent: execution intent to verify queue<br>Facts: Work lifecycle | Executor runs tests/verify suites; records evidence | UI shows verification pass/fail + logs |
| Iterate until green | Workflow loops on outcomes + gate rules (no infra knowledge) | Facts/receipts are the only completion signals | Infra can change without changing workflow semantics | UI shows timeline + "why" from facts/evidence (not infra retries/backoffs) |

---

### execution -> release -> done (publish + promote)

Release is still a Process concern (gates + sequencing), but what gets deployed is owned by GitOps promotion and digest-pinned images.

| Task | Process (Temporal) | Events (intent vs fact) | Infra (mechanics) | UI (what the user can see) |
| --- | --- | --- | --- | --- |
| Enter `release` column | Workflow emits progress fact; may require explicit approval (policy) | Fact: `PhaseEntered(phase_key=release)` + optional `Awaiting(kind=release_approval, ref=...)` | none | UI shows "ready to release" gate + required checks |
| Build/publish artifacts | Workflow Activity requests publish step (or triggers CI via Integration) | Intent: execution intent for publish class<br>Facts: Work lifecycle | CI/build system produces digest-pinned images (see `backlog/602-image-pull-policy.md`) | UI shows build status + produced digest(s) + evidence (logs) |
| Promote environments | Workflow waits on promotion evidence | Facts: release/promote outcomes emitted by owning domain(s); projections show rollout view | GitOps promotion is a Git change per `backlog/611-trunk-based-main-and-gitops-environment-promotion.md` | UI shows environment promotion timeline + PR links + rollout summary |
| Close the run | Workflow emits final progress fact and stops | Fact: terminal process fact (`RunCompleted` / `RunFailed`) | Infra is operational only (rollout health, smoke checks) | UI shows "done" with final decision + evidence pointers + promoted digests |

---

## Contract invariants (must not regress)

1. **Process is orchestration only**: workflows never encode infra mechanics (KEDA, Jobs, consumer lag, image tags).
2. **Activities are the side-effect boundary**: workflows initiate side effects only via Activities.
3. **Facts are not requests**: facts are pub/sub audit statements; execution queues are intent-only (496).
4. **Owner emits facts**: only the owning Domain emits domain facts, and only after persistence (outbox).
5. **Infra is swappable**: the "executor" can be KEDA Jobs today and a different runner pool tomorrow without changing process semantics.
6. **At-least-once everywhere**: Kafka delivery is at-least-once; Temporal Activities may run more than once; Domain/Process commands MUST be idempotent via explicit idempotency keys.
7. **Signal ingestion requires dedupe + history planning**: if facts are routed into long-lived workflows via `SignalWithStart`, workflows MUST dedupe by `message_id` and use `Continue-As-New` to avoid unbounded history growth.
8. **No “activities-only orchestration”**: long-lived state, waits, retries-as-control-flow, and gate decisions live in Workflows; Activities remain retryable side effects only.
