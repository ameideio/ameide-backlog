---
title: "527 Transformation — UISurface Primitive (v6: repository hierarchy is projection-derived)"
status: draft
owners:
  - transformation
  - platform
created: 2026-01-19
supersedes:
  - 527-transformation-uisurface.md
related:
  - 701-repository-ui-enterprise-repository-v6.md
  - 694-elements-gitlab-v6.md
  - 520-primitives-stack-v6.md
  - 496-eda-principles-v6.md
  - 527-transformation-projection-v6.md
  - 656-agentic-memory-v6.md
---

# 527 Transformation — UISurface Primitive (v6: repository hierarchy is projection-derived)

This v6 updates the Transformation UISurface posture to match the current platform direction:

- Enterprise Repositories are **Git-backed** (GitLab in-cluster, non-optional) (`backlog/694-elements-gitlab-v6.md`).
- The UI presents repository content as an **Enterprise Repository hierarchy** (folders + files) (`backlog/701-repository-ui-enterprise-repository-v6.md`).
- The repository hierarchy is **purely derived by the Projection** (not a canonical “workspace tree” or “workspace node” write model).

## Functional responsibilities (what the UISurface does)

1) Repository navigation (hierarchy-first)
- Render repository browsing as a folder/file tree at a selected `read_context` (published baseline by default).
- Let users browse folders and open **elements** (documents/models/processes) from that hierarchy.
- Surface Git submodules as Git tree `gitlink` entries (do not resolve them to other platform repositories as part of browsing).

2) Editing (element editors; extension-driven)
- Open the correct editor for the element’s underlying file format:
  - Markdown/text,
  - ArchiMate,
  - BPMN ProcessDefinitions (`processes/**/process.bpmn`).
- Editing produces proposals and governed publications (MR merge to `main`).
- Support file/folder operations as governed writes (rename/move paths; folders are Git directories, not separate objects).

3) Governance UX (product surface)
- Show policy, required checks, approvals, and publish/merge actions.
- Show audit pointers and timelines (MR/pipeline/commit SHA).

4) Observability UX (run timeline)
- Render process run timelines and evidence joined from process facts + domain facts (projection-owned).

## What the UISurface explicitly does NOT do (v6)

- It does not bypass the owner-only write rule by performing Git operations directly from the browser.
- It does not manage a canonical “workspace tree” / “workspace nodes”.
- It does not manage a Definition Registry for ProcessDefinitions.
  - ProcessDefinitions are governed artifacts stored as files in the Enterprise Repository and surfaced via repository browsing/search.

## Read/write rules (hard boundary)

- Reads: via Projection query services (`backlog/527-transformation-projection-v6.md`).
- Writes: via Domain commands (owner-only writes; Git operations are platform-owned) (`backlog/496-eda-principles-v6.md`).
- The browser never talks to GitLab directly; GitLab is a private storage adapter behind Domain/Projection (`backlog/694-elements-gitlab-v6.md`).

## Derived repository hierarchy (projection contract)

The UISurface does not persist hierarchy nodes. Instead, it relies on the Projection to provide:

- folder/file hierarchy nodes,
- directory listings and counts,
- stable element identifiers and baseline selectors (`published` == `main` commit SHA),
- backlinks/relationships and search as derived views.

See `backlog/701-repository-ui-enterprise-repository-v6.md` for the repository UI contract.
