# 582 – Local Dev Seeding (Dev Data Contract)

**Status:** Implemented (local/dev GitOps)  
**Owner:** Platform / Developer Experience  
**Motivation:** Local k3d + remote-first dev must have deterministic, reproducible “dev data” without code-level fallbacks (“magic”).  
**Related:** [444-terraform.md](444-terraform.md), [492-telepresence-reliability.md](492-telepresence-reliability.md), [492-telepresence-verification.md](492-telepresence-verification.md), [300-400/330-dynamic-tenant-resolution.md](300-400/330-dynamic-tenant-resolution.md), [427-platform-login-stability.md](427-platform-login-stability.md), `../gitops/ameide-gitops/backlog/485-keycloak-oidc-client-reconciliation.md`, `../db/flyway/sql/platform/*`, `../gitops/ameide-gitops/sources/values/**`

**Implementation summary (GitOps):**

- Chart: `../gitops/ameide-gitops/sources/charts/platform/dev-data/` (PostSync seed + PostSync verify Jobs).
- Local ArgoCD app: `local-platform-dev-data` (validated in `ameide-local` cluster; seed + verify succeed and are idempotent across re-syncs).

---

## 0. Verification (Evidence, 2025-12-22)

Validated in `ameide-local`:

- Argo apps sync green: `local-data-db-migrations`, `local-platform-dev-data`.
- Hook jobs succeed:
  - `data-db-migrations` (PreSync) completes successfully.
  - `platform-dev-data-dev-data-seed` then `platform-dev-data-dev-data-verify` (PostSync) both succeed and remain idempotent across repeated re-syncs.
- Platform DB contract check: seeded personas exist in `platform.users` and have expected role codes in `platform.organization_memberships` for org `atlas`.

**Migration note:** environments created before “Flyway schema-only” may retain legacy demo rows (e.g. `atlas-org` tenant/users) from historical Flyway seeds. To fully converge to the target-state dataset, wipe/recreate the local Postgres volumes/cluster once, then let `platform-dev-data` seed from scratch.

---

## Addendum (2025-12-30): “Seeded users must be complete” + local stability prerequisites

This addendum preserves the original intent of 582 and records a concrete failure mode we hit in local:

### Seeded personas must include platform DB facts (not just Keycloak)

If a persona can authenticate in Keycloak but does not have the expected **platform DB** tenant/org/memberships, the UI will (correctly) land in “no access / onboarding lane” flows.

This is not a bug in login; it is a missing seed contract fact.

Concrete expectation (baseline):
- A seeded “happy path” persona should already have:
  - canonical `platform.users` row
  - tenant + org membership(s) in the target org (e.g. `atlas`)
  - role codes consistent with the RBAC catalog

### Local cluster must run `www-ameide-platform` as a production image (Argo-managed)

We saw `upstream request timeout` / 500s on `/login`, `/accept`, and `/api/auth/*` even while unit tests were green.
Root cause: local Argo was temporarily pinned to a **dev-mode image** (`Dockerfile.dev`) while `NODE_ENV=production`, causing Next dev-server build/runtime failures behind the gateway.

Policy:
- For Argo-managed local (`ameide-local`), `www-ameide-platform` must run the **production** image (`Dockerfile.main`) pinned by digest.
- Tilt/dev images are fine for developer loops, but should not be the baseline behind the shared local gateway that E2E hits.

GitOps anchor (local):
- `../gitops/ameide-gitops/sources/values/env/local/apps/www-ameide-platform.yaml` (pins prod digest; see `ameideio/ameide-gitops#53`)

### Server-to-server calls must not use browser/public URLs

We also saw onboarding bootstrap report “Tenant provisioning is currently unavailable” when the server-side SDK tried to call the **public** Envoy URL instead of the in-cluster gRPC base URL.

Policy:
- **Update (648):** cluster deployments must not rely on `NEXT_PUBLIC_*` env vars for environment-specific URLs.
- Browser/client: use same-origin BFF routes (`/api/*`) and runtime-provided config (no compile-time `NEXT_PUBLIC_*`).
- Server-to-server: use `AMEIDE_GRPC_BASE_URL` for internal gRPC upstream calls.

Code anchor:
- `../services/www_ameide_platform/lib/sdk/server-client.ts` (base URL precedence)

## 1. Problem

We want **fully deterministic developer environments** (local k3d + shared AKS namespaces) where:

- A new developer can bootstrap the cluster and immediately log in with known personas.
- Authorization works because **platform memberships/roles** exist (not just Keycloak users).
- Telepresence workflows have the required runtime env values (org slug, tenant id) and can verify them deterministically.
- There are **no hidden defaults/fallbacks in application code** that mask missing data or configuration.

Today, “dev data” is split across multiple places with drift risk:

- **Terraform** (444) correctly focuses on infra + secrets, not application data.
- **Keycloak realm import** currently includes test personas in shared values, and realm import is create-only (no reconciliation).
- **Platform DB seeds** exist but have conflicting/duplicated role models (`admin/contributor/viewer/...` vs `owner/member`).
- Some services/SDKs still carry **hardcoded tenant defaults** (the very “magic” we want to remove).

---

## 2. Goals / Non-goals

### Goals

1. **Single, executable definition of “dev env is ready”** (the Dev Data Contract).
2. **Idempotent seeding** that can be run repeatedly (GitOps self-heal) and produces the same state.
3. **Read-only verification** that fails Argo sync (or CI) when the contract is not met.
4. **No app-code fallbacks** for tenant/org in cluster mode; missing dev data should fail fast with actionable errors.
5. **Environment scoping**: test personas are allowed in dev/local (optionally staging), and **not** in production.

### Non-goals

- Creating a second configuration system/DSL that duplicates “source of truth” already represented by the DB schema, Keycloak, or code.
- Using Terraform to insert business/application data into platform DB.
- Shipping large demo datasets or workflows in the contract (keep it small: “must-exist facts” only).

---

## 3. Current State (Evidence)

### Keycloak (personas + create-only import)

- Shared realm import defines realms/groups/roles and service accounts, but **does not** ship test personas anymore:  
  `../gitops/ameide-gitops/sources/values/_shared/platform/platform-keycloak-realm.yaml`
- Personas are seeded/reconciled via a GitOps hook Job (local/dev only):  
  `../gitops/ameide-gitops/environments/local/components/platform/auth/dev-data/component.yaml`
- Keycloak Operator realm import is **create-only**; existing realms are not updated. We already had to add a reconciler pattern for OIDC clients (see 485):  
  `../gitops/ameide-gitops/backlog/485-keycloak-oidc-client-reconciliation.md`

### Platform authorization depends on platform memberships

- `www-ameide-platform` checks org membership via `listOrganizationMemberships` (platform DB), not Keycloak realm roles:  
  `../services/www_ameide_platform/lib/auth/middleware/authorize.ts`

### DB baseline seed ambiguity

- (Historical) Flyway `V1__initial_schema.sql` and `R__seed_baseline_data.sql` seeded a default tenant/org/roles/memberships (and additional demo data), which created drift against the GitOps dev-data contract.
- **Update (2025-12-22):** GitOps now treats Flyway as **schema-only**; baseline/persona data is owned by `platform-dev-data` seed/verify Jobs (local/dev only).

### Code “magic” defaults still exist

- TypeScript SDK uses `DEFAULT_TENANT_ID ?? 'tenant-atlas'`:  
  `../packages/ameide_sdk_ts/src/platform/organization-client.ts`
- Inference service falls back to `DEFAULT_TENANT_ID="default"`:  
  `../services/inference/src/inference_service/context.py`

---

## 4. Policy: What owns dev data?

### Terraform (444) owns infra + secret plumbing

- AKS/k3d clusters, DNS, managed identities, KeyVault/Vault plumbing, and producing deterministic bootstrap outputs.
- Terraform **must not** become the “dev data insertion” system.

### GitOps owns “dev data”

GitOps is the right control plane for deterministic, repeatable, environment-scoped seeds because:

- Argo CD can order resources (sync waves) and run deterministic hook Jobs.
- Seeding can be *idempotent* and *self-healing* (run again on drift).
- Verification can be enforced as a gate (fail sync if missing).

---

## 5. The Dev Data Contract (keep it small)

The contract is a short list of **assertions** (must-exist facts), not “how-to” logic.

Contract should specify:

- Tenant(s)/org(s) required for local/dev.
- Personas (email/username) required for local/dev, and their required **platform** roles/memberships.
- Optional Keycloak facts: issuer URL / realm name, required realm roles, and persona credentials (via ExternalSecrets).

### Canonical baseline (recommended)

Keep the baseline dataset intentionally boring and stable:

- **Tenant:** `tenant-atlas`
- **Org:** `atlas` (platform organization key)
- **Canonical role codes:** `admin`, `contributor`, `viewer`, `guest`, `service` (avoid introducing `owner/member`)
- **Deterministic identity links:** seed platform users with opaque IDs (UUID) and link personas via external identity mapping `(issuer, sub) → user_id` so login can resolve memberships reliably (see `backlog/597-login-onboarding-primitives.md`).

### Persona set (more comprehensive, still practical)

Each persona that should “work” must exist in **both** systems:

- **Keycloak**: user exists in the tenant realm (issuer) and has a known password (via ExternalSecret).
- **Platform DB**: user exists (opaque ID) and has the expected membership state + role codes in org `atlas`, plus an external identity link `(issuer, sub) → user_id`.

Recommended personas:

- **Baseline (always seeded in `local` + `dev`)**
  - `admin@ameide.io` → `admin`
  - `contributor@ameide.io` → `contributor`
  - `viewer@ameide.io` → `viewer`
  - `guest@ameide.io` → `guest`
  - `service@ameide.io` → `service` (automation/service-path coverage)
- **E2E (seeded only where Playwright runs)**
  - `owner@acme.test` → `admin`, `contributor`
  - `viewer@acme.test` → `viewer`
  - `newmember@example.com` → `guest` (used for invitation/onboarding flows)
- **State / negative coverage (optional; enable when needed)**
  - `invited@ameide.io` → state `INVITED` (e.g., role `viewer`)
  - `suspended@ameide.io` → state `SUSPENDED` (e.g., role `viewer`)
  - `nomember@ameide.io` → platform user exists, **no membership** (must hit `Forbidden` paths deterministically)

If we later need org-switching coverage, add a second org (same tenant) and one multi-org persona; don’t introduce a second tenant/realm unless we’re actively validating multi-tenant isolation.

### Proposed contract format (example)

Keep it under ~30 lines and store it in Git (rendered into the verify Job).

```yaml
tenant:
  id: tenant-atlas
  slug: atlas
  issuer: https://auth.local.ameide.io/realms/atlas
organization:
  id: atlas
  slug: atlas
personas:
  baseline:
    - email: admin@ameide.io
      platformRoles: [admin]
    - email: contributor@ameide.io
      platformRoles: [contributor]
    - email: viewer@ameide.io
      platformRoles: [viewer]
    - email: guest@ameide.io
      platformRoles: [guest]
    - email: service@ameide.io
      platformRoles: [service]
  e2e:
    - email: owner@acme.test
      platformRoles: [admin, contributor]
    - email: viewer@acme.test
      platformRoles: [viewer]
    - email: newmember@example.com
      platformRoles: [guest]
keycloak:
  issuer: https://auth.local.ameide.io/realms/atlas
  realmName: atlas
```

Notes:
- Contract can name **role codes** (e.g., `admin`) rather than DB role IDs; the seed job resolves IDs by querying the DB.
- Repo/mock mode (“Mock Organization”) is **not** part of this contract; it’s a local test harness.

---

## 6. Implementation (GitOps): Seed + Verify Jobs

Argo CD supports deterministic ordering via sync waves and explicit hook phases.

### Pattern

1. **Seed Job (PostSync, wave +10)**  
   - Idempotently creates/updates anything missing to satisfy the contract.
   - Should be enabled only for `local` + `dev` (optionally `staging`).

2. **Verify Job (PostSync, wave +20)**  
   - Read-only assertions; exits non-zero if contract is not met.
   - Must run in the same environments where we expect deterministic dev data.

### Why Jobs (not docs)

- The contract is executable and therefore cannot “drift” silently.
- Verification failures show up as Argo sync failures (actionable signal).

### Where to implement

Add a small chart (or manifests via shared-values) in `ameide-gitops`, e.g.:

- `../gitops/ameide-gitops/sources/charts/platform/dev-data/` (implemented), applied as `platform-dev-data` in `env/local` + `env/dev`.

### Seed strategy (DB vs API)

**Preferred:** seed business state via platform APIs.  
**Reality today:** platform lacks a “create membership” endpoint; we only have list/update. Options:

1. **Short-term:** Seed via SQL (psql) using a DB admin/maintenance credential (scoped to local/dev namespaces). **Implemented** in `platform-dev-data` (seed + verify hooks).
2. **Mid-term:** Add platform service APIs for:
   - `upsertUser`
   - `createOrganizationMembership`
   - `assignOrganizationRoles`
   Then migrate the seed job to call platform APIs instead of direct SQL.

Verification should still read through:
- platform APIs (preferred), or
- direct SQL (acceptable if scoped + read-only).

---

## 7. Keycloak Personas: prevent “soup” + prevent prod leakage

### Keycloak import reality

Realm import is create-only, so “declared users” in the import are not a reconciliation mechanism. If we need ongoing declarative behavior, we must run a reconciler Job (same principle as the OIDC client-patcher).

### Recommendation

1. **Remove Playwright/test users from realm import** and move them behind environment flags (dev/local/staging only).
2. Implement a **persona reconciler Job** (idempotent) that ensures:
   - users exist by email,
   - users exist in the correct realm (issuer),
   - external identity links exist in the platform DB (`(issuer, sub) → user_id`),
   - and passwords are sourced from `playwright-int-tests-secrets` (already delivered via ExternalSecrets).
3. **Production:** do not include the persona secret or persona users by default.

If we must keep a production “break-glass” account, treat it as a separate, explicitly managed path (not shared defaults).

**Status update (GitOps):**
- Test personas removed from realm import; `platform-dev-data` seeds/verifies Keycloak personas in `local`/`dev`.
- `platform-dev-data` also seeds/verifies **platform DB**: tenant/org, users, and org memberships for the contract personas (local/dev only).
- `playwright-int-tests-secrets` is now env-scoped (empty in `_shared`, only present in `env/local` + `env/dev`).

---

## 8. Eliminate code-level fallbacks (“no magic”)

Principle: **cluster mode must fail fast** if tenant/org context is missing. Repo/mock mode can keep fixtures.

Targets to remove/restrict:

- SDK tenant fallback (`'tenant-atlas'`) → remove; in cluster mode, tenant context is derived from `x-issuer` and a server-side `issuer → tenant_id` mapping (tests/mocks can still inject explicit context).
- Inference default tenant (`"default"`) → require env var or explicit context; missing config should fail at startup (with a clear error).

Related policy: [300-400/330-dynamic-tenant-resolution.md](300-400/330-dynamic-tenant-resolution.md) already describes the migration goal: “no hardcoded tenant defaults anywhere”.

---

## 9. DB baseline cleanup (avoid conflicting role models)

Decide the canonical role codes and ensure seeds/migrations do not fight each other.

Proposed direction:

- Keep canonical org role codes: `admin`, `contributor`, `viewer`, `guest`, `service` (already used in DB baseline and Keycloak realm roles).
- Treat any “owner/member” demo roles as legacy or convert them to canonical codes (do not overwrite memberships unexpectedly).
- Repeatable seeds should add demo content only and must not redefine the platform’s base role model.

---

## 10. Rollout Plan (safe + measurable)

1. **Decide persona scope** (dev/local only vs dev+staging; never prod by default).
2. **Add Verify Job first** (read-only) to establish signal without changing state.
3. **Add Seed Job** (idempotent) and gate it to local/dev.
4. **Update Telepresence verification** to assert contract readiness beyond env vars (link to 492).
5. **Remove code fallbacks** once seed+verify are in place.
6. **Clean DB seeds** to remove role-model conflicts.

---

## 11. Acceptance Criteria

1. Fresh `../gitops/ameide-gitops/infra/scripts/deploy.sh local` (k3d) reaches a state where:
   - platform login works for required personas,
   - platform APIs succeed for org-scoped routes (memberships exist),
   - seed job is idempotent (safe to re-run),
   - verify job passes (contract satisfied).
2. Remote-first dev namespace (`ameide-dev`) also satisfies the contract (if enabled).
3. Production does not ship default test personas or their secrets.
4. No service relies on a hardcoded tenant/org fallback in cluster mode.

---

## 12. Open Questions (decisions required)

1. **Do Playwright personas exist in staging?** (CI convenience vs security posture)
2. **Do we accept direct SQL seeding for now, or prioritize adding platform APIs for memberships?**
3. **What is the canonical role model?** (`admin/contributor/viewer/...` vs `owner/member`)
4. **Do we want the contract to cover Keycloak group membership, or rely purely on platform memberships?**
