# 468 - Repo-wide Testing Front Door

**Status:** Active
**Created:** 2025-12-22
**Updated:** 2025-12-22

If you run one thing before opening/merging a PR, run the same gate CI runs:

```bash
./scripts/ci/run_core_quality_local.sh
```

This mirrors `.github/workflows/ci-core-quality.yml` and writes artifacts under `.test-results/` and `.coverage/`.

## Agent inner loop (one command)

For an AI agent (or an engineer) iterating locally and needing the fastest, most consistent “did I break anything?” signal with minimal cognitive overhead:

```bash
ameide dev inner-loop-test
```

This is intentionally **not** the full PR gate and does not do image builds/publishing or GitOps work. It runs **unit → integration → e2e** and writes artifacts under `artifacts/agent-ci/<timestamp>/`.

## Fast paths

```bash
# Reuse existing deps (skip pnpm install + uv sync)
SKIP_INSTALL=1 ./scripts/ci/run_core_quality_local.sh

# Skip Playwright e2e
SKIP_E2E=1 ./scripts/ci/run_core_quality_local.sh

# Fail on mypy errors (CI allows mypy to be soft)
ALLOW_MYPY_FAIL=0 ./scripts/ci/run_core_quality_local.sh
```

## Integration packs (unified contract / 430)

Integration packs are **owned by the component they test** and live alongside it:

- `services/<service>/__tests__/integration/run_integration_tests.sh`
- `packages/<package>/__tests__/integration/run_integration_tests.sh` (when present)
- `primitives/<kind>/<name>/tests/run_integration_tests.sh` (when present)
- `capabilities/<capability>/__tests__/integration/run_integration_tests.sh` (when present; see `591-capabilities-tests.md`)

They all honor:

- `INTEGRATION_MODE=repo|local|cluster` (default: `repo`)
- **Fail fast** on missing env vars in `cluster` mode

Run one pack (repo mode / mock-first):

```bash
INTEGRATION_MODE=repo bash services/chat/__tests__/integration/run_integration_tests.sh
```

Run all service packs (repo mode):

```bash
INTEGRATION_MODE=repo bash -lc 'for f in services/*/__tests__/integration/run_integration_tests.sh; do echo "==> $f"; bash "$f"; done'
```

For target-state rules (folder structure, `__mocks__/`, runner script contract), see `430-unified-test-infrastructure.md`.

## Operators (control plane)

Operator tests are intentionally separate from the `INTEGRATION_MODE` contract:

```bash
# Go unit tests for operator packages
make test-operators

# Fast reconciler correctness (envtest)
make test-operators-envtest

# Cluster acceptance (kind + Helm + KUTTL)
make test-acceptance
```

Acceptance harness details live in `tests/acceptance/README.md`.

## CI map

- Core quality gate: `.github/workflows/ci-core-quality.yml` (local: `./scripts/ci/run_core_quality_local.sh`)
- Integration packs: `.github/workflows/ci-integration-packs.yml`
- Operators envtest: `.github/workflows/ci-operators-envtest.yml`
- Operators acceptance: `.github/workflows/ci-operators-acceptance.yml`
