# 467 – Backstage Platform Factory

**Status:** In Progress
**Priority:** High
**Complexity:** Large
**Created:** 2025-12-07
**Updated:** 2025-12-07

> **Cross-References (Vision Suite)**:
>
> | Document | Relationship |
> |----------|-------------|
> | [470-ameide-vision.md](470-ameide-vision.md) | §5.3, §8 – Backstage template runs as "transformation records" |
> | [471-ameide-business-architecture.md](471-ameide-business-architecture.md) | §4 – Backstage as platform factory (business view) |
> | [472-ameide-information-application.md](472-ameide-information-application.md) | §4 – Catalog modeling, entity mappings |
> | [473-ameide-technology.md](473-ameide-technology.md) | §2.3 – Backstage as "factory" for primitives |
> | [475-ameide-domains.md](475-ameide-domains.md) | §2 Principle 5, §6 – Internal factory, not tenant UI |
> | [476-ameide-security-trust.md](476-ameide-security-trust.md) | §9.1 – Backstage security baseline |
> | [478-ameide-extensions.md](478-ameide-extensions.md) | Extension scaffolding workflow |
>
> **Deployment Architecture Suite**:
>
> | Document | Relationship |
> |----------|-------------|
> | [465-applicationset-architecture.md](465-applicationset-architecture.md) | How Backstage app is generated via ApplicationSet |
> | [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Rollout phase 356 (sub-phase `*56` post-bootstrap service) |
> | [464-chart-folder-alignment.md](464-chart-folder-alignment.md) | Chart at sources/charts/platform/backstage |
> | [426-keycloak-config-map.md](426-keycloak-config-map.md) | OIDC client pattern for Backstage |

---

## Update (2025-12-13): Local runtime postmortem (arm64)

**Symptoms**
- `platform-backstage` repeatedly failed health probes and restarted (often `ExitCode=137`), leaving ArgoCD `Degraded/ProgressDeadlineExceeded`.

**Root cause**
- The local k3d cluster runs on `arm64` nodes; `quay.io/janus-idp/backstage-showcase:latest` is `linux/amd64` only, so pods ran under emulation and were unstable.
- Vendor image layout differences (`/opt/app-root/src/...` vs `/app/...`) also matter for config + migration mounts.

**Fix shipped (GitOps)**
- Local overrides now use an `arm64`-capable Backstage image (multi-arch) and adjust config/migration mount paths.
- The Backstage chart gained explicit knobs so image filesystem/layout differences are handled declaratively (chart-level capability, not “hand edited manifests”).

**Follow-up**
- Decide whether we want to keep Janus IDP in hosted environments and, if so, ensure the chosen tag is multi-arch (or publish our own multi-arch mirror).

## Update (2025-12-17): AKS production CrashLoop from DB migration lock

**Symptoms**
- `production-platform-backstage` stuck `Progressing` with the pod CrashLooping.
- Logs show Knex migration lock errors:
  - `Can't take lock to run migrations: Migration table is already locked`
  - `MigrationLocked: Plugin 'catalog' startup failed`

**Working hypothesis**
- Backstage runs plugin DB migrations automatically during startup. Under restarts and/or concurrent rollout attempts, the migration lock can remain held and the pod keeps crashing before becoming Ready.

**Fix direction (GitOps-idempotent, vendor-aligned)**
1. Ensure Backstage DB migrations are executed in a controlled, single-run path (e.g., a dedicated Kubernetes Job or initContainer pattern that runs once per deployment revision).
2. Prevent concurrent migration attempts during rollout (e.g., `maxSurge: 0` for the Deployment, and avoid multiple replicas until migrations are stable).
3. Avoid “manual unlock” as a runbook step; treat it as a last-resort break-glass only.

**Recovery implemented (GitOps, deterministic)**
- Added an optional `Job` in `sources/charts/platform/backstage` that clears a stuck Knex migration lock in Postgres (updates `public.knex_migrations_lock.is_locked` when present).
- Enabled it in production via `sources/values/env/production/platform/platform-backstage.yaml` to recover the CrashLoop without manual `kubectl exec` / DB edits.
- Follow-up: disable the job once production is green again (keep the capability but default `enabled=false`).
- **Note:** if Argo shows a `ComparisonError` and the job never appears, check for Helm render failures in `templates/migrations-unlock-job.yaml` (a heredoc terminator must remain indented inside the YAML block scalar, or Helm will emit invalid YAML).
- **Note:** prefer a no-overlap `RollingUpdate` (`maxSurge: 0`, `maxUnavailable: 1`) for steady-state; if using `strategy.type=Recreate`, the chart must not render a `strategy.rollingUpdate` stanza (server-side diff/validation will fail, and SSA transitions can be tricky if `rollingUpdate` is owned by another field manager).
- **Note:** keep the unlock SQL runner conservative: prefer a `to_regclass(...)` presence check + `UPDATE ...` over a `DO $$ ... $$;` block to avoid quoting/termination pitfalls in minimal `sh` jobs.

## 1. Executive Summary

Backstage is designated as the Ameide "platform factory" – the internal developer portal that provides:

1. **Software Catalog** – Registry of Ameide primitives and their APIs
2. **Software Templates** – Scaffolder templates for creating new primitives
3. **TechDocs** – Technical documentation aggregation
4. **Integration Hub** – GitHub, Keycloak, ArgoCD integrations
5. **Primitive CR authoring** – Templates emit primitive CRDs so GitOps/Argo manage declarative primitive specs instead of raw deployments.

Per architecture doc 473 §4: "Backstage's Software Catalog and Software Templates / Scaffolder are used to create and manage all Ameide services (domain/process/agent) and their Helm charts."

---

## 2. Goals

1. Deploy Backstage as a platform service at phase 356 (sub-phase `*55`)
2. Integrate with Keycloak for OIDC authentication
3. Provision PostgreSQL database via existing CNPG cluster
4. Configure HTTPRoute for external access via Gateway API
5. Establish foundation for Software Templates

---

## 3. Non-Goals (Future Work)

- Custom Backstage Docker image with advanced plugins
- Full Software Template suite (Domain primitive, Process primitive, Agent primitive, UISurface)
- Catalog population with all existing services
- TechDocs publishing pipeline
- Keycloak organization/group sync
- ArgoCD plugin integration

---

## 4. Implementation Plan

### Phase 1: Database Setup

**Task:** Add `backstage` database to CNPG cluster

**File:** `sources/values/_shared/data/platform-postgres-clusters.yaml`

```yaml
# Add to managed.roles:
- name: backstage
  ensure: present
  login: true
  createdb: true  # Required for backstage_plugin_* databases
  passwordSecret:
    name: backstage-db-credentials

# Add to credentials.appUsers:
- name: backstage
  database: backstage
  username: backstage
  secretName: backstage-db-credentials
  template: backstage

# Add to databases:
- name: backstage
  owner: backstage
  schemas:
    - name: public
      owner: backstage
```

**Dependencies:** CNPG operator (phase 020), postgres-clusters (phase 250)

---

### Phase 2: Helm Chart

**Task:** Create custom Backstage chart

**Location:** `sources/charts/platform/backstage/`

**Structure:**
```
sources/charts/platform/backstage/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── service.yaml
    ├── httproute.yaml
    ├── configmap.yaml
    └── serviceaccount.yaml
```

**Key Templates:**

| Template | Purpose | Pattern Reference |
|----------|---------|-------------------|
| deployment.yaml | Backstage container | `www-ameide-platform/templates/deployment.yaml` |
| httproute.yaml | Gateway API routing | `www-ameide-platform/templates/httproute.yaml` |
| configmap.yaml | app-config.yaml | Custom for Backstage |

**Note:** Database credentials are managed by CNPG (per [412-cnpg-owned-postgres-greds.md](412-cnpg-owned-postgres-greds.md)), not ExternalSecrets. The deployment uses `envFrom` to inject credentials from the CNPG-generated secret.

**Backstage Configuration (app-config.yaml):**

```yaml
app:
  title: Ameide Platform Factory
  baseUrl: https://backstage.dev.ameide.io

backend:
  baseUrl: https://backstage.dev.ameide.io
  listen:
    port: 7007
  database:
    client: pg
    connection:
      host: ${POSTGRES_HOST}
      port: ${POSTGRES_PORT}
      user: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}
      database: backstage

auth:
  environment: production
  providers:
    oidc:
      production:
        metadataUrl: ${KEYCLOAK_ISSUER}/.well-known/openid-configuration
        clientId: backstage
        clientSecret: ${BACKSTAGE_OIDC_CLIENT_SECRET}
        prompt: auto
        signIn:
          resolvers:
            - resolver: emailMatchingUserEntityProfileEmail

catalog:
  rules:
    - allow: [Component, System, API, Resource, Location, Template, Group, User]
  locations:
    - type: url
      target: https://github.com/ameideio/ameide-gitops/blob/main/sources/backstage/catalog/all.yaml
      rules:
        - allow: [Location]
```

---

### Phase 3: Component Definition

**Task:** Register Backstage as ArgoCD component

**File:** `environments/_shared/components/platform/developer/backstage/component.yaml`

```yaml
name: platform-backstage
project: ameide
domain: platform
dependencyPhase: "platform"
componentType: "workload"
rolloutPhase: "356"
autoSync: true
chart:
  repoURL: https://github.com/ameideio/ameide-gitops.git
  path: sources/charts/platform/backstage
  version: main
syncOptions:
  - CreateNamespace=true
  - RespectIgnoreDifferences=true
  - SkipDryRunOnMissingResource=true
syncPolicy:
  automated:
    prune: true
    selfHeal: true
orphanedResources: {}
```

---

### Phase 4: Values Files

**Files to create:**

| File | Purpose |
|------|---------|
| `sources/values/_shared/platform/platform-backstage.yaml` | Shared configuration |
| `sources/values/env/dev/platform/platform-backstage.yaml` | Dev environment |
| `sources/values/env/staging/platform/platform-backstage.yaml` | Staging environment |
| `sources/values/env/production/platform/platform-backstage.yaml` | Production environment |

**Shared Values:**

```yaml
tier: platform
domain: platform
exposure: external

replicaCount: 1

image:
  repository: ghcr.io/backstage/backstage
  tag: "1.35.0"
  pullPolicy: IfNotPresent

imagePullSecrets:
  - name: ghcr-pull

service:
  type: ClusterIP
  port: 7007

resources:
  requests:
    cpu: 200m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi

postgresql:
  host: postgres-ameide-rw
  port: 5432
  database: backstage

httproute:
  enabled: true
  gateway: ameide
  sectionName: https

externalSecrets:
  enabled: true
  storeRef:
    kind: SecretStore
    name: ameide-vault
  refreshInterval: 1h
```

---

### Phase 5: Keycloak Integration

**Task:** Add Backstage OIDC client to Keycloak realm

**Location:** Keycloak realm configuration

```json
{
  "clientId": "backstage",
  "name": "Backstage Developer Portal",
  "description": "OIDC client for Backstage platform factory",
  "enabled": true,
  "publicClient": false,
  "standardFlowEnabled": true,
  "implicitFlowEnabled": false,
  "directAccessGrantsEnabled": false,
  "protocol": "openid-connect",
  "redirectUris": [
    "https://backstage.dev.ameide.io/api/auth/oidc/handler/frame",
    "https://backstage.staging.ameide.io/api/auth/oidc/handler/frame",
    "https://backstage.ameide.io/api/auth/oidc/handler/frame"
  ],
  "webOrigins": [
    "https://backstage.dev.ameide.io",
    "https://backstage.staging.ameide.io",
    "https://backstage.ameide.io"
  ],
  "defaultClientScopes": ["profile", "email", "roles"]
}
```

**Vault Secret:** Store client secret at `backstage-oidc-client-secret`

---

## 5. Technical Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Chart type | Custom wrapper | Matches existing Ameide patterns; easier integration |
| Authentication | Native OIDC via Janus IDP | Vendor-aligned with Red Hat Developer Hub; OIDC pre-configured |
| Database | Shared CNPG cluster | Follows existing pattern; no separate PostgreSQL |
| Rollout phase | 356 (platform, sub-phase `*55`) | Post-runtime bootstrap after Keycloak (350) + client-patcher (355) |
| Port | 7007 | Backstage default port |
| Docker image | `quay.io/janus-idp/backstage-showcase` | Production-ready OIDC support; aligns with RHDH vendor docs |

### 5.0.1 Vendor Alignment Decision (2025-12-07)

**Problem**: Stock `ghcr.io/backstage/backstage` image doesn't include OIDC provider module in backend.

**Vendor guidance** (Red Hat Developer Hub docs):
1. Backstage requires `@backstage/plugin-auth-backend-module-oidc-provider` registered in backend
2. Frontend needs SignInPage configured for OIDC
3. `auth.session.secret` is required for cookie signing

**Solution**: Use **Janus IDP** (`quay.io/janus-idp/backstage-showcase`) which:
- Ships with OIDC provider pre-configured
- Has SignInPage wired for OIDC
- Follows Red Hat Developer Hub patterns

**References**:
- [Red Hat Developer Hub Authentication](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.5/html/authentication_in_red_hat_developer_hub/)
- [Backstage OIDC Provider](https://backstage.io/docs/auth/oidc/)

---

## 5.1 Third-Party Chart Vendoring Strategy

Per [464-chart-folder-alignment.md](464-chart-folder-alignment.md) §1, third-party charts are vendored for reproducibility and auditability.

### Decision: Custom Wrapper vs Vendored Upstream

| Option | Description | When to Use |
|--------|-------------|-------------|
| **Custom wrapper** | Write our own chart with Deployment, Service, HTTPRoute | Simple services; need full control over templates |
| **Vendored upstream** | Copy official chart to `third_party/` and configure via values | Complex services with many features (subcharts, CRDs) |
| **Wrapper + dependency** | Our chart declares upstream as dependency in Chart.yaml | Need upstream complexity + custom templates |

**Backstage choice**: **Custom wrapper**

**Rationale**:
- Backstage official chart has heavy dependencies (Redis, bundled PostgreSQL, nginx)
- We already have CNPG and Gateway API infrastructure
- Database credentials use CNPG-managed secrets (per 412), not ExternalSecrets
- Custom wrapper integrates cleanly with existing patterns
- Simpler upgrade path (just update image tag)

### If Vendoring Were Needed

For future components requiring vendored upstream charts:

```bash
# Standard vendoring pattern
sources/charts/third_party/{registry}/{chart-name}/{version}/
├── Chart.yaml
├── values.yaml
├── templates/
└── charts/            # Subcharts (if any)
```

Example vendors in use:
```
third_party/
├── backstage/           # (if we vendored Backstage)
├── cnpg/cloudnative-pg/0.26.1/
├── grafana/loki/6.46.0/
├── temporal/temporal/0.70.0/
├── strimzi/strimzi-kafka-operator/0.48.0/
└── prometheus-community/kube-prometheus-stack/79.1.1/
```

---

## 5.2 Authentication Integration (OIDC + client-patcher)

Backstage uses the standard Ameide OIDC pattern with Keycloak. Per [426-keycloak-config-map.md](426-keycloak-config-map.md) §3.2:

### OIDC Client Configuration

The `backstage` client is defined in the Keycloak realm config and extracted via the `client-patcher` Job:

```yaml
# Per-env platform-keycloak-realm.yaml
clientPatcher:
  secretExtraction:
    clients:
      - clientId: backstage
        realm: ameide
        vaultPath: secret/backstage-oidc-client-secret
```

### Authentication Flow

```
┌─────────────┐     ┌─────────────┐     ┌───────────────┐     ┌───────────┐
│  Backstage  │────▶│  Keycloak   │────▶│ client-patcher│────▶│   Vault   │
│   (OIDC)    │     │   Realm     │     │     Job       │     │           │
└─────────────┘     └─────────────┘     └───────────────┘     └───────────┘
       │                                                             │
       │  ┌─────────────────────────────────────────────────────────┐
       └──│  ExternalSecret syncs secret from Vault to K8s          │
          └─────────────────────────────────────────────────────────┘
```

### Phase Ordering

| Phase | Component | Description |
|-------|-----------|-------------|
| 350 | Keycloak | Creates realm with `backstage` client |
| 355 | client-patcher | Extracts client secret to Vault |
| 356 | Backstage | Starts with working OIDC (ExternalSecret syncs immediately) |

**Note**: Phase 356 follows the sub-phase convention (see 447 "Sub-phase Convention" and "Platform (300-399)" band details).

### RBAC Integration

Backstage roles should map to Keycloak realm roles:

| Backstage Role | Keycloak Role | Permissions |
|----------------|---------------|-------------|
| `backstage-admin` | `platform-admin` | Full access to catalog, templates |
| `backstage-editor` | `developer` | Create from templates, edit catalog |
| `backstage-viewer` | `viewer` | Read-only catalog access |

### 5.2.1 OIDC Client Reconciliation Gap (2025-12-07)

**Status**: ⚠️ Blocking Issue Identified

**Problem**: The `backstage` OIDC client is defined in Git but does not exist in Keycloak.

**Root Cause Analysis**:

1. `KeycloakRealmImport` is **create-only** (vendor behavior per [426-keycloak-config-map.md](426-keycloak-config-map.md) §5)
2. The `ameide` realm was created **before** the backstage client was added to the config
3. `KeycloakRealmImport` won't update an existing realm - it's ephemeral (PostSync hook, deleted after success)
4. `client-patcher` only **extracts** secrets from existing clients, doesn't create missing ones

**Current Flow (Broken)**:

```
Git (backstage client) ──X──▶ Keycloak (client missing)
                                      │
                               client-patcher
                               "Client not found - skipping"
                                      │
                                      X (no secret in Vault)
                                      │
                               ExternalSecret FAILS
                                      │
                               Backstage FAILS (missing secret)
```

**Layer Analysis**:

| Layer | Status | Notes |
|-------|--------|-------|
| Client definition in Git | ✅ Done | `platform-keycloak-realm.yaml` has backstage client |
| Sync Git → Keycloak | ❌ Missing | KeycloakRealmImport is create-only |
| Secret extraction (Keycloak → Vault) | ⚠️ Partial | Works but skips missing clients |
| Secret sync (Vault → K8s) | ✅ Ready | ExternalSecret configured |
| Backstage consuming secret | ✅ Ready | Deployment env vars configured |

**This is a General Pattern Issue**:

This affects **all** OIDC clients added after initial realm creation:
- `backstage` (current blocker)
- `k8s-dashboard` (may have same issue)
- Any future OIDC-enabled service

**Solution**: See [485-keycloak-oidc-client-reconciliation.md](485-keycloak-oidc-client-reconciliation.md) for the declarative fix.

**Vendor References**:
- [Red Hat Developer Hub Authentication](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.5/html/authentication_in_red_hat_developer_hub/)
- [Backstage OIDC Provider](https://backstage.io/docs/auth/oidc/)
- [ArgoCD Keycloak Integration](https://argo-cd.readthedocs.io/en/release-2.13/operator-manual/user-management/keycloak/)

---

## 5.3 Telemetry & Observability Integration

Per [473-ameide-technology.md](473-ameide-technology.md) §7, all platform services must emit tracing, metrics, and structured logs.

### OpenTelemetry Configuration

```yaml
# In app-config.yaml (ConfigMap)
backend:
  # ... existing config ...

# OpenTelemetry auto-instrumentation
telemetry:
  enabled: true
  endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT}
  serviceName: backstage
  metricsDisabled: false
```

### Deployment Environment Variables

```yaml
# In deployment.yaml
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://otel-collector.ameide-{{ .Values.environment }}:4318"
  - name: OTEL_SERVICE_NAME
    value: "backstage"
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "deployment.environment={{ .Values.environment }},service.namespace=ameide-{{ .Values.environment }}"
```

### Prometheus Metrics

Backstage exposes metrics at `/metrics`. Create ServiceMonitor:

```yaml
# templates/servicemonitor.yaml
{{- if .Values.metrics.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "backstage.fullname" . }}
  labels:
    {{- include "backstage.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "backstage.selectorLabels" . | nindent 6 }}
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
{{- end }}
```

### Structured Logging

Backstage should emit JSON logs with correlation IDs:

```yaml
# In app-config.yaml
backend:
  logger:
    format: json
    level: info
  # Include trace context
  plugins:
    - package: '@backstage/plugin-logging'
```

Log fields required per platform standard:
- `trace_id`, `span_id` (from OTel)
- `tenant_id` (from request context)
- `primitive_type: platform`
- `primitive_name: backstage`

---

## 5.4 Helm Testing Requirements

Per [458-helm-chart-ci-testing.md](458-helm-chart-ci-testing.md), all custom charts must pass CI validation.

### Required Test Coverage

#### 1. Helm Lint (Automatic)

The workflow at `.github/workflows/helm-test.yaml` automatically lints `sources/charts/platform/backstage/`.

#### 2. Unit Tests (Required)

Create `tests/` directory with helm-unittest specs:

```
sources/charts/platform/backstage/
├── Chart.yaml
├── values.yaml
├── templates/
│   └── ...
└── tests/
    ├── deployment_test.yaml
    ├── configmap_test.yaml
    ├── externalsecrets_test.yaml
    └── httproute_test.yaml
```

**Example unit test** (`tests/deployment_test.yaml`):

```yaml
suite: deployment tests
templates:
  - templates/deployment.yaml
tests:
  - it: should render deployment
    asserts:
      - isKind:
          of: Deployment
      - equal:
          path: metadata.name
          value: RELEASE-NAME-backstage

  - it: should use correct image
    set:
      image:
        repository: ghcr.io/backstage/backstage
        tag: "1.35.0"
    asserts:
      - equal:
          path: spec.template.spec.containers[0].image
          value: "ghcr.io/backstage/backstage:1.35.0"

  - it: should include OTEL environment variables when telemetry enabled
    set:
      telemetry:
        enabled: true
        endpoint: "http://otel-collector:4318"
    asserts:
      - contains:
          path: spec.template.spec.containers[0].env
          content:
            name: OTEL_EXPORTER_OTLP_ENDPOINT
            value: "http://otel-collector:4318"
```

#### 3. Template Validation (Automatic)

The workflow templates each chart with real values and validates against Kubernetes schemas via kubeconform.

### CI Integration

Add Backstage to the workflow's chart discovery:

```yaml
# Already covered by existing pattern - platform charts at sources/charts/platform/
# are automatically discovered and tested
```

---

## 6. Dependencies

| Dependency | Phase | Status |
|------------|-------|--------|
| CNPG Operator | 020 | ✅ Deployed |
| ExternalSecrets Operator | 020 | ✅ Deployed |
| Gateway API | 340 | ✅ Deployed |
| Keycloak | 350 | ✅ Deployed |
| PostgreSQL Clusters | 250 | ✅ Deployed |
| Vault | - | ✅ External |

---

## 6.1 GitOps Alignment (Deployment Architecture)

Backstage follows the established Ameide GitOps patterns documented in [465-applicationset-architecture.md](465-applicationset-architecture.md).

### Dual ApplicationSet Model

Backstage is an **environment-scoped workload** (not a cluster-scoped operator), deployed via the `ameide` ApplicationSet:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ArgoCD ApplicationSets                          │
├─────────────────────────────────────────────┬───────────────────────────┤
│              ameide.yaml                    │      cluster.yaml         │
│         (Environment-Scoped)                │    (Cluster-Scoped)       │
├─────────────────────────────────────────────┼───────────────────────────┤
│  ✅ Backstage belongs here                  │  ❌ NOT here              │
│  3 apps: dev-*, staging-*, production-*     │  1 app: cluster-*         │
│  Phase 356 (sub-phase *55)                  │  Phases 010-030 only      │
└─────────────────────────────────────────────┴───────────────────────────┘
```

### File Location Conventions

Per [464-chart-folder-alignment.md](464-chart-folder-alignment.md), Backstage files follow the `platform` domain pattern:

| What | Convention | Backstage File |
|------|------------|----------------|
| Component | `components/{domain}/{subdomain}/{name}/component.yaml` | `platform/developer/backstage/component.yaml` |
| Chart | `charts/{domain}/{name}/` | `charts/platform/backstage/` |
| Shared values | `values/_shared/{domain}/{name}.yaml` | `values/_shared/platform/platform-backstage.yaml` |
| Env override | `values/{env}/{domain}/{name}.yaml` | `values/dev/platform/platform-backstage.yaml` |

### Rollout Phase Placement

Per [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md), Backstage sits at phase **356** in the platform band (300-399).

**Sub-phase convention** (from 447 "Sub-phase Convention"):
| Sub-phase | Purpose |
|-----------|---------|
| `*50` | Runtimes/workloads (e.g., Keycloak at 350) |
| `*55` | Post-runtime bootstraps (e.g., keycloak-realm at 355) |
| `*56+` | Services dependent on bootstraps (Backstage at 356) |

**Dependency chain**:

```
Phase 020  │ Operators (CNPG, ExternalSecrets)
Phase 130  │ ExternalSecrets SecretStores
Phase 250  │ PostgreSQL Clusters (creates backstage DB)
Phase 340  │ Gateway API (HTTPRoutes depend on this)
Phase 350  │ Keycloak (creates `backstage` client)
Phase 355  │ client-patcher (extracts secret to Vault) + keycloak-realm
Phase 356  │ ◀── Backstage (sub-phase *55: post-runtime bootstrap)
Phase 650  │ Apps (downstream services that consume Backstage)
```

### Values Resolution Order

When ArgoCD deploys `dev-platform-backstage`:

```yaml
valueFiles:
  - $values/sources/values/env/dev/globals.yaml                    # 1. Environment globals
  - $values/sources/values/_shared/platform/platform-backstage.yaml  # 2. Shared values
  - $values/sources/values/env/dev/platform/platform-backstage.yaml      # 3. Environment override
```

### Secrets Pattern

Backstage uses the standard ExternalSecrets + Vault pattern per [426-keycloak-config-map.md](426-keycloak-config-map.md):

```
Vault (external)
    └── kv/ameide/dev/backstage-oidc-client-secret
                 ▼
ExternalSecrets SecretStore (phase 130)
                 ▼
ExternalSecret → K8s Secret (phase 356, via Backstage chart)
                 ▼
Backstage deployment mounts secret as env var
```

### Application Generation

The `ameide` ApplicationSet matrix generates 3 Backstage Applications:

```
List elements × Git files = Applications

dev         × platform/developer/backstage = dev-platform-backstage
staging     × platform/developer/backstage = staging-platform-backstage
production  × platform/developer/backstage = production-platform-backstage
```

---

## 7. Rollout Plan

### 7.1 Development Environment

1. ✅ Add `backstage` database to CNPG cluster values
2. ✅ Deploy postgres-clusters update
3. ✅ Create Backstage Helm chart
4. ✅ Add Keycloak OIDC client to realm config
5. ✅ Configure client-patcher to extract secret to Vault
6. ✅ Create component definition
7. ✅ Create dev values file
8. ✅ Deploy to dev via ArgoCD
9. ⏳ Verify authentication flow (after keycloak-realm sync)
10. ✅ Verify database connectivity

### 7.2 Staging & Production

1. ✅ Create staging/production values files
2. ✅ Update hostnames
3. ✅ Deploy via ArgoCD
4. ⏳ Smoke test each environment

---

## 7.3 Implementation Progress

**Implementation Date:** 2025-12-07

### MVP Scope

The initial deployment is an MVP focused on:
- ✅ Database connectivity (CNPG-managed)
- ✅ HTTPRoute exposure via Gateway API
- ✅ ArgoCD deployment across all environments
- ✅ Authentication (OIDC via Keycloak - implemented 2025-12-07)
- ⏳ Telemetry integration (planned)
- ⏳ Helm unit tests (planned)

### Phase 1: Database Setup ✅

Added backstage role and database to CNPG cluster:

```yaml
# sources/values/_shared/data/platform-postgres-clusters.yaml
managed:
  roles:
    - name: backstage
      ensure: present
      login: true
      createdb: true  # Required for backstage_plugin_* databases
      passwordSecret:
        name: backstage-db-credentials

credentials:
  appUsers:
    - name: backstage
      database: backstage
      username: backstage
      secretName: backstage-db-credentials
      template: backstage

databases:
  - name: backstage
    owner: backstage
    schemas:
      - name: public
        owner: backstage
```

**Key Discovery:** Backstage requires `CREATEDB` privilege to create plugin databases (e.g., `backstage_plugin_app`, `backstage_plugin_auth`, `backstage_plugin_catalog`).

### Phase 2: Helm Chart ✅

Created custom Backstage chart at `sources/charts/platform/backstage/`:

| File | Purpose |
|------|---------|
| `Chart.yaml` | Chart metadata (appVersion: 1.35.0) |
| `values.yaml` | Default values |
| `templates/_helpers.tpl` | Template helpers |
| `templates/deployment.yaml` | Backstage container with CNPG envFrom |
| `templates/service.yaml` | ClusterIP service on port 7007 |
| `templates/httproute.yaml` | Gateway API routing |
| `templates/configmap.yaml` | app-config.yaml with techdocs section |
| `templates/serviceaccount.yaml` | ServiceAccount |

**Key Decision:** Uses CNPG-managed credentials directly (via `envFrom` secretRef) instead of ExternalSecrets. This aligns with [412-cnpg-owned-postgres-greds.md](412-cnpg-owned-postgres-greds.md) pattern.

Added `backstage` template to CNPG app-secrets:

```yaml
# sources/charts/foundation/operators-config/postgres_clusters/templates/app-secrets.yaml
{{- else if eq $template "backstage" }}
  POSTGRES_HOST: {{ $host | quote }}
  POSTGRES_PORT: {{ printf "%d" (int $port) | quote }}
  POSTGRES_USER: {{ $username | quote }}
  POSTGRES_PASSWORD: {{ $password | quote }}
  POSTGRES_URL: {{ printf "postgresql://%s:%s@%s:%d/%s" ... | quote }}
{{- end }}
```

### Phase 3: Component Definition ✅

Created ArgoCD component at `environments/_shared/components/platform/developer/backstage/component.yaml`:

```yaml
name: platform-backstage
project: ameide
domain: platform
dependencyPhase: "platform"
componentType: "workload"
rolloutPhase: "356"
autoSync: true
```

### Phase 4: Values Files ✅

Created all environment values files:

| File | Hostname |
|------|----------|
| `sources/values/_shared/platform/platform-backstage.yaml` | (shared config) |
| `sources/values/env/dev/platform/platform-backstage.yaml` | `backstage.dev.ameide.io` |
| `sources/values/env/staging/platform/platform-backstage.yaml` | `backstage.staging.ameide.io` |
| `sources/values/env/production/platform/platform-backstage.yaml` | `backstage.ameide.io` |

### Deployment Status

| Environment | ArgoCD App | Status |
|-------------|------------|--------|
| dev | dev-platform-backstage | ✅ Synced, Healthy |
| staging | staging-platform-backstage | ✅ Synced, Healthy |
| production | production-platform-backstage | ⏳ Synced, Progressing (namespace pending) |

### Issues Resolved During Implementation

1. **Permission denied to create database** - Added `createdb: true` to backstage role in CNPG config
2. **YAML parse error in app-secrets.yaml** - Removed inline Helm comments that caused parsing issues
3. **Missing techdocs config** - Added required `techdocs` section to ConfigMap app-config.yaml

---

## 8. Files Summary

### New Files Created

```
# Component registration ✅
environments/_shared/components/platform/developer/backstage/component.yaml

# Helm chart ✅
sources/charts/platform/backstage/Chart.yaml
sources/charts/platform/backstage/values.yaml
sources/charts/platform/backstage/templates/_helpers.tpl
sources/charts/platform/backstage/templates/deployment.yaml
sources/charts/platform/backstage/templates/service.yaml
sources/charts/platform/backstage/templates/httproute.yaml
sources/charts/platform/backstage/templates/configmap.yaml
sources/charts/platform/backstage/templates/serviceaccount.yaml

# Values files ✅
sources/values/_shared/platform/platform-backstage.yaml
sources/values/env/dev/platform/platform-backstage.yaml
sources/values/env/staging/platform/platform-backstage.yaml
sources/values/env/production/platform/platform-backstage.yaml
```

### Planned Files (Future)

```
# Telemetry (§5.3)
sources/charts/platform/backstage/templates/servicemonitor.yaml

# Helm unit tests (§5.4)
sources/charts/platform/backstage/tests/deployment_test.yaml
sources/charts/platform/backstage/tests/configmap_test.yaml
sources/charts/platform/backstage/tests/httproute_test.yaml
sources/charts/platform/backstage/tests/servicemonitor_test.yaml

# Backstage catalog (future)
sources/backstage/catalog/all.yaml
sources/backstage/catalog/domains/
sources/backstage/catalog/systems/
```

### Modified Files

```
# Database provisioning ✅
sources/values/_shared/data/platform-postgres-clusters.yaml

# CNPG app-secrets template ✅
sources/charts/foundation/operators-config/postgres_clusters/templates/app-secrets.yaml

# Keycloak realm config ✅ (OIDC client added 2025-12-07)
sources/values/_shared/platform/platform-keycloak-realm.yaml    # backstage client definition

# client-patcher configuration ✅ (2025-12-07)
sources/values/env/dev/platform/platform-keycloak-realm.yaml        # secretExtraction for backstage
sources/values/env/staging/platform/platform-keycloak-realm.yaml
sources/values/env/production/platform/platform-keycloak-realm.yaml

# ExternalSecret template ✅ (2025-12-07)
sources/charts/foundation/operators-config/keycloak_realm/templates/externalsecret-oidc-clients.yaml
```

---

## 9. Success Criteria

### Deployment (MVP)
- [ ] Backstage accessible at `backstage.dev.ameide.io`
- [ ] Health endpoint returns 200 at `/healthz`
- [x] ArgoCD shows `Synced` and `Healthy` (dev, staging)

### Authentication (§5.2) - Implemented 2025-12-07
- [x] `backstage` client added to Keycloak `ameide` realm config
- [x] client-patcher configured to extract secret to Vault (all environments)
- [x] ExternalSecret template updated to include backstage-client-secret
- [x] Backstage configmap includes auth section with OIDC provider
- [x] Deployment mounts OIDC secret as AUTH_OIDC_CLIENT_SECRET
- [ ] Keycloak SSO login working (pending keycloak-realm sync)

### Database
- [x] `backstage` database exists in CNPG cluster
- [x] `backstage` role has CREATEDB privilege for plugin databases
- [x] CNPG-managed credentials available via Secret
- [x] Backstage connects without errors in logs
- [x] Migrations complete successfully on startup

### Telemetry (§5.3) - Future
- [ ] Traces visible in Tempo/Grafana
- [ ] Metrics scraped by Prometheus (ServiceMonitor)
- [ ] Logs in JSON format with trace_id correlation

### CI (§5.4) - Future
- [ ] `helm lint` passes for Backstage chart
- [ ] `helm unittest` passes all tests
- [ ] Template renders for all environments (dev/staging/prod)

---

## 10. Future Backlog Items

### 10.1 Software Templates (Design-Time Integration)

> **Template philosophy**: For why Backstage templates should be "thin" (structure not business logic) and how this aligns with CLI scaffolding, see [484-ameide-cli.md §16](484-ameide-cli.md).

Per [472-ameide-information-application.md](472-ameide-information-application.md) §4, Backstage templates scaffold the primitive lifecycle:

| Template | Artifact Type | Output |
|----------|---------------|--------|
| `Domain primitive` | Runtime | Proto skeleton, Go/TS service, Domain CRD manifest, ArgoCD component |
| `Process primitive` | Runtime | Temporal workflow, ProcessDefinition loader, Process CRD manifest |
| `AgentController` | Runtime | AgentDefinition executor, tool registry, IAC manifest |
| `ProcessDefinition` | Design-time (UAF) | BPMN-compliant artifact from React Flow modeller |
| `AgentDefinition` | Design-time (UAF) | Declarative agent spec (tools, scope, risk tier, policies) |

**Note**: ProcessDefinitions and AgentDefinitions are **design-time artifacts** stored in Transformation Domain. Backstage templates can scaffold the runtime primitives that **execute** these definitions, but the definitions themselves come from the custom React Flow modeller (and other UAF UIs).

#### Tenant Extension Templates

Per [478-ameide-extensions.md](478-ameide-extensions.md) §4-5, templates include **namespace calculation logic** that determines target deployment based on:

* `tenant_id` - identifies the tenant
* `sku` - Shared / Namespace / Private
* `code_origin` - platform vs tenant

**Namespace calculation**:

```
if origin == "platform":
  if sku == "Shared": return "ameide-{env}"
  else: return "tenant-{id}-{env}-base"
else:  # tenant/agent origin
  return "tenant-{id}-{env}-cust"
```

**Security invariant**: Custom code never targets `ameide-*` namespaces. See [478-ameide-extensions.md](478-ameide-extensions.md) §7 for enforcement details.

### 10.2 Catalog Population

Populate Software Catalog with existing Ameide services per the entity mapping in [472](472-ameide-information-application.md) §4.1:

| Ameide Concept | Backstage Kind |
|----------------|----------------|
| Domain primitive | `Component` (+ custom `Domain`) |
| Process primitive | `Component` (+ custom `Process`) |
| Agent primitive | `Component` (+ custom `Agent`) |
| Proto service | `API` (grpc type) |
| ProcessDefinition | `Resource` (custom kind, links to UAF) |
| AgentDefinition | `Resource` (custom kind, links to UAF) |

### 10.3 Backlog Summary

| Item | Description |
|------|-------------|
| 479-backstage-templates | Domain/Process/Agent primitive scaffold templates |
| 480-backstage-catalog | Populate catalog with existing Ameide services and APIs |
| 481-backstage-techdocs | TechDocs publishing pipeline (from backlog markdown) |
| 482-backstage-argocd | ArgoCD plugin for deployment visibility |
| 483-backstage-uaf-integration | Link ProcessDefinition/AgentDefinition catalog entries to UAF |

**Note**: 478-ameide-extensions.md now documents the tenant extension model that will inform template development.

---

## 11. References

### External Documentation
- [Backstage Documentation](https://backstage.io/docs)
- [Backstage Helm Charts](https://github.com/backstage/charts)
- [Backstage OIDC Authentication](https://backstage.io/docs/auth/oidc/)
- [OpenTelemetry Node.js](https://opentelemetry.io/docs/languages/js/)

### Vision & Architecture Suite
| Document | Section | Relationship |
|----------|---------|--------------|
| [470-ameide-vision.md](470-ameide-vision.md) | §4.3, §5.3 | Backstage as "factory", template runs |
| [471-ameide-business-architecture.md](471-ameide-business-architecture.md) | §4 | Platform factory business view |
| [472-ameide-information-application.md](472-ameide-information-application.md) | §4 | Catalog modeling, entity mappings |
| [473-ameide-technology.md](473-ameide-technology.md) | §2.3, §4 | Technology blueprint, templates |
| [475-ameide-domains.md](475-ameide-domains.md) | §2, §6 | Internal factory principle |
| [476-ameide-security-trust.md](476-ameide-security-trust.md) | §9 | Security baseline |
| [478-ameide-extensions.md](478-ameide-extensions.md) | §4-6 | Tenant extension scaffolding, namespace strategy |

### Deployment Architecture Suite
| Document | Section | Relationship |
|----------|---------|--------------|
| [465-applicationset-architecture.md](465-applicationset-architecture.md) | Full | Dual ApplicationSet model |
| [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) | Sub-phase Convention, Platform band | Rollout phase 356 (sub-phase `*56`) |
| [464-chart-folder-alignment.md](464-chart-folder-alignment.md) | §1, GAPs | Chart conventions |
| [426-keycloak-config-map.md](426-keycloak-config-map.md) | §3.2 | client-patcher pattern |
| [458-helm-chart-ci-testing.md](458-helm-chart-ci-testing.md) | Full | Helm unit tests, CI workflow |
| [462-secrets-origin-classification.md](462-secrets-origin-classification.md) | §4 | Secret authority model |

### Operational Patterns
| Document | Topic | Relationship |
|----------|-------|--------------|
| [450-argocd-service-issues-inventory.md](450-argocd-service-issues-inventory.md) | OIDC secrets | client-patcher resolution |
| [412-cnpg-owned-postgres-greds.md](412-cnpg-owned-postgres-greds.md) | DB credentials | CNPG credential pattern |
| [436-envoy-gateway-observability.md](436-envoy-gateway-observability.md) | Telemetry | ServiceMonitor patterns |

### CLI Tooling
| Document | Section | Relationship |
|----------|---------|--------------|
| [484-ameide-cli.md](484-ameide-cli.md) | §3, §13 | `ameide primitive scaffold` generates Backstage-aligned templates; CLI and Backstage share same skeleton sources |
