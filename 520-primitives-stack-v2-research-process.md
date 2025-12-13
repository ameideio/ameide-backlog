Note: this document is supporting background; the consolidated, normative spec is `backlog/520-primitives-stack-v2.md`. Examples use placeholder namespaces (e.g., `acme.*`)—the canonical repo proto namespace is `ameide_core_proto.*` per `backlog/509-proto-naming-conventions.md`.

## Key constraints and primitives from the primary docs (what drives the design)

### Temporal Go SDK workflow code constraints (determinism + idempotency)

Temporal workflow code must be **deterministic** and **idempotent**. In Go SDK terms, that means (among other things):

* Use Temporal’s workflow APIs for time (`workflow.Now`, `workflow.Sleep`) instead of `time.Now` / `time.Sleep`.
* Use Temporal’s concurrency primitives (`workflow.Go`, `workflow.Channel`, `workflow.Selector`) instead of Go’s native goroutines/chans/select.
* Avoid randomized map iteration order (don’t `range` over maps in workflow logic unless you control ordering).
* Don’t directly affect external systems from workflow code except via Activities.

Also, the Go SDK explicitly warns that the **client** API is not for workflow code (“DO NOT USE THIS API INSIDE OF ANY WORKFLOW CODE!!!”).

### Worker Versioning (time-sensitive deprecation risk)

Temporal’s docs warn that support for the **experimental Worker Versioning method used before 2025** will be removed from Temporal Server in **March 2026**. Treat any generator/operator strategy that depends on the legacy/experimental method as a migration risk with a fixed deadline.

**Generator/operator implication (default):**

* Do not assume legacy/experimental worker versioning behavior.
* Prefer replay-safe workflow code evolution using Temporal’s workflow versioning mechanisms (e.g., `workflow.GetVersion` patching) and any current, stable worker deployment/versioning approach supported by the target Temporal Server version.

### Signals + Signal-With-Start as an atomic trigger primitive

Temporal’s **Signal-With-Start** is the client-side pattern that “signals if running, otherwise starts then signals” (atomically).
Important nuance from the Go client docs: `SignalWithStartWorkflow` defaults `WorkflowIDReusePolicy` to **AllowDuplicate**. If you care about “no accidental new run for the same business key after completion,” you must set the reuse policy deliberately.

### Workflow IDs, namespaces, and uniqueness

* A **Workflow Id** is unique **to an open execution within a Namespace**; reuse & conflict behavior is controlled by Workflow ID reuse/conflict policies.
* Namespaces are a major isolation/config boundary (retention, archival, etc.) and must be registered. Self-hosted: create/manage via CLI; Temporal Cloud: create/manage via Cloud UI or `tcld` (CLI + SDK namespace APIs don’t work with Cloud).

### Task Queues are “config-by-name”

Task Queues are lightweight routing constructs that workers poll and clients target; you generally don’t “provision” them like a database—they come into existence when used.

**Gotcha to bake into generation:** task queue naming mismatches do not reliably error; they can silently create separate queues and stall progress. Generators MUST emit a shared constant (and/or shared config loader) used by both worker and ingress/client so they cannot drift by accident.

### Worker registration + testing patterns

* Workers explicitly register workflow/activity functions.
* The Go SDK includes a testing story (testsuite) and workflow package docs show common testing/mocking patterns.

### Buf custom options + generate templates

* Custom options are implemented as **extensions on `google.protobuf.*Options` messages** (defined in `descriptor.proto`).
* `buf generate` runs local/remote protoc plugins from `buf.gen.yaml` and supports passing plugin options (`opt`, `types`, `exclude_types`, etc.).
* **Hard rule:** mark Ameide-specific options as **SOURCE-retained** by default; generators must read such options from `CodeGeneratorRequest.source_file_descriptors` (not only `proto_file`).
* **Hard rule:** build custom generators on a well-used protoc plugin harness (e.g., Buf’s `bufbuild/protoplugin`) and add golden tests that assert byte-for-byte deterministic output from fixed descriptor inputs.

### Kubernetes operator conditions conventions

* Standardized `metav1.Condition` / “conditions” are the idiomatic `.status` mechanism (including `type`, `status`, `reason`, `message`, `observedGeneration`, `lastTransitionTime`).
* `controller-runtime` reconciliation behavior is controlled by `(ctrl.Result, error)`; `RequeueAfter` / `Requeue` govern retries/backoff.
* Best practice: keep condition types positive-polarity (happy-path achieved state) and rely on `reason`/`message` for error detail; always set `observedGeneration` when updating conditions to avoid stale status ambiguity.

---

## Best approach for the three-plane “Process” primitive

### Design goals interpreted from your Success Criteria

* **Determinism is enforced by structure + guardrails, not just “tribal knowledge.”**
* **Versioning is first-class in the proto.**
* **Operator treats Temporal dependencies as external dependencies**: reconciles them, reports conditions, never hides errors behind pod crashes.
* **Human-owned code surface is tiny:** `*_impl.go` + `*_test.go`. Everything else is regenerated.

---

# 1) Proto plane: a Process definition pattern

## Recommendation: “Envelope + typed payload” for facts/commands

Use a single *Process Event envelope* carrying:

* required metadata (idempotency, time, producer, trace)
* a stable *business key* used for workflow ID derivation
* `oneof payload` containing strongly typed fact/command messages

Why:

* Strong typing enables generated routing and typed signal/update handlers.
* Envelope standardizes idempotency + workflow-key derivation (so ingress code is mostly generic).
* Signals are recorded in history, so you want schema evolution that’s explicit and consistent.

This aligns well with Temporal’s “workflow code must be deterministic/idempotent” and “don’t touch external systems except via Activities.”

**Interoperability requirement (cross-boundary):** emitted facts/events MUST be CloudEvents-compatible where practical (id/source/type/subject/time/schema), and the runtime MUST propagate W3C Trace Context (e.g., `traceparent`) across ingress → Temporal client calls → emitted facts/events so tracing and tooling don’t depend on bespoke metadata schemes.

## Recommendation: Make “trigger = Signal-With-Start” the default

* Every ingress message results in either:

  * `SignalWithStartWorkflow` for the first message for a workflow key, or
  * `SignalWorkflow` for subsequent messages.
* Use `WorkflowIDReusePolicy` explicitly (don’t rely on the default AllowDuplicate for SignalWithStart).

## Recommendation: Two orthogonal identifiers in the envelope

* **workflow key**: stable business identity (tenant + aggregate id, etc.) → **workflow ID derivation**
* **idempotency key**: per-message unique id → **event de-duplication**

Workflow ID uniqueness is scoped to Namespace, so the template MUST incorporate process name + business key to avoid collisions.

## Recommendation: Explicit versioning for workflow *start args* and *signal payloads*

Because Temporal records both workflow start arguments and signal payloads in history, **they must remain decodable on replay** for the entire time that workflow histories remain accessible for replay/reset/query operations. Temporal’s “workflow code must be deterministic” + versioning docs exist specifically because code/data changes can break replay.

So:

* Use Protobuf additive evolution for minor changes.
* For breaking changes, introduce `oneof`-versioned payloads and keep old variants until you’re confident no executions can replay them (often “> retention period” is a practical bound).

---

## Exact proto annotations/options to recommend

### A) Define custom options (in a shared `process/v1/process_options.proto`)

Below is the *pattern* (not the only numbering); the important part is **FileOptions + MessageOptions + FieldOptions** extensions so the plugin can discover process metadata from descriptors. This is how custom options are intended to work.

```proto
// process/v1/process_options.proto
// NOTE: keep this in proto2 syntax to define extensions cleanly.
syntax = "proto2";

package process.v1;

import "google/protobuf/descriptor.proto";

message ProcessOptions {
  optional string name = 1;              // stable process name: "order-fulfillment"
  optional string workflow_type = 2;      // Temporal workflow type name
  optional string default_task_queue = 3; // default task queue name
  optional WorkflowId workflow_id = 4;    // workflow ID derivation rules

  // Schema evolution contract:
  optional uint32 api_major = 10;         // bump on breaking changes
  optional uint32 api_minor = 11;         // bump on additive changes

  // Ingress bindings (logical subscriptions; transport binding is supplied via CRD/runtime config)
  repeated IngressSubscription ingress = 20;
}

message WorkflowId {
  enum Mode {
    MODE_UNSPECIFIED = 0;
    MODE_BUSINESS_KEY = 1;      // derive from envelope.meta.key
    MODE_IDEMPOTENCY_KEY = 2;   // derive from envelope.meta.idempotency_key (transactional workflows)
    MODE_TEMPLATE = 3;          // template with field paths
  }
  optional Mode mode = 1 [default = MODE_BUSINESS_KEY];
  optional string template = 2;  // e.g. "proc/{process}/{tenant}/{key}"
  optional string prefix = 3;    // e.g. "proc"
}

message IngressSubscription {
  optional string name = 1;      // logical name
  optional string kind = 2;      // "kafka", "nats", ...

  // Do not embed per-environment transport bindings (topic/subject URLs, auth, etc.) in proto.
  // Bind those via the Process CRD/runtime config and reference them here by logical name.
  optional string binding_ref = 3; // matches a runtime-config entry (e.g., spec.ingress.subscriptions[].name)
}

message FactOptions {
  // Marker only; stable identifiers and routing hints come from `(ameide_core_proto.common.v1.event_fact)`.
}

message CommandOptions {
  enum Delivery {
    DELIVERY_UNSPECIFIED = 0;
    SIGNAL = 1;   // fire-and-forget
    UPDATE = 2;   // request/response (Workflow Updates are not part of v2)
  }
  optional Delivery delivery = 1 [default = SIGNAL];
  // Stable identifiers and routing hints come from `(ameide_core_proto.common.v1.event_fact)`.
}

// Note: reuse the shared cross-primitive event/fact metadata spine for identifiers and routing hints,
// so Domain/Process/Projection/Integration don't drift into incompatible "event type" conventions.

message FieldMarkers {
  optional bool workflow_key = 1;
  optional bool idempotency_key = 2;
  optional bool schema_version = 3;
}

extend google.protobuf.FileOptions {
  optional ProcessOptions process = 50000;
}

extend google.protobuf.MessageOptions {
  optional FactOptions fact = 50001;
  optional CommandOptions command = 50002;
}

extend google.protobuf.FieldOptions {
  optional FieldMarkers markers = 50010;
}
```

### B) Example “Process definition proto” using those options

This is a concrete example for one process (`order-fulfillment`).

```proto
// example/processes/order/v1/order_fulfillment_process.proto
syntax = "proto3";

package example.processes.order.v1;

import "google/protobuf/timestamp.proto";
import "process/v1/process_options.proto";
import "ameide_core_proto/common/v1/eventing.proto";

option go_package = "github.com/acme/platform/example/processes/order/v1;orderv1";

option (process.v1.process) = {
  name: "order-fulfillment"
  workflow_type: "example.processes.order.v1.OrderFulfillmentWorkflow"
  default_task_queue: "order-fulfillment"
  workflow_id: {
    mode: MODE_TEMPLATE
    prefix: "proc"
    template: "proc/{name}/{meta.key.tenant_id}/{meta.key.order_id}"
  }
  api_major: 1
  api_minor: 0
  ingress: {
    name: "orders-facts"
    kind: "kafka"
    binding_ref: "orders-facts"
  }
};

// --- Common metadata for ingress + workflow routing ---
message OrderKey {
  string tenant_id = 1 [(process.v1.markers) = { workflow_key: true }];
  string order_id  = 2 [(process.v1.markers) = { workflow_key: true }];
}

message EventMeta {
  // Required: de-dupe key for at-least-once ingress.
  string idempotency_key = 1 [(process.v1.markers) = { idempotency_key: true }];

  google.protobuf.Timestamp observed_at = 2;
  string source = 3;

  // Required: stable key used to derive workflow ID.
  OrderKey key = 4;

  // Required: schema version for envelope/payload evolution.
  uint32 schema_version = 5 [(process.v1.markers) = { schema_version: true }];
}

// --- The envelope that ingress routes into the workflow ---
message OrderFulfillmentEvent {
  EventMeta meta = 1;

  oneof payload {
    OrderPlaced        order_placed        = 10;
    PaymentAuthorized  payment_authorized  = 11;
    ShipmentCreated    shipment_created    = 12;

    CancelOrder        cancel_order        = 50;
  }
}

// --- Facts (typed) ---
message OrderPlaced {
  option (process.v1.fact) = {};
  option (ameide_core_proto.common.v1.event_fact) = { type: "orders.OrderPlaced.v1" };
  string order_id = 1;
  google.protobuf.Timestamp placed_at = 2;
}

message PaymentAuthorized {
  option (process.v1.fact) = {};
  option (ameide_core_proto.common.v1.event_fact) = { type: "payments.PaymentAuthorized.v1" };
  string order_id = 1;
  string payment_id = 2;
  google.protobuf.Timestamp authorized_at = 3;
}

message ShipmentCreated {
  option (process.v1.fact) = {};
  option (ameide_core_proto.common.v1.event_fact) = { type: "shipping.ShipmentCreated.v1" };
  string order_id = 1;
  string shipment_id = 2;
  google.protobuf.Timestamp created_at = 3;
}

// --- Commands (typed) ---
message CancelOrder {
  option (process.v1.command) = { delivery: SIGNAL };
  option (ameide_core_proto.common.v1.event_fact) = { type: "orders.CancelOrder.v1" };
  string reason = 1;
}

// --- Workflow start args (versioned explicitly) ---
message OrderFulfillmentStartV1 {
  uint32 start_version = 1; // always 1 for V1
  OrderKey key = 2;
  google.protobuf.Timestamp started_at = 3;
}

// Optional: workflow result / completion record
message OrderFulfillmentResultV1 {
  string order_id = 1;
  string final_status = 2;
}
```

**Why these options/markers matter:**

* The plugin can derive workflow ID deterministically from `meta.key.*`.
* The plugin can enforce “every ingress message has idempotency_key + key”.
* The plugin can generate routing from `oneof payload`.
* The “start args version” is explicit and decoupled from code changes, which is critical because workflow history must replay deterministically across deployments.

---

## Modeling decisions (explicit answers to your bullets)

### Input facts/events: envelopes vs typed messages

**Recommendation:** `Envelope + oneof typed payload` (as above).
Avoid `google.protobuf.Any` for the main payload because you lose type-driven codegen and routing; if you need an “escape hatch,” add `google.protobuf.Any unknown = N;` *in addition* to the typed oneof.

### Workflow triggers: SignalWithStart patterns / command messages

**Recommendation:**

* All facts and most commands → signals (fire-and-forget).
* If/when you adopt Workflow Updates for commands that need synchronous responses, keep the same proto shape but allow `(process.v1.command).delivery = UPDATE` and generate both client stubs.

Signal-With-Start is the default trigger because it’s atomic and naturally supports “create-on-first-event.”

### Idempotency keys and workflow ID derivation

**Recommendation:**

* Envelope contains both:

  * `meta.key` (workflow routing key)
  * `meta.idempotency_key` (message de-dupe key)
* Workflow ID derivation is a **template** evaluated in ingress code; a good default is:

  * `proc/{process-name}/{tenant}/{business-id}`

Also: do **not** rely on the default `WorkflowIDReusePolicy` in `SignalWithStartWorkflow` (defaults to AllowDuplicate). Set it explicitly in generated ingress based on a proto option (see below).

### Versioning/compatibility rules for workflow inputs

**Recommendation:**

1. **Start args**: `StartV1`, `StartV2`, … (explicit version messages) or a wrapper `Start` message with `oneof { StartV1 v1 = 1; ... }`.
2. **Signal payloads**: evolve additively; for breaking changes introduce new message type and add it to the `oneof payload`.
3. **Workflow code changes that would break replay** must use Temporal’s workflow versioning mechanisms (`workflow.GetVersion` / patching / worker versioning approach), otherwise existing executions can fail on replay.
4. Keep old versions supported until you’re confident all executions that can replay those payloads are gone (namespace retention is a practical bound, but consider operational realities like “query closed workflows” / “resets”).

**Add one more file option (required):**
Extend `ProcessOptions` with something like:

* `bool singleton_per_key = true;`
* `enum reuse_policy = REJECT_DUPLICATE | ALLOW_DUPLICATE_FAILED_ONLY | ALLOW_DUPLICATE;`

Then generated ingress always sets the reuse policy explicitly.

---

# 2) Buf plugin plane: generated outputs + ownership split

## Concrete generated file layout (one process)

For the `order-fulfillment` example:

```
packages/ameide_core_proto/
  src/ameide_core_proto/…              # proto sources

packages/ameide_sdk_go/
  gen/go/…                             # protoc-gen-go outputs (SDK)

build/generated/
  process/
    order-fulfillment/                 # GENERATED-ONLY (safe to delete)
      cmd/
        worker/
          main.generated.go            # generated main wiring (imports human impl package)
          config.generated.go          # env/flags -> config struct
        ingress/
          main.generated.go
          config.generated.go
      internal/
        workflows/
          order_fulfillment_workflow.generated.go        # typed entrypoint + registration + constants
          order_fulfillment_workflow_signals.generated.go
        activities/
          order_fulfillment_activities.generated.go      # typed activity stubs + registration
        ingress/
          router.generated.go           # envelope -> handler -> Temporal call skeleton
          subscriptions.generated.go    # kafka/nats client hooks; pluggable adapters
      tests/
        determinism_policy.generated_test.go
        idempotency_invariants.generated_test.go
        routing_contract.generated_test.go

primitives/process/
  order-fulfillment/                   # HUMAN-OWNED (never cleaned)
    internal/
      workflows/
        order_fulfillment_workflow_impl.go               # HUMAN: workflow logic
      activities/
        order_fulfillment_activities_impl.go             # HUMAN: activity implementations
      ingress/
        router_impl.go                                   # HUMAN: router overrides (not supported in v2)
    tests/
      behavior_test.go                                   # HUMAN: business expectations
```

### Generated vs human-owned: strict contract

**Generated (overwritten every time):**

* `*.generated.go`
* `tests/*generated_test.go`
* all routing/registration/config skeletons

**Human-owned (never overwritten):**

* `primitives/process/<process-name>/**` (including `*_impl.go` for workflow + activities)
* additional non-generated tests (e.g., `*_test.go`) you add alongside

### Regeneration guarantees

* Generated files are **pure functions of** `(proto descriptors + plugin version + template defaults)`.
* Plugin writes a header comment like:

  * `// Code generated by protoc-gen-process. DO NOT EDIT.`
* Anything not matching that pattern is treated as human-owned and never touched.

This maps directly to your “developers only implement *_impl.go and tests” requirement.

---

## What each generated artifact does (and why)

### `cmd/worker/main.generated.go`

Generates:

* Temporal client dial + config (namespace, task queue, TLS, identity)
* Worker creation and registration:

  * `w := worker.New(c, taskQueue, worker.Options{...})`
  * `w.RegisterWorkflow(...)`
  * `w.RegisterActivity(...)`
* Worker start/shutdown:

  * `w.Run(worker.InterruptCh())`

Determinism enforcement:

* Generated workflow registration only references `internal/workflows/...generated.go`, which itself delegates to `_impl.go`.
* `_impl.go` is placed in a package where imports are lint-guarded (see CI section).

### `internal/workflows/order_fulfillment_workflow.generated.go`

Generates:

* workflow type name constant (from file option)
* typed workflow function signature
* signal handler wiring (from `OrderFulfillmentEvent.oneof payload`)
* a tiny delegation point:

  * `type Impl interface { Handle(ctx workflow.Context, start *StartV1) error }`
  * calls `NewImpl()` or `DefaultImpl` from `_impl.go`

Determinism expectations are documented in code comments that link to the Go workflow restrictions (time, goroutines, etc.).

### `cmd/ingress/main.generated.go` + `internal/ingress/router.generated.go`

Generates:

* subscription adapter interfaces (Kafka/NATS/etc.) and hookpoints
* decoding from transport envelope → `OrderFulfillmentEvent`
* workflow ID derivation from proto option template
* Temporal client calls:

  * `SignalWithStartWorkflow(ctx, workflowID, signalName, event, startOpts, workflowFn, startArgs...)`
  * or `SignalWorkflow` when already known-running
* idempotency policy:

  * always include `meta.idempotency_key` in the event delivered to workflow
  * optionally run ingress-side dedupe if transport can redeliver rapidly (still keep workflow-side dedupe)

**Hard generation rules (Temporal gotchas):**

* Export a single task-queue constant (e.g., `const DefaultTaskQueue = "..."`) and ensure both worker and ingress/client codepaths use it (overridable via the same env var / flag).
* Always set `WorkflowIDReusePolicy` explicitly for Signal-With-Start based on the process’s intended semantics (default is conservative: do not silently allow duplicates if the workflow ID is intended to be idempotent-by-key).

### Generated tests

You asked for tests that enforce determinism + idempotency invariants. Here’s what I recommend generating:

1. **Determinism policy test (static import/call guard)**

   * Parse `*_impl.go` under `primitives/process/<process-name>/internal/workflows/`
   * Fail if it imports:

     * `go.temporal.io/sdk/client` (explicitly forbidden for workflow code)
     * `time` (allowed only if used via `workflow.Now/Sleep` patterns)
     * `math/rand` / `crypto/rand` unless used only inside `workflow.SideEffect` (you can allowlist SideEffect usage)
   * Fail if it references:

     * `time.Now`, `time.Sleep`, `go` statement, `chan`, `select` (AST patterns)
       This is the highest-signal enforcement for “no banned calls.”

2. **Idempotency invariant test (behavioral)**

   * Using Temporal Go testsuite:

     * start workflow
     * send the **same** `OrderFulfillmentEvent` twice (same `idempotency_key`)
     * assert workflow state/result does not double-apply
       The harness is generated; the assertions are “TODO until you implement,” i.e., RED tests that become green when `_impl.go` implements dedupe.

3. **Workflow ID derivation contract test**

   * For several sample keys, assert `WorkflowID(key)` matches the template and is stable
   * Guards accidental changes to ID format (which is a compatibility break)

4. **Routing completeness test**

   * Ensure every `oneof payload` case has a generated handler stub
   * Fail if new facts/commands are added without regeneration

---

# 3) Operator plane: CRD + reconciliation + conditions + observability

## CRD spec (minimum viable, but future-proof)

**Kind:** `Process` (or `TemporalProcess`)
**Group/Version:** e.g. `process.platform.example.com/v1alpha1`

### `spec` (proposed)

```yaml
spec:
  image:
    worker: ghcr.io/acme/order-fulfillment-worker:1.2.3
    ingress: ghcr.io/acme/order-fulfillment-ingress:1.2.3

  temporal:
    address: temporal-frontend.temporal.svc:7233
    namespace: orders-prod
    taskQueue: order-fulfillment
    tls:
      mode: mTLS
      secretRef: temporal-mtls
    identity: order-fulfillment-operator

    namespaceManagement:
      mode: External | EnsureSelfHosted
      retention: 30d

  worker:
    replicas: 3
    resources: { requests: ..., limits: ... }
    concurrency:
      # map 1:1 to worker.Options where appropriate
      maxConcurrentWorkflowTaskExecutionSize: 1000
      maxConcurrentActivityExecutionSize: 2000
    podTemplate:
      serviceAccountName: process-worker
      nodeSelector: ...
      tolerations: ...

  ingress:
    replicas: 2
    subscriptions:
      - name: orders-facts
        kind: kafka
        topic: facts.orders.v1
        consumerGroup: order-fulfillment
        authSecretRef: kafka-auth
    resources: ...
```

### `status` (proposed)

```yaml
status:
  observedGeneration: 12
  conditions:
    - type: SecretsReady
      status: "True|False|Unknown"
      reason: ...
      message: ...
      observedGeneration: 12
      lastTransitionTime: ...
    - type: DependenciesReady
      status: "True|False|Unknown"
      reason: ...
      message: ...
      observedGeneration: 12
      lastTransitionTime: ...
    - type: TemporalReady
      status: "True|False|Unknown"
      reason: ...
      message: ...
      observedGeneration: 12
      lastTransitionTime: ...
    - type: NamespaceReady
    - type: WorkerReady
    - type: IngressReady
    - type: WorkloadReady
    - type: Ready
```

Use **standard `metav1.Condition`** fields (`type/status/reason/message/observedGeneration/lastTransitionTime`) as per the standardized conditions work.

## Reconciliation logic (external dependency aware)

Treat Temporal and ingress transport like external dependencies:

1. **Validate spec**

   * missing fields → set `Ready=False`, `reason=InvalidSpec`; do *not* create pods that will just crash.

2. **Resolve secrets/config**

   * ensure referenced Secrets exist (Temporal TLS, Kafka auth, etc.)
   * if missing: set `SecretsReady=False reason=SecretMissing` and also set the specific dependency condition (`TemporalReady=False` / `IngressReady=False`) as needed.

3. **Temporal connectivity check**

   * attempt Temporal client dial using provided credentials.
   * if unavailable: `TemporalReady=False reason=DialFailed`
   * do not treat as a “hard reconcile error” because controller-runtime backoff hides progress; use conditions + `RequeueAfter` for transient dependency problems. The reconcile return behavior is defined by controller-runtime’s `(Result, error)` contract.

4. **Namespace management**

   * `EnsureSelfHosted` mode:

     * use Temporal Namespace APIs (Go SDK `NamespaceClient`) to describe/register/update retention, etc. (self-hosted only).
   * `External` mode:

     * only verify it exists; if not: `NamespaceReady=False reason=NotFound`

5. **Deploy worker + ingress Deployments**

   * apply Deployment/Service/ServiceMonitor/etc.
   * update conditions based on readiness:

     * `WorkerReady` from Deployment available replicas & worker `/readyz`
     * `IngressReady` similarly

## Failure modes to surface clearly

* Temporal unreachable → `TemporalReady=False`, keep pods running (so logs/metrics available), but ingress MUST backoff.
* Namespace missing/unauthorized → `NamespaceReady=False`, ingress and worker MUST fail fast but *health endpoint reports not-ready*.
* Worker image crash → `WorkerReady=False reason=CrashLoopBackOff`
* Ingress cannot connect to Kafka → `IngressReady=False reason=UpstreamUnavailable`
* Proto/process mismatch (e.g., deployed binary doesn’t match CR’s declared process name) → `Ready=False reason=ContractMismatch`

## Observability contracts expected from generated code

Generated worker/ingress binaries MUST expose:

* **HTTP health endpoints**

  * `/healthz` = process is up
  * `/readyz` = Temporal reachable + worker polling active / ingress subscription active
  * bind address/port comes from `PORT_HTTP` (standard operator↔runtime waist)

* **Metrics endpoint** (Prometheus scrape)

  * `/metrics` includes:

    * Temporal SDK worker metrics (poll success rate, task execution, etc.)
    * ingress lag (consumer offsets)
    * idempotency drops count
    * signal-with-start vs signal counts

Temporal has an Observability guide for the Go SDK (metrics/logging/tracing); generated code MUST wire this consistently.

---

# Recommended namespace + task-queue management strategy (with pros/cons)

## Task queues

**Recommendation:** Operator-managed *by convention/config*, not “provisioned.”

Reason: Task Queues are addressed by name; workers poll them and clients dispatch to them; you don’t typically pre-create them as standalone resources.

So: task queue belongs in:

* proto default (`default_task_queue`) for codegen
* CRD override (`spec.temporal.taskQueue`) for deployment-time routing

## Namespaces

Namespaces are different: they’re a registered/configured entity (retention, archival, etc.) and management differs between self-hosted and Cloud.

### Option A: **External namespace management (default)**

**How:** Operator requires `spec.temporal.namespace` to exist; it only validates connectivity and existence.

**Pros**

* Works for **Temporal Cloud** (where SDK/CLI namespace registration APIs are not applicable).
* Least privilege: operator doesn’t need admin rights to create/modify namespaces.
* Avoids accidental drift in retention/archival done by app teams.

**Cons**

* Requires separate provisioning workflow (Cloud UI / `tcld` / Terraform).
* Drift between desired retention and actual can go unnoticed unless you add drift checks.

### Option B: **Operator-managed namespaces (self-hosted only)**

**How:** Add `spec.temporal.namespaceManagement.mode = EnsureSelfHosted` and provide operator credentials authorized to manage namespaces.

**Pros**

* Fully declarative “namespace exists with retention X.”
* Eliminates manual steps for self-hosted clusters.

**Cons**

* Requires high-privilege credentials to Temporal frontend.
* Risky if teams can unintentionally change retention/archival on shared clusters.
* Not portable to Temporal Cloud “the same way,” because Cloud namespace creation is via UI/`tcld`.

### Recommendation: **Hybrid default**

* Default to **External** for portability and least-privilege.
* Support **EnsureSelfHosted** as an opt-in for clusters where Temporal is owned/operated by the same platform team running the operator.

---

# CI guardrail plan (regen diff + compile/test gates + import policy)

## 1) Regen diff check

* `buf generate`
* `git diff --exit-code`
* Ensure plugin version pinned (remote plugin revision pinned) so codegen is stable. `buf generate` supports remote plugins + revision pinning.
* Ensure `buf.gen.yaml` uses `clean: true` for generated-only output directories so deletes/renames are correct.

## 2) Compile/test gates

* `go test ./...`
* Run generated tests (determinism policy, routing contract, idempotency invariants)

## 3) Import/call policy for determinism

Enforce in CI (and generate the test that enforces it):

* Disallow `go.temporal.io/sdk/client` imports in `primitives/process/<process-name>/internal/workflows/*_impl.go` (client docs warn against usage in workflows).
* Disallow `time.Now`, `time.Sleep`, raw goroutines, channels, `select` in workflow impl; require workflow equivalents.
* Optionally disallow map iteration (`range` over maps) in workflow impl unless explicitly sorted (AST lint).

## 4) Versioning compliance gate

* Require every process proto to set `option (process.v1.process).api_major/api_minor`
* Require every workflow start message to include explicit version (`start_version`) or use `oneof` versioning
* Gate that new facts/commands have stable `external_type` (don’t rely on fully-qualified message name as a “public API” unless you decide it’s stable)

## 5) Dependency/import policy

* Keep generated Go packages for workflows/activities isolated so developers don’t accidentally import application code that does nondeterministic things.
* In repo policy:

  * `internal/workflows` can only import:

    * `go.temporal.io/sdk/workflow`
    * your generated pb types
    * your `internal/activities` interfaces/stubs
  * anything else requires an explicit allowlist change.

---

# How this meets your Success Criteria

### Workflows remain deterministic

* Structural separation: client code only in ingress/worker mains; workflow impl is forced through `workflow.*` APIs.
* CI guardrails: AST-based banned imports/calls, plus testsuite harness.

### Workflow versioning strategy is explicit

* Proto declares API major/minor and versioned Start messages.
* Breaking code changes use Temporal’s workflow versioning mechanisms.

### Operator reconciles Temporal dependencies and surfaces clear conditions

* Standard `metav1.Condition`-based condition model with ObservedGeneration.
* Reconcile return semantics use controller-runtime contract properly.

### Developers only implement `_impl.go` and tests; structure from `buf generate`

* Codegen owns wiring, routing, registration, and the policy tests.
* Human code is limited to workflow/activity business logic.

---

Add a **proto-level “workflow kind”** option to choose between:

* `ENTITY` (one long-lived workflow per business key; uses Continue-As-New when history grows)
* `TRANSACTION` (one workflow per idempotency key / request)

That single knob lets the plugin generate the correct default Workflow ID strategy + WorkflowIDReusePolicy without humans making subtle mistakes around `SignalWithStart` defaults.
