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

1. **Secrets live only in a Key Vault (KV) as the source of truth** and are delivered to Kubernetes via **ExternalSecrets**.
   - Cloud: Azure Key Vault is the source of truth. If an in-cluster Vault exists, it is a CI/bootstrap-fed cache only (no “secret generation” or “Keycloak-generated secret extraction” as a steady-state writer).
   - Local: Vault KV may act as the source of truth (seeded from `.env/.env.local`) to keep local reproducible without cloud dependencies.
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

Some secret-like material is **owned and stored by a product’s database/state** and is not viable to “force into KV”:

- Coder External Auth tokens (stored in Coder DB per user)
- Session stores / internal signing keys generated and managed by the product (when applicable)

**Rule:** Treat this as **product state** (backup/restore, RBAC, retention), not as “a KV secret we sync with ExternalSecrets”.

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

2. **Public API Gateway URL** (browser-visible API origin)
   - Example: `https://api.dev.ameide.io`
   - Source of truth: GitOps values (`envoy.publicUrl`)
   - Output env: `NEXT_PUBLIC_ENVOY_URL`, `NEXT_PUBLIC_WS_URL` (see “Build-time vs runtime” below)

3. **Internal RPC URL** (server-to-server inside cluster)
   - Example: `http://envoy-grpc:9000`
   - Source of truth: GitOps values (`envoy.url`)
   - Output env: `AMEIDE_GRPC_BASE_URL`

4. **Identity Provider (IDP) Public Issuer**
   - Example: `https://auth.dev.ameide.io/realms/ameide`
   - Source of truth: GitOps values (`keycloak.issuer`)
   - Output env: `AUTH_KEYCLOAK_ISSUER`

5. **Identity Provider Internal Base** (optional)
   - Example: `http://keycloak:8080`
   - Source of truth: GitOps values (`keycloak.internalBase`)
   - Output env: `AUTH_INTERNAL_KEYCLOAK_BASE`, plus derived token/userinfo/jwks/revoke endpoints where needed.

### Environment Variable Categories

- **`*_URL` / `*_BASE_URL`**: must be classified as either public canonical, public API, internal RPC, or IDP.
- **`NEXT_PUBLIC_*`**: browser-reachable only. Never required for server-side RPC.
- **`AUTH_*`**:
  - Authoritative (Auth.js v5): `AUTH_*` are the canonical settings/inputs.
  - `AUTH_URL` is a setting (ConfigMap).
  - `AUTH_TRUST_HOST` is a required setting behind gateways/proxies (ConfigMap).
  - `AUTH_SECRET` and provider secrets are secrets (ExternalSecrets).
- **Test harness vars**:
  - CI/e2e-only vars must be prefixed or clearly documented.
  - Example: `AMEIDE_PLATFORM_BASE_URL` (CI/Playwright) is *not* a runtime setting.

### Build-time vs Runtime (Critical for Next.js + Digest Promotion)

Next.js `NEXT_PUBLIC_*` variables are typically **inlined at build time into browser bundles**. This creates a fundamental constraint:

- If we promote **one immutable image digest** across environments (dev → staging → prod), then any **environment-specific value used in browser JS must not rely on build-time inlining**, otherwise dev values can leak into prod bundles.

Policy:

- Any value that varies per environment and is consumed by **client-side code** must be provided via a **runtime mechanism** (e.g., server-rendered config, API endpoint, HTML bootstrap) or we must accept **per-environment builds** for that frontend artifact.
- Any `NEXT_PUBLIC_*` usage must be explicitly classified as either:
  - **Build-time input** (requires per-env build) or
  - **Runtime-provided to the browser** (do not rely on `NEXT_PUBLIC_*`).

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
  - gateway public/internal URLs (`envoy.publicUrl`, `envoy.url`)
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
  - Fail template render if Auth.js proxy requirements are violated (canonical URL present but `AUTH_TRUST_HOST` not set true in gateway deployments).
- Policy guardrails:
  - CI check rejects commits that add known secret patterns to values/configmaps.
  - CI check ensures `AUTH_URL` is set for any environment where auth is enabled.
  - CI check classifies `NEXT_PUBLIC_*` variables as build-time vs runtime and rejects environment-dependent “build-time” usage under digest promotion.

### Phase 2 – Reduce “duplicate URL knobs”

- Prefer a small set of source inputs in values:
  - `auth.url`, `auth.cookies.domain`
  - `auth.trustHost` (render `AUTH_TRUST_HOST`)
  - `envoy.url`, `envoy.publicUrl`, `websocket.publicUrl`
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
- ConfigMap contains required URL settings (canonical/public/internal) with expected formats.
- ExternalSecrets manifests reference KV keys and do not inline secret values.

### Runtime Verification (Cluster)

- ExternalSecrets sync healthy:
  - `Secret/www-ameide-platform-auth` exists and contains `AUTH_SECRET`, `AUTH_KEYCLOAK_SECRET`, `AUTH_KEYCLOAK_ADMIN_CLIENT_SECRET`.
- Cluster secret storage posture:
  - Secrets are encrypted at rest in etcd, and only the minimum service accounts can read the relevant Secrets.
- App pod has:
  - URL settings in env from ConfigMap,
  - secrets in env from Secret,
  - no secrets present in ConfigMap.
- Login flow:
  - Cookies scoped correctly (`AUTH_COOKIE_DOMAIN`) so sessions survive redirects.
- E2E/inner-loop tests:
  - `./ameide dev inner-loop-test` passes (mandatory agent check).

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

- Onboarding and cookie-domain alignment discussions: `319-onboarding-v2.md`.
- E2E base URL expectations: `371-e2e-playwright.md`.
- Domain standardization efforts: `366-local-domain-standardization.md`.
