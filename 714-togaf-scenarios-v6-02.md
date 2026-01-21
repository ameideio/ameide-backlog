---
title: "714 — v6 Scenario Slices v6-02 (Scenario B: Requirements + derived backlinks/impact + citeable context)"
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
  - 704-v6-enterprise-repository-memory-e2e-implementation-plan.md
  - 656-agentic-memory-v6.md
  - 710-gitlab-api-token-contract.md
---

# 714 v6-02 — Scenario B implementation consolidation

This document is an implementer-oriented consolidation for **Scenario B** from `backlog/714-togaf-scenarios-v6.md`.

Goal: specify, in concrete terms, what must be implemented across primitives to make Scenario B **real**, **Git-first**, and **explainable** (derived views with citations).

## Scenario B (recap)

**User story**
- As an architect, I can add requirements in the repository and later navigate “what depends on this requirement” via derived backlinks, without relationship CRUD.

**What this scenario proves (v6 posture)**
- Requirements are **canonical files** (Git content truth), not DB rows.
- Relationships are **inline-only references** inside authored files (either in content/body, or in file metadata/frontmatter).
- Backlinks/impact are **projection-only** (rebuildable), and every derived edge is citation-grade.
- Memory/context is **projection-owned** and returns only citeable bundles.

## EDA alignment (496)

Scenario B reinforces the “owner facts → derived projections” posture from `backlog/496-eda-principles-v6.md`:

* Canonical writes still flow through the Domain’s command surface (owner-only writes).
* Projection is a **derived read model**:
  * it may update incrementally by consuming owner facts (e.g., “change published” with audit pointers),
  * and it must always be rebuildable from canonical Git at a commit SHA + recorded audit pointers.
* Memory is not an owner and does not emit facts; it is a query surface over derived data (citeable context bundles).
* Broken/ambiguous references are a normal outcome of derived indexing:
  * surface them deterministically with origin citations,
  * and resolve them only via a governed change (Domain command), never via “projection writes”.

## Capability test placement (590/591)

Scenario B’s contract-pass test is a cross-primitive flow (Domain publish → Projection derive → Memory bundle) and should live as a **capability-owned test** under the repo’s `capabilities/` boundary (per `backlog/590-capabilities.md`), alongside Scenario A’s tests.

## Minimal repo artifacts

Scenario B uses these canonical file paths:
- `requirements/REQ-001.md` (must contain a stable ID)
- `architecture/standards/standard-a.md` (must reference `REQ-001` inline)

## Contracts and data shapes

### Identity (required everywhere)

- `{tenant_id, organization_id, repository_id}`

### Repository mapping (required)

Canonical mapping from `{tenant_id, organization_id, repository_id}` to the vendor remote:
- `provider = gitlab`
- `remote_id = <gitlab project id or path>` (opaque to UX)
- `published_ref = main`

### Read context (minimum required in Scenario B)

Projection must support:
- `published` → resolves to `resolved.commit_sha` for `target_branch` (typically `main`)
- `version_ref.commit_sha` → historical reads for audit replay

Scenario B does not require proposal-head reads (that is Scenario C), but the data model should not prevent it.

### Citations (minimum)

All read and derived outputs must be anchored to citations:
- File/tree reads: `{repository_id, commit_sha, path[, anchor]}`
- Derived edges must include **origin citations** that point to where the reference text lived:
  - `origin: {repository_id, commit_sha, path, anchor}`

### Stable identity in files (minimum)

To avoid path-based identity drift, each “element file” participating in derived views must carry a stable ID in its content.

**Scenario B MVP rule**
- `requirements/REQ-001.md` must include a stable ID value `REQ-001`.
- In v6, IDs are content, not path: moving a file does not change the ID.

**Identity scope and collisions (product rules)**
- IDs are **repository-unique** at a given `{repository_id, commit_sha}`.
- If multiple files claim the same `id` at the same commit SHA, Projection must treat the ID as **ambiguous**:
  - `GetElementById` must fail with an explicit “duplicate id” error and include the candidate citations `{repo, sha, path[, anchor]}`.
  - UI must surface this as a user-visible error (it is not safe to pick an arbitrary winner).
- If a file has no stable `id`, it is still readable by **path**, but it is not eligible for `element_id → path` navigation or derived views that require IDs (unless explicitly allowed by a slice).

### Inline reference grammar (Scenario B MVP)

For Scenario B, pick one simple reference form so Projection can be deterministic:

- The requirement declares an ID in frontmatter:
  - `id: REQ-001`
- A referencing file includes an inline token:
  - `ref: REQ-001`

Notes:
- This is an MVP grammar for the scenario test harness; it does not preclude supporting additional syntaxes later.
- Projection must emit deterministic `anchor` data that allows UI to show “why linked?” and is reproducible on rebuild at `{repository_id, commit_sha, path}`.

**Anchor strategy (Scenario B MVP)**
- For frontmatter locations: `anchor = "fm:<field>"` (e.g., `fm:id`, `fm:refs[0]`).
- For body/content tokens: `anchor = "L<line>:C<col>"` (1-based position of the `ref:` token).
 
UI intent:
- The “why linked?” affordance uses `{commit_sha, path, anchor}` to show the exact origin location (highlight the relevant line/field) at the immutable SHA.


## What to implement by primitive

### Domain primitive (canonical writes + optional publish validation)

**Domain responsibilities**
- Same write boundary as Scenario A: owner-only writes via MR.
- Optional but recommended for Scenario B: **publish validation** that rejects element files missing stable IDs (fail fast).

**Domain commands**
- Reuse Scenario A write loop: `EnsureChange → CreateCommit(actions[]) → PublishChange`.
- Optional validation hooks:
  - `ValidateArtifacts(change_id)` (can be implemented as part of PublishChange or CI validation).

### Projection primitive (derived backlinks/impact; rebuildable index)

**Projection responsibilities**
- Parse authored files at a specific commit SHA (typically published baseline) and extract:
  - element stable IDs (from file content)
  - inline references (from file content)
- Build a rebuildable index for:
  - `id → {path, citation}`
  - backlinks: `target_id → [{source_path, origin_citation, ref_kind}]`
- Provide deterministic query answers for a given `{repository_id, commit_sha}`.

**Projection query surface (minimum)**
- `GetElementById(scope, read_context, element_id)` → `{path, citation}`
- `ListBacklinks(scope, read_context, target_element_id)` → backlinks with origin citations
- `GetContextBundle(scope, read_context, target_element_id, limits…)` → citeable list of `{citation, excerpt?}`

**Edge behavior (must be deterministic)**
- Missing target ID:
  - backlinks can still exist (refs found), but targets resolve as “unresolved”; UI must show “Broken reference: <target_id>” with origin citations.
- Duplicate target ID:
  - treat as “ambiguous target”; UI must show an explicit error with candidate citations.
- Rebuildability:
  - Projection can rebuild the backlinks index from Git alone (no canonical relationship store).

### Memory primitive (projection-owned citeable context)

**Memory responsibilities (Scenario B)**
- No “semantic memory” required.
- Implement “context retrieval” strictly as a projection query that returns a citeable bundle.

**Memory output contract**
- Returns only citations + optional excerpts derived from the cited artifacts at the same `resolved.commit_sha`.
- Never returns uncited “facts”.

### UI surface (UX-pass; derived panels with explainability)

**UI responsibilities**
- Show requirements as Git-authored artifacts:
  - requirements open in the element editor (Document editor); the canonical file path + citation is still visible.
- Provide a derived backlinks/impact panel:
  - explicitly labeled **Derived**
  - every entry has a “why linked?” affordance that reveals origin citation (path + anchor + SHA).
  - broken references are shown explicitly (with origin citations) and are never auto-fixed
  - ambiguous element IDs are shown explicitly (with candidate citations); users must resolve via a governed change
- Provide a “citeable context” panel:
  - list of citations returned by memory bundle retrieval
  - copyable as JSON

#### UI placement (aligned to the current platform app)

The platform already scopes “Enterprise Repository” under `/org/:orgId/repo/:repositoryId/*` and uses `ListPageLayout`.

Scenario B assumes Scenario A already introduced a canonical Git surface (e.g. `/canonical`).

For Scenario B, add **derived views** alongside canonical browsing:
- `/org/:orgId/repo/:repositoryId/derived/requirements` (index)
- `/org/:orgId/repo/:repositoryId/canonical?path=requirements/REQ-001.md` (canonical view)

Also note: the app already has a modal route pattern for opening an “element editor”:
- `.../repo/:repositoryId/@modal/(.)element/:elementId`
Scenario B assumes this becomes the primary “open requirement / open standard” surface (editor-first), and that backlinks/impact/memory panels are implemented as **derived** overlays (no relationship CRUD).

Deep-link nuance (current platform reality):
- Direct `/org/:orgId/repo/:repositoryId/element/:elementId` routes currently redirect back to the repo root.
- Until that is changed, “open in new tab” should use the repo route plus a query param that opens the modal.
  - For Scenario B, `elementId=<stable_element_id>` is the primary mode (e.g. `elementId=REQ-001`), aligned to v6 element identity.
  - Also allow `elementPath=<urlencoded path>` as a compatibility escape hatch for “open by path” (still Git-first), but do not treat paths as element identity.

#### Wireframes (ASCII)

##### Screen 1 — Derived Requirements index (projection-backed)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ <ListPageLayout title="Requirements (Derived)" showActivityPanel={true}>      │
│   <PageHeader                                                                 │
│     title="Requirements (Derived)"                                            │
│     actions={[Rebuild projection] [Copy run report]}                          │
│   />                                                                          │
│   Repo: <repository_id>   Read: [Published ▼]   Resolved: main @ <sha>         │
├──────────────────────────────────────────────────────────────────────────────┤
│ Main (ListPageLayout children)                  activityPanel={<RepositorySidebar />} │
│ ┌──────────────────────── Requirements ─────────────────────────────────────┐ │
│ │ ID       Title                     Path                                   │ │
│ │ REQ-001  Unified repo contract      requirements/REQ-001.md     [Open]    │ │
│ │ REQ-002  ...                        requirements/REQ-002.md     [Open]    │ │
│ └──────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│ Activity panel suggestion:                                                    │
│ - “Derived view” explanation + citations policy                               │
│ - last projection build time, commit SHA indexed                              │
│ </ListPageLayout>                                                             │
└──────────────────────────────────────────────────────────────────────────────┘
```

##### Screen 2 — Requirement detail (element editor + derived impact + chat sidebar)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ <ElementEditorModal />                                                         │
│  ┌──────────────────── <EditorModalChrome /> ─────────────────────────────┐   │
│  │ title="Requirement REQ-001"   kindBadge="document"                      │   │
│  │ actions: [Propose change] [Fullscreen] [Close]                           │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│  Storage: requirements/REQ-001.md                                               │
│  Read: Published @ <sha>                                                        │
│  Citation: {repo, <sha>, "requirements/REQ-001.md"}                             │
│                                                                              │
│  ┌────────────────────────────── main ───────────────────────────────┬──────┐ │
│  │ <Tabs> ... [Document] [Properties] [Derived] [Evidence]            │      │ │
│  │                                                                    │      │ │
│  │ <EditorPluginHost /> (future: Document plugin)                     │ <ModalChatPanel /> │
│  │  Document tab shows canonical content                              │  assistant Q&A       │
│  │                                                                    │                      │
│  │  Derived tab (projection-backed):                                   │                      │
│  │   - Derived Backlinks: standard-a.md                                │                      │
│  │     "why linked?" → origin citation {repo,<sha>,path,anchor}        │                      │
│  │   - Citeable Context bundle (memory)                                │                      │
│  │                                                                    │                      │
│  └────────────────────────────────────────────────────────────────────┴──────┘ │
│  <ModalChatFooter /> (toggle chat / starters)                                   │
└──────────────────────────────────────────────────────────────────────────────┘
```

##### Screen 3 — Propose change (authoring inline refs inside the element editor)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ <ElementEditorModal />                                                        │
│  <EditorModalChrome title="Standard A" subtitle="Draft (proposal)" />         │
│  Draft banner: Change <change_id> • MR !<iid> • <branch_ref> → main            │
│                                                                              │
│  <Tabs> [Document] ...                                                        │
│                                                                              │
│  Document editor content includes inline reference authored in-file:          │
│   ref: REQ-001                                                                │
│                                                                              │
│  Actions: [Save draft] [Publish]                                              │
│   - Save draft → Domain `CreateCommit(actions[])`                             │
│   - Publish → Domain `PublishChange(expected_mr_head_sha)`                    │
│                                                                              │
│  RightSidebarTabs: <ModalChatPanel /> (optional)                              │
│  Footer: <ModalChatFooter />                                                  │
└──────────────────────────────────────────────────────────────────────────────┘
```

Key UX constraint:
- UI may assist users in inserting a reference (autocomplete), but the result must be an **inline token in the file**, not a DB relationship.

## Contract-pass E2E test (Scenario B)

### Mode A — `ameide test` (local-only)

Implementation expectations:
- Use in-process fakes where needed (e.g., Fake GitLab HTTP API), but keep the same Domain/Projection adapters and contracts.

**Steps**
1. Onboard repository mapping.
2. Publish two files via Domain write loop:
   - `requirements/REQ-001.md` (contains stable `id: REQ-001`)
   - `architecture/standards/standard-a.md` (contains `ref: REQ-001`)
3. Projection indexes the published baseline SHA (explicit call or implicit on publish event).
4. Assert derived backlinks:
   - `ListBacklinks(target_element_id="REQ-001")` returns at least `standard-a.md`
   - every backlink includes an origin citation `{repo, <sha>, path, anchor}`
5. Assert memory/context:
   - `GetContextBundle(target_element_id="REQ-001")` returns only citations at the same `<sha>`
6. Negative assertions:
   - missing target ID produces deterministic “unresolved” behavior (with origin citation)
   - requirement without stable ID is rejected (if publish validation is enabled)

### Mode B — `ameide test cluster` (real integration)

Implementation expectations (when cluster is available):
- Use real GitLab and real token posture (dev/local only), and verify the same run report schema.

## Non-goals (Scenario B)

- Proposal-head vs published comparison (Scenario C / v6-03).
- Governance outcomes enforcement (Slice 6).
- Process definitions and deployment (Slice 7).
- Full transformation run (Slice 8).
