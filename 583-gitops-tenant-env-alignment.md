# 583 – GitOps Tenant Env Alignment (`AMEIDE_TENANT_ID`)

**Status:** Draft  
**Owner:** GitOps  
**Motivation:** Align GitOps-rendered environment variables with the repo-wide tenant env standardization so cluster deployments fail fast (no “tenant magic”) and dev data seeding can be deterministic.  
**Related:** [582-local-dev-seeding.md](582-local-dev-seeding.md), [300-400/329-auth.md](300-400/329-auth.md), [300-400/330-dynamic-tenant-resolution.md](300-400/330-dynamic-tenant-resolution.md)

---

## 1) Background

Non-GitOps code is converging on a single canonical tenant env var:

- **Canonical:** `AMEIDE_TENANT_ID`
- **Goal:** remove scattered fallbacks like `DEFAULT_TENANT_ID`, `TENANT_ID`, `WORKFLOWS_TENANT_ID`, etc.

Today GitOps still emits the legacy vars in several charts and does **not** emit `AMEIDE_TENANT_ID`. Once the application changes land, this becomes a hard break: bootstrap/system calls will error because `AMEIDE_TENANT_ID` is missing.

This backlog is the GitOps-side work to keep cluster sync green while we eliminate tenant defaults in code.

---

## 2) Policy (keep it lean)

- `AMEIDE_TENANT_ID` is the **only** tenant env var GitOps should set going forward.
- Any legacy tenant env vars are **temporary compatibility shims** only (one rollout window), then removed.
- User-driven traffic must continue to use **JWT/session tenant context**; `AMEIDE_TENANT_ID` exists for **bootstrap/system contexts** (realm discovery/bootstrap jobs/background calls) where there is no user session.

---

## 3) Required GitOps changes (Phase A: compat)

For one release window, emit **both**:

- `AMEIDE_TENANT_ID` (new)
- the existing legacy var (old) so older images still work

### A. `platform` chart

**File (gitops repo):** `sources/charts/apps/platform/templates/configmap.yaml`  
**Current:** emits `DEFAULT_TENANT_ID`  
**Change:** also emit `AMEIDE_TENANT_ID` from the same value (`.Values.config.defaultTenant`).

### B. `www-ameide-platform` chart

**File (gitops repo):** `sources/charts/apps/www-ameide-platform/templates/configmap.yaml`  
**Current:** emits `DEFAULT_TENANT_ID`  
**Change:** also emit `AMEIDE_TENANT_ID` from the same value (`.Values.tenant.defaultId`).

### C. `workflows-runtime` chart

**File (gitops repo):** `sources/charts/apps/workflows-runtime/templates/configmap.yaml`  
**Current:** emits `WORKFLOWS_TENANT_ID`  
**Change:** also emit `AMEIDE_TENANT_ID` from the same value (`.Values.config.workflowsService.tenantId`).

### D. `alloy-gateway` shared values (telemetry tenant header)

**File (gitops repo):** `sources/charts/shared-values/infrastructure/alloy-gateway.yaml`  
**Current:** uses `DEFAULT_TENANT_ID` to set Tempo `X-Scope-OrgID` header  
**Change (recommended):**
- emit `AMEIDE_TENANT_ID` env (same value as today), and
- update the header to read `AMEIDE_TENANT_ID` (or coalesce `AMEIDE_TENANT_ID` then `DEFAULT_TENANT_ID`) during the compat window.

---

## 4) Cleanup (Phase B: remove legacy vars)

After all environments are running images that read `AMEIDE_TENANT_ID`:

- Remove `DEFAULT_TENANT_ID` from `platform` and `www-ameide-platform` charts.
- Remove `WORKFLOWS_TENANT_ID` from `workflows-runtime` chart.
- Remove `DEFAULT_TENANT_ID` from `alloy-gateway` and standardize Tempo tenant header on `AMEIDE_TENANT_ID`.

---

## 5) Verification checklist

- `helm template` renders `AMEIDE_TENANT_ID` in all affected ConfigMaps.
- Argo CD apps sync green post-rollout.
- `www-ameide-platform` bootstrap/system flows do not error with “`AMEIDE_TENANT_ID` environment variable is required but not set”.
- `workflows-runtime` can still call downstream services that require tenant context (callbacks/SDK calls).

---

## 6) Tracking

- [ ] Phase A: add `AMEIDE_TENANT_ID` to `platform` chart
- [ ] Phase A: add `AMEIDE_TENANT_ID` to `www-ameide-platform` chart
- [ ] Phase A: add `AMEIDE_TENANT_ID` to `workflows-runtime` chart
- [ ] Phase A: align `alloy-gateway` Tempo header env var
- [ ] Phase B: remove `DEFAULT_TENANT_ID` from charts
- [ ] Phase B: remove `WORKFLOWS_TENANT_ID` from charts
- [ ] Phase B: update docs/backlogs still referencing legacy vars (e.g., 582)

