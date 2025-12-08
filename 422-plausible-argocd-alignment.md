# 422: Plausible ArgoCD alignment (dev)

## What Plausible now does

- **External DBs only.** `DATABASE_URL` points to the CNPG Postgres service (`platform-postgres-clusters-rw.ameide.svc:5432/plausible`) using the CNPG-owned secret `plausible-db-credentials` (username/password keys). `CLICKHOUSE_DATABASE_URL` / `_READ` point to the Altinity ClickHouse HTTP endpoint on 8123 with creds from `clickhouse-auth` (user `plausible`).
- **Vendor init flow enabled.** Deployment now ships an initContainer that runs `/entrypoint.sh db createdb && /entrypoint.sh db migrate` before `run`. This creates/updates both Postgres and ClickHouse schema exactly as the Plausible image expects.
- **Secrets aligned with CNPG ownership.** The Deployment no longer consumes the cluster admin secret `postgres-ameide-auth`; it uses the per-app CNPG secret (`plausible-db-credentials`), satisfying backlog/412 (CNPG owns Postgres creds).
- **Flyway opt-out.** Plausible is removed from the Flyway image and db-migrations Job; schema creation/migrations are solely handled by the Plausible entrypoint. Remaining db-migrations values (local/staging/production) no longer list `plausible-db-credentials`, so Flyway stops mounting Plausible creds entirely.
- **Keycloak-backed SSO via oauth2-proxy.** The Plausible HTTPRoute now targets a dedicated `plausible-oauth2-proxy` Deployment (Bitnami chart) that performs OIDC code flow against Keycloak. The proxy skips auth on `/js/` and `/api/event` so visitor tracking remains anonymous, but every other route (dashboard, onboarding, `/api` UI) requires an Ameide SSO session. Secrets come from Vault via `ExternalSecret/plausible-oauth2-proxy`, which is fed by the Keycloak-generated client secret (`plausible-oidc-client-secret`) and a random cookie secret (`plausible-oauth-cookie-secret`). Keycloak itself owns the `plausible` client through `client-patcher`, keeping configuration declarative.

## Issues encountered

1) **Schema ownership conflict**
   - Flyway tried to own Plausible schema via `plausible-db-credentials`, while the runtime used the shared admin secret. ClickHouse schema was unmanaged because Plausible entrypoint migrations were never run.
   - Fix: Runtime now uses `plausible-db-credentials` and an initContainer to run Plausible’s entrypoint migrations; Flyway no longer touches Plausible.

2) **ArgoCD deletion loop on apps-plausible**
   - The Application was stuck deleting (seed Job finalizer), leaving Deployment missing.
   - Fix: Cleared finalizers, let the ApplicationSet recreate; now Synced/Healthy with the initContainer in place.

3) **OIDC onboarding still broken**
   - Even after oauth2-proxy fronting Plausible, Keycloak logins often bounce users to `/register` and the proxy emits `email ... isn't verified` errors for unverified accounts.
   - Status: unresolved (tracked alongside the rotation backlog in 487). Until Keycloak verification policy is sorted out, Plausible SSO cannot be considered production-ready.

## How to verify (dev)

- `argocd app get argocd/apps-plausible` should show Synced/Healthy and the Deployment spec containing `initContainers: plausible-db-init` with the entrypoint command.
- Pods should start with the init phase completing and main container running `run`; ClickHouse/Postgres should have the Plausible tables post-start.
- No Plausible-related Flyway locations or secrets remain in `data-db-migrations` values or Job template.
- `argocd app get argocd/apps-plausible-oauth2-proxy` should show Synced/Healthy and the oauth2-proxy Deployment wired to the Keycloak issuer (`https://auth.<env>.ameide.io/realms/ameide`) with `--skip-auth-route` entries for `/js/` and `/api/event`.
- Visiting `https://plausible.<env>.ameide.io` should redirect through Keycloak, while `https://plausible.<env>.ameide.io/js/script.js` and `https://plausible.<env>.ameide.io/api/event` remain publicly accessible (no login prompt, correct CORS headers).

## Guardrails / follow-ups

- Keep Plausible schema exclusively managed by Plausible entrypoint migrations; do not reintroduce it into Flyway.
- Mirror the initContainer enablement and CNPG secret wiring in any non-dev overlays.
- Monitor first rollout after image bumps to ensure `/entrypoint.sh db migrate` succeeds against ClickHouse/Postgres.***
- Keep oauth2-proxy enabled for every environment (dev/staging/prod): Plausible must never be exposed publicly without SSO, and `/js/` + `/api/event` are the only paths allowed to bypass auth.
- When onboarding new environments, add the Plausible OIDC client to the Keycloak realm values (`plausible` clientId, redirect URIs, `client-patcher` spec) and seed the Vault placeholders (`plausible-oidc-client-id`, `plausible-oidc-client-secret`, `plausible-oauth-cookie-secret`) instead of creating ad-hoc secrets.
