## Backlog 319 – Realm-Per-Tenant Onboarding (v3, Temporal-Orchestrated)

This revision replaces the imperative `IdentityOrchestrator` with a **Temporal workflows** while preserving the existing HTTP surface (`/registrations/bootstrap`, `/registrations/complete`) and the Auth.js gating (`hasRealOrganization`). The aim is to obtain reliable, observable, and extensible orchestration for tenant activation, realm provisioning, and future payment/infra steps—without introducing a brand-new “Onboarding Service”.

---

### 1. Goals & Non-Goals

**Goals**
- Model onboarding as a deterministic Temporal workflows (`OnboardingWorkflow`) that owns all cross-system orchestration.
- Retain the platform app as the API entry point; the workflows executes behind the existing POST handlers.
- Produce first-class visibility into onboarding state (`pending → provisioning_realm → persisting_org → activating_tenant → completed|failed`) and support retries/compensation.
- Keep UX semantics: middleware uses `session.user.hasRealOrganization` to gate `/onboarding`; fixing cookie domain/host remains a configuration task (orthogonal to orchestration).

**Non-Goals**
- Creating a separate onboarding microservice or GraphQL surface.
- Replacing Keycloak/Vault interactions; they move into Temporal activities but remain the same operations.
- Automating payment/infra provisioning immediately—those become child workflows hooked in later.

---

### 2. High-Level Architecture

| Component | Responsibility | Notes |
|-----------|----------------|-------|
| Platform Next.js app | Handles `/registrations/bootstrap`, `/registrations/complete`, session refresh, middleware gating. | Continues to issue `updateSession` with preferred org hints. |
| Temporal Server (already deployed) | Orchestration, state tracking, retries, signals. | Uses namespace `platform-onboarding` (new). |
| Onboarding Worker | Hosts `OnboardingWorkflow` and all unprivileged activities (tenant catalog, platform gRPC, event emission). | Deploy as `services/onboarding-worker` image with access to gRPC + Temporal, but **without** Keycloak/Vault creds. |
| Realm-Provisioner Worker (optional) | Hosts privileged activities for Keycloak/Vault operations. | Separate Deployment with restricted RBAC + NetworkPolicy; invoked via activity stubs from the main workflows. |
| Shared packages (`@ameideio/ameide-sdk-ts`) | Continue to supply protobuf clients and serialization helpers. | Activities use these to call gRPC. |

Deployment adds two Helm releases (workers) plus a Temporal Task Queue `onboarding`. HTTP services remain unchanged.

---

### 3. Workflow Definition

```ts
// pseudo-code
export const OnboardingWorkflow = wf.define(() => {
  const state = stateSignal('pending');

  async function ensureTenantActive(args) { ... }
  async function provisionRealm(args) { ... }
  async function persistOrganization(args) { ... }
  async function activateTenant(args) { ... }

  wf.setHandler(statusQuery, () => ({ state: state.get(), ...progressSnapshot }));
  wf.setHandler(paymentSignal, (payload) => { payment.complete(payload); });

  return wf.start(async ({ tenantId, realmName, orgInput, user }) => {
    await ensureTenantActive(tenantId);
    state.set('provisioning_realm');

    await provisionRealm({ tenantId, realmName, orgInput, user });
    state.set('persisting_org');

    const org = await persistOrganization({ tenantId, orgInput });
    state.set('activating_tenant');

    await activateTenant({ tenantId, org });
    state.set('completed');

    return { tenantId, organizationId: org.id };
  });
});
```

**Key properties**
- **Workflow ID:** `onboard/${tenantId}` (or `${kcSub}/${tenantId}` when multi-tenant). Combine with HTTP `Idempotency-Key` header. Use `workflowsIdReusePolicy = rejectDuplicate`.
- **Task Queue:** `onboarding` for the parent; `realm-provisioner` for privileged activities (via activity stubs).
- **Queries:** `status()` returns `{ state, tenantId, organizationId?, realmName?, lastError? }`.
- **Signals (extensible):** `paymentCompleted(txId)`, `infraReady(ref)` allow asynchronous completion.
- **Timeouts:** Workflow (8h) with heartbeating activities; activities sized appropriately (see Section 5).

---

### 4. Activities (Idempotent “Ensure” Semantics)

| Activity | Description | Idempotency Strategy | Worker |
|----------|-------------|----------------------|--------|
| `ensureTenantActive(tenantId)` | Reads tenant catalog; transitions `pending_onboarding → activating` and stamps Keycloak `tenantId`. | Re-reads catalog; safe if rerun. | Onboarding |
| `provisionRealm({ tenantId, realmName, orgId, admin })` | Creates Keycloak realm, seeds RBAC roles (`org-admin`, `org-member`, `tenant-admin`), provisions admin user, adds them to the **org-admin** + **tenant-admin** groups, creates `platform-app-{slug}` client, stores secret in Key Vault, **refreshes master token cache** immediately afterward. | Each step checks for existing realm/client/user before creation; secrets retrieved/updated if already present. | Realm-Provisioner |
| `createOrganization({ tenantId, name, slug })` | Calls platform gRPC → `createOrganization`, ensures membership created same transaction, and writes platform-side role bindings (admin user gets `ORG_ADMIN`, tenant-level `TENANT_ADMIN`). | Uses generated `orgId`; if entity exists, verifies membership/roles and returns existing ID. | Onboarding |
| `updateTenantCatalog({ tenantId, org })` | Marks tenant `active`, writes `realm_name`, `default_org_id`, emits `tenant.created` & `organization.created` via outbox. | Replays update via UPSERT; emits events with deterministic IDs to avoid duplicates. | Onboarding |
| `recordAuditEvent(...)` (optional) | Writes run metadata to audit log/Analytics. | Deterministic event ID seeded from workflows execution. | Onboarding |

Future child workflows plug into the parent after `provisionRealm` or before `activateTenant`.

---

### 5. Retry, Backoff, Compensation

| Activity | Retry Policy | Notes |
|----------|--------------|-------|
| `ensureTenantActive` | `initial=250ms`, `backoff=2x`, `maxInterval=5s`, `maxAttempts=8`. | Fails fast on `tenant not found`. |
| `provisionRealm` | `initial=1s`, `backoff=2x`, `maxInterval=30s`, `maxAttempts=10`. | Distinguish `409 realm exists` (success) vs `4xx` fatal. Heartbeat every 5s for long operations. |
| `createOrganization` | `initial=500ms`, `backoff=2x`, `maxInterval=5s`, `maxAttempts=6`. | Surfaces validation errors immediately. |
| `updateTenantCatalog` | `initial=250ms`, `maxAttempts=6`. | Non-retryable on `tenant not found`. |

**Compensation**
- If `provisionRealm` succeeded but `createOrganization` fails permanently, schedule a compensating activity `deleteRealm(realmName)` or set workflows state `reconcile_required` and emit an alert. Use Temporal’s continue-as-new or child workflows for cleanup to avoid losing history.
- Payment/infra child workflows will define their own compensation (e.g., refund, infrastructure teardown).

---

### 6. API Contract (Unchanged Endpoints)

**`POST /api/v1/registrations/bootstrap`**
- Remains synchronous; prepares catalog entry and sets Keycloak `tenantId`.
- After bootstrap completes, client refreshes session (`updateSession({ refreshTenant: true })`)—unchanged.

**`POST /api/v1/registrations/complete`**
- Extracts `(tenantId, kcSub, desired slug/name)` from session & payload.
- **Starts or gets** the workflows: `client.workflows.start(OnboardingWorkflow, { workflowsId, taskQueue: 'onboarding', args })`.
- Waits up to ~3s (`WorkflowClient.execute` with timeout). If workflows reaches `completed`, return `{ tenant, organization, membership }` (same shape as today).
- If still running, return **`202 Accepted`** with `{ runId: workflowsId }`; the wizard shows “Finishing setup…”.

**`GET /api/v1/registrations/status?runId=`**
- Uses `workflowsHandle.query(statusQuery)` to return progress `{ state, tenantId, organizationId?, lastError? }`.
- The frontend polls at 1–2s intervals until `completed` or `failed`. On failure, render the existing error UI.

Auth/session handling stays identical: after successful completion, the UI calls `updateSession` with `{ refreshTenant: true, refreshOrganizations: true, preferredOrgId: org.slug, organizationIds: [org.slug], hasRealOrganization: true }`. The middleware exits the onboarding gate once the session cookie (properly scoped to `.dev.ameide.io`) is refreshed.

---

### 7. Worker Deployments & Security

| Deployment | Image | Credentials | Network Policy |
|------------|-------|-------------|----------------|
| `www-ameide-platform` | unchanged | Auth.js, Redis, platform gRPC | Already deployed behind Envoy. |
| `onboarding-worker` | new | Temporal TLS, platform service token | Egress to Temporal + platform gRPC/DB. No Keycloak/Vault access. |
| `realm-provisioner-worker` (optional) | new | Keycloak master admin secret, Key Vault access, Temporal TLS | Egress only to Keycloak + Azure Key Vault. Can live in same repo; only runs privileged activities. |

Both workers share code via packages: `@/features/onboarding/worker`, `@/features/onboarding/activities`.

---

### 8. Observability & Operations

- **Temporal visibility:** Use Web UI (namespace `platform-onboarding`) to inspect runs, query status, and replay.
- **Logging:** Each activity logs structured payloads (tenantId, orgId, realmName, durations) to alloy/OTLP.
- **Metrics:** Emit counters (`onboarding.completed`, `onboarding.failed`, `realm.provision.duration_ms`) via OTEL.
- **Alerts:** 
  - Workflow stuck in `provisioning_realm` > 2m.
  - Activity failure > N retries (Temporal emits metrics for failure reasons; integrate with Prometheus).
  - `reconcile_required` states (requires manual intervention) surface via Slack/Webhook.
- **Events:** `updateTenantCatalog` publishes `tenant.created`, `organization.created` (existing backlog requirement). Ensure outbox uses workflows execution ID as dedupe key.

---

### 9. RBAC Implementation

| Step | System | Details |
|------|--------|---------|
| Seed roles | Keycloak realm | During `provisionRealm`, ensure realm roles (`org-admin`, `org-member`, `tenant-admin`, `viewer` as needed) and corresponding groups exist. Groups are prefixed consistently (e.g., `org-admin`, `tenant-admin`). |
| Assign bootstrap user | Keycloak realm | Add the onboarding user to both `org-admin` and `tenant-admin` groups; idempotent via `ensureGroupMembership(userId, groupName)`. |
| Platform membership | Platform service | `createOrganization` returns the new `organizationId` and writes `organization_memberships` with `role = ORG_ADMIN` and `tenant_role = TENANT_ADMIN` for the bootstrap user. Use UPSERT semantics so reruns reconcile. |
| Session hints | Auth.js | `updateSession({ preferredOrgId: org.slug, organizationIds: [org.slug], hasRealOrganization: true })` ensures the refreshed JWT reflects admin roles immediately. |
| Additional users | Invites / auto-tenant | When a new user joins the tenant, reuse the same activity helper to add them to Keycloak groups and insert platform role rows. Over time, this can become a standalone `EnsureMembershipActivity`. |

Temporal activities own the “ensure group membership” logic so retries remain safe. Platform role assignments stay in the same transaction that creates the organization membership to avoid divergence.

---

### 10. Rollout Plan

1. **Week 1**
   - Scaffold Temporal namespace + Helm releases for `onboarding-worker` (and optionally `realm-provisioner`).
   - Wrap existing orchestrator logic into Temporal activities (reuse service abstractions).
   - Implement workflows start from `/registrations/complete`, including 3-second fast path and `202` fallback.
   - Add `GET /registrations/status` endpoint + UI poller.
2. **Week 2**
   - Implement compensation for realm provisioning failures.
   - Harden retries (per-section policies), add feature flags.
   - Ensure cookie domain/host config is respected (AUTH_COOKIE_DOMAIN, NEXTAUTH_URL) so middleware gating works.
3. **Week 3+**
   - Add Temporal-based reconcile job for stuck runs.
   - Split Realm-Provisioner worker if stronger isolation required.
   - Introduce Payment/Infrastructure child workflows as downstream systems come online.
   - Instrument SLOs (p95 < 30s for fast path; 99% success without manual intervention).

---

### 11. Open Questions & Follow-Ups

- **Idempotency-Key semantics:** Do we accept caller-provided keys or standardise on `(tenantId)`? (recommendation: accept header; default to deterministic ID if absent).
- **Compensation policy:** Which failures should auto-clean realms vs. mark for manual reconcile?
- **Eventing contract:** Are `tenant.created` / `organization.created` enough or do we emit step-level audit events?
- **Temporal versioning:** Add `wf.patched` markers when evolving steps (e.g., migrating from gRPC only to child workflows).
- **UX status page:** Should `/registrations/status` feed an operator dashboard, or is Temporal UI sufficient until later?
- **RBAC propagation:** New tenant admins must receive the `org-admin` (and tenant admin) roles in both Keycloak groups and platform membership. Confirm whether Keycloak “groups” alone are authoritative or if the platform DB needs a parallel write; ensure the workflows updates both when a user is auto-assigned during onboarding and when additional users join later.

---

### 12. Summary

The onboarding v3 plan moves orchestration into Temporal, preserving the existing UX and HTTP API while unlocking deterministic state tracking, reliable retries, and room for payment/infra extensions. It honours backlog 319’s requirements—realm-per-tenant, fail-fast behaviour, immediate session hints—and decouples long-running provisioning from the Next.js request lifecycle in a way that matches our goal to add more “activation” steps soon.
