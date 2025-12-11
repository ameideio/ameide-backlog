# 459 – HTTPRoute Ownership Alignment

**Created**: 2025-12-05
**Updated**: 2025-12-09

> **Related documents:**
> - [441-networking.md](441-networking.md) – Networking architecture
> - [446-namespace-isolation.md](446-namespace-isolation.md) – Per-environment namespace isolation

## Overview

Align HTTPRoute ownership to follow the Gateway API recommended pattern: **apps own their routes, gateway owns infrastructure**.

Currently, HTTPRoutes are scattered across:
1. Gateway chart templates (`httproute-*.yaml`)
2. Gateway values (`extraHttpRoutes`)
3. App charts (`templates/httproute.yaml`)

**Target state**: Each app that needs external access owns its HTTPRoute in its own chart.

## Standard Pattern

```
sources/charts/apps/{app-name}/
├── templates/
│   ├── _helpers.tpl      # Standard Helm helpers
│   └── httproute.yaml    # App owns its route
└── values.yaml           # httproute.enabled, domain, etc.

sources/values/env/{env}/apps/{app-name}.yaml  # Per-env domain config
environments/_shared/components/apps/{category}/{app-name}/component.yaml
```

## Completed

- [x] **RT-1**: Keycloak HTTPRoute → `sources/charts/apps/keycloak/` → **2025-12-05**
- [x] **RT-9**: Plausible HTTPRoute → `sources/charts/apps/plausible/` → **2025-12-07** (corrected from platform-layers)
- [x] **RT-12**: Langfuse HTTPRoute → `sources/charts/apps/langfuse-route/` → **2025-12-06**
- [x] **RT-13**: www-ameide extraHttpRoute removed (verified duplicate, enabled chart route) → **2025-12-06**
- [x] **RT-14**: www-ameide-platform extraHttpRoute removed (verified duplicate, enabled chart route) → **2025-12-06**
- [x] **RT-16**: Keycloak paths restricted per vendor security docs → **2025-12-06**
- [x] **RT-10**: pgAdmin HTTPRoute → `sources/charts/platform/pgadmin/templates/httproute.yaml` (dev/staging/prod) → **2025-12-09**
- [x] **RT-2**: Grafana HTTPRoute → `sources/charts/third_party/grafana/grafana/10.3.0/templates/httproute.yaml` (env overrides in `sources/values/*/observability/platform-grafana.yaml`) → **2025-12-09**
- [x] **RT-3**: Prometheus HTTPRoute → `sources/charts/third_party/prometheus-community/kube-prometheus-stack/80.0.0/templates/httproute-prometheus.yaml` → **2025-12-09**
- [x] **RT-4**: Alertmanager HTTPRoute → `sources/charts/third_party/prometheus-community/kube-prometheus-stack/80.0.0/templates/httproute-alertmanager.yaml` → **2025-12-09**
- [x] **RT-11**: Temporal Web HTTPRoute → `sources/charts/third_party/temporal/temporal/0.70.0/templates/httproute-web.yaml` → **2025-12-09**
- [x] **RT-15**: Graph internal HTTPRoute → `sources/charts/apps/graph/templates/httproute-internal.yaml` → **2025-12-09**

## Routes in Gateway Chart Templates (to migrate)

| ID | Route File | Service | Target Chart | Priority |
|----|------------|---------|--------------|----------|
| RT-5 | `httproute-loki-https.yaml` | loki | `platform-layers/loki` | Low |
| RT-6 | `httproute-tempo-https.yaml` | tempo | `platform-layers/tempo` | Low |
| RT-7 | `httproute-metrics-https.yaml` | otel-collector | New `apps/otel-collector` | Low |
| RT-8 | `httproute-telemetry-https.yaml` | otel-collector | New `apps/otel-collector` | Low |
| RT-9 | `httproute-plausible-https.yaml` | plausible | ✅ `apps/plausible` | ~~High~~ Done |

## Routes in extraHttpRoutes (to migrate)

| ID | Route | Service | Target Chart | Priority |
|----|-------|---------|--------------|----------|
| RT-10 | `pgadmin` | pgadmin | New `apps/pgadmin` | Low (dev only) |
| RT-11 | `temporal-web` | data-temporal-web | `third_party/temporal/temporal/templates/httproute-web.yaml` | ~~Medium~~ Done |
| RT-12 | `langfuse-web` | platform-langfuse-web | ✅ `apps/langfuse-route` | ~~High~~ Done |
| RT-13 | `www-ameide` | www-ameide | ✅ Enabled in `apps/www-ameide` | ~~High~~ Done |
| RT-14 | `www-ameide-platform` | www-ameide-platform | ✅ Enabled in `apps/www-ameide-platform` | ~~High~~ Done |
| RT-15 | `graph-connect-internal` | graph | `apps/graph/templates/httproute-internal.yaml` | ~~Medium~~ Done |

## Keep in Gateway (legitimate gateway concerns)

| Route | Reason |
|-------|--------|
| `httproute-apex-https.yaml` | Apex domain redirect |
| `httproutefilter-cors.yaml` | CORS policy |
| `httproutes-additional.yaml` | Dynamic/catch-all |
| `httproute-repository.yaml` | Registry/GHCR mirror |

## Migration Steps Per Route

1. **Check if chart exists** for the service
2. **Add HTTPRoute template** following keycloak pattern:
   - `templates/httproute.yaml` with `{{- if .Values.httproute.enabled }}`
   - `templates/_helpers.tpl` if missing
   - Update `values.yaml` with `httproute` config
3. **Add per-env values** in `sources/values/env/{env}/apps/{name}.yaml`
4. **Add component.yaml** if new chart
5. **Remove from gateway** (template file or extraHttpRoutes entry)
6. **Test** helm template renders correctly
7. **Commit and sync**

## Priorities

### ~~High Priority~~ ✅ All Completed (2025-12-06)
- ~~RT-9: Plausible~~ → Done
- ~~RT-12: Langfuse~~ → Done
- ~~RT-13/14: Verify www-ameide duplicates~~ → Done
- ~~RT-16: Keycloak path security~~ → Done

### Low Priority (dev-only or complex)
- RT-5/6: Loki, Tempo
- RT-7/8: OTEL collector

## Backlog

### Low Priority (dev-only or complex)
- [ ] **RT-5**: Move loki route to its chart
- [ ] **RT-6**: Move tempo route to its chart
- [ ] **RT-7/8**: Create otel-collector chart or add routes
