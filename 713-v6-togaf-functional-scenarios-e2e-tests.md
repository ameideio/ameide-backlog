---
title: "713 — v6 TOGAF-ish functional scenarios (cross-primitive E2E tests)"
status: draft
owners:
  - platform
  - transformation
  - ui
created: 2026-01-20
updated: 2026-01-20
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
