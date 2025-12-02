# Reality check (Jan 2025)

> ⚠️ **Remote-first note:** References to “cluster mode” in this document still assume a local k3d environment. The default now is to run integration/e2e packs against the shared AKS dev namespace via Telepresence (see [435-remote-first-development.md](435-remote-first-development.md)). Update any k3d instructions before using them.

**What exists**

* Colocation: Most services/packages have `__tests__/unit` and `__tests__/integration`; some e2e exist (Playwright specs in `services/www_ameide_platform/.../__tests__/e2e`, minimal Python/Go e2e). Layout is uneven: many TS tests live deep under feature folders; agents runtime now lives at `services/agents_runtime` per the folder-underscore policy.
* Orchestration: `pnpm test:integration`/`pnpm test:e2e` drive `tools/integration-runner`, which discovers packs conventionally and renders Helm jobs (shared chart). Tilt has manual `integration-*` resources and builds images for packs; Playwright is a dedicated pack (`tests/playwright-runner`).
* Dual mode: Helpers enforce `INTEGRATION_TEST_MODE` (mock|cluster). Some suites only run in cluster (`services/agents_runtime/__tests__/integration` exits in mock). Mock-vs-live coverage is uneven across services.
* Artifacts: Integration/e2e artifacts land in `artifacts/integration/<service>/<timestamp>` or `artifacts/e2e/...`; runner can emit `summary.txt` and upload to S3. Unit/Jest CI output goes to `.test-results` and `.coverage`. No repo-wide JSON summary or single path scheme.
* CI: `ci-core-quality` runs platform Jest unit+integration and SDK tests with JUnit; `ci-go-integration` runs Go integration (mock; cluster only via manual dispatch); `ci-integration-packs` runs integration-runner (staging values). No single unit→integration→e2e pipeline for all services.
* Values/infra: Local integration values exist for many services; staging/prod overlays exist for a subset (e.g., no staging/prod graph/threads). Some runner `--list` entries show missing values (graph, ameide_core_proto).

**Gaps vs target**

* No repo-level `make/just test-{unit,integration,e2e}`; commands are per-service and CI-specific.
* Folder taxonomy not consistent (`__tests__/{unit,integration,e2e}` scattered, many nested feature paths; duplicate service names).
* Mock/live parity missing; some packs cannot run in mock and some lack non-local values files.
* Artifact formats not standardized to `artifacts/test-results/<service>/<suite>.xml` + JSON.
* No single README/contract for env vars (`TEST_ENV`/`TEST_PROFILE` vs `INTEGRATION_TEST_MODE`), Tilt buttons, or how to run by service.

**Sequenced plan (what to build next)**

1. **Repo-level entrypoints + artifacts (do first)**

   * Add a root `Makefile`/`justfile` with `test-unit`, `test-integration`, and `test-e2e` targets. Each accepts `SERVICE=<name>` (defaults to `all`) plus `TEST_ENV`/`TEST_PROFILE` (`mock|cluster`) and shells out to `scripts/test-unit.sh`, etc.
   * `scripts/test-unit.sh` dispatches by language:
     * TS: `pnpm --filter "<svc>" jest … --json --outputFile artifacts/test-results/<svc>/unit.results.json --reporters=default --reporters=jest-junit` (Jest already matches `__tests__` patterns; CLI docs confirm `--json/--outputFile` + reporters). 
     * Python: `pytest services/<svc>/__tests__/unit --junitxml=artifacts/test-results/<svc>/unit.junit.xml` (pytest’s `--junitxml` flag is stable).
     * Go: run through `gotestsum --junitfile artifacts/test-results/<svc>/unit.junit.xml ./services/<svc>/...` (or `go test -json | go-junit-report`).
     * Playwright: add the builtin `junit` reporter in `playwright.config.ts` targeting `artifacts/test-results/<svc>/e2e.junit.xml`.
   * `scripts/test-integration.sh` / `scripts/test-e2e.sh` wrap `tools/integration-runner` so every pack inherits the env/profile flags and writes artifacts under `artifacts/test-results/<pack>/integration-<profile>.junit.xml`. Extend integration-runner with a `--junit-root` flag that mounts `/artifacts/test-results`.
   * Canonical artifact shape:

     ```
     artifacts/
       test-results/
         <service-or-pack>/
           unit.junit.xml
           unit.results.json
           integration-mock.junit.xml
           integration-cluster.junit.xml
           e2e-mock.junit.xml
           e2e-cluster.junit.xml
     ```

     JUnit XML is the contract for CI/uploaders (pytest, jest-junit, gotestsum, Playwright all emit it); JSON is optional for internal analytics. Keep the integration-runner `summary.txt`, but link it from this tree.

2. **Env contract + dual-mode cleanup**

   * Define `TEST_ENV` (`local|ci|dev|staging|prod|cluster`) and `TEST_PROFILE` (`mock|cluster`). Always export `INTEGRATION_TEST_MODE=$TEST_PROFILE` for backwards compatibility until all packs switch over.
   * Add tiny helpers per language (TS, Py, Go) to read the env/profile once and derive endpoints/mocks. Example: `getTestConfig()` returning `{ env, profile }` for TS; `pytest` fixture that exposes the same; Go package for shared integration helpers.
   * Update `run_integration_tests.sh` scripts to source the helper, enforce required env vars per profile, and pass `TEST_ENV`/`TEST_PROFILE` through to Jest/pytest/go test.
   * Make every integration/e2e pack runnable in **both** profiles. For suites that currently exit early in mock mode, add mock fixtures (e.g., Playwright `page.route()` intercepts, Python stub servers, Go in-memory Connect stubs) so “mock” means “run against Tilt/k3d with fake upstreams” instead of “skip outright”.

3. **Layout + naming normalization**

   * Move scattered TS tests into each service’s `__tests__/{unit,integration,e2e}` (Jest already treats files in `__tests__` as tests; only need to move folders). Keep shared helpers in `tests/shared` or language-specific libs.
   * For Python, keep `test_*.py` but enforce `pytest.ini` `testpaths = services packages` so discovery respects the new layout.
   * For Go, leave `_test.go` next to code or collect them under `__tests__` per package if you want symmetry, but use `go test ./...`/`gotestsum` consistently.
   * Naming duplication resolved: agents runtime is at `services/agents_runtime` (underscored folders, hyphenated filenames where applicable); keep updating any straggler references in pnpm workspaces, Go modules, Dockerfiles, Helm values, and integration-runner metadata as they surface.

4. **Tilt buttons, Helm values, and Argo hooks**

   * Fill Helm value gaps so every pack has `infra/kubernetes/environments/{local,staging,production}/integration-tests/<pack>.yaml`. This lets `integration-runner --list` succeed everywhere and gives Tilt/k3d/CI the same manifests.
   * In Tiltfile, add `local_resource` entries such as `test-agents-unit` and `test-agents-int-mock` that simply call the root `just test-*` commands with `TEST_ENV=local` and `TEST_PROFILE=mock|cluster`. Use `auto_init=False` so they only run when a dev clicks the button; share the same deps as the services they exercise.
   * For staging/prod, add optional ArgoCD `PostSync` hooks that run the same test job Helm chart (with `TEST_ENV=staging`, `TEST_PROFILE=cluster`) to gate deployments on live smoke tests. Reuse the shared test-job chart so Tilt, CI, and Argo all execute identical jobs.

Document the above in `tests/README.md` (commands, env vars, artifact paths) and cross-link from service READMEs so the workflow is visible.
