> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# Backlog 323: Keycloak Realm Roles (v2)

**Status**: Single-realm implementation live; realm-per-tenant migration in progress
**Last Updated**: 2025-12-07
**Related**: 320-header.md · 322-rbac.md · 330-dynamic-tenant-resolution.md · 333-realms.md · 426-keycloak-config-map.md

---

## 1. Why This Matters
- RBAC enforcement depends on Keycloak tokens exposing user roles. Initial rollout lacked role claims, disabling protected navigation.
- Multi-tenancy requires tenant context in tokens until realm-per-tenant rollout completes.
- Realm-per-tenant target simplifies configuration but preserves the `roles` scope pattern.

---

## 2. Current Production Baseline (Single Realm `ameide`)

> **Path note (Nov 2025):** Realm configuration now lives at `gitops/ameide-gitops/sources/charts/foundation/operators-config/keycloak_realm` with environment overlays in `gitops/ameide-gitops/sources/values/<env>/platform/platform-keycloak-realm.yaml`. Historical references to `infra/kubernetes/charts/platform/keycloak-realm` refer to the retired location.
- **Client scopes**: `roles`, `tenant`, `groups` (all defaulted everywhere).
- **`roles` scope** adds both realm and client roles:

  ```yaml
  {
    "name": "roles",
    "protocol": "openid-connect",
    "attributes": {"include.in.token.scope": "true"},
    "protocolMappers": [
      {
        "name": "realm roles",
        "protocolMapper": "oidc-usermodel-realm-role-mapper",
        "config": {
          "claim.name": "realm_access.roles",
          "multivalued": "true",
          "access.token.claim": "true",
          "id.token.claim": "true",
          "userinfo.token.claim": "true"
        }
      },
      {
        "name": "client roles",
        "protocolMapper": "oidc-usermodel-client-role-mapper",
        "config": {
          "claim.name": "resource_access.${client_id}.roles",
          "multivalued": "true",
          "access.token.claim": "true",
          "id.token.claim": "true",
          "userinfo.token.claim": "true"
        }
      }
    ]
  }
  ```
- **`tenant` scope** maps Keycloak user attribute `tenantId` → token claim `tenantId`; required while multiple tenants share the realm.
- **JWT (after fixes)** now includes `realm_access.roles`, `resource_access.{client}`, and `tenantId` — unlocks RBAC checks in `lib/keycloak.ts` and `features/navigation/server/access.ts`.
- **Secrets ownership**: `platform-postgres-clusters` (CNPG) emits the `keycloak-db-credentials` Secret. Non-database secrets (`keycloak-bootstrap-admin`, `keycloak-master-bootstrap`, `platform-app-master-client`) are provisioned by the Layer‑15 component `foundation-keycloak-admin-secrets`. The `platform-keycloak` chart consumes those `existingSecret`s directly; no chart-scoped ExternalSecrets remain.
- **Client secret extraction**: After realm sync, the `client-patcher` Job extracts Keycloak-generated client secrets (e.g., `platform-app`, `argocd`) to Vault. See [426-keycloak-config-map.md §3.2](426-keycloak-config-map.md).

---

## 3. Application Integration Notes
- `extractRoles(decoded)` aggregates `realm_access.roles` and `resource_access` roles, de-duplicated.
- JWT callback enriches the session with `user.roles`, `user.isAdmin`, and `user.tenantId` (pulled from token claims).
- 40+ routes now resolve tenant context via `session.user.tenantId` (see backlog/330-dynamic-tenant-resolution.md).
- Manual verification: password grant + `jq` to inspect token; integration tests cover RBAC navigation paths.

---

## 4. Target State (Realm-per-Tenant)
- **Realm becomes the tenant**: `iss = https://keycloak/realms/{orgSlug}`; drop `tenant` & `groups` scopes entirely.
- **Only keep `roles` scope**; remove the client-role mapper when there is a single application per realm.
- **Users managed per realm** or federated from the tenant's IdP; no custom attributes required.
- **JWT** shrinks to standard claims plus `realm_access.roles`.
- **Code simplifications**:
  - Tenant derives from issuer (`extractTenantFromIssuer` helper).
  - Role extraction reads only `realm_access.roles`; delete `resource_access` iteration and `extractOrgGroups`.
  - Multi-realm NextAuth provider resolves realm from request (path or subdomain) before calling Keycloak.

---

## 5. Migration Outline
1. **Template**: Ship `keycloak-realm-template` Helm chart with minimal client scope set.
2. **Provisioning**: Implement automated realm creation (Keycloak Admin API) when new organizations are registered.
3. **Service updates**: add realm resolution, issuer-based tenant detection, and org↔realm mapping persistence.
4. **Testing**: extend integration/E2E coverage for multi-realm token validation and isolation.
5. **Cutover** (per tenant): create realm, migrate users, switch issuer, monitor; decommission shared realm after final tenant moves.

---

## 6. Key Risks & Mitigations
- **Auth downtime during cutover** → keep single realm as fallback; feature-flag multi-realm mode; monitor failure rate >5%.
- **Token issuer mismatches** → strict issuer validation, temporary dual-issuer acceptance, instrument logs/metrics.
- **Provisioning failures** → idempotent Keycloak Admin API wrapper with retries; document manual recovery flow.
- **Data loss** → backup Keycloak DB before migration; dry-run on staging; validate user counts pre/post move.

---

## 7. Success Criteria
- Client scopes reduced from 4 → 1; protocol mappers 6 → 1; YAML trimmed ~250 → <40 lines.
- JWT custom claims drop from 5 → 1; token size shrinks by ~50%.
- Extraction utilities cut from 3 to 2 functions; session payload depends only on issuer + realm roles.
- Realm creation succeeds >99%; auth failure rate stays <0.1% during rollout.

---

## 8. Useful Commands & References
- Inspect tokens:
  ```bash
  curl -s -X POST "https://auth.dev.ameide.io/realms/ameide/protocol/openid-connect/token" \
    --data-urlencode "grant_type=password" \
    --data-urlencode "client_id=platform-app" \
    --data-urlencode "client_secret=changeme" \
    --data-urlencode "username=admin" \
    --data-urlencode "password=***" | jq -r '.access_token' \
    | cut -d'.' -f2 | base64 -d | jq '{tenantId, realm_access}'
  ```
- Helm deployment (single realm baseline):
  ```bash
  helm upgrade keycloak-realm ./gitops/ameide-gitops/sources/charts/foundation/operators-config/keycloak_realm \
    -n ameide \
    -f ./gitops/ameide-gitops/sources/values/_shared/platform/platform-keycloak-realm.yaml \
    -f ./gitops/ameide-gitops/sources/values/env/dev/platform/platform-keycloak-realm.yaml
  ```
- Backups prior to migration:
  ```bash
  kubectl exec -n ameide keycloak-0 -- \
    /opt/keycloak/bin/kc.sh export --dir /tmp/keycloak-backup --realm ameide
  ```
- Further reading: 333-realms.md (architecture), 330-dynamic-tenant-resolution.md (tenant plumbing), Keycloak multi-tenancy docs.
