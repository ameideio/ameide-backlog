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
- There are **no deterministic defaults** for organization selection:
  - If the user has an explicit, user-configured preference/claim (e.g., `ameide_active_org`), it may be used.
  - Otherwise, the app must prompt the user to select an organization (and only then persist the selection in the session).

---

## Proposed implementation

### A) Keycloak: emit org context (membership and/or explicit preference)

**Preferred contract (vendor feature): Keycloak Organizations**

If we adopt Keycloak’s Organizations feature, we should lean on its documented scope/claim semantics rather than inventing new claims:

- Use an “all orgs” membership scope (e.g. `organization:*`) to obtain the full membership list without forcing selection at login.
- Only use an “active org” scope (e.g. `organization` / `organization:<alias>`) if we explicitly want Keycloak to drive org selection during authentication.

The important invariant remains: **no implicit selection when multiple orgs exist**.

**Current reality (already in GitOps): group-derived membership claim**

Today the Keycloak realm import already defines a client scope named `organizations` that maps group membership into an `org_groups` claim (full group paths). This is sufficient to carry “org membership list” without adopting the Organizations feature yet.

If we keep this path:

- Treat `org_groups` as the membership list input.
- Use explicit user choice to set `activeOrg` in the app session (never infer it from ordering).

Add Keycloak protocol mappers (client scope or per-client) that emit:

- `ameide_orgs` (list of org slugs/ids) OR reuse `groups` as the membership carrier
- optionally `ameide_active_org` (explicit per-user preference, not inferred by sorting)

Source options (choose one, keep it explicit and non-inferred):

1. User attributes (e.g. `ameide_orgs=["atlas","..."]`, `ameide_active_org="atlas"`)
2. Group membership mapping (`/orgs/<slug>` groups) → `groups` claim, with app-side parsing
3. Service accounts: prefer explicit user attributes over realm-wide fallbacks; avoid realm-level “bootstrap org” for user traffic

### B) Frontend: stop requiring env var; read from session/token

- Remove the hard requirement that forces `*_DEFAULT_ORG` env vars to exist.
- Prefer org membership + explicit preference claims from the session token over env vars.
- If no active org is present and membership is ambiguous, redirect to an organization selection flow (no implicit selection).
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
