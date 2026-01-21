# Migration remediations (2026-01) — subscription recreate + edge-first

This backlog item tracks **concrete remediations applied during the Azure subscription recreate / cutover**, with a focus on restoring **ArgoCD control-plane availability** and preventing repeat incidents.

Related:
- `backlog/711-azure-subscription-recreate.md`
- `backlog/712-traffic-manager-first-redesign.md`
- `backlog/454-coredns-cluster-scoped.md` (CoreDNS rewrite mechanism)
- `backlog/450-argocd-service-issues-inventory.md` (historical ArgoCD/gateway failure modes)

---

## 1) Incident: ArgoCD login loop (Dex issuer unreachable from argocd-server)

**Date:** 2026-01-21

### Symptoms

- Browser: `https://argocd.ameide.io` loads, but login redirects back to login (session never sticks).
- `argocd-server` logs show repeated invalid-session / token verification failures.

### Root cause

- `kube-system/coredns-custom` rewrites `argocd.ameide.io` → `envoy-argocd.argocd.svc.cluster.local`.
- `Service/argocd/envoy-argocd` existed but had **no endpoints**, because it was selecting a **non-existent** Envoy Gateway in the `argocd` namespace:
  - selector: `gateway.envoyproxy.io/owning-gateway-name=ameide`
  - actual control-plane Gateway is `gateway/cluster` (owning-gateway-name `cluster`)
- `argocd-server` performs OIDC session verification against its own Dex issuer:
  - issuer: `https://argocd.ameide.io/api/dex`
  - in-cluster DNS rewrite sent that traffic to the broken `envoy-argocd` Service (connection refused / issuer lookup failed)
  - result: `invalid session: failed to verify the token` → login loop

### Remediation (GitOps, permanent)

- Fix the `envoy-argocd` alias Service selector to target the **cluster Gateway** pods:
  - `ameide-gitops`: `sources/values/cluster/azure/gateway.yaml`
  - change: `environmentGateways.items[namespace=argocd].gatewayName: ameide → cluster`

### Operational note (one-time)

- ArgoCD needed a **hard refresh** of `Application/cluster-gateway` to pick up the new git revision quickly (`argocd.argoproj.io/refresh=hard`).

### Expected steady-state verification

- `Service/envoy-argocd` has endpoints (selects `gateway.envoyproxy.io/owning-gateway-name=cluster` pods).
- `argocd-server` logs stop showing issuer query failures (`Failed to query provider ... /api/dex/.well-known/openid-configuration`).
- Browser login completes without redirect loops.

