---
title: "653 – Agentic coding — test automation (no Telepresence; local checks + Argo preview E2E)"
status: draft
owners:
  - platform-devx
  - qa
  - gitops
created: 2026-01-12
suite: "650-agentic-coding"
source_of_truth: false
---

# 653 – Agentic coding — test automation (no Telepresence; local checks + Argo preview E2E)

## 0) Purpose

Define a single, consistent testing model that works for:

- Humans developing in Coder workspaces
- Automation tasks running in ephemeral workspaces
- Merge gating against deployed preview environments

This document inherits all decisions from `backlog/650-agentic-coding-overview.md`.

## 0.2 Status update (2026-01-17)

- Coder control plane + provisioners are healthy; recent workspaces are provisioning successfully.
- Phase 3 workspace routing RBAC is now bootstrap-managed (predeclared roles + bind-only delegation) to avoid RBAC escalation failures during Terraform applies.
- Known non-blocker: `coder show` “containers” warnings can occur in envbuilder-based workspaces because `docker` is intentionally not present.

Update (2026-01-19): platform smoke must catch code-server cached-arch failures

- Observed failure mode: a workspace pod can be `Running` while the VS Code (Web) app returns `502 Bad Gateway` because code-server never started.
- Root cause: the code-server install is cached on the workspace PVC (`/workspaces/.code-server`) and can be for the wrong CPU architecture after node pool churn (e.g., amd64 cache reused on arm64 nodes), causing `Exec format error`.
- Platform smoke must treat “app proxy path returns 502” as a platform failure and include a cache sanity check / remediation expectation.

## 0.1 Primary purpose of the CLI: reduce cognitive load for agents

This monorepo spans multiple runtimes and vendor toolchains (Go/Node/Python/Playwright/Kubernetes).

Decision: the Ameide CLI is the **no-brainer front door** that:

- discovers what tests exist and how to run them
- runs phases in strict order and fails fast
- produces evidence (JUnit) even when a vendor tool crashes
- avoids leaving state behind

## 1) Decisions (normative for test layers)

### 1.1 No Telepresence in any test layer

No test layer depends on Telepresence or cluster network interception.

### 1.2 Test layers are intentionally distinct

1. **Platform smoke:** “Coder workspace platform is healthy.”
2. **Developer inner loop:** “This change is locally correct.”
3. **Preview E2E:** “This change works when deployed.”
4. **Automation tasks:** repeatable helpers for (2) and for triage of (3).

## 2) Scope (in/out)

### 2.1 In scope

- Define and standardize test layers for the internal-first, Coder-orchestrated model.
- Define the CLI entrypoints and merge gate signals.
- Define what gets deleted (Telepresence dependencies) from the test model.

### 2.2 Out of scope

- Preview environment provisioning details (owned by GitOps/Argo CD).
- Exact Playwright test content; this doc defines where it runs and what it gates.

## 3) Terminology (additions to 650)

- **Platform smoke:** validates Coder + template provisioning and app proxy path, not product correctness.
- **Merge gate truth:** the test signal that blocks/permits merge (preview E2E).

## 4) Architecture (test layers)

### 4.1 Layer 1 — platform smoke (Coder correctness)

Goal: catch “Coder/workspace provisioning is broken” quickly.

Minimum checks:

- Create a workspace from the standard template.
- Validate code-server is reachable via the Coder app proxy path (outside the workspace).
- Validate a small in-workspace check (git works; toolchain present).
- Delete the workspace and verify cleanup.

Additional checks (to catch known failures):

- Validate the VS Code (Web) app is not just routed, but actually responds (no `502`) and `/healthz` is reachable.
- Validate the workspace does not have a non-runnable cached code-server install for the wrong CPU architecture (see 652 §4.2.1).

Notes:

- This is not Playwright E2E for the product; it is platform plumbing validation.
- Smoke should include a “first-user bootstrapped” assertion for the Coder control plane (no `/setup` state).
- Naming note: this repo also has an in-cluster ArgoCD app commonly called “platform-smoke” that validates public routing/control-plane HTTP endpoints; that is complementary, but it does not replace the workspace-provisioning smoke checks listed above.
- Implementation note (current): the workspace-provisioning platform smoke is implemented as GitHub Actions workflows (`.github/workflows/coder-devcontainer-smoke.yaml`, `.github/workflows/coder-devcontainer-e2e.yaml`) and is intentionally GitHub-hosted so it remains independent of the in-cluster ARC runner fleet.

### 4.2 Layer 2 — developer inner loop (`ameide test`)

Goal: a single, no-flags command that is runnable in a Coder workspace and produces a fast, actionable failure.

Decision: `ameide test` must be Telepresence-free and safe to run in-cluster.
Update (2026-01): “safe to run in-cluster” here means “safe to run in a Coder workspace without Kubernetes/Telepresence access”; deployed-system E2E is separate.

Proposed contract (phases; exact tools are implementation details):

- **Phase 0:** contract discovery + repo hygiene + formatting/lint/typecheck
- **Phase 1:** unit tests
- **Phase 2:** integration tests (in-process or containerized, but not requiring privileged pods)

Non-negotiables:

- phases 0/1/2 require no cluster access and no privileged capabilities
- agents run this with no flags; the CLI owns discovery and runner selection
- each phase emits JUnit (synthetic if needed)

Minimum toolchain requirement (workspace contract):

- `go`, `gofmt`
- `node`, `pnpm`
- `uv`, `uvx`
- `buf`

If any of these are missing, the inner loop fails fast and the platform smoke should catch it.

Note: for vendor-locked external executor profiles (e.g., D365FO), Phase 1/2 execution may be delegated to an external executor tool contract, but the agent still runs the same CLI front door and receives the same evidence shape (see 655).

### 4.3 Layer 3 — preview environment E2E (merge gate truth)

Goal: validate the deployed system via real ingress URLs.

Requirements:

- A preview environment is created per PR (Argo CD).
- Playwright runs against the preview base URL(s).
- E2E results become the merge gate signal.

Decision: Phase 3 exists as a CLI-owned command surface:

- `ameide test e2e` runs Playwright against the deployed preview URL (Phase 3)
- Phase 3 is intentionally not bundled into the “no-brainer” Phase 0/1/2 front door so it remains fast and universally runnable

Note: non-Kubernetes domains may define a different Phase 3 target (not an Argo preview environment). For D365FO, Phase 3 is defined in `backlog/655-agentic-coding-365fo.md`.

### 4.4 Layer 4 — automation tasks (repeatable helpers)

Tasks complement the system by making repeatability cheap:

- **Preflight task:** run Layer 2 in a clean ephemeral workspace.
- **E2E triage task:** if Layer 3 fails, collect diagnostics and propose a patch PR.

## 5) Implementation plan

- Make `ameide test` the no-brainer phases 0/1/2 front door for humans and agents in workspaces.
- Standardize a “platform smoke” workflow that validates Coder control plane + template provisioning + code-server reachability.
- Standardize preview environment E2E execution and connect it to PR merge gating.
- Provide an optional “preflight task” runner that executes the inner loop in a clean ephemeral workspace (see 651).

## 6) Testing (validating the testing system)

- A single “smoke of the smoke” run in dev that proves:
  - platform smoke passes
  - inner loop passes in a workspace
  - preview E2E can run against a known-good preview URL

## 7) Risks / guardrails

- Avoid conflating layers: platform smoke must not become “product E2E”.
- Avoid slow inner loop: keep it fast and deterministic; preview E2E is the slower truth.
- Avoid secret sprawl: automation must not rely on human browser sessions.

## 8) Deprecations (explicit deletions allowed)

The following are deprecated by this model:

- Telepresence-based E2E execution for agent inner-loop verification (`backlog/621-ameide-cli-inner-loop-test.md` as currently written).
- Telepresence-based E2E execution as an agent default; Phase 3 E2E is owned by `ameide test e2e` (cluster-only; Playwright-only).
- Telepresence verification backlogs as platform requirements (e.g., `backlog/492-telepresence-verification.md`).

## 9) References

- Normative decisions: `backlog/650-agentic-coding-overview.md`
- Developer environment: `backlog/652-agentic-coding-dev-workspace-coder.md`
- Automation runtime: `backlog/651-agentic-coding-ameide-coding-agent.md`
- CLI surface: `backlog/654-agentic-coding-cli-surface-coder.md`
- External executor profile (D365FO): `backlog/655-agentic-coding-365fo.md`
- Agent memory (curation + retrieval): `backlog/656-agentic-memory-v6.md`
