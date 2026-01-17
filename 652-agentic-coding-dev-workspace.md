---
title: "652 – Agentic coding — dev workspace (Coder workspaces; human-in-the-loop coding)"
status: draft
owners:
  - platform-devx
  - gitops
created: 2026-01-12
suite: "650-agentic-coding"
source_of_truth: false
---

# 652 – Agentic coding — dev workspace (Coder workspaces; human-in-the-loop coding)

## 0) Purpose

Define the human developer experience for agentic coding:

- A Kubernetes-hosted “dev machine” via **Coder workspaces**
- Browser VS Code via **code-server**
- A Kubernetes-safe devcontainer contract via **envbuilder**
- Human-in-the-loop coding with assistants (Codex/Claude) from within the workspace UX

This document inherits all decisions from `backlog/650-agentic-coding-overview.md`.

## 0.2 Status update (2026-01-17)

- Phase 3 “innerloop routing” RBAC is now treated as cluster bootstrap (predeclared roles + bind-only delegation to the Coder provisioner) so workspace Terraform does not trip RBAC privilege escalation.
- Workspace namespaces are labeled `gateway-access=allowed` and the Envoy Gateway listeners are configured to only accept routes from labeled namespaces (Gateway API trust model).

## 0.1 Primary UX goal: the workspace is “agent-native”

The workspace is not just a “human IDE in the cluster”; it is the place where agents and humans share a single, reproducible dev machine contract.

Key idea:

- the CLI + `AGENTS.md` encode “what to run and how” so agents do not reverse-engineer the monorepo

## 1) Decisions (inherited from 650; repeated for local clarity)

- Coder CE only; browser-first.
- No Telepresence; no privileged networking capabilities in workspace pods.
- `.devcontainer/coder/devcontainer.json` is the environment contract for workspaces and automation tasks.
- Inner loop happens in the workspace; outer loop is preview env truth.

## 2) Scope (in/out)

### 2.1 In scope

- Human workspaces in the dev cluster via Coder.
- A single, reproducible devcontainer contract for humans and automation tasks.
- Browser VS Code + terminal UX.
- In-workspace assistants (Codex minimum; other providers optional).

### 2.2 Out of scope

- Automation orchestration and task launcher details (see 651).
- Preview environment deployment and merge gating mechanics (see 653).

## 3) Terminology (additions to 650)

- **Workspace image:** the image produced by envbuilder from the devcontainer contract.
- **Lifecycle scripts:** devcontainer commands that run during create/start.

## 4) Architecture (workspace contract)

### 4.1 Devcontainer

Primary contract:

- `.devcontainer/coder/devcontainer.json`

Rules:

- Treat lifecycle scripts as product code (fast, deterministic, minimal network fetches).
- Prefer build-time pinning over “download tools every start”.

### 4.1.1 Storage (dev default)

- `/workspaces` is persistent via a per-workspace PVC (StorageClass `managed-csi-premium`, size `16Gi` fixed).
- Stopping a workspace must not delete the workspace namespace/PVC; deletion is the intentional “data destroy” event.
- `$HOME` is ephemeral unless we add a separate PVC (future option).

### 4.1.2 Toolchain contract (required)

The Coder devcontainer must ship the minimum toolchain required to run the “no-brainer” inner loop (`ameide test` / `ameide test ci`) without additional manual setup:

- Go (`go`, `gofmt`) — version pinned to the repo (`go.mod`).
- Node.js (`node`) — currently 22.
- pnpm (`pnpm`) — pinned to the repo’s `package.json#packageManager`.
- uv (`uv`, `uvx`) — pinned (currently `0.9.26`) and installed into the user PATH.
- buf (`buf`) — pinned (currently `v1.63.0`) and installed into the user PATH.

Implementation notes (current):

- Build-time pinning is preferred: Go and pnpm are installed via `.devcontainer/coder/devcontainer.json` features.
- `uv` and `buf` are installed by `.devcontainer/coder/postCreate.sh` (idempotent, pinned, no apt-get) via `.devcontainer/lib/tooling.sh`.

### 4.2 code-server app exposure

Workspace must expose code-server through a Coder app proxy surface (vendor-aligned pattern):

- Start code-server with `--auth none`.
- Use `/healthz` for readiness/smoke checks.
- Prefer the vendor-supported code-server module (vs ad-hoc daemon management in `startup_script`).

### 4.3 Template-scoped `AGENTS.md` (required)

Each Coder workspace template must ship with a scoped `AGENTS.md` that matches the workspace’s intended “agent profile”.

Canonical layout:

- `/workspaces/<profile>/AGENTS.md`
- `/workspaces/<profile>/repo/` (the checkout)

This enables:

- different guardrails per template (code vs gitops vs sre)
- a consistent “no-brainer” instruction set per agent type

## 5) Implementation plan

### 5.1 Human workflow (golden path)

1. Open Coder UI and create workspace from the standard template.
2. Land in VS Code (Web).
3. Create a feature branch and implement changes.
4. Run fast checks in-workspace (unit/lint/typecheck).
5. Push and open PR.
6. Use preview environment E2E as merge gate truth.

### 5.2 Human-in-the-loop assistants (Codex/Claude)

Principles:

- Assistants run **inside the workspace** (or via approved external APIs) with workspace-local context.
- Workspace identity and secrets are explicit; do not rely on “developer laptop” credentials.

Minimum deliverable:

- Codex is usable in the workspace **out of the box**:
  - Codex CLI installed (best-effort; **no pin/version knob**; “latest” by default)
  - Codex VS Code extension installed in code-server (`openai.chatgpt`, best-effort; no pin)
  - authentication pre-seeded via slot secrets (`Secret/codex-auth-rotating-0..2` preferred; falls back to `Secret/codex-auth-0..2`) using file-backed credentials (`cli_auth_credentials_store = "file"`) so CLI and extension share the same cache (see `backlog/613-codex-auth-json-secret.md`)
  - resilience: Codex installs must not block workspace readiness; failures are surfaced in script logs but should not flip the workspace to unhealthy

Optional deliverable:

- Claude (or other provider) via VS Code Web extension configuration, with secrets injected at runtime.

### 5.3 Hardening (workspace security posture)

Non-negotiables:

- No Docker socket mounting.
- No `CAP_NET_ADMIN`.
- No `/dev/net/tun`.
- Least-privilege RBAC (namespace-scoped by default).

### 5.4 Implementation checklist (dev cluster)

- Coder control plane deployed with environment-qualified access URL.
- Template uses envbuilder with `.devcontainer/coder`.
- Workspace network policy allows only required egress (DNS, Coder control plane, GitHub, approved APIs).
- Workspace quotas/limits prevent runaway resource usage.

## 6) Testing

- Platform smoke validates workspace provisioning and code-server reachability (see 653).
- Developer fast checks are executed via `ameide test` (see 653).

## 7) Risks / guardrails

- Avoid slow/non-deterministic startup: lifecycle scripts must not become “apt-get on every start”.
- Avoid tool drift: prefer pinned versions/digests for critical tools and base images.
- Avoid privilege creep: workspaces should not be used as generic cluster-admin shells.

## 8) Deprecations / replacements

- Deprecate any workspace patterns that require privileged networking (Telepresence, TUN, `CAP_NET_ADMIN`).
- Treat `.devcontainer/devcontainer.json` and `.devcontainer/agent/devcontainer.json` as legacy/unrelated to the Coder workspace contract; the authoritative contract is `.devcontainer/coder/devcontainer.json`.
 - Treat “one template for everyone” as legacy; templates are agent profiles with scoped instructions and guardrails (see 650).

## 9) References

- Normative decisions: `backlog/650-agentic-coding-overview.md`
- Coder platform implementation lives in the GitOps repo (submodule): `gitops/ameide-gitops/`
