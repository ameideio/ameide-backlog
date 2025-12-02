# Docker Buildx via Kubernetes Driver

> ⚠️ **Archived (Nov 6, 2025):** Local Tilt builds no longer provision a Kubernetes Buildx builder.
> Images are built with `docker build` against the embedded `k3d-dev-reg:5000` registry. This note
> remains for historical reference only.

**Created:** Nov 5, 2025  
**Owners:** Platform DX / Developer Productivity

## Problem Statement

Docker builds executed from Tilt run inside BuildKit containers that live on Docker’s default `bridge` network. Cluster-only hostnames such as `go.packages.dev.ameide.io` resolve to `127.0.0.1` in that network, so Go module downloads and other in-cluster calls fail with `connection refused`. We previously band-aided this by overriding `/etc/hosts` entries, but that breaks whenever endpoints move or services require TLS.

## Goals

- Run all Tilt-triggered Docker builds inside the k3d network using Buildx + the Kubernetes driver.
- Ensure the BuildKit pods inherit cluster DNS so `*.dev.ameide.io` resolves through Envoy without extra host overrides.
- Keep the workflow reproducible for every developer (DevContainer users, bare-metal Docker, CI).
- Document the bootstrap steps and fallbacks.

## Architecture Overview

- **Builder Provisioning:** `scripts/infra/ensure-buildx-k8s.sh` creates a named builder (`ameide-k8s`) using Docker Buildx’ Kubernetes driver. The builder runs BuildKit as pods in namespace `ameide`.
- **Networking:** Because BuildKit runs in-cluster, it shares k3d’s network topology and can reach internal services (Athens proxy, Verdaccio, etc.) directly.
- **Registry Access:** `infra/docker/buildkitd.toml` marks `docker.io/ameide` as an insecure HTTP registry so BuildKit can push images. Buildx loads this config via `--buildkitd-config`.
- **k3d Registry:** `infra/docker/local/k3d.yaml` now configures both `k3d-dev-reg:5000` and `docker.io/ameide` with `insecureSkipVerify: true`, letting cluster workloads pull images over HTTP. Any config change requires `scripts/infra/ensure-k3d-cluster.sh` to recreate the cluster.
- **Tilt Integration:** The Tiltfile runs the ensure script during startup, then uses a `custom_build_with_restart()` wrapper that shells out to `docker buildx build --builder ameide-k8s --push ...` when building Go-based services.

## Required Tooling

- Docker CLI 25.x with Buildx plugin (bundled on recent versions but still required).
- Access to the k3d cluster and kubeconfig context `ameide-local`.
- `kubectl` on the PATH (used by the ensure script).

## Bootstrap Steps

1. **Scripts:** Commit the ensure script and BuildKit config.
   ```bash
   scripts/infra/ensure-buildx-k8s.sh
   infra/docker/buildkitd.toml
   ```
2. **Tiltfile:** Run the ensure script at startup and replace `docker_build_with_restart()` calls with `buildx_build_with_restart()` for Go services.
3. **DevContainer:** Hook the ensure script into the container’s startup (see “DevContainer Integration” below).
4. **CI/CD:** Before invoking Tilt or standalone Buildx commands, run the ensure script (`scripts/infra/ensure-buildx-k8s.sh`) so the builder exists.

## DevContainer Integration

- The `postCreateCommand` invokes `scripts/infra/ensure-buildx-k8s.sh` after k3d reconciliation so the `ameide-k8s` builder exists before Tilt starts (see `backlog/354-devcontainer-tilt.md` for the full bootstrap sequence).
- Tilt owns all image builds once it comes up; no additional Docker commands run in the bootstrap script. This keeps registry imports aligned with the `packages:registry-images` local_resource.
- Principle: DevContainer scripts stop at provisioning prerequisites (cluster + builder). Helm/Tilt layers own every deployment and reconciliation step once the workspace is ready.
- Optionally set `AMEIDE_BUILDX_BUILDER` or `AMEIDE_BUILDX_NAMESPACE` via environment variables in the DevContainer config if per-developer overrides are needed.
- Ensure the DevContainer image includes recent Docker CLI + Buildx (VS Code’s default images already do).

## CI/CD Guidance

- CI jobs that run Tilt or standalone Buildx builds must call `scripts/infra/ensure-buildx-k8s.sh` after authenticating to the cluster.
- Developers launching Tilt manually run `scripts/dev/start-tilt.sh`, which now invokes the ensure script automatically.
- If CI uses a different namespace, pass `AMEIDE_BUILDX_NAMESPACE=<namespace>` to the script.
- When pushing to remote registries, update `infra/docker/buildkitd.toml` accordingly or provide an alternative config via `AMEIDE_BUILDKITD_CONFIG`.

## Reproducibility Checklist

- [ ] `scripts/infra/ensure-buildx-k8s.sh` committed and executable.
- [ ] `infra/docker/buildkitd.toml` checked in.
- [ ] Tiltfile uses `buildx_build_with_restart()` for Go images.
- [ ] DevContainer runs the ensure script on creation/update (and defers image builds to Tilt).
- [ ] CI pipelines call the ensure script before builds.
- [ ] Docs link added from the Tilt/Go packages backlog entries.
- [ ] `infra/docker/local/k3d.yaml` mirrors include `insecureSkipVerify: true` and the k3d cluster has been reconciled after the change.

## Open Questions / Follow-ups

- Should the ensure script also verify BuildKit pod replica count/health and recreate the builder when pods crash?
- Investigate caching strategies (e.g., BuildKit registry cache) now that builds run in-cluster.
- Evaluate Buildx for the TypeScript/Python images to remove remaining host overrides.
