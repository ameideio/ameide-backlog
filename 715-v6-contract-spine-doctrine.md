---
title: "715 — v6 Contract Spine Doctrine (Platform-wide Protobuf + SDK consumption + Kafka/CloudEvents event plane)"
status: draft
owners:
  - platform
  - architecture
  - core-proto
  - devx
created: 2026-01-21
supersedes:
  - "proto-consumption drift (SDK rings vs direct buf imports)"
  - "event envelope ambiguity (meta-in-proto vs CloudEvents)"
depends_on:
  - 496-eda-principles-v6.md
  - 509-proto-naming-conventions-v6.md
  - 537-primitive-testing-discipline.md
  - 590-capabilities.md
  - 714-togaf-scenarios-v6.md
  - 710-gitlab-api-token-contract.md
---

# 715 — v6 Contract Spine Doctrine

## 1) Purpose

Define the **single, long-term, platform-wide** way we design, govern, distribute, and test contracts across all primitives and capabilities in v6.

This doctrine eliminates drift by making contracts:

* **platform-wide** (single source of truth),
* **mechanically enforced** (lint + breaking + codegen gates),
* **uniformly consumed** (SDK-only consumption),
* **operationally consistent** (Kafka event plane + CloudEvents envelope),
* **product-verifiable** (Scenario Slices call the same seams as UI/process/agents).

This is the one standard all refactoring must converge to, regardless of current implementation.

---

## 2) Non-negotiables

### 2.1 Platform-wide contract system

* There is exactly **one** authoritative contract source: **`ameide-core-proto`** (the platform proto repository/module set).
* All cross-service seams (RPC and events) are defined here.
* No other repository defines “real” platform contracts.

**Clarification (current repo reality)**
- In the current monorepo, the authoritative proto sources live under `packages/ameide_core_proto/` and are published as a Buf/BSR module (e.g., `buf.build/ameideio/ameide`).
- This doctrine is phrased as “ameide-core-proto” to describe the *platform-wide contract authority*, not to force an immediate repo split.

### 2.2 Owner-only writes are enforced at the contract boundary

* Only Domain services expose mutation commands for canonical state.
* All other primitives (UI, Agents, Process, Projection) interact via Domain commands or Projection queries.
* This must be reflected in the proto surface design and review policy.

### 2.3 Git-first invariants remain product truth (not just engineering)

Contract design must support the v6 posture:

* citations `{repository_id, commit_sha, path[, anchor]}` are first-class,
* read contexts resolve deterministically to commit SHAs,
* evidence spine is first-class,
* derived views are explainable via citations.

---

## 3) Scope: where the doctrine applies

### Applies to

* Any interaction across deployable services
* Any interaction across primitives (Domain/Projection/Process/Agent/UISurface)
* Any message on the event plane (facts/events)

### Does not apply to

* Primitive-internal control flow and in-process function calls

---

## 4) Contract types and transports

### 4.1 Commands and Queries (synchronous)

* **Transport:** gRPC (or protocol-compatible alternative, but still generated from proto)
* **Schema:** Protobuf request/response
* **Rule:** no “direct adapter calls” across primitives; only proto-defined seams.

#### Mandatory shapes

* All requests include explicit scope identity:
  * `{tenant_id, organization_id, repository_id}` (plus optional additive identifiers)
* All read responses return citation-grade info when returning content.

### 4.2 Facts / Events (asynchronous)

* **Transport:** Kafka is the standard in-cluster event transport.
* **Envelope:** CloudEvents v1.0 **is mandatory** for events.
* **Payload:** Protobuf bytes.

#### Encoding rule (normative)

* CloudEvents **binary mode** on Kafka:
  * CloudEvents attributes in Kafka headers (per Kafka binding conventions)
  * Protobuf bytes in Kafka record value
* No meta/envelope fields embedded inside protobuf payload as the event envelope for new work.

---

## 5) Semantic identity and naming

### 5.1 CloudEvents `type` is the canonical event semantic identifier

* **Normative:** `ce.type` MUST be `io.ameide.<capability>.<event_name>.v<major>`
* `ce.source` MUST identify the producing component/service consistently
* `ce.id` MUST be a stable unique id for idempotency and replay

### 5.2 Proto package naming

Proto package naming is a code-level concern and may migrate gradually; however, new work must converge on:

* `io.ameide.core.v1` for platform core primitives
* `io.ameide.<capability>.<surface>.v1` for capability surfaces

**Important:** “platform-wide contract” does not mean “flat namespace.” We maintain **one proto source** while partitioning by package for ownership and evolution.

---

## 6) Platform Core vs Capability Surfaces (how we avoid cross-capability drift)

### 6.1 Platform Core packages (defined once, imported everywhere)

`io.ameide.core.v1` contains only global primitives:

* Identity: `ScopeIdentity { tenant_id, organization_id, repository_id, ... }`
* Citations: `Citation { repository_id, commit_sha, path, anchor? }`
* Read context: `ReadContext { published | proposal | sha | tag... }` + `ResolvedReadContext { commit_sha }`
* Evidence spine models: `EvidenceSpine { change_id, mr_ref, published_sha, citations[], governance_refs[] }`
* Common errors/status: `ErrorInfo`, `ValidationViolation`, `NotFound`, `Conflict`, etc.
* Pagination/time: `PageToken`, `PageInfo`, `Timestamp`, etc.

**Rule:** No capability may redefine these concepts.

### 6.2 Capability Surface packages (business semantics + service APIs)

Each capability gets its own package space inside the same core proto repo, e.g.:

* `io.ameide.repository.enterprise_repository.v1`
* `io.ameide.transformation.v1`
* `io.ameide.governance.v1`
* `io.ameide.memory.v1`

**Rule:** capability surfaces MUST import platform core primitives for identity/citation/evidence.

### 6.3 Compile-time drift prevention (normative)

We enforce:

* “core-only definitions” rule (no duplicated core messages elsewhere)
* “core import required” rule (identity/citations/evidence must be from core)
* CODEOWNERS per capability subtree in the core proto repo

---

## 7) Distribution and consumption model (SDK doctrine)

### 7.1 Single rule: services consume SDKs only

All primitives and capability services MUST consume **generated SDK artifacts**, not raw proto compilation steps inside each service.

This is the normative consumption model.

**Default (v6)**
- Primitives and scenario runners MUST import **Ameide wrapper SDK packages only**.
- Direct consumption of Buf-generated artifacts is not permitted outside the SDK build pipeline.
- SDKs MUST re-export generated message/stub types verbatim and may add only thin transport/envelope helpers.

### 7.2 What an SDK is allowed to contain

SDKs must be **thin helpers**:

* **Must include:** generated message types + generated service stubs (verbatim)
* **May include:** transport helpers (client builders), interceptors, retry policies, auth helpers, tracing helpers, CloudEvents/Kafka header helpers
* **Must NOT include:**
  * handwritten re-modeling of request/response types
  * “barrel re-export reshaping” that changes the contract surface
  * alternate domain models that become de facto API
  * business semantics or validation that belongs in services

**Interpretation:** the SDK cannot become “another contract layer.”

### 7.3 SDK generation is deterministic from core proto

* SDK generation is the output of the core proto pipeline.
* No manual edits to generated code.
* SDK versioning follows contract versioning.

---

## 8) Buf governance (mechanical enforcement)

### 8.1 Buf gates are mandatory for core proto changes

Every merge to `ameide-core-proto` must pass:

* `buf lint`
* `buf breaking` (against the latest main/published baseline)
* `buf generate` (managed mode or controlled plugins)

### 8.2 Breaking-change policy

* Backward compatibility is the default expectation.
* Breaking changes require a major version bump:
  * new `v2` package/service/event type
  * explicit migration plan
  * dual-publish window if needed

---

## 9) Capability-level contract packs (inside core proto repo)

Even though proto is centralized, we still treat capabilities as the ownership and review boundary.

### 9.1 Required structure in the core proto repo

* `proto/io/ameide/core/v1/**`
* `proto/io/ameide/<capability>/**`
* `capabilities/<capability>/CONTRACTS.md` (index + semantics)
  * list of services (RPC)
  * list of event types (CloudEvents `type`)
  * invariants (identity/citations/evidence requirements)
  * versioning rules and owners

This yields platform-wide consistency without semantic drift.

---

## 10) Scenario Slices as the incremental implementation + acceptance plan

### 10.1 Scenario runner must use SDK clients

Scenario slice runners MUST call the same proto seams (via SDKs) that UI/process/agents use.

No internal adapters, no “test-only shortcuts.”

### 10.2 Evidence spine is mandatory output

Every slice run outputs an audit-grade run report containing:

* scope identity
* proposal reference (MR or change ref)
* published SHA (baseline)
* citations used/returned
* governance outcomes (if applicable)

### 10.3 Tier-1 slices required for v6 “platform exists” claim

At minimum:

* browse canonical repo (Git tree + citations)
* governed publish with evidence spine
* derived backlinks/impact (projection-only, citeable)
* proposal vs published views by SHA

These are the product story; the contracts must support them.

---

## 11) Event plane: topic strategy (normative but flexible)

Kafka topic naming should not become the semantic identity (that’s `ce.type`).
Topic strategy can evolve, but the envelope and type identity remain stable.

**Default (v6)**

* Topics are capability-owned (e.g., `io.ameide.<capability>.events`) and may carry multiple event types.
* Consumers route/filter by CloudEvents `ce.type`.

The pattern `topic == ce.type` is **not** the default and requires an explicit exception for operational reasons (e.g., very high throughput, retention isolation, ACL isolation).

This avoids “topic sprawl as API design” while keeping operational control.

---

## 12) Migration plan (how we refactor everything to this)

### 12.1 Declare legacy patterns

Mark as legacy/migration-only:

* “meta-in-proto event envelope” patterns
* “services importing proto directly” (including direct Buf-generated imports; bypassing wrapper SDKs)
* duplicate identity/citation message definitions

### 12.2 Introduce platform core primitives and migrate usage

* Add `io.ameide.core.v1` and migrate services to import it.
* Add CI rule preventing redefinition elsewhere.

### 12.3 Consolidate SDK generation and enforce SDK-only consumption

* Update service templates to use SDKs only.
* Remove proto compilation steps from services.
* Add CI check that fails builds if services import raw proto modules directly.

### 12.4 Event migration to CloudEvents-on-Kafka

* Introduce standard library for emitting/consuming CloudEvents on Kafka.
* Dual-publish during migration if needed:
  * old envelope topics + new CloudEvents topics until consumers move
* After migration window, remove legacy envelope usage.

---

## 13) Definition of Done for “we aligned to 715”

We are aligned when:

* All new seams are defined in `ameide-core-proto`
* All services consume SDKs only
* All events on Kafka use CloudEvents envelope + protobuf payload
* Platform core primitives are imported everywhere (no duplicates)
* Scenario slices run using SDK clients and produce evidence spine reports

---

## 14) Non-goals

* This does not prescribe internal architecture inside a service.
* This does not mandate a specific Kafka cluster/operator configuration.
* This does not mandate a specific UI framework.
* This does not require one module per capability; it requires one authoritative core proto repo with capability-partitioned packages and ownership.

---

### Summary (single sentence)

**`ameide-core-proto` is the platform-wide contract source; platform core primitives are defined once and imported everywhere; capability surfaces live under their own packages with clear owners; primitives consume generated SDKs only (thin helpers + generated stubs); synchronous seams are proto RPC; asynchronous seams are Kafka events with CloudEvents envelope and protobuf payload; scenario slices validate everything by calling SDK clients and producing audit-grade evidence spines.**
