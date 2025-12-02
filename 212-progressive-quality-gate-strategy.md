# 212: Progressive Quality Gate Strategy

**Status:** Draft
**Priority:** High
**Effort:** 3-4 days
**Impact:** Developer Experience, CI/CD Performance

## Problem Statement

The current quality gate strategy has several performance and usability issues:

1. **No local validation**: Developers discover issues only after pushing to CI
2. **Sequential CI execution**: All checks run in a single job (~10-15 minutes)
3. **Redundant verification**: Quality checks duplicated across workflows
4. **No test tier strategy**: All 1,506 Python tests + full JS suite run on every push
5. **Slow feedback loop**: PRs wait 10-15 minutes for basic lint/type errors

**Current State:**
- Single workflows: [`.github/workflows/ci-core-quality.yml`](../.github/workflows/ci-core-quality.yml)
- Runs on all `push` and `pull_request` events
- Sequential execution: setup → lint → typecheck → tests
- No pre-commit or pre-push validation
- Same checks run for feature branches and main branch

## Goals

1. **Fast local feedback**: Catch formatting/lint errors before commit (< 10s)
2. **Pre-push validation**: Run unit tests before pushing (< 60s)
3. **Rapid PR feedback**: Parallel CI checks complete in 3-5 minutes
4. **Comprehensive main validation**: Full test suite + E2E on merge (8-12 minutes)
5. **No wasted CI time**: Skip irrelevant jobs based on changed files

## Proposed Solution: 5-Tier Quality Gate

### Tier 1: Pre-commit Hook (< 10 seconds)
**Goal:** Auto-fix formatting, catch obvious errors
**Trigger:** `git commit`
**Tools:** `pre-commit` framework

**Checks:**
- ✅ Ruff format (Python) - auto-fix
- ✅ Prettier (JS/TS/JSON/YAML) - auto-fix
- ✅ Trailing whitespace, EOF newlines
- ✅ Ruff check on staged Python files only
- ✅ ESLint on staged JS/TS files only
- ❌ No type checking, no tests, no builds

**Implementation:**
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.9
    hooks:
      - id: ruff-format
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]

  - repo: https://github.com/pre-commit/mirrors-prettier
    rev: v4.0.0-alpha.8
    hooks:
      - id: prettier
        types_or: [javascript, jsx, ts, tsx, json, yaml, markdown]

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json

  - repo: local
    hooks:
      - id: eslint
        name: eslint (pnpm exec)
        entry: pnpm exec eslint
        language: system
        types_or: [javascript, jsx, ts, tsx]
        args: [--max-warnings=0, --cache]
      - id: pre-push-quality-gate
        name: pre-push quality gate
        entry: scripts/git-hooks/pre-push
        language: script
        stages: [pre-push]
```

**Benefits:**
- Eliminates "fix formatting" commits
- Immediate feedback (< 10s)
- Reduces CI failures from lint errors

---

### Tier 2: Pre-push Hook (< 60 seconds)
**Goal:** Validate code quality before pushing to remote
**Trigger:** `git push`
**Tools:** Version-controlled pre-push hook (installed via `pre-commit`'s `pre-push` stage)

**Checks:**
- ✅ Type checking on affected files:
  - `mypy` on changed Python packages
  - `tsc --noEmit` on changed TS packages
- ✅ Unit tests for affected code:
  - `pytest` targeted to src/tests touched in the branch (fast fail, max 3 errors)
  - `jest --findRelatedTests --bail` on changed JS/TS files
- ❌ No integration tests, no E2E, no builds

**Implementation:**
```bash
#!/usr/bin/env bash
# scripts/git-hooks/pre-push
set -euo pipefail

echo "Running pre-push checks..."

DEFAULT_BRANCH="${DEFAULT_BRANCH:-main}"
UPSTREAM_REF="$(git rev-parse --abbrev-ref --symbolic-full-name @{upstream} 2>/dev/null || true)"

if [[ -n "$UPSTREAM_REF" ]]; then
  BASE_COMMIT="$(git merge-base HEAD "$UPSTREAM_REF")"
else
  echo "Fetching origin/${DEFAULT_BRANCH} to establish diff base..."
  git fetch --quiet origin "${DEFAULT_BRANCH}"
  BASE_COMMIT="$(git merge-base HEAD "origin/${DEFAULT_BRANCH}")"
fi

CHANGED_FILES="$(git diff --name-only "${BASE_COMMIT}" HEAD)"

if [[ -z "${CHANGED_FILES}" ]]; then
  echo "No relevant changes detected; skipping pre-push checks."
  exit 0
fi

mapfile -t PY_FILES < <(printf '%s\n' "${CHANGED_FILES}" | grep -E '\.py$' || true)
mapfile -t TS_FILES < <(printf '%s\n' "${CHANGED_FILES}" | grep -E '\.(ts|tsx)$' || true)
mapfile -t JS_FILES < <(printf '%s\n' "${CHANGED_FILES}" | grep -E '\.(js|jsx|ts|tsx)$' || true)

if (( ${#PY_FILES[@]} )); then
  echo "Type checking Python (targeted modules)..."
  mapfile -t PY_TARGETS < <(python - <<'PY'
import sys
from pathlib import Path
targets = {str(Path(path).parent) for path in sys.argv[1:] if path.endswith(".py")}
print("\n".join(sorted(targets)))
PY
    "${PY_FILES[@]}")
  if (( ${#PY_TARGETS[@]} )); then
    uvx mypy --config-file mypy.ini "${PY_TARGETS[@]}"
  fi

  echo "Running Python unit tests (changed scope)..."
  mapfile -t PY_TEST_TARGETS < <(python - <<'PY'
import sys
from pathlib import Path
def candidate(raw: str) -> str | None:
    path = Path(raw)
    if "tests" in path.parts:
        return str(path)
    guess = path.parent / "tests"
    return str(guess) if guess.exists() else None
targets: list[str] = []
seen: set[str] = set()
for raw in sys.argv[1:]:
    value = candidate(raw)
    if value and value not in seen:
        targets.append(value)
        seen.add(value)
if not targets:
    print("tests/unit")
else:
    print("\n".join(targets))
PY
    "${PY_FILES[@]}")
  uv run pytest --maxfail=3 -q "${PY_TEST_TARGETS[@]}"
fi

if (( ${#TS_FILES[@]} )); then
  echo "Type checking TypeScript..."
  pnpm exec tsc --noEmit --incremental --pretty false
fi

if (( ${#JS_FILES[@]} )); then
  echo "Running JS unit tests (related tests)..."
  pnpm exec jest --ci --bail --findRelatedTests --passWithNoTests "${JS_FILES[@]}"
fi
```

The hook is tracked in `scripts/git-hooks/pre-push` and executed through `pre-commit`'s managed `pre-push` stage, so onboarding simply requires `pre-commit install --hook-type pre-push`. The lightweight Python helper prefers nearby `tests/` directories and falls back to a shared smoke target, keeping the runtime tight while still exercising the code that changed.

**Benefits:**
- Prevents pushing broken code
- Fast feedback (30-60s typical)
- Reduces CI load from obvious failures

---

### Tier 3: PR Quality Gate (3-5 minutes)
**Goal:** Comprehensive validation with parallel execution
**Trigger:** `pull_request` events
**Workflow:** `.github/workflows/pr-quality-gate.yml`

**Job Topology (change filter + up to 6 parallel jobs):**

```yaml
jobs:
  determine-changes:
    runs-on: ubuntu-latest
    outputs:
      python: ${{ steps.filter.outputs.python }}
      javascript: ${{ steps.filter.outputs.javascript }}
      typescript: ${{ steps.filter.outputs.typescript }}
    steps:
      - uses: actions/checkout@v5
        with:
          fetch-depth: 0
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            python:
              - 'packages/**/*.py'
              - 'services/**/*.py'
              - 'pyproject.toml'
              - 'mypy.ini'
              - '.ruff.toml'
            javascript:
              - '**/*.js'
              - '**/*.jsx'
              - 'package.json'
            typescript:
              - '**/*.ts'
              - '**/*.tsx'
              - 'package.json'

  lint-python:
    needs: determine-changes
    runs-on: ubuntu-latest
    steps:
      - name: Skip (no Python changes)
        if: needs.determine-changes.outputs.python != 'true'
        run: echo "No Python changes detected; skipping lint."
      - uses: actions/checkout@v5
        if: needs.determine-changes.outputs.python == 'true'
      - uses: astral-sh/setup-uv@v7
        if: needs.determine-changes.outputs.python == 'true'
        with:
          enable-cache: true
      - run: uvx ruff check .
        if: needs.determine-changes.outputs.python == 'true'
    # Duration: ~35s when enabled

  lint-js:
    needs: determine-changes
    runs-on: ubuntu-latest
    steps:
      - name: Skip (no JS/TS changes)
        if: >
          needs.determine-changes.outputs.javascript != 'true' &&
          needs.determine-changes.outputs.typescript != 'true'
        run: echo "No JS/TS changes detected; skipping lint."
      - uses: actions/checkout@v5
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
      - uses: pnpm/action-setup@v4
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
      - uses: actions/setup-node@v5
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --no-frozen-lockfile
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
      - run: pnpm exec eslint . --ext .js,.jsx,.ts,.tsx --max-warnings=0 --cache
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
    # Duration: ~55s when enabled

  typecheck-python:
    needs: determine-changes
    runs-on: ubuntu-latest
    steps:
      - name: Skip (no Python changes)
        if: needs.determine-changes.outputs.python != 'true'
        run: echo "No Python changes detected; skipping mypy."
      - uses: actions/checkout@v5
        if: needs.determine-changes.outputs.python == 'true'
      - uses: astral-sh/setup-uv@v7
        if: needs.determine-changes.outputs.python == 'true'
      - run: uvx mypy --config-file mypy.ini packages
        if: needs.determine-changes.outputs.python == 'true'
    # Duration: ~2min when enabled

  typecheck-ts:
    needs: determine-changes
    runs-on: ubuntu-latest
    steps:
      - name: Skip (no TS changes)
        if: needs.determine-changes.outputs.typescript != 'true'
        run: echo "No TypeScript changes detected; skipping typecheck."
      - uses: actions/checkout@v5
        if: needs.determine-changes.outputs.typescript == 'true'
      - uses: pnpm/action-setup@v4
        if: needs.determine-changes.outputs.typescript == 'true'
      - uses: actions/setup-node@v5
        if: needs.determine-changes.outputs.typescript == 'true'
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --no-frozen-lockfile
        if: needs.determine-changes.outputs.typescript == 'true'
      - run: pnpm -r --if-present run typecheck
        if: needs.determine-changes.outputs.typescript == 'true'
    # Duration: ~1-2min when enabled

  test-python:
    needs: determine-changes
    runs-on: ubuntu-latest-4-cores
    steps:
      - name: Skip (no Python changes)
        if: needs.determine-changes.outputs.python != 'true'
        run: echo "No Python changes detected; skipping package tests."
      - uses: actions/checkout@v5
        if: needs.determine-changes.outputs.python == 'true'
      - uses: astral-sh/setup-uv@v7
        if: needs.determine-changes.outputs.python == 'true'
        with:
          enable-cache: true
      - run: uv sync --group test
        if: needs.determine-changes.outputs.python == 'true'
      - run: |
          uv pip install pytest-xdist
          uv run pytest -n auto packages/ \
            --junit-xml=.test-results/pytest-packages.xml \
            --cov=./ --cov-report=xml:.coverage/coverage-python.xml
        if: needs.determine-changes.outputs.python == 'true'
      - uses: codecov/codecov-action@v5
        if: needs.determine-changes.outputs.python == 'true'
        with:
          files: .coverage/coverage-python.xml
    # Duration: ~3-4min (parallel execution) when enabled

  test-js:
    needs: determine-changes
    runs-on: ubuntu-latest-4-cores
    steps:
      - name: Skip (no JS/TS changes)
        if: >
          needs.determine-changes.outputs.javascript != 'true' &&
          needs.determine-changes.outputs.typescript != 'true'
        run: echo "No JS/TS changes detected; skipping tests."
      - uses: actions/checkout@v5
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
      - uses: pnpm/action-setup@v4
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
      - uses: actions/setup-node@v5
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --no-frozen-lockfile
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
      - run: |
          pnpm --filter @ameide/core-proto run build
          pnpm --filter @ameideio/ameide-sdk-ts run build
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
      # www_ameide_platform tests
      - run: |
          pnpm -C services/www_ameide_platform run test:unit -- \
            --ci --coverage --coverageReporters=lcov
          pnpm -C services/www_ameide_platform run test:integration -- \
            --ci --coverage=false
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
      # ameide_sdk_ts tests
      - run: |
          pnpm -C packages/ameide_sdk_ts run test --ci --coverage
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
      - uses: codecov/codecov-action@v5
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
        with:
          files: |
            .coverage/www_ameide_platform/lcov.info
            .coverage/ameide_sdk_ts/lcov.info
    # Duration: ~2-3min when enabled
```

**Path Filters (optimize for changed files):**
```yaml
on:
  pull_request:
    paths:
      - '**.py'
      - '**.js'
      - '**.ts'
      - '**.tsx'
      - 'pyproject.toml'
      - 'package.json'

jobs:
  determine-changes:
    runs-on: ubuntu-latest
    outputs:
      python: ${{ steps.filter.outputs.python }}
      javascript: ${{ steps.filter.outputs.javascript }}
      typescript: ${{ steps.filter.outputs.typescript }}
    steps:
      - uses: actions/checkout@v5
        with:
          fetch-depth: 0
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            python:
              - 'packages/**/*.py'
              - 'services/**/*.py'
              - 'pyproject.toml'
            javascript:
              - '**/*.js'
              - 'package.json'
            typescript:
              - '**/*.ts'
              - '**/*.tsx'
              - 'package.json'
```

Each job executes a cheap "skip" step when its language is untouched, so required checks still report `success` while the expensive steps short-circuit.

**What runs:**
- ✅ All linting (Python + JS)
- ✅ All type checking
- ✅ **Package tests only** (1,506 Python tests)
- ✅ Frontend unit + integration tests
- ✅ Coverage collection
- ❌ No service tests (14 files)
- ❌ No E2E tests
- ❌ No Docker builds

**Branch Protection:**
- Require the lint/typecheck/test jobs below to pass before merge
- No stale reviews
- Require linear history

**Benefits:**
- 60-70% faster than current (3-5min vs 10-15min)
- Parallel execution maximizes GitHub Actions resources
- Path filters skip irrelevant jobs
- Coverage still collected and uploaded

---

### Tier 4: Main Branch Gate (8-12 minutes)
**Goal:** Final comprehensive validation on merge to main
**Trigger:** `push` to `main` branch
**Workflow:** `.github/workflows/main-quality-gate.yml`

**Jobs:**
All Tier 3 jobs PLUS:

```yaml
jobs:
  test-python-services:
    runs-on: ubuntu-latest
    needs: [test-python]  # Run after package tests pass
    steps:
      - uses: actions/checkout@v5
      - uses: astral-sh/setup-uv@v7
        with:
          enable-cache: true
      - run: uv sync --all-packages
      - run: |
          uv run pytest services/ \
            --junit-xml=.test-results/pytest-services.xml
    # Duration: ~2-3min
    # Runs: 14 service test files

  test-e2e:
    runs-on: ubuntu-latest-4-cores
    needs: [test-js]
    steps:
      - uses: actions/checkout@v5
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v5
      - uses: actions/cache@v4
        with:
          path: ~/.cache/ms-playwright
          key: playwright-${{ runner.os }}-${{ hashFiles('**/package.json') }}
      - run: pnpm install --no-frozen-lockfile
      - run: pnpm -C services/www_ameide_platform exec playwright install --with-deps
      - run: |
          pnpm -C services/www_ameide_platform run test:e2e:auth
          pnpm -C services/www_ameide_platform run test:e2e:archimate
          pnpm -C services/www_ameide_platform run test:e2e:bpmn
          pnpm -C services/www_ameide_platform run test:e2e:threads
    # Duration: ~5-7min

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: aquasecurity/trivy-action@0.33.1
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
      - uses: github/codeql-action/upload-sarif@v4
        with:
          sarif_file: trivy-results.sarif
    # Duration: ~1-2min

  validate-docker-builds:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [
          www-ameide,
          www_ameide_platform,
          platform,
          graph,
          threads,
          agents,
          agents_runtime,
          inference,
          inference_gateway,
          workflows,
          workflows_runtime
        ]
    steps:
      - uses: actions/checkout@v5
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v6
        with:
          context: .
          file: services/${{ matrix.service }}/Dockerfile
          push: false
          cache-from: type=gha
          cache-to: type=gha,mode=max
    # Duration: ~5-8min (parallel)
```

**What runs:**
- ✅ Everything from Tier 3
- ✅ Service-level integration tests
- ✅ End-to-end Playwright tests
- ✅ Security vulnerability scanning
- ✅ Docker build validation (no push)
- ❌ No image push to registry (separate workflows)

**Benefits:**
- High confidence before merge
- Catches integration issues
- Validates Docker builds work
- Security scanning on every main commit

---

### Tier 5: Release Pipeline (15-20 minutes)
**Goal:** Build, scan, and deploy production images
**Trigger:** Git tags (`v*`) or manual dispatch
**Workflow:** Existing `.github/workflows/cd-service-images.yml`

**Jobs:**
1. **verify** (subset of Tier 3 - quick validation)
2. **build** (matrix of 9 services)
   - Build Docker images
   - Push to ACR registry with SHA tags
   - Comprehensive Trivy scanning
   - Upload SARIF results

**Changes needed:**
- Remove redundant verification (rely on main branch gate)
- Add image signing with Cosign
- Update deployment manifests with new digests

**Benefits:**
- Fast path to production (verification already done)
- Immutable image tags (SHA256 digests)
- Security scanning results uploaded to GitHub Security

---

## Test Organization Summary

### Python Tests
```
Total: 1,506 test files in packages/ + 14 in services/

Distribution:
  - Pre-push: pytest targeted modules/directories (maxfail=3) from changed files
  - PR CI: pytest -n auto packages/ (1,506 files, parallel)
  - Main CI: pytest -n auto packages/ + pytest services/
  - Release: Quick smoke tests only
```

### JavaScript Tests
```
Total: 25+ test directories in www_ameide_platform

Distribution:
  - Pre-push: jest --findRelatedTests --bail
  - PR CI: Unit (with coverage) + Integration (no coverage)
  - Main CI: Full suite + E2E (auth, archimate, bpmn, threads)
  - Release: Smoke tests only
```

### Coverage Strategy
```
- Pre-commit: No coverage
- Pre-push: No coverage
- PR CI: Full coverage, uploaded to Codecov
- Main CI: Full coverage with threshold enforcement
- Release: No coverage (already validated)
```

---

## Performance Comparison

| Workflow | Current | Optimized | Speedup |
|----------|---------|-----------|---------|
| **Pre-commit** | N/A | 5-10s | ∞ (new) |
| **Pre-push** | N/A | 30-60s | ∞ (new) |
| **PR Checks** | 10-15min | 3-5min | **66% faster** |
| **Main Merge** | 10-15min | 8-12min | 20% faster |
| **Release** | 20-25min | 15-20min | 25% faster |

**Developer Impact:**
- Discover lint/format errors in < 10s (not 10min)
- PR feedback in 3-5min instead of 10-15min
- Reduced CI queue time from fewer failures
- Higher confidence with local validation

---

## Implementation Plan

### Phase 1: Pre-commit Setup (Day 1)
- [ ] Install pre-commit framework
- [ ] Create `.pre-commit-config.yaml`
- [ ] Test with sample commits
- [ ] Document in CLAUDE.md
- [ ] Add installation to onboarding guide

**Files to create:**
- `.pre-commit-config.yaml`
- `.git/hooks/pre-commit` (auto-generated)

**Testing:**
- Commit with formatting errors → auto-fixed
- Commit with lint errors → blocked with helpful message

---

### Phase 2: Pre-push Hook (Day 1)
- [ ] Create pre-push hook script under `scripts/git-hooks/pre-push`
- [ ] Wire hook into `pre-commit` with `stages: [pre-push]`
- [ ] Test with failing unit tests
- [ ] Document in CLAUDE.md

**Files to create:**
- `scripts/git-hooks/pre-push`
- `.pre-commit-config.yaml` entry for `pre-push` stage

**Testing:**
- Run `pre-commit install --hook-type pre-push`
- Push with failing tests → blocked
- Push with passing tests → allowed

---

### Phase 3: Split PR Workflow (Day 2)
- [ ] Create `.github/workflows/pr-quality-gate.yml`
- [ ] Implement parallel job matrix
- [ ] Add `determine-changes` job with `dorny/paths-filter` for optimization
- [ ] Test with sample PR

**Files to create:**
- `.github/workflows/pr-quality-gate.yml`

**Files to modify:**
- `.github/workflows/ci-core-quality.yml` → rename to `main-quality-gate.yml`

**Testing:**
- Create PR with Python changes → JS jobs skip
- Create PR with JS changes → Python jobs skip
- Verify parallel execution in Actions UI

---

### Phase 4: Optimize Test Execution (Day 2)
- [ ] Add `pytest-xdist` to dev dependencies
- [ ] Update pytest commands to use `-n auto`
- [ ] Configure jest for optimal CI performance
- [ ] Verify coverage still collected correctly

**Files to modify:**
- `pyproject.toml` - add pytest-xdist
- `.github/workflows/pr-quality-gate.yml` - update pytest commands
- `.github/workflows/main-quality-gate.yml` - update pytest commands

**Testing:**
- Run pytest locally with `-n auto`
- Verify coverage reports complete
- Check test duration improvement

---

### Phase 5: Main Branch Workflow (Day 3)
- [ ] Update `main-quality-gate.yml` with additional jobs
- [ ] Add service tests job
- [ ] Add E2E tests job
- [ ] Add security scanning job
- [ ] Add Docker build validation

**Files to modify:**
- `.github/workflows/main-quality-gate.yml`

**Testing:**
- Push to main → full suite runs
- Verify E2E tests execute
- Check Docker builds succeed

---

### Phase 6: Branch Protection (Day 3)
- [ ] Configure GitHub branch protection for `main`
- [ ] Require PR reviews
- [ ] Require status checks to pass
- [ ] Require linear history
- [ ] Document protection rules

**GitHub Settings:**
```yaml
Protection rules for main:
  - Require pull request reviews: 1
  - Require status checks:
    - lint-python
    - lint-js
    - typecheck-python
    - typecheck-ts
    - test-python
    - test-js
  - Require branches to be up to date
  - Require linear history
  - Do not allow bypassing
```

---

### Phase 7: Documentation & Training (Day 4)
- [ ] Update CLAUDE.md with new workflows
- [ ] Create CONTRIBUTING.md
- [ ] Document pre-commit setup
- [ ] Add troubleshooting guide
- [ ] Team training session

**Files to create/update:**
- `CONTRIBUTING.md`
- `CLAUDE.md` (Development Workflow section)
- `.github/PULL_REQUEST_TEMPLATE.md`

---

## Rollout Strategy

### Week 1: Soft Launch
1. Enable pre-commit hooks (optional, documented)
2. Run new PR workflows in parallel with old (don't enforce)
3. Monitor performance and failure rates
4. Gather feedback from team

### Week 2: Gradual Enforcement
1. Make pre-commit hooks required for new developers
2. Enable branch protection with new status checks
3. Deprecate old quality gate workflows
4. Continue monitoring

### Week 3: Full Migration
1. Remove old workflows
2. Make pre-commit hooks mandatory
3. Document lessons learned
4. Measure performance improvements

---

## Success Metrics

**Performance Targets:**
- [ ] Pre-commit hooks run in < 10s
- [ ] PR checks complete in < 5min (90th percentile)
- [ ] Main checks complete in < 12min (90th percentile)
- [ ] 50% reduction in "fix lint" commits
- [ ] 30% reduction in CI failures from preventable errors

**Developer Experience:**
- [ ] 90% of developers using pre-commit hooks
- [ ] Positive feedback on local validation
- [ ] Reduced "waiting for CI" complaints
- [ ] Faster PR merge times

**Cost Savings:**
- [ ] 40-50% reduction in GitHub Actions minutes
- [ ] Fewer wasted CI runs from simple errors
- [ ] Better resource utilization (parallel jobs)

---

## Risks & Mitigations

### Risk: Pre-commit hooks too slow
**Mitigation:** Only run on staged files, provide opt-out for large commits

### Risk: False positives in pre-push
**Mitigation:** Provide `--no-verify` escape hatch, document in error messages

### Risk: Path filters skip critical tests
**Mitigation:** Always run full suite on main branch, careful filter design

### Risk: Parallel jobs cause race conditions
**Mitigation:** Ensure jobs are truly independent, use job dependencies where needed

### Risk: Team resistance to hooks
**Mitigation:** Make hooks helpful (auto-fix), provide clear error messages, gather feedback

---

## Dependencies

**External:**
- GitHub Actions (existing)
- pre-commit framework (new)
- pytest-xdist (new)

**Internal:**
- Existing test suites must pass
- Docker builds must be working
- Branch protection permissions

**Breaking Changes:**
- None (all changes are additive)

---

## Follow-up Items

1. **Test Impact Analysis**: Track which tests are affected by code changes
2. **Test Sharding**: Further optimize large test suites
3. **Self-hosted Runners**: Consider for even faster builds
4. **Cache Optimization**: Investigate remote caching for dependencies
5. **Flaky Test Detection**: Implement automatic retry and flake tracking

---

## References

- Current workflows: [`.github/workflows/ci-core-quality.yml`](../.github/workflows/ci-core-quality.yml)
- Build workflows: [`.github/workflows/cd-service-images.yml`](../.github/workflows/cd-service-images.yml)
- Python config: [`pyproject.toml`](../pyproject.toml)
- Ruff config: [`.ruff.toml`](../.ruff.toml)
- MyPy config: [`mypy.ini`](../mypy.ini)
- Jest config: [`services/www_ameide_platform/jest.config.js`](../services/www_ameide_platform/jest.config.js)

---

## Appendix: Example PR Workflow

```yaml
# .github/workflows/pr-quality-gate.yml
name: PR Quality Gate

on:
  pull_request:
    types: [opened, synchronize, reopened]

concurrency:
  group: ${{ github.workflows }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  determine-changes:
    name: Determine Changed Paths
    runs-on: ubuntu-latest
    outputs:
      python: ${{ steps.filter.outputs.python }}
      javascript: ${{ steps.filter.outputs.javascript }}
      typescript: ${{ steps.filter.outputs.typescript }}
    steps:
      - uses: actions/checkout@v5
        with:
          fetch-depth: 0
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            python:
              - 'packages/**/*.py'
              - 'services/**/*.py'
              - 'pyproject.toml'
              - 'mypy.ini'
              - '.ruff.toml'
            javascript:
              - '**/*.js'
              - '**/*.jsx'
              - 'package.json'
            typescript:
              - '**/*.ts'
              - '**/*.tsx'
              - 'package.json'

  lint-python:
    name: Lint Python
    runs-on: ubuntu-latest
    needs: determine-changes
    steps:
      - name: Skip lint
        if: needs.determine-changes.outputs.python != 'true'
        run: echo "No Python changes detected; skipping lint."
      - uses: actions/checkout@v5
        if: needs.determine-changes.outputs.python == 'true'
      - uses: astral-sh/setup-uv@v7
        if: needs.determine-changes.outputs.python == 'true'
        with:
          enable-cache: true
      - run: uvx ruff check .
        if: needs.determine-changes.outputs.python == 'true'

  lint-js:
    name: Lint JavaScript/TypeScript
    runs-on: ubuntu-latest
    needs: determine-changes
    steps:
      - name: Skip lint
        if: >
          needs.determine-changes.outputs.javascript != 'true' &&
          needs.determine-changes.outputs.typescript != 'true'
        run: echo "No JS/TS changes detected; skipping lint."
      - uses: actions/checkout@v5
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
      - uses: pnpm/action-setup@v4
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
      - uses: actions/setup-node@v5
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --no-frozen-lockfile
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
      - run: pnpm exec eslint . --ext .js,.jsx,.ts,.tsx --max-warnings=0 --cache
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'

  typecheck-ts:
    name: TypeScript Type Check
    runs-on: ubuntu-latest
    needs: determine-changes
    steps:
      - name: Skip typecheck
        if: needs.determine-changes.outputs.typescript != 'true'
        run: echo "No TypeScript changes detected; skipping typecheck."
      - uses: actions/checkout@v5
        if: needs.determine-changes.outputs.typescript == 'true'
      - uses: pnpm/action-setup@v4
        if: needs.determine-changes.outputs.typescript == 'true'
      - uses: actions/setup-node@v5
        if: needs.determine-changes.outputs.typescript == 'true'
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --no-frozen-lockfile
        if: needs.determine-changes.outputs.typescript == 'true'
      - run: pnpm run typecheck
        if: needs.determine-changes.outputs.typescript == 'true'

  typecheck-python:
    name: Python Type Check (MyPy)
    runs-on: ubuntu-latest
    needs: determine-changes
    steps:
      - name: Skip mypy
        if: needs.determine-changes.outputs.python != 'true'
        run: echo "No Python changes detected; skipping mypy."
      - uses: actions/checkout@v5
        if: needs.determine-changes.outputs.python == 'true'
      - uses: astral-sh/setup-uv@v7
        if: needs.determine-changes.outputs.python == 'true'
      - run: uvx mypy --config-file mypy.ini packages
        if: needs.determine-changes.outputs.python == 'true'

  test-python:
    name: Python Tests
    runs-on: ubuntu-latest-4-cores
    needs: determine-changes
    steps:
      - name: Skip tests
        if: needs.determine-changes.outputs.python != 'true'
        run: echo "No Python changes detected; skipping package tests."
      - uses: actions/checkout@v5
        if: needs.determine-changes.outputs.python == 'true'
      - uses: astral-sh/setup-uv@v7
        if: needs.determine-changes.outputs.python == 'true'
        with:
          enable-cache: true
      - run: uv sync --group test
        if: needs.determine-changes.outputs.python == 'true'
      - run: uv pip install pytest-xdist
        if: needs.determine-changes.outputs.python == 'true'
      - run: |
          uv run pytest -n auto packages/ \
            --junit-xml=.test-results/pytest.xml \
            --cov=./ --cov-report=xml:.coverage/coverage-python.xml
        if: needs.determine-changes.outputs.python == 'true'
      - uses: codecov/codecov-action@v5
        if: needs.determine-changes.outputs.python == 'true'
        with:
          files: .coverage/coverage-python.xml

  test-js:
    name: JavaScript/TypeScript Tests
    runs-on: ubuntu-latest-4-cores
    needs: determine-changes
    steps:
      - name: Skip tests
        if: >
          needs.determine-changes.outputs.javascript != 'true' &&
          needs.determine-changes.outputs.typescript != 'true'
        run: echo "No JS/TS changes detected; skipping tests."
      - uses: actions/checkout@v5
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
      - uses: pnpm/action-setup@v4
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
      - uses: actions/setup-node@v5
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --no-frozen-lockfile
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
      - run: |
          pnpm --filter @ameide/core-proto run build
          pnpm --filter @ameideio/ameide-sdk-ts run build
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
      - run: |
          pnpm -C services/www_ameide_platform run test:unit -- --ci --coverage
          pnpm -C services/www_ameide_platform run test:integration -- --ci --coverage=false
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
      - run: pnpm -C packages/ameide_sdk_ts run test --ci --coverage
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
      - uses: codecov/codecov-action@v5
        if: >
          needs.determine-changes.outputs.javascript == 'true' ||
          needs.determine-changes.outputs.typescript == 'true'
        with:
          files: |
            .coverage/www_ameide_platform/lcov.info
            .coverage/ameide_sdk_ts/lcov.info

  summary:
    name: Quality Gate Summary
    runs-on: ubuntu-latest
    needs: [determine-changes, lint-python, lint-js, typecheck-ts, typecheck-python, test-python, test-js]
    if: always()
    steps:
      - run: |
          echo "Quality gate results:"
          echo "Lint Python: ${{ needs.lint-python.result }}"
          echo "Lint JS: ${{ needs.lint-js.result }}"
          echo "TypeCheck TS: ${{ needs.typecheck-ts.result }}"
          echo "TypeCheck Python: ${{ needs.typecheck-python.result }}"
          echo "Test Python: ${{ needs.test-python.result }}"
          echo "Test JS: ${{ needs.test-js.result }}"
```

---

**Created:** 2025-01-13
**Last Updated:** 2025-01-13
**Author:** Claude (AI Assistant)
**Reviewers:** TBD
