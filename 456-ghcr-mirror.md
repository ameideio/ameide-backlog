# GHCR Image Mirroring for Docker Hub Rate Limiting

**Status**: Implemented
**Date**: 2025-12-05
**Related**: [345-ci-cd-unified-playbook.md](345-ci-cd-unified-playbook.md), [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md), [450-argocd-service-issues-inventory.md](450-argocd-service-issues-inventory.md)

> **Cross-reference**: [490-vendor-charts-alignment.md](490-vendor-charts-alignment.md) tracks the lockfile + vendoring workflow that complements this GHCR mirroring effort.

## Problem Statement

Docker Hub rate limiting caused widespread `ImagePullBackOff` errors across all environments:

```
Failed to pull image "docker.io/library/busybox:1.36.1": 429 Too Many Requests
Server message: toomanyrequests: You have reached your unauthenticated pull rate limit
```

**Rate limits:**
- Anonymous: 100 pulls/6 hours (per IP)
- Authenticated: 200 pulls/6 hours (per user)
- Paid: Unlimited

The AKS cluster shares a NAT IP, so all unauthenticated pulls count against the same limit.

## Solution: GHCR Mirroring

Mirror all third-party Docker Hub images to GitHub Container Registry under `ghcr.io/ameideio/mirror/`.

**Benefits:**
- No rate limiting from GHCR
- Images hosted in same region as GitHub (faster pulls)
- Full control over image availability
- Weekly automated refresh keeps images current

## Infrastructure

### GitHub Actions Workflow

**File:** `.github/workflows/mirror-images.yaml`

```yaml
on:
  workflow_dispatch:
    inputs:
      dry_run:
        description: 'Dry run (just check, no push)'
        default: 'false'
  schedule:
    - cron: '0 2 * * 0'  # Weekly Sunday 2am UTC
```

**Required Secrets:**
- `DOCKERHUB_USERNAME` - Docker Hub account
- `DOCKERHUB_TOKEN` - Docker Hub access token
- `GITHUB_TOKEN` - Auto-provided, needs `packages: write`

### Local Mirroring Script

**File:** `infra/scripts/mirror-images-to-ghcr.sh`

For manual/local mirroring. Requires:
```bash
docker login ghcr.io  # with write access
docker login docker.io  # optional, for higher rate limits
```

## Mirrored Images

| Source Image | GHCR Target |
|-------------|-------------|
| **Temporal** | |
| temporalio/server:1.29.1 | temporalio-server:1.29.1 |
| temporalio/admin-tools:1.29.1-* | temporalio-admin-tools:1.29.1-* |
| temporalio/ui:2.42.1 | temporalio-ui:2.42.1 |
| **Grafana Stack** | |
| grafana/grafana:12.1.0 | grafana:12.1.0 |
| grafana/loki:3.5.7 | loki:3.5.7 |
| grafana/alloy:v1.11.3 | alloy:v1.11.3 |
| grafana/tempo:latest | tempo:latest |
| **Envoy Gateway** | |
| envoyproxy/envoy:distroless-v1.33.0 | envoy:distroless-v1.33.0 |
| envoyproxy/gateway:v1.3.0 | envoy-gateway:v1.3.0 |
| **Redis** | |
| redis:7.2.5 | redis:7.2.5 |
| redis:7.4.1 | redis:7.4.1 |
| **ClickHouse** | |
| clickhouse/clickhouse-server:25.10.2-alpine | clickhouse-server:25.10.2-alpine |
| altinity/clickhouse-operator:0.25.5 | clickhouse-operator:0.25.5 |
| altinity/metrics-exporter:0.25.5 | clickhouse-metrics-exporter:0.25.5 |
| **Vault** | |
| hashicorp/vault:1.21.0 | vault:1.21.0 |
| hashicorp/vault-k8s:1.4.1 | vault-k8s:1.4.1 |
| **Langfuse** | |
| langfuse/langfuse:3.120.0 | langfuse:3.120.0 |
| langfuse/langfuse-worker:3.120.0 | langfuse-worker:3.120.0 |
| **Plausible** | |
| plausible/analytics:v2.0.0 | plausible-analytics:v2.0.0 |
| **Database** | |
| postgres:16 | postgres:16 |
| **Utilities** | |
| library/busybox:1.36.1 | busybox:1.36.1 |
| curlimages/curl:8.9.1 | curl:8.9.1 |
| nginxinc/nginx-unprivileged:1.29-alpine | nginx-unprivileged:1.29-alpine |
| bitnami/os-shell:latest | bitnami-os-shell:latest |
| bitnami/kubectl:latest | bitnami-kubectl:latest |
| kiwigrid/k8s-sidecar:1.30.10 | k8s-sidecar:1.30.10 |
| dpage/pgadmin4:8.7 | pgadmin4:8.7 |
| otel/opentelemetry-collector-contrib:0.91.0 | otel-collector-contrib:0.91.0 |

## imagePullSecrets Configuration

All namespaces have a `ghcr-pull` secret containing GHCR credentials. Charts must reference this secret to pull from the private GHCR mirror.

### Chart-Specific Patterns

Different Helm charts expect `imagePullSecrets` in different formats:

```yaml
# Standard pattern (most charts)
imagePullSecrets:
  - name: ghcr-pull

# Global pattern (Grafana, ArgoCD)
global:
  imagePullSecrets:
    - name: ghcr-pull

# Tempo pattern (list of strings, not objects)
tempo:
  pullSecrets:
    - ghcr-pull

# Langfuse pattern (nested under image)
langfuse:
  image:
    pullSecrets:
      - name: ghcr-pull

# Redis Failover CRD pattern (per component)
redis:
  imagePullSecrets:
    - name: ghcr-pull
sentinel:
  imagePullSecrets:
    - name: ghcr-pull

# Alloy pattern (under global.image)
global:
  image:
    pullSecrets:
      - name: ghcr-pull

# Loki pattern (root level + sidecar)
imagePullSecrets:
  - name: ghcr-pull
loki:
  image:
    registry: ghcr.io
    repository: ameideio/mirror/loki
sidecar:
  image:
    repository: ghcr.io/ameideio/mirror/k8s-sidecar
```

## Files Modified

### Values Files

| File | Changes |
|------|---------|
| `sources/values/_shared/data/data-temporal.yaml` | GHCR mirrors for server + busybox init container + imagePullSecrets |
| `sources/values/_shared/data/data-temporal.yaml` | GHCR mirror for temporal server/ui/admin-tools images (TemporalCluster) |
| `sources/values/_shared/platform/platform-loki.yaml` | `loki.image` path, sidecar image, imagePullSecrets |
| `sources/values/_shared/platform/platform-grafana.yaml` | global.imagePullSecrets, GHCR image, downloadDashboardsImage |
| `sources/values/_shared/platform/platform-tempo.yaml` | `tempo.pullSecrets` format, GHCR image |
| `sources/values/_shared/platform/platform-alloy-logs.yaml` | global.image.pullSecrets, GHCR image |
| `sources/values/_shared/platform/platform-otel-collector.yaml` | imagePullSecrets, GHCR image |
| `sources/values/_shared/data/data-redis-failover.yaml` | imagePullSecrets on redis/sentinel, GHCR image |
| `sources/values/_shared/data/data-clickhouse.yaml` | imagePullSecrets, GHCR images |
| `sources/values/_shared/foundation/foundation-vault-bootstrap.yaml` | imagePullSecrets, GHCR image |
| `sources/values/_shared/platform/platform-langfuse.yaml` | pullSecrets on web/worker, GHCR images |
| `sources/values/common/argocd.yaml` | global.imagePullSecrets for Redis |
| `sources/charts/platform-layers/pgadmin/values.yaml` | imagePullSecrets, GHCR image |
| `sources/charts/platform-layers/plausible/values.yaml` | imagePullSecrets, GHCR image |

### Template Changes

| File | Change |
|------|--------|
| `sources/charts/foundation/vault-bootstrap/templates/cronjob.yaml` | Added `{{- with .Values.imagePullSecrets }}` block |
| `sources/charts/platform-layers/pgadmin/templates/deployment.yaml` | Added `{{- with .Values.imagePullSecrets }}` block |
| `sources/charts/platform-layers/plausible/templates/deployment.yaml` | Added `{{- with .Values.imagePullSecrets }}` block |

## Maintenance

### Adding New Images

1. Add to `.github/workflows/mirror-images.yaml` matrix:
   ```yaml
   - source: vendor/image:tag
     target: image:tag
   ```

2. Add to `infra/scripts/mirror-images-to-ghcr.sh`:
   ```bash
   ["vendor/image:tag"]="image:tag"
   ```

3. Run workflow manually or wait for weekly run

4. Update values file to use GHCR path:
   ```yaml
   image:
     repository: ghcr.io/ameideio/mirror/image
     tag: "tag"
   imagePullSecrets:
     - name: ghcr-pull
   ```

### Updating Image Versions

1. Update both mirroring files with new tag
2. Run workflow
3. Update values files with new tag

### Troubleshooting

**401 Unauthorized from GHCR:**
- Verify `ghcr-pull` secret exists in namespace
- Check imagePullSecrets is configured correctly for the chart
- GHCR packages may be "internal" (private) - credentials required

**ImagePullBackOff:**
- Check if image exists in GHCR: `gh api /orgs/ameideio/packages/container/mirror%2F<image>/versions`
- Verify image tag matches exactly
- Check pod events: `kubectl describe pod <name>`

**Workflow failures:**
- Verify DOCKERHUB_USERNAME/TOKEN secrets are set
- Check for Docker Hub rate limiting during pull phase
- Review workflow logs in GitHub Actions

## Related Incidents

- **2025-12-05 Temporal Recovery**: During GHCR mirror alignment, Temporal pods were stuck due to:
  1. Busybox init container (`cgr.dev/chainguard/busybox`) missing `nc` command - fixed by switching to GHCR mirror
  2. Migration job using Docker Hub image (rate limited) - fixed by adding GHCR mirror for `temporalio-admin-tools`

  See [backlog/423-temporal-argocd-recovery.md](423-temporal-argocd-recovery.md) for full incident details.

## Related Documentation

- [423-temporal-argocd-recovery.md](423-temporal-argocd-recovery.md) - Temporal ArgoCD recovery procedures
- [420-temporal-cnpg-dev-registry-runbook.md](420-temporal-cnpg-dev-registry-runbook.md) - Temporal CNPG dev registry runbook
- [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) - Cluster operator tolerations (related to node scheduling)
