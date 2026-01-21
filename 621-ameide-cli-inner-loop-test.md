> Superseded by the internal-only, Coder-based model in `backlog/650-agentic-coding-overview.md` and `backlog/653-agentic-coding-test-automation-coder.md`.

> **Test contract update:** the normative v2 test contract is `backlog/430-unified-test-infrastructure-v2-target.md`.  
> For the updated CLI posture, see `backlog/621-ameide-cli-inner-loop-test-v2.md`.

# 621 — `ameide test` (Agent Inner-Loop Verification)

## Principles / invariants (non-negotiable)

- This is an **inner-loop development tool** for an AI coding agent (and engineers), optimized to keep the agent productive and consistent without requiring deep knowledge of local cluster/tooling configuration.
- Agent “instructions” must stay minimal:
  - **“If `ameide test` passes, the work is complete; if it fails, fix the first actionable error and rerun.”**
  - The tool absorbs environment complexity; the agent does not.
- Ordering is strict: **contract → unit → integration** (never do cluster-only work before local tests are green).
- The runner must be **as smart as possible internally** (discovery, caching, preflight, actionable failure messages) while remaining a **no-brainer UX**:
  - **no flags**, **no launch modes**, **no user-chosen parameters** (besides `--help`).
  - one default behavior; fail-fast on first failure.
- Must be runnable in **CI and local** environments without host mutation or interactive prompts:
  - no host mutations (e.g., editing `/etc/hosts`)
  - no interactive prompts
- Must leave the environment **clean after each run**:
  - no lingering background processes, intercepts, Telepresence daemons, or stray resources created by the run.
  - small on-disk caches/state that improve iteration are allowed, but must be explicit, bounded, and safe to delete.
- Full CI/CD (images, preview envs, GitOps promotions) is **out of scope** for this tool.

## Objective

Provide a **single, fail-fast, no-flags** CLI entrypoint that an interactive AI agent can run locally (and in CI) to validate changes **without leaving state behind**:

- `ameide test`

This is intentionally **not** “full CI”: it is an agentic inner-loop tool that prioritizes **time-to-first-actionable-signal** and **explainable failures**.

CI uses the same two front doors:

- `ameide test` (Phase 0/1/2 only; local-only)
- `ameide test cluster` (Phase 4/5 only; cluster-only)

## Cross-references

- `backlog/468-testing-front-door.md` (front-door testing entrypoints; this backlog adds an agent-optimized inner-loop tool)
- `backlog/430-unified-test-infrastructure-v2-target.md` (normative test contract: phases, native tooling, JUnit evidence)
- `backlog/435-remote-first-development.md` (Telepresence as the default cluster dev substrate)
- `backlog/581-parallel-ai-devcontainers.md` (parallel agent slots; header-filtered intercepts)
- `.github/workflows/ci-core-quality.yml` (authoritative CI quality gate for PRs)

## Relationship to AmeideDevContainerService (Coder workspaces)

This command is designed for **workstation/local devcontainer** inner-loop flows where Telepresence is the substrate.

In **Kubernetes-hosted human workspaces** (AmeideDevContainerService; 626/628), Telepresence is intentionally **not supported**
(no `CAP_NET_ADMIN` / TUN device).

## Implementation status (current)

This backlog item is now **implemented** as a first-class Go CLI command:

- User entrypoint:
  - `ameide test`
- Go implementation:
  - CLI wiring: `packages/ameide_core_cli/internal/commands/test.go`
  - Orchestrator: `packages/ameide_core_cli/internal/innerloop/run.go`
  - Phase 0: `packages/ameide_core_cli/internal/innerloop/phase0.go`
  - Phase 1: `packages/ameide_core_cli/internal/innerloop/phase1.go`
  - Phase 2: `packages/ameide_core_cli/internal/innerloop/phase2.go`
  - Phase 4: `packages/ameide_core_cli/internal/innerloop/cluster.go`
  - Phase 5: `packages/ameide_core_cli/internal/innerloop/e2e.go`

Legacy runner scripts have been removed; the canonical entrypoints are the CLI commands above.

Implementation shipped and merged into `ameide` `main` via:

- `ameideio/ameide#582` (canonicalize `ameide test` Phase 0/1/2 front door)
- `ameideio/ameide#589` (Playwright E2E runner; now Phase 4 of `ameide test cluster`)
- `ameideio/ameide#584` (devcontainer/Coder toolchain provisioning so `ameide test` is runnable)
- `ameideio/ameide#590` / `ameideio/ameide#591` (`ameide dev` robustness improvements; adjacent inner-loop ergonomics)

## Deliverable shape (what the CLI does)

The CLI runs phases in strict order (fail-fast) and exits non-zero on the first failure.

Artifacts/logs are always written under a single run root:

- `artifacts/agent-ci/<timestamp>/`

It always prints **phase durations** and **total duration** at the end of execution (even when a later phase fails).

### Phase 0 — Contract (local only)

**Goal:** fail fast on contract drift (legacy “modes”, pack scripts) and validate vendor-driven discovery wiring before executing full suites.

- Go: compile/link checks for unit and integration-tagged code.
- Jest/Pytest: discovery/collect/list modes where available.
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

**Goal:** run local-only integration tests that exercise boundaries against mocks/stubs/in-memory doubles.

Execution rules (opinionated, fail-fast):

- No Kubernetes / Telepresence.
- No environment “mode” variables (no `INTEGRATION_MODE`).
- Use native tooling selectors per language (430v2):
  - **Go:** `go test -tags=integration ./...`
  - **Jest/TS:** repo-wide selection for `__tests__/integration/**` via Jest config
  - **Pytest/Python:** `@pytest.mark.integration` selection via pytest config

### Phase 4/5 — Cluster-only verification (separate commands; not part of `ameide test`)

This backlog originally described a Telepresence-based E2E harness. The current 430v2 contract keeps the agent/human front door (`ameide test`) strictly Phase 0/1/2, and runs cluster-only checks separately:

- `ameide test cluster`
- `ameide test cluster`

Constraints / invariants:

- Cluster-only checks must be explicit, fail-fast, and leave no state behind.
- Playwright base URL is explicit and deterministic:
  - `ameide test cluster` reads `AUTH_URL` from `ConfigMap/www-ameide-platform-config` and passes it to Playwright as `AMEIDE_PLATFORM_BASE_URL` (fail-fast); it also verifies it matches the HTTPRoute-derived ingress host.

Failure behavior:
- If any prerequisite is missing, cluster-only checks fail fast with an actionable message and still emit JUnit evidence under the run root.

## Non-goals

- Replacing CI: CI remains authoritative.
- Building/publishing images, preview environments, or GitOps promotions.
- Supporting multiple environments/configs via flags.
