# Backlog 417: Envoy Gateway route coverage

- Goal: align all public app/frontends to Envoy Gateway (Gateway API HTTPRoute/GRPCRoute) and remove residual Ingress usage that breaks health.
- Actions:
  - Catalog current routes/hosts and their backends (gateway chart `additionalHttpRoutes` and route templates).
  - Move any remaining Ingress-based apps (e.g., Langfuse) to Gateway HTTPRoutes and disable their Helm Ingress stanzas.
  - Ensure className/parentRefs target the platform gateway and hosts/certs match env domains.
  - Add Argo health overrides if needed for Gateway resources; avoid relying on Ingress LB status.
  - Wire a control-plane smoke test that validates Envoy xDS PKI (server `envoy-gateway` issued by `envoy-gateway-ca`, which is issued by the dev root CA `ameidet-ca`) and Gateway API status (`Accepted=True`, `Programmed=True`) before treating routes as healthy.
  - Keep a single source of truth for routes (values files) and document expected hosts → services mapping.

## Implementation status

- Envoy Gateway chart is the single source of truth for shared routes; dev uses `_shared` + `local` values, staging/production use `_shared` + env-specific values.
- Plausible HTTPRoute restored (previously dropped by an empty `additionalHttpRoutes` override).
- All public app Ingress stanzas removed/disabled: Langfuse (staging/production) and prod Grafana now rely on Gateway; staging web frontends moved to HTTPRoutes.
- Keycloak is exposed via Gateway, but production renders two HTTPRoutes (auth.dev.ameide.io/auth.ameide.io → keycloak:4000 and auth.ameide.io → keycloak:8080); needs dedupe/port alignment.
- Duplicate HTTPRoutes exist for some hosts (prometheus/alertmanager/loki/tempo UI vs. base routes); consider consolidating to a single route per host.
- Local dev now renders a second HTTPS listener (`https-local`) plus dedicated HTTPRoutes for `www.local.ameide.io` and `platform.local.ameide.io`, so tilt-only releases stay isolated from the Argo baseline.
- The Envoy xDS certificate chain is now managed by `platform-cert-manager-config` (`ameidet-ca` root, `envoy-gateway-ca` intermediate, and `envoy`/`envoy-gateway` TLS secrets), and `platform-control-plane-smoke` asserts this PKI wiring and the `gateway/ameide` `Accepted=True, Programmed=True` conditions so broken xDS/TLS cannot hide behind green HTTPRoute inventories.

## Route inventory (rendered from charts)

### Dev (k3d/local)
- GRPCRoute graph → apps-graph:8081 (internal listener)
- GRPCRoute platform → apps-platform:8082 (internal)
- GRPCRoute threads → apps-threads:8107 (internal)
- GRPCRoute workflows → apps-workflows:8086 (internal)
- HTTPRoute grafana-https: grafana.dev.ameide.io → platform-grafana:80
- HTTPRoute loki-https & ameide-loki-ui: loki.dev.ameide.io → platform-loki-gateway:80
- HTTPRoute tempo-https & ameide-tempo-ui: tempo.dev.ameide.io → platform-tempo:3200
- HTTPRoute prometheus-https & ameide-prometheus-ui: prometheus.dev.ameide.io → platform-prometheus(-prometheus / -kube-p-prometheus):9090
- HTTPRoute alertmanager-https & ameide-alertmanager-ui: alertmanager.dev.ameide.io → platform-prometheus-(alertmanager / kube-p-alertmanager):9093
- HTTPRoute metrics-https: metrics.dev.ameide.io → platform-otel-collector:8888
- HTTPRoute telemetry-https: telemetry.dev.ameide.io → platform-otel-collector:4317
- HTTPRoute argocd: argocd.dev.ameide.io → argocd-server:80
- HTTPRoute pgadmin: pgadmin.dev.ameide.io → pgadmin:80
- HTTPRoute temporal-web: temporal.dev.ameide.io → data-temporal-web:4001
- HTTPRoute langfuse-web: evals.dev.ameide.io → platform-langfuse-web:3000
- HTTPRoute plausible: plausible.dev.ameide.io → plausible-plausible:8000
- HTTPRoute keycloak: auth.dev.ameide.io → keycloak:8080
- App chart HTTPRoutes: www.dev.ameide.io → apps-www-ameide:3000; platform.dev.ameide.io → apps-www-ameide-platform:3001
- Tilt-only HTTPRoutes (via `https-local` listener): www.local.ameide.io → apps-www-ameide-tilt:3000 and platform.local.ameide.io → apps-www-ameide-platform-tilt:3001 (only active during `tilt up`).

### Staging
- GRPCRoute graph/platform/workflows/inference: api.staging.ameide.io → graph:8081 / platform:8082 / workflows:8086 / inference:9000
- GRPCRoute threads: internal (no hostname) → threads:8107
- HTTPRoute grafana-https: grafana.staging.ameide.io → grafana:80
- HTTPRoute loki-https & ameide-loki-ui: loki.staging.ameide.io → loki-gateway:80
- HTTPRoute tempo-https & ameide-tempo-ui: tempo.staging.ameide.io → tempo:3200
- HTTPRoute prometheus-https: prometheus.staging.ameide.io → prometheus-oauth2-proxy:80
- HTTPRoute alertmanager-https: alertmanager.staging.ameide.io → alertmanager-oauth2-proxy:80
- HTTPRoute metrics-https: metrics.staging.ameide.io → otel-collector:8888
- HTTPRoute telemetry-https: telemetry.staging.ameide.io → otel-collector:4317
- HTTPRoute keycloak: auth.staging.ameide.io → keycloak:8080
- HTTPRoute plausible: plausible.staging.ameide.io → plausible-plausible:8000
- HTTPRoute argocd: argocd.staging.ameide.io → argocd-server:80
- HTTPRoute pgadmin: pgadmin.staging.ameide.io → pgadmin:80
- HTTPRoute temporal-web: temporal.staging.ameide.io → temporal-oauth2-proxy:80
- HTTPRoute langfuse-web: evals.staging.ameide.io → platform-langfuse-web:3000
- HTTPRoute graph-connect (external): api.staging.ameide.io → graph:8081
- HTTPRoute graph-connect-internal: listener-only (no host) → graph:8081
- App chart HTTPRoutes: www.staging.ameide.io → www-ameide:3000; platform.staging.ameide.io → www-ameide-platform:3001

### Production
- GRPCRoute graph/platform/workflows/inference: api.ameide.io → graph:8081 / platform:8082 / workflows:8086 / inference:9000
- GRPCRoute threads: internal (no hostname) → threads:8107
- HTTPRoute grafana-https: grafana.ameide.io → grafana:80
- HTTPRoute loki-https & ameide-loki-ui: loki.ameide.io → loki-gateway:80
- HTTPRoute tempo-https & ameide-tempo-ui: tempo.ameide.io → tempo:3200
- HTTPRoute prometheus-https: prometheus.ameide.io → prometheus-oauth2-proxy:80
- HTTPRoute alertmanager-https & ameide-alertmanager-ui: alertmanager.ameide.io → alertmanager-oauth2-proxy:80
- HTTPRoute metrics-https: metrics.ameide.io → otel-collector:8888
- HTTPRoute telemetry-https: telemetry.ameide.io → otel-collector:4317
- HTTPRoute keycloak (two rendered): auth.dev.ameide.io/auth.ameide.io → keycloak:4000 and auth.ameide.io → keycloak:8080
- HTTPRoute plausible: plausible.ameide.io → plausible-plausible:8000
- HTTPRoute argocd: argocd.ameide.io → argocd-server:80
- HTTPRoute pgadmin: pgadmin.ameide.io → pgadmin:80
- HTTPRoute temporal-web: temporal.ameide.io → temporal-oauth2-proxy:80
- HTTPRoute langfuse-web: evals.ameide.io → platform-langfuse-web:3000
- HTTPRoute HSTS policy: *.ameide.io/ameide.io (no backend; policy only)
- App chart HTTPRoutes: www.ameide.io → www-ameide:3000; platform.ameide.io → www-ameide-platform:3001
