# 491 – DevContainer Auto-Contexts (kubectl, ArgoCD, Telepresence)

**Status:** Draft  
**Owner:** Platform  

## Goal

On every DevContainer start, we want kubectl, Telepresence, and the `argocd` CLI to be immediately usable without manual context juggling.

This covers two “developer targets”:
- **Shared AKS** (`ameide`) for remote-first development (Telepresence workflows).
- **Local k3d** (`k3d-ameide`) for fully local GitOps convergence (see `*.local.ameide.io` patterns in [109-local-development-domain-access.md](000-200/109-local-development-domain-access.md)).

## Contexts to pre-configure

We have **one shared AKS cluster (`ameide`)** with separate namespaces per environment (see [434-unified-environment-naming.md](434-unified-environment-naming.md#environment-matrix)). Every tool should surface predictable contexts that only differ by default namespace or endpoint.

### Shared AKS (`ameide`) contexts

| Tool          | Context Name       | Purpose / Namespace                        |
|---------------|--------------------|--------------------------------------------|
| `kubectl`     | `ameide-dev`       | Cluster `ameide`, namespace `ameide-dev`   |
| `kubectl`     | `ameide-staging`   | Cluster `ameide`, namespace `ameide-staging` |
| `kubectl`     | `ameide-prod`      | Cluster `ameide`, namespace `ameide-prod`  |
| `argocd` CLI  | `argocd.cluster`   | Remote Gateway (`argocd.ameide.io`)        |
| `argocd` CLI  | `argocd.dev`       | Optional env gateway (`argocd.dev.ameide.io`) |
| `argocd` CLI  | `argocd.local`     | Port-forward fallback (`127.0.0.1:8443`)   |
| Telepresence  | `ameide-dev` (default) | Intercepts in `ameide-dev` (override for staging/prod) |

### Local k3d (`k3d-ameide`) contexts

| Tool          | Context Name       | Purpose / Namespace                        |
|---------------|--------------------|--------------------------------------------|
| `kubectl`     | `k3d-ameide`       | Local k3d cluster, namespace `ameide-local` |
| `argocd` CLI  | `argocd.local`     | Local Gateway (`https://argocd.local.ameide.io`) |
| `argocd` CLI  | `argocd.local-pf`  | Port-forward fallback (`127.0.0.1:8443`)   |
| Telepresence  | (n/a)              | Telepresence is for shared clusters         |

> **Note:** This backlog covers the **developer bootstrap** that lives in `ameideio/ameide` (`tools/dev/bootstrap-contexts.sh`). The cluster-wide GitOps bootstrap continues to live in `ameideio/ameide-gitops: bootstrap/bootstrap.sh` per [435-remote-first-development.md](435-remote-first-development.md); that script installs Argo CD and applies ApplicationSets, whereas this one just ensures DevContainer sessions have working contexts against the already-bootstrapped AKS environments.

## Automation plan

### 1. Kubectl contexts

`.devcontainer/postCreate.sh` shells into `tools/dev/bootstrap-contexts.sh`, which:

1. Verifies `az login` is active (opens the device code flow if not) and runs `az aks get-credentials`.
2. Converts kubeconfig entries via `kubelogin -l azurecli` so token refresh works automatically.
3. Rewrites the target kube context (default `ameide-dev`, override `AMEIDE_AKS_CONTEXT`) to point at the shared AKS control plane and default namespace, then selects it.

Re-run manually with `bash tools/dev/bootstrap-contexts.sh` if you blow away `~/.kube/config`.

### 2. Telepresence defaults

`tools/dev/telepresence.sh` remains the entry point, but the context defaults now come from `~/.config/ameide/context.env`, which `tools/dev/bootstrap-contexts.sh` writes and `.bashrc` sources automatically. The helper reads `TELEPRESENCE_CONTEXT`/`TELEPRESENCE_NAMESPACE`, so overriding them before calling `./tools/dev/telepresence.sh connect` switches environments with no further setup. The helper now refuses to run on Linux when Telepresence prerequisites (`iptables` for DNS/routing) are missing, and the DevContainer image + `postCreate.sh` ensure both `iptables` and `sshfs` are installed before bootstrap runs so remote env mounts/DNS work immediately. The lifecycle of this script (connect, intercept, verification, logging) is tracked in [492-telepresence.md](492-telepresence.md).

### 3. Argo CD CLI contexts

`tools/dev/bootstrap-contexts.sh` also drives ArgoCD end-to-end:
- Shared AKS: log into the remote gateway (`argocd.ameide.io`, and optionally `argocd.dev.ameide.io` if it exists).
- Local k3d: prefer the local Gateway (`https://argocd.local.ameide.io`) when running a local cluster, and keep a port-forward fallback on `127.0.0.1:8443`.

The bootstrap decodes `argocd-initial-admin-secret`, stores the password under `$HOME/.config/argocd/admin-password`, and ensures `argocd app list` succeeds without manual steps.

### 4. Cleanup on exit

Add a trap to terminate the port-forward when the shell exits:

```bash
cleanup_argocd_pf() {
  if [[ -f /tmp/argocd-portforward.pid ]]; then
    kill "$(cat /tmp/argocd-portforward.pid)" >/dev/null 2>&1 || true
    rm /tmp/argocd-portforward.pid
  fi
}
trap cleanup_argocd_pf EXIT
```

## Verification checklist

1. Open the DevContainer and confirm:
   - Shared AKS: `kubectl config current-context` → `ameide-dev` (or the selected env).
   - Local k3d: `kubectl config current-context` → `k3d-ameide`.
   - Telepresence (AKS only): `telepresence status` shows the prepared context.
   - `argocd context` includes the right endpoint(s) for your target (`argocd.ameide.io` for AKS, `argocd.local.ameide.io` for local, and `127.0.0.1:8443` as fallback).

2. Run `argocd app list` without additional login steps.

3. Intercept a service via `./tools/dev/telepresence.sh intercept www-ameide-platform 3000:3000` and verify traffic reaches the local process.

4. Kill the port-forward (`pkill -f 'argocd-server 8443:443'`) and ensure the cleanup trap prevents orphaned processes.

## Open questions

- Do we still need the local k3d context, or can we drop it entirely?
- Should we mint read-only ArgoCD accounts per developer instead of reusing the admin secret?

Once this backlog is implemented, onboarding becomes `telepresence connect`, `argocd app list`, and `kubectl get pods`—no manual context fiddling required.

## Current status (Dec 9 2025)

- Added `tools/dev/bootstrap-contexts.sh` which:
  1. Runs `az aks get-credentials` and rewrites the desired context (`AMEIDE_AKS_CONTEXT`, default `ameide-dev`) to point at the real `ameide` cluster/user, with namespace `ameide-dev`.
  2. Drops Telepresence defaults into `~/.config/ameide/context.env` and sources them from `.bashrc`.
  3. Fetches `argocd-initial-admin-secret`, spins up a `kubectl port-forward` on `127.0.0.1:8443`, and logs the CLI into both the local endpoint and (best-effort) the public gateway.
  4. Prints `kubectl config current-context` + `argocd context` so failures are obvious.
- Wired `.devcontainer/postCreate.sh` to invoke `tools/dev/bootstrap-contexts.sh`, so every DevContainer start converges kubectl + Telepresence + ArgoCD contexts the same way.
- `.devcontainer/Dockerfile` now bakes `iptables`/`sshfs` into the base image, and `postCreate.sh` re-validates them to prevent Telepresence from failing with “no active session” or disabled volume mounts on fresh containers.
- Verified by running `bash tools/dev/bootstrap-contexts.sh`:

```
[bootstrap-contexts] Refreshing AKS credentials for cluster ameide
[bootstrap-contexts] Starting ArgoCD port-forward on 8443
[bootstrap-contexts] Ensuring ArgoCD CLI context for 127.0.0.1:8443
[bootstrap-contexts] Contexts configured:
ameide-dev
CURRENT  NAME              SERVER
*        127.0.0.1:8443    127.0.0.1:8443
         argocd.ameide.io  argocd.ameide.io
[bootstrap-contexts] Done. Port-forward PID: 37117
```

`kubectl get ns argocd` and `argocd account get-user-info` both succeeded immediately after running the script.

### Docker access inside the DevContainer

Several backlogs (480/481) now assume you can run `docker build` for services before pushing. The DevContainer ships with the Docker CLI, but the default `vscode` user is **not** in the `docker` group, so commands fail with:

```
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock
```

To enable Docker for the non-root user, run the following once inside the DevContainer:

```bash
sudo usermod -aG docker "$USER"
sudo chown root:docker /var/run/docker.sock
sudo chmod 660 /var/run/docker.sock
newgrp docker <<'EOF'
docker version
EOF
```

The `newgrp` step refreshes group membership without restarting the entire container; it also validates connectivity. If you prefer not to adjust group membership, you can always fall back to `sudo docker …`, but the backlog automation (including Tilt/Terraform scripts) assumes passwordless access, so updating the group is recommended. Document this prerequisite in service-specific READMEs whenever local `docker build` is required.
