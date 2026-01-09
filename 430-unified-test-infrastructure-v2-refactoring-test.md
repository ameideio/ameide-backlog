---
title: 430v2 – Unified Test Infrastructure (Repo Refactoring Plan)
status: proposal
owners:
  - platform
  - test-infra
created: 2026-01-09
updated: 2026-01-09
parent: 430-unified-test-infrastructure-v2.md
---

## Goal

Implement 430v2 end-to-end so that:

- all tests (unit/integration/e2e) run under `ameide dev inner-loop-test`
- Phase 1/2 are local-only and use native tooling (no packs/scripts, no modes)
- Phase 3 is cluster-only and Playwright-only (Telepresence/Tilt harness)
- JUnit XML is produced for every phase, always (synthetic JUnit on early failure)

Target contract: `backlog/430-unified-test-infrastructure-v2-target.md`

---

## Plan (time-boxed, no dual-contract forever)

### Step 1 — Make the CLI the single orchestrator (authoritative)

Update `ameide dev inner-loop-test` to:

- Phase 0:
  - enforce “no legacy surface” (no `INTEGRATION_MODE`, no `run_integration_tests.sh`, no `tools/integration-runner/*`) with a temporary allowlist to avoid bricking the repo during migration
  - validate vendor-driven discovery wiring (Go compile/link; Jest list; Pytest collect; Playwright list where applicable)
- Phase 1:
  - `go test ./...`
  - Jest unit suite (repo-wide convention)
  - Pytest unit suite (repo-wide convention)
- Phase 2:
  - `go test -tags=integration ./...`
  - Jest integration suite (repo-wide convention)
  - Pytest integration suite (repo-wide convention)
- Phase 3:
  - Telepresence/Tilt preflight + cleanup
  - Playwright only

Evidence contract:
- always write JUnit per phase
- on command failure before JUnit exists, write synthetic JUnit

### Step 2 — Remove “pack script” dependency from the contract

- Stop treating `**/__tests__/integration/run_integration_tests.sh` as required.
- Stop referencing `tools/integration-runner/*` as the execution substrate.
- Keep legacy scripts temporarily only if needed to bridge CI; do not generate new ones.

### Step 3 — Standardize classification in-code

Go:
- add `//go:build integration` to Phase 2 tests
- remove dual-mode test logic that branches on environment “mode”

TS/Jest:
- enforce `__tests__/unit/**` and `__tests__/integration/**`
- ensure JUnit reporter is enabled

Python/pytest:
- enforce `@pytest.mark.integration` for Phase 2
- configure strict markers + JUnit output in pytest config

### Step 4 — Make scaffolding generate v2 by default

All scaffolders must generate:

- unit tests that run in Phase 1
- integration tests that run in Phase 2 (mocked/stubbed only)
- e2e tests only where Playwright is applicable and only in Phase 3

Hard rule:
- scaffolded tests should be **RED by default** (failing assertions/placeholders), so an agent/human must implement them.
- do not scaffold `t.Skip()` as the placeholder mechanism (it hides missing coverage).

### Step 5 — Delete legacy tooling (after migration)

After the repo is migrated and CI gates are green:

- delete `tools/integration-runner/*`
- remove CI jobs/pipelines that execute pack scripts or rely on `INTEGRATION_MODE`
- remove docs that still describe the v1 contract as normative

---

## Acceptance criteria

- `ameide dev inner-loop-test` runs all three phases with strict ordering and fail-fast.
- All phases emit JUnit XML evidence into a stable artifacts structure.
- No test requires `INTEGRATION_MODE` to run.
- No test requires `run_integration_tests.sh` to run in Phase 1/2.
- Phase 3 is the only phase that can touch Telepresence/Tilt/Kubernetes.
