Edge Layer

Marketing → product handoff is manual, so users still get dropped between ameide.io and the platform even though the reference flow expects a single, progressive orchestrator (backlog/319-onboarding.md:205-209).
Critical onboarding screens are missing smart domain routing, in-wizard teammate invites, default workspace templates, and the first-task checklist that should drive TTFHW (backlog/319-onboarding.md:644-660).
The new /registrations/bootstrap endpoint is not rate-limited or instrumented, leaving the very first API that a prospect hits without abuse controls or funnel metrics (backlog/319-onboarding.md:186).
Identity & Access

Self-serve onboarding still depends on a Keycloak admin creating the account and setting tenantId, and right now realm provisioning 403s while the platform service is disabled, so the “secure-by-default” path does not actually complete end‑to‑end (backlog/319-onboarding.md:27-33,170-176,322-325).
Domain discovery, IdP brokering, SSO enforcement, and SCIM/JIT auto-join are all flagged as “not implemented,” leaving enterprise flows and tenant isolation guardrails out of scope (backlog/319-onboarding.md:215-219,276-280,422-444,592-595,679-682).
Passwordless/WebAuthn/MFA policy hooks—explicitly called out in the reference goals—are not wired up even though Keycloak supports them (backlog/319-onboarding.md:648-649).
Tokens don’t carry per-org role claims and there is no tenant-switch capability, so downstream services cannot make ABAC decisions at the edge as recommended (backlog/319-onboarding.md:458-477).
Tenant & Organization Layer

The canonical data model is missing the org_domains table and SSO policy fields that power domain claims, so you cannot enforce verified domains or auto-enroll policies (backlog/319-onboarding.md:299-305).
The platform schema only defines organizations, tenants, roles, memberships, teams, invitations, and users—there is no place to store seat caps, domain metadata, or per-tenant residency settings required by the reference architecture (db/flyway/sql/platform/V1__initial_schema.sql:13-181).
Tenant catalog automation is still pending; no job moves pending_onboarding → active, so provisioning is not idempotent and cannot safely retry (backlog/319-onboarding.md:31,184-187).
Billing & Entitlements

The API gateway is supposed to expose catalog, plans, and billing endpoints, but the backlog explicitly lists Billing/Plans as “not implemented,” and Phase‑2 items still include billing integration and entitlements (backlog/319-onboarding.md:215-219,679-684).
There are no subscriptions, entitlements, or usage tables in the platform database, so there is nowhere to record plan limits or trial state as required by the reference data model (db/flyway/sql/platform/V1__initial_schema.sql:13-181).
Provisioning & Configuration

The event-driven provisioning loop (outbox + Kafka/NATS) is missing, so Services A/B/Search/Audit cannot self-seed when a user/org is created; the backlog calls out that outbox rows, event bus, and idempotent consumers are all unbuilt (backlog/319-onboarding.md:221-228,572-587).
Default workspace/project/template creation is absent, so new tenants land in an empty experience instead of the sample data/checklist described in the reference flow (backlog/319-onboarding.md:655).
Platform dependencies are unhealthy (Keycloak realm roles missing, platform helm release disabled), so provisioning cannot succeed or rollback cleanly—contrary to the idempotent/saga guidance (backlog/319-onboarding.md:170-176).
Communications Layer

Invitation creation just returns the token/URL and never dispatches email, leaving notification, locale handling, and audit of outbound messages unimplemented (services/platform/src/invitations/service.ts:198-205; backlog/319-onboarding.md:161-164,692-699).
Lifecycle messaging (welcome emails, nudges, support hand-offs) and notification-channel seeding are explicitly listed as not implemented, so there is no communications pipeline backing onboarding KPIs (backlog/319-onboarding.md:579-581,659-660).
Data & Governance

Audit logs, PII encryption at rest, GDPR/DSR endpoints, and email-domain takeover protections are missing, so compliance guardrails from the reference stack are not met (backlog/319-onboarding.md:299-307,601-607).
pgAudit or equivalent database auditing is not configured even though RLS is, so you have no immutable record of admin/billing actions (backlog/319-onboarding.md:633-637).
Without DomainClaims/Entitlements/Usage tables there is no way to express regulatory residency, plan limits, or feature ownership boundaries (db/flyway/sql/platform/V1__initial_schema.sql:13-181).
Observability & Metrics

Funnel metrics, analytics pipelines, tracing that spans web→API→event bus, and alerting on onboarding failures are all absent, which undermines the “measure TTFHW” goal (backlog/319-onboarding.md:613-620).
Even the foundational event bus is not deployed in Kubernetes, so there is no telemetry stream for downstream consumers (backlog/319-onboarding.md:637-638).
Bootstrap monitoring/rate limiting remains on the TODO list, so abuse/spam or stuck tenants go undetected (backlog/319-onboarding.md:186).
Security & Compliance

High-risk controls—rate limiting/CAPTCHA on signup, PII encryption, DSR tooling, audit logs, and domain-verification gates—are still unimplemented, leaving the system short of “secure by default” (backlog/319-onboarding.md:600-607).
Keycloak admin privileges are misconfigured (missing realm-admin/create-realm), so provisioning currently 403s instead of compensating gracefully, violating the guardrail that provisioning must be idempotent and observable (backlog/319-onboarding.md:27-30).
Key Journeys

Self-serve signup cannot complete without manual Keycloak work and working platform service, so the core “signup → verify → org/workspace → first action” journey stalls before reaching the app (backlog/319-onboarding.md:27-33,170-176,322-325).
Invitation acceptance lacks enterprise guardrails such as domain-enforced SSO, so invited users can still bypass required IdPs (backlog/319-onboarding.md:416-417).
Enterprise SSO, JIT provisioning, and SCIM de/provisioning flows are entirely absent, which means the enterprise journey described in the reference architecture does not exist today (backlog/319-onboarding.md:422-444,590-596).
Because no events/outbox/webhooks are emitted, downstream systems never hear about user_signed_up, org_created, or workspace_provisioned, so provisioning jobs, nudges, billing, and analytics cannot react to onboarding milestones (backlog/319-onboarding.md:572-587,748-751).
Suggested next steps

Stabilize the core path: restore platform service + Keycloak privileges, add bootstrap rate limiting/instrumentation, and automate the pending_onboarding → active transition before onboarding more users.
Close Identity gaps: ship Domain Discovery + verification, enforce IdP routing, and add tokenized role claims/tenant switching so downstream services can honor least privilege.
Lay the foundations for Billing/Provisioning: add the missing schema (subscriptions/entitlements/domain_claims/provisioning_jobs), implement the outbox + event bus, and wire notification + analytics consumers so future flows (billing, nudges, provisioning) have data to build on.


