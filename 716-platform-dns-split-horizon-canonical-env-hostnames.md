# 716 – Platform DNS: split-horizon for canonical `*.{env}.ameide.io` (pods + public, no CoreDNS rewrite)

Goal: make `{service}.{env}.ameide.io` (starting with `coder.dev.ameide.io`) **canonical** and reachable from:

- users (public Internet, via Azure Front Door)
- pods (in-cluster, without relying on CoreDNS rewrite as the correctness mechanism)

This is required for vendor-aligned products that treat a single “Access URL” as canonical (Coder: `CODER_ACCESS_URL` used by both users and workspaces/agents).

Related:
- `backlog/715-azure-front-door-edge-stack.md`
- `backlog/626-ameide-devcontainer-service-coder.md`
- `backlog/454-coredns-cluster-scoped.md` (legacy: wildcard rewrite)
- `backlog/519-gitops-fleet-policy-hardening.md`
- `backlog/713-seeding-contract.md`

---

## Decision

1) **Public path remains edge-first**: public DNS publishes an explicit allow-list of `{service}.{env}.ameide.io` and points it to Azure Front Door (AFD).

2) **Pods use split-horizon DNS**: inside the AKS VNet, `{env}.ameide.io` resolves via **Azure Private DNS** to the in-cluster Gateway, not to AFD.

3) **Terraform is the DNS authority**:
- Terraform manages both public records (Azure DNS) and private records (Azure Private DNS).
- No manual Portal edits and no `kubectl apply` to “fix prod”.

4) **No `ExternalName` in the canonical path**:
- The canonical hostname resolves to the Envoy Gateway data plane Service (has endpoints) via normal DNS recursion.

---

## Constraints (important)

### AFD Standard constraint (no Premium)

We keep **AFD Standard** (no Private Link origins). This means:
- public traffic still goes: DNS → AFD → **public origin** (hardened by NSG/service-tag rules)
- split-horizon improves pod correctness (pods don’t need AFD), but it does **not** eliminate the public origin requirement

### TLS constraint

Pods still connect to `https://{service}.{env}.ameide.io`, so Envoy must present a certificate valid for the canonical hostname set (typically `*.{env}.ameide.io`) and workloads must trust that CA.

---

## Implementation (Azure / AKS)

### Public path (users)

`{service}.{env}.ameide.io` → public DNS → AFD → Envoy origin listener.

This is the “edge-first publish model” described in:
- `backlog/715-azure-front-door-edge-stack.md`

### Private path (pods): Azure Private DNS override

For each enabled non-prod env (e.g. `dev`):

1) Create an Azure **Private DNS zone** named `{env}.ameide.io` (resource type `Microsoft.Network/privateDnsZones`).
2) Link the private zone to the AKS VNet via a **VNet link** (so in-VNet resolvers can see the private answers).
3) Create a wildcard record so pods can resolve canonical hostnames without CoreDNS rewrite:

`*.dev.ameide.io` → CNAME → `envoy-ameide-dev.argocd.svc.cluster.local`

Why CNAME:
- avoids needing a stable “internal LB IP” (AFD Standard constraint)
- keeps Envoy as the single L7 edge
- uses a normal Kubernetes Service with endpoints as the canonical internal target

### Terraform location

The first implementation lives in:
- `infra/terraform/azure/main.tf` (cluster stack)

Key behaviors:
- Only enabled envs (from `config/clusters/azure.yaml`) are configured.
- Staging remains opt-in via enablement in that config file.

---

## Fail-fast acceptance (Coder)

For each enabled env:

- From a pod: `curl -fsS https://coder.{env}.ameide.io/healthz`
- From outside: `curl -fsS https://coder.{env}.ameide.io/healthz`

If the pod path fails:
- treat it as a **platform DNS/edge defect**, not “a Coder issue”

---

## Migration notes

This replaces “CoreDNS rewrite is the correctness mechanism” in managed clusters:
- CoreDNS rewrite remains acceptable as a narrow, temporary debugging tool
- but should not be relied on for canonical hostnames in shared cloud environments

