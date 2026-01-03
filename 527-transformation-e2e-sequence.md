# 527 Transformation — E2E Execution Sequence (Process vs Events vs Infra)

**Status:** Draft (normative intent; implementation exists for Transformation work execution)  
**Parent:** `backlog/527-transformation-capability.md`  
**Related:** `backlog/520-primitives-stack-v2.md`, `backlog/496-eda-principles.md`, `backlog/509-proto-naming-conventions.md`, `backlog/527-transformation-process.md`, `backlog/527-transformation-domain.md`, `backlog/527-transformation-integration.md`

This backlog rewrites the end-to-end “work execution” story as a clean separation:

- **Process** decides *what* happens next (orchestration only; Temporal workflows are deterministic).
- **Events** carry **intents** (requests) and **facts** (audit/outcomes) with strict 496 semantics.
- **Infra** decides *how* work runs (KEDA/Jobs/worker pools/external executors), without leaking into process semantics.
- **UI** shows a projection-driven timeline (facts + process facts + evidence refs), not infra internals.

Terminology (canonical in `backlog/520-primitives-stack-v2.md`):

- **Activity-inline execution:** the work runs inside the Process primitive’s Temporal Activity worker.
- **Activity-delegated execution:** a Process Activity initiates work but another runtime executes it; delegation is via **intent**, completion is observed via **facts**.

Infra contract (agentic coding via Codex):

- The AgentWork “coder” executor expects **authenticated Codex CLI** via an `auth.json` file.
- In-cluster, infra provides this by mounting `Secret/codex-auth` (created by ExternalSecret) and setting `CODEX_HOME=/codex-home`.
  - An initContainer copies `/codex-secret/auth.json` → `/codex-home/auth.json` (no secret bytes ever travel on the broker).
- If `codex-auth` is missing, the executor MUST fail fast with a clear error and record a `WorkFailed` outcome (Domain emits facts via outbox).

---

## Sequence A — End-to-end (detailed)

Sequence A is described in the three phases below (triage → iterations → release). The delegated execution seam is always the same:

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

### A.1 — User chats with Architect Agent (triage)

This phase is **interactive** and mostly lives in the **Agent primitive (LangGraph runtime)**. Temporal is optional until you want durable orchestration/gates.

| Task | Process (Temporal) | Events (intent vs fact) | Infra (mechanics) | UI (what the user can see) |
| --- | --- | --- | --- | --- |
| Chat turn: architecture question → answer | Optional: start a “triage run” workflow to capture gates/evidence; otherwise no workflow yet | none required | UISurface calls Agent primitive API (LangGraph) | Chat transcript + agent answer streamed; optional “triage run created” badge |
| Triage reads context | Workflow may allow projection reads (declared, time-bounded) but does not mutate state | none | Agent reads Projection query surfaces via SDK | UI can show “sources used” (projection queries / evidence refs), not raw infra calls |
| Triage produces an artifact | Workflow (if running) calls an Activity that requests a Domain write (proposal) | Domain receives a **command/intent** to create/update a draft artifact (e.g., “Requirement draft”, “Dev Brief”, “Gate checklist”) | Domain persists and emits facts via outbox | Draft artifact appears (Requirement/Brief) with version + status = draft |
| Triage needs deep repo/tool analysis | Workflow calls delegated Activity → Domain `RequestWork(work_kind=AGENT_WORK|TOOL_RUN)` | **Intent:** `WorkExecutionRequested` on `transformation.work.queue.*.v1`<br>**Facts:** `WorkRequested/Started/Completed/Failed` on `transformation.work.domain.facts.v1` | Executor runs agent/tool work in an isolated environment and records outcomes to Domain | UI shows “analysis requested/in progress/completed” with links to evidence (logs, patch, summary) |

**Sub-activities (recommended decomposition):**

- **Ask/Answer:** “Architect agent” answers questions using read-only tools (Projection queries, glossary, prior evidence).
- **Classify/Triage:** produce a structured triage output (scope, risks, proposed primitives touched, required gates).
- **Propose:** write a promotable draft artifact via Domain command (never direct writes).
- **Escalate to execution:** when the agent needs repo/tool work, it becomes a delegated WorkRequest (Sequence A).

---

### A.2 — Requirement stabilized → request work → dev/testing iterations

This phase is **durable orchestration**: Process sequences steps, but execution happens via delegated WorkRequests.

| Task | Process (Temporal) | Events (intent vs fact) | Infra (mechanics) | UI (what the user can see) |
| --- | --- | --- | --- | --- |
| Gate: “Requirement stabilized” | Workflow reaches a gate; waits for approval/confirmation | Process fact (optional) like `GateDecisionRecorded` for audit | none | UI shows “approval required”; user approves/rejects; decision recorded in timeline |
| Request scaffold/codegen | Workflow Activity calls Domain `RequestWork(work_kind=TOOL_RUN, action_kind=SCAFFOLD|GENERATE)` | **Intent:** `WorkExecutionRequested` to toolrun queue<br>**Facts:** `WorkRequested/…/Completed` | Executor runs `ameide`/`buf generate` and records evidence | UI shows “codegen requested/in progress/completed” + evidence links (logs, generated diff summary) |
| Request coding change | Workflow Activity calls Domain `RequestWork(work_kind=AGENT_WORK, action_kind=PUBLISH)` | **Intent:** `WorkExecutionRequested` to agentwork queue<br>**Facts:** lifecycle facts | Executor runs “coding agent” in an isolated pod/runner with `CODEX_HOME` seeded from `Secret/codex-auth` (and optional GitHub token) and records evidence (PR link, patches, summaries) | UI shows “coding run started”; on success shows PR URL + patch + summary; on fail shows error + evidence |
| Request verification | Workflow Activity calls Domain `RequestWork(work_kind=TOOL_RUN, action_kind=VERIFY)` | **Intent:** `WorkExecutionRequested` to verify queue<br>**Facts:** lifecycle facts | Executor runs tests/verify suites; uploads evidence | UI shows verification checks + pass/fail, with logs attached |
| Iterate until green | Workflow loops on outcomes + gate rules (no infra knowledge) | Facts are the only completion signals | Infra may change (KEDA, worker pool, external runners) without changing workflow semantics | UI shows iteration timeline and “why” (facts/evidence), not infra retries/backoffs |

**Sub-activities (recommended decomposition):**

- **Stabilize:** turn triage artifacts into an approved requirement slice (Domain-backed draft → approved).
- **Generate:** scaffold + codegen + SDK regen (tool-run).
- **Implement:** coding agent produces a PR/change set (agent-work).
- **Verify:** DoR/DoD checks, unit/integration tests, contract gates (tool-run).
- **Review/Approve:** humans/agents approve via Domain commands; workflow gates advance based on facts.

---

### A.3 — Release

Release is still a Process concern (gates + sequencing), but “what gets deployed” is owned by GitOps artifact promotion and digest-pinned images.

| Task | Process (Temporal) | Events (intent vs fact) | Infra (mechanics) | UI (what the user can see) |
| --- | --- | --- | --- | --- |
| Gate: “Ready to release” | Workflow requires explicit approval (policy) | Process fact (optional) for audit | none | UI shows “ready to release” gate + required checks; approval recorded |
| Build/publish artifacts | Workflow Activity requests `TOOL_RUN` publish step (or triggers CI via Integration) | **Intent:** `WorkExecutionRequested` for publish class<br>**Facts:** Work lifecycle | CI/build system produces digest-pinned images (see `backlog/602-image-pull-policy.md`) | UI shows build status + produced digest(s) + evidence (logs) |
| Promote to environments | Workflow waits on promotion evidence (PR merged / promotion recorded) | Domain facts record the release/promote outcome; projections show it | GitOps promotion is a **Git change** (digest copy) per `backlog/611-trunk-based-main-and-gitops-environment-promotion.md`; publishing may use the proto-aware fast path per `backlog/611-cd-proto-aware-image-publish-fast-path.md` | UI shows environment promotion timeline (dev→staging→prod), PR links, and rollout status summary |
| Close the run | Workflow emits final process facts and stops | Facts remain the audit trail | Infra is purely operational (rollout health, smoke checks) | UI shows “run complete” with final decision + pointers to evidence and promoted digests |

---

## Contract invariants (must not regress)

1. **Process is orchestration only**: workflows never encode infra mechanics (KEDA, Jobs, consumer lag, image tags).
2. **Activities are the side-effect boundary**: workflows initiate side effects only via Activities.
3. **Facts are not requests**: facts are pub/sub audit statements; execution queues are intent-only (496).
4. **Owner emits facts**: only the owning Domain emits domain facts, and only after persistence (outbox).
5. **Infra is swappable**: the “executor” can be KEDA Jobs today and a different runner pool tomorrow without changing process semantics.
6. **At-least-once everywhere**: Kafka delivery is at-least-once; Temporal Activities may run more than once; Domain/Process commands MUST be idempotent via explicit idempotency keys.
7. **Signal ingestion requires dedupe + history planning**: if facts are routed into long-lived workflows via `SignalWithStart`, workflows MUST dedupe by `message_id` and use `Continue-As-New` to avoid unbounded history growth.
