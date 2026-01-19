---
title: "651 – Agentic coding — Ameide coding agent (Coder tasks + Camunda; replaces 527 KEDA pipeline)"
status: draft
owners:
  - platform-devx
  - platform
  - transformation
created: 2026-01-12
suite: "650-agentic-coding"
source_of_truth: false
---

# 651 – Agentic coding — Ameide coding agent (Coder tasks + Camunda; replaces 527 KEDA pipeline)

## 0) Purpose

Specify the “agentic coding” automation runtime and orchestration:

- Replace the **527 KEDA/Kafka scaled-job executor pipeline** with **Coder-backed tasks** (ephemeral workspace executions).
- Replace “custom Temporal compiler / execution glue” with a **Camunda-first orchestration model** for agent runs (BPMN as the automation backbone).

This document inherits all decisions from `backlog/650-agentic-coding-overview.md`.

## 0.1 Primary UX goal: “agents don’t learn the repo”

The agent must not need to understand:

- which test runner applies to which package
- how to classify tests across languages
- which vendor CLIs to install/invoke
- how to avoid leaving state behind

The system encodes those rules in:

- the Ameide CLI (test front doors and orchestration)
- agent profile templates + `AGENTS.md` (guardrails and “what is allowed”)

## 1) Decisions (inherited from 650; repeated for local clarity)

- Execution is **internal-first** (external executors exist by exception; see 655).
- Telepresence is **not supported**.
- Dev environment contract is `.devcontainer/coder/devcontainer.json`.
- Automation runs as ephemeral, workspace-backed executions (“tasks”).

## 2) Scope (in/out)

### 2.1 In scope

- Standardize agent automation execution on Coder-backed tasks.
- Standardize orchestration of agent runs on Camunda BPMN workflows.
- Replace the current “KEDA scaled executor Jobs” pattern for agent execution as the default.

### 2.2 Out of scope

- Human interactive workspace UX (see 652).
- Preview environment deployment mechanics (see 653).

## 3) Terminology (additions to 650)

- **Orchestration plane:** Camunda workflows that schedule and supervise runs.
- **Execution plane:** Coder-backed task workspaces that run commands.
- **Task launcher:** component that creates/runs/destroys task workspaces and returns evidence.
- **Agent profile template:** a Coder template specialized for an agent type (code/gitops/sre) and its guardrails.

## 4) Architecture (as-is → to-be)

### 4.1 As-is (what we are deleting)

#### 4.1.1 KEDA executor pipeline (527)

Current posture (to be removed as the default automation substrate):

- Domain emits work events/intents.
- KEDA scales Kubernetes Jobs based on queue depth.
- Executor-image Jobs run agent/tooling and report outcomes.

#### 4.1.2 Custom workflow/runtime glue

Current posture (to be replaced in this initiative):

- Workflow orchestration and/or “compiler” responsibilities live in custom code paths.
- The orchestration surface is not standardized on a single “process runtime” for automation.

### 4.2 To-be (target architecture)

#### 4.2.1 Two planes

1. **Orchestration plane (Camunda):**
   - Owns run lifecycles, retries, concurrency limits, evidence capture state machine.
   - Encodes agent workflows as BPMN processes (e.g., “Implement change → run checks → open PR → request review”).

2. **Execution plane (Coder tasks):**
   - Runs the actual code-changing commands inside ephemeral workspace instances.
   - Uses the same devcontainer contract and toolchain as human workspaces.

#### 4.2.2 “Coder task” definition (implementation-independent)

An automation task is defined by:

- **Template:** which workspace template to use (and therefore which devcontainer contract).
- **Repo checkout:** repo URL + ref + clone strategy.
- **Command:** a single non-interactive command to execute (script entrypoint).
- **Evidence:** logs + artifacts (JUnit, Playwright reports, diffs, PR URL).
- **Timeout + cleanup:** hard TTL; always stop/destroy on completion.

#### 4.2.3 Credential model (default)

- Git identity uses a **bot identity** (GitHub App/token) scoped to:
  - create branches
  - push
  - open PRs
- Cluster identity inside the task is minimal:
  - no cluster-admin
  - no secret enumeration beyond what the task explicitly needs
- LLM provider credentials (if any) are injected as runtime secrets, never embedded in templates.

#### 4.2.4 Agent profiles (templates) and guardrails

We define templates by **agent type**, not by “tech stack”.

Minimum set:

1. **Code agent** (default)
   - primary repo: `ameide`
   - allowed edits: `services/`, `packages/`, `primitives/` (policy-defined)
   - forbidden edits by default: `gitops/`, submodule pointers, environment promotion artifacts

2. **GitOps agent**
   - primary repo: `ameide-gitops` (checked out as the only repo)
   - allowed edits: GitOps values/overlays/apps and promotion PRs
   - forbidden edits: application code and platform source repos

3. **SRE agent**
   - purpose: diagnostics/triage; may be read-heavy
   - write guardrails: stricter than code agent (often “no PR creation” unless explicitly enabled)

Guardrails must be enforced by multiple layers (defense in depth):

- Repo selection (what is even present in the workspace)
- Token scopes (GitHub bot credentials differ per profile)
- Cluster RBAC (namespace-scoped; profile-specific)
- CLI preflight checks (fail fast on forbidden diffs / missing evidence)
- `AGENTS.md` instructions (explicit do/don’t; required commands)

#### 4.2.5 Template-scoped `AGENTS.md` (required)

Each agent profile template must include an `AGENTS.md` scoped to the repo checkout it governs.

Canonical layout:

- `/workspaces/<profile>/AGENTS.md`
- `/workspaces/<profile>/repo/` (the checkout the agent edits)

The task launcher must set the agent’s working directory under that subtree so the correct instructions apply.

## 5) Implementation plan (no migration constraints; destructive is allowed)

### 5.1 Phase A — establish the task runtime contract

- Define a single “task entrypoint” contract (stdout/stderr, exit codes, evidence paths).
- Provide a stable runner command for tasks (example shape):
  - `ameide agent task run <task-kind>`

### 5.2 Phase B — implement “task launcher” service/worker

- Implement a launcher that can:
  - create a workspace instance (ephemeral)
  - wait for agent readiness
  - run the command
  - collect evidence
  - destroy the workspace
- Back the launcher with Camunda worker(s) so BPMN can schedule tasks.

### 5.3 Phase C — replace 527 execution substrate

- Stop creating/scaling KEDA executor Jobs for agent work.
- Route “agent work” execution requests to the Camunda process that launches Coder tasks.

### 5.4 Phase D — decompose agent workflows

Deliver at least these BPMN workflows:

- **Preflight:** lint/unit/typecheck in a clean task.
- **Patch:** run agent to implement a change in a task and open a PR.
- **Triage:** if preview E2E fails, run diagnostics task and propose a patch.

### 5.5 Documentation updates (required outputs)

- Update/retire `backlog/527-*.md` references for “agent work execution substrate”.
- Update `backlog/621-ameide-cli-inner-loop-test.md` to remove Telepresence assumptions (or replace with 653).
- Create a concise “how to run a task locally” section in `backlog/652-agentic-coding-dev-workspace-coder.md`.
- Add profile-specific `AGENTS.md` templates and ensure Coder templates reference them (see 650).

## 6) Testing

- Unit tests for the launcher/task-runner components.
- A single “task E2E” that:
  - launches an ephemeral task
  - runs a trivial command in the repo
  - collects an evidence bundle
  - verifies cleanup (no workspace leakage)

## 7) Risks / guardrails

- **Cost leakage:** tasks must have TTLs and enforced cleanup.
- **Privilege creep:** tasks must not default to broad cluster RBAC.
- **Orchestration sprawl:** Camunda BPMN is the single orchestration model for agent automation in this suite (no parallel orchestration engines).

## 8) Deprecations / replacements

- Replace the 527 KEDA/Kafka scaled executor Jobs pattern as the default execution substrate for code-changing automation.
- Replace “custom workflow/runtime glue” with Camunda-first orchestration for agent runs in this suite.

## 9) References

- Normative decisions: `backlog/650-agentic-coding-overview.md`
- Related (to be superseded as defaults): `backlog/527-transformation-agent.md`
