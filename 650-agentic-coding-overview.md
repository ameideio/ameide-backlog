---
title: "650 – Agentic coding (internal-first, Coder-orchestrated) — overview"
status: active
owners:
  - platform-devx
  - platform
created: 2026-01-12
suite: "650-agentic-coding"
source_of_truth: true
---

# 650 – Agentic coding (internal-first, Coder-orchestrated) — overview

## 0) Purpose

Define a single, consistent, internal-first model for “agentic coding” at Ameide:

- Humans develop in **Coder workspaces** (browser VS Code via code-server).
- Automated code-changing runs as **ephemeral, workspace-backed executions** (“Coder tasks” in this doc suite).
- Deployed-system truth is validated via **Argo CD preview environments** + **Playwright** against real ingress URLs.

This document is the **single source of truth** for definitions and decisions. The rest of the 650 suite inherits decisions from here and must not fork them.

## 0.1 Primary UX goal: reduce cognitive load (agents first)

This suite treats the Ameide CLI as the **cognitive-load absorber** for a complex monorepo with multi-vendor tooling (Go/Node/Python/Playwright/Kubernetes).

The intended agent instruction stays minimal:

- “Run the one CLI command; if it passes, you’re done; if it fails, fix the first actionable error and rerun.”

## 1) Decisions (normative; no forks in sub-docs)

### 1.1 In-cluster orchestration; internal-first execution

Default: development and automation executes **inside the cluster**.

- No “remote-from-laptop into cluster network” substrate is required for correctness.
- CI orchestration can run anywhere, but code/test execution is in-cluster (workspaces, tasks, preview env jobs/runners).

Exception (explicit): some domains have vendor-locked execution substrates (e.g., Dynamics 365 Finance & Operations development in a Windows dev VM + Visual Studio tooling).

In those cases:

- Orchestration still runs in-cluster (Coder workspace/task).
- Execution may run on an external executor substrate, but only behind a strict tool contract and profile guardrails.
- The CLI remains the front door; agents still run the same `ameide` commands.

### 1.2 No Telepresence (explicitly out of platform support)

Telepresence is **not part of the platform model** and is not a supported dependency for:

- Coder workspaces
- automation tasks
- platform test automation

Rationale: Telepresence’s VIF/TUN model generally requires elevated networking privileges that are incompatible with our workspace security posture.

### 1.3 Coder CE only (browser-first, not browser-only enforced)

We adopt **Coder Community Edition**.

- Supported UX is browser-first (code-server + web terminal).
- We do not depend on paid “browser-only enforcement” features.

### 1.4 Dev environment contract is Kubernetes-safe devcontainers

For in-cluster workspaces and task workspaces, the contract for a “developer machine” is:

- `.devcontainer/coder/devcontainer.json`

Implications:

- The dev environment is built with **envbuilder** (no Docker daemon in workspaces).
- Workspace pods remain unprivileged (no `/var/run/docker.sock`, no `CAP_NET_ADMIN`, no `/dev/net/tun` assumptions).

### 1.5 Two loops, two truths

We explicitly separate loops to avoid “mixed concerns” testing:

1. **Inner loop (seconds):** run a process in the workspace; hot reload is filesystem-based; run fast checks locally.
2. **Outer loop (minutes):** build artifacts + deploy a preview environment; run E2E against the preview ingress URL.

### 1.6 Workspaces for humans, tasks for automation

- **Workspace:** long-lived (hours), interactive, human-driven.
- **Task:** ephemeral (minutes), non-interactive by default, automation-driven, cleans itself up.

“Tasks” are implemented using Coder primitives (workspace/template/agent) and our own orchestration; we do not require a vendor “Tasks” feature.

### 1.6.1 Background work substrate is EDA v2 (Knative Broker/Trigger)

When the platform needs long-running, asynchronous execution (e.g., Phase 3 cluster E2E, heavy verify runs, evidence-producing work), the standard substrate is **EDA v2** (`backlog/496-eda-principles-v2.md`):

- Inter-primitive messaging uses CloudEvents (protobuf binary mode) routed through Knative Broker/Triggers.
- Long-running execution is triggered via a single standard: **Executor Service as the Trigger subscriber** (see `backlog/658-eda-v2-reference-implementation-refactor.md`).
- The executor ACKs quickly (2xx after durable acceptance), runs work asynchronously internally (Jobs/Kueue/etc.), and reports outcomes back to the owning Domain (facts after commit via outbox).

This is separate from Coder “tasks”: Coder tasks are primarily for **code-changing automation** (PRs, repo checks) in a dev workspace contract.

### 1.7 Templates are agent profiles (not technology choices)

We standardize Coder templates around **agent types** (profiles) rather than “technology stacks”.

Examples of first-class profiles:

- **dev/code**: repository code changes (services/packages), fast tests, PRs
- **gitops**: GitOps changes (Argo CD/apps/values), promotion workflows, minimal app code exposure
- **sre**: diagnostics/triage tooling, stricter write guardrails
- **external-executor** (vendor-locked): orchestration in-cluster, execution on a dedicated external substrate

Each profile is expected to carry different guardrails (repo access, secrets, RBAC, allowed edits).

### 1.7 Secrets and templates

Coder templates and template source repos are **not a secret boundary**.

- Templates reference secret *names* and runtime identities.
- Secrets live in: Keycloak (SSO), Kubernetes Secrets/ExternalSecrets, or a future external secret store integration.

### 1.8 Template-scoped agent instructions via `AGENTS.md`

Every Coder template/profile must include an `AGENTS.md` whose scope covers the repo checkout for that profile.

Canonical layout (example):

- `/workspaces/<profile>/AGENTS.md`
- `/workspaces/<profile>/repo/` (cloned target repo)

This makes the instructions:

- explicit per agent type
- enforced by scope (agents working under that subtree inherit the profile rules)

## 2) Scope (in/out)

### 2.1 In scope

- Coder workspaces for human development (dev cluster).
- Coder-backed task executions for automation (agent runs, preflight checks).
- Preview environment E2E (Argo CD) as merge gate truth.
- A single CLI “front door” for the agent inner loop that works in a Coder workspace.
- Vendor-locked external executors that are orchestrated from in-cluster workspaces/tasks (see 655).

### 2.2 Out of scope

- Telepresence-based workflows as platform requirements.
- Docker-in-Docker / privileged container development.
- “Run the full deployed system locally” as a required developer path.

## 3) Terminology (authoritative)

- **Coder control plane:** the Coder server (`coderd`) and its supporting services.
- **Coder template:** Terraform-based definition of workspace resources + agent startup.
- **Workspace:** a Coder instance for a human (stateful, interactive).
- **Task:** an ephemeral automation execution implemented as a Coder workspace instance run-to-completion.
- **Preview environment:** per-PR Argo CD environment with real ingress hostnames.
- **Inner loop:** dev server + fast tests in the workspace.
- **Outer loop:** build + deploy + Playwright against preview URL.
- **Agent profile (template):** a Coder template specialized for an agent type and its guardrails.
- **Template instructions (`AGENTS.md`):** scoped rules that define what an agent may do in that profile.
- **External executor:** a vendor-locked execution substrate controlled via a strict tool contract (e.g., a Windows dev VM service), orchestrated by an in-cluster agent.

## 4) Architecture (conceptual)

### 4.1 Human path (workspace)

1. Human opens Coder UI → workspace is provisioned via template.
2. Repo is cloned in the workspace.
3. Human edits in code-server and runs:
   - unit/lint/typecheck
   - dev server (hot reload)
4. Human pushes branch + opens PR.

### 4.2 Automation path (task)

1. Orchestrator creates an ephemeral workspace instance using the same template/contract.
2. It runs a single command (e.g., “run checks”, “apply codex patch”, “open PR”) and captures evidence.
3. It stops/destroys the workspace instance on completion.

### 4.3 Merge gate (preview env)

1. PR triggers build(s) and deploys a preview environment.
2. Playwright runs against the preview ingress URL(s).
3. Merge gate uses preview E2E as the fidelity signal.

## 5) Implementation plan (this suite)

Deliverables (the 650 suite):

- `backlog/650-agentic-coding-overview.md` (this file; normative)
- `backlog/651-agentic-coding-ameide-coding-agent.md`
- `backlog/652-agentic-coding-dev-workspace.md`
- `backlog/653-agentic-coding-test-automation.md`
- `backlog/654-agentic-coding-cli-surface.md`
- `backlog/655-agentic-coding-365fo.md`

## 6) Testing (how we prove this model works)

- Platform smoke proves “Coder works” (see 653).
- Inner-loop checks run in a workspace (see 653).
- Preview E2E proves deployed fidelity (see 653).
- Task E2E proves automation can run and clean up (see 651).

### 6.1 Success criteria

- A human can do productive development in a Coder workspace without privileged networking.
- An automation “task” can run in the same environment contract, produce a PR/evidence, and clean up.
- E2E merge gating happens against preview environments (no Telepresence dependency).
- The CLI “inner loop” is runnable in a Coder workspace and does not rely on Telepresence.

## 7) Risks / guardrails

- Avoid privilege creep: workspaces/tasks remain unprivileged by default.
- Avoid test drift: keep the layer split; do not mix “platform smoke” with “product E2E”.
- Avoid “escape hatches”: if a workflow requires Telepresence, it is out of model scope.

## 8) Deprecations / replacements (explicit)

This suite supersedes the following *as the platform-level default model*:

- Telepresence-centric inner-loop design: `backlog/621-ameide-cli-inner-loop-test.md`
- “Remote-first development” as the default substrate: `backlog/435-remote-first-development.md`
- KEDA-scaled, queue-depth executor pods as the default automation substrate: `backlog/527-*.md`
- Coder workspace docs that assume `.devcontainer/devcontainer.json` is primary: `backlog/626-*.md`, `backlog/628-*.md`, `backlog/629-*.md`

## 9) References

- Devcontainer contract: `.devcontainer/coder/devcontainer.json`
- Suite docs: `backlog/651-agentic-coding-ameide-coding-agent.md`, `backlog/652-agentic-coding-dev-workspace.md`, `backlog/653-agentic-coding-test-automation.md`
- CLI surface: `backlog/654-agentic-coding-cli-surface.md`
- External executor profile (D365FO): `backlog/655-agentic-coding-365fo.md`
- Agent memory (curation + retrieval): `backlog/656-agentic-memory.md`
