---
title: 430 – Unified Test Infrastructure
status: active
owners:
  - platform
  - test-infra
created: 2025-12-01
updated: 2025-12-02
supersedes:
  - 371-e2e-playwright.md
  - 376-integration-test-dual-modes.md
---

## Summary

This backlog defines the **target state** for all automated tests in the codebase. It consolidates and supersedes backlogs 371 and 376.

**This document describes the end state only.** Transitional patterns, backward compatibility shims, fallbacks, and workarounds are explicitly forbidden. All implementations must conform to the target architecture.

> ⚠️ **Remote-first reminder:** Wherever this backlog mentions a “cluster” profile or k3d-based execution, substitute the shared AKS namespace (`ameide-dev`) reached via Telepresence per [435-remote-first-development.md](435-remote-first-development.md).

### Cluster + image authority model

- **Single Argo CD control plane:** A single Argo deployment in the `argocd` namespace reconciles every environment namespace (`ameide-dev`, `ameide-staging`, `ameide-prod`) using Projects for logical boundaries rather than multiple controllers.
- **GitHub as source-of-truth:** `ameide-gitops` defines the image repository/tag per environment; CI publishes artifacts to GitHub Container Registry (GHCR) and updates GitOps values. Tests must never override images outside Git.
- **Tilt + Telepresence only for inner loop:** Cluster test profiles run against the Argo baseline in AKS. Tilt-only releases (`*-tilt`) continue to exist for developer workflows but are not part of the unified test contract.

---

## Non-Negotiable Principles

### 1. Two Modes, No Exceptions

Every integration suite supports exactly two dependency profiles:

| Mode | Environment | Dependencies |
|------|-------------|--------------|
| `repo` | Local, DevContainer, CI pre-merge | In-memory stubs only |
| `cluster` | AKS (`ameide-dev` namespace for dev profiles, staging/prod clusters in CI) | Real Kubernetes services |

**Forbidden:**
- Hybrid modes ("partial mock")
- Environment-specific branches other than mock/cluster
- Conditional skips based on mode
- Graceful degradation when dependencies missing

### 2. Single Implementation, Dual Execution

Each test has **one implementation** that runs in both modes:

```typescript
// Target pattern
const client = mode === 'cluster'
  ? createLiveClient(requireEnv('GRPC_ADDRESS'))
  : createMockClient(fixtures);

it('lists repositories', async () => {
  const repos = await client.listRepositories();
  expect(repos.length).toBeGreaterThan(0);
});
```

**Forbidden:**
- Separate test files for mock vs cluster
- `test.skip.if(mode === 'mock')`
- Different assertions per mode
- `JEST_EXTRA_ARGS` to filter tests by mode (transitional only)

### 3. Fail Fast, Always

| Scenario | Required Behavior |
|----------|-------------------|
| Missing env var (cluster mode) | **Exit 1** with clear message |
| Missing fixture (mock mode) | **Exit 1** with clear message |
| Unknown mode value | **Exit 1** at startup |
| External service unavailable | **Fail test** (not skip) |

**Forbidden:**
- Silent skips
- Default/fallback env var values in cluster mode
- `try/catch` that swallows missing dependencies
- Empty test suites that "pass" with no assertions

### 4. Language Alignment

| Test Type | Language | Framework |
|-----------|----------|-----------|
| Integration | Same as service | Jest (TS), pytest (Python), go test (Go) |
| E2E | TypeScript only | Playwright only |

**Forbidden:**
- Cypress, Selenium, or other E2E frameworks
- Python/Go E2E tests
- Integration tests in different language than service

### 5. Canonical Folder Structure

```
services/<service>/
  __mocks__/                    # Mock implementations (required)
    index.ts                    # Public API
    client.ts                   # Mock client factory
    fixtures.ts                 # Typed fixture data
  __tests__/
    unit/                       # Pure unit tests
    integration/                # Dual-mode tests
      run_integration_tests.sh  # Entry point (required)
      helpers.ts                # Mode-aware factories
      *.test.ts
```

**Forbidden:**
- Mocks scattered in test files
- Missing `__mocks__/` directory for services with integration tests
- `__tests__/integration/` without `run_integration_tests.sh`
- Fixture data inline in tests

---

## Mode Contract

### Environment Variable (tests only)

```bash
INTEGRATION_MODE=repo|local|cluster
```

**Rules:**
- Default: `repo` (safe for local development)
- `local` is treated as non-cluster and is not used for new suites
- Unknown values: **Exit 1** (never fallback)
- Case-insensitive: `REPO`, `Repo`, `repo` all valid

**Scope rule:** `INTEGRATION_MODE` is a **test runner / harness variable only**. Application runtime (including `www_ameide_platform`) must not branch behavior on `INTEGRATION_MODE` or `NEXT_PUBLIC_INTEGRATION_MODE`.

### Service-Specific Inputs

Integration packs inevitably need additional inputs (service address, tenant IDs, seeded extension IDs, etc.). Rather than inventing ad-hoc names per service, adopt this convention immediately:

- **Prefix:** `SERVICEPREFIX_FIELD` where `SERVICEPREFIX` matches the directory name in screaming snake case (e.g., `extensions-runtime` → `EXTENSIONS_RUNTIME`).
- **Field vocabulary:** prefer `BASE_URL`, `GRPC_ADDRESS`, `TENANT_ID`, `ORG_ID`, `USER_ID`, `EXTENSION_ID`, `VERSION`, etc. Re-use existing terms from other packs before adding new ones.
- **Optional suffixes:** use `_DEV`, `_STAGING` etc. only if absolutely necessary; prefer a single variable whose value changes with `TEST_ENV`.

CI must be able to verify that every pack declares its dependencies. Add a light-weight lint in `tools/integration-runner` (or a GitHub workflow) that:

1. Parses each `run_integration_tests.sh`.
2. Extracts the `required_vars=(...)` array used in cluster mode.
3. Ensures variables obey the `SERVICEPREFIX_FIELD` convention and are documented in that script’s header comment.

This keeps the system manifest-free while still enforcing uniform names and tooling visibility.

### Runner Script Contract

Every `run_integration_tests.sh` must:

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Source mode helper (required)
source "tools/integration-runner/integration-mode.sh"

# 2. Resolve and export mode (required)
MODE="$(integration_mode)"
export INTEGRATION_MODE="${MODE}"

# 3. Mode-specific setup (required)
if [[ "${MODE}" != "cluster" ]]; then
  # Non-cluster mode: no env var requirements
  echo "[service] INTEGRATION_MODE=${MODE}"
else
  # Cluster mode: validate required env vars
  require_cluster_mode "${MODE}"
  for var in GRPC_ADDRESS DATABASE_URL; do
    [[ -z "${!var:-}" ]] && { echo "${var} required"; exit 1; }
  done
fi

# 4. Emit structured log (required)
printf '%s\n' '{"event":"integration_test_started","mode":"'"${MODE}"'"}'

# 5. Source JUnit helper and run tests
source "tools/integration-runner/junit-path.sh"
JUNIT_PATH="$(resolve_junit_path service-name)"
```

### CLI Enforcement (Implemented)

The repository now enforces the runner contract via:

```bash
ameide dev verify --repo-root .
```

The verifier applies to both:

- `services/<service>/__tests__/integration/` integration packs
- `primitives/<kind>/<name>/tests/` integration packs
- `tests/integration/<pack>/` repo-root integration packs (cross-primitive orchestration)

The verifier fails when a pack exists but does not satisfy:

- **Services**: `services/<service>/__mocks__/` exists
- **Services**: `__tests__/integration/run_integration_tests.sh` exists and is executable
- **Primitives**: `tests/run_integration_tests.sh` exists and is executable
- **Repo-root**: `tests/integration/<pack>/run_integration_tests.sh` exists and is executable
- Runner sources `tools/integration-runner/integration-mode.sh` and `tools/integration-runner/junit-path.sh`
- Runner resolves mode via `MODE="$(integration_mode)"` (or equivalent) and exports `INTEGRATION_MODE`
- Runner declares a `required_vars=(...)` (or `required_env_vars=(...)`) array and every entry (when present) starts with the expected prefix (`SERVICEPREFIX_FIELD`)
- Runner emits an `integration_test_started` structured log event
- Runner contains no inline `resolve_junit_path()` fallback
- Runner contains no `${VAR:-default}` default expansions
- Runner does not install dependencies (`pnpm install`, `pip install`, `uv sync`, `apt-get install`, etc.)
- Runner does not reference legacy toggles (`RUN_INTEGRATION_TESTS`, `*_USE_LIVE_DB`, `*_INTEGRATION_ENABLED`, `SKIP_*TESTS`)
- At least one test file exists under the pack (beyond the runner script)

---

## Mock Layer Architecture

### Structure

```
services/<service>/
  __mocks__/
    index.ts         # export { createMockClient, fixtures }
    client.ts        # createMockClient() implementation
    fixtures.ts      # typed fixture data
    handlers.ts      # MSW handlers (if HTTP service)
```

### gRPC Mock Pattern (TypeScript)

```typescript
// __mocks__/client.ts
import { createRouterTransport } from '@connectrpc/connect';
import { Service } from '@ameide/core-proto';
import { fixtures } from './fixtures';

export function createMockTransport() {
  return createRouterTransport(({ service }) => {
    service(Service, {
      list: () => fixtures.items,
      get: (req) => {
        const item = fixtures.items.find(i => i.id === req.id);
        if (!item) throw new ConnectError('Not found', Code.NotFound);
        return item;
      },
    });
  });
}
```

### Fixture Pattern

```typescript
// __mocks__/fixtures.ts
import { Item, ItemStatus } from '@ameide/core-proto';

export const fixtures = {
  items: [
    { id: 'item-001', name: 'Test', status: ItemStatus.ACTIVE },
  ] satisfies Item[],

  empty: { items: [] },

  errors: {
    notFound: { id: 'does-not-exist' },
  },
};
```

### Client Factory Pattern

```typescript
// __tests__/integration/helpers.ts
import { getIntegrationMode, requireEnv } from 'tools/integration-runner/ts/mode';
import { createMockTransport } from '../../__mocks__';
import { createConnectTransport } from '@connectrpc/connect-node';

export function createClient() {
  const mode = getIntegrationMode();

  return mode === 'cluster'
    ? createConnectTransport({ baseUrl: requireEnv('GRPC_ADDRESS') })
    : createMockTransport();
}
```

---

## E2E Tests

### Location

```
services/www_ameide_platform/
  features/<feature>/
    __tests__/
      e2e/
        *.spec.ts
```

### Cluster Mode Only

E2E runs against the real environment (AKS `ameide-dev` reached via Telepresence). We do not support "mock E2E" because it requires the application runtime to branch on test-only configuration, which drifts.

### Personas

| Persona | Role | Source |
|---------|------|--------|
| `owner` | Org admin | `playwright-int-tests-secrets` |
| `viewer` | Read-only | `playwright-int-tests-secrets` |
| `newmember` | Fresh user | `playwright-int-tests-secrets` |

---

## Artifacts

### JUnit Output

All tests emit JUnit XML:

```bash
source tools/integration-runner/junit-path.sh
JUNIT_PATH="$(resolve_junit_path service-name)"
# → /artifacts/service-name/junit.xml
```

### Playwright Artifacts

| Type | Path |
|------|------|
| Report | `/artifacts/e2e/playwright-report/` |
| Traces | `/artifacts/e2e/traces/` |
| Screenshots | `/artifacts/e2e/screenshots/` |
| Videos | `/artifacts/e2e/videos/` |

---

## Tilt Targets

| Target | Mode | Purpose |
|--------|------|---------|
| `test-all-repo` | repo | All integration tests, in-memory |
| `test-all-cluster` | cluster | All integration tests, AKS (`ameide-dev`) |
| `e2e-playwright-run` | cluster | Playwright E2E suite |
| `integration-<service>` | cluster | Per-service integration job |

---

## CI Strategy

### Pre-Merge (Gates PR)

```yaml
test-integration-mock:
  mode: mock
  timeout: 10m
  required: true
```

### Post-Merge (Nightly)

```yaml
test-integration-cluster:
  mode: cluster
  timeout: 30m

test-e2e-playwright:
  mode: cluster
  timeout: 45m
```

---

## Forbidden Patterns

### Transitional Shims (Remove All)

| Pattern | Why Forbidden |
|---------|---------------|
| `JEST_EXTRA_ARGS` test filtering by mode | Tests must run same in both modes |
| `${VAR:-default}` in cluster mode | Cluster must require all vars |
| `resolve_junit_path()` inline fallback | Use helper consistently |
| Multiple URL env var checks | Single canonical var per service |

### Legacy Env Vars (Never Use)

| Variable | Replacement |
|----------|-------------|
| `RUN_INTEGRATION_TESTS` | `INTEGRATION_MODE=cluster` |
| `*_USE_LIVE_DB` | `INTEGRATION_MODE=cluster` |
| `*_INTEGRATION_ENABLED` | `INTEGRATION_MODE=cluster` |
| `SKIP_*_TESTS` | Remove; fail if dependencies missing |

### Anti-Patterns

```typescript
// FORBIDDEN: Conditional skip
test.skipIf(mode !== 'cluster')('requires cluster', () => {});

// FORBIDDEN: Different behavior per mode
const expected = mode !== 'cluster' ? 'mock-response' : 'real-response';

// FORBIDDEN: Swallowing errors
try { await client.call(); } catch { /* ignore in mock */ }

// FORBIDDEN: Empty test file
describe('cluster-only tests', () => {
  // nothing runs in mock mode
});
```

---

## Observability

### Structured Logging

Every test run emits:

```json
{
  "event": "integration_test_started",
  "service": "platform",
  "mode": "cluster",
  "env": {
    "INTEGRATION_MODE": "cluster",
    "GRPC_ADDRESS": "envoy.ameide.svc:443"
  }
}
```

### Metrics

| Metric | Labels |
|--------|--------|
| `integration_test_duration_seconds` | `service`, `mode`, `result` |
| `integration_test_result_total` | `service`, `mode`, `result` |

---

## Cross-References

| Backlog | Relationship |
|---------|--------------|
| [362-unified-secret-guardrails](./362-unified-secret-guardrails.md) | Test credentials via ExternalSecrets |
| [428-onboarding](./428-onboarding.md) | E2E requirements for onboarding |
| [484-ameide-cli](./484-ameide-cli.md) | CLI `verify` command implements test modes, scaffolds 430-compliant test structure |

### Superseded

| Backlog | Status |
|---------|--------|
| [371-e2e-playwright](./371-e2e-playwright.md) | Deprecated |
| [376-integration-test-dual-modes](./376-integration-test-dual-modes.md) | Deprecated |

---

## Decision Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2025-12-01 | Consolidate 371 + 376 | Single source of truth |
| 2025-12-01 | E2E = Playwright only | Consistent tooling |
| 2025-12-01 | Default mock mode | Safe for local dev |
| 2025-12-01 | No silent skips | Fail fast exposes gaps |
| 2025-12-02 | Target-state only | Remove transitional cruft |
| 2025-12-02 | Forbid shims explicitly | Clear migration pressure |
