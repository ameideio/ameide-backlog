# 703 – Platform default organization must be derived from Keycloak (remove GitOps hardcoding)

**Status:** Proposed  
**Owner:** Platform Auth + Frontend  
**Motivation:** The “default organization” used by `www-ameide-platform` must not be configured via GitOps values; it must be derived from the authenticated principal (Keycloak-issued claims / session), so cluster config stays environment-only and tenant/org routing stays identity-driven.

---

## Problem statement

`www-ameide-platform` currently has a startup/Auth.js callback invariant that requires one of:

- `NEXT_PUBLIC_DEFAULT_ORG`
- `AUTH_DEFAULT_ORG`
- `WWW_AMEIDE_PLATFORM_ORG_ID`

When missing, Auth.js fails the callback route and users (and CI verifiers) see:

- `.../api/auth/error?error=Configuration`

This violates the desired contract documented in:

- `backlog/492-telepresence-verification.md` (“Tenant + organization context must come from the auth session; org-default env vars are deprecated/test-only.”)

GitOps applied a stopgap to keep production SSO verification green by emitting these env vars from `organization.defaultId`, but this is not a stable ideal configuration.

---

## Desired behavior (target-state)

- GitOps sets **environment wiring only** (URLs, cookie domains, Keycloak issuer/endpoints, Redis, etc.).
- `www-ameide-platform` derives tenant/org context from the **auth session** (JWT + middleware headers), not from GitOps values.
- “Default org” selection is identity-driven and deterministic:
  - If the token/session includes an explicit default org claim, use it.
  - Else, resolve organizations from platform APIs and pick a deterministic default (e.g., smallest slug, newest membership, or “home org” policy), then persist it in the session.

---

## Proposed implementation

### A) Keycloak: emit a default org claim

Add a Keycloak protocol mapper (client scope or per-client) that emits a claim such as:

- `ameide_default_org` (string slug or stable org id)

Source options (choose one, keep it explicit):

1. User attribute (e.g. `default_org=atlas`)
2. Group attribute / role mapping (e.g. first matching group → org)
3. Realm-level config attribute for “bootstrap org” for service accounts (avoid for user traffic)

### B) Frontend: stop requiring env var; read from session/token

- Remove the hard requirement that forces `*_DEFAULT_ORG` env vars to exist.
- Prefer claim(s) from the session token over env vars.
- Keep env vars only as temporary compatibility fallback (and log a deprecation warning when used).

### C) GitOps: remove org default from desired state

Once the application no longer requires it:

- Remove `organization.defaultId` from `sources/values/env/*/apps/www-ameide-platform.yaml`.
- Stop emitting `WWW_AMEIDE_PLATFORM_ORG_ID` / `AUTH_DEFAULT_ORG` / `NEXT_PUBLIC_DEFAULT_ORG` from the chart.
- Keep only `AMEIDE_TENANT_ID` as the bootstrap “no-auth context” input (per `backlog/583-gitops-tenant-env-alignment.md`).

---

## Incident reference (why this exists)

- 2026-01-19: production SSO verify failed due to missing default-org env var; GitOps stopgap PR added env injection to restore green. This backlog item defines the exit plan to remove that stopgap.

---

## Acceptance criteria

- Platform SSO verification passes in dev/staging/prod with **no org-default env vars** set by GitOps.
- `www-ameide-platform` logs no longer show “DEFAULT_ORG … is required but not set”.
- Backlog docs updated to treat org-default env vars as removed (not merely deprecated).
