# 448 – Per-Environment Service Verification

**Status**: P0 Complete - P1 Pending
**Created**: 2025-12-04
**Updated**: 2025-12-04
**Related**: [445-argocd-namespace-isolation.md](445-argocd-namespace-isolation.md), [446-namespace-isolation.md](446-namespace-isolation.md), [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md), [449-per-environment-infrastructure.md](449-per-environment-infrastructure.md)

---

## Problem Statement

Services running in environment-specific namespaces (`ameide-dev`, `ameide-staging`, `ameide-prod`) require correct namespace-qualified FQDNs for cross-service communication. Hardcoded references to a non-existent `ameide` namespace cause connection failures.

---

## Completed Fixes

### Database Connection Strings

Added `postgresHost` override to per-environment vault-secrets-platform.yaml files:

| Environment | File | Value |
|-------------|------|-------|
| staging | `sources/values/staging/apps/platform/vault-secrets-platform.yaml` | `postgres-ameide-rw.ameide-staging.svc.cluster.local` |
| production | `sources/values/production/apps/platform/vault-secrets-platform.yaml` | `postgres-ameide-rw.ameide-prod.svc.cluster.local` |

### CoreDNS Configuration

Created per-environment CoreDNS configs with correct Envoy service references:

| Environment | File | Envoy Service |
|-------------|------|---------------|
| dev | `sources/values/dev/foundation/foundation-coredns-config.yaml` | `envoy.ameide-dev.svc.cluster.local` |
| staging | `sources/values/staging/foundation/foundation-coredns-config.yaml` | `envoy.ameide-staging.svc.cluster.local` |
| production | `sources/values/production/foundation/foundation-coredns-config.yaml` | `envoy.ameide-prod.svc.cluster.local` |

Updated platform coredns-config.yaml files:

| Environment | File | Value |
|-------------|------|-------|
| staging | `sources/values/staging/apps/platform/coredns-config.yaml` | `envoy.ameide-staging.svc.cluster.local` |
| production | `sources/values/production/apps/platform/coredns-config.yaml` | `envoy.ameide-prod.svc.cluster.local` |

Shared config (`_shared/foundation/foundation-coredns-config.yaml`) cleared - per-environment files now define rules.

### Gateway OTEL Endpoints

Already fixed - per-environment gateway files use correct FQDNs:
- dev: `otel-collector.ameide-dev.svc.cluster.local`
- staging: `otel-collector.ameide-staging.svc.cluster.local`
- production: `otel-collector.ameide-prod.svc.cluster.local`

---

## Remaining Work

### P1 - Smoke Tests

60 hardcoded `--namespace ameide` references across 7 files:

| File | Count |
|------|-------|
| `_shared/data/data-data-plane-smoke.yaml` | 17 |
| `_shared/platform/platform-observability-smoke.yaml` | 14 |
| `_shared/platform/platform-auth-smoke.yaml` | 10 |
| `_shared/data/data-data-plane-ext-smoke.yaml` | 9 |
| `_shared/platform/platform-control-plane-smoke.yaml` | 5 |
| `_shared/foundation/foundation-operators-smoke.yaml` | 3 |
| `_shared/platform/platform-secrets-smoke.yaml` | 2 |

**Fix**: Replace hardcoded namespace with `{{ .Release.Namespace }}` template variable.

### P2 - CI Validation

Add pre-commit or CI check to detect `.ameide.svc` patterns without environment qualification.

---

## Architecture Reference

Per [445](445-argocd-namespace-isolation.md) and [447](447-waves-v3-cluster-scoped-operators.md):

### Environment Namespaces

| Environment | Namespace | Gateway Service |
|-------------|-----------|-----------------|
| dev | `ameide-dev` | `envoy.ameide-dev.svc.cluster.local` |
| staging | `ameide-staging` | `envoy.ameide-staging.svc.cluster.local` |
| production | `ameide-prod` | `envoy.ameide-prod.svc.cluster.local` |

### Cluster-Scoped Services

| Service | Namespace | Notes |
|---------|-----------|-------|
| argocd | `argocd` | GitOps controller, cluster gateway |
| vault | `vault` | Shared secret store |
| cert-manager | `cert-manager` | Certificate management |
| external-secrets | `external-secrets` | Secret sync |
| cnpg-operator | `cnpg-system` | PostgreSQL operator |
| strimzi-operator | `strimzi-system` | Kafka operator |
| redis-operator | `redis-system` | Redis operator |
| keycloak-operator | `keycloak-system` | Keycloak operator |
| clickhouse-operator | `clickhouse-system` | ClickHouse operator |

---

## Validation

```bash
# Should return empty after P0 fixes
grep -rn "\.ameide\.svc" sources/values/ | grep -v "ameide-dev\|ameide-staging\|ameide-prod"

# P1: Will still show smoke test references until fixed
grep -rn "\-\-namespace ameide[^-]" sources/values/
```
