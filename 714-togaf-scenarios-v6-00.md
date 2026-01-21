---
title: "714 — v6 Scenario Slices v6-00 (Slice 0: Plumbing + contract harness scaffolding)"
status: draft
owners:
  - platform
  - transformation
  - repository
  - ui
  - agents
created: 2026-01-21
updated: 2026-01-21
related:
  - 714-togaf-scenarios-v6.md
  - 537-primitive-testing-discipline.md
  - 694-elements-gitlab-v6.md
  - 710-gitlab-api-token-contract.md
---

# 714 v6-00 — Slice 0 scaffolding (plumbing + contract harness)

Slice 0 exists to make the v6 program **implementable**: it scaffolds the plumbing and the contract harness so subsequent slices can add behavior without re-litigating wiring, identity propagation, or evidence/report shapes.

Slice 0 is **not** a user capability and does not try to “ship product value” directly. Its output is:

* a runnable **contract-pass** runner in `ameide test` (Phase 0/1/2),
* a skeleton set of services/primitives that can be invoked end-to-end,
* a valid `EvidenceSpineViewModel` emitted on every run (even if fields are empty where not applicable).

## What Slice 0 proves (v6 posture, minimal)

* Identity plumbing is consistent across all calls/events/logs: `{tenant_id, organization_id, repository_id}`.
* `read_context` exists and resolves to a deterministic commit SHA in the local runner (fake store).
* All reads and derived outputs are citation-grade (even in fakes): `{repository_id, commit_sha, path[, anchor]}`.
* The Evidence Spine is a **shared view model** (not “test-only output”).
* The Element Editor surface exists as a shell (optional), but persistence is not implemented here.

## EDA alignment (496)

Slice 0 scaffolds the plumbing so later slices can adopt the full evented posture from `backlog/496-eda-principles-v6.md` without refactoring.

In Slice 0 (local-only):

* Prefer RPC-like calls between primitives to prove the seams (Commands/Queries).
* Still treat interactions using 496 terms:
  * Domain endpoints are **Commands** (even if they are “no-op” in Slice 0).
  * Process/Agent can issue Commands (or Intents later) but never write canonical state directly.
  * Projection/Memory are **derived read models**.
* Even in fakes, preserve the required *metadata disciplines*:
  * idempotency keys on commands where relevant,
  * correlation/causation metadata propagation (at least in request context / logs).

In later cluster phases:

* Owners emit **Facts** only after commit (outbox / outbox-equivalent).
* Facts/intents move over Kafka using CloudEvents + Protobuf and `io.ameide.*` types per 496.

## Non-goals

* No GitLab required (local-only fakes are allowed and expected).
* No real MR/publish workflow (that is Slice 2 / Scenario A).
* No derived backlinks/impact parsing (that is Slice 3 / Scenario B).
* No cluster integration (that is `ameide test cluster` in Phase 3/4).

## Capability tags

Slice 0 covers:

* `cap:identity`
* `cap:citation`
* `cap:evidence.spine`
* `cap:repo.onboard` (local-only mapping; enough to exercise the contract)
* `cap:repo.read_tree` / `cap:repo.read_file` (fake repo store, not GitLab)

And introduces an internal enabling tag used only to coordinate scaffolding:

* `cap:plumbing.scaffold` (service wiring + harness; not a product feature)

## Capability-level test scaffolding (590/591)

Slice 0 must scaffold not only “scenario runners”, but also the **capability-owned test pack shape** so later slices land tests in the right place:

* Tests live under the repo’s `capabilities/` composition boundary (per `backlog/590-capabilities.md`), not scattered across primitive-kind folders.
* Slice 0 should introduce (or validate the existence of) a Transformation capability test pack folder (exact naming may vary by repo evolution), e.g.:
  * `capabilities/transformation/__tests__/integration/` (target state from 590), or
  * `capabilities/transformation/features/.../integration/` (current common pattern in this repo)
* Slice 0 DoD includes at least one runnable “plumbing test” in that pack that:
  * exercises the Slice 0 harness end-to-end (Domain/Projection/Memory/Process/Agent stubs),
  * produces an `EvidenceSpineViewModel`,
  * and fails fast on missing identity/citation discipline.

## What to implement by primitive (scaffolding only)

### Repository onboarding (platform capability)

Minimum:
* A single onboarding contract exists and can be called by the runner:
  * `UpsertRepositoryMapping(identity, provider, remote_id, published_ref)`
* For Slice 0, `remote_id` can be a fake identifier into an in-memory repo store.

### Domain primitive (stubbed write boundary)

Minimum:
* Domain process/service starts and can be called.
* Domain enforces the credential boundary at the interface level:
  * callers do not provide vendor credentials
  * Domain would be the only holder of write credentials later (out of scope here)
* For Slice 0, Domain may expose only a “no-op” command used by the runner:
  * `Domain.Ping(identity)` → returns `{ok:true}` (or equivalent)

### Projection primitive (read surface using a fake repo store)

Minimum:
* Projection process/service starts and can be called.
* Provide:
  * `ResolveReadContext(identity, read_context)` → `resolved.commit_sha`
  * `ListTree(identity, read_context, path)` → nodes + citations
  * `GetContent(identity, read_context, path)` → bytes + citation
* Deterministic errors:
  * missing path results in a single explicit error outcome (mirrors GitLab 404 semantics)

### Memory primitive (projection-owned context retrieval stub)

Minimum:
* Memory process/service starts and can be called.
* Provide:
  * `GetContextBundle(identity, read_context, selector, limits)` → citations (+ optional excerpts)
* Memory returns **only** citeable outputs (no uncited facts).

### Process primitive (orchestration stub)

Minimum:
* Process runtime (or a stub service) starts and can be called.
* Provide:
  * `StartProcess(identity, process_key, inputs)` → `process_instance_id`
* For Slice 0, it may not execute anything; it exists to prove wiring and identity propagation.

### Agent primitive (proposal/evidence stub)

Minimum:
* Agent entrypoint exists and can be called.
* Provide:
  * `Agent.Propose(identity, goal, context_bundle)` → citeable “proposal” output (text + citations)
* Agent must not write; it can only produce proposal/evidence.

### UI (optional in Slice 0)

Slice 0 does not require UI, but if we include a smoke-level surface it should be:

* A repository page shell (`<ListPageLayout />`) that can render a fake tree.
* An element editor shell (`<ElementEditorModal />` + `<EditorModalChrome />` + `<RightSidebarTabs />`) that can render:
  * `Storage: <path>`
  * `Read: <context> @ <sha>`
  * `Citation: {repository_id, commit_sha, path}`
* Persistence actions (“Save draft” / “Publish”) are hidden or disabled.

## Contract-pass runner (DoD)

Runner entry:
* `ameide test` includes Slice 0 as a runnable scenario.

Steps:
1. Create identity `{tenant_id, organization_id, repository_id}`.
2. Onboard mapping to a fake provider/repo store.
3. Projection resolves `published` → `resolved.commit_sha`.
4. Projection lists root tree and reads one file.
5. Memory returns a citeable context bundle for that file (optional excerpt).
6. Process stub starts and returns an instance id (no execution required).
7. Agent stub returns a citeable proposal string referencing the context bundle citations.
8. Emit `EvidenceSpineViewModel`:
   * must include `identity`, `repository` mapping, `citations`, and `summary`.
   * proposal/publish fields may be absent/empty in Slice 0.

Assertions:
* Every response includes identity (directly or via request context).
* Every read/context item is citeable to `{repository_id, commit_sha, path[, anchor]}`.
* Errors are deterministic for:
  * unknown repository mapping
  * unknown path
  * invalid read context
