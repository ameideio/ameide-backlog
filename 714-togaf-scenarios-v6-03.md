# Increment 3 — Proposal vs Published governance gate with impact preview

This increment corresponds to the “governed change with blast radius visibility” idea (what your earlier scenario set called **Scenario C**), built on top of:

* governed publish + audit anchor rules from Scenario A (MR is the proposal unit; canonical publish anchor is **target branch head SHA after publish**) 
* derived backlinks + “why linked?” origin citations from Scenario B (inline-only refs, projection-owned, deterministic broken/duplicate behavior) 
* Slice 0 contract-pass discipline (identity/read_context/citations/evidence view model) 

It also introduces a key new capability: **two read contexts** for derived views:

* **Published** baseline
* **Proposal head** (MR head)

Scenario B explicitly notes proposal-head reads are not required there but should not be prevented by the model—this increment makes them real. 

---

## 1) Capability user story and capability-level testing objectives

### User story (human and agent equally important)

* **As governance (human)**, I can review a proposed change to a travelling requirement (`REQ-TRAVEL-001`) and see a **derived blast radius** (what artifacts reference it) **for both Published and Proposal**, approve it, and publish it—without bypassing the governed publish loop.
* **As an agent**, I can generate an “impact preview” evidence packet comparing Published vs Proposal, using only citeable outputs (no uncited facts), and attach it to the governance review.

### Capability-level testing objective (contract-pass, vertical slice)

A **capability-owned integration test** proves from empty state:

1. **Baseline publish (Published A)**
   Domain publishes two canonical artifacts (governed MR loop):

   * `requirements/REQ-TRAVEL-001.md` as a typed TOGAF item (YAML frontmatter required): `id: REQ-TRAVEL-001`, `scheme: togaf`, `type: togaf.requirement`
   * `architecture/standards/standard-a.md` as a typed TOGAF item (YAML frontmatter required) that references `REQ-TRAVEL-001`:
     * preferred: frontmatter `links:` entries with `target_id: REQ-TRAVEL-001`
     * compatibility: inline `ref: REQ-TRAVEL-001`
     Capture `published_sha_A` (the `target_head_sha` from Domain evidence). 

2. **Open proposal (MR) and modify blast radius (Proposal head)**
   Domain creates a proposal (EnsureChange/CreateCommit) that:

   * edits `requirements/REQ-TRAVEL-001.md` (content change)
   * adds `architecture/standards/standard-b.md` (typed TOGAF item) referencing `REQ-TRAVEL-001` (blast radius expands) 

3. **Projection supports dual context queries**
   Projection can answer backlinks/impact for `REQ-TRAVEL-001` at:

   * `read_context = published` → returns **1** backlink (standard-a) with origin citations
   * `read_context = proposal(change_id | mr_iid)` → returns **2** backlinks (standard-a + standard-b) with origin citations

4. **Agent produces an impact preview evidence packet**
   Agent calls Projection in both contexts and produces:

   * a short “what changes” summary
   * citeable lists of backlinks at A and proposal head
   * citations for “why linked?” origins (not just file paths)

5. **Process enforces the governance gate**
   Process workflow requires:

   * impact preview packet exists (from Agent)
   * at least one human approval recorded
   * only then does Process call Domain `PublishChange` with the expected MR head SHA

6. **Publish and convergence (Published B)**
   Domain publishes (merges) and emits EvidenceSpineViewModel including:

   * MR iid
   * `target_head_sha` (**required canonical anchor**) 
     Projection’s `published` resolves to `published_sha_B == target_head_sha` and backlinks now return **2** at published.

#### Required negative assertions (same test pack)

* Attempt to publish without approval or without an impact preview fails deterministically (Process prevents; Domain publish endpoint is not callable by UISurface/Agent identity).
* Publish fails fast if MR head SHA changed since approval snapshot (Domain SHA guard remains authoritative). 
* CQRS rule: UI/Agent/Process read via **Projection**; Domain is **command-only** at the platform seam.
* UI/Agent/Process/Projection do not call GitLab APIs directly; only Domain uses `gitlab.com/gitlab-org/api/client-go`.

---

## 2) Primitive-by-primitive goals, what to build, and what to test

### Domain

#### Increment 3 goals

1. Keep canonical write ownership and SHA-safe publish. 
2. Support “proposal addressing” robustly enough for Projection to query MR head state without exposing GitLab APIs as platform semantics.
3. Ensure publish is **gateable** (so Process can be the durable governor) without creating a second “evidence spine” schema outside Domain.

#### Build

* Keep Scenario A command surface:

  * `EnsureChange → CreateCommit → PublishChange` with SHA guards. 
* Add **change introspection** (authoritative outcomes/evidence surface, not “canonical reads”):

  * `GetChange(scope, change_id)` → returns `{mr_iid, branch_ref, last_known_mr_head_sha (best-effort)}`
* Add **caller policy** on commands:

  * `PublishChange` callable only by **Process principal** (service-to-service auth), not by UISurface/Agent principals.
  * `EnsureChange/CreateCommit` callable by UISurface and (optionally) Agent under grants, but still Domain-owned.

> This is the cleanest way to make the Process gate non-bypassable without forcing Domain↔Process circular calls.

#### Domain individual tests

* **Unit:** `PublishChange` rejects calls from non-Process principals.
* **Unit:** publish rejects stale `expected_mr_head_sha` (already required). 
* **Unit:** EvidenceSpineViewModel remains Domain-owned and includes required publish anchor `target_head_sha`. 
* **Unit:** `GetChange` does not leak GitLab project IDs or vendor URLs as semantics (opaque only).

---

### Projection

#### Increment 3 goals

1. Add **proposal head read_context** resolution and deterministic queries for both contexts.
2. Derived backlinks/impact must be computed at any resolved SHA (published or proposal head) and remain rebuildable and citeable. 

#### Build

* Extend Projection `read_context` handling on all relevant queries to support:

  * `published` (canonical `main@sha`)
  * `version_ref.commit_sha`
  * `proposal_ref.change_id` (or `proposal_ref.mr_iid`) → resolves to MR head SHA (from Domain facts)
* Ensure derived indexing can run at an arbitrary SHA and serve:

  * `ListBacklinks(target_id, read_context)` including origin citations with anchors (fm:* / L:C strategy from Scenario B MVP). 
* Add a comparison helper (optional as an API, **not optional as a capability**):

  * `CompareBacklinks(target_id, published, proposal)` → returns `{added[], removed[], unchanged[]}` each with origin citations

#### Projection individual tests

* **Unit:** `ListBacklinks(..., proposal_ref)` resolves to the correct proposal head SHA (via consumed Domain facts).
* **Unit:** `ListBacklinks(REQ-TRAVEL-001, published)` and `ListBacklinks(REQ-TRAVEL-001, proposal)` differ deterministically when proposal adds/removes refs.
* **Unit:** `CompareBacklinks` returns correct added backlink (standard-b) with origin citations.
* **Rebuildability:** indexing the same SHA twice yields identical answers.

---

### Process

#### Increment 3 goals

1. Own a durable **governance review gate** that is explicitly non-bypassable.
2. Orchestrate “impact preview → approval → publish” while never writing canonical Git state itself.

#### Why Process is first-class here (beyond GitLab guardrails)

GitLab can protect `main` and require approvals/pipelines to merge, but it cannot express or enforce Ameide’s platform-specific governance prerequisites across **all channels** (UISurface + MCP + any future client), such as:

* “Impact preview packet exists” (Agent-produced evidence attached to the review)
* “Approval is recorded for the exact `proposal_head_sha` being published” (approval snapshot)
* timers/escalations and resumability of the review workflow (durable state machine)

Increment 3 requires Process so governance is not “whatever the UI did” and not bypassable by calling Domain commands directly.

#### Build

A persisted workflow `req.change_review` (or general `governance.review_gate`) with minimum states:

1. `STARTED`
2. `DRAFT_READY` (change exists: `change_id`, MR iid known)
3. `IMPACT_PREVIEW_READY` (agent packet attached)
4. `APPROVED` (human approval(s) recorded)
5. `PUBLISHING` (calls Domain.PublishChange with expected MR head SHA)
6. `PUBLISHED` (records Domain `target_head_sha`)
7. `FAILED` (deterministic failure code)

Rules:

* must not transition to `PUBLISHING` unless:

  * impact preview exists
  * approvals exist
* records correlation/causation metadata in state transitions (aligns with Slice 0 discipline). 

#### Process individual tests

* **Unit:** cannot publish without both impact preview + approval.
* **Unit:** resumes correctly after restart (durable).
* **Unit:** calls Domain.PublishChange only once with idempotency key; duplicate attempts are deterministic.
* **Unit:** records the approval snapshot head SHA and passes it as `expected_mr_head_sha` to Domain.

---

### Agent

#### Increment 3 goals

1. Produce an **impact preview evidence packet** comparing Published vs Proposal for a target item (`REQ-TRAVEL-001`).
2. Memory remains an Agent workflow concern: store citeable bundles and the produced packet.

#### Build

* `Agent.GenerateImpactPreview(scope, target_id, published_context, proposal_context)`:

  1. call Projection backlinks at published
  2. call Projection backlinks at proposal head
  3. call Projection “why linked?” origin snippets (optional but recommended for high trust)
  4. produce packet:

     * summary text
     * `added_refs[]`, `removed_refs[]`
     * citations_used[]
* Attach packet to Process instance (Process owns durable orchestration state; Agent does not invent an evidence spine schema).

#### Agent individual tests

* **Unit:** packet contains only citeable statements (citations_used required; no uncited facts).
* **Unit:** handles deterministic error cases:

  * ambiguous IDs (duplicate id) → reports ambiguity with candidate citations 
  * broken refs → reports unresolved targets with origin citations 
* **Unit:** cannot call Domain.PublishChange directly (even if it can call EnsureChange/CreateCommit under grants).

---

### UISurface

#### Increment 3 goals

1. Provide a **review UI** that makes Published vs Proposal explicit and usable:

   * show derived impact/backlinks for both contexts
   * show “added/removed” (impact preview)
2. Provide **human approval** action tied to Process (durable record).
3. Provide “Publish” action that goes through Process (not directly to Domain Publish).

#### Build

Minimum UX flow:

* “Open Requirement REQ‑TRAVEL‑001”
* “Start change” → Domain.EnsureChange
* Editor view (proposal) for REQ‑TRAVEL‑001
* Review screen:

  * context switcher: Published @ sha_A vs Proposal @ sha_head
  * derived backlinks panel with diffs
  * shows agent impact preview packet (with citations)
* Approve button:

  * calls Process “RecordApproval”
* Publish button:

  * calls Process “RequestPublish” (Process calls Domain.PublishChange)

#### UISurface individual tests

* **UI integration:** publish path never calls Domain.PublishChange; only Process endpoint.
* **UI smoke:** shows both contexts and their resolved SHAs; renders added backlink row with “why linked?” origin citation.
* **UI negative:** publish disabled until approval + impact preview exist (reflecting Process state).

---

### Integration

#### Increment 3 goals

1. Add the Domain facts required to resolve and read proposal head SHAs without making GitLab an API surface.
2. Expose protocol adapters: **an MCP server (Integration primitive)** where:
   * MCP reads route to Projection queries (impact preview, governance status, citeable diffs),
   * MCP writes route to Domain/Process commands, with **publish remaining non-bypassable**:
     * `EnsureChange/CreateCommit` can be invoked via Domain commands (under grants),
     * `PublishChange` remains callable only by the **Process principal** (and/or requires gate satisfaction validated via Projection state).

#### Build

* Extend Domain’s GitLab usage (client-go) + emitted facts with:

  * `get_mr_head_sha(remote_id, mr_iid)` (used by Projection read_context resolution)
  * `read_file_at_sha(remote_id, sha, path)`
* Ensure **only Domain** talks to GitLab REST; Projection is fed from Domain facts (Kafka/outbox).
* Extend MCP server tool set (non-semantic passthrough):

  * `impact_preview(target_id, published|proposal_ref)` routing to Projection/Agent
  * `governance.status(process_instance_id)` routing to Projection (materialized from Process facts)

#### Integration individual tests

* **Unit:** adapter returns deterministic MR head SHA (in-memory provider must emulate).
* **Unit:** protocol adapter returns citeable outputs unchanged (citations from Projection).
* **Static boundary:** `gitlab.com/gitlab-org/api/client-go` imports only within Domain.

---

## Increment 3 “Done means done”

Increment 3 completes only when:

* ✅ One capability-level contract-pass test proves:

  * published vs proposal dual-context derived impact
  * agent impact preview packet
  * process approval gate + non-bypassable publish
  * domain evidence spine with canonical publish anchor `target_head_sha` 
* ✅ Each primitive has at least one new behavior + one primitive-owned test.
* ✅ No competing evidence spine schema exists; approvals and impact packets live in Process state and are rendered in UI, while Domain EvidenceSpine remains the authoritative publish audit record. 
* ✅ GitLab remains purely a substrate behind Domain internals.

---

If you reply **“next”**, I’ll do **Increment 4**: “Requirement → Work Package → Delivery trace” where the derived graph begins to cross from EA artifacts into code/GitOps/process assets (still Git-first, still projection-derived, still governed publish), and again each primitive advances non-null.
