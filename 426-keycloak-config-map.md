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
    - Env overlays: `.../sources/values/env/dev/platform/platform-keycloak.yaml`, `.../sources/values/env/production/platform/infrastructure/keycloak.yaml`, etc.  
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
# sources/values/env/{env}/platform/platform-keycloak-realm.yaml
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

#### Service Account Login Secret (2026-01-18)

`client-patcher` authenticates to the Keycloak Admin API using:

1. **Primary**: `clientPatcher.loginSecretRef` (a service-account client credential stored in `Secret/keycloak-admin-sa`, projected by ESO from Vault).
2. **Fallback**: bootstrap admin authentication (break-glass) to avoid “auth broken → patcher can’t run → secrets can’t converge” deadlocks.

**GitOps runbook** (shared AKS clusters):

- Do not “fix” this path with imperative `kubectl apply/delete/annotate` actions.
- To force a clean `client-patcher` re-run, bump `clientPatcher.runId` in the environment values and sync `platform-keycloak-realm` via Argo CD.
- ESO refresh is periodic; `platform-secrets-smoke` provides retries + enforcement and fails the sync if placeholders remain or digests mismatch.

#### Known Limitation: Create-Only Realm Import (2025-12-07)

⚠️ **Gap Identified**: `KeycloakRealmImport` is create-only - it doesn't update existing realms.

**Implication**: OIDC clients added to Git after initial realm creation are **not created** in Keycloak. The `client-patcher` then skips secret extraction because the client doesn't exist.

**Affected services**:
- `backstage` (blocking)
- Any future OIDC client added post-realm-creation

#### Troubleshooting – Re-running client-patcher (GitOps)

1. Bump `clientPatcher.runId` in `sources/values/env/{env}/platform/platform-keycloak-realm.yaml`.
2. Sync `{env}-platform-keycloak-realm` in Argo CD and wait for health.
3. Use `platform-secrets-smoke` as the enforcement signal that ESO has caught up and secrets are non-placeholder.

**Solution**: See [485-keycloak-oidc-client-reconciliation.md](485-keycloak-oidc-client-reconciliation.md) for the declarative fix that extends `client-patcher` to ensure clients exist before extracting secrets.

#### Vault-Bootstrap Idempotency

The `vault-bootstrap` CronJob uses check-and-set semantics (CAS) to avoid overwriting Keycloak-generated secrets with fixture values. Fixtures are bootstrap-only; once `client-patcher` writes the real secret, vault-bootstrap does not overwrite it.

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
  - Prior to this work, the dev overlay at `sources/values/env/dev/platform/platform-keycloak.yaml` re‑enabled the `externalSecrets` block for the instance, causing:
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
# sources/values/env/{env}/platform/platform-keycloak.yaml

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

### 5.2 ArgoCD Logs & Health Investigation (2025-12-09)

Argo reports `platform-keycloak-realm` in every environment. When it goes `Progressing`/`Degraded`, use the same checklist so that hooks, Jobs, and ExternalSecrets are verified before re-syncing.

| Environment | Namespace | Argo CD Application | Hooks / resources to watch |
|-------------|-----------|---------------------|----------------------------|
| dev | `ameide-dev` | `dev-platform-keycloak-realm` | `KeycloakRealmImport` (hook wave `-10`), `Job/platform-keycloak-realm-client-patcher` (hook wave `-1`), `ExternalSecret/keycloak-realm-oidc-clients` |
| staging | `ameide-staging` | `staging-platform-keycloak-realm` | Same hooks as dev |
| production | `ameide-prod` | `production-platform-keycloak-realm` | Same hooks as dev |

#### 1. Sweep Argo health for all environments

```bash
envs=(dev staging production)
for env in "${envs[@]}"; do
  app="${env}-platform-keycloak-realm"
  echo "=== ${app} ==="
  argocd app get "${app}" \
    --grpc-web --plaintext -o json \
    | jq -r '"Sync=\(.status.sync.status) Health=\(.status.health.status) Revision=\(.status.sync.revision)"'
done
```

* `Sync=OutOfSync` usually means the PreSync hook deleted objects before a fresh apply—check events.
* `Health=Progressing` while `Sync=Synced` indicates Argo is still waiting on the Keycloak operator to report `Done=True` / `HasErrors=False` on the hook resources.
* `argocd app wait ${env}-platform-keycloak-realm --health --timeout 600 --grpc-web --plaintext` blocks until the health gate clears.

#### 2. Inspect events and hook logs

```bash
envs=(dev staging production)
for env in "${envs[@]}"; do
  app="${env}-platform-keycloak-realm"
  echo "--- ${app} recent events ---"
  argocd app events "${app}" --grpc-web --plaintext --tail 20
done

# Focus on the PreSync client-patcher Job without leaving the CLI context
argocd app logs dev-platform-keycloak-realm \
  --grpc-web --plaintext \
  --group batch --kind Job \
  --namespace ameide-dev \
  --name platform-keycloak-realm-client-patcher \
  --container client-patcher \
  --since-seconds 3600 --tail 200
```

If the CLI is not logged in, port-forward/login first (same pattern as §3.2 above). When granularity is needed, pull logs directly via Kubernetes:

```bash
kubectl logs -n ameide-staging \
  job/platform-keycloak-realm-client-patcher \
  -c client-patcher --tail=200 --previous=false
```

#### 3. Validate hook resources inside Kubernetes

```bash
envs=(dev staging production)
for env in "${envs[@]}"; do
  ns="ameide-${env}"
  echo "== ${ns} =="
  kubectl -n "${ns}" get jobs \
    -l app.kubernetes.io/name=keycloak-realm-client-patcher \
    -o custom-columns=NAME:.metadata.name,SUCCEEDED:.status.succeeded,ACTIVE:.status.active,START:.status.startTime

  # KeycloakRealmImport CRs are ephemeral: they exist only while the import runs.
  kubectl -n "${ns}" get keycloakrealmimports.k8s.keycloak.org \
    -l app.kubernetes.io/instance=platform-keycloak-realm -o yaml || true

  kubectl -n "${ns}" get externalsecret keycloak-realm-oidc-clients \
    -o jsonpath='{.status.conditions[?(@.type=="Ready")].reason}{" => "}{.status.conditions[?(@.type=="Ready")].status}{"\n"}'

  kubectl -n "${ns}" get cm keycloak-client-secret-versions -o jsonpath='{.metadata.name}{" updated at "}{.metadata.annotations.argocd\\.argoproj\\.io/sync-wave}{"\n"}' || true
done
```

Expected outcomes:

- `Job/platform-keycloak-realm-client-patcher` shows `SUCCEEDED=1` (or `ACTIVE=0` if it finished and Argo already deleted the hook).
- `KeycloakRealmImport` manifests may already be gone because the hook uses `hook-delete-policy: BeforeHookCreation`; if they linger, ensure `.status.conditions` report `Done=True` and `HasErrors=False`.
- `ExternalSecret/keycloak-realm-oidc-clients` should read `Ready=True` within a minute after the Job rewrites Vault.
- `ConfigMap/keycloak-client-secret-versions` timestamps advance whenever the client-patcher patches the digest list; stale timestamps hint that Vault was not updated.

#### 4. When health stays degraded

1. Force a clean `client-patcher` re-run by bumping `clientPatcher.runId` (Git) and syncing the Application.
2. Re-sync and watch until health clears:
   ```bash
   argocd app sync ${env}-platform-keycloak-realm --grpc-web --plaintext --timeout 600
   argocd app wait ${env}-platform-keycloak-realm --health --timeout 600 --grpc-web --plaintext
   ```
3. If `KeycloakRealmImport` reports errors, capture the operator pod logs (`kubectl logs deployment/keycloak-operator -n keycloak-system`) and record the failing realm JSON before retrying.

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
