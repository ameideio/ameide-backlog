> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# backlog/344 – Packages Runners Refactor

## Context
- The pre-refactor layer 29 Helm job relied on a single `packages-publisher` image (~4 GB) to publish and smoke-test Go, Python, and TypeScript packages; the new approach runs slimmer per-language runners inside layers 26–28.
- That image embeds the entire repository, invalidating cache on every change and slowing downloads in Tilt and CI.
- Secrets are already exposed via per-language ExternalSecrets; the runner image is only needed to provide tooling and scripts.

## Alignment & Principles
- **Backlog 344 (Local Package Registry Pipeline Refresh)**  
  - Keep the `publish → smoke → consume` contract intact; charts must still persist language state into ConfigMaps consumed by Tilt/CI.  
  - Preserve “no baked-in defaults” by failing fast when required env vars (from Vault secrets) are missing; no fallbacks baked into images.
- **Backlog 345 (CI/CD Unified Playbook)**  
  - Runner images must export the same `AMEIDE_{LANG}_SDK_VERSION` environment contracts so CI, Tilt, and Helm reuse identical interfaces.  
  - Jobs publish version metadata as build outputs compatible with CI artefact expectations.
- **Backlog 346 (Tilt Packages Logging Harmonization)**  
  - All job entrypoints source the shared logger (`scripts/lib/logging.sh`, adapters) so publish/smoke logs remain prefixed (`[packages:<lang>-<phase>]`).  
  - Helm Jobs should wrap script invocations with the standard `run_step` helpers to keep Tilt/CI panes aligned.
- **Backlog 347 (Service Tests Refactorings)**  
  - Downstream services must continue resolving SDKs through the internal registries; ConfigMap state remains the authoritative source for snapshot versions.  
  - Helm jobs may not introduce shortcuts (e.g., copying `dist/` artefacts); they must use the publish scripts exactly as services do.
- **Backlog 348 (Environment & Secrets Harmonization)**  
  - Secrets stay in Vault-managed `packages-*-env` ExternalSecrets; charts must not inline credentials.  
  - Environment overlays remain responsible for per-environment differences; runner charts stay environment-agnostic aside from image tags.

## Objectives
- Minimise container footprint while keeping credentials managed by Kubernetes secrets.
- Preserve the publish → smoke → ConfigMap pipeline for each language with no behavioural regressions.
- Allow Tilt to trigger language-specific flows with selector-based Helm invocations and consistent logging.
- Ensure runner images rebuild only when their language-specific inputs change and respect CI/CD contracts.

## Target Architecture
1. **One chart per language**
   - Split the current smoke chart into `package-go`, `package-python`, and `package-ts`.
   - Each chart renders a single Helm test Job and accepts common values (namespace, image tag, state ConfigMap name).
   - Jobs mount an `emptyDir` at `/workspace/tmp/tilt-packages` so publish and smoke steps share state.
  - Jobs copy JSON outputs into language-specific ConfigMaps (`go-packages-state`, `python-packages-state`, `ts-packages-state`) and now exercise representative upstream packages (Go `github.com/google/uuid`, npm `lodash`, pip `requests`) to verify both proxying and cache hits before marking the layer healthy.
2. **Slim per-language images**
   - New Dockerfiles under `infra/docker/local/images/package-publisher-{go,python,ts}`.
   - Base on language-specific slim images (e.g. `golang:<version>-alpine`, `ghcr.io/astral-sh/uv:python<version>`, `node:<version>-slim` + pnpm).
   - Copy only `scripts/`, `release/`, and the relevant `packages/*` subtree required by that language.
   - Add targets to `infra/docker/local/docker-bake.hcl` and extend `scripts/infra/build-local-packages-images.sh` to build/push all three.
3. **Helm orchestration**
   - Update `infra/kubernetes/helmfiles/26-package-npm.yaml`, `27-package-go.yaml`, and `28-package-py.yaml` to include the new charts with language-specific selectors.
   - Keep common job defaults (timeout, RBAC) centrally via shared values files.
   - Ensure each layer waits for its registry deployment before running the runner job.
4. **Tilt integration**
   - Drop language-specific publish `local_resource`s in favour of the existing `infra:<layer>` resources that call `run-helm-layer.sh` for layers 26–28.
   - After a job completes, invoke `infra/kubernetes/sync-*-packages-state.sh` helpers to fetch ConfigMap outputs into `tmp/tilt-packages/`.
   - Downstream Tilt resources depend on the synced state files instead of direct script execution, keeping service tests aligned with backlog 347 guidance.
5. **Image lifecycle**
   - Build/push runner images during environment bootstrap (same stage as registry deployment).
   - Tag with `dev` (local registry) and promote to `latest` in CI so clusters reuse cached layers.
   - Rebuild on language runtime updates or when relevant directories change; document triggers in the Dockerfiles.

## Implementation Notes (Go runner delivered)
- **Slim image** – `infra/docker/local/images/package-publisher-go/Dockerfile` installs only Go + kubectl, then copies `scripts/`, `release/`, and the Go package trees. `scripts/infra/build-local-packages-images.sh` bakes it under the `packages-publisher-go` target and imports it into k3d.
- **Shared logger & timing** – Runner scripts source `scripts/lib/logging.sh` to emit timestamped, duration-tagged log lines (`run_step` now prints “took Ns”).
- **Chart packaging** – `infra/kubernetes/charts/package-go-runner` renders a single Helm test Job with RBAC, ServiceAccount, and an `emptyDir` at `/workspace/tmp/tilt-packages`. The job runs both publish and smoke scripts and rewrites `go-packages-state`.
- **Secrets** – `packages-go-env` ExternalSecret now includes `GO_REGISTRY_TOKEN`, pulled from Vault key `packages/go/upload_token`, enabling authenticated uploads.
- **Selector discovery** – The chart declares `helmLayerSelector: "language=go"`. `run-helm-layer.sh` inspects values files and applies the selector automatically, so Tilt/CI need no extra env vars.
- **Tilt wiring** – `Tiltfile` now relies on `infra:26/27/28` resources; editing runner charts triggers the same Helm layer resources so logs stream back automatically.
- **State sync scripts** – `infra/kubernetes/sync-go-packages-state.sh` remains the bridge to local `tmp/tilt-packages/` after the job succeeds so dependent services can consume the new versions.
- **DNS alignment** – Runner scripts now rely on the CoreDNS overrides shipped in Helm layer 18; no `/etc/hosts` manipulation is required.
- **Telemetry parity** – Publish helpers now resolve the git SHA via `SOURCE_GIT_SHA` overrides so the runner works from a copied workspace without `.git`.

## Lessons Learned
- **Chart-owned selectors > Tilt overrides** – Keeping selectors in chart values avoids hard-coded Tilt env vars and works across CI and local runs; the helper can auto-detect them.
- **Hooked jobs need log harvesting** – Helm’s native `--logs` isn’t reliable for Jobs. Collecting pod logs via label query ensures failures surface in Tilt/CI panes.
- **Deleting existing Jobs breaks iterations** – Converting the runner to a `helm.sh/hook: test` with `before-hook-creation` lets Helm recreate the Job cleanly without manual deletes.
- **Vault keys must be end-to-end** – Authenticated registries need credentials exposed through ExternalSecrets; failing to include them surfaced immediately as 401 responses in the runner logs.
- **Runner images need kubectl** – Waits and ConfigMap updates happen in-cluster; the slim image must carry kubectl even if we keep the rest minimal.
- **Logger improvements matter** – Adding durations and timestamps paid off when tracking down slow steps and verifying the telemetry publish executed.
- **State cleanup should not remove mounts** – Use `find` to clear contents instead of `rm -rf` on the mount path; otherwise Jobs fail with “Device or resource busy”.
- **Script portability** – Publish helpers must work without git metadata (set via `SOURCE_GIT_SHA`) so runner images, CI, and Tilt can reuse them.


## Open Questions
- Should ConfigMap updates move to a separate “state-sync” chart to avoid rewriting from every job rerun?
- Do we need a follow-up Tilt target to clean up old pseudo-version JSON when the Helm run fails mid-way?
- How do we surface per-language status in CI dashboards (one aggregate stage vs three parallel stages)?
- Add chart-level tests (template assertions + optional integration smoke) so the per-language runner jobs stay covered as we iterate.
