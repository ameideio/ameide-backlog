# 395 – SDK Build, Docker, and Tilt North Star (workspace-first rings)

**Status:** Draft (align on Option A)  
**Owner:** Platform DX / Build Infra  
**Updated:** 2025-02-21

Doc relationships: backlog/388-ameide-sdks-north-star.md (why/versioning), backlog/390-ameide-sdk-versioning.md (version computation), backlog/391-resilient-cd-packages-workflow.md (cd-packages outputs), backlog/393-ameide-sdk-import-policy.md (runtime proto/SDK policy), backlog/394-dockerfile-tilt.md (Tilt/Tiltfile structure).

## Target shape (rings, aligned with 402/403/404/405)

- **Ring 1 (dev/Tilt/PR CI/prod images built in-repo):** workspace-first. TS/Go/Py services consume `packages/ameide_core_proto` stubs + workspace AmeideClient (`packages/ameide_sdk_{ts,go,python}`). Dockerfile.dev uses `--no-frozen-lockfile` / `GOWORK=auto` / editable `uv` installs; no registry/Buf dependence.
- **Ring 2 (publish smoke + external consumers):** published artifacts only. `cd-packages` emits `NPM_VERSION` / `PYPI_VERSION` / `GO_VERSION`; `cd-service-images` builds prod images with stamped locks, `GOWORK=off`, and tagged modules/wheels/npm packages to validate the release graph.
- **Two Dockerfiles per service:** `.dev` for Tilt/inner loop (workspace deps), `Dockerfile` for release images (frozen installs, published artifacts). Tilt always uses `.dev`; `cd-service-images` uses `Dockerfile`.
- **Registry/auth:** BuildKit secrets still carry tokens for private GitHub/npm if needed; Go prod builds can use the default proxy with `GOPRIVATE/GONOSUMDB=github.com/ameideio`. Buf BSR is only needed in Ring 0 for generation, not in service images.
- **No per-service proto generation:** Buf runs in `packages/ameide_core_proto`; Docker/Tilt build contexts copy the generated outputs, not `.proto` sources or buf.gen files.

**Ring application to Dockerfiles (quick reference, details in backlog/405-docker-files.md):**

- Dev/Tilt (`Dockerfile.dev`): workspace stubs + workspace SDKs; TS uses `pnpm install --no-frozen-lockfile`, Go uses `GOWORK=auto` with `go.work` + workspace modules, Python uses `uv sync` with editable installs. No Buf CLI/BSR or SDK registry access needed.
- CI/prod (`Dockerfile`): stamped locks and published SDK artifacts only; TS `pnpm install --frozen-lockfile` with `@ameideio/ameide-sdk-ts@$NPM_VERSION`, Go `GOWORK=off` + `go mod download` for tagged `ameide-sdk-go`, Python `uv sync --frozen` with `ameide-sdk-python==<version>`. Validates the release graph seen by external consumers.

## CI / Docker rules (prod images, Ring 2)

- Build from `services/*/Dockerfile` with `--frozen-lockfile` / `uv sync --frozen` / `go mod download` and `GOWORK=off`.
- Inject SDK versions from `cd-packages` outputs (`GO_SDK_VERSION`, `NPM_VERSION`, `PYPI_VERSION`) instead of copying workspace packages into the image.
- For Go images, use default proxies plus `GOPRIVATE/GONOSUMDB=github.com/ameideio`; fetch tagged `github.com/ameideio/ameide-sdk-go@<GO_VERSION>`. No `sdk/go` cache or Buf GOPROXY required.
- Validate with smoke tests that `go get`, `pnpm add @ameideio/ameide-sdk-ts@$NPM_VERSION`, and `pip install ameide-sdk-python==$PYPI_VERSION` succeed from registries.

## Tilt / inner loop rules (dev images, Ring 1)

- Tilt builds only `Dockerfile.dev` variants. Those use workspace links (`workspace:*`, `go.work` with `GOWORK=auto`, editable `uv` installs) and **never** rewrite manifests to published versions or require registry/Buf access.
- Guardrails stay on: proto barrel checks, SDK proto import bans, no service-local `buf.gen.yaml`, no `go.mod` replaces pointing at registries; services import stubs from `packages/ameide_core_proto`.
- SDK local_resources (core_proto + SDKs) remain manual/triggered as needed; they are not required for a standard service dev loop unless you are editing SDKs.
- Devcontainer bootstrap configures Buf credentials once per session via `scripts/lib/buf_netrc.sh`, which reads `BUF_TOKEN` from `.env`, writes `~/.netrc_buf`, and exports `NETRC`. Tilt, guardrail scripts, and pnpm tasks now assume that NETRC exists and never shell out to `buf login` mid-workflow.
- **Coexistence with ArgoCD (RollingSync):** Tilt temporarily disables Argo’s auto-sync for every `apps-*` release before it builds/dev deploys, then exposes a manual `resume-argocd-apps` hook so developers can hand control back to Argo when they are done. This is enforced via `tilt/tilt-argocd-sync.sh`, which uses `kubectl patch` to toggle `spec.syncPolicy.automated` on the generated Applications. The rule of thumb is “Ring 1 owns the release while Tilt is active; Ring 2 (Argo) owns it once you trigger the resume helper.” This prevents Argo from reverting a dev workload back to the Ring 2 image mid-session.

## Gaps / follow-ups

- Ensure `cd-service-images` docs/pipelines mirror the Ring 2 rules (stamped locks, `GOWORK=off`, tagged SDKs, no workspace copies).
- Delete any remaining references to `sdk/go` caches or Buf GOPROXY in service Dockerfiles and guardrail scripts (align with 404/405).
- Keep `scripts/sdk_policy_check_go_replaces.sh` and lock hygiene in CI so registry vs workspace modes don’t regress.
