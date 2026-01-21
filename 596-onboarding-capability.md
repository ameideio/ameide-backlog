Onboarding v2: Primitive Decomposition + Non‑Happy Path Hardening + Integration Test Pack

**Status:** Proposed
**Audience:** Platform + Identity + Tenancy owners; capability/test‑infra
**Scope:** Make onboarding robust under non-happy paths by decomposing it into primitives with explicit contracts and by shipping a repo-root integration test pack that gates end-to-end onboarding behavior.

## Use with

* `backlog/428-onboarding.md` (current onboarding semantics + flow)
* `backlog/300-400/333-realms.md` (realm-per-tenant reference design + invariants)
* `backlog/520-primitives-stack-v6.md` (stack invariants: operators + proto/codegen + CI gates; proto as behavior schema) 
* `backlog/533-capability-implementation-playbook.md` (implementation DAG + Node 8 verification gates) 
* `backlog/430-unified-test-infrastructure-v2-target.md` (normative test contract; strict phases + JUnit evidence)
* `backlog/468-testing-front-door.md` (repo-wide test entrypoints and conventions)

---

> **Update (2026-01): 430v2 contract**
>
> This doc was originally written in the v1 “integration pack / `INTEGRATION_MODE`” era. Keep those details for historical context, but treat the normative execution contract as:
> - `ameide test` (Phase 0/1/2: contract → unit → integration; local-only)
> - `ameide test cluster` (cluster-only Playwright E2E)
> - no `INTEGRATION_MODE`
> - no per-suite `run_integration_tests.sh` packs as the canonical path

## 1) Problem

Onboarding currently assumes “happy path” invariants that are frequently violated in real dev/prod-like conditions:

* **Implicit tenant readiness:** if `session.user.tenantId` exists, bootstrap skips (“tenant already present on session”), even when the tenant record is **not ready** (e.g., missing issuer mapping).
* **Hard failure on realm resolution:** org creation cannot proceed if tenant realm metadata is missing; this blocks onboarding before any org/membership creation.
* **Identity/tenant mismatch:** users can be stamped with a legacy or non-canonical tenant id (e.g. `atlas-org`) while the canonical tenant is different (e.g. `tenant-atlas`), causing misrouting and failed mappings.
* **Principal/user mismatch:** the app can resolve access/memberships using a different user identifier than seeded/catalog data (e.g., email-keyed seeded memberships vs runtime Keycloak `sub`/platform user id), producing a false “0 orgs” and misclassifying seeded users as `UNASSIGNED`.
* **Timeouts treated as “empty”:** org discovery timeouts can be incorrectly interpreted as “no orgs,” sending users into onboarding when access is unknown.
* **Dev personas accidentally enter onboarding:** seeded/admin personas should not end up in first-time onboarding; if they do, it should be an explicit “access resolution degraded” state with retry/diagnostics, not a self-serve provisioning attempt.

This turns onboarding into a brittle coupling of session fields + best-effort calls, and pushes diagnosis into “wipe/reseed” rather than application-level recovery.

---

## 2) Decision

Reframe onboarding as a **primitive-composed workflow** (Domain/Process/Projection/Integration/UISurface/(optional)Agent), with explicit contracts, explicit state, and explicit non-happy-path handling.

Additionally, ship a **repo-root integration pack** under `tests/integration/onboarding/` (without moving runtime primitives) that gates end-to-end onboarding behavior in both repo and cluster modes.

**Pivot target (clear outcome)**

* A seeded/admin user can log in and land in the app shell without calling registrations endpoints.
* If access resolution is degraded (TIMEOUT/ERROR) or tenant readiness is violated (issuer mapping missing/mismatch), the UI shows a typed, actionable state — never first-time onboarding and never a 500.

---

## 3) Capability decomposition into primitives

### 3.1 Domain primitives (canonical truth)

**Domain: Tenancy Catalog**

* Tenants (canonical id, status, required `issuer` (OIDC issuer URL) + `tenantSlug`; optional `realm_name` ops attribute)
* Orgs
* Memberships
* Tenant aliases (legacy → canonical mapping)
* Onboarding attempt record (idempotency keys; current step; last blocker)

**Domain: Identity Catalog**

* Users (canonical internal id; email as an attribute, not an identifier)
* External identities (issuer/provider + subject → user id link)
* Optional: legacy user aliases (email → user id), guarded by policy

**Domain operations (examples)**

* `ResolveUserContext(principal)` → canonical tenant/org/user context (do not trust raw `session.user.tenantId` or ad-hoc user identifiers)
* `EnsurePlatformUser(principal)` → canonical user id (link external identity by `(issuer, sub)`; email-linking is support-only recovery, audited, and disabled by default)
* `ValidateTenantReady(tenantId)` → returns *typed missing fields* (no throw-only API)
* `CreateOrGetOrganization(idempotencyKey, tenantId, orgSpec)` (idempotent)
* `CreateOrGetMembership(idempotencyKey, orgId, userId, role)` (idempotent)
* `ActivateTenant(tenantId)` (idempotent)
* `UpsertTenantIssuerMapping(tenantId, issuer, source)` (updates the routing view: `issuer → tenantId`)
* `UpsertTenantAlias(legacyTenantId, canonicalTenantId)`

**Tenant readiness invariants (minimum)**

* `tenant.issuer` exists and is a stable OIDC issuer URL for the tenant realm
* `tenant.tenantSlug` exists and is stable
* If `tenant.realm_name` is stored (ops attribute), it matches the issuer URL and the provisioned Keycloak realm

### 3.2 Process primitive (orchestrator, resumable)

**Process: Onboarding Orchestrator (Temporal)**

> **Note (v6 posture):** this is a **platform** workflow (Temporal is allowed here).  
> Business BPMN Process primitives (capability workflows authored as BPMN) execute on Zeebe/Flowable; BPMN is not compiled/executed on Temporal.

* Converts “principal + intent” into durable Domain outcomes via stepwise, resumable execution:

  1. **Preflight** (determine lane: UNASSIGNED vs SEEDED_PERSONA vs ASSIGNED_NO_ORG vs HAS_ORG)
  2. **EnsureTenantReady** (validate → repair/backfill or block with actionable code)
  3. **EnsureOrgAndMembership** (idempotent creates)
  4. **ActivateTenant** (idempotent transition)
* Emits process facts / step timeline suitable for Projection and tests.

### 3.3 Projection primitives (read models for UI/tests)

**Projection: OnboardingAttemptView**

* `status`: `IN_PROGRESS | BLOCKED | DONE`
* `step`: `PREFLIGHT | TENANT_READY | ORG_MEMBERSHIP | ACTIVATION`
* `blockers[]`: typed codes + details + correlation ids
* `timestamps`: for a timeline

**Projection: UserAccessView**

* canonical platform user id (resolved from the principal, not inferred)
* canonical tenant id
* org memberships
* **org discovery status**: `OK | EMPTY | TIMEOUT | ERROR` (timeout is not empty)
* **implementation note**: carry this status through the session payload (e.g. `session.user.organizationAccessStatus`) so routing/UI can distinguish `EMPTY` from `TIMEOUT/ERROR` (no “infer from empty list”)

### 3.4 Integration primitives (Keycloak / issuer mapping)

**Integration: Identity Adapter**

* Get principal identity attributes (issuer URL + subject)
* Never stamp tenant identity into tokens/IdP on normal login flows
* Resolve/provision tenant issuer mapping (realm-per-tenant is mandatory) and validate issuer via OIDC discovery metadata (no string parsing)

### 3.5 UISurface primitive (onboarding UI)

**UISurface: Onboarding Portal**

* Pure state-machine client of `OnboardingAttemptView` + `UserAccessView`
* No “infer from empty list” logic
* Lane-splitting:

  * UNASSIGNED → first-time onboarding (registrations flow)
  * SEEDED/HAS_ORG → route to app shell; show “access resolution degraded” banner on TIMEOUT/ERROR
  * ASSIGNED_NO_ORG → route to “Create org” flow (separate from first-time onboarding)
* Registrations endpoints are only used for UNASSIGNED; assigned lanes use a dedicated “Create org” API/command and never trigger tenant provisioning.

### 3.6 Optional Agent primitive (diagnostics + guided repair)

**Agent: Onboarding Support Agent (optional)**

* Reads projection + domain state
* Proposes repair actions (alias create, issuer mapping backfill) with guardrails
* Useful for dev/ops, not required for core completion

---

## 4) Non-happy-path semantics (explicit requirements)

These are the “must not regress” behaviors that fix the failures you saw:

1. **Bootstrap skip must mean “tenant ready enough”, not “tenantId exists.”**
   If tenant exists but missing required readiness fields, the process must attempt repair/backfill or return a typed `BLOCKED` state (no hard crash).

2. **Realm-per-tenant is a hard invariant (no exceptions).**
   Every tenant must have a dedicated realm. Missing/invalid realm metadata must trigger repair/provisioning or return a typed blocker (never a 500).

   * Missing issuer mapping: `TENANT_ISSUER_MISSING`
   * Issuer mismatch: `TENANT_ISSUER_MISMATCH`
   * Issuer not registered: `TENANT_NOT_FOUND_FOR_ISSUER`
   * Principal issuer/tenant mismatch: `TENANT_CONTEXT_MISMATCH` (requires re-auth or admin repair; never “auto-provision” under the wrong issuer)

3. **Canonical tenant resolution + aliasing.**
   If identity/session tenantId is legacy/non-canonical, the system canonicalizes (via alias map) and proceeds without user-visible crashes.

4. **Canonical principal identity resolution.**
   Access resolution must use a canonical platform user id derived from the principal (external identity link). If the system cannot resolve the principal to a platform user (or finds ambiguous matches), it must surface a typed blocker and must not misclassify the user as `UNASSIGNED`.

5. **Unknown ≠ empty.**
   Org discovery timeout must surface as `TIMEOUT` in projections and must not route the user into first-time onboarding.

6. **Idempotency + resumability across retries.**
   Replaying Start/Complete calls must converge on exactly one org + one membership + one activation transition (no duplicates).

7. **Seeded personas should never silently enter onboarding.**
   If they do (because access resolution degraded), the UI shows “we can’t verify access” with retry, not self-serve provisioning.

8. **Lane semantics + repair authority are explicit.**
   “Repair/backfill” must be bounded by lane + actor (system vs admin/support vs end user). Example default policy:

   * UNASSIGNED: system may provision realm + write issuer mapping as part of tenant bootstrap.
   * ASSIGNED_*: system may canonicalize via alias mapping, but realm provisioning and identity (IdP) stamping require explicit permission or admin action.

---

## 5) Repo structure: integration pack home

Use a repo-root integration pack for cross-primitive orchestration. Runtime code stays in primitives/services; only tests (and optional fixtures) live here.

```text
tests/integration/onboarding/
  run_integration_tests.sh
  (tests|fixtures|src)
```
 
The pack is discoverable via `pnpm test:integration -- --list` and is validated by the same 430 runner contract as service/primitive packs.

---

## 6) Contracts & APIs (proto-first, policy in implementation)

Add/extend proto contracts to support:

* Process commands: `StartAttempt`, `ResumeAttempt`, `SubmitOrgDetails`, `Retry`
* Queries: `GetOnboardingAttempt`, `GetUserAccess`
* Error codes / blocker model: stable enum + typed details
* Facts/events to drive projections: `OnboardingStepChanged`, `OnboardingBlocked`, `TenantIssuerMapped`, `TenantAliasAdded`, `OrganizationCreated`, `MembershipCreated`, `TenantActivated`

Keep environment config/policy out of proto; proto defines APIs/messages/facts as the behavior schema for codegen and SDKs. 

---

## 7) Implementation plan (testable, mapped to the playbook DAG)

This follows the capability implementation DAG and ends with “Verify & Package” gates. 

### Phase 0 — Baseline + guardrails (align with 597)

**Deliverables**

* Phase 0 checklist completed per `backlog/597-login-onboarding-primitives.md` (issuer-driven routing, pre-login issuer selection, multi-issuer callback validation, no login-time provisioning, no “TIMEOUT⇒EMPTY” collapse).
* Backlog hygiene: superseded/legacy tenant-resolution docs updated to avoid issuer/realm-name parsing and to treat `tenantId` claims as hints only (if present).

**Exit criteria**

* End-to-end failure chain is captured (what identifier mismatched, what became “empty”, what routed into onboarding).
* All implementers have a single “routing authority” statement: `issuer URL (iss)` → server-side `issuer → tenant_id` mapping, else typed blocker in-shell.

### Phase A — Capability boundary + test skeleton (fast)

**Deliverables**

* Integration pack skeleton:

  * `tests/integration/onboarding/run_integration_tests.sh`

**Exit criteria**

* Pack runs in `INTEGRATION_MODE=repo` and produces JUnit+artifacts per runner contract (even if tests are initially skipped/placeholder within allowed scaffolding rules). 

### Phase B — Contract design (proto)

**Deliverables**

* Proto updates for:

  * onboarding process commands
  * onboarding queries (attempt + access views)
  * blocker/error code taxonomy
  * domain/process facts required by projections

**Exit criteria**

* `buf lint` + `buf build` pass (Gate B), regen-diff clean after generation. 

### Phase C — Domain + Projection first (make non-happy paths representable)

**Deliverables**

* Domain:

  * canonical tenant resolution
  * canonical principal identity resolution (external identity links)
  * tenant readiness validator returning typed missing fields
  * alias mapping operations
  * idempotent create-or-get org/membership
* Projection:

  * `UserAccessView` with `OK|EMPTY|TIMEOUT|ERROR`
  * `OnboardingAttemptView` with blockers and step timeline

**Exit criteria**

* Unit tests cover:

  * alias canonicalization
  * readiness validator behaviors
  * TIMEOUT not mapped to EMPTY
* Repo-mode integration pack can validate the projection state transitions headlessly (no UI required). 

### Phase D — Process orchestrator (resumable onboarding)

**Deliverables**

* Temporal workflow:

  * Preflight lane selection
  * EnsureTenantReady (repair/backfill/block)
  * EnsureOrgAndMembership
  * ActivateTenant
* Emits process facts sufficient for projection timeline

**Exit criteria**

* Temporal deterministic tests for replay/resume
* Repo-mode integration tests:

  * retries converge (idempotency)
  * blocked states are stable + actionable

### Phase E — Integration (Keycloak/realm) + UISurface routing

**Deliverables**

* Integration adapter:

  * resolve/provision tenant issuer mapping + validate issuer via OIDC discovery (as permitted)
  * provision realm/client/roles only in onboarding/support flows (never on normal login)
* UISurface:

  * routes based on `UserAccessView` + `OnboardingAttemptView`
  * seeded persona path avoids onboarding
  * TIMEOUT produces retry UX, not onboarding

**Exit criteria**

* Cluster-mode integration pack can run end-to-end against deployed env (Telepresence or job), asserting on Domain/Process/Projection—not pods/logs. 

### Phase F — Verify & Package (quality gates)

Run per-kind verification + repo-wide gates:

* `ameide primitive verify` for each involved primitive kind (Domain/Process/Projection/Integration/UISurface/(Agent)), plus `buf lint`, `buf breaking`, regen-diff. 

**Exit criteria**

* All verification checks pass; no scaffold RED tests remain. 

---

## 8) Repo-root integration pack: required E2E tests

These are headless and assert on durable state/facts/projections (not “pods existed”). 

**Repo-mode dependency matrix (make it deterministic)**

| Component | Repo mode | Cluster mode |
| --- | --- | --- |
| Domain persistence | real DB (ephemeral) | real DB |
| Process runtime | Temporal test env (in-process) | real Temporal |
| Identity integration (Keycloak) | mock adapter with scripted outcomes (success/mismatch/not-found/timeout) | real Keycloak |
| Projections | real (in-process) | real (deployed) |
| UI | not required | optional |

**Repo mode (`INTEGRATION_MODE=repo`)** — fast, deterministic

* **E2E‑0 Happy path (UNASSIGNED)**: attempt completes → tenant/org/membership created → activation done
* **E2E‑1 Tenant exists but issuer mapping is not ready** (missing/mismatched issuer): attempt repairs/provisions (only when policy allows) or blocks with `TENANT_ISSUER_MISSING` / `TENANT_ISSUER_MISMATCH` (no crash)
* **E2E‑2 Principal identity canonicalization**: seeded memberships keyed to a legacy identifier become discoverable after linking principal → user id; seeded personas route to app shell (no registrations)
* **E2E‑3 TenantId mismatch canonicalization**: legacy id resolves via alias; onboarding succeeds
* **E2E‑4 Org discovery timeout**: projection shows `TIMEOUT`; onboarding not started; retry works
* **E2E‑5 Idempotency**: repeated start/complete converges to one org/membership
* **E2E‑6 Seeded persona**: never enters first-time onboarding; TIMEOUT yields retry banner
* **E2E‑7 Principal issuer mismatch**: projection shows `TENANT_CONTEXT_MISMATCH` (or `TENANT_NOT_FOUND_FOR_ISSUER`); user is not routed into provisioning
* **E2E‑8 Create-first-org (ASSIGNED_NO_ORG)**: creates org + membership without calling registrations/tenant bootstrap; respects readiness/repair policy

**Cluster mode (`INTEGRATION_MODE=cluster`)** — real dependencies

* Repeat **E2E‑0..E2E‑2**, and at least one of **E2E‑3/E2E‑4** to validate real networking + identity integration.

Runner contract: `INTEGRATION_MODE=repo|cluster`, single implementation, JUnit+artifacts, fail-fast in cluster mode on missing env. 

---

## 9) Acceptance criteria (Definition of Done)

1. **No hard crash** in onboarding when tenant issuer mapping is missing; system returns a typed issue or repairs/backfills and proceeds.
2. **Bootstrap skip logic** no longer depends solely on `session.user.tenantId`; it depends on tenant readiness.
3. **Canonical tenant resolution** exists and handles legacy tenant ids via alias mapping.
4. **Canonical principal identity resolution** exists so memberships/orgs are resolved for the canonical platform user id (not a legacy identifier); seeded personas are not misclassified as `UNASSIGNED`.
5. **Org discovery timeout** is represented as `TIMEOUT` (unknown) and never interpreted as “no orgs.”
6. **Idempotent onboarding**: retries do not create duplicates; workflow is resumable.
7. **Seeded personas do not enter first-time onboarding**; seeded admin can log in and land in the app shell without calling registrations endpoints; UI shows “access degraded” with retry when applicable.
8. **ASSIGNED_NO_ORG lane is supported**: CreateOrg runs without calling registrations/tenant bootstrap; permissions are enforced.
9. **Realm-per-tenant invariant enforced**: every tenant has an issuer mapping; mismatches are typed issues (not stack traces).
10. `tests/integration/onboarding/run_integration_tests.sh` exists and runs in both repo+cluster modes under the standard contract. 
11. All touched primitives pass verification and gates (`ameide primitive verify`, `buf lint`, regen-diff clean). 

---

## 10) Rollout / migration notes

* Introduce alias mappings and/or issuer mapping backfill as **application-level repair** rather than requiring DB wipe/reseed.
* Ship behind a feature flag if needed:

  * enable “canonicalization + unknown-not-empty” first
  * then enable “repair/backfill” paths
* Instrument blockers with correlation ids and projection timeline to make diagnosis self-serve.

---

If you want, I can also draft the **exact blocker/error code enum** + the minimal `OnboardingAttemptView` / `UserAccessView` schemas (proto-friendly) so Phase B is immediately actionable and your E2E tests can assert on stable codes instead of stack traces.
