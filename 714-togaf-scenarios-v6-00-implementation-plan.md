---
title: "714 — v6 Scenario Slices v6-00 (Implementation Plan: Increment 0 — Onboard + Explore baseline)"
status: draft
owners:
  - platform
  - transformation
  - repository
  - ui
  - agents
created: 2026-01-22
updated: 2026-01-22
related:
  - 496-eda-principles-v6.md
  - 714-togaf-scenarios-v6.md
  - 714-togaf-scenarios-v6-00.md
  - 710-gitlab-api-token-contract.md
  - 537-primitive-testing-discipline.md
---

# 714 v6-00 — Implementation Plan (Increment 0: Onboard + Explore baseline)

This is the **implementation plan** for `backlog/714-togaf-scenarios-v6-00.md` (Increment 0).

It is written to be:

1. **Contract-first** (proto-governed gRPC per `backlog/496-eda-principles-v6.md`),
2. **Local-only deterministic** (Phase 0/1/2 via `ameide test`; no cluster dependencies),
3. Aligned with the decision: **“client-go is the adapter; remove wrappers”**.

---

## 0) Scope and constraints

### In scope (Increment 0)

* A human and an agent can onboard a repository mapping, resolve `published` to a deterministic SHA, browse tree, open a file, and see citations.
* **CQRS rule:** Domain is command-only at the platform seam; Projection serves reads.
* Domain uses GitLab via `gitlab.com/gitlab-org/api/client-go` (no custom wrapper clients).
* Projection is **purely derived** from Domain facts (outbox→Kafka) and does **not** call GitLab.
* Local test strategy uses **mocks** (no HTTP stubs required):
  * Domain unit tests: `gitlab.com/gitlab-org/api/client-go/testing` (gomock TestClient).
  * Projection unit tests: apply Domain facts to projection state and assert read queries.

### Out of scope (Increment 0)

* Real GitLab dependency in Phase 0/1/2.
* Cluster-only E2E (Playwright / deployed environments); those remain `ameide test cluster`.

---

## 1) Current repo starting point (what already exists)

This plan intentionally builds on existing harnesses rather than inventing a new testing stack.

### Capability harnesses already present

* Deterministic, local-only Transformation capability integration harness exists:
  * `capabilities/transformation/features/enterprise_repository/integration/v6_togaf_scenarios_test.go`
* Domain/Projection DB harnesses already exist for in-process calls:
  * `primitives/domain/transformation/testkit/enterprise_repository_db_harness.go`
  * `primitives/projection/transformation/testkit/enterprise_repository.go`

### EDA wiring posture already used by capability tests

The current harness demonstrates a minimal owner→fact→projection loop:

1. Domain command writes to Domain DB and emits outbox events.
2. Test harness dispatches outbox once (in-process).
3. Projection store applies facts deterministically.

This matches the `496` doctrine (owner emits facts; projection derives).

---

## 2) Architecture rule update required in 714 docs (to match “no wrappers”)

Some 714 text currently encodes: “Projection owns reads and calls GitLab directly”.

With CQRS + outbox-derived Projection, the rule must become **seam-based**:

* ✅ Domain implementation code may import and use `gitlab.com/gitlab-org/api/client-go`.
* ❌ No GitLab client types cross platform contracts (proto/SDK/view models).
* ❌ UI/Agent/Process/Projection do not call GitLab (no direct REST, no `client-go`).
* ✅ Projection is fed by Domain facts (Kafka/outbox) and serves all read queries.

Increment 0 should update the Slice 0 doc text accordingly (remove import-boundary checks that no longer reflect the code).

---

## 3) Vendor-semantics hardening tasks (Domain)

Increment 0’s “onboard + explore baseline” can work without these hardenings, but we should bake them in early because they directly impact deterministic local tests and later increments.

### 3.1 Domain — `PublishChange` merge semantics

* Add bounded retry for transient “not mergeable yet” results:
  * retry on **405/422** a few times (short backoff),
  * treat **409** as a true concurrency conflict (return `Aborted` immediately).
* Ensure gRPC mapping distinguishes:
  * 401 → `Unauthenticated`
  * 403 → `PermissionDenied`
  * 409 → `Aborted`
  * 405/422 → `FailedPrecondition` (or `Aborted` only if actively retrying and the retry window expired)
  * 429 → `ResourceExhausted`
  * 5xx → `Unavailable`

### 3.2 Domain — branch “already exists” idempotency

* Treat “branch already exists” as success when:
  * status is **409**, or
  * status is **400** with an “already exists” style message/body.

### 3.3 Projection — missing path semantics for derived reads

Because Projection serves reads from derived state (not live GitLab), define and test one behavior:

* Missing path returns `NotFound` (and UI shows an explicit error).

### 3.4 Dependency hygiene

* Pin `gitlab.com/gitlab-org/api/client-go` as a **direct** dependency in both Domain and Projection modules to avoid workspace drift.

---

## 4) Local deterministic GitLab substrate (test standard)

### 4.1 Primitive unit tests (default): `client-go/testing` (gomock)

For **primitive-owned unit tests** (Domain/Projection handler logic), default to vendor-supported gomock mocks:

* `gitlab.com/gitlab-org/api/client-go/testing`

This validates our logic at the same call sites we use in production (`client-go` service methods), without needing HTTP stubs.

### 4.2 Capability/integration tests (default): `client-go/testing` mocks + outbox→projection apply

Capability contract-pass tests should run:

* Domain handlers with a `client-go/testing` TestClient injected (gomock expectations).
* Domain outbox dispatch in-process.
* Projection applies facts deterministically and serves read queries from derived state.

---

## 5) End-to-end local flow (Increment 0 contract-pass path)

Increment 0’s end-to-end path is “in-process services + deterministic mock substrate”.

### 5.1 Contract-pass (capability-level) test structure

Add (or carve out from the existing v6 scenarios pack) a dedicated Increment 0 test that asserts:

1. **Onboard mapping** (Domain command)
2. **Propagate fact** (Domain outbox → Projection apply)
3. **Read context resolution** (`published` → `resolved.commit_sha` returned by Projection reads)
4. **Browse tree** (Projection `ListTree`, which returns the resolved SHA)
5. **Open file** (Projection `GetContent`)
6. **Citations present** (repo_id + sha + path)

Recommended location (matches current repo taxonomy):

* `capabilities/transformation/features/enterprise_repository/integration/`

Recommended substrate:

* `client-go/testing` mocks injected into Domain handler
* Embedded Postgres harness via `capabilities/transformation/testkit` helpers

### 5.2 Primitive-level tests (Domain + Projection)

#### Domain primitive tests (local-only)

Focus on deterministic semantics (not GitLab correctness):

* `EnsureChange`:
  * idempotency behavior
  * MR find-or-create behavior
* `CreateCommit`:
  * action mapping (create vs update vs delete vs move)
  * branch-level optimistic concurrency behavior (`StartSHA`/expected last commit id)
* `PublishChange`:
  * SHA guard mismatch → deterministic error
  * transient 405/422 retry window → deterministic success/fail outcomes

#### Projection primitive tests (local-only)

* `ListTree`:
  * resolves `read_context.selector` to an immutable `resolved.commit_sha` (returned in the response `ReadContext`)
  * pagination loop exercised
  * missing path behavior matches the chosen rule (strict vs compat)
* `GetContent`:
  * resolves `read_context.selector` to an immutable `resolved.commit_sha` (returned in the response `ReadContext`)
  * raw bytes returned
  * citation includes `{repo_id, sha, path}`

---

## 6) UISurface wiring (proto-governed gRPC per 496)

Increment 0 UI is **not optional**, but it can be minimal.

### 6.1 Replace “UI → GitLab” with “UI → Projection/Domain”

Platform app should not call GitLab directly in `services/www_ameide_platform/app/api/gitlab/**`.

For Increment 0:

* Replace “gitlab repositories/tree/files” routes with platform routes that call:
  * Projection Enterprise Repository query service for reads:
    * `ListTree`
    * `GetContent`

### 6.2 RequestContext discipline (496)

All UI-initiated calls must include:

* `RequestContext` (tenant/org/repo identity, correlation id, request id)
* `occurred_at`
* `producer`

Increment 0 should define a single UI helper to construct these consistently (and reuse it for later increments).

### 6.3 Local UI test (Phase 0/1/2 friendly)

Increment 0 should add at least one UI integration test that:

* runs locally without GitLab,
* calls a platform server route,
* asserts the route uses ConnectRPC clients to Projection (not GitLab),
* asserts returned payload includes `Canonical main@<sha>` (i.e., `read_context=published`) and citations.

Substrate should be the same deterministic mock GitLab + in-process primitive services used by the Go capability tests.

---

## 7) “How to run locally” (developer/agent inner loop)

Increment 0 must support local-only verification via `ameide test` (Phase 0/1/2). Concretely this means:

* Go integration tests (tagged) cover the deterministic contract-pass path using the mock GitLab server.
* UI integration tests run without any external dependencies (no real GitLab, no Kubernetes).

This is the “no-brainer” inner loop expected for both humans and agents in workspaces.

---

## 8) Implementation status (what is already done)

As of **2026-01-22**, Increment 0 has the following concrete implementation coverage:

* ✅ Capability scenario `0/OnboardExploreBaseline` exists in the v6 scenario pack:
  * `capabilities/transformation/features/enterprise_repository/integration/v6_togaf_scenarios_test.go`
* ✅ Enterprise Repository Domain command seam supports Git remote mapping + MR-backed change/commit/publish:
  * `primitives/domain/transformation/internal/handlers/enterprise_repository_v1.go`
* ✅ Enterprise Repository Projection query seam supports `published` browsing (`ListTree`, `GetContent`):
  * `primitives/projection/transformation/internal/handlers/enterprise_repository.go`
* ✅ Platform UI is wired to proto seams for “Published @ SHA + tree + open file” (no direct GitLab routes):
  * `services/www_ameide_platform/app/(app)/org/[orgId]/repo/[repositoryId]/page.tsx`
* ✅ Primitive unit tests use `gitlab.com/gitlab-org/api/client-go/testing` (gomock):
  * Domain: `primitives/domain/transformation/internal/tests/enterprise_repository_v1_test.go`
  * Projection: `primitives/projection/transformation/internal/handlers/enterprise_repository_v1_test.go`
