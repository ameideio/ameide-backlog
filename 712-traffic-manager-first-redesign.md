# Traffic-manager-first redesign (portable, no DNS babysitting)

## Core idea

Make a **persistent global edge (“Traffic Manager”) the first-class, always-on entrypoint** for every environment. Clusters become **ephemeral origins** behind that edge.

## Status (2026-01-20)

Implemented in this repo (ready for CI plan/apply):
- Edge stack Terraform root exists (`infra/terraform/azure-edge`) and can manage:
  - AFD Standard profile + `preview`/`prod` endpoints (`*.azurefd.net`)
  - ArgoCD route to an origin HTTP listener (`origin-http` on port `8080`)
  - Optional ArgoCD custom domain cutover + BYOC TLS from an edge Key Vault
- Origin hardening for `origin-http:8080` is available as layered controls:
  - Network: NSG allowlist `AzureFrontDoor.Backend` and deny `Internet` on `8080` (Terraform-managed in the cluster stack).
  - App (optional): Envoy Gateway `SecurityPolicy` can enforce `X-Azure-FDID == edge.azure.frontDoorId` on the origin listener once edge runtime facts are published.
- Certbot-based Let’s Encrypt issuance to the edge Key Vault is automated in CI.

Pending execution (CI runs):
- Apply edge stack (no DNS cutover yet).
- Issue/import the Let’s Encrypt cert into the edge Key Vault.
- Cut over `argocd.<DNS_PARENT_ZONE_NAME>` to AFD + BYOC (TXT validation + CNAME).

Once the edge is in place, **DNS and public TLS stop being part of cluster bootstrap**. Cluster recreation/canary/cutover becomes:

> create new cluster → register as new origin (weight=0) → validate → shift weights → delete old origin/cluster

— **without repeatedly touching DNS zones/records**.

This directly eliminates the current failure chain observed during subscription recreate (see `backlog/711-azure-subscription-recreate.md`):

> DNS-01 Issuers stall → wildcard secret missing → HTTPS listeners invalid → routes degraded

Because **the cluster no longer depends on ACME DNS-01 for public ingress**.

---

## 1) Target architecture (conceptual)

```
Clients
  |
  |  (custom domain DNS points here once, then stays stable)
  v
[ Global Edge / Traffic Manager ]
  - terminates public TLS (certs live here)
  - health checks & weighted routing
  - optional WAF
  |
  |  (private preferred, public possible with restrictions)
  v
[ Cluster Gateway Origin (HTTP or origin-TLS) ]
  - Envoy Gateway (Gateway API)
  - routes to in-cluster Services
```

### Stable vs ephemeral

**Stable (persistent):**
- Custom domains + DNS records (point to edge)
- Public TLS certs for those domains (on edge)
- Edge routing policy (origins, weights, health probes, rules)

**Ephemeral (replaceable anytime):**
- AKS/EKS/GKE clusters + node pools
- In-cluster Gateways/HTTPRoutes/apps
- The gateway *origin endpoint* (one per cluster)

---

## 2) Concrete cloud mappings (portable pattern, different implementations)

### Azure (default for now): Azure Front Door Standard

- Weighted routing to origins is built-in (origin group weights).
- TLS on the edge supports Azure-managed certs or customer-managed certs (BYOC via Key Vault).

**Non-goal (for now):** AFD Premium features (e.g. Private Link origins). Keep the design compatible, but assume **Standard**.

**Placement decision (Azure):** the edge stack (AFD + BYOC Key Vault) lives in the **same subscription + resource group as the cluster** (RG `Ameide`).

Parent DNS zones can remain in a separate “DNS subscription/RG”; the edge stack manages DNS records via a dedicated Terraform provider alias.

Concrete mapping to repo variables (Azure):
- “Cluster/edge subscription” = `AZURE_SUBSCRIPTION_ID`
- “Cluster/edge RG” = `Ameide`
- “DNS subscription” = `DNS_PARENT_ZONE_SUBSCRIPTION_ID`
- “DNS RG” = `DNS_PARENT_ZONE_RESOURCE_GROUP`

Important vendor constraint: Azure Front Door requires the Key Vault used for BYOC to be in the **same subscription** as the Front Door profile (so BYOC Key Vault placement follows the edge stack).

### AWS: Global Accelerator (steering) + regional TLS terminator

AWS has two viable “traffic-manager-first” shapes; they are not equivalent:

- **CloudFront (L7 edge)**: best match for an “origin header” contract (can inject origin custom headers); can also be combined with origin allowlists using AWS-managed prefix lists.
- **Global Accelerator (L4 steering)**: good for weights/traffic dials, but cannot inject L7 headers. If we keep “header enforcement” as an invariant, CloudFront is the closer mapping.

### GCP: Global external Application Load Balancer

- Global external ALB supports weighted backends and can add custom request headers to origins.
- Internet NEGs enable “origins are endpoints” portability (hybrid/external backends).

---

## 3) New contract: “clusters never own public DNS/TLS”

### What we stop doing inside clusters (public ingress path)

- **Default path** (cloud): clusters do not need to mint public certificates to serve public traffic.
- Concretely, the baseline should not require:
  - ACME DNS-01 Issuers/Certificates for public hostnames to be present *at bootstrap time*
  - Wildcard secrets like `ameide-wildcard-tls` to exist for the platform to be “Healthy”
  - “Phase gating” of app HTTPRoutes just to keep Argo Healthy during bootstrap

### What we keep doing inside clusters

- Envoy Gateway + Gateway API routing
- A stable origin endpoint per environment/cluster (LB IP/hostname)
- Optional origin TLS (see below)

---

## 4) Origin security modes (pick one baseline; keep others as upgrades)

**Default baseline (portable, AFD Standard-compatible): Mode B — public origin with hard restrictions**

With Azure Front Door **Standard** (no Private Link origins), assume the origin is reachable over the public Internet. The design must prevent “direct-to-origin bypass”.

Baseline contract:
- **Network allowlist**: restrict origin ingress to Front Door backend IP space using the `AzureFrontDoor.Backend` service tag (no hardcoded IPs).
- **Instance binding**: validate the `X-Azure-FDID` header value so only *our* Front Door profile can reach the origin (service tag alone is insufficient).
- Optional defense-in-depth: inject a shared secret header (rule set) and enforce it at the origin.

WAF note for Standard: assume **custom rules** only; managed rule sets are a Premium capability. Keep this explicit so we don’t accidentally promise “OWASP managed rules” on Standard.

Weights note: weighted routing is probabilistic; at low request rates, weight distributions can look skewed.
Operational note: Azure Front Door origin weights are typically configured in a 1–1000 range; use larger gaps (e.g. 0→10→100→1000) when you want clearer intent during canary.

Rules note: rule sets can inject custom headers to origins; do not assume you can overwrite reserved Front Door headers.

**Upgrade (future, not now): Mode A — private origin**

- Edge reaches cluster over private connectivity (cloud-specific):
  - Azure: AFD Premium + Private Link to ILB (explicitly not in the “Standard for now” scope)
  - AWS/GCP: equivalents are possible but more bespoke

**Upgrade: Mode C — re-encrypt to origin (origin TLS)**

- Edge terminates public TLS and connects to origin using HTTPS.
- This improves transport security for the edge→origin hop, but must be compatible with AFD’s origin TLS validation model (do not assume custom CAs are supported).

---

## 5) GitOps / Terraform workflow redesign (end-to-end)

## Terraform CI drives all changes (non-negotiable)

This redesign must follow the repo GitOps rules:

- All infrastructure changes happen via **Git + Terraform in CI** (plan/apply workflows).
- ArgoCD reconciles cluster state from Git (plus runtime facts).
- No “fixes” in cloud portals/CLIs; no `kubectl apply/patch/delete` against shared cloud resources.

For Azure today, this means we continue to use the existing workflows:
- `.github/workflows/terraform-azure-plan.yaml`
- `.github/workflows/terraform-azure-apply.yaml`

And extend the Terraform code so that CI can manage the **edge stack** as well as the **cluster stack**, with the same plan→apply safety gates.

### Split into two first-class stacks

1) **Edge stack (persistent)**
- Global edge resource (AFD / GA(+ALB/NLB) / GCP global ALB)
- Custom domains + TLS binding (BYOC)
- Routing policy: origin groups, health probes, weights

2) **Cluster stack (ephemeral)**
- AKS/EKS/GKE + platform resources (Terraform + ArgoCD)
- Gateway origin endpoint (public LB or internal LB)
- Publish origin endpoint as a runtime fact / Terraform output

### Runtime facts (shifted role)

Facts now drive:
- “What is the origin endpoint?” (IP/FQDN) per environment/cluster
- Optional metadata (“origin mode”: public/private, origin port, etc.)

Facts should no longer be required just to decide whether public DNS/TLS exists.

---

## 6) Phases (what CI runs; no manual work)

### Phase 0 — edge bootstrap (zero DNS change)

Create edge with provider default hostname:
- Azure: `*.azurefd.net`
- AWS: GA static IPs
- GCP: LB IP / default hostname

Expose a **preview endpoint** that can target a chosen origin without requiring any DNS changes.

### Phase 1 — create new cluster (no DNS touch)

- Provision cluster + platform via Terraform/Argo
- Publish origin endpoint (runtime facts)
- Register origin in edge as **weight=0** for production route
- Register origin as **100%** behind preview route (for validation)

### Phase 2 — canary & cutover (no DNS record churn)

Shift edge weights:

> 0% → 1% → 10% → 50% → 100%

Rollback is simply shifting weights back.

### Phase 3 — decommission old cluster

- Remove old origin from edge
- Destroy old cluster resources

### Phase 0.5 — wire preview routing (the next safe step)

Objective: route `preview` (`*.azurefd.net`) to the **new cluster** origin without touching DNS/custom domains.

Implementation (Terraform in CI):
- Extend `infra/terraform/azure-edge` to create:
  - `azurerm_cdn_frontdoor_origin_group` (health probe `/healthz`, protocol HTTP, port `8080`)
  - `azurerm_cdn_frontdoor_origin` pointing at the **new cluster** gateway public IP/FQDN
  - `azurerm_cdn_frontdoor_route` on the `preview` endpoint to forward traffic to the origin group
- Keep “prod” endpoint unbound to custom domains (no DNS) and with no production routes yet.

Validation (read-only):
- `curl -fsS "https://<frontdoor preview host>/healthz"`

---

## 7) Certificates (Let’s Encrypt while keeping clusters DNS-independent)

Goal: **public certs live at the edge** (not in the cluster), but still issued by Let’s Encrypt, with automation.

### Pattern (portable)

- A GitHub Actions workflow issues/renews an ACME certificate via **DNS-01**.
- The certificate is uploaded into the cloud certificate store used by the edge:
  - Azure: Key Vault (Front Door BYOC)
  - AWS: ACM (for ALB/NLB/CloudFront depending on implementation)
  - GCP: Certificate Manager (for global external ALB)

This is the one unavoidable “DNS touch” when using Let’s Encrypt: `_acme-challenge` TXT records, fully automated via CI.

### Azure v1 implementation: certbot → Key Vault (BYOC) → Front Door uses latest

For Azure Front Door BYOC, the certificate must be present in Key Vault and meet Front Door certificate requirements (notably: PFX/PKCS12 and no EC key algorithms).

We implement issuance/renewal as a CI workflow that:

1) Uses `certbot` to issue/renew via DNS-01 against the **parent DNS zone subscription/RG**.
2) Packages the result as a PFX (full chain included).
3) Imports it into the **edge Key Vault** (same subscription as the AFD profile) via `az keyvault certificate import`.
4) Configures Front Door to use **Latest** certificate version so renewals are picked up without manual re-wiring.

Implementation artifacts in this repo:
- Workflow: `.github/workflows/certbot-argocd-cert-to-edge-kv.yaml`
- Hooks:
  - `infra/scripts/certbot/azuredns-auth-hook.sh`
  - `infra/scripts/certbot/azuredns-cleanup-hook.sh`

Required GitHub repo variables (Azure):
- `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`
- `DNS_PARENT_ZONE_NAME`, `DNS_PARENT_ZONE_SUBSCRIPTION_ID`, `DNS_PARENT_ZONE_RESOURCE_GROUP`

Minimal command shape (CI):

```bash
certbot certonly \
  --manual \
  --preferred-challenges dns \
  --manual-auth-hook infra/scripts/certbot/azuredns-auth-hook.sh \
  --manual-cleanup-hook infra/scripts/certbot/azuredns-cleanup-hook.sh \
  --non-interactive --agree-tos \
  --email "${LE_EMAIL}" \
  -d "argocd.ameide.io"
```

The auth/cleanup hooks use `az network dns record-set txt ...` to create and remove the `_acme-challenge` TXT record in Azure DNS.

### Origin protection and health probes (operational detail)

Front Door health probes must keep working even when origin protection is enabled. Ensure the chosen probe path (e.g. `/healthz`) is compatible with the `X-Azure-FDID` enforcement and does not get blocked by policy.

### Keep the current cluster-side Let’s Encrypt configuration (don’t delete it)

We should **not remove** the existing cert-manager DNS-01 / Workload Identity design (e.g. `backlog/438-cert-manager-dns01-azure-workload-identity.md` / `backlog/448-cert-manager-workload-identity.md`).

Instead:
- Treat it as a **supported fallback / future option** for “direct-to-cluster public ingress” (without an edge), or for origin TLS where it makes sense.
- Ensure the edge-first baseline does not break or invalidate those manifests; it simply makes them **non-critical** for cluster bring-up and canary.

---

## 8) Relationship to existing backlog items

- `backlog/711-azure-subscription-recreate.md`: this redesign makes “no DNS babysitting” architectural (clusters can be recreated and tested via edge preview endpoints without any production DNS changes).
- `backlog/438-cert-manager-dns01-azure-workload-identity.md` and `backlog/448-cert-manager-workload-identity.md`: remain valid for DNS-01 where needed, but public ingress no longer *depends* on cluster-managed DNS-01.
- `backlog/441-networking.md` and `backlog/450-envoy-gateway-per-environment-architecture.md`: still define the in-cluster Gateway API model; the key change is that **public TLS and DNS stability are owned by the edge**, not the environment Gateway.

---

## 9) Migration plan (from current state)

This migration plan is written for **Azure Front Door Standard** and assumes Terraform CI drives all steps.

### Step 1 — add an “edge stack” Terraform root (persistent)

Create a dedicated Terraform root module for the edge stack (separate state from cluster lifecycle):

- Location/ownership:
  - Create/manage the edge resources in the **cluster/edge subscription** and RG `Ameide`.
  - Manage parent DNS records (TXT/CNAME for the custom domain) via a DNS provider alias targeting `DNS_PARENT_ZONE_SUBSCRIPTION_ID` / `DNS_PARENT_ZONE_RESOURCE_GROUP`.
- AFD profile + endpoints:
  - **Production endpoint**: stable, later bound to custom domains
  - **Preview endpoint**: stable `*.azurefd.net` hostname used for validation without DNS changes
- Origin groups per environment (dev/staging/production), each with:
  - Health probe (HTTP) and load-balancing settings
  - Weighted origins (one origin per cluster)

Acceptance:
- Terraform plan shows only edge resources (no cluster resources).
- The preview endpoint exists and is reachable before any DNS changes.

### Step 2 — define the origin contract (cluster side)

Change the in-cluster Gateway model so that edge traffic never depends on wildcard public TLS:

- Add a dedicated **origin listener** on the environment Gateway:
  - Protocol: HTTP (baseline)
  - Port: 8080
  - Not coupled to any wildcard secret like `ameide-wildcard-tls`
- Expose a cheap probe path (e.g. `/healthz`) that returns 200 quickly (Front Door probes are frequent and distributed).
- Ensure the edge-facing routes attach to that origin listener:
  - Either route templates target `sectionName: origin-http`
  - Or app charts render both `https` (direct) and `origin-http` (edge) routes where needed

Acceptance:
- AFD can reach the cluster via the origin listener even when wildcard cert issuance is disabled or pending.
- ArgoCD health is no longer coupled to public wildcard certificate issuance.

### Step 3 — implement origin protection (edge header + in-cluster enforcement)

Prevent direct access to the origin LB bypassing Front Door.

Decisions:
- **Network allowlist** in Terraform (cluster stack): allow inbound only from the `AzureFrontDoor.Backend` service tag (plus required Azure infra IPs).
- **Instance binding** in cluster config: enforce `X-Azure-FDID == <our-frontdoor-id>`.
- Optional defense-in-depth:
  - Generate a shared-secret origin token in Terraform (edge stack).
  - Store it in the edge Key Vault (edge stack).
  - Sync it into the cluster via ExternalSecrets (cluster stack).
  - Configure AFD rule set to inject the header (edge stack).
  - Enforce it at the Gateway layer (cluster stack).

Acceptance:
- Requests reaching the origin without the expected Front Door identity are rejected (4xx).
- Requests via Front Door succeed (header injected automatically).
- Rotation is Terraform apply + Argo reconcile (no manual edits).

### Step 4 — register the current cluster as the initial origin

Wire the **current** environment gateway public IPs as origins in AFD origin groups:

- Weight 100 for the current origin in the production endpoint.
- Preview endpoint initially points to the current origin.

Acceptance:
- Preview endpoint can reach a minimal surface (health probe + at least one UI/service route).
- AFD health probes are green.

### Step 5 — cluster recreate becomes “add new origin, weight=0”

For a new cluster (e.g., subscription recreate):

1) Terraform apply cluster stack (new subscription) → publish gateway origin endpoint
2) Terraform apply edge stack → add **new origin** with weight 0 on production endpoint
3) Validate new origin via the **preview endpoint** (target new origin at 100% there)
4) Canary by shifting weights on production endpoint: 0 → 1 → 10 → 50 → 100
5) Remove old origin and destroy old cluster

Acceptance:
- Canary/rollback is only weight changes (Terraform apply), no DNS record churn.

### Step 6 — one-time custom domain cutover (then DNS stays stable)

Once the edge is established and preview validation is trusted:

- Bind the custom domains to the production AFD endpoint (BYOC from Key Vault, issued by Let’s Encrypt automation).
- Update DNS records once to point hostnames to AFD.

After this:
- Cluster recreates never require DNS changes for routing/canary.

---

## 10) Decisions (closed)

These are the “best decisions” for the first implementation, optimized for safety + Terraform CI control.

### 10.1 Edge service choice (Azure)

- **Use Azure Front Door Standard** (not Premium) for the first iteration.
- Use two endpoints per environment:
  - `prod`: for custom domains (once cutover happens)
  - `preview`: for pre-cutover validation on `*.azurefd.net` without DNS changes
- Place the edge resources (AFD + Key Vault for BYOC) in the **cluster/edge subscription** and RG `Ameide` (per current contract).

### 10.2 Origin mode (security baseline)

- **Public origin with hard restrictions** (Mode B) is the baseline, because AFD Standard does not provide Private Link origins.
- Mandatory controls:
  - Network allowlist via `AzureFrontDoor.Backend` service tag (Terraform)
  - Origin-side enforcement of `X-Azure-FDID` (bind to our Front Door profile)
  - Optional defense-in-depth shared secret header (rule set + origin enforcement)
  - WAF custom rules on AFD Standard (managed rule sets are not assumed)

### 10.3 Preview strategy

- Preview validation is done via the **AFD preview endpoint** (`*.azurefd.net`).
- Preview endpoint targets the “new” origin at 100% (separate from production weights).

### 10.4 What is exposed via the edge (default)

- Expose only the **customer/product traffic surfaces** via the edge:
  - `www`, `platform`, and API entrypoints required for normal operation.
- Keep operator tooling internal by default:
  - ArgoCD, Grafana, Prometheus, Loki, Tempo, GitLab, etc.
  - Access via port-forward / internal access mechanisms, not via the public edge.

If we later want an operator edge, it must be a separate set of domains/routes with stricter access controls.

---

## Related backlogs (bidirectional links)

This redesign intentionally intersects Terraform workflows, DNS/TLS strategy, and Gateway routing. Keep these links bidirectional:

- Cluster recreate / “no DNS touch” procedure: `backlog/711-azure-subscription-recreate.md`
- Front Door evaluation and rationale: `backlog/437-split-frontend-cdn-evaluation.md`
- Terraform CI workflow contract: `backlog/444-terraform.md`, `backlog/444-terraform-v2.md`, `backlog/702-terraform-ci-workflow-split-bootstrap-adopt-apply.md`
- Azure subscription + DNS prerequisites: `backlog/444-terraform-v2-az-sub-prereq.md`
- Azure DNS-01 / workload identity (kept as supported fallback): `backlog/438-cert-manager-dns01-azure-workload-identity.md`, `backlog/448-cert-manager-workload-identity.md`
- Infra baseline outputs / runtime facts linkage: `backlog/439-deploy-infrastructure.md`
- Gateway/TLS & routing architecture: `backlog/441-networking.md`, `backlog/450-envoy-gateway-per-environment-architecture.md`, `backlog/300-400/386-envoy-gateway-cert-tls-docs.md`
- Domain + certificate naming matrix (context): `backlog/434-unified-environment-naming.md`, `backlog/000-200/105-domain-strategy-ameide-io.md`

---

## 11) Local deployment compatibility (k3d)

This redesign is intended for shared cloud environments (AKS/EKS/GKE). Local k3d remains a first-class developer path.

### What must remain true for local

- Local clusters should continue to work **without** any global edge/traffic manager dependency.
- Local TLS behavior should remain whatever the current local profile expects (often self-signed/dev CA, not DNS-01).
- Local access patterns (port-forward, local DNS, Telepresence-only domains) should remain valid.

### Design implication

“Edge-first” is the default for cloud environments, but it must be **non-invasive** to local:
- Do not hard-require edge resources for core platform charts.
- Keep the current local certificate patterns (no cloud DNS API assumptions).

---

## References (vendor docs)

- Azure Front Door routing methods: https://learn.microsoft.com/azure/frontdoor/routing-methods
- Azure Front Door HTTPS custom domains: https://learn.microsoft.com/azure/frontdoor/standard-premium/how-to-configure-https-custom-domain
- Azure Front Door managed identity and Key Vault certs: https://learn.microsoft.com/azure/frontdoor/managed-identity
- Azure Front Door certificate requirements (domains): https://learn.microsoft.com/azure/frontdoor/domain
- Azure Front Door origin security (`AzureFrontDoor.Backend`, `X-Azure-FDID`): https://learn.microsoft.com/azure/frontdoor/origin-security
- Azure Front Door rule sets (modify request headers): https://learn.microsoft.com/azure/frontdoor/front-door-rules-engine-actions
- Azure Front Door Private Link to internal LB: https://learn.microsoft.com/azure/frontdoor/standard-premium/how-to-enable-private-link-internal-load-balancer
- AWS Global Accelerator endpoint weights: https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoints-endpoint-weights.html
- GCP traffic management for global external HTTPS LB: https://cloud.google.com/load-balancing/docs/https/setting-up-global-traffic-mgmt
