# 489 – pgAdmin Keycloak OIDC Alignment

**Status**: Draft  
**Created**: 2025-12-08  
**Owner**: Platform / Data Plane  
**Related**: [417-envoy-route-tracking.md](417-envoy-route-tracking.md) · [485-keycloak-oidc-client-reconciliation.md](485-keycloak-oidc-client-reconciliation.md) · [487-keycloak-client-secret-rotation.md](487-keycloak-client-secret-rotation.md) · [451-secrets-management.md](451-secrets-management.md)

---

## 1. Background

pgAdmin originally authenticated via Entra/Azure AD using oauthlib defaults. That approach diverged from the “environment-specific Keycloak realm is the IdP for internal tooling” direction captured in 471/476, and the Entra application was not maintained per-environment.  

Key issues prior to this work:

- **Placeholder clients** – Vault fixtures only contained `placeholder_pgadmin_client_id/secret`, so the pgAdmin Deployment never received a usable OIDC secret even though the chart expected it.
- **Manual identity** – Operators had to juggle multiple IdPs (Keycloak for most apps, Entra just for pgAdmin), making SSO policies inconsistent and breaking login in isolated environments (dev/staging) where Entra callback URLs were not whitelisted.
- **Runtime errors** – The UI showed `{"success":0,"errormsg":"'OAUTH2_TOKEN_URL'"}` followed by `Client not found` because pgAdmin tried to contact Keycloak with the placeholder client ID.

This backlog documents the now-working configuration and the guardrails needed to keep pgAdmin wired to Keycloak going forward.

---

## 2. Current architecture (Dec 2025)

### 2.1 Chart & values

- **Chart**: `sources/charts/platform/pgadmin`
- **Base values**: `sources/values/_shared/data/data-pgadmin.yaml`
  - `oidc.*` points at the in-cluster Keycloak service (`http://keycloak.<env>.svc`).  
  - External URL endpoints (`authorizationUrl`) use the public Keycloak ingress (`https://auth.<env>.ameide.io/...`).
  - ExternalSecret config maps `pgadmin-azure-oauth` → Vault keys `pgadmin-oidc-client-id` / `pgadmin-oidc-client-secret`.
- **Env-specific overrides**: `sources/values/<env>/data/data-pgadmin.yaml` override redirect URIs and Keycloak service hostnames per environment.

### 2.2 Identity + secrets

1. Keycloak realm values (`sources/values/<env>/platform/platform-keycloak-realm.yaml`)
   - Add `clientReconciliation.clients[]` entry for `pgadmin`.
   - Enable `clientPatcher.secretExtraction.clients[]` entry with `vaultKey: pgadmin-oidc-client-secret`.
2. Vault bootstrap fixture (`sources/values/_shared/foundation/foundation-vault-bootstrap.yaml`)
   - Seeds placeholder values for `pgadmin-oidc-client-id` (`pgadmin`) and `pgadmin-oidc-client-secret` so ESO can create the Secret even before Keycloak writes the real value.
3. `platform-keycloak-realm` PreSync Job (client-patcher)
   - Authenticates to Keycloak via the service account client (`keycloak-admin-sa`).
   - Ensures the `pgadmin` client exists and extracts the generated client secret.
   - Writes the secret to Vault (`secret/data/pgadmin-oidc-client-secret`) under the `keycloak-client-patcher` role.
4. ExternalSecret `pgadmin-azure-oauth-sync`
   - Watches Vault using the namespace-local SecretStore `ameide-vault`.
   - Materializes `Secret/pgadmin-azure-oauth` with keys `client-id` / `client-secret`.
5. pgAdmin Deployment
   - References the Secret for env vars `OAUTH2_CLIENT_ID` / `OAUTH2_CLIENT_SECRET`.
   - Mounts `config_local.py` (template ensures URLs exist; missing configuration now fails the template rendering instead of only surfacing at runtime).
   - Carries `reloader.stakater.com/auto: "true"` so Stakater Reloader automatically restarts the pod whenever `Secret/pgadmin-azure-oauth` changes (no more manual rollout restarts after rotation).

### 2.3 Gateway exposure

- HTTPRoute entries live under `sources/values/<env>/platform/platform-gateway.yaml` (platform domain) or `sources/values/<env>/data/data-pgadmin.yaml`.
- TLS hosts: `pgadmin.<env>.ameide.io`
- Service runs in namespace `ameide-<env>` with `clusterIP` Service `pgadmin`.

---

## 3. Verification steps

1. **Argo state**
   - `argocd app get argocd/dev-data-pgadmin` → Synced/Healthy, ConfigMap and Deployment configured, ExternalSecrets Ready.
   - `argocd app get argocd/dev-platform-keycloak-realm` → PreSync Job completed successfully (`platform-keycloak-realm-client-patcher` logs show “Processing client: pgadmin”).
2. **Vault contents**
   - From the environment namespace:  
     `kubectl --context ameide-dev-admin-admin -n ameide-dev run vault-read ... -- curl .../secret/data/pgadmin-oidc-client-secret` should return a non-placeholder base64 value.
3. **ExternalSecret & Secret**
   - `kubectl -n ameide-dev describe externalsecret pgadmin-azure-oauth-sync` → Ready=True.  
   - `kubectl -n ameide-dev get secret pgadmin-azure-oauth -o jsonpath='{.data.client-id}' | base64 -d` → `pgadmin`.
4. **Deployment pods**
   - `kubectl -n ameide-dev rollout status deployment pgadmin`.
   - Logs should show Keycloak redirect when hitting `/authenticate/login`. Example success log snippet:  
     `{"success": 0, "errormsg": "client ... requires login"}` replaced by Keycloak redirect + login UI.
5. **Browser test**
   - Visit `https://pgadmin.<env>.ameide.io/`. Should redirect to `https://auth.<env>.ameide.io/realms/ameide/protocol/openid-connect/auth?...client_id=pgadmin`.
   - After sign-in, pgAdmin loads with the Ameide SSO session.

---

## 4. Operational notes

- **Secret refresh** – If pgAdmin shows “Client not found,” it usually means Vault still holds the placeholder secret. Re-run the client-patcher Job (`kubectl delete job platform-keycloak-realm-client-patcher -n ameide-<env>`) or trigger a one-off job (see §5) to rewrite the secret from Keycloak.
- **PVC locking** – pgAdmin uses a single `ReadWriteOnce` PVC. When rolling the deployment, delete the old pod so the new replica can attach the volume faster (`kubectl delete pod ...`). We keep `replicaCount: 1` intentionally because pgAdmin isn’t stateless.
- **OIDC metadata** – Template validation enforces `metadataURL`, `authorizationUrl`, `tokenUrl`, `userinfoUrl`. If any Keycloak endpoint changes (e.g., different realm), update `_shared/data/data-pgadmin.yaml` first and then override per environment.
- **Groups & RBAC** – Default `allowedGroups` is empty; add Keycloak groups when we want to restrict pgAdmin to specific tenants (`oidc.allowedGroups[]`).  
- **Keycloak client** – Non-public, standard-flow enabled, redirect URIs limited to:
  - `https://pgadmin.dev.ameide.io/oauth2/authorize`
  - `https://pgadmin.dev.ameide.io`
  (per-environment overrides handled in values).

---

## 5. Manual remediation playbook

| Step | Command | Purpose |
|------|---------|---------|
| Delete stuck PreSync job | `kubectl delete job platform-keycloak-realm-client-patcher -n ameide-<env>` | Removes hook finalizer so Argo can recreate the Job. |
| Run targeted patcher (if Argo job unavailable) | Apply a temporary Job as shown in the Dec 2025 incident (script stored in `scripts/bootstrap-keycloak-temp-admin.sh` for reference). | Extracts pgAdmin client secret and writes it to Vault once. Delete the Job afterwards. |
| Force ESO refresh | `kubectl annotate externalsecret pgadmin-azure-oauth-sync external-secrets.io/refresh-trigger="$(date +%s)" --overwrite` | Reconciles pgAdmin Secret immediately if Vault already has the new value. |
| Restart pgAdmin | `kubectl rollout restart deployment pgadmin -n ameide-<env>` | Picks up new env vars (normally unnecessary because Reloader now handles Namespace restarts automatically). |

---

## 6. Follow-ups

1. **Argo hook stability** – Track down why the PreSync Job held an `argocd.argoproj.io/hook-finalizer`; ensure hooks complete cleanly so manual delete isn’t required (tied to backlog/487).
2. **Rotation observability** – Keep `keycloak-client-secret-versions` as telemetry only and extend `platform-secrets-smoke` assertions (mirroring backlog/487) so digest mismatches fail the release instead of mutating manifests.
3. **Reloader coverage** – ✅ DONE (deployment carries `reloader.stakater.com/auto` so secret updates automatically restart pods).
4. **Staging/Prod parity** – Confirm staging/prod values include the same Keycloak client entries and the secrets are non-placeholder.
5. **Runbook updates** – Update `451-secrets-management.md` to include the pgAdmin-specific verification steps above.

---

## 7. References & file map

- Keycloak realm values: `sources/values/{dev,staging,production}/platform/platform-keycloak-realm.yaml`
- Vault fixtures: `sources/values/_shared/foundation/foundation-vault-bootstrap.yaml`
- pgAdmin chart: `sources/charts/platform/pgadmin/`
- Shared pgAdmin values: `sources/values/_shared/data/data-pgadmin.yaml`
- Dev overlay: `sources/values/dev/data/data-pgadmin.yaml`
- ExternalSecret template: `sources/charts/platform/pgadmin/templates/externalsecret.yaml`
- Gateway exposure: `sources/values/{dev,staging,production}/platform/platform-gateway.yaml`
- Incident log (Dec 2025): `backlog/417-envoy-route-tracking.md` tracks the debugging steps leading up to this document.

---

## 8. Checklist (per environment)

| Item | Dev | Staging | Prod |
|------|-----|---------|------|
| `platform-keycloak-realm` includes `pgadmin` client spec | ✅ | ✅ | ✅ |
| `client-patcher` secretExtraction writes to Vault | ✅ (verified 2025-12-08) | ✅ (manual pgadmin-client-sync job 2025-12-08) | ✅ (manual pgadmin-client-sync job 2025-12-08) |
| ExternalSecret Ready + Secret non-placeholder | ✅ (`client-id=pgadmin`) | ✅ (secret rotated 2025-12-08) | ✅ (secret rotated 2025-12-08) |
| Deployment annotated for automatic restart | ✅ (platform-reloader handles rollouts) | ☐ | ☐ |
| Gateway host resolves & TLS valid | ✅ | ✅ (curl check 2025-12-08) | ✅ (curl check 2025-12-08) |
| Browser login via Keycloak succeeds | ✅ (user confirmation) | ☐ (needs manual user validation) | ☐ (needs manual user validation) |

Use this table as the acceptance checklist before closing out this backlog.

---

## 9. Implementation status (Jan 2026)

| Work item | Status | Notes |
|-----------|--------|-------|
| pgAdmin Deployment consumes Keycloak secrets via Vault → ESO → platform-reloader | ✅ | `sources/charts/platform/pgadmin/templates/deployment.yaml` now carries `reloader.stakater.com/auto: "true"` and reads the ExternalSecret-provided credentials end-to-end. |
| guardrail coverage in secrets smoke test | ✅ | `sources/values/_shared/platform/platform-secrets-smoke.yaml` verifies the `pgadmin-azure-oauth` secret’s digest against `keycloak-client-secret-versions`, keeping rotation telemetry read-only but enforced. |
| documentation parity with implementation | ✅ | This backlog reflects the deployed behavior (stating the reloader annotation and the telemetry-only ConfigMap usage) and is linked from backlog/487 for rotation completeness. |
| pending environment rollouts | ✅ | Dev/staging/prod now have live Keycloak clients, non-placeholder secrets, and healthy pods (see checklist above). |
