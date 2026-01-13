# 527 Transformation — Implementation Plan (target-state, no migration shims)

> **DEPRECATED (2026-01-12):** This implementation plan assumes a Temporal-backed Process primitive posture for BPMN-authored processes.  
> Current direction: BPMN-authored processes execute on **Camunda 8 / Zeebe**; see `backlog/527-transformation-process-v2.md`.

**Status:** Draft (implementation in progress)  
**Parent:** `backlog/527-transformation-capability.md`

This document is the **Implementation & Migration** layer plan for delivering the Transformation capability. Because we are **not live**, “migration” here means **implementation slicing + deletion of legacy code**, not compatibility shims, traffic splitting, or façade-first routing.

> **Update (2026-01): testing contract is 430v2**
>
> Any references in this plan to “integration modes” (`INTEGRATION_MODE`) or pack scripts (`run_integration_tests.sh`) should be treated as legacy. The normative repo contract is `backlog/430-unified-test-infrastructure-v2-target.md` (Unit → Integration → E2E; cluster interaction only in E2E; JUnit evidence mandatory).

> **Update (2026-01): memory-first clean target**
>
> Organizational memory is defined by `backlog/656-agentic-memory.md`. The clean-target refactor posture for the Transformation Domain/Projection that implements it is defined in `backlog/657-transformation-domain-clean-target.md` (Kafka-first ingestion, gRPC auth boundaries, proposal-only writes for execution agents).

> **Update (2026-01): clarify “KEDA vs Coder vs Executor”**
>
> This document still contains a legacy “WorkRequests + Kafka execution queue topics + KEDA ScaledJobs” execution substrate description.
> That posture is **not** the platform default going forward:
> - Code-changing automation defaults to **Coder workspaces + ephemeral task workspaces** (`backlog/650-agentic-coding-overview.md`).
> - Inter-primitive background execution under EDA v2 defaults to a **Knative Trigger subscriber Executor Service** (ACK fast + async work) (`backlog/658-eda-v2-reference-implementation-refactor.md`).
> Treat KEDA as a migration-era/local-dev implementation detail until this plan is refactored to the EDA v2 executor posture.

---

## Layer header (Implementation & Migration)

- **Primary ArchiMate element types used:** Work Package, Plateau, Deliverable, Gap.
- **Goal:** a single executable plan with explicit DoD gates (tests + verify).

---

## 0) Non-goals (explicit)

- No compatibility shims for legacy backends.
- No long-lived production use of `services/graph`, `services/repository`, or `services/transformation`.
- No canonical “Artifact” model. “Artifact” is UX vocabulary only; canonical nouns are **Element**, **View**, **Document**, **Relationship**, **Version**, **Definition**.
- No intermediate architecture that knowingly creates rework. Implement the target state directly.

---

## 0.0) Definition of success (v1 acceptance slice)

v1 “success” is a **workflow-driven** and **testable** slice (not a UI demo):

1) **Two governance workflows exist** (Scrum + TOGAF ADM) that operate over the same canonical element substrate and are observable via timelines:
   - `r2r.governance.scrum.v1`
   - `r2r.governance.togaf_adm.v1`

2) **Testing is automatic background work** (no human “run tests” action):
   - Process requests verification via Domain-owned WorkRequests,
   - runners execute deterministically and record evidence back into Domain,
   - Process waits on `WorkCompleted/WorkFailed` and gates promotions accordingly.

3) **Cluster UI harness verification is real and provable** (430-aligned; stable URLs; no Telepresence; no GitOps PRs):
   - WorkRequest remains `work_kind=tool_run`, `action_kind=verify`, selecting the harness via:
     - `verification_suite_ref=transformation.verify.ui_harness.gateway_overlay.v1`
     - queue: `transformation.work.queue.toolrun.verify.ui_harness.v1` (separate executor identity/RBAC)
   - The harness:
     - builds shadow images in-cluster (BuildKit) and records image digests,
     - deploys shadow workloads (baseline Deployments unchanged),
     - applies Gateway API overlay routes (`HTTPRoute`) using a non-guessable run key header (`X-Ameide-Run-Key=<nonce>`),
     - **fails closed unless it can prove WUT routing** (route `Accepted/Resolved` + overlay-only marker check),
     - runs Playwright against the stable URL and writes artifacts to `/artifacts/e2e/*`,
     - records evidence (artifacts + routing proof + digests + created/deleted resources + teardown proof),
     - cleans up deterministically (run anchor + `ownerReferences` + TTL janitor sweep).

4) **End-to-end tests exist and are runnable** per `backlog/430-unified-test-infrastructure-v2-target.md`:
   - unit + integration tests cover Domain/Process/Runner seams (idempotency, fact-after-persist, ack-after-durable-outcome),
   - cluster validation is E2E-only (Phase 3) under the repo contract,
   - both Scrum and TOGAF ADM scenarios have coverage at the “process requests work → evidence recorded → process continues” seam,
   - UI harness verification runs are included where applicable (edge-routable service changes).

This definition is intentionally stricter than “we can click through a UI”: success is measured by **automated workflows + automated evidence + automated gates**.

---

## 0.1) Implementation progress (repo snapshot; checklist)

This section is a lightweight status tracker against the work packages below.

- [x] WP-A1 Domain: enterprise repository write model implemented (`primitives/domain/transformation`).
- [x] WP-A2 Projection: enterprise repository read model implemented (`primitives/projection/transformation`).
- [x] WP-A2 Ingestion: outbox → projection relay implemented (`primitives/projection/transformation/cmd/relay`).
- [ ] WP-A2 Ingestion (clean target, 657): projection ingests from Kafka as the default runtime posture; relay is debug/recovery only.
- [x] WP-A3 UISurface: existing ArchiMate editor wired to primitives (`services/www_ameide_platform`).
- [x] WP-Z Deletion: legacy `services/*` backends removed (`services/graph`, `services/repository`, `services/transformation`).
- [x] WP-Z GitOps cleanup: legacy `graph`/`transformation` app components removed and gateway no longer routes non-graph proto traffic through `graph`.
- [x] WP-0 Repo health: confirm repo-wide codegen drift gates are green and enforceable in CI (regen-diff).
- [x] WP-B (CODE) WorkRequests substrate: Domain WorkRequest record + facts + queue-topic fanout implemented; executor implemented (`primitives/integration/transformation-work-executor`).
- [x] WP-B (CODE) Process ingress consumes Kafka domain facts (`PROCESS_INGRESS_SOURCE=kafka://`) and signals workflows (Temporal signals; deploy/wiring is a GitOps concern).
- [x] WP-B (CODE) Domain dispatcher publishes outbox topics to Kafka by default (no topic prefix filter).
- [ ] WP-B (SECURITY, 657): Domain + Projection gRPC services enforce authN/authZ at the boundary (no “trust internal callers” posture).
- [x] WP-B (TEST) Capability tests exist (`capabilities/transformation/__tests__/integration`) with Phase 2 local integration coverage for the WorkRequest seam and Phase 3 E2E (Playwright) coverage where applicable.
- [x] WP-B (TEST) Test contract alignment: follow `backlog/430-unified-test-infrastructure-v2-target.md` (no `INTEGRATION_MODE`; JUnit evidence; cluster interaction only in E2E).
- [x] WP-B (CODE) UI harness verification suite exists (`transformation.verify.ui_harness.gateway_overlay.v1`) and is requested as `action_kind=verify` (no special E2E action kind); runner proves routing via Gateway API overlay marker and records `/artifacts/e2e/*`.
- [ ] WP-B (CLUSTER) Cluster end-to-end depends on deployed wiring (dispatcher → broker → executor → domain facts → ingress → Temporal → projection). Code + tests assume those components exist and are reachable; environment health/config may still block execution.
- [x] GitOps parity (execution substrate): KEDA + Kafka work-queue topics + workbench + secret wiring exist in `ameide-gitops` (enabled in `local` + `dev`; disabled elsewhere).
- [x] GitOps parity (contract topics): Transformation fact-stream KafkaTopics exist as a dedicated component (`data-kafka-transformation-contract-topics`) and are asserted by `data-data-plane-smoke` (enabled in `local` + `dev`).
- [x] GitOps parity (runtime wiring): dispatcher/ingress/relay workloads exist and are enabled in `local` + `dev` (`domain-transformation-v0-dispatcher`, `process-transformation-v0-ingress`, `projection-transformation-v0-relay`).
- [x] GitOps parity (runtimes): Process/Agent/Projection/UISurface runtime components exist in `ameide-gitops` and local has a minimal “stacktest” set enabled (`process-ping-v0`, `agent-echo-v0`, `projection-foo-v0`, `uisurface-hello-v0`) to validate the substrate end-to-end.
- [x] GitOps parity (images): primitive images published by CI use the `ghcr.io/ameideio/primitive-<suffix>` repository name. GitOps MUST use the `primitive-` prefix for primitives and MUST deploy digest-pinned refs (local/dev digest bump PRs resolve `:dev` → digest; staging/prod move by promotion PRs). See `backlog/602-image-pull-policy.md` / `backlog/603-image-pull-policy.md`.
- [x] GitOps parity (gateway routing): platform gateway routes Transformation gRPC services to primitives (Domain CommandServices → `transformation-v0-domain`; Projection QueryServices → `transformation-v0-projection`) across environments.

Gates (currently passing):

- `go run ./packages/ameide_core_cli/cmd/ameide primitive verify --kind domain --name transformation --mode repo`
- `go run ./packages/ameide_core_cli/cmd/ameide primitive verify --kind projection --name transformation --mode repo`
- `pnpm -C services/www_ameide_platform run typecheck`

Quick verification (GitOps / cluster truth):

- `local-data-data-plane-smoke` asserts WorkRequests execution queue topics, Transformation contract topics, and WorkRequests runner KEDA `ScaledJob` objects (gated; enabled in `local` + `dev`).
- `local-foundation-operators-smoke` asserts KEDA operator deployments and the external metrics `APIService` (when KEDA is installed).
- Primitive runtime apps: ArgoCD Applications `local-process-ping-v0`, `local-agent-echo-v0`, `local-projection-foo-v0`, `local-uisurface-hello-v0` should be `Synced/Healthy`, and the corresponding `ameide.io/*` CRs should report `Ready=True`.
- `local-transformation-v0-runtime-wiring-smoke` asserts the Transformation runtime wiring deployments are `Available` (dispatcher, ingress, projection relay; enabled in `local` + `dev`).

### Historical note (preserve context)

- Earlier iterations carried cluster-only coverage in separate test files and used env flags to skip cluster suites when infra was unavailable.
- Current posture treats cluster validation as E2E-only under the repo contract and as GitOps smokes for “cluster truth” probes (no `INTEGRATION_MODE`).
- Platform Gateway should route Transformation primitive gRPC services internally (no legacy `graph` routing):
  - `kubectl -n ameide-local get grpcroute.gateway.networking.k8s.io transformation`
  - Smoke: `local-platform-transformation-routing-smoke` asserts the `GRPCRoute/transformation` is `Accepted=True` and `ResolvedRefs=True`.

### Operational gotcha: primitive image naming

`ameide` publishes primitive images from `primitives/**/Dockerfile.main` with the name:

- `primitive-` + `primitives/<kind>/<capability>` path (slashes become `-`)

Examples:

- `primitives/process/transformation` → `ghcr.io/ameideio/primitive-process-transformation@sha256:<digest>` (producer may also publish `:dev` as the local/dev discovery tag)
- `primitives/projection/transformation` → `ghcr.io/ameideio/primitive-projection-transformation@sha256:<digest>` (producer may also publish `:dev` as the local/dev discovery tag)

If GitOps references the wrong repository (e.g. missing the `primitive-` prefix), Kubernetes will hit `ImagePullBackOff` even when the CI build is green. Example of a wrong ref:

- `ghcr.io/ameideio/process-transformation@sha256:<digest>` (missing `primitive-`)

Note: the examples above describe the **build/publish tag convention**. GitOps target state is to deploy **digest-pinned** refs (see `backlog/602-image-pull-policy.md`) so rollouts are driven by Git changes, not by mutable tags.

### Recent failure mode (local): projection migrations CrashLoop

If `transformation-v0-projection` is `CrashLoopBackOff` with:

- `failed to ensure migrations ... syntax error at or near \"//\" (SQLSTATE 42601)`

…the deployed `ghcr.io/ameideio/primitive-projection-transformation@sha256:<digest>` is older than `ameide` PR **#348** (the fix removes an accidental `// nosemgrep...` prefix inside the SQL string literal).

Operational note (local): this caching failure mode was only relevant when deploying mutable tags (e.g. `:dev`). GitOps now pins primitives by digest (see `backlog/602-image-pull-policy.md`), so a rollout should be driven by a Git change + Argo sync. If debugging, compare the running pod image digest vs Git and re-sync the Application instead of forcing node cache deletes.

### Recent failure mode (local): process migrations CrashLoop

If `transformation-v0-process` is `CrashLoopBackOff` with:

- `ensure migrations: ensure schema_migrations: ERROR: syntax error at or near \"//\" (SQLSTATE 42601)`

…the deployed `ghcr.io/ameideio/primitive-process-transformation@sha256:<digest>` is older than the fix in `ameide` PR **#382**. This blocks the cluster-mode process seam test:

- `TestWorkRequestSeam_ProcessOrchestration_ClusterMode` in `capabilities/transformation/__tests__/integration`

Operational note: this caching failure mode was only relevant when deploying mutable tags (e.g. `:dev`). GitOps now pins primitives by digest (see `backlog/602-image-pull-policy.md`), so a rollout should be driven by a Git change + Argo sync.

## 1) Alignment to 520 (normative constraints)

Cross-reference: `backlog/520-primitives-stack-v2.md`.

We treat 520 as non-negotiable for this plan:

- **Proto is the behavior schema.** Services/messages/events are defined in protos; SDKs are generated from protos.
- **`buf generate` is canonical.** The CLI orchestrates scaffolding/verify; it does not replace Buf plugins.
- **Generated outputs are clobber-safe.** Implementation-owned code lives outside generated-only roots.
- **Guardrails are gates.** “Done” means regen-diff is clean, tests are green, and `go run ./packages/ameide_core_cli/cmd/ameide primitive verify` is meaningful.

## 1.1) Testing posture (normative): 537 RED→GREEN discipline

Cross-reference: `backlog/537-primitive-testing-discipline.md`.

This implementation plan is **test-driven by default**:

- Every primitive scaffold begins in **RED** with `AMEIDE_SCAFFOLD` markers; “progress” means replacing those tests with real assertions (GREEN) and keeping them green while refactoring.
- “Done” for any work package requires:
  - tests for the affected primitive(s) are present and green (unit + integration where applicable),
  - `ameide primitive verify` passes for the affected primitive(s),
  - repo-wide regen-diff gates remain clean (520 posture).

Implication: any new primitive/workflow surface MUST land with a failing test first (or as part of the same PR) so we never create “implementation without a harness”.

---

## 2) Target ownership (primitives)

Transformation is IT4IT-aligned by default (Transformation *is* the IT value-stream capability), but we keep strict primitive boundaries:

- **Domain primitive:** `primitives/domain/transformation`  
  Canonical writer for Transformation-owned design-time state:
  - Enterprise Repository (elements-only knowledge substrate + workspace tree)
  - Definition Registry (schema-backed definitions + promotions)
  - Governance (initiatives, baselines/promotions/approvals, evidence references)
- **Projection primitive:** `primitives/projection/transformation`  
  Read models + query services for UI/agents (browse/search/timelines), consuming facts idempotently.
- **Process primitive:** `primitives/process/transformation`  
  Orchestrates workflows (Temporal), emits process facts only, requests changes via domain commands/intents.
- **Integration primitives:** `primitives/integration/transformation-*`  
  Tool runners/adapters; they never become writers of canonical domain state.
- **UISurface:** `services/www_ameide_platform`  
  Thin: reads via projection query services; writes via domain command/intents.

---

## 2.1) Delivery ownership (CODE vs GITOPS)

This plan intentionally separates:

- **CODE (this repo: `ameide`)** — protos, service implementations, runner/job code, tests, and `ameide primitive verify` gates.
- **GITOPS (`ameide-gitops`)** — Kubernetes deployment manifests/charts (including KEDA, ServiceAccounts/RBAC, secrets wiring, env scoping), and environment-specific enablement (`local`/`dev` only where required).

Rule: a work package is not “done” unless **both** the CODE and GITOPS deliverables (where applicable) are complete and their respective gates pass.

## 3) Identity and storage model (locked decisions)

### 3.1 Scope identity (required everywhere)

All repository-scoped objects and envelopes are scoped by:

- `{tenant_id, organization_id, repository_id}`
- `repository_id` is the **only** repository identifier exposed in contracts and APIs (no `graph_id`).

### 3.2 Canonical storage (elements-only)

Enterprise Repository canonical storage is **elements-only** (303 direction):

- `Element` (notation kind + `type_key` + `body` + `metadata`)
- `ElementRelationship` (`SEMANTIC | CONTAINMENT | REFERENCE`)
- `ElementVersion` (immutable snapshots for views/docs; head/published pointers; no cascading versioning)
- Workspace organization is separate:
  - `WorkspaceNode` (tree)
  - `ElementAssignment` (node → element, optional fixed version reference)

### 3.3 “Diagram” naming

- UX: “diagram”
- Domain noun: **View** = an `Element` with `kind = *_VIEW` whose layout is stored in `ElementVersion.body`.

---

## 4) Proto naming conventions (Transformation-owned contracts)

Cross-reference: `backlog/509-proto-naming-conventions.md`.

### 4.1 Target package layout

Transformation-owned packages live under `ameide_core_proto.transformation.<subcontext>.v1`:

- `ameide_core_proto.transformation.knowledge.v1`  
  Enterprise Repository substrate (repositories/nodes/assignments/elements/relationships/versions + view layout payload schema).
- `ameide_core_proto.transformation.registry.v1`  
  Definition Registry (ProcessDefinition/AgentDefinition/ExtensionDefinition + versions/promotions).
- `ameide_core_proto.transformation.governance.v1`  
  Initiatives + baselines/promotions/approvals + evidence references.
- `ameide_core_proto.process.transformation.v1`  
  Transformation process facts catalog (`ActivityTransitioned`, `GateDecisionRecorded`, `ToolRunRecorded`, …).

Topic families follow `backlog/509-proto-naming-conventions.md` and `backlog/496-eda-protobuf-ameide.md`:

- `transformation.knowledge.domain.facts.v1`
- `transformation.registry.domain.facts.v1`
- `transformation.governance.domain.facts.v1`
- `transformation.work.domain.facts.v1`
- `transformation.process.facts.v1`

### 4.2 Service naming and ownership

To preserve the Domain/Projection boundary:

- Domain exposes **write surfaces** (`*CommandService`).
- Projection exposes **read surfaces** (`*QueryService`).

Example:

- `TransformationKnowledgeCommandService` (implemented by Domain primitive)
- `TransformationKnowledgeQueryService` (implemented by Projection primitive)

### 4.3 Deletions (not live, so break now)

- `ameide_core_proto.graph.v1` is transitional/legacy (the target-state substrate is `ameide_core_proto.transformation.knowledge.v1`). New work must not add new `graph.v1` dependencies; remove remaining `graph.v1` references as a dedicated cleanup step.
- `ameide_core_proto.transformation.v1.TransformationService` is legacy and must be removed/replaced by the split subcontext services above.
- Any proto vocabulary using “Artifact” as a canonical noun must be removed.

---

## 5) Work Packages and Plateaus

### WP-0 (Prerequisite) — Contracts + codegen drift must be green

**Goal:** make verification meaningful before we implement behavior.

**Progress checklist**

- [x] Enterprise Knowledge contracts exist and are in use:
  - [x] `packages/ameide_core_proto/src/ameide_core_proto/transformation/knowledge/v1/`
- [x] Governance contracts exist:
  - [x] `packages/ameide_core_proto/src/ameide_core_proto/transformation/governance/v1/`
- [x] Definition Registry contracts exist:
  - [x] `packages/ameide_core_proto/src/ameide_core_proto/transformation/registry/v1/`
- [x] Transformation process-facts contracts exist:
  - [x] `packages/ameide_core_proto/src/ameide_core_proto/process/transformation/v1/`
- [ ] Regen-diff gates are enforced repo-wide in CI (codegen drift is a hard error, not “best effort”).

**Deliverables**

- Proto packages exist under `packages/ameide_core_proto/src/ameide_core_proto/transformation/{knowledge,registry,governance}/v1/` (plus `process/transformation/v1/`):
  - [x] `transformation/knowledge/v1`
  - [x] `transformation/registry/v1`
  - [x] `transformation/governance/v1`
  - [x] `process/transformation/v1`
- Repo-wide SDK regeneration is deterministic and checked in.

**DoD (gates)**

- `pnpm -C packages/ameide_core_proto run generate:local`
- `pnpm -C packages/ameide_core_proto exec buf generate --template buf.gen.sdk-ts.local.yaml`
- `pnpm -C packages/ameide_core_proto exec buf generate --template buf.gen.sdk-go.local.yaml`
- `pnpm -C packages/ameide_core_proto exec buf generate --template buf.gen.sdk-python.yaml`
- `cd packages/ameide_core_proto && buf lint && buf breaking`
- Regen-diff: running the generation commands above yields `git diff` empty.

---

### WP-A (MVP) — Enterprise Repository editing (ArchiMate first, any-notation ready)

**MVP goal:** edit and view enterprise repository content, with the existing graphical editor wired through primitives.

**Explicitly out of scope for MVP**

- Initiatives, baselines/promotions/evidence bundles
- Scrum/TOGAF/PMI workflows
- Stored AgentDefinitions / agentic approvals

### WP-B (v1 substrate) — Work execution substrate (WorkRequests + evidence + timelines)

**Goal:** make the execution substrate a first-class, event-driven seam so “runner invocation pending” does not persist as a long-term architecture gap.

**Outcome:** Process can request a tool/agent step via a Domain-owned `WorkRequest`, an ephemeral execution backend runs it, Domain records evidence, Process emits process facts, and Projection/UISurface can show an audit-grade run timeline.

**Proto-first workflow (required)**

WP‑B is implemented **proto-first** so orchestration and evidence do not drift from the contract spine:

1. Update protos first (intents/facts/process facts + evidence references), then run regen-diff gates (520).
2. Implement Domain write surfaces + outbox facts for the new messages; add Domain tests for idempotency and “facts after persistence” (537).
3. Implement runner/job behavior that consumes `WorkExecutionRequested` (execution queue intent) and records outcomes/evidence via Domain commands (idempotent).
4. Implement Process orchestration (send intent → await facts → emit process facts) and Projection joins for timelines/citations.

**Deliverables (CODE — `ameide` repo)**

- Domain:
  - `WorkRequest` record + idempotency posture (`client_request_id`)
  - Domain facts: `WorkRequested`/`WorkCompleted`/`WorkFailed` (after persistence)
  - Command surface to record outcomes with evidence bundle linking (idempotent)
- Integration (runner/job code):
  - a WorkRequest **executor** entrypoint that executes exactly one WorkRequest per run (checkout `commit_sha` → run `ameide` actions → persist evidence → record outcome idempotently); this executor image is **devcontainer-derived** (full toolchain) and separate from the debug/admin workbench deployment
  - evidence descriptor shape that Process can cite in `ToolRunRecorded` (no log scraping)
- Process:
  - at least one workflow that requests a WorkRequest and awaits completion facts
  - emits `ToolRunRecorded` + step transitions (`ActivityTransitioned`) with correlation to the WorkRequest
- Projection:
  - materialize a run timeline that joins process facts to WorkRequest lifecycle facts with citations
- UISurface:
  - minimal “Execution timeline” view (optional for v1 substrate proof; headless E2E is the default proof)

**Deliverables (GITOPS — `ameide-gitops` repo)**

- Kafka (broker wiring; normative):
  - create the WorkRequest execution queue topics (dedicated per executor class) so KEDA scales only on execution intent lag (not on domain fact streams):
    - `transformation.work.queue.toolrun.verify.v1`
    - `transformation.work.queue.toolrun.generate.v1`
    - `transformation.work.queue.agentwork.coder.v1`
    - recommended (new): `transformation.work.queue.toolrun.verify.ui_harness.v1` (UI harness verification suite; Gateway API overlay routing + Playwright; separate executor/RBAC)
  - note: these queue topics are distinct from canonical domain fact streams (e.g., `transformation.*.facts.v1`) which exist for persistence/projection; scaling MUST NOT depend on them
  - define the consumer group naming convention per executor class/role (e.g., `transformation-workrequest-executor.v1.<kind>`) and bind it in KEDA trigger metadata
  - set retention/cleanup to reflect “Kafka is transport, not evidence” (short retention + delete policy; evidence is persisted in Domain and object storage)
- KEDA:
  - install/configure KEDA in `local`/`dev` (and any required broker scaler wiring)
  - KEDA ScaledJob (normative) with a Kafka trigger that schedules Kubernetes Jobs based on consumer group lag on execution queue topics carrying `WorkExecutionRequested` (Job is the Kafka consumer; scale is by consumer group lag)
- Kubernetes security + runtime wiring:
  - ServiceAccounts/RBAC + NetworkPolicy for runner Jobs (least privilege)
  - secrets/config injection required for repo checkout, evidence upload, and Domain callbacks
  - image deployment references (values contract): devcontainer workbench image + executor image for queue-driven Jobs
- Debug/admin workbench (local/dev only; not a processor):
  - deploy workbench pod(s) only in `local`/`dev` with admin-only access
  - verify workbench cannot consume execution queue intents and cannot become an orchestrator

**Implementation status (CODE; repo snapshot)**

- [x] Domain owns WorkRequests (SoR + facts-after-persistence + queue-topic fanout):
  - `RequestWork` persists a `WorkRequest` and emits:
    - canonical `transformation.work.domain.facts.v1`
    - a KEDA queue topic (`transformation.work.queue.*`) based on `WorkKind` + `ActionKind`
  - `RecordWorkStarted` / `RecordWorkOutcome` update status + emit `WorkStarted` / `WorkCompleted|WorkFailed`
- [x] Executor consumes `WorkExecutionRequested` and records outcomes durably:
  - `primitives/integration/transformation-work-executor` processes one `WorkExecutionRequested` per Kubernetes Job, records `started/outcome`, and commits Kafka offsets only after `RecordWorkOutcome` succeeds
  - Evidence durability: when `WORKREQUESTS_MINIO_*` is configured, evidence is uploaded to object storage and recorded as stable `s3://...` refs (no pod-local paths as truth)
  - Execution substrate: the executor image runtime is devcontainer-derived (toolchain parity with developer mode)
- [x] Process workflow exists (Temporal):
  - `ToolRunWorkflow` requests a WorkRequest, waits for completion signal, and emits `ToolRunRecorded` + `ActivityTransitioned`
- [x] Capability integration pack exists (front-door tests; 430 contract):
  - `capabilities/transformation/__tests__/integration` provides:
    - E2E‑0 WorkRequest seam tests (repo-mode + cluster-mode)
    - E2E‑1 Process orchestration seam test (repo-mode)
- [x] Process ingress supports Kafka (cluster E2E wiring pending):
  - `PROCESS_INGRESS_SOURCE=kafka://` consumes domain fact topics and signals workflows; cluster orchestration remains gated until GitOps deploys ingress + dispatcher + topics
- [ ] Projection “timeline assertions” are enabled in cluster mode:
  - projection must ingest process facts + work facts reliably and expose query services without sidecar port-name collisions (Telepresence is dev-only; see gotchas)
  - `primitive-projection-transformation` image now includes `projection-transformation-relay` for deployment as a standalone ingestion workload in cluster

**Implementation status (GitOps; repo snapshot)**

- [x] KEDA installed cluster-scoped (see `backlog/585-keda.md`).
- [x] Work-queue topics provisioned via `data-kafka-workrequests-topics` (enabled in `local` + `dev`; disabled elsewhere).
- [x] Transformation contract topics provisioned via `data-kafka-transformation-contract-topics` (enabled in `local` + `dev`; disabled elsewhere).
- [x] Workbench provisioned via `workrequests-runner` (enabled in `local` + `dev`; disabled elsewhere). ExternalSecrets templates exist, but `local` + `dev` currently set `secrets.enabled=false` so the workbench can start without Vault/ExternalSecrets.
  - Note: cluster-mode WorkRequests execution typically requires credentials:
    - `WORKREQUESTS_GITHUB_TOKEN` to `git clone` private repos (or set `TRANSFORMATION_TEST_REPO_URL` to a public repo for seam tests).
    - `WORKREQUESTS_MINIO_*` for durable evidence refs (`s3://...`) when MinIO upload is required by policy.
- [x] MinIO service-user scaffolding for WorkRequests (Vault-backed) exists (enabled in `local` + `dev`; disabled elsewhere).
- [x] KEDA `ScaledJob` resources enabled in `local` + `dev` (disabled elsewhere). **Note:** `scaledJobs.maxReplicaCount: 0` is the shared default (safety); `local` + `dev` set `maxReplicaCount: 1` to allow cluster-mode WorkRequest seam tests without enabling uncontrolled fan-out. Executor image is `ghcr.io/ameideio/primitive-integration-transformation-work-executor:<tag>`.
- [ ] Runtime hardening: RBAC/NetworkPolicy per executor class (toolrun vs agentwork) and staging/production rollout posture.

### GitOps artifact inventory (what exists in `ameide-gitops`)

Components (ApplicationSet-rendered):

- Topics:
  - `environments/_shared/components/data/core/kafka-workrequests-topics/component.yaml`
  - `environments/_shared/components/data/core/kafka-transformation-contract-topics/component.yaml`
- Runner/workbench + ScaledJobs:
  - `environments/_shared/components/apps/runtime/workrequests-runner/component.yaml`

Values (layered per 434):

- Topics (shared + env toggles):
  - `sources/values/_shared/data/data-kafka-workrequests-topics.yaml`
  - `sources/values/env/local/data/data-kafka-workrequests-topics.yaml` (enabled)
  - `sources/values/env/dev/data/data-kafka-workrequests-topics.yaml` (enabled)
  - `sources/values/_shared/data/data-kafka-transformation-contract-topics.yaml`
  - `sources/values/env/local/data/data-kafka-transformation-contract-topics.yaml` (enabled)
  - `sources/values/env/dev/data/data-kafka-transformation-contract-topics.yaml` (enabled)
- Runner/workbench/ScaledJobs (shared + env toggles):
  - `sources/values/_shared/apps/workrequests-runner.yaml`
  - `sources/values/env/local/apps/workrequests-runner.yaml` (enabled + ScaledJobs enabled)
  - `sources/values/env/dev/apps/workrequests-runner.yaml` (enabled + ScaledJobs enabled)

Secrets + bootstrap fixtures:

- Vault bootstrap fixtures (local/dev):
  - `sources/values/_shared/foundation/foundation-vault-bootstrap.yaml` includes:
    - `workrequests-domain-token` (`__generate__`)
    - `workrequests-minio-access-key` (`workrequests-runner`)
    - `workrequests-minio-secret-key` (`__generate__`)
- WorkRequests runner ExternalSecrets contract:
  - `workrequests-github-token` (Vault key: `ghcr-token`, property: `value`)
  - `workrequests-domain-token` (Vault key: `workrequests-domain-token`, property: `value`)
  - `workrequests-minio-credentials` (Vault keys: `workrequests-minio-access-key` / `workrequests-minio-secret-key`, property: `value`)
- MinIO service-user provisioning integration:
  - `sources/values/_shared/data/data-minio.yaml` supports a `workrequests` service user (enabled per env in `sources/values/env/{local,dev}/data/data-minio.yaml`)

### Concrete execution queue decisions (v1)

Kafka topics (dedicated per executor class so scaling does not depend on domain fact streams):

- `transformation.work.queue.toolrun.verify.v1`
- `transformation.work.queue.toolrun.generate.v1`
- `transformation.work.queue.agentwork.coder.v1`

Inventory + naming follow-up:

- `backlog/586-workrequests-execution-queues-inventory.md` (canonical inventory + rename plan if we choose capability-namespaced topics)

Consumer groups (current GitOps scaffolding; subject to future naming convention finalization):

- `transformation-work-queue-toolrun-verify-v1`
- `transformation-work-queue-toolrun-generate-v1`
- `transformation-work-queue-agentwork-coder-v1`
- recommended (new): `transformation-work-queue-toolrun-verify-ui-harness-v1` (UI harness verification suite; separate ServiceAccount/RBAC)

### Implementation gotchas captured (so we don’t regress)

- Strimzi topic config values MUST be rendered as strings for numeric fields like `retention.ms` to avoid scientific-notation formatting (`6.048e+08`) that Kafka rejects. See `sources/values/_shared/data/data-kafka-workrequests-topics.yaml` (`retentionMs: "604800000"`).
- KEDA’s Kafka scaler runs in `keda-system`; `bootstrapServers` MUST be namespace-qualified (e.g., `kafka-kafka-bootstrap.ameide-local:9092`) so the scaler can resolve the broker Service.
- Local k3d scheduling: Kafka may require tolerations for control-plane/master taints due to PVC/node pinning in single-node clusters.
- Gateway overlay routing (E2E harness):
  - Route isolation MUST use a non-guessable run key header (recommended `X-Ameide-Run-Key=<nonce>`), not a predictable WorkRequest id.
  - Overlay `HTTPRoute` rules MUST be strictly more specific than baseline routes (host + header match) so normal traffic remains unchanged.
  - The harness MUST fail closed unless it can prove routing correctness:
    - wait for `HTTPRoute` status `Accepted=True` and `ResolvedRefs=True` (or controller equivalent),
    - prove WUT is hit by asserting an overlay-only verification marker (preferred: response header on the overlay rule) is present with the run header and absent without it.
  - Cleanup MUST be deterministic:
    - create a run anchor object (recommended `ConfigMap wut-run-<work_request_id>`) and owner-reference all shadow Deployments/Services/HTTPRoutes,
    - teardown by deleting the anchor object (K8s GC),
    - verify “no resources remain for `work_request_id`”, and record that proof as evidence,
    - add a janitor sweep for leaked anchors older than TTL.
  - Shadow workloads MUST run with config parity (env/config/secret inputs) with the baseline service; do not “copy deployment spec” at runtime.
- Legacy (dev-only): Telepresence sidecar port-name collisions
  - if the `traffic-agent` container exposes a port named `grpc`, and the Service targets `targetPort: grpc`, gRPC traffic can route to the sidecar (e.g., port `9900`) instead of the intended app container (e.g., port `50051`)
  - fix: ensure the app container port name and Service `targetPort` name match and do not conflict with the sidecar (use unique names like `tm-grpc` consistently)

### Quick verification (ameide-local)

- Argo apps:
  - `kubectl -n argocd get application local-data-kafka-workrequests-topics local-workrequests-runner`
- Topics ready:
  - `kubectl -n ameide-local get kafkatopic.kafka.strimzi.io | rg 'toolrun|agentwork'`
- KEDA ScaledJobs present/ready:
  - `kubectl -n ameide-local get scaledjob.keda.sh | rg 'workrequests'`
- Workbench deployed:
  - `kubectl -n ameide-local get deploy workrequests-workbench`
- Transformation primitive routing present:
  - `kubectl -n ameide-local get grpcroute.gateway.networking.k8s.io transformation`
- Smoke coverage:
  - `local-data-data-plane-smoke` asserts the WorkRequests execution queue KafkaTopics are present/Ready when enabled (see `backlog/588-smoke-tests-inventory.md`).

WorkRequest seam tests (developer workflow):

- repo mode (fast): `pnpm test:integration -- --filter capability-transformation --mode=repo`
- cluster mode (headless; Domain must be reachable and a verify ScaledJob must be enabled in `local`):
  - `kubectl -n ameide-local port-forward svc/transformation-v0-domain 15051:50051`
  - `INTEGRATION_MODE=cluster TRANSFORMATION_DOMAIN_GRPC_ADDRESS=localhost:15051 go test -v ./capabilities/transformation/__tests__/integration`

### Current limitation (explicit)

Enabling ScaledJobs in `local` proves the GitOps substrate and KEDA/Kafka wiring. The executor image now implements the WorkRequest consume/record/commit discipline (Kafka offsets are committed only after durable outcomes are recorded in Domain).

The remaining “end-to-end” gaps are now wiring gaps (not missing code primitives):

- Process ingress supports Kafka in CODE (`PROCESS_INGRESS_SOURCE=kafka://`), but cluster orchestration remains gated until GitOps deploys ingress + dispatcher and wires Kafka env vars.
- Telepresence traffic-agent injection is now opt-in (`agentInjector.injectPolicy=WhenEnabled`) so baseline workloads and operator-managed primitives do not get sidecars by default (interactive/dev ergonomics only; the cluster E2E harness uses Gateway API header overlays, not Telepresence).
- WorkRequests runner ScaledJobs are now enabled in `local` + `dev` with `maxReplicaCount: 1` for cluster-mode seam tests.

**Remaining activities (WP‑B completion; checklists)**

- [x] GitOps: enable `workrequests-toolrun-verify` ScaledJob in `local`/`dev` (set `maxReplicaCount > 0`) and remove any ad-hoc “*-test” ScaledJob once stable.
- [x] GitOps: make Telepresence injection opt-in (`agentInjector.injectPolicy=WhenEnabled`) so `transformation-v0-projection:50051` does not route to a sidecar by default.
- [ ] GitOps: deploy Domain dispatcher (`/app/domain-transformation-dispatcher`) so outbox topics are published to Kafka (required for ingress + KEDA queues).
- [ ] GitOps: deploy Process ingress for Kafka (`PROCESS_INGRESS_SOURCE=kafka://`, `KAFKA_BROKERS`, `KAFKA_GROUP_ID`, optional `KAFKA_TOPICS`) so workflows are signaled by facts (not by test-only helpers). (Note: `/app/process-transformation-ingress` is shipped inside the `primitive-process-transformation` image; deploy it as a separate workload via command override.)
- [ ] Tests: un-gate cluster Process orchestration tests (`TRANSFORMATION_ENABLE_PROCESS_CLUSTER_TESTS=1`) and require Projection timeline assertions (ProcessQuery + WorkQuery) in cluster mode once projection wiring is fixed.

**Test ladder (TDD: small → large)**

We implement WP‑B using a strict “small → large” ladder per `backlog/537-primitive-testing-discipline.md`. Each rung becomes a gate for the next rung; do not jump ahead to cluster E2E until the lower layers are deterministic and green.

1. **Library unit tests (fast, deterministic)**
   - `packages/ameide_coding_helpers/**` returns stable, structured results for `doctor/verify/generate/scaffold` and never depends on VS Code/UISurface.
2. **CLI contract tests**
   - `packages/ameide_core_cli/**` proves the CLI invokes coding helpers and emits stable, machine-readable outputs suitable for evidence bundles.
3. **Domain integration tests (WorkRequest as SoR)**
   - Domain can create `WorkRequest` with `client_request_id` idempotency.
   - Domain emits `WorkRequested` / `WorkCompleted` / `WorkFailed` **after persistence** (outbox discipline).
4. **Runner component tests (no Kubernetes yet)**
   - A runner “job main” consumes a `WorkExecutionRequested` execution intent (or looks up `work_request_id`), checks out a repo fixture, runs `ameide` actions, and records outcomes/evidence back to Domain.
   - Proves “ack only after durable outcome recorded” (at-least-once safe) without requiring KEDA or a real broker.
5. **Process workflow tests (Temporal test env)**
   - Workflow requests a WorkRequest (domain intent), awaits completion facts, emits `ToolRunRecorded` / `GateDecisionRecorded` deterministically.
6. **Kubernetes substrate test (KEDA + Job)**
   - In a kind-style acceptance environment, a KEDA ScaledJob schedules an executor-image Job for a `WorkExecutionRequested`, and the Job records completion/evidence in Domain; duplicates are tolerated and converge via idempotency keys.
7. **Headless end-to-end (no UISurface)**
   - Full slice: Process → Domain WorkRequest → KEDA Job → Domain evidence/outcome → Process facts → Projection timeline; assertions run via APIs/queries only.

8. **Cluster UI E2E harness (stable URLs; no Telepresence; no GitOps churn)**
   - A dedicated UI harness verification WorkRequest drives:
     - BuildKit build (changed services only),
     - ephemeral shadow Deployments/Services (no baseline Deployment mutation),
     - Gateway API overlay route(s) keyed by a run header (e.g., `X-Ameide-Run-Key=<nonce>`),
     - Playwright against stable URLs (per `backlog/430-unified-test-infrastructure-status.md`),
     - artifact capture to `/artifacts/e2e/*` and evidence recording back into Domain.
   - This rung is non-agentic and must be repeatable and self-cleaning (run anchor + ownerReferences + teardown proof in evidence).
   - Note: keep `action_kind=verify` and select this harness via `verification_suite_ref` so the domain does not accrete test-specific verbs.

### WP‑C — Cluster E2E harness (Gateway API overlays; Playwright)

Goal: add a deterministic, non-agentic E2E WorkRequest execution path that validates UISurface flows against stable URLs without preview namespaces or GitOps PRs.

- [ ] Keep `action_kind = verify` and add `verification_suite_ref` so “UI harness via gateway overlays” is a suite, not a new verb.
- [ ] Add a dedicated E2E work queue topic + consumer group + executor workload:
  - topic: `transformation.work.queue.toolrun.verify.ui_harness.v1`
  - consumer group: `transformation-work-queue-toolrun-verify-ui-harness-v1`
  - executor image has additional RBAC to create/delete shadow Deployments/Services and Gateway API `HTTPRoute` objects in a fixed namespace.
- [ ] Define a deterministic service selection manifest in-repo (no heuristic selection):
  - changed-path roots → service id
  - build metadata (BuildKit context/dockerfile/image name)
  - runtime metadata (shadow deployment template, ports, readiness probe)
  - edge routing metadata (stable hostname + gateway attachment refs; eligible for overlay routing)
- [ ] Implement E2E harness runner behavior (Job):
  - create a run anchor object and generate a non-guessable run key,
  - build changed services via BuildKit,
  - record image digests in evidence and (preferred) deploy shadow workloads by digest,
  - deploy shadow service(s) with config parity,
  - apply overlay `HTTPRoute` keyed by a per-run header,
  - wait for `HTTPRoute` readiness (`Accepted=True`, `ResolvedRefs=True`) and prove WUT routing with an overlay-only verification marker before running Playwright,
  - run Playwright (`INTEGRATION_MODE=cluster`) against the stable URL with the header injected,
  - upload `/artifacts/e2e/*` and record evidence back into Domain (artifacts + routing proof + digests + resources created/deleted + teardown proof),
  - cleanup by deleting the run anchor object and verifying no labeled resources remain.
- [ ] Add a periodic janitor sweep for leaked `wut-run-*` anchors older than TTL.

DoD:

- A single end-to-end E2E run exists for `www-ameide-platform` against the stable URL in `ameide-local`, producing Playwright artifacts.
- The run proves routing correctness (route Accepted/Resolved + marker check) and leaves no shadow workloads/routes behind after completion (teardown evidence).

Debug/admin mode (required in `local`/`dev`; not a processor):

- Provide long-lived “workbench” pods for human attach/exec using the devcontainer toolchain image (`ghcr.io/ameideio/devcontainer:<tag>`).
- It MUST be deployed only in `local` and `dev`, and MUST NOT be deployed in `staging`/`production`.
- It MUST NOT consume execution queue intents and exists only to reproduce failures and run controlled “manual reruns” that still write outcomes/evidence back into Domain idempotently.
- It MUST NOT be conflated with external “agent slots” (`agent-01`, `agent-02`, `agent-03`) used for parallel developer-mode DevContainers (clones) (see `backlog/581-parallel-ai-devcontainers.md`).

**DoD (gates)**

- A single end-to-end run exists: Process creates WorkRequest → runner executes → Domain records evidence → Process emits `ToolRunRecorded` → Projection shows timeline with citations.
- Duplicate delivery is safe: replaying the same WorkExecutionRequested message produces one canonical outcome (idempotency proven by tests/logs).
- The end-to-end run is covered by tests at the correct boundary layers (per 537):
  - Domain tests prove `WorkRequest` idempotency and fact emission-after-persistence.
  - Runner tests prove “ack only after durable outcome recorded” (at-least-once safe).
  - Process tests prove “await facts then continue” and emit process facts deterministically.
  - Projection tests prove join/materialization for the execution timeline.

#### A1) Domain primitive: TransformationKnowledgeCommandService

**Scaffold (required)**

- `go run ./packages/ameide_core_cli/cmd/ameide primitive scaffold --kind domain --name transformation --lang go --proto-path <knowledge command service proto> --include-test-harness`
- `go run ./packages/ameide_core_cli/cmd/ameide primitive scaffold --kind domain --name transformation --lang go --proto-path <knowledge command service proto> --include-test-harness`

**Progress checklist**

- [x] Schema/migrations exist for the enterprise repository substrate (repositories, nodes, assignments, elements, relationships, versions).
- [x] Transactional outbox exists and facts are emitted after persistence (`transformation.knowledge.domain.facts.v1`).
- [x] Repository scoping is enforced on writes (`{tenant_id, organization_id, repository_id}` everywhere).
- [x] Optimistic concurrency is enforced for view head updates (expected head version id / version number).
- [x] Gate: `go run ./packages/ameide_core_cli/cmd/ameide primitive verify --kind domain --name transformation --mode repo` passes.

**Implementation requirements**

- Postgres schema/migrations for the enterprise repository substrate:
  - repositories, workspace nodes, assignments
  - elements, element_relationships, element_versions
  - domain outbox (transactional) for facts
- Repository scoping enforcement:
  - every write validates `{tenant_id, organization_id, repository_id}`
  - forbid cross-repo/cross-org references (relationships, assignments, version pointers)
- Versioning semantics:
  - view edits produce `ElementVersion` snapshots and advance `head_version_id`
  - optimistic concurrency for view head updates (expected head version id / version number)

**DoD (tests + verify)**

- `cd primitives/domain/transformation && ./tests/run_integration_tests.sh`
- `go run ./packages/ameide_core_cli/cmd/ameide primitive verify --kind domain --name transformation --mode repo`

#### A2) Projection primitive: TransformationKnowledgeQueryService

**Scaffold (required)**

- `go run ./packages/ameide_core_cli/cmd/ameide primitive scaffold --kind projection --name transformation --lang go --proto-path <knowledge query service proto> --include-test-harness`

**Progress checklist**

- [x] Materialized read model tables exist (repositories/nodes/assignments/elements/relationships/versions).
- [x] Idempotency is enforced (`projection_inbox(tenant_id,message_id)`).
- [x] Query RPCs exist for MVP browse/list/get.
- [x] Outbox → projection ingestion loop exists (bridge mode; Kafka is the normative transport):
  - [x] `primitives/projection/transformation/cmd/relay` tails the domain outbox and applies facts with durable offsets.
  - [ ] Replace bridge mode with Kafka consumers per topic family once the broker wiring is the system-of-record runtime transport.
- [x] Gate: `go run ./packages/ameide_core_cli/cmd/ameide primitive verify --kind projection --name transformation --mode repo` passes.

**Ingestion (bridge mode today; Kafka target-state)**

While Kafka is the normative broker for runtime seams, the current repo uses a bridge-mode ingestion loop to preserve “projection consumes facts” before the full Kafka wiring is complete:

- Domain writes append to a **domain outbox** table in the same Postgres cluster.
- Projection maintains:
  - `projection_inbox(tenant_id, message_id)` for idempotency
  - read models (`*_views`) for browse/search
- Projection runs an ingestion loop (poll outbox → apply → mark inbox) and exposes query RPCs from its read models.

Target-state:

- Domain outbox is published to Kafka topic families.
- Projection consumes Kafka topics with idempotency (`projection_inbox`) and durable offsets.

**DoD (tests + verify)**

- `cd primitives/projection/transformation && ./tests/run_integration_tests.sh`
- `go run ./packages/ameide_core_cli/cmd/ameide primitive verify --kind projection --name transformation --mode repo`

#### A3) UISurface wiring (existing graphical editor)

**Requirements**

- Portal reads via projection query services only.
- Portal writes via domain command surfaces only.
- No fallback to legacy `TransformationService` / “typed artifact” services.

**Progress checklist**

- [x] `services/www_ameide_platform` repository pages are wired to primitives (no legacy `client.graph.*` calls).
- [x] Existing ArchiMate view editor reads via projection and saves layout via domain commands.
- [x] MVP-excluded “Transformations/initiatives” pages and API routes removed from the portal.
- [x] Gate: `pnpm -C services/www_ameide_platform run typecheck` passes.
- [ ] Gate: `pnpm -C services/www_ameide_platform test:unit` passes.
- [ ] Targeted editor e2e suite exists and passes (or equivalent).

**De-scope**

- Remove MVP-excluded UI coupling:
  - retire “Transformations” pages until governance is implemented by primitives

**DoD**

- `pnpm -C services/www_ameide_platform test:unit`
- `pnpm -C services/www_ameide_platform typecheck`
- Targeted editor e2e/interaction tests (existing suite):
  - `pnpm -C services/www_ameide_platform test:e2e:archimate` (or the closest existing e2e target)

---

### WP-Z (Deletion) — Remove legacy services/*

**Goal:** eliminate the “old/new soup”. Everything flows through primitives.

**Progress checklist**

- [x] Deleted legacy services:
  - [x] `services/graph`
  - [x] `services/repository`
  - [x] `services/transformation`
- [x] Workspace tooling updated (remove deleted workspaces).
- [x] GitOps no longer deploys/routs the deleted services anywhere under `gitops/ameide-gitops/**`:
  - Removed the legacy app components (no more `local-graph` / `local-transformation` Applications).
  - Tightened the Gateway routing so only `ameide_core_proto.graph.*` is routed to the `graph` Service; `ameide_core_proto.transformation.*` is no longer routed through `graph`.
  - Disabled legacy graph routes by default in the platform gateway values (environments must explicitly opt back in if ever needed).
  - Legacy charts remain in-tree but are documented as legacy for historical reference only.
- [ ] Portal has no `/api/*` routes that proxy to deleted services.

**Delete (required)**

- `services/graph`
- `services/repository`
- `services/transformation`

**Also update**

- `pnpm-workspace.yaml` (remove deleted workspaces)
- GitOps (stop deploying those services and stop routing their proto services through the “graph” backend)
- Any portal `/api/*` routes that proxy to deleted services

**DoD**

- `pnpm -w test:unit` (or the repo’s equivalent unit suite) remains green
- `go test ./...` remains green

---

## 6) Next plateaus (after MVP)

These are explicitly sequenced so we reintroduce governance *after* the enterprise repository substrate is proven end-to-end.

### P1 — Initiatives (governance container)

- [ ] Domain: `Initiative` aggregate exists and references repository `element_id`/`version_id` (no forked repository truth).
- Domain: `Initiative` that references repository element/version ids (does not fork repository truth).
- Projection/UI: initiative browse/detail linking into repository modeling.
- DoD: contract gates + projection tests + one e2e flow “initiative → repository view”.

### P2 — Baselines/promotions/evidence

- [ ] Domain: baseline lifecycle exists and references `{element_id, version_id}` (reproducible).
- Domain: baseline lifecycle and promotion updates “published pointers” without cascading versioning.
- Process: approval gates emitting process facts (`GateDecisionRecorded`) and step facts (`ActivityTransitioned`).
- Projection/UI: baseline timeline + diff + evidence views.

### P3 — Definition Registry + operator consumption

- [ ] Domain: Definition Registry exists (schema-backed definitions, versions, promotions).
- Domain: schema-backed `ProcessDefinition` / `AgentDefinition` / `ExtensionDefinition` with versioning/promotions.
- Process: “Design → Deploy” workflows execute promoted definitions; CLI invoked via integration runners; tool runs recorded as process facts.

### P4 — Scrum / TOGAF / PMI overlays

- [ ] Methodology overlays exist (Scrum/TOGAF/PMI) as **definitions + validations + workflow paths + UISurface profiles** over the same element graph (no forked canonical model).
- Track overlay delivery plans:
  - `backlog/527-transformation-implementation-migration-scrum.md`
  - `backlog/527-transformation-implementation-migration-togaf-adm.md`

---

## 7) Cross-references

- `backlog/520-primitives-stack-v2.md` — canonical stack and gates
- `backlog/509-proto-naming-conventions.md` — package/topic naming
- `backlog/300-400/303-elements.md` — elements-only substrate
- `backlog/521f-external-verification-baseline.md` — verify expectations
- `backlog/527-transformation-scenario-scrum.md` — end-to-end “Requirement → Release” flow (Scrum track; CLI is a tool inside the process)
- `backlog/527-transformation-scenario-togaf-adm.md` — end-to-end “Requirement → Release” flow (TOGAF ADM track; CLI is a tool inside the process)
- `backlog/527-transformation-implementation-migration-scrum.md` — Scrum overlay progress tracker
- `backlog/527-transformation-implementation-migration-togaf-adm.md` — TOGAF ADM overlay progress tracker

---

## Implementation progress (image policy)

### 2025-12-26

- As 602/603 are implemented, transformation runtime/operator deployments should move away from relying on floating `:dev` tags and `imagePullPolicy` tricks; target state is digest-pinned refs updated via PRs.
