> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# backlog/362 – Unified Secret Guardrails & Bootstrap _(archived)_

> **Status:** Archived in favor of `backlog/362-unified-secret-guardrails-v2.md`. Keep this file for historical context (Nov 2025–Mar 2026). All new updates must land in the v2 backlog.

**Created:** Mar 2026  
**Owner:** Platform DX / Infrastructure  
**Supersedes:** backlog/348 (Environment & Secrets Harmonization), backlog/355 (Secrets Layer Modularization), backlog/360 (Secret Guardrail Enforcement)

---

## Why a new backlog?
The prior backlogs converged on the same outcome—“every workload reads secrets from Vault, charts guardrail misconfiguration, and automation keeps it honest”—but the work was scattered across multiple documents. This backlog consolidates the principles, codifies the direction that services own their credentials (with Layer 15 handling only shared bootstrap), and tracks the remaining execution steps. All open items in 348/355/360 should now be migrated here; those docs stay for historical reference but no longer receive updates.

> **Nov 2025 update:** CI guardrail enforcement is temporarily deferred. Keep the automation design documented here, but rely on Tilt + manual runs of `scripts/validate-hardened-charts.sh` until the deferral is lifted.

---

## Guiding Principles
1. **Service-Scoped Ownership** – Each Helm chart (or its namespace bootstrap chart) must define and document the ExternalSecrets it depends on. Layer 15 only provisions shared primitives: the `ClusterSecretStore`, Azure/Vault wiring, and baseline releases explicitly marked as “platform-wide”.
2. **Zero Inline Secrets** – Repository manifests never embed credentials. Every chart requires its users to supply `existingSecret` references (enforced via `required(...)`) or consumes secrets created by its own ExternalSecret template.
3. **Fail Fast Everywhere** – Preflight hooks (db-migrations, bootstrap scripts, smoke tests) validate that every referenced secret exposes the keys they need. Devs should learn about missing credentials before Flyway, Temporal, or Redis clients start.
4. **Consistent Bootstrap** – The GitOps-managed `foundation-vault-bootstrap` CronJob, Layer 15 Helmfile runs, and the Tilt guardrail must leave a fresh local cluster in the same shape as staging/prod: all required ExternalSecrets exist and reconcile successfully, even when the owning services are disabled.
5. **Automated Enforcement (Deferred)** – Long-term target is CI (helm lint/template + `scripts/validate-hardened-charts.sh`) plus infra health-check jobs that fail when charts violate the guardrails. During the deferral, teams must run the script via Tilt or locally before merging and capture regressions in review.

---

## Objectives
1. **Close Remaining Service Gaps** – Migrate any outstanding items from backlog/348 (env/secret split), backlog/355 (per-service releases), and backlog/360 (db-migrations regression) into this doc and drive them to completion.
2. **Layer 15 Boundary** – Reduce Layer 15 (`secrets-certificates`) to: `vault-secret-store`, shared secret bundles (core/observability/temporal), and *one* data-plane bundle that pre-provisions the platform/agents/graph/transformation/threads/agents-runtime/workflows/langgraph database secrets needed before migrations run. All other services must ship their own ExternalSecrets.
3. **Unified Vault Fixture Story** – Keep the `foundation-vault-bootstrap` CronJob fixture map (shared values + ConfigMap) current with every secret key referenced in repo charts (including LangGraph). Document how to seed new keys, how to trigger the CronJob for immediate reconciliation, and how to rotate them per environment.
4. **Guardrail Automation (Deferred)** – Keep `gitops/ameide-gitops/scripts/validate-hardened-charts.sh` ready to template the data-plane bundle + db-migrations, but pause CI wiring until enforcement resumes; document any manual checks performed in the interim.
5. **Operational Runbooks** – Every chart README and Layer 15 bundle must describe ownership, rotation steps, and the smoke tests that verify the secrets.

---

## Workstreams
### 1. Service Alignment
- Migrate backlog/348 checklists into this doc’s tracker (see below) and mark completed dates.
- Ensure each chart consumes secrets via `existingSecret` or an in-chart ExternalSecret; delete any remaining inline secrets or `createSecret` toggles.
- Update service READMEs with rotation/runbook details.

### 2. Layer 15 Rationalization
- Keep `vault-secret-store`, `vault-secrets-cert-manager`, `vault-secrets-k8s-dashboard`, `vault-secrets-neo4j`, `vault-secrets-observability`, and `vault-secrets-argocd` as-is. Temporal secrets now come from CNPG (no `vault-secrets-temporal` bundle).
- Introduce (or confirm) a **data-plane DB secrets** release that creates the platform/agents/graph/transformation/threads/agents-runtime/workflows/langgraph secrets *before* db-migrations runs—regardless of whether those services deploy.
- Remove per-service secret bundles that have been moved into their charts (pgAdmin, Plausible, Langfuse, etc.).

### 3. Local Bootstrap & Fixtures
- Keep the `foundation-vault-bootstrap` CronJob fixture data aligned with every ExternalSecret key (already updated for LangGraph); document required additions in this backlog whenever a new secret is referenced.
- Ensure the CronJob and `scripts/infra/bootstrap-db-migrations-secret.sh` leave a fresh DevContainer run with all data-plane secrets materialized automatically. Emit actionable errors (naming the missing secret/key) when reconciliation fails.
- Tilt still auto-runs `infra:db-secrets-bootstrap` (backed by `scripts/infra/bootstrap-db-migrations-secret.sh`) ahead of application resources so `agents-db-secret` and the rest of the data-plane credentials exist before any workload starts (Apr 2026 guardrail fix).
- Treat integration test jobs the same as services: every `*-int-tests-secrets` entry in the Layer 15 `integration-secrets` release must define the exact env keys the pods consume (DB URLs, API tokens, OTEL exporter endpoints, etc.), with matching fixtures inside the CronJob payload. The Tilt-managed `test-transformation` runner depends on `transformation-int-tests-secrets`, so missing keys (e.g., `OTEL_EXPORTER_OTLP_ENDPOINT`) surface as guardrail regressions rather than noisy runtime errors.

### 4. Automation & Guardrails
- **CI (Deferred):** Document the intended `.github/workflows/ci-helm.yml` integration for `gitops/ameide-gitops/scripts/validate-hardened-charts.sh`, but pause wiring/fail-fast gating until the deferral lifts. Track any manual validations performed per PR.
- **Tilt / Infra health-check:** Keep the existing guardrails (Tilt requiring infra layers, secrets-smoke wait knobs) documented here. Extend infra smoke tests to include db-migrations + ExternalSecret readiness.
- **Runtime validation:** db-migrations, workflows-runtime, Temporal namespace bootstrap, etc., must emit clear error messages when prerequisites are missing.

### 5. Documentation & Communication
- Update backlog/348/355/360 with a header pointing to this backlog for future updates.
- Add a short “Secret ownership” section to each service README (or chart README) summarizing where credentials live, how to rotate them, and how to re-run the guardrails.

---

## Service Tracker (migrated from backlog/348)
| Service | Status | Notes / Owner |
| --- | --- | --- |
| agents | ✅ (Nov 2025) | Guardrails + README done; keep verifying via `scripts/validate-hardened-charts.sh` (CI once enforcement resumes). |
| agents-runtime | ✅ (Nov 2025) | Same as above. |
| db-migrations | ✅ (Nov 2025 regulator, Mar 2026 fail-fast) | Guardrail merged (Mar 2026). `vault-secrets-platform` pre-provisions the data-plane DB secrets, `scripts/infra/bootstrap-db-migrations-secret.sh` enforces their presence, `secrets-smoke` verifies them, and the Flyway image now follows Tilt’s v3 registry flow so migrations always run the latest SQL. |
| graph | ✅ (Mar 2026, revalidated Apr 2026) | Inline secret fallback deleted; chart now requires `graph-db-credentials` and fails template rendering when the secret is missing. README + guardrails verified. |
| platform | ✅ (Apr 2026) | Chart now requires `platform-db-credentials`; README documents rotation + guardrails so Flyway/db-migrations keep using Vault-managed creds. |
| transformation | ✅ (Mar 2026) | README/runbook updated with rotation + integration secret guidance. Tilt now builds the `transformation-integration-test` image and Helm release, both of which rely on `transformation-int-tests-secrets` for DB + telemetry env (seeded via Layer 15 + the `foundation-vault-bootstrap` CronJob). |
| threads | ✅ (Mar 2026) | README covers DB + bearer token secrets, integration job wiring, and resync steps. |
| agents-runtime | ✅ (Mar 2026) | Runtime README documents `agents-runtime-db-secret` ownership and rotation workflow. |
| workflows | ✅ (Mar 2026, revalidated Apr 2026) | Helm now fails rendering unless `workflows-db-secret` exists; README documents rotation + integration secret guardrails. |
| inference-gateway (Redis) | ✅ (Apr 2026) | Redis buffering still optional, but the redis-failover release now requires the `redis-auth` secret rendered via ExternalSecret before inference-gateway may enable the feature. |
| Langfuse (app + bootstrap) | ✅ (Nov 2025) | Runbook documented. |
| pgAdmin / Plausible / Langfuse extras | ✅ (Apr 2026) | Plausible chart no longer renders inline secrets; `plausible-secret` must come from Vault/ExternalSecrets just like pgAdmin/Langfuse. |
| keycloak (operator instance) | ✅ (Apr 2026) | Keycloak CR now depends on `keycloak-db-credentials`/`keycloak-bootstrap-admin` secrets exclusively; inline secret templates + `createSecret` toggles removed. |
| www-ameide-platform (tenant bootstrap) | ✅ (May 2026) | Onboarding service now uses the Vault-backed master admin credentials to assign `realm-management/realm-admin` to the `platform-app-master` service account inside every tenant realm before provisioning client secrets, then refreshes the admin token cache. Guardrails stay intact (no inline secrets) while eliminating the manual post-provisioning step that previously caused 403s. |
| Integration test bundles | ✅ (Nov 2025) | Tilt resource `infra:30-integration-secrets` applies the shared ExternalSecret catalog (same values as `infra/kubernetes/values/integration/integration-secrets.yaml`). Integration jobs depend on that resource so Playwright + other runners receive secrets before launching. |

*(Update this table as work completes; remove rows once a service lands in a chart README with rotation/runbook steps and guardrail coverage—manual today, CI once re-enabled.)*

---

## Deliverables
1. **Layer 15 data-plane bundle** + documentation that ensures db-migrations finds all required secrets pre-install.
2. **Updated Vault fixtures + bootstrap scripts** covering every key referenced by the guardrails.
3. **Guardrail coverage** (manual while CI is deferred) that renders the hardened charts and exercises db-migrations fail-fast logic.
4. **Service README updates** describing secret ownership, rotation steps, and guardrails.
5. **Backlog archival note** on 348/355/360 pointing to this backlog.

---

## Risks & Mitigations
| Risk | Mitigation |
| --- | --- |
| Layer ordering regressions when adding/removing bundles | Keep Helmfile `needs` updated and document prerequisites in this backlog. |
| CI noise when new secrets are introduced | Even while CI is paused, require fixture entries in the `foundation-vault-bootstrap` CronJob ConfigMap + environment overlays before merging so automation is ready when re-enabled (use the legacy `scripts/vault/ensure-local-secrets.py --dump-fixtures` only for debugging). |
| Operational drift when teams disable bundles per environment | Use the per-environment `15-secrets.values.yaml` to track `installed` flags and keep infra health-check verifying them. |

---

## Next Check-in
- **Target date:** Apr 2026 (post Layer 15 data-plane bundle rollout; CI enforcement deferred—revisit timeline once reinstated). Document outcomes + remaining gaps inside this file.



Flow
The GitOps-owned `foundation-vault-bootstrap` CronJob is the canonical fixture generator. It renders a ConfigMap from the shared values, initializes/unseals Vault, enables the `secret/` kv-v2 mount, configures Kubernetes auth for `foundation-external-secrets`, and seeds the deterministic defaults before Layer 15 reconcilers run. The Helmfile layer then creates the ClusterSecretStore (infra/kubernetes/values/platform/vault-secret-store.yaml (line 1)) that points at Vault using the key map in infra/kubernetes/values/platform/vault-secrets.base.yaml (line 1). Layer 15 bundles (infra/kubernetes/values/platform/vault-secrets-*.yaml) render ExternalSecrets that sync Vault keys into namespace-scoped K8s Secrets, which application charts consume through existingSecret or extraSecretRefs. Guardrails such as scripts/infra/bootstrap-db-migrations-secret.sh (line 1) and infra/kubernetes/charts/platform/db-migrations/values.yaml (line 24) verify that every secret exists before Flyway or workloads run.
        ↓ (foundation-vault-bootstrap seeds Vault)
Vault kv/secret (keys like platform-db-url, minio-root-user, …)
        ↓ (ClusterSecretStore ameide-vault)
Layer 15 vault-secrets-* ExternalSecrets
        ↓
Namespace Secrets (platform-db-credentials, agents-db-secret, …)
        ↓
Service charts via existingSecret / extraSecretRefs
        ↓
db-migrations guardrail + runtime Pods
Creation Zones

Local fixture + sync: the CronJob enumerates every key/placeholder and seeds Vault so dev clusters immediately get deterministic credentials; scripts/infra/bootstrap-db-migrations-secret.sh (lines 1-230) sanity-checks them before db-migrations runs. (`scripts/vault/ensure-local-secrets.py` remains available for ad-hoc fixture dumps/debugging.)
Secret store + key map: infra/kubernetes/values/platform/vault-secret-store.yaml (line 1) defines the ameide-vault ClusterSecretStore. infra/kubernetes/values/platform/vault-secrets.base.yaml (lines 1-120) maps logical names (e.g., grafanaUser, platformDbUrl, wwwAmeideAuthSecret) to the Vault keys referenced everywhere else so one change updates every chart.
Data-plane bundle: infra/kubernetes/values/platform/vault-secrets-platform.yaml (lines 21-420) pre-provisions platform, agents, agents-runtime, graph, transformation, threads, and workflows database secrets so they exist before those charts or db-migrations install.
Integration test bundle: infra/kubernetes/helmfiles/15-secrets-certificates.yaml now includes the `integration-secrets` release (backed by infra/kubernetes/values/integration/integration-secrets.yaml) which renders all `*-int-tests-secrets` ExternalSecrets into `ameide-int-tests`. Integration workloads reference them via `envFrom` and never render their own copies.
Infra bundles: infra/kubernetes/values/infrastructure/cert-manager/vault-secrets.yaml (Azure DNS), .../kubernetes-dashboard/vault-secrets.yaml (dashboard OAuth proxy), .../neo4j/vault-secrets.yaml (Neo4j admin/TLS), infra/kubernetes/values/platform/vault-secrets-observability.yaml (Grafana admin, Loki/Tempo S3 users, MinIO service users), .../vault-secrets-argocd.yaml (ArgoCD admin + OIDC), plus the OAuth proxy charts (infra/kubernetes/values/infrastructure/prometheus-oauth2-proxy.yaml, .../alertmanager-oauth2-proxy.yaml, .../temporal-oauth2-proxy.yaml) materialize the shared infra secrets consumed by cert-manager, Grafana/Loki/Tempo, Temporal, Argo CD, and the OIDC sidecars.
Operator + service-owned secrets: infra/kubernetes/charts/operators-config/postgres_clusters/values.yaml (lines 1-70) (Postgres auth + replication), infra/kubernetes/charts/operators-config/redis-failover/values.yaml (lines 78-99) (Redis auth + ACLs), infra/kubernetes/charts/operators-config/keycloak_instance/templates/externalsecret.yaml (lines 1-120) (Keycloak DB, admin, master client), infra/kubernetes/values/platform/langfuse-bootstrap.yaml (lines 3-68) (Langfuse runtime + bootstrap keys), infra/kubernetes/values/platform/inference.yaml (lines 83-120) (inference DB + Langsmith/OpenAI/Langfuse API keys), infra/kubernetes/values/platform/www-ameide.yaml (lines 86-97) and .../www-ameide-platform.yaml (lines 161-179) (frontend auth + DB secrets), infra/kubernetes/values/platform/plausible.yaml (lines 23-63) (Plausible runtime/admin), and infra/kubernetes/charts/platform/pgadmin/templates/externalsecret.yaml (lines 1-29) plus default-credentials-externalsecret.yaml (lines 1-27) (pgAdmin OIDC + bootstrap creds) all define ExternalSecrets directly in their charts.
Service Map

Data-plane credentials (Layer 15-provisioned)

K8s Secret	Created in repo	Consuming services
platform-db-credentials	infra/kubernetes/values/platform/vault-secrets-platform.yaml (lines 21-65)	platform Deployment (infra/kubernetes/environments/staging/platform/platform.yaml (line 17)) and db-migrations job (infra/kubernetes/charts/platform/db-migrations/values.yaml (line 24))
agents-db-secret	infra/kubernetes/values/platform/vault-secrets-platform.yaml (lines 75-120)	agents chart via postgresql.auth.existingSecret (infra/kubernetes/values/platform/agents.yaml (lines 12-18)) and db-migrations (infra/kubernetes/charts/platform/db-migrations/values.yaml (line 26))
agents-runtime-db-secret	infra/kubernetes/values/platform/vault-secrets-platform.yaml (lines 131-180)	agents-runtime Deployment through extraSecretRefs (infra/kubernetes/values/platform/agents-runtime.yaml (lines 30-44)) and db-migrations (infra/kubernetes/charts/platform/db-migrations/values.yaml (line 28))
graph-db-credentials	infra/kubernetes/values/platform/vault-secrets-platform.yaml (lines 180-222)	graph chart’s existingSecret (infra/kubernetes/values/platform/graph.yaml (lines 12-20)) and db-migrations (infra/kubernetes/charts/platform/db-migrations/values.yaml (line 26))
transformation-db-credentials	infra/kubernetes/values/platform/vault-secrets-platform.yaml (lines 224-282)	transformation Deployment (infra/kubernetes/values/platform/transformation.yaml (lines 40-58)) and db-migrations (infra/kubernetes/charts/platform/db-migrations/values.yaml (line 30))
threads-db-credentials	infra/kubernetes/values/platform/vault-secrets-platform.yaml (lines 284-327)	threads Deployment (infra/kubernetes/values/platform/threads.yaml (lines 20-33)) and db-migrations (infra/kubernetes/charts/platform/db-migrations/values.yaml (line 31))
workflows-db-secret	infra/kubernetes/values/platform/vault-secrets-platform.yaml (lines 329-374)	workflows Deployment (infra/kubernetes/values/platform/workflows.yaml (lines 12-20)) and db-migrations (infra/kubernetes/charts/platform/db-migrations/values.yaml (line 32))
Service-owned / auxiliary secrets

K8s Secret	Created in repo	Used by
minio-root-credentials	infra/kubernetes/values/platform/agents.yaml (lines 50-64) ExternalSecret	agents + agents-runtime for artifact uploads (infra/kubernetes/values/platform/agents.yaml (lines 36-40), .../agents-runtime.yaml (lines 30-34))
inference-db-credentials	infra/kubernetes/values/platform/inference.yaml (lines 83-119)	inference Deployment mixes DB creds with Langsmith/OpenAI/Langfuse API keys via existingSecret (infra/kubernetes/values/platform/inference.yaml (lines 42-47))
langfuse-secrets / langfuse-bootstrap-credentials	infra/kubernetes/values/platform/langfuse-bootstrap.yaml (lines 3-68)	langfuse chart (ClickHouse/NextAuth/Redis config in infra/kubernetes/values/platform/langfuse.yaml (lines 8-44)) and the bootstrap job
clickhouse-auth	infra/kubernetes/charts/operators-config/clickhouse/templates/externalsecret-auth.yaml (lines 1-31)	Shared ClickHouse release (infra/kubernetes/helmfiles/40-data-plane.yaml) publishes admin/langfuse/plausible credentials; Langfuse and Plausible chart values reference their own ExternalSecrets that read the Vault keys (infra/kubernetes/values/platform/langfuse.yaml (lines 15-32), .../plausible.yaml (lines 1-32))
www-ameide-auth / www-ameide-keycloak	infra/kubernetes/charts/platform/www-ameide/templates/externalsecrets.yaml (lines 1-31) driven by infra/kubernetes/values/platform/www-ameide.yaml (lines 86-97)	www-ameide Next.js frontend (infra/kubernetes/values/platform/www-ameide.yaml (lines 63-83))
www-ameide-platform-auth / www-ameide-platform-db	infra/kubernetes/charts/platform/www-ameide-platform/templates/externalsecrets.yaml (lines 1-60) with values from infra/kubernetes/values/platform/www-ameide-platform.yaml (lines 161-179)	www-ameide-platform portal (infra/kubernetes/values/platform/www-ameide-platform.yaml (lines 61-139))
plausible-secret / plausible-admin	infra/kubernetes/values/platform/plausible.yaml (lines 23-63)	plausible chart (same file)
pgadmin-azure-oauth / pgadmin-bootstrap-credentials	infra/kubernetes/charts/platform/pgadmin/templates/externalsecret.yaml (lines 1-29) and .../default-credentials-externalsecret.yaml (lines 1-27)	pgadmin release for OAuth + admin login
redis-auth	infra/kubernetes/charts/operators-config/redis-failover/values.yaml (lines 78-99)	Langfuse (infra/kubernetes/values/platform/langfuse.yaml (lines 35-44)), www-ameide-platform sessions, and any workloads pointing at Redis Sentinel
Infra-focused secrets (Grafana admin, Loki/Tempo S3, MinIO service users, Azure DNS, Keycloak DB/admin, ArgoCD OIDC, Prometheus/Alertmanager/Temporal OAuth proxies, Temporal DB creds) are created in the Layer 15 values listed in the “Creation Zones” bullets above, and each is wired into its owning chart (Grafana, Loki/Tempo, cert-manager, Keycloak operator, ArgoCD, kube-prometheus-stack, Temporal).

Natural next step if you add a new ExternalSecret: update the `foundation-vault-bootstrap` fixture map (values + ConfigMap), note the addition in this backlog, and verify the CronJob plus the relevant Helm values render the new key before merging. Keep `scripts/vault/ensure-local-secrets.py` around for ad-hoc fixture dumps/debugging, but do not rely on it for steady-state bootstrap.

## Nov 2025 bootstrap update

* The GitOps-owned `foundation-vault-bootstrap` CronJob now enforces the pieces local Vault clusters were missing: it auto-enables the `secret/` KV v2 mount, binds the `foundation-external-secrets` service account, and seeds keys so ExternalSecrets can authenticate immediately after a fresh devcontainer bring-up. (`scripts/vault/ensure-local-secrets.py` remains available for manual fixture generation but is no longer in the default bootstrap path.)
* Layer 15’s platform/data-plane ExternalSecrets (platform-db-credentials, agents/agents-runtime, graph, transformation, threads/chat, workflows) are syncing from Vault again, the ad-hoc Secrets we created for Keycloak bootstrap/platform personas have been pruned, and `scripts/infra/bootstrap-db-migrations-secret.sh` now runs cleanly because every prerequisite Secret is present and populated from Vault instead of manual YAML.
