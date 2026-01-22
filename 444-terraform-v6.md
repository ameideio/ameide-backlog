# 444 – Terraform Infrastructure (v6: Layered, Multi-Vendor, Mixed Deployments)

**Created**: 2026-01-22  
**Status**: Draft (direction + contract)

This document defines the next “clean” Terraform contract for Ameide: three explicit layers that can be deployed on **any vendor**, and can be **mixed** (e.g., DNS in one place, edge in another, clusters in multiple vendors).

## Default platform contract (v6 baseline, vendor-aligned)

v6 intentionally standardizes on one default platform model across clouds (**AKS + EKS**):

1) **Edge terminates public TLS; clusters are HTTP origins only**
- Public HTTPS + certificates live at the edge (Azure Front Door on Azure; CloudFront on AWS).
- Clusters expose **only** an HTTP listener (`origin-http`) and route by `Host`.
- **No env HTTPS listeners in-cluster by default**, so missing `ameide-wildcard-tls` can never block Gateway/Route programming.

2) **Edge must preserve/forward the viewer Host header**
- Non-negotiable because in-cluster routing is Host-based.
- Azure Front Door must preserve the incoming Host.
- CloudFront must be configured to forward `Host` to the origin (Origin Request Policy).

3) **SSO/Dex uses a uniform secrets pipeline (no manual kubectl, no Terraform secrets)**
- Argo CD SSO config is runtime config (`argocd-cm` `dex.config`).
- Sensitive values resolve from Kubernetes Secrets (e.g., `argocd-secret`) via `$...` references.
- Use **External Secrets Operator (ESO)** everywhere; only the provider differs:
  - AKS: Azure Key Vault → ESO → `argocd/argocd-secret`
  - EKS: AWS Secrets Manager → ESO → `argocd/argocd-secret`

4) **Hard gates so it can’t regress**
- Gate A: before enabling Dex, required secret keys referenced by `dex.config` exist (e.g., `dex.keycloak.clientSecret`).
- Gate B: the issuer is reachable *from inside the cluster* (e.g., `https://auth.ameide.io/realms/ameide/.well-known/openid-configuration`).

This baseline keeps the **in-cluster GitOps configuration identical** between AKS/EKS. Only infra wiring differs (edge product + DNS provider + secret store provider).

## Current implementation status (repo reality)

v6 roots exist in this repo today (local Terraform runs; CI wiring is intentionally deferred):

- Layer 0 (DNS):
  - `infra/terraform/layer0-dns/azure` (assumes `ameide.io` exists; creates env sub-zones + NS delegations)
  - `infra/terraform/layer0-dns/aws` (creates env hosted zones; outputs NS records for delegation into `ameide.io`)
- Layer 1 (edge):
  - `infra/terraform/layer1-edge/azure-afd` (Azure Front Door Standard; managed certs; CNAME publishing)
  - `infra/terraform/layer1-edge/aws-cloudfront` (CloudFront + ACM DNS validation; CNAME publishing)
- Layer 2 (cluster):
  - `infra/terraform/layer2-cluster/azure-aks`
  - `infra/terraform/layer2-cluster/aws-eks`

Legacy roots (`infra/terraform/azure`, `infra/terraform/azure-edge`) and Azure Terraform CI workflows were removed as part of the v6 clean-slate transition.

**Bootstrap ownership:** ArgoCD install/config is owned by `bootstrap/argocd-bootstrap.sh` (wrapper `bootstrap/bootstrap.sh` is deprecated). Terraform roots must not install ArgoCD (the old `infra/terraform/modules/argocd-bootstrap/*` pattern is legacy and should not be used for v6).

### Hard-coded allow-list (temporary, v6 v1)

Until the provider-neutral “origin contract” and “dns.required_records” schema are implemented, Layer 1 is intentionally hard-coded:

- Azure AFD (`infra/terraform/layer1-edge/azure-afd/main.tf`): explicit hostname list (prod + dev) and explicit origin IP inputs.
- AWS CloudFront (`infra/terraform/layer1-edge/aws-cloudfront/main.tf`): explicit hostname list for one zone (defaults to `dev.ameide.io`) and an explicit `origin_domain_name`.

This is deliberate so we can converge on clean plans quickly. The long-term goal is that Layer 1 consumes Layer 2 outputs and produces Layer 0 DNS requirements (without manual duplication).

Related (history / superseded directions):
- `backlog/444-terraform.md` (current detailed implementation + local k3d workflows)
- `backlog/444-terraform-v2.md` (CI-first policy + runtime facts model)
- `backlog/444-terraform-v2-az-sub-prereq.md` (Azure subscription prerequisites; partially obsolete after clean-slate)
- `backlog/702-terraform-ci-workflow-split-bootstrap-adopt-apply.md` (CI workflow split; some pieces remain)
- `backlog/701-terraform-adopt-reality-aks-drift-realignment.md` (adopt/import playbook; should not be required in v6 steady-state)
- `backlog/697-terraform-env-enablement-single-source.md` (environment enablement / single-source intent)
- Edge/DNS incident context: `backlog/715-azure-front-door-edge-stack.md`, `backlog/712-traffic-manager-first-redesign.md`

---

## The model (3 layers)

### Layer 0 — DNS authority (`ameide.io` and delegations)

**Responsibility:** define where public DNS lives and how it delegates to environment zones (if using child zones like `dev.ameide.io`).

**Key rule:** there is exactly one writer for `ameide.io` (Azure DNS, Route53, Cloudflare, …). Terraform *may* manage it, but it must be a dedicated layer/state owned by the DNS authority.

### Layer 1 — Edge (global entrypoint + TLS + routing)

**Responsibility:** stable entrypoint for public traffic, with:
- TLS termination (managed or BYOC). **v6 baseline:** edge terminates public TLS; origin is HTTP.
- routing rules (host/path → origin)
- health checks
- canary/cutover controls (weights/priority where supported)
- origin protection (prevent bypass)

**Vendor implementations (examples):**
- Azure: Azure Front Door Standard/Premium
- AWS: CloudFront (+ ALB/NLB) or ALB with weighted target groups (regional)
- GCP: Global external Application Load Balancer

### Layer 2 — Cluster (compute + platform)

**Responsibility:** create a Kubernetes cluster (AKS/EKS/GKE/k3d) and the platform’s cloud primitives (identities, secret stores, logging sinks, etc.) so ArgoCD can reconcile everything.

**Critical contract:** clusters must publish a stable “origin endpoint contract” that Layer 1 can route to.

#### Bootstrap contract (uniform across local/AKS/EKS)

To keep the GitOps surface identical across vendors, ArgoCD bootstrap must converge the same invariants everywhere:

- **Single bootstrap owner:** Terraform Layer 2 provisions the cluster only. ArgoCD is installed/configured by `bootstrap/argocd-bootstrap.sh` (not by Terraform).
- **Single credential flow (no secrets in Terraform):** GHCR credentials are provided to the bootstrap process (local: from `.env`; cloud: from the platform secret store / CI environment). Terraform roots must not require GHCR tokens because they get persisted in state.
- **ArgoCD prerequisites:** bootstrap creates (or preserves) required secrets before Helm install/upgrade:
  - `argocd/ghcr-pull` (`kubernetes.io/dockerconfigjson`) for pulling GHCR-mirrored images during bootstrap
  - `argocd/argocd-redis` (`auth` key) when `redisSecretInit.enabled=false`
- **Argo runtime config lives in `argocd-cm`:** `dex.config` and `accounts.*` must be present in `ConfigMap/argocd-cm` (chart values under `configs.cm.*`). Misplacing these under `configs.params` (cmd-params) breaks Dex/SSO.
- **Repository secrets value types:** `configs.repositories` values are stored in `Secret` data and must be **strings** (e.g. `enableOCI: "true"`, not a boolean) to avoid Helm template failures.
- **SSO prerequisites (v6 baseline):** Dex/SSO secrets must be delivered via ESO into `argocd/argocd-secret` (Key Vault on AKS, Secrets Manager on EKS). No manual `kubectl patch` is allowed in steady state.

---

## Goals (v6)

1. **Provider portability** without “mode flags”: choose a vendor by selecting a Terraform root, not by toggling variables.
2. **Mixed deployments are supported**:
   - one edge routing to multiple clusters across vendors
   - multiple clusters serving the same hostname (canary/failover)
3. **Stable DNS publishing model**:
   - apex/root uses provider-appropriate “alias/flattening”
   - subdomains use CNAME to the edge default hostname wherever possible (avoid alias-per-target limits)
4. **Local-first** while we stabilize the contract and remove legacy drift patterns.
5. **No band-aids**: remove transition/adopt/multi-subscription workarounds from the steady-state contract; imports are break-glass only.

Non-goals:
- Perfect feature parity between clouds (capability parity is required; exact products are not).
- Making “DNS must be in Terraform” mandatory (Layer 0 can be external if that’s the governance choice).
- Shipping CI workflows now (deferred; see `backlog/444-terraform-v6-code-layout.md`).

---

## Vendor reality: custom domains + certificates are two-phase

Edge products typically require **DNS validation records** to exist before a custom domain and certificate can become “Active/Deployed”.

This means a single-pass “Layer 1 apply” is not a guaranteed convergence model:

1) Layer 1 creates/requests domain onboarding and emits required DNS records (TXT/CNAME, depending on vendor).  
2) Layer 0 (or external DNS owner) publishes them.  
3) Layer 1 must be applied again to converge to “Approved/Provisioned/Deployed”.

**Recommended v6 posture (no flags): split Layer 1 into two roots per vendor**

- `layer1-edge/<vendor>-core/`
  - creates the edge profile/endpoints/origin groups/routes using default vendor hostnames
  - does **not** attempt to onboard custom domains
- `layer1-edge/<vendor>-domains/`
  - creates custom domains + certificate bindings
  - emits DNS validation + final routing records via `dns.required_records[]`

This keeps “no mode flags” while making the validation loop explicit and operationally predictable.

**Current repo state:** we have one root per vendor for Layer 1 (not split yet). This is acceptable while we validate locally and converge on the final cross-layer contracts.

---

## Apex/root domain constraints (must be explicit)

“Apex”/root records (e.g., `ameide.io`) are constrained by DNS and vendor product behavior:

- Apex can’t be a normal CNAME in most DNS providers; it typically requires **alias/flattening**.
- Some edge products/tier combinations have limitations around **managed certificates for apex** and may require BYOC.

v6 default posture:
- Keep apex usage minimal (redirect to `www`), unless there is a hard requirement for apex hosting.
- Treat apex support as a Layer 0 + Layer 1 joint design item, not an afterthought.

**Current repo state:** we do not attempt to create or delete the parent zone (`ameide.io`) in Terraform. When Layer 0 manages DNS, it assumes the zone already exists and focuses on env sub-zones + delegations.

---

## Vendor capability matrix (baseline expectations)

This is not about picking identical products; it’s about being explicit on what the chosen edge can do.

| Edge vendor implementation | Custom domains + certs | Canary weights | Failover | Known cert/DNS constraints |
|---|---:|---:|---:|---|
| Azure Front Door (AFD) | Yes | Yes (weights) | Yes | DNS validation tokens; apex requires alias/flattening |
| AWS CloudFront | Yes | Not a simple global “weights” knob (baseline) | Yes (origin failover) | ACM region constraints; DNS validation records |
| GCP global external ALB | Yes | Yes (weighted backends) | Yes | DNS validation records depending on cert mode |

Where “weights” are not natively supported at the edge, v6 must implement canary at a different layer (regional LB, or DNS steering), and that must be documented in the vendor root.

---

## Edge reverse-proxy invariants (required by v6 baseline)

- **Host preserved:** edge forwards the viewer Host header to the origin (cluster routing depends on it).
- **Viewer HTTPS:** edge redirects HTTP → HTTPS for viewers.
- **Origin protocol:** edge forwards to the origin over **HTTP** by default (edge-terminates TLS profile).

---

## Mixed configurations (supported examples)

### Example A: Single-vendor (Azure-only)
- Layer 0: Azure DNS hosts `ameide.io` (zone is assumed pre-existing)
- Layer 1: Azure Front Door
- Layer 2: AKS cluster(s)

### Example B: Multi-cloud origins behind one edge (recommended mixed pattern)
- Layer 0: DNS authority (any provider) points `platform.ameide.io` to the chosen edge
- Layer 1: Azure Front Door (or other global edge)
- Layer 2: one AKS cluster + one EKS cluster as origins (weighted/priority routing)

### Example C: DNS outside Terraform
- Layer 0: managed by a separate DNS system (Cloudflare/Route53/enterprise DNS)
- Layer 1 + Layer 2: Terraform-managed in any vendor(s)
- Contract: Terraform outputs the exact records/delegations required, but does not write them.

---

## Terraform structure (no vendor flags)

### Required: separate roots/states per layer + vendor

Suggested layout:

```
infra/terraform/
  layer0-dns/
    azure/   # manages env sub-zones + delegations; assumes ameide.io exists
    aws/     # manages env hosted zones; emits delegation records for ameide.io
    cf/      # manages ameide.io in Cloudflare (optional)
  layer1-edge/
    azure-afd/               # current single-root implementation (core+domains)
    aws-cloudfront/          # current single-root implementation (core+domains)
    azure-afd-core/          # planned split
    azure-afd-domains/       # planned split
    aws-cloudfront-core/     # planned split
    aws-cloudfront-domains/  # planned split
    gcp-gclb-core/           # future
    gcp-gclb-domains/        # future
  layer2-cluster/
    azure-aks/
    aws-eks/
    gcp-gke/   # future
    local-k3d/ # kept
```

Each root has:
- its own backend config/state key (CI-provided)
- explicit outputs used by the next layer (never by direct resource references)

---

## Cross-layer contracts (outputs) — minimal, provider-neutral

### Layer 2 → Layer 1 (origin contract)

Layer 2 must output, per environment:
- `origin.address` (IP or hostname)
- `origin.protocol` (`http|https`)
- `origin.port_http` and (optional) `origin.port_https`
- `origin.tls_server_name` (SNI when `https`)
- `origin.expected_host_header` (what edge should send)
- `origin.health_protocol` + `origin.health_port` + `origin.health_path` (fast 200)
- `origin.timeout_seconds` (edge→origin)
- `origin.path_prefix` (optional; for edges that need a base path)
- `origin.weight` and `origin.priority` (edge may use one or both)
- `origin_security.mode` (validated enum; see “Origin security modes” below)

### Layer 1 → Layer 0 (DNS publishing requirements)

Layer 1 must output:
- `edge.default_hostname` (e.g., `...azurefd.net`)
- `edge.custom_domains[]` (the desired public hostnames)
- `dns.required_records[]` (exact desired records, expressed generically as `{id,purpose,type,name,value,ttl}`) so Layer 0 can implement or an external DNS owner can apply.

---

## Origin security modes (enforced contract)

`origin_security.mode` is not an aspiration; it must be a **validated enum** with an explicit support matrix per (edge vendor × origin vendor).

Example support matrix (illustrative; the real matrix lives in the vendor roots):

| Edge | Origin | Allowed `origin_security.mode` baseline |
|---|---|---|
| `azure-afd` | `azure-aks` | `afd_azure_service_tag + fdid_header` |
| `aws-cloudfront` | `aws-eks` | `cloudfront_prefix_list + secret_header` |
| `azure-afd` | `aws-eks` | `secret_header + additional_hardening_required` (only if explicitly accepted) |

If a mixed-vendor pairing lacks a strong “only from edge” control, v6 must document the risk and require an explicit approval to proceed.

---

## CI / GitOps contract (deferred)

The long-term GitOps rule remains: shared cloud environments should converge via Git + CI Terraform, then ArgoCD reconciliation.

**However, in v6 right now we are deliberately running Terraform locally** to converge on the code layout and cross-layer contracts before reintroducing CI jobs.

### Concurrency nuance (GitHub Actions)

Concurrency groups prevent parallel writers, but do not guarantee ordering across queued runs. Apply workflows should be:
- triggered from controlled events (merge + environment approvals), and
- configured so apply runs aren’t canceled/replaced mid-flight.

---

## Immediate follow-ups (implementation plan)

This is intentionally ordered and non-branching:

1. **Enforce HTTP-only origins in overlays** (AKS + EKS):
   - Ensure env Gateways expose `origin-http` and are `Programmed=True` without any wildcard TLS secrets.
   - Remove/avoid route attachments that assume `sectionName: https` under the edge-terminates profile.
2. **Make edge behave like a reverse proxy** (Host preserved):
   - CloudFront must forward `Host` to the origin (Origin Request Policy).
   - AFD must preserve the incoming Host.
3. **Make Dex deterministic with ESO** (same Kubernetes interface on AKS/EKS):
   - Deploy ESO once per cluster.
   - Materialize required Argo Dex keys into `argocd/argocd-secret` from the provider secret store.
4. **Enforce the two SSO gates in `backlog/580-sso-verifications.md`**:
   - in-cluster issuer well-known fetch succeeds (TLS-valid)
   - required secret keys exist before Dex is enabled
5. Converge the v6 roots to “clean, deterministic plan/apply” locally for both Azure and AWS (no adopt/import in steady-state).
6. Keep the edge allow-list explicit and versioned (no wildcard public exposure by default) until the provider-neutral contract schema lands.
7. After the contract is stable, reintroduce CI jobs per root (no multi-subscription gimmicks; no “adopt” in the normal path).
