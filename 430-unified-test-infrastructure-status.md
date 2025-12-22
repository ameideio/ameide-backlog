---
title: 430 – Unified Test Infrastructure Status
status: tracking
created: 2025-12-02
parent: 430-unified-test-infrastructure.md
---

## Overview

This document tracks implementation progress against the target state defined in [430-unified-test-infrastructure.md](./430-unified-test-infrastructure.md).

**Audit Date:** 2025-12-10

---

## Executive Summary

| Metric | Count | Percentage |
|--------|-------|------------|
| Total integration test suites | 18 | 100% |
| Using `integration-mode.sh` | 18 | 100% |
| Dual-mode compliant | 18 | 100% |
| Cluster-only (violations) | 0 | 0% |
| Services with `__mocks__/` | 15 | 100% |
| E2E cluster-only (by design) | 1 | 100% |

---

## Per-Service Status

### Legend

| Symbol | Meaning |
|--------|---------|
| **Compliant** | Meets 430 target state |
| **Partial** | Has mock mode but uses transitional shims |
| **Cluster-Only** | Requires cluster mode, no mock implementation |
| **Missing** | No `__mocks__/` directory |

---

### TypeScript Services

| Service | Mode Helper | Mock Mode | `__mocks__/` | Violations | Priority |
|---------|-------------|-----------|--------------|------------|----------|
| `platform` | Yes | **Compliant** | **Yes** | None | P1 |
| `graph` | Yes | **Compliant** | **Yes** | None | P1 |
| `www_ameide_platform` | Yes | **Compliant** | **Yes** | None | P1 |
| `www_ameide` | Yes | **Compliant** | **Yes** | None | P2 |
| `repository` | Yes | **Compliant** | **Yes** | None | P2 |
| `threads` | Yes | **Compliant** | **Yes** | None | P2 |
| `chat` | Yes | **Compliant** | **Yes** | None | P2 |
| `transformation` | Yes | **Compliant** | **Yes** | None | P3 |
| `workflows` | Yes | **Compliant** | **Yes** | None | P3 |

### Python Services

| Service | Mode Helper | Mock Mode | `__mocks__/` | Violations | Priority |
|---------|-------------|-----------|--------------|------------|----------|
| `inference` | Yes | **Compliant** | **Yes** | None | P2 |
| `agents_runtime` | Yes | **Compliant** | **Yes** | None | P2 |
| `workflows_runtime` | Yes | **Compliant** | **Yes** | None | P2 |
| `workflow_worker` | Yes | **Compliant** | **Yes** | None | P3 |

### Go Services

| Service | Mode Helper | Mock Mode | `__mocks__/` | Violations | Priority |
|---------|-------------|-----------|--------------|------------|----------|
| `agents` | Yes | **Compliant** | **Yes** | None | P2 |
| `inference_gateway` | Yes | **Compliant** | **Yes** | None | P3 |

### Go Packages

| Package | Mode Helper | Mock Mode | `__mocks__/` | Violations | Priority |
|---------|-------------|-----------|--------------|------------|----------|
| `ameide_sdk_go` | Yes | **Compliant** | N/A (Go pkg) | None | P2 |

### TypeScript Packages

| Package | Mode Helper | Mock Mode | `__mocks__/` | Violations | Priority |
|---------|-------------|-----------|--------------|------------|----------|
| `ameide_sdk_ts` | Yes | **Compliant** | N/A | None | - |

### Python Packages

| Package | Mode Helper | Mock Mode | `__mocks__/` | Violations | Priority |
|---------|-------------|-----------|--------------|------------|----------|
| `ameide_sdk_python` | Yes | **Compliant** | N/A | None | - |

### Build-Only Components

- `ameide_core_proto` has been removed from the integration suite roster. The Buf bundle now builds via `packages/ameide_core_proto/scripts/build_artifacts.sh` and no longer reports a faux dual-mode test. This keeps the integration metrics scoped to suites that exercise application behavior rather than code generation.

---

## Operators / Control Plane Testing (out-of-scope for 430 metrics)

This status doc (430) tracks **application integration suites** that follow the `INTEGRATION_MODE` contract (non-cluster vs cluster) and use `__mocks__/` where applicable. E2E is tracked here, but E2E runs **cluster-only**.

Primitive **operators** (Domain/Process/Agent/UISurface) are a different surface:

- They are **control-plane components** that reconcile Kubernetes resources and update CR `status.conditions`.
- Their “cluster truth” depends on Kubernetes controllers (Deployments, Jobs, GC, etc.), which is not represented by `INTEGRATION_MODE` or service `__mocks__/`.

### Current operator testing state

We treat operator testing as a two-layer acceptance model:

1. **Layer A (envtest):** fast, deterministic reconciler correctness (rendering/ownership/conditions logic).
2. **Layer B (kind + Helm + KUTTL):** real cluster acceptance (CRDs install, Helm deploy works, Deployments become Available, CRs reach `Ready=True`, negative cases behave correctly).

These are intentionally **not counted** in the “Total integration test suites” metric above, because they are not service integration suites and do not use `integration-mode.sh`.

## GitOps smoke tests (out-of-scope for 430 metrics)

In `ameide-gitops`, we also run ArgoCD **PostSync hook Jobs** (“smokes”) to validate **cluster truth after reconciliation** (CRDs ready, operators healthy, secrets present, data-plane dependencies ready, etc.).

Inventory: `backlog/588-smoke-tests-inventory.md`

Operational note: a **sync** re-runs hook Jobs; a refresh updates status only. For local testing (in `ameide-gitops`) we use `./scripts/argocd-force-sync.sh <app>`.

### Why we use kind (even if k3d exists)

We introduced **kind** for operator acceptance because the goal is a **portable, CI-friendly, deterministic** Kubernetes cluster for tests:

- **CI standardization:** kind is widely used and stable in GitHub Actions for ephemeral test clusters.
- **Upstream Kubernetes behavior:** kind runs upstream Kubernetes components; k3d runs k3s. For operator/CRD edge-cases, kind reduces “passes on k3s, fails elsewhere” risk.
- **No local registry requirement:** the acceptance suite uses `kind load docker-image` to inject locally-built operator images; k3d usually requires `k3d image import` or a registry workflow.
- **Remote-first compatibility:** this repo’s day-to-day dev model is AKS + Telepresence; any local k3d cluster is legacy/optional, while kind is used only as a self-contained test harness.

### Mapping to the 430 “dual-mode” intent

Although operators don’t use `INTEGRATION_MODE`, the intent is analogous:

- 430 “mock mode” ⇢ operator **envtest** (fast, deterministic, no external dependencies).
- 430 “cluster mode” ⇢ operator **kind acceptance** (real Kubernetes controllers + Helm install).

### Follow-ups (optional)

- Add a cluster-provider switch so the same KUTTL suites can run against an existing `k3d-ameide` cluster for developer convenience, while keeping kind as the CI default.

## Detailed Gaps by Service

### `platform` (P1)

**Status:** Compliant

**Notes:**
- `run_integration_tests.sh` runs full suite in both modes (no filtering)
- Cross-service test now dual-mode via mock clients; mock-mode run passes

---

### `graph` (P1)

**Status:** Compliant

**Notes:**
- Mode-aware handlers in helpers select mock vs. live graph service
- Mock mode passes without DB envs

---

### `www_ameide_platform` (P1)

**Status:** Compliant

**Notes:**
- Jest integration config exercises the full suite in both modes; tests select mock vs live clients in the harness layer.
- `getServerClient()` does not branch on `INTEGRATION_MODE`; runtime always uses the live transport.
- `tests/scripts/run-playwright-e2e.mjs` enforces the mode contract (defaulting to `repo`), injects deterministic credentials, and fail-fast validates base URLs per backlog 430.

---

### `repository` (P2)

**Status:** Compliant

**Notes:**
- `services/repository/__mocks__` exposes mock handlers plus fixture seeding so the entire metamodel stack runs in memory when `INTEGRATION_MODE` is not `cluster`.
- `createIntegrationHarness()` (helpers.ts) loads the same test cases in both modes by swapping the backing handlers/client only.
- Runner validates database env vars only when `MODE=cluster`, so local developers default to mock mode safely.

---

### `threads` (P2)

**Status:** Compliant

**Notes:**
- Added `services/threads/__mocks__/` with mock transport + fixtures
- Integration helpers now create mode-aware clients for append/send flows
- Runner supports mock + cluster modes with strict env validation
- Mock-mode suite passes without external services

---

### `chat` (P2)

**Status:** Compliant

**Notes:**
- `createThreadsClient()` pulls from `chat/__mocks__` whenever the mode helper returns `repo` or `local`, exposing the same API surface as the production gRPC service.
- Runner only enforces `CHAT_SERVICE_URL` et al in cluster mode; mock mode works with zero external dependencies.
- Fixtures ensure deterministic IDs, so the assertions in `threads.integration.test.ts` execute identically in both modes.

---

### `inference` (P2)

**Status:** Compliant

**Notes:**
- Added mock FastAPI + gRPC layer under `services/inference/__mocks__/` with fixtures and downstream service doubles.
- Integration helpers/conftest provide shared dual-mode clients; assertions run identically in mock/cluster.
- `run_integration_tests.sh` now defaults to `repo` mode and validates cluster env vars; non-cluster suite passes.

---

### `agents_runtime` (P2)

**Status:** Compliant

**Notes:**
- `services/agents_runtime/__mocks__` ships a full gRPC twin (agents + runtime services) plus MinIO stubs; pytest fixtures spin it up when `integration_mode()` resolves to `repo` or `local`.
- Runner enforces the MinIO + GRPC env vars only for cluster, while mock mode dumps structured log output and proceeds with in-memory transports.
- Tests share helpers that return mode-aware SDK clients, so the same assertions cover both transports without branching.

---

### `workflows_runtime` (P2)

**Status:** Compliant

**Notes:**
- `services/workflows_runtime/__mocks__/temporal.py` exposes `run_mock_platform_workflow`, letting the pytest helpers execute the Temporal workflows entirely in-process when `INTEGRATION_MODE` is not `cluster`.
- Cluster mode still verifies Temporal env vars and exercises the real worker via `Client.connect()`, but the test harness is identical.
- Runner emits the structured start log and only fails fast when cluster env vars are missing.

---

### `transformation` (P3)

**Status:** Compliant

**Notes:**
- The integration harness swaps between `createMockTransformationClient()` and the live service via `getServerClient()` based on `INTEGRATION_MODE`, ensuring single implementation / dual execution.
- Runner enforces database env vars only when `MODE=cluster`, so the default (`repo`) requires no backing Postgres.
- `services/transformation/__mocks__` contains typed fixtures plus handler reset utilities to keep tests deterministic.

---

### `workflow_worker` (P3)

**Status:** Compliant

**Notes:**
- Added `services/workflow_worker/__mocks__/temporal.py` plus shared helpers so Temporal workflows can execute entirely in-memory during mock runs while still capturing status updates.
- `run_integration_tests.sh` validates `TEMPORAL_*` env vars in cluster mode, emits structured logs, and defaults to mock mode for local developers.
- Integration tests rely on a single implementation that now switches transports via `run_platform_workflow`, keeping assertions identical in both modes.

---

### `ameide_sdk_go` (P2)

**Status:** Compliant

**Notes:**
- `TestGraphDualModeContract` now builds a `contractEnv` that instantiates either the mock server or the live cluster client while running the same assertions, so both transports exercise the identical repository contract.
- Runner enforces the dual-mode env contract and emits the structured start log; no transitional shims remain.
- Follow-up work shifts to broadening coverage (additional RPCs) rather than dual-mode plumbing.

---

## E2E Test Status

### Current State

| Feature | Spec Files | Mode | Status |
|---------|-----------|-----------|--------|
| `auth` | 2 | **Cluster** | Running |
| `navigation` | 5 | **Cluster** | Running |
| `graphs` | 1 | **Cluster** | Running |
| `onboarding` | 2 | **Cluster** | Running |
| `invitations` | 1 | **Cluster** | Running |

### Violations

1. **None** - E2E is cluster-only by design.

### Required Work

- None.

---

## `__mocks__/` Directory Status

### Existing (15 services)

| Service | Files | Quality |
|---------|-------|---------|
| `platform` | `index.ts`, `client.ts` | Good - full mock client |
| `graph` | `service.ts` | Good - mock handlers |
| `threads` | `client.ts`, `fixtures.ts`, `index.ts` | Good - mock transport |
| `www_ameide_platform` | `handlers.ts`, `next/`, `http.ts` | Full MSW + mock fetch stack |
| `agents` | `server.go` | Mock gRPC server (Go) |
| `inference_gateway` | `inference.go` | Mock inference backend (Go) |
| `inference` | `fixtures.py`, `http.py`, `grpc.py`, `websocket.py` | Dual-mode mock FastAPI + gRPC |
| `workflow_worker` | `temporal.py` | Temporal-free workflow harness |
| `repository` | `client.ts`, `fixtures.ts`, `index.ts` | Mock metamodel + repo services |
| `chat` | `client.ts`, `fixtures.ts` | Full ThreadsService mock transport |
| `transformation` | `client.ts`, `handlers.ts` | Mock client + service handlers |
| `agents_runtime` | `grpc.py`, `clients.py`, `fixtures.py` | Mock agents + runtime servers |
| `workflows` | `client.ts`, `fixtures.ts` | Workflows gRPC mock |
| `workflows_runtime` | `temporal.py`, `fixtures.py` | Mock Temporal harness |
| `www_ameide` | `index.ts`, `http.ts`, `fixtures.ts` | Mock fetch + server handlers |

### Missing

None. Every service-level integration suite now has a co-located `__mocks__/` directory.

---

## Runner Script Compliance

### Compliant Patterns (Good)

All 18 active runners:
- Source `tools/integration-runner/integration-mode.sh`
- Source `tools/integration-runner/junit-path.sh`
- Export `INTEGRATION_MODE`
- Emit structured JSON log

### Non-Compliant Patterns (Fix Required)

None. All runners follow the five-step contract from backlog 430.

---

## Prioritized Work Items

### P1 - Critical (Week 1-2)

| Task | Service | Effort |
|------|---------|--------|
| Remove `JEST_EXTRA_ARGS` from platform | `platform` | **Done** |
| Wire graph tests to mock layer | `graph` | **Done** |
| Consolidate www_ameide_platform URLs | `www_ameide_platform` | **Done** |
| Remove www_ameide_platform `testMatch` filtering | `www_ameide_platform` | **Done** |

### P2 - High (Week 3-4)

| Task | Service | Effort |
|------|---------|--------|
| Create `__mocks__/` for repository | `repository` | **Done** |
| Create `__mocks__/` for threads | `threads` | **Done** |
| Create `__mocks__/` for chat | `chat` | **Done** |
| Python mock layer for inference | `inference` | **Done** |
| Python mock layer for agents_runtime | `agents_runtime` | **Done** |
| Retire proto pseudo integration runner | `ameide_core_proto` | **Done** |
| Align dual-mode assertions for SDK Go | `ameide_sdk_go` | **Done** |

### P3 - Medium (Week 5-6)

| Task | Service | Effort |
|------|---------|--------|
| Create `__mocks__/` for transformation | `transformation` | **Done** |
| Python mock layer for workflows_runtime | `workflows_runtime` | **Done** |
| E2E cluster-only (no mock mode) | `www_ameide_platform` | **Done** |

---

## Metrics to Track

### Weekly Dashboard

| Metric | Current | Target |
|--------|---------|--------|
| Services with `__mocks__/` | 15/15 | 100% |
| Dual-mode compliant runners | 18/18 | 100% |
| `JEST_EXTRA_ARGS` usage | 0 | 0 |
| Cluster-only services | 0 | 0 |

### Definition of Done

A service is **430-compliant** when:
1. Has `__mocks__/` directory with typed fixtures
2. `run_integration_tests.sh` supports both modes
3. No `JEST_EXTRA_ARGS` or mode-based test filtering
4. All tests run identically in mock and cluster modes
5. Emits structured log on startup

---

## Appendix: File Locations

### Mode Helpers

```
tools/integration-runner/
  integration-mode.sh      # Bash mode helper
  integration_mode.py      # Python mode helper
  junit-path.sh            # JUnit artifact path resolver
  run-jest-integration.sh  # Shared Jest runner
  ts/mode.ts               # TypeScript mode helper
```

### Existing Mock Directories

```
services/platform/__mocks__/
  index.ts
  client.ts

services/graph/__mocks__/
  service.ts

services/threads/__mocks__/
  client.ts
  fixtures.ts
  index.ts

services/www_ameide_platform/__mocks__/
  handlers.ts
  next/

services/agents/__mocks__/
  server.go

services/inference_gateway/__mocks__/
  inference.go

services/inference/__mocks__/
  fixtures.py
  http.py
  grpc.py
  websocket.py

services/workflow_worker/__mocks__/
  temporal.py

services/repository/__mocks__/
  client.ts
  fixtures.ts
  index.ts

services/chat/__mocks__/
  client.ts
  fixtures.ts

services/transformation/__mocks__/
  client.ts
  fixtures.ts
  handlers.ts

services/agents_runtime/__mocks__/
  grpc.py
  clients.py
  fixtures.py

services/workflows/__mocks__/
  client.ts
  fixtures.ts

services/workflows_runtime/__mocks__/
  temporal.py
  fixtures.py

services/www_ameide/__mocks__/
  index.ts
  http.ts
  fixtures.ts
```

### Integration Test Runners

```
packages/ameide_sdk_go/__tests__/integration/run_integration_tests.sh
packages/ameide_sdk_python/__tests__/integration/run_integration_tests.sh
packages/ameide_sdk_ts/__tests__/integration/run_integration_tests.sh
services/agents/__tests__/integration/run_integration_tests.sh
services/agents_runtime/__tests__/integration/run_integration_tests.sh
services/chat/__tests__/integration/run_integration_tests.sh
services/graph/__tests__/integration/run_integration_tests.sh
services/inference/__tests__/integration/run_integration_tests.sh
services/inference_gateway/__tests__/integration/run_integration_tests.sh
services/platform/__tests__/integration/run_integration_tests.sh
services/repository/__tests__/integration/run_integration_tests.sh
services/threads/__tests__/integration/run_integration_tests.sh
services/transformation/__tests__/integration/run_integration_tests.sh
services/workflow_worker/__tests__/integration/run_integration_tests.sh
services/workflows/__tests__/integration/run_integration_tests.sh
services/workflows_runtime/__tests__/integration/run_integration_tests.sh
services/www_ameide/__tests__/integration/run_integration_tests.sh
services/www_ameide_platform/__tests__/integration/run_integration_tests.sh
```
