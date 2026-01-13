---
title: 642 – Camunda 8 Web Modeler + Identity (GitOps Enablement)
status: draft
owners:
  - platform
created: 2026-01-12
updated: 2026-01-13
---

## Summary

Enable **Camunda Web Modeler** in `ameide-gitops` in a **reproducible GitOps way** (no manual UI setup, no ad-hoc fixes), integrating with:

- **Keycloak SSO** (existing Ameide realm)
- **Gateway API** (Envoy Gateway) for HTTP + WebSocket routing
- **CNPG Postgres** for required persistent state
- **Azure Key Vault as source of truth** for secrets (materialized via External Secrets Operator)

This backlog is intentionally separated from the initial Camunda 8 baseline (`backlog/640-camunda-8-gitops-configuration.md`), which only deploys **Orchestration (Operate/Tasklist) + Connectors**.

## Access (dev)

Dedicated hosts (vendor-aligned; avoids path-based ambiguity and redirect/cookie issues):

- Web Modeler (UI): `https://modeler.dev.ameide.io/`
- Web Modeler (REST API): `https://modeler-api.dev.ameide.io/`
- Web Modeler (WebSockets): `wss://modeler-ws.dev.ameide.io/`
- Management Identity (UI): `https://identity.dev.ameide.io/`
- Operate/Tasklist (Orchestration apps): `https://camunda.dev.ameide.io/`

## Why it is not part of the baseline

Compared to Orchestration + Connectors, Web Modeler introduces additional platform requirements that must be solved “the one correct way”:

- **Management Identity dependency** (Web Modeler expects Identity for auth flows)
- **Database requirement** (Web Modeler requires Postgres; we do not deploy bundled Postgres subcharts)
- **SMTP requirement** (Web Modeler REST API requires mail settings; we need a deterministic dev/staging strategy)
- **WebSocket public routing** (Modeler websockets need explicit public host/port/path configuration under Gateway API)

## Goals

1. Web Modeler is enabled and usable in `dev` (and optionally `staging`), **not** in `production`.
2. User `admin@ameide.io` can log in via Keycloak and use Web Modeler without manual setup steps.
3. All configuration is GitOps-owned (charts/values/secrets keys documented) and converges via ArgoCD.
4. Secrets are sourced from **Azure Key Vault** and materialized via ExternalSecrets; no secret values in Git.

## Non-goals

- Buying/using any paid Camunda licenses or commercial-only modules.
- Introducing manual operator steps as part of the “standard deploy”.
- Replacing the baseline Camunda stack.

## Vendor-aligned auth model (must match chart defaults)

Web Modeler is not “just another redirect URL”. The upstream model requires that the browser UI can establish an authenticated session with the Web Modeler REST API, which in turn depends on:

- Correct redirect URIs for the Web Modeler UI client (e.g. `https://modeler.dev.ameide.io/login-callback`).
- Correct audience configuration:
  - `clientApiAudience` (default `web-modeler-api`) used for UI → REST API communication.
  - `publicApiAudience` (default `web-modeler-public-api`) only for the public API, not required for basic UI login.

## GitOps incident note (dev): Web Modeler login loop root cause + fix

Observed behavior:

- Web Modeler (`modeler.dev.ameide.io`) loops continuously to Keycloak login.
- Attempting to log into Modeler could also invalidate the Keycloak session used by Operate (appearing as “logged out”).

Root cause (confirmed from Web Modeler REST API logs):

- `POST /internal-api/login` returned `401` because the Keycloak **access token** for `web-modeler` was missing the `sub` claim (`Claim SUB not found`).
- Web Modeler UI posts `iamId=sub` to `/internal-api/login`; without `sub` the REST API rejects authentication, causing the login loop.

Fix (GitOps-owned, vendor-compatible):

- Ensure the Keycloak realm `profile` client-scope includes a protocol mapper that emits `sub` into **access tokens**.
- Ensure the Keycloak client reconciler is convergent (detaches client-scopes not in the desired spec) so “removed scopes” don’t linger and regress behavior.

Follow-up issue (dev): “Could not create the new project”

- Symptom: Modeler UI shows “Yikes! Could not create the new project” and REST API returns `404` for `POST /internal-api/projects` (the REST API intentionally returns 404 on failed org authorization).
- Root cause (vendor-aligned): Web Modeler authorizes org access based on **`permissions.<audience>`** in the access token. For the UI → REST API flow, the relevant audience is `web-modeler-api`, and the token must include permissions like `write:*` (and optionally `admin:*`) under `permissions.web-modeler-api`.
- Fix (GitOps-owned, standard Keycloak): implement the Camunda Identity / Web Modeler token contract using **standard Keycloak protocol mappers** and **client roles**:
  - Create a Keycloak client-scope `camunda-identity` with protocol mappers that emit:
    - `permissions.web-modeler-api` from client roles on client `web-modeler-api`
    - `permissions.web-modeler-public-api` from client roles on client `web-modeler-public-api`
  - Ensure the `web-modeler` UI client has `camunda-identity` attached as a default scope.
  - Create client roles `web-modeler-api:write:*` and `web-modeler-api:admin:*` and assign them to the platform admin group `/argocd-admin` (dev posture).

Implementation note (where this lives in GitOps):

- Keycloak reconciliation hook job: `sources/charts/foundation/operators-config/keycloak_realm/templates/client-patcher-job.yaml`
- Client specs: `sources/values/env/dev/platform/platform-keycloak-realm.yaml` (and env equivalents)

Operational note:

- Tokens already issued before this change will not gain the new claims; use an incognito window or clear site data for `modeler.dev.ameide.io` after the GitOps sync.

Operator note:

- After the fix is applied, users may need to clear site data / use an incognito window to avoid stale tokens.

## Required design decisions (must be fixed in this backlog)

### 1) Exposure model (hosts + paths)

Define the public URL scheme for:

- Web Modeler webapp
- Web Modeler restapi
- Web Modeler websockets

This must be compatible with the upstream chart’s expectations for:

- `global.identity.auth.webModeler.redirectUrl` (external base URL)
- websocket public host/port/path configuration

Explicitly: do **not** expose Web Modeler under `https://camunda.dev.ameide.io/modeler` (path-based) because the vendor auth model and redirect URIs assume a distinct Web Modeler “application base URL” and we want to avoid cross-app cookie/session ambiguity.

### 2) SMTP strategy

Pick an explicit dev/staging SMTP strategy that is reproducible:

- Use an existing platform SMTP provider if available, or
- Introduce a lightweight in-cluster SMTP service for non-production (GitOps-managed), or
- Configure “no-op/blackhole” SMTP for dev-only where supported (no notifications required for MVP).

### 3) Database strategy (CNPG)

Implement Web Modeler Postgres using CNPG:

- create database + role (Vault/ESO managed, no plain-text)
- wire `webModeler.restapi.externalDatabase.*` (or chart-supported secret refs)
- keep `webModelerPostgresql.enabled=false`

### 4) Keycloak integration

Add required Keycloak clients + redirect URIs for Web Modeler and (if needed) Management Identity:

- clients must be reconciled by the Keycloak realm GitOps flow
- client secrets must be extracted to Key Vault (then to K8s via ExternalSecrets)

### 5) Wrapper chart changes

Extend the `platform/camunda8` wrapper chart with:

- the extra `HTTPRoute`(s) needed for webapp/restapi/websockets
- ExternalSecrets for any additional client secrets / DB credentials / SMTP credentials
- environment enablement matrix (dev/staging on; prod off)

### 6) Version pinning

- Pin the Camunda Helm chart version in Git (no `latest`).
- Treat “upgrade to latest chart” as an explicit change with its own validation/smoke steps.

## Acceptance criteria

- In `dev`, `admin@ameide.io` can open Web Modeler, authenticate via Keycloak, and reach:
  - web UI
  - REST API (successful `/internal-api/login` and a stable authenticated session)
  - websockets endpoint (basic connectivity check)
- In `production`, Web Modeler and Identity are not deployed and no routes exist.
- A smoke test exists that validates the public routes and auth redirect correctness, **wired through ArgoCD** (PostSync hook Jobs), not CI:
  - Web Modeler UI: `https://modeler.dev.ameide.io/` returns `302` to `/login`
  - Web Modeler REST API: `https://modeler-api.dev.ameide.io/` returns `401`
  - Web Modeler WebSockets host: `https://modeler-ws.dev.ameide.io/` returns `404` (basic route is live)
  - Identity UI: `https://identity.dev.ameide.io/` returns `302` to `/auth/login`

Implementation location (dev today):

- Values: `sources/values/env/dev/apps/platform-smoke.yaml`
- Runner chart: `sources/charts/foundation/platform-smoke`
