---
title: "527 — Transformation Methodology Dictionary (v6: Git-backed elements, derived repository hierarchy)"
status: draft
owners:
  - transformation
  - platform
created: 2026-01-19
supersedes:
  - 527-transformation-methodology-dictionary.md
related:
  - 694-elements-gitlab-v6.md
  - 701-repository-ui-enterprise-repository-v6.md
  - 527-transformation-projection-v6.md
  - 527-transformation-proto-v6.md
  - 520-primitives-stack-v6.md
---

# 527 — Transformation Methodology Dictionary (v6: Git-backed elements, derived repository hierarchy)

This v6 dictionary keeps the “cross-methodology mapping” goal, but aligns to the current repository posture:

- Canonical authored artifacts are **Git-backed elements** (files in the Enterprise Repository).
- Repository navigation is a folder/file hierarchy in the UI, and that hierarchy is **projection-derived** (not canonical state).
- Relationships are authored as normal references inside files and later reconstructed into a graph by projections.

## Canonical (platform) terms (v6)

- **Enterprise Repository**: a Git-backed repository (GitLab project) that holds canonical authored artifacts (`backlog/694-elements-gitlab-v6.md`).
- **Published baseline**: the commit SHA on `main` (optionally tagged).
- **Element**: a canonical authored artifact stored as a file (or file set) in Git, presented as “element” in the UI (`backlog/701-repository-ui-enterprise-repository-v6.md`).
- **Element reference (audit-grade)**: `{repository_id, commit_sha, path[, anchor]}` (see `backlog/527-transformation-proto-v6.md`).
- **Relationship**: an authored reference within element content (links/IDs); graph is derived.
- **Repository hierarchy node**: a projection-derived file/directory/submodule node used for navigation (not canonical state).

## Repository hierarchy (how the UI should feel)

The UI presents a repository hierarchy as the Git file tree (folders + files) at a selected `read_context`, derived by the Projection.

See `backlog/701-repository-ui-enterprise-repository-v6.md`.

## “Harmonized handshake” anchors (still valid; updated to Git-backed refs)

Regardless of methodology (Scrum vs TOGAF ADM vs PMI), workflows and governance should converge on a tiny set of stable pointers.

Recommended anchor: a “change” element whose outgoing references anchor authoritative snapshots used by governance and workflows.

Minimum anchors (required):

1) `ref:requirement` → authoritative requirement snapshot (Git-backed element reference)
2) `ref:deliverables_root` → deliverables package/root snapshot (Git-backed element reference)

Optional anchors:

3) `ref:baseline` → published baseline pointer (commit SHA on `main`, optionally tagged)
4) `ref:release` → release record (Git-backed artifact and/or external refs as evidence)
5) `evidence:*` → evidence refs (logs/artifacts/links)

The exact representation of these anchors (inline references vs structured metadata) is a convention; the key requirement is that Projection can reconstruct and cite them reproducibly.

Operational note: “inline-only relationships” (including these anchors) are authored inside Git-backed files, not via relationship CRUD. Backlinks/impact/graphs are projection-derived, rebuildable, and citation-grade (see `backlog/694-elements-gitlab-v6.md` and `backlog/656-agentic-memory-v6.md`).
