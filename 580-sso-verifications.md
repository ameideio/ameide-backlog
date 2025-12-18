# 580 — SSO Verifications (ArgoCD + Platform)

**Status**: 🟡 Draft (operational runbook)  
**Created**: 2025-12-17  
**Scope**: `local` + `azure` (`dev`, `staging`, `production`)  
**Related**: [525-backlog-first-triage-workflow.md](525-backlog-first-triage-workflow.md), [450-argocd-service-issues-inventory.md](450-argocd-service-issues-inventory.md), [485-keycloak-oidc-client-reconciliation.md](485-keycloak-oidc-client-reconciliation.md), [460-keycloak-oidc-scopes.md](460-keycloak-oidc-scopes.md), [451-secrets-management.md](451-secrets-management.md), [519-gitops-fleet-policy-hardening.md](519-gitops-fleet-policy-hardening.md)

---

## Goal

Define the **minimum, deterministic checks** to validate:

1. ArgoCD public endpoints are reachable and correctly routed.
2. Dex is initialized and can reach Keycloak.
3. Keycloak accepts ArgoCD’s OIDC request (redirect URIs + scopes).
4. ArgoCD has the correct Dex client secret (Vault → ESO → `argocd-secret`).
5. End-to-end SSO login works without manual cluster poking.
6. `www.{env}.ameide.io` login routes to `platform.{env}.ameide.io` and Platform Auth.js can reach Keycloak (redirect_uri + scopes + CSRF).

This backlog is intentionally “standard/unopinionated”: it verifies vendor-aligned behavior and avoids band-aids.

## Policy (non-negotiable): runtime configuration only

- **Configuration must not be baked into images.** Images must be build-once/run-anywhere across `dev/staging/production` for the same code revision.
- Environment-specific values (hosts, issuers, redirect URIs, cookie domains) must be injected at runtime via Kubernetes (`ConfigMap`/`Secret`) and must be observable from the running Pods.
- If `www` renders an incorrect login link while runtime values are correct, treat it as an application build/runtime-config defect (e.g., `NEXT_PUBLIC_*` used only at build time). Fix in the app so runtime config controls the rendered link.

---

## Environment Model (namespaces)

ArgoCD runs in a **cluster-shared** namespace:
- `argocd` (ArgoCD + Dex + Envoy for `argocd.*`)

Keycloak is **per environment namespace**:
- Azure: `ameide-dev`, `ameide-staging`, `ameide-prod`
- Local: `ameide-local`

SSO endpoints (canonical issuers):
- Azure prod: `https://auth.ameide.io/realms/ameide`
- Azure dev: `https://auth.dev.ameide.io/realms/ameide`
- Azure staging: `https://auth.staging.ameide.io/realms/ameide`
- Local: `https://auth.local.ameide.io/realms/ameide`

ArgoCD endpoints:
- Azure prod: `https://argocd.ameide.io`
- Local: `https://argocd.local.ameide.io`

---

## Primary “All Green” Gate (recommended)

Run the repo-provided end-to-end verifier:

```bash
infra/scripts/verify-argocd-sso.sh --all
```

What it validates:
- OIDC discovery contains expected scopes (`profile`, `email`).
- Keycloak accepts the ArgoCD auth request for `client_id=argocd` (no `invalid_scope` / `invalid_request`).
- `SecretStore/ameide-vault` is Ready in `argocd`.
- `ExternalSecret/argocd-secret-sync` is Ready in `argocd`.
- `argocd/argocd-secret` contains a **non-placeholder** `dex.keycloak.clientSecret` (printed as sha256 only).
- Full browser-style login: ArgoCD → Dex → Keycloak → ArgoCD session established.

If this passes, SSO is considered healthy for both local and azure.

---

## Platform SSO Gate (www → platform → Keycloak)

This gate validates that the browser flow works for:

- `https://www.{env}.ameide.io/` (login button/link)
- `https://platform.{env}.ameide.io/login` (Auth.js → Keycloak)

Recommended verifier:
```bash
infra/scripts/verify-platform-sso.sh --azure
```

### A) `www` login link correctness

Expected:
- `www.{env}` renders a login link to `https://platform.{env}.ameide.io/login`

Smoke:
```bash
curl -fsSL "https://www.ameide.io/" | head -c 200000 | rg -o 'href="[^"]*login[^"]*"' | head
curl -fsSL "https://www.staging.ameide.io/" | head -c 200000 | rg -o 'href="[^"]*login[^"]*"' | head
curl -fsSL "https://www.dev.ameide.io/" | head -c 200000 | rg -o 'href="[^"]*login[^"]*"' | head
```

Failure mapping:
- `href="//login"` → broken link construction in `www-ameide` (often caused by `NEXT_PUBLIC_*` being build-time-only in `next start` images; runtime `ConfigMap` changes won’t affect compiled client code).

### B) Keycloak accepts Platform client request (redirect_uri + scopes)

Expected:
- Keycloak should not return `invalid_scope` or “Invalid parameter: redirect_uri” for `client_id=platform-app`.

Smoke (prod example):
```bash
curl -sS -D - -o /dev/null \
  'https://auth.ameide.io/realms/ameide/protocol/openid-connect/auth?client_id=platform-app&redirect_uri=https%3A%2F%2Fplatform.ameide.io%2Fapi%2Fauth%2Fcallback%2Fkeycloak&response_type=code&scope=openid+profile+email&state=probe' | \
  rg -n '^(HTTP/|location:|set-cookie:)'
```

Failure mapping:
- `error=invalid_scope` → Platform client scopes not attached to `platform-app` (see 460, 485).
- “Invalid parameter: redirect_uri” → `platform-app` missing `platform.{env}` callback allowlist (see 485).

### C) Auth.js CSRF + sign-in initiation (Platform)

Expected:
- `GET /api/auth/csrf` returns `200` and sets CSRF cookies for the correct domain.
- `POST /api/auth/signin/keycloak` returns `302` to Keycloak auth URL (not `MissingCSRF`).

Smoke (staging example):
```bash
tmp="$(mktemp -d)"; jar="$tmp/cookies"; csrf="$tmp/csrf.json"
curl -sS -c "$jar" -o "$csrf" 'https://platform.staging.ameide.io/api/auth/csrf'
token="$(jq -r '.csrfToken' <"$csrf")"
curl -sS -b "$jar" -c "$jar" -D - -o /dev/null \
  -X POST 'https://platform.staging.ameide.io/api/auth/signin/keycloak' \
  -H 'content-type: application/x-www-form-urlencoded' \
  --data-urlencode "csrfToken=$token" \
  --data-urlencode 'callbackUrl=https://platform.staging.ameide.io/' \
  --data-urlencode 'json=true' | rg -n '^(HTTP/|location:|set-cookie:)'
rm -rf "$tmp"
```

Failure mapping:
- `.../login?error=MissingCSRF` → `AUTH_COOKIE_DOMAIN` mismatch (cookies ignored by browser). Fix `auth.cookies.domain` in the Platform values for that env.
- `HTTP 500` on `/api/auth/csrf` (or `/login`) → Platform runtime failure (app crash/misconfig). Check `kubectl -n ameide-{env} logs deploy/www-ameide-platform` and `kubectl -n ameide-{env} describe pod` for the failing container, then fix the underlying app/config issue.

---

## Fast Smoke (no browser, no secrets)

### A) ArgoCD + Dex public reachability

Azure:
```bash
curl -fsS https://argocd.ameide.io/ >/dev/null
curl -fsS https://argocd.ameide.io/api/dex/.well-known/openid-configuration >/dev/null
```

Local (host routing via Envoy LB; use `verify-argocd-sso.sh --local` to avoid manual `--resolve` wiring).

Expected:
- `200` for `/` and `.../openid-configuration`

Failure mapping:
- `502` on `/api/dex/...` often means Dex didn’t start or ArgoCD server can’t proxy to Dex. See 450 “Dex upstream Keycloak issuer misconfigured”.

### B) Keycloak accepts ArgoCD auth request (scopes + redirect_uri)

Azure prod example:
```bash
curl -fsS -o /dev/null \
  'https://auth.ameide.io/realms/ameide/protocol/openid-connect/auth?client_id=argocd&redirect_uri=https%3A%2F%2Fargocd.ameide.io%2Fapi%2Fdex%2Fcallback&response_type=code&scope=openid+profile+email+groups'
```

Expected:
- HTTP `200` login form (or redirect to login page), not an error page.

Failure mapping:
- “Invalid parameter: redirect_uri” → Keycloak client `argocd` missing Dex callback URL(s). See 450.
- `error=invalid_scope` → missing realm/client scope linkage. See 450, 460, 485.

---

## Secret Sync Gate (Dex client secret correctness)

Dex token exchange failures are frequently caused by **mismatched client secrets**.

### Required resources

Both local and azure should have, in namespace `argocd`:
- `SecretStore/ameide-vault` (Ready)
- `ExternalSecret/argocd-secret-sync` (Ready)
- `Secret/argocd-secret` containing:
  - `dex.keycloak.clientSecret`
  - `dex.oauth.clientSecret` (same value)

Check readiness (no secret values printed):
```bash
kubectl -n argocd get secretstore ameide-vault
kubectl -n argocd get externalsecret argocd-secret-sync
kubectl -n argocd describe externalsecret argocd-secret-sync | rg -n 'Ready|SecretSynced|Error'
```

Failure mapping:
- `SecretSyncedError could not get secret data from provider` → Vault/ESO wiring issue (451) or missing Vault key.
- Dex logs: `unauthorized_client` → ArgoCD is using the wrong secret; ensure `argocd-secret-sync` is Ready and restart Dex. See 450.

Dex log check:
```bash
kubectl -n argocd logs deploy/argocd-dex-server --tail=200 | rg -n 'unauthorized_client|invalid_client|failed to get token' || true
```

---

## ArgoCD CLI SSO verification (common pitfall)

Symptom:
- `http: named cookie not present`

Cause:
- `argocd login --sso` defaults to an **HTTP** callback (`http://localhost:8085/...`), but session cookies are `Secure`, so the browser won’t send them over HTTP.

Fix (use HTTPS callback; `--callback` is scheme/host/port only):
```bash
argocd login argocd.ameide.io --grpc-web --sso --sso-launch-browser=false --callback https://localhost:8085
```

Local:
```bash
argocd login argocd.local.ameide.io --grpc-web --insecure --sso --sso-launch-browser=false --callback https://localhost:8085
```

---

## What “credentials” are expected (avoid confusion)

There are two independent login paths:

1. **ArgoCD local account** (break-glass): `admin` (password may be random/bootstrapped).
2. **Keycloak SSO users** (seeded personas, deterministic for verification):
   - `admin@ameide.io` (admin) — password comes from `Secret/playwright-int-tests-secrets` key `E2E_SSO_PASSWORD`
   - `user@ameide.io` (readonly) — password comes from `Secret/playwright-int-tests-secrets` key `E2E_VIEWER_PASSWORD`

Prefer SSO for normal use; treat ArgoCD local admin as break-glass.

---

## Troubleshooting Decision Tree (by error string)

1. **`Invalid parameter: redirect_uri`** (Keycloak UI)
   - Fix Keycloak client `argocd` redirect URIs (`/api/dex/callback`) in reconciliation. See 450.

2. **`invalid_scope`** (Keycloak redirect / Dex error page)
   - Ensure realm has built-in scopes (`profile`, `email`) and the `argocd` client is linked to them.
   - Linkage uses dedicated endpoints (see 485). See also 460.

3. **Dex logs `unauthorized_client`**
   - `argocd/argocd-secret` has wrong `dex.keycloak.clientSecret`.
   - Verify `argocd-secret-sync` Ready; restart Dex. See 450.

4. **ArgoCD CLI `http: named cookie not present`**
   - Use HTTPS callback for CLI SSO. See section above.

---

## Acceptance Criteria

SSO is “verified” when:
- `infra/scripts/verify-argocd-sso.sh --all` succeeds, and
- Dex logs contain no new `unauthorized_client` / `invalid_scope` errors during a login attempt, and
- `https://argocd.ameide.io/api/dex/.well-known/openid-configuration` returns `200`.
