> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

> **DEPRECATED**: This backlog has been superseded by 430v2:
> - `backlog/430-unified-test-infrastructure-v2.md`
> - `backlog/430-unified-test-infrastructure-v2-target.md`
>
> The content below is retained for historical reference. All new work should follow 430v2.
>
> Also note: image references and registry examples in this document may mention legacy local registries; the current standard is `ghcr.io/...@sha256:...` (immutable digests).

---

# Integration Testing Framework

**Status**: ✅ Integration Test Infrastructure Complete (October 2026)
**Priority**: High
**Started**: November 2025 (rebooted January 2026)
**Owner**: Platform Team
**Last Updated**: October 26, 2026 (18:30 UTC)

---

## Intent

Deliver a repeatable, language-agnostic integration testing framework that validates service-to-service behaviour inside the Kubernetes mesh, provides fast feedback for developers, and scales across CI, Tilt, and local workflows. The same foundation should eventually host contract and end-to-end suites (e.g., Playwright) so the platform maintains a single execution model across the test pyramid.

---

## Guiding Principles

1. **Run where it matters** – Execute tests inside a production-like Kubernetes namespace with cluster DNS, service mesh sidecars, and RBAC enforced.  
2. **Package the tests** – Bake each critical suite into the image we already ship (service image today, dedicated pack later) so dependencies are immutable and no runtime copy is required.  
3. **Own your contract** – Service teams author and maintain their integration packs; platform supplies shared tooling, runners, and observability.  
4. **One command, many surfaces** – `pnpm test:integration -- --filter <service> --env=staging` drives shared-cluster runs, while Tilt triggers reuse the same entrypoint for local feedback.  
5. **Deterministic data** – Tests own their fixtures and clean up after themselves so suites stay idempotent.  
6. **Actionable feedback** – Logs, metrics, and traces from test runs must make failures diagnosable within minutes.

---

## Goals

| Goal | Status | Notes |
|------|--------|-------|
| Fast feedback loop (<10 min) | ✅ Live for E2E | Playwright build optimized (8s-64s); Inference pack runs in-cluster |
| In-cluster execution for all core services | 🟡 Partial | Inference + E2E live; graph/workflows/platform packs outstanding |
| Language-agnostic packs (Python/Node/Go) | ☐ Pending | Document baked-in entrypoint examples per language |
| CI-ready workflows | ✅ Live | GitHub Actions validates staging plan; Playwright + inference ready |
| Tilt integration with per-service buttons | ✅ Live | E2E working with full log output; integration tests use `kubectl exec` |
| Non-interactive authentication for E2E | ✅ Complete | Playwright global setup automates Keycloak SSO using `E2E_SSO_*` env vars |
| Full test output visibility | ✅ Complete | Playwright logs stream to Tilt UI with job monitoring |
| Observability + flaky detection | ☐ Pending | Prometheus metrics + alerting backlog'd |
| Shared framework for integration + E2E | ✅ Live | Playwright uses test-job-runner chart; clear label separation (test.e2e vs test.integration) |

Success is declared when all core services (inference, graph, workflows, platform) ship integration packs, run in CI, and expose Tilt buttons.

---

## Convention-Based Test Discovery

**Integration runner automatically discovers test packs using directory conventions:**

### Integration Tests
- **Locations**: `services/<service>/{__tests__/integration,tests/integration}`, `packages/<package>/{__tests__/integration,tests/integration,src/__tests__/integration,src/tests/integration}`
- **Discovery**: Automatic – no config.yaml required; runner scans both `services/**` and workspace `packages/**`
- **Runner**: `pnpm test:integration`
- **Values**: Defaults to `infra/kubernetes/environments/local/integration-tests/<name>.yaml` for services and `infra/kubernetes/environments/local/integration-tests/packages/<name>.yaml` for packages; override via `valuesByEnv` when needed
- **Examples**: `services/inference/__tests__/integration/test_inference_api.py`, `packages/ameide_sdk_ts/__tests__/integration/ameide-client.integration.test.ts`

### E2E Tests (Playwright)
- **Location**: `**/__tests__/e2e/` (anywhere in codebase)
- **Discovery**: Automatic via Playwright `testMatch` pattern
 - **Runner**: Tilt `e2e-playwright` resource (shared test job via Helm)
 - **Log Controls**: `e2e-playwright-log-tail` (stream), `...-status` (diagnostics), `...-reset` (force restart)
- **Example**: `services/www_ameide_platform/features/threads/__tests__/e2e/threads-connect.smoke.spec.ts`

**Current Discovered Packs**:
```bash
$ pnpm test:integration -- --list
16 test pack(s) discovered:
agents              registry.dev.ameide.io:5000/ameide/agents                             timeout=600
agents_runtime      registry.dev.ameide.io:5000/ameide/agents_runtime                     timeout=600
ameide_core_proto          registry.dev.ameide.io:5000/ameide/ameide_core_proto                         timeout=600
ameide_sdk_go         registry.dev.ameide.io:5000/ameide/ameide_sdk_go                        timeout=600
ameide_sdk_python     registry.dev.ameide.io:5000/ameide/ameide_sdk_python                    timeout=600
ameide_sdk_ts         registry.dev.ameide.io:5000/ameide/ameide_sdk_ts                        timeout=600
graph               registry.dev.ameide.io:5000/ameide/graph                              timeout=600
inference           registry.dev.ameide.io:5000/ameide/inference                          timeout=600
inference_gateway   registry.dev.ameide.io:5000/ameide/inference_gateway                  timeout=600
platform            registry.dev.ameide.io:5000/ameide/platform                           timeout=600
threads             registry.dev.ameide.io:5000/ameide/threads                            timeout=600
transformation      registry.dev.ameide.io:5000/ameide/transformation                     timeout=600
workflows           registry.dev.ameide.io:5000/ameide/workflows                          timeout=600
workflows_runtime   registry.dev.ameide.io:5000/ameide/workflows_runtime                  timeout=600
www_ameide          registry.dev.ameide.io:5000/ameide/www_ameide                         timeout=600
www_ameide_platform registry.dev.ameide.io:5000/ameide/www_ameide_platform                timeout=600
```

**Separation of Concerns**:
- Integration tests (`__tests__/integration`) test service-to-service APIs, contracts, connectivity
- E2E tests (`__tests__/e2e`) test full user journeys through the browser with Playwright
- Both use the same underlying Kubernetes Job infrastructure but are triggered separately

---

## Scope

**In scope**
- Service-to-service API validation (HTTP, gRPC, Temporal workflows)
- Data contract checks (schemas, payload shapes, error handling)
- Network/connectivity smoke suites
- Per-service test ownership and documentation

**Out of scope (tracked separately)**
- Full UI end-to-end tests (Playwright pods are governed by backlog/303-elements.md)
- Chaos and load testing (future transformation)
- Third-party SaaS contract tests (backlog item TBD)

---

## Architecture Overview

```
┌────────────────────────┐
│ Service Image          │  (primary app image + baked-in tests)
│  ├─ App code + tests    │
│  ├─ Runtime deps        │
│  └─ Test entry script   │
└────────────┬───────────┘
             │
      Helm Chart: test-job-runner
             │
┌────────────▼────────────┐
│ Kubernetes Job / Pod    │  (namespace: ameide-int-tests)
│  ├─ Runs pack entrypoint│
│  ├─ Emits logs/metrics  │
│  └─ Pushes junit & logs │
└────────────┬────────────┘
             │
   Orchestrator CLI (pnpm test:integration)
             │
  ┌──────────┴──────────┐
  │ Tilt local_resource │──Manual triggers (local clusters)
  │ Staging orchestrator│──`pnpm test:integration --env=staging`
  │ CI Pipeline         │──PR / nightly gates
  └─────────────────────┘
```

### Core Components

- **Test Pack Discovery (Convention-Based)** – Integration runner automatically discovers services with `__tests__/integration/` directories. No config.yaml required. Tests are baked into service images during build.
- **Integration Tests** – Located at `services/<service>/__tests__/integration/`. Discovered automatically by scanning the services directory.
- **E2E Tests (Playwright)** – Located at `**/__tests__/e2e/`. Discovered automatically by Playwright's testMatch pattern. Managed separately from integration tests via `e2e-playwright` Tilt resource.
- **Job Template** – Helm chart (`infra/kubernetes/charts/test-job-runner`) parameterised with image, command, env vars, secrets, and resource requests.
- **Test Orchestrator** – Node-based CLI under `tools/integration-runner/` invoked via `pnpm test:integration`. Auto-discovers packs, renders Helm values, and monitors Job status, surfacing logs and junit artifacts.
- **Storage & Reporting** – Artifacts are copied from the Kubernetes Job back into `artifacts/integration/<service>/<timestamp>/` via `kubectl cp`. External storage, metrics, and log aggregation are future enhancements.
- **Tilt Integration** – `integration-run` resource triggers the orchestrator for discovered packs. E2E tests use the shared `e2e-playwright` Helm resource (feature filtering handled inside Playwright).

### Local (Tilt) vs Staging/Prod Execution

| Aspect | Local k3d + Tilt loop | Staging (runner-managed) |
|--------|-----------------------|------------------------------|
| Build path | `Tiltfile:356-408` uses `docker_build_with_restart` against `services/inference/Dockerfile.dev`, live-syncing source and test files into the long-lived inference pod. | CI builds `services/inference/Dockerfile`, publishes to ACR, and the runner pulls that tag through Helm values. |
| Deployment | Tilt applies `infra/kubernetes/charts/platform/inference` with `infra/kubernetes/environments/local/platform/inference.yaml`, keeping the inference Deployment running in the local `ameide` namespace. | The runner renders `infra/kubernetes/charts/test-job-runner` and creates a one-off Job in `ameide-int-tests` (or staging/prod namespaces). |
| Test trigger | `tilt trigger test:integration-inference` runs `kubectl exec` inside the existing pod, reusing the live-updated image. | `pnpm test:integration -- --filter inference --env=staging` launches the Job, tails pod logs, and copies `/artifacts` once the run completes. |
| Artifacts | Stay inside the k3d cluster unless manually copied; no MinIO/S3 upload. | Saved under `artifacts/integration/<service>/<timestamp>/` and exposed as CI artifacts. |
| Purpose | Fast feedback while iterating on code/tests with live reload. | Deterministic gating in staging once images are published. |

Until a dedicated test pack image ships, the local Tilt workflows intentionally diverges so developers keep live-reload while staging runs follow the runner/Helm Job model. Prod adoption remains a future milestone.

---

## Environment Alignment

- **Container registry** – Local runs resolve to `docker.io/ameide/inference`; CI rewrites staging values to `ameide.azurecr.io/inference:<sha>` before launching the job.  
- **Secret management** – Pack reuses existing service credentials; dedicated Key Vault bindings for test jobs are still pending.  
- **Helm / Helmfile** – Runner deploys the shared chart directly via `helm upgrade --install`. Helmfile/Argo CD ownership is future work.  
- **Argo CD** – No dedicated Argo CD application yet; orchestration remains manual until automation lands.

---

## Repository Layout & File Organization

- `services/<service>/tests/integration/` and `packages/<package>/{__tests__/integration,tests/integration}` – Owner-managed tests + metadata (`config.yaml`, fixtures, testcases) discovered automatically by the runner.  
- `tools/integration-runner/` – Orchestrator CLI package (Node) and shared utilities.  
- `infra/kubernetes/charts/test-job-runner/` – Helm chart consumed by helmfile + Argo CD.  
- `infra/kubernetes/environments/<env>/integration-tests/` – Environment-specific values rendered by helmfile/Argo CD (one YAML per service).  
- `docs/integration-testing.md` – Cross-team playbook (to author as part of rollout).  
- `tilt/` – Existing Tilt modules; new `tilt/integration_tests.py` encapsulates local_resource definitions and image build wiring.

This layout keeps integration assets co-located with their owning service while reusing our established Helm and Tilt infrastructure.

---

## Existing Test Assets

- `tests/` (repo root) contains the legacy pytest harness: smoke checks (`tests/smoke/`), rich shared utilities (`tests/shared/`), Docker lifecycle fixtures (`tests/fixtures/`), and Playwright CI plumbing (`tests/automation/`).  
- The original documentation references `tests/integration/` and `tests/e2e/`, but those suites were removed; only smoke tests and shared helpers remain.  
- `tests/shared/` exports retry, HTTP, gRPC, Testcontainers, and docker-compose helpers that we can migrate into service-owned packs; they currently assume local Docker workflows rather than Kubernetes.  
- Playwright automation is consolidating into a repo-level runner image (`tests/playwright-runner/`) that layers on `mcr.microsoft.com/playwright`. Specs from `services/www_ameide_platform/tests/automation/` are being migrated into this pack so the orchestrator can execute them.  
- `tests/README.md` now documents the integration-runner workflows; keep it aligned as additional packs come online.

---

## Test Categories Supported

| Category | Typical Execution Surface | Notes |
|----------|---------------------------|-------|
| Unit | Developer machine, service CI job, Tilt auto-run | Unit packs auto-trigger on code changes (Tilt `auto_init=True`) to preserve the tight feedback loop; teams may keep classic local runners as well. |
| Contract / Integration | Kubernetes namespace (`ameide-int-tests`), CI, Tilt (manual by default) | Primary focus of this backlog item—validate service-to-service contracts, Temporal workflows, and data flows. |
| End-to-End (UI / Playwright) | Kubernetes namespace (optional browsers), staging/prod namespaces | Playwright packs reuse the same chart/orchestrator; manual triggers avoid accidental long runs but can be scheduled in CI. |

Nothing prevents packaging unit suites as lightweight packs for hermetic execution; unit jobs are auto-triggered locally, while integration and E2E remain manual unless teams explicitly opt into watch mode.

---

## Implementation Plan

### 0. Prerequisites
- Dedicated namespace: `ameide-int-tests` (provisioned via existing helmfile workflows).  
- Helm chart bootstrap (`infra/kubernetes/charts/test-job-runner`) wired into `infra/kubernetes/helmfile.yaml`.  
- MinIO bucket `integration-test-artifacts` + service account credentials.  
- Observability plumbing: Prometheus scrapable endpoint, Grafana dashboard skeleton.  
- `pnpm` workspace package `tools/integration-runner` scaffolded.
- Argo CD application specs for staging/prod (`infra/kubernetes/environments/{staging,prod}/integration-tests.yaml`) to ensure GitOps parity.

### 1. Add Integration Tests (Convention-Based)

**Simply create tests in `__tests__/integration/` - no configuration needed:**

```bash
# Create integration tests for any service
mkdir -p services/myservice/__tests__/integration
touch services/myservice/__tests__/integration/test_api.py

# Discovery is automatic
pnpm test:integration -- --list
# myservice will appear automatically
```

**Directory structure:**
```
services/myservice/
├── src/
├── __tests__/
│   ├── integration/          ← Auto-discovered by integration-run
│   │   ├── test_api.py
│   │   └── test_connectivity.py
│   ├── e2e/                  ← Auto-discovered by Playwright
│   │   ├── user-flow.spec.ts
│   │   └── smoke.spec.ts
│   └── unit/                 ← Run by service test suite
│       └── test_helpers.py
└── Dockerfile
```

**Service image must include tests:**
```dockerfile
# Copy tests into the image
COPY services/myservice/__tests__ /app/services/myservice/__tests__
COPY services/myservice/tests /app/services/myservice/tests  # if using legacy tests/ dir

# Ensure test runner script is executable
RUN chmod +x /app/services/myservice/tests/run_integration_tests.sh
```

**Test runner script** (`services/myservice/tests/run_integration_tests.sh`):
```bash
#!/usr/bin/env bash
set -euo pipefail
JUNIT_PATH="${PYTEST_JUNIT_PATH:-/artifacts/junit.xml}"
pytest "/app/services/myservice/__tests__/integration" -v --junitxml="$JUNIT_PATH"
```

### 2. ~~Define Service Config~~ No Longer Needed

**Convention-based discovery eliminates config.yaml requirement.**

The integration runner auto-generates pack metadata:
- Service name from directory name
- Image from `docker.io/ameide/<service>`
- Release name as `<service>-test`
- Values file from `infra/kubernetes/environments/<env>/integration-tests/<service>.yaml`
- Image source from `Deployment/<service>` in `ameide` namespace

**Only create Helm values file if you need custom configuration:**
```yaml
# infra/kubernetes/environments/local/integration-tests/myservice.yaml
namespace: ameide-int-tests
image: docker.io/ameide/myservice
command: /app/services/myservice/tests/run_integration_tests.sh
env:
  - name: DATABASE_URL
    value: postgresql://user:pass@postgres:5432/test
```

### 3. Author Tests

**Use the `__tests__/integration` convention:**
```
services/<service>/
├── __tests__/
│   ├── integration/        ← Auto-discovered
│   │   ├── test_api.py
│   │   ├── test_connectivity.py
│   │   └── conftest.py
│   └── e2e/                ← Auto-discovered by Playwright
│       └── smoke.spec.ts
└── tests/                  ← Legacy location (still supported)
    └── run_integration_tests.sh
```

**Python example (`services/inference/tests/integration/test_graph_contract.py`)**
```python
import httpx

def test_elements_contract():
    response = httpx.get(
        "http://graph.ameide.svc.cluster.local:8080/api/repositories/demo/elements",
        timeout=10.0,
    )
    response.raise_for_status()
    payload = response.json()
    assert "elements" in payload
    assert all("id" in e and "type" in e for e in payload["elements"])
```

**Node example using Jest (`services/graph/tests/integration/elements.test.ts`)**
```typescript
import fetch from 'node-fetch';

describe('Repository integration', () => {
  it('responds to health check', async () => {
    const res = await fetch(`${process.env.REPOSITORY_BASE_URL}/health`, { timeout: 8000 });
    expect(res.status).toBe(200);
  });
});
```

**Go example (`services/inference_gateway/tests/integration/health_test.go`)**
```go
package integration

import (
    "context"
    "testing"
    "time"

    workflowsv1 "github.com/ameideio/ameide-sdk-go/proto/ameide_core_proto/workflows_runtime/v1"
    "google.golang.org/grpc"
)

func TestWorkflowReachable(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    conn, err := grpc.DialContext(ctx,
        "workflows-service.ameide.svc.cluster.local:8086",
        grpc.WithInsecure(),
        grpc.WithBlock(),
    )
    if err != nil {
        t.Fatalf("dial failed: %v", err)
    }
    defer conn.Close()

    client := workflowsv1.NewWorkflowServiceClient(conn)
    resp, err := client.ListWorkflowDefinitions(ctx, &workflowsv1.ListWorkflowDefinitionsRequest{
        TenantId: "integration-tests",
    })
    if err != nil {
        t.Fatalf("list definitions failed: %v", err)
    }
    if resp == nil {
        t.Fatalf("expected response, got nil")
    }
}
```

### 4. Integrate with Orchestrator

`tools/integration-runner` responsibilities:
- Parse CLI flags (`--filter`, `--env`, `--watch`, `--build-only`, `--dry-run`).  
- Discover packs, render execution plans, and support dry runs without touching Kubernetes.  
- For `--build-only`, run `docker build` against the service Dockerfile declared in `config.yaml`.  
- Cluster execution currently targets staging only (`--env=staging`), rendering Helm values, installing the Job, streaming logs, and copying `/artifacts` back into the graph.  
- Local iterations stay on Tilt; the runner does not support a direct `--local` execution path anymore.  
- Surface friendly command logging so failures are debuggable.

Workspace script in `package.json`:
```json
{
  "scripts": {
    "test:integration": "node tools/integration-runner/bin/integration-runner.js",
    "test:integration:all": "node tools/integration-runner/bin/integration-runner.js --all",
    "test:e2e": "node tools/integration-runner/bin/integration-runner.js -- --filter playwright"
  }
}
```

Service-level scripts (optional) re-export the orchestrator:
```json
{
  "scripts": {
    "test:integration": "pnpm test:integration -- --filter graph --env=staging"
  }
}
```

Run staging packs via `pnpm test:integration -- --filter <service> --env=staging`. Append `--dry-run` when you only want to inspect the execution plan.

### 5. Kubernetes Job Template

`infra/kubernetes/charts/test-job-runner/templates/job.yaml` (excerpt):
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "test-job-runner.fullname" . }}
  namespace: {{ .Values.namespace }}
spec:
  ttlSecondsAfterFinished: {{ .Values.ttlSecondsAfterFinished }}
  backoffLimit: 1
  template:
    spec:
      serviceAccountName: {{ .Values.serviceAccountName }}
      restartPolicy: Never
      containers:
        - name: {{ .Chart.Name }}
          image: {{ .Values.image }}
          imagePullPolicy: IfNotPresent
          command: ["/bin/sh", "-c", {{ quote .Values.command }}]
          env: {{- toYaml .Values.env | nindent 12 }}
          volumeMounts:
            - name: artifacts
              mountPath: /artifacts
          resources: {{- toYaml .Values.resources | nindent 12 }}
      volumes:
        - name: artifacts
          emptyDir: {}
```

Note: `imagePullPolicy` here is a test-runner concern; GitOps-managed environment workloads should follow `backlog/602-image-pull-policy.md` (pin by digest/SHA).

Default `values.yaml` sets sensible resources (256Mi/200m) and calls `/app/tests/run_integration_tests.sh` when available (falling back to `/app/services/<service>/tests/run_integration_tests.sh`). Add optional sidecar for Prometheus exporter when needed.

### 6. Tilt Integration

`Tiltfile` pattern:
```python
local_resource(
    name="test:integration-graph",
    cmd="pnpm test:integration --filter graph",
    trigger_mode=TRIGGER_MODE_MANUAL,
    labels=["test.graph"],
    resource_deps=["graph", "postgres"],
)
```

Guidelines:
- Keep labels feature-specific (`test.graph`, `test.workflows`).  
- Surface job logs in Tilt via `read_file('tmp/integration-graph.log')` (to be implemented by orchestrator).  
- Tilt button should fail fast if required resources are not healthy.
- Leverage existing `docker_build`/`custom_build` blocks that push to the internal registry with live-reload; packs reuse the per-service image target so code changes hot-reload without full rebuilds.  
- For unit packs, set `auto_init=True` to execute on code changes; integration and Playwright packs default to manual triggers but can opt into watch mode by passing `--watch` to the orchestrator.  
- Add reusable helper in `tilt/integration_tests.py` to avoid duplicating build + resource wiring.

### 7. CI/CD Integration

GitHub Actions automation (`.github/workflows/ci-integration-packs.yml`):
```yaml
name: Integration Packs

on:
  pull_request:
    paths:
      - 'services/**/tests/integration/**'
      - 'tools/integration-runner/**'
      - 'infra/kubernetes/charts/test-job-runner/**'
      - 'infra/kubernetes/environments/**/integration-tests.yaml'
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: pnpm/action-setup@v3
        with:
          version: 10.19.0
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --no-frozen-lockfile
      - name: Plan staging run
        run: pnpm test:integration -- --filter inference --env=staging --dry-run
      - name: Upload artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: integration-artifacts
          path: artifacts/integration
          if-no-files-found: ignore
```

Service image builds continue to live in the per-service pipelines. The integration workflows now focuses on validating the execution plan for staging and surfacing any configuration drift before the manual trigger runs.

**Staging / Prod (Argo CD)**
- Integration test chart values checked into `infra/kubernetes/environments/staging/integration-tests.yaml` and `.../production/...`.  
- Argo CD applications sync the chart with `runPolicy: manual` to allow on-demand validation post-deploy.  
- Promotion workflows: once CI passes, Argo CD pull request updates image tags; integration test job runs automatically in staging, optionally in prod during smoke windows.

### 8. Playwright Runner (E2E) - ✅ Complete with Non-Interactive Authentication

**Status**: Production-ready with optimized builds, log controls, and non-interactive authentication

**Architecture**:
- Image built from `mcr.microsoft.com/playwright:v1.49.0-jammy`
- Installs dependencies via `pnpm install --no-frozen-lockfile` (cached via Docker layers)
- Runner entrypoint is the CLI (`ameide test e2e`); job values should not invoke bash wrappers
- Writes junit/HTML/video artifacts to `/artifacts` (emptyDir volume)
- Pack definition lives in `tests/playwright-runner/config.yaml`, enabling `pnpm test:e2e` (a thin wrapper around the integration runner) to target the same Helm release in `local`, `staging`, or `production` via `--env=<env>`.
- Runner uploads those artifacts to `s3://integration-test-artifacts/<env>/<service>/<timestamp>/` when `INTEGRATION_ARTIFACT_BUCKET` is configured (MinIO endpoint exposed via `INTEGRATION_ARTIFACT_ENDPOINT`). Engineers can fetch them with `aws s3 ls --endpoint-url <minio> s3://integration-test-artifacts/...` or `mc cp` without needing pod access.

**Authentication** (✅ October 26, 2026; guardrail tightened per backlog/362-unified-secret-guardrails.md):
- Playwright global setup automates the Keycloak login UI (no manual steps)
- Credentials supplied via `E2E_SSO_USERNAME` / `E2E_SSO_PASSWORD` sourced from the integration Vault secrets bundle; no inline fallback is permitted. `E2E_USERNAME` / `E2E_PASSWORD` are legacy aliases that mirror the SSO values for older helpers.
- Global setup authenticates once and saves state to `tests/.auth/state.json`
- All tests reuse authenticated session for fast execution
- Logs include the authenticated user and roles for traceability

**Tilt Integration** (under `test.e2e` label):
- `e2e-playwright` resource keeps the Kubernetes Job definition in sync with the latest image.
- Helper buttons (`e2e-playwright-run`, `...-log-tail`, `...-status`, `...-reset`) delete/recreate the job, stream logs, and copy artifacts on success.
- Job monitoring still surfaces proper exit codes (0=success, 1=failure, 124=timeout) and forwards stdout into Tilt.

**Log Streaming** (✅ October 26, 2026):
- Script waits for container ready state before streaming logs
- Foreground log streaming shows all Playwright output in Tilt UI
- Background job monitoring detects completion/failure every 2 seconds
- Visible output includes:
  - Authentication setup messages
  - Test discovery and execution
  - Individual test results
  - Final summary (e.g., "2 passed (12.7s)")

**Build Optimization**:
- Custom `.dockerignore` reduces context from 1.53GB → ~100MB
- Layer caching optimizes pnpm install --no-frozen-lockfile (41s → 2s on cache hit)
- BuildKit inline cache enabled
- Specific Tilt deps/ignore lists prevent unnecessary rebuilds

**Performance**:
- First build: ~64s (was 173s)
- Code change: ~22s (was 173s)
- Test change only: ~8s (was 173s)

**Configuration**:
- Pack config: `tests/playwright-runner/config.yaml`
- Values: `infra/kubernetes/values/e2e/playwright.yaml` + environment overlays under `infra/kubernetes/environments/<env>/integration-tests/e2e/playwright.yaml`
- Dockerfile: `tests/playwright-runner/Dockerfile` (optimized for layer caching)
- Auth env vars: `E2E_SSO_USERNAME`, `E2E_SSO_PASSWORD`

**Key Files**:
- Job monitoring script: [`build/scripts/run-playwright-job.sh`](build/scripts/run-playwright-job.sh)
- Tilt resources & helpers: [`Tiltfile`](Tiltfile)
- Sample tests: [`services/www_ameide_platform/features/editor/__tests__/e2e/`](services/www_ameide_platform/features/editor/__tests__/e2e/)

**Production Readiness**:
- ✅ Non-interactive authentication working
- ✅ Full test output visible in Tilt logs
- ✅ Job failure detection working
- ✅ Fast feedback loop (<2 minutes for smoke tests)
- Gate execution behind Helm value (`playwright.enabled`) when needed
- Observability and alerting integration pending

### 9. Observability

- **Metrics** *(planned)*: `integration_test_duration_seconds`, `integration_test_failures_total`, `integration_test_flaky_total`. Packs will emit via Pushgateway once metrics plumbing lands.  
- **Logging**: STDOUT streamed during execution; cluster runs pipe `kubectl logs job/<name>`. Loki integration remains on the roadmap.  
- **Tracing** *(optional)*: Add OpenTelemetry instrumentation for complex flows (e.g., Temporal) when needed.  
- **Dashboards** *(planned)*: Grafana board with success rate per service, last run duration, flakes over time.  
- **Alerting** *(planned)*: PagerDuty integration when failure rate exceeds threshold for two consecutive runs.

### 10. Data Management

- Seed disposable databases using fixtures stored under `fixtures/`.  
- For shared resources (e.g., Temporal), create dedicated namespaces per test run (`temporal namespace int-test-<guid>`).  
- Enforce teardown in `entrypoint.sh` (trap on EXIT).  
- Use synthetic tenants to avoid clashing with manual QA data.

### 11. Troubleshooting Playbook

| Symptom | Likely Cause | Mitigation |
|---------|--------------|------------|
| Job stuck in `ImagePullBackOff` | Image not published or registry auth missing | Confirm CI published the image (`pnpm test:integration` workflows) and Helm values include imagePullSecrets |
| DNS resolution errors | Job not running in cluster network or service name wrong | Validate namespace, run `kubectl exec` busybox inside same namespace |
| Timeouts | Target service not healthy or wrong port | Check Tilt/Helm deployment, inspect service endpoints |
| Missing junit files | Entry script didn’t write to `/artifacts` | Ensure test runner outputs junit (pytest `--junitxml=/artifacts/junit.xml`, jest `--outputFile`) |
| Flaky retries | External dependency not isolated | Increase retry budgets or introduce mock service |

Troubleshooting snippets should live beside the orchestrator README for quick copy/paste.

---

## Service Implementation Tracker

| Deliverable | Inference | www-ameide | Repository | Workflow | Platform | Notes |
|-------------|-----------|------------|----------|----------|----------|-------|
| `__tests__/integration` directory | ✅ Present (2 files) | 🟡 Present (empty) | ✅ Present (1 file) | ✅ Present (1 file) | ✅ Present (1 file) | Auto-discovered when directory exists |
| Service image w/ test entrypoint | ✅ Tests baked in | ☐ Not started | ✅ Script ready | ✅ Script ready | ✅ Script ready | `.dockerignore` must allow test files |
| ~~Service config.yaml~~ | ~~N/A~~ | ~~N/A~~ | ~~N/A~~ | ~~N/A~~ | ~~N/A~~ | **Convention-based - no config needed** |
| Auto-discovered by runner | ✅ Yes | ✅ Yes | ✅ Ready | ✅ Ready | ✅ Ready | `pnpm test:integration -- --list` |
| Test suite with junit output | ✅ 25 tests passing | ☐ No tests | 🟡 1 test (needs deps) | 🟡 1 test (needs deps) | 🟡 1 test (needs deps) | pytest/jest outputs junit |
| Tilt integration | ✅ `test:integration-inference` | ☐ Button exists, no tests | ✅ `test:integration-graph` | ✅ `test:integration-workflows` | ✅ `test:integration-platform` | kubectl exec into Deployment |
| Resource dependencies | ✅ Configured | ☐ N/A | ✅ Auto-rebuild on trigger | ✅ Auto-rebuild on trigger | ✅ Auto-rebuild on trigger | `resource_deps` ensures service ready |
| Jest @swc/jest configured | N/A (Python) | ☐ Not started | ✅ Dependencies added | ✅ Dependencies added | ✅ Dependencies added | Required for TypeScript test execution |
| Local tests working | ✅ 25 passed in 8.94s | ☐ Not applicable | 🟡 Infrastructure ready | 🟡 Infrastructure ready | 🟡 Infrastructure ready | Via `tilt trigger test:integration-*` |
| CI job enforcement | ☐ Not enforced | ☐ Not started | ☐ Not started | ☐ Not started | ☐ Not started | Gate PRs + nightly runs |
| Metrics + logs instrumentation | ☐ Not started | ☐ Not started | ☐ Not started | ☐ Not started | ☐ Not started | Pushgateway metrics pending |

### Service Checklists

**Callout:** The `ci-integration-packs` GitHub workflows now validates the staging execution plan on each PR/push (`pnpm test:integration --env=staging --dry-run`) and uploads any generated artifacts to `artifacts/integration/inference/<timestamp>/`.

**Inference (pilot service)** – Status: ✅ Fully Working (October 26, 2026)
- [x] Tests in `services/inference/__tests__/integration/` (2 files: test_inference_api.py, test_service_connectivity.py)
- [x] Provide pytest wrapper (`services/inference/tests/run_integration_tests.sh`) emitting junit
- [x] Auto-discovered by integration runner (no config.yaml needed)
- [x] Fixed `.dockerignore` to include test files in image
- [x] Tilt button `test:integration-inference` working via kubectl exec
- [x] 25 integration tests passing consistently (health, invoke, streaming, connectivity, env checks)
- [x] Tests persist across pod restarts (baked into image)
- [ ] Backfill runbook for staging execution and failure triage
- [ ] Expand coverage for WebSocket end-to-end and agent routing scenarios
- [ ] Enable CI gating (currently manual execution only)
- [ ] Enforce pack as a required PR gate
- [ ] Add deterministic fixtures/cleanup for persistence and checkpoint validation

**Repository (Node/Jest)** - Status: ✅ Infrastructure Complete (October 26, 2026)

**Rationale**: Core service already being consumed by inference tests, but has no in-cluster integration tests of its own. Only unit tests and one E2E test exist. Repository exposes 3 critical gRPC services that need validation.

**Completed Steps**:

1. ✅ **Created test directory** at `services/graph/__tests__/integration/`
2. ✅ **Created test runner script** at `services/graph/__tests__/integration/run_integration_tests.sh`
3. ✅ **Added Jest integration config** at `services/graph/jest.integration.config.mjs`
4. ✅ **Added @swc/jest dependencies** to `services/graph/package.json`
5. ✅ **Created Tilt resource** `test:integration-graph` with `resource_deps=['graph']`
6. ✅ **Verified auto-rebuild** - Service rebuilds automatically before test execution
7. ✅ **Test infrastructure verified** - Jest loads, discovers tests, executes successfully

**Current Status**:
- ✅ Test directory structure in place
- ✅ Test runner script created and executable
- ✅ Jest configuration working (loads successfully)
- ✅ @swc/jest transformer configured
- ✅ Tilt integration complete with automatic service rebuild
- ✅ Test execution verified end-to-end
- 🟡 Tests need additional dependencies: `@testcontainers/postgresql`
- 🟡 Existing test file: `graph-api.test.ts` (1 test discovered)

**Test Files**:
- ✅ `graph-api.test.ts` - ElementService gRPC contract tests (needs @testcontainers)
- ☐ `graph-service.test.ts` - RepositoryService lifecycle tests (to be added)
- ☐ `connectivity.test.ts` - External connectivity tests (to be added)

**Known Considerations**:
- Repository uses Connect-RPC (HTTP/2) not pure gRPC - tests should use @connectrpc/connect client
- Service exposes both gRPC (port 8081) and HTTP REST API - integration tests focus on gRPC
- Database state isolation needed - use transaction rollbacks or test-specific tenant IDs
- Event bus publishing may need mocking or verification via side effects

**Next Steps**:
- [ ] Add `@testcontainers/postgresql` to devDependencies
- [ ] Expand test coverage: graph-service.test.ts, connectivity.test.ts
- [ ] Add database seeding/cleanup logic
- [ ] Document test execution patterns

**Verified via Tilt CLI** (October 26, 2026):
```bash
$ tilt trigger test:integration-graph
→ Repository service rebuilds automatically
→ Test runner executes successfully
→ Jest discovers and runs tests
→ Tests fail only on missing @testcontainers dependency (not infrastructure issue)
```

Target: **✅ Infrastructure completed October 26, 2026**

**Workflow (Temporal)** - Status: ✅ Infrastructure Complete (October 26, 2026)

**Completed Steps**:
1. ✅ **Created test directory** at `services/workflows/src/__tests__/integration/`
2. ✅ **Created test runner script** at `services/workflows/src/__tests__/integration/run_integration_tests.sh`
3. ✅ **Added Jest integration config** at `services/workflows/jest.integration.config.mjs`
4. ✅ **Added @swc/jest dependencies** to `services/workflows/package.json`
5. ✅ **Created Tilt resource** `test:integration-workflows` with `resource_deps=['workflows-service']`
6. ✅ **Verified auto-rebuild** - Service rebuilds automatically before test execution
7. ✅ **Test infrastructure verified** - Jest loads, discovers tests, executes successfully

**Current Status**:
- ✅ Test directory structure in place
- ✅ Test runner script synced into pod
- ✅ Jest configuration working
- ✅ @swc/jest transformer configured
- ✅ Tilt integration complete with automatic service rebuild
- 🟡 Existing test file: `definition-graph.test.ts` (Temporal contract tests)
- 🟡 Tests need Temporal-specific dependencies

**Next Steps**:
- [ ] Create synthetic workflows + teardown logic
- [ ] Capture workflows execution traces as artifacts
- [ ] Add Temporal client contract tests
- [ ] Wire CI (including nightly longer-running suite)

Target: **✅ Infrastructure completed October 26, 2026**

**Platform (tenant management)** - Status: ✅ Infrastructure Complete (October 26, 2026)

**Completed Steps**:
1. ✅ **Created test directory** at `services/platform/src/__tests__/integration/`
2. ✅ **Created test runner script** at `services/platform/src/__tests__/integration/run_integration_tests.sh`
3. ✅ **Added Jest integration config** at `services/platform/jest.integration.config.mjs`
4. ✅ **Added @swc/jest dependencies** to `services/platform/package.json`
5. ✅ **Created Tilt resource** `test:integration-platform` with `resource_deps=['platform']`
6. ✅ **Verified auto-rebuild** - Service rebuilds automatically before test execution
7. ✅ **Test infrastructure verified** - Jest loads, discovers tests, executes successfully

**Current Status**:
- ✅ Test directory structure in place
- ✅ Test runner script synced into pod
- ✅ Jest configuration working
- ✅ @swc/jest transformer configured
- ✅ Tilt integration complete with automatic service rebuild
- 🟡 Existing test file: `platform-service.test.ts` (org/tenant API tests)
- 🟡 Tests need platform-specific dependencies

**Next Steps**:
- [ ] Implement integration fixtures for org/tenant APIs
- [ ] Mock or provision identity providers as needed
- [ ] Add database seeding/cleanup
- [ ] Enable CI gating + dashboards

Target: **✅ Infrastructure completed October 26, 2026**

**Inference-gateway (Go)**  
- [ ] Build the Go `integration.test` binary inside the service image and invoke it from `tests/run_integration_tests.sh`  
- [ ] Add HTTP/gRPC contract checks against backend services  
- [ ] Register config + dependencies  
- [ ] Add Tilt button (`test.inference_gateway`)  
- [ ] Include suite in nightly runs (not PR-blocking initially)  
- Target: **Q3 2026**

---

## Inference Integration Test Suite – Detailed Status

### Current State (October 2026)

**Execution**
- Staging runs execute through the integration runner (`pnpm test:integration -- --filter inference --env=staging`) and reuse the primary service image.
- Local feedback relies on Tilt (`test:integration-inference`), which shells into the live pod via `kubectl exec` and calls `/app/tests/run_integration_tests.sh`.

**Artifacts**
- Pytest writes junit output to `/artifacts/junit.xml`; the orchestrator copies results into `artifacts/integration/inference/<timestamp>/`.

**Test Inventory**
- `pytest --collect-only` currently discovers 25 tests.

```text
services/inference/tests/integration/testcases/test_inference_api.py (15 tests)
  • TestHealthEndpoints (2 tests)
  • TestInvokeEndpoint (5 tests)
  • TestThreadEndpoints (3 tests)
  • TestStreamingEndpoint (2 tests)
  • TestWebsocketEndpoint (1 test, skips if websocket dependency or endpoint unavailable)
  • TestErrorHandling (2 tests)

services/inference/tests/integration/testcases/test_service_connectivity.py (10 tests)
  • TestRepositoryServiceConnectivity (2 tests)
  • TestWorkflowServiceConnectivity (1 test)
  • TestRepositoryToolSimulation (2 tests)
  • TestInferenceServiceHealth (2 tests)
  • TestEnvironmentConfiguration (3 tests)
```

### Coverage Snapshot
- ✅ Happy-path coverage for `/ok`, `/health`, `/agents/invoke`, and `/agents/stream`.
- ✅ Negative-path validation for missing/malformed requests.
- ✅ Connectivity smoke tests to graph and workflows services with basic tool simulations.
- ⚠️ WebSocket coverage limited to a single handshake test that skips when the endpoint is unreachable.
- ⚠️ No explicit assertions for thread persistence, checkpointing, or agent-routing.
- ⚠️ No contract-level checks for platform service gRPC endpoints.

### Dependencies & Configuration
- Relies on the existing inference service deployment and its dependencies (PostgreSQL, graph, workflows).
- Test runner installs `grpcio` on start; other dependencies ship with the baked service image.
- Helm values currently inject only service URLs and junit output path:

```yaml
REPOSITORY_SERVICE_URL: http://graph.ameide.svc.cluster.local:8080
WORKFLOWS_GRPC_ADDRESS: workflows-service.ameide.svc.cluster.local:8086
PLATFORM_SERVICE_URL: http://platform-service.ameide.svc.cluster.local:8082
PYTEST_JUNIT_PATH: /artifacts/junit.xml
```

### Known Issues
1. **gRPC import deprecation warning** – `pytest` surfaces a warning from generated workflows stubs (`ameide_core_proto` import).
2. **No external artifact sink** – Results live only in the workspace (`artifacts/integration/...`). Upload to MinIO/S3 and metrics emission are outstanding.
3. **Staging trigger remains manual** – Runner invocation is still ad hoc; no automated staging gate yet.
4. **Test dependencies installed on every run** – `run_integration_tests.sh` reinstalls pytest and dependencies each time (7-8 seconds overhead)

### Next Actions for Inference Suite
- [x] ~~Fix Tilt trigger to use kubectl exec~~ ✅ Completed October 26, 2026
- [x] ~~Ensure tests persist in image~~ ✅ Fixed `.dockerignore` October 26, 2026
- [ ] Optimize test dependency installation (cache pytest/requirements between runs)
- [ ] Expand coverage for WebSocket edge cases, agent routing, and persistence checkpoints
- [ ] Enforce the pack as a required PR gate once runtime is stable (<10 min)
- [x] Wire artifact uploads (MinIO/S3) via `INTEGRATION_ARTIFACT_*` env vars; duration/failure metrics now ship via `INTEGRATION_RUNNER_PUSHGATEWAY`.
- [ ] Document the staging run playbook in `docs/runbooks/inference-integration.md`
- [ ] Add artifact collection from local test runs (currently `/artifacts/junit.xml` stays in pod)

---

## Relationship to Playwright E2E

**E2E tests are completely separate from integration tests:**

### Discovery Convention
- **E2E tests**: Located in `**/__tests__/e2e/*.{test,spec}.{js,ts}`
- **Integration tests**: Located in `services/*/__tests__/integration/`
- **No overlap**: Integration runner ignores E2E tests; Playwright ignores integration tests

### Current E2E Test Locations
```bash
services/www_ameide_platform/features/
├── threads/__tests__/e2e/
│   ├── threads-connect.smoke.spec.ts
│   ├── threads-connect.error.spec.ts
│   └── ...
├── graph/__tests__/e2e/
├── navigation/__tests__/e2e/
└── workflows/__tests__/e2e/
```

### Execution
- **Playwright config**: `services/www_ameide_platform/playwright.config.ts`
  - `testDir: './features'`
  - `testMatch: '**/__tests__/e2e/**/*.{test,spec}.{js,ts}'`
- **Tilt resources** (all under `test.e2e` label):
  - `e2e-playwright` - Helm-managed Kubernetes Job (auto-rebuilds on code changes)
  - `e2e-playwright-log-tail` - Manual trigger to stream live logs
  - `e2e-playwright-status` - Manual trigger to check job status and diagnostics
  - `e2e-playwright-reset` - Manual trigger to delete job and force re-run
- **Integration orchestration** (under `test.integration` label):
  - `integration-build` - Build any test pack image
  - `integration-plan` - Preview execution plan for staging
  - `integration-run` - Execute test pack in cluster
- Environment overlays in `infra/kubernetes/environments/<env>/integration-tests/e2e/playwright.yaml`

### Log Access
Stream logs from running Playwright job:
```bash
# Via Tilt
tilt trigger e2e-playwright-log-tail

# Direct kubectl
kubectl logs -n ameide-int-tests -l app.kubernetes.io/instance=e2e-playwright -f
```

---

## Recent Changes (October 2026)

### ✅ Playwright E2E Non-Interactive Authentication Complete (October 26, 2026 20:10 UTC)

**Achievement**: Playwright E2E tests now support non-interactive authentication with full test output visible in Tilt logs

**Problem Solved**:
- Previous approach required interactive Keycloak login flow (manual user interaction)
- Test output was not visible in Tilt logs, making debugging difficult
- Job failures were not detected immediately

**Solution Delivered**:
1. **Automated Keycloak SSO** - Playwright global setup drives the Keycloak login UI using `E2E_SSO_*` credentials, then saves state to `tests/.auth/state.json`
2. **Log Streaming** - `build/scripts/run-playwright-job.sh` streams Playwright output into Tilt and CI logs
3. **Job Monitoring** - Background process watches pod status and terminates early on failure
4. **Sample Tests** - Example specs in `services/www_ameide_platform/features/editor/__tests__/e2e/` demonstrate the pattern

**Files Modified**:
- Legacy: `infra/kubernetes/environments/local/integration-tests/playwright.yaml` (now superseded by `infra/kubernetes/environments/local/integration-tests/e2e/playwright.yaml`) - Added auth environment variables
- [`infra/kubernetes/environments/local/platform/www-ameide-platform.yaml`](infra/kubernetes/environments/local/platform/www-ameide-platform.yaml:98) - (legacy) previously added `AUTH_TEST_USERS` for credentials provider; removed in favour of Keycloak-only auth
- [`build/scripts/run-playwright-job.sh`](build/scripts/run-playwright-job.sh:29-71) - Complete rewrite for log streaming
- [`Tiltfile`](Tiltfile:905-998) - shared `e2e-playwright` resource with correct image name
- Created test files:
  - `services/www_ameide_platform/features/editor/__tests__/e2e/editor-smoke.spec.ts`
  - `services/www_ameide_platform/features/editor/__tests__/e2e/editor-basic.spec.ts`
  - `services/www_ameide_platform/features/editor/__tests__/e2e/editor.page.ts`
  - `services/www_ameide_platform/features/editor/__tests__/e2e/README.md`

**Technical Details**:
- **Authentication Flow**: Playwright global setup → `/login` handler → Keycloak UI → admin@local.test session saved to `tests/.auth/state.json`
- **Log Streaming**: Waits for container ready state (up to 60s) → streams logs in foreground → monitors job status in background
- **Exit Codes**: 0 (success), 1 (failure), 124 (timeout)
- **Job Lifecycle**: Create job → wait for pod ready → stream logs → detect completion → exit with status

**Verification**:
```bash
$ tilt trigger e2e-playwright-run
# Output shows:
✅ Authenticated as: admin@local.test
   Roles: admin
✅ Auth state saved to: tests/.auth/state.json

Running 2 tests using 2 workers
  2 passed (12.7s)
```

**Impact**:
- ✅ Fast, non-interactive authentication (<5 seconds)
- ✅ Full test output visible in Tilt logs (no more blind debugging)
- ✅ Immediate failure detection (within 2 seconds)
- ✅ Sample tests demonstrate Page Object Model pattern
- ✅ Ready for CI integration

**Next Steps**:
- Expand test coverage for element editor features
- Add more Page Object Model helpers
- Document authentication setup for new tests

### ✅ TypeScript Services Integration Test Infrastructure Complete (October 26, 2026 18:30 UTC)

**Achievement**: Integration test infrastructure completed for all TypeScript services (graph, workflows, platform)

**What Was Delivered**:
1. **Test Structure** - All services now follow standard `__tests__/integration/` pattern with test runner scripts
2. **Jest Configuration** - Added `@swc/jest` and `@swc/core` dependencies to all TypeScript services
3. **Tilt Integration** - Created `test:integration-*` resources with automatic service rebuild via `resource_deps`
4. **End-to-End Verification** - Tested via Tilt CLI, confirmed all infrastructure working correctly

**Files Created/Modified**:
- `services/graph/__tests__/integration/run_integration_tests.sh` ✅
- `services/graph/jest.integration.config.mjs` ✅
- `services/graph/package.json` (added @swc/jest dependencies) ✅
- `services/workflows/src/__tests__/integration/run_integration_tests.sh` ✅
- `services/workflows/jest.integration.config.mjs` ✅
- `services/workflows/package.json` (added @swc/jest dependencies) ✅
- `services/platform/src/__tests__/integration/run_integration_tests.sh` ✅
- `services/platform/jest.integration.config.mjs` ✅
- `services/platform/package.json` (added @swc/jest dependencies) ✅
- `Tiltfile` (lines 1207-1283: integration tests with resource_deps) ✅

**Tilt Resources Created**:
```python
# All tests use test.integration label, alphabetically ordered
- integration-plan (dry-run execution plan)
- test:integration-inference (depends on 'inference')
- test:integration-platform (depends on 'platform')
- test:integration-graph (depends on 'graph')
- test:integration-workflows (depends on 'workflows-service')
```

**How It Works**:
```bash
# Trigger any test - Tilt automatically rebuilds service first
$ tilt trigger test:integration-graph

# Tilt workflows:
1. Checks if 'graph' service needs updating
2. Rebuilds service if test files changed
3. Waits for service to be ready
4. Executes test runner script inside pod via kubectl exec
5. Jest loads, discovers tests, runs them in-cluster
```

**Verification Results**:
- ✅ Repository: Jest runs, discovers `graph-api.test.ts`, fails only on missing `@testcontainers` dependency
- ✅ Workflow: Jest runs, discovers `definition-graph.test.ts`, fails only on missing test dependencies
- ✅ Platform: Jest runs, discovers `platform-service.test.ts`, fails only on missing test dependencies
- ✅ Inference: 25 tests passing (Python via pytest)

**Status**: Infrastructure 100% complete. Remaining work is adding test-specific dependencies (testcontainers, etc.) and implementing test logic.

**Documentation**:
- [INTEGRATION_TESTS_FINAL_STATUS.md](/workspace/INTEGRATION_TESTS_FINAL_STATUS.md) - Complete status report
- [INTEGRATION_TESTS_VERIFIED.md](/workspace/INTEGRATION_TESTS_VERIFIED.md) - CLI verification results
- [INTEGRATION_TESTS_SETUP.md](/workspace/INTEGRATION_TESTS_SETUP.md) - Setup guide

---

## Recent Changes (October 2026)

### ✅ Convention-Based Discovery Implemented
- **Removed config.yaml requirement**: Integration runner now auto-discovers services with `__tests__/integration/` directories
- **Simplified onboarding**: Create `__tests__/integration/test_*.py` and it's automatically discovered
- **Separated E2E from integration**: Removed Playwright from integration-runner; E2E now runs via the shared `e2e-playwright` Tilt resource (feature filtering handled inside Playwright).
- **Updated discovery logic** ([integration-runner.js:97-147](tools/integration-runner/bin/integration-runner.js#L97-L147))

### ✅ Tilt Resource Reorganization (October 2026)
- **Renamed E2E resources**: `playwright-e2e` → `e2e-playwright`
- **Renamed integration orchestration**: `test:e2e-*` → `integration-*` to separate concerns
  - `test:e2e-build` → `integration-build`
  - `test:e2e-plan` → `integration-plan`
  - `test:e2e-run` → `integration-run`
- **Added E2E log controls** (manual triggers under `test.e2e` label):
  - `<resource>-log-tail` - Stream live logs from running Playwright job
  - `<resource>-status` - Check job/pod status and diagnostics
  - `<resource>-reset` - Delete job to force re-run
- **Clear label separation**:
  - `test.e2e` - Playwright E2E tests and controls
  - `test.integration` - Generic integration orchestration tools

### ✅ Playwright Build Performance Optimization (October 2026)
- **Created `tests/playwright-runner/.dockerignore`**: Reduced build context from 1.53GB → ~100MB (15x smaller)
- **Optimized Dockerfile layer caching**: Separated package.json from source code, pnpm install --no-frozen-lockfile now cached (41s → 2s on cache hit)
- **Enhanced Tiltfile configuration**: Specific deps list, ignore patterns, BuildKit inline cache enabled
- **Performance improvements**:
  - First build (no cache): 173s → 64s (2.7x faster)
  - Code change (deps cached): 173s → 22s (7.8x faster)
  - Test file change (all cached): 173s → 8s (21x faster)

### Current Status
```bash
$ pnpm test:integration -- --list
2 test pack(s) discovered:
inference           docker.io/ameide/inference                timeout=600
www-ameide          docker.io/ameide/www-ameide               timeout=600
```

### Migration Path for Services
1. **Create integration tests**: `mkdir -p services/myservice/__tests__/integration`
2. **Add test files**: `touch services/myservice/__tests__/integration/test_api.py`
3. **Auto-discovered**: Run `pnpm test:integration -- --list` to verify
4. **No config needed**: Pack metadata auto-generated from conventions

---

## Follow-up Backlog

### Immediate Next Steps
- Add `__tests__/integration` to remaining services (graph, workflows, platform, inference_gateway)
- Update `.dockerignore` to include integration tests for other services (if needed)
- Update www-ameide test runner configuration (currently no tests exist)
- Document the local vs CI/staging test execution patterns in service READMEs

### Tilt Integration (October 2026) - ✅ COMPLETED
- ✅ Fixed `.dockerignore` to include integration test files (was excluding `*_test.py` and `test_*.py`)
- ✅ Removed generic `integration-run` resource (designed for CI/staging, incompatible with local kubectl exec pattern)
- ✅ Removed `integration-build` resource (not needed for local dev)
- ✅ Created per-service test resources: `test:integration-inference` (working with 25 tests passing)
- ✅ Updated integration-runner to skip execution in local mode (shows plan + instructions only)
- ✅ No cross-dependencies between test resources and services in Tilt
- ✅ Verified tests persist across pod restarts (baked into image correctly)

### Future Enhancements
- Add external artifact sinks (MinIO/S3 uploads) and metrics emission (Prometheus/Pushgateway) to the runner
- Introduce optional log aggregation (Loki) once storage/metrics plumbing is in place
- Apply E2E build optimizations pattern to other test pack images (layer caching, specific .dockerignore)
- Consider adding test result caching to avoid re-installing pytest dependencies on every run

---

## Open Questions

1. Policy for long-running jobs—do we abort after N minutes or allow per-team overrides?  
2. How to surface integration flakes in engineering dashboards (Looker vs Grafana).  
3. Requirements for running packs against staging/prod-like namespaces (feature environments).  
4. What release criteria unlock Playwright runner execution in production (helm toggle ownership, alerting)?

---

## Next Actions

- [x] Platform: deliver `tools/integration-runner` with execution modes + documentation.
- [x] Platform: publish Helm chart + example values (`infra/kubernetes/charts/test-job-runner`).
- [x] Platform: reorganize Tilt resources (e2e-playwright, integration-*, log controls).
- [x] Platform: optimize Playwright build performance (layer caching, reduced context).
- [x] Frontend team: consolidate Playwright specs into the shared runner and wire staging/prod Helm overlays.
- [ ] Inference team: enable CI gating with the new pack + ensure image publishes gate merges.
- [ ] Repository team: copy integration tests into the service image and expose a `run_integration_tests.sh` wrapper.
- [ ] Workflow team: design Temporal fixtures + cleanup tooling.
- [ ] Observability: add Prometheus metrics spec to `docs/observability/integration-tests.md`.
- [x] Platform: emit Pushgateway metrics from the integration runner (see `INTEGRATION_RUNNER_PUSHGATEWAY`).
- [ ] Platform: add MinIO upload tooling for integration and E2E artifacts.
- [ ] Platform: apply E2E build optimization pattern to other test pack images.

---

## References

- `services/inference/tests/integration/` – first adopter pack (containerised January 2026)  
- `infra/kubernetes/charts/test-job-runner/` – shared Job template and values  
- `tools/integration-runner/README.md` – orchestrator usage and execution modes  
- Backlog links: [301-threads.md](./301-threads.md), [305-workflows.md](./305-workflows.md), [304-context.md](./304-context.md)

---





Playwright behave exactly like the other AMEIDE services

Here’s the concrete plan to make Playwright behave exactly like the other AMEIDE services—Tilt owns the build + Helm release, and the integration runner only executes against what Tilt already deployed.

1. Reset the Helm values / config
Revert any hard‑coded tags (e.g., the tilt-local override) in:

infra/kubernetes/values/e2e/playwright.yaml
infra/kubernetes/environments/local/integration-tests/e2e/playwright.yaml
tests/playwright-runner/config.yaml
Those files should go back to the neutral state where the image is either absent or set to a generic placeholder (like docker.io/ameide/playwright-e2e:latest). Tilt will inject the real tag when it deploys.

2. Let Tilt build & push the image
Keep the existing custom_build('docker.io/ameide/playwright-e2e', ...) section, but rely on Tilt’s digest tagging (the $EXPECTED_REF, e.g. docker.io/ameide/playwright-e2e:tilt-abcdef1234567890).
Ensure the custom build still copies all Playwright sources and finishes with a k3d image import (as it already does) so the cluster sees the exact Tilt tag.
No changes needed here, other than making sure we’re no longer hard-coding a secondary tag like tilt-local.

3. Add a dedicated Helm resource in Tilt
Create a helm_resource in the Tiltfile so Tilt applies the job chart using the built image:

# After the custom_build
playwright_helm = helm_resource(
    name='e2e-playwright',
    chart='infra/kubernetes/charts/test-job-runner',
    namespace='ameide-int-tests',
    values=[
        'infra/kubernetes/values/e2e/playwright.yaml',
        'infra/kubernetes/environments/local/integration-tests/e2e/playwright.yaml',
    ],
    set=[
        # Tilt substitutes the exact digest tag it just built
        'image='+tilt_substitute('docker.io/ameide/playwright-e2e'),
    ],
    image_deps=['docker.io/ameide/playwright-e2e'],
    resource_deps=['infra-health-check'],  # optional if infra must be ready
)
Key points:

tilt_substitute('docker.io/ameide/playwright-e2e') tells Tilt to inject the precise Tilt tag into Helm.
image_deps wires rebuilds (when the image changes, Tilt reapplies Helm automatically).
No runner involvement—Tilt is the one pushing and deploying the job.
If the chart expects additional values (namespace, env vars, etc.), keep them in the values file; Tilt will merge the injected image tag with those.

4. Update the integration runner’s local behaviour
Once Tilt owns the release, the runner should not do another helm upgrade. Instead, for --env=local runs:

Add a guard in tools/integration-runner/bin/integration-runner.js (within executeClusterPack) so that when envKey === 'local', it skips the Helm apply phase. It can log something like:

console.log(gray('   Skipping Helm deploy for local run (Tilt-managed).'));
The runner should still:

Check the job exists (Tilt put it there).
Stream logs / fetch artifacts.
Wait for completion.
Any build logic inside the runner (--build-only) is unnecessary for local runs now, so the runner may skip builds when envKey === 'local'.

5. Optional helpers
If you want a manual “rerun” button in Tilt, create a local resource that simply deletes/recreates the job:

local_resource(
    name='playwright-e2e-rerun',
    cmd='bash -lc "kubectl -n ameide-int-tests delete job playwright-e2e-test-job-runner --ignore-not-found && kubectl -n ameide-int-tests create job --from=cronjob/<or release> ..."',
    trigger_mode=TRIGGER_MODE_MANUAL,
    deps=[],
)
But strictly speaking, Tilt will redeploy the job each time the image changes. For manual re-runs, you can also keep a simple script that deletes the job so Helm re-creates it on the next sync.

6. Integration runner documentation
Update any docs or README to clarify:

“In local mode, Tilt is responsible for building the image and applying the playwright-e2e job. The integration runner only tails logs and waits for completion. Do not pass --build-only or run pnpm test:e2e with --env=local expecting it to deploy the job.”
Summary
Responsibility	Owner	What changes
Build & tag Playwright image	Tilt	Keep custom_build, remove ad-hoc tags
Push/import to k3d registry	Tilt	Already handled
Deploy Helm release	Tilt	Add helm_resource with image_deps + tilt_substitute
Run job / stream logs	Integration runner	Skip Helm apply in local mode
With this setup, the job image always matches Tilt’s digest tag, and the runner simply executes against the already-deployed environment, matching the rest of the AMEIDE platform workflows.

**Status:** ✅ Implemented via `tests/playwright-runner/config.yaml`, the `pnpm test:e2e` wrapper, and the integration runner’s `--env=local` flow that now tails the Tilt-managed Job instead of reapplying Helm.

---

## Implementation progress (image refs)

### 2025-12-26

- This work aligns with 602/603: local inner-loop should still be deterministic at the “deployed artifact” level (digest-pinned refs), with automation driving updates rather than floating tags.
