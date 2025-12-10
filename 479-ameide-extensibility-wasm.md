Here’s a complete rewritten spec you can drop in as something like
`4xz-ameide-extensibility-wasm.md`.

---

# 479 – Ameide Extensibility (Tier 1 WASM + Tier 2 Primitives)

**Status:** Draft v1
**Owner:** Architecture / Platform
**Audience:** Platform engineers, domain architects, solution architects, security
**Depends on:** 470 (Vision), 471 (Business), 472 (App/Info), 473 (Tech), 478 (Extensions), 476 (Security & Trust)     

---

## 1. Purpose

This document defines Ameide’s **extensibility model**, with two coordinated tiers:

1. **Tier 1 – Small extensions via WASM**
   Agent‑ and human‑authored **WebAssembly (WASM) extensions**, executed by a **shared platform runtime** in `ameide-{env}`, wired into:

   * **Processes (BPMN)** – “extension tasks” in ProcessDefinitions. 
   * **Domains** – explicit extension points (CoC‑style hooks) in Domain primitives. 
   * **Agents** – deterministic tools for Agent primitives.

2. **Tier 2 – Large extensions via primitives (existing 478 model)**
   Full **Domain/Process/Agent primitives** running in **tenant namespaces** (`tenant-{id}-{env}-base` / `tenant-{id}-{env}-cust`) created via **Backstage templates + GitOps**. 

Goals:

* Give tenants and agents a **lightweight way** to add custom business logic without spinning up new services for every small change.
* Keep **transformation** as the owner of design‑time artifacts (ProcessDefinitions, AgentDefinitions, ExtensionDefinitions). 
* Align mentally with **SAP** (in‑app vs side‑by‑side) and **Dynamics X++ Chain of Command** patterns.
* Preserve and extend the **security and tenancy invariants** defined in 470/476/478.  

---

## 2. Positioning & Context

### 2.1 Relationship to 470–473 & 478

This spec sits directly below the Vision + Architecture suite:

* 470 – establishes **Design ≠ Deploy ≠ Runtime** and Transformation as a domain. 
* 471 – defines **business concepts** (tenants, orgs, processes, agents, workspaces). 
* 472 – defines **Domain/Process/Agent primitives**, Transformation design tooling and Transformation Domain primitive. 
* 473 – defines **K8s/GitOps/Backstage/Temporal** tech stack. 
* 478 – defines **Tier 2 tenant extensions**: repo strategy, namespace topology, Backstage templates, PrimitiveImplementationDraft. 

This document **does not replace 478**. Instead:

* Tier 2 (478) remains the model for **new primitives & services**.
* This doc introduces Tier 1 as a **lighter path** for small, often agent‑generated extensions.

### 2.2 Two-tier extensibility at a glance

**Tier 1 – WASM extensions (“small customs”)**

* Unit: `ExtensionDefinition` (design‑time artifact, stored in Transformation). 
* Runtime: `WasmExtensionRuntime` shared service in `ameide-{env}`.
* Invoked from:

  * BPMN Extension Tasks in ProcessDefinitions,
  * Domain extension points,
  * Agent deterministic tools.
* Best for:

  * small validation rules,
  * pricing/discount tweaks,
  * routing/approval decisions,
  * deterministic helper functions for agents.

**Tier 2 – Primitives (“big customs”)**

* Unit: Domain/Process/Agent primitive created via **Backstage**, deployed via GitOps.
* Runtime: services in:

  * `ameide-{env}` for platform product, and
  * `tenant-{id}-{env}-base` / `tenant-{id}-{env}-cust` for tenant code, per 478. 
* Best for:

  * new domains or major process variants,
  * integrations,
  * multi‑endpoint services with their own APIs & storage.
* Runtime representation: `Domain`, `Process`, and `Agent` CRDs reconciled by Ameide operators across platform and tenant namespaces.

Backstage is now clearly **Tier 2‑only** for new primitives; it does not sit in the hot path for small Tier 1 WASM extensions.

### 2.3 Alignment with proto-first & CQRS patterns

Tier 1 and Tier 2 extensions ride on the same substrate described elsewhere:

* Contracts live in `packages/ameide_core_proto` (Buf-managed) and flow into workspace SDKs + GitOps manifests per [472 §2.7](472-ameide-information-application.md#27-contract--implementation--runtime-chain). Extension runtime APIs (`InvokeExtension`, host-call adapters) follow the same Buf/SDK conventions documented in [480](480-ameide-extensibility-wasm-service.md).
* Process/Domain primitives that wrap Tier 1 hooks or Tier 2 tenant code inherit the Watermill CQRS plumbing (commands/events) described in [472 §3.3.1](472-ameide-information-application.md#331-event-driven-cqrs-runtime-watermill) so the event stream stays uniform.
* SDK publishing for extension tooling follows the outer-loop policies in [388](388-ameide-sdks-north-star.md) but internal dev/CI images always consume workspace SDKs (no registry dependency), consistent with [402](402-buf-first-generated-clients.md)/[403](403-python-buf-first-migration-plan.md)/[404](404-go-buf-first-migration.md).

Readers should treat this spec as “how Tier 1 WASM and Tier 2 primitives fit into that existing chain,” not as a new architectural layer.

---

## 3. Conceptual Model for Tier 1 (WASM)

### 3.1 ExtensionDefinition

A **design‑time artifact** managed by the Transformation Domain primitive (Transformation design tooling), analogous to ProcessDefinitions and AgentDefinitions. 

```text
ExtensionDefinition
  id: string
  tenant_id: string
  kind: "process_hook" | "domain_hook" | "agent_tool"
  abi: "json_in_json_out_v1"
  wasm_blob_ref: string        # pointer to object storage
  version: int
  limits:
    cpu_millis: int
    memory_mb: int
    timeout_ms: int
  risk_tier: "low" | "medium" | "high"
  metadata:
    description: string
    created_by: "user:<id>" | "agent:<id>"
    tags: [string]
```

**Kinds:**

1. `process_hook` – invoked by **Process primitive** as an Extension Task in BPMN.
2. `domain_hook` – invoked by **Domain primitive** at explicit extension points (CoC‑style).
3. `agent_tool` – invoked by **Agent primitive** as a deterministic helper.

**ABI**: For v1, all kinds share:

```text
fn run(input_json: bytes) -> output_json: bytes
```

### 3.2 WasmExtensionRuntime (shared platform service)

A **multi‑tenant platform service** per environment:

* Namespace: `ameide-{env}` (platform plane). 
* Owned: **entirely by Ameide**; code lives in platform repo (not tenant repos). 
* Responsibilities:

  1. Resolve `(tenant_id, extension_id, version)` to a WASM module (via `wasm_blob_ref`) stored in the shared **MinIO object storage service** (`data-minio` ApplicationSet component).
  2. Execute `run(input_json)` in a WASM sandbox with:

     * CPU/mem/time limits,
     * No direct FS/network/env access,
     * A small host API (logging + optional host calls).
  3. Enforce host‑call policies:

     * All host calls use **existing Ameide APIs** with **user/tenant context** (see §6.3).

**RPC surface** (internal‑only, used by primitives):

```protobuf
service WasmExtensionRuntime {
  rpc InvokeExtension(InvokeExtensionRequest)
      returns (InvokeExtensionResponse);
}

message ExecutionContext {
  string tenant_id      = 1;
  string org_id         = 2;
  string user_id        = 3;  // "system" or agent id allowed
  string actor_type     = 4;  // USER | AGENT | SYSTEM
  string session_id     = 5;
  string correlation_id = 6;
  string env            = 7;  // dev / staging / prod
  string risk_tier      = 8;  // LOW | MEDIUM | HIGH
}

message InvokeExtensionRequest {
  string extension_id   = 1;
  string version        = 2;  // or "latest"
  ExecutionContext ctx  = 3;
  bytes  input_json     = 4;
}

message InvokeExtensionResponse {
  bytes  output_json    = 1;
  string error_code     = 2;
  string error_msg      = 3;
}
```

### 3.3 Where it sits in the existing architecture

* **Design‑time**: Transformation Domain primitive + Transformation design tooling manage ExtensionDefinitions, versions, and promotions with the same revision/promotion pattern as ProcessDefinitions and AgentDefinitions. 
* **Deploy‑time**: WasmExtensionRuntime is deployed as a **standard platform Helm chart** in `ameide-{env}`, via the same GitOps & ApplicationSet patterns as other platform primitives. 
* **Runtime**:

  * Process primitives, Domain primitives, Agent primitives call `InvokeExtension(...)` with a populated `ExecutionContext`.
  * All durable state, auth, and tenancy enforcement remain in primitives and domains; WASM is “pure function + host calls”.

### 3.4 Deployment approach & operator options

We do **not** need a special Wasm operator for the Tier 1 runtime. The recommended path is to treat `wasm-extension-runtime` like any other platform service:

1. Build a container that embeds Wasmtime/WasmEdge, loads tenant modules from object storage, and exposes `InvokeExtension` over gRPC/HTTP.
2. Package that binary as a **Helm chart** (e.g. `charts/wasm-extension-runtime`) with a Deployment, ClusterIP Service, config for storage buckets and host-call allowlists, and standard resource limits.
3. Add an ArgoCD `Application`/`ApplicationSet` entry so each `ameide-{env}` namespace gets the runtime just like our other primitives. Sync/rollback flows stay identical to the rest of the platform plane.

This keeps all security invariants: code is platform-owned, modules are data, and nothing special runs on the worker nodes.

If we ever need to run Wasm workloads as **pods** (not just within our service), we can install node/runtime operators via ArgoCD-managed Helm charts:

* **KWasm operator** – installs a containerd shim and `RuntimeClass` for WasmEdge/Wasmtime so pods can set `runtimeClassName: wasmedge`. Still alpha in places and only solves node-level execution.
* **SpinKube** – bundles the Fermyon Spin operator + shim; introduces a `SpinApp` CRD and a specific component model that differs from our JSON-in/JSON-out hooks.
* **wasmCloud platform chart** – deploys wasmCloud’s lattice/operator; powerful but opinionated, with its own capability contracts and NATS dependencies.

Those frameworks are useful if we adopt node-level Wasm generally, but they are **optional** for Tier 1. We stay leanest and most controllable by simply shipping our runtime service via Helm + ArgoCD.

### 3.5 Storage substrate (MinIO)

`wasm_blob_ref` objects live in Ameide’s existing **MinIO installation** (`data-minio`, see backlog/380). The storage model is:

* Buckets/prefixes partitioned by tenant (`s3://extensions/<tenant>/<extension>/<version>.wasm`), owned by Transformation but deployed via the shared MinIO chart.
* Access controlled via MinIO service users and policies managed by the same provisioning jobs that serve other workloads. The runtime receives credentials through ExternalSecrets so primitives never handle MinIO keys directly.
* Promotion between environments copies (or re-encrypts) blobs into the destination environment’s MinIO namespace; the `wasm_blob_ref` encodes bucket, key, and checksum so the runtime can verify integrity before caching modules locally.

Using MinIO keeps Tier 1 aligned with the rest of the platform’s artifact storage strategy and avoids inventing a bespoke blob service.

---

## 4. Hook Points

### 4.1 Process hooks (BPMN extension tasks)

**Design‑time**

* Transformation design tooling’s BPMN modeller (React Flow) gains an **“Extension Task”** type. 
* Each Extension Task references:

  * `extension_id`
  * Optional `version` (or `latest`)
  * Input mapping (process variables → JSON)
  * Output mapping (JSON → process variables)

**Runtime**

When a Process primitive (Temporal‑backed) executes an Extension Task:

1. It gathers input variables according to the mapping.
2. Builds `ExecutionContext` from process context (tenant, org, user, actor type, correlation id).
3. Calls `WasmExtensionRuntime.InvokeExtension()`.
4. Writes the result into process variables and continues.

Usage examples:

* Custom **discount/routing rule** per tenant.
* Extra **approval logic** before moving to a next stage.

### 4.2 Domain hooks (CoC‑style)

Each Domain primitive can define explicit extension points, e.g.:

```text
BeforeCreateOrder
AfterCreateOrder
BeforePostInvoice
AfterPostInvoice
```

**Design‑time**

* For each extension point, add a config object in domain config (stored in the domain’s persistence):

```protobuf
message DomainExtensionConfig {
  string extension_id = 1;  // optional
  string version      = 2;
}
```

* Transformation Domain primitive exposes APIs & UIs to manage these configs per tenant.

**Runtime**

At a hook:

1. Domain primitive builds a small JSON view of the current state (e.g. order draft).
2. Builds `ExecutionContext` from the current call:

   * `tenant_id`, `org_id`, `user_id` from JWT / session. 
3. Calls `InvokeExtension(...)`.
4. Interprets the result (e.g. `ok` / `validationMessages` / `derivedFields`) and proceeds or fails the command.

This gives a **Chain‑of‑Command‑like experience** without loading tenant code into the domain process.

### 4.3 Agent tools

An AgentDefinition may reference an extension as a **deterministic tool**, e.g.:

* “Calculate risk score for an opportunity.”
* “Enforce policy for discount suggestion.”

Agent primitive:

1. Calls `InvokeExtension(...)` with an `ExecutionContext` representing the user or agent. 
2. Uses the result as deterministic input to the LLM chain (logged, auditable, replayable).

---

## 5. SAP & Dynamics Analogies

### 5.1 SAP

* **SAP in‑app extensibility / BAdIs**:

  * Small code snippets at pre‑defined hook points inside S/4HANA.
  * **Analogy:** Ameide `process_hook` & `domain_hook` ExtensionDefinitions.
* **SAP side‑by‑side (BTP)**:

  * Full apps running separately, calling S/4 via APIs/events.
  * **Analogy:** Ameide Tier 2 primitives in tenant namespaces (478).

So:

* Tier 1 WASM ≈ **SAP in‑app extension** equivalents, but sandboxed and controlled via Transformation.
* Tier 2 primitives ≈ **SAP BTP side‑by‑side apps**.

### 5.2 Dynamics 365 X++ Chain of Command (CoC)

* CoC = wrap a method with pre/`next()`/post logic inside the same app.
* **Analogy:** Ameide `domain_hook` extensions:

  * Instead of overriding methods in‑process, Domain primitives call a WASM hook before/after key operations.
* Difference:

  * Dynamics runs extensions inside the same CLR sandbox as MS code.
  * Ameide runs extensions in a **separate WASM runtime** with narrow host APIs and strong auth/tenancy controls.

---

## 6. Security & Governance

Tier 1 must fit into 476’s security & trust principles and 478’s extension invariants, with a carefully defined exception for WASM running in `ameide-*`.  

### 6.1 Threat model (incremental risk)

New risks introduced by WASM:

1. **Privilege escalation / data overreach** – extension tries to access data from other tenants/orgs.
2. **Data exfiltration** – extension attempts to leak data via host calls.
3. **Resource abuse** – expensive computations starving the runtime.
4. **Supply chain** – tampered modules or toolchains.
5. **Agent mis‑use** – agent generates unsafe or non‑compliant logic.

### 6.2 Invariants and carve‑outs

From 478:

> Custom code never runs in shared `ameide-*` namespaces; custom services run in `tenant-{id}-{env}-cust` namespaces. 

We **preserve this for Tier 2 primitives**. For Tier 1 we introduce a **narrow carve‑out**:

> **WASM modules are treated as data executed by a sandboxed, platform‑owned runtime in `ameide-{env}`, not as independently deployed services.**

Invariants:

1. **No tenant services in `ameide-*`**

   * Only the **WasmExtensionRuntime** runs there, fully platform‑owned.

2. **WASM sandbox**

   * No direct network, filesystem, or environment access.
   * Limited instruction count / fuel, memory, and wall‑clock time per invocation.

3. **No raw credential access**

   * WASM modules see only:

     * Input JSON,
     * Host functions (strictly controlled),
     * No tokens, secrets, or DB handles.

4. **All side‑effects via existing APIs + user/tenant context**

   * Host calls back into Ameide must:

     * Use existing proto/SDK APIs. 
     * Use the **same context** (tenant, org, user/agent) as the calling primitive.

5. **Deterministic boundary**

   * Primitives remain the sole owners of durable state & authorization; WASM is not allowed to write directly to storage.

### 6.3 Host calls, API, and user context

Inside WASM we expose:

```ts
// conceptual host API
declare function host_log(level: string, msg: string): void;

declare function host_call(
  service: string,
  method: string,
  requestJson: string
): string; // throws on error
```

**Implementation details (in the runtime, not in WASM):**

1. `ExecutionContext` is passed from the primitive to the runtime.
2. When WASM calls `host_call(...)`, the runtime:

   * Checks **allowlists** per `extension_id` and `risk_tier`:

     * `LOW`: no host calls.
     * `MEDIUM`: can call certain read‑only APIs.
     * `HIGH`: may call a wider set, but only under strict governance.
   * Builds a **short‑lived internal token** that encodes:

     * `tenant_id`, `org_id`, `user_id`, `actor_type`, scopes.
   * Calls the target Domain/Process/Agent primitive via the existing SDKs. 
3. Domain/Process/Agent primitives perform **normal auth & RLS**:

   * If context lacks rights, the call fails.

**WASM never sees or creates tokens itself**, and cannot change `tenant_id` / `org_id` / `user_id`.

### 6.4 Risk tiers & promotion rules

* `low` – pure functions (mapping, formatting, simple scoring):

  * No host calls.
  * Auto‑promotion possible for dev with tests.

* `medium` – read‑only enrichment, validations:

  * Limited host calls.
  * Requires human review for staging/prod.

* `high` – pricing, approvals, compliance:

  * Host calls allowed to a curated subset of APIs.
  * Requires multi‑party approval and stronger testing before prod.

Risk tier is a property of the ExtensionDefinition; Transformation Domain primitive enforces **environment‑specific promotion rules**, similar to how ProcessDefinitions/AgentDefinitions are governed. 

### 6.5 Observability & incident response

* Each invocation produces:

  * `tenant_id`, `org_id`, `user_id`, `actor_type`,
  * `extension_id`, version,
  * Hook kind (`process`, `domain`, `agent`),
  * Latency, resource usage,
  * Outcome (OK, business error, technical error).
* SRE:

  * Per‑extension SLOs and error budgets.
  * Circuit breakers to disable or roll back a specific extension version if it misbehaves.

---

## 7. Lifecycle of an Agent‑Developed Extension

This section focuses on the **Tier 1 path** for an extension primarily written by an agent, with humans in the loop.

### 7.1 Actors

* **Tenant stakeholder** – expresses requirement in natural language.
* **Transformation Agent primitive** – helps design the extension and wiring. 
* **Transformation Domain primitive** – stores ExtensionDefinitions, governs promotion. 
* **Compile Worker** – deterministic service that compiles source → WASM and runs validations.
* **WasmExtensionRuntime** – executes modules at runtime.
* **Process/Domain/Agent primitives** – call extensions through the runtime.

### 7.2 Stages

#### Stage 0 – Requirement capture

Example requirement:

> “For ACME, deals over 50k with low margin must go to an extra approval step, and we need a custom risk score combining margin and churn risk.”

Captured as:

* Backlog item and/or initiative in Transformation domain (Change‑the‑business process). 

#### Stage 1 – Design in Transformation

The Transformation agent + humans:

1. Identify the hook points:

   * BPMN: insert Extension Task in L2O ProcessDefinition before approval. 
   * Domain: optional `BeforeCreateOrder` hook.
2. Create an `ExtensionDefinition` draft:

   * `kind = "process_hook"`,
   * Input/output JSON schema,
   * Initial risk tier (likely `medium` or `high`),
   * Description.

All stored in Transformation Domain primitive as draft revision.

#### Stage 2 – Agent writes source code

The agent uses its toolset to produce a minimal function in an allowed language (Rust, AssemblyScript, TinyGo) with the `run(input_json) -> output_json` ABI.

It attaches this **source code** to the `ExtensionDefinition` draft via Transformation APIs (not direct Git).

#### Stage 3 – Compile & validate

Compile Worker:

1. Fetches the draft.
2. Compiles source → WASM using a pinned toolchain.
3. Runs:

   * ABI checks (exported `run` function),
   * Static policy checks,
   * Unit tests with synthetic + example inputs.
  4. If successful:

     * Stores the WASM blob in a tenant-scoped MinIO bucket/prefix (`s3://extensions/<tenant>/<extension>/<version>.wasm`).
     * Links `wasm_blob_ref` + new version to the ExtensionDefinition.

On failure, the draft remains in a failed state with detailed compile/test errors.

#### Stage 4 – Review & promotion

Depending on `risk_tier`:

* `low`:

  * Auto‑promote to `dev` environment after compile/tests.
* `medium`/`high`:

  * Human reviewers (process owner, architect) inspect:

    * Spec & code summary,
    * Test coverage & sample outputs,
    * Risk assessment.
  * Approve or request changes.

Promotion steps:

* `draft → in_review → promoted_dev → promoted_staging → promoted_prod`

Each promotion updates the runtime configuration for that env.

#### Stage 5 – Deploy to runtime

On promotion to an env:

1. Transformation emits `ExtensionVersionPromoted(...)` event.
2. Platform GitOps pipeline updates WasmExtensionRuntime Helm values for that env:

   * Known extensions (IDs/versions),
   * Resource limits,
   * Risk tier & host‑call policies.
3. ArgoCD syncs; runtime config reloads.

No new primitives/services are deployed; the runtime simply becomes aware of the new extension/version.

#### Stage 6 – Invocation in business flow

* **Process**: L2O Process primitive hits the Extension Task, calls runtime, uses result to drive routing/approval.
* **Domain**: OrdersDomain primitive `BeforeCreateOrder` calls runtime to run custom validations.
* **Agent**: L2O Agent primitive calls runtime to compute a deterministic risk score.

Everything is observable and tied back to:

* `tenant_id`, `org_id`, `user_id`,
* `extension_id`, version,
* Process/domain/agent context.

#### Stage 7 – Evolution & rollback

To change behaviour:

1. Agent/human creates new revision of ExtensionDefinition with updated source.
2. Repeat compile → review → promotion.
3. Older versions remain available; config supports:

   * rolling back per env, and/or
   * pinning a specific process to a specific version if needed.

---

## 8. Tier 2 Primitives: Confirmed Status

Tier 2 remains exactly as specified in 478:

* **Namespace model**:

  * `ameide-{env}` – shared product plane.
  * `tenant-{id}-{env}-base` – dedicated product plane for Namespace/Private.
  * `tenant-{id}-{env}-cust` – custom code for all SKUs. 
* **Repositories**:

  * Platform code in `ameide-platform` + `ameide-gitops`.
  * Tenant code in `tenant-{id}-controllers` + `tenant-{id}-gitops`. 
* **Backstage templates** and `PrimitiveImplementationDraft` remain the path for new primitives and services. 

Tier 1 (this doc) is additive and does **not** change any Tier 2 rules.

---

## 9. Implementation Backlog (High‑level)

Short version of the work needed to make Tier 1 real.

### Epic EXT‑WASM‑01 – Core runtime & artifact model

* Define `ExtensionDefinition` in Transformation protos + persistence.
* Implement Compile Worker (source → WASM, tests, validations).
* Implement WasmExtensionRuntime service (shared chart, RPC interface, sandbox, host API).
* Wire extension metrics into existing observability stack.

### Epic EXT‑WASM‑02 – Hooking into processes & domains

* Extend BPMN/Transformation design tooling modeller with “Extension Task” type.
* Teach Process primitive to invoke runtime at Extension Tasks.
* Add extension point pattern to at least one reference Domain primitive (e.g. Orders).
* Provide SDK helpers for primitives to call WasmExtensionRuntime with ExecutionContext.

### Epic EXT‑WASM‑03 – Agent lifecycle

* Transformation APIs for managing ExtensionDefinition drafts, source upload, promotion.
* Agent tools for:

  * Proposing extension specs,
  * Generating code,
  * Triggering compile & tests,
  * Requesting promotion.
* Basic UI for human review & approval.

### Epic EXT‑WASM‑04 – Security & governance

* Implement sandboxing and host‑call policy enforcement.
* Integrate risk tiers into Transformation governance flows.
* Add guardrails for agent behaviour (max risk tier per agent, env limitations).
* Define SRE playbooks & circuit breakers for extensions.

### Epic EXT‑WASM‑05 – SAP/Dynamics parity examples

* Create reference examples:

  * SAP “in‑app”‑style pricing BAdI mapped to process/domain hooks.
  * Dynamics CoC “before/after validate” mapped to a domain hook.
* Document these for sales/solution teams.

---

## 10. Performance Envelope & Guardrails

Tier 1 will only feel “slow” if we design it poorly. This section captures the latency expectations and the rules that keep the runtime snappy.

### 10.1 What the extra hop costs

**Primitive → WasmExtensionRuntime → WASM module** adds the following work:

* Proto marshal/unmarshal – microseconds on each side.
* In‑cluster gRPC hop over warm HTTP/2 connections – typically **1–5 ms p50–p95**, occasionally ~10 ms in a noisy cluster.
* WASM function execution for a “small rule” – microseconds to sub‑millisecond.
* JSON encode/decode inside the runtime – microseconds to sub‑millisecond.

So the incremental cost of an extension call is usually **~1–5 ms**, which sits in the same ballpark as any other intra‑cluster primitive/domain call. For contrast:

* Domain primitive → Postgres + RLS + index scan: **5–30 ms**.
* Process primitive → Domain primitive + Temporal activity hop: **dozens of ms**.

### 10.2 Why that cost is usually acceptable

* Process primitives are **Temporal workflows** that complete over seconds to hours; inserting a 5 ms policy hook at a stage boundary is negligible compared to human latency.
* Domain primitives already pay DB and network costs. Budgeting **50–150 ms** for a domain command leaves plenty of headroom for one extension call, especially if the WASM result saves follow‑up hops.
* The platform prioritises **clarity, determinism, and tenant isolation** (“Design ≠ Deploy ≠ Runtime”) over shaving every millisecond, so the shared runtime hop is an intentional, governable tradeoff.

### 10.3 When it will slow things down

We still need to treat Tier 1 like an RPC hop, not an in‑process helper. Problems arise when:

1. **Hot, chatty UI validations** – calling an extension per keystroke or per line item causes dozens of gRPC → WASM → host round trips, dragging down perceived latency.
2. **Loops over large collections** – iterating 500 invoice lines and invoking the runtime for each one burns ~2.5 s just on extension hops, before any real work is done.
3. **Chatty host calls from inside WASM** – making sequential `host_call`s (“get customer”, “get last 10 orders”, “get discounts”) multiplies DB and network latency and defeats the purpose of keeping logic small.

### 10.4 Design rules that keep it fast

1. **Coarse‑grained contracts** – primitives gather all required inputs (full order header, lines, risk flags, etc.) and call the runtime **once per business decision**, not per field or row.
2. **Minimise host calls** – defaults: `low` tier = none, `medium` = 0–1 read‑only call, `high` = tightly curated list for approvals/pricing. Let primitives orchestrate IO; WASM stays mostly pure logic.
3. **Cache and preload modules** – WasmExtensionRuntime keeps modules, host connections, and worker pools warm so only the gRPC hop and function execution remain on the hot path.
4. **Co‑locate when needed** – latency‑sensitive primitives can run near the runtime (node affinity or Unix Domain Socket mode) without changing the security model.
5. **Promote hot paths to Tier 2** – if a “custom rule” grows into logic that runs per keystroke, needs multiple DB calls, or demands independent scaling/SLOs, it should graduate into a dedicated primitive (Tier 2) instead of stressing Tier 1.

We will back these rules with linting, modelling guardrails, and latency histograms per `extension_id` so teams see when an extension is drifting into “chatty helper” territory.

---

That’s the full rewritten specification with your updated assumptions:

* **Tier 2**: tenant-scope namespaces and Backstage primitive model from 478 remain untouched.
* **Tier 1**: a shared platform WASM runtime in `ameide-{env}` for small extensions, wired into processes, domains, and agents, with the latency guardrails captured in §10.
* **Host calls** from WASM always go through existing APIs using the same user/tenant context and authorization model as the rest of the platform.

---

## 11. Cross-References

| Document | Relationship |
|----------|-------------|
| [472-ameide-information-application.md](472-ameide-information-application.md) | Defines `ExtensionDefinition` as a first-class artifact (§2.5) |
| [473-ameide-technology.md](473-ameide-technology.md) | Introduces `extensions-runtime` as a platform service (§3.3.5) |
| [476-ameide-security-trust.md](476-ameide-security-trust.md) | Captures the sandbox carve-out and host-call policies (§6, §8.4) |
| [480-ameide-extensibility-wasm-service.md](480-ameide-extensibility-wasm-service.md) | Implements the shared runtime described in this doc |
