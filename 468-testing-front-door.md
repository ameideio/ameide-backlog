# 468 - Repo-wide Testing Front Door

**Status:** Active
**Created:** 2025-12-22
**Updated:** 2025-12-22

> **Update (2026-01): 430v2 contract**
>
> The normative contract is now:
> - `backlog/430-unified-test-infrastructure-v2-target.md`
>
> In particular, v2 removes:
> - `INTEGRATION_MODE`
> - per-component `run_integration_tests.sh` “packs” as the canonical execution path

If you run one thing before opening/merging a PR, run the same gate CI runs:

```bash
ameide ci test
```

This mirrors the 430v2 test contract phases in CI (Phase 0/1/2) and writes artifacts under `artifacts/agent-ci/`.

## Agent inner loop (one command)

For an AI agent (or an engineer) iterating locally and needing the fastest, most consistent “did I break anything?” signal with minimal cognitive overhead:

```bash
ameide dev inner-loop-test
```

This is intentionally **not** the full PR gate and does not do image builds/publishing or GitOps work. It runs **unit → integration → e2e** and writes artifacts under `artifacts/agent-ci/<timestamp>/`.

## Fast paths

Fast paths are owned by the CLI (caching/dependency checks) rather than shell flags.

## Integration packs (unified contract / 430)

The historical “integration pack” model (scripts + `INTEGRATION_MODE`) is **legacy**.

The v2 contract is:
- Phase 1 (Unit): local, pure
- Phase 2 (Integration): local, mocked/stubbed only
- Phase 3 (E2E): cluster only, Playwright only

Keep the pack section below for historical context only.

### Legacy (v1): integration packs + `INTEGRATION_MODE`

Integration packs (legacy) are **owned by the component they test** and live alongside it:

- `services/<service>/__tests__/integration/run_integration_tests.sh`
- `packages/<package>/__tests__/integration/run_integration_tests.sh` (when present)
- `primitives/<kind>/<name>/tests/run_integration_tests.sh` (when present)
- `capabilities/<capability>/__tests__/integration/run_integration_tests.sh` (when present; see `591-capabilities-tests.md`)

They all honored:

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

For the current target-state rules, see `backlog/430-unified-test-infrastructure-v2-target.md`.

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

- Core quality gate: `.github/workflows/ci-core-quality.yml` (tests: `ameide ci test`)
- Integration packs: `.github/workflows/ci-integration-packs.yml`
- Operators envtest: `.github/workflows/ci-operators-envtest.yml`
- Operators acceptance: `.github/workflows/ci-operators-acceptance.yml`
