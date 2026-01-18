# 658 — EDA v2 Reference Implementation Refactor (Knative Broker/Trigger + CloudEvents)

> **Superseded:** replaced by `backlog/658-eda-v6-reference-implementation-refactor.md` (hybrid posture + `io.ameide.*`; Kafka remains the standard event plane).

**Status:** Draft (decision + plan)  
**Priority:** High  
**Standard:** `backlog/496-eda-principles-v2.md` (active; replaces v1)  
**Related:** `backlog/509-proto-naming-conventions.md`, `backlog/527-transformation-implementation-migration.md`, `backlog/657-transformation-domain-clean-target.md`

---

## 0.1 Repo status notes (2026-01-14)

Partial alignment work has already landed in the Ameide repo even though this backlog is marked “superseded”:

- `io.ameide.transformation.work.v1.WorkExecutionRequested` does **not** embed an envelope (`meta` is reserved), consistent with the 496 CloudEvents envelope rule.
- The transformation work executor consumes CloudEvents headers (`ce-id`, `ce-correlationid`, `ce-causationid`, `ce-traceparent`, `ce-tracestate`) instead of expecting in-proto metadata.

This is compatible with both “EDA v2 (Knative)” and “EDA v3 (Kafka)” at the message contract level; the remaining work is the delivery plane standardization.

## 0) Question

Should we refactor the existing “reference implementation” (Transformation + WorkRequests/execution substrate + verify tooling) to comply with **EDA v2**?

---

## 1) Answer (recommended)

Yes — if `backlog/496-eda-principles-v2.md` is truly the **active standard**, the reference implementation must be refactored.

If we do not refactor, we should instead downgrade 496 v2 to “target/aspirational” and explicitly keep the current Kafka/topic/KEDA posture as the standard. Right now the docs set is internally inconsistent.

---

## 2) Why this matters

EDA standards only “exist” if:

- the reference implementation demonstrates them end-to-end, and
- the CLI/verify gates can enforce them.

Today, multiple transformation backlogs still describe:

- **topic-family contracts** as primary routing,
- **execution queue topics + KEDA** as the standard way to deliver “intents”,
- **CloudEvents as optional binding** and sometimes envelope-in-proto semantics.

Those are incompatible with 496 v2, which makes:

- **CloudEvents mandatory** for inter-primitive messaging,
- **Knative Broker + Trigger mandatory** as the routing spine,
- **direct Kafka topic coupling forbidden** as an inter-service contract.

Clarification: this backlog is about the **inter-primitive execution substrate** (Broker/Trigger delivery + executor). It does **not** replace the developer/agent code-changing substrate in `backlog/650-agentic-coding-overview.md` (Coder workspaces + ephemeral “task workspaces” for PR automation).

---

## 3) What “refactor the reference implementation” means (scope)

This backlog scopes the refactor to what must be true for the platform to say “we comply with EDA v2”:

1. **Transformation Domain facts:** emitted via transactional outbox → dispatcher → **Broker publish** (CloudEvents HTTP binary protobuf).
2. **Transformation Projection ingestion:** receives facts via **Trigger delivery** (subscriber), dedupes on CloudEvents `id` (inbox), and ACKs 2xx on duplicates.
3. **Work execution intents (tool runs / agent work):** delivered via Broker/Trigger like any other inter-primitive intent; no direct Kafka “execution topic contract”.
4. **DLQ semantics:** non-2xx → retry/backoff → DLQ; replay procedure exists (new `id`, link via `causationid`).
5. **`ameide primitive verify`:** validates 496 v2 conformance (CloudEvents headers, dataschema conventions, outbox/inbox presence, Trigger filters by `type`, DLQ delivery spec).

---

## 4) Key design decision (single standard): Executor Service subscriber

We need one enforceable “execution substrate” for long-running tool runs / agent work that is:

- compliant with `backlog/496-eda-principles-v2.md`
- vendor-aligned with Knative (Triggers deliver to a subscriber Destination)
- compatible with protobuf bytes in CloudEvents HTTP **binary** mode

**Decision:** adopt **Option A only** as the platform standard.

### 4.1 Standard: Executor primitive is the Trigger subscriber

- A dedicated executor primitive is the Trigger subscriber for `ameide.<bc>.intent.work-execution.requested.v1` (and future work-intent types).
- It implements inbox dedupe keyed on CloudEvents `id` and returns **2xx quickly** after durable acceptance.
- It performs the long work asynchronously as an **internal implementation detail** (Kubernetes Jobs, Kueue, internal workers, etc.).
- It reports outcomes back to the owning Domain write surface; the Domain emits facts after commit (outbox → broker).

This keeps the inter-primitive contract pure: **CloudEvents over HTTP via Broker/Trigger**.

### 4.2 Non-standard/experimental: JobSink

Knative JobSink is currently `v1alpha1` and mounts an event as a JSON file, which complicates the “protobuf bytes in HTTP binary mode” standard.

Treat JobSink as a potential future optimization, not part of the enforceable platform standard.

---

## 5) Delivery increments (each increment advances all loops)

Each increment must implement a runnable end-to-end system across:

- publish loop (outbox → broker)
- delivery loop (broker → trigger → subscriber)
- consumer loop (inbox dedupe + idempotent effects)
- failure loop (retry/backoff + DLQ)
- verification loop (`ameide primitive verify` + `ameide test`)

### Increment 1 — Minimal compliant fact delivery (Domain → Projection)

- Domain emits one fact type via outbox → dispatcher → Broker publish (CloudEvents).
- Projection receives via Trigger, applies inbox dedupe on `ce-id`, materializes one read model.
- Broker/Trigger delivery spec is configured with retries + DLQ sink.
- `ameide primitive verify` checks: CloudEvents binary mode, required extensions, outbox/inbox presence, Trigger filter by `type`.

**Exit criteria**
- Kill/retry produces duplicates without double-applying effects; DLQ receives poison events; verify passes.

### Increment 2 — Minimal compliant intent delivery (Work execution)

- Define one WorkExecutionRequested intent type (CloudEvents `type` naming per 496 v2).
- Deliver via Trigger to the **Executor primitive** (the standard subscriber).
- Executor:
  - dedupes by CloudEvents `id`
  - ACKs 2xx after durable acceptance (no long-running HTTP requests)
  - runs the work asynchronously (Jobs/Kueue/internal pool) and records outcomes back into the Domain
- Domain emits outcome facts after commit (outbox → broker).

**Exit criteria**
- A work item can be requested, executed, and observed as facts in the projection; failures go to DLQ; replay works.

### Increment 3 — Tenant isolation + observability as enforcement

- `tenantid` becomes mandatory everywhere; consumers reject missing tenant.
- Metrics/logging required fields are present and verified (ce_id/type/source/tenantid/outcome).
- CLI verify checks tenant headers and basic isolation enforcement.

**Exit criteria**
- Cross-tenant leakage tests pass; ops can see per-tenant DLQ and delivery failures.

---

## 6) Backlog impacts (what must be updated/deprecated)

If we adopt this refactor, the following backlog sets must be updated to remove the Kafka/topic/KEDA posture as the “contract surface”:

- Transformation contract docs that treat topic families as primary routing (legacy posture).
- WorkRequests execution docs that encode “execution queue topics” as contracts.
- Any docs that describe CloudEvents as “optional binding” for inter-primitive messaging (this is incompatible with 496 v2).

This refactor is the “bridge” that makes:

- `backlog/496-eda-principles-v2.md` enforceable,
- `backlog/657-transformation-domain-clean-target.md` truthful,
- and `ameide primitive verify` meaningful.
