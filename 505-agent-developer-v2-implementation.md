# 505 – Agent Developer Execution Environment (V2 Implementation Plan)

**Status:** Planning (ready for execution)  
**Owner:** Platform / Agents (Process + Agent operators, CLI, Devcontainer, Transformation)  
**Depends on:** 471-476 foundations, 477 (primitive stack), 496 (EDA Principles), 500 (Agent operator), 504 (Agent vertical slice), 505-agent-developer.md (legacy notes), 505-agent-developer-v2.md (target architecture), 506-scrum-vertical-v2.md and 508-scrum-protos.md (Scrum data/process blueprint)

> **Context map:** See [507-scrum-agent-map.md](507-scrum-agent-map.md) for how these dependencies align across Transformation → Process → Agent layers.

> This backlog turns the target architecture in `505-agent-developer-v2.md` into a concrete delivery plan. It is scoped to Ameide’s internal agent capability (Process primitives + AmeidePO + AmeideSA + AmeideCoder) and covers code, infra, GitOps, CLI, and migration work required to make the multi-agent stack production ready.

---

## Grounding & cross-references

- **Architecture grounding:** Executes the architecture defined in `505-agent-developer-v2.md` on top of the primitive/EDA foundations in `471-ameide-business-architecture.md`, `472-ameide-information-application.md`, `473-ameide-technology.md`, `475-ameide-domains.md`, `476-ameide-security-trust.md`, `477-primitive-stack.md`, and `496-eda-principles-v2.md`.  
- **Scrum stack dependencies:** Uses the Scrum domain/process contracts from `300-400/367-1-scrum-transformation.md`, `506-scrum-vertical-v2.md`, and `508-scrum-protos.md` as the canonical data/event model that Process primitives, AmeidePO/AmeideSA, and AmeideCoder must obey.  
- **Primitive & operator tracks:** Aligns workstreams with the primitive operator/vertical-slice backlogs (`498-domain-operator.md`, `499-process-operator.md`, `500-agent-operator.md`, `501-uisurface-operator.md`, `502-domain-vertical-slice.md`, `503-operators-helm-chart.md`, `504-agent-vertical-slice.md`), with Projection/Integration operators tracked as v2 work in `backlog/520-primitives-stack-v2.md`.  
- **Navigation:** `507-scrum-agent-map.md` provides the cross-layer map this implementation plan plugs into; legacy notes and migration concerns remain in `505-agent-developer.md`.

---

## 0. Executive summary

1. **Process primitive(s)** own the sprint/ADM lifecycle and follow the Scrum event model in `506-scrum-vertical-v2.md` and `508-scrum-protos.md`: they consume **Scrum domain facts** (e.g., `SprintStarted`, `SprintBacklogCommitted`, `ProductBacklogItemDoneRecorded`, `IncrementUpdated`) from `scrum.domain.facts.v1`, emit **process facts** (`SprintBacklogReadyForExecution`, `SprintBacklogItemReadyForWork`, etc.) on `scrum.process.facts.v1`, and issue **Scrum domain intents** on `scrum.domain.intents.v1`. They never write Transformation state directly.
2. **AmeidePO** becomes a LangGraph AgentDefinition that consumes process facts, normalizes Dev Briefs, and issues Scrum domain intents (`CommitSprintBacklogRequested`, `RefineProductBacklogItemRequested`, `RecordProductBacklogItemDoneRequested`, `RecordIncrementRequested`).
3. **AmeideSA** becomes a separate LangGraph AgentDefinition that handles technical decomposition, repo digest requests, and delegates to the executor via canonical work handover messages (bus-native; optional A2A transport binding).
4. **AmeideCoder** is the executor runtime (devcontainer service and/or CI-like runner) that consumes work intents, runs the OBSERVE→ACT loop internally with Ameide CLI + editors, and publishes work facts + evidence.
5. **Operators, CLI, GitOps, security, evidencing, observability, and migration docs** are updated to support the split roles and new protocols.

This backlog is successful when:
- All four runtime components (Process primitive + PO/SA/Coder Agent primitives) run in dev/staging with the contract described in the target doc and the canonical Scrum contract (506-v2/508).
- PO/SA never shell into repos; only AmeideCoder touches git/build/test endpoints.
- Transformation shows complete lifecycle evidence (commands, events, artifacts) for an automated Product Backlog Item from `SprintBacklogItemReadyForWork` to `ProductBacklogItemDoneRecorded` / `IncrementUpdated`.

---

## 1. Workstreams & status snapshot

| Track | Scope | Owners | Dependencies | Status |
|-------|-------|--------|--------------|--------|
| **P0 – Process primitive** | Temporal workflow + EDA events | Process operator team | 496, 505-v2 §5.1 | 🟡 In progress (workflow skeleton exists, needs sprint/phase events) |
| **P1 – Transformation contracts** | New commands + event schemas, SDK regen | Transformation domain | 496, packages/ameide_core_proto | 🟡 In progress (proto draft) |
| **P2 – AmeidePO Agent** | LangGraph DAG refactor + prompts | Agent runtime team | primitives/agent/ameide-coder, 504 guardrails | 🔴 Not started |
| **P3 – AmeideSA Agent** | New LangGraph DAG + repo digest tool | Agent runtime team | SA DAG assets new work | 🔴 Not started |
| **P4 – Executor runtime (AmeideCoder)** | Devcontainer service/runner, work handover consumption, evidence production (optional A2A binding) | DevX/Infracore | 504, Ameide CLI, 505-v2 handover | 🔴 Not started in GitOps (devcontainer service is still a placeholder/optional component) |
| **P5 – Operator / CRD support** | `runtime_role`, REST binding annotations, tool grants | Operator team | 500 backlog | 🟡 Partial (runtime_type done) |
| **P6 – CLI & tooling** | Repo bootstrap + codegen gates, prompt updates, `primitive verify` coverage | CLI team | 504, 505-v2 norms, 520 | 🟡 Partial |
| **P7 – GitOps & env rollout** | CR manifests, ApplicationSet wiring | Platform SRE | 503, GitOps repo | 🔴 Not started |
| **P8 – Security & observability** | Secrets, NetworkPolicies, logs/metrics | SecOps + Observability | 8.x sections in 505-v2 | 🟡 Not started |
| **P9 – Migration & demo** | Bridge from develop_in_container, rollout playbook | Agents + DX | 505 legacy doc | 🔴 Not started |

Legend: 🟢 complete · 🟡 in progress · 🔴 not started

---

## 2. Detailed plan by track

### 2.1 Track P0 – Process primitive(s)

**Goal:** ship Temporal workflows that own the governance lifecycle, publish the events catalogued in 496 §11 and `506-scrum-vertical-v2.md`, and integrate with Transformation commands and data model decisions from `506-scrum-vertical-v2.md`.

*Tasks*
1. **Proto + SDK updates**
   - Add `scrum.process.facts.v1` proto file with `SprintBacklogReadyForExecution`, `SprintBacklogItemReadyForWork`, and any additional process facts required by 506-v2 (e.g., `SprintTimeboxReachedEnd`, `SLAWarning`). Scrum domain facts such as `SprintStarted`, `SprintBacklogCommitted`, `ProductBacklogItemDoneRecorded`, and `IncrementUpdated` remain in `scrum.domain.facts.v1` as defined in 508.
   - Regenerate Go/TS/Python SDKs.
2. **Workflow definitions**
   - Implement `primitives/process/ameide-governance` with BPMN-to-Temporal mapping guided by 506's workflow blueprint (timeboxes, signals, gating).
   - Encode `sprintId`, `cycleId`, `timebox` fields in workflow state.
3. **Event emission**
   - Build `EmitProcessEvent` activity that writes to the event bus defined in 496.
   - Ensure `tenant_id`, `process_scope`, `timebox_id` are set.
4. **Integration with Transformation**
   - Ensure workflows **only** interact with Transformation via the bus contracts in 506-v2/508: emit Scrum domain intents on `scrum.domain.intents.v1` (e.g., `StartSprintRequested`, `EndSprintRequested`, `CommitSprintBacklogRequested`, `RecordProductBacklogItemDoneRequested`, `RecordIncrementRequested`) and react to Scrum domain facts on `scrum.domain.facts.v1` (e.g., `SprintStarted`, `SprintBacklogCommitted`, `ProductBacklogItemDoneRecorded`, `IncrementUpdated`). **AmeidePO** remains responsible for issuing these intents; Process workflows observe the resulting domain facts to complete.
   - Provide sample integration tests (Temporal + fake Transformation service).

*Acceptance criteria*
- E2E test: simulate `SprintBacklogCommitted` → Process workflow → `SprintBacklogReadyForExecution` and `SprintBacklogItemReadyForWork` events emitted with correct payload.
- CLI `ameide primitive verify --mode all` passes for the new Process primitive.

### 2.2 Track P1 – Transformation contracts

**Goal:** extend the Transformation domain to implement `ameide_core_proto.transformation.scrum.v1` (artifacts, intents, facts, queries) and enforce audit trails for Scrum artifact changes.

*Tasks*
1. **Proto evolution**
   - Add the `transformation_scrum_*` proto files from `508-scrum-protos.md` under `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/` and register the `ScrumDomainIntent`, `ScrumDomainFact`, and `ScrumQueryService` contracts.
2. **Domain implementation**
   - Update `primitives/domain/transformation` handlers to validate commands (Definition of Done and any optional readiness/policy fields), plus the evidence set.
   - Write events into the outbox (per 496).
3. **SDK + CLI update**
   - Regenerate SDKs; surface helper methods for publishing Scrum domain intents and querying Scrum artifacts. Keep any “complete requirement” style CLI helpers as thin adapters over the canonical Scrum intents only if needed for migration.
4. **Access control**
   - Ensure only AmeidePO agent identities (service account) can publish Scrum domain intents that record PBIs as Done or update Increments (e.g., `RecordProductBacklogItemDoneRequested`, `RecordIncrementRequested`).

*Acceptance criteria*
- Transformation unit tests cover both commands and event emission.
- AmeidePO DAG can call the command in integration tests (mocked).

### 2.3 Track P2 – AmeidePO AgentDefinition

**Goal:** refactor the existing `primitives/agent/ameide-coder` DAG into a true Product Owner agent without coding tools, wired to process events and Transformation commands.

*Tasks*
1. **Definition split**
   - Create `primitives/agent/ameide-po` with LangGraph DAG nodes listed in 505-v2 §6.1.
   - Remove access to `develop_in_container`, CLI commands, or repo tools.
2. **Dev Brief template**
   - Implement a `normalize_dev_brief` node that formats scope/constraints/acceptance criteria.
3. **Process event subscription**
   - Add EDA subscriber (via Process SDK) to feed PO1 node.
4. **Command emission**
   - Add DAG node for issuing Scrum domain intents (`CommitSprintBacklogRequested`, `RefineProductBacklogItemRequested`, `RecordProductBacklogItemDoneRequested`, `RecordIncrementRequested`).
5. **Prompts & memory**
   - Update prompts under `primitives/agent/ameide-po/prompts/*`.
   - Document state machine invariants and audit logging.

*Acceptance criteria*
- `ameide primitive verify --mode all` passes in `primitives/agent/ameide-po`.
- Demo scenario: synthetic sprint event → PO creates Dev Brief → delegates to SA (mock).

### 2.4 Track P3 – AmeideSA AgentDefinition

**Goal:** implement the Solution Architect DAG, including repo digest requests, technical plans, and work handover delegation (optional A2A transport binding).

*Tasks*
1. **New DAG bootstrap**
   - `primitives/agent/ameide-sa` with nodes SA1–SA7 (bootstrapped via Backstage template or equivalent repo template).
2. **Repo digest interface**
   - Define `WorkRequested(work_kind=REPO_DIGEST)` payload; update the executor runtime to produce a read-only digest artifact.
   - Optional: keep an A2A binding for interactive tooling, mapping taskId ⇄ work_id.
3. **Skill invocation**
   - Add `skill=develop_product_backlog_item` invocation node with guardrail plan.
4. **Streaming handling**
   - Implement work-fact consumption in the DAG (progress + completion), feeding review decisions.
   - Optional: if A2A binding is enabled, consume SSE as a transport binding for the same work facts.
5. **Result packaging**
   - Node to collate `prUrl`, artifacts, risk summary for PO.

*Acceptance criteria*
- Integration test: SA publishes mock work intent, consumes work facts, handles repo digest + follow-up iteration.
- No repo access occurs outside the executor runtime.

### 2.5 Track P4 – Executor runtime (AmeideCoder)

**Goal:** evolve the devcontainer service into an executor runtime that consumes work intents, produces work facts + evidence, and (optionally) exposes an A2A transport binding.

*Tasks*
1. **Work handover consumption + publication**
   - Consume `agent.work.intents.v1` messages idempotently (work_id keyed).
   - Publish `agent.work.facts.v1` for `WorkProgressed`/`WorkCompleted`/`WorkFailed` with evidence references.
2. **Optional A2A binding**
   - If enabled, implement `POST /v1/message:send` / `POST /v1/message:stream` / task endpoints as a transport binding over `work_id`.
   - Optional: stand up a **proto-defined gRPC/Connect service** that handles the same operations so internal systems can stay proto-driven even while honoring the public REST binding.
3. **Agent Card (optional)**
   - Serve `/.well-known/agent-card.json` (and legacy `agent.json`) only when A2A is enabled.
   - Include `skills`, `transport.rest`, `auth` metadata.
4. **Repo digest support**
   - Add internal handler for `work_kind=REPO_DIGEST` to summarize repo state (read-only).
5. **Workspace manager**
   - Support both persistent pool and per-task ephemeral volumes (feature flag).
6. **Auth & rate limiting**
   - Validate auth for Solution Architect → Executor delegation (per risk tier policy).
7. **Observability**
   - Emit metrics: task lifecycle durations, phase timings, failure reasons.
8. **CLI integration**
   - Keep Ameide CLI available as internal tool, remove reliance on `develop_in_container` entrypoint.

*Acceptance criteria*
- Contract tests pass for the work handover contract (including idempotency + evidence references).
- If A2A is enabled, contract tests (per A2A spec) pass; include negative cases.
- End-to-end run (PO→SA→Coder) completes inside dev cluster.

### 2.6 Track P5 – Operator / CRD changes

**Goal:** extend the Agent operator + CRD schemas to handle runtime roles, REST binding metadata, and security annotations.

*Tasks*
1. **CRD schema update**
   - Add `spec.runtimeRole` enum (`product_owner`, `solution_architect`, `a2a_server`).
   - Add `spec.a2a` section (e.g., `exposedEndpoints`, `agentCardPath`).
2. **Controller changes**
   - Enforce risk tiers based on runtime role (e.g., `a2a_server => high`).
   - Automatically inject required env vars (Process endpoints, Transformation command hosts, devcontainer URL).
3. **Tool grants**
   - For PO/SA, limit tools to Process/Transformation APIs.
   - For Coder, allow the `develop_product_backlog_item` skill + Ameide CLI internal tools.
4. **Status**
   - Report `WorkHandoverReady` condition; optionally `A2AEndpointReady`, `AgentCardPublished` if A2A is enabled.
5. **Helm & GitOps**
   - Update `operators/helm` and GitOps overlays to include new CRD fields.

*Acceptance criteria*
- `kubectl get agent` shows runtime role + work handover readiness (and A2A endpoint status if enabled).
- Admission webhook rejects misconfigured roles (e.g., Coder without high risk tier).

### 2.7 Track P6 – CLI & tooling

**Goal:** ensure CLI guardrails, codegen gates, and prompts align with the 3-agent architecture.

*Tasks*
1. **Prompt updates**
   - `prompts/agent/*` differentiate PO, SA, Coder personas.
2. **Bootstrap templates + codegen**
   - Repo skeletons for PO/SA/Coder are created via Backstage templates (or equivalent repo templates).
   - Proto-derived wiring/tests are produced via `buf generate` into generated roots; the CLI may wrap `buf` as an orchestrator but must not become a parallel generator system.
3. **Verify enhancements**
   - `primitive verify` checks for `runtimeRole`, optional `a2a` annotations, and required event subscriptions.
4. **Docs**
   - Update `packages/ameide_core_cli/internal/commands/primitive_prompt.go` to mention work handover boundaries (and optional A2A binding).
5. **Tests**
   - Add golden tests for each role.

*Acceptance criteria*
- CLI docs and prompts mention SA↔Coder interactions; no mention of PO calling Coder directly.

### 2.8 Track P7 – GitOps & environment rollout

**Goal:** declare the new primitives in the Ameide GitOps repo and wire ApplicationSets for dev/staging/prod.

*Tasks*
1. **CR manifests**
   - `gitops/ameide-gitops/primitives/process/ameide-governance.yaml`
   - `.../agent/ameide-po.yaml`, `.../agent/ameide-sa.yaml`, `.../agent/ameide-coder.yaml`
2. **ApplicationSets**
   - Add components for each primitive, ensure order (Process → PO → SA → Coder).
3. **Secrets & ExternalSecrets**
   - Define `ExternalSecret` templates for Process (Temporal creds), PO/SA (LLM key), Coder (git + editor tokens).
4. **Rollout playbooks**
   - Document sequencing, toggles, rollback steps.

*Acceptance criteria*
- `argocd app sync` can bring up all four runtime components in dev.
- Runbook includes how to rotate work handover credentials (and A2A auth keys if the optional A2A binding is enabled).

### 2.9 Track P8 – Security & observability

**Goal:** enforce the compensating controls enumerated in 505-v2 §8-9.

*Tasks*
1. **Secrets rotation**
   - Ensure AmeideCoder git tokens are scoped to repos/branches per requirement.
2. **NetworkPolicies**
   - Only allow explicitly declared delegation/integration paths (e.g., Solution Architect → Executor if an A2A binding is enabled), plus Executor → Git/CI/outbound.
3. **Audit logging**
   - Centralized logging of work intents/facts (metadata only, not prompts); if A2A is enabled, log A2A transport metadata as a derived view.
4. **Telemetry**
   - Prometheus metrics for Process events, PO decisions, SA delegation, Coder tasks.
5. **Runbooks**
   - Incident response: compromised Coder workspace, failed Process event, etc.

*Acceptance criteria*
- Security review sign-off, threat model updated.
- Dashboards showing end-to-end flow.

### 2.10 Track P9 – Migration & demos

**Goal:** transition from the interim `develop_in_container` tool to the event-driven work handover contract (with optional A2A binding) and provide demos/documentation of the new workflow.

*Tasks*
1. **Compatibility layer**
   - Keep `develop_in_container` tool as a thin shim that publishes `agent.work.intents.v1` messages (and optionally uses A2A if enabled) until all DAGs are updated.
2. **Docs**
   - Update backlog 505 (legacy) and README entries to point to v2 components.
3. **Demo script**
   - Extend `scripts/demo-agent-slice.sh` to run Process→PO→SA→Coder scenario.
4. **Training material**
   - Internal wiki session explaining new roles, how to debug, how to swap into Coder workspace safely.

*Acceptance criteria*
- Demo run recorded (video or doc) showing requirement processed end-to-end with new architecture.
- Old two-agent DAG is sunset once v2 run is stable.

---

## 3. Timeline / sequencing

| Sprint | Key Deliverables |
|--------|------------------|
| **S1** | Proto/SDK updates (Tracks P0, P1), Process workflow skeleton, Operator CRD extension PR opened. |
| **S2** | AmeidePO DAG MVP, executor work-handover consumption + evidence publishing (optional A2A binding), CLI codegen/verify updates. |
| **S3** | AmeideSA DAG, Process events hitting PO/SA, dev cluster GitOps deployment, NetworkPolicies. |
| **S4** | Observability dashboards, demo run, migration docs, begin removing develop_in_container dependency. |

Dependencies:
- P0 + P1 must land before P2/P3 can integrate with real events.
- P4 (executor runtime + work handover) must land before P3 can be validated (A2A binding is optional).
- P5 (operator support) must land before GitOps rollout.

---

## 4. Risks & mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Executor work handover latency / workspace contention | PO/SA runs stall, timeouts | Implement per-task workspace queue w/ backpressure; expose metrics + autoscaling (optional A2A binding must not add hidden latency) |
| Transformation contract drift | Product Backlog Items stuck in Process | Version events, keep compatibility escort endpoints until all consumers upgraded |
| Security regression (SA accidentally gets repo write) | Data loss | Enforce NetworkPolicy + tool grants, run automated tests verifying no repo binaries in SA image |
| CLI drift | Agents rely on outdated prompts | Add integration test invoking prompts for each role; gating release on CLI alignment |
| Observability gap | Hard to debug multi-agent flow | Instrument each stage with consistent correlation IDs (`productBacklogItemId`, `sprintId`, `work_id`) |

---

## 5. Deliverables checklist

- [ ] Process primitive code + tests + CR manifests
- [ ] Transformation command/event implementation + SDK regeneration
- [ ] AmeidePO AgentDefinition + prompts
- [ ] AmeideSA AgentDefinition + prompts + repo digest tool
- [ ] AmeideCoder executor runtime (work handover consumption + evidence; optional A2A binding)
- [ ] Operator CRD updates + Helm/GitOps overlays
- [ ] CLI codegen/verify/prompt updates
- [ ] GitOps ApplicationSets for Process/PO/SA/Coder
- [ ] Security artifacts (NetworkPolicy, secrets, threat model)
- [ ] Observability dashboards + runbooks
- [ ] Migration guide & demo script

---

## 6. Sign-offs required

- **Architecture:** Platform architecture board acknowledges 505-v2 as target and accepts this plan.
- **Security:** Security review (threat modeling + NetworkPolicy verification).
- **Operations:** GitOps/infra team approves ApplicationSet rollout steps.
- **Product/Stakeholder:** Transformation/product owners confirm the lifecycle satisfies governance requirements (Definition of Done evidence and any optional readiness/policy checks).

---

## 7. References

- [505-agent-developer-v2.md](505-agent-developer-v2.md) – target architecture
- [500-agent-operator.md](500-agent-operator.md) – CRD/operator status
- [504-agent-vertical-slice.md](504-agent-vertical-slice.md) – CLI/operator guardrails
- [496-eda-principles-v2.md](496-eda-principles-v2.md) – event contracts
- [477-primitive-stack.md](477-primitive-stack.md) – placement of primitives
- [A2A Protocol](https://a2a-protocol.org/latest/specification/) – optional transport binding reference

--- 
