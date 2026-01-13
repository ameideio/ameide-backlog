# Devcontainer Bootstrap Simplification

> **Status – Historical Reference:** Remote-first development ([435-remote-first-development.md](../435-remote-first-development.md)) is the supported workflow, and any remaining local k3d bootstrap work should use Terraform per [444-terraform.md](../444-terraform.md). Keep this file for context on how Tilt assumed responsibility for local reconciliation; do not extend it for new remote-first initiatives.
>
> **Update (2026-01):** Tilt-based orchestration (`Tiltfile`, `scripts/dev/start-tilt.sh`, Tilt UI) is deprecated/removed from the canonical workflow. This document remains as historical context only.

**Created:** Nov 2025  
**Owner:** Platform DX / Developer Productivity

## Context

The DevContainer bootstrap script previously ran Helmfile status/sync checks, rebuilt local package registry images, and polled ExternalSecrets before handing control to Tilt. Those steps duplicated logic already codified in the Tiltfile (`packages:registry-images` local_resource, `infra:*` helm resources, `wait_for_tilt_services_ready` health gates) and introduced inconsistent behavior between root and `vscode` users.

## DevContainer bootstrap flow

1. **Workspace preparation (postCreate & postStart)**  
   - Create writable directories (`/workspace/build/generated/langgraph_agents`, `/home/vscode/.kube`) and fix ownership so the `vscode` user can write build outputs.  
   - Ensure global git config points to `/home/vscode/.config/git/config`.

2. **k3d cluster reconciliation (postCreate)**  
   - Run `scripts/infra/ensure-k3d-cluster.sh` (see `backlog/338-devcontainer-startup-hardening.md`) to provision the `ameide` cluster and registry mirror defined in `infra/docker/local/k3d.yaml`.  
   - Merge the generated kubeconfig into `/home/vscode/.kube/config`, wait for all nodes to report `Ready`, and surface warnings if they lag.
   - On subsequent container starts, `.devcontainer/postStartCommand.sh` simply restarts the existing cluster if Docker paused it and re-applies the kube context aliases.

3. **Docker Buildx via Kubernetes driver (postCreate)**  
   - *(Archived)* Legacy flows executed `scripts/infra/ensure-buildx-k8s.sh` to provision a Kubernetes Buildx builder. The current setup relies on the default Docker daemon instead.
   - This step runs before Tilt so subsequent builds automatically share the in-cluster BuildKit pods.
   - Buildx state persists across container restarts; the post-start script reuses the builder without recreating it.

4. **Tooling updates (postCreate)**  
   - *(Historical)* Keep Tilt current (download the latest release if needed) and install Playwright browsers in the persistent cache.  
   - Helm plugin installs remain idempotent (`helm diff`) for parity with CI.

5. **Tilt launch (postCreate & postStart)**  
   - *(Historical)* Tilt used to be launched via `scripts/dev/start-tilt.sh --detached` and served as the single orchestrator.
- **Current workflow:** Use Telepresence + the Ameide CLI:
  - `ameide dev inner-loop verify` / `up` / `down` (interactive UI iteration)
  - `ameide test` (Phase 0/1/2 only; strict verification + JUnit evidence)
  - `ameide test e2e` (cluster-only Playwright E2E)

## Tilt responsibilities

Tilt is now the single orchestrator for the following concerns (see `backlog/339-helmfile-refactor.md` for layer context):

- **Package registry images** — The `packages:registry-images` local_resource shells out to `scripts/infra/build-local-packages-images.sh`, ensuring Verdaccio, devpi, Athens, and publisher images are rebuilt/imported before Helm layers 26–28 execute.
- **Helm layers** — Each `infra:<layer>` resource calls `infra/kubernetes/scripts/run-helm-layer.sh`, mirroring the layered Helmfile structure introduced in backlog 339. The critical path is tracked in `INFRA_CRITICAL_PATH`, so application resources depend on every layer being reconciled.
- **Health checks** — `wait_for_tilt_services_ready` queries `tilt alpha get uiresource -ojson` to confirm runtime/update status, replacing the earlier ad-hoc bootstrap probes (registry health, ExternalSecrets waits). Additional layer-specific smoke tests continue to live in `scripts/infra` and are invoked through the Tilt resources themselves.
- **Image builds** — Go/TS/Python service builds run with `docker build --push`, targeting the embedded `k3d-dev-reg:5000` registry.

## Impact

- Tilt is the single source of truth for image builds, Helm layer reconciliation, and package registry health. DevContainer bootstrap only guarantees the cluster + builder exist and then defers.
- Developers no longer encounter duplicate Helmfile runs or manual registry builds when the container comes up; all reconciliation happens in one place with consistent logging.
- Failures surface through Tilt’s UI/logs, eliminating the need to cross-reference bootstrap summaries.

## Follow-ups

- Verify CI and docs reference the new flow (Tilt owns reconciliation once the environment is provisioned).
- Prune unused environment variables and flags (e.g., `SKIP_PACKAGE_IMAGE_BUILD`) when no longer referenced by Tilt or scripts.
