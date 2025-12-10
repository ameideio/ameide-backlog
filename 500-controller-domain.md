Here’s a full first cut of the **IDC** document you can drop in as something like
`48x-intelligent-domain-controller.md`.

---

# 48x – Intelligent Domain Controllers (IDC)

**Status:** Draft
**Owner:** Platform / Architecture
**Audience:** Platform engineers, domain teams, SRE, “architect agent” implementers
**Depends on:** 471 (Business), 472 (App/Info), 473 (Tech), 475 (Domains), 476 (Security & Trust), 478 (Extensions), 479–480 (WASM), 461 (IDC/IPC/IAC control plane)

---

## 1. Purpose

This document defines **Intelligent Domain Controllers (IDCs)** as the **declarative control‑plane representation** of Ameide domain services.

* In 472/475, a **DomainController** is the runtime service that owns data + rules for a bounded context (e.g. Orders, Product, Transformation). 
* In 461/473, an **IntelligentDomainController** is a **Kubernetes CRD** reconciled by an operator into concrete workloads (Deployments, DB schemas, events, etc.).

This spec explains:

* What an IDC **CRD** looks like.
* What the **IDC operator** must do.
* How IDCs interact with IPC/IAC, WASM Tier 1, security, multi‑tenancy and Backstage.
* How IDCs support **scaffolding + lifecycle at scale** (hundreds of services).

It describes the **target state only** (no migration plan).

---

## 2. IDC in the overall architecture

### 2.1 Relationship to DomainControllers

* **DomainController** (logical / runtime):

  * A stateless service + DB schema that owns aggregates, invariants, and domain events (e.g. OrdersDomainController). 
* **IntelligentDomainController** (IDC, control‑plane):

  * A **custom resource** (`kind: IntelligentDomainController`) that declares **how that DomainController should exist** (image, proto package, multi‑tenant DB, events, extensions, etc.). 

The **IDC operator** reconciles each IDC CR into:

* Deployments, Services, HPAs, ServiceMonitors.
* DB schema + users (via CNPG) and migrations Jobs.
* Kafka/NATS topics for domain events.
* NetworkPolicies, labels, standard sidecars. 

### 2.2 Relationship to IPC, IAC, UAF & Extensions

* **IPCs** (IntelligentProcessControllers) bind **BPMN tasks** to IDC **primitives** and IAC tools.
* **IACs** (IntelligentAgentControllers) encapsulate agent graphs + tools; they call IDCs via proto APIs and tool grants. 
* **UAF / Transformation**:

  * Owns **ProcessDefinitions** and **AgentDefinitions** as design‑time artifacts; IDCs are **runtime controllers** that those artifacts target.
* **Tier 1 WASM**:

  * IDCs may expose **extension hooks** (e.g. `BeforePostInvoice`) wired to `ExtensionDefinition`s executed by the shared `extensions-runtime` service.

---

## 3. IDC CRD: shape & responsibilities

### 3.1 High‑level responsibilities (from 461)

An `IntelligentDomainController` CRD **owns**: 

* **Data model anchor**

  * Reference to the domain model artifacts (proto package, optional logical model in UAF).
  * Guarantees `tenant_id` / `org_id` columns and multi‑tenant invariants.
* **Proto API contract**

  * One or more gRPC/Connect services (e.g. `OrdersService`) with methods marked as **primitives** consumable by IPCs.
* **Lifecycle & state machine**

  * Allowed states and transitions for key aggregates (e.g. Order: Draft → Confirmed → Fulfilled).
* **Event streams**

  * Topics and schemas for domain events (e.g. `OrderCreated`, `OrderConfirmed`).
* **Determinism contract**

  * IDC operations are deterministic and replay‑safe; no LLM calls in core decision paths. Agents may **propose**, IDC must validate.

### 3.2 CRD skeleton (conceptual)

Target API group: `ameide.io/v1`.

```yaml
apiVersion: ameide.io/v1
kind: IntelligentDomainController
metadata:
  name: orders
  namespace: ameide-prod
spec:
  displayName: "Orders Domain"
  sku:
    tier: shared | namespace | private
  runtime:
    image: ghcr.io/ameideio/orders-service:v1.3.0
    proto:
      module: buf.build/ameide/orders
      package: ameide.orders.v1
    resources:
      class: default-medium
  dataModel:
    # abstract, not a full table list
    primaryArtifact: "uaf://datamodel/orders.v1"   # optional UAF logical model
    protoPackages:
      - ameide.orders.v1
    multiTenancy:
      mode: rls
      tenantColumn: tenant_id
      orgColumn: organization_id
  lifecycle:
    aggregates:
      - name: Order
        idField: id
        stateField: status
        states: [DRAFT, CONFIRMED, FULFILLED, CANCELLED]
  events:
    topic: "orders.events"
    schemas:
      - OrderCreated
      - OrderConfirmed
      - OrderShipped
  extensions:
    wasmHooks:
      - hook: BeforeCreateOrder
        extensionId: "orders_custom_validation"
        version: "latest"
  security:
    allowedCallers:
      processes: ["l2o/*", "o2c/*"]
      agents: ["pricing-agent", "architect-agent"]
```

> Note: This replaces the earlier “full table list in YAML” example with **references** to proto/UAF artifacts as the canonical data model source, so we don’t have to describe hundreds of tables inline.

### 3.3 Status subresource

The operator maintains:

```yaml
status:
  phase: Ready | Progressing | Degraded
  conditions:
    - type: MigrationsApplied
      status: "True"
      reason: "Succeeded"
    - type: DeploymentAvailable
      status: "True"
      reason: "MinimumReplicasAvailable"
  endpoints:
    grpc: "orders.ameide-prod.svc.cluster.local:8080"
    http: "https://api.ameide.io/orders"
  metrics:
    p95LatencyMs: 35
    errorRate: 0.001
```

These fields allow GitOps, IPCs, IACs and UIs to **discover & monitor** domain controllers at the CRD level instead of scraping Deployments directly. 

---

## 4. IDC Operator

### 4.1 Responsibilities

The **IDC operator** reconciling `IntelligentDomainController` CRs must:

1. **Provision application workloads**

   * Deployment, Service, HPA, ServiceMonitor, PodDisruptionBudget.
   * Inject standard sidecars (OTel, log agent), env vars, labels.

2. **Provision data plane**

   * For each IDC:

     * Ensure CNPG cluster/schema and RLS policies.
     * Create DB roles for the service (no superuser).
     * Run migrations Jobs based on the referenced proto/UAF data model.

3. **Wire events**

   * Create / configure topics (Kafka/NATS) for domain events.
   * Ensure outbox → topic wiring (or equivalent) is enabled.

4. **Enforce SKU + namespace placement**

   * Use `spec.sku.tier` + tenant SKU rules (from 478) to place workloads into:

     * `ameide-{env}` (shared),
     * `tenant-{id}-{env}-base` (namespace/private).

5. **Apply security defaults**

   * NetworkPolicies, PodSecurityContext, RBAC, service accounts.
   * Ensure RLS is enabled for `tenant_id` and `organization_id` per 476/329. 

6. **Publish status & metrics**

   * Reflect rollout state into `status.conditions`.
   * Expose basic SLO metrics for p95 latency, error rate, and migrations.

7. **Own Gateway API exposure**

   * When an IDC publishes HTTP/GRPC endpoints, the operator must render the corresponding `HTTPRoute`/`GRPCRoute` resources in the IDC release (see 417/459). Hostnames, listeners, and oauth2-proxy chains live with the controller so the platform gateway only provides shared infrastructure—no more `extraHttpRoutes` patches. This keeps ownership clear and allows Argo health to reason about exposure status alongside the rest of the controller spec.

### 4.2 Interaction with GitOps & Backstage

* **GitOps**:

  * ArgoCD syncs **only the IDC CRD manifests + operator**.
  * All child Deployments/Services/Jobs are **owned by the operator**, so cross‑cutting changes (sidecars, policies) can be rolled out centrally. 
* **Backstage**:

  * A Backstage `Component` (type `domain-controller`) references the IDC: `spec.system`, `spec.api`, and a link to the IDC YAML.
  * The “Domain Controller” template produces:

    * proto skeleton and service repo,
    * IDC `kind: IntelligentDomainController` spec,
    * GitOps manifest that ArgoCD consumes. 

### 4.3 Metadata & status contract

All controller backlogs share a **common metadata contract** so operators, GitOps, and Backstage can reason about them consistently (see 461/447). For IDCs we standardize on:

| Field | Purpose |
|-------|---------|
| `metadata.labels.ameide.io/controller-tier=domain` | Lets policy agents, dashboards, and the Architect Agent filter for domain controllers only. |
| `metadata.labels.ameide.io/domain=<bounded-context>` | Matches the canonical entry in 475 so Backstage/catalog tooling can group controllers with their domain. |
| `metadata.labels.ameide.io/sku=<shared|namespace|private>` | Signals placement/SKU choice to the operator and ensures tenant variants stay inside `tenant-*` namespaces. |
| `metadata.annotations.argocd.argoproj.io/rollout-phase=350` | Aligns with 447 so domain controllers deploy after foundational operators but before app tiers; tenant-specific overrides can choose a later phase (e.g., 650) without changing the spec body. |

Status objects must expose at least `DeploymentAvailable`, `MigrationsApplied`, and `DataPlaneHealthy` conditions; ArgoCD health scripts key off these fields, and the Architect Agent reads them when proposing changes. This section applies to IAC/IPC specs as well, but IDCs anchor the contract by publishing the canonical label set other controllers reference.

To eliminate drift, this IDC backlog is the **canonical definition** of the controller metadata surface. IAC (501) and IPC (502) now link back to this table and inherit keys verbatim; the controller conformance tests called `controller-contract` fail any CRDs that omit the `ameide.io/*` labels or roll-out annotations defined here. Whenever we expand the contract (for example, new SKU labels or rollout annotations), we update _this_ section first and downstream docs simply reference it instead of redefining the list.

The required `status.conditions` set is equally strict because downstream tooling consumes the exact condition names:

| Condition | Description |
|-----------|-------------|
| `DeploymentAvailable` | Mirrors Kubernetes deployment health so GitOps and rollout automation can block promotions until pods are ready. |
| `MigrationsApplied` | Indicates schema + data reconciliation succeeded; IPC/IAC lint checks read this before allowing new bindings to target an IDC. |
| `DataPlaneHealthy` | Aggregates DB + eventing probes; Architect Agents treat a `False` value as a hard stop when proposing dependent changes. |

Additional conditions (e.g., `PrimitivesPublished`) are allowed, but the three above are mandatory, and they must always pair with the shared `status.phase` values (`Ready`, `Progressing`, `Degraded`). Controllers that conform to this contract automatically get ArgoCD health parity because the shared Lua check simply inspects the standardized condition list.

---

## 5. Information model & domain contracts

### 5.1 Data model references

Given each domain may have **hundreds of tables**, we **do not** try to encode full DDL in the IDC spec.

Instead, IDCs carry **pointers** to the authoritative models:

* `spec.dataModel.protoPackages` → Buf module + proto packages that define domain messages & service contracts. 
* `spec.dataModel.primaryArtifact` → optional UAF / Transformation artifact representing the logical data model (e.g. ER/ArchiMate/DSL). 

The operator can:

* Validate DB schema against proto types (critical fields, `tenant_id`, `organization_id`).
* Run migrations generated from the service repo or DDL artifacts, but **not** from the IDC spec itself.

### 5.2 Primitives & API surface

In 461, IDCs expose **primitives** — a subset of methods annotated for process binding:

* All methods are defined in proto (`OrdersService`).
* A subset is tagged with `primitive: true` in Buf/annotations.
* IPCs bind BPMN service tasks to these primitives.

Example (conceptual):

```protobuf
service OrdersService {
  rpc CreateOrder(CreateOrderRequest) returns (Order) {
    option (ameide.primitive) = true;
  }
  rpc ConfirmOrder(ConfirmOrderRequest) returns (Order) {
    option (ameide.primitive) = true;
  }
  rpc ListOrders(ListOrdersRequest) returns (ListOrdersResponse) {
    // not a primitive; used only by UIs/agents
  }
}
```

The IDC spec lists these primitives so the IPC operator can validate binding correctness.

### 5.3 Lifecycle & state machines

IDCs declare **aggregate lifecycles** (states & transitions). 

* Used by:

  * AgentControllers (reasoning about valid transitions).
  * UIs (guarding buttons / actions).
  * Validation rules in DomainControllers.

Examples:

* `Order: DRAFT → CONFIRMED → FULFILLED | CANCELLED`
* `Invoice: DRAFT → POSTED → SETTLED | WRITEOFF`

The operator doesn’t enforce these transitions directly, but may:

* Generate enum types & helper code.
* Expose them via metadata for agents.

### 5.4 Events

Events are declared in the IDC spec, but implemented in code:

* Topic name and event types (`OrderCreated`, `OrderConfirmed`, `OrderShipped`,…).
* Operator ensures topics exist and that the service has credentials for publishing.

---

## 6. Multi‑tenancy, security & isolation

IDCs embody the **“tenant isolation first”** principle from 476: 

* Every domain table must carry `tenant_id` (and typically `organization_id`).
* Application‑level filtering is **not enough**; RLS is mandatory at DB level.
* JWT claims (`tenantId`, `orgId`) propagate into IDC services, enforced by shared interceptors.

The IDC operator must:

1. Ensure RLS policies exist for all tables belonging to the domain schema.
2. Enforce per‑SKU naming/placement so custom controllers run in `tenant-{id}-{env}-cust` and never in shared `ameide-*` namespaces (per 478).
3. Attach NetworkPolicies so tenant custom controllers cannot bypass API layers to talk directly to platform DBs.

---

## 7. Lifecycle & upgradability at scale

This is the bit you care about for “**hundreds of services**”: scaffolding is only step 0; we need **continuous change**.

### 7.1 Change vectors

IDCs have four main change types:

1. **Config‑only** (resources, autoscaling, env vars).
2. **Data model evolution** (new fields, tables, indexes).
3. **API surface evolution** (new primitives, deprecations).
4. **Infra / cross‑cutting** (sidecars, tracing, security baselines).

All of these are expressed as **Git changes to IDC CRs** or linked artifacts, and reconciled by the operator.

### 7.2 Versioning & rollout

* Each IDC spec is **immutable per commit**; Git + Argo provide history.
* Operators support **phased rollout** per 447 (e.g. rollout‑phase 350 for platform controllers). 
* For breaking changes (schema or API):

  * Use **shadow deployments** and dual traffic (e.g. v1 + v2) governed via IDC fields or operator annotations.
  * IPC bindings can declare minimum IDC versions if necessary.

### 7.3 Architect Agent interaction

The **Architect Agent** (IAC) uses the IDC abstraction as its “Lego bricks”:

* Reads existing IDC specs via **tools** (`read-domain-specs`). 
* Proposes changes or new IDCs (`propose-idc-change`, `generate-idc`) and opens PRs via `core-platform-coder`.
* Human/policy gates decide whether to merge; operators roll out.

This is what turns “hundreds of services” into **config management**, not manual Helm hacking.

---

## 8. Interaction with IPC, IAC, UI, WASM

* **IPC**:

  * IPC specs reference IDC **names + primitives** in their `bindings.serviceTasks`.
  * IPC operator validates those references against IDC CRs before compiling BPMN → Temporal.

* **IAC**:

  * IAC specs declare tools that target IDCs (read/query/proposal).
  * IDC operator publishes an “introspection” endpoint so agents know which aggregates & primitives exist.

* **UI**:

  * Next.js workspaces call DomainControllers via Ameide SDK. They **do not** call the IDC operator or CRDs directly.
  * Backstage uses IDC CRD metadata to populate its catalog (owner, system, links).

* **WASM Tier 1**:

  * IDC’s `extensions.wasmHooks` list domain hooks wired to `ExtensionDefinitions`.
  * At runtime the DomainController calls `extensions-runtime` with execution context; the IDC spec is the single source of truth for which hooks are active.

---

## 9. Worked example: Orders IDC

A more complete example aligned with 461 but updated to “abstract data model”:

```yaml
apiVersion: ameide.io/v1
kind: IntelligentDomainController
metadata:
  name: orders
  namespace: ameide-prod
spec:
  displayName: "Orders Domain"
  sku:
    tier: shared
  runtime:
    image: ghcr.io/ameideio/orders-service:v1.3.0
    proto:
      module: buf.build/ameide/orders
      package: ameide.orders.v1
    resources:
      class: default-medium
  dataModel:
    primaryArtifact: "uaf://datamodel/orders.v1"
    protoPackages:
      - ameide.orders.v1
    multiTenancy:
      mode: rls
      tenantColumn: tenant_id
      orgColumn: organization_id
  lifecycle:
    aggregates:
      - name: Order
        idField: id
        stateField: status
        states: [DRAFT, CONFIRMED, FULFILLED, CANCELLED]
  events:
    topic: "orders.events"
    schemas:
      - OrderCreated
      - OrderConfirmed
      - OrderShipped
  primitives:
    services:
      - name: OrdersService
        methods:
          - CreateOrder
          - ConfirmOrder
  extensions:
    wasmHooks:
      - hook: BeforeCreateOrder
        extensionId: "orders_custom_validation"
        version: "latest"
  security:
    allowedCallers:
      processes: ["l2o/*", "o2c/*"]
      agents: ["pricing-agent", "architect-agent"]
```

The operator then:

* Provisions the `orders` Deployment + DB schema (with RLS).
* Creates `orders.events` topic.
* Ensures `OrdersService` is reachable at a stable DNS name and registered in the catalog.

This same pattern is repeated for **every domain** in 475 (Sales, Product, Pricing, Billing, Transformation, etc.).

---

## 10. Open questions

For future refinement:

1. **How strict should data‑model checks be?**

   * Validate only `tenant_id`/`org_id` + a few invariants?
   * Or fully diff DB schema vs proto/UAF model?

2. **State machine enforcement**

   * Operator‑enforced hooks (e.g. generating interceptors) vs pure doc/metadata?

3. **Per‑tenant IDC variants**

   * Do we allow tenant‑specific IDC CRs (`orders-tenant-123`), or do all tenant variants live in Tier 2 controllers?

4. **Standard IDC labels/annotations**

   * We should formalize a label set (`ameide.io/domain`, `ameide.io/controller-tier`, `ameide.io/sku`) to keep dashboards and policies consistent.
