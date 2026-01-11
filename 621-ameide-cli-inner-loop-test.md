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
- Must be runnable in **CI and local** environments without host mutation or interactive prompts:
  - no host mutations (e.g., editing `/etc/hosts`)
  - no interactive prompts
  - note: Phase 3 depends on Telepresence; in environments where Telepresence requires elevated privileges for its daemon, the environment must be preconfigured to run it non-interactively (e.g., NOPASSWD sudo in devcontainers/CI images).
- Must leave the environment **clean after each run**:
  - no lingering background processes, intercepts, Telepresence daemons, or stray resources created by the run.
  - small on-disk caches/state that improve iteration are allowed, but must be explicit, bounded, and safe to delete.
- Full CI/CD (images, preview envs, GitOps promotions) is **out of scope** for this tool.

## Objective

Provide a **single, fail-fast, no-flags** CLI entrypoint that an interactive AI agent can run locally (and in CI) to validate changes end-to-end **without leaving state behind**:

- `ameide dev inner-loop-test`

This is intentionally **not** “full CI”: it is an agentic inner-loop tool that prioritizes **time-to-first-actionable-signal** and **explainable failures**.

CI uses a sibling command that runs the same contract phases without cluster E2E:

- `ameide ci test` (Phase 0/1/2 only)

## Cross-references

- `backlog/468-testing-front-door.md` (front-door testing entrypoints; this backlog adds an agent-optimized inner-loop tool)
- `backlog/430-unified-test-infrastructure-v2-target.md` (normative test contract: phases, native tooling, JUnit evidence)
- `backlog/435-remote-first-development.md` (Telepresence as the default cluster dev substrate)
- `backlog/581-parallel-ai-devcontainers.md` (parallel agent slots; header-filtered intercepts)
- `.github/workflows/ci-core-quality.yml` (authoritative CI quality gate for PRs)

## Relationship to AmeideDevContainerService (Coder workspaces)

This command is designed for **workstation/local devcontainer** inner-loop flows where Telepresence is the substrate.

In **Kubernetes-hosted human workspaces** (AmeideDevContainerService; 626/628), Telepresence is intentionally **not supported**
(no `CAP_NET_ADMIN` / TUN device), so Phase 3 must be considered “not runnable” there.

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

Legacy runner scripts have been removed; the canonical entrypoints are the CLI commands above.

## Deliverable shape (what the CLI does)

The CLI runs phases in strict order (fail-fast) and exits non-zero on the first failure.

Artifacts/logs are always written under a single run root:

- `artifacts/agent-ci/<timestamp>/`

It always prints **phase durations** and **total duration** at the end of execution (even when a later phase fails).

### Phase 0 — Contract (local only)

**Goal:** fail fast on contract drift (legacy “modes”, pack scripts) and validate vendor-driven discovery wiring before executing full suites.

- Go: compile/link checks for unit and integration-tagged code.
- Jest/Pytest/Playwright: discovery/collect/list modes where available.
- Always emits JUnit evidence for Phase 0 (synthetic if needed).

### Phase 1 — Standard build/lint/unit (local only)

**Goal:** catch fast failures without touching Kubernetes or Telepresence.

Canonical tasks (toolchains directly):

- **TypeScript**
  - Lockfile-aware install by detection: `pnpm install --frozen-lockfile` only when needed
  - `pnpm lint`
  - `pnpm typecheck`
  - Jest unit suite (repo-wide `__tests__/unit/**`; orchestrated by the CLI with JUnit evidence)
- **Python**
  - Lockfile-hash-aware venv sync by detection: `uv sync --all-packages --dev` only when needed
  - `uvx ruff check …` (targeted packages)
  - `uv run -m pytest -q packages --ignore-glob='*/__tests__/integration/*'` (+ junit under `RUN_ROOT`)
- **Go**
  - `go test ./...` (default tags only; integration-tagged tests are excluded by Go build constraints)

**Invariant:** Phase 1 must not require `kubectl` or `telepresence`.

### Phase 2 — Integration tests (local mocked/stubbed only)

**Goal:** run local-only integration tests that exercise boundaries against mocks/stubs/in-memory fakes.

Execution rules (opinionated, fail-fast):

- No Kubernetes / Telepresence.
- No environment “mode” variables (no `INTEGRATION_MODE`).
- Use native tooling selectors per language (430v2):
  - **Go:** `go test -tags=integration ./...`
  - **Jest/TS:** repo-wide selection for `__tests__/integration/**` via Jest config
  - **Pytest/Python:** `@pytest.mark.integration` selection via pytest config

### Phase 3 — E2E (cluster; Telepresence + Playwright)

**Goal:** run Playwright E2E against a stable URL while validating local changes via Telepresence intercepts.

Constraints / invariants:

- Only Phase 3 may touch `kubectl` and `telepresence`.
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

- Phase 3 scopes `telepresence connect -- <cmd>` and `telepresence intercept -- <cmd>` via hidden CLI entrypoints.
  - (Implementation detail: Telepresence requires a process entrypoint; these are hidden commands, not user-facing “modes”.)

Prerequisites (must be explicit; no guesswork):

- `telepresence` installed and compatible with header-filtered intercepts (Telepresence/traffic-manager >= 2.25).
- A reachable Kubernetes context/namespace with the target workload (`www-ameide-platform`) and sidecar injection enabled.
- Playwright test secrets present in-cluster (e.g., `playwright-int-tests-secrets` in the configured namespace).
- E2E ingress host is fixed: `https://platform.local.ameide.io` (Phase 3 must fail fast if ingress/auth prerequisites are missing).

Failure behavior:
- If any prerequisite is missing, Phase 3 fails fast with an actionable message and still emits JUnit evidence under the run root.

## Non-goals

- Replacing CI: CI remains authoritative.
- Building/publishing images, preview environments, or GitOps promotions.
- Supporting multiple environments/configs via flags.
