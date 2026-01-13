> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

---

> **DEPRECATED**: This backlog has been superseded by 430v2:
> - `backlog/430-unified-test-infrastructure-v2.md`
> - `backlog/430-unified-test-infrastructure-v2-target.md`
>
> The content below is retained for historical reference. All new work should follow 430v2.
>
> **Update (2026-01-13):** Configuration ownership and URL variable taxonomy is standardized in `backlog/648-config-secrets-taxonomy-ci-only-deploy.md` (KV-only secrets, ConfigMap-only settings, CI-only desired-state changes). Treat any knobs/fallbacks described below as legacy context only.

---

# backlog/371 – Playwright Auth & Runner Hardening (DEPRECATED)

**Created:** Nov 2025  
**Owner:** Platform DX / Test Infra  
**Scope:** Align Keycloak bootstrap, Auth.js config, Playwright harness, and the e2e runner with the vendor “ideal state” for deterministic, secret-safe logins.

---

## Target Outcomes

1. **Realm bootstrap & migrations**
   - Canonical realm JSON lives in `operators-config/keycloak_realm` with no inline secrets and only `${PLACEHOLDER}` references for mutable values.
   - Personas (owner/viewer/newmember) keep deterministic metadata, but passwords are provided via placeholders resolved per environment; `realmImportPlaceholders` + `ExternalSecret` wiring keep manifests secret-free.
   - Realm changes after bootstrap land as idempotent `infra/keycloak/migrations/####-*` jobs instead of re-applying the import, and every env runs a “realm ready” gate (e.g., client exists) before tests.

2. **Unified secrets guardrails**
   - Local overlays may use plaintext, but staging/prod reference `ExternalSecret` sources only; Git never stores passwords.
   - `playwright-int-tests-secrets` (or equivalent) carries persona credentials **and** `AUTH_SECRET`; overlays wire them into Keycloak placeholders, Auth.js env, and the runner.
   - Tilt resource `infra:30-integration-secrets` applies that ExternalSecret bundle automatically (no manual annotations) before Playwright runs.
   - Non-local envs fail fast if the Auth.js secret is missing; rotation is handled by updating the secret source without rebuilding images.

3. **Auth.js configuration**
   - `AUTH_URL` points to the publicly reachable host (no in-cluster-only ports); `trustHost: true` is enforced in application code (no `AUTH_TRUST_HOST` knob), and `AMEIDE_PLATFORM_BASE_URL` matches the same host the runner uses.
   - `/api/auth/providers` and `/api/auth/csrf` stay healthy; `/internal/auth-health` exposes a simple check for runner gates.
   - `AUTH_SECRET` comes from `playwright-int-tests-secrets`; services never boot without it outside local overlays.

4. **Playwright harness**
   - Global setup performs the three vendor steps: (1) browser-context health probes for `/api/auth/providers` + `/api/auth/csrf` (fail fast with actionable errors), (2) programmatic CSRF fetch + POST to `/api/auth/signin/keycloak` that follows the Keycloak redirect inside the browser context with `ignoreHTTPSErrors: true`, and (3) persists `storageState.json` for reuse.
   - UI fallback remains only after the API path fails; each Playwright worker reuses the saved session instead of repeating interactive logins mid-run.

5. **Runner guardrails**
   - Test-job-runner chart enforces `backoffLimit: 0`, sets `ttlSecondsAfterFinished`, and adds an init/gate container that polls the auth health endpoint, Keycloak `/health`, and validates Realm-client readiness before kicking off Playwright.
   - Helm/runner values guarantee `AMEIDE_PLATFORM_BASE_URL` matches `AUTH_URL`, publish Playwright HTML + JUnit artifacts, and push pod logs with concise context.
   - Tilt workflow remains hands-off: `tilt trigger infra:30-integration-secrets` materializes the Playwright ExternalSecrets (no manual annotations), `tilt trigger e2e-playwright` rebuilds the image, and `tilt trigger e2e-playwright-run` executes the job end-to-end with artifacts.

6. **Observability & rotation**
   - Auth logs include structured events around providers/CSRF (user, provider, outcome, redacted); dashboards track `/api/auth/providers`, `/api/auth/csrf`, Keycloak redirect rates, and realm health so engineers can identify the failing layer in one Grafana panel.
   - Persona secrets have a documented rotation runbook; higher envs disable or tightly scope test accounts when not in use, and secret rotation never requires manifest churn.

7. **Security & hygiene for personas**
   - Personas carry the minimum Keycloak roles required for test coverage; prod/staging prefer disabled or time-boxed accounts with rotation enforced via the ExternalSecret.
   - Even if persona credentials leak, ingress or feature flags ensure the blast radius stays near zero.

8. **Repo shape & contracts**
   - Repo layout follows the vendor checklist (`infra/kubernetes/charts/operators-config/keycloak_realm`, `infra/keycloak/migrations`, environment overlays, `services/www_ameide_platform/tests/{setup,README}`) so contributors know where to touch.
   - Day-to-day loop stays documented: (1) `tilt trigger e2e-playwright`, (2) `tilt trigger e2e-playwright-run`, (3) observe gates → global setup → tests → artifacts, (4) remediate via auth-health if redirect never happens, (5) re-trigger.

---

## Execution Tracker

| Task | Owner | Status |
| --- | --- | --- |
| Replace inline Keycloak passwords with `${PLACEHOLDER}` + hook up `realmImportPlaceholders` |  | ☑ |
| Introduce `infra/keycloak/migrations/` skeleton + first example job |  | ☑ |
| Provision `playwright-int-tests-secrets` via ExternalSecret (Vault-backed) and wire env overlays; the Keycloak realm chart must not create this secret (personaSecret.create=false) |  | ☑ |
| Enforce `AUTH_SECRET` sourcing via ExternalSecret + non-local boot guard |  | ☑ |
| Update Auth.js env (`AUTH_URL`, `AUTH_TRUST_HOST`) + `/internal/auth-health` route |  | ☑ |
| Refactor `tests/setup/global-setup.ts` to vendor browser-context flow + storage-state reuse + TLS ignore |  | ☑ |
| Enhance `test-job-runner` with ttl/backoff/gates; expose auth health + realm-ready init container; publish artifacts |  | ☑ |
| Add dashboards/log guidance, persona hygiene, and repo contracts to README/backlog |  | ☑ |
| Prove deterministic Keycloak redirect (`/login-actions/authenticate` fallback ready, no manual cleanup) |  | ☐ |

--- 

## Notes

- Aligns directly with backlog/362 (Unified Secret Guardrails); no inline credentials remain once this backlog lands.
- Vendor reference: Auth.js/Playwright Keycloak recommendations (Nov 2025) – stored in `backlog/371 refs/`.
- Persona rotation steps now live in `services/www_ameide_platform/tests/README.md`; use the `playwright-int-tests-secret-sync` ExternalSecret to rotate credentials per environment.
- Runner metrics can be pushed to Prometheus via `INTEGRATION_RUNNER_PUSHGATEWAY`, providing duration/success signals for dashboards that track `/api/auth/providers` + `/api/auth/csrf` health probes.
- Repo scaffolding described in “Repo shape & contracts” lives under `infra/` + `services/www_ameide_platform/tests/`; new migrations or overlays must follow that layout so the Tilt loop stays deterministic.
- Nov 26 2025 update: the bootstrap fixtures now publish a 48-character `www-ameide-platform-auth-secret`, `scripts/vault/ensure-local-secrets.py` validates the minimum length before writing secrets, and Tilt renders the shared `integration-secrets` overlay so local runs consume the same ExternalSecrets (including `playwright-int-tests-secret-sync`) that ArgoCD uses. This closes the auth-health regression where `/api/auth/providers` crashed due to a too-short Auth.js secret.
- Current gap (Nov 2025): Login stability is blocked mainly by the deterministic Keycloak redirect contract described in “Playwright harness”. Continue monitoring the API flow; if failures resurface, focus on the `login-actions/authenticate` POST fallback.
