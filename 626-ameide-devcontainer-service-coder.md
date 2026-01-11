---
title: "626 – AmeideDevContainerService (Coder-based, human workspaces in AKS dev)"
status: active
owners:
  - platform-devx
  - gitops
created: 2026-01-11
---

# 626 – AmeideDevContainerService (Coder-based, human workspaces in AKS dev)

## 0) Purpose

Provide a **human-facing coding workspace** that runs **inside the AKS dev cluster**, is **browser-accessible**, and gives a good “devcontainer-like” developer experience without requiring:

- k3d
- Docker-in-Docker / Docker builds
- VS Code Desktop “Reopen in Container” to a Kubernetes pod

This backlog is **not** the automated/queue-driven coding executor (527 “coder agent”); see §1.

## 0.0 Decision: Coder Community Edition (CE) only

We stick to **Coder Community Edition** (no paid features).

Implications:

- We implement **browser-first**, not “browser-only enforced”.
- The supported workflow is: Coder UI → code-server (VS Code Web) + web terminal.
- We do not claim to block SSH/CLI access at the platform level; we simply do not make it the supported human workflow.

## 0.1 Access URL (environment-scoped)

Expose Coder at an environment-qualified host:

- `coder.dev.ameide.io` (enabled)
- `coder.staging.ameide.io` (disabled; wired)
- `coder.ameide.io` or `coder.production.ameide.io` (disabled; wired)

## 1) Non-negotiable separation of concerns

We keep two execution surfaces intentionally separate:

1) **AmeideCodingAgent** (automated executor, queue-driven; 527 WorkRequests)
2) **AmeideDevContainerService** (human interactive workspace; this backlog)

Hard rule: the human workspace **MUST NOT** be able to “accidentally become” a WorkRequests executor:

- it does not join Kafka consumer groups for `transformation.work.queue.*`
- it does not have the runner entrypoint / identity
- it does not receive the high-privilege repo/bus credentials intended for automated execution

Related: `backlog/527-transformation-integration.md` (workbench pod is “not a processor”), `backlog/527-transformation-capability.md` (hard rules), `backlog/505-agent-developer-v2.md` (executor runtime posture).

## 2) Scope

### In scope

- AKS **dev only** enablement (disabled for local k3d, staging, production)
- Browser-first access (VS Code Web via code-server)
- Repo-as-source-of-truth (GitHub clone/push/PR from inside workspace)
- Environment/namespace-aware defaults (workspace “knows” where it runs)
- Security hardening aligned with GitOps policies: digest pinning, NetworkPolicy, quotas
- Coder SSO integrated with in-cluster Keycloak (OIDC)

### Out of scope (explicit)

- Replacing Telepresence-based remote-first development (`backlog/435-remote-first-development.md`)
- Acting as a generic CI runner / WorkRequests processor
- Docker builds inside the workspace
- k3d workflows inside the workspace
- Staging/prod workspaces

## 3) User experience (human workflow)

1. Developer opens AmeideDevContainerService from the platform UI (Coder UI) and lands in **VS Code (Web)**.
2. Workspace is already configured for:
   - git (clone of target repo + branch workflow)
   - kubectl context defaults scoped to the workspace namespace (if kubectl is included)
   - standard Ameide toolchain required for editing and repo workflows
3. Developer makes changes, runs local/unit checks, pushes branch, opens PR.

### 3.1 Workspace lifecycle (cost control)

- Workspaces must auto-stop when idle (dev-only does not mean “always on”).

### 3.2 “Normal SSO” expectation (Keycloak is necessary, not sufficient)

Even with `CODER_OIDC_*` configured against Keycloak, Coder still requires a **first user** to exist in its DB.
Without it, Coder presents `/setup`.

Decision (dev): **auto-bootstrap** the first admin user so humans go straight to Keycloak SSO without manual setup.

Implementation (dev):

- Coder OIDC is configured via `sources/values/env/dev/platform/platform-coder.yaml`.
- The first admin user is bootstrapped via an Argo hook Job + ExternalSecret in `sources/charts/platform/coder/templates/`.

## 4) Dev environment contract (devcontainer alignment without Docker)

We treat `.devcontainer/devcontainer.json` (in the **code repo**, default `github.com/ameideio/ameide`) as the primary environment contract, but we do **not** require Docker daemon in-cluster.

Required approach (Kubernetes-native; no fallbacks):

- Use Coder’s devcontainer support via **envbuilder** (no Docker daemon requirement).

Decision notes:

- If we require deterministic versions (kubectl/helm/argocd/codex/gh), we should prefer “build-time pinning” over “download on every start”.
- Envbuilder must be treated as part of the supply chain: versions pinned, images traceable, and policy aligned (digests, multi-arch) unless we explicitly define an exception.
- Devcontainer lifecycle scripts (`onCreateCommand`, `postCreateCommand`, `postStartCommand`, etc.) run as part of workspace lifecycle; treat them as product code (avoid “slow apt-get every start” patterns unless intentional).

Related: `backlog/603-image-pull-policy.md`, `backlog/622-gitops-hardening-k3d-aks.md`.

## 4.1 Envbuilder prerequisites (must be explicit)

Define and pin:

- envbuilder base image (digest pinned)
- envbuilder builder image (digest pinned)
- caching strategy:
  - none (slowest, simplest), or
  - registry-backed caching (requires push access and registry auth)

## 4.2 Web IDE baseline (code-server)

- Run code-server with `--auth none` (Coder is the auth boundary).
- Expose code-server via a `coder_app`.
- Use `/healthz` for readiness checks (see 629).

## 5) Identity, credentials, and storage

### 5.1 GitHub identity

Human workflows should use either:

- interactive auth (GitHub CLI login / device flow)

No “cluster-wide bot” GitHub token should be mounted by default into human workspaces.

Templates must never embed secret values; they may only reference Secret names (non-sensitive) and rely on runtime mounts.

### 5.2 Kubernetes identity

The workspace runs with a namespace-scoped ServiceAccount:

- enough to `kubectl get` / `kubectl logs` / basic dev actions in that namespace
- not enough for cluster-wide reads, and not enough to read secrets by default

### 5.3 Persistent storage

Ephemeral node storage is acceptable. Default workspace storage can be per-workspace `emptyDir`:

- `/workspaces` (repo clone, caches)
- `/home/<user>` (editor settings, extensions)

Optional: add a feature flag later for PVC-backed persistence if we need faster warm starts or stable editor state.

Do not share mutable credential state between workspaces by default (avoids cross-talk described in `backlog/624-devcontainer-context-isolation.md`).

## 5.4 Telepresence and the Ameide CLI

Telepresence is embedded in the `ameide` CLI and should be usable wherever the CLI runs.

Policy for Kubernetes-hosted workspaces (this backlog): **Telepresence is not supported.**

Rationale:

- Telepresence requires elevated networking capabilities (e.g., `CAP_NET_ADMIN`) that we do not grant to developer workspaces.
- Telepresence’s value proposition is routing **cluster traffic to a developer-controlled process**; in a Kubernetes-hosted workspace we use in-cluster routing patterns (Gateway overlays, debug endpoints, port-forward) instead of Telepresence intercepts.

Requirements:

- The `ameide` CLI is present and functional.
- Any Telepresence entrypoints in the CLI MUST fail fast with a clear message that Telepresence is not supported in this workspace, and point developers to the supported in-cluster workflow.

Related: `backlog/492-telepresence-verification.md:27-34`, `backlog/621-ameide-cli-inner-loop-test.md:108-120`.

## 6) Networking (AKS dev)

We assume default-deny egress is the target posture (see `backlog/441-networking.md`), so the workspace needs explicit egress allowlists:

- DNS (CoreDNS)
- Coder control plane connectivity (workspace → coderd)
- Keycloak issuer endpoints (OIDC)
- GitHub (clone/push/PR)
- OpenAI endpoints (only if Codex is enabled in the human workspace)
- Kubernetes API (in-cluster)

If the cluster does not enforce egress yet, this backlog still defines the allowlist as a required artifact so we can adopt default-deny later without breaking workspaces.

### 6.1 Coder connectivity requirements (baseline)

- Workspace agent must be able to reach the Coder access URL (`CODER_ACCESS_URL`) over HTTP/HTTPS.
- Ingress/proxy must support WebSockets (required for browser UX).

### 6.2 Egress allowlists and FQDN reality

Kubernetes NetworkPolicy is IP/label-based. If we require “only GitHub/OpenAI”, we must implement one of:

- Cilium FQDN/network policies (if available in the cluster), or
- an egress proxy/NAT gateway approach with allowlisting at the proxy.

## 7) Enablement matrix

- `local` (k3d): disabled
- `dev` (AKS): enabled
- `staging`: disabled
- `production`: disabled

But: all overlays must be wired so it is “one value flip” to enable in other envs later, without re-architecting.

## 8) Acceptance criteria

1. A developer can open VS Code (Web) and work in a repo inside AKS dev without Docker/k3d.
2. Workspace defaults match its deployment namespace/environment (no manual context juggling).
3. Workspace cannot consume WorkRequests queues or act as an automated executor.
4. Workspace can push a branch and open a PR to the repo’s configured base branch.
5. GitOps overlays exist for all envs (enabled only in dev), and NetworkPolicy/quotas are defined.
6. Coder CE posture is reflected in docs: browser-first is supported; “browser-only enforced” is not claimed.
7. Workspaces auto-stop when idle.

## 9) Implementation notes (current)

- Coder templates (under the AmeideDevContainerService umbrella):
  - `ameide-dev` at `sources/coder/templates/ameide-dev/` (defaults to `github.com/ameideio/ameide` with `.devcontainer`)
  - `ameide-gitops` at `sources/coder/templates/ameide-gitops/` (defaults to `github.com/ameideio/ameide-gitops` with `.devcontainer/coder`)
- code-server runs as a sidecar container (pinned image), not installed at runtime via curl.
- Workspace RBAC is namespace-scoped and does not grant `secrets` read access by default.
