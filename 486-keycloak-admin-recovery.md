# 486 – Keycloak Admin Recovery

**Status**: Draft
**Created**: 2025-12-07
**Priority**: P0 (Blocking all Keycloak realm operations)
**Related**: [426-keycloak-config-map.md](426-keycloak-config-map.md), [485-keycloak-oidc-client-reconciliation.md](485-keycloak-oidc-client-reconciliation.md), [462-secrets-origin-classification.md](462-secrets-origin-classification.md)

---

## 1. Problem Statement

The `client-patcher` Job fails with `invalid_grant` (Invalid user credentials) because the admin password in `keycloak-bootstrap-admin` Secret doesn't match what Keycloak has stored.

### Root Cause

Per [Keycloak vendor documentation](https://www.keycloak.org/server/all-config):

> *"Temporary bootstrap admin password. **Used only when the master realm is created.**"*

And per [RHBK Operator Guide](https://docs.redhat.com/en/documentation/red_hat_build_of_keycloak/26.2/html-single/operator_guide/):

> *"If a master realm has already been created for your cluster, then the spec.bootstrapAdmin is effectively ignored."*

**Conclusion**: Once the master realm exists, changing `KC_BOOTSTRAP_ADMIN_*` env vars or the `keycloak-bootstrap-admin` Secret has no effect.

### Affected Environments

| Environment | Status | Impact |
|-------------|--------|--------|
| dev | Blocked | client-patcher fails |
| staging | Blocked | client-patcher fails |
| production | Blocked | client-patcher fails |

---

## 2. Vendor-Aligned Recovery Options

Per [Keycloak Bootstrap Admin Recovery](https://www.keycloak.org/server/bootstrap-admin-recovery):

### Option 1: `bootstrap-admin user` (Temporary Admin User)

Creates a temporary admin **user** in the master realm:

```bash
kubectl exec -n ameide-{env} keycloak-0 -- \
  /opt/keycloak/bin/kc.sh bootstrap-admin user \
  --username temp-admin \
  --password '<secure-password>'
```

**Pros**: Simple, creates normal admin user
**Cons**: Password-based auth still subject to drift

### Option 2: `bootstrap-admin service` (Temporary Admin Service Account) - RECOMMENDED

Creates a temporary admin **service account** (OIDC client with client credentials):

```bash
kubectl exec -n ameide-{env} keycloak-0 -- \
  /opt/keycloak/bin/kc.sh bootstrap-admin service \
  --client-id temp-admin-sa \
  --client-secret '<secure-secret>'
```

**Pros**:
- Service account auth is vendor-recommended for automation
- Client credentials don't drift like passwords
- Can be rotated via Keycloak Admin API

**Cons**: Requires updating client-patcher to use client credentials grant

### Option 3: Full Reset (Not Recommended)

Delete the master realm and let Keycloak recreate:

```bash
# DESTRUCTIVE - loses all realm data
kubectl exec -n ameide-{env} postgres-ameide-0 -- \
  psql -U postgres -d keycloak -c "DELETE FROM realm WHERE name='master'"
```

**Cons**: Loses all realm configuration, users, clients

---

## 3. Recommended Recovery Procedure

### Step 1: Create Temporary Admin Service Account

For each environment:

```bash
# Dev
kubectl exec -n ameide-dev keycloak-0 -- \
  /opt/keycloak/bin/kc.sh bootstrap-admin service \
  --client-id keycloak-admin-sa \
  --client-secret 'TempAdminSecret2025!'

# Staging
kubectl exec -n ameide-staging keycloak-0 -- \
  /opt/keycloak/bin/kc.sh bootstrap-admin service \
  --client-id keycloak-admin-sa \
  --client-secret 'TempAdminSecret2025!'

# Production
kubectl exec -n ameide-prod keycloak-0 -- \
  /opt/keycloak/bin/kc.sh bootstrap-admin service \
  --client-id keycloak-admin-sa \
  --client-secret 'TempAdminSecret2025!'
```

### Step 2: Store Service Account Credentials in Vault

Write the service account credentials to Vault so client-patcher can use them:

```bash
vault kv put secret/keycloak-admin-sa \
  client-id=keycloak-admin-sa \
  client-secret='TempAdminSecret2025!'
```

### Step 3: Update client-patcher to Use Service Account Auth

Modify authentication in client-patcher-job.yaml:

```bash
# Current (password grant - broken)
$KC config credentials \
  --server "${KEYCLOAK_URL}" \
  --realm master \
  --user "${KEYCLOAK_ADMIN_USER}" \
  --password "${KEYCLOAK_ADMIN_PASSWORD}"

# New (client credentials grant - recommended)
$KC config credentials \
  --server "${KEYCLOAK_URL}" \
  --realm master \
  --client "${KEYCLOAK_ADMIN_CLIENT_ID}" \
  --secret "${KEYCLOAK_ADMIN_CLIENT_SECRET}"
```

### Step 4: Create Permanent Admin Service Account

Use the temporary SA to create a permanent one via Admin API:

```bash
# Create permanent admin client
$KC create clients -r master -f - <<EOF
{
  "clientId": "platform-admin-sa",
  "serviceAccountsEnabled": true,
  "publicClient": false,
  "protocol": "openid-connect"
}
EOF

# Get client UUID
CLIENT_UUID=$($KC get clients -r master -q clientId=platform-admin-sa | jq -r '.[0].id')

# Assign admin role
$KC add-roles -r master \
  --uusername service-account-platform-admin-sa \
  --rolename admin
```

### Step 5: Extract Permanent SA Secret to Vault

```bash
# Get client secret
SECRET=$($KC get clients/$CLIENT_UUID/client-secret -r master | jq -r '.value')

# Write to Vault
vault kv put secret/keycloak-platform-admin-sa \
  client-id=platform-admin-sa \
  client-secret="$SECRET"
```

### Step 6: Update client-patcher Values

```yaml
clientPatcher:
  auth:
    method: client_credentials  # Instead of password
    clientIdKey: keycloak-platform-admin-sa
    secretPath: secret/keycloak-platform-admin-sa
```

### Step 7: Delete Temporary Admin SA

```bash
$KC delete clients -r master -i $TEMP_CLIENT_UUID
```

---

## 4. Prevention

### 4.1 Use Service Account Auth by Default

All automated Keycloak admin operations should use service account authentication:

- `client-patcher` Job
- `master-bootstrap` Job
- Any future admin automation

### 4.2 Document Bootstrap Admin Limitations

Add warning to `keycloak-bootstrap-admin` Secret:

```yaml
metadata:
  annotations:
    ameide.io/warning: |
      This secret is ONLY used when the master realm is first created.
      Changing this secret after initial bootstrap has NO EFFECT.
      For admin recovery, see backlog/486-keycloak-admin-recovery.md
```

### 4.3 Monitor Admin Auth Failures

Add alerting for Keycloak admin authentication failures:

```yaml
- alert: KeycloakAdminAuthFailure
  expr: increase(keycloak_admin_auth_failures_total[5m]) > 0
  annotations:
    summary: "Keycloak admin authentication failing"
    runbook: "backlog/486-keycloak-admin-recovery.md"
```

---

## 5. Success Criteria

- [ ] client-patcher authenticates successfully in dev
- [ ] client-patcher authenticates successfully in staging
- [ ] client-patcher authenticates successfully in production
- [ ] backstage client created and secret extracted
- [ ] Service account auth documented and implemented
- [ ] Temporary admin SA cleaned up

---

## 6. References

- [Keycloak Bootstrap Admin Recovery](https://www.keycloak.org/server/bootstrap-admin-recovery)
- [Keycloak All Config](https://www.keycloak.org/server/all-config)
- [RHBK Operator Guide](https://docs.redhat.com/en/documentation/red_hat_build_of_keycloak/26.2/html-single/operator_guide/)
- [Keycloak Admin CLI](https://wjw465150.gitbooks.io/keycloak-documentation/content/server_admin/topics/admin-cli.html)
