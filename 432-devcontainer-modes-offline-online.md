# 432 – Devcontainer modes: offline (full) and online (telepresence + Tilt)

> **⚠️ RETIRED – SUPERSEDED BY [435-remote-first-development.md](435-remote-first-development.md)**
>
> This document describes the **dual-mode approach** (offline-full + online-telepresence).
> The project is migrating to a **single remote-first mode** where:
> - **No offline mode** – All development connects to shared remote AKS cluster
> - **No local k3d** – DevContainer only needs Telepresence prerequisites (Tilt removed)
> - Infrastructure code moved to `ameide-gitops` repo
> - Telepresence workflows are owned by the Ameide CLI (`ameide dev inner-loop verify/up/down`)
>
> See [435-remote-first-development.md](435-remote-first-development.md) for the new approach.
>
> **Update (2026-01):** 435 itself is superseded by the internal-only, Coder-based model in `backlog/650-agentic-coding-overview.md`.

> **Related documents:**
> - [435-remote-first-development.md](435-remote-first-development.md) – **NEW**: Remote-first development (single mode)
> - [367-bootstrap-v2.md](367-bootstrap-v2.md) – Bootstrap design (moved to ameide-gitops)
> - [429-devcontainer-bootstrap.md](429-devcontainer-bootstrap.md) – DevContainer bootstrap (superseded)
> - [434-unified-environment-naming.md](434-unified-environment-naming.md) – Environment naming standardization

## Update – Minimal profile retired
- **2025-12-02:** The profile-based "offline-minimal" experiment has been removed. Maintaining two GitOps projections bloated chart logic, left Argo in `Unknown`/`Degraded` states when services were intentionally absent, and added cognitive load to the DevContainer. The codebase now ships a single offline profile (full stack) plus the online/telepresence workflow.
- Cleanup steps completed in this change:
  - Deleted `infra/environments/dev/bootstrap-minimal.yaml`, `gitops/.../argocd/profiles/minimal/**/*`, and `gitops/.../sources/values/profile/minimal/**/*`.
  - Simplified `.devcontainer/postCreate.sh` and `tools/dev/devcontainer-mode.sh` so the only local bootstrap path is `offline-full` (k3d + Argo). Telepresence mode still bypasses local bootstrap entirely.
  - Removed profile-aware logic from `gitops/ameide-gitops/environments/dev/argocd/apps/ameide.yaml`; every Application now renders from shared + env-specific values, eliminating empty templating and ApplicationSet skips.

## Goals
- Give contributors a predictable k3d-based DevContainer that provisions the full dev stack (Argo, GitOps apps, local registry, waits, and image builds) without special casing subsets.
- Support an "online-telepresence" workflow that skips local cluster bootstrap, assumes a healthy remote AKS dev cluster, and lets Telepresence + the Ameide CLI target that context for rapid service iteration.
- Preserve the `*-tilt` split so locally-built services never fight Argo reconciliation, regardless of mode.

## Implementation Status: ✅ Offline full + Online telepresence
- Offline bootstrap always uses `infra/environments/dev/bootstrap.yaml`. There is no longer a `profile` dimension or Kustomize overlay that prunes Applications.
- Telepresence mode remains opt-in; it just reuses the remote context wiring added previously and does not reintroduce the minimal profile.

### Architecture Overview
```
Mode Selection (DevContainer)
         │
         ▼
┌──────────────────────────────────────────────┐
│  offline-full        → env=dev (k3d + Argo)  │
│  online-telepresence → remote AKS context    │
└──────────────────────────────────────────────┘
         │
         ▼
Bootstrap Config (infra/environments/dev/bootstrap.yaml)
         │
         ▼
ApplicationSet (environments/dev/argocd/apps/ameide.yaml)
         │
         └── All Applications render with shared + env values
```

### Key Files Updated
| Component | File | Change |
|-----------|------|--------|
| DevContainer | `.devcontainer/postCreate.sh` | Mode dispatcher only supports `offline-full` and `online-telepresence`; default mode is now `offline-full`.
| DevContainer toggle | `tools/dev/devcontainer-mode.sh` | Reapply script now just re-runs the full bootstrap (minimal mode removed).
| GitOps ApplicationSet | `gitops/ameide-gitops/environments/dev/argocd/apps/ameide.yaml` | Deleted profile-specific value files and the third-party skip list; Helm releases always read `_shared` + env overlays.
| GitOps values & overlays | `gitops/ameide-gitops/sources/values/profile/` | Directory removed; charts rely solely on `_shared`, `dev`, or Tilt/local overlays.

## Target modes
### Offline-full
- Applies the full GitOps tree via `bootstrap-v2.sh` (install Argo, projects, root apps, secrets, waits, port-forwards, and dev image builds).
- Runs in the DevContainer by default (`.devcontainer/postCreate.sh` without args).

### Online-telepresence
- **Update (2026-01):** Devcontainer “modes” were removed to reduce cognitive load. Remote-first dev uses a single bootstrap path + the Ameide CLI.
- Installs the Telepresence CLI; the `ameide` CLI owns Telepresence connect/intercept as part of `ameide dev inner-loop up|down|verify` and `ameide dev inner-loop-test`.
- Tilt-based orchestration is deprecated/removed from the core repo; do not rely on `TILT_REMOTE` or a repo `Tiltfile` as part of the remote-first workflow.
- Documents the workflow in `docs/dev-workflows/telepresence.md`.

### Retired: Offline-minimal
- The minimal profile previously disabled first-party apps and heavy data plane components. It introduced persistent Argo health noise (missing services, invalid specs) and duplicate Helm/Kustomize plumbing. Removing it restored a single source of truth for GitOps and simplified onboarding.

## Backlog (epics + tasks)

### Epic DC-0 – Mode selection & wiring ✅ COMPLETED
- `.devcontainer/postCreate.sh` accepts `{offline-full|online-telepresence}` and logs the choice (default: offline-full).
- Mode map now directly selects the single bootstrap config or remote context; no profile indirection remains.

### Epic DC-2 – Offline-minimal mode ❌ RETIRED
- All profile-specific values/templates were deleted. Any future "lightweight" dev story should reuse Tilt/Terraform caches rather than a second GitOps projection.

### Epic DC-3 – Runtime switching between offline modes 🔄 IN PROGRESS
- `tools/dev/devcontainer-mode.sh offline-full [--reset]` reapplies the bootstrap with optional k3d reset. No alternate mode exists anymore; script is retained for convenience when re-syncing the full stack.

### Epic DC-4 – Online-telepresence mode

> **Note:** This epic is aligned with [434-unified-environment-naming.md](434-unified-environment-naming.md#epic-env-3) for telepresence implementation.

- **DC-30 – Devcontainer bootstrap for online-telepresence** ✅ **Update (2026-01):** modes removed; devcontainer no longer persists mode files.
- **DC-31 – Telepresence connect helper** ✅ Telepresence is installed automatically inside the DevContainer; `ameide dev inner-loop verify` and `ameide dev inner-loop up` are the canonical workflows.
- **DC-32 – Tilt remote toggle (`TILT_REMOTE`)** ⚠️ Deprecated: Tilt-based workflows are removed from the core repo (historical context only).
- **DC-33 – Telepresence bootstrap config** ✅ `infra/environments/dev/bootstrap-telepresence.yaml` describes the AKS context and auto-connect settings; `postCreate.sh` consumes it.

**Blocker:** Remote AKS cluster must be migrated to `ameide-gitops` structure before telepresence mode is fully functional. See [434 Migration Needed](434-unified-environment-naming.md#migration-needed) for the cluster-side work.

### Epic DC-5 – Docs & UX
- **DC-50 – Devcontainer modes HOWTO** ⏳ Needs refresh to reflect the simplified mode matrix and the removal of the minimal profile trace.
