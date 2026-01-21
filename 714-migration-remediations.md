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

---

## 2) Incident: ArgoCD login loop (Dex issuer TLS “unknown authority” in-cluster)

**Date:** 2026-01-21

### Symptoms

- `argocd-server` logs show token verification failures when calling the issuer discovery endpoint:
  - `tls: failed to verify certificate: x509: certificate signed by unknown authority`
  - issuer: `https://argocd.ameide.io/api/dex`
- Browser symptom presents as a login loop (session never stabilizes).

### Root cause

- ArgoCD’s OIDC issuer URL is **public** (`https://argocd.ameide.io/api/dex`), but in-cluster DNS resolution was not stable:
  - `kube-system/coredns-custom` has a production catch-all rewrite:
    - `rewrite name regex (.+\\.)?ameide\\.io envoy.ameide-prod.svc.cluster.local`
  - with Kubernetes resolver defaults (`ndots:5`), clients may attempt search-suffix variants first, so `argocd.ameide.io` could be captured by the catch-all rule even if an exact match rule exists.
- The in-cluster target (Gateway/Envoy path) presented a cert chain that the client did not trust (self-signed / not in system roots), so issuer discovery failed and ArgoCD rejected the session.

### Remediation (GitOps, permanent)

Make in-cluster resolution of the issuer hostname `argocd.ameide.io` converge on the same endpoint as external clients:

- Add a CoreDNS rewrite that maps `argocd.ameide.io` to the Azure Front Door endpoint hostname (publicly trusted cert).
- Make the rewrite resilient to resolver search suffixes (`ndots`) so it wins before the catch-all rewrite.
- Implementation in `ameide-gitops`:
  - `sources/values/cluster/azure/coredns-config.yaml`

### Expected steady-state verification

- From inside the cluster, `argocd.ameide.io` resolves to Front Door (not `envoy-*` ClusterIPs).
- From inside the cluster, `curl https://argocd.ameide.io/api/dex/.well-known/openid-configuration` succeeds without `-k`.
- `argocd-server` logs stop showing `x509: certificate signed by unknown authority` for issuer discovery.
