# 424 – Tilt-only releases (www + platform) to avoid Argo/Tilt tug-of-war

**Status:** Implemented  
**Intent:** Fully separate Tilt’s inner-loop releases from Argo’s GitOps baselines. Tilt owns `*-tilt` releases (distinct names/hosts/secrets); Argo owns `apps-*`. No shared ownership, no adoption/pause/orphan dance.

---

## Why we changed this
- **Symptom:** Tilt applies failed with image/ownership drift because Argo auto-sync kept restoring the GitOps image and metadata on shared releases.
- **Root causes:** ApplicationSet reasserts `syncPolicy.automated`; adoption/pause/guardrails were fragile and created ownership churn.
- **Solution:** Isolation. Give Tilt its own release names, hosts, and (when needed) secrets, and stop touching Argo-managed resources.

---

## Final architecture (www + platform)
- **Baseline (Argo):** `apps-www-ameide`, `apps-www-ameide-platform` remain Argo-owned with GitOps images/tags and existing secrets/ExternalSecrets.
- **Tilt-only:** `apps-www-ameide-tilt`, `apps-www-ameide-platform-tilt` are Helm’d by Tilt with Tilt image tags and distinct names/hosts.
- **Ownership:** Argo is unaware of `*-tilt`; Tilt never deploys or adopts baseline releases.
- **Keycloak (shared instance):** Remains Argo-owned and shared across dev + Tilt. Both baseline and Tilt use the same realm/clients; Tilt reuses Argo auth secrets and continues to use the canonical issuer `https://auth.dev.ameide.io/realms/ameide`, while Envoy routes both `auth.dev.ameide.io` and `auth.local.ameide.io` to the same Keycloak service.

---

## What we changed (code/config)
### Tiltfile
- Only `*-tilt` Helm releases are defined for web/apps; integration deps updated to `*-tilt`.
- Removed Argo pause, guard, adoption, and orphan preflight; Helm apply helper now runs a plain `helm upgrade --install`.
- SDK deps scoped per service (TS/Go/Py) instead of a blanket guardrail.

### Values and routing
- Added tilt values for all apps: `gitops/ameide-gitops/sources/values/dev/apps/apps-*-tilt.yaml` with `nameOverride/fullnameOverride=<service>-tilt`, Tilt registries/tags.
- Disabled ExternalSecrets in all tilt values; services reuse Argo-managed secrets via `existingSecret` (e.g., `www-ameide-auth`, `www-ameide-platform-auth`, `inference-db-credentials`).
- Hosts/HTTPRoutes:
  - `www.local.ameide.io` → HTTPRoute in `apps-www-ameide-tilt` (gateway `ameide`, section `https-local`).
  - `platform.local.ameide.io` → HTTPRoute in `apps-www-ameide-platform-tilt` (gateway `ameide`, section `https-local`).
  - Other tilt services stay internal unless hosts are added to their tilt values.
  - Keycloak in dev: `auth.dev.ameide.io` (canonical issuer) and `auth.local.ameide.io` (local alias) both route via Envoy to the shared Keycloak instance; apps in dev and Tilt are configured to use the `auth.dev.ameide.io` issuer.
- Redis auth: tilt platform values now wire `redis.passwordSecret` (`redis-auth/password`) to avoid NOAUTH and keep parity with Argo releases. No inline secrets introduced.

### DNS / host resolution
- With the remote-first architecture, developers hit the public `*.ameide.io` domains directly (Envoy in AKS serves both `dev` and `local` aliases), so the old dnsmasq helper was removed. Telepresence handles routing cluster traffic back to the local process without requiring host-level resolver tweaks.

---

## How to use (developer loop)
1) Run `tilt up -- --only=apps-www-ameide-tilt,apps-www-ameide-platform-tilt` (or include other `*-tilt` services you need).  
2) Browse:
   - Argo baseline: `https://www.dev.ameide.io`
   - Tilt web: `https://www.local.ameide.io`
   - Tilt platform: `https://platform.local.ameide.io`
3) Auth/Keycloak secrets are reused; no extra secret provisioning for tilt releases.
4) Argo will keep reconciling baseline releases, but Tilt no longer touches them.

---

## Notes and limitations
- Tilt releases do not create or adopt ExternalSecrets; they rely on Argo-managed secrets being present in the namespace. If a secret is missing, create it via Argo/GitOps first.
- Runtime configuration for `apps-www-ameide-platform-tilt` (e.g., `AUTH_URL`, `AUTH_KEYCLOAK_ISSUER`, `AUTH_SECRET`, Redis/DB URLs) is derived from the same Helm values/ExternalSecrets chain as the Argo baseline. The devcontainer must not be treated as a source of truth for these app env vars.
- If you need external access for other tilt services, add unique hosts in their tilt values and ensure DNS covers them.
- Cleanup of `*-tilt` releases on Tilt stop is not automated (optional follow-up).
- Keycloak client (`platform-app`) must allow local host callbacks: `platform.local.ameide.io` is present in redirect/web origins in the shared realm values (see 426-keycloak-config-map.md), and the Next.js dev server allows the local origin via `allowedDevOrigins`.

---

## Follow-ups (optional)
- Add tilt hosts for other services as needed.
- Optional hook to delete `*-tilt` releases when Tilt stops.
