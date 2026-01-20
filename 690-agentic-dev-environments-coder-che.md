---
title: "690 – Agentic dev environments — big picture (Coder + Che)"
status: draft
owners:
  - platform-devx
  - platform
  - gitops
created: 2026-01-17
suite: "agentic-coding"
source_of_truth: false
---

# 690 – Agentic dev environments — big picture (Coder + Che)

## 0) Purpose

Articulate a single “big picture” for **agentic development environments** where:

- **Coder** and **Eclipse Che** are both supported as workspace/task providers.
- Platform contracts (CLI front doors, evidence, secrets, guardrails, GitOps) remain **provider-independent**.
- Platform operators retain control over **cluster bootstrap** and “routing power” (RBAC + Gateway API trust model).

This doc is a narrative map. Normative details remain in:

- Coder-first model: `backlog/650-agentic-coding-overview.md`
- Che provider: `backlog/680-che.md`
- Test layers: `backlog/653-agentic-coding-test-automation-coder.md`
- CLI front doors: `backlog/654-agentic-coding-cli-surface-coder.md`

## 1) One contract, multiple providers

Provider choice is a runtime decision; the contract is not.

Common expectations (both providers):

- Same deterministic entrypoints: `ameide test` and `ameide test cluster`.
- Same evidence layout and semantics (JUnit always produced).
- Same guardrails (unprivileged pods, no Telepresence assumptions).
- Same credential model (bot Git, Codex slots, no secret embedding in templates).

Provider-specific responsibilities:

- **Coder:** provision workspaces via templates (Terraform) + app proxy (code-server).
- **Che:** provision workspaces via `DevWorkspace` CRs (Devfile) + IDE attachment and Kubernetes RBAC-driven UX.

## 2) The planes (how everything composes)

This framing is shared across Coder and Che.

1. **Control plane (workspace provider):** creates/starts/stops/deletes workspaces and exposes the IDE UX.
   - Coder: `coderd` + provisioners.
   - Che: Che operator + `CheCluster` + DevWorkspace controller.

2. **Orchestration plane (runs/tasks):** supervises automation lifecycles (timeouts, retries, evidence, cleanup).
   - Standard: Camunda BPMN (see `backlog/651-agentic-coding-ameide-coding-agent.md`).

3. **Ingress/data plane (traffic):** Envoy Gateway + Gateway API (`HTTPRoute`) provide controlled routing.
   - Deterministic “divert only Playwright traffic” uses header-matched `HTTPRoute` rules (see `backlog/417-envoy-route-tracking.md`).

4. **Workload plane (execution):** workspace namespaces (or vCluster-synced namespaces) where pods run unprivileged.

## 3) Cluster bootstrap: RBAC + Gateway trust (GitOps-owned)

### 3.1 Gateway API bidirectional trust model

- Gate route attachment by namespace labels (example: `gateway-access=allowed`) and enforce it via `allowedRoutes.namespaces.selector` on the Gateway listener.
- Workspaces/tasks may label their own namespaces to opt-in, but cannot bypass listener policy.

### 3.2 “Granting routing power” is bootstrap, not workspace Terraform

To avoid RBAC privilege-escalation failures and keep authority centralized:

- Predeclare the allowed permission set as centrally managed RBAC objects (ClusterRoles/Roles).
- Allow workspace provisioners to **bind** only those predeclared roles (`bind` verb + `resourceNames`), rather than minting arbitrary Roles.

This pattern is required for both providers if tasks/workspaces need to manage routing primitives (Services, HTTPRoutes) as part of inner-loop or Phase 3 test harnesses.

## 4) Routing model: one baseline, one diverted (header-tagged)

Use Gateway API as intended:

- A baseline route serves the normal “dev” experience.
- A second route diverts only automation traffic by matching on a dedicated header (Playwright/CLI-owned).

Operational implications:

- Diverted routing is policy-controlled (namespace gate + RBAC).
- The CLI owns route lifecycle and waits for `Accepted`/`ResolvedRefs` before running Playwright.

## 5) Environment contract: devcontainer + devfile (generated, not forked)

We want “two providers” without “two dev machines”.

Decision direction:

- Define a single authoritative **environment contract** (tools, entrypoints, evidence paths, guardrails).
- Generate provider-native artifacts from it:
  - Coder: `.devcontainer/coder/devcontainer.json` (+ envbuilder build inputs)
  - Che: Devfile/DevWorkspace template (+ commands, `postStart` for task mode only)

Until generation exists, keep drift small by explicitly mapping:

- repo checkout convention
- entrypoint command(s)
- evidence paths
- secrets materialization

## 6) Identity and auth (sharp edge: “login works, RBAC fails”)

Provider differences matter here:

- **Coder:** authorization is mostly Coder-side; Kubernetes RBAC is a provisioning concern for in-workspace Kubernetes access.
- **Che:** the UI/gateway uses Kubernetes authn/authz; if Kubernetes does not accept the OIDC issuer/claims, the dashboard will fail with `401/403`.

Vendor-aligned mitigation when the host managed control plane cannot trust external OIDC:

- Run Che in a vCluster whose API server is configured to trust Keycloak OIDC (see `backlog/680-che-increment-2-vcluster.md`).

Additional sharp edges (provider-specific, but recurrent):

- **oauth2-proxy CSRF cookie collisions:** multiple oauth2-proxy apps sharing `*.dev.ameide.io` can cause “Unable to find a valid CSRF token”. Prefer per-app cookie names (Che must not reuse the default `_oauth2_proxy*` cookie name).
- **OIDC username claim mismatch:** Che may bind RBAC to one identity (display name) while Kubernetes authenticates as another (email) → dashboard `forbidden`. Standardize on `email` end-to-end.
- **Scale-from-zero scheduling vs workspace timeouts:** if `workspaces-che` is autoscaled to `0`, DevWorkspace startup can exceed default progress timeouts. Ensure the DevWorkspace operator `workspace.progressTimeout` is set high enough (e.g. `10–15m`) so workspaces don’t fail before nodes appear.
  - In Che, this is typically controlled via `CheCluster.spec.devEnvironments.startTimeoutSeconds` (which Che uses to reconcile `DevWorkspaceOperatorConfig/devworkspace-config`).

## 7) What “good” looks like (end-to-end)

### 7.1 Human workflow (Coder or Che)

- Open IDE in provider UI.
- Run `ameide test` (Phase 0/1/2) repeatedly during development.
- Push PR; preview environment E2E is merge-gate truth.

### 7.2 Automation workflow (task mode)

- Orchestrator creates ephemeral run (Coder workspace instance or Che DevWorkspace).
- Run a single deterministic command.
- Collect evidence; always cleanup (stop/delete).

## 8) Backlog map (where details live)

- Coder model and tasks:
  - `backlog/650-agentic-coding-overview.md`
  - `backlog/651-agentic-coding-ameide-coding-agent.md`
  - `backlog/652-agentic-coding-dev-workspace-coder.md`
  - `backlog/653-agentic-coding-test-automation-coder.md`
  - `backlog/654-agentic-coding-cli-surface-coder.md`
- Che provider:
  - `backlog/680-che.md`
  - `backlog/680-che-increment-1.md`
  - `backlog/680-che-increment-2-vcluster.md`
- Shared infra topics that directly constrain both providers:
  - `backlog/441-networking.md`
  - `backlog/442-environment-isolation.md`
  - `backlog/430-unified-test-infrastructure-status.md`

## 9) Open questions (to keep explicit)

1. **Canonical “environment contract” spec location:** where the single source of truth lives (and how generation is triggered).
2. **Provider selection UX:** who chooses provider (user, team, orchestrator policy) and how it is recorded for audit.
3. **Phase 3 inner-loop routing scope:** which teams/runs are permitted to create transient `HTTPRoute` objects and in which namespaces.
