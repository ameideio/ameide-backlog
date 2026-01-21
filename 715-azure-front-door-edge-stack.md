# Azure Front Door (Standard) — edge stack runbook + incident pitfalls (2026-01)

This backlog item documents the **actual Azure Front Door (AFD Standard) configuration** we converged on during the 2026-01 subscription recreate, plus a **pitfall log** of what went wrong and how we fixed it.

Goal: make “edge-first” repeatable and boring:

> Terraform (CI) creates/updates AFD + DNS records + Key Vault cert bindings → clusters are ephemeral origins → canary/cutover is weights/routing (not DNS babysitting).

Related:
- `backlog/437-split-frontend-cdn-evaluation.md`
- `backlog/711-azure-subscription-recreate.md`
- `backlog/714-migration-remediations.md`
- `backlog/468-testing-front-door.md` (testing front door; different “front door” concept)

---

## 1) Current target configuration (what “correct” looks like)

### 1.1 Azure placements (subscriptions / resource groups)

**Edge subscription (AFD + edge Key Vault):**
- Subscription: `e4652c74-2582-4efc-b605-9d9ed2210a2a`
- Resource group: `Ameide`

**DNS subscription (Azure DNS zones/records):**
- Subscription: `92f1f4d0-00d9-4ffb-bce3-f2c3be49d748`
- Resource group: `Ameide`
- Zone: `ameide.io`

**Origin subscription (AKS / Public IPs):**
- Subscription: `e4652c74-2582-4efc-b605-9d9ed2210a2a` (same as edge for now)
- Resource group: `Ameide`

> Note: BYOC for AFD requires the Key Vault to be in the **same subscription** as the AFD profile. This is why the edge Key Vault is in the edge subscription.

### 1.2 Terraform roots and CI workflows

Terraform root:
- `infra/terraform/azure-edge`

CI workflows:
- Plan: `.github/workflows/terraform-azure-edge-plan.yaml`
- Apply: `.github/workflows/terraform-azure-edge-apply.yaml`
- Force-unlock (break-glass): `.github/workflows/terraform-azure-edge-force-unlock.yaml`

Certificate automation (Let’s Encrypt → Key Vault cert objects):
- Wildcard (apex + wildcard): `.github/workflows/certbot-azure-keyvault-renew.yaml` → KV cert name `ameide-io`
- Platform explicit host: `.github/workflows/certbot-platform-cert-to-edge-kv.yaml` → KV cert name `platform-ameide-io`
- ArgoCD explicit host (historical): `.github/workflows/certbot-argocd-cert-to-edge-kv.yaml` → KV cert name `argocd-ameide-io`

### 1.3 AFD profile and endpoints

Profile:
- Name: `ameide-edge`
- SKU: `Standard_AzureFrontDoor`
- Managed identity: user-assigned identity (Front Door → Key Vault access)

Endpoints:
- `ameide-preview` (default `*.azurefd.net` hostname for pre-DNS validation and internal smoke)
- `ameide-prod` (where custom domains are bound)

### 1.4 Custom domains (frontend hostnames)

Custom domains managed:
- `argocd.ameide.io`
- `ameide.io` (apex)
- `platform.ameide.io`
- `www.ameide.io`
- `*.dev.ameide.io` and other explicitly-published `{service}.{env}.ameide.io` hostnames (each as a separate custom domain)

Binding and cert source:
- BYOC is used (Key Vault certificates imported by certbot workflows).

### 1.5 DNS records (Azure DNS)

We deliberately treat apex and subdomains differently:

- **Apex** `ameide.io`:
  - Use **Azure DNS Alias A** to the AFD endpoint resource ID.
  - Reason: apex cannot be a CNAME in public DNS.

- **Subdomain** `platform.ameide.io`:
  - Use **CNAME** to the AFD endpoint hostname (e.g. `ameide-prod-<id>.z02.azurefd.net`).
  - Reason: in practice, apex-style alias records for subdomains created activation ambiguity during the incident; the CNAME path is the most vendor-standard onboarding path for AFD subdomains.

- **Subdomain** `www.ameide.io`:
  - Use **CNAME** to the same AFD endpoint hostname.

- **Subdomain** `argocd.ameide.io`:
  - Use **CNAME** to the same AFD endpoint hostname.

For `{service}.{env}.ameide.io` hostnames (e.g. `auth.dev.ameide.io`):
- Publish only an explicit allow-list (no wildcard `*` records pointing to the cluster).
- Use **Azure DNS Alias A** records targeting the AFD endpoint resource ID (so record type can be updated in-place without switching to CNAME).

### 1.6 Origin groups, origins, routes (AFD → cluster)

All origins point to **cluster Public IPs**, using **HTTP** to origin (AFD Standard baseline).

ArgoCD:
- Origin group: `ameide-argocd`
- Origin: `ameide-argocd` → cluster ArgoCD public IP (port `8080`)
- Route: `ameide-argocd` on prod endpoint
  - `link_to_default_domain = true` (also serves on `*.azurefd.net` for debugging)
  - `customDomains[]` includes `argocd.ameide.io` when enabled

Public sites:
- Origin group: `ameide-public-sites`
- Origins:
  - `ameide-ameide` (host header `ameide.io`)
  - `ameide-platform` (host header `platform.ameide.io`)
  - Both origins point at the **prod Envoy Gateway public IP** (port `8080`)
- Routes:
  - `ameide-io`: bound only to `ameide.io` (`link_to_default_domain=false`)
  - `platform-ameide-io`: bound only to `platform.ameide.io` (`link_to_default_domain=false`)
  - `www-ameide-io`: bound only to `www.ameide.io` (`link_to_default_domain=false`)

Per-environment services:
- For each published hostname (e.g. `auth.dev.ameide.io`), AFD has:
  - a dedicated custom domain
  - a dedicated origin group/origin setting `originHostHeader` to that hostname
  - a dedicated route bound only to that hostname

Preview health:
- Origin group: `ameide-<env>-healthz`
- Route: `ameide-<env>-healthz` on preview endpoint (`link_to_default_domain=true`, no custom domains)

---

## 2) “How we configured it” (declarative contract)

### 2.1 Source of truth

The authoritative intent is:
- Terraform in `infra/terraform/azure-edge` (CI plan/apply)
- Certbot workflows for issuing/importing certs into the edge Key Vault

Manual portal changes are considered *temporary*, and should be adopted/imported back into Terraform state or removed.

### 2.2 Identity and permissions (the minimum that actually worked)

There are two distinct permission planes:

1) **Terraform runner identity (GitHub OIDC)**  
Used by workflows to create/update:
- AFD resources in the edge subscription
- DNS records in the DNS subscription (when `dns_parent_zone_*` vars are set)

2) **Front Door managed identity (user-assigned)**  
Used by AFD at runtime to read cert material from the edge Key Vault:
- Role on Key Vault: `Key Vault Secrets User` (RBAC)

Additionally, if Terraform attaches `dns_zone_id` on custom domains:
- AFD can manage its own domain validation records and may require `DNS Zone Contributor` on the zone.

---

## 3) Pitfall log (what bit us, and how to recognize it fast)

### 3.1 “CN=*.azureedge.net” on a custom domain

**Symptom:** `openssl s_client -servername <custom-host>` shows `CN=*.azureedge.net`.

**Meaning:** AFD is not presenting the BYOC certificate for that hostname. The most common causes:
- the custom domain is not fully activated at the edge yet (propagation delay), or
- the custom domain is not actually bound to a route (route host mismatch → 404), or
- AFD cannot retrieve/use the Key Vault cert (RBAC/firewall/cert format), or
- the cert used by the bound secret does not satisfy AFD’s validation for that hostname.

**Fast checks:**
- Compare `ameide.io` vs `platform.ameide.io` SNI against the same endpoint host.
- Inspect AFD custom domain + route objects (via `az afd … show`) for:
  - `deploymentStatus`
  - route `customDomains[]` includes the expected domain ID

### 3.2 AFD generic 404 (“We weren’t able to find your Azure Front Door Service configuration”)

**Meaning:** request didn’t match a route for that frontend host, or no healthy origin was available.

Common root causes:
- custom domain not associated with the route (host mismatch)
- origin group unhealthy (probe path wrong, port wrong, LB not serving)
- route created but not yet propagated

### 3.3 “It’s approved/succeeded in ARM but still not working on the edge”

AFD has real propagation time. During the incident we saw:
- custom domain provisioning take tens of minutes
- endpoint deployment status being misleading (UI/ARM showing success but edge still serving default cert)

**Operational stance:** treat AFD “activation” as eventually consistent; allow time and keep a deterministic verification script (SNI + curl).

### 3.4 Terraform apply interrupted → state lock and/or orphaned Azure resources

We hit:
- stale terraform state locks after canceling an apply
- partially-created AFD resources existing in Azure but not in TF state

Fixes we adopted:
- an explicit force-unlock workflow (`terraform force-unlock` on `azure-edge.tfstate`)
- “adopt/import” steps in the apply workflow to import pre-existing AFD resources and DNS records

### 3.5 “Not in Front Door” hostnames still work (direct-to-cluster bypass)

If `*.dev.ameide.io` (or any explicit A record) points directly to the cluster ingress Public IP, users can bypass Front Door entirely.

Controls we converged on:
- DNS: remove wildcard `*` records pointing at the cluster; publish only explicit `{service}.{env}` records.
- Network: deny direct Internet to the Envoy ingress ports (at least 80/443/8080) and only allow AFD backend + AzureLoadBalancer.

### 3.5 Let’s Encrypt issuance gotchas

#### Wildcard + explicit subdomain in the same order

Let’s Encrypt rejects requesting `platform.ameide.io` alongside `*.ameide.io` in the same certificate request as “redundant”.

**Resolution:** use two certificates:
- wildcard cert: `ameide.io` + `*.ameide.io`
- explicit host cert: `platform.ameide.io`

#### TXT overwrite during multi-SAN issuance

Certbot can run multiple parallel DNS-01 challenges for the same `_acme-challenge` record name (multiple TXT values). If the automation overwrites the record-set instead of appending values, issuance fails.

**Resolution:** the auth hook (`infra/scripts/certbot-azure-dns-auth.sh`) must preserve existing TXT values and add/remove only its own value.

### 3.6 ArgoCD OIDC login loops caused by edge/DNS inconsistencies

If ArgoCD is configured with issuer `https://argocd.ameide.io/api/dex`, then **argocd-server must be able to resolve and reach that hostname** with a trusted cert.

During the incident, split-horizon rewrites caused issuer discovery/keys fetch to hit the wrong internal service or present an untrusted chain, resulting in `invalid session: failed to verify the token` and browser 401s.

The full remediation timeline is captured in `backlog/714-migration-remediations.md`.

---

## 4) Troubleshooting checklist (fastest path)

For a broken hostname `X`:

1) Verify DNS target:
   - `dig X +short`
   - ensure it resolves to the AFD endpoint (either via CNAME to `*.azurefd.net` or via apex alias A)
2) Verify edge TLS (SNI):
   - `openssl s_client -connect <afd-endpoint-host>:443 -servername X </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -ext subjectAltName`
3) Verify route match:
   - `curl -I https://X/` and inspect if it’s AFD generic 404 vs origin response
4) Verify origin reachability (AFD health):
   - check the probe path exists and returns 200 quickly (`/healthz`)
5) Verify AFD control-plane wiring:
   - custom domain is attached to the route (`customDomains[]` on the route)
   - custom domain uses the expected secret
   - secret points to the correct Key Vault certificate (versionless)

---

## 5) Next hardening items (non-blocking but recommended)

- Origin protection (AFD Standard baseline):
  - network allowlist via `AzureFrontDoor.Backend` service tag where possible
  - instance binding via `X-Azure-FDID` validation
- Standard WAF expectations:
  - Standard is “custom rules only” baseline; managed rule sets require Premium/classic.
- Make edge wiring cover all public hostnames explicitly:
  - add custom domain + route per public hostname (avoid “catch-all” surprises).
