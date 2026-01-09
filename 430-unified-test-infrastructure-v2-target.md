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

- One front door: `ameide dev inner-loop-test`
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

All verification for agentic coding and CI gates is driven through:

- `ameide dev inner-loop-test`

This command:
- runs phases in strict order (fail-fast)
- emits JUnit evidence for each phase
- does not require users to understand runner specifics

### Phase ordering (strict)

1) **Phase 1: Unit** (local, pure)
2) **Phase 2: Integration** (local, mocked/stubbed only)
3) **Phase 3: E2E** (cluster only, Playwright only)

Invariants:
- Phase 1 and Phase 2 **must not** interact with Kubernetes, Tilt, or Telepresence.
- Cluster wiring (Telepresence/Tilt preflight/cleanup) is **Phase 3 only**.

---

## Definitions

### Unit (Phase 1)

- Local only; deterministic; fast.
- No Kubernetes / Tilt / Telepresence.
- No network calls to “real” dependencies.
- Uses the default unit runner for the language/tooling.

### Integration (Phase 2)

- Local only.
- Tests boundaries (serialization, adapters, client/server seams) against **mocks/stubs/in-memory fakes**.
- No Kubernetes / Tilt / Telepresence.
- **No environment-driven branching** inside tests (“mode”). A test either belongs in Phase 2 or it does not exist.

### E2E (Phase 3)

- Cluster only; remote-first.
- **Playwright only**.
- Playwright hits a real base URL (cluster ingress/gateway).
- Telepresence/Tilt is allowed only as part of the Phase 3 harness (preflight + routing + cleanup).

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

## Forbidden patterns (v2)

- `INTEGRATION_MODE` (or any “repo/local/cluster” mode variable) as part of test execution.
- `run_integration_tests.sh` or “integration pack” scripts as required entrypoints for Phase 1/2.
- Any “Phase 2 in cluster” execution (that is Phase 3).
- Skipping tests based on environment availability; missing deps must fail fast.
