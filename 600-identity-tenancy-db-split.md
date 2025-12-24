# 600: Option B — Split Identity + Tenancy catalogs into dedicated Postgres databases (CNPG)

**Status:** Proposed  
**Audience:** Platform / Identity / Tenancy owners; GitOps + DB migrations owners  
**Scope:** GitOps + migrations + cutover plan for moving Identity/Tenancy catalog persistence out of the `platform` database into **two dedicated databases** (`identity`, `tenancy`) in the existing CNPG cluster.

---

## Why this exists

Backlog `597-login-onboarding-primitives.md` defines Identity Catalog + Tenancy Catalog as **separate Domain primitives** with explicit responsibilities, but the current GitOps topology provisions only a single `platform` database for the platform catalog tables and seeds `issuer → tenant_id` into that store.

This backlog tracks **Option B**: enforce the per-domain persistence boundary by splitting the current platform catalog into:

- **`identity` DB**: canonical users + external identity links (`(issuer, subject) → user_id`)
- **`tenancy` DB**: tenants/orgs/memberships + routing view (`issuer → tenant_id`)

This is not required for 597 functional correctness, but is a structural alignment step to reduce coupling and make ownership boundaries enforceable.

---

## Constraints (non-negotiable)

- Must remain **fully aligned to 597**: issuer-first routing is authoritative; no `tenantId` claim authority; no compatibility shims that re-introduce tenantId-driven routing.
- GitOps must not introduce tenant-specific overlays/apps or tenant secrets in Git (see 597).
- Keep CNPG as the credential authority model in GitOps (see `412-cnpg-owned-postgres-greds.md`).
- Flyway remains **schema-only**; seed/demo data stays in GitOps seed jobs (see `300-400/363-migration-architecture.md`).

---

## Current state (GitOps reality)

- CNPG cluster `postgres-ameide` provisions multiple databases including `platform`, but **no** `identity` or `tenancy` databases yet (`sources/values/_shared/data/platform-postgres-clusters.yaml`).
- `platform` database currently acts as the landing zone for:
  - seeded personas and access seed state
  - issuer routing seed (`issuer → tenant_id`) required by 597 login resolution

---

## Decision

Adopt Option B:

1. Add `identity` and `tenancy` databases (and app roles/secrets) to the CNPG cluster.
2. Add new schema-only migrations for these databases.
3. Migrate/cut over runtime reads/writes away from `platform` DB for identity/tenancy concerns.
4. Update seed jobs so 597-required routing + personas are seeded into the new canonical stores.

Non-goal: introduce new “N tenant realms in GitOps” or tenant-specific GitOps overlays.

---

## Deliverables (GitOps-owned)

### A) CNPG databases + credentials

- Add CNPG `Database` resources for:
  - `identity`
  - `tenancy`
- Add CNPG managed roles + Secrets for:
  - `identity-db-credentials` (role `identity`)
  - `tenancy-db-credentials` (role `tenancy`)
- Ensure each database has `public` schema owned by its role (align with existing pattern).

### B) Migrations wiring (schema-only)

- Extend `sources/charts/platform/db-migrations` to migrate the two new databases:
  - require the new secrets in the `existingSecrets` list
  - run `flyway migrate` for `identity` and `tenancy` targets
- Add new migration trees:
  - `schema-only/sql/identity/**`
  - `schema-only/sql/tenancy/**`

### C) Seed updates (dev/local only, schema-only unchanged)

- Move 597 seed responsibilities:
  - `(issuer, subject) → user_id` links → `identity` DB
  - `issuer → tenant_id` routing view + tenant/org/membership seed → `tenancy` DB
- Keep any remaining platform-only catalog seed (if any) in `platform` DB.

### D) Smokes / verification

- Add/extend smokes so GitOps fails fast when:
  - `identity` DB credentials exist and connect
  - `tenancy` DB credentials exist and connect
  - migrations for both DBs have run to the expected baseline version

---

## Deliverables (runtime-owned, tracked here for completeness)

### E) Cutover plan (minimal risk)

1. **Prepare**: Deploy DBs + migrations first (no runtime cutover).
2. **Backfill**: Run a one-time migration job to copy relevant tables/data from `platform` DB into `identity`/`tenancy` DBs.
3. **Cutover**: Switch runtime config so Identity/Tenancy primitives read/write only the new DBs.
4. **Lockdown**: Remove writes to legacy tables in `platform` DB (drop permissions or block at app layer).
5. **Cleanup**: Optionally remove legacy tables from `platform` DB after a soak period.

Implementation note: avoid dual-writes unless strictly necessary; prefer a short maintenance window with deterministic backfill + cutover.

---

## Data model (minimum viable, 597-aligned)

This section is intentionally minimal: it captures only what 597 requires GitOps to support.

### `identity` DB

- `users` (canonical, opaque `user_id`; email is an attribute, not an identifier)
- `external_identities`:
  - `issuer` (URL, opaque)
  - `subject`
  - `user_id` (FK to `users`)

### `tenancy` DB

- `tenants` (canonical `tenant_id`, `tenant_slug`, status)
- `tenant_routing_view` (or equivalent):
  - unique `issuer` → `tenant_id` (+ optional `realm_name` ops attribute)
- `organizations`
- `memberships`
- `tenant_aliases` (legacy tenant id → canonical tenant id), if needed for migrations

---

## Risks / tradeoffs

- Cross-database joins disappear; runtime must fetch identity + memberships via two calls (which is architecturally fine but must be accounted for).
- More DB secrets + migrations targets increase operational surface area (more ways for GitOps to fail early; that’s a feature, but requires better smokes).
- Backfill/cutover must be deterministic to avoid “issuer routing mismatch” regressions (597 blocks in-shell if routing view incomplete).

---

## Acceptance (definition of done)

- GitOps provisions `identity` and `tenancy` DBs and credentials in all environments.
- Schema-only migrations run successfully for both DBs.
- Dev/local seeding writes:
  - external identity links to `identity`
  - issuer routing view + tenant/org/membership seed to `tenancy`
- 597 baseline smokes still pass (Keycloak realm import + platform-auth-smoke).
- Login routing remains issuer-first and does not depend on tenantId token claims.

---

## References

- `backlog/597-login-onboarding-primitives.md` (required invariants + GitOps baseline)
- `backlog/428-onboarding.md` (realm-per-tenant + issuer-first onboarding constraints)
- `backlog/412-cnpg-owned-postgres-greds.md` (CNPG credential authority model)
- `backlog/300-400/363-migration-architecture.md` (Flyway schema-only + seed jobs ownership)
- `backlog/472-ameide-information-application.md` (per-domain SQL stores expectation)
