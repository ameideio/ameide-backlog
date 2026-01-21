---
title: "706 — Transformation v6 (Coordinated implementation plan: increments across primitives)"
status: draft
owners:
  - transformation
  - platform
  - ui
created: 2026-01-20
related:
  - 496-eda-principles-v6.md
  - 520-primitives-stack-v6.md
  - 527-transformation-v6-index.md
  - 660-transformation-primitive-v4-goals-invariants.md
  - 668-transformation-primitive-v4-implementation-plan-v2.md
  - 694-elements-gitlab-v6.md
  - 701-repository-ui-enterprise-repository-v6.md
  - 703-transformation-enterprise-repository-proto-v1.md
  - 704-v6-enterprise-repository-memory-e2e-implementation-plan.md
  - 705-transformation-agentic-alignment-v6.md
  - 656-agentic-memory-v6.md
  - 650-agentic-coding-overview-v6.md
  - 617-transformation-uisurface-wireframes.md
  - 710-gitlab-api-token-contract.md
---

# 706 — Transformation v6 (Coordinated implementation plan: increments across primitives)

This backlog exists because multiple v6 tracks must ship **together** to produce a usable end-to-end (E2E) loop:

- Proto/contract seams
- Transformation Domain/Process/Projection
- Enterprise Repository (GitLab CE) + Repository UI (Git tree)
- Agents (executor + architect) and Memory
- Platform app UI surface (Transformation/Repository screens)

It is a **coordination doc**: it does not replace detailed backlogs (704, 668, 656, 650). It defines a single set of increments so we keep those backlogs aligned.

## 0) Non-negotiables (v6)

These apply to every increment:

1) **Scope identity is explicit**
- Target is always `{tenant_id, organization_id, repository_id}`; optional `initiative_id` is additive, never a substitute.

1.1) **No pre-seeded data for increment verification**
- Exit criteria for every increment must be provable without relying on pre-existing GitLab projects or platform rows.
- The verification harness must create a fresh GitLab project (with an initial commit, e.g., `initialize_with_readme=true`), onboard/register the platform mapping, run the increment loop, and best-effort cleanup (delete project).

2) **Enterprise Repository is canonical Git**
- Canonical authored artifacts are Git files in the tenant Enterprise Repository (GitLab): `backlog/694-elements-gitlab-v6.md`.

3) **Repository hierarchy == Git tree**
- UI browsing hierarchy is folders + files derived from the Git tree at a specific `read_context` (commit SHA), not a separate “workspace node tree”: `backlog/701-repository-ui-enterprise-repository-v6.md`.

3.1) **Virtual hierarchies are derived-only**
- Any TOGAF/category views are derived projections over Git files; they must not become a separate canonical hierarchy or a write surface.

4) **Relationships are inline-only**
- Relationships exist only as inline references embedded in element content; projections derive backlinks/graphs from inline references only (no `relationships/**`, no relationship CRUD): `backlog/701-repository-ui-enterprise-repository-v6.md`.

5) **GitLab CE only**
- Governance is platform-owned; GitLab is storage + CI + CE enforcement primitives (protected `main`, MR-only merges, pipeline gating as configured): `backlog/694-elements-gitlab-v6.md`.

6) **Submodules are Git-level only**
- Treat submodules as `gitlink` entries in the tree; do not resolve/flatten into other platform repositories during browsing: `backlog/701-repository-ui-enterprise-repository-v6.md`.

7) **Owner-only writes + contract-first integration**
- Canonical writes go through the owning Domain; inter-primitive seams follow `backlog/496-eda-principles-v6.md`. GitLab REST is an internal storage adapter, not an inter-primitive integration surface.

## 1) What needs alignment in 600+ backlogs (actionable deltas)

This list is intentionally short: only items that affect coordinated v6 delivery.

- **UISurface wireframes drift:** `backlog/617-transformation-uisurface-wireframes.md` still implies “relationships/workspace assignments” and “folder assignment” concepts; v6 requires Git-tree-first browsing and inline-only relationships.
- **Editor drift:** `backlog/649-element-editor-refactoring-proposal.md` and `backlog/641-camunda-bpmn-element-editor-platform.md` are Element-API/DB-centric; v6 Enterprise Repository editing is Git-backed (MR/change-based). Reuse the UI plugin architecture if desired, but do not treat Element DB CRUD as the v6 canonical editor contract.
- **Local registry drift:** `backlog/622-gitops-hardening-k3d-aks.md` must not recommend local registries for Argo-managed workflows; v6 image posture is GHCR + digest-pinned (no local registries).

## 2) Increment map (the one coordinated v6 ladder)

Each increment is “done” only when there is an observable E2E loop (even if minimal) and the involved backlogs can be updated to mark that increment complete.

**Implementation progress notation (by phase)** (used at the end of each increment):
- `Provisioning/Onboarding`: seedless repo create + platform mapping registration (no pre-seeded data)
- `Proto`, `Domain`, `Projection`, `UI`, `Agents/MCP`, `Process`, `GitOps`, `Seedless E2E`

### Increment 0 — Contract + identity lock (no product UI required)

**Outcome**
- v6 identity and invariants are explicit and unambiguous in code and docs.

**Proto/contract**
- Lock the Enterprise Repository seam (`io.ameide.transformation.enterprise_repository.v1`) as “Git tree + governed change/publish”: `backlog/703-transformation-enterprise-repository-proto-v1.md`.
- Lock the minimum citation shape for reads/memory: `{repository_id, commit_sha, path[, anchor]}` (and `read_context` that resolves to commit SHA): `backlog/701-repository-ui-enterprise-repository-v6.md`, `backlog/656-agentic-memory-v6.md`.

**Domain/Projection**
- Ensure owner-only write posture is enforced (GitLab API access remains internal to owner/projection adapters): `backlog/496-eda-principles-v6.md`.

**Cross-read**
- `backlog/704-v6-enterprise-repository-memory-e2e-implementation-plan.md`
- `backlog/705-transformation-agentic-alignment-v6.md`

**Implementation progress (by phase; 2026-01-20):** Provisioning/Onboarding: not started; Proto: done; Domain: done; Projection: done; UI: n/a; Agents/MCP: n/a; Process: n/a; GitOps: in-progress; Seedless E2E: not started.

### Increment 1 — Read-only Enterprise Repository (Git tree + citations)

**Outcome**
- Platform app can browse the Enterprise Repository Git tree and open files at a resolved commit SHA; reads return citations.

**Bootstrap (required for exit criteria)**
- Create a tenant GitLab project for the test run (ensure at least one commit) and onboard/register it as a platform `{tenant_id, organization_id, repository_id}`.
- Browsing/read APIs operate only on platform `repository_id` (GitLab `project_id` remains an internal pointer).

**Projection**
- `ListTree`/`GetContent` over Git with `read_context → commit_sha` resolution; return `FILE | DIRECTORY | GITLINK`: `backlog/704-v6-enterprise-repository-memory-e2e-implementation-plan.md`.

**UI surface**
- Repository screen uses the Git tree as the hierarchy (Radix-aligned tree/list), supports folder navigation, and shows `commit_sha` for the current read context: `backlog/701-repository-ui-enterprise-repository-v6.md`, `backlog/617-transformation-uisurface-wireframes.md`.

**Memory**
- Minimal memory is allowed to be “fetch these cited paths” (no semantic search yet) as long as citations are commit-SHA anchored: `backlog/656-agentic-memory-implementation-mvp-v6.md`.

**Implementation progress (by phase; 2026-01-20):** Provisioning/Onboarding: not started; Proto: done; Domain: done; Projection: done; UI: not started; Agents/MCP: n/a; Process: n/a; GitOps: in-progress; Seedless E2E: not started.

### Increment 2 — Governed write loop (change → commit → publish)

**Outcome**
- A user can draft changes (branch/MR), commit edits (batched), and publish to `main` via SHA-safe merge; audit pointers are recorded.

**Domain**
- Implement `EnsureChange` + commit batching + last-known SHA guards + SHA-safe publish; record audit pointer set (repo/project id, MR IID, pipeline id+status if applicable, resulting `main` commit SHA): `backlog/701-repository-ui-enterprise-repository-v6.md`, `backlog/704-v6-enterprise-repository-memory-e2e-implementation-plan.md`.

**GitOps**
- Token posture is least-privilege and matches the contract: `backlog/710-gitlab-api-token-contract.md`.

**UI surface**
- Editor UX is change-based (working branch), not “direct write”; publish is an explicit governed action.
- Folder operations are real Git operations performed on a change branch (rename/move paths via commit batching; no “empty folders”).

**Implementation progress (by phase; 2026-01-20):** Provisioning/Onboarding: not started; Proto: done; Domain: done; Projection: done; UI: not started; Agents/MCP: n/a; Process: n/a; GitOps: in-progress; Seedless E2E: not started.

### Increment 3 — Relationships + Memory (derived, projection-owned)

**Outcome**
- Backlinks/impact/graph views work, derived only from inline references; memory retrieval returns citeable context.

**Projection**
- Inline reference extraction + backlink derivation; optional indexing/semantic retrieval remains projection-owned and rebuildable: `backlog/656-agentic-memory-v6.md`, `backlog/535-mcp-read-optimizations.md`.

**UI surface**
- Relationship/impact views are derived-only and make the derivation rule explicit (no relationship artifacts): `backlog/701-repository-ui-enterprise-repository-v6.md`.

**Implementation progress (by phase; 2026-01-20):** Provisioning/Onboarding: not started; Proto: not started; Domain: n/a; Projection: not started; UI: not started; Agents/MCP: not started; Process: n/a; GitOps: n/a; Seedless E2E: not started.

### Increment 4 — ProcessDefinitions as files (design-time + deploy posture)

**Outcome**
- ProcessDefinitions exist as governed files in the Enterprise Repository and can be validated and deployed to the runtime.

**Repo layout**
- ProcessDefinitions live under `processes/<module>/<process_key>/v<major>/process.bpmn`; `process_key` is the BPMN `<process id>`: `backlog/694-elements-gitlab-v6.md`, `backlog/701-repository-ui-enterprise-repository-v6.md`.

**Process primitive**
- Process v4 delivery aligns to file-backed ProcessDefinitions and cluster smokes: `backlog/668-transformation-primitive-v4-implementation-plan-v2.md`.

**Implementation progress (by phase; 2026-01-20):** Provisioning/Onboarding: not started; Proto: not started; Domain: n/a; Projection: not started; UI: not started; Agents/MCP: n/a; Process: in-progress (repo-only exists; cluster deploy pending); GitOps: in-progress; Seedless E2E: not started.

### Increment 5 — Transformation capability E2E (domain + process + agent + projection + UI)

**Outcome**
- A minimal Transformation scenario runs end-to-end with the real primitives: domain fact → process start → executor work → governed publish → projection/UI shows progress and evidence.

**Process**
- Kafka→Zeebe ingress and worker semantics per v4 plan: `backlog/668-transformation-primitive-v4-implementation-plan-v2.md`.

**Agents**
- Executor uses the DevX execution substrate (Coder tasks + CLI front doors) without violating owner-only writes: `backlog/650-agentic-coding-overview-v6.md`, `backlog/705-transformation-agentic-alignment-v6.md`.

**UI surface**
- Kanban/progress is projection-owned; “move card” is an intent that reconciles to projection truth: `backlog/617-transformation-uisurface-wireframes.md`, `backlog/614-kanban-projection-architecture.md`.

**Implementation progress (by phase; 2026-01-20):** Provisioning/Onboarding: not started; Proto: in-progress; Domain: in-progress; Projection: in-progress; UI: not started; Agents/MCP: in-progress; Process: in-progress; GitOps: in-progress; Seedless E2E: not started.

## 3) How to keep sub-plans aligned (ongoing discipline)

- Treat 706 increments as the “program ladder”.
- Keep detailed plans as “how” documents:
  - 704: Enterprise Repository + Memory E2E plan
  - 668: Process v4 delivery plan
  - 656: Memory MVP plan(s)
  - 650 suite: DevX agentic coding execution substrate
- When a sub-plan adds an increment, either:
  - map it to an existing 706 increment, or
  - update 706 first (so numbering and dependencies remain consistent).
