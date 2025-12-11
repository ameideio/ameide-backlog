> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

> **DEPRECATED**: This backlog has been superseded by [430-unified-test-infrastructure.md](./430-unified-test-infrastructure.md).
>
> The content below is retained for historical reference. All new work should follow backlog 430.

---

Cross-Cutting

✅ Integration Helm resources now depend on `ameide_core_proto` before applying (`Tiltfile:1171,1192-1201`), so Buf build/push always runs ahead of the jobs per backlog/356.
✅ Inline credentials have been replaced with ExternalSecrets for agents, agents-runtime, graph, platform, workflows, and www_ameide_platform (local/staging/prod integration overlays + new vault key `integration-test-bearer-token`). Remaining services still need the same treatment if new secrets appear.
✅ Shared integration secret catalog (within `infra/kubernetes/charts/test-job-runner/values.yaml`) now powers the Layer 15 `integration-secrets` release; environment overlays simply mount the generated Secrets (no `integrationSecrets.use` blocks needed).

🚧 **Policy:** every service (existing or new) must ship a runnable `__tests__/integration` pack wired into the shared runner. (As of Nov 2025 all existing services comply; keep this note to guard future additions.)

Open items:
- ✅ Threads / inference / inference-gateway now mirror the ExternalSecret wiring (Vault-backed secrets + validated env vars). Keep watching for new credentials so we replicate the pattern immediately.
- ✅ Integration namespace shared chart/value completed via `test-job-runner/values.yaml` catalog; environment overlays now just reference service names.
test-agents

✅ Helm/runner now use the single `/app/__tests__/integration/run_integration_tests.sh` entrypoint, the runner emits JUnit via `gotestsum`, and DB/MinIO creds come from the Vault-backed `agents-int-tests-secrets`.
⏳ Nothing else pending here unless we add further env validation.
test-agents-runtime

✅ All envs now call `/app/__tests__/integration/run_integration_tests.sh`, pull MinIO credentials from `agents-runtime-int-tests-secrets`, and the runner now fails fast when AGENTS/MINIO env vars are missing.
test-graph

✅ Integration jobs now point at `/app/__tests__/integration/run_integration_tests.sh` and hydrate `GRAPH_DATABASE_URL`/`DATABASE_URL` from the `graph-int-tests-secrets` ExternalSecret.
ℹ️ Suite timeouts now respect the 100 s cap; revisit the runner only if we see future env validation gaps.
test-threads

✅ Integration overlays (local/staging/prod) now use the shared job schema, pull DB + bearer token secrets via `threads-int-tests-secrets`, and drop the inline Postgres URLs. Runner path auto-detection remains for legacy images but is covered by the single command entrypoint.
test-transformation

✅ Integration jobs use the shared entrypoint, pull DB credentials from `transformation-int-tests-secrets` ExternalSecret (`transformation-db-url`, `transformation-db-username`, `transformation-db-password`), and the Jest runner validates required env vars before executing tests.
test-inference

✅ Jobs now invoke the shared entrypoint, set `INFERENCE_BASE_URL`/`INFERENCE_GRPC_ADDRESS`, and the pytest runner validates those env vars before hitting the cluster.
test-inference-gateway

✅ Command now calls `/app/__tests__/integration/run_integration_tests.sh`; the runner discovers the compiled binary, emits JUnit via `gotestsum`, and honors `INTEGRATION_TEST_TIMEOUT`.
⚠️ No sensitive env vars today, but if Redis/Auth config ever lands here we need to mirror the ExternalSecret pattern.
test-platform

✅ Single entrypoint, JUnit output, and Vault-backed `platform-int-tests-secrets` replace inline DB URLs. No remaining blockers tracked here.
test-workflows

✅ Uses the unified runner path and now sources all DB URLs from `workflows-int-tests-secrets`. Temporal env vars remain inline (non-secret) and look fine.
test-www-ameide

Still green; nothing to change.
test-www-ameide-platform

✅ JEST runs the full suite again (no forced `--testPathPatterns`), the Vault-backed `www-ameide-platform-int-tests-secrets` provides `INTEGRATION_TEST_BEARER_TOKEN`, and both the runner + test suite now hard-require that token at startup. Coordinate with platform auth owners to keep the Vault secret populated with a real service account token per environment.
test-workflows-runtime

✅ Integration pack landed (Nov 2025). `/app/__tests__/integration/run_integration_tests.sh` now runs the pytest suite (config smoke + PlatformWorkflow execution/guard scenarios) using the Temporal test environment. The job image ships pytest + timeout plugin, and new values (`infra/kubernetes/environments/local/integration-tests/workflows-runtime.yaml`) provide the Temporal + workflows service endpoints. Keep the runner green alongside `test-all` and extend coverage as new runtime features land.
