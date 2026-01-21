---
title: "714 — v6 Scenario Slices v6-01 (Scenario A: Governed Publish of Architecture Vision)"
status: draft
owners:
  - platform
  - transformation
  - repository
  - ui
created: 2026-01-21
updated: 2026-01-21
related:
  - 714-togaf-scenarios-v6.md
  - 703-transformation-enterprise-repository-proto-v1.md
  - 704-v6-enterprise-repository-memory-e2e-implementation-plan.md
  - 710-gitlab-api-token-contract.md
---

# 714 v6-01 — Scenario A implementation consolidation

This document is an implementer-oriented consolidation for **Scenario A** from `backlog/714-togaf-scenarios-v6.md`.

Goal: specify, in concrete terms, what must be implemented across primitives to make Scenario A **real**, **GitLab‑aligned**, and **audit-grade**.

## Scenario A (recap)

**User story**
- As an enterprise architect, I can propose and publish an Architecture Vision (and Statement of Architecture Work) so it becomes the canonical baseline, citeable by commit SHA.

**What this scenario proves (v6 posture)**
- **Git is canonical**: authored artifacts are files at immutable commit SHAs.
- **MR is the proposal unit**: all canonical changes flow through a Merge Request.
- **Owner-only writes**: only the owning Domain performs canonical git operations.
- **Audit anchor is merge-method agnostic**: the canonical publish anchor is **target branch head SHA after publish** (merge/squash SHAs are supplemental).
- **Reads are citation-grade**: every read response is anchored to `{repository_id, commit_sha, path[, anchor]}`.

## EDA alignment (496)

Scenario A is a concrete application of `backlog/496-eda-principles-v6.md`:

* `EnsureChange`, `CreateCommit`, `PublishChange` are **Commands** addressed to the Domain (the Owner).
  * They must be idempotent (idempotency keys) and must propagate correlation/causation metadata across calls.
* The Domain emits **Facts** only after commit:
  * After publish succeeds and the Domain durably records audit pointers (MR id, `target_head_sha`, optional merge/squash SHAs), it may emit a `ChangePublished` fact.
  * Projections consume that fact (or the recorded audit pointers) to converge derived views.
* Projection is a **derived read model**:
  * it can update incrementally from facts, but must remain rebuildable from Git + audit pointers (no canonical relationship/state store).

## Capability test placement (590/591)

Scenario A’s contract-pass test should be treated as a **capability-owned vertical slice test** (Transformation capability), not a primitive-owned invariant test.

Place it under the repo’s `capabilities/` composition boundary (per `backlog/590-capabilities.md`), typically under something like:

* `capabilities/transformation/__tests__/integration/` (target state), or
* `capabilities/transformation/features/enterprise_repository/integration/` (common current pattern).

## Minimal repo artifacts

Scenario A uses these canonical file paths:
- `architecture/vision.md`
- `architecture/statement-of-work.md`

## Contracts and data shapes

### Identity (required everywhere)

- `{tenant_id, organization_id, repository_id}`

These identifiers must be carried through request contexts, events/facts, logs, and the run report.

### Repository mapping (required)

Canonical mapping from `{tenant_id, organization_id, repository_id}` to the vendor remote:
- `provider = gitlab`
- `remote_id = <gitlab project id or path>` (opaque to UX; policy-level detail)
- `published_ref = main` (default baseline branch)

**Onboarding note (normative)**
- Treat “repository onboarding / mapping upsert” as a **single shared platform capability** (admin/bootstrap), not something each Domain invents differently.
- Domain consumes this mapping for all reads/writes, but onboarding policy and UX should be centralized (one “Repository Onboarding” primitive/service contract).

### Read context (minimum required in Scenario A)

Projection must support:
- `published` → resolves to a deterministic `resolved.commit_sha` for `target_branch` (typically `main`)
- `version_ref.commit_sha` → explicit historical reads for verification/replay

### Citations (minimum)

Every tree node and file read must return:
- `{repository_id, commit_sha, path[, anchor]}`

### Evidence spine (minimum fields for Scenario A)

Use `EvidenceSpineViewModel` from `backlog/714-togaf-scenarios-v6.md` as the shared shape across UI and contract-pass runs.

Scenario A requires (minimum):
- Identity: `{tenant_id, organization_id, repository_id}`
- Proposal:
  - `mr_iid`
  - `proposal_head_sha` (best-effort; may require retry/backoff if MR diff fields are async)
- Publish:
  - `target_branch` (usually `main`)
  - `target_head_sha` (**required**, the canonical publish anchor)
  - optional: `merge_commit_sha`, `squash_commit_sha` (supplemental evidence; may vary by merge method)
- Citations:
  - citeable reads performed during verification (`GetContent` results anchored to `target_head_sha`)

## What to implement by primitive

### Domain primitive (canonical writes + publish evidence)

**Domain responsibilities**
- Own the write boundary: only Domain is configured with GitLab write credentials.
- Implement governed change lifecycle: **EnsureChange → Commit batch → Publish**.
- Enforce concurrency via SHA guards (`expected_last_commit_id`, `expected_mr_head_sha`).
- Emit/record evidence spine on publish.

**Domain commands (minimum)**
- `UpsertRepositoryGitRemote(scope, provider, remote_id, idempotency_key)`
  - classify as **platform onboarding** (admin/bootstrap), not part of the everyday change lifecycle
  - prefer implementing this behind a single shared “Repository Onboarding” service contract, then having the Domain consume the resulting mapping
- `EnsureChange(scope, idempotency_key)` → returns:
  - `change_id`, `branch_ref`, `mr_iid`, `last_commit_id`
- `CreateCommit(change_id, actions[], expected_last_commit_id, idempotency_key)` → returns:
  - `head_commit_sha`, `last_commit_id`
- `PublishChange(change_id, expected_mr_head_sha)` → returns:
  - publish evidence (must include `target_head_sha`)

**Domain invariants to enforce**
- Never write directly to `main`; always propose via branch+MR.
- Publish must be SHA-safe: reject if the MR head does not match the expected SHA.
- Fail fast (no compatibility shims): mismatch → explicit error.

**Domain events/facts (minimum)**
- `RepositoryGitRemoteUpserted`
- `ChangePublished` (must include: `mr_iid`, `target_branch`, `target_head_sha`, and citations for changed paths)

### Projection primitive (reads + citation-grade responses)

**Projection responsibilities**
- Resolve `read_context` → `resolved.commit_sha` deterministically.
- Serve Git-tree reads strictly from Git (no synthetic hierarchy).
- Return citations on every node and file.
- Handle vendor edge cases deterministically:
  - missing path may be `404` (not empty list)
  - ref semantics must support commit SHA for tree/file reads in the target environment (or define a tested fallback)

**Projection query surface (minimum)**
- `ListTree(scope, read_context, path)` → nodes + `read_context.resolved.commit_sha` + citations
- `GetContent(scope, read_context, path)` → bytes + `read_context.resolved.commit_sha` + citation

**Convergence rule**
After publish, `published` must resolve to the new `target_head_sha` (avoid stale caches).

### UI surface (UX-pass, minimal but real)

UI is not required for contract-pass initially, but Scenario A’s UX-pass must reflect the same contracts.

**UI responsibilities**
- Git-tree-first repository browser:
  - show “Published @ <resolved_sha>”
- Element editor (not a raw file viewer):
  - open “elements” in an editor surface (modal is preferred, aligned to existing routing)
  - still display canonical storage `{path}` + citation `{repo_id, sha, path[, anchor]}` as a trust feature
  - Scenario A can open by **path** (stable IDs are not required yet); Scenario B introduces ID-based navigation (`element_id → path`) via Projection
- Change-based editing (not DB CRUD):
  - “Propose change” → Domain `EnsureChange`
  - “Save” → Domain `CreateCommit`
  - “Publish” → Domain `PublishChange`
- Evidence display:
  - show MR id + `target_head_sha` after publish

#### UI placement (aligned to the current platform app)

The current platform app already uses `/org/:orgId/repo/:repositoryId/*` as the “Enterprise Repository” scope and renders pages with `ListPageLayout` (two-column main + optional right activity panel). This scenario adds a **canonical Git** surface inside that scope:

- `GET /org/:orgId/repo/:repositoryId` currently acts like “Repository home” (today: ArchiMate views).
- Add a repository-local nav (or repo-local tabs) that includes a **Canonical (Git)** entry, e.g.:
  - `/org/:orgId/repo/:repositoryId/canonical` (or `/git`, `/browse`)

Also note: the app already has a modal route pattern for opening an “element editor”:
- `.../repo/:repositoryId/@modal/(.)element/:elementId`
but `ElementEditorModal` is currently a placeholder. Scenario A requires implementing a real element editor host and re-wiring persistence to v6 Domain commands (not `elementService.updateElement`).

Deep-link nuance (current platform reality):
- Direct `/org/:orgId/repo/:repositoryId/element/:elementId` routes currently redirect back to the repo root.
- Until that is changed, “open in new tab” should use the repo route plus a query param that opens the modal.
  - In Scenario A, prefer `elementPath=<urlencoded path>` to avoid implying stable ID semantics too early.
  - Once Projection supports `element_id → path` resolution (Scenario B), `elementId=<element_id>` becomes the primary addressing mode.

Wireframes below assume `ListPageLayout` + an internal split for tree/list, and an editor modal overlay for element editing.

#### Wireframes (ASCII)

##### Screen 1 — Canonical repository browser (published baseline)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ <HeaderClient /> (existing)                                                   │
│  - <ScopeTrail /> • <NavTabs /> • Search • User                                │
├──────────────────────────────────────────────────────────────────────────────┤
│ <ListPageLayout title="Enterprise Repository" showActivityPanel={true}>       │
│   <PageHeader                                                                │
│     title="Enterprise Repository"                                             │
│     description="Canonical Git-backed artifacts at a resolved commit SHA."    │
│     actions={[Copy citation] [Open in GitLab] [Propose change]}               │
│   />                                                                         │
│  Repo: <repository_id>   Read: [Published ▼]   Resolved: main @ <sha>         │
├──────────────────────────────────────────────────────────────────────────────┤
│ Main (ListPageLayout children)                  activityPanel={<RepositorySidebar />} │
│ ┌──────────── Tree (Git) ────────────┐  ┌───────── File list (Git) ────────┐ │
│ │ /                                  │  │ architecture/                     │ │
│ │  architecture/                     │  │  - vision.md        (open)        │ │
│ │   > vision.md                      │  │  - statement-of-work.md (open)    │ │
│ │  requirements/                     │  │ README.md                           │ │
│ │  ...                               │  └───────────────────────────────────┘ │
│ └────────────────────────────────────┘                                         │
│                                                                               │
│ Activity panel suggestion (RepositorySidebar-like):                             │
│ - “About / Stats / Recent activity” (derived from commits + projection)         │
│ </ListPageLayout>                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

Behavior notes:
- The “Resolved: main @ `<sha>`” line is the **trust feature**: it is always explicit and copyable.
- `Read: Published` resolves to **target branch head SHA** (merge-method agnostic).
- Missing paths are handled deterministically (GitLab may return `404` for tree paths; UI shows “Not found at `<sha>`”).

##### Screen 2 — Element editor modal (citation-grade read; view mode + chat sidebar)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ <ElementEditorModal /> (Radix <Dialog />)                                     │
│  ┌──────────────────── <EditorModalChrome /> ─────────────────────────────┐  │
│  │ title="Architecture Vision"   kindBadge="document"                      │  │
│  │ actions: [Open in new tab] [Fullscreen] [Close]                         │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│  Storage: architecture/vision.md                                              │
│  Read: Published @ <sha>                                                      │
│  Citation: {repository_id, commit_sha:<sha>, path:"architecture/vision.md"}   │
│                                                                              │
│  ┌────────────────────────────── main ───────────────────────────────┬──────┐ │
│  │ <Tabs> <TabsList> <TabsTrigger>                                    │      │ │
│  │  [Document] [Properties] [Derived] [Evidence]                      │      │ │
│  │                                                                   <RightSidebarTabs> │
│  │ <EditorPluginHost /> (mocked in current shell)                     │ <ModalChatPanel /> │
│  │  - Document (read-only)                                            │  "Element assistant" │
│  │  - Properties/Derived/Evidence are overlays                         │  (messages area)     │
│  │                                                                    </RightSidebarTabs> │
│  └────────────────────────────────────────────────────────────────────┴──────┘ │
│  <ModalChatFooter /> (toggle chat / starters; bottom bar)                      │
└──────────────────────────────────────────────────────────────────────────────┘
```

##### Screen 3 — Propose change (change-based editing inside the element editor + chat sidebar)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ <ElementEditorModal />                                                        │
│  ┌──────────────────── <EditorModalChrome /> ─────────────────────────────┐  │
│  │ title="Architecture Vision"   subtitle="Draft (proposal)"              │  │
│  │ actions: [View MR] [Publish] [Fullscreen] [Close]                      │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│  Draft banner: Change <change_id> • MR !<iid> • <branch_ref> → main            │
│                                                                              │
│  ┌────────────────────────────── main ───────────────────────────────┬──────┐ │
│  │ <Tabs> ... [Document] [Properties] [Derived] [Evidence]            │      │ │
│  │                                                                    │      │ │
│  │ <EditorPluginHost /> (future: Document plugin)                     │ <ModalChatPanel /> │
│  │  - editing the document content                                    │  assistant context   │
│  │  - Save draft → Domain `CreateCommit(actions[])`                    │  Q&A / impact notes  │
│  │                                                                    │                      │
│  └────────────────────────────────────────────────────────────────────┴──────┘ │
│  <ModalChatFooter />  (ask assistant; opens sidebar)                           │
└──────────────────────────────────────────────────────────────────────────────┘
```

Behavior notes:
- `Propose change` calls Domain `EnsureChange` and returns `{change_id, mr_iid, branch_ref}` immediately.
- `Save draft` can create one commit per save, or batch multiple file edits per commit; chunking is allowed but must preserve citations.
- The editor is “element-first” UX, but persistence is Git-first: edits become **file actions** on a change branch (no canonical DB authoring surface).

##### Screen 4 — Publish confirmation (evidence spine)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ Published                                                                      │
│ Target branch: main                                                            │
│ Published baseline SHA (anchor): <target_head_sha>                             │
│ MR: !<iid>  (supplemental: merge_commit_sha / squash_commit_sha if present)    │
│ Actions: [Open published baseline] [Copy run report]                           │
├──────────────────────────────────────────────────────────────────────────────┤
│ Run report (audit-grade)                                                       │
│ - identity: {tenant_id, organization_id, repository_id}                        │
│ - publish: {target_branch:"main", target_head_sha:"..."}                       │
│ - citations:                                                                    │
│   - {repo, <target_head_sha>, "architecture/vision.md"}                        │
│   - {repo, <target_head_sha>, "architecture/statement-of-work.md"}             │
└──────────────────────────────────────────────────────────────────────────────┘
```

Failure/edge-case UX (must be explicit and fail-fast):
- **Publish SHA mismatch** (MR head advanced): show “Publish blocked: proposal changed” and require explicit refresh/retry (no silent rebase).
- **MR diff refs async**: UI may need to “wait until proposal SHA is available” before enabling publish evidence checks.
- **Pipeline gating**: if used, use GitLab “auto-merge” wording (not deprecated “merge when pipeline succeeds”) and surface pipeline status in the change header.

### Agent (optional; proposal/evidence only)

If included in Scenario A:
- Reads via Projection only (citations).
- Produces draft text / summaries with citations.
- Submits proposals via Domain commands (never direct GitLab writes).

### Process (optional; orchestration only)

If included in Scenario A:
- Orchestrates the sequence (EnsureChange → Commit → Publish) by calling Domain commands.
- Must not perform git operations directly.

## GitLab substrate mapping (implementation detail)

### Reads (projection)
- List tree: `GET /projects/:id/repository/tree?ref=<sha>&path=<...>`
- Read file content: `GET /projects/:id/repository/files/:file_path/raw?ref=<sha>`

### Writes (domain)
- Create branch: `POST /projects/:id/repository/branches?branch=<...>&ref=main`
- Find MR by source branch: `GET /projects/:id/merge_requests?state=opened&source_branch=<...>&target_branch=main`
- Create MR: `POST /projects/:id/merge_requests`
- Create commit with actions[]: `POST /projects/:id/repository/commits` (supports `create`, `update`, `move`, `delete`)
- Merge MR with SHA guard: `PUT /projects/:id/merge_requests/:iid/merge` with `sha=<expected_head>`

### Merge method nuance (must be reflected in evidence)
- Record `target_head_sha` (post-merge `main` head) as canonical.
- Record merge/squash SHAs only as supplemental fields.

### Rate/size limits (must be planned for early)
- Commit batching must chunk actions/commits under vendor size limits.
- “Citeable context bundles” must chunk reads and apply cache/backoff.
- Runner must surface throttling/fallbacks in the run report (so evidence remains explainable).

## Contract-pass E2E test (Scenario A)

### Mode A — `ameide test` (local-only)

Implementation expectations:
- Use in-process mocks where needed (e.g., an in-memory GitLab HTTP API), but keep the same Domain/Projection adapters and contracts.

**Steps**
1. Onboard repository mapping.
2. EnsureChange → obtain `{change_id, branch_ref, mr_iid, last_commit_id}`.
3. CreateCommit (batched actions) → write both files.
4. PublishChange with `expected_mr_head_sha`.
5. Read back from `published` via Projection and assert:
   - `read_context.resolved.commit_sha == target_head_sha`
   - content matches expected edits
   - citations match `{repository_id, target_head_sha, path}`
6. Assert evidence spine includes:
   - `mr_iid`, `target_branch`, `target_head_sha`

**Negative assertions (vendor-aligned)**
- Missing path is handled deterministically (e.g., `404` → NotFound).
- Stale `expected_mr_head_sha` fails publish fast.

### Mode B — `ameide test cluster` (real integration)

Implementation expectations (when cluster is available):
- Use real GitLab and real token posture (dev/local only) and verify the same run report schema.

## Non-goals (Scenario A)

- Derived backlinks/impact/memory (covered in later slices/scenarios).
- Governance outcomes (Slice 6).
- Process definition files and deployment (Slice 7).
- Full transformation run (Slice 8).
