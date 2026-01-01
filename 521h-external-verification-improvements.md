# 521h — External Verification Improvements Log

Track changes to external verification gates (test requirements, scaffold-marker enforcement, 430 integration-pack contract enforcement, CI wiring).

Baseline: `backlog/521f-external-verification-baseline.md`

---

## Log

Add new entries here when external verification behavior changes.

### 2026-01-01

- `GitOps / Gate`: added diff-scoped suite gating (run only relevant checks) while keeping a single required-check-safe always-run gate job; removed duplicate `push` triggers from the called workflows to avoid post-merge re-runs.
- `GitOps / Gate`: added workflow linting + an ARC-first runner policy check (reject `ubuntu-latest` in `.github/workflows/**`).
- `CI Metrics (Weekly)`: added a scheduled snapshot workflow that reports queue time, runner occupancy, time-to-first-signal, and time-to-green using job timestamps (no `/timing` dependency).

### 2025-12-28

- `CI / Core Quality Gate`: gate JS/Python/Go SDK steps behind path-scoped change detection to reduce unnecessary installs/tests.
- `CD / Service Images`: compute a diff-based image build matrix and skip expensive prepare steps when no images are selected; enable cancel-in-progress for superseded runs.
