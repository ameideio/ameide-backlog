# 344 – Packages Versioning Strategy

## Goal

Ensure every internally published package (TypeScript, Python, Go) shares a single
snapshot strategy so Tilt, Helm, and CI runs behave deterministically without
forcing developers to pin stale artifacts.

## Snapshot Format

- **TypeScript** (`@ameide/*` packages)  
  Format: `<base>-dev.<UTC timestamp>.<git sha>`  
  Example: `0.12.0-dev.20251106083012.a0e03c2d4532`  
  Source of truth: `scripts/tilt-packages-npm-publish.sh`
- **Python (SDK, Buf-managed)**  
  Format: `<base>.dev<UTC timestamp>+g<git sha>`  
  Example: `0.1.0.dev20251106082643+ga0e03c2d4532`  
  Base version lives in `ameide_sdk.version.BASE_VERSION`; the legacy telemetry package was retired when Buf-generated wheels replaced it.
- **Go (SDK)**  
  Format: `v0.0.0-<UTC timestamp>-<git sha>` (Go pseudo version)  
  Example: `v0.0.0-20251106082752-a0e03c2d4532`

For all languages, explicit overrides (e.g. `AMEIDE_TS_SDK_VERSION`,
`AMEIDE_PYTHON_SDK_VERSION`, `AMEIDE_GO_SDK_VERSION`) still take precedence for
release promotions.

## Publish Scripts

- TypeScript: `scripts/tilt-packages-npm-publish.sh` writes `ts.json` under
  `tmp/tilt-packages/`.
- Python: `scripts/tilt-packages-python-publish.sh` invokes
  `release/publish_python.sh`; the runner persists `python.json`.
- Go: `scripts/tilt-packages-go-publish.sh` emits `go.json`.

### ConfigMap Mapping

| Language / Package        | Local State File          | ConfigMap Key              |
|---------------------------|---------------------------|----------------------------|
| TypeScript – SDK          | `tmp/tilt-packages/ts.json`            | `ts.json`                   |
| Python – SDK              | `tmp/tilt-packages/python.json`        | `python.json`               |
| Go – SDK                  | `tmp/tilt-packages/go.json`            | `ameide-sdk-go.json`        |

All publishers resolve the timestamp via `date -u +%Y%m%d%H%M%S` and derive the
git SHA through `git rev-parse --short=12 HEAD`, falling back to
`SOURCE_GIT_SHA` if the repository metadata is unavailable (e.g. in a release tarball).

## Consumer Guidance

- Docker/Tilt builds always fetch the latest snapshot from the internal
  registries; teams should avoid pinning specific snapshot versions unless
  preparing a release candidate.
- Helm/Tilt workflows read the JSON state files to install consistent versions
  across local dev and CI.

## Sync Helpers

- `infra/kubernetes/sync-go-packages-state.sh` and `infra/kubernetes/sync-python-packages-state.sh`
  copy ConfigMap data into `/workspace/tmp/tilt-packages/` for local development.
- TypeScript consumers read the ConfigMap directly; no sync helper is required.

## Outstanding Follow-Ups

- Regenerate and commit `services/agents/go.sum` once the registry-backed
  `go mod tidy` flow lands.
