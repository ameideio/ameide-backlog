---
title: "680 (Increment 1) — Che baseline + SSO on AKS (ameide)"
status: draft
owners:
  - platform-devx
  - platform
  - gitops
created: 2026-01-15
suite: "agentic-coding"
source_of_truth: false
---

# 680 (Increment 1) — Che baseline + SSO on AKS (ameide)

This increment delivers the smallest “real” Che installation on the shared AKS cluster `ameide`:

- Eclipse Che operator installed (CRDs + controller)
- Che instance (`CheCluster`) reachable on the public dev domain
- SSO via the existing Keycloak (`auth.dev.ameide.io`) works end-to-end

Normative design constraints remain in `backlog/680-che.md` (do not fork contracts).

## Goal

Provide a working, SSO-protected Che UI (VS Code web) in **dev** so teams can validate:

- routing model on Envoy Gateway (Gateway API)
- Keycloak OIDC integration
- baseline operational footprint on AKS

This increment does **not** attempt workspace image parity or agent entrypoints (those are later increments).

## Non-goals

- Devcontainer parity image build + digest pinning
- “Task mode” / Camunda-driven runs
- Codex slot mounting + depletion-aware selection
- PR automation

## Target (AKS ameide / dev)

- Che UI host: `che.dev.ameide.io`
- Workspace/public endpoint host pattern: `*.dev.ameide.io` (dev-only, see Risks)
- OIDC issuer: `https://auth.dev.ameide.io/realms/ameide`

## Vendor prerequisites (make CR-created workspaces usable)

If we create a test workspace via Kubernetes APIs (not via the Che dashboard):

- The `DevWorkspace` MUST include `spec.contributions` for the IDE (or we must prove Che default editor applies to CR-created workspaces).
- The user namespace must already exist (user has logged into Che at least once, or we pre-provision it).

## GitOps delivery plan (authoritative path)

Hard rule: no manual `kubectl apply/patch/delete` to “fix” cloud resources. All mutation is Git → CI → ArgoCD.

### 1) Add GitOps components (dev only)

- `platform-che-operator` (disabled by default; enabled in dev)
  - Wraps vendored `eclipse-che/eclipse-che` chart.
  - Installs CRDs + `Deployment/che-operator` + webhooks.
- `platform-che` (disabled by default; enabled in dev)
  - Creates `CheCluster` (apiVersion `org.eclipse.che/v2`) in `ameide-dev`.
  - Creates `HTTPRoute` to expose `Service/che-gateway:8080` on:
    - `che.dev.ameide.io`
    - `*.dev.ameide.io` (for workspace/public endpoints)
  - Creates `ExternalSecret` to materialize the Che OAuth secret required by Che (`Secret` key `oAuthSecret` + label `app.kubernetes.io/part-of=che.eclipse.org`).

### 2) Add Keycloak OIDC client for Che (dev realm)

Update `platform-keycloak-realm` dev overlay so Keycloak becomes the source of truth for Che auth:

- Add OIDC client `che` with redirect URI `https://che.dev.ameide.io/oauth/callback`
- Extend the Keycloak client-patcher secret-extraction list:
  - `clientId: che`
  - `vaultKey: che-oidc-client-secret`

Che consumes the secret via ExternalSecrets (no GitHub credentials in workspaces).

## Acceptance criteria

- ArgoCD renders two new dev Applications and they converge to `Healthy/Synced`:
  - `dev-platform-che-operator`
  - `dev-platform-che`
- `kubectl -n ameide-dev get checluster eclipse-che` shows `status.chePhase=Active`
- `curl -I https://che.dev.ameide.io/` redirects into the Che OAuth flow and reaches Keycloak:
  - redirects to `https://auth.dev.ameide.io/realms/ameide/...`
- Human can open Che UI and reach VS Code web for a workspace (manual browser validation).

## Verification checklist (CLI)

- Argo applications exist and are Healthy/Synced:
  - `kubectl -n argocd get application | rg 'dev-platform-che'`
- Che operator is running:
  - `kubectl -n ameide-dev get deploy,pod | rg -i 'che-operator'`
- Che gateway service exists:
  - `kubectl -n ameide-dev get svc che-gateway`
- CheCluster is Active:
  - `kubectl -n ameide-dev get checluster eclipse-che -o jsonpath='{.status.chePhase}{"\n"}'`
- SSO smoke:
  - `curl -sSI https://che.dev.ameide.io/ | rg -i 'location:' | head`

## Risks / known sharp edges

- `*.dev.ameide.io` wildcard routing can catch “unknown” dev subdomains and send them to Che. Existing explicit host routes (e.g. `coder.dev.ameide.io`, `platform.dev.ameide.io`) should still win, but this needs to be kept in mind.
- If this becomes problematic, follow-up increment should move workspace/public endpoints to a dedicated base domain (e.g. `*.ws.dev.ameide.io`) which requires both DNS + Gateway listener changes.
