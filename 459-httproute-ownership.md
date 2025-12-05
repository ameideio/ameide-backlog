# 459 – HTTPRoute Ownership Alignment

**Created**: 2025-12-05
**Updated**: 2025-12-05

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

sources/values/{env}/apps/{app-name}.yaml  # Per-env domain config
environments/_shared/components/apps/{category}/{app-name}/component.yaml
```

## Completed

- [x] **RT-1**: Keycloak HTTPRoute → `sources/charts/apps/keycloak/` → **2025-12-05**

## Routes in Gateway Chart Templates (to migrate)

| ID | Route File | Service | Target Chart | Priority |
|----|------------|---------|--------------|----------|
| RT-2 | `httproute-grafana-https.yaml` | grafana | New `apps/grafana` | Medium |
| RT-3 | `httproute-prometheus-https.yaml` | prometheus | New `apps/prometheus` | Medium |
| RT-4 | `httproute-alertmanager-https.yaml` | alertmanager | New `apps/alertmanager` | Medium |
| RT-5 | `httproute-loki-https.yaml` | loki | `platform-layers/loki` | Low |
| RT-6 | `httproute-tempo-https.yaml` | tempo | `platform-layers/tempo` | Low |
| RT-7 | `httproute-metrics-https.yaml` | otel-collector | New `apps/otel-collector` | Low |
| RT-8 | `httproute-telemetry-https.yaml` | otel-collector | New `apps/otel-collector` | Low |
| RT-9 | `httproute-plausible-https.yaml` | plausible | Existing `apps/platform/plausible` | High |

## Routes in extraHttpRoutes (to migrate)

| ID | Route | Service | Target Chart | Priority |
|----|-------|---------|--------------|----------|
| RT-10 | `pgadmin` | pgadmin | New `apps/pgadmin` | Low (dev only) |
| RT-11 | `temporal-web` | data-temporal-web | `platform-layers/temporal` | Medium |
| RT-12 | `langfuse-web` | platform-langfuse-web | `platform-layers/langfuse` | High |
| RT-13 | `www-ameide` | www-ameide | **Verify duplicate** - has `apps/www-ameide` | High |
| RT-14 | `www-ameide-platform` | www-ameide-platform | **Verify duplicate** - has `apps/www-ameide-platform` | High |
| RT-15 | `graph-connect-internal` | graph | Existing `apps/graph` | Medium |

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
3. **Add per-env values** in `sources/values/{env}/apps/{name}.yaml`
4. **Add component.yaml** if new chart
5. **Remove from gateway** (template file or extraHttpRoutes entry)
6. **Test** helm template renders correctly
7. **Commit and sync**

## Priorities

### High Priority (existing charts, just add route)
- RT-9: Plausible
- RT-12: Langfuse
- RT-13/14: Verify www-ameide duplicates

### Medium Priority (may need new charts)
- RT-2/3/4: Observability (grafana, prometheus, alertmanager)
- RT-11: Temporal
- RT-15: Graph internal route

### Low Priority (dev-only or complex)
- RT-5/6: Loki, Tempo
- RT-7/8: OTEL collector
- RT-10: pgAdmin

## Backlog

- [ ] **RT-9**: Move plausible route to `apps/platform/plausible`
- [ ] **RT-12**: Move langfuse route to `platform-layers/langfuse`
- [ ] **RT-13**: Verify www-ameide extraHttpRoute is duplicate, remove if so
- [ ] **RT-14**: Verify www-ameide-platform extraHttpRoute is duplicate, remove if so
- [ ] **RT-2**: Create `apps/grafana` chart with HTTPRoute
- [ ] **RT-3**: Create `apps/prometheus` chart with HTTPRoute
- [ ] **RT-4**: Create `apps/alertmanager` chart with HTTPRoute
- [ ] **RT-11**: Add HTTPRoute to temporal chart
- [ ] **RT-15**: Add internal route to graph chart
- [ ] **RT-5**: Move loki route to its chart
- [ ] **RT-6**: Move tempo route to its chart
- [ ] **RT-7/8**: Create otel-collector chart or add routes
- [ ] **RT-10**: Create pgadmin chart (dev-only)
