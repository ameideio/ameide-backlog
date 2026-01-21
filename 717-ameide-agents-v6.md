---
title: "717 — Ameide Agents (v6: Agent primitives, LangGraph DAGs, human-in-loop, Codex + memory)"
status: draft
owners:
  - agents
  - platform
  - transformation
  - platform-devx
created: 2026-01-21
updated: 2026-01-21
related:
  - 310-agent-runtime.md
  - 310-agents-v2.md
  - 500-agent-operator.md
  - 504-agent-vertical-slice.md
  - 505-agent-developer-v2.md
  - 507-scrum-agent-map.md
  - 520-primitives-stack-v6.md
  - 527-transformation-agent.md
  - 534-mcp-protocol-adapter.md
  - 537-primitive-testing-discipline.md
  - 650-agentic-coding-overview-v6.md
  - 651-agentic-coding-ameide-coding-agent.md
  - 654-agentic-coding-cli-surface-coder.md
  - 656-agentic-memory-v6.md
  - 718-codex-cli-web-search-internals.md
  - 701-repository-ui-enterprise-repository-v6.md
  - 705-transformation-agentic-alignment-v6.md
  - 714-togaf-scenarios-v6.md
  - 714-togaf-scenarios-v6-02.md
---

# 717 — Ameide Agents (v6)

Define the v6 **agent taxonomy** and the non-negotiable platform rules for how agents:

- are built (**Agent primitives**, LangGraph DAGs),
- access the platform (Projection/Domain tools, memory/citations),
- perform work (Codex CLI for research and development; verification via `ameide` front doors),
- remain **human-in-the-loop** and **owner-only-write** aligned.

This doc exists to prevent naming drift across the v6 stacks (Transformation, Enterprise Repository, DevX agentic coding).

## 0) Non-negotiables

1) **All agents are Agent primitives.**
- Every Ameide agent is implemented as an **Agent primitive** (per `backlog/520-primitives-stack-v6.md`) and managed by the agent control plane/operator (`backlog/500-agent-operator.md`).

2) **All agents are LangGraph-based.**
- The agent runtime is a LangGraph DAG that orchestrates tools (see the LangGraph sections in `backlog/505-agent-developer-v2.md` and agent testing guidance in `backlog/537-primitive-testing-discipline.md`).

3) **Human-in-the-loop is the default.**
- Agents produce reviewable proposals + evidence; publication is a governed action.
- “Human-in-loop” is enforced by:
  - tool grants + risk tiers (AgentDefinition),
  - owner-only writes (Domain owns canonical writes),
  - and workflow gates (Process/UISurface).

4) **Owner-only writes remains mandatory.**
- Agents and UIs do not bypass owner boundaries (`backlog/496-eda-principles-v6.md`, `backlog/701-repository-ui-enterprise-repository-v6.md`).

5) **Reproducible reads are mandatory.**
- Retrieval must be anchored to `read_context` and citations:
  - Git-backed citations: `{repository_id, commit_sha, path[, anchor]}` (`backlog/656-agentic-memory-v6.md`, `backlog/714-togaf-scenarios-v6-02.md`).

## 1) Role taxonomy (v6; naming standard)

This doc standardizes role naming and deprecates older “PO/SA/Executor” naming in agent backlogs.

### 1.1 The four roles

| Role | Primary intent | Reads | Writes | Default tooling |
|------|----------------|------|--------|-----------------|
| **AmeideEnterpriseArchitect** | enterprise posture, governance constraints, standards alignment | Projection/Memory | ❌ (proposal-only) | Codex CLI (research + verification), Projection tools |
| **AmeideSolutionArchitect** | technical plan + decomposition + delegation | Projection/Memory | ❌ (proposal-only) | Codex CLI (research + verification), Projection tools |
| **AmeideTechnicalArchitect** | implementation constraints, guardrails, verification plan | Projection/Memory | ❌ (proposal-only) | Codex CLI (verification), Projection tools |
| **AmeideDeveloper** | implement changes and produce evidence | Projection/Memory + repo checkout | ✅ (constrained) | Codex CLI (development), `ameide test` |

Notes:
- “Writes” here means canonical repo writes / PR creation / publication. Architects may draft plans and request changes, but do not directly mutate canonical repos by default.
- A “developer” is still governed: all publishing is gated; evidence is required.

### 1.2 Deprecations (terminology only)

Deprecated naming (do not use for new work):
- **Product Owner (AmeidePO)**
- **Solution Architect (AmeideSA)** (as a name; its responsibilities map to the new architect roles)
- **Executor / Coder (AmeideCoder)** (as a name; responsibilities map to AmeideDeveloper)
- “Supervisor/coder” shorthand

Migration guidance:
- When you encounter older docs using PO/SA/Executor naming, treat them as historical role labels and map them onto the roles above.

## 2) Tooling and access model

### 2.1 Memory and facts access (Projection-owned)

Architects and developers access “memory” through Projection query tools:
- Memory is **projection-owned** (derived, rebuildable) and returns citation-grade outputs (`backlog/656-agentic-memory-v6.md`).
- “Facts” (domain facts/process facts) remain the audit trail; memory/projection reads are derived views over canonical stores (`backlog/496-eda-principles-v6.md`).

Transport:
- In-platform agents should prefer SDK clients; external/devtool compatibility can be provided via MCP protocol adapters (`backlog/534-mcp-protocol-adapter.md`).

### 2.2 Codex CLI usage (two modes)

Codex CLI is used in two distinct modes:

1) **Research + verification (Architect roles)**
- Purpose: resolve open questions, validate against vendor documentation / industry standards, and run verification commands.
- Output: a reviewable plan/proposal and evidence/citations (including external citations when applicable).
- If web search is required for the task, enable it explicitly; see `backlog/718-codex-cli-web-search-internals.md`.

2) **Development execution (AmeideDeveloper)**
- Purpose: implement changes in a repo checkout and produce PR + evidence.
- Verification must use the CLI front doors (`backlog/654-agentic-coding-cli-surface-coder.md`), with `./ameide test` as the default end-to-end check.

## 3) Supervision cadence (architect oversight of developer execution)

We standardize a supervision loop:

- An Ameide architect agent (Enterprise/Solution/Technical) oversees the AmeideDeveloper’s active work and periodically verifies it.
- “AmeideCoder” / “coder” references in older docs map to **AmeideDeveloper** (`§1.2`).

### 3.1 Purpose

- Catch drift early (wrong direction, missing evidence, policy violations).
- Keep humans in the loop by surfacing checkpoints regularly, not only at the end.

### 3.2 Mechanism (normative shape)

- The architect agent runs a **review tick** every `review_interval_minutes` (configurable).
- Each tick reviews the developer’s current outputs:
  - diff/patch state (what changed, where),
  - verification evidence (latest `./ameide test` run + failures),
  - citations and scope alignment where applicable (Transformation/Enterprise Repository work),
  - guardrails (forbidden paths, missing required evidence).
- The tick produces one of:
  - **continue** (no action required),
  - **adjust** (send constraints / plan correction to the developer agent),
  - **stop + escalate** (request human decision when risk tier/policy requires it).

Implementation note:
- The cadence should be implemented as a Process timer/schedule or an agent-side loop, but it must remain observable (facts/evidence) and configurable via AgentDefinition config.

## 3) v6 stack alignment (don’t mix concerns)

This role taxonomy applies across the three stacks described in `backlog/705-transformation-agentic-alignment-v6.md`:

- **Transformation (tenant change)**: architects reason over Enterprise Repository + projections; governed publish is through the owning Domain.
- **Enterprise Repository + Memory**: Git-backed canonical files; derived backlinks/impact/memory are projection-owned and citeable.
- **DevX agentic coding (platform codebase)**: developers implement changes in the Ameide codebase and return PR + evidence.

## 4) Scenario mapping (TOGAF-ish)

The “architect” user in Scenario Slice B (“Requirements + derived backlinks/impact + citeable context”) maps to the architect roles here:
- Scenario B reference: `backlog/714-togaf-scenarios-v6-02.md`.
- Requirement: derived views must explain “why linked?” via origin citations `{repo, sha, path, anchor}`.

## 5) Definition of done (for this doc)

This taxonomy is “adopted” when:
- new agent work uses the architect/developer role names above,
- older docs cross-reference this mapping and stop introducing new PO/SA/Executor terminology,
- Agent primitive implementations and tests follow LangGraph invariants per `backlog/537-primitive-testing-discipline.md`.
