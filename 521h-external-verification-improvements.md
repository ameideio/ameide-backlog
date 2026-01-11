# 521h — External Verification Improvements Log

Track changes to external verification gates (test requirements, scaffold-marker enforcement, 430 integration-pack contract enforcement, CI wiring).

Baseline: `backlog/521f-external-verification-baseline.md`

---

## Log

Add new entries here when external verification behavior changes.

### 2026-01-08

- CI flake hardening (Buf/BSR eventual consistency):
  - Go integration and extensions-runtime CI now generate Go SDK stubs from the local proto source-of-truth (avoid consuming `buf.build/...:main` immediately after merge).
  - Proto CI now triggers when proto distribution verification/sync tooling changes (e.g. `release/verify_*`, `scripts/ci/generate_bsr_artifacts.sh`, `sync_from_bsr.sh`) so the gates reflect tooling edits.
  - Go SDK verification retry logic was hardened so transient `go mod download` failures in the Buf proxy path actually retry under `set -e`.

### 2026-01-01

- `GitOps / Gate`: added diff-scoped suite gating (run only relevant checks) while keeping a single required-check-safe always-run gate job; removed duplicate `push` triggers from the called workflows to avoid post-merge re-runs.
- `GitOps / Gate`: added workflow linting + an ARC-first runner policy check to prevent new/modified CI workflows from hardcoding `runs-on: ubuntu-latest` (runner selection should be GitHub-config-driven).
- `CI Metrics (Weekly)`: added a scheduled snapshot workflow that reports queue time, runner occupancy, time-to-first-signal, and time-to-green using job timestamps (no `/timing` dependency).

### 2025-12-28

- `CI / Core Quality Gate`: gate JS/Python/Go SDK steps behind path-scoped change detection to reduce unnecessary installs/tests.
- `CD / Service Images`: compute a diff-based image build matrix and skip expensive prepare steps when no images are selected; enable cancel-in-progress for superseded runs.

### 2026-01-02

- PR-facing workflows: remove trigger-level `paths`/`paths-ignore` on `pull_request` to prevent “workflow skipped ⇒ required checks pending”; implement diff-based scoping internally plus a single always-run `Gate` job per workflow to preserve required-check correctness while skipping irrelevant suites.
- Concurrency: include `${{ github.workflow }}` in concurrency groups across CI/CD workflows to reduce accidental cross-workflow cancellation.
