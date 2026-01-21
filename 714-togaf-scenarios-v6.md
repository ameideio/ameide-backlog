---
title: "714 — v6 Scenario Slices (Product ladder + incremental cross‑primitive implementation & test spec)"
status: draft
owners:
  - platform
  - transformation
  - ui
  - repository
  - agents
created: 2026-01-21
updated: 2026-01-21
supersedes:
  - 713-v6-togaf-functional-scenarios-e2e-tests.md
related:
  - 496-eda-principles-v6.md
  - 520-primitives-stack-v6.md
  - 537-primitive-testing-discipline.md
  - 656-agentic-memory-v6.md
  - 694-elements-gitlab-v6.md
  - 701-repository-ui-enterprise-repository-v6.md
  - 703-transformation-enterprise-repository-proto-v1.md
  - 704-v6-enterprise-repository-memory-e2e-implementation-plan.md
  - 705-transformation-agentic-alignment-v6.md
  - 710-gitlab-api-token-contract.md
---

# 714 — v6 Scenario Slices

## 1. Purpose

This document defines a **single approach** that replaces both:

* the “coordination ladder” intent (historically 706), and
* **713** (TOGAF-ish functional scenarios / E2E tests)

by introducing **Scenario Slices**: *real-life, user-valued journeys* that are also the **incremental implementation plan** and the **incremental test plan**.

A Scenario Slice is:

* a **product capability** you can demonstrate to real users,
* implemented by **all required primitives together** (Domain / Projection / Repo substrate / Process / Agents / UI),
* validated by a **cross-primitive E2E test** that produces an **audit-grade run report**,
* constrained by v6 invariants (Git-first, owner-only writes, inline-only relationships, derived-only views, GitLab CE only).

This turns “increment planning” and “scenario testing” into the same thing.

---

## 2. Principles (v6 invariants, product-oriented)

These apply to every slice.

### 2.1 Canonical truth split

* **Content truth** is Git: authored artifacts are files at an immutable **commit SHA**.
* **Governance truth** is platform DB: approvals, outcomes, waivers, process state.
* **View truth** is projection: graphs/backlinks/search/memory are **derived**, rebuildable, citation-grade.

### 2.2 Git-tree-first UX

* Repository navigation hierarchy is the **Git tree at a specific commit SHA**, not a synthetic workspace tree.
* “EA views” (catalogs, matrices, roadmaps, impacts) are **derived overlays**.

### 2.3 Inline-only relationships

* Relationships exist only as inline references **inside authored files**.
* No relationship CRUD; no relationship sidecar store. Projection may index edges/backlinks but **never becomes canonical**.

### 2.4 Owner-only writes

* Only the owning **Domain** performs canonical writes (branch/MR/commit/merge).
* Agents/Process/UI never hold GitLab write credentials; they express intent via Domain commands.

### 2.5 GitLab CE substrate only

* Depend only on GitLab **Free-tier compatible** primitives (self-managed “CE/Free” posture): repository storage, Merge Requests as proposal units, pipelines as validation, protected branches, and the public GitLab HTTP APIs for commits/files/tree/compare/merge.
* Do not depend on paid-tier planning and governance features (e.g., approval rules); keep governance enforcement in platform policy + owner-only writes.

### 2.6 Audit-grade by default

Every user-visible claim must be explainable as:

* “Here’s the file(s), at this repo, at this commit SHA (and anchor).”

That’s not hardening; it’s the **core product trust feature**.

### 2.7 Element-editor-first UX (no “raw file viewer” as the product surface)

* Artifacts are opened and edited through an **Element Editor** surface (modal preferred), not a standalone “file viewer page.”
* The Element Editor must still make the underlying canonical storage explicit:
  * storage path (e.g. `architecture/vision.md`)
  * read context and resolved commit SHA (e.g. `Published @ <sha>`)
  * citation object `{repository_id, commit_sha, path[, anchor]}`
* Editing is **change-based** and persists via the v6 write loop (Domain commands), not DB CRUD.

---

## 3. What a Scenario Slice is

Each slice is a **vertical slice** that must deliver:

1. **User value** (a real TOGAF-ish capability),
2. **A minimal usable UX surface** (even if thin),
3. **A cross-primitive E2E runner** that exercises the same contracts the UX uses,
4. **An evidence spine** (run report) tying together: identity → proposal → publish → citations → governance outcomes (if applicable).

### 3.1 Slice definition template

Every slice has the same shape:

* **Name**
* **Primary user story** (EA language)
* **Value proposition** (what becomes possible, who benefits)
* **Journey** (baseline → proposal → review → publish → derived views)
* **Repo artifacts** (paths + minimal content requirements)
* **Primitive responsibilities**

  * GitLab substrate
  * Domain
  * Projection
  * UI
  * Memory (projection-owned derived view)
  * Process (optional)
  * Agent (optional)
* **Definition of Done**

  * Contract-pass (API-level E2E)
  * UX-pass (optional early; mandatory for Tier‑1 slices)
* **Run report requirements** (evidence spine outputs)

---

## 4. Capability tags (alignment mechanism)

Instead of referencing “Inc 3 / Inc 4” etc., every slice declares **capability tags**. These tags are stable even if slice ordering changes.

Core capability tags:

* `cap:identity`
* `cap:citation`
* `cap:repo.onboard`
* `cap:repo.read_tree`
* `cap:repo.read_file`
* `cap:change.ensure`
* `cap:change.commit_batch`
* `cap:change.publish`
* `cap:evidence.spine`
* `cap:proposal.read_context` (MR head / proposal SHA)
* `cap:derived.backlinks`
* `cap:derived.impact`
* `cap:memory.citeable_context`
* `cap:baseline.compare`
* `cap:governance.outcome`
* `cap:process.definition_files`
* `cap:process.deploy_by_sha`
* `cap:transformation.run_e2e` (process + agent + publish)

Each primitive maintains a “capability support” checklist per tag (small, explicit).

---

## 5. Test runner spec (product E2E, not “hardening”)

### 5.1 Two front doors, one scenario definition

Scenarios are defined once, but can run through two contracts only (per `backlog/537-primitive-testing-discipline.md`):

* **`ameide test` (Phase 0/1/2)**: local-only, self-contained runs using in-process fakes where needed (e.g. Fake GitLab HTTP API), no cluster dependencies.
* **`ameide test cluster` (Phase 3/4)**: real dev/local cluster integration (real GitLab + real cluster plumbing) when the cluster is available.

Both modes must produce the **same run report schema** so product value is consistent.

### 5.2 Run report schema (evidence spine)

Every run produces an audit-grade report:

* **Identity**: `{tenant_id, organization_id, repository_id}`
* **Repository mapping**: internal pointer (opaque), never shown as canonical
* **Proposal evidence**:

  * MR identifier (iid or equivalent)
  * proposal head commit SHA (if relevant; may require waiting for MR diff refs to populate)
  * pipeline pointer (if used)
* **Publish evidence**:

  * target branch (typically `main`)
  * target head commit SHA (“published baseline”; the audit anchor you show everywhere)
  * optional: merge/squash commit SHAs (if provided by the vendor)
* **Citations used/returned**:

  * list of `{repository_id, commit_sha, path[, anchor]}` referenced in reads, derived views, memory outputs
* **Governance evidence** (if applicable):

  * approvals/outcome records IDs + linkage to MR + published SHA
* **Derived view evidence** (if applicable):

  * backlink/impact query results + origin citations
* **Summary**:

  * “What capability was demonstrated?” in EA language (one paragraph)

### 5.3 Definition of Done levels

To avoid “tests without product,” each slice has two pass levels:

* **Contract-pass** (mandatory for all slices): E2E runner passes and produces run report.
* **UX-pass** (mandatory for Tier‑1 slices, optional early for others): minimal UI workflow smoke aligns with the same contracts and produces the same evidence spine.

### 5.4 Vendor edge cases and limits (GitLab)

These are non-negotiable realities the contract-pass runner and the primitives must handle explicitly:

* **Merge-method nuance:** the canonical publish anchor is the **target branch head SHA after publish**; merge/squash commit SHAs are supplemental evidence (project settings may vary).
* **MR proposal SHA availability:** proposal diff/head fields can populate asynchronously; the runner must retry/backoff when resolving “proposal head SHA.”
* **Tree/file API error semantics:** missing paths may return `404` (not empty lists); treat this deterministically in contract tests.
* **Ref semantics:** prove in your environment that the chosen “tree by ref” endpoint supports `ref=<commit_sha>`; if not, define and test a fallback strategy.
* **Rate/size limits:** commit batching and citeable context bundles must be budget-aware (chunk actions/reads, cache, and backoff); runner should surface throttling/fallbacks in the run report.

---

## 6. The Scenario Slice Ladder (superseding 706 increments + 713 scenarios)

Below is the recommended ladder. Each slice is a **product milestone** and a **cross-primitive implementation target**.

### Tiering

* **Tier‑1**: must ship to claim “v6 platform exists.”
* **Tier‑2**: expands EA utility.
* **Tier‑3**: portfolio/planning richness.

---

# Slice 1 (Tier‑1): Canonical Architecture Repository Portal

**Capability tags**
`cap:identity`, `cap:citation`, `cap:repo.onboard`, `cap:repo.read_tree`, `cap:repo.read_file`

## User story

As an enterprise architect, I can browse the canonical Enterprise Repository (as stored in Git) and open artifacts at an immutable baseline so I can trust what I’m reading.

## Value proposition

* Replaces “EA docs drift” with a **canonical architecture repository** experience.
* Establishes the platform as the *place to read architecture truth*.
* Enables “shareable, immutable references” (by SHA) as a trust feature.

## Journey

1. Onboard an Enterprise Repository into the platform
2. Browse Git tree at `published` read context
3. Open an artifact in the **Element Editor** and see resolved commit SHA + citation

## Repo artifacts (minimum)

* `architecture/vision.md` (can be simple placeholder)
* `README.md`

## Primitive responsibilities

* **Domain**: onboard repository mapping (or platform service does mapping, but Domain remains the canonical identity gate)
* **Projection**: resolve `read_context → commit_sha`, list tree, read file, return citations
* **UI**: repository screen (Git-tree hierarchy), open artifacts via the Element Editor; show resolved SHA everywhere
* **Memory**: optional, but if present: “fetch cited paths” bundle
* **GitLab substrate**: project exists with at least one commit

## DoD

* Contract-pass: scenario runner proves browsing + file read + citations, and handles missing path behavior deterministically.
* UX-pass: user can browse and open an artifact via the Element Editor.

---

# Slice 2 (Tier‑1): Governed Publish of Architecture Vision

**Capability tags**
`cap:change.ensure`, `cap:change.commit_batch`, `cap:change.publish`, `cap:evidence.spine`, plus Slice 1 tags

## User story

As an enterprise architect, I can propose and publish an Architecture Vision so it becomes the canonical baseline, with evidence.

## Value proposition

* Architecture changes become **governed**, reviewable, auditable.
* This is the core “EA-as-code” differentiator: **publish is explicit** and leaves evidence.

## Journey

1. Browse baseline (Slice 1)
2. Propose change (creates change/MR)
3. Edit two files (vision + statement of work)
4. Publish → baseline advances to a new `main` SHA
5. UI shows evidence spine (MR + published SHA)

## Repo artifacts

* `architecture/vision.md`
* `architecture/statement-of-work.md`

## Primitive responsibilities

* **Domain**: EnsureChange, commit batching, publish with SHA guards, record evidence spine
* **Projection**: converge reads to new published SHA immediately after publish
* **UI**: change-based editing UX; publish is explicit governed action
* **GitOps/token contract**: least privilege consistent with owner-only writes; tokens must be expiring/rotating (no “forever bot” assumption)

**Vendor notes (GitLab)**
- Treat the audit anchor as **target branch head SHA after publish** (merge-method agnostic), and record merge/squash SHAs as optional supplemental evidence.
- If pipeline gating is used, prefer GitLab “auto-merge” semantics; do not depend on deprecated “merge when pipeline succeeds” API parameters.

## DoD

* Contract-pass: E2E publish loop + evidence spine recorded.
* UX-pass: “Propose → Publish” visible and understandable in EA language.

---

# Slice 3 (Tier‑1): Derived Backlinks & Impact with Explainability

**Capability tags**
`cap:derived.backlinks`, `cap:derived.impact`, `cap:memory.citeable_context`, plus Slice 2 tags

## User story

As an architect, I can navigate “what references this?” and “what is impacted?” without relationship CRUD, and every derived link explains itself via citations.

## Value proposition

* Turns static repo artifacts into a navigable EA knowledge system.
* Enables governance quality: reviewers can see blast radius and traceability.

## Journey

1. Publish two artifacts with inline references (e.g., standard referencing a requirement)
2. Open requirement → see derived backlinks panel
3. Ask memory/assistant: “What depends on REQ‑001?” → citeable context returned

## Repo artifacts

* `requirements/REQ-001.md` (must contain stable id)
* `architecture/standards/standard-a.md` (inline reference to REQ‑001)

## Primitive responsibilities

* **Projection**: parse inline reference grammar, build backlink index, return origin citations
* **UI**: derived panels (“Impact / Backlinks”) explicitly derived + show “why linked?” citations
* **Memory**: retrieval returns citeable bundles only (no uncited facts)
* **Domain**: enforce “stable id present” rule on publish (optional but recommended)

## DoD

* Contract-pass: backlink query returns edges with origin citations.
* UX-pass: backlink/impact panel visible and drillable.

---

# Slice 4 (Tier‑1): Proposal vs Published Review (MR head blast radius)

**Capability tags**
`cap:proposal.read_context`, plus Slice 3 tags

## User story

As governance, I can see the blast radius of a principle/standard change **before publishing**, comparing proposal (MR head) vs published baseline.

## Value proposition

* This is the EA equivalent of “review diff + impact,” but EA-first:

  * “If we change Principle P‑001, what standards/patterns are affected?”
* Makes governance faster and safer.

## Journey

1. Publish baseline containing P‑001 and a referencing standard
2. Propose change to P‑001 (MR head)
3. View backlinks/impact at:

   * published baseline SHA
   * proposal head SHA
4. Optionally attach “impact summary” evidence to the governance record

## Repo artifacts

* `principles/P-001.md`
* `architecture/standards/standard-a.md` referencing P‑001

## Primitive responsibilities

* **Domain**: expose proposal read context (MR head SHA) deterministically
* **Projection**: support derived queries at arbitrary SHAs (published + proposal)
* **UI**: explicit selector: “Published vs Proposal”; show resolved SHAs
* **Agent** (optional): generate citeable impact summary (no direct writes)

## DoD

* Contract-pass: backlink results differ/align correctly per SHA.
* UX-pass: reviewer can switch contexts and understand what they’re seeing.

---

# Slice 5 (Tier‑2): Baseline Compare (Gap analysis: as‑is vs to‑be)

**Capability tags**
`cap:baseline.compare`, plus Slice 2 tags

## User story

As an architect, I can compare two baselines and see what changed, with citations to both commits.

## Value proposition

* Enables TOGAF-ish gap assessment and change narratives:

  * “What changed between approved baseline A and proposed baseline B?”
* Immediate EA utility even with file-level diffs.

## Journey

1. Publish baseline A (record SHA A)
2. Publish baseline B (record SHA B)
3. Compare A vs B → show file-level changes + optional summaries, all citeable

## Primitive responsibilities

* **Projection**: compare helper that outputs (adds/modifies/deletes) with citations to both SHAs
* **UI**: baseline compare screen (EA language: “As‑Is vs To‑Be”)
* **Agent** (optional): citeable diff summary

## DoD

* Contract-pass: compare output reproducible and citeable.
* UX-pass: user can select two baselines and understand what changed.

---

# Slice 6 (Tier‑2): Governance Outcomes as Platform Truth (tied to Git evidence)

**Capability tags**
`cap:governance.outcome`, plus Slice 2 + 4 tags

## User story

As a governance board, I can record approvals/waivers/outcomes in the platform and tie them to the MR and published baseline SHA.

## Value proposition

* Separates “what is approved” (platform governance truth) from “what exists in Git” (content truth).
* Removes reliance on paid GitLab features for governance planning.

## Journey

1. Proposal exists (MR)
2. Board records outcome (approve/waive/request changes) in platform
3. Publish is allowed/blocked based on governance truth
4. Outcome is visible in EA language, linked to MR + published SHA

## Primitive responsibilities

* **Platform governance service**: canonical outcome store + policy checks
* **Domain**: enforce publish gating via a single, explicit contract (either: Domain requires a governance attestation input, or Process gates publish and only calls Domain when allowed)
* **Projection**: join governance truth with evidence spine in views
* **UI**: governance timeline/outcomes panel in EA language

**Vendor notes (GitLab)**
- Do not rely on paid-tier approval rules for enforcement. Any GitLab approvals are advisory only; enforcement lives in platform governance truth + protected branch + owner-only merge identity.

## DoD

* Contract-pass: governance outcome linked to MR + published SHA.
* UX-pass: board can record and later audit outcomes.

---

# Slice 7 (Tier‑3): ProcessDefinitions as governed files, deployed by SHA

**Capability tags**
`cap:process.definition_files`, `cap:process.deploy_by_sha`, plus Slice 2 tags

## User story

As a platform operator / governance designer, I can version governance/transformation processes as repo artifacts and deploy them deterministically, tied to commit SHA.

## Value proposition

* “Governance as code” becomes real: processes are reviewed, published, and auditable.
* Enables repeatable transformation workflows.

## Journey

1. Propose change adding/updating BPMN under `processes/...`
2. Publish baseline
3. Process runtime deploys definition tied to published SHA
4. UI shows “process active at SHA”

## DoD

* Contract-pass: deploy record references commit SHA and process key.
* UX-pass: user can see which process version is active.

---

# Slice 8 (Tier‑3, but strategic): Transformation Run E2E (process + agent + publish + evidence)

**Capability tags**
`cap:transformation.run_e2e`, plus Slice 2 + 3 + 6 + 7 tags

## User story

As a transformation lead, I can run a transformation that produces governed changes in the Enterprise Repository, with progress and evidence visible in the platform.

## Value proposition

* The platform doesn’t just store architecture; it **executes transformation work** safely.
* Produces a complete evidence spine: initiative → process → agent → MR → publish SHA → impact.

## Journey

1. Start transformation initiative (platform)
2. Process runs and assigns work to executor agent
3. Agent reads context via Projection/Memory (citations)
4. Agent proposes change via Domain (no direct git)
5. Publish occurs after governance checks
6. UI shows status + evidence + derived impact

## DoD

* Contract-pass: full run report links process instance + MR + published SHA + citations.
* UX-pass: minimal transformation screen showing progress and evidence.

---

## 7. Reference grammar and stable identity (must be specified early)

Because Slices 3–4 depend on it, this is part of the spec (not optional).

### 7.1 Stable identity in files

Every “element” file that participates in derived views must contain a stable ID (e.g., frontmatter `id:` or equivalent).

### 7.2 Inline reference grammar

Inline references must be parseable and citeable (including anchor where possible).
Examples (illustrative only; choose one canon):

* `ref:REQ-001`
* `ref:{id}`
* or Markdown links with a structured target

### 7.3 Rename/move semantics

* Citations are immutable: `{sha, path}` refers to what existed *there* at *that* SHA.
* After moves, new citations point to new paths at new SHAs; projections may optionally derive “moved-from” hints but must cite sources.

---

## 8. Scenario catalog (absorbed from 713)

This section fully absorbs `backlog/713-v6-togaf-functional-scenarios-e2e-tests.md` by preserving the TOGAF-ish scenarios and expressing each scenario as:

- a **product-visible journey**,
- a **slice coverage** list (what ladder slices it exercises), and
- the **cross-primitive contract-pass test** (plus optional UX-pass expectations).

### Scenario A — Architecture Vision becomes a canonical baseline (ADM Phase A)

**User story**
- As an enterprise architect, I can propose and publish an Architecture Vision (and Statement of Architecture Work) so it becomes the canonical baseline, citeable by commit SHA.

**Slice coverage**
- Slice 2 (Governed publish)
- Slice 1 (Repository portal)

**Capability tags**
`cap:identity`, `cap:citation`, `cap:repo.onboard`, `cap:repo.read_tree`, `cap:repo.read_file`, `cap:change.ensure`, `cap:change.commit_batch`, `cap:change.publish`, `cap:evidence.spine`

**Primitive contributions (complete target)**
- **GitLab substrate:** stores the canonical files; MR is the proposal unit; protected `main` is the canonical baseline; pipelines validate; publish advances the **target branch head SHA** (merge/squash SHAs are supplemental evidence depending on merge method).
- **Domain (owner-only writes):** creates/owns the “change” abstraction, opens/updates the MR, commits the file changes, publishes (merges) with SHA-safe semantics, and records the evidence spine `{mr_iid, target_branch, target_head_sha, citations}` (plus optional merge/squash SHAs).
- **Projection (derived views):** resolves `read_context` → `commit_sha`, serves Git-tree reads (`ListTree`/`GetContent`) strictly from Git, and returns citation-grade content responses.
- **Process (orchestration only):** optional here; can drive the “publish loop” (draft → review → publish) but must only call Domain commands (no direct git ops).
- **Agent (proposal/evidence only):** optional here; can draft the vision text and propose it, but must not hold write credentials; it reads via Projection and submits a proposal via Domain.
- **UI surface (EA-first UX):** provides “browse canonical baseline” + “propose change” + “publish” UX and must show commit SHA/citations and evidence spine in the user journey.

**Contract-pass E2E test**
1. Onboard mapping.
2. Propose change (MR) that adds/edits:
   - `architecture/vision.md`
   - `architecture/statement-of-work.md`
3. Publish (merge to `main`) with SHA-safe semantics.
4. Read back from `published` and assert:
   - `read_context.resolved.commit_sha` equals the new `main` SHA.
   - file reads return citations anchored to that SHA + path.
   - audit pointers exist (MR + resulting `main` SHA).

### Scenario B — Requirements capture with traceability hooks (Stakeholder concerns → requirements)

**User story**
- As an architect, I can add requirements in the repository and later navigate “what depends on this requirement” via derived backlinks, without relationship CRUD.

**Slice coverage**
- Slice 3 (Derived backlinks/impact)
- Slice 2 (Governed publish)

**Capability tags**
`cap:derived.backlinks`, `cap:derived.impact`, `cap:memory.citeable_context`, plus Scenario A tags

**Primitive contributions (complete target)**
- **GitLab substrate:** requirements and referencing artifacts are authored as files; MRs propose changes; the commit graph provides “what was true” at any SHA.
- **Domain (owner-only writes):** ensures requirement/standard files are created/updated via MR-backed publish; enforces canonical authoring rules (e.g., stable IDs present) at write time where appropriate.
- **Projection (derived views):** parses inline references to build a rebuildable backlinks/impact index; every derived edge must carry an origin citation `{commit_sha, path[, anchor]}` (analogous to GitLab’s “derived knowledge graph” posture: index from repos, never canonicalize).
- **Process (orchestration only):** optional here; can orchestrate “requirements intake” steps (capture → validate → publish), but cannot write artifacts directly.
- **Agent (proposal/evidence only):** optional here; can help draft requirements, suggest ref updates, or produce an “impact summary” for reviewers; must cite sources and must not create relationships in a DB.
- **UI surface:** shows requirement detail with “Derived impact/backlinks” panels that are explicitly derived (no relationship CRUD), and provides “why is this linked?” via citations.

**Contract-pass E2E test**
1. Onboard mapping.
2. Publish two artifacts:
   - `requirements/REQ-001.md` (contains a stable id and at least one inline ref)
   - `architecture/standards/standard-a.md` (references `REQ-001` inline)
3. Run projection indexing for `published` SHA.
4. Query backlinks/impact for `requirements/REQ-001.md` and assert:
   - at least one backlink exists (from `standard-a.md`),
   - every derived edge includes origin citations to `{commit_sha, path[, anchor]}`,
   - broken refs (if introduced) are surfaced deterministically.
5. Call `memory.get_context` for `REQ-001` and assert:
   - context bundle items are all citeable (no non-cited “memory facts”),
   - `read_context.resolved.commit_sha` matches `published`.

### Scenario C — Principles change with blast radius visibility (Principles → standards/patterns)

**User story**
- As governance, I can change an architecture principle and see what standards and artifacts reference it before publishing.

**Slice coverage**
- Slice 4 (Proposal vs published review)
- Slice 3 (Derived backlinks/impact)

**Capability tags**
`cap:proposal.read_context`, plus Scenario B tags

**Primitive contributions (complete target)**
- **GitLab substrate:** proposal lives in an MR (reviewable discussion, diffs, pipeline results); baseline lives in `main` at a commit SHA.
- **Domain (owner-only writes):** creates/updates principle artifacts via MR; ensures the proposal view is addressable and publish is SHA-safe.
- **Projection (derived views):** supports querying backlinks for both `published` (baseline) and “proposal head” contexts by SHA; produces deterministic results per SHA.
- **Process (orchestration only):** can implement a governance gate (“principle change review”) that uses projection-derived blast radius as evidence.
- **Agent (proposal/evidence only):** can generate an impact summary and propose mitigations (e.g., “update these standards”), but must only suggest/propose changes via Domain.
- **UI surface:** makes “proposal vs published” explicit and lets reviewers inspect derived blast radius at both SHAs (analogous to GitLab MR view vs default branch view).

**Contract-pass E2E test**
1. Onboard mapping.
2. Publish initial artifacts:
   - `principles/P-001.md`
   - `architecture/standards/standard-a.md` referencing `P-001`
3. Propose change updating `P-001` (proposal head view).
4. Query backlinks for `P-001` at:
   - `published` (baseline)
   - proposal head (MR head SHA)
5. Assert:
   - both views are citeable and have distinct `resolved.commit_sha` if content changed,
   - backlink results are consistent with the repo content at each SHA.

### Scenario D — Architecture Definition Document evolves under MR governance (ADD)

**User story**
- As an architect, I can evolve the ADD via proposals and publish with evidence so auditors can pin “what the ADD said” at a baseline SHA.

**Slice coverage**
- Slice 2 (Governed publish)
- Slice 1 (Repository portal)

**Capability tags**
Scenario A tags, plus (optionally) `cap:baseline.compare` when diff UX exists

**Primitive contributions (complete target)**
- **GitLab substrate:** file moves/renames are real Git operations; the Git tree is the hierarchy; MRs record review and diffs; the audit anchor is the **target branch head SHA** after publish (merge/squash SHAs supplemental).
- **Domain (owner-only writes):** applies rename/move operations via commit actions, enforces “no empty folders” (Git-native), publishes via MR merge, and records evidence.
- **Projection (derived views):** must reflect the Git tree after moves (no synthetic folder model), and must return citations pointing to the new paths at the new SHA; can optionally provide “moved-from” hints as derived metadata (with citations).
- **Process (orchestration only):** optional; can orchestrate “document restructuring” workflows (split, rename, publish) and require review gates.
- **Agent (proposal/evidence only):** optional; can propose a restructure plan and produce a “mapping” from old → new sections as evidence, but all canonical edits still go via Domain.
- **UI surface:** shows Git-tree navigation; supports move/rename in the change editor as Git operations; surfaces evidence (MR + SHAs) for auditors.

**Contract-pass E2E test**
1. Onboard mapping.
2. Publish initial `add/architecture-definition.md`.
3. Propose change with edits and at least one rename/move (e.g., split into `add/sections/*.md`).
4. Publish.
5. Assert:
   - `published` resolves to new `main` SHA,
   - moved paths are reflected in Git tree,
   - citations point to the new paths at the new SHA (no “synthetic folders”).

### Scenario E — ABB/SBB coverage: “requirements satisfied by building blocks”

**User story**
- As a domain architect, I can declare building blocks and reference requirements so the platform can derive coverage views without a canonical relationship store.

**Slice coverage**
- Slice 3 (Derived backlinks/impact)

**Capability tags**
Scenario B tags

**Primitive contributions (complete target)**
- **GitLab substrate:** building blocks and requirements are authored as files; publish is MR-backed; history provides audit.
- **Domain (owner-only writes):** enforces canonical element authoring patterns (stable IDs in files) and publishes updates safely.
- **Projection (derived views):** computes coverage/backlinks (REQ ← ABB ← SBB) as a rebuildable index; traversal must be bounded and every edge must cite its origin.
- **Process (orchestration only):** optional; can run a “coverage check” workflow (e.g., ensure every requirement is referenced by at least one ABB) and attach results as evidence.
- **Agent (proposal/evidence only):** optional; can suggest missing links or missing building blocks, but must propose changes via Domain; can generate a citeable “coverage report” for review.
- **UI surface:** presents “coverage” as a derived view (catalog/matrix/graph-like UX) with drill-down citations, and never offers relationship CRUD as canonical truth.

**Contract-pass E2E test**
1. Onboard mapping.
2. Publish:
   - `requirements/REQ-001.md`
   - `building-blocks/ABB-001.md` referencing `REQ-001` inline
   - `building-blocks/SBB-001.md` referencing `ABB-001` inline
3. Query derived view:
   - backlinks for `REQ-001` returns `ABB-001` (and transitively `SBB-001` if traversal is supported)
4. Assert:
   - traversal is bounded and citeable,
   - each edge includes an origin citation.

### Scenario F — Gap analysis via baseline compare (as-is vs to-be)

**User story**
- As an architect, I can compare two baselines and see what changed, with all changes citeable to the underlying commits.

**Slice coverage**
- Slice 5 (Baseline compare)

**Capability tags**
`cap:baseline.compare`, plus Scenario A tags

**Primitive contributions (complete target)**
- **GitLab substrate:** baselines are Git refs (commit SHAs and/or tags); compare is fundamentally Git diff semantics.
- **Domain (owner-only writes):** optional here; may create/publish the “to-be” baseline; must record baseline identities (if tags are created, do it via the owning Domain policy).
- **Projection (derived views):** provides baseline resolution and diff results (file-level and optionally semantic summaries) as derived views; all comparisons must cite both SHAs.
- **Process (orchestration only):** optional; can run an “ADM gap assessment” process that consumes baseline diffs and produces an evidence artifact (still stored in Git via Domain).
- **Agent (proposal/evidence only):** optional; can summarize diffs and propose follow-up changes; summaries must include citations to changed artifacts/SHAs.
- **UI surface:** provides “compare baselines” UX and makes ref resolution explicit (tag → SHA) and citeable (like GitLab “compare branches/tags” UX, but EA-first).

**Contract-pass E2E test**
1. Onboard mapping.
2. Publish baseline A (capture `sha_a`).
3. Publish baseline B with edits and new artifacts (capture `sha_b`).
4. Run a baseline diff query and assert:
   - the diff is Git-native (file additions/modifications/deletions),
   - any rendered diff output includes citations to `{sha_a, path}` and `{sha_b, path}`.

### Scenario G — Opportunities & Solutions: Work Packages deliver building blocks

**User story**
- As a planner, I can create work packages that reference the building blocks they deliver, and reviewers can see the impact/traceability before publish.

**Slice coverage**
- Slice 3 (Derived backlinks/impact)
- Slice 4 (Proposal vs published review)

**Capability tags**
Scenario C tags

**Primitive contributions (complete target)**
- **GitLab substrate:** work packages are Git-authored artifacts; MRs are the governance proposal unit.
- **Domain (owner-only writes):** publishes WP artifacts and changes; ensures proposal head is addressable by SHA for review.
- **Projection (derived views):** derives “delivers” relationships from inline refs; supports proposal vs published queries; returns citations for every derived relationship.
- **Process (orchestration only):** can implement a planning workflow (create WP → review → publish) and can require “impact preview” evidence from Projection before publish.
- **Agent (proposal/evidence only):** can propose updating the WP delivered set and can generate an impact/evidence summary; cannot write directly.
- **UI surface:** supports “proposal review” with impact preview panels; publish advances baseline and UI reads converge to `published` SHA.

**Contract-pass E2E test**
1. Onboard mapping.
2. Publish:
   - `building-blocks/ABB-001.md`
   - `work-packages/WP-001.md` referencing `ABB-001` inline
3. Propose update to `WP-001` to add/remove delivered items.
4. Query backlinks/impact for `ABB-001` at proposal head and assert:
   - derived references update with the proposal head SHA,
   - publish advances the canonical baseline and the derived view converges.

### Scenario H — Transition Architectures / Plateaus as pinned baselines

**User story**
- As a program architect, I can pin a plateau and later compare it to another plateau to understand changes, all anchored to SHAs/tags.

**Slice coverage**
- Slice 5 (Baseline compare) with baseline refs/tags

**Capability tags**
Scenario F tags

**Primitive contributions (complete target)**
- **GitLab substrate:** plateau baselines are Git refs (tags or recorded SHAs) and must be immutable; compare is Git-native.
- **Domain (owner-only writes):** is the policy gate for creating plateau refs (if tags are used) and for publishing artifacts that define a plateau.
- **Projection (derived views):** resolves plateau refs → SHAs, provides compare outputs and any derived “what changed” summaries with citations.
- **Process (orchestration only):** can orchestrate plateau governance (proposal → board review → tag/pin) and record evidence of “why this plateau exists”.
- **Agent (proposal/evidence only):** can draft plateau notes, propose tagging, and generate a citeable change summary between plateaus.
- **UI surface:** shows plateau selection/compare with explicit ref resolution, and must never treat “plateau metadata” as canonical unless stored in Git or governance DB with evidence.

**Contract-pass E2E test**
1. Onboard mapping.
2. Publish plateau “TA-2026Q1” (either a tag or a recorded commit SHA).
3. Publish plateau “TA-2026Q2”.
4. Compare the two baselines and assert:
   - both baselines resolve to immutable SHAs,
   - the comparison output is citeable and reproducible.

### Scenario I — Governance gate outcome is platform truth, tied to Git evidence

**User story**
- As a governance board, I can record approvals/outcomes in the platform and tie them to the MR + resulting baseline SHA, without relying on paid GitLab features.

**Slice coverage**
- Slice 6 (Governance outcomes)

**Capability tags**
`cap:governance.outcome`, plus Scenario C tags

**Primitive contributions (complete target)**
- **GitLab substrate:** provides CE-available primitives (protected `main`, MR review/discussion, pipelines); do not depend on paid-tier planning features.
- **Domain (owner-only writes):** publishes canonical artifacts via MR; emits/records audit pointers tying governance outcomes to MR + resulting `main` SHA.
- **Projection (derived views):** joins governance truth (platform DB) with content truth (Git) into a projection-friendly view (evidence spine), without making projections canonical.
- **Process (orchestration only):** is the primary home for gates; it should enforce “no publish without required approvals” by calling Domain and checking governance DB state (not by bypassing it).
- **Agent (proposal/evidence only):** can prepare evidence packets (impact summaries, check results) and attach them to the governance record; must cite all sources.
- **UI surface:** presents governance outcomes and the evidence spine in EA language (“approved”, “waived”, “compliant”), and links back to MR + SHA.

**Contract-pass E2E test**
1. Onboard mapping.
2. Propose change (MR).
3. Record platform approvals (governance truth) referencing the change.
4. Publish.
5. Assert:
   - governance outcome is queryable and references MR + resulting `main` SHA,
   - GitLab enforcement used is CE-compatible (protected `main`, MR-only merges, pipeline gating as configured).

### Scenario J — “What was true then?” (audit replay by SHA)

**User story**
- As an auditor, I can replay exactly what the platform showed at a point in time.

**Slice coverage**
- Slice 1–5 progressively, plus projection rebuildability requirements

**Capability tags**
Scenario F tags, plus rebuildability requirements (projection)

**Primitive contributions (complete target)**
- **GitLab substrate:** the commit graph is the immutable timeline; any claim must be anchored to SHAs/tags.
- **Domain (owner-only writes):** records the evidence spine for publishes so “what happened” can be replayed, and ensures publish semantics are deterministic.
- **Projection (derived views):** guarantees rebuildability and determinism: rebuilding the same projection for the same commit SHA yields the same answers; derived results always cite origins.
- **Process (orchestration only):** optional; can provide “audit replay” workflows (select SHA → rebuild projection → generate report) but must not invent truth.
- **Agent (proposal/evidence only):** optional; can narrate a replay report (“what changed and why”) but must include citations and must not introduce non-cited facts.
- **UI surface:** provides a “time travel” read context selector (baseline/tag/SHA) and makes the resolved SHA explicit everywhere.

**Contract-pass E2E test**
1. Onboard mapping.
2. Publish two commits; record `sha_1` and `sha_2`.
3. Query `GetContent` and backlinks at `sha_1` and assert:
   - all outputs are citeable to `sha_1`,
   - rebuilding projections for `sha_1` yields the same answers (golden snapshot).

---

## 9. How this supersedes 713

* All TOGAF-ish scenarios and their cross-primitive E2E tests are now defined in this document (Section 8).
* `backlog/713-v6-togaf-functional-scenarios-e2e-tests.md` becomes a short pointer to this document.

---

## 10. Operational guidance: how teams work with this spec

1. For each slice, each primitive team adds a small “slice implementation checklist” to its own backlog.
2. A slice is “shippable” only when:

   * Contract-pass is green in CI (mock mode),
   * Contract-pass is green in cluster mode,
   * UX-pass is complete for Tier‑1 slices.
3. Every slice demo must include:

   * the user journey,
   * the evidence spine output,
   * one “why should I trust this?” explanation using citations.

---

## 11. Non-goals (explicit)

* This spec does not define performance/security/operability test suites (tracked separately).
* This spec does not mandate a specific file schema; it mandates **stable IDs + parseable references + citations**.
* This spec does not require agents/process in Tier‑1 slices except where explicitly stated.

---

### Summary

**Scenario Slices** give you:

* the realism and EA language of 713,
* the coordinated incremental delivery intent of 706,
* but in a single ladder that is **product-first**, **incrementally implementable**, and **provable end-to-end** without violating v6 invariants.
