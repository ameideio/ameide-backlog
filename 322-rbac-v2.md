# Backlog Item 322: RBAC Status Snapshot (v2)

**Date**: 2025-11-01  
**Owner**: Platform Authentication & Authorization  
**Related**: [320-header.md](./320-header.md), [323-keycloak-realm-roles.md](./323-keycloak-realm-roles.md), [333-realms.md](./333-realms.md), [329-authz.md](./329-authz.md)

---

## 1. Current Position

| Area | Status | Notes |
|------|--------|-------|
| Canonical roles | ✅ `admin`, `contributor`, `viewer`, `guest`, `service` driven end-to-end (Keycloak import, proto/SDK, DB seeds, helpers, tests). | No prefixed or legacy literals remain in code/tests. |
| Migration workflows | 🟡 Single Flyway baseline (`V1__initial_schema.sql`) committed; all incremental migrations removed pending rebuild. | Need to regenerate schema deltas before staging/prod cutover. |
| Enforcement | 🟡 Core HTTP/server actions use canonical guards, but threads/onboarding/legacy APIs still bypass authorization. | Finish coverage for threads history/stream, onboarding status, and invitation mutations. |
| Tests | 🟡 Helper/unit coverage in place; end-to-end RBAC scenarios missing for threads/onboarding paths. | Add regression tests for role-based menu gating and API guards before GA. |
| Observability | ⚠️ Minimal dashboarding; no dedicated 401/403 trend monitoring. | Needs work before multi-tenant GA. |

---

## 2. Gaps / Follow-ups

| Gap | Owner | Target |
|-----|-------|--------|
| Membership & invitation enforcement parity (owner overrides, negative paths) | Platform API team | piggyback backlog/329-authz, Q1 FY26 |
| Systematic coverage thresholds (CI gating, integration assertions for every route) | DX tooling | align with testing council, Q4 FY25 |
| Authorization telemetry (403/401 dashboards, anomaly alerts) | Observability | add to Alloy/Grafana roll-up, next sprint |
| RBAC regression automation cadence (quarterly Playwright + integration sweep) | Platform QA | seed schedule + env prerequisites, Q1 FY26 |
| Customer-managed roles/catalog support | Platform product | design doc in progress; keep realm bootstrap minimal until spec lands |

---

## 3. Future Direction

1. **Capability-based delegation** – research policy-as-code (OPA/Rego) or capability token service so automation/cross-org delegation can extend beyond simple roles without fragmenting realms.
2. **Minimal realm import** – continue shipping only bootstrap roles (`admin`, `service`) and drive customer-defined roles through platform APIs to avoid per-tenant divergence.
3. **Residency-aware RBAC** – start tagging service accounts/data pipelines now so jurisdictional policies (EU/US partitions, data-processing approvals) can be layered without structural rewrites.

---

## 4. Checklist Refresher

- [x] Canonical role catalog in code/tests
- [ ] Flyway delta migrations published
- [ ] Realm bootstrap verified outside local
- [ ] Ownership enforcement extended to threads/onboarding surfaces
- [ ] Coverage gating + dashboards in place
- [ ] Authorization telemetry live
