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

### Step 1: Write Service Account Credentials to Vault

All environments consume the `keycloak-admin-sa` Secret via `ExternalSecret/keycloak-admin-sa-sync`, which reads Vault keys `keycloak-admin-sa-client-id` and `keycloak-admin-sa-client-secret`. Rotate the values by writing to Vault (per environment) and forcing the ExternalSecret to refresh:

```bash
export ENV=staging   # dev | staging | prod
vault kv put secret/keycloak-admin-sa-client-id value=keycloak-admin-sa
vault kv put secret/keycloak-admin-sa-client-secret value='TempAdminSecret2025!'
kubectl -n ameide-${ENV} annotate externalsecret keycloak-admin-sa-sync \
  force-refresh=$(date +%s) --overwrite
```

### Step 2: Run `kc.sh bootstrap-admin service` from an Ephemeral Job

Use this **only for first-time creation** of the service account (or if you intentionally delete the client). Once the client exists, follow the rotation note above instead.

`kc.sh bootstrap-admin service` starts a Quarkus instance on port `9000`. Running it inside the already-running `keycloak-0` Pod fails with `Address already in use`. Instead, launch a one-shot Job that reuses the same Keycloak image and DB credentials:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: keycloak-admin-sa-bootstrap
  namespace: ameide-staging
spec:
  template:
    spec:
      restartPolicy: Never
      serviceAccountName: default
      containers:
        - name: bootstrap
          image: quay.io/keycloak/keycloak:26.4.0
          command:
            - /bin/bash
            - -c
            - |
              /opt/keycloak/bin/kc.sh bootstrap-admin service \
                --client-id keycloak-admin-sa \
                --client-secret "${CLIENT_SECRET}"
          env:
            - name: CLIENT_SECRET
              valueFrom:
                secretKeyRef:
                  name: keycloak-admin-sa
                  key: client-secret
            - name: KC_DB
              value: postgres
            - name: KC_DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: keycloak-db-credentials
                  key: username
            - name: KC_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: keycloak-db-credentials
                  key: password
            - name: KC_DB_URL_HOST
              value: postgres-ameide-rw
EOF
```

Adjust the namespace and DB host per environment. When the Job completes, delete it (`kubectl delete job keycloak-admin-sa-bootstrap`)—the `keycloak-admin-sa` Secret already contains the minted credentials.

### Step 3: Switch `client-patcher` to `loginSecretRef`

Use the Helm values override to tell the job to authenticate via the service account:

```yaml
# sources/values/{env}/platform/platform-keycloak-realm.yaml
clientPatcher:
  loginSecretRef:
    name: keycloak-admin-sa
    clientIdKey: client-id
    clientSecretKey: client-secret
```

The chart template converts this into `valueFrom.secretKeyRef` entries so `kcadm.sh` runs with `--client` / `--secret`. This lines up with vendor guidance to prefer service accounts for automation.

### Step 4: Re-run Keycloak Realm Apps & Refresh ExternalSecrets

Re-run the PreSync `platform-keycloak-realm-client-patcher` job via Argo CD and force a refresh on the downstream `ExternalSecret` so new secrets flow without manual deletion:

```bash
# Trigger job + wait per environment (requires argocd CLI login)
argocd app sync staging-platform-keycloak-realm --timeout 600 --grpc-web --plaintext
argocd app wait staging-platform-keycloak-realm --timeout 600 --grpc-web --plaintext

# Force ExternalSecret to re-read Vault/K8s
kubectl -n ameide-staging annotate externalsecret keycloak-realm-oidc-clients \
  force-refresh=$(date +%s) --overwrite
```

Repeat for production. The ExternalSecret status should flip to `Ready=True` once Vault returns the new secrets, unblocking Backstage and other OIDC consumers.

> **Existing service account rotation:** Once the client already exists, do **not** re-run `kc.sh bootstrap-admin service` (Keycloak will reject it with a duplicate client error). Instead, mint a new secret with `kcadm` and push it to Vault:
>
> ```bash
> ADMIN_USER=$(kubectl -n ameide-{env} get secret keycloak-bootstrap-admin -o jsonpath='{.data.username}' | base64 -d)
> ADMIN_PASS=$(kubectl -n ameide-{env} get secret keycloak-bootstrap-admin -o jsonpath='{.data.password}' | base64 -d)
> CLIENT_JSON=$(kubectl -n ameide-{env} exec keycloak-0 -- env ADMIN_USER="$ADMIN_USER" ADMIN_PASS="$ADMIN_PASS" bash -c '
>   KCADM=/opt/keycloak/bin/kcadm.sh
>   $KCADM config credentials --server http://localhost:8080 --realm master --user "$ADMIN_USER" --password "$ADMIN_PASS" >/dev/null
>   $KCADM get clients -r master -q clientId=keycloak-admin-sa
> ')
> CLIENT_ID=$(echo "$CLIENT_JSON" | jq -r '.[0].id')
> NEW_SECRET=$(kubectl -n ameide-{env} exec keycloak-0 -- env ADMIN_USER="$ADMIN_USER" ADMIN_PASS="$ADMIN_PASS" CLIENT_ID="$CLIENT_ID" bash -c '
>   KCADM=/opt/keycloak/bin/kcadm.sh
>   $KCADM config credentials --server http://localhost:8080 --realm master --user "$ADMIN_USER" --password "$ADMIN_PASS" >/dev/null
>   $KCADM create clients/$CLIENT_ID/client-secret -r master -o --fields value
> ' | jq -r '.value')
> vault kv put secret/keycloak-admin-sa-client-secret value="${NEW_SECRET}"
> kubectl -n ameide-{env} annotate externalsecret keycloak-admin-sa-sync force-refresh=$(date +%s) --overwrite
> ```
>
> After Vault + ExternalSecret refresh, re-run the Keycloak realm App so `client-patcher` consumes the rotated secret.

Once the temporary `keycloak-admin-sa` flow is stable, create a dedicated long-lived admin client so the stop-gap secret can be rotated out of the namespace.

### Step 5: Create Permanent Admin Service Account

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

### Step 6: Extract Permanent SA Secret to Vault

```bash
# Get client secret
SECRET=$($KC get clients/$CLIENT_UUID/client-secret -r master | jq -r '.value')

# Write to Vault
vault kv put secret/keycloak-platform-admin-sa \
  client-id=platform-admin-sa \
  client-secret="$SECRET"
```

### Step 7: Update client-patcher Values

Once the permanent admin service account secret lives in Vault (surfaced via ExternalSecret), point `client-patcher` at that Secret:

```yaml
clientPatcher:
  loginSecretRef:
    name: platform-admin-sa
    clientIdKey: client-id
    clientSecretKey: client-secret
```

At that point the temporary `keycloak-admin-sa` Secret can be deleted in favor of the Vault-managed one.

### Step 8: Delete Temporary Admin SA

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

### 4.4 Cluster Recreation Considerations

- **Fresh CNPG clusters** – When the Keycloak database starts empty, `keycloak-bootstrap-admin` is honored once and this runbook is not needed.
- **Re-created AKS clusters with existing CNPG state** – The master realm already exists, so bootstrap admin credentials remain ignored and `client-patcher` will fail until the service-account recovery steps run again.
- **Disaster-recovery restores** – Restoring a database backup into a different cluster behaves like the “existing data” case above; add this runbook to DR scripts so the service account is recreated immediately after restore.

Treat “mint admin service account + re-run client-patcher” as a standard post-rebuild task any time the Keycloak database persists across cluster operations.

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
