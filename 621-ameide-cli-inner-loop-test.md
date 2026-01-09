# 621 — `ameide dev inner-loop-test` (Agent Inner-Loop Verification)

## Principles / invariants (non-negotiable)

- This is an **inner-loop development tool** for an AI coding agent (and engineers), optimized to keep the agent productive and consistent without requiring deep knowledge of local cluster/tooling configuration.
- Agent “instructions” must stay minimal:
  - **“If `ameide dev inner-loop-test` passes, the work is complete; if it fails, fix the first actionable error and rerun.”**
  - The tool absorbs environment complexity; the agent does not.
- Ordering is strict: **unit → integration → e2e** (never do cluster/Telepresence work before local tests are green).
- The runner must be **as smart as possible internally** (discovery, caching, preflight, actionable failure messages) while remaining a **no-brainer UX**:
  - **no flags**, **no launch modes**, **no user-chosen parameters** (besides `--help`).
  - one default behavior; fail-fast on first failure.
- Must be runnable in **CI and local** environments without privileged access:
  - **no `sudo`**, **no host mutations** (e.g., editing `/etc/hosts`), no interactive prompts.
- Must leave the environment **clean after each run**:
  - no lingering background processes, intercepts, Telepresence daemons, or stray resources created by the run.
  - small on-disk caches/state that improve iteration are allowed, but must be explicit, bounded, and safe to delete.
- Full CI/CD (images, preview envs, GitOps promotions) is **out of scope** for this tool.

## Objective

Provide a **single, fail-fast, no-flags** CLI entrypoint that an interactive AI agent can run locally (and in CI) to validate changes end-to-end **without leaving state behind**:

- `ameide dev inner-loop-test`

This is intentionally **not** “full CI”: it is an agentic inner-loop tool that prioritizes **time-to-first-actionable-signal** and **explainable failures**.

## Cross-references

- `backlog/468-testing-front-door.md` (front-door testing entrypoints; this backlog adds an agent-optimized inner-loop tool)
- `backlog/430-unified-test-infrastructure-v2-target.md` (normative test contract: phases, native tooling, JUnit evidence)
- `backlog/435-remote-first-development.md` (Tilt + Telepresence as the default cluster dev substrate)
- `backlog/581-parallel-ai-devcontainers.md` (parallel agent slots; header-filtered intercepts)
- `.github/workflows/ci-core-quality.yml` (authoritative CI quality gate for PRs)

## Implementation status (current)

This backlog item is now **implemented** as a first-class Go CLI command:

- User entrypoint:
  - `ameide dev inner-loop-test`
- Go implementation (modular; Phase 3 is split across `phase3_*.go`):
  - CLI wiring: `packages/ameide_core_cli/internal/commands/dev_inner_loop.go`
  - Orchestrator: `packages/ameide_core_cli/internal/innerloop/run.go`
  - Phase 1: `packages/ameide_core_cli/internal/innerloop/phase1.go`
  - Phase 2: `packages/ameide_core_cli/internal/innerloop/phase2.go`
  - Phase 3: `packages/ameide_core_cli/internal/innerloop/phase3_runner.go` + `packages/ameide_core_cli/internal/innerloop/phase3_*.go`
- Tilt orchestration (Phase 3):
  - `Tiltfile` resource `e2e-playwright-ci` runs the hidden CLI runner:
    - Implementation detail: the CLI re-execs itself in small, hidden “phase3 scope” entrypoints so Telepresence can run a concrete process `-- <command>` while connected/intercepted.

Legacy bash runner scripts remain in-tree for reference and parity comparison (do not delete yet):

- `tools/dev/agent-local-ci.sh`
- `tools/dev/agent-local-ci/**`

## Deliverable shape (what the CLI does)

The CLI runs exactly three phases in order and exits non-zero on the first failure.

Artifacts/logs are always written under a single run root:

- `artifacts/agent-ci/<timestamp>/`

It always prints **phase durations** and **total duration** at the end of execution (even when a later phase fails).

### Phase 1 — Standard build/lint/unit (local only)

**Goal:** catch fast failures without touching Kubernetes or Telepresence.

Canonical tasks (toolchains directly):

- **TypeScript**
  - Lockfile-aware install by detection: `pnpm install --frozen-lockfile` only when needed
  - `pnpm lint`
  - `pnpm typecheck`
  - `pnpm test:unit`
- **Python**
  - Lockfile-hash-aware venv sync by detection: `uv sync --all-packages --dev` only when needed
  - `uvx ruff check …` (targeted packages)
  - `uv run -m pytest -q packages --ignore-glob='*/__tests__/integration/*'` (+ junit under `RUN_ROOT`)
- **Go**
  - `go test ./...` (default tags only; integration-tagged tests are excluded by Go build constraints)

**Invariant:** Phase 1 must not require `kubectl`, `telepresence`, or `tilt`.

### Phase 2 — Integration tests (local mocked/stubbed only)

**Goal:** run local-only integration tests that exercise boundaries against mocks/stubs/in-memory fakes.

Execution rules (opinionated, fail-fast):

- No Kubernetes / Tilt / Telepresence.
- No environment “mode” variables (no `INTEGRATION_MODE`).
- Use native tooling selectors per language (430v2):
  - **Go:** `go test -tags=integration ./...`
  - **Jest/TS:** repo-wide selection for `__tests__/integration/**` via Jest config
  - **Pytest/Python:** `@pytest.mark.integration` selection via pytest config

### Phase 3 — E2E (cluster; Tilt + Telepresence + Playwright)

**Goal:** run Playwright E2E against a stable URL while validating local changes via Telepresence intercepts.

Constraints / invariants:

- Only Phase 3 may touch `kubectl`, `telepresence`, `tilt`.
- Parallel-safe routing is mandatory:
  - Require Telepresence **HTTP header filtered** intercepts (`--http-header "X-Ameide-Agent=<id>"`)
  - Hard-fail if header filtering is unavailable (no global intercept fallback)
  - Playwright must send the routing header (implemented via Playwright config in `services/www_ameide_platform`)
- No privileged host assumptions:
  - never edits `/etc/hosts`
  - uses FQDNs / in-cluster Service discovery and preflights to reject loopback/poisoned DNS
- Cleanup is strict (workstation):
  - no lingering intercept
  - **always `telepresence quit -s`** at the start and end of Phase 3 to avoid stale sessions and guarantee cleanliness

Vendor-aligned orchestration shape:

- Phase 3 invokes `tilt ci … -- --resources e2e-playwright-ci`
- Tilt runs the hidden CLI runner:
- which scopes `telepresence connect -- <cmd>` and `telepresence intercept -- <cmd>`
  - (Implementation detail: Telepresence requires a process entrypoint; this is not a user-facing “mode” and is not part of the public contract.)

## Non-goals

- Replacing CI: CI remains authoritative.
- Building/publishing images, preview environments, or GitOps promotions.
- Supporting multiple environments/configs via flags.
