# 624 – DevContainer Context Isolation (kubectl + Azure + Ameide defaults)

**Status:** Draft  
**Owner:** Platform / Developer Experience  
**Related backlogs:** [491-auto-contexts.md](491-auto-contexts.md), [492-telepresence-verification.md](492-telepresence-verification.md), [581-parallel-ai-devcontainers.md](581-parallel-ai-devcontainers.md), [435-remote-first-development.md](435-remote-first-development.md), [613-codex-auth-json-secret.md](613-codex-auth-json-secret.md)

## Problem

The DevContainer uses **named Docker volumes** to persist credentials and configuration across rebuilds, and to share them across multiple parallel containers (agent slots).

Today this includes:

- `~/.kube` (`kubectl` contexts + `current-context`)
- `~/.azure` (Azure CLI auth state + selected subscription/tenant)
- `~/.config/ameide` (defaults like Telepresence context/namespace written by bootstrap)

When multiple DevContainers are running concurrently, sharing these paths causes **cross-talk**:

- `kubectl config use-context ...` in one container changes `current-context` for all containers sharing the volume.
- `az account set ...` in one container changes the default subscription for all containers sharing the volume.
- `tools/dev/bootstrap-contexts.sh` writes shared defaults (e.g., Telepresence context/namespace), so one container can accidentally redirect another container’s inner-loop to the wrong environment.

This is especially risky when one container targets dev and another targets staging/prod, or when a developer uses multiple Azure subscriptions/tenants (e.g., personal vs work, multiple customer subs).

## Key clarifications

- “No host” is not a useful assumption for DevContainers: even if workloads run on a remote Kubernetes cluster, the DevContainer still runs on *some* compute substrate (local Docker engine, Codespaces VM, etc.) with persistent storage semantics (named volumes). Shared volumes are shared state.
- Context isolation here is about **developer tooling state**, not cluster-side isolation. Namespaces/RBAC still enforce access on the cluster.

## Goals

1. Allow multiple DevContainers (including parallel agent slots) to run concurrently without mutating each other’s `kubectl` context, namespace, or Azure subscription defaults.
2. Preserve the ability to “bootstrap once” when desired, but make isolation the default when parallelism is expected.
3. Make it hard to accidentally run commands against the wrong environment.

## Proposed changes

### 1) Isolate per-slot/per-workspace credential volumes

Update DevContainer mounts so that **agent slots** (and optionally all devcontainers) use per-workspace or per-slot named volumes.

Prefer using the Dev Containers `${devcontainerId}` variable as the suffix to avoid basename collisions across unrelated workspaces, while remaining stable across rebuilds.

Example patterns:

- `ameide-kube-${localWorkspaceFolderBasename}-${devcontainerId}` → `/home/vscode/.kube`
- `ameide-azure-${localWorkspaceFolderBasename}-${devcontainerId}` → `/home/vscode/.azure`
- `ameide-ameide-config-${localWorkspaceFolderBasename}-${devcontainerId}` → `/home/vscode/.config/ameide`

This works naturally with the “agent slot” pattern because each slot has a distinct workspace folder basename (`agent-01`, `agent-02`, …).

### 1.1) Telepresence state locations (note)

Telepresence has both:

- global workstation config (typically under `~/.config/telepresence/` on Linux)
- per-cluster configuration that can be stored as extensions to `KUBECONFIG`

Isolating `~/.kube` per slot also isolates Telepresence’s per-cluster config. If we later choose to persist Telepresence’s global config directory, it should follow the same per-slot isolation approach (avoid broad changes like setting `XDG_CONFIG_HOME` unless we also isolate Telepresence under it).

### 2) Decide what to do with `~/.codex`

`~/.codex` contains Codex/OpenAI auth/config and may legitimately be shared, but teams may also want to isolate it (different accounts / principals).

Provide one of:

- **Default shared, optional isolation**: keep `ameide-codex` shared unless a new env var (or an alternate devcontainer config) opts into per-slot isolation. Prefer using `CODEX_HOME` for isolation rather than ad-hoc paths.
- **Full isolation**: treat `~/.codex` like `~/.kube` and isolate per slot for safety.

### 3) Update bootstrap behavior for agent profiles

The current `agent` bootstrap profile intentionally skips bootstrapping and reuses existing shared defaults. With isolated volumes, the first run for an agent slot will not have `~/.config/ameide/context.env`.

Update the agent flow so that:

- If `~/.config/ameide/context.env` is missing, print a single “run bootstrap” command (or optionally auto-bootstrap when `AMEIDE_AUTO_BOOTSTRAP=1`).
- If present, continue to reuse it for that slot.

### 4) Make tool state directories explicit (env vars)

To avoid ambiguity about where tools read/write state inside containers, explicitly set:

- `AZURE_CONFIG_DIR=/home/vscode/.azure` (Azure CLI)
- `CODEX_HOME=/home/vscode/.codex` (Codex)

## Migration / rollout

1. Introduce new isolated-volume mounts in `.devcontainer/agent/devcontainer.json` first (lowest risk; targets parallel usage explicitly).
2. Optionally introduce a second config for humans who run multiple non-agent devcontainers concurrently (e.g., `.devcontainer/isolated/devcontainer.json`), leaving the default unchanged for “single container” workflows.
3. Document the behavior and recommended patterns in `.devcontainer/README.md`.

## Acceptance criteria

1. Two devcontainers can run side-by-side with different `kubectl config current-context` values without affecting each other.
2. Two devcontainers can run side-by-side with different `az account show` subscriptions without affecting each other.
3. `tools/dev/bootstrap-contexts.sh` writes context defaults that apply only to the current container/slot.
4. The “agent slot” workflow remains fast and reliable (no hidden shared-state coupling).

## References

- `.devcontainer/devcontainer.json` (named volume mounts)
- `.devcontainer/agent/devcontainer.json` (agent slot config)
- `tools/dev/bootstrap-contexts.sh` (writes context defaults)
- [491-auto-contexts.md](491-auto-contexts.md) (desired kubectl/Telepresence/argocd contexts)
- [492-telepresence-verification.md](492-telepresence-verification.md) (context bootstrap + verification)
