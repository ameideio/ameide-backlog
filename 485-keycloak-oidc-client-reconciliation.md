# 485 – Keycloak OIDC Client Reconciliation

**Status**: Draft
**Created**: 2025-12-07
**Priority**: P1 (Blocking Backstage OIDC)
**Related**: [426-keycloak-config-map.md](426-keycloak-config-map.md), [462-secrets-origin-classification.md](462-secrets-origin-classification.md), [477-backstage.md](477-backstage.md)

---

## 1. Problem Statement

OIDC clients defined in Git are not created in Keycloak when added after initial realm creation.

### Root Cause

1. **`KeycloakRealmImport` is create-only**: Vendor behavior - it bootstraps a realm once and doesn't update existing realms
2. **`client-patcher` only extracts**: It reads secrets from existing clients but doesn't create missing ones
3. **Gap in GitOps flow**: No mechanism to reconcile Git → Keycloak for OIDC clients

### Current Flow (Broken)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CURRENT BROKEN FLOW                               │
└─────────────────────────────────────────────────────────────────────────────┘

Git (client definition)
        │
        ▼
KeycloakRealmImport (PostSync hook)
        │
        └──▶ "Realm exists, nothing to do" (CREATE-ONLY)
                    │
                    X (client NOT created)
                    │
                    ▼
           client-patcher Job
                    │
                    └──▶ "Client 'backstage' not found - skipping"
                                │
                                X (no secret in Vault)
                                │
                                ▼
                         ExternalSecret
                                │
                                └──▶ ERROR: "Secret does not exist"
                                        │
                                        ▼
                                 Application FAILS
```

### Affected Services

| Service | Client ID | Status |
|---------|-----------|--------|
| Backstage | `backstage` | ❌ Blocked |
| K8s Dashboard | `k8s-dashboard` | ⚠️ Unknown (may have same issue) |
| Any future OIDC service | TBD | ⚠️ Will hit same issue |

---

## 2. Solution: Declarative Client Reconciliation

Extend `client-patcher` to **ensure clients exist** before extracting secrets.

### Target Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           TARGET DECLARATIVE FLOW                           │
└─────────────────────────────────────────────────────────────────────────────┘

Git (client definitions in values)
        │
        ▼
Helm renders ConfigMap with client specs
        │
        ▼
client-patcher Job (post-install/post-upgrade)
        │
        ├──▶ Phase 1: ENSURE CLIENTS EXIST
        │         │
        │         └──▶ For each client in spec:
        │                   GET /clients?clientId=X
        │                   If missing: POST /clients (create)
        │                   If exists: Optional update
        │
        └──▶ Phase 2: EXTRACT SECRETS (existing logic)
                  │
                  └──▶ For each client in secretExtraction:
                          GET /clients/{id}/client-secret
                          Write to Vault
                          │
                          ▼
                       ExternalSecrets sync
                          │
                          ▼
                       Application starts ✅
```

---

## 3. Implementation Design

### 3.1 Values Structure

Add `clientSpecs` to define OIDC clients declaratively:

```yaml
# sources/values/_shared/platform/platform-keycloak-realm.yaml
clientPatcher:
  enabled: true

  # NEW: Client specifications (ensure these exist in Keycloak)
  clientSpecs:
    - clientId: backstage
      realm: ameide
      spec:
        name: "Backstage Developer Portal"
        protocol: openid-connect
        publicClient: false
        standardFlowEnabled: true
        implicitFlowEnabled: false
        directAccessGrantsEnabled: false
        serviceAccountsEnabled: false
        redirectUris:
          - "https://backstage.dev.ameide.io/api/auth/oidc/handler/frame"
          - "https://backstage.staging.ameide.io/api/auth/oidc/handler/frame"
          - "https://backstage.ameide.io/api/auth/oidc/handler/frame"
        webOrigins:
          - "https://backstage.dev.ameide.io"
          - "https://backstage.staging.ameide.io"
          - "https://backstage.ameide.io"
        defaultClientScopes:
          - openid
          - profile
          - email
          - groups

    - clientId: k8s-dashboard
      realm: ameide
      spec:
        name: "Kubernetes Dashboard"
        protocol: openid-connect
        publicClient: false
        standardFlowEnabled: true
        redirectUris:
          - "https://k8s-dashboard.dev.ameide.io/oauth2/callback"
          - "https://k8s-dashboard.staging.ameide.io/oauth2/callback"
          - "https://k8s-dashboard.ameide.io/oauth2/callback"

  # EXISTING: Secret extraction configuration
  secretExtraction:
    enabled: true
    clients:
      - clientId: backstage
        vaultKey: backstage-oidc-client-secret
      - clientId: argocd
        vaultKey: argocd-dex-client-secret
      - clientId: k8s-dashboard
        vaultKey: k8s-dashboard-oidc-client-secret
```

### 3.2 ConfigMap Template

Add a ConfigMap to hold client specs as JSON:

```yaml
# keycloak_realm/templates/client-specs-configmap.yaml
{{- if .Values.clientPatcher.clientSpecs }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "keycloak-realm.fullname" . }}-client-specs
  namespace: {{ .Release.Namespace }}
data:
  clients.json: |
    {{ .Values.clientPatcher.clientSpecs | toJson }}
{{- end }}
```

### 3.3 client-patcher Job Enhancement

Add `ensure_client()` function to `client-patcher-job.yaml`:

```bash
#######################################
# Client Reconciliation Functions (NEW)
#######################################

ensure_client() {
  local realm=$1
  local client_id=$2
  local client_spec=$3  # JSON string

  echo "[Reconcile] Checking client: ${client_id} in realm: ${realm}"

  # Check if client exists
  local existing=$($KC get clients -r "${realm}" -q clientId="${client_id}" 2>/dev/null || true)

  if [[ -z "${existing}" || "${existing}" =~ ^[[:space:]]*\[[[:space:]]*\][[:space:]]*$ ]]; then
    echo "[Reconcile] Client ${client_id} not found - creating"

    # Write spec to temp file
    echo "${client_spec}" > /tmp/client-${client_id}.json

    # Create client
    $KC create clients -r "${realm}" -f /tmp/client-${client_id}.json

    if [[ $? -eq 0 ]]; then
      echo "[Reconcile] Client ${client_id} created successfully"
    else
      echo "[Reconcile] ERROR: Failed to create client ${client_id}"
      return 1
    fi
  else
    echo "[Reconcile] Client ${client_id} exists"
    # Optional: Update client if drift detected
    # local uuid=$(echo "${existing}" | jq -r '.[0].id')
    # $KC update "clients/${uuid}" -r "${realm}" -f /tmp/client-${client_id}.json
  fi
}

reconcile_clients() {
  echo ""
  echo "=== Client Reconciliation Phase ==="

  if [[ ! -f /config/clients.json ]]; then
    echo "[Reconcile] No client specs found - skipping reconciliation"
    return 0
  fi

  local count=$(jq 'length' /config/clients.json)
  echo "[Reconcile] Processing ${count} client specs"

  for i in $(seq 0 $((count - 1))); do
    local client_id=$(jq -r ".[$i].clientId" /config/clients.json)
    local realm=$(jq -r ".[$i].realm" /config/clients.json)
    local spec=$(jq -c ".[$i].spec" /config/clients.json)

    # Add clientId to spec (required by Keycloak)
    spec=$(echo "${spec}" | jq --arg cid "${client_id}" '. + {clientId: $cid}')

    ensure_client "${realm}" "${client_id}" "${spec}"
  done
}
```

### 3.4 Job Execution Order

Update the main execution block:

```bash
#######################################
# Main Execution
#######################################

# Step 1: Login to Keycloak
keycloak_login

# Step 2: Patch primary client (existing behavior)
# ... existing code ...

# Step 3: NEW - Reconcile clients from specs
reconcile_clients

# Step 4: Login to Vault (if secret extraction enabled)
vault_login

# Step 5: Extract and sync client secrets
# ... existing code ...
```

---

## 4. Authentication for client-patcher

### Current Issue

The bootstrap admin credentials (`keycloak-bootstrap-admin`) may not work because:
- Keycloak Operator creates a temporary admin that gets disabled
- Password may have been changed

### Recommended Approach

Use **service account client credentials** (same as `master-bootstrap-job`):

```yaml
clientPatcher:
  # Use platform-app-master for Keycloak admin API access
  loginRealm: master
  loginClientId: platform-app-master
  loginClientSecret: ${PLATFORM_APP_MASTER_SECRET}  # From Secret
```

The `platform-app-master` client:
- Has `realm-admin` composite role in master realm
- Can create/update clients in any realm
- Credentials are managed via existing ExternalSecrets

---

## 5. Phase Ordering

Ensure proper sync wave ordering:

| Phase | Component | Description |
|-------|-----------|-------------|
| 350 | Keycloak | Instance ready |
| 352 | KeycloakRealmImport | Creates realm (first time only) |
| 354 | master-bootstrap | Ensures platform-app-master client |
| 355 | client-patcher | **Ensures OIDC clients + extracts secrets** |
| 356+ | Applications | Backstage, k8s-dashboard, etc. |

---

## 6. Files to Modify

```
# Helm chart updates
sources/charts/foundation/operators-config/keycloak_realm/
├── templates/
│   ├── client-patcher-job.yaml      # Add ensure_client() logic
│   └── client-specs-configmap.yaml  # NEW: Client specs ConfigMap
└── values.yaml                       # Add clientSpecs schema

# Values updates (all environments)
sources/values/_shared/platform/platform-keycloak-realm.yaml  # Add clientSpecs
sources/values/dev/platform/platform-keycloak-realm.yaml      # Dev-specific URIs
sources/values/staging/platform/platform-keycloak-realm.yaml  # Staging URIs
sources/values/production/platform/platform-keycloak-realm.yaml  # Prod URIs
```

---

## 7. Testing Plan

### 7.1 Local Validation

```bash
# Helm template to verify ConfigMap
helm template keycloak-realm \
  sources/charts/foundation/operators-config/keycloak_realm \
  -f sources/values/_shared/platform/platform-keycloak-realm.yaml \
  -f sources/values/dev/platform/platform-keycloak-realm.yaml \
  | grep -A 50 "client-specs"
```

### 7.2 Dev Environment

1. Deploy updated chart to dev
2. Verify client-patcher job creates backstage client
3. Verify secret extracted to Vault
4. Verify ExternalSecret syncs
5. Verify Backstage starts with OIDC

### 7.3 Idempotency Test

1. Run ArgoCD sync again
2. Verify client-patcher doesn't fail on existing client
3. Verify no duplicate clients created

---

## 8. Success Criteria

- [ ] `backstage` client exists in Keycloak `ameide` realm
- [ ] `backstage-oidc-client-secret` exists in Vault
- [ ] `keycloak-realm-oidc-clients` K8s Secret contains `backstage-client-secret` key
- [ ] Backstage deployment starts successfully
- [ ] OIDC login to Backstage works via Keycloak
- [ ] client-patcher is idempotent (safe to re-run)

---

## 9. Future Enhancements

1. **Drift detection**: Compare Keycloak state with Git spec, log differences
2. **Strict reconciliation**: Option to update existing clients to match Git
3. **Public client support**: Handle PKCE clients (no secret extraction)
4. **Realm-per-tenant**: Scale pattern to per-tenant clients (see [333-realms.md](333-realms.md))

---

## 10. Related Documents

| Document | Relationship |
|----------|--------------|
| [426-keycloak-config-map.md](426-keycloak-config-map.md) | Keycloak GitOps architecture |
| [462-secrets-origin-classification.md](462-secrets-origin-classification.md) | OIDC secrets are Service-Generated |
| [477-backstage.md](477-backstage.md) | Blocked by this issue |
| [333-realms.md](333-realms.md) | Future realm-per-tenant scaling |
| [450-argocd-service-issues-inventory.md](450-argocd-service-issues-inventory.md) | ArgoCD OIDC uses same pattern |
