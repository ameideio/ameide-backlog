> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# Backlog 348 – Env & Secrets Harmonization TODOs

## Chart defaults with baked-in credentials
- [ ] infra/kubernetes/values/infrastructure/pgadmin.yaml:36-38 — defaultCredentials still specify `admin@ameide.invalid` / `DisabledLoginOnly!`; require ExternalSecret inputs only.
- [ ] infra/kubernetes/charts/platform/workflows/values.yaml:12-19 — PostgreSQL auth block hardcodes `dbuser/dbpassword`; switch to a required secret reference.
- [ ] infra/kubernetes/values/integration/agents-runtime-integration.yaml:8-16 — Helm test env lists `minioadmin` access keys; move to a Vault-backed secret or ExternalSecret.
- [ ] infra/kubernetes/values/integration/threads-integration.yaml:31-33 — Integration values embed a full Postgres URL with `dbuser:dbpassword`; replace with secret projections.

## Local environment overlays leaking credentials
- [ ] infra/kubernetes/environments/local/infrastructure/keycloak.yaml:24-55 — Database and admin bootstrap passwords (`dbpassword`, `C1tr0n3lla!`) remain inline; migrate to secrets.
- [ ] infra/kubernetes/environments/local/platform/langfuse.yaml:82-85 — MinIO root user/password (`minioadmin`) defined inline; source via ExternalSecret.
- [ ] infra/kubernetes/environments/local/platform/langfuse-bootstrap.yaml:7-12 — `LANGFUSE_PUBLIC_KEY/SECRET_KEY/ADMIN_PASSWORD` seeded with fixed strings; move to Vault fixture and ExternalSecret.
- [ ] infra/kubernetes/environments/local/integration-tests/agents.yaml:15-26 — `DATABASE_URL` plus MinIO access keys hardcode credentials.
- [ ] infra/kubernetes/environments/local/integration-tests/agents-runtime.yaml:25-30 — MinIO access/secret keys fixed to `minioadmin`.
- [ ] infra/kubernetes/environments/local/integration-tests/graph.yaml:16-19 — Postgres URLs include `dbuser:dbpassword`.
- [ ] infra/kubernetes/environments/local/integration-tests/platform.yaml:18-20 — Postgres URLs include `dbuser:dbpassword`.
- [ ] infra/kubernetes/environments/local/integration-tests/workflows.yaml:19-24 — Postgres URLs include `dbuser:dbpassword`.

## Staging integration manifests with inline credentials
- [ ] infra/kubernetes/environments/staging/integration-tests/agents.yaml:18-32 — `DATABASE_URL` uses `dbuser:dbpassword`; ensure MinIO creds stay secret-backed.
- [ ] infra/kubernetes/environments/staging/integration-tests/graph.yaml:17-19 — Postgres URLs embed `dbuser:dbpassword`.
- [ ] infra/kubernetes/environments/staging/integration-tests/platform.yaml:17-19 — Postgres URLs embed `dbuser:dbpassword`.
- [ ] infra/kubernetes/environments/staging/integration-tests/repository.yaml:17-19 — Postgres URLs embed `dbuser:dbpassword`.
- [ ] infra/kubernetes/environments/staging/integration-tests/threads.yaml:31-33 — Postgres URLs embed `dbuser:dbpassword`.
- [ ] infra/kubernetes/environments/staging/integration-tests/workflows.yaml:17-21 — Postgres URLs embed `dbuser:dbpassword`.

## Production integration manifests with inline credentials
- [ ] infra/kubernetes/environments/production/integration-tests/agents.yaml:18-20 — Postgres URLs embed `dbuser:dbpassword`.
- [ ] infra/kubernetes/environments/production/integration-tests/graph.yaml:17-19 — Postgres URLs embed `dbuser:dbpassword`.
- [ ] infra/kubernetes/environments/production/integration-tests/platform.yaml:17-19 — Postgres URLs embed `dbuser:dbpassword`.
- [ ] infra/kubernetes/environments/production/integration-tests/repository.yaml:17-19 — Postgres URLs embed `dbuser:dbpassword`.
- [ ] infra/kubernetes/environments/production/integration-tests/threads.yaml:31-33 — Postgres URLs embed `dbuser:dbpassword`.
- [ ] infra/kubernetes/environments/production/integration-tests/workflows.yaml:17-21 — Postgres URLs embed `dbuser:dbpassword`.
