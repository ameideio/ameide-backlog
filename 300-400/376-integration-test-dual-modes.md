---
title: 376 – Integration Test Dual Modes
status: deprecated
owners:
  - platform
created: 2025-01-15
superseded_by: 430-unified-test-infrastructure.md
---

---

> **DEPRECATED**: This backlog has been superseded by [430-unified-test-infrastructure.md](./430-unified-test-infrastructure.md).
>
> The content below is retained for historical reference. All new work should follow backlog 430.
>
> RPC transport/env-var determinism for cluster-mode tests is tracked in [589-rpc-transport-determinism.md](../589-rpc-transport-determinism.md).

---

## Problem

Integration suites currently run with best-effort heuristics: missing env vars or external services often result in silent skips or half-configured resources. This causes:

- Fragile local experience: engineers do not know whether a suite exercised the real service or mocked dependencies.
- Cluster (CI / in-cluster runners) experience inconsistent: some suites mock, others fail, leaving coverage gaps.
- Hard to reason about required secrets, database instances, or cluster credentials for each suite.

## Goal

Introduce explicit execution modes for all integration suites:

1. **Cluster mode** – runs inside an environment where platform services, databases, and secrets exist. Tests must fail fast if any required dependency (env var, schema, Temporal namespace, etc.) is missing. No mocks allowed in this mode.
2. **Non-cluster mode (`repo`/`local`)** – runs outside the cluster (local laptops, devcontainers, CI pre-merge jobs). Suites automatically swap all external calls with deterministic mocks/stubs and never attempt to reach live services. Missing env vars should trigger the mocked path, not implicit failures, and suites **must not silently skip** in non-cluster mode just because live dependencies are unavailable—either mocks are wired correctly or the suite fails loudly.

## Approach

### Execution mode contract

- Canonical env var: `INTEGRATION_MODE=repo|local|cluster`. No aliases, no heuristics—helpers must read this one variable via a shared parser.
- Treat the value as a strict discriminated union. Unknown strings → throw at startup so typos never silently fall back to `repo`.
- Default to `repo` whenever the variable is absent so laptops/devcontainers stay safe by default, but once a value is provided the helper must obey it exactly.
- Local/devcontainer runs should stay on `repo`/`local`; cluster mode is reserved for CI/in-cluster runners with real dependencies. Don’t expect cluster mode to work locally.
- Each integration helper inspects the parsed mode once and configures clients accordingly. Business tests remain untouched.
- `mode=cluster` is “live integration”: assert **all** required env vars (`WORKFLOWS_DATABASE_URL`, `AMEIDE_GRPC_BASE_URL`, Temporal namespaces, etc.) and fail immediately if any dependency is missing. Never fall back to mocks in this path.
- `mode=repo|local` is “mocked integration”: all external calls swap to deterministic stubs. Missing mock fixtures/config should also fail loudly—no partial/implicit behavior.

### Mode semantics & naming

- Surface the two modes in docs/CI as **Integration (cluster)** vs. **Integration (mocked)** so everyone knows which signal they are looking at.
- CI jobs, Tilt resources, and dashboards should label results explicitly (e.g., `integration-repo`, `integration-cluster`) to avoid conflating the two.
- Document the expectations per mode (live dependencies, latency, flake profile) so service teams understand what “integration” guarantees in each context.

### Mock scaffolding

- Centralize mocks, e.g. in `services/<svc>/__tests__/integration/helpers.ts`.
- Provide mock transport + seed data per service to simulate workflows:
  - Threads: stub gRPC layer returning canned responses.
  - Platform/transformation: stub Postgres via in-memory adapters or sqlite.
  - Workflows: stub Temporal + Postgres via fixtures (or use existing migration scripts and in-memory DB if cheap).
- Document what is mocked vs. real per suite to avoid diverging behavior.
- Add a thin “dual-mode” contract harness (per service) that can run against both modes and compare critical fields/status codes to catch drift early.

### Cluster footprint

Identify services and assets touched when `mode=cluster`:

| Suite | Dependencies |
| --- | --- |
| `services/workflows/__tests__/integration/*` | Postgres (workflows schema), Temporal namespace, env secrets (`WORKFLOWS_DATABASE_URL`, `TEMPORAL_ADDRESS`, `WORKFLOWS_TEMPORAL_NAMESPACE`) |
| `services/threads/__tests__/integration/*` | Threads service gRPC endpoint (Envoy), Postgres, feature flags |
| `services/platform/__tests__/integration/*` | Platform service gRPC endpoint, Postgres (`PLATFORM_DATABASE_URL`) |
| `services/transformation/__tests__/integration/*` | Transformation service gRPC endpoint, Postgres (`PLATFORM_DATABASE_URL`, `DATABASE_URL`) |

Work initiated in backlog #347/#361 already tackled skipping/failing logic; this RFC extends that by formalizing the mode toggle, not re-implementing mocks per suite.

### CI strategy

- **PR / pre-merge**: always run `integration-mock` (fast, deterministic). Optionally add a very small `integration-cluster-smoke` target when the infra budget allows.
- **Main / nightly**: run the full `integration-cluster` matrix plus `integration-mock` for parity. Clearly label which job blocks merges.
- `pnpm test:integration` defaults to `repo`. Provide `pnpm test:integration --mode=cluster` (and a smoke subset) for engineers with cluster credentials so they can reproduce “live” issues locally.
- CI, Tekton, Tilt, and GitHub Actions must explicitly set `INTEGRATION_MODE` instead of inferring from other env vars.

## Current status by language (2025-02-26 audit)

### JavaScript/TypeScript
- **Env contract** – `INTEGRATION_MODE` enforced via the shared helper; defaults to `repo`, rejects invalid strings. Helm values for SDKs/services set the mode explicitly.
- **Helper plumbing** – Runners for `chat`, `graph`, `platform`, `repository`, `threads`, `transformation`, `workflows`, `www_ameide`, `www_ameide_platform` branch on mode; non-cluster mode pins `JEST_EXTRA_ARGS` to mock suites, cluster branches still assert live env vars. A shared JUnit helper now scopes reports to `/artifacts/<service>/junit.xml` with safe fallbacks.
- **Mock scaffolding** – Mock-mode suites now exist for the services above (in-process/in-memory stubs only; no shared `packages/test-support`, no drift harness). Coverage is placeholder but keeps mock mode green.
- **SDKs** – TS SDK integration tests run dual-mode (router transport for mock; live Envoy/gRPC in cluster) with Helm values wiring the mode.
- **CI/Tilt** – Tilt split into `test-all-repo` vs. `test-all-cluster`; GitHub Actions still a single non-cluster path (no cluster-smoke matrix).
- **Gaps** – Missing shared mock library, telemetry, drift checks, and stronger service docs.

### Python
- **Env contract** – Mode helper available in `tools/integration-runner/integration_mode.py` but not yet adopted across Python services/runners.
- **Helper plumbing** – Python service integration scripts still rely on ad-hoc env checks; no dual-mode branching or mocks beyond the SDK.
- **Mock scaffolding** – Only the Python SDK uses an in-process gRPC stub for mock mode; service suites lack deterministic mocks.
- **SDKs** – Python SDK integration tests run dual-mode (mock via local gRPC stub; cluster gated on envoy envs); Helm values set the mode.
- **CI/Tilt** – No Python-specific split; relies on global runner behavior.
- **Gaps** – Need mode helper adoption, mock libraries, and dual-run harnesses for Python services.

### Go
- **Env contract** – Shared parser added at `packages/ameide_sdk_go/testsupport/integrationmode`; SDK and Go service runners source it and fail fast on invalid/absent values.
- **Mock scaffolding** – Go SDK mock mode uses bufconn Graph stub (`testsupport/mocks`); agents and inference-gateway suites run in-memory gRPC stubs in mock mode, cluster mode enforces live envs. Other Go packs/services remain cluster-only.
- **CI/Tilt** – Helm values for Go SDK/inference-gateway set `INTEGRATION_MODE`; no Go-specific cluster/matrix split in CI yet.
- **Gaps** – Need shared mock kit for remaining Go services, dual-run drift harness, and CI job split.

### Cross-cutting
- **Documentation** – Root/docs mention the enum and samples; `tests/integration/README.md` includes a package matrix, but service docs remain sparse on mode semantics.
- **Guardrails & signals** – Mode parsing is strict where adopted; no structured telemetry or dual-mode comparisons yet. JUnit outputs are namespaced per service to avoid collisions in mock runs.

## Required refactorings to align with dual-mode contract

- **Mode contract + runner**: Change `parseIntegrationMode` default to `repo` and keep strict validation; ensure `pnpm test:integration` and `tools/integration-runner` always inject `INTEGRATION_MODE=repo` unless explicitly overridden. Surface mode in logs/artifacts. Error on unset in cluster jobs so Helm/Tilt must set `cluster` intentionally.
- **Tilt/Helm/CI wiring**: Add explicit `integration-repo` vs. `integration-cluster` jobs/targets. Update Tilt resources to set `INTEGRATION_MODE=cluster` only for cluster-backed runs; Helm chart values should include the env var and refuse to start without it. Split GitHub Actions/ Tekton into repo (default/gating for PRs) and cluster (nightly/smoke) jobs with clear labels.
- **Service entrypoints**: Standardize `run_integration_tests.sh` (and Go/Python equivalents) to parse `INTEGRATION_MODE` once, fail fast on invalid values, and branch: cluster → assert required live deps; repo/local → inject mock wiring and skip live connectivity. Remove ad-hoc toggles (`*_INTEGRATION_USE_LIVE_DB`, `RUN_INTEGRATION_TESTS`).
- **Mock scaffolding**: Provide deterministic mocks per service (typed stubs + fixtures) usable by integration suites: Postgres/SQLite/in-memory adapters for `platform`, `repository`, `threads`, `chat`, `transformation`; Temporal/db fixtures for `workflows`; gRPC/service stubs for `agents`, `inference`, `inference_gateway`, `graph`; HTTP/api stubs for `www_ameide` and `www_ameide_platform`. Centralize common utilities under a shared package (e.g., `packages/test-support` or Go/Python equivalents).
- **Test harness updates**: Refactor suites to consume the mode-aware helpers (no direct env probing). Replace skip-on-missing-env logic with explicit mode checks; ensure mocks are used in non-cluster mode (`repo`/`local`) and live dependencies are required in `cluster` mode. Add drift checks where mock + cluster both exist.
- **Telemetry/guardrails**: Emit structured logs/metrics when mode is selected (and when falling back to mocks). Fail tests if mock prerequisites are missing or if a cluster run attempts to stub a dependency. Add assertion helpers to prevent mixed mode usage mid-run.
- **Docs and samples**: Document the contract in `README.md`, service READMEs, and `tests/integration/README.md`; add `.env.integration` sample illustrating `repo` defaults and cluster overrides. Include a mode-to-dependency matrix and examples for both modes.
- **Dual-mode contract tests**: Add golden-path harnesses that execute both modes for critical flows and compare outputs (status codes/payload shapes) to detect mock drift early.

## Work Items

1. **Define env contract** – Document `INTEGRATION_MODE` (strict enum, default repo, failure on invalid) in repo root README + `services/*/README.md`. Provide sample `.env.integration`.
2. **Update helpers** – Each integration helper reads `INTEGRATION_MODE`. Use discriminated union to instantiate either real clients or mocks.
3. **Mock libraries** – Build shared mock utilities (gRPC stubs, Postgres in-memory wrappers) under `packages/test-support`.
4. **Documentation** – Update backlog #361 or create new doc describing the matrix of suites vs. dependencies, how to run locally vs. cluster/mocked, and how CI jobs are named.
5. **CI wiring** – Update GitHub Actions / Tekton / Tilt flows to set `INTEGRATION_MODE`, publish separate repo vs. cluster jobs, and spell out which one gates merges.
6. **Telemetry & guardrails** – Emit structured logs/metrics when a suite enters `mock` unexpectedly, and fail tests when either real or mock prerequisites are missing.
7. **Dual-mode contract tests** – For a handful of golden paths per service, execute both modes and compare results to detect mock drift quickly.

## Risks / Considerations

- Mock drift: ensure proto/grpc contracts stay aligned by reusing generated types and the dual-mode contract harness.
- Runtime parity: some suites may rely on DB migrations (e.g., workflows). Consider using embedded Postgres or sqlite for mock mode if we need SQL semantics beyond simple stubs.
- Developer ergonomics: document the prerequisites + smoke subset for `pnpm test:integration --mode=cluster` so engineers can debug against real infra without running the whole matrix.

## Next Steps

1. Ratify `INTEGRATION_MODE` as the single knob (including validation + default behavior).
2. Inventory all integration suites and categorize dependencies.
3. Implement helper changes iteratively per service, starting with workflows (already partially gated).
4. Update CI/Tilt pipelines to surface `integration-mock` vs. `integration-cluster` jobs and document merge-blocking behavior.
5. Update backlog entry once implementation is complete with real metrics (pass/fail rates per mode).
