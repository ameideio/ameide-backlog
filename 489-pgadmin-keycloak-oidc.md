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

### 2.4 CNPG server registry automation

- `sources/charts/platform/pgadmin-servers` (controller) watches `Cluster` and `Database` CRDs from CloudNativePG and continuously renders `ConfigMap/pgadmin-servers` with the 13 shared databases (`agents` … `workflows`).  
- Shared values (`sources/values/_shared/data/data-pgadmin.yaml`) set `.servers.enabled=true`, point the Deployment at that ConfigMap, and mount it at `/pgadmin4/servers.json`.
- **pgAdmin image constraint** – we have upgraded the mirror to 9.10 so the upstream `PGADMIN_REPLACE_SERVERS_ON_STARTUP=True` knob is available, but we still keep the init container to guarantee multi-user imports and preserve compatibility with any environments that roll back to 8.x.
- To keep things declarative we now ship an init container (`templates/deployment.yaml`) that:
  1. Shares the same PVC + ConfigMap volume as the main pod.
  2. Runs `/venv/bin/python3 /pgadmin4/setup.py load-servers /pgadmin4/servers.json --user "${PGADMIN_DEFAULT_EMAIL}" --replace`.
  3. Skips automatically when the SQLite DB has not been created yet (the primary container will then import servers on first boot).
- Using `--replace` ensures we always end up with **exactly** the definitions from the ConfigMap instead of endlessly appending duplicates when the controller emits updates.
- Toggle via values: setting `.servers.replaceOnStartup=false` disables the init container for ad‑hoc testing.
- `servers.additionalUsers[]` lets us specify extra pgAdmin identities (e.g., Keycloak SSO emails). The init container checks whether those users already exist in `pgadmin4.db` and, if so, runs `load-servers ... --replace` for each so their UI shows the same CNPG inventory once they have signed in at least once.
- Caveat: `setup.py load-servers ... --replace` wipes any credentials stored inside pgAdmin itself. Always back connection passwords with `pgpass`/secrets rather than the UI fields.

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
6. **Server registry import**
   - `kubectl -n ameide-<env> get configmap pgadmin-servers -o jsonpath='{.data.servers\.json}' | jq '.Servers | length'` → `13`.
   - `kubectl -n ameide-<env> exec deploy/pgadmin -- /venv/bin/python3 /pgadmin4/setup.py load-servers /pgadmin4/servers.json --user "$PGADMIN_DEFAULT_EMAIL" --replace --help` (dry run) confirms CLI availability.
   - Sanity-check the SQLite contents if the UI still shows “zero servers”:  
     `kubectl -n ameide-<env> exec -i deploy/pgadmin -- python - <<'PY' ... SELECT id,email FROM user; SELECT name FROM server; PY`.
   - Remember: servers belong to the pgAdmin user you import for. Today we target `pgadmin-admin-email` from Vault; other OIDC identities stay empty until we automate imports for them.

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
| Database registry automation | ✅ | `sources/charts/platform/pgadmin-servers` deploys a controller that reads CNPG Cluster/Database CRDs and publishes `ConfigMap/pgadmin-servers`. `sources/charts/platform/pgadmin/templates/deployment.yaml` now adds an init container that runs `setup.py load-servers ... --replace` on every restart so the mirrored 9.10 image always matches the ConfigMap. Until multi-user imports land, operators must log in using the bootstrap account (`pgadmin-admin-email`) to see the shared entries or run the CLI manually for their Keycloak email (or list their address under `.servers.additionalUsers`). |

---

### 9.1 Verification snapshot (Dec 9 2025)

- `argocd app get argocd/dev-data-pgadmin-servers` / `.../staging-data-pgadmin-servers` → **Synced/Healthy** after the security-context fix (commit `4266c0c3`).
- `kubectl -n ameide-{dev,staging} get configmap pgadmin-servers -o jsonpath='{.data.servers\.json}'` shows the expected thirteen database entries (platform, agents, …, workflows) pointing at `postgres-ameide-rw.<env>.svc` with per-role usernames.
- Controller logs confirm steady reconciliation: `Updated pgAdmin servers.json (...)` followed by periodic `No pgAdmin servers.json changes` messages.
- pgAdmin Deployment logs only show normal `/browser` traffic and liveness probes—no errors about missing `servers.json`. Browser tree emptiness now points to client cache issues, not backend wiring.

These checks serve as the regression baseline whenever someone reports missing databases in the UI.

---

## 10. Database registration (pgAdmin + CNPG)

We also need to capture the database-facing half of pgAdmin so operators stop assuming it is a purely manual UI workflow. Upstream pgAdmin already exposes declarative hooks for “registered servers”; our gap is the glue that turns CNPG resources into the JSON that pgAdmin consumes.

### 10.1 Vendor-supported declarative pgAdmin inputs

- **`servers.json` is the source of truth.** Mapping `/pgadmin4/servers.json` (or setting `PGADMIN_SERVER_JSON_FILE`) lets the container preload server definitions at startup, using the same schema as the Import/Export Servers commands (`setup.py load-servers` / `dump-servers`).
- **`PGADMIN_REPLACE_SERVERS_ON_STARTUP=True` makes it idempotent.** Newer pgAdmin releases (9.x) re-apply `servers.json` _every_ time the pod starts when that env var is set, which the docs explicitly call out as appropriate for declarative management—not just a one-time bootstrap.[1]
- **Passwords live elsewhere.** The JSON omits passwords for security; the supported approaches are a mounted pgpass file (`PGPASS_FILE`) or `PasswordExecCommand` inside the JSON to fetch credentials on the fly.[1][2]
- **Takeaways for this repo.** Our chart already manages OIDC + Vault secrets; we should add a ConfigMap/Secret mount for `servers.json` plus the env vars above so pgAdmin treats that file as authoritative instead of letting admins click through the UI.

### 10.2 What CloudNativePG does (and does not) provide

- **CNPG owns databases, not pgAdmin.** CNPG’s declarative Database/Cluster CRDs manage users, roles, and connection Secrets, but there is no upstream controller that registers clusters in pgAdmin or any GUI.[4]
- **There _is_ a helper CLI.** The `kubectl cnpg pgadmin4` recipe can emit a pgAdmin deployment that connects to a single CNPG cluster, but it’s a one-time manifest generator—there is no continuous reconciliation for a shared pgAdmin instance.[5]
- **Connection endpoints are deterministic.** Every CNPG cluster exposes `*-rw` / `*-ro` Services that we can derive from the Cluster metadata, so generating pgAdmin server entries is purely a rendering task.[6]

### 10.3 Proposed automation model

1. **Treat `servers.json` as an artifact.** Create a ConfigMap (or Secret if we embed `PasswordExecCommand`) that contains the pgAdmin server list. Mount it at `/pgadmin4/servers.json` and set:

   ```yaml
   env:
     - name: PGADMIN_SERVER_JSON_FILE
       value: /pgadmin4/servers.json
     - name: PGADMIN_REPLACE_SERVERS_ON_STARTUP
       value: "True"
   ```

2. **Keep credentials out of the JSON.** Either mount a pgpass file sourced from CNPG-generated Secrets or add `PasswordExecCommand` entries that exec a tiny script capable of reading those Secrets.
3. **Generate the file from CNPG CRs.** Options range from a CronJob that templates `servers.json` via `kubectl get cluster,database` to a lightweight controller that watches CNPG resources and rewrites the ConfigMap whenever clusters/databases or their connection Secrets change. The key requirement is that CNPG remains the source of truth; pgAdmin simply reflects it.
4. **Gate rollouts via Argo.** When the ConfigMap changes, Argo restarts pgAdmin (or Reloader does, courtesy of the existing annotation) so the new servers appear automatically.

This closes the last “manual” gap: pgAdmin becomes declarative on both the identity (Keycloak) _and_ database fronts once we land the generator that emits `servers.json` from CNPG resources.

[1]: https://www.pgadmin.org/docs/pgadmin4/latest/container_deployment.html
[2]: https://www.pgadmin.org/docs/pgadmin4/latest/import_export_servers.html
[4]: https://cloudnative-pg.io/documentation/1.25/declarative_database_management/
[5]: https://www.gabrielebartolini.it/articles/2024/03/cloudnativepg-recipe-4-connecting-to-your-postgresql-cluster-with-pgadmin4/
[6]: https://cloudnative-pg.io/documentation/current/quickstart/
