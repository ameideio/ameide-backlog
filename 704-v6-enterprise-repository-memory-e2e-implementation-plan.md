---
title: "704 — v6 Enterprise Repository + Memory (E2E implementation plan: cross-primitive increments)"
status: draft
owners:
  - platform
  - transformation
  - ui
created: 2026-01-19
related:
  - 300-400/329-authz.md
  - 496-eda-principles-v6.md
  - 509-proto-naming-conventions-v6.md
  - 511-process-primitive-scaffolding-v3.md
  - 520-primitives-stack-v6.md
  - 527-transformation-v6-index.md
  - 527-transformation-e2e-sequence-v6.md
  - 527-transformation-methodology-dictionary-v6.md
  - 527-transformation-projection-v6.md
  - 527-transformation-proto-v6.md
  - 527-transformation-uisurface-v6.md
  - 534-mcp-protocol-adapter.md
  - 535-mcp-read-optimizations.md
  - 536-mcp-write-optimizations.md
  - 656-agentic-memory-v6.md
  - 656-agentic-memory-implementation-v6.md
  - 656-agentic-memory-implementation-mvp-v6.md
  - 657-transformation-domain-clean-target-v2.md
  - 694-elements-gitlab-v6.md
  - 695-gitlab-configuration-gitops.md
  - 701-repository-ui-enterprise-repository-v6.md
  - 703-transformation-enterprise-repository-proto-v1.md
---

# 704 — v6 Enterprise Repository + Memory (E2E implementation plan: cross-primitive increments)

This is the cross-primitive, end-to-end (E2E) implementation plan for the v6 posture:

- **Enterprise Repository = canonical Git files**
- **Repository UI hierarchy = Git tree**
- **Agentic “memory” = projection-owned retrieval over Git + citations**
- **Governance truth lives in the platform; GitLab CE provides storage + CI + enforcement primitives**

The intent is to ship **usable E2E increments**: each increment advances proto seams + Domain/Projection + UI + (where applicable) agents/processes, and has a product-visible “loop” that can be exercised.

## 0) Non‑negotiables (normative; apply to every increment)

These are requirements, not “guidelines”:

1) **Scope identity is always explicit**
- Target scope is `{tenant_id, organization_id, repository_id}` (no workspace id for repo browsing).
- Reference: `backlog/527-transformation-proto-v6.md`, `backlog/496-eda-principles-v6.md`.

2) **Hierarchy equals Git tree**
- Repository browsing hierarchy is the Git file tree (folders + files) at a selected `read_context`.
- No optional “virtual hierarchy conventions” for Enterprise Repository browsing; if the path exists in Git, it exists in the UI. Folder operations are first-class in UX (implemented as Git path changes).
- Reference: `backlog/701-repository-ui-enterprise-repository-v6.md`, `backlog/703-transformation-enterprise-repository-proto-v1.md`.

3) **Relationships are inline-only**
- Relationships are only inline references inside element content; no relationship sidecar files, no `relationships/**` folder, no relationship CRUD.
- Graph/backlinks are projection-derived only.
- Reference: `backlog/694-elements-gitlab-v6.md`, `backlog/701-repository-ui-enterprise-repository-v6.md`, `backlog/656-agentic-memory-v6.md`.

4) **GitLab CE only**
- Do not depend on GitLab Premium/Ultimate features (approvals framework, external status checks, etc.).
- Platform owns governance truth and approval logic; GitLab enforces only via CE primitives (protected branch, MR-only merges, pipeline gating as configured).
- Reference: `backlog/694-elements-gitlab-v6.md`, `backlog/701-repository-ui-enterprise-repository-v6.md`, `backlog/695-gitlab-configuration-gitops.md`.

5) **Owner-only writes; Git provider APIs are internal adapters**
- UISurface/Agents/Processes do not call GitLab REST directly for canonical operations; they call the owner’s command surface.
- Calling GitLab REST is a primitive-internal implementation detail of Domain (write) and Projection (read/index).
- Reference: `backlog/496-eda-principles-v6.md`.

6) **Submodules are Git-level only**
- Treat submodules as Git tree `gitlink` entries (surface them as such); do not resolve/flatten into other repositories as part of browsing.
- Reference: `backlog/701-repository-ui-enterprise-repository-v6.md`, `backlog/703-transformation-enterprise-repository-proto-v1.md`.

7) **Permission-trimmed retrieval**
- Permission trimming (authz) is enforced at query time (no post-hoc filtering); do not return paths/content the caller cannot read.
- Reference: `backlog/656-agentic-memory-v6.md`, `backlog/527-transformation-projection-v6.md`, `backlog/300-400/329-authz.md`.

8) **Reproducible reads**
- Every retrieval that influences decisions must include `read_context` plus citations sufficient to reproduce “what was read” (v6: `{repository_id, commit_sha, path[, anchor]}`).
- Reference: `backlog/534-mcp-protocol-adapter.md`, `backlog/656-agentic-memory-v6.md`.

## 1) Transition strategy (avoid “rename legacy protos”)

The current implementation substrate still contains **elements-first** concepts (nodes/elements/versions/baselines) and citations in `{element_id, version_id}` form.

v6 should not “reinterpret” those APIs into Git operations.

Instead:

- Introduce **new v6 seams** for “Enterprise Repository = Git tree” and “memory over Git”.
- Keep legacy surfaces as historical/compat-only until removed.
- Reference: `backlog/703-transformation-enterprise-repository-proto-v1.md`, `backlog/656-agentic-memory-v6.md`.

Legacy (pre-v6) repository docs exist and may still describe “graph/workspace tree” postures (nodes, smart folders, artifact lifecycles, etc.). They are **historical context only** and should not be used as requirements for this v6 Git-tree plan:

- `backlog/200-300/150-enterprise-repository-business-scenario.md`
- `backlog/200-300/151-enterprise-repository-functional.md`
- `backlog/200-300/154-enterprise-repository-implementation-plan.md`
- `backlog/200-300/220-repository-entity-model-ui.md` (and variants)
- `backlog/200-300/230-journey-01-draft-to-baseline.md` (and related 230 journeys)

## 2) Increments overview (E2E slices)

Each increment below lists:

- Product-visible outcome (what a user/agent can do)
- Required work by primitive (Proto / Domain / Projection / Process / Agent / UI)
- Backlog docs to cross-read/keep aligned for that increment
- Exit criteria (minimum verification)

### Increment 0 — v6 seam scaffolding (no user-facing change required)

**Outcome**
- New, explicit v6 seams exist (proto + service stubs) without breaking legacy.

**Proto**
- Add `io.ameide.transformation.enterprise_repository.v1` contract (read Git tree + governed Git writes).
- Optional: add `io.ameide.transformation.memory.v1` façade so MCP/UI aren’t forced to depend on legacy “knowledge” nouns.
- Reference: `backlog/703-transformation-enterprise-repository-proto-v1.md`, `backlog/496-eda-principles-v6.md`, `backlog/534-mcp-protocol-adapter.md`.

**Domain**
- Implement GitLab storage adapter capability (internal): Commit API batching, `last_commit_id` concurrency guards, MR IID usage, SHA-safe publish.
- Define and persist the **audit pointer set** (repo/project id, MR IID, pipeline id+status if applicable, resulting `main` commit SHA; optional tag).
- Reference: `backlog/701-repository-ui-enterprise-repository-v6.md`, `backlog/694-elements-gitlab-v6.md`.

**Projection**
- Implement Git read adapter capability (internal): list tree at `{ref → commit_sha}`, read content at `{commit_sha, path}`.
- Reference: `backlog/527-transformation-projection-v6.md`.

**UI / Agents / Process**
- No UI changes required; no workflows required; keep hidden behind flags.

**Exit criteria**
- All new proto packages compile and are callable in a dev environment (even if they return unimplemented for some methods).

**Cross-references (read together)**
- `backlog/496-eda-principles-v6.md`
- `backlog/694-elements-gitlab-v6.md`
- `backlog/701-repository-ui-enterprise-repository-v6.md`
- `backlog/703-transformation-enterprise-repository-proto-v1.md`

---

### Increment 1 — Read-only Enterprise Repository (Git tree browsing)

**Outcome**
- A user can browse the repository **as a file tree** at `published` baseline and open files.
- Every open/read returns `read_context.resolved.commit_sha` plus `{commit_sha, path[, anchor]}` citations.

**Proto**
- Projection: `ListTree(read_context, path)` + `GetContent(read_context, path)`.
- Reference: `backlog/703-transformation-enterprise-repository-proto-v1.md`, `backlog/534-mcp-protocol-adapter.md`.

**Domain**
- Implement `published` baseline resolution to **commit SHA** (e.g., `main` head).
- Tag support is optional in this increment; commit SHA support is mandatory.
- Reference: `backlog/694-elements-gitlab-v6.md`.

**Projection**
- Enforce permission trimming at query time; do not rely on client filtering.
- Return nodes with explicit `kind: FILE | DIRECTORY | GITLINK`.
- Reference: `backlog/527-transformation-projection-v6.md`, `backlog/656-agentic-memory-v6.md`, `backlog/300-400/329-authz.md`.

**UI**
- Repository screen: tree view (Radix-aligned), open/read panel for content, show the resolved commit SHA.
- Submodules: display as `GITLINK` node; show pinned SHA; do not “enter” as another repo automatically.
- Reference: `backlog/701-repository-ui-enterprise-repository-v6.md`.

**Agents/MCP (optional)**
- If exposed, `memory.get_context` can start as `paths[]` direct fetch (no semantic search yet).
- Reference: `backlog/534-mcp-protocol-adapter.md`.

**Exit criteria**
- Browse → open file works end-to-end for at least one tenant repo; citations include commit SHA + path.

**Cross-references**
- `backlog/701-repository-ui-enterprise-repository-v6.md`
- `backlog/694-elements-gitlab-v6.md`
- `backlog/656-agentic-memory-v6.md`
- `backlog/534-mcp-protocol-adapter.md`

---

### Increment 2 — Governed edits (proposal/change branch + MR; commit batching)

**Outcome**
- A user can start a change, edit files, rename/move files/folders, and see the change preview (MR-backed).
- No publishing yet; still “propose” only.

**Proto**
- Domain: `EnsureChange` and `CreateCommit(actions[], expected_last_commit_id, idempotency_key)`.
- Include `MOVE_PATH` and “folder move = prefix rename” semantics.
- Reference: `backlog/703-transformation-enterprise-repository-proto-v1.md`, `backlog/496-eda-principles-v6.md`.

**Domain**
- Implement idempotency policy (repeat request returns same resulting head SHA / MR).
- Enforce optimistic concurrency using `expected_last_commit_id` (lost-update prevention).
- Reference: `backlog/701-repository-ui-enterprise-repository-v6.md`.

**Projection**
- Support reading from `head` selector (change branch/MR ref) for preview browsing.
- Reference: `backlog/656-agentic-memory-v6.md` (read_context selectors).

**UI**
- “Start Change” mode.
- Editor save commits via Domain commit batching.
- Tree supports rename/move (including folder move).
- Reference: `backlog/701-repository-ui-enterprise-repository-v6.md`.

**Agents/MCP (optional)**
- Expose `memory.propose` to create/update a change and apply patch actions; still proposal-only.
- Reference: `backlog/536-mcp-write-optimizations.md`.

**Exit criteria**
- Edit → save produces commit on MR branch; move/rename works; concurrency conflict is surfaced deterministically.

**Cross-references**
- `backlog/701-repository-ui-enterprise-repository-v6.md`
- `backlog/703-transformation-enterprise-repository-proto-v1.md`
- `backlog/536-mcp-write-optimizations.md`

---

### Increment 3 — Approvals + publish (SHA-safe merge + CE-only gating)

**Outcome**
- Curators approve in the platform UI and publish by merging to `main`.
- Domain records durable audit pointers and advances “published baseline”.

**Proto**
- Domain: `PublishChange(change_id, expected_mr_head_sha)`.
- Response includes audit pointers (repo/project id, MR IID, pipeline id+status if applicable, resulting `main` commit SHA; optional tag).
- Reference: `backlog/701-repository-ui-enterprise-repository-v6.md`, `backlog/496-eda-principles-v6.md`.

**Domain**
- Governance truth is platform-owned; GitLab CE enforcement primitives only.
- Merge is SHA-safe: only merge the evaluated MR head SHA; stale head → refuse and re-evaluate.
- Reference: `backlog/694-elements-gitlab-v6.md`, `backlog/701-repository-ui-enterprise-repository-v6.md`.

**Projection**
- `published` selector resolves to the new `main` commit SHA.
- Index refresh may be eventual; correctness requires citations remain accurate.
- Reference: `backlog/656-agentic-memory-v6.md`.

**UI**
- Approvals UI: platform-authored approvals; show readiness and audit pointers after publish.
- Reference: `backlog/701-repository-ui-enterprise-repository-v6.md`.

**Process (optional in this increment)**
- May remain UI-driven; if implemented, publish emits a baseline-advanced fact.
- Reference: `backlog/527-transformation-e2e-sequence-v6.md`.

**Exit criteria**
- Propose → approve → publish updates `main`; browser at `published` shows new content; audit pointers exist.

**Cross-references**
- `backlog/496-eda-principles-v6.md`
- `backlog/694-elements-gitlab-v6.md`
- `backlog/695-gitlab-configuration-gitops.md`
- `backlog/701-repository-ui-enterprise-repository-v6.md`

---

### Increment 4 — Agentic memory (Git-citeable retrieval + derived backlinks)

**Outcome**
- `memory.get_context` returns a context bundle anchored to `published` commit SHA with Git citations.
- Backlinks/impact graph is available as a derived view (inline references only).

**Proto**
- Memory query surface (façade) returns `read_context.resolved.commit_sha` + item citations `{repository_id, commit_sha, path[, anchor]}`.
- Reference: `backlog/656-agentic-memory-implementation-mvp-1-minimal-end-to-end-v6.md`, `backlog/534-mcp-protocol-adapter.md`.

**Projection**
- Index Git content at the resolved commit SHA (minimum: keyword + path filters + inline reference extraction).
- Graph traversal is bounded and citeable.
- Reference: `backlog/535-mcp-read-optimizations.md`, `backlog/656-agentic-memory-v6.md`.

**Agents/MCP**
- MCP adapter exposes `memory.get_context` (read) and `memory.propose` (write to proposals).
- Enforce “cite only what was returned”.
- Reference: `backlog/534-mcp-protocol-adapter.md`, `backlog/536-mcp-write-optimizations.md`.

**UI**
- “Backlinks/Impact” panel for a file.
- “Copy citations” UX for reproducibility.
- Reference: `backlog/701-repository-ui-enterprise-repository-v6.md`.

**Exit criteria**
- “How do I run tests?” scenario works: retrieve citeable context at published SHA → propose patch against that SHA → publish → next retrieval returns new citeable truth.
- Reference scenario: `backlog/656-agentic-memory-implementation-mvp-v6.md`.

**Cross-references**
- `backlog/656-agentic-memory-v6.md`
- `backlog/656-agentic-memory-implementation-mvp-v6.md`
- `backlog/534-mcp-protocol-adapter.md`
- `backlog/535-mcp-read-optimizations.md`

---

### Increment 5 — ProcessDefinitions as Git artifacts (validate + deploy + evidence)

**Outcome**
- ProcessDefinitions authored as BPMN files in the Enterprise Repository can be validated/deployed to the selected runtime(s), with evidence tied to Git commits.

**Proto**
- Process catalog query surface for `processes/<module>/<process_key>/v<major>/process.bpmn`.
- Define `process_key` identity = BPMN `<process id>`.
- Reference: `backlog/701-repository-ui-enterprise-repository-v6.md`, `backlog/520-primitives-stack-v6.md`.

**Process primitive**
- Implement validate/deploy workflows (runtime profiles: Zeebe/Camunda 8 and Flowable).
- Reference: `backlog/520-primitives-stack-v6.md`, `backlog/527-transformation-e2e-sequence-v6.md`.

**Domain**
- Publish emits baseline-advanced signal/fact; process reacts via hybrid messaging (RPC and/or events).
- Reference: `backlog/496-eda-principles-v6.md`.

**Projection**
- Index ProcessDefinitions from Git and expose deploy/validation status views derived from evidence and audit pointers.
- Reference: `backlog/527-transformation-projection-v6.md`.

**UI**
- Process catalog view + per-process deploy status by baseline.
- Reference: `backlog/701-repository-ui-enterprise-repository-v6.md`.

**Exit criteria**
- Publishing a BPMN change triggers validate+deploy; evidence is linked to the published commit SHA.

**Cross-references**
- `backlog/520-primitives-stack-v6.md`
- `backlog/527-transformation-e2e-sequence-v6.md`
- `backlog/701-repository-ui-enterprise-repository-v6.md`
- `backlog/511-process-primitive-scaffolding-v3.md`

---

### Increment 6 — Operational hardening (rebuildability, observability, evaluation)

**Outcome**
- The system is operable and debuggable: indexing is incremental, retrieval is traceable, and the memory loop has regression coverage.

**Projection**
- Incremental indexing keyed by commit SHA; rebuild-from-scratch is a supported recovery path.
- Retrieval trace logs and explicit “why included” fields.
- Reference: `backlog/656-agentic-memory-implementation-mvp-2-operational-end-to-end-v6.md`.

**Agents**
- Golden queries + deterministic diff views.
- Reference: `backlog/656-agentic-memory-implementation-mvp-2-operational-end-to-end-v6.md`.

**UI**
- Baseline diff view (Git-native) with citeable before/after anchors.
- Reference: `backlog/701-repository-ui-enterprise-repository-v6.md`.

**Exit criteria**
- Repeatable “golden suite” for memory; deterministic diffs; rebuild works without changing canonical truth.

**Cross-references**
- `backlog/656-agentic-memory-implementation-mvp-2-operational-end-to-end-v6.md`
- `backlog/535-mcp-read-optimizations.md`

## 3) GitOps/config dependency note (do not edit GitOps here)

GitLab/runner/branch-protection configuration is owned by GitOps.

- Treat `gitops/ameide-gitops/` as a pinned dependency in this repo.
- Reference posture: `backlog/695-gitlab-configuration-gitops.md` (and the GitOps repo it points to).

## 4) “Definition of usable” (what counts as done per increment)

An increment is considered shipped only if:

- there is a user-visible loop (browse/read, propose, approve/publish, retrieve citeable context),
- the loop is reproducible (read_context + citations) and CE-compatible,
- and scope identity `{tenant_id, organization_id, repository_id}` is carried end-to-end.
