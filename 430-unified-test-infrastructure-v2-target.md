---
title: 430v2 – Unified Test Infrastructure (Target Contract)
status: proposal
owners:
  - platform
  - test-infra
created: 2026-01-09
updated: 2026-01-09
parent: 430-unified-test-infrastructure-v2.md
supersedes:
  - 430-unified-test-infrastructure.md
---

## Goal

Define a single, repo-wide, **low-cognitive-load** testing contract:

- Local front door: `ameide dev inner-loop-test` (Phase 0/1/2/3)
- CI front door: `ameide ci test` (Phase 0/1/2)
- Strict phases: **Unit → Integration → E2E**
- Native tooling per language (Go/Jest/Pytest/Playwright)
- **No test “modes”** (`INTEGRATION_MODE` et al.)
- **No per-component runner scripts** as the canonical execution path (no `run_integration_tests.sh`)
- **JUnit XML is mandatory evidence** for every phase

This is the **normative contract**. Historical v1 patterns are kept for context only.

Cross-references:
- 621 front door: `backlog/621-ameide-cli-inner-loop-test.md`
- Remote-first / cluster posture: `backlog/435-remote-first-development.md`
- CI alignment (quality gate): `backlog/414-quality-gate-refactor.md`, `backlog/610-ci-rationalization.md`

---

## Contract

### Single front door

All verification is driven through the CLI (no pack scripts/modes):

- `ameide dev inner-loop-test`
- `ameide ci test`

These commands:
- runs phases in strict order (fail-fast)
- emits JUnit evidence for each phase
- does not require users to understand runner specifics

### Phase ordering (strict)

0) **Phase 0: Contract** (local only; vendor-driven discovery)
1) **Phase 1: Unit** (local, pure)
2) **Phase 2: Integration** (local, mocked/stubbed only)
3) **Phase 3: E2E** (cluster only, Playwright only)

Invariants:
- Phase 1 and Phase 2 **must not** interact with Kubernetes or Telepresence.
- Cluster wiring (Telepresence preflight/cleanup) is **Phase 3 only**.
- Phase 0 must be able to prove the contract using vendor tooling “discovery/collect/list” capabilities and must emit JUnit evidence (synthetic if needed).

---

## Definitions

### Unit (Phase 1)

- Local only; deterministic; fast.
- No Kubernetes / Telepresence.
- No network calls to “real” dependencies.
- Uses the default unit runner for the language/tooling.

### Integration (Phase 2)

- Local only.
- Tests boundaries (serialization, adapters, client/server seams) against **mocks/stubs/in-memory fakes**.
- No Kubernetes / Telepresence.
- **No environment-driven branching** inside tests (“mode”). A test either belongs in Phase 2 or it does not exist.

### E2E (Phase 3)

- Cluster only; remote-first.
- **Playwright only**.
- Playwright hits a real base URL (cluster ingress/gateway).
- Telepresence is allowed only as part of the Phase 3 harness (preflight + routing + cleanup).

---

## Classification and discovery (repo-wide conventions)

The contract intentionally avoids “tooling zoo”, so classification must be predictable:

### Go

- Unit: standard `go test ./...`
- Integration: opt-in via build tags:
  - integration tests must include `//go:build integration`
  - Phase 2 runs `go test -tags=integration ./...`

### TypeScript (Jest)

Pick one repo-wide convention and enforce it:

- Unit: `**/__tests__/unit/**`
- Integration: `**/__tests__/integration/**`

Jest config (root or per-workspace) must implement these selectors (e.g., `testMatch` / `testRegex`) and must enable a JUnit reporter.

### Python (pytest)

Pick one repo-wide convention and enforce it:

- Integration: `@pytest.mark.integration`
- Unit: “not integration”

Pytest configs must:
- register the `integration` marker
- enable strict markers (typos fail)
- emit JUnit XML

### Playwright (E2E only)

- E2E tests live under each UI/service’s Playwright location (existing Playwright conventions).
- E2E is selected and executed only by Phase 3.

---

## JUnit evidence contract (mandatory)

Every phase produces JUnit XML at a stable path under a run root (the orchestrator owns run-root creation).

Requirements:
- Each phase writes exactly one JUnit XML file (or a small, deterministic set) under that phase’s artifacts directory.
- If a runner fails before producing JUnit (tool crash, config error, missing deps), the orchestrator must emit a **synthetic JUnit** file that captures:
  - command invoked
  - exit code (if available)
  - stderr/stdout tail
  - phase name

Tooling expectations (non-negotiable):
- Jest: JUnit reporter enabled (e.g., `jest-junit`).
- Pytest: `--junitxml=...`.
- Playwright: built-in JUnit reporter.
- Go: orchestrator converts `go test -json` output into JUnit.

---

## Phase 0 (Contract) — how correctness is verified

Phase 0 is a **preflight contract check** that prevents “zoo drift” and keeps the repo vendor-aligned.

Phase 0 must validate using native tooling *without running a full suite*:

- **Go:** compile/link checks for both default tags and `integration` tag without executing test binaries (preferred):
  - `go test -exec /bin/true ./...`
  - `go test -tags=integration -exec /bin/true ./...`
  - (fallback) `go test -run '^$' ...` when `-exec` is unavailable.
- **Jest:** discovery via `jest --listTests` (unit vs integration selectors) and fail on misconfiguration.
- **Pytest:** discovery via `pytest --collect-only` (unit vs integration selectors) and fail on misconfiguration.
- **Playwright:** discovery via `playwright test --list` (no cluster required).

Additionally, Phase 0 must enforce the v2 “no legacy surface” rule:
- no `INTEGRATION_MODE`
- no `run_integration_tests.sh`
- no `tools/integration-runner/*` execution contract

**Migration safety valve (temporary):** to avoid breaking the repo instantly, Phase 0 may support a repo-root allowlist file:
- `.ameide/test-contract-allowlist.txt`

This allowlist must be treated as a time-boxed migration artifact and should trend toward empty; new violations must be blocked.

**Toolchain availability:** Phase 0 may treat Jest/Pytest/Playwright discovery as best-effort when the toolchain is not installed yet (e.g., no `node_modules/` or no `.venv/`). Phase 1 is responsible for syncing dependencies; Phase 0 must still enforce the “no legacy surface” rule.

---

## Forbidden patterns (v2)

- `INTEGRATION_MODE` (or any “repo/local/cluster” mode variable) as part of test execution.
- `run_integration_tests.sh` or “integration pack” scripts as required entrypoints for Phase 1/2.
- Any “Phase 2 in cluster” execution (that is Phase 3).
- Skipping tests based on environment availability; missing deps must fail fast.
