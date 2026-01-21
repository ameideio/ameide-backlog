---
title: "716 – Platform DNS: Split-horizon for canonical *.dev.ameide.io (Envoy, Terraform authority)"
status: draft
owners:
  - platform
  - gitops
created: 2026-01-21
---

# 716 – Platform DNS: Split-horizon for canonical `*.dev.ameide.io` (Envoy, Terraform authority)

## Purpose

Make `*.{env}.ameide.io` (starting with `coder.dev.ameide.io`) **truly canonical** and reachable from:

- humans (public internet)
- in-cluster agents/workspaces/jobs (pods)

…without relying on cluster-side DNS rewriting as the correctness mechanism.

This is required for vendor-aligned services that treat a single “access URL” as canonical (Coder: `CODER_ACCESS_URL` used by both users and workspaces/agents).

## Scope framing (what changes vs what stays)

- **Public edge-first stays**: public DNS remains allow-listed and points to the edge (AFD Standard).
- **This proposal changes the in-cluster contract**: pods must be able to use the canonical hostname(s) without relying on CoreDNS rewriting or hairpinning to public IPs.

## Decision (single clean long-term solution)

1) **Use split-horizon DNS at the DNS layer**, not CoreDNS rewrite, for `{service}.{env}.ameide.io`.
2) **Keep Envoy Gateway as the single L7 edge** (no service mesh pivot).
3) **Keep Terraform as the DNS authority** (Single Writer): Terraform manages both public and private DNS records.
4) **Do not use `Service type: ExternalName` in the critical path** for the canonical gateway target. The canonical internal target must be a normal Service with endpoints (ClusterIP/LoadBalancer).

## Constraints / non-goals (AFD Standard reality)

### AFD Standard constraint: public origins remain

This proposal assumes **AFD Standard**, not Premium.

- We keep the current “edge-first” posture: public DNS allow-list → AFD → Envoy origin.
- Under AFD Standard, the origin remains **publicly reachable** (even if locked down). Removing public origin reachability requires either **AFD Premium + Private Link** or removing AFD entirely.

### TLS and trust are part of the platform contract

If pods use `https://{service}.{env}.ameide.io`, the internal path must present a certificate valid for that hostname and workloads must trust the issuing CA.

This is not optional: it is inherent to “canonical HTTPS works everywhere”.

### DNS ownership guardrails (prevent drift / break-glass repeats)

- Terraform remains the single writer for DNS records and private-zone wiring.
- No Terraform module may create/modify child-zone delegation unless that zone is explicitly declared as owned by that stack/state.

## Architecture (Azure; applies to all envs)

### Public path (users)

- Public DNS: `{service}.{env}.ameide.io` points to the **edge** (AFD / public gateway).
- TLS is terminated at the edge (AFD) and/or at Envoy (depending on the edge model), but the **presented certificate chain must validate for the canonical hostname**.
- Proxies must preserve **WebSocket** upgrade semantics end-to-end (Coder requires this for workspace apps/terminal).

### Private path (pods)

- Private DNS zone (Azure Private DNS): `{env}.ameide.io` linked to the AKS VNet.
- Private DNS records: `{service}.{env}.ameide.io` resolves to the **internal gateway** (internal LB / private VIP) that routes by Host/SNI to the same Envoy routes.
- The internal gateway path must not introduce NAT between workspaces and the Coder server (Coder networking constraint).
- Proxies must preserve **WebSocket** upgrade semantics end-to-end (same as the public path).

Result: `coder.dev.ameide.io` is the same hostname everywhere, but resolves to different targets inside vs outside.

### Coder workspace apps (wildcard)

If Coder is configured with a wildcard app domain (e.g. `CODER_WILDCARD_ACCESS_URL=*.{env}.ameide.io` or a dedicated app wildcard under the env domain), the same split-horizon contract applies:

- pods must resolve and route the wildcard hostnames without CoreDNS rewriting
- external clients must resolve and route the same hostnames via the public edge

## Success (“Done means”)

This work is done when all of these are true (and remain true after a cluster recreate):

1) From a pod in the cluster, `curl -fsS https://coder.dev.ameide.io/healthz` returns `200` with body `OK`.
2) From a public client, the same request succeeds.
3) DNS evidence:
   - inside the cluster, `coder.dev.ameide.io` resolves to the **private edge** target (private DNS answer)
   - outside the cluster, it resolves to the **public edge** target (public DNS answer)
4) No CoreDNS wildcard rewrite is required for correctness of `coder.dev.ameide.io` (and any configured Coder wildcard app domain).
5) WebSockets work end-to-end for Coder apps/terminal over the canonical hostname.

## What this replaces (legacy)

Deprecate as the primary correctness mechanism in managed clusters:

- cluster-wide CoreDNS wildcard rewrite `(.+\\.)?dev\\.ameide\\.io → envoy.ameide-dev.svc.cluster.local`
- multi-hop in-cluster DNS alias chains using `ExternalName` for the canonical gateway identity

CoreDNS rewrite remains a valid *tool* for narrowly-scoped, temporary overrides (debug/local dev), but not the platform contract for managed clusters.

## Failfast contract (Coder-specific)

Coder requires that workspaces/agents can reach `CODER_ACCESS_URL`. Therefore, for each enabled environment:

- `coder.{env}.ameide.io` must resolve and route correctly from pods **without** relying on CoreDNS rewrite.
- `curl -fsS https://coder.{env}.ameide.io/healthz` must succeed from:
  - a pod in the cluster (private DNS answer)
  - a public client (public DNS answer)

If this contract is not true, treat it as a **platform DNS/edge defect**, not a Coder-specific incident.

## Portability note (AWS)

The **contract** is cloud-agnostic and maps 1:1 to AWS:

- Azure Private DNS split-horizon ↔ Route 53 private hosted zone split-view
- Envoy Gateway + Gateway API are portable across AKS/EKS
- Avoiding `ExternalName` in the critical path is Kubernetes semantics (portable)

What becomes cloud-specific is the **DNS implementation plumbing** (Azure Private DNS zone + VNet link vs Route 53 private hosted zone + VPC association) and the edge/origin privacy model (AFD Standard vs AWS equivalents).

## Cross references

- Current CoreDNS rewrite mechanism (legacy for managed correctness): `backlog/454-coredns-cluster-scoped.md`
- Fleet policy (must prefer split-horizon): `backlog/519-gitops-fleet-policy-hardening.md`
- Coder access URL requirement + /setup seeding context: `backlog/626-ameide-devcontainer-service-coder.md`, `backlog/713-seeding-contract.md`
- Incident evidence for rewrite fragility (issuer discovery/certs): `backlog/715-azure-front-door-edge-stack.md`
