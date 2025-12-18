Note: this document is supporting background; the consolidated, normative spec is `backlog/520-primitives-stack-v2.md`. Examples use placeholder namespaces (e.g., `acme.*`)—the canonical repo proto namespace is `ameide_core_proto.*` per `backlog/509-proto-naming-conventions.md`.

## What NiFi gives you for “flows as code” (concepts + automation surfaces)

### Flow artifacts: Templates, Flow Definitions, and Versioned Flows

**Templates (legacy, XML)**
NiFi supports creating reusable “Templates” from parts of a flow and exporting/importing them as XML. This is explicitly positioned as a way to share reusable building blocks across NiFi instances.

**Flow Definition download (JSON, process-group scoped)**
From a Process Group context menu, NiFi can **download a “flow definition” as a JSON file**, intended for backup or importing into **NiFi Registry** using the **NiFi CLI**. It has an option to include/exclude referenced controller services that live outside the process group (external services).
NiFi also notes that downloading a flow definition from a versioned process group does **not** include version-control metadata.

**Versioned flows with NiFi Registry (required)**
NiFi provides UI and REST flows for **placing a process group under version control**, committing local changes, reverting, and changing versions (including rollback).
This model is what you want when you need reproducible deployments + upgrades without click-ops.

### Parameter Contexts and secrets-safe configuration

**Parameter Contexts are first-class and global**
Parameters live inside **Parameter Contexts**, which are globally defined/accessible in the NiFi instance. A Process Group can be assigned **one** Parameter Context, and a Parameter Context can be assigned to multiple Process Groups.

**Sensitive-safe behavior with versioned flows**
NiFi makes two critical guarantees that match your “secrets stay in Kubernetes” requirement:

1. **Sensitive properties / sensitive values**
   NiFi encrypts “Sensitive Values” in the stored flow configuration and **does not include Sensitive Values in exported Flow Definitions**.

2. **Versioned flows + parameters**
   When exporting a versioned flow to a Flow Registry, NiFi stores parameter metadata, but **Sensitive Parameter values are not stored**.
   Additionally, sensitive properties MUST only reference sensitive parameters, and the value MUST be exactly a single parameter reference (to avoid leaking parts of a secret).

This is the exact mechanism you want:

* generated artifacts contain placeholders and references like `#{erp.api.token}`
* the operator injects actual values from K8s Secrets by updating Parameter Contexts.

### Operational mechanics you can treat as health signals

**Queues and backpressure**
Each Connection “houses a FlowFile Queue.”
NiFi backpressure is configured per Connection using:

* object count threshold (FlowFiles)
* data size threshold (bytes/KB/MB/GB/TB)
  and when exceeded, the upstream component is no longer scheduled. Defaults are `10,000 objects` and `1 GB`.

These are ideal for operator-driven conditions like `BackpressureOK`.

**REST endpoints for health and observability (NiFi API)**
Key endpoints you can use without UI/manual work:

* **Flow status summary**: `GET /flow/status` returns controller status (high-level counts)
* **Bulletins**: `GET /flow/bulletin-board` provides bulletin board access (warnings/errors surfaced by components)
* **System diagnostics**: `GET /system-diagnostics` provides diagnostics (CPU/memory/etc.)
* **Queue inspection**: `GET /flowfile-queues` + `GET /flowfile-queues/{id}` for queue details
* **Provenance events**: `GET /provenance-events` (and related search endpoints)

### Deployment and upgrade mechanics you can automate

**Import/upload and upgrade actions (NiFi API)**
NiFi REST exposes automation primitives you can build an operator around:

* **Upload a “versioned flow definition” and create a process group**:
  `POST /process-groups/{id}/process-groups/upload`

* **Replace process group** (for non-registry style replacements / “flow definition swaps”):
  `POST /process-groups/{id}/replace-requests` (async replacement flow)

* **Version control**:

  * download latest versioned process group: `GET /versions/process-groups/{id}/download`
  * initiate revert request: `POST /versions/revert-requests/process-groups/{id}` (async; stops processors / disables controller services and restarts)
  * initiate update request to a different version: `POST /versions/update-requests/process-groups/{groupId}` (async update)

**Parameter Context API surfaces**
NiFi REST includes endpoints for Parameter Context creation and assignment (operator-managed config): `POST /parameter-contexts`, `GET /parameter-contexts`, and process-group assignment endpoints.

---

## NiFi Registry: what to use and what to automate

NiFi Registry’s REST API is explicitly “an interface to a registry with operations for saving, versioning, reading NiFi flows and components.”
It provides bucket/flow/version primitives such as:

* `POST /buckets`
* `POST /buckets/{bucketId}/flows`
* `POST /buckets/{bucketId}/flows/{flowId}/versions`
* `GET /buckets/{bucketId}/flows/{flowId}/diff/{versionA}/{versionB}`
* `GET .../versions/{versionNumber}/export`
* `POST .../versions/import`

That’s enough to implement “flow snapshot as code” pushed from CI **or** from your operator (reading the generated snapshot artifact).

---

## Kubernetes deployment options for NiFi

### “Official” building blocks (vendor sources)

* NiFi clustering is ZooKeeper-backed; ZooKeeper elects a **Cluster Coordinator** and a **Primary Node**, and nodes report heartbeat/status to the coordinator.
* Cluster setup is driven by `nifi.properties` (e.g., `nifi.cluster.is.node`, heartbeat, secure cluster protocol flag).
* NiFi security configuration includes:

  * `nifi.sensitive.props.key` for encrypting sensitive properties
  * keystore/truststore paths and passwords (`nifi.security.keystore`, `nifi.security.truststore`, etc.)
  * keystore/truststore auto-reload (`nifi.security.autoreload.enabled`)

### Approach 1: Helm-managed NiFi (community, common, but caveats)

The widely used `cetic/helm-nifi` chart exists, but the maintainers explicitly state **it is not maintained anymore** (“Maintainers Wanted / project is not maintained”).
It also shows a typical pattern (StatefulSet + storage + service exposure knobs, using `apache/nifi` image).

**Implication**: Helm can install NiFi, but it won’t naturally give you:

* CRD-driven flow lifecycle
* strongly reconciled “desired state” for process groups, parameter contexts, registry versions
* rich status conditions per integration

You’d have to bolt those on.

### Approach 2: Operator-managed NiFi (community operator pattern)

NiFiKop positions itself as an operator to automate provisioning/operations of NiFi clusters on Kubernetes, defining a `NifiCluster` object and reconciling resources.
It also calls out features like rolling upgrades, encrypted communication using SSL, and “Advanced Dataflow … management via CRD.”
It explicitly highlights “dataflow lifecycle management” including registry client, parameter context, and dataflow via K8s resources.

**Implication**: Even if you don’t adopt NiFiKop wholesale, the operator approach aligns directly with your primitive’s goals: reconcile flows + params + status, no click-ops.

---

## Default architecture (MUST): reuse NiFiKop as the NiFi control plane (avoid reinvention)

If your IntegrationOperator is intended to manage **NiFi clusters + registry + dataflows + parameter contexts**, you are overlapping heavily with NiFiKop’s problem space. A “best-practice” architecture is:

* Treat **NiFiKop** as the underlying NiFi control plane (cluster lifecycle, Registry integration, dataflow lifecycle, parameter contexts with Secret refs, autoscaling).
* Make your **Integration primitive/operator** a *thin adapter*:
  * reconcile `Integration` (your CR) → a set of NiFiKop CRs (cluster refs, parameter contexts, registry clients, dataflows/process groups, rollout strategy)
  * keep reconcile short and idempotent; rely on NiFiKop’s controllers for the heavy lifting
  * map NiFiKop/NiFi status into your primitive’s `status.conditions[]` (`FlowSynced`, `BackpressureOK`, `BulletinsOK`, `Ready`)

This reduces maintenance burden and leverages upstream battle-testing (including ordering/rollout edge cases around parameter context readiness and flow updates).

## Security model that matches your success criteria

### Secrets in Kubernetes, not in generated artifacts

* Generated NiFi flows MUST reference parameters (`#{...}`), and sensitive properties must reference **only** sensitive parameters, with a single parameter reference value.
* Versioned flows exported to registry do **not** store sensitive parameter values.
* NiFi encrypts sensitive values at rest in its flow config, controlled by `nifi.sensitive.props.key`.

### TLS and authentication

* TLS is configured via keystore/truststore properties in `nifi.properties`.
* For auth, NiFi supports:

  * default **Single User** login identity provider (auto-generated credentials)
  * OIDC config properties and dedicated REST callback resources for OIDC flows

**Operator implication**: Your IntegrationOperator MUST support:

* mTLS between operator and NiFi API
* OIDC/SAML/SingleUser depending on environment
* always manage `nifi.sensitive.props.key` via K8s Secret (and never rotate casually; rotation requires migration tooling)

---

# Recommended “NiFi flows as code” strategy

## Choose NiFi Registry versioned flows as the canonical deployment unit

**Why this is the “smartest default”:**

1. **Built-in upgrade semantics**: NiFi supports versioned flows with “commit”, “revert local changes”, “change version”, including rollback.
2. **Parameter behavior is designed for multi-env**: exporting to registry stores parameter metadata but **not sensitive values**, and import merges contexts by name (sensitive values remain unset).
3. **API support for operator workflows**: async update/revert requests are first-class, so your operator can coordinate safe rollouts and report conditions.
4. **Drift control**: “local changes” become something you can detect + fail on (FlowSynced / DriftDetected), instead of silently diverging.

## Keep flow-definition JSON as a fallback / bootstrap path

NiFi also supports downloading process-group “flow definition” JSON and uploading “versioned flow definition” files via REST.
Use this when:

* registry isn’t available in some env
* you need a “single artifact” import/export workflow

But for day-2 operations and upgrades, Registry wins.

---

# Proto shape: custom options that describe ports + contracts cleanly

## Design goals

* Proto is the source of truth for:

  * ports (direction + transport binding)
  * message schema types on each port
  * mapping hints (content-type, routing keys, required headers) are not expressed in proto in v2
* Minimal “stringly-typed” references.
* Easy for plugins to consume via descriptors.

## Proposed option schema

### Core idea: ports are declared as RPC methods, not for RPC transport, but for descriptors

This avoids having to put message type names in strings: the method input type *is* the schema.

You define:

* a **service** per integration (purely declarative)
* each **rpc** = one port
* attach a custom `port` option to each method

### `integration/options.proto` (conceptual)

Define custom options as **descriptor extensions** (this is standard protobuf custom options). Put the *extension declarations* in a dedicated `syntax = "proto2";` file (e.g., `acme/integration/v1/options.proto`), and keep your integration port declarations and message schemas in normal `proto3` files that import those options.

* Extend `google.protobuf.MethodOptions` with:

  * `direction`: IN | OUT
  * `transport`: oneof { kafka, http, s3, … }
    * `kafka`: use a logical `binding_ref` (CRD/runtime binds to env-specific topic/consumer group/auth)
    * `http`: `method` plus an `endpoint_ref` (logical name bound by CRD/runtime), not a literal URL
  * `content_type`
  * `routing_key` / partition key hints
  * required headers list
  * “expected envelope” (domain fact vs integration fact)

* Extend `google.protobuf.MessageOptions` with:

  * `fact_type` / `schema_subject` (for schema registry naming)
  * compatibility policy hint (e.g., FILE vs PACKAGE breaking strictness)
  * reuse a shared cross-primitive event/fact metadata module (so Domain/Process/Projection/Integration don’t drift)

(You’ll define these with `extend google.protobuf.*Options` using `descriptor.proto`—standard protobuf custom options.)

**Hard rule:** mark Ameide-specific options as **SOURCE-retained** by default so they are available to codegen but do not leak into runtime descriptor pools; generators must read such options from `CodeGeneratorRequest.source_file_descriptors`.

**Guardrail:** do not place per-environment endpoints/secrets in proto. Bind those via the Integration CRD and inject into NiFi (e.g., Parameter Context values) at runtime.

**Interoperability requirement (cross-boundary):** where integrations consume/emit events/facts, use CloudEvents-compatible envelopes and propagate W3C Trace Context (e.g., `traceparent`) across NiFi boundaries (Kafka headers, HTTP headers) so downstream observability and tooling align on shared standards.

## Example `.proto` snippet

```proto
syntax = "proto3";

package acme.integration.order_to_erp.v1;

import "google/protobuf/empty.proto";
import "google/protobuf/timestamp.proto";

// Your shared integration option definitions live here:
import "acme/integration/v1/options.proto";
import "ameide_core_proto/common/v1/eventing.proto";

// Domain facts (owned elsewhere):
import "acme/domain/orders/v1/facts.proto";

// --- Integration-owned schema (what you emit / write) ---

message ErpOrderUpsertRequest {
  string erp_order_id = 1;
  string source_order_id = 2;
  google.protobuf.Timestamp updated_at = 3;

  // Optional mapping hint: headers that must be present for outbound HTTP calls, etc.
  map<string, string> required_headers = 10;
}

// Optional: emitted fact/event envelope in your “integration facts” domain
message ErpUpsertedFact {
  string source_order_id = 1;
  string erp_order_id = 2;
  google.protobuf.Timestamp occurred_at = 3;

  option (ameide_core_proto.common.v1.event_fact) = {
    type: "integration.erp.order_upserted.v1"
    stream_ref: "integration.erp.facts"
  };
}

// --- Port declarations (the contract surface) ---
// Not RPC in reality: this is a descriptor container for tooling/operators.

service OrderToErpIntegration {
  // Consumes domain facts from Kafka
  rpc OrdersIn(acme.domain.orders.v1.DomainFactEnvelope) returns (google.protobuf.Empty) {
    option (acme.integration.v1.port) = {
      direction: IN
      transport: {
        kafka: {
          binding_ref: "orders-in"
        }
      }
      content_type: "application/x-protobuf"
      envelope: DOMAIN_FACT
    };
  }

  // Writes to ERP over HTTP
  rpc ErpHttpOut(ErpOrderUpsertRequest) returns (google.protobuf.Empty) {
    option (acme.integration.v1.port) = {
      direction: OUT
      transport: {
        http: {
          method: POST
          // Don't place per-environment URLs in proto; bind via CRD/runtime config.
          endpoint_ref: "erp-orders"
        }
      }
      content_type: "application/json"
      required_headers: ["Authorization", "X-Correlation-Id"]
    };
  }

  // Emits an integration fact to Kafka (required)
  rpc ErpFactOut(ErpUpsertedFact) returns (google.protobuf.Empty) {
    option (acme.integration.v1.port) = {
      direction: OUT
      transport: { kafka: { binding_ref: "erp-facts-out" } }
      content_type: "application/x-protobuf"
      envelope: INTEGRATION_FACT
    };
  }
}
```

---

# Buf plugins: what to generate deterministically

## Why Buf is the right guardrail surface

Buf’s core capabilities line up with your CI guardrails:

* `buf generate` is driven by `buf.gen.yaml`
* generation supports ordered plugin execution and composition
* breaking change detection is a first-class workflow
* custom plugins can be packaged and pushed (including as remote plugins)
* protoc plugin invocation model is standard (“protoc-gen-NAME”)

Determinism requirements (non-negotiable):

* pin remote plugin versions (and ideally revisions) in `buf.gen.yaml`
* use `clean: true` for generated-only output directories so deletes/renames are correct
* enforce a regen-diff gate in CI (`buf generate`, fail if `git diff` is non-empty)
* implement custom generators on a standard plugin harness (e.g., Buf’s `bufbuild/protoplugin`) and add golden tests that assert byte-for-byte deterministic output from fixed descriptor inputs

## Plugin outputs for the Integration primitive

### 1) NiFi flow artifact

**Default**: generate a **NiFi Registry “versioned flow snapshot” JSON** representing a process group skeleton:

* boundary: standardized ingress/egress subgroups per port
* internal queue topology
* controller service stubs required by declared transports
* parameter references for anything env-specific

This aligns with NiFi’s registry model and its REST API for saving/versioning flows.

**Fallback**: also emit a “flow definition JSON” compatible with NiFi’s process group “download flow definition” notion, for environments using upload/import without a registry.

### 2) Parameter Context definitions (placeholders only)

Generate:

* `parameter-context.yaml` (or JSON) listing:

  * parameter name
  * description
  * sensitive flag
  * default: empty/placeholder
* generator ensures:

  * any sensitive processor property references **only** a sensitive parameter and uses `#{param}` exactly

This is the “secrets in K8s only” keystone.

**NiFi alignment:** NiFi’s versioned-flow + Parameter Context model is explicitly designed for multi-env: parameter metadata is stored, but sensitive values are not stored in exported artifacts. Treat this as a hard rule: proto and generated artifacts carry placeholders/references only; the operator binds actual values at runtime via Parameter Context updates.

### 3) Schema registry artifacts

Generate at minimum:

* `descriptors.pb` (FileDescriptorSet) for all message types used by ports (domain + integration)

Optionally:

* JSON Schema (or AsyncAPI-style docs) derived from protos using existing schema plugins (Buf ecosystem includes schema-generation plugin repos you can build on, e.g., protoschema-plugins).

### 4) Tests and harness scaffolding

Generate two layers:

**Structural tests (fast, deterministic, run in CI always)**

* parse the generated flow snapshot JSON and assert:

  * a port exists for every annotated rpc
  * parameter names referenced in the flow exist in the parameter context file
  * no sensitive values appear in the exported artifacts (NiFi doesn’t include sensitive values in exported flow definitions; your generator MUST also never write them)
  * connection backpressure thresholds are present and match defaults/overrides

**Behavior tests (integration harness, run in CI or nightly)**

* spin up ephemeral NiFi + NiFi Registry
* push the snapshot to registry and deploy it
* set parameters via REST (secrets injected as test values)
* drive inputs and validate outputs
* use NiFi REST for black-box assertions:

  * queue state and pressure via flowfile queue endpoints
  * bulletins for errors
  * provenance to confirm end-to-end movement

### Generated vs implementation-owned split

**Generated (clobber-safe)**

* SDK outputs under `packages/ameide_sdk_ts/src/_proto/**`, `packages/ameide_sdk_python/src/ameide_sdk/_proto/**`, `packages/ameide_sdk_go/internal/proto/**`, and `packages/ameide_sdk_go/proto/**`
* `build/generated/integration/<integration>/nifi/flow.snapshot.json`
* `build/generated/integration/<integration>/nifi/parameter-context.yaml`
* `build/generated/integration/<integration>/schemas/descriptors.pb`
* `build/generated/integration/<integration>/tests/*_structural_test.*`
* `build/generated/integration/<integration>/harness/deploy_and_assert.*`

**Implementation-owned**

* `primitives/integration/<integration>/**` (mapping scripts/transforms, custom processors/NARs, and human behavior tests/golden cases)

**How to prevent clobbering**

* generator never edits human directories
* generator references human assets only via **parameters** (e.g., `#{mapping.script.path}`), so you can swap them without rewriting the flow snapshot
* regeneration is enforced via `regen-diff` in CI (see below)

---

# Operator internals: CRD surface and reconcile strategy

## CRD design

You want two layers (even if implemented by one operator):

### 1) `NiFiCluster` (platform resource)

Fields (spec):

* image/version (NiFi + optionally Registry)
* replicas, resources
* storage (separate PVs for repos; admin guide emphasizes external locations for repositories to ease upgrades)
* cluster properties + ZooKeeper connectivity (cluster is ZooKeeper-driven)
* ingress/service exposure
* security:

  * keystore/truststore secret refs (`nifi.security.keystore`, `nifi.security.truststore`, etc.)
  * `nifi.sensitive.props.key` secret ref
  * auth mode (SingleUser / OIDC), incl. OIDC endpoints if configured

Status:

* `NiFiReady` (pods up, cluster formed, API reachable)
* `RegistryReady` (if managed)

### 2) `Integration` (your new primitive)

Fields (spec):

* `clusterRef`
* `artifactRef`:

  * configmap ref OR OCI blob digest containing:

    * flow snapshot
    * parameter context schema
    * descriptors.pb
    * mapping scripts
* `registryTarget`:

  * bucket/name and flow name
  * desired version strategy (pin, latest, semver map)
* `parameters`:

  * `contextName`
  * `valuesFrom`: list of secret/configmap refs to map into parameter keys
  * mark which are sensitive (so operator uses correct APIs)
* `rolloutStrategy`:

  * `InPlaceVersionUpdate` (default): sync process group to a new registry version
  * `BlueGreen` deployments are not supported in v2
* `operationalPolicy`:

  * backpressure thresholds defaults/overrides (align with NiFi semantics)
  * provenance retention knobs (or at least repo sizing hooks)
  * retry policy defaults for adapters (where applicable)

Status conditions:

* `SecretsReady`
* `DependenciesReady` — NiFi reachable + Registry reachable + auth/TLS configured
* `WorkloadReady` — NiFi cluster healthy (pods/cluster)
* `FlowSynced` — versioned PG “up to date” (no drift)
* `FlowRunning` — processors scheduled and running as expected
* `BackpressureOK` — queues below thresholds
* `BulletinsOK` — no ERROR bulletins
* `Ready` — umbrella condition: all required are true

(Use `metav1.Condition` everywhere and set `observedGeneration` to the Integration CR’s `.metadata.generation`; keep condition types positive-polarity and use `reason`/`message` for detail.)

Primitive-specific (required):

* `RegistrySynced` — snapshot imported/version created
* `FlowDeployed` — process group created/updated
* `EndpointReachable` — sinks reachable

## Reconcile outline

1. **Ensure cluster exists and secure baseline config**

* set `nifi.sensitive.props.key` from Secret
* mount keystore/truststore and configure `nifi.security.*`
* if using OIDC, configure `nifi.security.user.oidc.*` properties and rely on OIDC REST resources

2. **Ensure Registry has the desired flow version**

* call NiFi Registry REST:

  * create bucket/flow if missing
  * import or create a new version from the generated snapshot (`.../versions/import` or `.../versions`)
* record the produced `(bucketId, flowId, versionNumber)` in Integration status

3. **Deploy or update the NiFi process group**

* initial deploy: upload/import the versioned flow definition to create the process group
  `POST /process-groups/{id}/process-groups/upload`
* upgrades:

  * use registry version update requests (async): `POST /versions/update-requests/process-groups/{groupId}`
  * support revert requests for drift cleanup: `POST /versions/revert-requests/process-groups/{id}`

4. **Apply Parameter Context values**

* create/update Parameter Context via `/parameter-contexts` endpoints and assign to the integration PG

---

## Reconcile discipline note (keep reconcile short)

NiFi flow operations can be heavy (uploads, version updates, stop/start cycles, drift checks). Keep reconcile responsive:

* Reconcile MUST be **idempotent** and **short**: do not block the controller on long NiFi update requests.
* For long-running NiFi operations, use NiFi’s async request patterns (or an owned Job/work queue) and track progress in `status.conditions[]` (`FlowSynced=False reason=SyncInProgress`, etc.).
* Watch owned resources (and any “request ID” state) and requeue on events, not sleep loops.
* inject non-sensitive values from ConfigMaps and sensitive values from Secrets
* verify sensitive constraints: sensitive properties reference only sensitive parameters

5. **Start and monitor**

* poll:

  * `/flow/status` for coarse state
  * `/flow/bulletin-board` for errors
  * `/flowfile-queues` for queues/backpressure
  * `/system-diagnostics` for node health
* set status conditions accordingly

## Runtime contract between operator and generated artifacts

* Generated flow snapshot contains **parameter references only** (no values)
* Generated parameter context file defines the keyspace + sensitivity classification
* Operator is responsible for:

  * mapping K8s Secret/ConfigMap keys → parameter keys
  * ensuring sensitive values never appear in Git or generated files (rely on NiFi’s guarantees + your own checks)

---

# CI guardrail plan

## 1) Buf checks (always)

* `buf lint` (style/import policy)
* `buf breaking` against mainline (schema compatibility)
  Buf positions breaking change detection as a core workflow, with configurable strictness categories (FILE / PACKAGE / WIRE / WIRE_JSON).

## 2) Deterministic regeneration checks

* `buf generate` is configured via `buf.gen.yaml`
* CI runs generation and fails if `git diff` is non-empty (`regen-diff` gate)

## 3) Structural tests (generated)

* run generated structural tests on every PR (fast)
* include “no sensitive literals” assertions (belt + suspenders with NiFi export behavior)

## 4) Behavior tests (hybrid)

* generated harness deploys to ephemeral NiFi + Registry
* human-authored “behavior tests” provide golden vectors:

  * input fact → expected outbound request/body/event
* assertions use NiFi APIs:

  * provenance (`/provenance-events`)
  * bulletins (`/flow/bulletin-board`)
  * queues (`/flowfile-queues`)

---

# Decision matrix

## NiFi-only integration logic vs NiFi plus sidecar adapter service

| Dimension                  | NiFi-only                                              | NiFi + small sidecar adapter                                             |
| -------------------------- | ------------------------------------------------------ | ------------------------------------------------------------------------ |
| Strong typing & validation | Limited to what NiFi processors/record readers enforce | Strong: sidecar can use generated SDKs and strict protobuf validation    |
| Mapping complexity         | Script-heavy transformations can get brittle           | Complex transformations live in code with tests                          |
| Operational footprint      | Fewer moving parts                                     | More pods + versioning + monitoring                                      |
| Upgrade model              | Pure “flow version” rollout                            | Flow + service rollout coordination                                      |
| Performance                | Great for streaming routing/ETL; depends on processors | Potential extra hop/latency but predictable CPU/mem profiles             |
| Security boundaries        | All inside NiFi                                        | Clear boundary: NiFi calls sidecar over mTLS/localhost, secrets isolated |

**Rule of thumb**

* Start **NiFi-only** when the integration is mostly routing, format transforms, enrichment via standard connectors.
* Introduce a sidecar when you need **hard guarantees** (strong validation, complex business mapping, strict contract enforcement) without embedding complex logic in NiFi scripts.

## Helm-managed NiFi vs Operator-managed NiFi

| Dimension       | Helm (e.g., cetic/helm-nifi)                            | Operator (your IntegrationOperator / NiFiKop-style)                                                   |
| --------------- | ------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Day-2 lifecycle | Mostly manual / external automation                     | Reconciled desired state, CRD-driven                                                                  |
| Flow deployment | Not inherently modeled                                  | First-class (deploy/update/version sync)                                                              |
| Health/status   | Indirect                                                | Native conditions (FlowSynced, BackpressureOK, etc.)                                                  |
| Upgrade safety  | Helm upgrade patterns, but app-aware draining is on you | Can implement app-aware rollouts (stop sources, drain queues)                                         |
| Project health  | Chart explicitly “not maintained anymore”               | Operator approach aligns to CRD pattern; NiFiKop advertises rolling upgrades + CRDs for dataflow mgmt |

---

# Extra operator considerations for safe upgrades

NiFi upgrade guidance includes:

* preserve custom processors by using a separate custom NAR library directory
* stop sources and drain queues before shutdown
* keep repositories in external locations to ease upgrades

Your operator MUST encode these as:

* a rollout preflight (block upgrades if queues non-empty beyond policy)
* a “drain mode” (stop ingress ports first)

---

## Bottom line implementation recommendation

1. **Use NiFi Registry versioned flows as the canonical flow artifact**, and drive deployments via the NiFi + NiFi Registry REST APIs (no click-ops, no bespoke CLI).
2. **Proto declares ports as annotated service methods**, so plugins can deterministically generate:

   * flow snapshots
   * parameter contexts (placeholders only)
   * descriptor sets / schemas
   * structural tests + harness
3. **IntegrationOperator reconciles**:

   * NiFi cluster (secure by default, secrets via K8s)
   * registry versions
   * process group sync + parameter context injection
   * status conditions from NiFi REST signals (status/bulletins/queues/diagnostics)

If you want, I can translate the above into:

* a concrete CRD YAML schema (`openAPIV3Schema`) for `Integration`
* a minimal “generated artifact bundle” layout with exact filenames + digests
* a reconcile state machine (phases + transitions) keyed to the REST endpoints we cited
