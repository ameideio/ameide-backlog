Below is a vendor‑docs‑aligned way to wire **Dev Containers** (VS Code / Dev Container spec), **k3d** (local k8s), **Argo CD** (GitOps), and **Tilt** (inner‑loop dev) together for your two repos. _Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable.

> **Bootstrap location:** The CLI previously referenced here as `tools/bootstrap/bootstrap-v2.sh` now resides in the `ameideio/ameide-gitops` repository (`bootstrap/bootstrap.sh`). Developer bootstrap inside `ameideio/ameide` is limited to `.devcontainer/postCreate.sh` plus `tools/dev/bootstrap-contexts.sh`, which connect the DevContainer to the shared AKS cluster per [435-remote-first-development.md](435-remote-first-development.md). Treat the k3d/bootstrap instructions below as historical context unless you are working inside the GitOps repo.

* **`ameide`** – the main app source (we’ll put the Dev Container and `Tiltfile` here).
* **`ameide-gitops`** – the GitOps repo Argo CD will watch.

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable. _Status checkpoints:_ `.devcontainer/postCreate.sh` (which historically shelled into `tools/bootstrap/bootstrap-v2.sh` before the bootstrap CLI moved to `ameide-gitops/bootstrap/bootstrap.sh`) loads `.env`, installs k3d/Tilt/argocd CLI, applies Argo CD manifests, registers repo credentials, runs `kubectl port-forward`, and logs everything to `/home/vscode/.devcontainer-bootstrap.log`.

---

## 0) What you’ll have when done

* Open `ameide` in a Dev Container that already has Docker‑CLI, `kubectl`, `k3d`, `tilt`, and `argocd` CLI installed.
* Create a **k3d** cluster with a **local registry** that Tilt auto‑detects and uses for fast image pushes. ([k3d.io][1])
* Install **Argo CD** in the cluster and point it at **`ameide-gitops`** using a declarative `Application` manifest. ([Argo CD][2])
* Use **Tilt** for the inner loop (build, push to the local registry, apply manifests, live‑reload, port‑forward). ([Tilt][3])

You can keep Argo CD managing the “baseline/platform + GitOps app definitions” while letting Tilt own rapid edits to the `ameide` workload to avoid fights (details in §5).

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable. _Status checkpoints:_ Verified 2025-11-12 via the bootstrap-v2 log (`/home/vscode/.devcontainer-bootstrap.log`, emitted by `.devcontainer/postCreate.sh`) showing registry/cluster ready, Argo rollout, and repo secret applied. Argo CD accessible through https://localhost:8080 once the background port-forward starts.

---

## 1) Dev Container in `ameide`

The Dev Container spec defines a `devcontainer.json` that tools (VS Code Dev Containers or the `devcontainer` CLI) use to create a reproducible dev environment. We’ll forward the **host Docker socket** into the container (aka “docker‑outside‑of‑docker”) and add common k8s tooling. ([Dev Containers][4]) _Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable.

> **Why the Docker socket?** k3d runs Kubernetes nodes *as Docker containers*, so the devcontainer must be able to talk to your host’s Docker daemon. The official **Docker outside of Docker** feature is `ghcr.io/devcontainers/features/docker-outside-of-docker:1`. ([Containers.dev][5]) _Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable.

Create **`.devcontainer/devcontainer.json`** in `ameide`:

```jsonc
{
  "name": "ameide",
  "build": {
    "dockerfile": "Dockerfile",
    "context": ".."
  },
  "features": {
    // forward host Docker socket + install docker CLI
    "ghcr.io/devcontainers/features/docker-outside-of-docker:1": {},
    // optional: kubectl + helm convenience
    "ghcr.io/devcontainers/features/kubectl-helm-minikube:1": {}
  },
  // forward the Docker socket (cross-orchestrator spec property)
  "mounts": [
    "source=/var/run/docker.sock,target=/var/run/docker.sock,type=bind"
  ],
  // install k3d, tilt, argocd CLI once container is created
  "postCreateCommand": "bash .devcontainer/postCreate.sh",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-kubernetes-tools.vscode-kubernetes-tools",
        "redhat.vscode-yaml"
      ]
    }
  }
}
```

> `mounts`, `features`, and lifecycle commands are first‑class Dev Container spec properties. ([Dev Containers][6])

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable. _Status checkpoints:_ `.devcontainer/postCreate.sh` used to call `k3d registry create` / `k3d cluster create` indirectly via `tools/bootstrap/bootstrap-v2.sh`; those steps now live in the GitOps bootstrap (`ameide-gitops/bootstrap/bootstrap.sh`). Inspect `/home/vscode/.devcontainer-bootstrap.log` for the latest legacy run. Tilt builds run against this cluster (`allow_k8s_contexts('k3d-ameide')` in the new `Tiltfile`).

**`.devcontainer/Dockerfile`** (minimal base that uses the spec + apt tools as needed):

```dockerfile
FROM mcr.microsoft.com/devcontainers/base:ubuntu

# Any language/toolchain setup specific to ameide goes here (Node/Go/Java/etc.)
# kubectl/helm come from the feature; docker CLI comes from the feature
RUN apt-get update && apt-get install -y curl ca-certificates git && rm -rf /var/lib/apt/lists/*
```

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable. _Status checkpoints:_ Controller deployments + statefulset were rolled out inside the legacy bootstrap; repo secret + the root `ameide` Application / `project-ameide` manifests are applied from `gitops/ameide-gitops`. Argo CD UI + CLI were tested with the regenerated `argocd-initial-admin-secret`.

**`.devcontainer/postCreate.sh`** – install k3d, Tilt, and the Argo CD CLI using vendor installers:

```bash
set -euo pipefail

# --- k3d (official script) ---
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
# docs also describe package manager alternatives; the curl script is the official path.  # source: k3d docs

# --- Tilt (official script) ---
curl -fsSL https://raw.githubusercontent.com/tilt-dev/tilt/master/scripts/install.sh | bash

# --- Argo CD CLI (official docs: Linux via curl) ---
curl -sSL -o /tmp/argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
install -m 555 /tmp/argocd-linux-amd64 /usr/local/bin/argocd
rm /tmp/argocd-linux-amd64

echo "postCreate done."
```

Vendor references: Dev Containers (“create a dev container”), Dev Container JSON reference, and the Docker‑outside‑of‑docker Feature; Argo CD CLI install steps shown for Linux/macOS/WSL. ([Visual Studio Code][7])

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable. _Status checkpoints:_ As of 2025-11-12 the repo secret (`repo-ameide-gitops`) and `Application`/`AppProject` manifests reside in `gitops/ameide-gitops/environments/dev/argocd/` and are applied automatically during bootstrap; `argocd app sync ameide` now succeeds once Argo CD √ repo connection is established.

---

## 2) Create a k3d cluster with a local registry

Tilt works best with a **local registry**. k3d has built‑in support and Tilt will **auto‑detect** it for fast pushes/pulls. ([k3d.io][1])

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable.

Inside your Dev Container terminal:

```bash
# 1) make a predictable registry name + port (easy to script & share)
k3d registry create ameide.localhost --port 5001

# 2) create the cluster and tell it to use the registry created above
#    Note: k3d adds "k3d-" prefix to resource names when referencing them.
k3d cluster create ameide \
  --agents 1 \
  --registry-use k3d-ameide.localhost:5001 \
  --kubeconfig-switch-context
```

Those two flows (create registry + cluster using it) are straight from the k3d “Using a local registry” guide (notice the `k3d-` prefix in `--registry-use`). ([k3d.io][1])

Verify:

```bash
kubectl get nodes
```

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable.

---

## 3) Install Argo CD in‑cluster

From the **Argo CD Getting Started** page:

```bash
# create the argocd namespace and install the default manifest
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access the UI locally via port-forward:
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get the initial admin password and log in with the CLI
argocd admin initial-password -n argocd | tr -d '\n' | pbcopy  # (mac) or save it
# then:
argocd login localhost:8080 --username admin --password <PASTE> --insecure
```

Those steps (namespace, apply `install.yaml`, port‑forwarding the server, and CLI login/password retrieval) are precisely what the official docs outline. ([Argo CD][2])

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable. _Status checkpoints:_ `tools/bootstrap/bootstrap-v2.sh` (invoked from `.devcontainer/postCreate.sh`) re-installs Argo CD, reapplies the GitOps manifests, and regenerates `argocd-initial-admin-secret` so the CLI/UI login flow can be tested each run.

---

## 4) Point Argo CD at `ameide-gitops` (declarative ApplicationSets)

Argo CD recommends modeling Applications *declaratively* and applying them via `kubectl`. For dev we use AppProjects plus an **ApplicationSet** that fans out to every first-party component under `environments/dev/components/apps/**/component.yaml`. ([Argo CD][8]) _Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable. _Status checkpoints:_ As of 2025-11-12 the declarative project/app/repo manifests live under `gitops/ameide-gitops/environments/dev/argocd/` and bootstrap applies them (with `clusterResourceWhitelist` allowing namespaces).

In **`ameide-gitops/environments/dev/argocd/`** you’ll find:

**`project-ameide.yaml`**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: ameide
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: Project for ameide apps
  sourceRepos:
    - '*'           # tighten later to just your repos
  destinations:
    - namespace: ameide
      server: https://kubernetes.default.svc
```

**`applications/ameide.yaml`** – root Application that applies AppProjects/repos and the RollingSync ApplicationSet (`ameide-dev`) which renders one Application per component from `environments/dev/components/**/component.yaml` using a shared template for values, charts, and sync options.

A sample component (e.g., `environments/dev/components/apps/core/graph/component.yaml`) supplies the fields consumed by the template:

```yaml
name: apps-graph
project: apps
namespace: ameide
tier: apps
wave: wave50
chart:
  repoURL: https://github.com/ameideio/ameide-gitops.git
  path: sources/charts/apps/graph
  version: main
```

Applying `project-ameide.yaml`, `apps/apps.yaml`, and the component files ensures Argo CD renders one Application per service (e.g., `apps-graph`, `apps-workflows`). `.devcontainer/postCreate.sh` historically exported `BOOTSTRAP_ARGO_APPSETS=foundation,data-services,platform,apps` when it called `tools/bootstrap/bootstrap-v2.sh`; the GitOps bootstrap now handles that sequencing automatically. Run `kubectl apply -n argocd -f environments/dev/argocd/apps/apps.yaml` manually if you tweak it outside bootstrap.

> **Repo credentials:** If `ameide-gitops` is private, add repo credentials via `argocd repo add` or the declarative “repositories” Secret as described in the docs. ([Argo CD][8])

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable.

---

## 5) Keep Tilt + Argo CD from “fighting”

Argo CD continuously reconciles **live cluster** state to what’s in **Git**; Tilt, by design, changes the **live resources** to speed your inner‑loop. To avoid tug‑of‑war:

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable.

### Recommended pattern: let Argo CD manage platform & app definitions; let Tilt manage the live `ameide` Deployment during dev

* Keep `ameide` **manual** (no `automated` sync policy) so Argo CD won’t revert your Tilt changes automatically. Manual mode still surfaces drift (the app shows `OutOfSync`); you decide when to click “Sync” to reconcile that drift. ([Argo CD][2])
* If you still need automated sync for other reasons, set `automated: { prune: false, selfHeal: false }` to reduce contention; `selfHeal` is the flag that re‑applies drift. ([Argo CD][9])
* In practice you pick one of these mutually exclusive shapes:

  ```yaml
  # Pure monitor-only (manual syncs)
  spec:
    syncPolicy: {}

  # Auto when Git changes, but never heals/prunes Tilt edits
  spec:
    syncPolicy:
      automated:
        prune: false
        selfHeal: false
  ```

* You can also annotate specific resources with **`Prune=false`** or configure **`ignoreDifferences`** for fields Tilt touches (e.g., container images), optionally with `RespectIgnoreDifferences=true` in `syncOptions`. (See “Sync Options” and “Diff Customization” docs.) ([Argo CD][10])

This is the same hybrid the Tilt team recommends—GitOps for the outer loop, local k8s for the inner loop. ([Tilt Blog][11]) Because the RollingSync ApplicationSet now stays applied by default, you’ll see the per-service Applications inside Argo; flip them to manual sync or `automated: { prune:false, selfHeal:false }` as shown above so they remain monitor-only while Tilt edits workloads directly. Tilt’s v4 layout is documented in [backlog/373](./373-argo-tilt-helm-north-start-v4.md).

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable.

---

## 6) Tilt in `ameide`

Create a **`Tiltfile`** in `ameide` to build your image and apply your dev manifests. Core APIs are `docker_build(...)` (build & push) and `k8s_yaml(...)` (apply manifests), then optional port‑forwards via `k8s_resource(...)`. ([Tilt][12])

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable.

**`Tiltfile`**

```python
# Allow running against your k3d context (Tilt already auto-allows k3d, but being explicit helps in teams)
allow_k8s_contexts('k3d-ameide')  # see docs

# Build the app image. The first argument must match the image name in your YAML.
# Tilt will auto-detect the k3d local registry and inject the fully-qualified image name for you.
docker_build('ameide', '.', live_update=[sync('.', '/app')])

# Register dev manifests; consider a kustomize overlay or helm chart for dev
k8s_yaml('k8s/dev')  # or k8s/dev.yaml, etc.

# Make it easy to access the app locally
k8s_resource('ameide', port_forwards=8080)
```

Notes:

* Tilt **auto‑detects** a k3d registry and rewrites image refs in your YAML to point at it, so you don’t need to hard‑code `localhost:5001/...` anywhere. ([Tilt][13])
* `allow_k8s_contexts(...)` keeps you from accidentally deploying to prod contexts. ([Tilt][14])

Run it:

```bash
tilt up
```

Tilt will watch file changes, rebuild the image, push to the local registry, patch your workload, and keep a live UI with logs & health. ([Tilt][3])

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable.

---

## 7) Example repo layout (one pragmatic option)

`ameide` (app repo)

```
.
├─ .devcontainer/
│  ├─ devcontainer.json
│  ├─ Dockerfile
│  └─ postCreate.sh
├─ Tiltfile
├─ Dockerfile           # image build for the app
└─ k8s/
   └─ dev/              # dev-only Kustomize/Helm or raw YAML for ameide
      ├─ namespace.yaml
      ├─ deployment.yaml
      └─ service.yaml
```

`ameide-gitops` (Argo CD repo)

```
.
└─ environments/
   └─ dev/
      ├─ argocd/
      │  ├─ project-ameide.yaml
      │  └─ applications/
      │     └─ ameide.yaml      # Root Application + RollingSync ApplicationSet
      └─ apps/
         └─ ameide/   # app manifests for GitOps (may be slim if Tilt owns dev)
            ├─ kustomization.yaml
            └─ base-or-overlay files...
```

The Argo CD docs show minimal Application/Project specs and the “App of Apps / Bootstrapping” pattern if you want a higher‑level “cluster baseline” app that in turn creates the others. ([Argo CD][8])

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable.

---

## 8) Daily workflow

1. **Open** `ameide` in VS Code → “Reopen in Container”. (Dev Containers are driven by `devcontainer.json`.) ([Visual Studio Code][7])

2. **Create cluster + registry** (first time only):

   ```bash
   k3d registry create ameide.localhost --port 5001
   k3d cluster create ameide --agents 1 --registry-use k3d-ameide.localhost:5001 --kubeconfig-switch-context
   ```

   ([k3d.io][1])

3. **Install Argo CD** (first time) & **apply apps** from `ameide-gitops`:

   ```bash
   kubectl create ns argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   kubectl apply -n argocd -f environments/dev/argocd/
   ```

   ([Argo CD][2])

4. **Develop with Tilt**:

   ```bash
   tilt up
   ```

   (Tilt will build/push to the k3d registry and update your k8s resources automatically.) ([Tilt][15])

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable.

---

## 9) Optional Argo CD tune‑ups for dev clusters

If you *do* want Argo CD running with your live changes and not marking things OutOfSync:

* Add **`ignoreDifferences`** in the `Application` to ignore fields like the Deployment’s `containers[].image` and enable `RespectIgnoreDifferences=true` in `syncOptions`. (Argo CD supports jsonPointers/JQ path expressions for per‑field ignores.) ([Argo CD][16])
* Use resource annotations like **`argocd.argoproj.io/sync-options: Prune=false`** on things you never want Argo CD to delete during dev. ([Argo CD][10])

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable.

---

## 10) Why these exact steps map to vendor docs

* **Dev Containers**: `devcontainer.json` powers VS Code Dev Containers / Dev Container CLI; `mounts`, `features` (e.g., Docker outside of Docker), and lifecycle commands are in the spec. ([Visual Studio Code][7])
* **k3d**: official docs show **using a local registry** and the `--registry-use k3d-<name>:<port>` flow. ([k3d.io][1])
* **Argo CD**: official **Getting Started** has the install manifest + port‑forward + CLI login & password retrieval; **Declarative Setup** shows canonical `Application`/`AppProject` specs; **Sync Options / Diff Customization** explain drift controls. ([Argo CD][2])
* **Tilt**: official docs cover install, the `Tiltfile` APIs (`docker_build`, `k8s_yaml`, `k8s_resource`/port‑forwards), and local cluster choices (k3d + registry integration). ([Tilt][3])

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable.

---

## 11) GitOps migration tracker (live log)

*2025-11-12 update:* we decomposed Helmfile **12-crds** into two GitOps components:

- `environments/dev/components/foundation/crds/gateway-api/` – vendors the upstream Gateway API standard install manifest.
- `environments/dev/components/foundation/crds/prometheus-operator/` – captures the rendered CRDs from `prometheus-community/prometheus-operator-crds` v17.0.0.

`foundation-dev` ApplicationSet (`environments/dev/argocd/apps/foundation.yaml`) now has RollingSync steps `wave00`, `wave05`, and **`wave10`**, so namespaces → smoke → CRDs flow in order once these commits reach `ameide-gitops` default branch. Bootstrap continues to `kubectl apply` the ApplicationSet during devcontainer creation; after pushing, verify with (see the “RollingSync verification & inventory” section in `gitops/ameide-gitops/README.md` for the full checklist):

```bash
kubectl -n argocd get applications foundation-crds-gateway foundation-crds-prometheus
```

_Next milestone:_ convert Helmfile **15-secrets-certificates** into `environments/dev/components/foundation/certificates/**` (wave15) so we can continue the bottom-up replacement with no Helmfile fallback.

*2025-11-12 update (pm)*: unblocked `ameide` Application health by:

- pulling images from the local k3d registry for dev; GHCR seeding was removed from bootstrap (Docker Hub pull secret remains for workloads that need it).
- pointing the dev Deployment at the local registry base image and pinning a keep-alive entrypoint so the placeholder container stays Running until the real `ameide` image is published.

*Layer 15 kickoff:* Helmfile **15-secrets-certificates** is now partially decomposed. We rendered the `jetstack/cert-manager` and `external-secrets/external-secrets` charts into `environments/dev/components/foundation/operators/{cert-manager,external-secrets}/` and added a `wave15` step to the foundation ApplicationSet. Vault + secret-store bundles remain in Helmfile and will be migrated next.

*2025-11-12 update (evening):* Finished decomposing Helmfile **15-secrets-certificates**. Vault PVC/server, ClusterSecretStores, all secret-store bundles, the integration-secrets job, and the secrets smoke test now live under `environments/dev/components/foundation/secrets/**` (waves 16–21). Helmfile `15-secrets-certificates.yaml` has been deleted, and ExternalSecrets are healthy across the board (Argo still reports them OutOfSync because the controller injects provider defaults; we annotated the resources with `argocd.argoproj.io/compare-options: IgnoreExtraneous` to keep the noise local).

---

## 12) Operational snapshot (2025‑11‑13)

* **Helm-only GitOps:** All third-party charts are unpacked under `gitops/ameide-gitops/sources/charts/third_party/**`, and every ApplicationSet declares a secondary `$values` source so Helm always loads `_shared/<tier>/<component>.yaml` plus the environment override. There are no runtime dependencies on remote Helm repos.
* **Redis operator health:** `kubectl -n redis-operator get pods` currently prints “No resources found”, so the operator and the `data-redis-failover` Application remain `OutOfSync`. Confirm/publish `quay.io/spotahome/redis-operator:v1.2.4` (or mirror it) before re-syncing the foundation/data waves.
* **Outstanding work before we can declare the cluster “green”:**
  - Publish or mirror every `docker.io/ameide/*:dev` image (agents, platform, workflows runtime, namespace guard) or add `imagePullSecrets` so pods stop hitting `ImagePullBackOff`.
  - Restore pgAdmin bootstrap/OAuth secrets (or keep OAuth disabled) so the Deployment leaves `CreateContainerConfigError`.
  - Delete failed `master-realm-*` jobs and re-sync `platform-keycloak-realm` to validate the updated realm JSON. Dex now depends on this: `dex.keycloak.clientSecret` must exist in `argocd-secret` (via `vault-secrets-argocd`), and the Keycloak `ameide` realm must be reachable or Dex logs “Realm does not exist”.
  - Prune Helmfile-era resources once Argo owns their namespaces so `OrphanedResourceWarning` reflects real drift.
  - Mark migrated Helmfile releases as `installed: false` (see `infra/kubernetes/helmfiles/22-control-plane.yaml` → `gateway`) so future `helmfile sync` runs stay render-only while Argo continues reconciling the live resources.

## Copy‑paste starter snippets you can adapt

### `k8s/dev/deployment.yaml` (referenced by Tilt)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ameide
  namespace: ameide
spec:
  replicas: 1
  selector:
    matchLabels: { app: ameide }
  template:
    metadata:
      labels: { app: ameide }
    spec:
      containers:
        - name: ameide
          image: ameide:dev   # Tilt matches this name and rewrites it for the k3d registry
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: ameide
  namespace: ameide
spec:
  selector: { app: ameide }
  ports:
    - port: 8080
      targetPort: 8080
```

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable.

### Optional Argo CD “ignore image” example (put into `app-ameide.yaml` if you want Argo CD to ignore Tilt’s image swaps)

```yaml
spec:
  syncPolicy:
    syncOptions:
      - RespectIgnoreDifferences=true
  ignoreDifferences:
    - group: apps
      kind: Deployment
      name: ameide
      namespace: ameide
      jqPathExpressions:
        - .spec.template.spec.containers[].image
```

(Uses `ignoreDifferences` + “RespectIgnoreDifferences” sync option). ([Argo CD][16])

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable.

---

### That’s it

Open `ameide` in the container, bring up the k3d+registry, install Argo CD, apply the `ameide` Application, and run `tilt up`. You’ll get a tight inner loop with Tilt while keeping Argo CD’s GitOps structure in place for the outer loop—following the vendor docs for each tool. ([Argo CD][2])

If you want, I can tailor the exact YAML/Kustomize overlay paths to your current `ameide-gitops` structure—just share its folders or a gist.

_Current status:_ the repo now follows this vendor workflow. _Migration plan:_ not applicable.

[1]: https://k3d.io/v5.6.3/usage/registries/ "Using Image Registries - k3d"
[2]: https://argo-cd.readthedocs.io/en/stable/getting_started/ "Getting Started - Argo CD - Declarative GitOps CD for Kubernetes"
[3]: https://docs.tilt.dev/?utm_source=chatgpt.com "Getting Started With Tilt | Tilt"
[4]: https://devcontainers.github.io/implementors/spec/?utm_source=chatgpt.com "Development Container Specification"
[5]: https://containers.dev/features "Features"
[6]: https://devcontainers.github.io/implementors/json_reference/ "Dev Container metadata reference"
[7]: https://code.visualstudio.com/docs/devcontainers/containers?utm_source=chatgpt.com "Developing inside a Container"
[8]: https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/ "Declarative Setup - Argo CD - Declarative GitOps CD for Kubernetes"
[9]: https://argo-cd.readthedocs.io/en/release-2.9/user-guide/auto_sync/?utm_source=chatgpt.com "Automated Sync Policy - Argo CD - Read the Docs"
[10]: https://argo-cd.readthedocs.io/en/latest/user-guide/sync-options/ "Sync Options - Argo CD - Declarative GitOps CD for Kubernetes"
[11]: https://blog.tilt.dev/2021/06/21/wip-vs-harness.html?utm_source=chatgpt.com "WIP Development vs Harness Development | Tilt Blog"
[12]: https://docs.tilt.dev/dependent_images.html?utm_source=chatgpt.com "Getting Started with Image Builds | Tilt"
[13]: https://docs.tilt.dev/personal_registry.html?utm_source=chatgpt.com "Setting up any Image Registry - Tilt"
[14]: https://docs.tilt.dev/api.html?utm_source=chatgpt.com "Tiltfile API Reference | Tilt"
[15]: https://docs.tilt.dev/choosing_clusters.html?utm_source=chatgpt.com "Choosing a Local Dev Cluster | Tilt"
[16]: https://argo-cd.readthedocs.io/en/stable/user-guide/diffing/ "Diff Customization - Argo CD - Declarative GitOps CD for Kubernetes"
