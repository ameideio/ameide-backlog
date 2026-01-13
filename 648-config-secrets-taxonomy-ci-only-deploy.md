# 648 – Config/Secrets Taxonomy + CI-Only Deployments

## Problem

We currently have recurring confusion and drift around:

- **What is a “secret” vs a “setting”.**
- **Where each should live** (Vault/Key Vault vs Helm values vs ConfigMaps vs CI env).
- **How environments are promoted** (GitOps digest promotion) vs “manual” changes (kubectl/Argo UI).
- **How URL/base-url variables are named and sourced** across Terraform DNS, Gateway HTTPRoutes, app env, and test harnesses.

The result is:

- Developers and operators have to memorize too many knobs.
- Environment parity is fragile (dev/staging/prod diverge).
- Secrets risk leaking into the wrong surface (values, CI logs, charts).
- Debugging login/onboarding and e2e often turns into guessing which URL variable is “the real one”.

This backlog establishes a **single end-to-end contract**: where configuration belongs, how it is named, and how we verify it.

## Goal (Non-Negotiable)

1. **Secrets live only in Key Vault (KV) as the source of truth** and are delivered to Kubernetes via **ExternalSecrets**.
   - Cloud: Azure Key Vault is the source of truth.
   - Local: a local KV instance may be used for local clusters, but the contract is identical: KV is the only writable secret store and Kubernetes receives secrets only via ExternalSecrets.
   - Note: syncing KV → Kubernetes necessarily materializes values as Kubernetes `Secret` objects; clusters must enforce **encryption at rest** for Secrets and **least-privilege RBAC** to read them.
2. **Settings live only in GitOps values → rendered ConfigMaps** (ArgoCD-managed).
3. **CI is the only supported path to change desired state** (PRs to Git).
   - **Argo CD is the only actor applying desired state to clusters.**
   - No `kubectl apply`, no “click ops”, no manual Argo overrides for Argo-managed workloads (break-glass only; revert via GitOps).
4. Provide a **taxonomy** for env vars and URL configuration, with concrete examples and guardrails.
5. Provide a **refactoring plan + verification plan** so we can land this safely and incrementally.

## Non-Goals

- Changing the underlying secret store provider (Vault vs Azure Key Vault) beyond enforcing “KV-only” semantics.
- Rewriting Auth.js / Keycloak flows; this item is about configuration surfaces, not authentication product behavior.
- Creating a new configuration framework; we stick to Helm + ArgoCD + ExternalSecrets + Terraform DNS.

## Definitions

### “Secret”

Anything that would be a security incident if exfiltrated, including:

- Signing/encryption keys (e.g., Auth.js secret)
- OAuth client secrets (Keycloak clients, admin clients)
- Database credentials and connection URLs
- Redis passwords
- Tokens (GitHub, Buf, registries, API keys)

**Rule:** Secrets must not appear in Helm values, ConfigMaps, Git history, or CI logs. They must be sourced from KV and synchronized to K8s Secrets via ExternalSecrets.

### “Setting”

Non-sensitive configuration that can be safely stored in Git and rendered by Helm, including:

- Public hostnames (e.g., `https://platform.dev.ameide.io`)
- Internal service endpoints (e.g., `http://envoy-grpc:9000`)
- Feature toggles, timeouts, log levels, environment names
- Cookie domains / SameSite / secure flags (not the secret key itself)

**Rule:** Settings must not be stored in Secrets as a habit. They belong in GitOps values and render into ConfigMaps.

### “Product-managed secret state” (allowed exception)

Removed. This backlog defines the configuration contract for AMEIDE workloads: secrets are stored in KV and delivered via ExternalSecrets. Product-specific internal state is out of scope for this taxonomy.

## Taxonomy (Required)

### URL Categories

1. **Public Canonical URL** (browser-visible “this app” host)
   - Example: `https://platform.dev.ameide.io`
   - Source of truth: GitOps values (`auth.url`)
   - Output env: `AUTH_URL`
   - Must match:
     - Terraform DNS records (`platform.<env>.ameide.io`)
     - HTTPRoute hostnames for the app
     - Cookie domain scoping

2. **Internal RPC URL** (server-to-server inside cluster)
   - Example: `http://envoy-grpc:9000`
   - Source of truth: GitOps values (`envoy.url`)
   - Output env: `AMEIDE_GRPC_BASE_URL`

3. **Identity Provider (IDP) Public Issuer**
   - Example: `https://auth.dev.ameide.io/realms/ameide`
   - Source of truth: GitOps values (`keycloak.issuer`)
   - Output env: `AUTH_KEYCLOAK_ISSUER`

4. **Identity Provider Internal Base** (optional)
   - Example: `http://keycloak:8080`
   - Source of truth: GitOps values (`keycloak.internalBase`)
   - Output env: `AUTH_INTERNAL_KEYCLOAK_BASE`, plus derived token/userinfo/jwks/revoke endpoints where needed.

### Environment Variable Categories

- **`*_URL` / `*_BASE_URL`**: must be classified as either public canonical, public API, internal RPC, or IDP.
- **Browser (`NEXT_PUBLIC_*`)**:
  - **Not allowed for cluster deployments** in AMEIDE: client bundles must remain environment-invariant under digest promotion.
  - Client-visible endpoints/config must be provided via **runtime mechanisms** (server-rendered config, bootstrap JSON, etc.), not compile-time `NEXT_PUBLIC_*`.
- **`AUTH_*`**:
  - Authoritative (Auth.js v5): `AUTH_*` are the canonical settings/inputs.
  - `AUTH_URL` is a setting (ConfigMap).
  - `AUTH_TRUST_HOST=true` is required in gateway/proxy deployments (fail-fast). Application code must require it, and GitOps must reject deployments where it is not explicitly set to `true`.
  - `AUTH_SECRET` and provider secrets are secrets (ExternalSecrets).
- **Test harness vars**:
  - CI/e2e-only vars must be prefixed or clearly documented.
  - Example: `AMEIDE_PLATFORM_BASE_URL` (CI/Playwright) is *not* a runtime setting.

### Build-time vs Runtime (Critical for Next.js + Digest Promotion)

Next.js `NEXT_PUBLIC_*` variables are **inlined at build time into browser bundles**. This creates a fundamental constraint:

- If we promote **one immutable image digest** across environments (dev → staging → prod), then any **environment-specific value used in browser JS must not rely on build-time inlining**, otherwise dev values can leak into prod bundles.

Policy:

- For AMEIDE cluster deployments, **do not use `NEXT_PUBLIC_*` for environment-specific values**.
- Client-visible config must be runtime-provided by the server (or derived from same-origin requests).

## URL & Auth Variable Map (Where Values Live Today)

This is the “answer in 60 seconds” map for URL-related inputs across Terraform, GitOps, Kubernetes, and CI.

### Public hostnames (DNS + Gateway)

- **Terraform (DNS)** owns the records:
  - `platform.<env>.ameide.io` → Gateway/ingress address
  - `auth.<env>.ameide.io` → Keycloak gateway route
  - `api.<env>.ameide.io` → Envoy/API gateway route
- **GitOps (Gateway API HTTPRoute)** owns the hostnames actually served by the cluster:
  - `www-ameide-platform` HTTPRoute hostnames (must include `platform.<env>.ameide.io`)
  - Keycloak HTTPRoute hostnames (must include `auth.<env>.ameide.io`)

### Runtime settings (ConfigMaps via GitOps)

`www-ameide-platform` must get these from a ConfigMap rendered by the `ameide-gitops` chart values:

- `AUTH_URL`
  - **Owner/source**: GitOps values (`auth.url`)
  - **Surface**: `ConfigMap/www-ameide-platform-config` → pod env
- `AMEIDE_GRPC_BASE_URL`
  - **Owner/source**: GitOps values (`envoy.url`)
  - **Surface**: ConfigMap → pod env
- Keycloak settings (issuer + explicit endpoints)
  - **Owner/source**: GitOps values (`keycloak.issuer`, `keycloak.internalBase`, plus any required static defaults)
  - **Surface**: ConfigMap → pod env

### Runtime secrets (KV → ExternalSecrets → Kubernetes Secret)

`www-ameide-platform` must get these from a Kubernetes `Secret` produced by ExternalSecrets:

- `AUTH_SECRET`
- `AUTH_KEYCLOAK_SECRET`
- `AUTH_KEYCLOAK_ADMIN_CLIENT_SECRET`

Contract:

- **Owner/source**: Key Vault (KV)
- **Delivery**: ExternalSecrets (ESO) → Kubernetes `Secret/www-ameide-platform-auth`
- **Consumption**: Deployment `envFrom.secretRef` (no inline secret values in values/ConfigMaps)

### Test-only base URL (CI/E2E runner)

- `AMEIDE_PLATFORM_BASE_URL`
  - **Owner/source**: CI/runner contract (not a runtime setting)
  - **Where set today**:
    - `ameide test e2e` reads `AUTH_URL` from `ConfigMap/www-ameide-platform-config` and passes it to Playwright as `AMEIDE_PLATFORM_BASE_URL` (and verifies it matches the HTTPRoute host).
  - **Used by**: Playwright baseURL (`services/www_ameide_platform/playwright.config.ts`)

## Repository Contract (Single Source of Truth)

### Terraform

- Owns **DNS records** (e.g., `platform.<env>.ameide.io`, `api.<env>.ameide.io`, `auth.<env>.ameide.io`).
- Must not attempt to manage application env vars.

### GitOps (`ameide-gitops`)

- Owns **Kubernetes desired state**:
  - HTTPRoute hostnames
  - ConfigMaps containing non-secret settings
  - ExternalSecrets templates mapping KV keys → K8s Secrets
- Owns per-environment values for:
  - canonical public host (`auth.url`)
  - cookie domain (`auth.cookies.domain`)
  - server-only gRPC base URL (`envoy.url`)
  - Keycloak issuer and endpoints (`keycloak.*`)

### Application Repos

- Own code that reads env vars.
- Must not require developers to wire secrets locally by hand for cluster deployments; the cluster path must be fully GitOps-driven.

### CI Workflows (Required deployment-only path)

- Build images and publish to `ghcr.io` (immutable digests).
- Create GitOps promotion PRs by copying digests forward (dev → staging → prod).
- Must not “kubectl apply” Argo-managed resources directly.

## Required Platform Defaults (Enforcement)

### External Secrets Operator (ESO)

- ESO must authenticate to Azure Key Vault using **Workload Identity** (no deprecated identity mechanisms).
- Every `ExternalSecret` must define a `refreshInterval` appropriate for rotation and incident response.
- “KV-only” means:
  - KV is the only editable source of secret values.
  - Kubernetes Secrets are derived artifacts and must be treated as sensitive data stores (RBAC + encryption at rest).

### Argo CD

“No manual changes” is enforced by configuration and access control, not only by policy documents:

- Argo Applications that are declared “Argo-managed” must enable automated drift correction (**self-heal**) so manual edits are reverted.
- If we require fully declarative deletions, enable automated **prune** for those apps.
- Argo RBAC must restrict “sync/override/edit live” to platform operators; product teams should change desired state via Git PRs only.

### Smokes (in ArgoCD, not CI)

“Smokes” are part of the **GitOps convergence contract** and must run **inside ArgoCD** as hook Jobs (PostSync), so that:

- A deployment is not considered “done” unless the cluster can prove the public routes and required dependencies are reachable.
- Failures surface in ArgoCD (the same control plane that applies desired state), not in ad-hoc manual checks.

CI’s role is:

- build/publish immutable artifacts (images, charts)
- update desired state in Git (digests/values)

ArgoCD’s role is:

- apply desired state
- run in-cluster smokes as part of the rollout gate

Implementation note:

- Today the smoke Jobs use minimal container images (e.g., curl/kubectl) and a small script wrapper to enforce deterministic assertions (expected `302/401/404/200`, expected headers, etc.).
- If we ever adopt a strict “no shell in jobs” rule, we must replace the script wrappers with a single pinned “smoke runner” binary/image; otherwise smokes become too weak to act as a gate.

## Refactoring Plan

### Phase 0 – Inventory (no behavior change)

- Produce an inventory of:
  - All env vars consumed by each app.
  - Which are secrets vs settings.
  - Current sources (values/configmap/secret/ci).
- Output: a table for each deployed app:
  - `VAR_NAME`, category, owner (Terraform/GitOps/App/CI), source-of-truth path.

### Phase 1 – Guardrails (prevent regressions)

- Helm guardrails:
  - Fail template render if any secret is set inline in values (except explicit dev-only fixtures when permitted).
  - Fail template render if canonical URL + HTTPRoute hostname mismatch (when both are present).
  - Fail template render if any `NEXT_PUBLIC_*` keys are emitted for `www-ameide-platform` (cluster path).
- Policy guardrails:
  - CI check rejects commits that add known secret patterns to values/configmaps.
  - CI check ensures `AUTH_URL` is set for any environment where auth is enabled.
  - CI check rejects `NEXT_PUBLIC_*` usage for cluster deployments (digest promotion invariant).

### Phase 2 – Reduce “duplicate URL knobs”

- Prefer a small set of source inputs in values:
  - `auth.url`, `auth.cookies.domain`
  - `envoy.url`
  - `keycloak.issuer`, `keycloak.internalBase`
- Derive the rest where possible (token/jwks/userinfo/revoke endpoints) to reduce per-env churn and drift.

### Phase 3 – Normalize “secrets only in KV”

- Migrate remaining inline secrets into KV.
- Ensure ExternalSecrets templates are the only path from KV → K8s Secret.
- Ensure apps read secrets from env (Secret) and settings from env (ConfigMap), but the source is enforced.

### Phase 4 – CI-only deploy enforcement

- Document and enforce that:
  - Images are only built in CI and referenced by digest.
  - Desired state is only changed via GitOps PRs; ArgoCD is the only cluster applier.
  - “Manual patches” are considered break-glass and must be undone by updating GitOps.
- Add/confirm a “break-glass” runbook that includes:
  - how to identify drift,
  - how to revert drift via GitOps,
  - how to rotate secrets in KV and validate sync.

## Verification Plan (Definition of Done)

### Static Verification (CI)

- `helm template` renders for all env overlays without secret material in ConfigMaps.
- Secret scan for values/templates rejects known sensitive keys.
- ConfigMap contains required settings with expected formats (`AUTH_URL`, `AMEIDE_GRPC_BASE_URL`, Keycloak endpoints), and contains **no** `NEXT_PUBLIC_*` keys for cluster deployments.
- ExternalSecrets manifests reference KV keys and do not inline secret values.

### Runtime Verification (Cluster)

- ExternalSecrets sync healthy:
  - `Secret/www-ameide-platform-auth` exists and contains `AUTH_SECRET`, `AUTH_KEYCLOAK_SECRET`, `AUTH_KEYCLOAK_ADMIN_CLIENT_SECRET`.
- Cluster secret storage posture:
  - Secrets are encrypted at rest in etcd, and only the minimum service accounts can read the relevant Secrets.
- App pod has:
  - URL settings in env from ConfigMap,
  - secrets in env from Secret,
  - no secrets present in ConfigMap,
  - no `NEXT_PUBLIC_*` runtime config for cluster deployments.
- Login flow:
  - Cookies scoped correctly (`AUTH_COOKIE_DOMAIN`) so sessions survive redirects.
- E2E/inner-loop tests:
  - `./ameide test` passes (mandatory agent check).

## Acceptance Criteria

- A developer can answer “where does this value come from?” for any URL/secret in ≤ 60 seconds using the taxonomy and inventory.
- No secrets are committed in GitOps values or ConfigMaps.
- CI is the only supported deployment path; GitOps promotion uses immutable digests.
- A consistent naming scheme exists for URL variables, and tests do not require ad-hoc overrides to find the platform host.

## Historical Context (Why This Exists)

- We historically mixed “settings” into Kubernetes Secrets and “secrets” into values/ConfigMaps/CI env; this item defines a single contract to remove drift and guesswork.
- We historically used NextAuth-era `NEXTAUTH_*` variables in some places; the contract is now Auth.js v5 `AUTH_*` only, and any remaining `NEXTAUTH_*` usage is migration debt to delete (not a supported interface).
- We historically allowed manual cluster edits (`kubectl`, Argo UI) that could linger when drift correction wasn’t enforced; this item makes “Git desired state + Argo apply” the norm (with self-heal/RBAC as enforcement).

## Notes / Related Backlogs

- Onboarding and cookie-domain alignment discussions: `backlog/300-400/319-onboarding-v2.md`.
- E2E base URL expectations: `backlog/300-400/371-e2e-playwright.md`.
- Domain standardization efforts: `backlog/300-400/366-local-domain-standardization.md`.
