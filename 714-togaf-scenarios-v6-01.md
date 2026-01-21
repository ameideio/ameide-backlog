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

### Read context (minimum required in Scenario A)

Projection must support:
- `published` → resolves to a deterministic `resolved.commit_sha` for `target_branch` (typically `main`)
- `version_ref.commit_sha` → explicit historical reads for verification/replay

### Citations (minimum)

Every tree node and file read must return:
- `{repository_id, commit_sha, path[, anchor]}`

### Evidence spine (minimum fields for Scenario A)

The platform must be able to produce an audit-grade run/audit record with:
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

Wireframes below assume `ListPageLayout` + an internal split for tree/list, and an editor modal overlay for element editing.

#### Wireframes (ASCII)

##### Screen 1 — Canonical repository browser (published baseline)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ Header (existing): Org switcher • Tabs • Search • User                        │
├──────────────────────────────────────────────────────────────────────────────┤
│ PageHeader: Enterprise Repository                                             │
│  Repo: <repository_id>   Read: [Published ▼]   Resolved: main @ <sha>         │
│  Actions: [Copy citation] [Open in GitLab] [Propose change]                   │
├──────────────────────────────────────────────────────────────────────────────┤
│ Main (ListPageLayout)                              Activity panel (optional)  │
│ ┌──────────── Tree (Git) ────────────┐  ┌───────── File list ──────────────┐ │
│ │ /                                  │  │ architecture/                     │ │
│ │  architecture/                     │  │  - vision.md        (open)        │ │
│ │   > vision.md                      │  │  - statement-of-work.md (open)    │ │
│ │  requirements/                     │  │ README.md                           │ │
│ │  ...                               │  └───────────────────────────────────┘ │
│ └────────────────────────────────────┘                                         │
│                                                                               │
│ Activity panel suggestion (RepositorySidebar-like):                             │
│ - “About / Stats / Recent activity” (derived from commits + projection)         │
└──────────────────────────────────────────────────────────────────────────────┘
```

Behavior notes:
- The “Resolved: main @ `<sha>`” line is the **trust feature**: it is always explicit and copyable.
- `Read: Published` resolves to **target branch head SHA** (merge-method agnostic).
- Missing paths are handled deterministically (GitLab may return `404` for tree paths; UI shows “Not found at `<sha>`”).

##### Screen 2 — Element editor modal (citation-grade read; view mode)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ (Modal) Element editor                                                       │
│ Element: Architecture Vision  (kind: Document)                                │
│ Storage: architecture/vision.md                                               │
│ Read: Published @ <sha>                                                       │
│ Citation: {repository_id, commit_sha:<sha>, path:"architecture/vision.md"}    │
│ Actions: [Copy citation] [History] [Propose change] [Open in GitLab]          │
├──────────────────────────────────────────────────────────────────────────────┤
│ Tabs: [Document] [Properties] [Derived] [Evidence]                            │
│                                                                              │
│ Document (read-only)                                                         │
│  # Architecture Vision                                                       │
│  ...                                                                          │
│                                                                              │
│ Derived/Properties tabs are view-only overlays (projection-backed)            │
└──────────────────────────────────────────────────────────────────────────────┘
```

##### Screen 3 — Propose change (change-based editing inside the element editor)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ (Modal) Element editor — Draft (proposal)                                     │
│ Change: <change_id>    MR: !<iid>    Source: <branch_ref> → Target: main      │
│ Actions: [Save draft] [View MR] [Publish]                                     │
├──────────────────────────────────────────────────────────────────────────────┤
│ Tabs: [Document] [Properties] [Derived] [Evidence]                            │
│                                                                              │
│ Document (editable)                                                          │
│  # Architecture Vision                                                       │
│  ...                                                                          │
│                                                                              │
│ Draft banner                                                                  │
│  - This is a proposal in MR !<iid>                                            │
│  - Save draft → Domain `CreateCommit(actions[])`                              │
│  - Publish → Domain `PublishChange(expected_mr_head_sha)`                     │
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
- Use in-process fakes where needed (e.g., Fake GitLab HTTP API), but keep the same Domain/Projection adapters and contracts.

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
