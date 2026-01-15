---
title: "680 (Increment 2) — Che on vCluster (devcluster) for Kubernetes OIDC on AKS"
status: draft
owners:
  - platform-devx
  - platform
  - gitops
created: 2026-01-15
suite: "agentic-coding"
source_of_truth: false
---

# 680 (Increment 2) — Che on vCluster (devcluster) for Kubernetes OIDC on AKS

This increment addresses the core blocker discovered in Increment 1:

- Che’s **Kubernetes authorization model** relies on Kubernetes API authentication + RBAC enforcement (gateway includes oauth2-proxy + kube-rbac-proxy).
- On managed AKS, the Kubernetes API server does **not** authenticate arbitrary external OIDC issuers like our Keycloak (`auth.dev.ameide.io`).
- Result: Che dashboard/backend calls to the Kubernetes API using the Keycloak bearer token return `401 Unauthorized`, and the UI shows:
  - “Failed to fetch the list of devWorkspaces… Unauthorized”
  - “Unable to get user profile data… Unauthorized”

Vendor-aligned solution: run Che on a **virtual Kubernetes cluster (vCluster)** whose API server can be configured to trust Keycloak OIDC.

Normative constraints remain in `backlog/680-che.md`. Increment 1 scope remains in `backlog/680-che-increment-1.md`.

## Goal

Provide a working Che UI on `che.dev.ameide.io` where:

- user login succeeds via Keycloak (SSO), and
- dashboard can list DevWorkspaces / user profile without `401` because the **vCluster API server** accepts the OIDC token and Kubernetes RBAC can be evaluated.

## Non-goals

- Full task-mode provider integration (Camunda, agent entrypoints, PR automation).
- Workspace image parity, digest pinning, or end-to-end agent workflows.
- Eliminating all Keycloak instability issues (those are separate hardening items).

## Big picture (what changes vs Increment 1)

- Increment 1 deployed Che directly on AKS and validated “app SSO”, but Che’s Kubernetes auth chain fails because AKS cannot validate Keycloak JWTs.
- Increment 2 introduces a **devcluster** (vCluster) hosted on AKS:
  - Che runs *inside* the vCluster.
  - The vCluster API server is configured with Keycloak OIDC, so Kubernetes authn/authz works the way Che expects.
  - Workloads still run on AKS nodes; the difference is which Kubernetes API server performs auth + RBAC decisions.

## Target (AKS ameide / dev)

- Public Che UI host: `che.dev.ameide.io`
- OIDC issuer: `https://auth.dev.ameide.io/realms/ameide`
- vCluster name (proposed): `che-devcluster`
- vCluster host namespace (proposed): `che-devcluster` (on AKS)

## Delivery plan (GitOps)

Hard rule: no manual `kubectl apply/patch/delete` for cloud resources. All mutation is Git → CI → ArgoCD.

### 1) Deploy vCluster (dev only)

Add a new GitOps component (dev only):

- `platform-vcluster-che`:
  - Installs a vCluster control plane (Helm) into host namespace `che-devcluster`.
  - Configures the vCluster API server with external OIDC:
    - issuer URL = `https://auth.dev.ameide.io/realms/ameide`
    - client ID / audiences aligned with the token Che forwards (see “identityToken” note)
    - username claim mapping + groups claim mapping
  - Exposes the vCluster API server via an internal `Service` only (no public ingress).

### 2) Register vCluster as an ArgoCD destination (dev only)

ArgoCD must be able to deploy into the vCluster API server:

- Materialize an ArgoCD cluster secret in `argocd` namespace pointing at the vCluster API server.
- The cluster secret MUST be materialized via our secrets pipeline (Vault + ExternalSecrets), not committed in Git.

### 3) Deploy Che into the vCluster (dev only)

Add a new GitOps component (dev only):

- `platform-che` (vCluster target):
  - Same vendor operator + `CheCluster` model, but deployed into the **vCluster** rather than the AKS API server.
  - Ensure the Che gateway service is reachable from host ingress (see next step).

### 4) Expose Che UI through the host Gateway (Envoy)

Keep the public ingress plane on the host cluster:

- Route `che.dev.ameide.io` on Envoy Gateway to the Che gateway service that is synced/exported into the host namespace.

## Vendor knob to track explicitly: identity token type

Che supports forwarding either:

- `id_token` (default), or
- `access_token`

via `CheCluster.spec.networking.auth.identityToken`.

This increment must explicitly record which token is used and ensure the vCluster API server is configured to accept it (audiences/claims).

## Acceptance criteria

- ArgoCD applications converge to `Healthy/Synced`:
  - `dev-platform-vcluster-che`
  - `dev-platform-che` (targeting the vCluster destination)
- Che UI on `https://che.dev.ameide.io/` does not show the dashboard “Unauthorized” errors.
- Dashboard endpoints work end-to-end:
  - listing DevWorkspaces works
  - user profile fetch works
- Smoke evidence from logs:
  - `che-dashboard` no longer produces `401` responses from the Kubernetes API server when listing namespace resources.

## Verification checklist (CLI, read-only)

- Argo apps are Healthy/Synced:
  - `kubectl -n argocd get application | rg 'dev-platform-(vcluster-che|che)'`
- vCluster control plane is healthy:
  - `kubectl -n che-devcluster get pod`
- Che is deployed into the vCluster (via Argo destination):
  - `kubectl -n argocd get application dev-platform-che -o yaml | rg 'destination'`
- Che dashboard “Unauthorized” is gone (service logs):
  - `kubectl -n <che-namespace-in-vcluster> logs deploy/che-dashboard --since=10m | rg -n 'Unauthorized|statusCode\": 401'`

## Risks / known sharp edges

- vCluster introduces an additional control plane component; we must track its resource footprint and failure modes.
- Mapping/sync behavior (namespaces, services, routes) must be carefully scoped so Che is reachable on `che.dev.ameide.io` without hijacking other dev traffic.

