> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# backlog/348 – Environment & Secrets Harmonization

> **Superseded:** Ongoing tracking moved to `backlog/362-unified-secret-guardrails.md`. Keep this file for historical context only.

## Principles
- **Single source of truth** – Charts and environment overlays define runtime configuration; container images ship only build-time defaults.
- **Config vs secret separation** – Non-sensitive knobs live in ConfigMaps, while credentials and tokens are delivered exclusively via Vault-managed secrets.
- **Environment parity** – Local, staging, and production overlays follow the same structure so differences are intentional and documented.
- **No inline credentials** – Repository manifests never embed passwords, URLs with secrets, or API keys; all such values originate from Vault or per-environment secret stores. (`scripts/vault/ensure-local-secrets.py` emits the dev fixtures.)

## Goal
Align how application services consume configuration versus sensitive data. We want clear separation between ConfigMap-backed “knobs”, Vault-managed secrets, and environment overlays so local, staging, and production behave predictably and we avoid leaking credentials into manifests.

- Cross references:
  - backlog/355 – Secrets Layer Modularization (Layer 15 refactor)
  - backlog/360 – Secret Guardrail Enforcement (gap log + automation)
  - backlog/338 – DevContainer Startup Hardening (bootstrap wiring)
  - backlog/341 – Local Secret Store (provider/seeding abstraction)

## Services in scope
The following releases pull per-environment values and/or Vault secrets today and should be reviewed in this spike:

- agents (`gitops/ameide-gitops/sources/charts/apps/agents`)
- agents-runtime (`gitops/ameide-gitops/sources/charts/apps/agents-runtime`)
- coredns-config (`infra/kubernetes/charts/platform/coredns-config`)
- db-migrations (`infra/kubernetes/charts/platform/db-migrations`)
- devpi-registry (`infra/kubernetes/charts/platform/devpi-registry`)
- gateway (`gitops/ameide-gitops/sources/charts/apps/gateway`)
- go-module-uploader (`infra/kubernetes/charts/platform/go-module-uploader`)
- graph (`infra/kubernetes/charts/platform/graph`)
- inference (`infra/kubernetes/charts/platform/inference`)
- inference-gateway (`infra/kubernetes/charts/platform/inference_gateway`)
- langfuse (3rd-party chart + `langfuse-bootstrap`)
- namespace (`infra/kubernetes/charts/platform/namespace`)
- pgadmin (`infra/kubernetes/charts/platform/pgadmin`)
- platform (`gitops/ameide-gitops/sources/charts/apps/platform`)
- plausible (`infra/kubernetes/charts/platform/plausible`)
- registry-alias (`infra/kubernetes/charts/platform/registry-alias`)
- registry-mirror (`infra/kubernetes/charts/platform/registry-mirror`)
- runtime-agent-langgraph (retired; folded into `apps-agents-runtime`)
- threads (`infra/kubernetes/charts/platform/threads`)
- transformation (`infra/kubernetes/charts/platform/transformation`)
- workflows (`infra/kubernetes/charts/platform/workflows`)
- workflows-runtime (`infra/kubernetes/charts/platform/workflows_runtime`)
- www-ameide (`infra/kubernetes/charts/platform/www-ameide`)
- www-ameide-platform (`infra/kubernetes/charts/platform/www-ameide-platform`)

## Approach
- **Inventory** every chart + environment overlay pair to document where configuration lives today (ConfigMap, Secret, values file).
- **Classify** each value as config vs secret, moving credentials into Vault-backed ExternalSecrets and keeping non-sensitive defaults in ConfigMaps or overlay values.
- **Refactor** charts incrementally: eliminate inline passwords, ensure ExternalSecrets expose all required keys, and keep chart defaults environment-agnostic.
- **Update documentation** after each change—append comments beneath the original findings rather than overwriting them so the history of decisions remains visible.
- **Validate** with a repeatable checklist (helm lint/template, Vault sync scripts, rollout of the affected deployment, targeted smoke tests) captured in the Test Activities outline.

### Target Implementation Checklist

Every service should converge on the following pattern (record activities in this backlog + backlog/360 when work completes):

1. **Configuration vs Secret boundary** – Shared defaults define only non-sensitive knobs; per-environment overlays override `environment`, hostnames, or performance tuning. All credentials (URLs with passwords, API keys, tokens) originate from Vault-managed ExternalSecrets.
2. **Secret guardrails** – Helm templates enforce `required(...)` on credentials unless `existingSecret` is explicitly set, and runtime code validates URLs/credentials at startup (e.g., `getDatabaseConfig()` parsing).
3. **Validation & automation** – `helm template` (local + staging values) and Tilt smoke tests run cleanly; CI/health checks fail fast when secrets are missing. Results should be captured in the service’s backlog entry (see tracker below) and mirrored in backlog/360’s gap log.
4. **Documentation** – Service README / backlog note documents where secrets live, how they’re rotated, and any overrides necessary for optional features (Redis buffering, Langfuse bootstrap, etc.).

### Service Alignment Tracker

Use this table to track progress against the checklist. When an item is completed, append the activity details under the service’s subsection below and update backlog/360’s gap log (so DevContainer/Layer 15 work stays in sync).

| Service | Target alignment | Notes / Next action | Tracking |
| --- | --- | --- | --- |
| agents | ✅ Complete – 2025-11-07 | MinIO config now provided exclusively via overlays, and both DB/MinIO secrets are required via Helm guardrails. | Logged below / backlog/360 (resolved) |
| agents-runtime | ⚠ Regression – Mar 2026 | `db-migrations` guard fails when `agents-runtime-db-secret` is missing because the chart stays disabled; Layer 15 must pre-provision the ExternalSecret so migrations can run independently. | backlog/360 + backlog/355 |
| coredns-config | ✅ Complete | Only local overlay tweaks required. | – |
| db-migrations | ✅ Complete – 2025-11-07 | Inline secret fallback removed; chart now requires Vault-provisioned secrets and renders cleanly via helm template. | Logged below / backlog/360 (resolved) |
| devpi-registry | ✅ Complete | Secrets provisioned via ExternalSecret. | – |
| gateway | ✅ Complete | Monitor TLS scope only. | – |
| go-module-uploader | ✅ Complete | Maintain Vault token rotation. | – |
| graph | ⚠ Regression – Mar 2026 | Local migrations fail until `graph-db-credentials` exists; add a Layer 15 data-plane bundle + README/runbook updates so the secret materializes before the chart installs. | backlog/360 + backlog/355 |
| inference | ✅ Complete | Re-run validation when adding new keys. | – |
| inference-gateway | ✅ Complete – 2025-11-07 | Redis buffering now consumes Vault-managed secrets; chart enforces config when enabled. | Logged below / backlog/360 (resolved) |
| langfuse (app + bootstrap) | ⚠ Pending | Document Vault rotation / bootstrap steps. | backlog/360 gap log |
| namespace | ✅ Complete | – | – |
| pgadmin | ✅ Complete | – | – |
| platform | ✅ Complete | Guardrails enforced via `required(...)` + URL validation. | backlog/360 (reference) |
| plausible | ✅ Complete | – | – |
| registry-alias | ✅ Complete | Monitor IP changes. | – |
| registry-mirror | ✅ Complete | Revisit when registry auth introduced. | – |
| runtime-agent-langgraph | ✅ Retired | Merged into `apps-agents-runtime`; no separate chart or secret required. | backlog/360 + backlog/355 |
| threads | ⚠ Regression – Mar 2026 | `threads-db-credentials` is absent on fresh clusters, so `db-migrations` aborts; Layer 15/local fixture updates must seed the secret ahead of the service release. | backlog/360 + backlog/355 |
| transformation | ⚠ Regression – Mar 2026 | Secret `transformation-db-credentials` only appears after the chart installs; migrations need that secret earlier. | backlog/360 + backlog/355 |
| workflows | ✅ Complete – 2025-11-07 | Helm template validation captured below; monitor future env-var cleanup needs. | Logged below / backlog/360 (resolved) |
| workflows-runtime | ✅ Complete | – | – |
| www-ameide | ✅ Complete | – | – |
| www-ameide-platform | ✅ Complete | – | – |

## Per-service notes

### agents
- **Chart**: Deployment consumes `agents-config` ConfigMap plus the Vault-backed `agents-db-secret`. ExternalSecret templates hydrate the DB credentials from Vault.
- **Env overlays**: Local/staging/production overlays now set `environment` and `observability.deploymentEnvironment`; the base chart no longer embeds cluster-specific defaults.
- **ConfigMap**: As of 2025‑11‑07, `config.minio.*` fields are required inputs (no more baked-in endpoints). Shared defaults keep only non-sensitive knobs, while overlays provide the concrete endpoint/bucket/prefix.
- **Secrets**: Both `postgresql.auth.existingSecret` and the MinIO secret fields are required at template-render time; the inline fallback values were removed.
- **Update (2025‑11‑07):**  
  - Added Helm `required(...)` guards for MinIO endpoint/bucket/prefix and secret names.  
  - Removed shared MinIO defaults from `values.yaml`; overlays (`charts/values/platform/agents.yaml` + per-environment overrides) now supply the runtime values.  
  - Re-ran `helm template agents-test charts/platform/agents -f charts/values/platform/agents.yaml -f environments/local/platform/agents.yaml` to verify the new guardrails.
- **Status:** ✅ Completed – 2025‑11‑07. Target checklist satisfied (config/secret split, guardrails, validation, docs updated).
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [x] Validate

### agents-runtime
- **Chart**: Similar pattern to agents with ConfigMap + ExternalSecret; runtime deployment loads config map and secret refs. As of 2025‑11‑07, MinIO config and OTEL headers are treated as required inputs instead of baked-in defaults.
- **Env overlays**: Local/staging/production overlays now supply `environment`, `observability.deploymentEnvironment`, MinIO endpoint/bucket/prefix, OTEL endpoint, and gRPC target; the base chart only declares placeholders.
- **ConfigMap**: Validates `config.agentsGrpcAddress`, MinIO endpoint/bucket/prefix, and OTEL endpoint via `required(...)`.
- **Secrets**: `minio.secret.*` fields are required (no implicit `minio-root-credentials`), so Vault-provisioned credentials must be referenced explicitly.
- **Update (2025‑11‑07):**  
  - Cleared shared MinIO/OTEL defaults from `charts/platform/agents_runtime/values.yaml`.  
  - Added Helm guardrails for `environment`, MinIO config, and MinIO secret references.  
  - Validated with `helm template agents-runtime-test charts/platform/agents_runtime -f charts/values/platform/agents-runtime.yaml -f environments/local/platform/agents-runtime.yaml`.
- **Status:** ✅ Completed – 2025‑11‑07.
- **Regression (2026-03-11):** `db-migrations` now hard-fails when `AGENTS_RUNTIME_DATABASE_*` env vars are empty, but the `agents-runtime-db-secret` ExternalSecret is created only when the chart installs. On fresh local clusters we run migrations before enabling the workload, so the guardrail never finds the secret.
- **Follow-up:** Move the ExternalSecret (or a copy) into the Layer 15 data-plane bundle tracked in backlog/355, add fixture overrides to `scripts/vault/ensure-local-secrets.py`, and document the local `kubectl annotate externalsecret agents-runtime-db-secret-sync ...` resync steps. Re-run `run-migrations.sh` once the bundle exists and record the validation result here.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [x] Validate

### coredns-config
- **Chart**: Single ConfigMap that rewrites `.dev.ameide.io` hostnames for k3d.
- **Env overlays**: Local overlay defines rewrite rules; other environments continue to rely on cluster DNS defaults.
- **ConfigMap**: Holds CoreDNS rewrite rules; non-secret but environment-specific.
- **Secrets**: None.
- **Status:** ✅ Completed – 2026-03-07. Confirmed only the local overlay requires rewrites and documented the decision here to prevent unnecessary staging/production churn.
- **Next steps:** None required unless additional clusters introduce bespoke host mappings; reopen if staging/prod networking changes.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor *(document-only; no chart changes needed)*
  - [x] Document
  - [x] Validate *(Helm template checks for local, staging, production)*

### db-migrations
- **Chart**: Pre-install Job that applies Flyway migrations. Previously offered a `createSecret` toggle; now consumes only pre-existing Vault secrets via `migrations.existingSecrets`.
- **Env overlays**: Local enables the release (staging/prod leave it disabled) and supplies additional secret names via `existingSecrets`.
- **ConfigMap**: None.
- **Secrets:** As of 2025‑11‑07 the chart requires `migrations.existingSecrets` to include at least one secret; `templates/secret.yaml` has been deleted and `envFrom` always references Vault-provisioned secrets.
- **Update (2025‑11‑07):**
  - Removed `migrations.createSecret`, `migrations.secretName`, and the inline secret template.
  - Added a Helm `required(...)` check to ensure `existingSecrets` is non-empty.
  - Validated with `helm template db-migrations-test charts/platform/db-migrations -f charts/platform/db-migrations/values.yaml -f environments/local/platform/db-migrations.yaml`.
- **Status:** ✅ Completed – 2025‑11‑07. All credentials now flow exclusively from Vault-managed secrets.
- **Regression (2026-03-11):** The job’s guardrail now fails before Flyway because `graph-db-credentials`, `transformation-db-credentials`, `threads-db-credentials`, and `agents-runtime-db-secret` are absent on fresh local clusters. Track the fix in backlog/360/355: pre-provision those secrets via Layer 15, extend `scripts/vault/ensure-local-secrets.py` to seed them, and teach `scripts/infra/bootstrap-db-migrations-secret.sh` to emit actionable errors when they are missing.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [x] Validate

### devpi-registry
- **Chart**: Deployment + Service + PVC; optional Secret manifests still render when inline credentials are present.
- **Env overlays**: Local currently injects root/upload passwords inline; staging/prod already reference `devpi-registry-root` and `devpi-registry-upload` secrets sourced from Vault.
- **ConfigMap**: None.
- **Secrets**: Vault-backed `packages-python-env` and `devpi-registry-*` secrets exist, but the chart still allows (and defaults to) plaintext passwords via `values.yaml`.
- **Status:** ✅ Refactor complete – 2026-03-07. Chart now requires pre-provisioned secrets (`devpi-registry-root`, `devpi-registry-upload`); inline password fields removed and ExternalSecrets provision those secrets across environments.
- **Follow-up:** Run `helm template` for local/staging/production and execute a Tilt deploy to confirm the registry recovers with the Vault-sourced credentials before closing the validation step.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [ ] Validate

### gateway
- **Chart**: Gateway API resources (Gateway, HTTPRoute, GRPCRoute, policies). No ConfigMap/Secret objects.
- **Env overlays**: Local overlay enables listeners and route additions; staging/prod overlays tailor domains and TLS.
- **ConfigMap**: N/A.
- **Secrets**: N/A (relies on referenced cert secrets).
- **Comment:** Listener hostnames now derive from `domain`, keeping base values environment-agnostic while overlays override where needed.
- **Status:** Complete – chart renders cleanly (`helm template`) and no secret dependencies are required.
- **Suggested action**: Monitor for future TLS certificate scope changes; otherwise no further work.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [x] Validate

### go-module-uploader
- **Chart**: Deployment references optional Secret for upload token; persistent volume config.
- **Env overlays**: Local/staging only tweak image tags and persistence; no production override yet.
- **ConfigMap**: None (env vars inline in Deployment).
- **Secrets**: Upload token expected via secret ref when provided.
- **Comment:** Chart now provisions an ExternalSecret (`go-module-uploader-token-sync`) reading `packages/go/upload_token`; deployment consumes the resulting `go-module-uploader-token` secret.
- **Status:** Complete – Vault-backed token wiring added and chart renders successfully via `helm template`.
- **Suggested action**: Ensure the Vault key is rotated as needed; otherwise no further harmonisation work.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [x] Validate

### graph
- **Chart**: Deployment with ConfigMap for runtime knobs and ExternalSecret for DB creds.
- **Env overlays**: Provide `environment`, `useSsl`, HTTPRoute toggle, and enforce `existingSecret`.
- **ConfigMap**: Sets default tenant, pooling limits, OTEL endpoint; inherits environment tags.
- **Secrets**: `graph-db-credentials-sync` draws URL/user/pass from Vault.
- **Comment:** Removed the legacy `GRAPH_DATABASE_SSL` env (ConfigMap now relies solely on `DATABASE_SSL`) and defaulted the Helm template to tolerate missing `config.useSsl` by using `default false`.
- **Status:** ⚠ Regression – 2026-03-11.
- **Regression:** `db-migrations` fails before Flyway because the guardrail expects `graph-db-credentials` even when the graph chart is disabled (typical for local-only migration runs). The ExternalSecret currently lives inside the service chart, so migrations on fresh clusters never see the secret.
- **Next steps:** Move the ExternalSecret (or an equivalent) into the forthcoming Layer 15 data-plane bundle, seed fixture overrides inside `scripts/vault/ensure-local-secrets.py`, and expand `infra/kubernetes/environments/local/platform/README.md` to cover the `kubectl annotate externalsecret graph-db-credentials-sync ...` resync instructions. Re-run `bash infra/kubernetes/scripts/run-migrations.sh` once the bundle exists and capture the validation output here.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [x] Validate

### inference
- **Chart**: Deployment pulls ConfigMap (`inference-config`) and secret; includes ExternalSecret seeding numerous API keys.
- **Env overlays**: Override worker counts, environment labels, service URLs per env.
- **ConfigMap**: Non-secret runtime wiring (graph/workflows endpoints, redis host) and telemetry values.
- **Secrets**: ExternalSecret maps to Vault keys for Postgres, Langsmith, OpenAI, Langfuse credentials; the chart now requires `secrets.existingSecret` and no longer scaffolds an inline Secret fallback.
- **Comment:** Removed the legacy Secret template, scrubbed unused `langsmith/openai/langfuse` inline values, and made the ExternalSecret-provisioned secret mandatory via `secrets.existingSecret`.
- **Status:** Complete – Vault is the sole source for credentials, and `helm template` with the local overlay renders successfully after the change.
- **Suggested action**: None at this time; re-run `helm template`/Tilt smoke when adding new secret keys.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [x] Validate

### inference-gateway
- **Chart**: Deployment consumes env vars only; no ConfigMap. As of 2025‑11‑07 the chart enforces `environment` and OTEL endpoint inputs and supports Redis buffering credentials.
- **Env overlays**: Local/staging/prod overlays supply `environment`, OTEL deployment env, and network-specific addresses.
- **Redis update (2025‑11‑07):** Added optional Vault-backed secret wiring so that when `redis.enabled=true`, `redis.url`, username, and password secret must be defined. The Deployment sets `REDIS_URL`, `REDIS_USERNAME`, and `REDIS_PASSWORD` accordingly.
- **Status:** ✅ Completed – 2025‑11‑07. Chart renders cleanly (`helm template inference-gateway-test ...`) and future Redis buffering can now source credentials from Vault.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [x] Validate

### langfuse (app + bootstrap)
- **Chart**: Third-party chart uses values to reference ExternalSecrets; `langfuse-bootstrap` job seeds credentials and has its own ExternalSecrets.
- **Env overlays**: Local enables ingress, local credentials, and S3 mini-deployment; staging/prod mostly rely on Vault.
- **ConfigMap**: Managed by upstream chart; not directly in repo.
- **Secrets**: `langfuse-secrets` / `langfuse-bootstrap-credentials` come from Vault, with local fallback.
- **Comment:** Local credentials now mirror the deterministic Vault fixtures (`lf_api_*`, `lf_admin_password`), keeping bootstrap secrets aligned across environments.
- **Rotation runbook (2025-11-07):**
  1. Update the source secrets (`langfuse-api-public-key`, `langfuse-api-secret-key`, `langfuse-admin-password`, etc.) in `scripts/vault/ensure-local-secrets.py`. These map 1:1 to the Vault keys defined in `infra/kubernetes/values/platform/vault-secrets.yaml`.
  2. For local dev, rerun `scripts/vault/ensure-local-secrets.py` (invoked automatically by Helmfile layer 15) so the new fixtures materialize inside the in-cluster Vault.
  3. For staging/production, rotate the upstream secrets in the backing secret store (Vault/AKV) and allow the `langfuse-secrets-sync` / `langfuse-bootstrap-credentials-sync` ExternalSecrets to reconcile. No Helm changes are required.
  4. Restart the `langfuse-bootstrap` job (if enabled) to seed the new admin credentials, then redeploy the Langfuse chart.
- **Status:** Complete – chart renders for all overlays after syncing local credentials with Vault values.
- **Suggested action**: Document rotation steps for `langfuse-bootstrap` when changing the Vault secrets.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [x] Validate

### namespace
- **Chart**: Simple Namespace manifest.
- **Env overlays**: None.
- **ConfigMap/Secrets**: N/A.
- **Status:** Complete – no refactor required; keep checklist for reference.
- **Suggested action**: none.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [ ] Refactor
  - [x] Document
  - [ ] Validate

### pgadmin
- **Chart**: Deployment with ConfigMap for application config, ExternalSecret for Azure OIDC client.
- **Env overlays**: Provide image repo overrides, hostnames, resource tweaks.
- **ConfigMap**: Contains OAuth configuration referencing env vars (`OAUTH2_CLIENT_ID/SECRET`) expected from Secret.
- **Secrets**: ExternalSecret now also seeds admin bootstrap credentials; Deployment reads email/password via secretKeyRef rather than inline values.
- **Comment:** Introduced `pgadmin-bootstrap-credentials` ExternalSecret (Vault-backed) and local defaults now resolve from Vault fixtures.
- **Status:** Complete – helm template succeeds with base values; no plaintext credentials remain in manifests.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [x] Validate

### platform
- **Chart**: Deployment referencing ConfigMap + Secret; ExternalSecret seeds `platform-db-credentials`.
- **Env overlays**: Local sets `environment` and direct password; staging/prod point to Vault secret.
- **ConfigMap**: Default tenant, DB pooling settings, OTEL endpoint, environment tags.
- **Secrets**: Vault-provisioned secret carries `PLATFORM_DATABASE_URL/USER/PASSWORD`.
- **Comment:** Removed the legacy `PLATFORM_DATABASE_SSL` env and defaulted `DATABASE_SSL` to handle missing `config.useSsl`, aligning with `@ameideio/configuration` expectations.
- **Status:** Complete – helm template renders cleanly across all overlays; configuration now relies solely on Vault-provided connection details.
- **Suggested action**: None pending.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [x] Validate

### plausible
- **Chart**: Deployment + Service + seed Job; ConfigMap holds static config, ExternalSecrets manage admin credentials.
- **Env overlays**: Local changes base URL, resources, enables seed job.
- **ConfigMap**: Base URL, proxies, logging; all non-secret.
- **Secrets**: Vault-backed `plausible-secret` and `plausible-admin`; the deployment now always sources `SECRET_KEY_BASE` from the secret and defaults the secret name when unspecified.
- **Comment:** Removed unused `secret.secretKeyBase` value and ensured the chart picks up the Vault secret deterministically.
- **Status:** Complete – helm template renders and no inline credentials remain.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [x] Validate

### registry-alias
- **Chart**: Service/Endpoints aliasing registry hosts.
- **Env overlays**: Local only; others disable install.
- **ConfigMap/Secrets**: None.
- **Comment:** No refactor required; validated via `helm template` to confirm manifest renders.
- **Status:** Complete – keep watching for IP changes if the alias target moves.
- **Suggested action**: none.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor *(not applicable)*
  - [x] Document
  - [x] Validate

### registry-mirror
- **Chart**: DaemonSet + ConfigMap when enabled.
- **Env overlays**: Local uses upstream defaults; staging/prod disable.
- **ConfigMap**: Renders `registries.yaml` if enabled.
- **Secrets**: None today.
- **Comment:** No secret integration required; validated with `helm template`. Documented follow-up if future creds needed.
- **Status:** Complete – ready for re-enable once credential requirements are known.
- **Suggested action**: None right now; revisit if registry auth is introduced.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor *(not applicable)*
  - [x] Document
  - [x] Validate

### runtime-agent-langgraph
Retired and merged into `apps-agents-runtime`; no separate chart, ConfigMap, or secret remains. Guardrails now rely on `agents-runtime-db-secret` only.

### threads
- **Chart**: Deployment uses ConfigMap and secret; ExternalSecret seeds DB credentials.
- **Env overlays**: Local toggles SSL false and sets plain password; staging enables SSL and resources.
- **ConfigMap**: Default agent IDs, integration defaults, inference gateway URL; some values arguably per-env.
- **Secrets**: `threads-db-secret-sync` maps Vault DB values.
- **Update (2025-11-07):** Ran `helm template` for local/staging overlays:
  - `helm template threads-test charts/platform/threads -f charts/values/platform/threads.yaml -f environments/local/platform/threads.yaml`
  - `helm template threads-test charts/platform/threads -f charts/values/platform/threads.yaml -f environments/staging/platform/threads.yaml`
  Both rendered cleanly. Tilt smoke remains part of standard infra checks.
- **Status:** ⚠ Regression – 2026-03-11.
- **Regression:** `db-migrations` now stops immediately when `THREADS_DATABASE_*` env vars are empty. The `threads-db-credentials` secret is only created once the threads chart installs, so local migrations that run before the service gets enabled never pass the guardrail.
- **Next steps:** Move the ExternalSecret (or an equivalent copy) into Layer 15, seed defaults via `scripts/vault/ensure-local-secrets.py`, and add a local runbook entry covering `kubectl annotate externalsecret threads-db-secret-sync ...` so developers can force reconciliation. Re-run the migrations job after the bundle ships and update this section with the validation evidence.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [x] Validate

### transformation
- **Chart**: Deployment pulls ConfigMap plus secret; ConfigMap currently carries `PLATFORM_DATABASE_URL` with credentials baked into defaults.
- **Env overlays**: Local flips nodeEnv/logLevel and references existing secret; staging overrides replica count and TLS.
- **ConfigMap**: Exports numerous env vars, including DB URL placeholder.
- **Secrets**: Vault ExternalSecret `transformation-db-credentials`.
- **Update (2025-11-07):** Validated with:
  - `helm template transformation-test charts/platform/transformation -f charts/values/platform/transformation.yaml -f environments/local/platform/transformation.yaml`
  - `helm template transformation-test charts/platform/transformation -f charts/values/platform/transformation.yaml -f environments/staging/platform/transformation.yaml`
  Both succeeded; Tilt smoke remains part of standard infra checks.
- **Status:** ⚠ Regression – 2026-03-11.
- **Regression:** The migrations job expects `TRANSFORMATION_DATABASE_*` env vars even when the transformation chart remains disabled, but the ExternalSecret only materializes in that chart. Fresh clusters therefore miss the secret and the guardrail fails before Flyway.
- **Next steps:** Ensure Layer 15 (or a bootstrap helper) pre-provisions `transformation-db-credentials`, add fixture values + README guidance for resyncing, and record the rerun of `run-migrations.sh` once the secret is available.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [x] Validate

### workflows
- **Chart**: Deployment with ConfigMap for Temporal settings + secret for DB; ExternalSecret seeds credentials.
- **Env overlays**: Set environment string, TLS, and secret usage; local still sets raw password.
- **ConfigMap**: Contains Temporal namespace/address, DB pooling toggles.
- **Secrets**: `workflows-db-secret-sync` reads from Vault.
- **Update (2025-11-07):** Ran `helm template workflows-test charts/platform/workflows -f charts/values/platform/workflows.yaml -f environments/local/platform/workflows.yaml` and the staging variant; both render successfully.
- **Status:** ✅ Refactor + validation complete – overlay structure remains production-aligned. Future cleanup of duplicated env vars tracked separately.
- **Comment:** Plaintext password removed; local overlay now points to the Vault-managed `workflows-db-secret`.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [x] Validate *(helm template local + staging)*

### workflows-runtime
- **Chart**: Deployment referencing ConfigMap only; no Secret usage.
- **Env overlays**: Local/staging adjust environment, temporal namespace, addresses.
- **ConfigMap**: Holds Temporal connection details, workflow service address, OTEL settings.
- **Secrets**: None.
- **Update (superseded):** Temporal namespaces are now declared via the Temporal operator (`TemporalNamespace` CRs in `data-temporal`); the old `temporal-namespace-bootstrap` job is removed.
- **Status:** ✅ No refactor required – defaults now assume production; overlays provide environment-specific values, and validation completed via Helm upgrade after namespace bootstrap.
- **Notes:** Confirmed no sensitive data in ConfigMap; base values now use production identifiers so hosted overlays explicitly set their target environments.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [x] Validate

### www-ameide
- **Chart**: Next.js frontend; ConfigMap holds runtime URLs, analytics config; ExternalSecret seeds Auth.js secret when enabled.
- **Env overlays**: Local configures Gateway HTTPRoute and service endpoints; staging uses legacy Ingress definitions.
- **ConfigMap**: Contains numerous env-style values (platform URLs, plausible endpoints).
- **Secrets**: `www-ameide-auth-sync` maps auth secret + Keycloak secret from Vault.
- **Status:** ✅ Refactor complete – 2026-03-07. Chart now enforces the pre-existing auth secret, removed inline Secret template, and defaults to production-safe settings with environment identifiers supplied per overlay.
- **Notes:** ExternalSecret remains the sole source for `AUTH_SECRET`/Keycloak credentials; ConfigMap carries public URLs and logging toggles only.
- **Next steps:** `helm template` (local/staging/production) and run the marketing site smoke test in Tilt to close validation.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [ ] Validate

### www-ameide-platform
- **Chart**: Next.js platform UI; ConfigMap extremely verbose with Auth.js, Keycloak, Redis, Telemetry variables; ExternalSecrets provide auth + DB credentials.
- **Env overlays**: Local overlay adds dozens of overrides (tenant IDs, Redis sentinel, watchers); staging sets image + telemetry endpoint.
- **ConfigMap**: Mixes environment toggles, URLs, Redis sentinel settings—many are environment-specific.
- **Secrets**: Vault-managed `www-ameide-platform-auth` and `www-ameide-platform-db`.
- **Status:** ✅ Refactor complete – 2026-03-07. Chart now requires the Vault-sourced auth/database secrets, removed the inline Secret template, and defaults to production-safe settings; local/staging overlays explicitly set their environments.
- **Notes:** ExternalSecrets provision all sensitive values; ConfigMap carries only public URLs and feature toggles. No secrets remain in shared values.
- **Next steps:** Run `helm template` and exercise the login flow in Tilt to finish validation.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [x] Refactor
  - [x] Document
  - [ ] Validate

## Cross-cutting observations
- Several services (platform, threads, transformation, workflows) still define inline database passwords in the local overlay even though Vault provides deterministic defaults. Moving local dev to Vault-only secrets will shrink drift.
- **Comment:** Local overlays for agents, platform, threads, and workflows now rely on Vault-provisioned secrets, and the transformation connection string now comes from the Vault-managed secret as well (no credentials remain in the ConfigMap).
- ConfigMap templates frequently include database SSL flags and pool settings duplicated across services. A shared helper or global values file could reduce the chance of inconsistent tuning.
- ExternalSecret definitions consistently target `secret/data/<key>` with a `value` property. Any new secrets added via this spike must continue that pattern to keep `scripts/vault/ensure-local-secrets.py` functioning.
- Frontend charts (`www-ameide*`) rely on ConfigMaps for large collections of ENV-like values. Harmonization should document which are safe in ConfigMaps and which should be secrets (Auth.js secrets and Redis passwords already live in Vault).

### Regression – database secrets missing before migrations (Mar 2026)
- Running `bash infra/kubernetes/scripts/run-migrations.sh` (or inspecting `kubectl logs -n ameide job/db-migrations`) now fails on fresh local clusters because the guardrail validates `graph-db-credentials`, `transformation-db-credentials`, `threads-db-credentials`, and `agents-runtime-db-secret`. Those ExternalSecrets are rendered only after their service charts install, so migrations abort before Flyway runs (BackoffLimitExceeded).
- Fix requires coordination with backlog/355 (Layer 15 bundle), backlog/360 (guardrail tracking), and this backlog (service docs). We need a Layer 15 data-plane bundle that pre-provisions the ExternalSecrets, Vault fixture updates in `scripts/vault/ensure-local-secrets.py`, and README/runbook updates under `infra/kubernetes/environments/local/platform/` describing how to re-sync the secrets without enabling the workloads.
- Once the bundle exists, add a `kubectl get secret <name>` preflight (or reuse `scripts/infra/bootstrap-db-migrations-secret.sh`) so migrations fail fast with a friendly help message and CI can assert the secrets exist. Record the verification in each service section below.

## Suggested next steps
1. Strip inline database credentials from local overlays (platform, threads, transformation, workflows, agents) and rely on Vault defaults. Update documentation for local bootstrap commands.
- **Comment:** Completed for agents, platform, threads, workflows, transformation, and inference; local environments now depend solely on Vault-provisioned secrets for database credentials.
2. For each ConfigMap, flag values that differ per environment and push those into overlay files to avoid accidental promotion of dev-only URLs.
3. Simplify inline secret templates (`createSecret`, plaintext passwords) by mandating ExternalSecret usage everywhere; remove unused code paths once Vault data verified.
4. Produce a follow-up matrix mapping Vault keys → consuming services so rotations become predictable (build on `scripts/vault/ensure-local-secrets.py`).
5. Validate frontend ConfigMaps (`www-*`) to ensure no sensitive tokens leak; if any remain, migrate them to secrets and source through Vault.
6. Pre-provision the graph/threads/transformation/agents-runtime database secrets via Layer 15 so `db-migrations` succeeds even when those charts are disabled, and document the local `kubectl annotate externalsecret ...` resync steps.
- **Checklist:**
  - [x] Inventory
  - [x] Classify
  - [ ] Refactor
  - [x] Document
  - [ ] Validate
