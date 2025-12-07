# 460: Keycloak OIDC Scopes & Vendor-Aligned Realm Pattern

**Created**: 2025-12-06
**Status**: In Progress
**Priority**: Critical (blocking dev auth)

## Summary

The `ameide` realm is missing **built-in OIDC client scopes** (`profile`, `email`), causing NextAuth `OAuthCallbackError` / `invalid_scope` loops when Auth.js requests `scope=openid profile email`. This backlog tracks the immediate fix and establishes the vendor-aligned realm pattern for future realm management.

---

## Root Cause Analysis

### Symptoms
- Auth redirect loop on dev
- Keycloak logs: `Invalid scopes: openid profile email`
- NextAuth error: `OAuthCallbackError: OAuth Provider returned an error`

### Root Cause
The `ameide` realm JSON in `keycloak_realm/values.yaml` defines only custom scopes (`roles`, `tenant`, `groups`, `organizations`) in `clientScopes[]`. When this array is present, Keycloak treats it as the **COMPLETE** list—it does NOT merge with built-in defaults.

**Key vendor distinction:**
- `openid` is **NOT a client scope**—it's the mandatory OIDC protocol-level scope that tells Keycloak "this is an OpenID Connect request"
- `profile` and `email` ARE built-in client scopes that must exist and be linked to the client
- When Auth.js sends `scope=openid profile email`, Keycloak validates that `profile` and `email` are valid client scopes for that client—if they don't exist, it returns `invalid_scope`

### Historical Context
- Previous `invalid_scope` fix documented in `old/104-keycloak-config-v2.md` removed `offline_access` from client request—a workaround, not root cause fix
- The realm JSON was hand-crafted rather than exported from a working Keycloak instance, missing vendor defaults

---

## Activity Tracking

### Phase 1: Immediate Fix (Critical)

| ID | Activity | Status | Commit | Notes |
|----|----------|--------|--------|-------|
| 460-1 | Add built-in client scopes to `keycloak_realm/values.yaml` | ✅ Done | `59d6e93` | Added `profile`, `email`, `offline_access` with protocol mappers. Removed erroneous `openid` client scope—`openid` is a protocol-level meta-scope, NOT a client scope. |
| 460-2 | Commit OIDC scopes fix | ✅ Done | `59d6e93` | fix(keycloak-realm): remove erroneous openid client scope (460) |
| 460-3 | Apply fix to dev Keycloak database | ✅ Done | N/A | KeycloakRealmImport is create-only—applied fix directly to PostgreSQL via `kubectl exec`. Added `profile` and `email` client scopes with protocol mappers, linked to all clients. |
| 460-4 | Verify dev OIDC discovery | ✅ Done | N/A | `scopes_supported` now includes `profile`, `email`. Verified 2025-12-06. |
| 460-5 | Apply fix to staging/production if needed | ✅ N/A | | Staging/production only have `master` realm—`ameide` realm not yet created. When created, it will use the fixed `values.yaml` with correct scopes. No database fix needed. |

### Phase 2: Documentation Updates

| ID | Activity | Status | Backlog | Notes |
|----|----------|--------|---------|-------|
| 460-6 | Update 323 - add OIDC scopes dependency note | ✅ Done | [323](323-keycloak-realm-roles.md) | §2 + §3 |
| 460-7 | Update 333 - add export-based template note | ✅ Done | [333](333-realms.md) | Realm template section |
| 460-8 | Update 426 - add OIDC scopes section | ✅ Done | [426](426-keycloak-config-map.md) | New §3.x |
| 460-9 | Update 426 - clarify realm JSON source | ✅ Done | [426](426-keycloak-config-map.md) | §2/3 |
| 460-10 | Update 426 - cross-reference this backlog | ✅ Done | [426](426-keycloak-config-map.md) | §6 |

### Phase 3: Vendor-Aligned Pattern (Future)

| ID | Activity | Status | Notes |
|----|----------|--------|-------|
| 460-11 | Document export-based realm workflow | ⬜ Pending | Configure → Test → Export → Commit |
| 460-12 | Export working `ameide` realm from Keycloak | ⬜ Pending | After dev verified working |
| 460-13 | Replace hand-crafted JSON with export | ⬜ Pending | Use export as chart input |
| 460-14 | Add realm template to 333 for realm-per-tenant | ⬜ Pending | Export becomes template |

### Phase 4: Validation & Tests

| ID | Activity | Status | Notes |
|----|----------|--------|-------|
| 460-15 | Add OIDC discovery check to validation | ⬜ Pending | `.well-known/openid-configuration` |
| 460-16 | Add password grant smoke test | ⬜ Pending | Verify token claims per 323 |
| 460-17 | Update `validate-keycloak.sh` | ⬜ Pending | Include scope verification |
| 460-18 | Add smoke test to `auth-smoke.yaml` | ⬜ Pending | Argo Helm test jobs |

---

## Files to Modify

| File | Change | Phase |
|------|--------|-------|
| `sources/charts/foundation/operators-config/keycloak_realm/values.yaml` | Add standard OIDC scopes | 1 |
| `backlog/323-keycloak-realm-roles.md` | Add OIDC scopes dependency note | 2 |
| `backlog/333-realms.md` | Add export-based template note | 2 |
| `backlog/426-keycloak-config-map.md` | Add OIDC scopes section, clarify source | 2 |
| `infra/kubernetes/scripts/validate-keycloak.sh` | Add scope verification | 4 |
| `sources/charts/shared-values/tests/auth-smoke.yaml` | Add token grant test | 4 |

---

## Cross-References

### Related Backlogs

| Backlog | Relationship |
|---------|--------------|
| [323-keycloak-realm-roles.md](323-keycloak-realm-roles.md) | Defines RBAC + token shape; assumes OIDC scopes exist |
| [333-realms.md](333-realms.md) | Realm-per-tenant ADR; needs export-based template pattern |
| [426-keycloak-config-map.md](426-keycloak-config-map.md) | GitOps map; needs OIDC scopes section |
| [462-secrets-origin-classification.md](462-secrets-origin-classification.md) | Client secret extraction flow (client-patcher → Vault) |
| [old/104-keycloak-config-v2.md](old/104-keycloak-config-v2.md) | Previous `invalid_scope` workaround (archived) |
| [447-third-party-chart-tolerations.md](447-third-party-chart-tolerations.md) | §12 documents Keycloak `spec.scheduling` |
| [442-environment-isolation.md](442-environment-isolation.md) | Keycloak tolerations entry |

### ArgoCD Applications

| Application | Namespace | Notes |
|-------------|-----------|-------|
| `dev-platform-keycloak-realm` | `argocd` | PostSync hooks create realm |
| `staging-platform-keycloak-realm` | `argocd` | Same chart |
| `production-platform-keycloak-realm` | `argocd` | Same chart |

---

## Verification Commands

### OIDC Discovery
```bash
curl -sk "https://auth.dev.ameide.io/realms/ameide/.well-known/openid-configuration" | \
  jq '{issuer, scopes_supported}'
```

### Keycloak Logs (no invalid_scope)
```bash
kubectl logs -n ameide-dev -l app.kubernetes.io/name=keycloak --tail=100 | grep -i "invalid"
```

### Token Grant Test
```bash
curl -sk -X POST "https://auth.dev.ameide.io/realms/ameide/protocol/openid-connect/token" \
  -d "grant_type=password" \
  -d "client_id=platform-app" \
  -d "client_secret=${CLIENT_SECRET}" \
  -d "username=${TEST_USER}" \
  -d "password=${TEST_PASS}" \
  -d "scope=openid profile email roles tenant" | jq .
```

---

## Technical Details

### OIDC Scope Types (Vendor Terminology)

| Scope | Type | Description |
|-------|------|-------------|
| `openid` | **Protocol scope (meta)** | Mandatory OIDC scope—tells Keycloak "this is an OpenID Connect request". NOT a client scope. |
| `profile` | **Built-in client scope** | Maps user profile claims (`preferred_username`, `name`, `family_name`, `given_name`) |
| `email` | **Built-in client scope** | Maps email claims (`email`, `email_verified`) |
| `offline_access` | **Built-in client scope** | Enables refresh tokens for long-lived sessions |
| `address`, `phone` | **Built-in client scopes** | Standard OIDC scopes (not used by Auth.js) |
| `roles`, `web-origins` | **Built-in client scopes** | Keycloak-specific, auto-created in new realms |
| `tenant`, `groups`, `organizations` | **Custom client scopes** | Ameide-specific (323-keycloak-realm-roles.md) |

### Required for Auth.js

The `platform-app` client MUST have these as **default client scopes**:
- `profile` - so `scope=openid profile email` resolves the `profile` client scope
- `email` - so `scope=openid profile email` resolves the `email` client scope

When Auth.js sends `scope=openid profile email`:
1. `openid` signals this is an OIDC request (not a client scope lookup)
2. `profile` and `email` are looked up as client scopes linked to `platform-app`
3. Protocol mappers from those scopes populate the token claims

### Protocol Mappers (profile scope)

| Mapper | Claim | User Attribute |
|--------|-------|----------------|
| username | `preferred_username` | `username` |
| family name | `family_name` | `lastName` |
| given name | `given_name` | `firstName` |
| full name | `name` | (computed) |

### Protocol Mappers (email scope)

| Mapper | Claim | User Attribute |
|--------|-------|----------------|
| email | `email` | `email` |
| email verified | `email_verified` | `emailVerified` |

---

## Vendor-Aligned Realm Pattern

### Current Problem

Hand-crafted realm JSON in `keycloak_realm/values.yaml` is error-prone:
- Missing vendor defaults (OIDC scopes)
- Diverges from actual Keycloak state over time
- No validation that JSON matches expected Keycloak behavior

### Recommended Pattern

Use **Keycloak's own realm export** as the source of truth:

1. **Configure** realm via Keycloak Admin Console until behavior is correct
2. **Test** auth flow works (Auth.js + RBAC)
3. **Export** realm via Admin Console or API
4. **Commit** export JSON as chart input
5. **Sync** via ArgoCD to apply across environments

This ensures:
- All vendor defaults preserved
- Protocol mappers match Keycloak's implementation
- Realm state is reproducible and auditable

### Export Workflow

```bash
# Export realm via Keycloak Admin API
curl -sk -X POST "https://auth.dev.ameide.io/admin/realms/ameide/partial-export" \
  -H "Authorization: Bearer ${ADMIN_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"exportClients": true, "exportGroupsAndRoles": true}' \
  -o realm-export.json

# Clean up export (remove per-instance data)
jq 'del(.id, .clients[].id, .roles.realm[].id)' realm-export.json > realm-clean.json

# Use as chart input
cp realm-clean.json sources/charts/foundation/operators-config/keycloak_realm/files/ameide.json
```

---

## Historical Fixes (For Reference)

### spec.scheduling Fix (2025-12-06)

Import job tolerations are documented in:
- [426-keycloak-config-map.md §5.1](426-keycloak-config-map.md)
- [447-third-party-chart-tolerations.md §12](447-third-party-chart-tolerations.md)
- [442-environment-isolation.md](442-environment-isolation.md)

Commits: `81f157b`, `8b85655`

---

## Decision Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2025-12-06 | Fix by adding scopes to existing JSON, not replacing with export | Immediate fix; export-based pattern is follow-up |
| 2025-12-06 | Document in new backlog 460 vs updating 426 directly | Cleaner tracking; 426 remains the map, 460 is the incident |
| 2025-12-06 | Keep realm-per-tenant (333) unchanged | This fix is for single-realm baseline; 333 north star intact |
| 2025-12-06 | Apply fix via direct PostgreSQL modification | KeycloakRealmImport is create-only (does NOT update existing realms). Keycloak Admin API returned 403—admin user lacks realm-management permissions. Direct DB modification was the only viable path for immediate fix. |
| 2025-12-06 | Document database fix as one-time workaround | Future: export realm from dev Keycloak, use export as source of truth. This avoids hand-crafted JSON and ensures scopes are always present. |
