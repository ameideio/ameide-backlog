> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

> **⚠️ DEPRECATED – SUPERSEDED BY [435-remote-first-development.md](435-remote-first-development.md)**
>
> This document describes the **local k3d registry** workflow which is no longer used.
> The project has migrated to a **remote-first model** where:
> - **No local k3d cluster** – All development targets shared AKS dev cluster
> - **No local registry** – Images push to GHCR (`ghcr.io/ameideio/...`)
> - **Telepresence intercepts** – Traffic routing without local container builds
>
> See [435-remote-first-development.md](435-remote-first-development.md) for the current approach.

# Registry Routing Hardening

**Created:** Jan 2030  
**Owners:** Platform DX / Developer Experience

## Problem Statement

Local clusters rely on a plain-HTTP k3d registry that must remain reachable at `docker.io/ameide`, while language package mirrors (npm/Go/PyPI) live behind Envoy on the `packages.*` hostnames. The current CoreDNS override rewrites every `*.dev.ameide.io` lookup to the Envoy Gateway service (`infra/kubernetes/charts/platform/coredns-config/templates/configmap.yaml`), but Envoy does not listen on port 5000. Kubelet therefore resolves `docker.io` to Envoy, image pulls hit the wrong endpoint, and pods fail with `ImagePullBackOff`. Even if DNS is corrected manually, containerd previously required the ad-hoc `scripts/infra/apply-registries-config.sh` mirror to be run on every node, leaving developer clusters inconsistent.

## Desired End State

- Developers can push/pull images against a single local registry endpoint (`localhost:5000` / `docker.io/ameide`) with no Envoy dependency or manual containerd edits beyond the k3d-provisioned mirror.  
- CoreDNS continues to rewrite only the hostnames that traverse Envoy (packages, web, API); `docker.io` resolves directly to the local registry when present and otherwise falls back to `localhost`.  
- Containerd mirror configuration ships automatically with the platform stack so every developer or CI environment trusts the HTTP registry without manual scripting.

## Implemented Solution

1. **k3d-managed registry at cluster creation**  
   - `infra/kubernetes/scripts/setup-cluster.sh` now calls `k3d cluster create ... --registry-create k3d-dev-reg:0.0.0.0:5000 --registry-config infra/kubernetes/environments/local/registries.yaml`.  
   - The registry container (`k3d-dev-reg`) is published on `localhost:5000`, and k3d injects the mirror into every node automatically.

2. **Containerd mirror via registries.yaml (no DaemonSet)**  
   - `infra/kubernetes/environments/local/registries.yaml` maps `docker.io/ameide` to the in-cluster alias `http://k3d-dev-reg:5000`.  
   - The `registry-mirror` Helm component is disabled for the local environment; mirror provisioning happens entirely during cluster bootstrap.

3. **Host convenience + documentation**  
   - README/onboarding now instruct developers to add `127.0.0.1 docker.io` to `/etc/hosts` if they prefer the friendly hostname when pushing, otherwise use `localhost:5000`.  
   - Tilt comments, helper scripts, and the GC utility reference the `k3d-dev-reg` container directly.

4. **Gateway/CoreDNS untouched**  
   - Envoy Gateway no longer exposes a registry listener in local environments, and CoreDNS rewrites remain focused on application/package hostnames.

## Verification

- Running `k3d cluster create ameide --registry-create k3d-dev-reg:0.0.0.0:5000 --registry-config infra/kubernetes/environments/local/registries.yaml` produces a cluster whose nodes already trust the registry (validated by pulling an image via `kubectl run --image=k3d-dev-reg:5000/...`).
- `scripts/infra/helm-ops.sh preflight dns docker.io` succeeds after adding the optional `/etc/hosts` entry, confirming host resolution for legacy tooling.
- Pushing a test image to `localhost:5000/foo:dev` and deploying it via `kubectl run` completes without Envoy involvement. Envoy logs remain unchanged.
- `scripts/registry-gc.sh --stats` reports against the `k3d-dev-reg` container, proving helper tooling understands the new registry name.
- `infra/kubernetes/scripts/run-helm-layer.sh infra/kubernetes/helmfiles/60-qa.yaml` now succeeds end-to-end; the container catalog probe reaches the in-cluster alias and the remaining smoke jobs stay green.

## Documentation Updates

- `README.md` and `infra/kubernetes/README.md` now show the simplified `k3d cluster create ... --registry-create k3d-dev-reg:0.0.0.0:5000 --registry-config ...` command and call out the optional `/etc/hosts` entry for `docker.io`.
- Inline comments in the `Tiltfile` describe the registry as the direct k3d endpoint rather than an Envoy listener.
- `scripts/registry-gc.sh` defaults to the `k3d-dev-reg` container, matching the new k3d naming scheme, and helper messaging reflects the direct registry setup.
- Onboarding notes in `infra/kubernetes/scripts/setup-cluster.sh` explain where the registry lives and how to access it from host and cluster.


## Tasks

- [x] Switch local cluster bootstrap to `k3d-dev-reg` and rely on k3d’s built-in registries.yaml provisioning.  
- [x] Update `infra/kubernetes/environments/local/registries.yaml` to mirror `docker.io/ameide` to `k3d-dev-reg:5000`.  
- [x] Disable the `registry-mirror` Helm release for the local environment to avoid redundant host patching.  
- [x] Refresh documentation, helper scripts, and Tilt comments to describe the new registry workflow.  
- [x] Surface `docker.io` inside the cluster (Service/Endpoints or CoreDNS stub) so pods resolve to the k3d registry instead of `127.0.0.1`.  
- [x] Point `charts/platform-smoke` at the in-cluster registry service (or gracefully skip the container catalog check when the service is absent).  
- [x] Migrate remaining Helm charts to `image.repository`/`image.tag` so Tilt always injects `k3d-dev-reg` coordinates. *(Core services, migrations, Tekton helpers, and ClickHouse/Plausible bootstrap now rely on the new fields; docs/readmes and linting have been aligned with the new schema.)*  
- [ ] Add an optional shell wrapper (or Make target) that injects `--add-host docker.io:host-gateway` for ad-hoc Docker builds outside Tilt.  
- [ ] Evaluate whether integration tests need a separate configuration when the registry hostname is mapped via `/etc/hosts` versus `localhost:5000`.

## Cleanup Checklist

- [x] Retire `scripts/infra/apply-registries-config.sh` now that the Helm-managed mirror ships automatically (script deleted; docs updated accordingly).  
- [x] Update build scripts (`build/scripts/build-all-images.sh`, `build/scripts/verify-digests.sh`, `scripts/registry-gc.sh`, etc.) to default to `docker.io/ameide`, keeping configurable overrides for CI.  
- [x] Sweep Dockerfiles, npm configs, and service READMEs for drift and align them with the final hostnames (`docker.io/ameide`, `packages.*`).  
- [x] Remove ad-hoc `/etc/hosts` edits or devcontainer hooks that hardcode registry IPs once DNS + Envoy routing is reliable.  
- [x] Re-run Tilt smoke tests (`packages:health`, `packages:status`, `packages:ts-smoke`) and document the expected success output using the new routing. *(All three checks now succeed; `packages:ts-smoke` verifies the Verdaccio tarball includes `dist/index.js` and `dist/proto/raw`.)*  
- [x] Ensure CI workflows (`cd-service-images.yml`, `cd-packages.yml`) reference the same hostnames so dev/staging remain in lockstep (confirmed they already target `ameide.azurecr.io`/Verdaccio and required no changes).

## Impact on Package Registries

The hardening work targets the container registry path only. Package registries (npm/Go/PyPI) stay on their existing Envoy listeners and CoreDNS rewrites; the registry mirror DaemonSet touches only containerd’s `docker.io/ameide` configuration. SDK publishing and installs keep using the `packages.*` hostnames without change.

## Open Issues

- Provide a documented (and cross-platform) wrapper for `docker build` that injects the necessary `--add-host ... host-gateway` entries so ad-hoc builds resolve `docker.io` and `packages.*` without manual flags.
- Confirm integration and CI workflows that invoke Docker outside the devcontainer either use the wrapper or rely on `localhost:5000`, and document the expectation.
- Capture troubleshooting steps for Windows/macOS developers where editing `/etc/hosts` requires elevation—consider offering a lightweight script or alternate hostname (`localhost:5000`) in all examples.

## Unified Design Proposal

- **Registry backend plumbing**: Replace the `ExternalName` hop with a `Service` (no selector) plus `EndpointSlice` that points to the k3d registry’s reachable address, following Envoy Gateway’s “routing outside Kubernetes” pattern. Attach a `BackendTrafficPolicy` if we need upstream connect/idle tuning.
- **HTTPRoute timeouts & behavior**: Configure the registry `HTTPRoute` with `timeout: 0s`, set `idle` timeout to ~3600s, and enable `proxy_100_continue` via Gateway extensions if Docker pushes show handshake stalls. Ensure access logging is on so we can spot 408/504 or retry loops.
- **DNS / node resolution**: Adopt Pattern B—pods still hit Envoy for `packages.*`, but nodes resolve `docker.io` directly to the registry. Provision `/etc/rancher/k3s/registries.yaml` with the correct mirror endpoint and add a node-level preflight (`nsenter` + `getent`) so we fail fast if resolution/trust is off.
- **Containerd mirror provisioning**: Prefer k3d’s native `--registry-create` / `--registry-config` flags so every node starts with the mirror config. Keep a privileged DaemonSet (with bundled `nsenter`/restart tooling) only as an opt-in remediation path for brownfield clusters.
- **Docker host mapping**: Ship a shared `docker` wrapper (or CLI plugin) that injects the required `--add-host … host-gateway` entries so ad-hoc builds resolve `packages.*` the same way Tilt does. Store the host list in a single config that both Tilt and the wrapper read.
- **Onboarding & preflight**: Extend onboarding scripts to run the Docker wrapper preflight (host + container resolution) and the node-level DNS check. Document troubleshooting (e.g., regenerating `registries.yaml`, restarting containerd) alongside the wrapper usage.
- **Observability & CI guardrails**: Enable Envoy access logs/metrics for the registry listener, add a CI job that performs a large test push via the wrapper, and ensure Tilt/wrapper share config so we detect drift early. Update READMEs/runbooks to reflect the Envoy-routing + direct-node access split.

## Vendor Review Notes

- Chosen approach aligns with k3d/k3s guidance: use `--registry-create` + `--registry-config` at cluster creation so containerd trusts the local registry without in-cluster mutations.
- Skipping an Envoy listener for the registry avoids unsupported Gateway API backends and eliminates timeout tuning for long Docker uploads.
- Package registries remain behind Envoy/CoreDNS; only the container registry bypasses Envoy, matching the split-horizon model warned about in earlier reviews.

## Lightweight Dev-Box Plan (No Envoy/CoreDNS/DaemonSet)

- **Cluster + registry one-liner**: `k3d cluster create ameideegistry-create k3d-dev-reg:0.0.0.0:5000` spins up a local `registry:2`, binds it to `localhost:5000`, and seeds every node’s `registries.yaml` with the mirror automatically.
- **Push/pull workflow**: push from the host with `docker push localhost:5000/<image>` and pull in the cluster via the k3d alias (`k3d-dev-reg:5000/<image>`), matching the k3d docs.
- **Keep vanity hostname**: optionally supply a `registries.yaml` that maps `docker.io/ameide` to `http://k3d-dev-reg:5000` during cluster creation and add a host-only `/etc/hosts` entry for convenience.
- **Docs/onboarding impact**: set `REGISTRY_HOST` defaults (e.g., `k3d-dev-reg:5000`), document the two command snippets above, and note that this path is dev-only/insecure HTTP.
- **What we avoid**: no Envoy listener, CoreDNS rewrite, or privileged DaemonSet; all registry wiring happens at cluster creation using vendor-supported flags.

## Expected Outcomes

- `k3d cluster create ... --registry-create k3d-dev-reg:0.0.0.0:5000 --registry-config ...` yields a cluster whose nodes can pull `k3d-dev-reg:5000/<image>` without additional setup (validated via `kubectl run`).
- Host pushes to `localhost:5000/<image>` (or `docker.io/ameide/<image>` after adding the optional `/etc/hosts` entry) succeed, and the image is immediately consumable by workloads.
- `scripts/infra/helm-ops.sh preflight dns docker.io` and `infra/kubernetes/scripts/infra-health-check.sh` pass on fresh machines once the `/etc/hosts` entry is applied.
- `scripts/registry-gc.sh --stats` reports registry usage against the `k3d-dev-reg` container, confirming tooling alignment with the new naming.
- Tilt-driven builds continue to publish to `docker.io/ameide/...` and deploy successfully; no Envoy listener or CoreDNS tweaks are required to support image pushes.
- Documentation/onboarding steps run end-to-end without invoking the retired `registry-mirror` DaemonSet or manual mirror scripts.

```
Local registry workflow

Devcontainer / Host (Tilt, docker wrapper)
    |
    | (1) docker build / push → localhost:5000 or docker.io/ameide
    v
Local Registry Container (k3d-dev-reg)
    |
    | (2) registries.yaml mirror → http://k3d-dev-reg:5000
    v
k3s Nodes (containerd)
    |
    | (3) kubectl deploy pulls image
    v
Application Pods

Package dependency workflow

Build scripts / Tilt ──(P1 DNS via CoreDNS)──▶ Envoy packages listeners ──(P2)──▶ Verdaccio / Go proxy / PyPI
```
