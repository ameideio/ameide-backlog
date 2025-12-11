# SDK Proto Dependency Hardening _(needs refresh)_

> **Update (2026-02-17):** Verdaccio/devpi/Athens references describe a retired architecture. Buf Managed Mode now supplies all SDKs; consult backlog/356 for up-to-date distribution guidance.

**Created:** Jan 8, 2026  
**Owners:** Platform DX / Core SDK

## Problem Statement

Tilt builds, service Dockerfiles, and the TypeScript SDK had blurred boundaries between protobuf sources and downstream consumers. Numerous services copied `packages/ameide_core_proto` directly or regenerated code on-the-fly, undermining the principle that only the SDKs own generated artifacts. Recent fixes converged on a single rule: protobufs live in `packages/ameide_core_proto`, SDKs sync/generated code from there, and every other service depends on published SDK outputs only. This backlog item tracks the follow-through so we do not regress.

## Desired End State

- `packages/ameide_core_proto` is consumed only by SDK build scripts (Go/Python/TS) and CI linting; application services never copy it.
- All SDKs fail fast if protobuf sources are missing or stale, without hidden fallbacks.
- Service Dockerfiles consume SDK packages exactly as published (npm/uv/go module) and no longer invoke proto generation or `pnpm --filter "@ameide/core-proto"` directly.
- Tilt ensures SDK resources build first; downstream services declare explicit dependencies.
- Documentation and CI guardrails flag any attempt to import the proto workspace outside SDK boundaries.

## Current Status (Buf Managed Mode – Feb 2026)

- Buf pushes (`.github/workflows/ci-proto.yml`) publish `buf.build/ameideio/ameide`; no generated language artifacts remain committed under `packages/ameide_sdk_*`.
- TypeScript applications import `@buf/ameideio_ameide.bufbuild_es` directly (for example `services/platform/package.json`, `services/www_ameide_platform/src/proto/index.ts`) and construct runtime clients with `@connectrpc/connect` instead of consuming Connect RPC codegen artifacts.
- Go services (`services/agents/go.mod`, `services/inference_gateway/go.mod`) depend on `buf.build/gen/go/...` modules; legacy `go.packages.dev.ameide.io` `replace` directives were removed.
- Python services consume the Buf-generated wheels (`ameideio-ameide-protocolbuffers-python`, `ameideio-ameide-grpc-python`) declared in `packages/ameide_sdk_python`.
- Release helpers under `release/` now verify Buf SDK availability via `buf registry sdk version` and `buf registry sdk download` instead of interacting with Verdaccio/devpi/Athens mirrors.
- Tilt resources wrap Buf downloads through lightweight `src/proto/index.ts` re-export shims so services share a consistent import surface without local regeneration.
- Remaining TODO: tighten CI smoke coverage so every service build asserts the expected Buf SDK versions before container layers are cached.

## Historical Notes (pre-Buf registries)

### Completed Work (TS SDK)

- Removed proto references from Tilt service definitions, keeping only SDK rebuild triggers.
- Simplified TypeScript service Docker builds to copy `packages/ameide_sdk_ts` only; stripped `buf generate` steps from service images.
- Updated `sync-proto.mjs` to require `packages/ameide_core_proto`, copy raw `.proto` files into `dist/proto/raw`, and prune non-proto artifacts so runtime consumers no longer rely on the proto workspace.
- Release tooling publishes the TypeScript SDK to Verdaccio (`release/publish.sh`, `publishConfig`), establishing the internal registry as the single source of truth.
- Tilt helper now injects `extra_hosts` for both `host.docker.internal` and `docker.io`, preferring Docker's `host-gateway` alias and falling back to `172.17.0.1` (or `VERDACCIO_HOST_IP`) so every Docker/BuildKit invocation can reach Verdaccio without touching CoreDNS or per-service Dockerfiles.
- Helmfile now deploys a `registry-mirror` DaemonSet that copies `infra/kubernetes/environments/local/registries.yaml` onto every k3d node and restarts containerd/k3s automatically, replacing the brittle one-off mirror script.
- `packages:npm-publish` runs end-to-end from Tilt: regenerates proto + SDK tarballs, publishes snapshot builds to Verdaccio, and persists metadata for downstream smoke checks.

### Recent Progress (Registry Infrastructure - Nov 1, 2025)

- **Registry Alignment:** Verdaccio now serves under `npm.packages.dev.ameide.io`, reserving `registry.*` for container images while all language package tooling lives behind the `packages.*` namespace.
- **CoreDNS Integration:** Namespace Helm chart rewrites `npm.packages.dev.ameide.io`, `go.packages.dev.ameide.io`, and `pypi.packages.dev.ameide.io` for in-cluster consumers; the legacy registry rewrite was removed once the Envoy listener stabilized.
- **Tiltfile Updates:** Health checks, SDK builds, and `extra_hosts` helpers target the `packages.*` hosts via Docker's host-gateway mapping instead of host-level `/etc/hosts` edits.
- **HTTPRoute Migration:** Envoy HTTPRoutes expose the npm, Go, and PyPI registries on their dedicated hostnames across environments.

### Recent Progress (Registry Resilience - Feb 2026)

- **Dockerfile fallback:** All Node-based service and app Dockerfiles now default `NPM_CONFIG_REGISTRY` to `https://registry.npmjs.org/`, with `ARG NPM_REGISTRY` permitting overrides. The workspace `.npmrc` still pins the `@ameide` scope to Verdaccio, so internal packages resolve identically, while third-party installs keep flowing even if `npm.packages.dev.ameide.io` is degraded.
- **Opt-in override:** CI/CD jobs or air-gapped environments can restore the Verdaccio-only path by passing `--build-arg NPM_REGISTRY=http://npm.packages.dev.ameide.io`, keeping the build knobs explicit instead of hardcoding the internal endpoint.

### Recent Progress (Go/Python SDKs)

- **Go SDK parity:** `scripts/sync-proto.mjs` now mirrors the TS guardrails. It asserts `packages/ameide_core_proto` is present, runs `pnpm -F @ameide/core-proto run generate:go-sdk`, and verifies `.pb.go`/Connect outputs before returning success. `client.go` now exposes the same service surface as the TS SDK, and the legacy `ameide_protos` build tags/stubs were removed so `go build` fails fast when generated types are missing instead of silently substituting no-op clients.
- **Python SDK parity:** `scripts/sync_proto.py` aligns with the TS flow by validating the proto workspace, copying the raw `.proto` tree into `src/ameide_sdk/proto/raw`, and regenerating stubs only when `_pb2.py` files are absent. `pyproject.toml` now ships the raw proto snapshots with the wheel, ensuring downstream tools have the same inputs the TS SDK publishes.
- **Validation:** `python3 -m compileall -q src` succeeds with the stricter Python packaging. `go test ./...` now passes after trimming the Go client surface to the proto packages emitted by `packages/ameide_core_proto`.
- **Release automation:** Added `release/publish_go.sh`, `release/publish_python.sh`, and `publish_all_sdks.sh` so Go/Python packaging mirrors the TS workflow. The helpers now generate local artifacts under `packages/ameide_sdk_{go,python}/dist` for manual upload.
- **Registry access:** Verdaccio now runs fully open for local development—no htpasswd secret required—and all package scopes default to `$all` for publish/unpublish. This simplifies developer onboarding and matches the friction-free experience of the Go/PyPI mirrors.
- **Tilt parity:** SDK Tilt resources call `release/tilt_ameide_sdk_{ts,go,python}.sh` for uniform install/test logging, matching the standalone `release/verify_all_sdks.sh` runner.

### Recent Progress (Package Registries - Jan 2026)

- **Unified package endpoints:** Envoy Gateway now fronts npm (`npm.packages.dev.ameide.io`), Go (`go.packages.dev.ameide.io`), and PyPI (`pypi.packages.dev.ameide.io`) registries, all deployed via Helmfile with emptyDir defaults for local dev.
- **Private Go proxy:** New `go-module-registry` service accepts multipart uploads from `release/publish_go.sh`, persists module metadata, and serves the GOPROXY endpoints required by `go get`/`go install`. The chart now publishes an Envoy `HTTPRoute`, so `https://go.packages.dev.ameide.io/healthz` answers 200 out of the box.
- **Internal PyPI mirror:** `pypiserver` powers `pypi.packages.dev.ameide.io`; `release/publish_python.sh` now builds wheels/sdists and uploads them with Twine using the seeded `admin/admin123` credentials. An accompanying `HTTPRoute` exposes the service at `https://pypi.packages.dev.ameide.io/`.
- **Tilt host mapping & health checks:** The shared Docker `extra_hosts` helper maps all `packages.*` hostnames (npm, Go, PyPI) through host-gateway or bridge IP so container builds resolve the registries without editing `/etc/hosts`. Tilt’s registry resources were consolidated under the `packages:*` namespace (`packages:health`, `packages:status`, `packages:ts-sdk-publish`, `packages:ts-sdk-smoke`, `packages:go-sdk-publish`, `packages:go-sdk-smoke`, `packages:python-sdk-publish`, `packages:python-sdk-smoke`) and now probe the HTTPS listeners successfully.

### Recent Progress (Registry Hardening - Jan 2030)

- **Gateway parity:** Envoy Gateway now exposes `docker.io/ameide` via a dedicated listener and HTTPRoute that forwards to the k3d registry using an ExternalName service, allowing pushes/pulls to stay inside the Gateway surface.  
- **DNS guardrails:** CoreDNS gained a `stopHosts` allow-list (with `docker.io` seeded for local dev), and the infrastructure health check now fails fast when DNS cannot resolve the registry.  
- **Containerd automation:** The new `registry-mirror` Helm component installs the insecure mirror ConfigMap and restarts containerd/k3s automatically, replacing the manual `scripts/infra/apply-registries-config.sh` step.  
- **Tooling alignment:** Local build and maintenance scripts now default to `docker.io/ameide`, keeping Docker pushes and diagnostics aligned with the new hostname.

### Parity Status (Go/Python SDKs)

- Go: private GOPROXY deployed at `go.packages.dev.ameide.io`; remove the lingering `replace go.packages.dev.ameide.io/ameide-sdk-go` overrides once downstream builds consume it by default.
- Python: internal PyPI mirror available at `pypi.packages.dev.ameide.io`; migrate Dockerfiles to `pip install ameide-sdk` and drop local `packages/ameide_sdk_python` mounts.
- Proto: remain source of truth; ensure builds that publish Go/Python SDKs run after proto changes.

### Open Issues

- ✅ **RESOLVED (Nov 2, 2025)**: Verdaccio ingress returns HTTP 200 for `GET /@ameideio/ameide-sdk-ts` via `npm.packages.dev.ameide.io`, with CoreDNS rewrites ensuring pods resolve the hostname.
- ✅ **RESOLVED (Nov 2, 2025)**: Tilt-triggered Docker builds resolve the `packages.*` hosts using the shared `extra_hosts` helper, eliminating prior `getaddrinfo ENOTFOUND` failures.
- Historical OOM and LiveUpdate rebuild issues no longer block the current path (services no longer run TSC inside containers), but we should confirm the new registry flow eliminates the `ERR_MODULE_NOT_FOUND` crash once Tilt builds pick up the Verdaccio hostname reliably.
- `release/publish.sh` still packages `@ameide/core-proto` alongside the SDK; either update the tooling to skip the proto tarball or reflect the continued proto publish in this backlog item so the scope stays consistent.

### Follow-Up Tasks (All SDKs)

1. **TS SDK Runtime Consumption:** ✅ Tilt now pulls the SDK from Verdaccio; once Verdaccio serves 200/metadata, verify `dist/index.js` and `dist/proto/raw` land in each service image. *Owner: Platform DX.*
2. **Go SDK Packaging:** ✅ Go proxy online; follow-up is removing the service-level `replace go.packages.dev.ameide.io/ameide-sdk-go` overrides once downstream builds default to the proxy. *Owner: Core SDK (Go).* 
3. **Python SDK Distribution:** ✅ PyPI mirror active; migrate Dockerfiles to `pip install ameide-sdk` so images stop mounting `packages/ameide_sdk_python`. *Owner: Core SDK (Python).* 
4. **Proto + SDK Release Pipeline:** ✅ `publish_all_sdks.sh` now packages TS/Go/Python artifacts together; update release playbooks/tag conventions once registries are ready. *Owner: Core SDK / Release Engineering.*
5. **CI Guardrail:** Extend `scripts/sdk_policy_enforce_proto_imports.sh` (or add lint) to block reintroducing `packages/ameide_core_proto` into service Dockerfiles, Tilt, or runtime scripts. *Owner: Platform DX.* (Host-gateway change reduces the need to edit Dockerfiles; lint still required.)
6. **Docs & Playbooks:** Update `docs/ci-cd-playbook.md`, SDK READMEs, and CONTRIBUTING to describe the Verdaccio/npm/go proxy/uv publish flow and downstream consumption. *Owner: Docs Guild.*
7. **Verification:** Add automated checks (e.g., integration tests or smoke builds) that install each SDK from the registry/proxy to ensure artifacts contain all generated code. Include a Verdaccio health gate (Tilt `local_resource`) so failures surface early. *Owner: Core SDK.*

### Risks / Watchpoints

- Merge conflicts in long-lived branches may reintroduce proto references; add code-review checklist items.
- SDK builds fail unless `pnpm generate:ts` outputs stay current; schedule periodic cron to run `buf lint` + SDK builds (publish script now enforces this for TS, but Go/Python still at risk).
- External contributors might depend on the now-removed fallback; communicate the new workflow in CONTRIBUTING (call out the need to run `helmfile -e local sync` so the `registry-mirror` release applies the mirror automatically).
- Verdaccio availability is now the single point of failure: if the registry pod or Envoy route breaks, all service builds halt. Add alerting and push-button remediation (restart Verdaccio, reapply registry mirror) to the playbook.

### Exit Criteria

- CI guardrail landed and passing.
- No service Dockerfile or Tilt section references `packages/ameide_core_proto` (enforced by lint).
- SDK README and CI docs updated.
- Release pipeline validated by publishing a dry-run SDK build from main.
- ✅ **COMPLETED**: Tilt dev env stabilizes: platform/graph/threads/transformation pods boot using SDK distribution without proto workspace, and Verdaccio responds 200 for `@ameideio/ameide-sdk-ts` during `pnpm install --no-frozen-lockfile` at `npm.packages.dev.ameide.io`.

### Decision Log (Nov 2025)

| Date | Decision | Rationale | Follow-up |
|------|----------|-----------|-----------|
| 2025-11-01 | Keep Verdaccio as the authoritative npm registry and avoid falling back to “build & copy” proto assets. | Maintains single source of truth and aligns with backlog goal (services consume SDK distributions only). | Ensure Verdaccio is resilient (health checks + docs). |
| 2025-11-01 | Resolve Docker build DNS by adding host-gateway mapping via Tilt, not by editing each Dockerfile or tweaking CoreDNS. | Matches vendor guidance (Docker/Tilt). Keeps CoreDNS focused on in-cluster traffic. | Document the helper change; monitor for regressions when new services are added. |
| 2030-01-15 | Replace the ad-hoc mirror script with the Helm-managed `registry-mirror` release. | Keeps containerd mirrors in sync automatically across developer and CI clusters. | Remove legacy script references from docs; ensure Helmfile runs as part of onboarding. |
| 2025-11-01 | Use `packages:npm-publish` (Tilt resource) to regenerate/publish packages before dependent builds run. | Guarantees Verdaccio hosts the latest SDK snapshots; no manual npm publish required. | Investigate Verdaccio 500 responses; consider versioned dev tags to avoid unpublish/publish loops. |
| 2025-11-01 | Defer CoreDNS changes for host builds; treat build networking separately. | Prevents accidental cluster-side regressions; keeps k3s defaults intact. | None—host mapping covers the build use case. |
| 2025-11-02 | Serve Verdaccio via `docker.io` on port 8080 while Docker images stay on `docker.io/ameide`. | Aligns build-time DNS with Docker/Tilt host-gateway guidance and avoids per-service overrides. | Update Tiltfile, HTTPRoute, and SDK docs to reference the shared hostname. |
| 2025-11-02 | Update namespace CoreDNS configuration to rewrite `docker.io` and retire the temporary compatibility alias. | Keeps in-cluster consumers aligned with the new hostname. | Verify Helm release notes; ensure DNS caches pick up the change post-upgrade. |
| 2026-01-12 | Deploy dedicated package registries (`npm.packages.dev.ameide.io`, `go.packages.dev.ameide.io`, `pypi.packages.dev.ameide.io`) via Helmfile. | Keeps every language SDK on a curated internal registry with reproducible infrastructure. | Migrate service Dockerfiles and build scripts to consume the registries instead of mounting workspace packages. |
