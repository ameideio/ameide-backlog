# 485 – Keycloak OIDC Client Reconciliation

**Status**: ✅ Implemented
**Created**: 2025-12-07
**Completed**: 2025-12-07
**Priority**: P1 (Blocking Backstage OIDC)
**Related**: [426-keycloak-config-map.md](426-keycloak-config-map.md), [462-secrets-origin-classification.md](462-secrets-origin-classification.md), [467-backstage.md](467-backstage.md)

---

## Addendum (2025-12-14): Make client-patcher reproducible (no runtime tool downloads)

While exercising this flow on local `arm64`, we hit a GitOps failure mode unrelated to the reconciliation logic itself but critical for reproducibility/self-heal:

- **Symptom:** `platform-keycloak-realm-client-patcher` PreSync hook Job failed (`BackoffLimitExceeded`) and blocked ArgoCD sync, leaving the realm app stuck `OutOfSync`.
- **Root cause:** runtime-downloaded tooling made the Job non-deterministic (arch/libc mismatches + transient network/HTTP errors during tool downloads).
- **Fix shipped:** `client-patcher` is now a single, multi-arch Job that:
  - runs as `alpine:3.19`
  - installs `curl`/`jq` via `apk` (no GitHub binary downloads)
  - uses Keycloak Admin REST (no `kcadm.sh`)
  - patches rotation metadata (`keycloak-client-secret-versions`) via the Kubernetes API (no `kubectl`).

This is now part of the “fleet policy hardening” surface (519): hook Jobs must be deterministic and multi-arch safe if local requires `arm64`.

## Addendum (2025-12-17): Reconcile drift for existing clients (redirect URIs)

Client reconciliation originally ensured “exists” only. This is insufficient for fixing real-world drift, because:
- `KeycloakRealmImport` is create-only (realm exists → no updates),
- some clients (e.g., `argocd`) can exist but be misconfigured (missing redirect URIs), producing runtime auth failures.

Policy:
- Keep create-only as the safe default.
- Allow an explicit “update existing” mode that merges the declared spec into the existing client representation (idempotent, avoids deleting unknown fields), for cases where we must correct drift deterministically.

## Addendum (2025-12-17): Keycloak client-scope linkage must be reconciled via dedicated endpoints

Observed failure mode: OAuth requests fail with `invalid_scope` even when realm discovery advertises the scope (e.g., `profile`, `email`, `groups`).

Root cause:
- In Keycloak, “client scopes attached to a client” are modeled as separate resources. Updating the client JSON representation is not sufficient to ensure the scope is attached as a default/optional scope for that client.

Remediation (GitOps, vendor-aligned):
- During reconciliation, after ensuring the client exists, explicitly attach required scopes using the Admin API:
  - `PUT /admin/realms/{realm}/clients/{id}/default-client-scopes/{scopeId}` with body `{}` (empty JSON)
  - `PUT /admin/realms/{realm}/clients/{id}/optional-client-scopes/{scopeId}` with body `{}` (empty JSON)
  - Note: the endpoint uses the **client UUID** (`id`), not the `clientId` string, and updating `defaultClientScopes` / `optionalClientScopes` in the client JSON representation is not sufficient (Keycloak treats scope linkage as separate resources).
- Keep realm-scope reconciliation (ensuring `profile`/`email` scopes exist) as a prerequisite for client-scope attachment.

## Addendum (2025-12-17): client-patcher Keycloak auth must be debuggable and resilient

Observed failure mode: the `client-patcher` can loop on “Token not available yet” and eventually fail authentication while the Keycloak Service is reachable.

Contributing factors:
- `curl -f` suppresses non-2xx response bodies, so an actionable `invalid_client` / `unauthorized_client` payload is lost (appears as empty response).
- Service-account client-credentials login (`keycloak-admin-sa`) can be temporarily invalid/misconfigured; the Job already has access to the bootstrap admin secret and should be able to fall back to password grant deterministically.

Remediation (GitOps, vendor-aligned):
- Stop using `curl -f` for the token request; capture the response body and parse `access_token`.
- If service-account login fails but `KEYCLOAK_ADMIN_USER/PASSWORD` are available, fall back to password grant (`admin-cli`) so the job can still reconcile clients/secrets.
- Ensure Argo hook cleanup cannot deadlock on failures: include `HookFailed` in `argocd.argoproj.io/hook-delete-policy` so a failed `client-patcher` Job does not leave the Application stuck “waiting for deletion”.

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
sources/values/env/dev/platform/platform-keycloak-realm.yaml      # Dev-specific URIs
sources/values/env/staging/platform/platform-keycloak-realm.yaml  # Staging URIs
sources/values/env/production/platform/platform-keycloak-realm.yaml  # Prod URIs
```

---

## 7. Testing Plan

### 7.1 Local Validation

```bash
# Helm template to verify ConfigMap
helm template keycloak-realm \
  sources/charts/foundation/operators-config/keycloak_realm \
  -f sources/values/_shared/platform/platform-keycloak-realm.yaml \
  -f sources/values/env/dev/platform/platform-keycloak-realm.yaml \
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

- [x] `backstage` client exists in Keycloak `ameide` realm ✅
- [x] `backstage-oidc-client-secret` exists in Vault ✅
- [x] `keycloak-realm-oidc-clients` K8s Secret contains `backstage-client-secret` key ✅
- [ ] Backstage deployment starts successfully (pending: Backstage not yet deployed)
- [ ] OIDC login to Backstage works via Keycloak (pending: Backstage not yet deployed)
- [x] client-patcher is idempotent (safe to re-run) ✅

---

## 9. Future Enhancements

1. **Drift detection**: Compare Keycloak state with Git spec, log differences
2. **Strict reconciliation**: Option to update existing clients to match Git
3. **Public client support**: Handle PKCE clients (no secret extraction)
4. **Realm-per-tenant**: Scale pattern to per-tenant clients (see [333-realms.md](333-realms.md))

---

## 10. Implementation Notes

### 10.1 Issues Encountered During Implementation

This section documents all issues encountered during implementation for future reference.

#### Issue 1: ArgoCD Hook Timing (PostSync vs PreSync)

**Problem**: Initially the client-patcher Job used `PostSync` hook, but this created a chicken-and-egg problem:
- PostSync hooks only run AFTER all resources become healthy
- ExternalSecrets were failing because they couldn't find secrets in Vault
- ArgoCD marked app as "OutOfSync/Degraded" and retried indefinitely

**Symptom**:
```
error processing spec.data[2] (key: backstage-oidc-client-secret), err: Secret does not exist
```

**Vendor Confirmation (2025-12-07)**: Per ArgoCD docs: *"PostSync executes after all Sync hooks completed and were successful, a successful application, and **all resources in a Healthy state.**"* ([source](https://argo-cd.readthedocs.io/en/release-2.0/user-guide/resource_hooks/))

This confirms PostSync was architecturally unsuitable for bootstrap operations.

**Solution**: Changed to `PreSync` hook with sync-wave `-1`:
```yaml
annotations:
  argocd.argoproj.io/hook: PreSync
  argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
  argocd.argoproj.io/sync-wave: "-1"
```

This ensures the client-patcher runs BEFORE ArgoCD evaluates resource health, breaking the cycle.

**Commit**: `b2ab5e3c fix(keycloak): change client-patcher to PreSync hook for ArgoCD`

#### Issue 2: Vault Policy Missing `backstage-*` Path

**Problem**: The client-patcher Job logged "Secret written successfully" but the secret was never actually written to Vault.

**Root Cause**: The `keycloak-client-patcher` Vault policy only allowed write access to:
- `secret/data/platform-app-*`
- `secret/data/argocd-*`
- `secret/data/k8s-dashboard-*`

The `backstage-*` path was not included.

**Symptom**:
- Job logs showed success: `[Vault] Secret written successfully`
- But ExternalSecret still failed: `Secret does not exist`
- Manual curl to Vault returned: `{"errors":["permission denied"]}`

**Why Logs Were Misleading**: The `vault_write_secret()` function checked for `.errors` in the response, but curl wasn't failing with `--fail` flag properly, so the "success" message was printed even when Vault returned a permission denied error.

**Solution**: Added `backstage-*` path to vault-bootstrap cronjob.yaml:
```hcl
path "secret/data/backstage-*" {
  capabilities = ["create", "update", "read"]
}
```

**Commit**: `93881b49 feat(vault): add backstage-* path to keycloak-client-patcher policy`

#### Issue 3: Pod Scheduling Failures on Dev Nodes

**Problem**: Manual test pods and the client-patcher job failed to schedule.

**Symptom**:
```
0/15 nodes are available: 5 node(s) had untolerated taint {ameide.io/environment: dev}
```

**Root Cause**: Dev environment uses dedicated nodes with taints. Jobs must include tolerations and nodeSelector.

**Solution**: Added tolerations and nodeSelector to job specs:
```yaml
tolerations:
  - key: "ameide.io/environment"
    value: "dev"
    effect: "NoSchedule"
nodeSelector:
  ameide.io/pool: dev
```

#### Issue 4: ServiceAccount Not Found

**Problem**: Initial attempts used `serviceAccountName: external-secrets` which didn't exist.

**Symptom**:
```
pods "platform-keycloak-realm-client-patcher-" is forbidden: error looking up service account ameide-dev/external-secrets: serviceaccount "external-secrets" not found
```

**Solution**: Changed to `serviceAccountName: default` which has the `keycloak-client-patcher` Vault role bound to it.

#### Issue 5: Helm Deep Merge Limitation

**Problem**: Dev values completely overrode shared values for `clientPatcher` section instead of deep merging.

**Symptom**: Secrets defined in shared values weren't being extracted because dev values replaced the entire `clientPatcher` structure.

**Root Cause**: Helm performs shallow merge of values files, not deep merge. When dev values defined `clientPatcher.clientReconciliation`, it replaced the shared `clientPatcher` entirely.

**Solution**: Dev values must include all required fields, or use Helm's `default` function. We chose to have dev values be comprehensive for clarity.

#### Issue 6: ArgoCD CLI Issues with Port-Forward

**Problem**: ArgoCD CLI commands failed with various errors when using `--port-forward`.

**Symptoms**:
```
configmap "argocd-cm" not found
```
```
services is forbidden: User "system:serviceaccount:argocd:argocd-server" cannot list resource
```

**Workaround**: Used `kubectl patch app` instead of ArgoCD CLI for sync operations:
```bash
kubectl patch app dev-platform-keycloak-realm -n argocd --type merge \
  -p '{"operation":{"initiatedBy":{"username":"cli"},"sync":{"prune":true,"revision":"HEAD"}}}'
```

#### Issue 7: CronJob Template Not Updated After Sync

**Problem**: After syncing vault-bootstrap, the CronJob was updated but running jobs still used old template.

**Symptom**: Vault policy update wasn't applied because scheduled job ran with cached template.

**Solution**: Created manual job from updated CronJob to apply policy immediately:
```bash
kubectl create job --from=cronjob/foundation-vault-bootstrap-vault-bootstrap vault-bootstrap-manual-$(date +%s) -n ameide-dev
```

#### Issue 8: KeycloakRealmImport PostSync Hook Creates Bootstrap Deadlock

**Problem**: Staging and Production environments were stuck in Degraded state because the `ameide` realm was never created. The `KeycloakRealmImport` resource was configured as a `PostSync` hook, creating a bootstrap deadlock.

**Root Cause**: ArgoCD's PostSync hooks only execute **after all resources are Healthy**. This created a circular dependency:

```
PostSync: KeycloakRealmImport → creates realm + clients
    ↑ NEVER RUNS (app never reaches Healthy)
Sync: ExternalSecret → DEGRADED (secret doesn't exist in Vault)
    ↑
PreSync: client-patcher → skips (realm doesn't exist, no clients to extract)
```

**Symptom**:
- Staging/Prod keycloak-realm apps showed "Degraded"
- ExternalSecret `keycloak-realm-oidc-clients` error: `Secret does not exist`
- Only `master` realm existed in Keycloak (no `ameide` realm)
- Dev worked because the realm was created before this hook ordering issue was introduced

**Vendor Confirmation (2025-12-07)**:
- ArgoCD docs: *"PostSync executes after all Sync hooks completed and were successful, a successful application, and **all resources in a Healthy state.**"* ([source](https://argo-cd.readthedocs.io/en/release-2.0/user-guide/resource_hooks/))
- RHBK Realm Import docs: *"The Realm Import CR only supports creation of new realms and does not update or delete those."* ([source](https://docs.redhat.com/pt-br/documentation/red_hat_build_of_keycloak/22.0/html/operator_guide/realm-import-))

Combined with ArgoCD PostSync semantics, this created an unrecoverable bootstrap deadlock in staging/production.

**Solution**: Changed `KeycloakRealmImport` from PostSync to PreSync with wave `-10`:
```yaml
# Before (broken):
annotations:
  argocd.argoproj.io/hook: PostSync
  argocd.argoproj.io/hook-delete-policy: HookSucceeded
  argocd.argoproj.io/sync-wave: "10"

# After (fixed):
annotations:
  # Run realm import BEFORE client-patcher (wave -1) and BEFORE ExternalSecrets (Sync phase)
  # This breaks the bootstrap deadlock: realm must exist before secrets can be extracted
  argocd.argoproj.io/hook: PreSync
  argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
  argocd.argoproj.io/sync-wave: "-10"
```

**Target Flow**:
```
PreSync wave -10: KeycloakRealmImport → creates realm + clients
PreSync wave -1:  client-patcher → extracts secrets to Vault
Sync:             ExternalSecret → fetches secrets from Vault ✓
PostSync:         master-bootstrap → assigns roles (if needed)
```

**File Modified**: `sources/charts/foundation/operators-config/keycloak_realm/templates/realm-import.yaml`

### 10.2 Final Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      IMPLEMENTED DECLARATIVE FLOW                           │
└─────────────────────────────────────────────────────────────────────────────┘

Git (client definitions in values.yaml)
        │
        ▼
Helm template → ConfigMap (client-specs)
        │
        ▼
ArgoCD Sync (PreSync phase, sync-wave: -1)
        │
        ▼
client-patcher Job
        │
        ├──▶ Phase 1: RECONCILE CLIENTS
        │         │
        │         └──▶ ensure_client() → kcadm.sh create clients
        │                   │
        │                   ▼
        │              Client created in Keycloak ✅
        │
        ├──▶ Phase 2: PATCH PRIMARY CLIENT
        │         │
        │         └──▶ patch_client() → kcadm.sh update
        │
        └──▶ Phase 3: EXTRACT SECRETS
                  │
                  └──▶ extract_and_sync_client_secret()
                          │
                          ├──▶ kcadm.sh get client-secret
                          │
                          └──▶ curl POST Vault → Secret in Vault ✅
                                   │
                                   ▼
                              ExternalSecrets sync
                                   │
                                   ▼
                              K8s Secret created ✅
                                   │
                                   ▼
                              ArgoCD: App Healthy ✅
```

### 10.3 Files Modified

| File | Change |
|------|--------|
| `sources/charts/foundation/operators-config/keycloak_realm/templates/client-patcher-job.yaml` | Added PreSync hook, ensure_client(), reconcile_clients() |
| `sources/charts/foundation/operators-config/keycloak_realm/templates/client-reconciliation-configmap.yaml` | NEW: ConfigMap with client specs |
| `sources/charts/foundation/operators-config/keycloak_realm/values.yaml` | Added clientReconciliation schema |
| `sources/values/_shared/platform/platform-keycloak-realm.yaml` | Added clientReconciliation with backstage/argocd clients |
| `sources/values/env/dev/platform/platform-keycloak-realm.yaml` | Added dev-specific clientReconciliation |
| `sources/charts/foundation/vault-bootstrap/templates/cronjob.yaml` | Added backstage-* to keycloak-client-patcher policy |

---

## 11. Related Documents

| Document | Relationship |
|----------|--------------|
| [426-keycloak-config-map.md](426-keycloak-config-map.md) | Keycloak GitOps architecture |
| [462-secrets-origin-classification.md](462-secrets-origin-classification.md) | OIDC secrets are Service-Generated |
| [467-backstage.md](467-backstage.md) | Blocked by this issue |
| [333-realms.md](333-realms.md) | Future realm-per-tenant scaling |
| [450-argocd-service-issues-inventory.md](450-argocd-service-issues-inventory.md) | ArgoCD OIDC uses same pattern |
