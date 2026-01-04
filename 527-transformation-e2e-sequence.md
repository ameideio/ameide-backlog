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
