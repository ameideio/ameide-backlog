Note: this document is supporting background; the consolidated, normative spec is `backlog/520-primitives-stack-v2.md`. Examples use placeholder namespaces (e.g., `acme.*`)—the canonical repo proto namespace is `ameide_core_proto.*` per `backlog/509-proto-naming-conventions.md`.

## Primary-source findings that shape the design

### Buf: deterministic `buf generate`, remote plugin pinning, and output hygiene

* `buf generate` is driven by a generation template (`buf.gen.yaml`) and supports `version: v2`.
* The template supports `clean: true`, which **deletes the directories / zip files / jar files** specified by each plugin’s `out` before generation (default is false). This is the most important lever for handling deletions/renames without stale generated files.
* Remote plugins are referenced with `remote: buf.build/<owner>/<plugin>[:<version>]`. Plugins MUST be pinned for reproducible generation, and are invoked in the order listed, with results combined before writing output.
* Remote plugins can also be pinned with a **`revision`** (“sequence number that Buf increments when rebuilding or repackaging the plugin”).
* Buf “managed mode” can set file options like `go_package_prefix` centrally (instead of sprinkling `go_package` across every proto).
* Buf dependency management uses `buf.yaml` and `buf.lock`; `buf.lock` is the dependency manifest and MUST NOT be hand-edited. `buf dep update` updates it.
* **Hard rule:** build custom generators on a well-used protoc plugin harness (e.g., Buf’s `bufbuild/protoplugin`) and add golden tests that assert byte-for-byte deterministic output from fixed descriptor inputs.

### Protobuf: custom options / annotations best practices

* Custom options are implemented by extending the appropriate descriptor options message (e.g., `google.protobuf.ServiceOptions`, `MessageOptions`, `MethodOptions`). In `.proto`, custom option names are used in parentheses (e.g., `(my_option)`), and they resolve to an extension field.
* Protobuf guidance recommends using **field numbers 50000–99999** for custom options.
* Each options type has its own “namespace” for extensions, so you can reuse the same numeric tag across different option types (e.g., `ServiceOptions` and `MessageOptions`) without conflict.
* **Hard rule:** mark Ameide-specific options as **SOURCE-retained** by default so they are available to codegen but do not leak into runtime descriptor pools; generators must read such options from `CodeGeneratorRequest.source_file_descriptors`.

### gRPC Go generation conventions

* `protoc-gen-go-grpc` generates Go service bindings; output file naming convention is `path/to/file_grpc.pb.go`.
* **Forward-compatibility rule:** service implementations MUST embed `Unimplemented<ServiceName>Server`. Opting out exists (`require_unimplemented_servers=false`) but is forbidden.
* **Hard rule for generated stubs:** embed `Unimplemented<ServiceName>Server` **by value**, not pointer; a nil embedded pointer can panic when an unimplemented RPC is invoked (official gRPC-Go guidance).
* **Drift detection rule:** do not rely on the embedded `Unimplemented<ServiceName>Server` to “cover missing methods.” Generated skeletons MUST emit explicit per-RPC stub methods (returning `Unimplemented`) and/or a handler interface that humans must implement, so missing wiring is caught as a compile/CI failure.
* Generator trap to avoid: do not parse `.proto` text for type resolution. Always resolve service/method/request/response symbols via descriptors across imports (e.g., return types like `google.protobuf.Empty`) to avoid “must be defined in the same .proto” failure modes seen in proto-skeleton generators.

**Recommended generated unit test (guards the nil-embed foot-gun):**

Generate a test per service that instantiates the server implementation as a zero value (no fields initialized) and calls an RPC method that the human has not implemented yet; assert it does not panic and returns an Unimplemented error. This fails fast if the embedded `Unimplemented<Service>Server` is accidentally switched to a pointer embed.

### Operator/controller + Gateway API (HTTPRoute)

* Kubebuilder describes controllers/operators as control loops; reconciliation MUST be **idempotent** and follow established Kubernetes conventions (including Status/Conditions).
* `metav1.Condition` is intended for direct use as `.status.conditions[]` and includes standard fields like `lastTransitionTime`, etc.
* Gateway API:

  * Backend references across namespaces require `ReferenceGrant`.
  * Implementations are expected to honor `ServicePort.appProtocol` when set.
  * Implementations MUST set `ResolvedRefs` to `False` with reason `UnsupportedProtocol` if they can’t use the specified/inferred protocol.
  * Implementations “usually” set `Accepted`, `Programmed`, and `ResolvedRefs` conditions for in-scope Routes.

### Transactional outbox: “DB transaction coupling” + reliable dispatch

* AWS Prescriptive Guidance frames transactional outbox as resolving the “dual write” problem (DB + message/event) and explains that the DB update and event record MUST be written atomically; it calls out duplicates/idempotency and ordering concerns.
* Microservices.io describes the core requirement: if the DB transaction commits, messages must be sent; if it rolls back, messages must not be sent—and traditional 2PC is typically not an option.
* Debezium’s outbox docs describe the pattern’s goal as avoiding inconsistencies between internal DB state and externally consumed events (and shows an ecosystem-standard “outbox table + relay” approach).

---

## Proposed “smartest” combined implementation (Proto → Buf generation → Operator)

### High-level principle

1. **Proto is the only place you declare shape + intent**: Domain services, which messages are “facts/events”, and routing hints for EDA (partition key, idempotency key, stable event type identifiers).
2. **Generators produce only generated code** (and generated tests/harness) into **generated-only directories**. Human code lives elsewhere. This makes `clean: true` safe and enables “regen in CI and fail on diff.”
3. **Operator is stable and language-agnostic**: it reconciles Deployments/Services/HTTPRoutes/Jobs/Secrets references; runtime behavior is controlled via a small set of **cross-language conventions** (env vars, ports, health endpoints).

---

# Deliverable 1: Recommended proto annotation schema

### 1) Define options once (shared module)

Create `proto/acme/platform/domain/v1/options.proto` (keep custom option extensions in a dedicated `proto2` file).

Key best-practice choices:

* Use extensions on `ServiceOptions`, `MethodOptions`, and `MessageOptions`.
* Use option field numbers in the **50000–99999** range.
* Reuse the **same numeric tag** across different option types to reduce bookkeeping.

```proto
// proto/acme/platform/domain/v1/options.proto
syntax = "proto2";

package acme.platform.domain.v1;

import "google/protobuf/descriptor.proto";

option go_package = "github.com/acme/platform/gen/go/acme/platform/domain/v1;domainv1";

// Marks a gRPC service as a “Domain service” and declares defaults for generators.
message DomainServiceOptions {
  optional string domain = 1; // e.g. "payments"

  message OutboxDefaults {
    optional bool enabled = 1;
    // Logical stream namespace (bound to environment-specific topic names outside proto).
    optional string stream_ref_prefix = 2;         // e.g. "payments"
    repeated string partition_key_fields = 3; // default partition key if event doesn't override
  }
  optional OutboxDefaults outbox = 2;

  message GatewayDefaults {
    // “Expose HTTP plane” (e.g., grpc-gateway / connect / bespoke HTTP mux) on PORT_HTTP.
    optional bool expose_http = 1;
    optional string path_prefix = 2;               // e.g. "/payments" (used by generated HTTP mux)
  }
  optional GatewayDefaults gateway = 3;
}

// Event/fact metadata MUST be standardized once across primitives.
// Prefer annotating event/fact payload messages using:
//
//   `option (ameide_core_proto.common.v1.event_fact) = { ... }`
//
// from `packages/ameide_core_proto/src/ameide_core_proto/common/v1/eventing.proto`,
// instead of re-defining per-primitive fact metadata options here.

// Optional: per-RPC hints (useful for generated RED tests + harness expectations).
message DomainMethodOptions {
  optional bool command = 1;              // vs query
  repeated string emits = 2;     // fully qualified message names
  optional string idempotency_key_field = 3; // request field to enforce idempotency (empty disables)
}

// Extend descriptor options. Custom option usage requires parentheses. 
extend google.protobuf.ServiceOptions {
  optional DomainServiceOptions domain = 50001;
}

extend google.protobuf.MethodOptions {
  optional DomainMethodOptions method = 50001;
}
```

### 2) Use it in a Domain service proto

Example `proto/acme/payments/v1/payments.proto`.

```proto
syntax = "proto3";

package acme.payments.v1;

import "acme/platform/domain/v1/options.proto";
import "ameide_core_proto/common/v1/eventing.proto";
import "google/api/annotations.proto"; // gRPC-HTTP transcoding is not part of v2.

service PaymentsService {
  option (acme.platform.domain.v1.domain) = {
    domain: "payments"
    outbox: {
      enabled: true
      stream_ref_prefix: "payments"
      partition_key_fields: ["payment_id"]
    }
    gateway: {
      expose_http: true
      path_prefix: "/payments"
    }
  };

  rpc Authorize(AuthorizeRequest) returns (AuthorizeResponse) {
    option (acme.platform.domain.v1.method) = {
      command: true
      emits: ["acme.payments.v1.PaymentAuthorized"]
      idempotency_key_field: "idempotency_key"
    };

    // Optional: HTTP mapping if you generate an HTTP plane.
    option (google.api.http) = {
      post: "/v1/payments:authorize"
      body: "*"
    };
  }
}

message AuthorizeRequest {
  string idempotency_key = 1;
  string payment_id = 2;
  int64 amount_cents = 3;
  string currency = 4;
}

message AuthorizeResponse {
  string authorization_id = 1;
}

// FACT / EVENT message.
message PaymentAuthorized {
  option (ameide_core_proto.common.v1.event_fact) = {
    type: "payments.payment_authorized.v1"
    stream_ref: "payments.facts"
    partition_key_fields: ["payment_id"]
    idempotency_key_fields: ["payment_id", "authorization_id"]
  };

  string payment_id = 1;
  string authorization_id = 2;
  int64 authorized_at_unix_ms = 3;
}
```

Why this schema works well with your trio:

* The operator doesn’t need to understand these options; they’re consumed by generation + runtime.
* The outbox dispatcher can publish events using the per-message `stream_ref` + partition key derivation (topic binding is environment config, not proto).
* Idempotency guidance is first-class because duplicates are a real consideration in transactional outbox systems.

---

# Deliverable 2: Generated file tree for one example Domain service (and human-owned split)

Assume repo layout:

* `packages/ameide_core_proto/src/ameide_core_proto/` contains source-of-truth protos
* `packages/ameide_sdk_go/gen/go/` contains **SDK outputs** (Go pb + grpc bindings + domain metadata code)
* `primitives/domain/` contains runnable domain services (human-owned)
* `build/generated/domain/` contains generated runtime skeletons/tests/harness (safe to delete/recreate)

### Example: Payments domain service

```text
repo/
├─ packages/
│  ├─ ameide_core_proto/
│  │  └─ src/ameide_core_proto/…
│  │     ├─ acme/platform/domain/v1/options.proto
│  │     └─ acme/payments/v1/payments.proto
│  │
│  └─ ameide_sdk_go/
│     └─ gen/go/…
│        ├─ acme/platform/domain/v1/options.pb.go
│        ├─ acme/payments/v1/payments.pb.go
│        ├─ acme/payments/v1/payments_grpc.pb.go
│        └─ acme/payments/v1/payments_factmeta_gen.go        # GENERATED by domain plugin
│
├─ build/
│  └─ generated/
│     └─ domain/
│        └─ payments/
│           ├─ runtime/
│           │  ├─ server_gen.go                              # GENERATED: grpc server wrapper + interceptors
│           │  ├─ http_gen.go                                # GENERATED: http mux + health endpoints
│           │  ├─ outbox_gen.go                              # GENERATED: outbox writer + table constants
│           │  ├─ dispatcher_gen.go                          # GENERATED: reliable background dispatcher skeleton
│           │  └─ config_gen.go                              # GENERATED: env var parsing + defaults
│           ├─ tests/
│           │  ├─ red_gen_test.go                            # GENERATED: RED tests (rate/errors/duration)
│           │  └─ integration_harness_gen_test.go             # GENERATED: integration harness
│           └─ doc/
│              └─ conventions_gen.md                          # GENERATED: env/ports/endpoints contract
│
├─ primitives/
│  └─ domain/
│     └─ payments/
│     ├─ cmd/
│     │  └─ payments/
│     │     └─ main.go                                     # HUMAN: tiny main; calls generated runtime Run()
│     ├─ internal/
│     │  ├─ handlers/
│     │  │  └─ payments_handlers.go                        # HUMAN: implements generated Handler interface
│     │  ├─ domain/
│     │  │  └─ authorize.go                                # HUMAN: business logic, pure Go
│     │  └─ wiring/
│     │     └─ deps.go                                     # HUMAN: DI/wiring if desired
│     └─ Dockerfile                                        # HUMAN
│
└─ buf.gen.yaml
```

### Naming + ownership rules (so `clean: true` is safe)

* Everything under **`packages/ameide_sdk_go/gen/go/`** and **`build/generated/domain/`** is treated as **generated-only** and can be deleted/recreated (`clean: true`).
* Human code lives under **`primitives/domain/<name>/...`** only.
* Generated Go files are suffixed `_gen.go` and include `// Code generated … DO NOT EDIT.` headers (standard protoc style).
* Human code never imports proto source directories; it imports Go packages from the Go SDK module path (backed by `packages/ameide_sdk_go/gen/go/...`).

### How drift becomes a compile failure (not runtime surprise)

* If a new RPC is added, `payments_grpc.pb.go` will change (generated by `protoc-gen-go-grpc`) and the generated runtime wrapper’s handler interface changes too, forcing compilation failures until human code implements the method. This relies on standard gRPC-Go generation behavior.
* Embedding `Unimplemented<Service>Server` is enforced in generated wrappers to preserve forward compatibility.

---

# Deliverable 3: Proposed `buf.gen.yaml` plugin list + versioning strategy

## Recommended `buf.gen.yaml` (v2, pinned, clean, deterministic)

This uses:

* Official Go + gRPC Go remote plugins
* One custom “domain-go” plugin (invoked twice) that generates:

  * event metadata helpers in `gen/go/...`
  * runtime skeleton in `build/generated/domain/...`
  * RED tests + integration harness

```yaml
# buf.gen.yaml
version: v2

# Clean outputs before writing new files (prevents stale files on rename/delete).
clean: true

plugins:
  # 1) Go messages
  - remote: buf.build/protocolbuffers/go:v1.36.0
    revision: 1
    out: gen/go
    opt: paths=source_relative

  # 2) Go gRPC service bindings
  - remote: buf.build/grpc/go:v1.6.0
    revision: 1
    out: gen/go
    opt:
      - paths=source_relative
      # Don't set require_unimplemented_servers=false; default requires embedding
      # Unimplemented<...>Server and that's required.

  # 3a) Domain SDK extras (custom) — safe to clean because out is generated-only
  - remote: buf.build/acme/domain-go:v0.9.0
    revision: 3
    out: gen/go
    opt:
      - mode=sdk
      - go_import_prefix=github.com/acme/platform/gen/go

  # 3b) Domain runtime + outbox + tests (custom) — safe to clean because out is generated-only
  - remote: buf.build/acme/domain-go:v0.9.0
    revision: 3
    out: build/generated/domain
    opt:
      - mode=runtime
      - sdk_import_prefix=github.com/acme/platform/gen/go
      - require_domain_service_option=true
```

### Why these Buf choices satisfy “deterministic generation”

* `clean: true` deletes plugin output directories before generation, eliminating stale generated files after deletes/renames.
* Remote plugin pinning:

  * Buf recommends specifying plugin versions for reproducible generation.
  * `revision` further pins the exact build if a plugin is rebuilt/repackaged.
* Plugin order is deterministic (invoked in order, results combined before writing).

## Versioning strategy (practical + enforceable)

1. **Pin every remote plugin** with both `:<version>` and `revision`.
2. **Commit `buf.lock`** to pin schema dependencies; Buf describes it as the dependency manifest and says it MUST NOT be hand-edited.
3. Upgrade flow:

   * update `buf.gen.yaml` plugin versions/revisions in one PR,
   * run `buf dep update` only when intentionally upgrading module deps.

## “Regen in CI and fail on diff” plan

In CI:

```bash
buf generate
git diff --exit-code
```

This is valid because:

* outputs are deterministic and cleaned first (`clean: true`).

---

# Deliverable 4: DomainOperator reconciliation outline

## CRD design: `DomainService` (language-agnostic contract)

### Spec fields (what users configure)

```yaml
apiVersion: ameide.io/v1
kind: Domain
metadata:
  name: payments
spec:
  image: ghcr.io/ameide/payments:1.2.3
  replicas: 2

  ports:
    grpc: 9090
    http: 8080

  expose:
    enabled: true
    gatewayRef:
      name: public-gateway
      namespace: gateway-system
    hostnames:
      - payments.example.com
    pathPrefix: /payments

  db:
    # Reference to a Secret that holds connection info.
    secretRef:
      name: payments-db
    # Migration strategy owned by operator.
    migrations:
      enabled: true
      job:
        image: ghcr.io/ameide/payments:1.2.3   # same image, different command
        command: ["/app", "migrate"]
        backoffLimit: 3

  outbox:
    enabled: true
    dispatcher:
      mode: sidecar            # sidecar | separateDeployment
      interval: 250ms

  resources:
    requests:
      cpu: "200m"
      memory: "256Mi"
    limits:
      cpu: "1"
      memory: "1Gi"
```

### Status.conditions (what operator reports)

Use `.status.conditions[]metav1.Condition` (standard Kubernetes pattern).
Set `observedGeneration` on each condition update (match the Domain CR’s `.metadata.generation`) and use positive-polarity condition types with actionable `reason`/`message` values.

Condition set (baseline):

* `SecretsReady` — all referenced secrets exist (DB, broker, etc.)
* `DependenciesReady` — external deps reachable (DB, broker, etc.)
* `MigrationSucceeded` — migration job completed successfully
* `WorkloadReady` — Deployment Available
* `RouteReady` — HTTPRoute Accepted/Programmed/ResolvedRefs OK (or whichever your Gateway implementation supports)
* `OutboxReady` — dispatcher running and can connect to broker/DB
* `Ready` — aggregate condition (true when all required are true)

Gateway API route conditions vary by implementation, but implementer guidance says routes “usually” include `Accepted`, `Programmed`, and `ResolvedRefs`.

## Reconciliation steps (idempotent control loop)

Kubebuilder emphasizes controllers as reconciliation loops and recommends idempotent reconcile logic.

### Outline (per DomainService)

1. **Fetch DomainService**.
2. **Resolve DB secret** (`spec.db.secretRef`):

   * if missing → set `DBReady=False` and return (requeue).
3. **Ensure ServiceAccount** and labels/ownerRefs.
4. **Migrations (if enabled)**:

   * Ensure a `Job` exists for current `.metadata.generation` (or image digest).
   * Watch Job status; if failed → `MigrationSucceeded=False`.
   * If succeeded → `MigrationSucceeded=True`.
   * Pattern of checking Job completion via status conditions is common in operator tutorials.
5. **Ensure Deployment**:

   * App container env/ports set (see mapping below).
   * Add sidecar dispatcher if `outbox.dispatcher.mode=sidecar`.
6. **Ensure Service**:

   * ports `grpc` and `http`.
   * set port `appProtocol` where appropriate; Gateway API expects implementations to honor it.
7. **Ensure HTTPRoute (if exposure enabled)**:

   * Create HTTPRoute pointing to the Service `http` port.
   * If GatewayRef is cross-namespace, ensure the required `ReferenceGrant` model is respected (Gateway API requires ReferenceGrant for cross-namespace references).
8. **Update status conditions** in a stable order:

   * Use `observedGeneration` and `lastTransitionTime` per `metav1.Condition` conventions.

## RBAC needs (operator)

At minimum:

* `ameide.io`: `domains`, `domains/status`, `domains/finalizers`
* Core: `services`, `configmaps`, `secrets` (read), `serviceaccounts`
* Apps: `deployments`, `replicasets`
* Batch: `jobs`
* Gateway API: `httproutes`, `referencegrants` (depending on your design), and read access to `gateways`

(Exact verbs: get/list/watch/create/update/patch/delete for owned resources; get/list/watch for referenced secrets/gateways.)

## Mapping CRD → runtime conventions (env vars, mounts, health endpoints)

### Health endpoints

* Kubernetes recommends using probes (readiness/liveness/startup) and notes a common pattern is low-cost HTTP endpoints for readiness/liveness.
* gRPC has a standard Health Checking Protocol and servers expose a health checking service.

**Runtime contract (generated):**

* HTTP server:

  * `GET /healthz` → liveness
  * `GET /readyz` → readiness
* gRPC server:

  * implement standard `grpc.health.v1.Health` service (`Check`, `Watch`) for gRPC-native health checks

### Env var contract (language-agnostic)

| Concern        | Operator sets                       | Runtime uses                          |
| -------------- | ----------------------------------- | ------------------------------------- |
| Ports          | `PORT_GRPC`, `PORT_HTTP`            | listen gRPC/HTTP on these             |
| DB             | `DB_DSN` (from secret)              | open DB pool; required before serving |
| Migration mode | `APP_MODE=migrate` (Job)            | run migrations and exit               |
| Outbox         | `OUTBOX_ENABLED`, `OUTBOX_INTERVAL` | enable background dispatcher          |
| Broker         | `BROKER_URLS` / `KAFKA_BROKERS`     | publish events                        |

This keeps the operator stable across languages: every runtime just honors the same env vars.

---

# Deliverable 5: Decision matrices (2+ alternatives)

## A) “Generate gRPC server skeleton from proto” vs “SDK-only + generic handlers”

| Dimension                          | Generate gRPC server skeleton (required)                                                                                                                                                              | SDK-only + generic handlers                                                                                                                    |
| ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Drift detection                    | **Compile-time**: new RPCs change generated server wrapper + handler interface → build fails until implemented. This aligns with gRPC-Go generation expectations (including Unimplemented embedding). | Weaker: generic handler often routes by string/descriptor; missing methods can become runtime errors unless you build extra compile-time glue. |
| Consistency                        | Strong: generator can enforce interceptors, metrics, outbox wiring, health endpoints uniformly.                                                                                                       | Medium: you’ll reinvent conventions in each service unless you also ship a heavy framework.                                                    |
| Developer UX                       | Fast onboarding: add proto → regenerate → implement interface stubs.                                                                                                                                  | Initially simpler, but “where do I add logic?” becomes less obvious; more runtime config.                                                      |
| Language portability               | Medium: you need one server-skeleton generator per language.                                                                                                                                          | High: generic handler concept can be made cross-language, but you still need runtime frameworks per language.                                  |
| Best fit for your success criteria | **Best**: deterministic generation + compile failures catch drift early.                                                                                                                              | Risk: more “shape-only” systems drift into runtime surprises.                                                                                  |

**Recommendation:** generate server skeletons (but keep human code out of generated dirs). It most directly satisfies “drift caught by CI/compile failures, not runtime surprises,” and it aligns with gRPC-Go forward-compatibility conventions.

## B) “One plugin per kind” vs “one monolithic plugin”

| Dimension                | One plugin per kind                                                                                                                                               | One monolithic plugin (required)                                                              |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Versioning complexity    | Higher: you must pin and update many plugin versions/revisions. Buf recommends pinning plugin versions for reproducibility, and you’d multiply that surface area. | Lower: pin exactly one “domain-go” plugin version/revision plus official language plugins.    |
| Atomicity of conventions | Harder: cross-cutting conventions (env vars, health endpoints, outbox metadata) can split across plugins and drift.                                               | Easier: one generator owns the entire runtime contract and can keep it coherent.              |
| Build performance        | Potentially parallelizable, but Buf already invokes plugins in deterministic order and combines results.                                                          | Similar; generator can still output multiple files/packages.                                  |
| Extensibility            | Good if different teams own different areas.                                                                                                                      | Good if you treat the monolith plugin as the “Domain runtime API surface” with strict semver. |

**Recommendation:** use a monolithic **domain runtime plugin** per target language (e.g., `domain-go`, later `domain-java`), because your runtime contract is cross-cutting (gRPC + outbox + HTTP plane + tests). Use separate plugins only when ownership boundaries are strong enough to justify the coordination overhead.

---

## Extra implementation details (ties the trio together cleanly)

### Outbox runtime semantics (what generated code MUST enforce)

Based on AWS + microservices.io + Debezium:

* Write domain state + outbox row in the **same DB transaction** (atomicity requirement).
* Dispatcher publishes from outbox asynchronously; duplicates are possible, so downstream MUST be idempotent (or you provide consumer-side dedupe helpers).
* Preserve ordering where required (use per-event partition key and/or sequence numbers).

**Interoperability requirement (cross-boundary):** emit facts/events as CloudEvents on the wire (headers/attributes depending on transport) and propagate W3C Trace Context (e.g., `traceparent`), mapping the shared `event_fact.type` to CloudEvents `type` so cross-system tooling can interoperate without bespoke envelopes.

### Gateway exposure (HTTPRoute) gotchas to bake into operator conventions

* Cross-namespace references require ReferenceGrant; don’t rely on “it works on my cluster.”
* Set `ServicePort.appProtocol` for `http`/`grpc` ports so Gateways that honor it can route correctly.
* Surface route readiness via conditions consistent with Gateway implementer guidance (`Accepted`, `Programmed`, `ResolvedRefs`).

---

## Summary: why this proposal meets your success criteria

* **Deterministic generation / minimal flags / no bespoke CLI**:
  `buf generate` + pinned remote plugin versions/revisions + `clean: true` + CI diff check.
* **Runtime imports SDK outputs only**:
  Go import paths point to `gen/go/...` via managed mode and consistent `go_package` strategy (no “proto source dir” imports).
* **Operator language-agnostic and stable**:
  It only knows env vars/ports/health endpoints + Kubernetes resources; it does not parse Go. Status uses standard conditions.
* **Drift caught by CI/compile failures**:
  gRPC-Go + generated server wrapper forces compile-time conformance; CI regenerates and fails on diff. Unimplemented embedding ensures forward-compatibility.

If you want, I can also include a concrete example of:

* the generated Go handler interface + wrapper pattern (showing where `Unimplemented<Service>Server` is embedded by value),
* a suggested outbox table DDL + polling/locking approach (Postgres vs MySQL variants),
* and an example HTTPRoute manifest the operator would create from the CRD (hostnames/pathPrefix/backendRef).
