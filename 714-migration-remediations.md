# Migration remediations (2026-01) — subscription recreate + edge-first

This backlog item tracks **concrete remediations applied during the Azure subscription recreate / cutover**, with a focus on restoring **ArgoCD control-plane availability** and preventing repeat incidents.

Related:
- `backlog/711-azure-subscription-recreate.md`
- `backlog/715-azure-front-door-edge-stack.md`
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

---

## 3) Incident: ArgoCD login loop (Dex issuer discovery 404 due to `/api/.well-known` path)

**Date:** 2026-01-21

### Symptoms

- ArgoCD UI loads, but API calls return 401 and the UI “bounces” back to login.
- Browser console shows:
  - `401 (Unauthorized)` on `/api/v1/applications`, `/api/v1/clusters`
  - `invalid session: failed to verify the token`
- `argocd-server` logs show repeated:
  - `Failed to verify token ... failed to query provider "https://argocd.ameide.io/api/dex": 404 Not Found`

### Root cause

- Dex is served at a non-root base path:
  - in-cluster: `https://argocd-dex-server:5556/api/dex/*` returns 200
  - in-cluster: `https://argocd-dex-server:5556/*` returns 404
- However, ArgoCD was configured to talk to Dex at the root:
  - `ConfigMap/argocd-cmd-params-cm`: `server.dex.server=https://argocd-dex-server:5556`
- During session verification, ArgoCD initializes an OIDC provider for issuer `https://argocd.ameide.io/api/dex` and must fetch discovery + JWKS. With `server.dex.server` pointing at the wrong base, provider init/lookup produced 404s, so ArgoCD rejected the session:
  - `invalid session: failed to verify the token`

### Remediation (GitOps, permanent)

Make ArgoCD use the Dex server at the correct base path by setting the effective cmd param:

- Set `configs.params.server.dex.server=https://argocd-dex-server:5556/api/dex` in the **effective** `configs.params` block (the only block that the Helm chart renders into `argocd-cmd-params-cm`).
- Implementation in `ameide-gitops`:
  - `sources/values/common/argocd.yaml`
- Deploy via CI:
  - `Terraform Azure Apply + Verify` workflow (which runs `bootstrap/bootstrap.sh --install-argo` to Helm-upgrade ArgoCD)

### Note (mitigation / compatibility)

HTTPRoute rewrites were added to normalize some nonstandard discovery/JWKS path variants observed during the incident. With the cmd-params fix in place, these should no longer be required for ArgoCD itself; consider removing them once traffic confirms no dependencies remain.

### Expected steady-state verification

- `ConfigMap/argocd-cmd-params-cm` contains:
  - `server.dex.server=https://argocd-dex-server:5556/api/dex`
- From inside the cluster:
  - `curl -k https://argocd-dex-server:5556/api/dex/.well-known/openid-configuration` returns 200
  - `curl -k https://argocd-dex-server:5556/.well-known/openid-configuration` returns 404
- `argocd-server` logs stop emitting `failed to query provider ... 404 Not Found` and `invalid session: failed to verify the token`.
- Automated check passes:
  - `infra/scripts/verify-argocd-sso.sh --azure` prints `OK   [azure] ArgoCD SSO logged in as: admin@ameide.io`.

---

## 4) Incident: `platform.ameide.io` serves `CN=*.azureedge.net` + AFD 404 (custom domain not activating)

**Date:** 2026-01-21

### Symptoms

- `https://ameide.io/` works and serves the Let’s Encrypt cert (BYOC via Key Vault).
- `https://platform.ameide.io/` resolves to the same Front Door endpoint host, but:
  - TLS presents `CN=*.azureedge.net` (default Front Door cert)
  - Front Door serves the generic 404 page (“We weren’t able to find your Azure Front Door Service configuration”)

### Root cause (most likely)

- The BYOC certificate imported into Key Vault was issued only for:
  - `ameide.io`
  - `*.ameide.io` (wildcard)
- Azure Front Door’s custom-domain binding / secret validation appears to require an **explicit SAN match** for subdomains in some cases (wildcard SAN is not treated as sufficient for binding/activation).
  - Evidence: the Front Door secret’s `subjectAlternativeNames` contained `ameide.io` and `*.ameide.io` but not `platform.ameide.io`, and only `ameide.io` activated.

### Remediation (GitOps, permanent)

- Attempted to re-issue the wildcard cert with an explicit `platform.ameide.io` SAN, but Let’s Encrypt rejects this as redundant when `*.ameide.io` is present in the same order.
- Final approach:
  - Keep the wildcard cert as `ameide.io` + `*.ameide.io` (covers the broad set of subdomains).
  - Mint a **separate**, explicit single-host cert for `platform.ameide.io`.
  - Bind `platform.ameide.io` to a dedicated Front Door secret backed by that platform cert.
- Implementation:
  - `ameide-gitops`: `.github/workflows/certbot-platform-cert-to-edge-kv.yaml` (mints/imports Key Vault cert `platform-ameide-io`)
  - `ameide-gitops`: `infra/terraform/azure-edge/main.tf` (platform custom domain uses the dedicated secret)

### Expected steady-state verification

- `openssl s_client -connect <afd-endpoint>:443 -servername platform.ameide.io` presents the Let’s Encrypt cert (not `*.azureedge.net`).
- `curl -I https://platform.ameide.io/` returns a non-AFD-generic response and reaches the origin route.

---

## 5) Remediation: Terraform state adoption/import (DNS + identities) to restore deterministic applies

**Date:** 2026-01-21

### Why this mattered

During the migration we had pre-existing Azure resources (especially DNS recordsets) that were **not in Terraform state**, causing apply failures like “already exists” and leaving us unable to converge changes through CI.

### Remediation

- Use `.github/workflows/terraform-azure-adopt.yaml` (`workflow_dispatch`, confirm `adopt-azure`) to import known-existing resources into the `azure.tfstate` backend before applying changes.
- Fix a failure mode where the adopt job aborted under `set -euo pipefail` when the `apps` nodepool state entry did not exist yet.

### Evidence

- Successful adopt/import + plan run on branch `fix/split-horizon-private-dns-ignore-disable-dns`: GitHub Actions run `21218845045`.
