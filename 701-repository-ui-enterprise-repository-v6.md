---
title: "701 — Repository UI (v6: GitLab-backed Enterprise Repository view)"
status: draft
owners:
  - platform
  - ui
  - transformation
created: 2026-01-19
related:
  - 694-elements-gitlab-v6.md
  - 695-gitlab-configuration-gitops.md
  - 520-primitives-stack-v6.md
  - 496-eda-principles-v6.md
  - 527-transformation-v6-index.md
---

# 701 — Repository UI (v6: GitLab-backed Enterprise Repository view)

## Functional story (what users do)

This backlog aligns the Ameide Platform UI “Repository” experience with the v6 posture:

- The Enterprise Repository is **canonical Git content** (files), backed by **in-cluster GitLab** (`backlog/694-elements-gitlab-v6.md`).
- GitLab is a **private platform subsystem**; users do not log into GitLab directly.
- The UI is the governance/product surface: users browse, edit, propose, approve, and publish changes using the platform.

### Primary user journey (happy path)

1) User opens an organization and selects an Enterprise Repository.
2) UI shows a repository “workspace”:
   - file tree under `elements/`, `relationships/`, `processes/`
   - baseline selector (published baseline is `main` commit SHA; tags optional)
3) User edits artifacts (docs/diagrams/config, including BPMN process definitions) through the UI.
4) The platform creates/updates a change branch and MR; checks run.
5) Approvals happen in the UI (governance truth in platform DB).
6) Platform merges the MR into `main` and records immutable audit pointers (MR id, pipeline id, commit SHA).
7) Projections refresh (index/search/graph/timeline) from owner facts and/or baseline change signals.

### ProcessDefinitions in the Enterprise Repository

Under v6, ProcessDefinitions are **design-time governed artifacts stored as files** in the Enterprise Repository (not in a Definition Registry):

```text
processes/<module>/<process_key>/v<major>/process.bpmn
```

Promoting a ProcessDefinition (at a high level):

- change the ProcessDefinition files in Git,
- validate against the target runtime (Zeebe/Camunda 8 or Flowable),
- deploy the Process primitive worker implementation that provides the BPMN task side-effects,
- roll out safely and observe (progress/timeline).

Multi-tenancy implications (repo scoping, per-tenant process catalogs) are **explicitly out of scope here**; capture as a separate backlog.

## Non-goals (v6)

- Direct GitLab UI usage as the product surface.
- UI access to GitLab credentials or Git operations.
- Treating GitLab as optional: v6 assumes GitLab is part of the default platform deployment.
- Finalizing the “agent memory model” (IDs/citations/semantic overlays); keep as TBD (projection concern).

## Current implementation reality (what exists today)

The current “Repository” UI is largely **element/graph-shaped**, not Git-backed.

### UI routes and what they do today

- Repository list: `services/www_ameide_platform/app/(app)/org/[orgId]/repo/page.tsx`
  - language is “graphs/elements”, not “repo files”.
  - uses `useRepositories(orgId)` which calls `GET /api/repositories` (see `services/www_ameide_platform/lib/api/hooks.ts` and `services/www_ameide_platform/app/api/repositories/route.ts`).
- Repository root: `services/www_ameide_platform/app/(app)/org/[orgId]/repo/[repositoryId]/page.tsx`
  - lists “ArchiMate views” by calling `transformationKnowledgeQuery.listElements` with `ElementKind.ARCHIMATE_VIEW`.
  - creates a “New view” via `transformationKnowledgeCommand.createElement` (DB-canonical element posture).
- ArchiMate view editor: `services/www_ameide_platform/app/(app)/org/[orgId]/repo/[repositoryId]/archimate/views/[viewId]/page.tsx`
  - reads and writes ArchiMate content via element/version/relationship APIs.
- Definition Registry: `services/www_ameide_platform/app/(app)/org/[orgId]/repo/[repositoryId]/registry/page.tsx`
  - reads `transformationRegistryQuery.listDefinitions` and renders Definition Registry entries.
  - conflicts with v6: ProcessDefinitions should be files under `processes/**`.
- “Open Element” browser dialog: `services/www_ameide_platform/features/elements/graph/ElementRepositoryBrowser.tsx`
  - browses repository “contents” via `useRepositoryData(...)`, which ultimately reads `GET /api/repositories/:id`.
- “Workflow definitions” settings UI: `services/www_ameide_platform/app/(app)/org/[orgId]/settings/workflows/[workflowId]/page.tsx`
  - authors JSON workflow definitions via a separate `workflows` runtime API (not BPMN process definitions stored in Git).

### ConnectRPC proxy posture (already aligned)

The UI calls services via a Connect proxy that keeps the internal gateway private:

- Connect proxy: `services/www_ameide_platform/app/api/proto/[...path]/route.ts`

This is consistent with the GitLab posture: the browser should not talk to GitLab directly either; it should call owner/projection APIs.

## Target-state UI decomposition (v6)

The repository UI becomes “Git-backed files + projections + governance” rather than “graph elements”.

### Repository workspace tabs (suggested)

- **Files**
  - file tree browser rooted at the Enterprise Repository
  - open file viewer/editor by type:
    - Markdown/text
    - images/binaries (preview only)
    - BPMN (`processes/**`) with runtime-specific validation (Zeebe/Flowable)
    - domain-specific editors (e.g., ArchiMate) backed by files (not DB elements)
- **Changes**
  - MR / branch status for the current transformation/change
  - checks + evidence
- **Governance**
  - approvals, required checks, sequencing gates
  - publish/merge actions (platform-owned)
- **Timeline**
  - facts and outcomes anchored to audit pointers (MR id / commit SHA / pipeline id)
- **Search / Graph** (projection)
  - derived index/graph views over files (rebuildable)

### Owner vs projection read/write rules (UI contract)

- UI writes always go to the owning Domain primitive (commands); Domain is the only canonical writer (including Git operations).
- UI reads should prefer Projection query APIs for:
  - file listing, search, graph traversal, impact analysis,
  - timelines and derived views.
- UI should not require GitLab APIs; GitLab remains a private storage adapter behind Domain/Projection.

## Required backend seams to support the UI (high-level)

This backlog does not define the full proto set, but it defines the UI’s minimal needs.

### Projection query surface (read path)

Minimum queries needed to build a repository browser:

- list directory entries (path → children + metadata)
- fetch file content (path + ref → bytes/text + type hints)
- baseline selection support (published baseline commit SHA; optional tags)
- search (full-text + structured as available)
- backlinks/relationships view (derived graph from links + relationship files; code graphs optional)

### Domain command surface (write/govern path)

Minimum commands needed for “edit files with governance”:

- begin/ensure a change (branch + MR) scoped to a repository
- commit/update files on the change branch (idempotent)
- request/record approvals (governance truth)
- merge/publish (merge MR → `main`, record audit pointers, emit facts)

## Deprecations / replacements implied by v6

This backlog makes the following direction explicit:

- **Definition Registry is deprecated** for ProcessDefinitions:
  - replace `repo/<id>/registry` with a `processes/**` file view in the repo workspace.
- “Enterprise Repository UI == element graph browser” is superseded:
  - the repository root view should be file/tree-first, with graph/search as derived projections.
- ArchiMate and other editors must move from “DB element CRUD” to “file-backed artifact editing” (with projections to index/relate).

## Incremental delivery plan (minimize disruption)

1) Add a “Files” tab to the repository workspace that can:
   - render the default layout (`elements/`, `relationships/`, `processes/`)
   - open files read-only from the published baseline.
2) Add “Edit → propose” flow:
   - edit file in UI → Domain creates/updates MR → UI shows checks/governance state.
3) Replace Definition Registry view:
   - stop treating process definitions as registry entries; browse `processes/**` instead.
4) Migrate existing editors:
   - ArchiMate view editor persists to files in `elements/**` (with projection indexing for search/graph).
5) Switch the repository landing page:
   - default to Files (not “ArchiMate views”), with derived navigation layered on top.

## Open questions / explicit TBDs

- Multi-tenancy boundaries for repositories and process catalogs (tenant-only vs shared templates).
- Agent memory model and citations under Git-first (“memory is projection” is accepted; specifics TBD).
- Level of code graph indexing (reuse external graph indexers vs build in-house); treated as projection implementation detail.
