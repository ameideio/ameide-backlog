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
Big picture map (Coder + Che): `backlog/690-agentic-dev-environments-coder-che.md`.

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

- Che UI host (env convention): `che.<env>.ameide.io` (dev = `che.dev.ameide.io`)
- Workspace/public endpoints: not exposed via wildcard in this increment (see Routing Strategy)
- OIDC issuer: `https://auth.dev.ameide.io/realms/ameide`
- Che instance is deployed once per cluster; this increment enables Che in `dev` only.

## Deliberate deviation: Gateway API (Envoy) instead of Ingress

Upstream Che on AKS guidance assumes an Ingress controller (e.g. NGINX Ingress) and Che components that create Kubernetes `Ingress` resources will be routable.

This increment deliberately deviates:

- We expose the Che “front door” (`Service/che-gateway`) via **Gateway API** (`HTTPRoute`) on Envoy Gateway.
- We do **not** install an Ingress controller for Che.
  - Namespace gate: the namespace that contains the Che `HTTPRoute` must satisfy the gateway’s `allowedRoutes.namespaces.selector` (currently `gateway-access=allowed`).

Implication:

- Che UI + IDE flows that stay on `https://che.dev.ameide.io/...` are expected to work.
- DevWorkspace public endpoints that are exposed by creating Kubernetes `Ingress` resources may NOT work until we either:
  - add an Ingress controller, or
  - validate/configure a Che routing strategy that keeps endpoints on the single Che host (no per-endpoint Ingress), or
  - extend the Gateway listeners/HTTPRoutes in a way that does not hijack unrelated `*.dev.ameide.io` hostnames.

## Vendor prerequisites (make CR-created workspaces usable)

If we create a test workspace via Kubernetes APIs (not via the Che dashboard):

- The `DevWorkspace` MUST include `spec.contributions` for the IDE (or we must prove Che default editor applies to CR-created workspaces).
- The user namespace must already exist (user has logged into Che at least once, or we pre-provision it).

## Vendor prerequisite: stable Kubernetes username claim (avoid 403 after login)

Che’s gateway enforces authorization via Kubernetes RBAC. If Kubernetes authenticates the OIDC token into a different username string than the one Che used to create per-user RoleBindings, the dashboard will show `403 Forbidden` for DevWorkspaces, user profile, pods, etc.

For vendor-aligned portability, prefer:

- Kubernetes API server username claim = `email` (and disable username prefixing), and
- Che server property `CHE_OIDC_USERNAME__CLAIM=email`

Increment 2 (`backlog/680-che-increment-2-vcluster.md`) applies this pattern in the vCluster deployment.

## GitOps delivery plan (authoritative path)

Hard rule: no manual `kubectl apply/patch/delete` to “fix” cloud resources. All mutation is Git → CI → ArgoCD.

### 1) Add GitOps components (dev only)

- `platform-che-operator` (disabled by default; enabled in dev)
  - Wraps vendored `eclipse-che/eclipse-che` chart.
  - Installs CRDs + `Deployment/che-operator` + webhooks.
- `platform-che` (disabled by default; enabled in dev)
  - Creates `CheCluster` (apiVersion `org.eclipse.che/v2`) in `ameide-dev`.
  - Creates `HTTPRoute` to expose `Service/che-gateway:8080` on `che.dev.ameide.io`.
  - Creates `ExternalSecret` to materialize the Che OAuth secret required by Che:
    - `CheCluster.spec.networking.auth.oAuthSecret` is set to the Secret *name* (not the secret value).
    - The referenced Secret contains key `oAuthSecret` and is labeled `app.kubernetes.io/part-of=che.eclipse.org`.

### 1.1 CheCluster minimum auth config (explicit)

This increment MUST explicitly configure these `CheCluster` fields (dev values shown):

- `spec.networking.hostname: che.dev.ameide.io`
- `spec.networking.domain: dev.ameide.io` (workspace base domain; may evolve with routing strategy)
- `spec.networking.auth.identityProviderURL: https://auth.dev.ameide.io/realms/ameide`
- `spec.networking.auth.oAuthClientName: che`
- `spec.networking.auth.oAuthSecret: che-oauth-secret` (Secret name; value is stored in the Secret, not in the CR)

Note: TLS is terminated at Envoy Gateway for `che.dev.ameide.io` in this increment; we do not require Che to manage per-Ingress TLS secrets.

#### Failure mode (observed): Keycloak `Invalid parameter: redirect_uri`

If `spec.networking.hostname` is omitted, Che’s gateway oauth2-proxy can derive `redirect_url` from `domain`, leading to callbacks like:

- `https://dev.ameide.io/oauth/callback` (wrong host)

Keycloak will reject the auth request with:

- `400` “Invalid parameter: redirect_uri”

Fix is GitOps-only: set `spec.networking.hostname` to the actual Che host (`che.dev.ameide.io`) so the oauth2-proxy `redirect_url` matches the Keycloak client allowlist.

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
- DevWorkspace operator is present (Che dependency):
  - `kubectl get crd | rg -i '^devworkspaces\\.workspace\\.devfile\\.io'`
- Che gateway service exists:
  - `kubectl -n ameide-dev get svc che-gateway`
- CheCluster is Active:
  - `kubectl -n ameide-dev get checluster eclipse-che -o jsonpath='{.status.chePhase}{"\n"}'`
- SSO smoke:
  - `curl -sSI https://che.dev.ameide.io/ | rg -i 'location:' | head`

## Risks / known sharp edges

- Workspace/public endpoints may be exposed by Che via Kubernetes `Ingress` resources; those will not be reachable without an Ingress controller or a validated single-host routing strategy.
