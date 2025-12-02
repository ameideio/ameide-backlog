
# Backlog 344: Local Package Registry Pipeline Refresh _(Archived)_

> **Status – superseded (2026-02-17):** The Verdaccio/devpi/Athens stack has been decommissioned in favour of Buf’s Managed Mode and BSR-hosted SDKs. This document is retained for historical reference only; none of the procedures below reflect the active platform. Modern guidance lives in backlog/356-protovalidate-managed-mode.md and backlog/357-protovalidate-managed-mode_impact.md.

## Immediate follow-up for the GHCR rollout
- **Fail fast, no fallbacks:** every build or bootstrap script must exit non-zero with a descriptive error if `GHCR_USERNAME` or the PAT (`GHCR_TOKEN`/`GITHUB_TOKEN`) is missing. Silent fallbacks (e.g., trying anonymous Docker Hub pulls) are now forbidden—surface the exact variable that is unset and stop.
- **Multi-arch coverage:** every service image referenced in `gitops/ameide-gitops/sources/values/_shared/apps/*.yaml` now points at `ghcr.io/ameideio/*`. Publish those images as multi-arch (or at least `linux/arm64`) so the k3d cluster can pull them. Once the registries contain the matching manifests, run `argocd app sync apps-<component>` to clear the current `ImagePullBackOff` state.
- **Mandatory `.env` entries:** add the following to your repo-local `.env` (already gitignored) before opening the devcontainer:
  ```
  GHCR_USERNAME=<your GitHub login or robot user>
  GITHUB_TOKEN=<PAT with repo + read:packages scopes>
  ```
  The devcontainer bootstrap now fails immediately if those are absent because it can no longer mint the Argo repo credential or the `ghcr-pull` secret without them.

## Principles
- No baked-in defaults: scripts must fail if required environment variables or credentials are missing; Helm/Vault provide the values per environment.
- Single source of truth: Tilt/CI read the same env vars as production (no per-script overrides).
- Publish → smoke → consume: every artifact is tested via dedicated smoke scripts before any dependent builds run.
- No fallbacks or interim migrations: run the target architecture only; if requirements change, update docs and scripts rather than adding transitional code.

## Context
- Three language-specific package registries keep development, CI, and runtime builds decoupled from workspace sources.
- Every language follows a `publish → smoke-test → consume` contract so breakages surface before downstream services build.
- Hostnames: `go.packages.dev.ameide.io`, `pypi.packages.dev.ameide.io`, `npm.packages.dev.ameide.io`, plus `docker.io` for container images.
- Kubernetes layers 25–28 deploy the package registries and run the language-specific publish/smoke Helm tests that persist version metadata; Tilt and CI simply wait on those tests.

## Architecture Overview
| Language   | Registry Host                      | In-Cluster Components                                  | Helm Test Reference            |
|------------|------------------------------------|--------------------------------------------------------|--------------------------------|
| Go         | `http://go.packages.dev.ameide.io`   | Athens proxy (`athens-proxy`) + uploader sharing PVC   | Layer 27 `package-go-runner` Helm test |
| Python     | `http://pypi.packages.dev.ameide.io` | devpi registry (mirror + upload indices)               | Layer 28 `package-python-runner` Helm test |
| TypeScript | `http://npm.packages.dev.ameide.io`  | Verdaccio (`npm-registry` + `verdaccio-route`) + runner | Layer 26 `package-ts-runner` Helm test |
| .NET (planned) | `http://nuget.packages.dev.ameide.io` (proposed) | NuGet proxy (Nexus/Artifactory or similar) to mirror nuget.org | _TBD (see Outstanding Work)_ |

Detailed wiring, Helm test definitions, and environment variables live in the per-language documents listed above.

## Cross-Cutting Topics
- **Helmfile Layer Execution** – `infra/kubernetes/scripts/run-helm-layer.sh` (backlog #339) runs each layer; layers 25–28 stand up shared infrastructure, language registries, and their publish/smoke Helm tests.
- **Secret Sourcing & Vault** – `scripts/vault/ensure-local-secrets.py` seeds registry credentials/env vars consumed by Helm tests and workloads.
- **State Artifact Persistence** – Helm jobs write version JSON to the language-specific ConfigMaps; helper scripts sync them to `/workspace/tmp/tilt-packages/` for dev/CI access.
- **CI / Tilt Alignment** – Tilt `infra:<layer>` resources and CI wrappers both depend on the same Helm layer tests, keeping local and pipeline flows identical.
- **Routing & Storage** – Envoy exposes `*.packages.dev.ameide.io`; Athens/devpi/Verdaccio remain PVC-backed today with roadmap items for RWX/object storage.
- **In-cluster Resolution** – Helm layer 18 (`helmfiles/18-coredns-config.yaml`) programs CoreDNS so `registry|npm|go|pypi.packages.dev.ameide.io` resolve to Envoy inside the cluster; runners rely on standard DNS (no `/etc/hosts` hacks).
- **Observability & Runbooks** – Helm smoke tests plus Prometheus metrics feed alerting; per-language backlogs capture runbook links and remaining tasks.

## Shared Publish → Verify Loop
1. Run layers 25–28 to deploy shared routing and the Verdaccio, Athens, and devpi workloads (`infra/kubernetes/scripts/run-helm-layer.sh infra/kubernetes/helmfiles/25-package-registries.yaml` … `28-package-py.yaml`); each layer’s Helm tests now drive the language-specific publish + smoke cycles.
2. The `package-ts-runner`, `package-go-runner`, and `package-python-runner` Helm tests reuse the slim language images to publish SDK artifacts and execute the smoke suites (e.g., Node 22 ESM import coverage for `@ameideio/ameide-sdk-ts`).
3. Each runner job persists resolved versions as JSON in ConfigMaps (`go-packages-state`, `python-packages-state`, `ts-packages-state`) before Tilt/CI sync them into `tmp/tilt-packages/`.
4. Local developers/CI sync those JSON artifacts into `tmp/tilt-packages/` using the provided scripts before building dependent services.
5. Consumers build against the internal registries only—no workspace fallbacks or editable installs remain.

## Tooling Alignment
- `infra/kubernetes/scripts/run-helm-layer.sh` orchestrates layers (per backlog #339); layers 25–28 deploy the registries and run the language-specific publish/smoke suites. The helper auto-builds/imports the slim `packages-publisher-{ts,go,python}` images unless `SKIP_PACKAGES_PUBLISHER_BUILD=1` is set.
- Tilt resources depend on `infra:<layer>` so workloads wait for successful Helm tests.
- CI pipelines call the same layer scripts/helmfile selectors; Vault-sourced secrets power publish/test environments.

## Outstanding Work
1. **CI Alignment** – Harden Helmfile wrappers in GitHub Actions (timeouts, retries) and document how release workflows override package versions.
2. **Go Hardening** – Document retention/immutability expectations, add a smoke test for private upstream access, and evaluate switching staging/prod storage to an object-store backend.
3. **npm Operations** – Wire alerting on the Verdaccio SLOs and revisit RWX/vendor options once availability requirements exceed the single-replica window (baseline smoke now asserts Node 22 import/exports compatibility).
4. **.NET Alignment** – Define the NuGet mirror architecture (hostname, proxy choice, publish/smoke flow) and capture it in a dedicated backlog doc so all four ecosystems share the same pattern.
5. **Publisher Image Footprint** – Track the size of `packages-publisher-{ts,go,python}` images (<1 GB target); prune dependencies or adjust bases if they start creeping up.
6. **Envoy Port Ownership** – Registry charts still hard-code service ports/host aliases; consolidate external host/port mapping in Envoy routes and scrub redundant port config from per-service values.
7. **Publication Workflow Split** – Evaluate moving publish/smoke out of the Helm test path (e.g., dedicated K8s job/controller) so infrastructure layers deploy without rebuilding the publisher image, while still gating consumers on successful artifact publication.

## Implementation Success Criteria
- [ ] Layers 25–28 apply cleanly (shared routing plus Verdaccio, Athens, devpi) without pending releases.
  - Tests:
    - `infra/kubernetes/scripts/run-helm-layer.sh infra/kubernetes/helmfiles/25-package-registries.yaml`
    - `infra/kubernetes/scripts/run-helm-layer.sh infra/kubernetes/helmfiles/26-package-npm.yaml`
    - `infra/kubernetes/scripts/run-helm-layer.sh infra/kubernetes/helmfiles/27-package-go.yaml`
    - `infra/kubernetes/scripts/run-helm-layer.sh infra/kubernetes/helmfiles/28-package-py.yaml`
- [ ] Layer 26 npm runner publishes + smokes TypeScript packages and updates `ts-packages-state`.
  - Test: `infra/kubernetes/scripts/run-helm-layer.sh infra/kubernetes/helmfiles/26-package-npm.yaml`
  - Remote coverage: smoke job fetches `lodash` through Verdaccio twice to confirm proxying and cached tarball access.
- [ ] Layer 27 Go runner publishes + smokes Go modules and updates `go-packages-state`.
  - Test: `infra/kubernetes/scripts/run-helm-layer.sh infra/kubernetes/helmfiles/27-package-go.yaml`
  - Remote coverage: smoke job fetches `github.com/google/uuid@v1.6.0` and validates cached `.info/.mod/.zip` endpoints in Athens.
- [ ] Layer 28 Python runner publishes + smokes Python packages and updates `python-packages-state`.
  - Test: `infra/kubernetes/scripts/run-helm-layer.sh infra/kubernetes/helmfiles/28-package-py.yaml`
  - Remote coverage: smoke job downloads `requests==2.32.3` via devpi and probes the cached `+files` artifact.
- [ ] `infra/kubernetes/scripts/tilt-packages-health-summary.sh` reports healthy registry resources after Helm tests finish.
  - Test: `infra/kubernetes/scripts/tilt-packages-health-summary.sh`

## Implementation Progress
- 2025-02-12 – Refactored the Go module uploader into `services/go_module_registry/internal/registry`, relocated unit coverage under `__tests__/unit`, and confirmed `GOWORK=off go test ./__tests__/unit/...` passes.
- 2025-02-15 – Brought the npm stack in line with the backlog: Tilt now exports Verdaccio registry env vars for TypeScript packages, all Node Dockerfiles default to `http://npm.packages.dev.ameide.io`, the Verdaccio HTTPRoute is deployed as `verdaccio-route`, Verdaccio runs with tuned uplink retries/TTL plus Prometheus scraping, and the TypeScript SDK Jest suite is green.
- 2025-02-20 – Replaced the Python pypiserver deployment with devpi: new `devpi-registry` Helm chart, bespoke bootstrap image, warm-cache preseed tooling, Prometheus exporter + ServiceMonitor, and updated publish/smoke scripts pointing at `http://pypi.packages.dev.ameide.io/root/ameide/+simple/`.
- 2025-03-25 – Extended the TypeScript SDK smoke test to extract the published tarball and perform Node 22 ESM imports (`@ameideio/ameide-sdk-ts` and `@ameideio/ameide-sdk-ts/platform/organizations`), preventing regressions like the missing `exports.default` entry surfacing in production.
- 2025-11-05 – Removed the bespoke Envoy proxy for devpi and standardised on the shared `envoy` alias; all scripts, smoke jobs, and docs now reference `http://pypi.packages.dev.ameide.io/root/ameide/+simple/` without `:8080`.
- 2025-11-05 – Hardened Athens routing: uploader now mirrors artifacts into both the canonical and legacy paths, `ATHENS_NETWORK_MODE=fallback` and the patched upstream filter keep `/@v/list` local, and the Helm smoke test verifies every download endpoint.
- 2025-11-06 – Augmented language runners to exercise upstream proxies: npm smoke hits `lodash`, Go smoke hits `github.com/google/uuid@v1.6.0`, and devpi smoke downloads `requests==2.32.3`, validating that each registry caches third-party packages before declaring the layer healthy.

## Lessons Learned (Nov 2025)
- Smoke tests must exercise the real download protocol (`/@v/{list,info,mod,zip}`); lightweight health checks masked Athens regressions until we expanded coverage.
- Helmfile should encode every “kubectl patch” we depend on—ConfigMaps drifted between runs until we added the post-sync hook.
- When a proxy participates in both publish and fetch flows (Athens), alignment on filenames and directory layout is essential; mirroring artifacts avoided object-store incompatibilities.

## Verification Matrix
| Language   | Package                        | Helm Test Segment                 | State Artifact                         | Registry Host                         | Status |
|------------|--------------------------------|-----------------------------------|----------------------------------------|----------------------------------------|--------|
| Go         | `ameide-sdk-go`                | Layer 27 `package-go-runner` | `/workspace/tmp/tilt-packages/go.json`           | `http://go.packages.dev.ameide.io`  | ✅ |
| Python     | `ameide-sdk`                   | Layer 28 `package-python-runner` | `/workspace/tmp/tilt-packages/python.json`            | `http://pypi.packages.dev.ameide.io/root/ameide/+simple/` | ✅ |
| TypeScript | `@ameideio/ameide-sdk-ts`          | Layer 26 `package-ts-runner` | `/workspace/tmp/tilt-packages/ts.json`              | `http://npm.packages.dev.ameide.io` | ✅ |

## References
- `backlog/344-packages-go.md`
- `backlog/344-packages-py.md`
- `backlog/344-packages-npm.md`
- (TBD) `backlog/344-packages-nuget.md`
- `infra/kubernetes/helmfiles/25-package-registries.yaml`
- Tiltfile resources depend on the `infra:25` → `infra:28` chain and surface `packages:health` as the registry status gate
