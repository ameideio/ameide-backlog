# 583 – GitOps Tenant Env Alignment (`AMEIDE_TENANT_ID`)

**Status:** Implemented (fail-fast target-state)  
**Owner:** GitOps  
**Motivation:** Align GitOps-rendered environment variables with the repo-wide tenant env standardization so cluster deployments fail fast (no “tenant magic”) and dev data seeding can be deterministic.  
**Related:** [582-local-dev-seeding.md](582-local-dev-seeding.md), [300-400/329-auth.md](300-400/329-auth.md), [300-400/330-dynamic-tenant-resolution.md](300-400/330-dynamic-tenant-resolution.md)

---

## 1) Background

Non-GitOps code is converging on a single canonical tenant env var:

- **Canonical:** `AMEIDE_TENANT_ID`
- **Goal:** remove scattered legacy vars like `DEFAULT_TENANT_ID`, `TENANT_ID`, `WORKFLOWS_TENANT_ID`, etc.

GitOps now emits **only** `AMEIDE_TENANT_ID` and requires explicit values so the cluster fails fast at render/sync time if tenant context is missing.

---

## 2) Policy (keep it lean)

- `AMEIDE_TENANT_ID` is the **only** tenant env var GitOps should set going forward.
- User-driven traffic must use **issuer-first tenant routing** (OIDC issuer URL → server-side `issuer → tenant_id` mapping; see `backlog/597-login-onboarding-primitives.md`). `AMEIDE_TENANT_ID` exists only for **bootstrap/system contexts** where there is no authenticated principal.
- **No tenant-specific GitOps requirement:** GitOps sets a single bootstrap tenant ID per environment; it does not encode per-tenant routing for user traffic.
- Canonical dev tenant ID format is `tenant-<slug>` (e.g. `tenant-atlas`) to align with platform validation.

---

## 3) GitOps changes (implemented)

### A. Remove legacy env vars

- Removed `DEFAULT_TENANT_ID` from charts and standardized on `AMEIDE_TENANT_ID` only.
- Removed `WORKFLOWS_TENANT_ID` from `workflows-runtime` and standardized on `AMEIDE_TENANT_ID` only.

### B. Fail fast on missing tenant config

- `platform`: `AMEIDE_TENANT_ID` is required (`config.defaultTenant`).
- `graph`: `AMEIDE_TENANT_ID` is required (`config.defaultTenant`).
- `workflows-runtime`: `AMEIDE_TENANT_ID` is required (`config.workflowsService.tenantId`).
- `www-ameide-platform`: `AMEIDE_TENANT_ID` is required (`tenant.defaultId`).

### C. Observability tenant header

- `alloy-gateway` now uses `AMEIDE_TENANT_ID` exclusively for Tempo `X-Scope-OrgID`.

---

## 5) Verification checklist

- `helm template` renders `AMEIDE_TENANT_ID` in all affected ConfigMaps.
- Argo CD apps sync green post-rollout.
- `www-ameide-platform` bootstrap/system flows do not error with “`AMEIDE_TENANT_ID` environment variable is required but not set”.
- `workflows-runtime` can still call downstream services that require tenant context (callbacks/SDK calls).

---

## 6) Tracking

- [x] Remove `DEFAULT_TENANT_ID` from charts
- [x] Remove `WORKFLOWS_TENANT_ID` from charts
- [x] Standardize on `AMEIDE_TENANT_ID` only
- [x] Require explicit tenant values (fail fast)
