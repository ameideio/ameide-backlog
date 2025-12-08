# 426 – Keycloak configuration & GitOps map

**Status**: Draft
**Last Updated**: 2025-12-07

> **Cross-References (Deployment Architecture Suite)**:
>
> | Document | Purpose |
> |----------|---------|
> | [465-applicationset-architecture.md](465-applicationset-architecture.md) | How the two ApplicationSets work |
> | [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Rollout phases & operator deployment |
> | [464-chart-folder-alignment.md](464-chart-folder-alignment.md) | Chart folder structure & domain alignment |
>
> **Related (Keycloak & Secrets)**:
> [323-keycloak-realm-roles.md](323-keycloak-realm-roles.md) · [323-keycloak-realm-roles-v2.md](323-keycloak-realm-roles-v2.md) · [333-realms.md](333-realms.md) · [364-argo-configuration-v5.md](364-argo-configuration-v5.md) · [375-rolling-sync-wave-redesign.md](375-rolling-sync-wave-redesign.md) · [418-secrets-strategy-map.md](418-secrets-strategy-map.md) · [462-secrets-origin-classification.md](462-secrets-origin-classification.md) · infra/README.md

---

## 1. Purpose

Give a single, up-to-date view of how Keycloak is deployed and managed via GitOps:

- Which Argo CD Applications and charts make up the Keycloak stack.
- How instance vs realm vs admin secrets are split across layers.
- How health checks, sync waves, and operators interact.
- Where to look for deeper details (realm roles, multi-tenancy, secrets strategy).

This document is a **map** that ties together more detailed backlogs (323, 333, 364, 418) and the GitOps tree under `gitops/ameide-gitops`.

---

## 2. Component inventory (GitOps + Argo CD)

Keycloak is delivered as an operator-centric stack, split into **foundation** and **platform** layers.

**Argo CD Applications (dev):**

- `argocd/platform-keycloak-crds`  
  - **Responsibility**: Installs Keycloak CRDs (`Keycloak`, `KeycloakRealmImport`).  
  - **Source**: `gitops/ameide-gitops/sources/charts/foundation/common/raw-manifests`  
    - Values: `.../sources/values/_shared/platform/platform-keycloak-crds.yaml` (+ dev overlay).  
  - **Notes**: Aligns with CRD guidance in 364‑argo-configuration-v5 (CRDs in their own Application, wave-gated).

- `argocd/foundation-keycloak-operator`  
  - **Responsibility**: Deploys the Keycloak Operator (controller + RBAC).  
  - **Source**: `gitops/ameide-gitops/sources/charts/foundation/operators/keycloak-operator`  
    - CRDs vendored under `crds/`.  
  - **Notes**: Follows operator best practices; health is standard Deployment/Pod health (no custom Lua).

- `argocd/foundation-keycloak-admin-secrets`  
  - **Responsibility**: Owns Keycloak admin **ExternalSecrets** and the derived Kubernetes Secrets:
    - `ExternalSecret keycloak-bootstrap-admin-sync` → `Secret keycloak-bootstrap-admin`
    - `ExternalSecret keycloak-master-bootstrap-sync` → `Secret keycloak-master-bootstrap`
    - `ExternalSecret keycloak-master-platform-app-sync` → `Secret platform-app-master-client`  
  - **Source**: `gitops/ameide-gitops/sources/charts/foundation/common/raw-manifests`  
    - Values: `.../sources/values/_shared/foundation/foundation-keycloak-admin-secrets.yaml` (+ env overlays).  
  - **Notes**: This is the sole owner of these ExternalSecrets in dev after the 2025‑12‑01 refactor (see §4).

- `argocd/platform-keycloak` (instance)  
  - **Responsibility**: Declares the **Keycloak** custom resource (instance: image, DB, hostname, probes, etc.).  
  - **Source**: `gitops/ameide-gitops/sources/charts/foundation/operators-config/keycloak_instance`  
    - Shared values: `.../sources/charts/shared-values/infrastructure/keycloak.yaml`  
    - Platform shared overlay: `.../sources/values/_shared/platform/platform-keycloak.yaml`  
    - Env overlays: `.../sources/values/dev/platform/platform-keycloak.yaml`, `.../sources/values/production/platform/infrastructure/keycloak.yaml`, etc.  
  - **Notes**:
    - Uses `database.secretName: keycloak-db-credentials` (CNPG-owned; see backlog/412-cnpg-owned-postgres-greds.md).  
    - Uses `bootstrapAdmin.secretName: keycloak-bootstrap-admin` (produced by `foundation-keycloak-admin-secrets`).  
    - `externalSecrets.enabled` is **false** at the shared layer; dev no longer overrides this, so the instance consumes Secrets but does not own any ExternalSecrets.

- `argocd/platform-keycloak-realm` (realm config)  
  - **Responsibility**: Realm configuration (`ameide` realm, client scopes, clients, users) and bootstrap jobs.  
  - **Source**: `gitops/ameide-gitops/sources/charts/foundation/operators-config/keycloak_realm`  
    - Values: `.../sources/values/_shared/platform/platform-keycloak-realm.yaml` + env overlays.  
  - **Key resources**:
    - `ConfigMap keycloak-realm-import` – JSON realm definitions (`ameide-realm.json`, etc.).  
    - `KeycloakRealmImport master-realm` / `ameide-realm` – declarative realm imports consumed by the operator.  
    - PostSync Jobs `platform-keycloak-realm-master-bootstrap`, `platform-keycloak-realm-client-patcher` – bootstrap the `roles` scope and service accounts (see 323‑keycloak-realm-roles-v2.md).
  - **Local host coverage**: The `platform-app` client includes `platform.local.ameide.io` in redirect/web origins to keep Tilt-only releases aligned with the shared Keycloak issuer (see 424-tilt-www-ameide-separate-release.md).

**Supporting components:**

- Gateway routing: `gitops/ameide-gitops/sources/charts/apps/gateway/templates/httproute-keycloak.yaml`  
  - Routes `auth.dev.ameide.io` / `auth.ameide.io` to the in-cluster `keycloak` Service with `X-Forwarded-*` headers.  
  - Ties into the Envoy Gateway stack described in 364‑argo-configuration-v5.md and 417-envoy-route-tracking.md.

- Argo CD / Dex integration: `gitops/ameide-gitops/sources/values/common/argocd.yaml`  
  - `dex.config` includes a Keycloak OIDC connector pointing at `http://keycloak.ameide.svc.cluster.local:8080/realms/ameide`, with client `argocd`.  
  - Depends on the `argocd-secret` wiring from `vault-secrets-argocd` (see 364‑argo-configuration-v5.md and infra/README.md).

- Validation helpers:
  - `infra/kubernetes/scripts/validate-keycloak.sh` – CLI script to validate operator, CR, pods, and realm imports.  
  - `gitops/ameide-gitops/sources/charts/shared-values/tests/auth-smoke.yaml` – Argo/Helm test jobs that gate auth readiness (Keycloak CR `Ready`, pods Ready, secrets present, realm bootstrap job done).

---

## 3. Realm & roles configuration (tie-in to 323)

Realm behavior and role/tenant claims are governed by backlog 323:

- **Baseline realm**: single realm `ameide` (see 323-keycloak-realm-roles.md and 333-realms.md).  
  - `KeycloakRealmImport` CRDs (from the `keycloak_realm` chart) embed:
    - Client scopes: `roles`, `tenant`, `groups`, `organizations`.  
    - Default scopes configured per realm and per client (`platform-app`, `platform-app-master`, etc.).  
    - Clients and protocol mappers as described in 323‑v2 (e.g., `realm_access.roles`, `resource_access.${client_id}.roles`, `tenantId` from user attributes).

- **JWT expectations** (323‑v2):  
  - Access tokens include:
    - `realm_access.roles` – realm roles.  
    - `resource_access.{client}.roles` – client roles.  
    - `tenantId` – tenant context (until realm-per-tenant rollout completes).  
  - Application code uses these claims for RBAC and navigation (see 320-header.md, 322-rbac.md, 330-dynamic-tenant-resolution.md).

- **Realm-per-tenant target**:
  - Longer-term goal (Section 4 of 323‑v2 and 333-realms.md) is to move to **realm-per-tenant** and drop the `tenant` and `groups` scopes.
  - This backlog (426) documents the current single-realm baseline and how it is wired through GitOps; it does **not** change the realm-per-tenant roadmap or token shape described in 323‑v2.

### 3.1 OIDC Scopes (2025-12-06)

The `ameide` realm MUST include built-in OIDC client scopes alongside custom scopes:

| Scope | Type | Purpose | Required By |
|-------|------|---------|-------------|
| `openid` | **Protocol scope (meta)** | Mandatory OIDC scope—tells Keycloak "this is an OpenID Connect request". NOT a client scope. | Auth.js Keycloak provider |
| `profile` | **Built-in client scope** | Maps user profile claims (`preferred_username`, `name`, `family_name`, `given_name`) | Auth.js session |
| `email` | **Built-in client scope** | Maps email claims (`email`, `email_verified`) | User identification |
| `offline_access` | **Built-in client scope** | Enables refresh tokens for long-lived sessions | Optional |

**How it works**: When Auth.js sends `scope=openid profile email`:
1. `openid` signals this is an OIDC request (not a client scope lookup)
2. `profile` and `email` are looked up as client scopes linked to `platform-app`
3. Protocol mappers from those scopes populate the token claims

**Warning**: If `clientScopes[]` is present in realm JSON, Keycloak treats it as the COMPLETE list. Missing client scopes cause `invalid_scope` errors.

**Source of realm JSON**: `keycloak-realm-import` should ideally be generated from a Keycloak realm export (`RealmRepresentation`). Treat Keycloak as the configuration authority—export after tests pass, commit to Git.

> **Cross-reference**: See [460-keycloak-oidc-scopes.md](460-keycloak-oidc-scopes.md) for the incident details and vendor-aligned realm pattern.

### 3.2 Client Secret Extraction (client-patcher Job)

**Added**: 2025-12-07

OIDC client secrets in Keycloak follow the **Service-Generated** authority model: Keycloak generates them, and the `client-patcher` Job extracts them to Vault.

#### Architecture

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│    Keycloak     │     │  client-patcher │     │      Vault      │     │ ExternalSecrets │
│  (generates)    │────▶│   (extracts)    │────▶│   (stores)      │────▶│  (syncs to K8s) │
└─────────────────┘     └─────────────────┘     └─────────────────┘     └─────────────────┘
```

#### Helm Hook

The `client-patcher` Job runs as an **Argo CD PreSync hook** (sync wave `-1`). This ensures the job creates clients and writes Vault secrets **before** ExternalSecrets attempt to read them.

```yaml
# keycloak_realm/templates/client-patcher-job.yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
    argocd.argoproj.io/sync-wave: "-1"
```

> **Troubleshooting (2025-12-08)**: When this job was still a PostSync hook, Argo would wait for `ExternalSecret/keycloak-realm-oidc-clients` to be Healthy before allowing the job to run, resulting in a deadlock (`Secret does not exist`). Moving to PreSync unblocks the flow.

#### Vault Policy

The `keycloak-client-patcher` Vault role grants write access to these paths:

| Path Pattern | Clients Stored |
|--------------|----------------|
| `secret/data/platform-app-*` | `platform-app`, `platform-app-master` |
| `secret/data/argocd-*` | `argocd` (Dex OIDC) |
| `secret/data/k8s-dashboard-*` | `k8s-dashboard` |

Policy definition: `sources/charts/foundation/vault-bootstrap/templates/cronjob.yaml`

#### Per-Environment Configuration

Each environment configures `secretExtraction` and the Vault target in its values file:

```yaml
# sources/values/{env}/platform/platform-keycloak-realm.yaml
clientPatcher:
  serviceAccountName: default
  secretExtraction:
    enabled: true
    clients:
      - clientId: platform-app
        vaultKey: platform-app-client-secret
      - clientId: argocd
        vaultKey: argocd-dex-client-secret
      - clientId: k8s-dashboard
        vaultKey: k8s-dashboard-oidc-client-secret
    masterRealmClients:
      - clientId: platform-app-master
        vaultKey: platform-app-master-client-secret
  vault:
    addr: "http://vault-core-{env}.ameide-{env}.svc.cluster.local:8200"
    kvPath: secret
    authRole: keycloak-client-patcher
  loginSecretRef:
    name: keycloak-admin-sa
    clientIdKey: client-id
    clientSecretKey: client-secret
```

#### Flow

1. **Keycloak generates** client secrets during realm import (or regeneration)
2. **client-patcher authenticates via service account** (`loginSecretRef`) and extracts secrets via the Keycloak Admin API
3. **client-patcher writes** to Vault KV v2 at configured paths
4. **ExternalSecrets sync** Vault secrets to Kubernetes Secrets
5. **Applications consume** the synced Secrets (e.g., `AUTH_KEYCLOAK_SECRET`)

#### Service Account Login Secret (2025-12-08)

`foundation-keycloak-admin-secrets` now renders `ExternalSecret/keycloak-admin-sa-sync`, which projects Vault keys `keycloak-admin-sa-client-id` and `keycloak-admin-sa-client-secret` into the namespace-scoped `Secret/keycloak-admin-sa`. Rotate the credentials by updating Vault and forcing an ExternalSecret refresh:

```bash
# Example (dev)
vault kv put secret/keycloak-admin-sa-client-id value=keycloak-admin-sa
vault kv put secret/keycloak-admin-sa-client-secret value='TempAdminSecret2025!'
kubectl -n ameide-dev annotate externalsecret keycloak-admin-sa-sync \
  force-refresh=$(date +%s) --overwrite
```

`kc.sh bootstrap-admin service` spins up an embedded Keycloak to mint the client. Running it **inside** the `keycloak-0` Pod fails because the existing server already binds to port `9000`. Run the bootstrap command from a short-lived Job so it doesn't clash with the StatefulSet process (the Job reuses the ExternalSecret-managed Secret for its env vars):

```bash
kubectl apply -f - <<'EOF'
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
EOF
```

Key points:

1. Pass the same DB connection env vars the StatefulSet uses (mount `keycloak-db-credentials`, hostname, etc.).
2. Set a unique client ID/secret per environment by writing to Vault; `ExternalSecret/keycloak-admin-sa-sync` rehydrates the Kubernetes Secret automatically.
3. Because both the Job and the `client-patcher` read from the same ExternalSecret-managed Secret, client credentials never leave Vault/Kubernetes.

Until Vault automation manages these credentials, this namespace-scoped Secret keeps the GitOps flow unblocked and is referenced directly by the chart. Track the manual Secret ownership in [486-keycloak-admin-recovery.md](486-keycloak-admin-recovery.md).

**Secret rotation (existing service account)**  
Running the bootstrap Job again after the client exists just produces `duplicate key` errors. Rotate secrets in-place instead:

```bash
# 1. Grab the client UUID and ask Keycloak to generate a fresh secret (done from your workstation)
ADMIN_USER=$(kubectl -n ameide-{env} get secret keycloak-bootstrap-admin -o jsonpath='{.data.username}' | base64 -d)
ADMIN_PASS=$(kubectl -n ameide-{env} get secret keycloak-bootstrap-admin -o jsonpath='{.data.password}' | base64 -d)
CLIENT_JSON=$(kubectl -n ameide-{env} exec keycloak-0 -- env ADMIN_USER="$ADMIN_USER" ADMIN_PASS="$ADMIN_PASS" bash -c '
  KCADM=/opt/keycloak/bin/kcadm.sh
  $KCADM config credentials --server http://localhost:8080 --realm master --user "$ADMIN_USER" --password "$ADMIN_PASS" >/dev/null
  $KCADM get clients -r master -q clientId=keycloak-admin-sa
')
CLIENT_ID=$(echo "$CLIENT_JSON" | jq -r '.[0].id')
NEW_SECRET=$(kubectl -n ameide-{env} exec keycloak-0 -- env ADMIN_USER="$ADMIN_USER" ADMIN_PASS="$ADMIN_PASS" CLIENT_ID="$CLIENT_ID" bash -c '
  KCADM=/opt/keycloak/bin/kcadm.sh
  $KCADM config credentials --server http://localhost:8080 --realm master --user "$ADMIN_USER" --password "$ADMIN_PASS" >/dev/null
  $KCADM create clients/$CLIENT_ID/client-secret -r master -o --fields value
' | jq -r '.value')

# 2. Push the new secret to Vault and refresh ESO
vault kv put secret/keycloak-admin-sa-client-secret value="${NEW_SECRET}"
kubectl -n ameide-{env} annotate externalsecret keycloak-admin-sa-sync force-refresh=$(date +%s) --overwrite

# 3. Re-sync the Keycloak realm App so client-patcher picks up the new secret
argocd app sync {env}-platform-keycloak-realm --timeout 600 --grpc-web --plaintext
```

This keeps Keycloak as the authority (per vendor docs), ensures Vault + ExternalSecret stay in sync, and avoids rerunning `bootstrap-admin service` once the temp client already exists.

#### Known Limitation: Create-Only Realm Import (2025-12-07)

⚠️ **Gap Identified**: `KeycloakRealmImport` is create-only - it doesn't update existing realms.

**Implication**: OIDC clients added to Git after initial realm creation are **not created** in Keycloak. The `client-patcher` then skips secret extraction because the client doesn't exist.

**Affected services**:
- `backstage` (blocking)
- Any future OIDC client added post-realm-creation

#### Troubleshooting – Re-running client-patcher & ExternalSecret Refresh (2025-12-08)

1. Re-run the PreSync job via Argo CD so Keycloak clients + Vault secrets are recreated:

   ```bash
   # Requires argocd CLI login / port-forwarding
   argocd app sync staging-platform-keycloak-realm --timeout 600 --grpc-web --plaintext
   argocd app wait staging-platform-keycloak-realm --timeout 600 --grpc-web --plaintext
   ```

   Repeat for production as needed.

Once `client-patcher` writes new client secrets to Vault, the corresponding `ExternalSecret` may keep reporting `SecretSyncedError: Secret does not exist` until it refreshes. Force a refresh without deleting the resource:

```bash
kubectl -n ameide-staging annotate externalsecret keycloak-realm-oidc-clients \
  force-refresh=$(date +%s) --overwrite
# repeat for ameide-prod when needed
```

Watch `kubectl describe externalsecret ...` for `Ready: True`, then confirm the generated `Secret keycloak-realm-oidc-clients` contains the expected keys (`argocd-client-secret`, `backstage-client-secret`, etc.). This manual refresh is safe because the ExternalSecrets operator treats the `force-refresh` annotation as a one-shot signal to re-read Vault.

**Solution**: See [485-keycloak-oidc-client-reconciliation.md](485-keycloak-oidc-client-reconciliation.md) for the declarative fix that extends `client-patcher` to ensure clients exist before extracting secrets.

#### Vault-Bootstrap Idempotency

The `vault-bootstrap` CronJob uses `--check-and-set=0` (CAS) to avoid overwriting Keycloak-generated secrets with fixture values:

```bash
# Only writes if key doesn't exist (CAS=0)
vault kv put -cas=0 secret/platform-app-client-secret value="placeholder"
```

This ensures fixtures are bootstrap-only; once client-patcher writes the real secret, vault-bootstrap won't overwrite it.

> **Cross-reference**: See [462-secrets-origin-classification.md](462-secrets-origin-classification.md) for the secrets authority taxonomy and compliance matrix.

---

## 4. Secrets & ownership model (tie-in to 418, 462)

Backlog 418-secrets-strategy-map.md defines the high-level secrets posture (CNPG owns DB creds, Vault + External Secrets own app credentials, charts consume). Keycloak now conforms to that model.

> **Secret origin classification:** All Keycloak-related secrets follow the authority taxonomy in [462-secrets-origin-classification.md](462-secrets-origin-classification.md):
> - **Database credentials** → Cluster-managed (CNPG-owned)
> - **Admin/bootstrap secrets** → External (Vault/ESO from Azure KV)
> - **OIDC client secrets** (`platform-app`, `platform-app-master`, `argocd`, `k8s-dashboard`) → Cluster-managed (Service-generated, extracted by client-patcher)

- **Database credentials**  
  - Authority: CNPG (see 412-cnpg-owned-postgres-greds.md).  
  - Secret: `keycloak-db-credentials` in namespace `ameide`.  
  - Consumer: `platform-keycloak` via:
    - `database.secretName: keycloak-db-credentials`  
    - `db.existingSecret: keycloak-db-credentials` (for operator-side DB wiring).

- **Admin / bootstrap secrets**  
  - Authority: Vault + External Secrets, encapsulated by `foundation-keycloak-admin-secrets` (Layer 15).  
  - ExternalSecrets (foundation-owned, **not** owned by `platform-keycloak`):
    - `keycloak-bootstrap-admin-sync` → `Secret keycloak-bootstrap-admin`  
    - `keycloak-master-bootstrap-sync` → `Secret keycloak-master-bootstrap`  
    - `keycloak-master-platform-app-sync` → `Secret platform-app-master-client`  
  - Consumers:
    - `platform-keycloak` references `bootstrapAdmin.secretName: keycloak-bootstrap-admin`.  
    - `keycloak_realm` chart uses `platform-app-master-client` via placeholders in `KeycloakRealmImport` for service account credentials.

- **Refactor (2025‑12‑01)**  
  - Prior to this work, the dev overlay at `sources/values/dev/platform/platform-keycloak.yaml` re‑enabled the `externalSecrets` block for the instance, causing:
    - Argo SharedResourceWarning conditions.  
    - Ambiguous ownership of the three Keycloak admin ExternalSecrets.  
  - We removed that overlay so that:
    - `externalSecrets.enabled: false` at the shared `platform-keycloak` layer takes effect in dev.  
    - `foundation-keycloak-admin-secrets` is the **single authoritative owner** of the admin ExternalSecrets, matching the intent in 323‑v2 and 418.

- **What this buys us (relative to 418):**
  - Clear “who owns what” between CNPG, Vault/ExternalSecrets, and the Keycloak instance chart.  
  - No inline credentials in charts; all secrets are sourced from CNPG or Vault.  
  - A simpler rotation story: rotate database creds via CNPG; rotate admin creds via Vault/ExternalSecrets; Keycloak instance and realm charts are strictly consumers.

---

## 5. Health, sync behavior & loops (tie-in to 364, 419, 421)

Argo CD and the Keycloak Operator interact via CRDs and status conditions. This section captures the current health model and how we resolved the Keycloak realm import loop.

- **CRDs + operator readiness**  
  - `platform-keycloak-crds` installs CRDs and uses standard Argo health.  
  - `foundation-keycloak-operator` deploys the operator; readiness is derived from Deployment/Pod status and smoke tests (`auth-smoke` in shared-values/tests).

- **Keycloak instance (`Keycloak` CR)**  
  - Argo Application: `platform-keycloak`.  
  - Health today is based on the operator’s `.status.conditions` for `Keycloak` plus basic Kubernetes health; the instance is considered Healthy when the operator reports Ready and pods are running.  
  - Scripts and smoke tests:
    - `infra/kubernetes/scripts/validate-keycloak.sh` checks the operator, `Keycloak` CR, pods, and realm imports.  
    - `auth-smoke.yaml` includes checks for the `Keycloak` CR `Ready` condition and pods.

- **Realm imports (`KeycloakRealmImport` CRs)**  
  - Argo health customization in `gitops/ameide-gitops/sources/values/common/argocd.yaml`:
    - `resource.customizations.health.k8s.keycloak.org_KeycloakRealmImport` inspects `.status.conditions` while the CR exists:
      - `HasErrors=True` → `Degraded`  
      - `Done=True` → `Healthy`  
      - `Started=True` → `Progressing`  
      - Otherwise → `Progressing` (“Waiting for realm import”).  
  - Vendor guidance treats `KeycloakRealmImport` as an **ephemeral, create-only** resource: it bootstraps a realm and is deleted after a successful import, rather than serving as long-lived configuration.
  - Our GitOps implementation follows that model by:
    - Annotating each `KeycloakRealmImport` rendered by `keycloak_realm/templates/realm-import.yaml` as an Argo CD `PostSync` hook with `argocd.argoproj.io/hook: PostSync` and `argocd.argoproj.io/hook-delete-policy: HookSucceeded` (plus `sync-wave: "10"`).  
    - Allowing Argo to create the CR at the end of the sync, wait for `Done=True` / `HasErrors` via the health Lua, and then prune the CR once the import succeeds.  
    - Relying on the operator’s create-only semantics: re-running the same RealmImport on later syncs does not update or delete existing realms; it is effectively a no-op when the realm is already present.

- **Argo orchestration & waves**  
  - The ordering follows the RollingSync and wave design in 364‑argo-configuration-v5.md and 375-rolling-sync-wave-redesign.md:
    - CRDs → operator → secrets (Vault / ExternalSecrets) → Keycloak instance → realm config / imports → dependent services (Dex, gateway routes, apps).  
  - This ensures:
    - The operator is live before `Keycloak` / `KeycloakRealmImport` CRs appear.  
    - Admin secrets exist before Keycloak tries to bootstrap admin users or service accounts.  
    - Dex and the gateway are wired only after the `ameide` realm is reachable.

### 5.1 Import Job Scheduling (2025-12-06)

Keycloak realm imports run as Jobs created by the Keycloak operator. Per [operator docs](https://www.keycloak.org/operator/advanced-configuration):

- Import jobs **inherit** tolerations/nodeSelector from `spec.scheduling` on the Keycloak CR
- `spec.unsupported.podTemplate` applies **only to server pods**, NOT import jobs
- When using node taints (e.g., `ameide.io/environment=dev:NoSchedule`), `spec.scheduling.tolerations` MUST be set

**Values file pattern** (per-environment):

```yaml
# sources/values/{env}/platform/platform-keycloak.yaml

# Import jobs inherit from spec.scheduling
scheduling:
  tolerations:
    - key: "ameide.io/environment"
      value: "dev"
      effect: "NoSchedule"

# Server pods use unsupported.podTemplate for nodeSelector
# (spec.scheduling.nodeSelector not yet supported by operator)
unsupported:
  podTemplate:
    spec:
      nodeSelector:
        ameide.io/pool: dev
```

**Helm template** (`keycloak_instance/templates/keycloak.yaml`):

```yaml
{{- with .Values.scheduling }}
scheduling:
{{ toYaml . | indent 4 }}
{{- end }}
```

> **Cross-reference**: See [442-environment-isolation.md](442-environment-isolation.md) for the node pool strategy and [447-third-party-chart-tolerations.md](447-third-party-chart-tolerations.md) §12 for Keycloak-specific details.

---

## 6. Cross-references & next steps

Use this backlog as the **entry point**; follow these for deeper dives:

- **Realm & roles**
  - `backlog/323-keycloak-realm-roles.md` – original realm roles design.
  - `backlog/323-keycloak-realm-roles-v2.md` – current single-realm `ameide` baseline and realm-per-tenant roadmap.
  - `backlog/333-realms.md` – broader realm/multi-tenancy architecture.
  - `backlog/460-keycloak-oidc-scopes.md` – OIDC scopes incident and vendor-aligned realm pattern.

- **Argo CD / GitOps orchestration**  
  - `backlog/364-argo-configuration-v5.md` – Argo CD layering, RollingSync, and argocd-cm health overrides.  
  - `backlog/375-rolling-sync-wave-redesign.md` – wave-level ordering for foundation → data → platform → apps.  
  - `backlog/419-clickhouse-argocd-resiliency.md`, `421-argocd-strimzi-kafkanodepool-health.md` – patterns for other operators’ health checks (useful analogs for Keycloak).

- **Secrets & Vault**  
  - `backlog/418-secrets-strategy-map.md` – secrets authority map and gaps.  
  - `backlog/412-cnpg-owned-postgres-greds.md` – CNPG ownership of DB credentials.  
  - `backlog/413-vault-bootstrap-readiness.md` – Layer 15 bootstrap/readiness model.

- **Operational runbooks**  
  - `infra/README.md` – Authentication architecture section (URLs, flows, Redis sessions).  
  - `infra/kubernetes/scripts/validate-keycloak.sh` – operator/realm validation.  
  - `gitops/ameide-gitops/sources/charts/shared-values/tests/auth-smoke.yaml` – auth smoke tests run via Argo Helm test jobs.

**Future work for this backlog (426):**

- [ ] Add an explicit `Keycloak` CR health customization in `argocd-cm` mirroring the operator's `Ready` condition (following the pattern used for Strimzi/Kafka).
- [ ] Document a rotation runbook for admin credentials (`keycloak-bootstrap-admin`, `keycloak-master-bootstrap`, `platform-app-master-client`) that ties Vault paths → ExternalSecrets → Keycloak operator behavior.
- [ ] Update this map once realm-per-tenant becomes the default, especially how realm imports and Keycloak instances are templated per organization.

---

## 7. Vendor Documentation Validation (2025-12-07)

All architectural decisions in this document have been validated against upstream vendor documentation.

### 7.1 Bootstrap Admin Behavior

**Statement**: `KC_BOOTSTRAP_ADMIN_*` environment variables are only read when the master realm is created. Subsequent changes to the `keycloak-bootstrap-admin` Secret are ignored.

**Vendor Confirmation**:
- Keycloak "all config" docs: *"Temporary bootstrap admin password. **Used only when the master realm is created.**"* ([source](https://www.keycloak.org/server/all-config))
- RHBK Operator Guide: *"If a master realm has already been created for your cluster, then the spec.bootstrapAdmin is effectively ignored."* ([source](https://docs.redhat.com/en/documentation/red_hat_build_of_keycloak/26.2/html-single/operator_guide/))

**Implication**: Admin credential drift cannot be fixed by updating the Secret. Use `kc.sh bootstrap-admin service` for recovery. See [486-keycloak-admin-recovery.md](486-keycloak-admin-recovery.md).

### 7.2 KeycloakRealmImport CREATE-ONLY Behavior

**Statement**: `KeycloakRealmImport` CRs only create new realms; they do not update or delete existing realms.

**Vendor Confirmation**:
- RHBK Realm Import docs: *"If a Realm with the same name already exists… it will not be overwritten."* *"The Realm Import CR only supports creation of new realms and does not update or delete those."* ([source](https://docs.redhat.com/pt-br/documentation/red_hat_build_of_keycloak/22.0/html/operator_guide/realm-import-))

**Implication**: OIDC clients added to Git after initial realm creation require the `client-patcher` reconciliation pattern. See [485-keycloak-oidc-client-reconciliation.md](485-keycloak-oidc-client-reconciliation.md).

### 7.3 ArgoCD PostSync Hook Requirements

**Statement**: PostSync hooks only run after all resources are Healthy. This creates bootstrap deadlocks when secrets depend on PostSync operations.

**Vendor Confirmation**:
- ArgoCD Resource Hooks: *"PostSync executes after all Sync hooks completed and were successful, **a successful application, and all resources in a Healthy state.**"* ([source](https://argo-cd.readthedocs.io/en/release-2.0/user-guide/resource_hooks/))

**Implication**: Bootstrap operations that must complete before ExternalSecrets can sync must use PreSync hooks with sync-waves.

### 7.4 OIDC Client Secrets are Service-Generated

**Statement**: Keycloak generates OIDC client secrets; they should not be stored in external systems like Azure Key Vault.

**Vendor Confirmation**:
- Keycloak Confidential Client Credentials: *"The secret is automatically generated for you and the **Regenerate Secret** button allows you to recreate this secret if you want or need to."* ([source](https://wjw465150.gitbooks.io/keycloak-documentation/content/server_admin/topics/clients/oidc/confidential.html))

**Implication**: Use `client-patcher` to extract Keycloak-generated secrets to Vault. Do not use Azure KV fixtures for OIDC secrets. See [462-secrets-origin-classification.md](462-secrets-origin-classification.md).

### 7.5 Service Account Authentication for Automation

**Statement**: Automated jobs (like `client-patcher`) should use service account client credentials rather than admin user/password.

**Vendor Confirmation**:
- Keycloak Admin CLI: *"When logging in with Admin CLI you specify… username, or alternatively **only specify a client id**, which will result in special service account being used… When you log in using a clientId, you need the client secret only."* ([source](https://wjw465150.gitbooks.io/keycloak-documentation/content/server_admin/topics/admin-cli.html))
- Keycloak Service Accounts: *"Service accounts are explicitly recommended for non-human / automated tasks and use the **client credentials** grant."* ([source](https://www.keycloak.org/securing-apps/client-registration-cli))

**Implication**: Switch `client-patcher` to service account auth to avoid admin password drift issues. See [486-keycloak-admin-recovery.md](486-keycloak-admin-recovery.md).

### 7.6 Helm Values Merge Behavior

**Statement**: Helm performs key-based map merges; environment values can replace entire sections.

**Vendor Confirmation**:
- Helm Charts: *"When a user supplies custom values, these values will override the values in the chart's values.yaml file."* ([source](https://helm.sh/docs/topics/charts))

**Clarification**: Helm merges maps recursively at key boundaries. If an environment values file defines `clientPatcher:`, it replaces the base `clientPatcher` map unless subkeys are re-declared.
