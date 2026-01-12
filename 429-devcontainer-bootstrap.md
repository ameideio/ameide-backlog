# 429 – DevContainer Bootstrap (k3d + Argo CD + GitOps)

> **⚠️ SUPERSEDED BY [435-remote-first-development.md](435-remote-first-development.md)**
>
> This document describes the **legacy local k3d bootstrap**. The project is migrating to a
> **remote-first development model** where:
> - Infrastructure code lives in `ameideio/ameide-gitops` repo (not `ameideio/ameide`)
> - DevContainer connects to shared remote AKS cluster via telepresence
> - No local k3d cluster or ArgoCD bootstrap
>
> See [435-remote-first-development.md](435-remote-first-development.md) for the new approach.
>
> **Update (2026-01):** 435 itself is superseded by the internal-only, Coder-based model in `backlog/650-agentic-coding-overview.md`.

> **Related documents:**
> - [435-remote-first-development.md](435-remote-first-development.md) – **NEW**: Remote-first development architecture
> - [367-bootstrap-v2.md](367-bootstrap-v2.md) – Design document (bootstrap logic moved to ameide-gitops)
> - [432-devcontainer-modes-offline-online.md](432-devcontainer-modes-offline-online.md) – Devcontainer modes (retired - single remote mode)
> - [434-unified-environment-naming.md](434-unified-environment-naming.md) – Environment naming standardization

This document is the **implementation reference** for the legacy devcontainer bootstrap mechanics. For design rationale and target state, see [367-bootstrap-v2.md](367-bootstrap-v2.md).

## Remote-first bootstrap workflow (2025-12)

Even though this file mainly documents the historical k3d flow, engineers still land here looking for a “what do I run now?” answer. The active remote-first flow is intentionally lightweight and stitches together the documents listed above:

1. **Auto contexts + kube cred refresh** – `tools/dev/bootstrap-contexts.sh` (documented in [491-auto-contexts.md](491-auto-contexts.md)) pulls AKS credentials, sets the unified contexts from [434-unified-environment-naming.md](434-unified-environment-naming.md), and seeds Telepresence defaults. `postCreate.sh` calls this automatically, but you can rerun it as needed. Before the helper runs we now guarantee Telepresence’s system dependencies (`iptables` for DNS/routing and `sshfs` for remote env mounts) are installed in the DevContainer, so every remote-only session starts with a healthy CLI stack.
2. **Argo CD CLI bootstrap** – Follow [491-argocd-login.md](491-argocd-login.md) to port-forward, mint the short-lived token, and run `argocd login argocd.ameide.io --grpc-web --insecure` inside the devcontainer. The wrapper in `.devcontainer/postCreate.sh` now shells out to the same helper so the token lifecycle is consistent.
3. **Environment sanity check** – `kubectl get ns` (expect all `ameide-*` namespaces) and `argocd app list | grep ameide-dev` before starting feature work. This replaces the older “install Argo into k3d” section in this doc.

## End-to-end inner-loop cycle

Per the “refactor & rerun” ask, the current expected cycle for shared services (e.g. `extensions-runtime`) is:

1. **Local build + tests**
   ```bash
   go test ./...
   sudo docker build -t extensions-runtime-dev-local -f services/extensions-runtime/Dockerfile.dev .
   sudo docker build -t extensions-runtime-release-local -f services/extensions-runtime/Dockerfile.release .
   sudo docker run --rm --entrypoint /app/extensions-runtime extensions-runtime-dev-local --version
   ```
   (Any config errors are acceptable as long as glibc/linking and health handlers start cleanly.)
2. **Commit + push** – Stage intentional changes only (`git status` should be clean apart from your work), then `git commit` and `git push`. The DevContainer already hosts a logged-in `gh` CLI.
3. **CI / CD verification** – Watch `gh run watch <run-id>` for the `CD / Service Images` workflow. Success there proves every service built and new images were pushed to GHCR.
4. **Argo CD rollout** – `argocd app sync dev-platform-extensions-runtime && argocd app wait dev-platform-extensions-runtime` followed by `kubectl get pods -n ameide-dev -l app.kubernetes.io/name=extensions-runtime`. Health must reach `Healthy` before moving on; logs should show the gRPC health server coming up.

This cycle was executed after the MinIO endpoint + gRPC health refactor (see commits `62b853d5`, `38452a77`), so the instructions are now proven-good. If anything regresses, update this section so future contributors do not have to rediscover the steps.

### Current bootstrap split (2025-12)

- **GitOps / cluster bootstrap** now lives in the `ameideio/ameide-gitops` repository (`bootstrap/bootstrap.sh`). It installs Argo CD, applies the RollingSync ApplicationSet, and prepares AKS/k3d clusters for every environment. CI/CD and platform operators invoke that script.
- **Developer bootstrap** lives in the `ameideio/ameide` application repo and is intentionally lightweight: `.devcontainer/postCreate.sh` installs/validates `iptables` and `sshfs` (in addition to the baked image packages) and then calls `tools/dev/bootstrap-contexts.sh` to refresh AKS credentials, set the `kubectl`/Telepresence defaults, and log the `argocd` CLI into the shared control plane via a port-forward. This is the only bootstrap that runs automatically when opening the DevContainer.

The remainder of this document describes the historical k3d-based flow for context; defer to [backlog/435-remote-first-development.md](435-remote-first-development.md) plus [491-auto-contexts.md](491-auto-contexts.md) for the active developer bootstrap instructions.

## 1. High-level goals

Opening the repo in a DevContainer should:

- Provision a **local Kubernetes cluster** (k3d) plus an internal registry.
- Install and configure **Argo CD** in a dedicated namespace.
- Register the repo’s **GitOps source** and apply the Argo projects/apps that converge the cluster to the desired state for `dev`.
- Prepare tooling for the inner-loop (**Tilt**, **kubectl**, **helm**, **argocd**, **bicep**, **yq**, Python deps).
- Expose the **Argo CD UI** on `https://localhost:8443` with a known admin credential.
- Do all of this **repeatably and idempotently**, with a **true dry-run mode** and clear **phase timing** and logs.

Everything is orchestrated via:

- `.devcontainer/devcontainer.json`
- `.devcontainer/postCreate.sh`
- `tools/bootstrap/bootstrap-v2.sh` and its `lib/*.sh` modules
- GitOps content under `gitops/ameide-gitops/environments/dev/argocd/**`

This backlog documents the process end-to-end for contributors and for future refactors of the bootstrap pipeline.

---

## 2. Entrypoint: DevContainer lifecycle

### 2.1 devcontainer.json

File: `/.devcontainer/devcontainer.json`

Key points:

- Uses the repo’s `.devcontainer/Dockerfile` as the image build for VS Code / DevContainer:
  - Adds Docker-in-Docker support via `ghcr.io/devcontainers/features/docker-outside-of-docker`.
  - Adds `kubectl` and `helm` via `ghcr.io/devcontainers/features/kubectl-helm-minikube`.
- Binds the host Docker socket:
  - `mounts: ["source=/var/run/docker.sock,target=/var/run/docker.sock,type=bind"]`
  - This is required for `k3d` to manage the cluster from inside the container.
- Injects a few **containerEnv** values from the host:
  - `PKG_GITHUB_COM_TOKEN`, `GITHUB_TOKEN`, `GHCR_TOKEN`, `GHCR_USERNAME`, `NODE_AUTH_TOKEN`.
- Forwards ports:
  - `8443` – Argo CD HTTPS.
  - `10350` – Tilt UI.
- Sets `postCreateCommand` to:
  - `bash .devcontainer/postCreate.sh`

The DevContainer image is intentionally thin; the heavy lifting happens in `postCreate.sh` and `bootstrap-v2.sh`.

### 2.2 postCreate.sh

File: `/.devcontainer/postCreate.sh`

Responsibilities:

1. **Logging**:
   - All output is tee’d to `${DEVCONTAINER_BOOTSTRAP_LOG:-/home/vscode/.devcontainer-bootstrap.log}`.
   - Every line is prefixed with `[devcontainer-postCreate]`.

2. **Buf credentials**:
   - Sources `scripts/lib/buf_netrc.sh` and calls `buf_setup_netrc`.
   - Fails fast if `BUF_TOKEN` is not set in `.env`.

3. **Toolchain installation** (if missing):
   - `k3d` via upstream installer.
   - `helm` pinned to `v3.19.2` (Helm 4 is explicitly rejected due to SSA conflicts with Argo).
   - `tilt` via upstream installer.
   - `argocd` CLI from the latest Argo release.
   - `bicep` CLI based on architecture.
   - `yq` CLI.
   - Python module `pyyaml` via `pip` (user install).

4. **argocd wrapper**:
   - Writes `/home/vscode/.local/bin/argocd`:
     - Ensures the kube context is the dev cluster (`k3d-ameide` by default).
     - Reads `argocd-initial-admin-secret` from `argocd` namespace.
     - Logs into Argo CD via `argocd login` (using `--insecure` and `--grpc-web` as configured).
   - Ensures `$HOME/.local/bin` is on `PATH`.

5. **Bootstrap CLI invocation**:
   - Sets:
     - `BOOTSTRAP_CLI = tools/bootstrap/bootstrap-v2.sh`
     - `CONFIG_PATH = infra/environments/dev/bootstrap.yaml`
   - Builds `args`:
     - `--config infra/environments/dev/bootstrap.yaml`
     - `--reset-k3d`
     - `--lint-images`
     - `--lint-helm`
     - `--validate-waves`
     - `--install-argo`
     - `--apply-repo-creds`
     - `--apply-projects-ops`
     - `--apply-root-apps`
     - `--apply-dockerhub-secret`
     - `--wait-tiers`
     - `--show-admin-password`
     - `--build-images`
     - `--port-forward`
   - Logs and executes: `tools/bootstrap/bootstrap-v2.sh` with those options.

6. **Post-bootstrap dependency setup**:
   - `.devcontainer/bump-deps.sh`:
     - Bumps TS/Go/Python dependencies (go mod tidy, pnpm update, uv sync, etc.).
   - `.devcontainer/install-deps.sh`:
     - Installs workspace dependencies (`pnpm install`, `go mod download`, `uv sync`).

If **any** step fails, `postCreate.sh` logs a failure message and exits non-zero; the DevContainer creation shows this in the VS Code UI.

---

## 3. Bootstrap CLI: bootstrap-v2.sh

File: `/tools/bootstrap/bootstrap-v2.sh`

This script orchestrates all bootstrap steps. It is designed as a **single entrypoint** with:

- Strict mode: `set -Eeuo pipefail`.
- Rich error handling: `on_error` + `on_exit` traps.
- Pluggable behavior via flags and a YAML config file.
- A modular library under `tools/bootstrap/lib/*.sh`.

### 3.1 Flags and configuration

Key flags:

- `--config <file>` – Bootstrap config (`infra/environments/dev/bootstrap.yaml`).
- `--kubeconfig <path>` – Override kubeconfig file.
- `--context <name>` – Override kube context.
- `--gitops-root`, `--gitops-env` – Override GitOps directory and environment.
- `--bicep-outputs` – Bicep deployment outputs JSON.
- `--env-file` – Env file for secrets (default `.env`).
- `--wait-tier` / `--wait-tiers` – RollingSync tiers to wait for.
- `--timeout` – Timeout for tier waits (default 900s).
- `--output text|json` – Human or machine-readable summary.
- `--lint-images`, `--lint-helm`, `--validate-waves`.
- `--install-argo`, `--apply-repo-creds`, `--apply-projects-ops`, `--apply-root-apps`, `--apply-dockerhub-secret`.
- `--wait-tiers` – Wait for `<tier>-<env>` ApplicationSets.
- `--show-admin-password` – Show argocd admin password.
- `--port-forward` – Start `kubectl port-forward` for Argo.
- `--reset-k3d` – Recreate k3d cluster/registry before bootstrapping.
- `--build-images` – Build and push dev images.
- `--skip-wave-validation`, `--skip-helm-lint`.
- `--dry-run` – **Plan only; no cluster/k3d/helm/install/secret changes.**
- `--verbose`, `--debug` – More verbose logging and `set -x` in debug.

Config file: `infra/environments/dev/bootstrap.yaml`

- Controls:
  - Cluster: `context`, `type`, `name`, `registry` (`name`, `port`).
  - GitOps: `root`, `env`, `argoInstallManifest`, `rootApplication`.
  - Secrets:
    - Repo: manifest path, username env, token env.
    - DockerHub: strategy, secretName, registry, namespaces, username env, token env.
  - Bicep outputs: path to JSON.

`load_config()` uses `config_get`/`config_get_path` (from `common.sh`) to read values from YAML and fall back to sensible defaults.

### 3.2 Global behavior

- `on_error`:
  - Captures the failing command, function, and line number:
    - `[func:line] 'command' failed with exit N`.
- `on_exit`:
  - Computes total duration in seconds and `Xm Ys`.
  - If `OUTPUT_MODE=json`:
    - Prints a JSON summary:
      - `{status, cluster, argoVersion, durationSeconds, error}`.
  - Otherwise:
    - Logs:
      - `bootstrap complete for context … (argo=…, duration=Xm Ys (Ns))` or `bootstrap FAILED: <message>`.
    - Logs a **phase duration summary** if any phases executed:
      - `phase duration summary:`
      - `  - lint: 1m 14s (74s)`
      - `  - install-argo: 1m 2s (62s)`
      - etc.
    - If **not dry-run** and `--show-admin-password` was used:
      - Attempts to read `argocd-initial-admin-secret` one more time.
      - Logs a final **Argo CD admin access** block:
        - `URL: https://localhost:8443` (if port-forward is enabled).
        - `Username: admin`
        - `Password: <current admin password>` or an error if unreadable.

- Locking:
  - Uses `/tmp/bootstrap-v2.lock` to prevent concurrent runs.
  - `acquire_lock` is called at the top of `main()`.
  - `release_lock` is called from `on_exit`.

### 3.3 Phases and timing

Phases are managed by `start_phase(name)` / `end_phase()` in `common.sh`:

- On phase start:
  - Logs: `phase 'name' started`.
  - Records the start time and the order.
- On phase end:
  - Logs: `phase 'name' completed in Xm Ys (Ns)` or `... in Ns` for short phases.
  - Stores the duration in `PHASE_DURATIONS[name]`.

Typical phases for a full (non-dry-run) dev bootstrap:

1. `lint` – Image values, Helm charts, wave metadata.
2. `cluster-setup` – k3d reset, Bicep outputs, cluster readiness.
3. `build-images` (optional).
4. `install-argo` – Helm or manifest-based Argo CD install.
5. `apply-repo-creds`.
6. `show-admin-password`.
7. `apply-projects-ops`.
8. `apply-root-apps`.
9. `apply-dockerhub-secret`.
10. `wait-tiers`.
11. `port-forward`.
12. `mock-cert-export`, `mock-tilt` (if enabled).

In **dry-run**:

- Only `lint` and `cluster-steps (dry-run)` phases run.
- All cluster/helm/Argo modifications are logged as `[dry-run] would ...` and are not executed.

---

## 4. Library modules

### 4.1 common.sh

File: `tools/bootstrap/lib/common.sh`

Provides:

- Guards (`BOOTSTRAP_COMMON_LOADED`).
- Logging:
  - `log`, `verbose`, `debug`.
  - Optional timestamps via `BOOTSTRAP_LOG_TIMESTAMP`.
- Command checks:
  - `require_cmd` – returns non-zero if a command is missing.
  - `require_cmd_version` – checks minimum version and warns/fails accordingly.
- Path utilities:
  - `resolve_path`, `parse_comma_list`.
- Env and config:
  - `load_env_file` – sources `.env`-style file with `set -a`.
  - `load_bicep_outputs` – loads Bicep outputs JSON into a global `BICEP_OUTPUTS` map.
  - `config_get` / `config_get_path` – wrappers around `yq` for config values and repo-relative paths.
- Locking:
  - `acquire_lock`, `release_lock`.
- Phase timing:
  - `start_phase`, `end_phase`, storing durations.
- Parallel utilities:
  - `parallel_start`, `parallel_wait_all`, `parallel_reset`.
- Retry helper:
  - `retry max_attempts delay cmd...`.

### 4.2 cluster.sh

File: `tools/bootstrap/lib/cluster.sh`

Responsibilities:

- `check_kubectl_version`:
  - Probes `kubectl version --short` or falls back to full `kubectl version`.
  - Tracks if `--short` is supported via `CLUSTER_SHORT_SUPPORTED`.
- `wait_for_cluster_ready`:
  - Polls `/readyz` and `kubectl version` until both succeed or timeout (`CLUSTER_READY_TIMEOUT`, default 300s).
  - Aborts with a descriptive error on timeout.

### 4.3 k3d.sh

File: `tools/bootstrap/lib/k3d.sh`

Responsibilities:

- Cleanup helpers:
  - `cleanup_stale_docker_containers filter`.
  - `cleanup_stale_docker_network name`.
- Registry management:
  - `delete_k3d_registry_if_exists name`.
  - `create_k3d_registry name port`.
- Cluster management:
  - `delete_k3d_cluster_if_exists name`.
  - `create_k3d_cluster name registry_name registry_port`.
  - All critical calls (cluster/registry create/delete) now check exit status and log warnings on failures.
- Reset flow:
  - `maybe_reset_k3d_environment`:
    - Validates `CLUSTER_TYPE` / `TARGET_CONTEXT` are `k3d`.
    - Requires `k3d` and `docker`.
    - Deletes existing cluster/registry, then recreates them.
    - Switches kube context to `k3d-<cluster>`.
    - Waits for cluster ready via `wait_for_cluster_ready`.
- Container network access:
  - `running_inside_container` detection.
  - `ensure_k3d_container_network_access`:
    - Ensures the DevContainer is attached to the k3d Docker network (`k3d-<cluster>`).
    - Optionally updates kubeconfig cluster URL to use `k3d-<cluster>-serverlb:6443`.
  - `maybe_configure_k3d_container_access` – convenience wrapper.

### 4.4 argocd.sh

File: `tools/bootstrap/lib/argocd.sh`

Responsibilities:

- Namespace:
  - `ensure_argocd_namespace`:
    - Creates `ARGOCD_NAMESPACE` (default `argocd`) if missing.
    - Handles races where another process creates it.
- Installation:
  - `install_argo`:
    - If `ARGO_MANIFEST_PATH` is set and exists:
      - Creates namespace, applies manifest into that namespace.
      - Records version as `manifest:<filename>`.
      - Invokes `wait_for_argo_rollouts` to wait for core deployments/statefulset in parallel.
    - Else:
      - Uses `helm upgrade --install` with:
        - `--namespace "${ARGOCD_NAMESPACE}" --create-namespace`.
        - Base and env values from GitOps tree.
        - `--wait --atomic` for rollback on failure.
      - Records chart version.
    - For manifest installs, rollout failures are logged but do not hard-fail bootstrap.
- Rollout waits:
  - `wait_for_argo_rollouts`:
    - Waits for:
      - `deployment/argocd-repo-server`
      - `deployment/argocd-application-controller`
      - `deployment/argocd-server`
      - `deployment/argocd-applicationset-controller`
      - `statefulset/argocd-redis`
    - Runs all waits in parallel and logs readiness or warnings.
- Repo credentials:
  - `apply_repo_credentials`:
    - Uses `REPO_CREDENTIALS_MANIFEST` (typically `gitops/ameide-gitops/environments/dev/argocd/repos/ameide-gitops.yaml`).
    - Resolves username from `REPO_USERNAME_ENV` or `GITHUB_USERNAME` or default `git`.
    - Resolves password from `REPO_PASSWORD_ENV` or `GITHUB_TOKEN`.
    - Enforces that a password is present; fails with a clear error otherwise.
    - Does a targeted `envsubst` of `${ARGOCD_REPO_USERNAME} ${ARGOCD_REPO_PASSWORD}` only.
    - Applies the resulting Secret via `kubectl_argocd`.
    - Waits for `argocd-repo-server` rollout (best effort).
- Admin password:
  - `show_argocd_admin_password`:
    - Waits for `argocd-server` pods to be `Ready`.
    - Polls `argocd-initial-admin-secret` up to 150 attempts (2s interval).
    - If `ARGOCD_PASSWORD_FILE` is set:
      - Writes the password there with mode 600 and logs the path.
    - Else, if stdout is a TTY:
      - Prints a framed block with the password.
    - Else:
      - Logs a warning and suppresses password output (so CI logs are not contaminated).

### 4.5 gitops.sh

File: `tools/bootstrap/lib/gitops.sh`

Responsibilities:

- Small wrappers:
  - `apply_directory_if_exists path` – recursive apply for a directory or single-file apply for a file, with logging.
  - `apply_yaml_file` – logs and applies a single manifest.
  - `has_kustomization` – detects kustomization in a directory.
  - `apply_kustomization_dir` – applies a kustomization directory.
- Projects and ops:
  - `apply_projects_and_ops`:
    - Applies all `project-*.yaml` manifests under `${GITOPS_ARGO_DIR}`.
    - Applies the `ops` directory (if present) recursively.
- Root applications:
  - `apply_root_applications`:
    - If `ROOT_APPLICATION_PATH` is a single file: apply it.
    - If a directory:
      - If it is a kustomization: apply via `-k`.
      - Else:
        - Apply all `*.yaml` / `*.yml` files.
        - Apply any subdirectories that contain kustomizations.
      - Logs if nothing was applied.

### 4.6 secrets.sh

File: `tools/bootstrap/lib/secrets.sh`

Responsibilities:

- `apply_dockerhub_secret`:
  - Strategy `env`:
    - Uses `DOCKERHUB_USERNAME_ENV` / `DOCKERHUB_TOKEN_ENV`.
    - Defaults namespaces to `["ameide"]` if none are configured.
    - For each namespace:
      - Ensures the namespace exists (creates if needed).
      - Creates/updates the Docker registry secret via `--dry-run=client | kubectl apply`.
    - Performs the work in **parallel** for all namespaces and logs per-namespace success/failure.
  - Strategies `skip`, `managed-identity`:
    - Logs that DockerHub secrets are skipped.
  - Unknown strategies:
    - Logs an error and returns non-zero.

### 4.7 status.sh

File: `tools/bootstrap/lib/status.sh`

Responsibilities:

- Waiting for ApplicationSet tiers:
  - `wait_for_single_tier appset_name`:
    - Waits for `.status.conditions` to report `ResourcesUpToDate=True`.
    - Then fetches all associated `Application` objects, checks:
      - `status.health.status == Healthy`
      - `status.sync.status == Synced`
    - Logs warnings for any app not Healthy/Synced and fails the tier.
  - `wait_for_tiers`:
    - Runs `wait_for_single_tier` in **parallel** for all tiers in `WAIT_TIERS`.
    - Logs per-tier success/failure and returns non-zero if any tier fails.

### 4.8 portforward.sh

File: `tools/bootstrap/lib/portforward.sh`

Responsibilities:

- Readiness:
  - `ensure_argocd_server_ready`:
    - Waits for `argocd-server` pods in `ARGOCD_NAMESPACE` to be Ready.
    - Uses `ARGOCD_SERVER_WAIT_TIMEOUT` (default 120s).
- Port forwards:
  - `start_port_forward_instance name local_port remote_port address pid_file log_file`:
    - Stops any existing forward tracked by `pid_file`.
    - Attempts up to three starts:
      - `kubectl port-forward svc/${ARGOCD_SERVER_SERVICE_NAME} --address ${address} ${local_port}:${remote_port}`.
    - Logs PID and log file location on success; logs a failure message after 3 attempts.
  - `start_argocd_port_forwards`:
    - For `dev`, when `ARGOCD_PORT_FORWARD_ADDRESS` is unset:
      - Uses `0.0.0.0:8443 -> svc/argocd-server:443`.
    - Otherwise:
      - Uses `address:8080 -> 443`.

---

## 5. Dry-run semantics

When `--dry-run` is specified:

- Lint phase:
  - `lint-image-values` **does run** (local, safe).
  - `lint-helm` is **skipped**:
    - Logs: `[dry-run] skipping helm lint (would add/update Helm repos and download chart dependencies)`.
  - `validate-waves` **does run** (local `python3` script).
- Cluster phase:
  - No `kubectl`, `k3d`, `helm`, secrets, or port-forward invocations run.
  - Instead, `cluster-steps (dry-run)` logs everything that would happen:
    - `would reset k3d environment`, `would build/push dev images`, `would install/upgrade Argo CD`, etc.
- `on_exit` still:
  - Logs total duration and phase summary.
  - **Skips** the final Argo admin access block.

The invariant: in dry-run, **no external state** is changed (no Helm repo updates, no cluster changes, no secrets applied).

---

## 6. Argo CD admin access summary

On a successful, non-dry-run bootstrap with `--show-admin-password`:

- `show_argocd_admin_password` logs the password during bootstrap (subject to TTY rules).
- `on_exit` repeats a summarized view:

```text
[bootstrap-v2] Argo CD admin access:
[bootstrap-v2]   URL: https://localhost:8443
[bootstrap-v2]   Username: admin
[bootstrap-v2]   Password: <current_password_or_error>
```

Implementation details:

- If `--port-forward` is used:
  - URL is `https://localhost:8443`.
- Otherwise:
  - URL is a placeholder, assuming the in-cluster service endpoint.
- Password retrieval:
  - `kubectl_argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`.

This makes it easy for a developer to copy-paste the Argo CD URL and credentials from the end of the bootstrap log.

---

## 7. Future improvements (notes)

Potential follow-ups:

- Extract a dedicated `parallel.sh` or "worker pool" module for bounded concurrency across lint, tier waits, and secret creation.
- Add JSON-structured phase timing output for `--output json` to feed CI dashboards.
- Provide a `--no-lint` flag for faster iterative bootstraps in trusted local environments.
- Extend dry-run to optionally simulate tier readiness/health by inspecting current cluster state without requiring Argo to be installed.
- Document and standardize environment variables controlling namespace, port-forward address, and password output location for multi-dev setups (e.g., `ARGOCD_NAMESPACE`, `ARGOCD_SERVER_SERVICE_NAME`, `ARGOCD_PASSWORD_FILE`).

---

## 8. Integration with other backlogs

### Telepresence mode ([432](432-devcontainer-modes-offline-online.md), [434](434-unified-environment-naming.md))

> **Update (2026-01):** The Tilt-based “devcontainer modes” and `tools/dev/telepresence.sh` helper script are deprecated/removed.
> Telepresence is now driven by the Ameide CLI, and the dev UX is intentionally “no flags, no modes”.

Current contract:
- Bootstrap kube contexts once via `tools/dev/bootstrap-contexts.sh`.
- Use `ameide dev inner-loop verify` to validate Telepresence + header-filtered intercept routing.
- Use `ameide dev inner-loop up/down` for cluster-only UI hot reload.
- Use `ameide dev inner-loop-test` for strict phase gating with JUnit evidence (Phase 0 → 3).

### Domain rename ([434](434-unified-environment-naming.md))

The domain rename from `tilt.ameide.io` → `local.ameide.io` is complete. Gateway listeners and cert-manager SANs reference `*.local.ameide.io`; host-level dnsmasq helpers were removed now that we rely on the remote AKS cluster + Telepresence. See [367-bootstrap-v2.md](367-bootstrap-v2.md#38-domain-naming) for the authoritative list.

### Remote AKS bootstrap ([367](367-bootstrap-v2.md))

Before bootstrap-v2 can target AKS clusters, the remote GitOps structure must be migrated from the legacy path (`ameide.git/infra/kubernetes/gitops/`) to `ameideio/ameide-gitops`. See [434-unified-environment-naming.md](434-unified-environment-naming.md#migration-needed) for migration tasks.
