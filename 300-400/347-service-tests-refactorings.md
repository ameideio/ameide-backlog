> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# Service Test Refactorings Guide

**Created:** Jan 2026  
**Owner:** Platform DX / Quality Engineering  
**Related Backlog:** 335, 336, 337, 338, 339, 340, 341, 342, 343, 344, 345, 346

> **Update (2026-01): 430v2 contract**
>
> This guide is historical (it reflects the “integration pack / integration-runner” era). Treat `backlog/430-unified-test-infrastructure-v2-target.md` as the normative contract (strict phases, native tooling, JUnit evidence; no `INTEGRATION_MODE`; no `run_integration_tests.sh` packs as the canonical path).

---

## Why this guide exists

Between late 2025 and early 2026 the platform executed a series of infrastructure, SDK, and Helmfile refactors (backlog items 335 – 346). Each change subtly altered how service-level tests compile, obtain dependencies, and execute inside Tilt, Helm, and CI. This guide captures those refactorings from the vantage point of service tests so future migrations stay aligned with the architecture decisions already in production.

---

## Summary of relevant refactors

### Lessons learned (2026-11 iteration)

- Ensure service and integration images resolve SDK dependencies exclusively through the internal language registries (`npm.packages.dev.ameide.io`, `go.packages.dev.ameide.io`, `pypi.packages.dev.ameide.io`)—drop editable installs or direct workspace mounts.
- Run container installs without `--ignore-scripts`; registries publish prebuilt artifacts, but SDKs often re-run `build/sync-proto` during install. Skipping scripts leaves `dist/` empty and produces “module not found” failures only visible at runtime.
- Let pnpm manage workspace wiring. Manual `ln -s …/node_modules/<pkg>` hacks look convenient for dev images but create dangling symlinks once the root package stops depending on the SDK. If resolution breaks, fix the dependency graph or publish a new snapshot—don’t paper over it with filesystem tricks.
- Keep runtime configuration in chart values so Tilt/Helm overlays provide a single source of truth for env vars across environments (local, staging, production).
- Standardise integration jobs on the shared Helm chart at `infra/kubernetes/charts/test-job-runner/` to guarantee consistent hooks, service accounts, and metadata.
- Cap integration-runner timeouts at 100 seconds; rely on real timeout signals (pytest-timeout) instead of ad-hoc sleeps to keep feedback tight without masking regressions.
- remember tests should help improve the solution, passing tests with workarounds undermine the quality of the solution. refactor for the best solution
- continue iterating until full green

### Container + SDK handling deep dive

1. Confirm Verdaccio credentials (`VERDACCIO_TOKEN`, `NPM_CONFIG_REGISTRY`) exist in both build and runtime stages; unauthenticated installs fall back to the public registry.
2. Keep the root `.npmrc` aligned with service-level `.npmrc`; contradictory registry entries produce hard-to-debug 404s.
3. Cache the pnpm store with `--mount=type=cache`, never as read-only—pnpm writes metadata on every install.
4. Run `pnpm store prune` in CI to avoid cache bloat from obsolete SDK tarballs.
5. Bump `services/www_ameide_platform/package.json` whenever a new SDK snapshot is published; don’t rely on `npm:` aliases in production.
6. Skip `pnpm fetch`; local registries emit fresh tarballs per publish and cached fetches quickly go stale.
7. Use `pnpm install --no-frozen-lockfile` for deterministic CI builds; drop the flag only when iterating locally.
8. Persist SDK publish metadata in `tmp/tilt-packages/ts.json` so downstream smoke scripts validate the exact version.
9. Smoke-test resolution with `node -e "console.log(require.resolve('@ameideio/ameide-sdk-ts/package.json'))"` immediately after install.
10. Ensure the SDK’s `browser` field shields client bundles from `node:` dependencies by mapping server-only files to `false`.
11. Run `pnpm -C packages/ameide_sdk_ts run build:clean` before every publish so the tarball reflects the latest sources.
12. Consume Verdaccio packages in services; never copy `packages/ameide_sdk_ts/dist` directly into app code.
13. Leverage `PNPM_IGNORE_SCRIPTS=true pnpm pack` during publish to avoid double-running build scripts.
14. Guard publish scripts with `set -euo pipefail` so failed builds stop the pipeline immediately.
15. Document each SDK publish (new exports, breaking removals) in a changelog for service teams.
16. Audit `exports` to include every subpath used by services and tests; missing entries quietly break bundlers.
17. Add a CI job that imports `@ameideio/ameide-sdk-ts/browser` to keep the subpath alive.
18. Delete `src/proto` and `dist/proto` before running `sync-proto` to prevent stale files from sneaking into the tarball.
19. Log SDK version ↔ commit hash pairings in `tmp/sdk-publish-log.json` for quick regression tracing.
20. Bundle `.proto-manifest.json` inside the tarball so consumers can verify generated artifacts.
21. Prefix publish logs consistently (`[ameide-sdk-ts]`) for easy grepping in CI and Tilt.
22. Retain `"type": "module"` in the SDK package so bundlers treat emitted code as ESM.
23. Keep `"sideEffects": false` only while exports remain pure; revisit after adding polyfills or stateful helpers.
24. Run `pnpm audit --prod` before publishing to surface vulnerable transitive dependencies.
25. Track pnpm store size with `du -sh "$(pnpm store path)"` to stay within CI disk quotas.
26. Verify browser bundles via `pnpm dlx rollup @ameideio/ameide-sdk-ts/browser` to catch `node:` imports.
27. Measure bundle size with `size-limit` to flag accidental weight increases.
28. Publish source maps and `.d.ts.map` files to improve editor navigation for service developers.
29. Attach SBOM artifacts to SDK releases to satisfy supply-chain audits.
30. Rotate Verdaccio admin tokens and document the rotation cadence in infra runbooks.
31. Set `AMEIDE_TS_SDK_VERSION` in CI and Tilt before publishing so snapshots remain deterministic.
32. Keep `scripts/tilt-packages-ts-smoke.sh` synchronized with publish outputs to avoid false positives.
33. Provide a `pnpm run smoke` command in the SDK that exercises key exports before publishing.
34. Trigger CDN or proxy cache invalidation in a `postpublish` hook when mirrored artifacts change.
35. Pin pnpm via the repository `packageManager` field to stop CLI drift across contributors.
36. Run `corepack enable` in all container stages so pinned pnpm versions stay consistent.
37. Execute Docker builds from the repo root to preserve pnpm workspace linking.
38. Remove `.next` before rebuilding in CI to eliminate stale incremental artifacts.
39. Avoid caching `node_modules/.cache`; Next.js caches contain absolute paths that break across hosts.
40. Set `NODE_ENV` appropriately (`production` for builds, `development` for dev containers) to match runtime behavior.
41. Mirror `.env.example` updates into Docker ENV defaults to catch missing configuration early.
42. Standardize on `WORKDIR /app/services/www_ameide_platform` before running build commands.
43. Copy only `public`, `.next/standalone`, and `.next/static` into production images to shrink surface area.
44. Grant execute permissions (`chmod +x`) to integration scripts during the image build.
45. Run containers as non-root (`nextjs:nodejs`) to satisfy security scanners.
46. Expose only the ports each image actually uses (`3000` prod, `3001` dev).
47. Set `NEXT_TELEMETRY_DISABLED=1` in build and runtime phases for privacy compliance.
48. Pass secret arguments (`AUTH_*`) only to the build stage and avoid persisting them in final images.
49. Log every Dockerfile adjustment in this backlog entry so future refactors understand historical context.
50. Schedule a monthly “container sanity” rebuild that runs smoke tests end-to-end against Verdaccio snapshots.

### Domain & proto reshaping (#335)

- **What changed:** Initiatives → transformations, chat → threads, metamodel/repository → graph. gRPC surfaces were renamed and consolidated, and repository context became mandatory.
- **Test impact:** Service and integration tests must instantiate clients from the new SDK namespaces (`graphService`, `transformationService`, `threads`) and pass `repositoryId` on every repository-scoped RPC. Legacy cross-repository assertions are no longer valid; repository fixtures need to be seeded per test.
- **Guidance:** When touching service tests, confirm fixtures and golden responses reference the new enums (`TRANSFORMATION_*`, `ELEMENT_KIND_*`) and that proto imports come from the generated graph/transformation packages.
- **Outstanding gap:** Capture an explicit service-test migration plan in `backlog/335-renaming.md` (fixtures to rewrite, repository scaffolding, smoke coverage expectations) so the breaking proto changes remain guarded.

### SDK ownership & dependency hardening (#336)

- **What changed:** Only the language SDK workspaces generate code from `packages/ameide_core_proto`. Services consume published SDKs via the Verdaccio/npm, Go, and PyPI mirrors, and Tilt/CD pipelines enforce that boundary.
- **Test impact:** Unit and service tests should import client libraries from the published SDKs, not from raw proto outputs. Test harnesses invoking codegen must run through the same `sync-proto` helpers as production to avoid drift.
- **Guidance:** Audit service test utilities for direct `packages/ameide_core_proto` references and replace them with SDK imports. Wire test bootstrap scripts to fail fast when the SDK snapshot is missing or stale; do not copy proto files into test fixtures manually.
- **Outstanding gap:** Land the lint/CI guardrail that blocks proto workspace imports in service tests and add a smoke case that exercises the published SDK artifacts to prove the hardened path works end-to-end.

### Registry routing & build plumbing (#337, #344)

- **What changed:** k3d now owns the Docker registry lifecycle (`k3d-dev-reg:5000`), and language registries live behind `*.packages.dev.ameide.io`. Publish → smoke scripts were harmonised and use shared logging.
- **Test impact:** Service test images and helper jobs must push/pull via the mirror endpoints (e.g., `k3d-dev-reg:5000/...` inside clusters). Package-based tests should resolve dependencies through the language-specific mirrors to exercise the hardened path.
- **Guidance:** Update Helm test charts and Tilt resources to rely on the new hostnames, and ensure service tests that publish temporary SDK snapshots respect the harmonised logging + JSON state contracts.

### Devcontainer bootstrap & Helm layering (#338, #339)

- **What changed:** Post-create now reconciles the k3d cluster, runs layered Helmfiles, seeds Vault secrets, and executes Helm smoke jobs before Tilt launches. Helmfile was split into deterministic layers with shared defaults and per-layer test hooks.
- **Test impact:** Service test suites can assume Helm layers 10 – 45 plus `platform-smoke` have converged before they start. Layer-specific re-runs (`infra:<layer>` Tilt resources) give deterministic entry points for integration and smoke tests.
- **Guidance:** Structure new service test refactors around the layered model: define Helm smoke jobs alongside the service’s Helm release, and rely on `run-helm-layer.sh` for orchestration. Avoid bespoke bootstrap scripts; hook into existing layers instead.

### Secrets lifecycle & Vault migration (#340, #341, #342)

- **What changed:** ExternalSecrets moved from central manifests to per-service ownership, with HashiCorp Vault now acting as the sole store for local and (eventually) hosted environments.
- **Test impact:** Service tests requiring credentials should source them from the per-service ExternalSecrets templates. Local test bootstrap must invoke `scripts/vault/ensure-local-secrets.py` to ensure Vault roles and sample secrets exist before assertions run.
- **Guidance:** When refactoring service tests, validate that fixtures reference the per-service secret templates and that Vault sync runs as part of the test harness. Do not reintroduce inline Kubernetes Secrets or Azure Key Vault assumptions.

### Tilt + Helm North Star v3 (#343)

- **What changed:** Tilt now injects `image.repository`/`image.tag` directly into Helm releases; digest files and `image.graph` were removed. Shared helpers enforce digest-first rendering while supporting Tilt’s mutable tags.
- **Test impact:** Helm-based service tests (smoke jobs, migration jobs) must expose `image.repository`/`image.tag` fields and opt into the same helpers so Tilt rebuilds automatically refresh the test job image.
- **Guidance:** When backfilling service test charts or jobs, use the shared image helper snippet and set `image_keys` in Tilt’s `helm_resource`. Ensure multi-container jobs (e.g., migrations + smoke) cover every image block.
- **Outstanding gap:** Audit Helm smoke/test charts (e.g., `platform-smoke`, service-specific smoke jobs) to confirm they adopted the new helper; track any stragglers in the migration checklist so Tilt rebuilds do not deploy stale images.

### Package + CI/CD pipelines (#345, #346)

- **What changed:** CI/CD and local publish flows standardised on `AMEIDE_{LANG}_SDK_VERSION`, Cosign/SBOM gates, and uniform logging adapters.
- **Test impact:** Service tests that publish or consume SDK snapshots must propagate the language-specific env vars and emit harmonised logs so Tilt and CI parse results consistently.
- **Guidance:** Align service test scripts with the logging adapters, and when snapshots are published during tests, record the version in the shared `tmp/tilt-packages/<lang>.json` schema for downstream smoke steps.

---

## Service test refactoring checklist

1. **Contracts first:** Update SDK client usage, proto enums, and request parameters to match the graph/transformation/threaded services. Validate repository scoping in fixture setup.
2. **Dependency hygiene:** Import SDKs via the language registry mirrors; ensure test bootstrap refreshes snapshots with the hardened `sync-proto` flows.
3. **Environment readiness:** Hook into Helm layers and Tilt resources rather than bespoke scripts. Re-run relevant layers (`run-helm-layer.sh`) as part of test setup when infrastructure changes.
4. **Secrets alignment:** Reference per-service ExternalSecrets templates and confirm Vault sync runs as a prerequisite. Avoid embedding credentials in test charts.
5. **Image wiring:** Use the shared image helper to support Tilt’s `image.repository`/`image.tag` injection across deployments, CronJobs, and smoke jobs.
6. **Logging & observability:** Source the shared logging adapters in publish/smoke/test scripts to keep Tilt panes and CI logs readable. Emit the snapshot version for any package interactions.
7. **Validation gates:** Extend or reuse Helm smoke tests to cover new service surfaces and secrets. Add CI guardrails (lint/test) when new patterns emerge to prevent regressions.

---

## Next steps

- [ ] Define and publish the test migration plan inside `backlog/335-renaming.md` (fixtures, repository scaffolding, coverage expectations).
- [ ] Land the proto-import lint + SDK-from-registry smoke test referenced in #336.
- [ ] Audit Helm smoke/test charts for the image helper upgrade and record remaining work in the v3 checklist.
- [ ] Expand automated smoke coverage for services newly scoped to repository-aware APIs.
- [ ] Add Vault-backed secret verification to the standard service test harness.
- [ ] Mirror this guide into `docs/testing/service-tests.md` once adoption is complete.
