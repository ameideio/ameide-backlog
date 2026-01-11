---
title: "627 – AmeideDevContainerService adoption: backlog + implementation impact (update/remove plan)"
status: draft
owners:
  - platform-devx
  - gitops
created: 2026-01-11
---

# 627 – AmeideDevContainerService adoption: backlog + implementation impact (update/remove plan)

## 0) Intent

If we adopt **Coder-based human workspaces** (626), we must update existing docs and placeholders so we do not conflate:

- **AmeideCodingAgent** (automated executor; WorkRequests; 527/505)
- **AmeideDevContainerService** (human workspace; browser IDE; 626)

This backlog lists what to update, what to deprecate, and what to remove.

## 0.1 Decision: Coder Community Edition (CE) only

We do not adopt paid Coder features. Any language that implies “browser-only enforced” must be removed or rewritten as “browser-first”.

## 1) Naming and terminology cleanup

### 1.1 Rename to avoid “Coder” overload

- Prefer **AmeideCodingAgent** for automated executor runtime currently described as “AmeideCoder” in:
  - `backlog/505-agent-developer-v2.md`
  - `backlog/505-agent-developer-v2-implementation.md`
  - `backlog/500-agent-operator.md`

Preserve historical references as “Formerly AmeideCoder (renamed for clarity)” rather than rewriting history.

### 1.2 Human workspace naming

- Use **AmeideDevContainerService** (or “AmeideDevWorkspace”) for the Coder-hosted human workspace.
- Avoid calling human workspaces “executor runtime” or “coder agent”.

## 2) Backlogs requiring updates (by file)

### 2.1 Agent/executor architecture

- `backlog/505-agent-developer-v2.md`
  - Split “executor runtime” (automated) vs “human dev workspace” explicitly.
  - Reconcile the statement “No cluster credentials inside coder workspace” with the new naming:
    - keep it for the automated executor runtime
    - allow namespace-scoped kube identity for human workspaces
- `backlog/505-agent-developer-v2-implementation.md`
  - Track P4 “Executor runtime” should remain automated (WorkRequests), not the human service.
  - Add a parallel track (or dependency) for human workspaces (626/628).
- `backlog/500-agent-operator.md`
  - Ensure `develop_in_container` remains clearly a compatibility bridge to an automated executor surface (or is deprecated), and is not confused with the human workspace.

### 2.2 WorkRequests / runner substrate

- `backlog/527-transformation-capability.md`
  - Keep the workbench semantics (“not a processor”) and explicitly state whether the human workspace replaces the workbench pod or complements it.
- `backlog/527-transformation-integration.md`
  - Update “IDE attach” guidance to reflect the chosen attach mechanism (Coder apps / code-server) for the human workspace.
- `backlog/586-workrequests-execution-queues-inventory.md`
  - If we keep both surfaces:
    - “workbench pod” remains the admin/debug escape hatch
    - “AmeideDevContainerService” is the developer UX surface
  - If we replace workbench with Coder workspaces:
    - remove “workbench deployed” claims and replace with “Coder workspaces available”

### 2.3 Devcontainer bootstrap + context assumptions

- `backlog/491-auto-contexts.md`
  - Mark which parts are “local devcontainer on a workstation” vs “in-cluster human workspace”.
  - Remove/avoid assumptions that Azure CLI device auth is available inside a Kubernetes workspace.
- `backlog/492-telepresence-verification.md`
  - Keep “CAP_NET_ADMIN required” guidance, and explicitly state Telepresence is not supported in Kubernetes-hosted workspaces (626); Telepresence remains a workstation/devcontainer workflow targeting AKS dev.
- `backlog/624-devcontainer-context-isolation.md`
  - Keep as “local parallel devcontainers” (developer-mode) unless we explicitly extend it to workspace PVC isolation.

### 2.4 Security/networking/image policy

- `backlog/441-networking.md`
  - Add egress allowlists specifically for developer workspaces (DNS, Keycloak issuer, GitHub, OpenAI optional, coderd).
- `backlog/442-environment-isolation.md`
  - Ensure Coder is dev-only enabled and does not introduce cross-env routing, shared public IP collisions, or node-pool affinity violations.
- `backlog/463-multi-tenant-rbac-quotas.md` and `backlog/300-400/316-security.md`
  - Ensure workspace namespaces/pods have quotas/limits patterns, even if they are “dev-only”.
- `backlog/603-image-pull-policy.md` and `backlog/622-gitops-hardening-k3d-aks.md`
  - Define how workspace images are pinned (digest + multi-arch) or document a controlled exception if envbuilder builds dynamically.

### 2.5 Namespace label conventions

- `backlog/465-applicationset-architecture-preview-envs-v2.md`
  - Ensure workspace namespaces carry required labels (e.g., `ameide.io/environment=dev`) so shared NetworkPolicy/Gateway patterns do not break.
- `backlog/442-environment-isolation.md`
  - Ensure dev-only enablement is consistent with env isolation and node-pool scheduling.
## 2.6 Authn/Authz (OIDC)

- Add a Coder OIDC integration doc slice that references the in-cluster Keycloak issuer and client configuration.
- Ensure any legacy “admin password” workflows for developer tooling do not become the default for Coder.

## 2.7 Template secret hygiene

- Add an explicit policy statement: do not embed secret values in Coder templates; treat templates as readable by all template users.
- Templates may reference Secret *names* (non-sensitive) and rely on runtime mounts/ExternalAuth, but must not carry credentials in cleartext.

## 3) GitOps implementation cleanup targets

### 3.1 Existing placeholder devcontainer service

`backlog/505-agent-developer-v2.md` and `backlog/603-image-pull-policy.md` note that `ameide-gitops` currently has a disabled placeholder for a “devcontainer service”.

Decide and execute one of:

1) **Repurpose** it into “AmeideDevContainerService (human)” deployment assets (Coder control plane + template integration), or
2) **Keep** it for the automated executor runtime (AmeideCodingAgent) and create separate assets for the human workspace, or
3) **Remove** the placeholder and replace it with explicit Coder-managed components.

## 4) Acceptance criteria

1. The docs use unambiguous names: automated executor vs human workspace.
2. 505/500/527 no longer conflict on “who has cluster creds” because the statement is scoped to the automated executor runtime.
3. GitOps repo has exactly one “human workspace” implementation story (not a stub + a second system).
4. Any deprecated artifacts remain referenced only as historical notes, not as implied current state.
