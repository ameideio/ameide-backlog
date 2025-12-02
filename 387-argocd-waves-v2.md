# Argo CD rollout phases v2: single orchestrated ladder

Objective: a single ApplicationSet + RollingSync enforces global ordering (CRDs -> operators -> configs/secrets -> bootstraps -> smokes) across foundation, platform, data, and apps. ExternalSecrets CRDs stay with CRDs; cross-set races are eliminated.

## Dependency fixes to apply
- Vault vs storage: if Vault uses PVCs backed by `foundation-managed-storage`, treat Vault as a runtime (move `vault-core` to 150; keep storage in 140). Only move storage higher (010/110) if it is true substrate for many runtimes. Vault bootstrap Jobs belong with runtimes (150) or a small post-runtime foundation band (155) if created.
- Split CRDs from operators: add CRD-only apps (e.g., `foundation-crds-cert-manager` at 110; domain CRDs for CNPG/Strimzi/Envoy/Prometheus in their bands) and set `installCRDs=false` on operator charts in *20 bands.
- Platform/auth hardening: Keycloak CRDs live in `platform-keycloak-crds` at 310 and the operator chart skips CRDs; Keycloak continues at 350 and the realm import at 355 with DB credentials in 330.
- Temporal namespace bootstrap order: namespace-creation Jobs/CRs should run post-runtime (new 455 band or a higher sync-wave inside 450). Keep 440 for configs that do not require a running Temporal cluster.
- Vault webhook certs are secrets/config CRs: place `vault-webhook-certs` in 130 with other certs/secrets, not 120.
- SecretStore ordering: if the provider depends on Vault being reachable, place it after Vault runtime (155) instead of relaxing health checks; keep ESO/Vault path verification in smokes.
- Temporal operator depends on cert-manager: document/retain this dependency so the operator stays behind foundation 120.

## Terminology and target state
- RollingSync phases: group Applications by a phase label (`rollout-phase`) and advance through steps in order. The number is only a selector; ordering is defined by the step list.
- Argo sync-wave annotation: `metadata.annotations.argocd.argoproj.io/sync-wave` is only for intra-app resource/child-app ordering (namespaces/CRDs → controllers → secrets/config → workloads → smokes). Do not use “wave” to describe the RollingSync label.
- Target end state: a single ApplicationSet orchestrating all components via RollingSync steps keyed to the phase label; per-app sync-wave annotations remain available for finer ordering inside each Application.
- RollingSync behaviour: each phase waits for all included Apps to be Healthy before advancing; treat phases as concurrency buckets (use different phases if strict ordering is required). `maxUpdate` is best-effort, not a hard cap.

## Target rollout-phase bands (pre-foundation substrate + bands; consistent *10 CRDs / *20 operators / *30 secrets / *40 configs/bootstraps / *50 runtimes / *99 smokes)
Notes on intent:
- CRDs: always land first so later resources can be created/applied and can dry-run.
- Operators/controllers: land after CRDs so controllers are ready to reconcile owned types.
- Secrets: include secret providers (SecretStore/ClusterSecretStore) and Secret/ExternalSecret payloads; depend on CRDs and controllers existing.
- Configs/bootstraps: provider configs that establish shared backends (e.g., storage classes); keep them in their domain band, ahead of workloads and aligned with dependencies.
- Runtimes: workloads/payloads that rely on the above.
- Smokes: last, to validate previous layers.

- Prerequisites (0–99)
  - 010: shared namespaces/CRDs needed before any domain (root/system/argocd or other third-party substrate such as ExternalSecrets CRDs; keep here only while globally required; domain CRDs live with their domain).
  - 020: bootstrap operators/controllers only (no managed resources; third-party/general setup).
  - 030–050: unused (no secret providers, secrets, configs, or workloads in prerequisites).
  - 099: reserved for bootstrap/substrate smokes if ever needed (avoid workloads here by default).

- Foundation: 100–199
  - 100: namespaces (root of all foundation objects; use when namespaces are foundation-local—cluster-global substrate namespaces sit in 010).
  - 110: CRDs (foundation CRDs such as `foundation-crds-cert-manager`; keep redis/clickhouse/CNPG in data, gateway/envoy in platform, prometheus in observability)
  - 120: operators/controllers (cert-manager, external-secrets controller; rely on CRDs and set `installCRDs=false`). If operator charts bundle runtime CRs, split them so runtimes stay in their domain bands. Ready/smoke probes (e.g., external-secrets-ready) live in smokes at 199.
  - 130: secret providers and shared secrets (vault-webhook-certs; shared kube/repo pull-secrets; reserve domain Secret/ExternalSecret payloads for their domains). Place Vault-dependent secret stores after Vault runtime if needed.
  - 140: configs/bootstraps (cluster-specific substrate configs like managed-storage, coredns-config)
  - 150: runtimes/workloads (Vault server and other foundation payloads that start after providers/configs exist)
  - 155: post-runtime bootstraps (optional; vault bootstrap Jobs or similar that must run after foundation runtimes are live; Vault-dependent secret stores fit here)
  - 199: smokes (foundation-operators-smoke, foundation-bootstrap-smoke; validate prior layers)

- Data core: 200–299
  - 210: CRDs (data CRDs: redis, clickhouse, CNPG, kafka/strimzi if CRDs are separate)
  - 220: operators/controllers (CNPG, redis-operator, clickhouse-operator, strimzi, other data operators)
  - 230: secrets (data Secret/ExternalSecret payloads: clickhouse-secrets, data DB creds, other data payload secrets)
  - 240: configs/bootstraps (db-migrations prep/config, configmaps)
  - 250: runtimes/workloads (clickhouse, kafka-cluster, redis-failover, minio)
  - 260: post-runtime bootstraps (db-migrations execution)
  - 299: smokes (data-plane-smoke)

- Platform: 300–399
  - 310: CRDs for platform (envoy-crds, gateway-api, other platform-specific CRDs)
  - 320: operators/controllers (envoy-gateway, keycloak-operator, and other platform operators)
  - 330: secrets (platform Secret/ExternalSecret payloads: registry creds, passwords, TLS keys when rendered as Secret, keycloak-db-credentials, vault-secrets-platform, cert-manager-config issuers/certs). For Envoy Gateway, this phase establishes the full xDS PKI chain (`ameidet-ca` root CA, `envoy-gateway-ca` intermediate, and the `envoy`/`envoy-gateway` server/client certs) so that control-plane and data-plane can do mutual TLS without self-signed one-offs.
  - 340: configs/bootstraps (gateway, keycloak-runtime config/DB, other platform configs; keep grafana-datasources in observability 540 instead of here)
  - 350: runtimes/workloads (envoy data-plane, other platform runtimes)
  - 355: post-runtime bootstraps (realm imports such as keycloak-realm that depend on the runtime)
  - 399: smokes (control-plane-smoke, secrets-smoke, auth-smoke). `platform-control-plane-smoke` must validate both Gateway API state (e.g., `gateway/ameide` `Accepted=True`, `Programmed=True`) and the Envoy xDS certificate chain (server cert `envoy-gateway` issued by `envoy-gateway-ca`, which in turn is issued by the dev root CA `ameidet-ca`) so broken PKI/xDS wiring fails the wave before apps depend on it.

- Data extended: 400–499
  - 410: CRDs (temporal operator CRDs, etc.)
  - 420: operators/controllers (temporal operator; depends on cert-manager in 120)
  - 430: secrets (temporal/db Secret payloads generated by CNPG; no Vault ExternalSecret bundle)
  - 440: configs/bootstraps (pre-runtime configs that do not require a running cluster)
  - 450: runtimes/workloads (temporal, pgadmin)
  - 455: post-runtime bootstraps (temporal-namespace-bootstrap or Jobs/CRs that must talk to the running cluster)
  - 499: smokes (data-plane-ext-smoke)

- Observability: 500–599
  - 510: CRDs (monitoring/observability CRDs such as prometheus-operator CRDs if split via a CRD-only chart)
  - 520: operators/controllers (prometheus operator or other obs operators)
  - 530: secrets (datasource secrets, observability credentials as Secret/ExternalSecret, vault-secrets-observability)
  - 540: configs/bootstraps (grafana-datasources/configmaps, alertmanager configs, dashboards, rules)
  - 550: runtimes/workloads (grafana, loki, tempo, otel-collector, alloy-logs, langfuse, langfuse-bootstrap, prometheus agents/servers)
  - 599: smokes (observability-smoke or similar)
  
- Apps: 600–699
  - 610: CRDs (unlikely/none)
  - 620: operators/controllers (if any app-side operator)
  - 630: secrets (app Secret/ExternalSecret payloads)
  - 640: configs (app/frontends configmaps)
  - 650: runtimes/workloads (app backends, frontends)
  - 699: smokes (apps-platform-smoke; reserve future app smokes here)

## Target design (steady state)
- Single ApplicationSet at Argo sync-wave annotation 0 (`environments/dev/argocd/apps/ameide.yaml`). One git generator consumes all component descriptors (foundation/platform/data/apps) and applies the `rollout-phase` bands below. Child Applications may still use the Argo `sync-wave` annotation internally for intra-app ordering.
- RollingSync steps keyed to phases: 010,020,099,100,110,120,130,140,150,155,199,210,220,230,240,250,260,299,310,320,330,340,350,355,399,410,420,430,440,450,455,499,510,520,530,540,550,599,610,620,630,640,650,699. Conservative `maxUpdate` for substrate/CRDs/operators; higher for stateless tiers. Keep 240/250/260, 340/350/355, and 440/450/455 separate when dependencies require.
- Template keeps syncPolicy as today (CreateNamespace, RespectIgnoreDifferences, SSA where applicable, SkipDryRunOnMissingResource for CRDs if needed) and labels (componentType/domain/dependencyPhase). Phase assignments adhere to the bands (secrets at *30, configs at *40, runtimes at *50). OrphanedResources/ignoreDifferences settings remain. Operator charts set `installCRDs=false`; CRD-only apps exist for cert-manager, prometheus-operator, gateway/envoy, CNPG/Strimzi, etc.

## Current dev inventory (rollout-phase → components)
Generated from `./wave_lint.sh --format json` (rolloutPhase values as of dev):
- 010: foundation-namespaces; foundation-crds-external-secrets
- 110: foundation-crds-cert-manager
- 120: foundation-cert-manager; foundation-external-secrets
- 130: foundation-vault-webhook-certs; foundation-ghcr-pull-secret (registry pull secret ExternalSecret for GHCR)
- 140: foundation-coredns-config; foundation-managed-storage
- 150: foundation-vault-core
- 155: foundation-vault-bootstrap; foundation-vault-secret-store
- 199: foundation-bootstrap-smoke; foundation-external-secrets-ready; foundation-operators-smoke
- Data ladder update: 250 platform-postgres-clusters (CNPG) now owns app DB secrets and roles, replacing foundation-vault-secrets-platform/temporal and keycloak-db-credentials ExternalSecrets.
- Platform/auth alignment now in ladder: 310 platform-keycloak-crds; 320 foundation-keycloak-operator; 350 platform-keycloak; 355 platform-keycloak-realm (DB creds now produced by CNPG at 250).
- Envoy Gateway ladder: 310 platform-envoy-crds; 320 platform-envoy-gateway; 330 platform-cert-manager-config (issues Envoy PKI: `ameidet-ca`, `envoy-gateway-ca`, and the `envoy`/`envoy-gateway` TLS secrets); 340 platform-gateway (Gateway/HTTPRoute/GRPCRoute CRs). The `platform-control-plane-smoke` test under 399 is the guardrail that asserts this chain is wired correctly and that `gateway/ameide` is `Accepted=True, Programmed=True` before higher bands proceed.

Notes:
- Some prereq apps retain a `foundation-` prefix even though they run in 010; treat the prefix as historical naming, not a guarantee of band 100–199.
- Vault-dependent secret stores now sit in 155 (post-runtime) so RollingSync waits for Vault to be reachable. Managed-storage stays in config (140) unless elevated to substrate. CNPG-owned secrets remove the cross-band dependency on platform/data DB ExternalSecrets at 330/430.
