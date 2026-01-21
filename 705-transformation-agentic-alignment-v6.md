---
title: "705 — Transformation Agentic Alignment (v6: coding + architect agents + memory + processes)"
status: draft
owners:
  - transformation
  - platform
  - platform-devx
created: 2026-01-20
related:
  - 496-eda-principles-v6.md
  - 520-primitives-stack-v6.md
  - 527-transformation-v6-index.md
  - 706-transformation-v6-coordinated-implementation-plan.md
  - 527-transformation-e2e-sequence-v6.md
  - 527-transformation-domain-v3.md
  - 527-transformation-process-v4.md
  - 527-transformation-projection-v6.md
  - 527-transformation-proto-v6.md
  - 527-transformation-agent.md
  - 717-ameide-agents-v6.md
  - 657-transformation-domain-clean-target-v2.md
  - 694-elements-gitlab-v6.md
  - 695-gitlab-configuration-gitops.md
  - 701-repository-ui-enterprise-repository-v6.md
  - 703-transformation-enterprise-repository-proto-v1.md
  - 704-v6-enterprise-repository-memory-e2e-implementation-plan.md
  - 534-mcp-protocol-adapter.md
  - 535-mcp-read-optimizations.md
  - 536-mcp-write-optimizations.md
  - 656-agentic-memory-v6.md
  - 656-agentic-memory-implementation-v6.md
  - 656-agentic-memory-implementation-mvp-v6.md
  - 650-agentic-coding-overview-v6.md
  - 651-agentic-coding-ameide-coding-agent.md
  - 653-agentic-coding-test-automation-coder.md
  - 654-agentic-coding-cli-surface-coder.md
---

# 705 — Transformation Agentic Alignment (v6: coding + architect agents + memory + processes)

This backlog is an **alignment map**: it connects the “agentic coding” suite (DevX) to the Transformation v6 posture (Git-backed Enterprise Repository + governed workflows) and clarifies how the documents relate.

It does **not** replace the existing v6 indices; it prevents cross-suite drift by making boundaries and seams explicit.

## 0) The three stacks (don’t mix them)

### A) Agentic coding (DevX) — platform codebase automation

- Purpose: change the Ameide codebase via Coder workspaces/tasks, open GitHub PRs, run `ameide test`, validate via preview envs.
- Canonical doc: `backlog/650-agentic-coding-overview-v6.md` (plus `651–655`).

### B) Transformation (capability) — tenant change workflows

- Purpose: orchestrate tenant change work using the Transformation Domain/Process/Projection primitives and governed publish.
- Canonical v6 index: `backlog/527-transformation-v6-index.md` (Git-backed repository substrate).

### C) Enterprise Repository + Memory — Git-backed organizational knowledge

- Purpose: treat tenant “Enterprise Repository” as canonical Git files (GitLab), browse as Git tree, and provide projection-owned retrieval (“memory”) with citations.
- Canonical docs: `backlog/694-elements-gitlab-v6.md`, `backlog/701-repository-ui-enterprise-repository-v6.md`, `backlog/656-agentic-memory-v6.md`.

All three stacks share the safety invariant “proposal → review → publish”, but they do **not** share storage backends or identifiers.

## 1) Current v6 document map (what to read; what each doc owns)

### 1.1 Shared v6 principles (global)

- **Integration + ownership rules (contract-first, owner-only writes):** `backlog/496-eda-principles-v6.md`
- **Primitive taxonomy and boundaries:** `backlog/520-primitives-stack-v6.md`

### 1.2 Transformation v6 (canonical “tenant change” story)

- **v6 index (what is current):** `backlog/527-transformation-v6-index.md`
- **E2E story (Git-backed repo + request→wait→resume):** `backlog/527-transformation-e2e-sequence-v6.md`
- **Domain posture (owner):** `backlog/527-transformation-domain-v3.md`
- **Process semantics:** `backlog/527-transformation-process-v4.md`
- **Projection posture:** `backlog/527-transformation-projection-v6.md`
- **Contract notes (what protos *mean* in v6):** `backlog/527-transformation-proto-v6.md`
- **Agent primitive roles + governance semantics:** `backlog/527-transformation-agent.md`
- **Clean-target framing (Git-backed substrate + hybrid EDA; memory model TBD):** `backlog/657-transformation-domain-clean-target-v2.md`

### 1.3 Enterprise Repository + UI (Git tree, CE-only)

- **Git-backed Enterprise Repository decisions:** `backlog/694-elements-gitlab-v6.md`
- **GitLab config is GitOps-owned:** `backlog/695-gitlab-configuration-gitops.md`
- **Repository UI contract (Git tree, MR-based, CE-only):** `backlog/701-repository-ui-enterprise-repository-v6.md`
- **New proto seam proposal (Git tree browse + governed Git writes):** `backlog/703-transformation-enterprise-repository-proto-v1.md`
- **E2E implementation increments (cross-primitive plan):** `backlog/704-v6-enterprise-repository-memory-e2e-implementation-plan.md`

### 1.4 Agentic memory (projection-owned retrieval + citations)

- **Memory contract (Git-first posture; model TBD):** `backlog/656-agentic-memory-v6.md`
- **Implementation mapping (what can ship before final model):** `backlog/656-agentic-memory-implementation-v6.md`
- **Increment plan (scenario-based):** `backlog/656-agentic-memory-implementation-mvp-v6.md`

### 1.5 MCP adapter (agentic access protocol binding; optional transport)

- **MCP adapter pattern (proto-first tool surfaces):** `backlog/534-mcp-protocol-adapter.md`
- **Read optimizations (semantic retrieval; projection-owned):** `backlog/535-mcp-read-optimizations.md`
- **Write optimizations (proposal-first writes):** `backlog/536-mcp-write-optimizations.md`

### 1.6 Agentic coding (DevX execution substrate; reused by “Executor” role)

- **DevX model (Coder workspaces + tasks; CLI front doors):** `backlog/650-agentic-coding-overview-v6.md`
- **Coding agent runtime/orchestration (tasks):** `backlog/651-agentic-coding-ameide-coding-agent.md`
- **Test automation model:** `backlog/653-agentic-coding-test-automation-coder.md`
- **CLI surface:** `backlog/654-agentic-coding-cli-surface-coder.md`

## 2) How the pieces relate (the intended v6 shape)

### 2.1 “Executor” is an execution substrate, not a writer

In `backlog/527-transformation-agent.md`, the “Executor” role is the entity that produces PRs/patches + evidence.

Alignment rule:

- Executor (and any “coding agent”) uses the **DevX execution substrate** (Coder tasks + CLI front doors) to do work and produce evidence.
- Canonical writes still go through the **owning Domain** (owner-only writes), and “publish” still advances a governed baseline in the Enterprise Repository.

Note (naming):
- Role naming is being standardized in `backlog/717-ameide-agents-v6.md`. Treat “Product Owner / Solution Architect / Executor” as legacy labels when reading older backlogs.

### 2.2 “Solution Architect” agent ≠ “coding agent”

The “Solution Architect” role in `backlog/527-transformation-agent.md` is proposal/plan-oriented:

- reads via projections,
- produces reviewable plans/proposals,
- does not execute repo writes directly.

The coding agent is the constrained executor that runs tasks and produces PR/evidence.

### 2.3 Memory is a projection concern over Git

Memory (`backlog/656-agentic-memory-v6.md`) is not a second canonical store:

- Canonical truth: Git files in the Enterprise Repository (`backlog/694-elements-gitlab-v6.md`)
- Retrieval truth: projection-owned indexes + citeable `read_context` anchored to commit SHA (`backlog/701-repository-ui-enterprise-repository-v6.md`)
- Transport for external agents/devtools: MCP adapter (`backlog/534-mcp-protocol-adapter.md`)

### 2.4 Repository UI is Git-tree-first; graph/backlinks are derived

UI browsing must be Git tree at `read_context` (`backlog/701-repository-ui-enterprise-repository-v6.md`), while relationship graphs are derived from inline references (no relationship CRUD).

## 3) Alignment work to do (concrete next steps)

1) **Unify “work execution” language across 527 and 650**
- Keep the v6 invariant: Domain issues durable work identity; Process orchestrates; Executor runs in a constrained substrate; outcomes return as evidence.
- For “how it runs”, treat Coder tasks as the default execution substrate (per `650/651`) instead of KEDA-scaled Jobs (historical posture).
- References: `backlog/527-transformation-agent.md`, `backlog/651-agentic-coding-ameide-coding-agent.md`, `backlog/496-eda-principles-v6.md`.

2) **Adopt the Git-tree seam for Enterprise Repository UX**
- Do not repurpose legacy workspace-node APIs for repo browsing.
- Implement the new seam proposed in `backlog/703-transformation-enterprise-repository-proto-v1.md` and ship via the increments in `backlog/704-v6-enterprise-repository-memory-e2e-implementation-plan.md`.

3) **Lock a minimal v6 citation contract**
- For memory/read_context, converge on `{repository_id, commit_sha, path[, anchor]}` as the minimum audit-grade citation.
- Keep the memory “model” (IDs beyond path/commit) explicitly TBD until decided.
- References: `backlog/656-agentic-memory-v6.md`, `backlog/534-mcp-protocol-adapter.md`.

4) **Use a single increment ladder**
- Keep detailed delivery plans (704/668/656/650) but align them to `backlog/706-transformation-v6-coordinated-implementation-plan.md`.

## 4) Definition of done (for alignment)

Alignment is “done” when:

- the doc set clearly separates DevX agentic coding vs Transformation Enterprise Repository,
- Transformation agents’ roles map cleanly onto an execution substrate (Coder tasks) without violating owner-only writes,
- Repository UI and Memory are Git-backed and citeable by commit SHA + path,
- and the v6 increments are the “single plan” for shipping the end-to-end loop.
