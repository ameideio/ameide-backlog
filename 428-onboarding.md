# 428: Consolidated Onboarding & Identity Provisioning

**Status:** Planning
**Created:** 2025-12-01
**Owner:** Platform / Identity
**Supersedes:** backlog/319-onboarding.md, backlog/369-onboarding-v2.md, backlog/319-onboarding-v4.md

---

## Executive Summary

This backlog consolidates all onboarding-related requirements from prior specifications (319, 369, 319-v4) into a single, prioritized execution plan. It covers the complete journey from lead capture through fully-provisioned tenant, incorporating lessons learned and aligning with the current architecture (realm-per-tenant, Temporal orchestration, unified secret guardrails).

---

## 1. Current State Assessment

### What Works (Production Ready)

| Component | Status | Implementation |
|-----------|--------|----------------|
| Frontend Wizard | ✅ | 3-step onboarding UI in `app/(app)/onboarding/page.tsx` |
| Middleware Gate | ✅ | Forces new users to complete onboarding before app access |
| Registration API | ✅ | `POST /api/v1/registrations/complete` with tenant validation |
| Identity Orchestrator | ✅ | `features/identity/lib/orchestrator.ts` - atomic org/membership creation |
| Invitation System | ✅ | 7 endpoints: create, validate, accept, revoke, resend, list |
| Token Security | ✅ | SHA-256 hashing, one-time use, expiration |
| Database Schema | ✅ | Organizations, memberships, roles, invitations, users, teams |
| RLS Enforcement | ✅ | Row-level security on platform tables |
| JWT Tenant Claims | ✅ | `tenantId` claim via Keycloak protocol mapper |
| RBAC Roles | ✅ | admin, contributor, viewer, guest, service (canonical catalog) |

### Current Blockers

| Blocker | Impact | Remediation |
|---------|--------|-------------|
| Keycloak realm provisioning (403) | Self-serve tenant creation fails | Restore `realm-admin` + `create-realm` roles to `platform-app-master` |
| Platform service unavailable or misrouted | Organization creation/lookup throws `ConnectError` | Ensure platform release is enabled **and** the caller uses the correct RPC protocol/route |
| Tenant catalog automation | No `pending_onboarding` → `active` transition | Implement status workflow job |
| Admin token cache invalidation | Provisioned realms lack scoped permissions | Refresh token cache after realm creation |

> **Update (2025-12-22):** `ConnectError: [unimplemented] HTTP 404` during org fetch is frequently a **routing / protocol mismatch** (Connect vs gRPC) rather than an application-level “method missing” error. The target transport architecture and GitOps routing boundaries are tracked in `backlog/589-rpc-transport-determinism.md`.

### Not Implemented (Gaps)

| Feature | Priority | Phase |
|---------|----------|-------|
| Email delivery (invitations, verification) | High | 1 |
| Event-driven provisioning (outbox + bus) | High | 2 |
| Domain discovery & verification | Medium | 2 |
| Corporate SSO (IdP brokering) | Medium | 2 |
| SSO enforcement & JIT auto-join | Medium | 2 |
| SCIM 2.0 provisioning | Low | 3 |
| Billing/entitlements integration | Medium | 3 |
| WebAuthn/Passwordless MFA | Low | 3 |
| Audit logging (structured) | Medium | 2 |
| Rate limiting on bootstrap | High | 1 |

---

## 2. Architecture

### Two-Level Tenancy Model

```
DEPLOYMENT (Kubernetes Cluster)
  └── TENANT (Infrastructure Isolation)
       ├── Keycloak Realm (realm-per-tenant)
       ├── Database Schema (RLS policies)
       └── ORGANIZATION (User Workspace)
            ├── Memberships (user ↔ org with roles)
            ├── Teams (functional groupings)
            └── Invitations (pending joins)
```

### Key Architectural Decisions

1. **Realm-Per-Tenant**: Each organization gets its own Keycloak realm for complete identity isolation (see backlog/333-realms.md)
2. **JWT-Based Tenant Resolution**: `tenantId` flows from Keycloak → session → all services
3. **No Hardcoded Fallbacks**: API routes fail-secure when tenantId missing
4. **Temporal Orchestration**: Long-running onboarding workflows via Temporal (target architecture)
5. **Vault-Managed Secrets**: All credentials via ExternalSecrets (see backlog/362-unified-secret-guardrails.md)

### Keycloak Capacity & Sharding Strategy

> ⚠️ **Scalability Constraint**: Keycloak realm-per-tenant does not scale linearly to 10k+ realms per cluster.

**Observed Limits (Keycloak 26.x)**:
- Community reports performance degradation beyond ~300-500 realms
- With aggressive cache tuning, ~1000 realms is achievable but requires dedicated capacity planning
- 10k tenants per cluster with realm-per-tenant is **not realistically supported** by current Keycloak

**Sharding Strategy**:
```
┌─────────────────────────────────────────────────────────────────────────┐
│                    KEYCLOAK SHARDING MODEL                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Tier 1: Enterprise Accounts (realm-per-tenant)                         │
│    - Dedicated Keycloak realm per organization                          │
│    - Full SSO/SCIM/IdP brokering capabilities                           │
│    - Max ~500 realms per Keycloak cluster                               │
│    - Route by tenant → Keycloak shard (tenant_keycloak_shard_id)        │
│                                                                         │
│  Tier 2: SMB/Self-Serve Accounts (shared realm)                         │
│    - Shared `ameide` realm with group-based tenant isolation            │
│    - JWT `tenantId` claim from user attributes                          │
│    - No per-tenant IdP brokering (use social/email auth)                │
│    - Scales to 100k+ tenants per Keycloak cluster                       │
│                                                                         │
│  Upgrade Path: SMB → Enterprise                                         │
│    - Migrate user from shared realm to dedicated realm                  │
│    - Link existing platform.users/memberships to new realm              │
│    - One-time migration workflow (Temporal)                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Capacity Planning**:
| Metric | Phase 1 Target | Phase 5 Target | Notes |
|--------|----------------|----------------|-------|
| Realms per Keycloak cluster | 50 | 500 | Load test required at 100, 250, 500 |
| Total tenants (all tiers) | 1,000 | 10,000 | Hybrid model enables scale |
| Keycloak clusters (production) | 1 | 2-3 | Shard by tenant tier + region |

**Implementation Notes**:
- Add `keycloak_shard_id` column to `platform.tenants` table
- Realm provisioner client (`realm-provisioner`) uses minimal scopes per shard
- Token cache refresh is shard-aware
- Monitoring: alert when any shard exceeds 80% realm capacity

### Data Model

```sql
-- Core tables (platform schema)
platform.tenants           -- Infrastructure isolation containers
platform.organizations     -- User workspaces within tenants
platform.organization_roles    -- Role definitions per org
platform.organization_memberships  -- User ↔ org with role_ids
platform.users             -- Platform user records
platform.invitations       -- Pending membership invitations
platform.teams             -- Team groupings
platform.team_memberships  -- User ↔ team assignments

-- Missing tables (to be added)
platform.org_domains       -- Verified domain claims
platform.sso_connections   -- IdP configurations per org
platform.subscriptions     -- Billing plan bindings
platform.entitlements      -- Feature/seat entitlements
platform.provisioning_jobs -- Async provisioning status
platform.audit_events      -- Structured audit log
```

---

## 3. Requirements

### 3.1 Functional Requirements

#### FR-1: Self-Serve Signup (B2C)
- **FR-1.1**: User registers via Keycloak (email/password or social)
- **FR-1.2**: Middleware detects missing organization, redirects to `/onboarding`
- **FR-1.3**: Wizard collects organization name/slug in ≤ 3 screens
- **FR-1.4**: System creates organization + owner membership atomically
- **FR-1.5**: User redirected to `/org/{slug}/dashboard` on success
- **FR-1.6**: Tenant realm provisioned if realm-per-tenant is enabled

#### FR-2: Invitation Flow
- **FR-2.1**: Org admin creates invitation with email + role
- **FR-2.2**: System generates secure token (SHA-256 hashed)
- **FR-2.3**: Invitation email sent with acceptance link
- **FR-2.4**: Invitee authenticates (existing or new account)
- **FR-2.5**: System validates token, creates membership
- **FR-2.6**: Duplicate invitations handled gracefully

#### FR-3: Enterprise SSO (B2B)
- **FR-3.1**: Domain discovery API returns IdP requirements
- **FR-3.2**: Verified domains enforce SSO authentication
- **FR-3.3**: JIT provisioning creates membership on first SSO login
- **FR-3.4**: Admin configures IdP via self-service wizard
- **FR-3.5**: SCIM 2.0 endpoint for automated provisioning
- **FR-3.6**: SCIM endpoints support filtering (`userName`, `externalId`, `id`), pagination (`startIndex`, `count`), and PATCH semantics
- **FR-3.7**: SCIM endpoints secured with OAuth 2.0 bearer tokens scoped to single organization

#### FR-3.8: Identity Source-of-Truth & Precedence

When multiple identity sources are configured, the following precedence applies:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    IDENTITY SOURCE PRECEDENCE                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. SCIM (highest authority when enabled)                               │
│     - SCIM is authoritative source of memberships when configured       │
│     - SCIM DELETE → deactivates account (no login), does NOT purge data │
│     - SCIM-disabled accounts CANNOT be re-activated by JIT              │
│                                                                         │
│  2. JIT Provisioning                                                    │
│     - Creates membership on first SSO login if user not SCIM-managed    │
│     - Creates "candidate" user if SCIM is enabled but hasn't synced yet │
│     - JIT respects domain SSO enforcement                               │
│                                                                         │
│  3. Manual Invitations                                                  │
│     - Blocked when SCIM is active for the organization                  │
│     - Allowed for non-SCIM orgs or for roles SCIM doesn't manage        │
│                                                                         │
│  Edge Cases:                                                            │
│     - Personal email user + later company SSO enforcement:              │
│       User must link account or create new org account                  │
│     - SSO enforced but SCIM hasn't provisioned user:                    │
│       JIT creates candidate membership with default role                │
│     - SCIM deprovision + user attempts SSO:                             │
│       Access denied (SCIM authority preserved)                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### FR-4: Provisioning
- **FR-4.1**: Tenant realm created with proper admin permissions
- **FR-4.2**: Default roles seeded in new organizations
- **FR-4.3**: Default workspace/project created (optional)
- **FR-4.4**: Events emitted for downstream consumers
- **FR-4.5**: Provisioning status visible to user

#### FR-5: Billing Integration

> **Design Principle**: Stripe (or billing provider) is the **source of truth** for subscription state. Platform DB stores a projection updated via webhooks, never mutated directly.

- **FR-5.1**: Plan selection during onboarding (optional, feature-flagged)
- **FR-5.2**: Default path: sign up → trial environment → payment capture before trial ends
- **FR-5.3**: Payment-before-provisioning is **opt-in** for enterprise contracts only
- **FR-5.4**: Subscription created with configurable trial period (default: 14 days)
- **FR-5.5**: Seat limits enforced on membership creation (invites blocked when at limit)
- **FR-5.6**: `subscriptions.status`, dates, and plan updated **only via provider webhooks**
- **FR-5.7**: Usage records keyed by `(organization_id, metric_key, period, external_usage_id)` for idempotency

**Grace Period & Blocking Behavior**:
```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SUBSCRIPTION STATE TRANSITIONS                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  trialing → active                                                      │
│    - Payment method attached, trial converted                           │
│    - Full feature access                                                │
│                                                                         │
│  active → past_due                                                      │
│    - Payment failed, retry in progress                                  │
│    - Existing users retain access                                       │
│    - NEW invites/seats BLOCKED                                          │
│    - Premium features may be limited                                    │
│                                                                         │
│  past_due → canceled                                                    │
│    - Terminal payment failure or explicit cancellation                  │
│    - All users lose access (hard lock)                                  │
│    - Data retained per retention policy                                 │
│                                                                         │
│  trialing → canceled (no payment)                                       │
│    - Trial expired without payment                                      │
│    - Access revoked, data retained 30 days                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Non-Functional Requirements

#### NFR-1: Performance
| Metric | Target | Measurement |
|--------|--------|-------------|
| Signup to first dashboard | ≤ 90 seconds | End-to-end timer |
| Invitation acceptance | ≤ 30 seconds | Click to org access |
| Realm provisioning | ≤ 10 seconds | Keycloak API call |
| Page load (onboarding wizard) | ≤ 2 seconds | LCP metric |

#### NFR-2: Reliability
| Metric | Target |
|--------|--------|
| Onboarding success rate | ≥ 99% |
| Provisioning idempotency | 100% |
| Rollback on partial failure | Automatic |
| Recovery from Keycloak outage | Graceful degradation |

#### NFR-3: Security
| Requirement | Implementation |
|-------------|----------------|
| Token entropy | 256-bit secure random |
| Token storage | SHA-256 hash only |
| Token expiry | 7 days (configurable) |
| Rate limiting | 10 signups/IP/hour, 100 invites/org/day |
| CAPTCHA | On public signup after threshold |
| PII encryption | At rest (AES-256) |
| Audit trail | All sensitive actions logged |
| JWT validation | iss, aud, exp checks on every request |
| RLS enforcement | Database-level tenant isolation |
| MFA for org owners | **Required** for admin/owner roles (Phase 2 priority) |
| Realm provisioner | Dedicated `realm-provisioner` client with minimal scopes |

**SCIM-Specific Security Controls**:
| Requirement | Implementation |
|-------------|----------------|
| SCIM authentication | OAuth 2.0 bearer tokens with SCIM-specific scopes |
| SCIM rate limits | Independent limits: 100 ops/min per org, 1000 ops/hour |
| SCIM anomaly detection | Alert on mass deletes (>10% users in 1 hour), frequent role changes |
| SCIM audit logging | Before/after snapshots for all attribute changes |
| SCIM token rotation | 90-day max lifetime, rotation workflow documented |

**GDPR vs SCIM Distinction**:
- SCIM `DELETE` → **deactivates** account (no login), data retained
- GDPR `/deletion-request` → **full deletion** after retention window
- Billing data excluded from GDPR deletion where legally required

#### NFR-4: Observability
| Requirement | Implementation |
|-------------|----------------|
| Distributed tracing | OpenTelemetry spans across all services |
| Onboarding ID propagation | `app.onboarding.id` attribute |
| Funnel metrics | signup_started → email_verified → org_created → first_action |
| Error alerting | 403/401 rate anomaly detection |
| SLA dashboards | Grafana panels for TTFHW (Time To First Happy Workflow) |

**Required Trace Attributes** (for enterprise troubleshooting):
| Attribute | When Set | Purpose |
|-----------|----------|---------|
| `app.onboarding.id` | All onboarding spans | Correlate signup journey |
| `app.tenant.id` | Post-tenant creation | Filter by tenant |
| `app.organization.id` | Post-org creation | Filter by org |
| `app.user.id` | All authenticated spans | User journey |
| `app.invitation.id` | Invitation flows | Invitation tracking |
| `app.idp.provider` | SSO login spans | Filter by IdP (okta, entra, google) |
| `app.scim.system` | SCIM operations | Filter by provisioning source |
| `app.billing.plan_id` | Billing operations | Plan-specific issues |

#### NFR-5: Scalability
| Requirement | Target | Notes |
|-------------|--------|-------|
| Concurrent signups | 100/second | Distributed across app replicas |
| Tenants per cluster (all tiers) | 10,000 | Hybrid model: ~500 realm-per-tenant + shared realm |
| Realms per Keycloak cluster | 500 | See Keycloak Capacity & Sharding Strategy |
| Users per organization | 10,000 | RLS + indexed queries |
| Invitations in flight | 1,000,000 | Partitioned by org |

**SLOs (Service Level Objectives)**:
| SLO | Target | Window |
|-----|--------|--------|
| Signup success rate | 99% | 30-day trailing |
| Signup completion time (p99) | ≤ 90s | 30-day trailing |
| Invitation acceptance (p99) | ≤ 30s | 30-day trailing |
| SCIM sync latency (p95) | ≤ 5s | 24-hour trailing |
| Keycloak realm provisioning (p99) | ≤ 15s | 30-day trailing |

#### NFR-6: Compliance
| Requirement | Implementation |
|-------------|----------------|
| GDPR data export | `/api/v1/users/{id}/export` endpoint |
| GDPR data deletion | Cascading soft delete with 30-day retention |
| DSR automation | Temporal workflow for deletion requests |
| Audit retention | 7 years for billing, 2 years for access logs |
| Data residency | Tenant-level region configuration |

---

## 4. Execution Plan

### Phase 1: Stabilize Core Path (Sprint 1-2)

**Objective**: End-to-end self-serve signup completes without manual intervention

| Task | Owner | Estimate | Dependencies |
|------|-------|----------|--------------|
| 1.1 Restore Keycloak `platform-app-master` permissions | Infra | 2h | - |
| 1.2 Enable platform Helm release | Infra | 1h | - |
| 1.3 Implement admin token cache refresh after realm creation | Platform | 4h | 1.1 |
| 1.4 Add `pending_onboarding` → `active` status automation | Platform | 4h | 1.2 |
| 1.5 Add rate limiting to `/api/v1/registrations/bootstrap` | Platform | 2h | - |
| 1.6 Add instrumentation/metrics to bootstrap endpoint | Platform | 2h | - |
| 1.7 E2E test: complete signup flow with healthy backend | QA | 4h | 1.1-1.4 |

**Exit Criteria**:
- [ ] New user can complete signup without errors
- [ ] Organization created with owner membership
- [ ] Keycloak realm provisioned (if enabled)
- [ ] Bootstrap endpoint rate-limited and instrumented

#### Phase 1 – Implementation Progress

**Status by Task**

| Task | Status | Notes |
|------|--------|-------|
| 1.1 Restore Keycloak `platform-app-master` permissions | ✅ Implemented in config; confirm in target clusters | `master-realm.json` grants `create-realm` / `admin` to `service-account-platform-app` and `service-account-platform-app-master` (see `gitops/ameide-gitops/sources/charts/foundation/operators-config/keycloak_realm/values.yaml`). ExternalSecrets (`foundation-keycloak-admin-secrets.yaml`) provision the admin credentials, and `platform-keycloak-realm` wiring ensures the master bootstrap and admin client secret sync are in place. Cluster operators should verify these roles are active in live Keycloak instances. |
| 1.2 Enable platform Helm release | ✅ Declared and wired via gitops | The core platform service is defined as a Helm release (`apps-platform` in `gitops/ameide-gitops/environments/*/components/apps/core/platform/component.yaml`) and uses environment-specific values under `sources/values/*/platform`. Additional infrastructure (Keycloak, gateway, PostgreSQL) is enabled via `platform` components; enabling the platform path in a given cluster is now an operational toggle (syncing those components) rather than an implementation gap. |
| 1.3 Implement admin token cache refresh after realm creation | ✅ Implemented | `createTenantRealm` in `services/www_ameide_platform/lib/keycloak-admin.ts` calls `invalidateAdminTokenCache(config)` after creating the new realm, then obtains a fresh admin token to configure the realm client and service account. |
| 1.4 Add `pending_onboarding` → `active` status automation | ✅ Implemented | `services/platform/src/tenants/service.ts` creates tenants with status `pending_onboarding`; `IdentityOrchestrator.completeRegistration` (`services/www_ameide_platform/features/identity/lib/orchestrator.ts`) calls `client.tenants.updateTenant` with `status: TenantStatus.ACTIVE` and slug/realm metadata once organization + owner membership are created. |
| 1.5 Add rate limiting to `/api/v1/registrations/bootstrap` | ✅ Implemented | `services/www_ameide_platform/app/api/v1/registrations/bootstrap/route.ts` now enforces a per-user in-memory limit of **10 bootstrap attempts per hour**, returning `429` with `Retry-After` and a structured JSON error. Covered by a new integration test in `features/onboarding/__tests__/integration/registration-api.test.ts`. |
| 1.6 Add instrumentation/metrics to bootstrap endpoint | ✅ Implemented | The bootstrap route was already wrapped in `withRequestTelemetry` and now also records OpenTelemetry metrics via `services/www_ameide_platform/lib/onboarding-metrics.ts` (`onboarding_bootstrap_requests_total`, `onboarding_bootstrap_errors_total`, `onboarding_bootstrap_rate_limit_hits_total`, `onboarding_bootstrap_duration_seconds`) with status + HTTP code tags. |
| 1.7 E2E test: complete signup flow with healthy backend | ✅ Test suite implemented; cluster-only E2E | A full onboarding journey is covered in Playwright (`services/www_ameide_platform/features/onboarding/__tests__/e2e/onboarding-flow.spec.ts` and `new-user-redirect.spec.ts`) including tenant bootstrap, organization creation, redirect behavior, and post-onboarding access. These suites are designed as **cluster-only UI E2Es** (they require live Keycloak + platform and seeded data) and run whenever `INTEGRATION_MODE=cluster` is set for the Playwright job. Dual-mode (`INTEGRATION_MODE=repo|local|cluster`) applies to backend integration packs; the onboarding UI flows are exercised only in cluster environments, not in mocked mode. |

**Integration Test Dual-Mode Alignment (376)**

- Onboarding E2E tests explicitly follow the dual-mode contract:
  - Cluster runs use `INTEGRATION_MODE=cluster` and point at the in-cluster `www-ameide-platform` service (`gitops/ameide-gitops/sources/values/env/local/integration-tests/playwright.yaml`), exercising real Keycloak + platform flows end-to-end.
  - Mocked-mode integration for onboarding remains limited to unit/integration tests that stub gRPC and database dependencies; no attempt is made to run the Playwright signup journey in non-cluster mode (`repo`/`local`).
- `tools/integration-runner` and the Playwright integration values ensure `INTEGRATION_MODE` is always set explicitly; unknown values fail fast per #376, and cluster-mode tests are never run implicitly from local/devcontainer workflows.

**Additional Work Completed (Beyond Initial Phase 1 Tasks)**

- **Stricter realm configuration guarantees**
  - `getKeycloakConfig` in `services/www_ameide_platform/lib/keycloak-admin.ts` now requires that `AUTH_KEYCLOAK_ISSUER` include a `/realms/<realm>` segment; it no longer falls back to a default realm. Misconfigured issuers fail fast with a clear error message.
  - Gitops already provides explicit issuers per environment (e.g. `https://auth.dev.ameide.io/realms/ameide`, `https://auth.ameide.io/realms/ameide`) via `keycloak.issuer`, making the realm effectively mandatory.

- **Onboarding telemetry improvements**
  - Introduced a dedicated onboarding meter in `services/www_ameide_platform/lib/onboarding-metrics.ts` to separate bootstrap funnel metrics from chat/runtime traffic.
  - All bootstrap responses now emit consistent metrics (status: `success` / `error` / `rate_limited`, HTTP status code, and duration), improving observability for abuse detection and “time to tenant reserved”.

- **Bootstrap error handling and UX**
  - Existing error mapping for tenant provisioning (`TENANT_SLUG_CONFLICT`, `TENANT_INPUT_INVALID`, transient ConnectError cases) remains in place; new rate limiting errors re-use the same logging and telemetry patterns so operators can correlate spikes in `429` with upstream capacity and abuse patterns.

**Known Gaps / Pending Work (Phase 1 and Adjacent)**

- Cluster-level verification that the **Keycloak master realm bootstrap** (`masterBootstrap`) and **ExternalSecret** wiring for `platform-app-master` are healthy and that service accounts actually hold `create-realm` + `realm-admin` in practice (not just in gitops manifests).
- Operational confirmation that the **platform service Helm releases** are enabled and stable in all target environments (dev/staging/production), including dependencies such as Postgres, Redis, and Envoy gateway.
- A dedicated **end-to-end onboarding test** that runs against a real cluster, exercising:
  - First-time login via Keycloak.
  - `/api/v1/registrations/bootstrap` → pending tenant creation and realm provisioning.
  - `/api/v1/registrations/complete` → organization + membership creation and tenant status transition to `active`.
  - Post-onboarding access to the main dashboard with the correct `tenantId` in JWT/session.

---

## Implementation Audit (2025-12-02)

### Audit Scope

Verified Phase 1 implementation status against requirements defined in this backlog. Audit covered:
- Frontend registration/onboarding components
- Backend API routes and rate limiting
- Identity orchestrator and tenant status transitions
- Invitation system implementation
- Keycloak admin integration
- Telemetry/metrics implementation

### Summary

| Component | Status | Notes |
|-----------|--------|-------|
| Registration Bootstrap API | ✅ Complete | Rate limiting, telemetry, error handling implemented |
| Identity Orchestrator | ✅ Complete | Atomic org creation, status transition to ACTIVE |
| Invitation System | ✅ Complete | 7 endpoints, SHA-256 hashing, timing-safe comparison |
| Keycloak Admin | ✅ Complete | `createTenantRealm`, `invalidateAdminTokenCache` |
| Onboarding Metrics | ✅ Complete | 4 metrics tracked via OpenTelemetry |
| E2E Tests | ✅ Complete | Comprehensive Playwright coverage |
| Tenant Status Mapping | ✅ Complete | `pending_onboarding` surfaced as first-class enum |

### Remediation Completed: `pending_onboarding` Status Mapping

**Location**: [services/platform/src/tenants/service.ts](services/platform/src/tenants/service.ts), [packages/ameide_core_proto/src/ameide_core_proto/platform/v1/tenants.proto](../packages/ameide_core_proto/src/ameide_core_proto/platform/v1/tenants.proto)

**Fixes implemented**:
- Proto: Added `TENANT_STATUS_PENDING_ONBOARDING = 4` to `TenantStatus`.
- Platform service: `statusToEnum`/`statusFromEnum` map `pending_onboarding` to/from the new enum.
- SDKs: TS SDK status union/mapping updated; regenerated proto bundles (TS/Go) so clients receive the correct enum.
- Validation: `services/platform` and `packages/ameide_sdk_ts` test suites pass after regeneration.

### Detailed Audit Findings

#### 1. Registration Bootstrap API

**Files Audited**:
- [services/www_ameide_platform/app/api/v1/registrations/bootstrap/route.ts](services/www_ameide_platform/app/api/v1/registrations/bootstrap/route.ts)
- [services/www_ameide_platform/lib/onboarding-metrics.ts](services/www_ameide_platform/lib/onboarding-metrics.ts)

**Findings**:
- ✅ Rate limiting: 10 attempts/hour per user (in-memory)
- ✅ Telemetry: `withRequestTelemetry` wrapper + 4 dedicated metrics
- ✅ Error handling: Structured JSON responses with `Retry-After` header
- ✅ Tenant slug validation and conflict detection

**Metrics Implemented**:
| Metric | Type | Labels |
|--------|------|--------|
| `onboarding_bootstrap_requests_total` | Counter | status, http_code |
| `onboarding_bootstrap_errors_total` | Counter | status, http_code |
| `onboarding_bootstrap_rate_limit_hits_total` | Counter | - |
| `onboarding_bootstrap_duration_seconds` | Histogram | status |

#### 2. Identity Orchestrator

**Files Audited**:
- [services/www_ameide_platform/features/identity/lib/orchestrator.ts](services/www_ameide_platform/features/identity/lib/orchestrator.ts)
- [services/www_ameide_platform/features/identity/types/index.ts](services/www_ameide_platform/features/identity/types/index.ts)

**Findings**:
- ✅ `completeRegistration()` creates organization + owner membership atomically
- ✅ Transitions tenant status to `ACTIVE` after org creation
- ✅ Uses `TenantStatus.ACTIVE` enum value correctly
- ✅ Writes realm metadata (slug, realm name) to tenant record

#### 3. Invitation System

**Files Audited**:
- [services/www_ameide_platform/app/api/v1/invitations/](services/www_ameide_platform/app/api/v1/invitations/)
- [services/www_ameide_platform/features/invitations/](services/www_ameide_platform/features/invitations/)

**Findings**:
- ✅ 7 API endpoints: create, validate, accept, revoke, resend, list, get by ID
- ✅ Token generation: 32-byte random tokens
- ✅ Token storage: SHA-256 hashing (raw token never stored)
- ✅ Token validation: Timing-safe comparison via `timingSafeEqual`
- ✅ Expiration: 7-day default with `expiresAt` field
- ✅ One-time use: Status set to `accepted` after use

#### 4. Keycloak Admin Integration

**Files Audited**:
- [services/www_ameide_platform/lib/keycloak-admin.ts](services/www_ameide_platform/lib/keycloak-admin.ts)

**Findings**:
- ✅ `createTenantRealm()`: Lines 1520-1621
  - Creates realm with proper defaults
  - Configures client scopes (`roles`, `tenant`)
  - Creates `platform-app` client with service account
  - Calls `invalidateAdminTokenCache()` after creation
- ✅ `invalidateAdminTokenCache()`: Lines 110-125
  - Clears cached admin token for specific config
  - Forces re-authentication on next API call
- ✅ `getKeycloakConfig()`: Strict realm parsing from `AUTH_KEYCLOAK_ISSUER`

#### 5. E2E Test Coverage

**Files Audited**:
- [services/www_ameide_platform/features/onboarding/__tests__/e2e/](services/www_ameide_platform/features/onboarding/__tests__/e2e/)

**Findings**:
- ✅ `onboarding-flow.spec.ts`: Full wizard journey
- ✅ `new-user-redirect.spec.ts`: Middleware redirect behavior
- ✅ Cluster-only mode (`INTEGRATION_MODE=cluster`)
- ✅ Covers: tenant bootstrap, org creation, redirect, post-onboarding access

### Audit Action Items

| Priority | Item | Owner | Estimate |
|----------|------|-------|----------|
| **P0** | Add `PENDING_ONBOARDING` to proto enum | Platform | 1h |
| **P0** | Update `statusToEnum()` mapping | Platform | 30m |
| P1 | Add integration test for status round-trip | QA | 2h |
| P2 | Verify Keycloak permissions in staging/prod clusters | Infra | 1h |

---

### Phase 2: Communications & Events (Sprint 3-4)

**Objective**: Automated notifications and downstream provisioning

| Task | Owner | Estimate | Dependencies |
|------|-------|----------|--------------|
| 2.1 Email service integration (SendGrid/SES) | Platform | 8h | - |
| 2.2 Invitation email templates | Design/Platform | 4h | 2.1 |
| 2.3 Welcome email on signup completion | Platform | 2h | 2.1 |
| 2.4 Implement outbox table for events | Platform | 4h | - |
| 2.5 Event relay to NATS/Kafka | Platform | 8h | 2.4 |
| 2.6 Provisioning consumer (default workspace) | Platform | 8h | 2.5 |
| 2.7 Notification consumer (welcome messages) | Platform | 4h | 2.5 |
| 2.8 Structured audit logging | Platform | 4h | 2.4 |

**Exit Criteria**:
- [ ] Invitation emails sent automatically
- [ ] Welcome email on signup
- [ ] `organization.created`, `membership.created` events published
- [ ] Default workspace created for new orgs
- [ ] Audit events recorded

### Phase 3: Enterprise Identity (Sprint 5-8)

**Objective**: Domain verification, SSO, and JIT provisioning

| Task | Owner | Estimate | Dependencies |
|------|-------|----------|--------------|
| 3.1 `org_domains` table migration | Platform | 2h | - |
| 3.2 Domain verification API (DNS TXT) | Platform | 8h | 3.1 |
| 3.3 Domain discovery API | Platform | 4h | 3.1 |
| 3.4 SSO connection management API | Platform | 16h | - |
| 3.5 Keycloak IdP brokering setup | Platform | 16h | 3.4 |
| 3.6 SSO enforcement in login flow | Platform | 8h | 3.3, 3.5 |
| 3.7 JIT membership provisioning | Platform | 8h | 3.6 |
| 3.8 Admin SSO wizard UI | Frontend | 16h | 3.4 |
| 3.9 SCIM 2.0 Users endpoint | Platform | 16h | - |
| 3.10 SCIM 2.0 Groups endpoint | Platform | 8h | 3.9 |

**Exit Criteria**:
- [ ] Org admin can verify domain ownership
- [ ] SSO-enforced domains redirect to IdP
- [ ] JIT creates membership on first SSO
- [ ] SCIM endpoint accepts provisioning requests

### Phase 4: Billing & Entitlements (Sprint 9-10)

**Objective**: Plan selection, payment, and seat management

| Task | Owner | Estimate | Dependencies |
|------|-------|----------|--------------|
| 4.1 Subscriptions/entitlements schema | Platform | 4h | - |
| 4.2 Payment provider integration (Stripe) | Platform | 16h | - |
| 4.3 Plan selection step in onboarding | Frontend | 8h | 4.1 |
| 4.4 Subscription creation workflow | Platform | 8h | 4.2 |
| 4.5 Seat limit enforcement | Platform | 4h | 4.1 |
| 4.6 Billing portal integration | Frontend | 8h | 4.2 |
| 4.7 Usage metering | Platform | 16h | 4.1 |

**Exit Criteria**:
- [ ] User can select plan during onboarding
- [ ] Payment method attached via Stripe
- [ ] Seat limits enforced on invitations
- [ ] Billing portal accessible

### Phase 5: Temporal Orchestration (Sprint 11-12)

**Objective**: Migrate to Temporal for robust, observable workflows

| Task | Owner | Estimate | Dependencies |
|------|-------|----------|--------------|
| 5.1 Temporal namespace for onboarding | Infra | 4h | - |
| 5.2 CustomerOnboardingWorkflow parent | Platform | 16h | 5.1 |
| 5.3 IdentityEnrollmentWorkflow child | Platform | 8h | 5.2 |
| 5.4 ProvisioningWorkflow child | Platform | 8h | 5.2 |
| 5.5 PaymentSetupWorkflow child | Platform | 8h | 5.2, 4.2 |
| 5.6 ContractWorkflow child (e-sign) | Platform | 8h | 5.2 |
| 5.7 KYCPipelineWorkflow child | Platform | 8h | 5.2 |
| 5.8 Signal handlers for human steps | Platform | 8h | 5.2 |
| 5.9 Search attributes for visibility | Platform | 4h | 5.2 |
| 5.10 Temporal UI dashboards | Infra | 4h | 5.2 |

**Exit Criteria**:
- [ ] Onboarding runs as Temporal workflow
- [ ] Human steps (email verify, MFA) handled via signals
- [ ] Compensation/saga on failures
- [ ] Workflow status visible in UI

---

## 5. Database Migrations Required

```sql
-- Phase 2: Events
CREATE TABLE platform.outbox_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(64) NOT NULL,
    aggregate_id VARCHAR(255) NOT NULL,
    event_type VARCHAR(128) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    published_at TIMESTAMPTZ,
    retry_count INTEGER DEFAULT 0
);

CREATE TABLE platform.audit_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id VARCHAR(255) NOT NULL,
    actor_user_id VARCHAR(255),
    action VARCHAR(128) NOT NULL,
    resource_type VARCHAR(64) NOT NULL,
    resource_id VARCHAR(255),
    payload JSONB NOT NULL DEFAULT '{}'::jsonb,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Phase 3: Enterprise Identity
CREATE TABLE platform.org_domains (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id VARCHAR(255) NOT NULL,
    organization_id VARCHAR(255) NOT NULL REFERENCES platform.organizations(id),
    domain VARCHAR(255) NOT NULL,
    verification_token TEXT,
    verified_at TIMESTAMPTZ,
    sso_required BOOLEAN DEFAULT FALSE,
    auto_join BOOLEAN DEFAULT FALSE,
    idp_alias VARCHAR(128),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ,
    UNIQUE (tenant_id, domain)
);

CREATE TABLE platform.sso_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id VARCHAR(255) NOT NULL,
    organization_id VARCHAR(255) NOT NULL REFERENCES platform.organizations(id),
    provider_type VARCHAR(64) NOT NULL, -- 'saml', 'oidc'
    idp_alias VARCHAR(128) NOT NULL,
    display_name VARCHAR(255),
    metadata_url TEXT,
    config JSONB NOT NULL DEFAULT '{}'::jsonb,
    enabled BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ
);

-- Phase 4: Billing
CREATE TABLE platform.subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id VARCHAR(255) NOT NULL,
    organization_id VARCHAR(255) NOT NULL REFERENCES platform.organizations(id),
    plan_id VARCHAR(128) NOT NULL,
    status VARCHAR(64) NOT NULL, -- 'trialing', 'active', 'past_due', 'canceled'
    provider_subscription_id VARCHAR(255),
    current_period_start TIMESTAMPTZ,
    current_period_end TIMESTAMPTZ,
    trial_end TIMESTAMPTZ,
    canceled_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ
);

CREATE TABLE platform.entitlements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id VARCHAR(255) NOT NULL,
    organization_id VARCHAR(255) NOT NULL REFERENCES platform.organizations(id),
    feature_key VARCHAR(128) NOT NULL,
    value_limit INTEGER,
    value_boolean BOOLEAN,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Phase 5: Temporal state (application-level)
CREATE TABLE platform.provisioning_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id VARCHAR(255) NOT NULL,
    organization_id VARCHAR(255),
    user_id VARCHAR(255),
    workflow_id VARCHAR(255),
    run_id VARCHAR(255),
    status VARCHAR(64) NOT NULL, -- 'pending', 'running', 'completed', 'failed'
    current_stage VARCHAR(128),
    error_message TEXT,
    metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ
);
```

---

## 6. API Surface

### New Endpoints Required

```
# Phase 1 (existing, needs fixes)
POST /api/v1/registrations/complete     -- Orchestrate signup
POST /api/v1/registrations/bootstrap    -- Self-serve tenant creation

# Phase 2
POST /api/v1/notifications/send         -- Internal notification dispatch
GET  /api/v1/provisioning/status/{id}   -- Check provisioning progress

# Phase 3
GET  /api/v1/domains/{domain}           -- Domain discovery
POST /api/v1/organizations/{id}/domains -- Claim domain
POST /api/v1/organizations/{id}/domains/{domainId}/verify -- Verify ownership
GET  /api/v1/organizations/{id}/sso     -- List SSO connections
POST /api/v1/organizations/{id}/sso     -- Create SSO connection
PUT  /api/v1/organizations/{id}/sso/{connId} -- Update SSO
DELETE /api/v1/organizations/{id}/sso/{connId} -- Remove SSO

# Phase 3 (SCIM)
GET    /scim/v2/Users                   -- List users
POST   /scim/v2/Users                   -- Create user
GET    /scim/v2/Users/{id}              -- Get user
PUT    /scim/v2/Users/{id}              -- Replace user
PATCH  /scim/v2/Users/{id}              -- Update user
DELETE /scim/v2/Users/{id}              -- Delete user
GET    /scim/v2/Groups                  -- List groups
POST   /scim/v2/Groups                  -- Create group

# Phase 4
GET  /api/v1/plans                      -- List available plans
POST /api/v1/organizations/{id}/subscription -- Create subscription
GET  /api/v1/organizations/{id}/subscription -- Get current subscription
POST /api/v1/organizations/{id}/subscription/portal -- Billing portal session
GET  /api/v1/organizations/{id}/entitlements -- List entitlements

# Phase 6 (Compliance)
POST /api/v1/users/{id}/export          -- GDPR data export
POST /api/v1/users/{id}/deletion-request -- GDPR deletion request
```

---

## 7. Temporal Workflow Design (Target State)

### State Ownership Model

> **Design Principle**: Temporal is the **source of truth** for workflow execution state. The `platform.provisioning_jobs` table is a **projection** for fast UI queries, not an authority.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TEMPORAL VS DB STATE OWNERSHIP                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Temporal (Source of Truth):                                            │
│    - Workflow execution state (running, completed, failed, canceled)    │
│    - Current stage / child workflow progress                            │
│    - Signal history (email verified, payment attached, etc.)            │
│    - Retry counts, timeouts, compensation state                         │
│                                                                         │
│  platform.provisioning_jobs (Projection):                               │
│    - Last-known workflow snapshot for UI/API queries                    │
│    - Updated via activities on state changes (not direct mutation)      │
│    - May lag behind Temporal by seconds during rapid transitions        │
│    - Queries: "show me all onboardings in 'verifying' stage"            │
│                                                                         │
│  Sync Mechanism:                                                        │
│    - StateSnapshotActivity runs after each major stage transition       │
│    - Writes to provisioning_jobs with workflow_id, run_id, stage        │
│    - Idempotent (same stage update is a no-op)                          │
│    - On query, if provisioning_jobs.updated_at > 5min, check Temporal   │
│                                                                         │
│  Recovery:                                                              │
│    - If DB projection is stale, Temporal is authoritative               │
│    - Replay sync worker can rebuild projections from workflow history   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Activity vs Workflow Separation

> **All external calls (Keycloak, Stripe, email, KYC, DB writes) are implemented as Activities, never directly inside workflow code.** This preserves Temporal's determinism guarantees.

| Type | Examples | Notes |
|------|----------|-------|
| **Workflow** (pure orchestration) | `CustomerOnboardingWorkflow`, `ProvisioningWorkflow` | No side effects, only control flow |
| **Activity** (side effects) | `createKeycloakRealm`, `sendWelcomeEmail`, `createStripeSubscription` | Retryable, timeout-bounded |
| **Signal Handler** | `EmailVerified`, `PaymentMethodAttached` | Human steps, awaited in workflow |

### Parent Workflow: CustomerOnboardingWorkflow

```typescript
export async function CustomerOnboardingWorkflow(input: OnboardingInput): Promise<OnboardingResult> {
  // 1) Lead capture & enrichment (optional CRM integration)
  const lead = await upsertLead(input);

  // 2) Identity enrollment (account + email verification)
  const identity = await executeChild(IdentityEnrollmentWorkflow, { email: input.email });

  // 3) MFA enrollment (optional based on policy)
  if (input.requireMFA) {
    await executeChild(MFAEnrollmentWorkflow, { userId: identity.userId });
  }

  // 4) B2B gates (domain verification, admin approval)
  if (input.accountType === 'B2B') {
    await executeChild(DomainVerificationWorkflow, { domain: input.companyDomain });
    await executeChild(AdminApprovalWorkflow, { userId: identity.userId });
  }

  // 5) Contract & payment (parallel)
  const [contract, payment] = await Promise.all([
    executeChild(ContractWorkflow, { userId: identity.userId, planId: input.planId }),
    executeChild(PaymentSetupWorkflow, { userId: identity.userId, planId: input.planId }),
  ]);

  // 6) Provisioning (with saga compensation)
  const provisioning = await executeChild(ProvisioningWorkflow, {
    userId: identity.userId,
    planId: input.planId,
    region: input.region,
  });

  // 7) Welcome & activation
  await executeChild(WelcomeAndActivationWorkflow, { userId: identity.userId });

  return { status: 'COMPLETED', userId: identity.userId };
}
```

### Signals (Human Steps)

- `EmailVerified(token)` - User clicked verification link
- `MFAEnrollmentCompleted(factorType)` - User enrolled MFA
- `ContractSigned(documentId)` - E-sign completed
- `PaymentMethodAttached(methodId)` - Card added
- `AdminApproved()` / `AdminRejected(reason)` - B2B gate decision
- `UserAborted(reason)` - User canceled onboarding

### Search Attributes

- `LeadId`, `Email`, `CustomerId`
- `AccountType` (B2C/B2B)
- `PlanId`, `Region`
- `LifecycleStage` (lead, verifying, provisioning, active)
- `KYCStatus`, `PaymentStatus`, `ProvisioningStatus`

---

## 8. Test Plan

### Unit Tests
- [ ] Identity orchestrator error handling
- [ ] Role assignment logic
- [ ] Token generation/validation
- [ ] Domain verification logic

### Integration Tests
- [ ] Registration API with mock Keycloak
- [ ] Invitation acceptance flow
- [ ] Provisioning with test tenant
- [ ] Event publishing to test bus

### E2E Tests (Playwright)
- [ ] Happy path: signup → org creation → dashboard
- [ ] Invitation flow: create → email → accept → org access
- [ ] Error paths: missing tenant, expired token, duplicate org slug
- [ ] Cross-tenant isolation verification

### Performance Tests
- [ ] 100 concurrent signups
- [ ] 1000 invitations/minute
- [ ] Realm provisioning latency

---

## 9. Dependencies

### Infrastructure
- Keycloak with realm-management permissions
- PostgreSQL with platform schema
- Temporal cluster for workflows (Phase 5)
- NATS/Kafka for event bus (Phase 2)
- Redis for session cache
- Vault for secret management

### External Services
- Email provider (SendGrid/SES)
- Payment provider (Stripe)
- E-signature (DocuSign/HelloSign) - optional
- KYC provider - optional

### Internal Services
- Platform service (gRPC)
- Agents service (realm-scoped)
- Workflows service

---

## 10. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Keycloak admin permissions incorrect | High | High | Automated permission verification in CI |
| Realm provisioning timeout | Medium | Medium | Async provisioning with status polling |
| Email deliverability issues | Medium | High | Transactional email provider, delivery monitoring |
| Payment integration complexity | Medium | Medium | Phased rollout, start with free tier |
| SCIM compatibility edge cases | High | Low | Extensive conformance testing |
| Temporal workflow versioning | Low | High | Feature flags, gradual migration |

---

## 11. Success Metrics

| Metric | Current | Phase 1 Target | Phase 5 Target |
|--------|---------|----------------|----------------|
| Signup success rate | 0% (blocked) | 95% | 99% |
| Time to first dashboard | N/A | < 2 min | < 90 sec |
| Invitation acceptance rate | N/A | 70% | 85% |
| SSO setup success | N/A | - | 90% |
| TTFHW (Time to First Happy Workflow) | N/A | < 10 min | < 5 min |

---

## 12. Cross-References & Dependency Map

### 12.1 Primary References (Onboarding-Specific)

| Backlog | Relationship | Key Dependencies |
|---------|--------------|------------------|
| [319-onboarding.md](./319-onboarding.md) | Superseded | Original implementation status |
| [319-onboarding-v4.md](./319-onboarding-v4.md) | Superseded | Temporal workflow design (incorporated) |
| [369-onboarding-v2.md](./369-onboarding-v2.md) | Superseded | Gap analysis summary |

### 12.2 Identity & Authentication Dependencies

| Backlog | Relationship | Alignment Requirements |
|---------|--------------|------------------------|
| [426-keycloak-config-map.md](./426-keycloak-config-map.md) | **Critical** | Realm provisioning must use `foundation-keycloak-admin-secrets` for admin credentials; realm imports via `KeycloakRealmImport` CR; ExternalSecrets owned by Layer 15 |
| [427-platform-login-stability.md](./427-platform-login-stability.md) | **Critical** | `/login` page uses PublicLayout (no server-side auth); Redis warmup on startup; circuit breaker on org fetch (5s timeout); onboarding pages must follow the same middleware-only auth patterns |
| [322-rbac.md](./322-rbac.md) | Required | Canonical roles (`admin`, `contributor`, `viewer`, `guest`, `service`) must be seeded during org creation |
| [331-tenant-resolution.md](./331-tenant-resolution.md) | Required | `tenantId` JWT claim required for all authenticated operations |
| [333-realms.md](./333-realms.md) | Roadmap | Realm-per-tenant architecture (future state) |

### 12.3 Secrets & Infrastructure Dependencies

| Backlog | Relationship | Alignment Requirements |
|---------|--------------|------------------------|
| [412-cnpg-owned-postgres-greds.md](./412-cnpg-owned-postgres-greds.md) | **Critical** | Platform DB credentials (`platform-db-credentials`) owned by CNPG; no Vault-authored DB secrets; rotation via Secret update |
| [418-secrets-strategy-map.md](./418-secrets-strategy-map.md) | Required | Follow secrets authority model: CNPG for DB, Vault for app secrets; no inline secrets in charts |
| [362-unified-secret-guardrails.md](./362-unified-secret-guardrails.md) | Required | ExternalSecrets for all platform credentials; fail-fast when secrets missing |

### 12.4 Observability Dependencies

| Backlog | Relationship | Alignment Requirements |
|---------|--------------|------------------------|
| [334-logging-tracing-v2.md](./334-logging-tracing-v2.md) | Required | Onboarding flows must emit OpenTelemetry spans with `onboarding.id`, `tenant.id`, `user.id` attributes; JSON logs with trace correlation; Grafana derived fields for `trace_id` |

### 12.5 Routing & GitOps Dependencies

| Backlog | Relationship | Alignment Requirements |
|---------|--------------|------------------------|
| [417-envoy-route-tracking.md](./417-envoy-route-tracking.md) | Required | Onboarding endpoints routed via Gateway API HTTPRoutes; `platform.dev.ameide.io` → `www-ameide-platform:3001` |
| [424-tilt-www-ameide-separate-release.md](./424-tilt-www-ameide-separate-release.md) | Development | Tilt releases (`*-tilt`) isolated from Argo baseline; dev testing via `platform.local.ameide.io` |

### 12.6 Alignment Matrix

| This Backlog Phase | Related Backlogs | Constraints |
|--------------------|------------------|-------------|
| **Phase 1: Core Path** | 426, 427, 412, 418 | Keycloak admin secrets from Layer 15; CNPG-owned DB creds; PublicLayout for login/register |
| **Phase 2: Events** | 334, 418 | OTel spans for all events; audit events follow logging blueprint |
| **Phase 3: SSO** | 426, 333 | IdP brokering via Keycloak operator; realm imports as PostSync hooks |
| **Phase 5: Temporal** | 334, 362 | Workflow secrets from Vault; spans with workflow/execution IDs |

---

## 13. Infrastructure Constraints (from Related Backlogs)

### 13.1 Secrets (from 412, 418, 462)

> **Secret origin classification:** All secrets referenced by onboarding follow the authority taxonomy in [462-secrets-origin-classification.md](./462-secrets-origin-classification.md). In particular, OIDC client secrets (`platform-app-master`) are **service-generated** and extracted by client-patcher—not sourced from Azure KV fixtures.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         SECRETS AUTHORITY MODEL                          │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────┐                                                          │
│  │   CNPG     │──────▶ platform-db-credentials                           │
│  │ (Postgres) │──────▶ keycloak-db-credentials                           │
│  └────────────┘        (CNPG is source of truth for DB roles/passwords)  │
│                                                                          │
│  ┌────────────┐                                                          │
│  │   Vault    │──────▶ keycloak-bootstrap-admin (via ExternalSecret)     │
│  │ + ESO L15  │──────▶ platform-app-master-client (via ExternalSecret)   │
│  └────────────┘──────▶ www-ameide-platform-auth (via ExternalSecret)     │
│                                                                          │
│  ⚠️ NEVER: Vault → CNPG secrets (Vault is downstream mirror only)        │
│  ⚠️ NEVER: Inline secrets in Helm charts                                 │
│  ⚠️ NEVER: Direct ALTER ROLE without updating Secret                     │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**Onboarding Implications:**
- Registration API uses `platform-db-credentials` (CNPG-owned) for DB access
- Keycloak realm provisioning uses `keycloak-bootstrap-admin` (Vault/ESO-owned)
- Frontend auth uses `www-ameide-platform-auth` (Vault/ESO-owned)

### 13.2 Keycloak (from 426)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      KEYCLOAK GITOPS LAYERING                            │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  foundation-keycloak-admin-secrets (Layer 15)                            │
│      │                                                                   │
│      ├── ExternalSecret keycloak-bootstrap-admin-sync                    │
│      ├── ExternalSecret keycloak-master-bootstrap-sync                   │
│      └── ExternalSecret keycloak-master-platform-app-sync                │
│                    │                                                     │
│                    ▼                                                     │
│  platform-keycloak (Keycloak CR)                                         │
│      │   - bootstrapAdmin.secretName: keycloak-bootstrap-admin           │
│      │   - database.secretName: keycloak-db-credentials (CNPG)           │
│      │   - externalSecrets.enabled: false (consumes, doesn't own)        │
│      │                                                                   │
│      ▼                                                                   │
│  platform-keycloak-realm                                                 │
│      │   - KeycloakRealmImport CRs (PostSync hook)                       │
│      │   - realm-master-bootstrap Job                                    │
│      │   - client-patcher Job                                            │
│      │                                                                   │
│      ▼                                                                   │
│  Onboarding provisioning (this backlog)                                  │
│      - Creates tenant realms via Keycloak Admin API                      │
│      - Must have realm-management/realm-admin permissions                │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**Onboarding Implications:**
- Realm provisioning requires `platform-app-master` service account to have `realm-admin` role
- After realm creation, must refresh admin token cache
- Realm imports are ephemeral (create-only, deleted after success)

### 13.3 Login Stability (from 427)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      ROUTE GROUP ARCHITECTURE                            │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  app/                                                                    │
│    (public)/           ← PublicLayout (no server-side auth, fast)        │
│      login/                                                              │
│      register/         ← Onboarding start (optional)                     │
│    (onboarding)/       ← OnboardingLayout (minimal auth)                 │
│      onboarding/                                                         │
│    (app)/              ← Authenticated app routes (use middleware        │
│                          headers: x-pathname, x-tenant-id, x-org-home)   │
│      org/[orgId]/                                                        │
│    healthz/route.ts    ← force-static, bypasses auth                     │
│                                                                          │
│  Performance Requirements:                                               │
│    - /login: ~4-6s subsequent requests (no org fetch)                    │
│    - /healthz: 50-200ms                                                  │
│    - Onboarding wizard: < 2s LCP                                         │
│                                                                          │
│  Circuit Breakers:                                                       │
│    - Redis: 3s timeout (dev), 10s (prod)                                 │
│    - Org fetch: 5s timeout, return [] on timeout                         │
│    - Redis warmup on app startup                                         │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**Onboarding Implications:**
- Onboarding wizard pages should use minimal auth (skip org context fetch and heavy session logic)
- Rely on middleware-injected headers (`x-tenant-id`, `x-user-id`, `x-org-home`) for routing decisions
- Add Redis JSON.parse error handling in session store
- Warmup Redis connection on app startup

### 13.4 Observability (from 334)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      TELEMETRY REQUIREMENTS                              │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Onboarding Spans (OpenTelemetry):                                       │
│    - app.onboarding.id                                                   │
│    - app.tenant.id                                                       │
│    - app.user.id                                                         │
│    - app.organization.id                                                 │
│    - app.invitation.id (for invite flows)                                │
│                                                                          │
│  JSON Logs (Pino/structlog):                                             │
│    { "trace_id": "...", "span_id": "...", "onboarding_id": "...",        │
│      "tenant_id": "...", "event": "organization.created", ... }          │
│                                                                          │
│  Grafana Dashboards:                                                     │
│    - Onboarding funnel metrics                                           │
│    - Signup latency percentiles                                          │
│    - Error rate by step                                                  │
│    - Derived field: trace_id for 1-click pivot                           │
│                                                                          │
│  Collector Pipeline:                                                     │
│    www-ameide-platform → otel-collector:4317 → Loki/Tempo/Prometheus     │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**Onboarding Implications:**
- All registration/onboarding endpoints must emit spans with attributes above
- Use structured logging with trace correlation
- Create onboarding-specific Grafana dashboard

### 13.5 Routing (from 417, 424)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      GATEWAY ROUTES (ONBOARDING)                         │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Dev (Argo Baseline):                                                    │
│    platform.dev.ameide.io → apps-www-ameide-platform:3001                │
│    auth.dev.ameide.io     → keycloak:8080                                │
│                                                                          │
│  Dev (Tilt Inner Loop):                                                  │
│    platform.local.ameide.io → apps-www-ameide-platform-tilt:3001          │
│    auth.dev.ameide.io      → keycloak:8080 (shared)                      │
│                                                                          │
│  Staging/Production:                                                     │
│    platform.{staging.}ameide.io → www-ameide-platform:3001               │
│    auth.{staging.}ameide.io     → keycloak:8080                          │
│                                                                          │
│  SCIM Endpoint (Phase 3):                                                │
│    api.{env}.ameide.io/scim/v2/* → platform:8082 (gRPC gateway)          │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 14. References

### Primary Sources
- [db/flyway/sql/platform/V1__initial_schema.sql](../db/flyway/sql/platform/V1__initial_schema.sql) - Current schema

### Identity & Auth
- [backlog/426-keycloak-config-map.md](./426-keycloak-config-map.md) - Keycloak GitOps architecture
- [backlog/427-platform-login-stability.md](./427-platform-login-stability.md) - Login performance fixes
- [backlog/322-rbac.md](./322-rbac.md) - Role-based access control
- [backlog/331-tenant-resolution.md](./331-tenant-resolution.md) - JWT tenant claims
- [backlog/333-realms.md](./333-realms.md) - Realm-per-tenant architecture

### Secrets & Infrastructure
- [backlog/412-cnpg-owned-postgres-greds.md](./412-cnpg-owned-postgres-greds.md) - CNPG credential ownership
- [backlog/418-secrets-strategy-map.md](./418-secrets-strategy-map.md) - Secrets authority model
- [backlog/362-unified-secret-guardrails.md](./362-unified-secret-guardrails.md) - Secret guardrails

### Observability
- [backlog/334-logging-tracing-v2.md](./334-logging-tracing-v2.md) - OpenTelemetry blueprint

### GitOps & Routing
- [backlog/417-envoy-route-tracking.md](./417-envoy-route-tracking.md) - Gateway route inventory
- [backlog/424-tilt-www-ameide-separate-release.md](./424-tilt-www-ameide-separate-release.md) - Tilt isolation

---

## Appendix A: Onboarding User Journey

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           SELF-SERVE SIGNUP (B2C)                               │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │  Email   │───▶│ Keycloak │───▶│ Middleware│───▶│ Wizard   │───▶│Dashboard │  │
│  │  Entry   │    │  Auth    │    │  Gate    │    │ 3 Steps  │    │  Ready   │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│       │               │               │               │               │         │
│       │               │               │               │               │         │
│       ▼               ▼               ▼               ▼               ▼         │
│   [Rate Limit]   [JWT+tenantId]  [hasOrg=false]  [Create Org]   [Provision]    │
│   [CAPTCHA]      [Email Verify]  [Redirect]      [Owner Role]   [Events]       │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│                           INVITATION FLOW                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │  Admin   │───▶│  Email   │───▶│  Invitee │───▶│  Accept  │───▶│   Org    │  │
│  │ Creates  │    │  Sent    │    │  Clicks  │    │   API    │    │  Access  │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│       │               │               │               │               │         │
│       │               │               │               │               │         │
│       ▼               ▼               ▼               ▼               ▼         │
│   [Validate]     [SHA-256]       [Auth Gate]    [Membership]    [Session]      │
│   [Rate Limit]   [Token URL]     [SSO Check]    [Role Assign]   [Refresh]      │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│                           ENTERPRISE SSO (B2B)                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │  Domain  │───▶│  Verify  │───▶│   IdP    │───▶│   JIT    │───▶│   SSO    │  │
│  │  Claim   │    │  DNS TXT │    │  Config  │    │ Provision│    │ Enforced │  │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘  │
│       │               │               │               │               │         │
│       │               │               │               │               │         │
│       ▼               ▼               ▼               ▼               ▼         │
│   [Admin]        [48h TTL]       [SAML/OIDC]    [Auto-Join]     [Policy]       │
│   [Org Owner]    [Polling]       [Keycloak]     [Default Role]  [Bypass Block] │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Appendix B: Decision Log

| Date | Decision | Rationale | Alternatives Considered |
|------|----------|-----------|------------------------|
| 2025-10 | Realm-per-tenant | Enterprise SSO isolation | Single realm with groups |
| 2025-10 | JWT tenantId | Stateless auth, session independence | Cookie-based tenant |
| 2025-11 | Temporal for orchestration | Durability, visibility, compensation | Direct async calls |
| 2025-11 | Vault for secrets | Unified guardrails, rotation | K8s secrets only |
| 2025-12 | Consolidated backlog | Single source of truth | Separate phase docs |
| 2025-12 | Hybrid tenancy model | Keycloak realm limits require tiered approach | Pure realm-per-tenant |
| 2025-12 | SCIM as highest identity authority | Enterprise IdP expectations | JIT-first approach |
| 2025-12 | Stripe as billing source of truth | Webhook-driven, avoid split-brain | DB-first approach |
| 2025-12 | Temporal owns workflow state | Projection pattern for UI | DB as source of truth |

---

## Appendix C: Architecture Review Summary

> **Review Date**: 2025-12-01
> **Status**: Highly aligned with vendor recommendations; key refinements applied

### Strong Alignment Areas

| Area | Assessment |
|------|------------|
| JWT + RLS tenant isolation | Matches SaaS architecture best practices |
| Invitation token security | SHA-256 hash, expiry, one-time use per vendor guidance |
| Temporal workflow design | Parent/child pattern with signals per Temporal docs |
| Transactional outbox + events | Canonical pattern for reliable event emission |
| Billing state model | Stripe subscription lifecycle correctly modeled |

### Key Refinements Applied (This Review)

| Issue | Resolution |
|-------|------------|
| Keycloak 10k realms/cluster unrealistic | Added hybrid tenancy model: ~500 enterprise realms + shared realm for SMB |
| SCIM endpoints underspecified | Added filtering, pagination, PATCH, OAuth scope requirements |
| Identity source precedence undefined | Documented SCIM > JIT > manual invites with edge cases |
| Billing source-of-truth ambiguous | Clarified: Stripe authoritative, DB is projection via webhooks |
| Temporal vs DB state ownership | Documented: Temporal authoritative, provisioning_jobs is projection |
| MFA priority too low | Moved org owner MFA to Phase 2 (required for admin roles) |
| SCIM security controls missing | Added rate limits, anomaly detection, audit logging |

### Remaining Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Keycloak shard routing complexity | Medium | High | Start with single cluster, defer sharding until 250+ enterprise tenants |
| SCIM IdP compatibility edge cases | High | Low | Extensive conformance testing against Entra, Okta, Google |
| SMB → Enterprise realm migration | Low | Medium | Design migration workflow early, test with pilot tenants |
| Trial → payment conversion friction | Medium | Medium | A/B test payment timing, default to trial-first |

### References

- [Keycloak Multi-Tenancy Guide](https://skycloak.io/blog/the-ultimate-best-guide-on-keycloak-multi-tenancy-part-1/)
- [Keycloak Realm Scalability Discussion](https://github.com/keycloak/keycloak/discussions/11074)
- [Microsoft Entra SCIM Implementation](https://learn.microsoft.com/en-us/entra/identity/app-provisioning/use-scim-to-provision-users-and-groups)
- [SCIM Best Practices](https://securityboulevard.com/2025/06/scim-best-practices-building-secure-and-extensible-user-provisioning/)
- [Stripe SaaS Billing Best Practices](https://stripe.com/resources/more/best-practices-for-saas-billing)
- [Temporal Workflow Best Practices](https://docs.temporal.io/best-practices)
