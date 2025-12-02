# Helm Unit Tests (416)

**Status**: Draft  
**Intent**: Introduce chart-level unit tests with `helm-unittest` plus optional `kubeconform` schema checks so Helm changes fail fast before Argo or cluster apply.

> ⚠️ **Registry update:** Image-registry assertions referencing `k3d-ameide.localhost:5001` describe the retired local workflow. Dev/staging/prod baselines now pull the repositories and tags defined in `ameide-gitops` (published to GitHub Container Registry); update the suites so they validate `ghcr.io/ameide/...` entries and the GitOps-driven tag patterns before enforcing them.

## Scope
- All first-party charts under `gitops/ameide-gitops/sources/charts/**` (exclude `third_party/`).
- Runner script in `scripts/ci` to install the plugin and execute suites across charts.
- Seed suites for high-risk charts, then expand chart-by-chart using a shared pattern.

## Implementation status
- Runner added: `scripts/ci/test_helm_unittests.sh` (pinned plugin v1.0.3, skips charts without suites, configurable glob).
- Suites added:
  - `apps/platform`: db secret required, CNPG secret wiring, service/container port alignment.
  - `platform-layers/db-migrations`: enabled/disabled toggle, required secrets (fail fast), envFrom order, Flyway defaults.
  - `foundation/operators-config/postgres_clusters`: managed roles to per-app secrets (CNPG model).
  - `foundation/operators-config/postgres_clusters`: namespace override coverage.
  - `platform-layers/pgadmin`: ExternalSecret target defaults and data wiring.
  - `foundation/helm-test-jobs`: hook annotations, hook delete policy, multiple job names/weights.
  - `apps/inference-gateway`: HTTPRoute toggle/hostnames and SecurityPolicy gating.
  - `foundation/vault-bootstrap`: CronJob name/schedule.
  - Image tag/registry tests: `apps/workflows`, `apps/agents`, `apps/agents-runtime`, `apps/inference` (k3d dev registry, no underscores).
  - `foundation/vault-webhook-certs`: CA + serving Certificate presence.
- Templates hardened: db-migrations now `fail`s when `migrations.existingSecrets` is empty.
- Runner passes on current suites (`bash scripts/ci/test_helm_unittests.sh`).

## Decisions
- Test files live at `tests/*_test.yaml` in each chart root (plugin default).
- Runner discovers `Chart.yaml` roots (not `tests/` dirs) and invokes `helm unittest <chart> --file 'tests/**/*_test.yaml'`.
- Values are passed with `--values/-v` or `values:` inside suites. `-f/--file` remains reserved for test suite globs.
- Negative tests rely on `required`/`fail` guards in templates so `failedTemplate` assertions can catch missing inputs.
- Kubeconform (optional gate) validates `helm template` output with cached CRD schemas for CNPG Cluster, ExternalSecret, Gateway API, etc.

## Initial Test Matrix
- `apps/platform`: require `postgresql.auth.existingSecret`; verify Deployment envFrom/secretKeyRef wiring and service port matches values; render with repo `_shared` + `local` values via `--values`.
- `platform-layers/db-migrations`: `migrations.enabled` toggle; fail when `migrations.existingSecrets` is empty; assert envFrom order and Flyway env defaults.
- `foundation/operators-config/postgres_clusters`: cover new `cluster`/`managed.roles` schema and legacy `.Values.ameide` fallback; require role names/passwordSecret.name; namespace override honored.
- `apps/inference-gateway`: `httpRoute.enabled` on/off; hostnames templating; default gateway namespace/section; SecurityPolicy gated on CORS flag.
- `foundation/helm-test-jobs`: single-test shorthand (`command`/`script`) produces a Job with `helm.sh/hook: test` and chart-default Argo hook delete policy; list form renders distinct job names/weights.
- ExternalSecret emitters (e.g., `platform-layers/pgadmin`, app charts): assert target `creationPolicy`/`deletionPolicy`/template defaults and required remote keys block rendering.

## Runner Outline
1. Install `helm-unittest` plugin if missing.
2. Find charts: `find gitops/ameide-gitops/sources/charts -name Chart.yaml -not -path '*/third_party/*'`.
3. For each chart dir: `helm unittest <dir> --file 'tests/**/*_test.yaml'` (default glob ok if no subdirs).
4. Optional: `helm template <dir> ... | kubeconform -strict -summary -schema-location default -schema-location <cached CRD schemas>`.

## Follow-Ups
- Add CI job alongside `scripts/ci/lint_helm_charts.sh` to run the runner; cache `~/.cache/helm` and `~/.cache/kubeconform`.
- Expand suites to remaining charts as they change; keep “render with repo values” coverage in sync with ApplicationSet `_shared` + env-specific value files.
- Document local invocation in repo README (helm lint + helm unittest + optional kubeconform).

## Risks / Mitigations
- **Template guards missing**: add `required`/`fail` where negative tests expect failure.  
- **CRD schemas unavailable**: host cached schemas in CI artifact or fall back to default `kubernetesjsonschema.dev` with retries.  
- **Plugin drift**: pin `helm-unittest` version in the runner to avoid breaking changes.

## Concrete suites to codify (per chart)

### foundation/operators-config/postgres_clusters (CNPG)
- managed_roles_test: per-app roles point at `<app>-db-auth` secrets; superuser/app secrets default to cluster name.
- legacy_fallback_test: `.Values.ameide` populates `spec.managed.roles` identically to new schema.
- cnpg_schema_valid (optional): render a representative Cluster and validate with kubeconform + CNPG schema.

### apps/platform (consumer of CNPG creds)
- db_secret_required: missing `postgresql.auth.existingSecret` fails (failedTemplate).
- db_secret_wiring: with `postgresql.auth.existingSecret: platform-db-auth`, Deployment `envFrom` and `env[].valueFrom.secretKeyRef` reference that secret and do not reference legacy vault secrets.
- service_port_tracks_values: overridden `service.port` matches Service port and containerPort.

### platform-layers/db-migrations
- migrations_toggle: `enabled=false` renders no Job; `enabled=true` with minimal secrets renders one Job.
- secrets_required: `enabled=true` + empty `migrations.existingSecrets` fails (failedTemplate).
- envfrom_order_and_flyway_defaults: envFrom preserves secret order; Flyway env vars present with defaults and point to the DB secret.

### foundation/vault-core
- readiness_probe_relaxed: readinessProbe hits `/v1/sys/health` with `standbyok`, `perfstandbyok`, `sealedcode=200`, `uninitcode=200`.
- liveness_probe_policy: livenessProbe is stricter or omitted per decision.
- rollout_phase_label: rollout-phase label/annotation equals `"150"`.

### foundation/vault-bootstrap
- cron_schedule: CronJob schedule is `"*/1 * * * *"`.
- stable_name: CronJob name matches expected stable value (manual trigger friendly).
- rollout_phase_label: rollout-phase label/annotation equals `"155"`.

### dev registry image naming (per first-ring app: agents-runtime, inference-gateway, workflows-runtime, workflow-worker, www-ameide, www-ameide-platform, etc.)
- dev_image_registry: main container image matches `^k3d-ameide\.localhost:5001/ameide/<hyphenated-name>:dev$`.
- no_underscores: image string does not contain `_`.

### apps/inference-gateway
- httproute_toggle: disabled -> no HTTPRoute/SecurityPolicy; enabled -> exactly one of each.
- hostnames_and_defaults: hostnames match values; parentRefs namespace/section use defaults when unset.
- securitypolicy_gated: SecurityPolicy appears only when CORS/auth flag is on and references same hostnames/gateway as the HTTPRoute.

### foundation/helm-test-jobs
- command_shorthand: single test renders one Job with `helm.sh/hook: test`, Argo hook annotation, and default hook-delete policy.
- list_renders_distinct: multiple tests render uniquely named Jobs (and weights if supported).

### ExternalSecret emitters (e.g., platform-layers/pgadmin; pattern applies to other charts)
- target_semantics: ExternalSecret has intended `target.creationPolicy`/`deletionPolicy` and template with expected keys (e.g., PGADMIN_DEFAULT_EMAIL/PASSWORD).
- remote_keys_required: missing required remote keys (email/password) fails via failedTemplate.
- schema_valid (optional): kubeconform against ExternalSecret CRD schema.

### Cross-cutting render smoke
- renders_with_repo_values: for each first-party chart, render with `_shared` + env values (as ApplicationSet does) and assert at least one expected resource with key labels/annotations (e.g., rollout-phase, app labels). Add kubeconform checks for CRD-heavy charts (CNPG Cluster, ExternalSecret, Gateway API).
