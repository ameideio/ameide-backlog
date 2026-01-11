---
title: "628 – Implement Coder for AmeideDevContainerService (AKS dev-only) + delivery plan"
status: draft
owners:
  - platform-devx
  - platform-sre
  - gitops
created: 2026-01-11
---

# 628 – Implement Coder for AmeideDevContainerService (AKS dev-only) + delivery plan

## 0) Goal

Deliver the **Coder-based** human workspace described in `backlog/626-ameide-devcontainer-service-coder.md` with a concrete, GitOps-aligned implementation plan.

This backlog is about the **human workspace platform**, not the WorkRequests executor runtime.

## 0.1 Decision: Coder Community Edition (CE) only

We stick to **Coder Community Edition**:

- no “browser-only enforced” requirements
- do not rely on Premium features (e.g., workspace proxies, external provisioners)

## 1) Architecture summary (what we are implementing)

Components:

- **Coder control plane** in AKS dev (coderd + DB + provisioners)
- **AmeideDevContainerService template** that provisions:
  - a workspace Pod (or Deployment) with the dev toolchain
  - PVCs (optional) for `/home` and `/workspaces`
  - `coder_app` for **VS Code (Web)** (code-server)
  - namespace-aware environment defaults

Build strategy:

- Required: **envbuilder** from repo `devcontainer.json` (no Docker daemon inside pods, no base-image fallback).

## 1.1 Install mechanics (GitOps-owned)

- Install Coder via a GitOps-managed Helm release (Argo CD Application) in AKS dev.
- Configure `CODER_ACCESS_URL` to `https://coder.dev.ameide.io` and ensure ingress supports WebSockets.

## 2) GitOps delivery shape (repo changes expected)

### 2.1 New GitOps components (AKS dev enabled only)

- `cluster` scope:
  - (if required) CRDs/controllers for Coder (or Helm chart installation)
- `env/dev` scope:
  - Coder namespace + services
  - NetworkPolicies (ingress/egress) for coderd and workspaces
  - OIDC configuration referencing in-cluster Keycloak issuer (see §3)

### 2.2 Values contract requirements

We must be able to express:

- `enabled: true|false` per environment (dev-only enabled)
- image references (digest pinned, multi-arch where required)
- egress allowlists (DNS, GitHub, OpenAI optional)
- resource requests/limits defaults
- storage class and PVC sizing
  - (default) ephemeral workspace storage sizing and limits

Related hardening constraints: `backlog/519-gitops-fleet-policy-hardening.md`, `backlog/622-gitops-hardening-k3d-aks.md`.

## 2.3 Template delivery pipeline (not GitOps-native)

Argo CD deploys Coder, but Coder templates are not Kubernetes manifests.

Define:

- where the Terraform template lives (repo/path)
- how CI pushes template versions to Coder (Coder CLI/API) and promotes after E2E passes
- how we keep the template version in sync with the GitOps release (operational runbook)

## 3) Security model (implementation requirements)

Hard requirements:

- **No WorkRequests queue consumption** from human workspaces (no Kafka creds, no consumer group identity).
- Namespace-scoped ServiceAccounts for workspaces; no cluster-admin.
- Default-deny NetworkPolicy posture supported (explicit allowlists).
- ResourceQuota/LimitRange patterns applied (dev-only doesn’t mean unbounded).

## 3.1 Authentication (Keycloak OIDC)

Coder must use in-cluster Keycloak as the OIDC provider.

Requirements:

- Coder access URL uses env-qualified hostname: `coder.dev.ameide.io`.
- Keycloak has a dedicated OIDC client for Coder with redirect URIs for the configured access URL.
- NetworkPolicy allows workspace → Keycloak issuer endpoints for auth flows where needed.

## 3.2 Template secret hygiene

- Do not embed secret values in templates (assume templates are readable by all template users).
- Prefer Keycloak SSO for Coder and runtime secret mounts by Secret name when automation is required.

## 4) Implementation plan (phased)

### Phase 0 — Decisions (blockers to unblock implementation)

1. Confirm envbuilder specifics for strict `.devcontainer` parity (no fallback):
   - base image (digest pinned)
   - builder image (digest pinned)
   - caching strategy (none vs registry-backed) and registry auth
2. Confirm naming:
   - automated executor: AmeideCodingAgent
   - human workspace: AmeideDevContainerService (or AmeideDevWorkspace)
3. Confirm whether human workspaces get kubectl:
   - if yes: namespace-scoped kube identity only
   - if no: document how developers debug cluster state (web terminal only, logs via platform, etc.)
4. Telepresence stance (decided): Telepresence is not supported in Kubernetes-hosted workspaces; CLI must fail fast with a clear message and direct users to the supported in-cluster workflow.
5. Workspace auto-stop/idle policy (decided): enable auto-stop on idle (default 30 minutes) for developer workspaces.

### Phase 1 — Control plane deploy (AKS dev only)

1. Add GitOps component(s) to install Coder control plane in `ameide-dev`.
2. Configure access URL/routing for browser usage (`coder.dev.ameide.io`).
3. Lock down RBAC for who can create/start workspaces (dev-only group).
4. Configure Keycloak OIDC integration (SSO) and validate login end-to-end.

### Phase 2 — Workspace template (AmeideDevContainerService)

1. Create the template (Terraform-based in Coder) to provision the workspace pod and app.
2. Ensure repo clone + git/gh setup works.
3. Ensure workspace identity/storage/network policies match 626.
4. Ensure workspace default storage is ephemeral (node-backed) and sized appropriately.
5. Prefer `coder_script` for lifecycle automation/glue (over legacy startup mechanisms) where we need deterministic “on start” behavior.

### Phase 3 — Operational hardening

1. NetworkPolicy tighten + explicit egress allowlists.
2. Quotas/LimitRanges applied.
3. Image policy alignment (digest pinning, multi-arch where needed).
4. Observability: logs/metrics baseline for coderd and workspaces.

### Phase 4 — Documentation + adoption

1. Update/remove docs per `backlog/627-ameide-devcontainer-service-impact-and-cleanup.md`.
2. Onboarding doc: “open workspace → branch → PR”.

## 5) Acceptance criteria

1. Coder is deployed in AKS dev only and is reachable via browser.
2. Developers can create a workspace and open VS Code (Web).
3. Workspace has the intended toolchain without Docker/k3d.
4. Workspace is namespace-aware and does not have cross-env credential leakage.
5. NetworkPolicy + quotas exist and do not block the intended workflow.
