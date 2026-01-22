## Increment 1 — Governed publish of Architecture Vision

This increment takes the Slice‑0 wiring and makes a **real governed publish loop**: **propose → commit → publish → read back at published SHA**, with an **audit‑grade Evidence Spine** owned by Domain. It is essentially “Scenario A,” but rewritten to satisfy your updated doctrine: **no optionals** and **all six primitives must advance**.  

---

# 1) Capability user story and capability-level testing objective

## User story

* **As a human (enterprise architect)**, I can draft and publish:

  * `architecture/vision.md`
  * `architecture/statement-of-work.md`
  * `requirements/REQ-TRAVEL-001.md`
    as canonical baseline content via MR-backed governance, where each file is a typed item with YAML frontmatter (minimum: `id`, `scheme: togaf`, and a TOGAF `type` key). I can see the canonical `main@sha` anchor and evidence. 
* **As an agent**, I can help draft the two documents from citeable repository context and submit a proposal through the platform seams; I cannot publish or write directly.

## Capability-level testing objective

A single **capability-owned contract-pass test** (Transformation capability; Phase 0/1/2 local, with a later Phase 4/5 cluster variant) proves, from empty state:

## Agentic deliverables (Scenario A)

Scenario A must ship clear agent-side value in addition to the UI/process path.

- **Architect agent deliverables**
  - Can draft and/or review `architecture/vision.md` and `architecture/statement-of-work.md` as proposals anchored to a resolved `read_context`.
  - Can verify Canonical `main@sha` by retrieving citeable reads at `read_context=published` and producing a citeable review summary for humans.
  - Can supervise an active developer publish run via periodic review ticks (`review_interval_minutes` per `backlog/717-ameide-agents-v6.md`) by checking: diff state, evidence (`./ameide test` where applicable), and publish anchor (`target_head_sha`).
- **Developer agent deliverables**
  - Can execute the governed write loop (Domain commands) to create/update/publish a change and attach evidence to the evidence spine.
  - Can run the verification front doors required by the slice before publish and return evidence (tests/logs).
- **Human-in-the-loop deliverables**
  - There is a required approval checkpoint before publish, and the decision is recorded into the evidence spine (who/when/what was approved).

## EDA alignment (496)

1. Onboard repository mapping for `{tenant_id, organization_id, repository_id}`. 
2. Start a durable process instance `arch.publish_vision` (or equivalent).
3. Agent produces a citeable draft proposal (citations only; no uncited “facts”).
4. Process coordinates Domain commands to:

   * `EnsureChange` (MR created or reused)
   * `CreateCommit` (writes the three files as typed TOGAF items with YAML frontmatter; Vision/SOW both reference `REQ-TRAVEL-001` via frontmatter `links:` and/or inline `ref:` tokens)
   * `PublishChange` (merges to main with SHA-safety) 
5. Domain produces an **EvidenceSpineViewModel** with **minimum Scenario A fields**, especially:

   * MR iid
   * **target branch head SHA after publish** (`target_head_sha`) as the canonical anchor 
6. Projection resolves `published` to the new `target_head_sha` and can read back both files with citations `{repository_id, commit_sha, path[, anchor]}`. 
7. UISurface demonstrates the same flow end-to-end (human draft path + “use agent draft” path) without bypassing ownership.

### Required negative assertions (in the same test pack)

* Publish fails fast if `expected_mr_head_sha` is stale (MR advanced). 
* No direct writes to `main` are possible (Domain enforces MR-backed publish). 
* CQRS rule: UI/Agent/Process read via **Projection**; Domain is **command-only** at the platform seam.
* UI/Agent/Process/Projection do not call GitLab directly; only Domain may call `gitlab.com/gitlab-org/api/client-go` (no wrappers), and GitLab types never cross the proto seam.

## Travelling requirement overlay (all increments)

Increment 1 formally “births” the travelling requirement in the governed publish loop:

* `requirements/REQ-TRAVEL-001.md` is published alongside Vision/SOW.
* `architecture/vision.md` and `architecture/statement-of-work.md` each include an explicit reference to `REQ-TRAVEL-001` (so Projection backlinks can explain the hop deterministically in later increments).

---

# 2) Primitive-by-primitive goals, build, and tests

## Domain

### Increment 1 goals

* Own canonical write boundary and **MR-backed publish lifecycle**.
* Emit/record the **Domain-owned Evidence Spine** for a publish (others must only render/consume it). 

### Build (Increment 1)

**Commands (minimum)**

* `EnsureChange(scope, idempotency_key)` → `{change_id, branch_ref, mr_iid, last_commit_id}`
* `CreateCommit(change_id, actions[], expected_last_commit_id, idempotency_key)` → `{head_commit_sha, last_commit_id}`
* `PublishChange(change_id, expected_mr_head_sha)` → `{target_branch, target_head_sha, ...supplemental}`

**GitLab substrate integration (implementation detail; not a platform contract)**

* Required operations for Scenario A:

  * get default branch head SHA
  * create branch
  * find-or-create merge request
  * create commit (with expected head guard)
  * accept/merge merge request (with expected head guard)
* Implementation uses `gitlab.com/gitlab-org/api/client-go` (pin a single major version; currently v1) directly in Domain internals (no wrapper clients; no GitLab types in Domain contracts).

**Invariants**

* Never write to `main` directly; always branch+MR. 
* Publish must be SHA-safe; mismatch → explicit error. 
* Evidence Spine canonical publish anchor = **target branch head SHA after publish** (`target_head_sha`). 
* Artifact policy (Increment 1): `architecture/vision.md` and `architecture/statement-of-work.md` must be typed items with YAML frontmatter including at least `id`, `scheme: togaf`, and `type: togaf.*`.
* GitLab SDK usage is confined to Domain internals (pin a single major version; currently v1); no GitLab client types leak across platform contracts.

### Domain tests (individual)

* **Unit:** idempotent `EnsureChange` returns same MR for same idempotency key.
* **Unit:** `PublishChange` rejects stale `expected_mr_head_sha` (fail fast).
* **Unit:** EvidenceSpineViewModel includes required fields for Scenario A:

  * identity
  * `mr_iid`
  * `target_branch`
  * `target_head_sha` (required)
  * citations of verification reads at `target_head_sha` (if Domain includes them) 
* **Static boundary:** UISurface/Agent/Process/Projection do not import `gitlab.com/gitlab-org/api/client-go` (Domain may); GitLab types never cross proto contracts.

---

## Projection

### Increment 1 goals

* Own canonical reads at an immutable SHA and converge to the new canonical `main@sha` after publish.
* Serve canonical reads from derived state built from Domain facts (outbox→Kafka), not by calling GitLab.

### Build (Increment 1)

* `ListTree(scope, read_context, path)` → nodes + citations
* `GetContent(scope, read_context, path)` → bytes/text + citation

**Convergence rule**

* After Domain publishes, `published` must resolve to the new `target_head_sha` (no stale cache). 

### Projection tests (individual)

* **Unit:** after a publish fact is applied, resolving `published` yields exactly the new `target_head_sha`.
* **Unit:** reading `architecture/vision.md` at `target_head_sha` returns the new content and citation `{repo_id, target_head_sha, path}` (served from derived state).
* **Unit:** deterministic NotFound semantics for missing paths (explicit error, no “empty content”). 

---

## Process

### Increment 1 goals

* Own the durable “publish vision” orchestration so the publish loop is not just a UI script.
* Coordinate by issuing commands to Domain and (optionally) requesting agent assistance, without writing canonical state itself. 

### Why Process is first-class here (beyond GitLab guardrails)

GitLab can guard a merge (protected `main`, approvals, required pipelines), but Increment 1 also needs a **platform-owned, durable workflow** that:

* is consistent across UISurface and MCP clients (not “whatever the UI clicked”),
* coordinates “draft → change → commit → publish” as a resumable state machine (restart-safe),
* captures governance decision metadata (who approved what) in Process state, while Domain remains the sole writer of canonical Git and the Domain-owned Evidence Spine remains the authoritative publish audit record.

### Build (Increment 1)

A persisted workflow `arch.publish_vision` (minimal states are fine, but durable):

1. `STARTED`
2. `DRAFT_READY` (manual draft or agent draft attached)
3. `CHANGE_OPENED` (Domain EnsureChange done)
4. `COMMITTED` (Domain CreateCommit done)
5. `PUBLISH_REQUESTED`
6. `PUBLISHED` (Domain PublishChange returned `target_head_sha`)
7. `FAILED` (with explicit error code)

**Hard rule**

* Process issues **commands** to Domain; it does not call GitLab directly and does not write Git. 

### Process tests (individual)

* **Unit:** workflow can resume after restart mid-way (durability).
* **Unit:** process never calls GitLab adapters directly (can be enforced by dependency injection boundary).
* **Unit:** process cannot transition to `PUBLISHED` without receiving Domain publish response containing `target_head_sha`.

---

## Agent

### Increment 1 goals

* Produce a citeable draft for the vision + statement-of-work, and submit it as a proposal into the workflow.
* Maintain **Memory as agent workflow state**: store citations used + rationale; never store uncited platform truth.

### Build (Increment 1)

* `Agent.DraftArchitectureVision(scope, read_context, inputs)`:

  * Fetches citeable context via Projection calls (through SDKs)
  * Produces:

    * `vision_md` text + citations_used
    * `statement_of_work_md` text + citations_used
* Agent “memory” store (internal to Agent primitive):

  * `Memory.append(run_id, {citations, notes, outputs})`
  * Strict “cite-only” output policy

### Agent tests (individual)

* **Unit:** output contains only citeable claims (every paragraph references at least one citation, or the agent must explicitly mark it as “assumption” and still cite why it assumed).
* **Unit:** agent cannot call Domain publish/write endpoints directly (it must submit to Process or request Domain through an explicit seam).
* **Unit:** memory entries store citations + notes; reject attempts to store arbitrary “facts.”

---

## UISurface

### Increment 1 goals

* Provide a human UX for the same governed publish loop:

  * draft (manual or agent-assisted)
  * review
  * publish
  * view evidence spine (Domain-owned)
* UI must invoke **only** Domain/Projection/Process/Agent seams; never GitLab directly. 

### Build (Increment 1)

Minimum UX flow:

1. “Start Architecture Vision publish” (starts Process)
2. Choose draft mode:

   * Manual editor for the two files **OR**
   * “Generate draft with agent” (calls Agent, displays content + citations)
3. “Submit draft” (Process advances; Process calls Domain EnsureChange/CreateCommit)
4. “Publish” (Process calls Domain PublishChange)
5. “Published” screen:

   * shows `target_head_sha` (canonical anchor)
   * shows MR iid
   * shows citations for verification reads at `target_head_sha` 

### UISurface tests (individual)

* **UI integration:** asserts UI uses Process/Domain/Projection/Agent SDKs and does not import GitLab SDK packages or internal GitLab adapter packages.
* **UI smoke:** end-to-end through the flow with local harness provider.
* **Playwright e2e (cluster, descriptive)**

  * Start the `arch.publish_vision` workflow and complete a full publish path through Process (never direct Domain publish).
  * Draft (manual or agent-assisted) `architecture/vision.md`, `architecture/statement-of-work.md`, and `requirements/REQ-TRAVEL-001.md`.
  * Verify the publish results screen shows the Evidence Spine anchor: MR iid + `target_head_sha`.
  * Verify `published` now resolves to `target_head_sha` by re-opening the three files and confirming their citations reference the same SHA.
  * Negative: attempt to publish with stale MR head SHA fails deterministically and does not “half publish.”

---

## Integration — external protocol adapters and imports/exports

### Increment 1 goals

1. Extend outward-facing protocols: **an MCP server (Integration primitive)** that exposes platform seams to coding agents.
2. Keep protocol adapters semantics-free:
   * MCP **reads** route to **Projection** queries.
   * MCP **writes** route to **Domain** commands.
3. Enforce substrate hygiene: GitLab remains a substrate behind Domain internals; no direct GitLab access from UISurface/Agent/Process/Projection/Integration.

### Build (Increment 1)

Extend the MCP server tool surface (Integration primitive) with explicit CQRS direction:

**Reads (MCP → Projection queries)**

* `repo.list_tree(...)` → Projection.ListTree
* `repo.get_content(...)` → Projection.GetContent
* `repo.get_evidence(...)` → Projection.GetEvidence (Domain-owned EvidenceSpineViewModel, materialized by Projection)

**Writes (MCP → Domain commands)**

* `repo.ensure_change(...)` → Domain.EnsureChange
* `repo.create_commit(...)` → Domain.CreateCommit
* `repo.publish_change(...)` → Domain.PublishChange

Write governance rule:

* `repo.publish_change` is not a bypass: Domain must validate required preconditions (e.g., “approval recorded for process_instance_id”) by reading Projection state.

### Integration tests (individual)

* **Unit:** MCP tools route reads to Projection and writes to Domain, and preserve deterministic errors.
* **Static boundary:** only Domain imports `gitlab.com/gitlab-org/api/client-go`; only Integration imports MCP protocol libs (Integration primitive exposing Projection queries to coding agents).

---

# 3) Increment 1 deliverables checklist

Increment 1 is complete only when:

* ✅ One capability-level contract-pass test demonstrates Scenario A end-to-end across all primitives. 
* ✅ Each primitive has at least one new behavior + one primitive test (non-null progress).
* ✅ Evidence Spine is produced by Domain and consumed by UI/Process/Agent without competing schemas. 
* ✅ GitLab boundary is enforced (Domain-only GitLab access; static import rule + wiring test).
* ✅ Both human and agent paths are exercised (manual draft path + agent draft path).
