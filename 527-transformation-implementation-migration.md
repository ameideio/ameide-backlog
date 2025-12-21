# 527 Transformation — Implementation Plan (target-state, no migration shims)

**Status:** Draft (implementation in progress)  
**Parent:** `backlog/527-transformation-capability.md`

This document is the **Implementation & Migration** layer plan for delivering the Transformation capability. Because we are **not live**, “migration” here means **implementation slicing + deletion of legacy code**, not compatibility shims, traffic splitting, or façade-first routing.

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

## 0.1) Implementation progress (repo snapshot; checklist)

This section is a lightweight status tracker against the work packages below.

- [x] WP-A1 Domain: enterprise repository write model implemented (`primitives/domain/transformation`).
- [x] WP-A2 Projection: enterprise repository read model implemented (`primitives/projection/transformation`).
- [x] WP-A2 Ingestion: outbox → projection relay implemented (`primitives/projection/transformation/cmd/relay`).
- [x] WP-A3 UISurface: existing ArchiMate editor wired to primitives (`services/www_ameide_platform`).
- [x] WP-Z Deletion: legacy `services/*` backends removed (`services/graph`, `services/repository`, `services/transformation`).
- [ ] WP-0 Repo health: confirm repo-wide codegen drift gates are green and enforceable in CI (regen-diff).
- [ ] GitOps parity: add checked-in GitOps components for Process/Agent/UISurface primitives in `gitops/primitives/**` and update guardrails to reference them consistently.

Gates (currently passing):

- `go run ./packages/ameide_core_cli/cmd/ameide primitive verify --kind domain --name transformation --mode repo`
- `go run ./packages/ameide_core_cli/cmd/ameide primitive verify --kind projection --name transformation --mode repo`
- `pnpm -C services/www_ameide_platform run typecheck`

## 1) Alignment to 520 (normative constraints)

Cross-reference: `backlog/520-primitives-stack-v2.md`.

We treat 520 as non-negotiable for this plan:

- **Proto is the behavior schema.** Services/messages/events are defined in protos; SDKs are generated from protos.
- **`buf generate` is canonical.** The CLI orchestrates scaffolding/verify; it does not replace Buf plugins.
- **Generated outputs are clobber-safe.** Implementation-owned code lives outside generated-only roots.
- **Guardrails are gates.** “Done” means regen-diff is clean, tests are green, and `go run ./packages/ameide_core_cli/cmd/ameide primitive verify` is meaningful.

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
  - `ElementAssignment` (node → element, optional pinned version)

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

Topic families follow 509:

- `transformation.domain.intents.v1`
- `transformation.domain.facts.v1`
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

**Deliverables**

- Domain:
  - `WorkRequest` record + idempotency posture (`client_request_id`)
  - Domain facts: `WorkRequested`/`WorkCompleted`/`WorkFailed` (after persistence)
  - Intent to record outcomes with evidence bundle linking (idempotent)
- Integration (runner backend):
  - runner implementation that consumes `WorkRequested` facts and executes one WorkRequest per run (KEDA ScaledJob → devcontainer-derived Kubernetes Job)
  - evidence bundle creation (logs + artifacts) + idempotent outcome recording in Domain
  - KEDA ScaledJob (normative) with timeouts/retries/history limits (cleanup via Job TTL only)
- Process:
  - at least one workflow that requests a WorkRequest and awaits completion
  - emits `ToolRunRecorded` + step transitions (`ActivityTransitioned`) with correlation to the WorkRequest
- Projection:
  - materialize a run timeline that joins process facts to WorkRequest lifecycle facts with citations
- UISurface:
  - minimal “Execution timeline” view: WorkRequests + evidence links + gate decisions

**DoD (gates)**

- A single end-to-end run exists: Process creates WorkRequest → runner executes → Domain records evidence → Process emits `ToolRunRecorded` → Projection shows timeline with citations.
- Duplicate delivery is safe: replaying the same WorkRequested message produces one canonical outcome (idempotency proven by tests/logs).

#### A1) Domain primitive: TransformationKnowledgeCommandService

**Scaffold (required)**

- `go run ./packages/ameide_core_cli/cmd/ameide primitive scaffold --kind domain --name transformation --lang go --proto-path <knowledge command service proto> --include-test-harness`
- `go run ./packages/ameide_core_cli/cmd/ameide primitive scaffold --kind domain --name transformation --lang go --proto-path <knowledge command service proto> --include-test-harness`

**Progress checklist**

- [x] Schema/migrations exist for the enterprise repository substrate (repositories, nodes, assignments, elements, relationships, versions).
- [x] Transactional outbox exists and facts are emitted after persistence (`transformation.knowledge.domain.facts.v1`).
- [x] Repository scoping is enforced on writes (`{tenant_id, organization_id, repository_id}` everywhere).
- [ ] Optimistic concurrency is enforced for view head updates (expected head version id / version number).
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
- [x] Outbox → projection ingestion loop exists (no broker dependency):
  - [x] `primitives/projection/transformation/cmd/relay` tails the domain outbox and applies facts with durable offsets.
- [x] Gate: `go run ./packages/ameide_core_cli/cmd/ameide primitive verify --kind projection --name transformation --mode repo` passes.

**Ingestion (target-state, minimal infra)**

To avoid introducing a message broker dependency in MVP while still preserving “projection consumes facts”:

- Domain writes append to a **domain outbox** table in the same Postgres cluster.
- Projection maintains:
  - `projection_inbox(tenant_id, message_id)` for idempotency
  - read models (`*_views`) for browse/search
- Projection runs an ingestion loop (poll outbox → apply → mark inbox) and exposes query RPCs from its read models.

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
- [ ] GitOps no longer deploys/routs the deleted services anywhere under `gitops/ameide-gitops/**`.
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

- [ ] Separate profile domains exist (Scrum/TOGAF/PMI) and govern the same repository (no forked canonical model).
- Separate profile domains/processes that govern the same repository (no forked canonical model).

---

## 7) Cross-references

- `backlog/520-primitives-stack-v2.md` — canonical stack and gates
- `backlog/509-proto-naming-conventions.md` — package/topic naming
- `backlog/300-400/303-elements.md` — elements-only substrate
- `backlog/521f-external-verification-baseline.md` — verify expectations
- `backlog/527-transformation-scenario.md` — end-to-end “Design → Deploy” narrative (CLI is a tool inside the process)
