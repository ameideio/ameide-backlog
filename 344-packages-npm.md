> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.


# npm Packages Architecture _(Archived)_

> **Status – superseded (2026-02-17):** The Verdaccio-based npm mirror has been removed. Buf’s BSR now serves the TypeScript SDKs directly; the implementation notes below are retained for historical reference only.

## Principles
- No baked-in defaults: scripts must fail if required environment variables or credentials are missing; Helm/Vault provide the values per environment.
- Single source of truth: Tilt/CI read the same env vars as production (no per-script overrides).
- Publish → smoke → consume: every artifact is tested via dedicated smoke scripts before any dependent builds run.
- No fallbacks or interim migrations: run the target architecture only; if requirements change, update docs and scripts rather than adding transitional code.

## Goals
- Provide an in-cluster npm registry at `npm.packages.dev.ameide.io` for all internal TypeScript packages.
- Cache third-party dependencies via Verdaccio uplinks so repeated installs stay inside the cluster.
- Execute publish/smoke via Helmfile-managed jobs so every environment shares the same automation.
- Keep CI reproducible by pinning versions and using the same registry endpoint as development.

## Components
- **Verdaccio (`npm-registry`)**
  - Vendored chart: `infra/kubernetes/charts/third_party/verdaccio/verdaccio/4.26.1/verdaccio-4.26.1.tgz`.
  - Configured via `infra/kubernetes/values/infrastructure/verdaccio.yaml` with `uplinks.npmjs.cache=true`, scope rules, and relaxed auth for local workflows. Default resources now reserve 256 Mi / 1 Gi and export `NODE_OPTIONS=--max-old-space-size=1024` so large upstream tariff pulls (e.g. `playwright-core`, `@typescript-eslint/*`) no longer crash the registry while warming the cache.
  - Persistence enabled (`persistence.enabled: true` in base values, overrides per environment such as `infra/kubernetes/environments/local/infrastructure/verdaccio.yaml`). Storage path `/verdaccio/storage/data` mounted from the PVC so cached packages survive pod restarts.
  - Service listens on port 4873; HTTPRoute (`verdaccio-route`) exposes `http://npm.packages.dev.ameide.io` through Envoy Gateway.
- **Gateway Routing**
  - Defined in `gitops/ameide-gitops/sources/charts/apps/gateway/values.yaml` and environment overrides so all `npm.packages.*` hosts terminate at Verdaccio.
  - HTTPS termination handled by Envoy; Verdaccio itself remains HTTP-only inside the cluster.

## Workflows
### Helmfile / Environment Sync
1. Helmfile layers 25 and 26 deploy shared routing and Verdaccio (`infra/kubernetes/helmfiles/25-package-registries.yaml`, `26-package-npm.yaml`). Layer 26 now refreshes only the TypeScript-specific registry images (`verdaccio`, `packages-publisher-ts`, `kubectl`, `curl`) via the refactored `scripts/infra/build-local-packages-images.sh`, keeping re-runs fast while guaranteeing the in-cluster cache is primed.
2. The layer 26 `package-ts-runner` Helm test uses the `packages-publisher-ts` image to run `scripts/tilt-packages-npm-publish.sh`, followed by the SDK smoke script. The job now installs the SDK tarball into a throwaway npm project so runtime dependencies (`@connectrpc/connect`, etc.) resolve before ESM smoke imports, and exercises a known upstream package (`lodash`) twice to confirm Verdaccio proxies remote content and serves the cached tarball on subsequent hits.
3. Registry configuration (`NPM_REGISTRY_URL`, `NPM_CONFIG_REGISTRY`, `YARN_REGISTRY`, tokens) is injected from Vault-backed secrets seeded via `scripts/vault/ensure-local-secrets.py`; CoreDNS (layer 18) now resolves `npm.packages.dev.ameide.io` inside the cluster so the runner can rely on standard DNS. Registry image imports happen in the same step, so devs no longer need to pre-run `scripts/infra/build-local-packages-images.sh` manually.
4. The job writes `ts.json` into the `ts-packages-state` ConfigMap; TypeScript consumers read the ConfigMap directly (no sync helper required) when populating `/workspace/tmp/tilt-packages/`.

### CI / Release
1. CI applies layers 25 and 26 and relies on the layer 26 Helm test to publish/smoke the TypeScript packages. The image import helper is idempotent, so repeated pipeline invocations only rebuild the images whose sources changed.
2. For production promotions, the workflow runs the Helm test with release versions and publishes to Verdaccio before optionally mirroring to npmjs.
3. Verdaccio caches any external packages requested, so repeated Helm test runs stay inside the cluster.

### Consumers
- Services set `NPM_REGISTRY_URL`/`NPM_CONFIG_REGISTRY` to `http://npm.packages.dev.ameide.io` (Dockerfiles expose an `ARG NPM_REGISTRY` defaulting to the internal endpoint).
- Workspaces pin internal packages via `npm:@ameide/<pkg>@dev` aliases.
- Third-party packages are transparently proxied/cached by Verdaccio’s `uplinks.npmjs` configuration.

## Operational Notes
- **Availability Posture**: Stick with a single replica for now, but keep PVC snapshots (`resourcePolicy: keep` and `backup.ameide.io/enabled=true`) so we can fast-restore on failure. If uptime requirements tighten beyond a few minutes, reassess toward RWX-backed multi-replica or a managed vendor proxy.
- **Persistence**: Local environment uses the `local-path` storage class (5 Gi); higher environments inherit cloud defaults. Adjust `size`/`storageClass` per environment file as needed.
- **Offline Build Policy**: Verdaccio keeps npmjs metadata/tarballs warm for 12 hours (`maxage: 12h`, `fail_timeout: 5m`, `max_fails: 4`). That gives CI/Tilt an offline window while still refreshing daily; targeted image rebuilds mean each layer seeds only the tarballs it needs, reducing warm-up latency without sacrificing cache coverage.
- **Authentication**: Local dev still allows auto-provisioned users, but staging/prod disable self-registration (`max_users: -1`). Registry logins should use pre-generated npm tokens stored in the `verdaccio-htpasswd` secret; rotate quarterly or when an engineer offboards.
- **Token Rotation**: To rotate, generate a new npm token, update the secret (`kubectl create secret generic verdaccio-htpasswd --from-file=htpasswd=... --dry-run=client -o yaml | kubectl apply -f -`), then trigger `helmfile sync` so pods reload credentials.
- **Health Checks**: Helm smoke tests wait for `deployment/npm-registry` and curl `/-/ping`. Additional runtime checks live in `scripts/infra/packages-status.sh`.
- **Upstream Outage**: Because Verdaccio caches tarballs + metadata, short npmjs outages are masked once dependencies are hot. Long outages still require the cache to have the necessary versions.
- **Observability**: Prometheus scrapes `/-/metrics` via the baked-in ServiceMonitor. Track SLOs of p95 publish latency < 5 s and cache-hit ratio ≥ 80 % over a rolling day; alert off Prometheus rules as we hook alerts.


## Cross-Cutting Topics
- **Helmfile Test Integration** – Layer 26 deploys Verdaccio and immediately runs the `package-ts-runner` Helm test to publish and smoke TypeScript artifacts across environments.
- **Secret Sourcing & Vault** – `scripts/vault/ensure-local-secrets.py` seeds Verdaccio tokens and registry defaults for Helm tests and workloads.
- **State Artifact Persistence** – Helm jobs emit `ameide-sdk-ts.json` into `ts-packages-state`; workflows read it directly or sync into `/workspace/tmp/tilt-packages/`.
- **CI Alignment** – GitHub Actions invoke Helmfile with `--include-tests packages:verdaccio`, keeping CI on the same automation.
- **Tilt / Local Dev Flow** – Tilt waits on the Helm job via `k8s_resource` dependencies, then consumes the ConfigMap for package versions.
- **Deployment Health & Rollout** – `deployment/npm-registry` readiness gates releases while the Verdaccio PVC maintains cached tarballs.
- **Routing & Gateway** – Envoy Gateway serves `npm.packages.dev.ameide.io` through the `verdaccio-route` HTTPRoute for consistent ingress.
- **Persistence & Storage Strategy** – Verdaccio uses a single PVC with snapshot hooks; RWX or managed mirrors remain future options.
- **Observability & Alerts** – ServiceMonitor metrics back SLOs for publish latency and cache hit ratio.
- **Documentation & Onboarding References** – This doc aligns with TypeScript onboarding/runbooks describing the Helmfile-first registry flow.


## Helmfile Wiring
- Layer 26 Helm test mounts the Verdaccio secret and shared state volume before running the publish + smoke scripts.
- Publish/smoke jobs produce `ameide-sdk-ts.json` in the ConfigMap, which downstream tasks read or copy into `/workspace/tmp/tilt-packages/`.
- Tilt depends on the Helm test via `k8s_resource` so services only build after Helm finishes; no local publish scripts remain.
- Vault seeder populates npm tokens, scoped registry URL, and CLI defaults so Helm tests, CI, and developers use identical credentials.

## Implementation Progress
- 2025-02-15 – Injected `NPM_REGISTRY_URL`, `NPM_CONFIG_REGISTRY`, `PNPM_REGISTRY`, and `YARN_REGISTRY` into all Tilt TypeScript package resources and live-update installs so every `pnpm install --no-frozen-lockfile/publish` hits `http://npm.packages.dev.ameide.io`.
- 2025-02-15 – Defaulted every TypeScript service Dockerfile (dev + runtime) to the Verdaccio endpoint and wired runtime stages to export `NPM_REGISTRY_URL`, ensuring containers stay on the internal mirror without extra args.
- 2025-02-15 – Renamed the Verdaccio HTTPRoute to `verdaccio-route` and updated smoke tests to match, keeping Gateway manifests consistent with the backlog docs.
- 2025-02-15 – Documented the single-replica + snapshot posture, enabled Prometheus scraping/ServiceMonitor, enforced token-only auth outside local, and tuned Verdaccio uplink retries/TTL for a 12 h offline window.
- Testing: `pnpm --filter @ameideio/ameide-sdk-ts build` and `NODE_OPTIONS="--experimental-vm-modules --max-old-space-size=8192" pnpm --filter @ameideio/ameide-sdk-ts exec jest --runInBand --passWithNoTests` both pass after correcting the TS SDK test imports and proto contract assertions.
- Next up: revisit RWX/vendor options once uptime requirements exceed the fast-restore window and automate alerting on the new SLOs.

## Testing Strategy
- **Unit**: `pnpm test` / lint in SDK and telemetry packages pre-publish.
- **Helm Smoke**: Publish + `pnpm view` + tarball inspection run inside the Helm test pod; failure blocks the Helmfile sync.
- **Integration**: Optional e2e job pulls versions from the ConfigMap and installs solely from Verdaccio to compile dependent services.

## Implementation Success Criteria
- [ ] Helmfile package tests finish successfully, proving Verdaccio served the freshly published SDK and telemetry packages.
  - Test: `helmfile sync -e local --include-tests --selector name=packages-npm`
- [ ] Verdaccio runs with persistence enabled and caches external packages.
  - Test: `kubectl -n ameide rollout status deployment/npm-registry && kubectl -n ameide get pvc -l app.kubernetes.io/name=verdaccio`
- [ ] Publish scripts push snapshot builds without hitting public npm while smoke scripts confirm availability.
  - Test: `kubectl -n ameide logs job/packages-npm --tail=-1 | rg "npm view @ameide"`
- [ ] All services consume packages via `npm.packages.dev.ameide.io` (no `file:` or workspace links remaining).
  - Test: `rg "npm.packages.dev.ameide.io" -g 'package.json' services packages`
- [ ] Operational docs stay updated with registry host, auth instructions, and recovery steps.
  - Test: `rg "verdaccio" docs/onboarding`


## Environment Variables
*All environment variables below must be populated via Helm/vault-provisioned secrets; scripts should not carry defaults.*
- `NPM_REGISTRY_URL=http://npm.packages.dev.ameide.io`
- `NPM_CONFIG_REGISTRY=http://npm.packages.dev.ameide.io` (also honoured by pnpm)
- `YARN_REGISTRY=http://npm.packages.dev.ameide.io`
- `NPM_AUTH_TOKEN` (per environment when auth enabled)
