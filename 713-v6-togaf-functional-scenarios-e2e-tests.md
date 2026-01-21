---
title: "713 — v6 TOGAF-ish functional scenarios (cross-primitive E2E tests)"
status: draft
owners:
  - platform
  - transformation
  - ui
created: 2026-01-20
updated: 2026-01-21
related:
  - 496-eda-principles-v6.md
  - 520-primitives-stack-v6.md
  - 537-primitive-testing-discipline.md
  - 656-agentic-memory-v6.md
  - 694-elements-gitlab-v6.md
  - 701-repository-ui-enterprise-repository-v6.md
  - 703-transformation-enterprise-repository-proto-v1.md
  - 704-v6-enterprise-repository-memory-e2e-implementation-plan.md
  - 710-gitlab-api-token-contract.md
---

# 713 — v6 TOGAF-ish functional scenarios (cross-primitive E2E tests)

This backlog item defines **functional (user-visible)** scenarios expressed in TOGAF-ish language and, for each scenario, the **cross-primitive E2E test** that validates it end-to-end under v6.

The intent is to keep the mental model aligned with the GitLab substrate:

- GitLab CE provides repos + MRs + pipelines as collaboration primitives.
- The platform owns governance truth and EA-first UX.
- Projections own all derived views (graphs/backlinks/search/memory) and must be citation-grade.

## Test harness contract (applies to every scenario)

**Two test front doors only** (per `backlog/537-primitive-testing-discipline.md`):

- `ameide test` (Phase 0/1/2): mocked runs only (no cluster; no real GitLab).
- `ameide test cluster` (Phase 3/4): real dev/local cluster integration only.

**Harness invariants**

- Starts from empty state: creates a new GitLab project for each run (under the dev/local E2E group), onboards the mapping, then runs the scenario.
- Produces an audit-grade run report including:
  - identity `{tenant_id, organization_id, repository_id}`
  - the governing MR identifier (and pipeline pointer if applicable)
  - the resulting `published` resolution (`main` commit SHA)
  - all citations returned/used (`{repository_id, commit_sha, path[, anchor]}`)
- Cleans up (best-effort delete project); failures must be surfaced (no silent leaks).

**Cross-primitive validation**

Each scenario test must exercise (at minimum):

- **Domain** for governed writes (MR-backed propose/publish) and governance evidence.
- **Projection** for Git-tree reads and any derived view (search/backlinks/memory).
- **UI** is not required for the test to pass initially, but the test must validate the same contract the UI depends on (read_context, citations, evidence spine).
- **Process/Agent** are scenario-specific and must not bypass owner-only writes.

## Scenarios (TOGAF-ish, v6-aligned)

### Scenario A — Architecture Vision becomes a canonical baseline (ADM Phase A)

**User story**
- As an enterprise architect, I can propose and publish an Architecture Vision (and Statement of Architecture Work) so it becomes the canonical baseline, citeable by commit SHA.

**Primitive contributions (complete target)**
- **GitLab substrate:** stores the canonical files; MR is the proposal unit; protected `main` is the canonical baseline; pipelines validate; merge commit SHA is the immutable evidence anchor.
- **Domain (owner-only writes):** creates/owns the “change” abstraction, opens/updates the MR, commits the file changes, publishes (merges) with SHA-safe semantics, and records the evidence spine `{mr_iid, merge_commit_sha(main), citations}`.
- **Projection (derived views):** resolves `read_context` → `commit_sha`, serves Git-tree reads (`ListTree`/`GetContent`) strictly from Git, and returns citation-grade content responses.
- **Process (orchestration only):** optional here; can drive the “publish loop” (draft → review → publish) but must only call Domain commands (no direct git ops).
- **Agent (proposal/evidence only):** optional here; can draft the vision text and propose it, but must not hold write credentials; it reads via Projection and submits a proposal via Domain.
- **UI surface (EA-first UX):** provides “browse canonical baseline” + “propose change” + “publish” UX and must show commit SHA/citations and evidence spine in the user journey.

**Increment target**
- Inc 3 (publish + evidence) in `backlog/704-v6-enterprise-repository-memory-e2e-implementation-plan.md`.

**E2E test**
1. Create project + onboard mapping.
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

**Primitive contributions (complete target)**
- **GitLab substrate:** requirements and referencing artifacts are authored as files; MRs propose changes; the commit graph provides “what was true” at any SHA.
- **Domain (owner-only writes):** ensures requirement/standard files are created/updated via MR-backed publish; enforces canonical authoring rules (e.g., stable IDs present) at write time where appropriate.
- **Projection (derived views):** parses inline references to build a rebuildable backlinks/impact index; every derived edge must carry an origin citation `{commit_sha, path[, anchor]}` (analogous to GitLab’s “derived knowledge graph” posture: index from repos, never canonicalize).
- **Process (orchestration only):** optional here; can orchestrate “requirements intake” steps (capture → validate → publish), but cannot write artifacts directly.
- **Agent (proposal/evidence only):** optional here; can help draft requirements, suggest ref updates, or produce an “impact summary” for reviewers; must cite sources and must not create relationships in a DB.
- **UI surface:** shows requirement detail with “Derived impact/backlinks” panels that are explicitly derived (no relationship CRUD), and provides “why is this linked?” via citations.

**Increment target**
- Inc 4 (derived backlinks + citeable memory).

**E2E test**
1. Create project + onboard mapping.
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

**Primitive contributions (complete target)**
- **GitLab substrate:** proposal lives in an MR (reviewable discussion, diffs, pipeline results); baseline lives in `main` at a commit SHA.
- **Domain (owner-only writes):** creates/updates principle artifacts via MR; ensures the proposal view is addressable and publish is SHA-safe.
- **Projection (derived views):** supports querying backlinks for both `published` (baseline) and “proposal head” contexts by SHA; produces deterministic results per SHA.
- **Process (orchestration only):** can implement a governance gate (“principle change review”) that uses projection-derived blast radius as evidence.
- **Agent (proposal/evidence only):** can generate an impact summary and propose mitigations (e.g., “update these standards”), but must only suggest/propose changes via Domain.
- **UI surface:** makes “proposal vs published” explicit and lets reviewers inspect derived blast radius at both SHAs (analogous to GitLab MR view vs default branch view).

**Increment target**
- Inc 4 (backlinks/impact + proposal vs published views).

**E2E test**
1. Create project + onboard mapping.
2. Publish initial artifacts:
   - `principles/P-001.md`
   - `architecture/standards/standard-a.md` referencing `P-001`
3. Propose change updating `P-001` (MR head view).
4. Query backlinks for `P-001` at:
   - `published` (baseline)
   - `head` / proposal (MR)
5. Assert:
   - both views are citeable and have distinct `resolved.commit_sha` if content changed,
   - backlink results are consistent with the repo content at each SHA.

### Scenario D — Architecture Definition Document evolves under MR governance (ADD)

**User story**
- As an architect, I can evolve the ADD via proposals and publish with evidence so auditors can pin “what the ADD said” at a baseline SHA.

**Primitive contributions (complete target)**
- **GitLab substrate:** file moves/renames are real Git operations; the Git tree is the hierarchy; MRs record review and diffs; the merge commit SHA is the audit anchor.
- **Domain (owner-only writes):** applies rename/move operations via commit actions, enforces “no empty folders” (Git-native), publishes via MR merge, and records evidence.
- **Projection (derived views):** must reflect the Git tree after moves (no synthetic folder model), and must return citations pointing to the new paths at the new SHA; can optionally provide “moved-from” hints as derived metadata (with citations).
- **Process (orchestration only):** optional; can orchestrate “document restructuring” workflows (split, rename, publish) and require review gates.
- **Agent (proposal/evidence only):** optional; can propose a restructure plan and produce a “mapping” from old → new sections as evidence, but all canonical edits still go via Domain.
- **UI surface:** shows Git-tree navigation; supports move/rename in the change editor as Git operations; surfaces evidence (MR + SHAs) for auditors.

**Increment target**
- Inc 3 (publish + evidence) and Inc 6 (baseline diffs optional).

**E2E test**
1. Create project + onboard mapping.
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

**Primitive contributions (complete target)**
- **GitLab substrate:** building blocks and requirements are authored as files; publish is MR-backed; history provides audit.
- **Domain (owner-only writes):** enforces canonical element authoring patterns (stable IDs in files) and publishes updates safely.
- **Projection (derived views):** computes coverage/backlinks (REQ ← ABB ← SBB) as a rebuildable index; traversal must be bounded and every edge must cite its origin.
- **Process (orchestration only):** optional; can run a “coverage check” workflow (e.g., ensure every requirement is referenced by at least one ABB) and attach results as evidence.
- **Agent (proposal/evidence only):** optional; can suggest missing links or missing building blocks, but must propose changes via Domain; can generate a citeable “coverage report” for review.
- **UI surface:** presents “coverage” as a derived view (catalog/matrix/graph-like UX) with drill-down citations, and never offers relationship CRUD as canonical truth.

**Increment target**
- Inc 4 (derived views).

**E2E test**
1. Create project + onboard mapping.
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

**Primitive contributions (complete target)**
- **GitLab substrate:** baselines are Git refs (commit SHAs and/or tags); compare is fundamentally Git diff semantics.
- **Domain (owner-only writes):** optional here; may create/publish the “to-be” baseline; must record baseline identities (if tags are created, do it via the owning Domain policy).
- **Projection (derived views):** provides baseline resolution and diff results (file-level and optionally semantic summaries) as derived views; all comparisons must cite both SHAs.
- **Process (orchestration only):** optional; can run an “ADM gap assessment” process that consumes baseline diffs and produces an evidence artifact (still stored in Git via Domain).
- **Agent (proposal/evidence only):** optional; can summarize diffs and propose follow-up changes; summaries must include citations to changed artifacts/SHAs.
- **UI surface:** provides “compare baselines” UX and makes ref resolution explicit (tag → SHA) and citeable (like GitLab “compare branches/tags” UX, but EA-first).

**Increment target**
- Inc 6 (baseline diffs).

**E2E test**
1. Create project + onboard mapping.
2. Publish baseline A (capture `sha_a`).
3. Publish baseline B with edits and new artifacts (capture `sha_b`).
4. Run a baseline diff query (or projection helper) and assert:
   - the diff is Git-native (file additions/modifications/deletions),
   - any rendered diff output includes citations to `{sha_a, path}` and `{sha_b, path}`.

### Scenario G — Opportunities & Solutions: Work Packages deliver building blocks

**User story**
- As a planner, I can create work packages that reference the building blocks they deliver, and reviewers can see the impact/traceability before publish.

**Primitive contributions (complete target)**
- **GitLab substrate:** work packages are Git-authored artifacts; MRs are the governance proposal unit.
- **Domain (owner-only writes):** publishes WP artifacts and changes; ensures proposal head is addressable by SHA for review.
- **Projection (derived views):** derives “delivers” relationships from inline refs; supports proposal vs published queries; returns citations for every derived relationship.
- **Process (orchestration only):** can implement a planning workflow (create WP → review → publish) and can require “impact preview” evidence from Projection before publish.
- **Agent (proposal/evidence only):** can propose updating the WP delivered set and can generate an impact/evidence summary; cannot write directly.
- **UI surface:** supports “proposal review” with impact preview panels; publish advances baseline and UI reads converge to `published` SHA.

**Increment target**
- Inc 4 (derived views) + Inc 3 (publish evidence).

**E2E test**
1. Create project + onboard mapping.
2. Publish:
   - `building-blocks/ABB-001.md`
   - `work-packages/WP-001.md` referencing `ABB-001` inline
3. Propose update to `WP-001` to add/remove delivered items.
4. Query backlinks/impact for `ABB-001` at proposal head and assert:
   - derived references update with the MR head SHA,
   - publish advances the canonical baseline and the derived view converges.

### Scenario H — Transition Architectures / Plateaus as pinned baselines

**User story**
- As a program architect, I can pin a plateau and later compare it to another plateau to understand changes, all anchored to SHAs/tags.

**Primitive contributions (complete target)**
- **GitLab substrate:** plateau baselines are Git refs (tags or recorded SHAs) and must be immutable; compare is Git-native.
- **Domain (owner-only writes):** is the policy gate for creating plateau refs (if tags are used) and for publishing artifacts that define a plateau.
- **Projection (derived views):** resolves plateau refs → SHAs, provides compare outputs and any derived “what changed” summaries with citations.
- **Process (orchestration only):** can orchestrate plateau governance (proposal → board review → tag/pin) and record evidence of “why this plateau exists”.
- **Agent (proposal/evidence only):** can draft plateau notes, propose tagging, and generate a citeable change summary between plateaus.
- **UI surface:** shows plateau selection/compare with explicit ref resolution, and must never treat “plateau metadata” as canonical unless stored in Git or governance DB with evidence.

**Increment target**
- Inc 6 (baseline management/diffs).

**E2E test**
1. Create project + onboard mapping.
2. Publish plateau “TA-2026Q1” (either a tag or a recorded commit SHA).
3. Publish plateau “TA-2026Q2”.
4. Compare the two baselines and assert:
   - both baselines resolve to immutable SHAs,
   - the comparison output is citeable and reproducible.

### Scenario I — Governance gate outcome is platform truth, tied to Git evidence

**User story**
- As a governance board, I can record approvals/outcomes in the platform and tie them to the MR + resulting baseline SHA, without relying on paid GitLab features.

**Primitive contributions (complete target)**
- **GitLab substrate:** provides CE-available primitives (protected `main`, MR review/discussion, pipelines); do not depend on paid-tier planning features.
- **Domain (owner-only writes):** publishes canonical artifacts via MR; emits/records audit pointers tying governance outcomes to MR + resulting `main` SHA.
- **Projection (derived views):** joins governance truth (platform DB) with content truth (Git) into a projection-friendly view (evidence spine), without making projections canonical.
- **Process (orchestration only):** is the primary home for gates; it should enforce “no publish without required approvals” by calling Domain and checking governance DB state (not by bypassing it).
- **Agent (proposal/evidence only):** can prepare evidence packets (impact summaries, check results) and attach them to the governance record; must cite all sources.
- **UI surface:** presents governance outcomes and the evidence spine in EA language (“approved”, “waived”, “compliant”), and links back to MR + SHA.

**Increment target**
- Inc 3 (publish + evidence).

**E2E test**
1. Create project + onboard mapping.
2. Propose change (MR).
3. Record platform approvals (governance truth) referencing the change.
4. Publish.
5. Assert:
   - governance outcome is queryable and references MR + resulting `main` SHA,
   - GitLab enforcement used is CE-compatible (protected `main`, MR-only merges, pipeline gating as configured).

### Scenario J — “What was true then?” (audit replay by SHA)

**User story**
- As an auditor, I can replay exactly what the platform showed at a point in time.

**Primitive contributions (complete target)**
- **GitLab substrate:** the commit graph is the immutable timeline; any claim must be anchored to SHAs/tags.
- **Domain (owner-only writes):** records the evidence spine for publishes so “what happened” can be replayed, and ensures publish semantics are deterministic.
- **Projection (derived views):** guarantees rebuildability and determinism: rebuilding the same projection for the same commit SHA yields the same answers; derived results always cite origins.
- **Process (orchestration only):** optional; can provide “audit replay” workflows (select SHA → rebuild projection → generate report) but must not invent truth.
- **Agent (proposal/evidence only):** optional; can narrate a replay report (“what changed and why”) but must include citations and must not introduce non-cited facts.
- **UI surface:** provides a “time travel” read context selector (baseline/tag/SHA) and makes the resolved SHA explicit everywhere.

**Increment target**
- Inc 1 (read-only) + Inc 4 (derived views) + Inc 6 (rebuildability).

**E2E test**
1. Create project + onboard mapping.
2. Publish two commits; record `sha_1` and `sha_2`.
3. Query `GetContent` and backlinks at `sha_1` and assert:
   - all outputs are citeable to `sha_1`,
   - rebuilding projections for `sha_1` yields the same answers (golden snapshot).

## Notes / non-goals

- These are functional scenarios; security/performance/operability checks are tracked separately.
- Scenarios assume v6 posture: **no relationship CRUD**, **owner-only writes**, **Git-tree hierarchy**, **GitLab CE-only** primitives.
