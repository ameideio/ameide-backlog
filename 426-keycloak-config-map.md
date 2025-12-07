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

The `client-patcher` Job is a Helm **post-install/post-upgrade** hook in the `keycloak_realm` chart:

```yaml
# keycloak_realm/templates/client-patcher-job.yaml
annotations:
  "helm.sh/hook": post-install,post-upgrade
  "helm.sh/hook-weight": "10"
  "helm.sh/hook-delete-policy": before-hook-creation
```

#### Vault Policy

The `keycloak-client-patcher` Vault role grants write access to these paths:

| Path Pattern | Clients Stored |
|--------------|----------------|
| `secret/data/platform-app-*` | `platform-app`, `platform-app-master` |
| `secret/data/argocd-*` | `argocd` (Dex OIDC) |
| `secret/data/k8s-dashboard-*` | `k8s-dashboard` |

Policy definition: `sources/charts/foundation/vault-bootstrap/templates/cronjob.yaml`

#### Per-Environment Configuration

Each environment configures `secretExtraction` in its values file:

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
```

#### Flow

1. **Keycloak generates** client secrets during realm import (or regeneration)
2. **client-patcher extracts** secrets via Keycloak Admin API using admin credentials
3. **client-patcher writes** to Vault KV v2 at configured paths
4. **ExternalSecrets sync** Vault secrets to Kubernetes Secrets
5. **Applications consume** the synced Secrets (e.g., `AUTH_KEYCLOAK_SECRET`)

#### Known Limitation: Create-Only Realm Import (2025-12-07)

⚠️ **Gap Identified**: `KeycloakRealmImport` is create-only - it doesn't update existing realms.

**Implication**: OIDC clients added to Git after initial realm creation are **not created** in Keycloak. The `client-patcher` then skips secret extraction because the client doesn't exist.

**Affected services**:
- `backstage` (blocking)
- Any future OIDC client added post-realm-creation

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

- [ ] Add an explicit `Keycloak` CR health customization in `argocd-cm` mirroring the operator’s `Ready` condition (following the pattern used for Strimzi/Kafka).  
- [ ] Document a rotation runbook for admin credentials (`keycloak-bootstrap-admin`, `keycloak-master-bootstrap`, `platform-app-master-client`) that ties Vault paths → ExternalSecrets → Keycloak operator behavior.  
- [ ] Update this map once realm-per-tenant becomes the default, especially how realm imports and Keycloak instances are templated per organization.
