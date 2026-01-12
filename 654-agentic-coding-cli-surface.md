---
title: "654 – Agentic coding — Ameide CLI surface (test front doors + profiles)"
status: draft
owners:
  - platform-devx
  - test-infra
  - qa
created: 2026-01-12
suite: "650-agentic-coding"
source_of_truth: false
---

# 654 – Agentic coding — Ameide CLI surface (test front doors + profiles)

## 0) Purpose

Define the **complete, stable CLI surface** that agents and humans use to run verification in a complex monorepo.

Primary outcome: the CLI is the **cognitive-load absorber** so agents do not need to learn:

- which vendors/toolchains exist
- how tests are classified across languages
- which commands to run in which order
- how to produce evidence consistently
- how to avoid leaving behind local/cluster state

This backlog inherits the internal-first model from `backlog/650-agentic-coding-overview.md`.

## 1) Decisions (normative for the CLI surface)

### 1.1 “No-brainer” means no flags for core verification

The core verification entrypoints accept **no flags** (besides `--help`) and are stable contracts for agents:

- ordering is strict
- failures are actionable
- evidence is always produced
- cleanup is guaranteed

### 1.1.1 Phase 3 is part of the CLI surface (but not part of the Phase 0/1/2 front door)

Phase 3 (E2E) is part of the CLI surface as a **separate, explicit command**:

- Phase 0/1/2 front door: fast, local-only verification (no cluster, no privileged capabilities)
- Phase 3: Playwright E2E against a **deployed target URL** (preview environment truth)

This keeps the “no-brainer” front door fast and universally runnable (including in Coder workspaces and tasks) while still providing a deterministic, CLI-owned Phase 3.

### 1.2 The CLI defines test front doors; vendor tools are implementation details

Agents run Ameide commands; the CLI owns the mapping to vendor runners (Go/Jest/Pytest/Playwright) and their configuration.

### 1.3 Evidence is mandatory

Every phase produces JUnit evidence. If a vendor runner fails before producing JUnit, the CLI emits **synthetic JUnit** describing the failure.

### 1.4 Profiles are first-class (templates are agent types)

The CLI surface must work consistently across agent profiles:

- **code** profile (app/service/package changes)
- **gitops** profile (GitOps-only changes)
- **sre** profile (triage/diagnostics; typically stricter writes)
- **365fo** profile (external executor; D365 Finance & Operations via Windows dev VM tooling)

Profiles imply different guardrails and “allowed edits”.

### 1.5 Template-scoped instructions (`AGENTS.md`) are required

Every Coder template/profile must provide a scoped `AGENTS.md` governing that workspace’s repo checkout.

The CLI must assume it is being run under a profile’s instruction scope (agent working directory is under the subtree covered by that `AGENTS.md`).

## 2) Scope (in/out)

### 2.1 In scope

- Define the target command groups and naming conventions.
- Define which commands are “agent front doors” vs “power tools”.
- Define scenario mapping (workspace vs task vs CI vs preview E2E).
- Define how profiles affect guardrails and which front doors apply.

### 2.2 Out of scope

- Implementing the CLI refactors (separate backlog(s) or PR work).
- Specifying the full test taxonomy per language (owned by `backlog/430-unified-test-infrastructure-v2-target.md`).

## 3) Terminology (additions to 650)

- **Front door command:** no-flags, agent-safe contract for verification.
- **Power tool command:** may accept flags and pass-through args; not the agent default.
- **Profile:** workspace/task identity that selects guardrails and permissible operations.

## 4) Current CLI surface (as-is; for alignment)

Current command groups include:

- `ameide ci test` (Phase 0/1/2)
- `ameide ci e2e` (Playwright runner; accepts pass-through args)
- `ameide dev inner-loop-test` (currently includes a cluster E2E phase)
- `ameide doctor` (preflight deterministic requirements)
- `ameide verify` (repo/primitives invariants)

Current drift vs internal-first model:

- `ameide dev inner-loop-test` currently advertises Phase 3 cluster E2E behavior, which conflicts with the internal-first test model where “no-brainer” verification is Phase 0/1/2 and deployed E2E is a separate preview-env layer.
- `ameide dev inner-loop` is Telepresence-centric and is out of scope for the internal-first platform model.

## 5) Target CLI surface (to-be)

### 5.1 Top-level structure

The CLI surface is organized by intent:

- **`ameide test`**: no-brainer verification front door (Phase 0/1/2) for agents and humans
- **`ameide e2e`**: deployed-system E2E runner (Playwright) against a provided base URL (preview env truth)
- **`ameide smoke`**: platform smoke (Coder/template provisioning; not product E2E)
- **`ameide verify`**: repo/primitives invariants (structural correctness)
- **`ameide doctor`**: toolchain and environment preflight (deterministic readiness)

Notes:

- `ameide ci ...` is an execution context, not a different contract. CI should call the same front doors (or stable aliases).
- Profile selection is implicit (workspace/task template) or explicit (profile file); it must not require user flags.

### 5.2 Core front doors (no flags)

1. `ameide test`
   - runs Phase 0/1/2 in strict order
   - produces evidence under a stable run root
   - never touches cluster networking, Telepresence, or privileged capabilities

2. `ameide e2e`
   - runs Phase 3: Playwright against a deployed target (preview env truth)
   - is intentionally not part of the Phase 0/1/2 front door

3. `ameide smoke`
   - validates platform plumbing (Coder + template provisioning + code-server reachability)
   - does not validate the product itself

### 5.3 Profile-aware guardrails (defense in depth)

The CLI must implement profile-aware guardrails as part of front door execution:

- detect profile (see §5.4)
- fail fast on forbidden diffs for that profile (example: code profile may not edit `gitops/`)
- emit evidence explaining which guardrail fired

Guardrails complement (but do not replace) token/RBAC enforcement.

### 5.4 Profile detection (no user flags)

Profile is detected in this order:

1. Explicit repo-local marker file (authoritative):
   - `.ameide/agent-profile` containing `code|gitops|sre|365fo`
2. Workspace-provided environment:
   - `AMEIDE_AGENT_PROFILE=...` (set by the template)
3. Heuristic fallback:
   - if repo root appears to be `ameide-gitops`, assume `gitops`
   - otherwise assume `code`

The template-scoped `AGENTS.md` defines additional instructions for the agent, but the profile marker is what the CLI uses for enforcement.

## 6) Scenario mapping (what runs where)

### 6.1 Human inner loop (Coder workspace; code profile)

- default: `ameide test`
- optional interactive checks: start dev server inside workspace
- deployed truth: `ameide e2e` runs against preview env URL (typically driven by CI)

### 6.2 Agent execution (Coder task; code profile)

- preflight: `ameide doctor --strict` (power tool; may be called by orchestrator)
- verification: `ameide test`
- output: evidence bundle + PR URL (if the task is “patch”)

### 6.3 CI verification (repo gate)

- `ameide test` (or alias `ameide ci test`) for Phase 0/1/2

### 6.4 CI deployed-system truth (preview env)

- `ameide e2e` (or alias `ameide ci e2e`) against preview base URL(s) (Phase 3)

### 6.5 Platform smoke (Coder/template)

- `ameide smoke` (or `ameide ci smoke`) to validate:
  - template provisioning
  - app proxy reachability
  - workspace cleanup

### 6.6 External executor profile (D365FO)

For vendor-locked domains (D365FO), the CLI still presents the same front doors, but the implementation delegates execution to an external executor tool contract.

Expected behavior for the `365fo` profile:

- `ameide test` (Phase 0/1/2) calls the FO tool surface (MCP bridge → VM executor) for verification and emits JUnit evidence in the workspace/task run root.
- `ameide e2e` (Phase 3) validates the full chain (Coder task → VM executor → git push → workspace mirror verification), as specified in `backlog/655-agentic-coding-365fo.md`.

## 7) Evidence and artifacts

The CLI owns a deterministic artifacts layout under repo root:

- `artifacts/agent-ci/<run-id>/phase0/`
- `artifacts/agent-ci/<run-id>/phase1/`
- `artifacts/agent-ci/<run-id>/phase2/`
- `artifacts/agent-ci/<run-id>/e2e/` (when applicable)

Each phase produces JUnit in its phase directory.

## 8) Deprecations / replacements

- Replace `ameide dev inner-loop-test` with `ameide test` as the canonical front door for agents (Phase 0/1/2 only).
- Remove Telepresence-centric workflows (`ameide dev inner-loop`) from the primary platform model; if retained temporarily, they live under an explicit `legacy` namespace and are not agent defaults.

## 9) References

- Internal-first model: `backlog/650-agentic-coding-overview.md`
- Test automation layers: `backlog/653-agentic-coding-test-automation.md`
- Test taxonomy contract: `backlog/430-unified-test-infrastructure-v2-target.md`
- Front door history: `backlog/468-testing-front-door.md`
- Prior inner-loop design (superseded): `backlog/621-ameide-cli-inner-loop-test.md`
- External executor profile (D365FO): `backlog/655-agentic-coding-365fo.md`
- Agent memory (curation + retrieval): `backlog/656-agentic-memory.md`
