# 521h — External Verification Improvements Log

Track changes to external verification gates (test requirements, scaffold-marker enforcement, 430 integration-pack contract enforcement, CI wiring).

Baseline: `backlog/521f-external-verification-baseline.md`

---

## Log

Add new entries here when external verification behavior changes.

### 2025-12-28

- `CI / Core Quality Gate`: gate JS/Python/Go SDK steps behind path-scoped change detection to reduce unnecessary installs/tests.
- `CD / Service Images`: compute a diff-based image build matrix and skip expensive prepare steps when no images are selected; enable cancel-in-progress for superseded runs.
