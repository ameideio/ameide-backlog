# 458 – Helm Chart CI Testing

> **Related documents:**
> - [086-k8s-deploy-preflight-checks.md](old/086-k8s-deploy-preflight-checks.md) – Deployment preflight checks (archived)
> - [111-helm-tilt-north-star.md](old/111-helm-tilt-north-star.md) – Helm/Tilt strategy (archived)
> - [447-third-party-chart-tolerations.md](447-third-party-chart-tolerations.md) – Third-party chart configuration issues
> - [490-vendor-charts-alignment.md](490-vendor-charts-alignment.md) – Lockfile + vendoring enforcement that feeds Helm validation inputs

## Summary

Implement GitHub Actions workflow to validate Helm charts before merge. Currently, chart issues (wrong value paths, missing tolerations, template errors) are only discovered after ArgoCD sync fails in the cluster.

**Status**: ✅ Implemented

---

## Problem Statement

### Current State

- **137 Helm charts** in `sources/charts/` (apps, foundation, platform-layers, third_party)
- **No pre-merge validation** - errors discovered only at runtime
- **Complex value composition** - charts use layered values from `_shared/`, `dev/`, `staging/`, `production/`
- **Third-party charts** have non-obvious value paths (documented in [447](447-third-party-chart-tolerations.md))

### Recent Issues Caused by Missing CI

| Issue | Root Cause | Detection |
|-------|------------|-----------|
| Langfuse ImagePullBackOff | Wrong YAML path (`langfuse.image` vs `langfuse.web.image`) | Runtime pod failure |
| Prometheus node-exporter conflicts | Template worked but config was wrong | Runtime port conflicts |
| Missing tolerations | Values existed but at wrong path | Runtime Pending pods |

### Why This Matters

1. **Fast feedback** - Catch errors in PR, not after merge
2. **Prevent outages** - Bad configs don't reach ArgoCD
3. **Developer confidence** - Know your changes work before merge
4. **Documentation** - CI failures explain correct value paths

---

## Solution Design

### Workflow: `helm-test.yaml`

```yaml
name: Helm Chart Validation

on:
  pull_request:
    paths:
      - 'sources/charts/**'
      - 'sources/values/**'
  push:
    branches: [main]
    paths:
      - 'sources/charts/**'
      - 'sources/values/**'

jobs:
  lint:
    # helm lint on all charts

  template:
    # helm template with real values

  kubeconform:
    # Validate generated manifests against K8s schemas
```

### Test Levels

#### Level 1: Helm Lint (Fast, All Charts)

```bash
# Lint all custom charts
find sources/charts/{apps,foundation,platform-layers,data,cluster} \
  -name Chart.yaml -execdir helm lint . \;
```

**Catches**: YAML syntax, missing required values, Chart.yaml issues

#### Level 2: Helm Unit Tests (helm-unittest)

```bash
# Run unit tests for charts with tests/ directory
helm unittest sources/charts/apps/inference-gateway
```

**Catches**: Template logic errors, conditional rendering issues, incorrect values

**Charts with unit tests**: 14 charts including `agents`, `agents-runtime`, `gateway`, `inference`, `inference-gateway`, `platform`, `workflows`, `workflows-runtime`, `helm-test-jobs`, `postgres_clusters`, `vault-bootstrap`, `vault-webhook-certs`, `db-migrations`, `pgadmin`

#### Level 3: Helm Template (Per Environment)

```bash
# Template with composed values (mimics ArgoCD)
helm template www-ameide sources/charts/apps/www-ameide \
  -f sources/values/_shared/apps/www-ameide.yaml \
  -f sources/values/env/dev/apps/www-ameide.yaml \
  -f sources/values/env/dev/globals.yaml
```

**Catches**: Template rendering errors, value path mismatches, missing required values

#### Level 4: Kubeconform (Schema Validation)

```bash
# Validate against K8s API schemas
helm template ... | kubeconform -strict -kubernetes-version 1.30.0
```

**Catches**: Invalid K8s resource specs, deprecated API versions, missing required fields

---

## Implementation

### Phase 1: Core Validation

1. **helm-test.yaml workflow**
   - Trigger on PR + push to main
   - Path filters for `sources/charts/**` and `sources/values/**`
   - Matrix over environments (dev, staging, production)

2. **Chart discovery script**
   - Find all custom charts (exclude `third_party/` for lint)
   - Map charts to their values files

3. **Template test matrix**
   - For each chart × environment combination
   - Compose values in correct order (shared → env-specific → globals)

### Phase 2: Enhanced Validation

1. **Kubeconform integration**
   - Validate against K8s 1.30 schemas
   - Custom CRD schemas for Strimzi, CNPG, etc.

2. **Policy checks (optional)**
   - Required labels present
   - Resource limits defined
   - Tolerations for tainted pools

### Phase 3: Third-Party Charts

1. **Template-only validation** (no lint - they have their own CI)
2. **Values path validation** - ensure our values match chart's schema

---

## Chart → Values Mapping

### Custom Charts

| Chart Path | Shared Values | Env Values |
|------------|---------------|------------|
| `apps/www-ameide` | `_shared/apps/www-ameide.yaml` | `{env}/apps/www-ameide.yaml` |
| `apps/inference` | `_shared/apps/inference.yaml` | `{env}/apps/inference.yaml` |
| `platform-layers/langfuse-bootstrap` | `_shared/platform/langfuse-bootstrap.yaml` | `{env}/platform/platform-langfuse-bootstrap.yaml` |
| `foundation/vault-bootstrap` | `_shared/foundation/foundation-vault-bootstrap.yaml` | - |

### Third-Party Charts (Template Only)

| Chart | Values Prefix | Notes |
|-------|---------------|-------|
| `third_party/prometheus` | `platform-prometheus` | Multiple subcomponents |
| `third_party/grafana/loki` | `platform-loki` | singleBinary mode |
| `third_party/langfuse` | `platform-langfuse` | web + worker subcharts |

---

## Success Criteria

1. **PR blocking** - Failed helm template blocks merge
2. **All envs tested** - dev, staging, production values validated
3. **Fast feedback** - < 5 minutes for full validation
4. **Clear errors** - Failure messages indicate exact issue

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| False positives | Blocked valid PRs | Allow `[skip-helm-test]` in commit message |
| Missing CRD schemas | Kubeconform failures | Download CRD schemas, or skip strict mode for CRDs |
| Slow CI | Developer frustration | Parallel jobs, caching |
| Third-party chart changes | Unexpected failures | Pin versions, separate job |

---

## Out of Scope

- **Runtime validation** - Covered by ArgoCD sync status
- **Security scanning** - Separate concern (Trivy, etc.)
- **Full deployment tests** - Requires live cluster

---

## Implementation Notes

### Workflow File: `.github/workflows/helm-test.yaml`

The workflow implements four jobs:

1. **discover-charts**: Finds all custom charts and charts with unit tests
2. **lint**: Runs `helm lint` on all custom charts
3. **unit-test**: Runs `helm unittest` on charts with `tests/*_test.yaml` files
4. **template-test**: Runs `helm template` with environment-specific values for dev/staging/production
5. **summary**: Aggregates results from all jobs

### Key Features

- **Path filters**: Only runs on changes to `sources/charts/**` or `sources/values/**`
- **Matrix testing**: Template tests run in parallel for all three environments
- **Skip non-charts**: Directories without `Chart.yaml` are skipped
- **Kubeconform validation**: Generated manifests are validated against K8s schemas
- **CRD skip list**: Custom resources (Application, Certificate, Kafka, etc.) are skipped in kubeconform

### Test Fixes Made

| Chart | Issue | Fix |
|-------|-------|-----|
| `agents` | Missing minio secret values | Use shared values file instead of inline values |
| `agents-runtime` | Missing minio secret values | Use shared values file instead of inline values |
| `vault-webhook-certs` | Template directive not working | Split into separate tests per template |

---

## References

- [Helm lint docs](https://helm.sh/docs/helm/helm_lint/)
- [Kubeconform](https://github.com/yannh/kubeconform)
- [helm-unittest](https://github.com/helm-unittest/helm-unittest)
- [chart-testing (ct)](https://github.com/helm/chart-testing)
