> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.


# Go Packages Architecture _(Archived)_

> **Status – superseded (2026-02-17):** Internal Go module registries were removed as part of the Buf Managed Mode migration. The details below are preserved for historical context only; see backlog/356-protovalidate-managed-mode.md for the current workflow.

## Principles
- No baked-in defaults: scripts must fail if required environment variables or credentials are missing; Helm/Vault provide the values per environment.
- Single source of truth: Tilt/CI read the same env vars as production (no per-script overrides).
- Publish → smoke → consume: every artifact is tested via dedicated smoke scripts before any dependent builds run.
- No fallbacks or interim migrations: run the target architecture only; if requirements change, update docs and scripts rather than adding transitional code.

## Goals
- Maintain a single in-cluster proxy (`go.packages.dev.ameide.io`) that caches both Ameide and third-party Go modules.
- Run the entire publish → smoke loop as Helmfile-managed jobs so local dev, CI, and production reuse one deployment path.
- Keep CI reproducible: builds pin module versions while relying on cached artifacts from the proxy.
- Align production releases with immutable module behaviour (git tags still possible) while keeping dev workflows intact.

## Components
- **Athens Proxy (`athens-proxy`)**
  - Vendored official Helm chart (`infra/kubernetes/charts/third_party/gomods/athens-proxy/0.15.3`).
  - Runs in namespace `ameide`, listens on port 8080, exposes `/healthz`, `/readyz`, `/version`, and the Go download protocol.
  - Stores module archives under `/var/lib/athens` backed by a PersistentVolumeClaim (`athens-proxy-storage`); sizes per environment: 5 Gi (local), 20 Gi (staging), 50 Gi (production).
  - Fetches upstream modules on cache miss and persists them for subsequent consumers.
- **Go Module Uploader (`go-module-uploader`)**
  - Existing Go binary (`services/go_module_registry`) packaged as a Deployment (`infra/kubernetes/charts/platform/go-module-uploader`).
  - Mounts the same PVC as Athens via `existingClaim: athens-proxy-storage` and writes directly into the Athens directory structure.
  - Exposes `/upload` (multipart form: module, version, info, mod, zip) and `/healthz`; no read endpoints.
- **Gateway Routing**
  - Envoy Gateway routes `go.packages.dev.ameide.io/upload` to the uploader Service and all other paths to Athens (`gitops/ameide-gitops/sources/charts/apps/gateway/values.yaml`, environment overrides under `gitops/ameide-gitops/sources/values/*/apps/platform/gateway.yaml`).
  - Ensures publish scripts continue working while consumers transparently hit Athens.

## Workflows
### Helmfile / Environment Sync
1. Apply layers 25 and 27 (`infra/kubernetes/scripts/run-helm-layer.sh infra/kubernetes/helmfiles/25-package-registries.yaml` and `27-package-go.yaml`) to deploy Athens plus the uploader; layer 27 now rebuilds/imports only the Go-specific images (`go-module-registry`, `packages-publisher-go`, `kubectl`, `curl`) through the refactored `scripts/infra/build-local-packages-images.sh` prior to running tests.
2. The `package-go-runner` Helm test uses the slim `packages-publisher-go` image to invoke `scripts/tilt-packages-go-publish.sh`, followed by the smoke scripts. The smoke phase now also fetches `github.com/google/uuid@v1.6.0` twice to prove Athens proxies upstream modules and exposes their cached `.info/.mod/.zip` endpoints. Targeted image imports mean offline/local workflows inherit warm caches automatically—no need for a separate `build-local-packages-images.sh` invocation.
3. Required configuration (`GO_MODULE_PATH`, `GO_REGISTRY_URL`, `GOPRIVATE`, `GONOSUMDB`) is injected via the `packages-go-env` ExternalSecret, which pulls values from Vault (`scripts/vault/ensure-local-secrets.py` seeds local values).
4. After publishing completes, the runner job writes the resolved pseudo-version JSON (`ameide-sdk-go.json`) into the ConfigMap `go-packages-state`, keeping downstream consumers environment-agnostic. The helper `infra/kubernetes/sync-go-packages-state.sh` copies this JSON file to `tmp/tilt-packages/` on demand.

### CI / Release
1. CI workflows apply layers 25 + 27; the layer 27 Helm test automatically publishes and smokes the Go packages as part of the pipeline run.
2. Release pipelines publish immutable tags, rerun the layer 27 job, and record the refreshed pseudo-version JSON.
3. Because Athens holds state on the PVC, modules stay available across restarts; the runner job continues to probe health and smoke endpoints on every execution.

### Consumers
- Containers and CI set:
  ```bash
  export GOWORK=off
  export GOPROXY="http://go.packages.dev.ameide.io,direct"
  export GOPRIVATE="go.packages.dev.ameide.io/*"
  export GONOSUMDB="go.packages.dev.ameide.io/*"
  ```
- The `services/go_module_registry` Dockerfile now exposes `GOPROXY`/`GOSUMDB` build args; local workflows override them to hit the internal proxy while ad‑hoc builds retain the public defaults.
- Before building, they pin module versions via `go mod edit -require go.packages.dev.ameide.io/<module>@${VERSION}` using the JSON state written during publish.
- Telemetry modules follow the same contract.
- Local developers can pull the latest pseudo-versions into `tmp/tilt-packages/` by running `infra/kubernetes/sync-go-packages-state.sh` after Helm tests complete; Tilt consumes those JSON files without executing publish scripts.

## Operational Notes
- **Charts & Helmfile**: Athens, the uploader, and the Go packages runner all ship in layers 25/27; the Helm test bundled with layer 27 handles publish/smoke duties.
- **PVC Ownership**: Athens creates `athens-proxy-storage`; uploader references it using `existingClaim`. Storage classes/size overrides live in `infra/kubernetes/environments/*/platform/athens-proxy.yaml`.
- **Routing**: HTTPRoute changes reside in the gateway chart and environment overrides, ensuring `/upload` hits the uploader, everything else the proxy.
- **Health & Smoke**: Layer 27 waits for `deployment/athens-proxy` and `deployment/go-module-uploader`, publishes both modules via the uploader, then verifies served versions through the proxy endpoints before updating ConfigMap state.
- **Upstream Credentials**: If Athens must fetch private repos, wire secrets via chart values (`netrc`, SSH config). The uploader does not need upstream access because publish scripts send the artifacts explicitly.
- **Future Simplification**: Removing the uploader would require switching Tilt/CI to tag-based publishing. Keeping both preserves the fast dev loop while Athens guarantees cached delivery.
- **Storage Evolution**: For staging/production, evaluate swapping the disk PVC for an object-store backend (Athens supports S3, GCS, MinIO) to simplify backups and reduce PVC hot-spot risk.
- **Retention & Immutability**: Athens treats versions as immutable; document GC expectations and add a periodic audit to cull unused pseudo-versions so the volume does not grow without bound.
- **Private Upstream Smoke Test**: Add a CI check that temporarily publishes a private module and verifies Athens can fetch it using the configured credentials (netrc/SSH), ensuring private repos stay reachable.

## Cross-Cutting Topics
- **Helmfile Test Integration** – Layer 27 deploys Athens + uploader and immediately runs the `package-go-runner` Helm test to publish and smoke both Go modules in every environment.
- **Secret Sourcing & Vault** – Vault seeds `secret/data/packages/go` and external-secrets mirrors it into Kubernetes for the Helm test and workloads.
- **State Artifact Persistence** – Publish/smoke jobs write pseudo-version JSON into the `go-packages-state` ConfigMap; consumers pull the files into `/workspace/tmp/tilt-packages/` via `infra/kubernetes/sync-go-packages-state.sh`.
- **CI Alignment** – GitHub Actions invoke Helmfile with `--include-tests` to reuse the same jobs; no bespoke Go publish scripts remain in CI.
- **Tilt / Local Dev Flow** – Tilt waits on `packages:health` and consumes the ConfigMap so local developers never run publish scripts manually.
- **Deployment Health & Rollout** – Athens and the uploader share the `athens-proxy-storage` PVC; rollouts gate on deployment readiness probes.
- **Routing & Gateway** – Envoy routes `/upload` to the uploader and all other paths to Athens so publish and fetch traffic share the hostname.
- **Persistence & Storage Strategy** – Single PVC today with plans to evaluate S3/GCS backends for staging/prod to ease retention and backups.
- **Observability & Alerts** – Helm tests cover health endpoints; upcoming work adds private-module smoke coverage and retention audits.
- **Documentation & Onboarding References** – This doc underpins Go onboarding materials and references Helmfile layers 25/27 as the source of truth.

- Layer 27 Helm test mounts the `packages-go-env` secret and drives the publish + smoke scripts through the slim `packages-publisher-go` image.
- The Helm job updates the `go-packages-state` ConfigMap, which stores the latest pseudo-version JSON document for the SDK.
- Tilt, CI, and Docker builds reference the JSON artifacts (via `infra/kubernetes/sync-go-packages-state.sh` or direct ConfigMap mounts) to stay in sync without re-running publish scripts.
- Vault seeding (`scripts/vault/ensure-local-secrets.py`) ensures `secret/data/packages/go/*` keys exist so environments can render the ExternalSecret automatically.

### Team Integration Guidance
- **Platform Engineering**
  - Ensure `helmfile --selector stage=package-registries test --include-tests` is executed in CI prior to Go consumer builds (see checklist below).
  - Monitor Helm test logs in `logs/helm-tests/*` for publish/smoke regressions; triage failures before unblocking downstream jobs.
- **Service Teams**
  - Consume pseudo-version state via `tmp/tilt-packages/go*.json` (synced through the new script) instead of invoking publish scripts.
  - Replace any legacy `replace` directives or workspace references with `go.packages.dev.ameide.io/<module>@<version>`.
- **Release / Ops Enablement**
  - Update runbooks with Helm test failure remediation (e.g., redeploy Athens, verify ConfigMap writes, re-run sync script).
  - Own alerting backlog items for Athens/uploader availability and ConfigMap drift.

## Implementation Checklist
- [x] Layer 27 Helm test publishes and smokes both Go modules using the `packages-publisher-go` image.
- [x] `packages-go-env` ExternalSecret surfaces registry/module env vars from Vault.
- [x] `go-packages-state` ConfigMap stores `ameide-sdk-go.json` pseudo-version documents.
- [x] Added `infra/kubernetes/sync-go-packages-state.sh` for local sync of ConfigMap state to `tmp/tilt-packages/`.
- [ ] Update CI workflows to run `helmfile --selector stage=package-registries test --include-tests` before Go consumer builds (owners: Platform Eng).
- [ ] Document Helm test failure triage and recovery steps in the operational runbook (owners: Ops Enablement).
- [ ] Automate alerting for ConfigMap drift / missing keys in `go-packages-state` (owners: SRE).
- Developers can refresh the local state files by running `infra/kubernetes/sync-go-packages-state.sh`, which copies the ConfigMap contents into `tmp/tilt-packages/`.

## Implementation Progress
- 2025-02-12 – `services/go_module_registry` now wraps uploader handlers in `internal/registry`, exposing `/upload` only and writing artifacts atomically into the shared PVC.
- 2025-02-12 – Go uploader unit coverage relocated to `services/go_module_registry/__tests__/unit`, asserting duplicate protection and sorted version lists.
- 2025-02-20 – Helm test + ConfigMap pipeline wired (`packages-publisher-go` image, `packages-go-env` ExternalSecret, `go-packages-state` ConfigMap, local sync script).
- Next up: document and automate PVC GC expectations once object storage evaluation lands.

## Recent Implementation Notes (Nov 2025)
- `services/go_module_registry` duplicates every publish into both the canonical `<module>/<version>` layout and the legacy `<module>/@v` tree so Athens’ filesystem backend and download API stay in sync.
- Layer 27 now runs a post-sync `kubectl patch configmap/athens-proxy-upstream` step to pin the filter to `D` + explicit `+ go.packages.dev.ameide.io/*`, preventing redirects to the public proxy.
- `ATHENS_NETWORK_MODE` is set to `fallback` in every environment override; the Helm smoke job confirmed this prevents `/@v/list` regressions while upstream is unreachable.
- The smoke test curls `/@v/{list,info,mod,zip}` so failures surface real storage/routing drift rather than silent publish errors.

## Lessons Learned
- Athens will happily accept uploads that only populate `<module>/<version>`, but its download path still reads `@v`; mirroring artifacts avoided hard-to-debug 404s.
- Leaving Athens in `strict` mode hides local cache issues behind upstream failures—`fallback` combined with the smoke test keeps confidence high.
- Filters must be enforced centrally (via Helm post-sync) instead of relying on manual ConfigMap edits; without that automation the fix drifted on every helmfile run.

## Testing Strategy
- **Unit/CI**: `go test ./...` within `packages/ameide_sdk_go`, and `GOWORK=off go test ./__tests__/unit/...` for `services/go_module_registry`.
- **Helm Smoke**: Helm test job publishes both modules, calls `curl` against the proxy/uploader health endpoints, runs `go list -m` against Athens, and fails the release if any step errors.
- **End-to-end**: Services rebuild automatically once the Helm test refreshes the pseudo-version ConfigMap; downstream GitHub Actions call `go build` with the exported pseudo-version.

## Implementation Success Criteria
- [ ] Helmfile layer 27 completes with the Go publish + smoke steps passing in every environment.
  - Test:
    - `infra/kubernetes/scripts/run-helm-layer.sh infra/kubernetes/helmfiles/27-package-go.yaml`
- [ ] Athens (`deployment/athens-proxy`) and the uploader (`deployment/go-module-uploader`) roll out across local/staging/prod with the `athens-proxy-storage` PVC bound.
  - Test: `kubectl -n ameide rollout status deployment/athens-proxy && kubectl -n ameide get pvc athens-proxy-storage`
- [ ] Publish jobs deposit state JSON in the shared ConfigMap so consumers build with the recorded pseudo-version.
  - Test: `kubectl -n ameide get configmap go-packages-state -o yaml | yq '.data'`
- [ ] CI pipelines consume modules solely via `go.packages.dev.ameide.io` without fallback to workspace sources.
  - Test: `rg "go.packages.dev.ameide.io" .github/workflows -g '*.yml' && rg "replace" packages/*/go.mod`
- [ ] Documentation (`backlog/344-packages-go.md`) reflects the Helmfile-driven flow and stays referenced by onboarding docs.
  - Test: `rg "344-packages-go" docs/onboarding`


## Environment Variables
*All environment variables below must be populated via Helm/vault-provisioned secrets; scripts should not carry defaults.*
- `GOPROXY=http://go.packages.dev.ameide.io,direct`
- `GOWORK=off`
- `GO_REGISTRY_URL=http://go.packages.dev.ameide.io`
- `GO_MODULE_PATH` (per module, e.g. `go.packages.dev.ameide.io/ameide-sdk-go`)
- `GOPRIVATE=go.packages.dev.ameide.io/*`
- `GONOSUMDB=go.packages.dev.ameide.io/*`
