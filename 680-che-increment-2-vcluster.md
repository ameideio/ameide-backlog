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
- On our current AKS configuration, the Kubernetes API server is not configured (or not permitted) to authenticate our external OIDC issuer (Keycloak at `auth.dev.ameide.io`) for Kubernetes API requests from Che’s gateway/RBAC path.
- Result: Che dashboard/backend calls to the Kubernetes API using the Keycloak bearer token return `401 Unauthorized`, and the UI shows:
  - “Failed to fetch the list of devWorkspaces… Unauthorized”
  - “Unable to get user profile data… Unauthorized”

Vendor-aligned solutions:

- Preferred when feasible: configure the managed control plane (AKS/EKS/GKE) to accept the external OIDC issuer for Kubernetes API authentication, subject to provider constraints and org policy.
- Fallback / portability strategy: run Che on a **virtual Kubernetes cluster (vCluster)** whose API server can be configured to trust Keycloak OIDC without changing the host cluster control plane.

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

- Increment 1 deployed Che directly on AKS and validated “app SSO”, but Che’s Kubernetes auth chain fails because the host cluster control plane is not currently validating Keycloak JWTs for the Che gateway/RBAC path.
- Increment 2 introduces a **devcluster** (vCluster) hosted on AKS:
  - Che runs *inside* the vCluster.
  - The vCluster API server is configured with Keycloak OIDC, so Kubernetes authn/authz works the way Che expects.
  - Workloads still run on AKS nodes; the difference is which Kubernetes API server performs auth + RBAC decisions.

In other words, vCluster is **control-plane isolation** (auth + RBAC), not network isolation.

### Optional dev workflow: Tilt/hot-reload against the host cluster

If we want “hot-reload” workflows without changing where traffic enters the platform:

- Che + DevWorkspace control plane runs in the vCluster.
- The workspace Pod (devcontainer) still runs on the host cluster (via vCluster sync).
- Inside the devcontainer, developers can run Tilt targeting the **host** cluster API context (not the vCluster):
  - `tilt up --context <host> --namespace <dev-namespace>`
  - Tiltfile should include `allow_k8s_contexts([...])` guardrails.

Important: our cloud GitOps rules still apply. Tilt-driven writes to shared AKS must be explicitly scoped (dedicated dev namespaces, least-privilege RBAC) and must not be treated as the authoritative mutation path for platform resources (Git → CI → ArgoCD remains authoritative).

## Two kubeconfigs model (recommended)

To make the “workspace runs here, deploy there” story explicit:

- **vCluster kubeconfig/context**: used by Che components for workspace lifecycle and Kubernetes RBAC evaluation (inside the vCluster control plane).
- **host cluster kubeconfig/context**: optionally mounted into the devcontainer for developer tools (Tilt, kubectl) to deploy into a dedicated host namespace with least privilege.

## Target (AKS ameide / dev)

- Public Che UI host: `che.dev.ameide.io`
- OIDC issuer: `https://auth.dev.ameide.io/realms/ameide`
- vCluster name (proposed): `che-devcluster`
- vCluster host namespace (proposed): `che-devcluster` (on AKS)

## Delivery plan (GitOps)

Hard rule: no manual `kubectl apply/patch/delete` for cloud resources. All mutation is Git → CI → ArgoCD.

### 1) Deploy vCluster + bootstrap Che inside it (dev only)

Add a new GitOps component (dev only):

- `platform-vcluster-che`:
  - Installs a vCluster control plane (Helm) into host namespace `che-devcluster`.
  - Configures the vCluster API server with external OIDC:
    - issuer URL = `https://auth.dev.ameide.io/realms/ameide`
    - `--oidc-client-id=che` (audience)
    - username/groups claim mapping
  - Installs Che dependencies inside the vCluster:
    - cert-manager (CRDs + controller)
    - Che operator (Helm)
    - DevWorkspace operator (bootstrap Job)
    - `CheCluster` (bootstrap Job, applied only after CRDs exist)
  - Exposes the vCluster API server via an internal `Service` only (no public ingress).

If we intend to support the “two kubeconfigs” model, this increment should also define:

- host-side developer namespaces (per-dev or per-run),
- a least-privilege ServiceAccount + Role/RoleBinding for Tilt in those namespaces, and
- a secure path to mount a kubeconfig for that ServiceAccount into the devcontainer.

### 2) Expose Che UI through the host Gateway (Envoy)

Keep the public ingress plane on the host cluster:

- Route `che.dev.ameide.io` on Envoy Gateway to the Che gateway service that is synced/exported into the host namespace.

## Vendor knob to track explicitly: identity token type

Che supports forwarding either:

- `id_token` (default), or
- `access_token`

via `CheCluster.spec.networking.auth.identityToken`.

This increment must explicitly record which token is used and ensure the vCluster API server is configured to accept it (audiences/claims).

## Trusted certificates (CA bundles): vendor model vs stability mitigation

Che supports importing additional trusted CAs (for untrusted/self-signed endpoints) via ConfigMaps and propagating trust into Che server/dashboard and workspaces.

During deployment we observed `che-server` repeatedly restarting while importing a large CA bundle into the JVM truststore (`Certificate was added to keystore` loop), causing probes to fail and the dashboard to error.

Current mitigation (dev-only, to unblock):

- Disable the Che server’s “extra CA bundle import” configmap hook while keeping Kubernetes API CA trust enabled.
- Constraint: `auth.dev.ameide.io` and Git endpoints MUST be publicly trusted (no private CA required) for this mitigation to be acceptable.

Follow-up hardening (vendor-aligned):

- If we need private CA trust, re-enable CA import using a minimal, validated PEM bundle (no accidental “full system CA” injection), or adjust startup probes based on measured startup time rather than disabling trust mechanisms.

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
