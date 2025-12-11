> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.


# Python Packages Architecture (devpi) _(Archived)_

> **Status – superseded (2026-02-17):** The devpi-backed Python registry was removed when SDK distribution moved to Buf Managed Mode. The instructions below are historical only.

## Principles
- No baked-in defaults: scripts must fail if required environment variables or credentials are missing; Helm/Vault provide the values per environment.
- Single source of truth: Tilt/CI read the same env vars as production (no per-script overrides).
- Publish → smoke → consume: every artifact is tested via dedicated smoke scripts before any dependent builds run.
- No fallbacks or interim migrations: run the target architecture only; if requirements change, update docs and scripts rather than adding transitional code.

## Goals
- Serve Ameide Python wheels/sdists from an in-cluster mirror that also caches third-party packages.
- Keep all environments aligned on the single host `pypi.packages.dev.ameide.io`.
- Execute publish → smoke via Helmfile-managed jobs to avoid per-tooling drift.

## Components
- **devpi Registry (`devpi-registry`)**
  - Deploy devpi-server via `infra/kubernetes/charts/platform/devpi-registry` (new chart) with PVC persistence.
  - Enable mirror mode against `https://pypi.org/simple` (`root/pypi` index) so third-party wheels cache on first use.
  - Expose the internal index for Ameide packages (e.g., `root/ameide`) configured for uploads.
  - Authentication: anonymous read in local, token/htpasswd in staging/prod (per-env secrets).
- **Gateway Routing**
  - Envoy HTTPRoute keeps the existing hostname `pypi.packages.dev.ameide.io` and forwards to the devpi service.
- **Storage**
  - PVC sizes: 5 Gi (local), 20 Gi (staging), 50 Gi (production); evaluate object storage if growth accelerates.

## Cross-Cutting Topics
- **Helmfile Test Integration** – Layer 28 deploys devpi and immediately runs the `package-python-runner` Helm test to publish and smoke Python packages for every environment.
- **Secret Sourcing & Vault** – Vault seeds devpi credentials at `secret/data/packages/python`, mirrored into Kubernetes for jobs and workloads.
- **State Artifact Persistence** – Helm jobs emit `python.json` into the `python-packages-state` ConfigMap; local workflows sync those files into `/workspace/tmp/tilt-packages/`.
- **CI Alignment** – Release workflows call Helmfile with tests enabled, reusing the same publish automation and values overrides.
- **Tilt / Local Dev Flow** – Tilt ties builds to `packages:health`, consuming the ConfigMap instead of running local publish scripts.
- **Deployment Health & Rollout** – `deployment/devpi-registry` readiness and PVC binding gate consumer traffic during rollouts.
- **Routing & Gateway** – Envoy forwards `pypi.packages.dev.ameide.io` to devpi, keeping ingress consistent across environments.
- **Persistence & Storage Strategy** – Per-environment PVC sizing with planned evaluation of object storage as volumes grow.
- **Observability & Alerts** – Prometheus exporter + ServiceMonitor expose bootstrap state, storage usage, and mirror freshness for alerting.
- **Documentation & Onboarding References** – Python onboarding links here to explain the Helmfile-first devpi workflow and required env vars.

## Workflows
### Helmfile / Environment Sync
1. Helmfile layers 25 and 28 deploy the routing + devpi workloads. Layer 28 refreshes only the Python-specific images (`devpi-registry`, `packages-publisher-python`, `kubectl`, `curl`) through the shared `scripts/infra/build-local-packages-images.sh` helper before the Helm test runs, keeping re-runs quick without risking stale caches.
2. The layer 28 `package-python-runner` Helm test runs `scripts/tilt-packages-python-publish.sh`, followed by the smoke scripts, using the slim `packages-publisher-python` image; the smoke pass now downloads `requests==2.32.3` through devpi and revalidates the cached artifact via the `+files` link, confirming remote mirrors stay warm. SDK wheels stamp snapshot versions (e.g. `0.1.0.dev20251106123000+gabcd1234ef56`) so reruns never collide with existing artifacts. DNS for `pypi.packages.dev.ameide.io` is handled by the CoreDNS overrides deployed in layer 18, and the targeted image imports mean developers no longer have to pre-build registry images manually.
3. Devpi credentials and index defaults (`DEVPI_USER`, `DEVPI_PASSWORD`, `AMEIDE_PYPI_*`, `PIP_INDEX_URL`, `PIP_TRUSTED_HOST`, `UV_DEFAULT_INDEX`) are sourced from Vault-seeded secrets and mounted into the pod.
4. The runner writes `python.json` into the `python-packages-state` ConfigMap, and `infra/kubernetes/sync-python-packages-state.sh` copies it to `/workspace/tmp/tilt-packages/` for consumers.

### CI / Release
1. CI workflows invoke Helmfile with tests enabled to reuse the same publish/smoke automation.
2. For release trains, the workflow can pass explicit snapshot values (`AMEIDE_PYTHON_SDK_VERSION`) via Helmfile values to stamp immutable artifacts; otherwise the publish jobs derive timestamped versions automatically.
3. Optional cache warm-up still runs via Helm chart parameters (`preseed`) during sync.

### Consumers
- Dockerfiles install the latest snapshot from devpi without pinning:
  ```bash
  export PIP_INDEX_URL="http://pypi.packages.dev.ameide.io/root/ameide/+simple/"
  export PIP_TRUSTED_HOST="pypi.packages.dev.ameide.io"
  # legacy command removed in 2026 once Buf-managed wheels replaced the bespoke telemetry package
  ```
- When third-party packages are required, devpi’s mirror serves them; no `PIP_EXTRA_INDEX_URL` fallback is permitted in the target state.
- Integration tests and runtime builds rely solely on the internal mirror (no editable installs).

## Helmfile Wiring
- Helm tests mount the devpi secret and shared state volume, run publish+smoke, and save JSON outputs.
- Tilt resources depend on the Helm job via `packages:health` so services only build once versions are refreshed; the Tiltfile no longer runs Python publish scripts directly.
- Vault seeder seeds `secret/data/packages/python` with devpi credentials, upload URLs, and pip index defaults for local parity.

## Implementation Progress
- 2025-02-20 – Completed devpi migration: Helm chart, publish/smoke tooling, Docker consumers, cache pre-seeding, and Prometheus metrics are all aligned on `root/ameide`.
- 2025-11-05 – Retired the dedicated proxy service; Python publish/smoke jobs now rely on the shared `envoy` alias and the canonical `http://pypi.packages.dev.ameide.io/root/ameide/+simple/` host (no `:8080` dependency).

## Testing Strategy
- **Unit**: `uv pip compile`, `uv pip sync`, `pytest` for the SDK before publishing.
- **Helm Smoke**: Publish + `curl` simple index + optional `pip download` run inside the Helm test job and fail the release if any step errors.
- **Integration**: Consumers read the Helm job’s ConfigMap and install via the internal index; additional e2e tests can confirm runtime builds succeed using those versions only.

## Implementation Success Criteria
- [ ] Helmfile package tests succeed, proving SDK wheels are published and fetchable from devpi.
  - Test: `helmfile sync -e local --include-tests --selector name=packages-python`
- [ ] devpi deployments stay healthy in all environments with persistence enabled.
  - Test: `kubectl -n ameide rollout status deployment/devpi-registry && kubectl -n ameide get pvc -l app.kubernetes.io/name=devpi-registry`
- [ ] Third-party dependencies resolve via the devpi mirror without reaching public PyPI.
  - Test: `kubectl -n ameide logs job/packages-python --tail=-1 | rg "Using cached"`
- [ ] CI/Docker builds use only `pypi.packages.dev.ameide.io`; no `PIP_EXTRA_INDEX_URL` fallback remains.
  - Test: `rg "pypi.packages.dev.ameide.io" -g 'requirements*.txt' services packages && rg "PIP_EXTRA_INDEX_URL" -g 'Dockerfile*' -L`
- [ ] Observability monitors devpi metrics/health endpoints with alerts on cache miss storms or publish failures.
  - Test: `kubectl -n monitoring get servicemonitor devpi-registry`

## Environment Variables
*All environment variables below must be populated via Helm/vault-provisioned secrets; scripts should not carry defaults.*
- `PIP_INDEX_URL=http://pypi.packages.dev.ameide.io/root/ameide/+simple/`
- `PIP_TRUSTED_HOST=pypi.packages.dev.ameide.io`
- `UV_DEFAULT_INDEX=http://pypi.packages.dev.ameide.io/root/ameide/+simple/`
- `DEVPI_USER`, `DEVPI_PASSWORD` (for authenticated uploads)
- `AMEIDE_PYPI_REPOSITORY_URL`, `AMEIDE_PYPI_USERNAME`, `AMEIDE_PYPI_PASSWORD` (publish credentials consumed by the Tilt/release scripts).

## Implementation Tasks
- [x] Build or vendor a `devpi-registry` Helm chart with persistence, mirror configuration, and auth hooks.
- [x] Update Helmfile layer 28 to deploy devpi and remove pypiserver assets.
- [x] Configure secrets (`devpi-admin` token for uploads, read-only tokens for CI) per environment. *(Local chart defaults embed dev credentials; staging/prod reference external secrets.)*
- [x] Update publish scripts to target devpi URLs and handle authentication via environment variables.
- [x] Adjust smoke scripts to query devpi’s simple index.
- [x] Update Dockerfiles/Tilt to use the new index path (`…/root/ameide/+simple/`).
- [x] Pre-seed critical third-party packages and document cache warm-up procedures.
- [x] Instrument devpi with Prometheus scraping (process metrics + custom HTTP stats) and define basic SLOs (e.g., publish latency, cache hit ratio).

## Current Status
- Devpi chart and supporting Docker image landed; Helmfile + Gateway wiring point to the new service across all environments.
- Local/staging/production overrides capture storage sizing, upload credentials, optional mirror pre-seeding, and Prometheus ServiceMonitor placement.
- Publish/smoke scripts and consumer Dockerfiles now honor the `root/ameide/+simple/` index path; metrics exporter surfaces registry health to Prometheus.

## Cache Warm-Up & Pre-Seeding
- The devpi image now honours `DEVPI_PRESEED_*` environment variables (plumbed via Helm values) to download and stage third-party wheels into `root/pypi` during bootstrap.
- Configure packages with `infra/kubernetes/environments/<env>/platform/devpi-registry.yaml`:

  ```yaml
  preseed:
    enabled: true
    packages:
      - "grpcio==1.62.1"
      - "protobuf==4.25.3"
    requirements:
      - "pydantic==2.7.0"
    pipArgs:
      - "--only-binary=:all:"
  ```

- Set `preseed.refresh: true` to re-run warm-up on every rollout (useful for keeping long-lived mirrors current) or `DEVPI_FORCE_BOOTSTRAP=true` for a one-off refresh.
- Pre-seeded artifacts upload directly into the `root/pypi` mirror, so subsequent SDK publishes and CI jobs avoid the initial cold-miss latency against public PyPI.

## Observability & SLOs
- The container now ships with a lightweight Prometheus exporter (`devpi-metrics-exporter.py`) exposing:
  - `devpi_up`, `devpi_status_scrape_seconds` — liveness and status scrape timings.
  - `devpi_storage_{capacity,used}_bytes` — PVC utilisation.
  - `devpi_mirror_serial`, `devpi_pypi_serial` — mirror sync progress against upstream.
  - `devpi_bootstrap_{state,timestamp_seconds}` — installation/refresh traces.
- Metrics are served on port `9150` (configurable via `metrics.port`) and scraped through a `ServiceMonitor` when Prometheus Operator CRDs are installed. Override the target namespace or attach custom labels via:

  ```yaml
  metrics:
    serviceMonitor:
      namespace: monitoring
      labels:
        release: kube-prometheus-stack
      interval: 30s
      scrapeTimeout: 10s
  ```

- Suggested SLO starter points:
  - **Publish success:** `devpi_up == 1` and smoke scripts passing; alert on two consecutive failures.
  - **Mirror freshness:** `devpi_mirror_serial` within 5 minutes of `devpi_pypi_serial`.
  - **Capacity headroom:** `devpi_storage_used_bytes / devpi_storage_capacity_bytes < 0.8`; alert at 70% (warning) and 85% (critical).
