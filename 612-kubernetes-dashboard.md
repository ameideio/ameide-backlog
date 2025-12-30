# 612 – Kubernetes Dashboard (GitOps + SSO + local auto-login)

**Status**: Implemented (local), ready for promotion (dev/staging/production)  
**Owner**: Platform  
**Created**: 2025-12-30  
**Related**: `450-argocd-service-issues-inventory.md`, `485-keycloak-oidc-client-reconciliation.md`, `487-keycloak-client-secret-rotation.md`

---

## 1. Problem

We want a Kubernetes UI surface that:

1. Is deployed declaratively via Argo CD (no `kubectl apply` / manual install steps).
2. Is protected behind SSO (OIDC via Keycloak).
3. Supports a good local developer experience: “hit URL → see Dashboard”.

Upstream Kubernetes Dashboard expects Kubernetes credentials (bearer token) and does **not** natively consume OIDC identity. That creates a UX gap between “SSO gates the HTTP endpoint” and “Dashboard is authenticated to Kubernetes”.

---

## 2. Goals

- Install upstream `kubernetes-dashboard` via GitOps in each environment.
- Expose it through the platform Gateway (Gateway API `HTTPRoute`), not Ingress.
- Enforce login at the edge with `oauth2-proxy` + Keycloak (OIDC).
- Provide a clean local path to “hit URL → see Dashboard” without manual token pasting.

---

## 3. Non-goals / constraints

- Per-user Kubernetes RBAC and per-user audit attribution are **not** solved by Dashboard alone.
- AKS end-to-end verification may be blocked from devcontainers due to control-plane network/DNS constraints.

---

## 4. High-level architecture

### 4.1 Common (all envs)

```
Browser
  → Platform Gateway (Envoy Gateway)
    → Service/kubernetes-dashboard-oauth2-proxy
      → (upstream) Dashboard Kong proxy
        → Dashboard API/Auth/Web
```

`oauth2-proxy` enforces Keycloak SSO. Dashboard itself still expects Kubernetes authentication via bearer token.

### 4.2 Local (developer UX: auto-login)

Local adds a token-injecting hop behind `oauth2-proxy`:

```
Browser
  → Gateway
    → oauth2-proxy (SSO)
      → token-proxy (Envoy + Lua)
        → dashboard-kong-proxy
```

`token-proxy` injects a projected ServiceAccount token on every request:

- Kubernetes identity: `ServiceAccount/kubernetes-dashboard-viewer`
- RBAC: `ClusterRole/view` (cluster-scoped read-only)
- Result: once SSO is complete, the Dashboard “just opens”.

This is a **shared identity** model (everyone who passes SSO shares the same Kubernetes identity).

---

## 5. GitOps implementation

### 5.1 Components (Argo CD)

Component definitions added for both `_shared` and `local` generators:

- `environments/_shared/components/platform/infrastructure/kubernetes-dashboard/component.yaml`
- `environments/_shared/components/platform/infrastructure/kubernetes-dashboard-oauth2-proxy/component.yaml`
- `environments/local/components/platform/infrastructure/kubernetes-dashboard/component.yaml`
- `environments/local/components/platform/infrastructure/kubernetes-dashboard-oauth2-proxy/component.yaml`

### 5.2 Values (Helm)

Dashboard:

- Shared: `sources/values/_shared/platform/kubernetes-dashboard.yaml`
- Env overlays:
  - `sources/values/env/local/platform/kubernetes-dashboard.yaml`
  - `sources/values/env/dev/platform/kubernetes-dashboard.yaml`
  - `sources/values/env/staging/platform/kubernetes-dashboard.yaml`
  - `sources/values/env/production/platform/kubernetes-dashboard.yaml`

Notes:
- `kong.proxy.http.enabled=true` (cluster-internal HTTP so proxies can upstream without TLS complexity).
- Local overlay creates `kubernetes-dashboard-viewer` + `ClusterRoleBinding` and installs the local `token-proxy` (Envoy+Lua).

oauth2-proxy:

- Shared: `sources/values/_shared/platform/kubernetes-dashboard-oauth2-proxy.yaml`
- Env overlays:
  - `sources/values/env/local/platform/kubernetes-dashboard-oauth2-proxy.yaml`
  - `sources/values/env/dev/platform/kubernetes-dashboard-oauth2-proxy.yaml`
  - `sources/values/env/staging/platform/kubernetes-dashboard-oauth2-proxy.yaml`
  - `sources/values/env/production/platform/kubernetes-dashboard-oauth2-proxy.yaml`

Notes:
- Secrets come from Vault via ESO `ExternalSecret` (client secret + cookie secret).
- Local uses in-cluster Keycloak issuer URL to avoid hairpin/public-IP issues (`http://keycloak.<ns>.svc:8080/realms/ameide`), with issuer verification skipped for local convenience.

### 5.3 Gateway route

Dashboard is exposed via `extraHttpRoutes` in the platform gateway values:

- `sources/values/env/local/platform/platform-gateway.yaml`
- `sources/values/env/dev/platform/platform-gateway.yaml`
- `sources/values/env/staging/platform/platform-gateway.yaml`
- `sources/values/env/production/platform/platform-gateway.yaml`

Backend is `Service/kubernetes-dashboard-oauth2-proxy` on port `80`.

---

## 6. Access model (what users do)

### 6.1 Local

- Open: `https://k8s-dashboard.local.ameide.io/`
- Complete Keycloak login
- Dashboard opens immediately (shared read-only identity)

If you need a raw Kubernetes token for debugging, use:

- `scripts/local/k8s-dashboard-token.sh`

### 6.2 Dev / Staging / Production

Default posture remains:

- SSO gates access (Keycloak via oauth2-proxy)
- Dashboard still requires a Kubernetes bearer token (manual paste)

This preserves least-surprise vendor behavior for non-local clusters.

---

## 7. Security notes

Local auto-login is a deliberate tradeoff:

- ✅ Good UX for local/dev
- ✅ No token distribution to humans
- ⚠️ Shared Kubernetes identity (no per-user RBAC/audit)
- ⚠️ If the SSO gate is bypassed, Dashboard access becomes “whoever can reach it gets `view`”

Mitigations:

- Keep token-injection local-only.
- Use `ClusterRole/view` (not admin).
- Keep the service internal to the cluster and only reachable via the Gateway route protected by oauth2-proxy.

---

## 8. Test evidence (local)

Verified in local cluster:

- `HTTPRoute/k8s-dashboard` exists and routes to `kubernetes-dashboard-oauth2-proxy`
- `oauth2-proxy` initiates login (302 to Keycloak)
- `token-proxy` returns `200` on `GET /api/v1/me` without client-supplied `Authorization` header

---

## 9. Follow-ups

1. Decide whether non-local clusters should stay “manual token” or move to a per-user Kubernetes auth model.
2. If we want per-user RBAC, plan for Kubernetes API server OIDC integration (or a different UI surface that supports OIDC natively).
3. If we keep shared-identity auto-login, add guardrails:
   - stricter allowed email/groups at `oauth2-proxy`
   - private-only exposure for AKS (internal LB + private DNS)
   - monitoring for `oauth2-proxy` failures and unauthorized Dashboard hits

