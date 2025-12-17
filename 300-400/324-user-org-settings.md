Below is a practical **information architecture + UX backlog** for a B2B, multi‑tenant SaaS with a **two‑column Settings layout** (left menu, center content). It's organized into **User Settings** and **Organization Settings**, with **MVP-first priorities**, **user stories & acceptance criteria**, and **Keycloak** (IdP) integration points called out where relevant.

> Onboarding and organization creation now flow through the realm-per-tenant architecture described in [`backlog/319-onboarding-v2.md`](./319-onboarding-v2.md). This backlog covers what users see **after** that flow completes.

---

## 📊 IMPLEMENTATION STATUS (as of 2025-10-30)

### ✅ Completed Features
- **User Profile**: Full profile page with personal info, roles, org memberships, Keycloak integration
- **User Settings**: Complete implementation - notifications, appearance (theme/density), preferences (language/timezone), privacy, security sections
- **Organization Settings**: Core features framework, graph management, transformation management, billing info display
- **Authentication**: NextAuth v5 + Keycloak, Redis sessions, token refresh, backchannel logout
- **Authorization**: Permission-based RBAC, role checks, route-level access control
- **Invitations**: Full invitation lifecycle (create, validate, accept, resend, revoke)
- **Header/Navigation**: User menu with org switcher, profile links, server-side RBAC filtering

### 🚧 Partially Implemented
- **Org Access Management**: Invitations, membership management, and teams now backed by platform APIs; custom roles management still incomplete
- **Repository/Initiative Settings**: Workflow sections exist; access control/collaborators are placeholders

### ❌ Not Yet Implemented
- User: API tokens (personal), connected accounts, data export/deletion
- Org: Team member assignment tooling, custom roles, policies (MFA/SSO/IP), SSO wizard, SCIM, webhooks, developer settings (API keys/OAuth), data management, compliance docs, danger zone

### 🔀 Deviations from Backlog
1. **Route structure**: Uses `/user/profile` and `/org/[orgId]/settings` instead of `/settings/me/*` and `/settings/org/:orgId/*`
2. **Layout**: Settings use tabs within pages rather than dedicated two-column shell
3. **User settings**: Implemented as single tabbed page rather than separate routes per section
4. **Org switcher**: In user menu dropdown instead of top header position
5. **Focus**: Current implementation prioritizes operational features (workflows, transformations, repos) over administrative features (SSO, teams, policies)

### 🎯 Next Priority Areas (per backlog MVP)
1. ✅ ~~Teams management UI~~ (2025-10-30 - needs backend)
2. ✅ ~~User management UI~~ (2025-10-30 - needs backend integration)
3. Backend APIs for teams (create, edit, delete, member assignment)
4. Backend integration for user management (suspend, remove with confirmation)
5. SSO Connections wizard + domain verification
6. Roles & Permissions matrix UI
7. Audit log (read-only view)
8. Webhooks configuration
9. Policies (MFA required, SSO only, session controls)
10. Two-column settings shell with proper URL structure (architectural decision)

---

## 0) Ground rules & layout

> **STATUS**: 🔀 **Partially implemented with deviations**

**Shell layout**

* **Global header (top):**
  - ✅ Product logo implemented
  - ❌ Tenant switcher not implemented (no multi-tenant UI yet)
  - 🔀 Organization switcher exists in user menu dropdown, not cascading in header
  - ❌ Global search (⌘/Ctrl‑K) not implemented
  - ❌ Environment badge not implemented
  - ❌ Help link not implemented
  - ✅ User avatar menu implemented
  - **Current location**: [HeaderClient.tsx](services/www_ameide_platform/features/header/components/HeaderClient.tsx), [HeaderUserMenu.tsx](services/www_ameide_platform/features/header/components/HeaderUserMenu.tsx)

* **Primary app nav (left):**
  - ✅ Implemented via contextual navigation system
  - Uses tabs at page level instead of persistent left sidebar
  - **Current location**: [features/navigation/](services/www_ameide_platform/features/navigation/)

* **Settings pages:**
  - ❌ **Two‑column layout not implemented**
  - 🔀 Uses single-page tabbed interface instead
  - 🔀 Routes are `/user/profile/settings` and `/org/[orgId]/settings` (not `/settings/me/*` or `/settings/org/:orgId/*`)
  - ❌ No dedicated settings shell with left sub-nav
  - 🔀 Content uses tabs with sections, not separate routes
  - ✅ Actions are generally right-aligned in forms
  - **Current location**: [app/(app)/user/profile/settings/page.tsx](services/www_ameide_platform/app/(app)/user/profile/settings/page.tsx), [app/(app)/org/[orgId]/settings/page.tsx](services/www_ameide_platform/app/(app)/org/[orgId]/settings/page.tsx)

**Roles (suggested)**

> **STATUS**: 🚧 **Partially implemented**

* ❌ **Platform Owner** (super-admin across tenants) - not implemented
* ❌ **Tenant Admin** (manages tenant-level items, all orgs inside tenant) - not implemented
* ✅ **Org Admin** (manages a single organization) - implemented via `admin` role
* ❌ **Security Admin** (SSO, policies) - not implemented as distinct role
* ❌ **Billing Admin** - not implemented as distinct role
* 🔀 **User Manager** (people/teams, invites) - partially via admin role; no dedicated role
* ❌ **Developer** (API, webhooks, OAuth clients) - not implemented
* ❌ **Auditor** (read-only config + logs) - not implemented
* ✅ **Member** (end-user) - implemented as default/`user` role
* **Current implementation**: Basic permission system in [lib/auth/authorization.ts](services/www_ameide_platform/lib/auth/authorization.ts) with roles: `admin`, `user`, `viewer`
* **Keycloak integration**: Roles extracted from Keycloak realm and client roles via [lib/keycloak.ts](services/www_ameide_platform/lib/keycloak.ts)

> **Auth model note (Keycloak):** ✅ **IMPLEMENTED** - Using Keycloak for **authentication & high-level identity** (OIDC/SAML, MFA, sessions). NextAuth v5 configured with Keycloak provider. Redis-backed session store. Token refresh coordination with distributed locks. Fine-grained permissions defined in app code. Single realm model in use. **GAP**: IdP brokering per organization not yet implemented.

---

## 1) User Settings (two‑column)

> **STATUS**: 🔀 **Implemented with different structure** - Uses single page with tabs instead of left menu navigation. Route: `/user/profile/settings` instead of `/settings/me/*`

**Left menu** (PLANNED vs ACTUAL)

1. ✅ Profile - Implemented at `/user/profile` (separate page)
2. ✅ Preferences (language, time zone, theme, accessibility) - Implemented as tab
3. ✅ Security (MFA/passkeys, credentials*) - Implemented as tab with link to Keycloak
4. ✅ Sessions & Devices - Implemented as security section
5. ✅ Notifications - Implemented as tab
6. ❌ API Tokens (personal) - Not implemented
7. ❌ Connected Accounts (SSO links / OAuth consents) - Not implemented
8. ✅ Organizations & Roles - Displayed on profile page
9. 🚧 Data & Privacy (exports, account deletion**) - Placeholder buttons exist, no backend
   * ✅ Credentials managed via Keycloak - deep link to account portal implemented
   ** 🚧 Account deletion UI exists but marked as placeholder

**Current implementation**: [app/(app)/user/profile/settings/page.tsx](services/www_ameide_platform/app/(app)/user/profile/settings/page.tsx) with tabs: Notifications, Appearance, Preferences, Privacy, Security

### 1.1 Profile (MVP)

> **STATUS**: 🔀 **Implemented with some differences**

* **Fields:**
  - ✅ name (full name field)
  - 🚧 avatar (shows initials, upload not implemented)
  - ✅ job title
  - ❌ phone - not implemented
  - ❌ preferred pronouns - not implemented
  - ✅ bio field (additional, not in spec)
  - ✅ email (read-only, from Keycloak)
  - ✅ user ID, Keycloak ID (read-only)
  - ✅ roles display (read-only)
  - ✅ organizations list with active indicators

* **Actions:**
  - ❌ upload avatar (with crop) - not implemented
  - ✅ save - implemented with validation
  - ✅ link to Keycloak account management

* **Key states:**
  - ✅ validation errors - implemented
  - ✅ success toast - implemented
  - 🚧 empty avatar - shows initials, no upload

* **Acceptance:**
  - ✅ Profile details persist and appear on reload
  - ❌ Avatar upload not yet implemented

**Current location**: [app/(app)/user/profile/page.tsx](services/www_ameide_platform/app/(app)/user/profile/page.tsx)
**API**: `GET/PATCH /api/user/profile` via `useUserProfile` hook

### 1.2 Preferences (MVP)

> **STATUS**: ✅ **Fully implemented**

* **Fields:**
  - ✅ language/locale - implemented
  - ✅ time zone (auto-detect + manual) - implemented
  - ✅ date formats - implemented
  - ❌ number formats - not separately implemented
  - ✅ **theme** (light/dark/system) - fully implemented in Appearance tab
  - ✅ **density** (comfortable/compact) - implemented as compact/comfortable/spacious
  - ✅ **accessibility** (reduced motion) - implemented
  - ❌ focus outlines - not separately configured
  - ✅ keyboard shortcuts toggle - implemented (bonus feature)

* **Acceptance:**
  - ✅ Preferences apply immediately with auto-save
  - ✅ Persisted per user via API

**Current location**: Preferences tab in [app/(app)/user/profile/settings/page.tsx](services/www_ameide_platform/app/(app)/user/profile/settings/page.tsx)
**API**: `GET/PATCH /api/user/settings` with auto-save via `useAutoSaveSettings` hook

### 1.3 Security (MVP)

> **STATUS**: 🔀 **Partially implemented with Keycloak delegation**

* **Sections:**
  - 🔀 **Two‑factor / MFA / Passkeys**: Shows "Manage MFA" link to Keycloak, no in-app UI
  - ✅ **Password / Credentials:** Deep link to Keycloak account management implemented
  - ✅ **Active Sessions**: Displays current session info
  - 🚧 **Account Actions**: Export data and delete account buttons (placeholders)

* **Keycloak integration:**
  - ✅ Deep link to Keycloak Account Console implemented
  - ❌ Admin API integration for viewing/managing factors not implemented
  - ❌ WebAuthn/passkey management not exposed

* **Acceptance:**
  - 🚧 User directed to Keycloak for MFA management (external)
  - ❌ Org MFA policy enforcement not implemented
  - ❌ Banner blocking exit until MFA configured - not implemented

**Current location**: Security tab in [app/(app)/user/profile/settings/page.tsx](services/www_ameide_platform/app/(app)/user/profile/settings/page.tsx)
**Keycloak link**: Opens external Keycloak account portal

### 1.4 Sessions & Devices (MVP)

> **STATUS**: 🚧 **Basic info shown, no management**

* **Table:**
  - ✅ Current session displayed (browser, device info)
  - ✅ Session expiry shown
  - ❌ Location/IP not shown
  - ❌ Created at not shown
  - ❌ **Revoke** action not implemented
  - ❌ Multiple device listing not implemented

* **Keycloak integration:**
  - ❌ Listing active sessions via Admin API not implemented
  - ❌ Revoke session functionality not implemented

* **Acceptance:**
  - ❌ Cannot revoke sessions yet

**Current location**: Security tab in [app/(app)/user/profile/settings/page.tsx](services/www_ameide_platform/app/(app)/user/profile/settings/page.tsx)

### 1.5 Notifications (MVP)

> **STATUS**: ✅ **Fully implemented**

* **Per-user overrides:**
  - ✅ Channels: Email, mobile push, browser notifications
  - ❌ Slack/Teams integration not implemented
  - ❌ Frequency settings not implemented (instant/daily/weekly)
  - ✅ Categories: Email updates, mentions, weekly digest
  - ❌ Specific categories (org invites, role changes, integration failures) not granular

* **Acceptance:**
  - ✅ Toggles persist with auto-save
  - ❌ Org enforcement/override indicators not implemented

**Current location**: Notifications tab in [app/(app)/user/profile/settings/page.tsx](services/www_ameide_platform/app/(app)/user/profile/settings/page.tsx)
**API**: `GET/PATCH /api/user/settings` with auto-save

### 1.6 API Tokens (personal) (P1)

> **STATUS**: ❌ **Not implemented**

* ❌ No personal API token management UI
* ❌ No backend for token creation/revocation
* **Gap**: Complete feature missing

### 1.7 Connected Accounts (P1)

> **STATUS**: ❌ **Not implemented**

* ❌ No connected accounts UI
* ❌ No OAuth consent management
* ❌ No Keycloak federated identity display
* **Gap**: Complete feature missing

### 1.8 Organizations & Roles (MVP)

> **STATUS**: ✅ **Implemented on profile page**

* **List:**
  - ✅ All orgs displayed with names
  - ✅ Active/current org indicator
  - ✅ Roles displayed separately in roles section
  - ❌ Team(s) not shown
  - ❌ **Leave organization** CTA not implemented

* **Acceptance:**
  - ✅ Accurate reflection of memberships
  - ❌ Cannot leave org (no UI/API)

**Current location**: Organizations section in [app/(app)/user/profile/page.tsx](services/www_ameide_platform/app/(app)/user/profile/page.tsx)

### 1.9 Data & Privacy (P2)

> **STATUS**: 🚧 **Placeholder UI only**

* 🚧 Export personal data - button exists, no backend
* ❌ Device/location history visibility - not implemented
* 🚧 Account deletion request - button exists, no backend

* **Acceptance:**
  - ❌ Export not functional
  - ❌ Deletion request not functional

**Current location**: Privacy tab in [app/(app)/user/profile/settings/page.tsx](services/www_ameide_platform/app/(app)/user/profile/settings/page.tsx)

---

## 2) Organization Settings (two‑column)

> **STATUS**: 🔀 **Implemented with different structure** - Uses tabs instead of two-column layout. Route: `/org/[orgId]/settings` instead of `/settings/org/:orgId/*`

> Scope is per **selected organization** within the current tenant. Show an org switcher just under the tenant switcher.
> 🔀 **Current**: Org switcher in user menu dropdown, not in settings header

**Left menu** (PLANNED vs ACTUAL)

1. 🚧 Overview - Partial (billing/identity cards only)
2. ❌ Profile & Branding - Not implemented
3. **Access** - 🚧 **Partially implemented**
   - 🔀 3.1 Users - Invitation system working, user list not in settings
   - ❌ 3.2 Teams - Not implemented
   - ❌ 3.3 Roles & Permissions - Not implemented as UI
   - ❌ 3.4 Policies (MFA required, SSO only, IP allowlist) - Not implemented
4. **Security** - ❌ **Not implemented**
   - ❌ 4.1 SSO Connections - Not implemented
   - ❌ 4.2 Provisioning (SCIM) - Not implemented
   - ❌ 4.3 Audit Log - Not implemented
5. **Integrations** - ❌ **Not implemented**
   - ❌ 5.1 Webhooks - Not implemented
   - ❌ 5.2 Apps (Slack/Teams/Jira/ServiceNow/…) - Not implemented
6. ❌ Notifications (org defaults) - Not implemented
7. 🚧 Billing (plan, invoices, payment methods) - Display only, no management
8. ❌ Developer (OAuth clients, org API keys) - Not implemented
9. ❌ Data Management (imports/exports, retention, residency) - Not implemented
10. ❌ Compliance & Legal (DPA, subprocessors, domain verification) - Not implemented
11. ❌ Danger Zone (org transfer/archival/delete) - Not implemented

**What IS implemented instead:**
- ✅ Features toggles (insights, graph, transformations, governance, workflows)
- ✅ Repositories management (create, edit, delete, set default)
- ✅ Initiatives management (create, edit, archive, set default)
- ✅ Risk taxonomy configuration
- ✅ Workflows catalog and runs
- ✅ Agents instances and node catalog

**Current location**: [app/(app)/org/[orgId]/settings/page.tsx](services/www_ameide_platform/app/(app)/org/[orgId]/settings/page.tsx)

### 2.1 Overview (MVP)

> **STATUS**: 🚧 **Minimal implementation**

* **Cards:**
  - ❌ org name - not on overview (shown in header)
  - ❌ domain(s) - not implemented
  - 🚧 plan/limits - shown in Billing & Identity card
  - ❌ seat usage - not implemented
  - 🚧 SSO status - shown in Billing & Identity card
  - ❌ outstanding tasks - not implemented

* **Acceptance:**
  - ❌ No real-time counts
  - ❌ No deep links to sections

**Current location**: Billing & Identity card in [app/(app)/org/[orgId]/settings/page.tsx](services/www_ameide_platform/app/(app)/org/[orgId]/settings/page.tsx)

### 2.2 Profile & Branding (MVP)

> **STATUS**: ❌ **Not implemented**

* ❌ No org profile editing
* ❌ No branding configuration (logo, colors)
* ❌ No support email field
* ❌ No default locale/timezone per org
* ❌ No business address

**Gap**: Complete section missing

### 2.3 Access

#### 2.3.1 Users (MVP)

> **STATUS**: ✅ **User management integrated with platform APIs (invite, role change, suspend/remove)**

* **Table:**
  - ✅ User management page at `/org/[orgId]/users`
  - ✅ User list with name, email, role(s), status (active/pending/suspended)
  - 🚧 Activity column shows membership timeline (true "last seen" still pending)
  - ✅ **Invite** - Invite flow backed by `/api/v1/invitations`
  - ✅ **Change role** - Updates membership roles via `/api/v1/organizations/[orgId]/memberships/:id`
  - ✅ **Suspend / Restore** - Membership state toggles persisted via PATCH
  - ✅ **Resend invite** - Uses invitation resend endpoint with success feedback
  - 🚧 **Remove user** - Deactivates membership (no confirmation dialog yet)

* **Invite flow (MVP):**
  - ✅ Enter email, choose role, optional message
  - ❌ Multiple emails not supported
  - ❌ Teams assignment not supported
  - ✅ Token-based magic link
  - ✅ Creates membership on acceptance

* **Acceptance:**
  - ❌ Cannot invite multiple emails at once
  - ✅ Pending invites tracked with expiry
  - ✅ Resend works
  - ✅ Suspension and removal actions take effect immediately

**Current location**:
- UI: [app/(app)/org/[orgId]/users/page.tsx](services/www_ameide_platform/app/(app)/org/[orgId]/users/page.tsx)
- Components: [app/(app)/org/[orgId]/users/components/](services/www_ameide_platform/app/(app)/org/[orgId]/users/components/)
- API: `/api/v1/invitations` and `/api/v1/organizations/[orgId]/invitations`
- API: `/api/v1/organizations/[orgId]/memberships` (list/update/delete memberships)
- Backend: [features/invitations/lib/service.ts](services/www_ameide_platform/features/invitations/lib/service.ts)
- Tests: [features/invitations/__tests__/](services/www_ameide_platform/features/invitations/__tests__/)

#### 2.3.2 Teams (MVP)

> **STATUS**: 🚧 **Team CRUD integrated with platform service; member assignment pending**

* **UI Implementation:**
  - ✅ Teams page at `/org/[orgId]/teams`
  - ✅ Card-based grid layout with team info
  - ✅ Team metadata: name, description, member count, visibility (public/private), created date
  - ✅ **Create Team** - Dialog backed by `/api/v1/organizations/[orgId]/teams`
  - ✅ **Edit Team** - Dialog updates via `/api/v1/organizations/[orgId]/teams/:id`
  - 🚧 **Manage Members** - Placeholder action; UI and API pending
  - ✅ **Delete Team** - Confirmation dialog, deletes via platform API
  - ✅ Empty state with call-to-action
  - ✅ Activity panel with stats and recent activity

* **Pending Backend:**
  - ❌ Member assignment system (add/remove users per team)
  - ❌ Team-scoped roles & permissions
  - ❌ Visibility/access controls enforcement beyond UI flag

**Current location**:
- UI: [app/(app)/org/[orgId]/teams/page.tsx](services/www_ameide_platform/app/(app)/org/[orgId]/teams/page.tsx)
- Uses ListPageLayout with ActivityPanel
- API: `/api/v1/organizations/[orgId]/teams` (list/create) and `/api/v1/organizations/[orgId]/teams/:teamId` (update/delete)

#### 2.3.3 Roles & Permissions (MVP)

> **STATUS**: 🚧 **Read-only matrix surfaced; editing & audits outstanding**

* **Default roles:**
  - ✅ Basic roles defined in code (`admin`, `user`, `viewer`)
  - ✅ Permission system exists
  - ✅ Roles matrix surfaced in org settings (read-only)
  - ❌ No custom roles
  - ❌ No UI for assigning capabilities

* **Acceptance:**
  - 🚧 Role counts visible, but editing must be done via API/Keycloak
  - ❌ Audit of role changes not implemented

**Current location**: Roles overview in [app/(app)/org/[orgId]/settings/page.tsx](services/www_ameide_platform/app/(app)/org/[orgId]/settings/page.tsx); backend definitions in [lib/auth/authorization.ts](services/www_ameide_platform/lib/auth/authorization.ts)

#### 2.3.4 Policies (P1)

> **STATUS**: ❌ **Not implemented**

* ❌ No MFA requirement policy
* ❌ No SSO-only enforcement
* ❌ No session length configuration
* ❌ No IP allowlist/deny
* ❌ No geographic restrictions
* ❌ No JIT user creation toggle

**Gap**: Complete feature missing

### 2.4 Security

#### 2.4.1 SSO Connections (MVP)

> **STATUS**: ❌ **Not implemented**

* ❌ No SSO wizard
* ❌ No provider selection (Okta, Azure AD, Google Workspace, Generic)
* ❌ No metadata/discovery URL input
* ❌ No attribute mapping
* ❌ No test connection
* ❌ No enforce toggle
* ❌ No domain verification
* ❌ No Keycloak IdP broker configuration via Admin API

**Gap**: Complete feature missing - this is a high-priority MVP item per backlog

#### 2.4.2 Provisioning (SCIM) (P1)

> **STATUS**: ❌ **Not implemented**

* ❌ No SCIM endpoint generation
* ❌ No SCIM token management
* ❌ No sync status
* ❌ No manual resync

**Gap**: Complete feature missing (P1 priority)

#### 2.4.3 Audit Log (MVP)

> **STATUS**: ❌ **Not implemented**

* ❌ No audit log UI
* ❌ No event filtering
* ❌ No export capability
* ❌ Backend event logging may exist but no user-facing view

**Gap**: Complete feature missing - this is an MVP item per backlog

### 2.5 Integrations

#### 2.5.1 Webhooks (MVP)

> **STATUS**: ❌ **Not implemented**

* ❌ No webhook configuration UI
* ❌ No event subscription
* ❌ No delivery log
* ❌ No secret management
* ❌ No retry policy configuration

**Gap**: Complete feature missing - this is an MVP item per backlog

#### 2.5.2 Apps (Slack/Teams/Jira/ServiceNow/…) (P1)

> **STATUS**: ❌ **Not implemented**

* ❌ No app gallery
* ❌ No OAuth integration flows
* ❌ No per-channel routing

**Gap**: Complete feature missing (P1 priority)

### 2.6 Notifications (org defaults) (MVP)

> **STATUS**: ❌ **Not implemented**

* ❌ No org-level notification policy configuration
* ❌ No category defaults
* ❌ No enforcement settings
* ❌ No routing configuration
* ❌ No templates

**Gap**: Complete feature missing - this is an MVP item per backlog

### 2.7 Billing (if applicable) (MVP)

> **STATUS**: 🚧 **Display only**

* **Plan & usage:**
  - ✅ Plan name displayed (Enterprise)
  - ✅ Renewal date shown
  - ❌ Seats consumed not tracked
  - ❌ Overage rules not displayed
  - ❌ No usage metrics

* **Invoices & payment methods:**
  - ❌ No invoice list
  - ❌ No payment method management
  - ❌ No PO info

* **Contacts:**
  - ❌ No billing contacts management

* **Acceptance:**
  - ❌ No permission gating shown
  - 🚧 Read-only display exists

**Current location**: Billing & Identity card in [app/(app)/org/[orgId]/settings/page.tsx](services/www_ameide_platform/app/(app)/org/[orgId]/settings/page.tsx)

### 2.8 Developer (P1)

> **STATUS**: ❌ **Not implemented**

* ❌ No org API keys
* ❌ No OAuth client management
* ❌ No sandbox/documentation

**Gap**: Complete feature missing (P1 priority)

### 2.9 Data Management (P1)

> **STATUS**: ❌ **Not implemented**

* ❌ No import/export UI
* ❌ No retention policies
* ❌ No residency selection

**Gap**: Complete feature missing (P1 priority)

### 2.10 Compliance & Legal (P1)

> **STATUS**: ❌ **Not implemented**

* ❌ No DPA access
* ❌ No subprocessors list
* ❌ No breach contacts
* ❌ No data request workflows
* ❌ No domain verification

**Gap**: Complete feature missing (P1 priority)

### 2.11 Danger Zone (P1)

> **STATUS**: ❌ **Not implemented**

* ❌ No transfer ownership
* ❌ No archive org
* ❌ No delete org

**Gap**: Complete feature missing (P1 priority)

---

## 3) Backlog: Epics → MVP / P1 / P2

### Epic A: Settings Shell & Navigation (MVP)

> **STATUS**: 🔀 **Partially implemented with different architecture**

* **Stories**
  - 🔀 Settings structure exists but different:
    - ❌ Not `/settings/me/:section` → Actually `/user/profile/settings` (single page)
    - ❌ Not `/settings/org/:orgId/:section` → Actually `/org/[orgId]/settings` (single page)
  - ❌ Left sub-nav not implemented (uses tabs instead)
  - ❌ Tenant switcher not implemented
  - 🔀 Org switcher exists in user menu dropdown (not header)
  - ❌ Keyboard search not implemented

* **Acceptance**:
  - 🔀 Tab anchors work, not separate routes
  - ✅ Responsive design implemented
  - 🔀 Uses tabs instead of collapsed nav on mobile

**Deviation**: Different UX pattern (tabs vs. two-column navigation)

### Epic B: User Settings (MVP)

> **STATUS**: ✅ **Mostly complete**

* ✅ Profile - implemented
* ✅ Preferences - implemented
* 🔀 Security (MFA/passkeys) - links to Keycloak
* 🚧 Sessions & Devices - basic display only
* ✅ Notifications - implemented
* ✅ Organizations & Roles - implemented

* **Acceptance**:
  - ✅ Forms validate and persist
  - ✅ Success toasts shown
  - ❌ Security actions not audited (no audit log)

### Epic C: Organization Access (MVP)

> **STATUS**: 🚧 **Users & teams integrated; custom roles and audit log outstanding**

* ✅ Users - invitations, role changes, suspend/remove backed by membership APIs
* ✅ Teams - create/edit/delete powered by platform TeamService
* 🚧 Roles & Permissions - backend only (read-only matrices now surfaced)
* ❌ Audit Log (read) - not implemented

* **Acceptance**:
  - ✅ Invites emailed and tracked with resend/cancel
  - ✅ User management page with live data, role/suspend/remove actions
  - ✅ Teams management page with live CRUD and delete confirmation
  - 🚧 Roles still require custom management UI/auditing
  - ❌ Audit log experience absent

**Progress**: Access CRUD flows land end-to-end; focus shifts to roles matrix, audit log, and policy enforcement

### Epic D: SSO (MVP)

> **STATUS**: ❌ **Not implemented**

* ❌ SSO Connections wizard - not implemented
* ❌ Domain verification - not implemented
* ❌ "Enforce SSO" policy - not implemented

**Major gap**: Complete MVP epic missing

### Epic E: Webhooks & Org Notifications (MVP)

> **STATUS**: ❌ **Not implemented**

* ❌ Webhook config - not implemented
* ❌ Delivery log - not implemented
* ❌ Org notifications defaults - not implemented

**Major gap**: Complete MVP epic missing

### Epic F: Developer & Provisioning (P1)

> **STATUS**: ❌ **Not implemented**

* ❌ Org API keys - not implemented
* ❌ OAuth clients - not implemented
* ❌ SCIM provisioning - not implemented

### Epic G: Policies & Security Hardening (P1)

> **STATUS**: ❌ **Not implemented**

* ❌ MFA required - not implemented
* ❌ SSO-only login - not implemented
* ❌ IP allowlist - not implemented
* ❌ Session lifetime - not implemented

### Epic H: Billing (if in-scope) (MVP)

> **STATUS**: 🚧 **Read-only display**

* 🚧 Plan display - shown
* ❌ Seats tracking - not implemented
* ❌ Invoices - not implemented
* ❌ Payment methods - not implemented

* **Acceptance**:
  - ❌ No permission gating
  - ❌ No invoice export

### Epic I: Data Management & Compliance (P1/P2)

> **STATUS**: ❌ **Not implemented**

* ❌ Import/export - not implemented
* ❌ Retention - not implemented
* ❌ Residency - not implemented
* ❌ DPA & subprocessors - not implemented

---

## 4) Keycloak touchpoints (practical)

> **IMPLEMENTATION STATUS**:

* **Login/SSO:**
  - ✅ OIDC redirect to Keycloak implemented via NextAuth v5
  - ❌ IdP brokering per org not implemented
  - ❌ Domain-based routing not implemented
  - **Current**: [app/(auth)/auth.ts](services/www_ameide_platform/app/(auth)/auth.ts)

* **MFA/Passkeys:**
  - ✅ Deep link to Keycloak Account Console implemented
  - ❌ Embedded controls via Admin API not implemented
  - ❌ In-app MFA status display not implemented
  - **Current**: Security tab links to external Keycloak portal

* **Sessions & Devices:**
  - ✅ Basic session info displayed (current session only)
  - ❌ Keycloak Admin API integration for listing sessions not implemented
  - ❌ Revoke session not implemented
  - **Current**: Redis-backed session store in [lib/session-store.ts](services/www_ameide_platform/lib/session-store.ts)

* **Invitations:**
  - ✅ Pending membership created in app DB
  - ✅ Token-based magic link system
  - ❌ Keycloak user pre-creation not implemented
  - ✅ Membership binding on acceptance works
  - **Current**: [features/invitations/lib/service.ts](services/www_ameide_platform/features/invitations/lib/service.ts)

* **Policies:**
  - ❌ SSO-only policy not implemented
  - ❌ MFA required policy not implemented
  - ❌ No Keycloak auth flow customization per org
  - **Gap**: Complete policy enforcement missing

* **SSO wizard:**
  - ❌ No SSO wizard UI
  - ❌ No Keycloak Admin API integration for IdP management
  - ❌ No attribute mappers configuration
  - ❌ No test login flow
  - **Gap**: Complete feature missing

* **Token claims:**
  - ✅ Roles extracted from Keycloak JWT via [lib/keycloak.ts](services/www_ameide_platform/lib/keycloak.ts)
  - ✅ Permission evaluation server-side in app
  - ❌ Custom tenantId/orgIds claims not in tokens
  - **Current**: Uses realm and client roles from standard claims

* **SCIM:**
  - ❌ SCIM not implemented in app
  - ❌ No user federation from external IdPs
  - **Gap**: Complete feature missing

---

## 5) Permission matrix (excerpt)

> **IMPLEMENTATION STATUS**: 🚧 **Basic role system exists; granular roles not implemented**

| Page / Action                 | Platform Owner | Tenant Admin |  Org Admin |  Sec Admin | Billing Admin | User Manager | Developer | Auditor | Member | **IMPLEMENTED?** |
| ----------------------------- | -------------: | -----------: | ---------: | ---------: | ------------: | -----------: | --------: | ------: | -----: | ---------------- |
| User Settings (self)          |              ✓ |            ✓ |          ✓ |          ✓ |             ✓ |            ✓ |         ✓ |       ✓ |      ✓ | ✅ All users |
| Org: Overview                 |              ✓ |            ✓ |          ✓ |          ✓ |             ✓ |            ✓ |         ✓ |       ✓ |   View | 🚧 Minimal |
| Org: Users (invite/suspend)   |              ✓ |            ✓ |          ✓ |            |               |            ✓ |           |    View |        | 🚧 Invite only |
| Org: Teams                    |              ✓ |            ✓ |          ✓ |            |               |            ✓ |           |    View |        | ❌ |
| Org: Roles & Permissions      |              ✓ |            ✓ |          ✓ |          ✓ |               |              |           |    View |        | ❌ No UI |
| Org: Policies                 |              ✓ |            ✓ |          ✓ |          ✓ |               |              |           |    View |        | ❌ |
| Org: SSO Connections          |              ✓ |            ✓ |          ✓ |          ✓ |               |              |           |    View |        | ❌ |
| Org: Audit Log                |              ✓ |            ✓ |          ✓ |          ✓ |             ✓ |            ✓ |         ✓ |       ✓ |        | ❌ |
| Org: Webhooks                 |              ✓ |            ✓ |          ✓ |            |               |              |         ✓ |    View |        | ❌ |
| Org: Notifications (defaults) |              ✓ |            ✓ |          ✓ |            |               |            ✓ |           |    View |        | ❌ |
| Org: Billing                  |              ✓ |            ✓ |       View |            |             ✓ |              |           |    View |        | 🚧 Display only |
| Org: Developer (API/OAuth)    |              ✓ |            ✓ |          ✓ |            |               |              |         ✓ |    View |        | ❌ |
| Danger Zone                   |              ✓ |            ✓ | ✓(guarded) | ✓(guarded) |               |              |           |    View |        | ❌ |

**Current roles in system**: `admin`, `user`, `viewer` (basic set)
**Missing specialized roles**: Platform Owner, Tenant Admin, Security Admin, Billing Admin, User Manager, Developer, Auditor

> ✅ **IMPLEMENTED**: Permission checks by **capabilities** in [lib/auth/authorization.ts](services/www_ameide_platform/lib/auth/authorization.ts) and [features/navigation/server/rbac.ts](services/www_ameide_platform/features/navigation/server/rbac.ts)

---

## 6) UX details & patterns

> **IMPLEMENTATION STATUS**:

* **Tables:**
  - ❌ Server-side pagination not consistently implemented
  - ❌ Column filters not implemented
  - ❌ Fuzzy search not implemented
  - ❌ Selectable rows not implemented
  - ❌ Bulk actions not implemented

* **Forms:**
  - ✅ Inline validation implemented
  - ✅ Error display implemented
  - ✅ Auto-save in user settings (no explicit "Save" button)
  - ❌ Unsaved changes guard not implemented

* **Empty states:**
  - 🚧 Some empty states exist
  - ❌ Not consistent across all features
  - ❌ Docs links not present

* **Toasts & banners:**
  - ✅ Success toasts implemented
  - ✅ Error toasts implemented
  - ❌ Persistent banners for required actions (e.g., MFA) not implemented

* **Auditability:**
  - ❌ Audit event system not exposed to users
  - ❌ No user-facing audit log
  - 🚧 Backend may log events but not accessible

* **Accessibility:**
  - ✅ Keyboard navigable
  - 🚧 Focus management on dialogs partially implemented
  - ✅ ARIA labels in components
  - ✅ Sufficient contrast
  - ✅ Motion-reduced animations option in preferences

* **i18n:**
  - ✅ Language preference setting exists
  - ❌ Strings not fully externalized
  - ❌ RTL support not implemented

---

## 7) Routes (example)

> **DEVIATION**: Current implementation uses different route structure

### PLANNED Routes (from backlog):
```
/settings/me/profile                              ❌ Not implemented
/settings/me/preferences                          ❌ Not implemented
/settings/me/security                             ❌ Not implemented
/settings/me/sessions                             ❌ Not implemented
/settings/me/notifications                        ❌ Not implemented
/settings/me/api-tokens                           ❌ Not implemented
/settings/me/connected-accounts                   ❌ Not implemented
/settings/me/memberships                          ❌ Not implemented

/settings/org/:orgId/overview                     ❌ Not implemented
/settings/org/:orgId/profile                      ❌ Not implemented
/settings/org/:orgId/users                        ❌ Not implemented
/settings/org/:orgId/teams                        ❌ Not implemented
/settings/org/:orgId/roles                        ❌ Not implemented
/settings/org/:orgId/policies                     ❌ Not implemented
/settings/org/:orgId/security/sso                 ❌ Not implemented
/settings/org/:orgId/security/scim                ❌ Not implemented
/settings/org/:orgId/security/audit-log           ❌ Not implemented
/settings/org/:orgId/integrations/webhooks        ❌ Not implemented
/settings/org/:orgId/integrations/apps            ❌ Not implemented
/settings/org/:orgId/notifications                ❌ Not implemented
/settings/org/:orgId/billing                      ❌ Not implemented
/settings/org/:orgId/developer                    ❌ Not implemented
/settings/org/:orgId/data                         ❌ Not implemented
/settings/org/:orgId/compliance                   ❌ Not implemented
/settings/org/:orgId/danger                       ❌ Not implemented
```

### ACTUAL Routes (current implementation):
```
/user/profile                                     ✅ Profile page with personal info
/user/profile/settings                            ✅ Settings page with tabs
  ?tab=notifications                              ✅ Notifications settings
  ?tab=appearance                                 ✅ Theme, density settings
  ?tab=preferences                                ✅ Language, timezone, date format
  ?tab=privacy                                    ✅ Privacy settings
  ?tab=security                                   ✅ Security settings

/org/[orgId]/settings                             ✅ Org settings page with sections
  (Features, Repositories, Initiatives, Billing,
   Governance, Workflows, Agents - all on one page)

/org/[orgId]/users                                ✅ User management (2025-10-30)
/org/[orgId]/teams                                ✅ Teams management (2025-10-30)

/org/[orgId]/repo/[repositoryId]/settings         ✅ Repository settings (placeholder)
/org/[orgId]/transformations/[transformationId]/settings  ✅ Initiative settings (placeholder)

API Routes (invitation system):
/api/v1/invitations                               ✅ Create/validate invitations
/api/v1/organizations/[orgId]/invitations         ✅ List org invitations
/api/v1/invitations/[id]/resend                   ✅ Resend invitation
/api/user/profile                                 ✅ User profile CRUD
/api/user/settings                                ✅ User settings CRUD
```

---

## 8) User stories & acceptance (sample, condensed)

> **IMPLEMENTATION STATUS**:

**As an Org Admin, I can invite users**

* ✅ When I submit valid email and a role, the system creates pending invite, sends email
* ✅ Shows as "Pending" with expiry (backend tracks this)
* ✅ Email links membership without duplicating accounts
* ❌ **Gap**: No UI list to view pending invites
* ❌ **Gap**: Cannot submit multiple emails at once

**As a Security Admin, I can configure SSO and enforce it**

* ❌ **NOT IMPLEMENTED** - No SSO wizard
* ❌ Cannot add IdP
* ❌ Cannot test connection
* ❌ Cannot verify domain ownership
* ❌ No "Enforce SSO" toggle
* **Major gap**: Complete story not implemented

**As a Member, I can register MFA or a passkey**

* 🔀 **PARTIAL** - Redirected to Keycloak for MFA setup
* ❌ No in-app factor management
* ❌ Org policy for required MFA not implemented
* ❌ No blocking mechanism if MFA required

**As an Auditor, I can export audit logs**

* ❌ **NOT IMPLEMENTED** - No audit log UI
* ❌ Cannot filter by date range, actor, action
* ❌ Cannot export CSV
* **Major gap**: Complete story not implemented

---

## 9) Data model (minimum viable)

> **IMPLEMENTATION STATUS**:

* **Tenant(id, name, …)** - ❌ Not implemented (no multi-tenant data model yet)
* **Organization(id, tenantId, name, domains[], branding, policies, …)** - 🚧 Basic org exists, no domains/branding/policies
* **User(id, email, name, keycloakUserId, …)** - ✅ Implemented (synced from Keycloak)
* **Membership(id, userId, orgId, teams[], roles[])** - 🚧 Basic membership exists, no teams field
* **Role(id, orgScope, name, capabilities[])** - 🚧 Roles exist in code, not in DB as entities
* **Team(id, orgId, name, members[])** - ❌ Not implemented
* **Invite(id, orgId, email, rolePreset, inviterId, expiresAt, status)** - ✅ Implemented
* **AuditLog(id, actorUserId, action, targetType, targetId, metadata, ip, ua, createdAt)** - ❌ Not exposed to users
* **Webhook(id, orgId, url, secret, events[], status)** - ❌ Not implemented
* **Delivery(id, webhookId, eventId, status, code, latencyMs, responseSnippet)** - ❌ Not implemented
* **SSOConnection(id, orgId, provider, metadata, status, enforced)** - ❌ Not implemented
* **NotificationPreference(userId, category, channel, value, enforcedByOrg?)** - 🚧 Basic preferences exist, no org enforcement

**Current backend**: Platform service manages orgs and memberships. User service syncs shadow users from Keycloak.

---

## 10) Implementation tips (Keycloak)

> **IMPLEMENTATION STATUS**:

* ✅ **OIDC** - Using OIDC with Keycloak via NextAuth v5
* ❌ **Attribute mappers** - Using standard claims, no custom mappers yet
* ❌ **Domain → IdP routing** - Not implemented (no domain verification or per-org IdP)
* ✅ **"Change password"** - Deep link to Keycloak Account Console implemented (no credential storage in app)
* ✅ **App-level authorization** - Centralized permission system in [lib/auth/authorization.ts](services/www_ameide_platform/lib/auth/authorization.ts)
* ❌ **`/whoami` endpoint** - Not explicitly implemented (uses session data directly)

**Recommendations for next steps**:
1. Implement domain verification system
2. Add per-org IdP brokering via Keycloak Admin API
3. Create `/api/whoami` endpoint for faster capability checks
4. Configure attribute mappers for custom claims (tenantId, orgIds)

---

## 11) What to build first (recommended MVP order)

> **PROGRESS CHECK**:

1. **Settings shell + switchers** - 🔀 Different approach (tabs instead of two-column)
2. **User Settings:** Profile, Preferences, Security, Sessions, Notifications, Memberships - ✅ **MOSTLY DONE**
3. **Org Settings:** Overview, Users, Teams, Roles & Permissions, Audit Log - 🚧 **Users (invite only), rest missing**
4. **SSO wizard + Domain verification** - ❌ **NOT STARTED** ⚠️ High priority gap
5. **Webhooks + Org notifications defaults** - ❌ **NOT STARTED** ⚠️ High priority gap

**Next recommended priorities** (based on gaps vs. backlog):
1. 🔴 **SSO wizard + Domain verification** (Epic D - MVP)
2. 🔴 **Audit Log (read-only)** (Epic C - MVP)
3. 🔴 **Teams management** (Epic C - MVP)
4. 🔴 **User management UI** (list, suspend, role changes)
5. 🔴 **Webhooks** (Epic E - MVP)
6. 🟡 **Org notifications defaults** (Epic E - MVP)
7. 🟡 **Two-column settings shell** (Epic A - architectural decision needed)

---

## 12) Definition of Done (settings features)

> **CURRENT COMPLIANCE**:

* ✅ Permission gates enforced on API and UI - Implemented via RBAC system
* ❌ Audit events for all privileged actions - No user-facing audit system
* 🚧 Empty states, error states, and loading skeletons - Partially implemented
* ✅ Accessibility checks (keyboard, screen reader labels) - Implemented
* 🚧 i18n keys in place - Language setting exists but strings not externalized
* ❌ Telemetry for key funnels - Not implemented

**Additional DoD items to consider**:
* ✅ Forms validate and persist correctly
* ✅ Success/error feedback via toasts
* ✅ Mobile responsive design
* ❌ Unsaved changes guards
* ❌ Comprehensive test coverage for all settings

---

## 📋 SUMMARY: Critical Gaps vs. Backlog MVP

### 🔴 Missing MVP Features (High Priority):
1. **SSO wizard + Domain verification** - Complete epic missing
2. **Audit Log** - No user-facing view
3. **Custom roles & matrix management** - No admin UI for role composition
4. **Team member assignment** - Cannot add/remove users to teams from UI
5. **Webhooks** - Complete feature missing
6. **Org notification defaults** - Complete feature missing

### 🟡 Architectural Deviations:
1. **Route structure** - Using tabs vs. separate routes
2. **Layout pattern** - Single page tabs vs. two-column navigation
3. **Org switcher location** - Dropdown vs. header
4. **Settings organization** - Operational features prioritized over admin features

### ✅ Strengths:
1. Solid authentication foundation (Keycloak + NextAuth)
2. Working invitation system
3. Complete user settings implementation
4. Permission-based RBAC framework
5. Good test coverage for implemented features

---

**Strategic recommendation**: With access tooling in place, focus on enterprise controls—SSO configuration, audit logging, custom roles/policies, and team membership assignment—before expanding integrations. Re-evaluate whether to refactor to a dedicated settings shell or continue iterating on the current tabbed layout.
