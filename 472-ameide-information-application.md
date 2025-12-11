Here's a first cut of the **Application & Information Architecture** doc, aligned with the vision + business docs we already sketched and wired to vendor concepts (Backstage, BPMN-compliant definitions via custom React Flow modeller, Temporal, K8s, Buf, etc.).

---

# 3 – Application & Information Architecture (Draft)

**Audience:** Platform & domain engineers, solution architects, "platform-facing" agents
**Scope:** How Ameide is structured as software (domains, processes, agents, UI) and how information flows and is stored across the platform.

> **Core Invariants**: See [470-ameide-vision.md §0 "Ameide Core Invariants"](470-ameide-vision.md) for the canonical list (four primitives, Graph read-only, Transformation as domain, proto chain, tenant isolation, Backstage internal).

---

## Implementation Status (2025-02-14)

- ✅ The Domain operator handles CNPG secrets and migration jobs so runtime CRDs truly manage persistence concerns (`operators/domain-operator/internal/controller/reconcile_db.go:36-158`).
- ✅ Graph write paths are now blocked, keeping the projection layer aligned with this document’s read-only assumption (`services/graph/src/graph/service.ts:118-135`, `services/graph/src/graph/service.ts:4879-4905`).
- ⚠️ ProcessDefinitions/AgentDefinitions continue to live in the workflows runtime instead of Transformation, and the Process operator does not fetch/checksum definitions yet (`services/workflows/src/repositories/definitions-graph.ts:1-80`, `operators/process-operator/internal/controller/process_controller.go:54-149`).

## 1. Objectives & Constraints

### 1.1 What this document does

* Defines the **core application building blocks**:

  * Domain primitives
  * Process primitives
  * Agent primitives
  * UISurfaces / process views
* Describes how **information is modeled**:

  * Per-domain SQL stores
  * Process definitions (BPMN) stored in **Transformation Domain**
  * Transformation artifacts (managed via Transformation design tooling UIs, stored by Transformation Domain)
  * Cross-domain knowledge graph (read-only projection)
* Shows how **Backstage** is used as the “declarative front-door” to design domains/processes/agents.
* Aligns with:

  * Proto-based APIs & SDK strategy
  * Unified Artifact Framework for transformation artifacts
  * Temporal-backed Process primitive execution and SaaS-like ops model.

### 1.2 Constraints & design principles (from previous docs)

We carry forward the earlier principles and make them concrete here:

1. **Process-first**
   *Primary abstraction is an end-to-end process (L2O, O2C, T2C…), not a module (Sales, Procurement). Manual workspaces are second-class views over these processes.*

2. **Universal DDD**
   *All behavior lives in bounded contexts (domains), each with its own model, storage and APIs.*

3. **Transformation as a domain**
   *Transformation (requirements, Transformation design tooling artifacts, governance) is modeled like O2C: domain + processes + agents, not a side-console.*

4. **Agentic from any angle**
   *Agents can read from knowledge, call domains, and be invoked from processes in a controlled, typed way.*

5. **Proto-first contracts**
   *Every domain/process/agent exposes a proto-based API, with Buf/BSR governance and SDKs.*

6. **Kubernetes-style declarative state**
   *We declare desired state (e.g. “invoice posted”) and primitives + workflows converge actual state, similar to K8s reconciliation loops and CRDs.*

---

## 2. Core Application Building Blocks

### 2.1 Domain primitives (Domain layer)

**Responsibility:** Encapsulate business concepts, invariants and persistence for a single bounded context.

*Examples:*

* *Sales* domain (Leads, Opportunities, Quotes)
* *Billing* domain (Invoices, Payments, Dunning)
* *Transformation* domain (Initiatives, Backlog Items, Transformation design tooling artifacts)

**Characteristics**

* **Deterministic, low-entropy behavior**

  * Pure business rules, minimal AI inside the domain (agents live next to it).
* **Own their data**

  * Dedicated logical schemas / DBs (multi-tenant) and migrations.
* **Expose proto-based APIs**

  * CRUD + business methods encoded in `ameide.<domain>.v1` protos, versioned by Buf.
* **Emit domain events**

  * Changes are published to a streaming layer (K8s events or Kafka, detail in tech doc).

**Information model**

* **Authoritative** for its aggregate roots (Customer, Opportunity, Invoice, Initiative, etc.).
* **Tenant-aware** – every record tagged with `tenant_id`.
* **Projectable** – each domain can opt-in to project parts of its model into the cross-domain Graph for analysis and agent reasoning (see §3.4).
* **Runtime representation** – each Domain primitive exists as a `Domain` CRD. The Domain operator reconciles the CRD into Deployments, Services, HPAs, DB schemas, and ServiceMonitors, enforcing standard Ameide policies.

### 2.2 Process primitives (Process layer)

**Responsibility:** Orchestrate **cross-domain** flows such as L2O, O2C, Procure-to-Pay, or Transformation workflows (Scrum/Togaf ADM).

*Design-time:* **ProcessDefinitions** are BPMN-compliant artifacts produced by a **custom React Flow modeller** and stored in the **Transformation Domain** (modelled via Transformation design tooling UIs). At runtime they are executed by **Process primitives** backed by Temporal workflows.

**Key concepts**

* **ProcessDefinition** (design-time, stored in the Transformation Domain)

  * BPMN-compliant artifact + metadata (tenant, version, process type like L2O/O2C/T2C).
  * Defines process stages, tasks, gateways, and bindings to domains/agents.
  * Stored and versioned in the **Transformation Domain** with revisions & promotions (modelled via Transformation design tooling UIs).
* **Process primitive** (runtime)

  * Logical unit that "executes" a ProcessDefinition:

    * Loads ProcessDefinition from the Transformation Domain.
    * Maps BPMN tasks to Domain primitive API calls and/or Agent primitive tools.
    * Handles business-level errors and compensations.
  * Backed by Temporal workflows.
* **Execution**

  * Long-running process instances in Temporal representing specific business cases.

**Why BPMN-compliant**

* Standard semantics for business users (stages, gateways, lanes).
* Custom React Flow modeller provides visual editing; definitions are BPMN-compliant but NOT Camunda-stack.

**Information model**

* **Process instance state** (per tenant):

  * Process variables (e.g. `leadId`, `quoteId`, `orderId`)
  * Execution state (activity id, token position)
  * Audit trail (who approved, when)
* Stored in **Temporal** plus Ameide projections (for consolidated reporting).
* **Runtime representation** – Process primitives are declared via `Process` CRDs that reference ProcessDefinition versions, Temporal namespaces, rollout policies, and dependent Domain/Agent primitives. The Process operator reconciles those CRDs into worker Deployments, Services, and monitoring assets.

### 2.3 Agents (Agent layer)

**Responsibility:** Non-deterministic, AI-powered components that:

* Read and summarize data (domain stores, graph, logs)
* Propose changes (requirements, configurations, data corrections)
* Generate new domain/process/agent definitions via Backstage templates

*Design-time:* **AgentDefinitions** are declarative specs stored in the **Transformation Domain** (modelled via Transformation design tooling UIs). At runtime they are executed by **Agent primitives**.

**Key concepts**

* **AgentDefinition** (design-time, stored in Transformation Domain)

  * Declarative spec for an agent: tools, orchestration graph, scope, risk tier, policies.
  * Stored alongside ProcessDefinitions in the Transformation Domain.
  * Subject to governance (review, promotion) before runtime use.
* **Agent primitive** (runtime)

  * Loads an AgentDefinition from the Transformation Domain and runs the LLM/tool loop.
  * Enforces scope/risk policies at execution time.
  * Always invoked via domain/process APIs; never becomes source of truth.

**Characteristics**

* **Explicit tool contracts** – each AgentDefinition specifies:

  * Which domain/process APIs the Agent primitive can call.
  * What Transformation design tooling / transformation artifacts it can read or modify.
* **Runtime**

  * LangGraph / OpenAI / other inference runtimes orchestrate tools via the Agent primitive.
* **Determinism boundary**

  * Agent primitives never become the authoritative "source of truth" for domain data; they make proposals or trigger domain commands.

**Information model**

* **AgentDefinitions** (stored per tenant in Transformation Domain, modelled via Transformation design tooling UIs):

  * Prompt & behavior description
  * Allowed tools & scopes (e.g. "Transformation agent can only touch transformation artifacts and Backstage templates")
  * Risk tier and policy bindings
* **Interaction history**

  * Persisted in Chat/Threads services for audit and context.
* **Runtime representation** – Agent primitives are instantiated from `Agent` CRDs that reference AgentDefinitions, risk tier, runtime chassis, and tool grants. The Agent operator manages Deployments, Secrets, and observability for each agent runtime.

### 2.4 UISurfaces & process views

We keep two complementary UI modes:

1. **Traditional “ERP-style” workspaces**

   * List/detail pages over domain entities (Customers, Opportunities, Orders, Invoices).
   * Implemented as microfrontends per domain, using the TS SDK to call proto-based APIs.

2. **Process views** (process-first principle)

   * A timeline / swimlane view of L2O/O2C/T2C, driven by process instance state.
   * Users see “where they are” in L2O, not “which form they’re on”.

Information-wise:

* UI does **not** own data; it binds:

  * Domain state → Domain primitives (via SDKs)
  * Process state → Process primitives (via SDKs)
  * Knowledge/analytics → Graph and Transformation design tooling projections

### 2.5 Extensions (Tier 1 WASM)

*Design-time*: `ExtensionDefinition` artifacts are stored in the Transformation Domain alongside ProcessDefinitions and AgentDefinitions via Transformation design tooling UIs.

*Runtime*: A shared `extensions-runtime` service in `ameide-{env}` executes WebAssembly modules for three kinds of hooks:

* `process_hook` – BPMN Extension Tasks in ProcessDefinitions.
* `domain_hook` – explicit extension points in Domain primitives.
* `agent_tool` – deterministic helpers for Agent primitives.

Extensions are sandboxed, multi-tenant, and never own durable state; all data access goes back through domain/process/agent APIs with the same tenant/org/user context as the caller. See [479-ameide-extensibility-wasm.md](479-ameide-extensibility-wasm.md) and [480-ameide-extensibility-wasm-service.md](480-ameide-extensibility-wasm-service.md).

### 2.6 Primitive CRDs & operators

At runtime Ameide treats every Domain/Process/Agent/UISurface primitive as a declarative Kubernetes custom resource:

* `Domain` CRD – domain runtime desired state (image, config, DB bindings, resources)
* `Process` CRD – process runtime desired state (image, Temporal namespace/bindings, rollout policy)
* `Agent` CRD – agent runtime desired state (runtime chassis, model/provider config, tool grants, risk tier)
* `UISurface` CRD – UI runtime desired state (image, routing, auth scopes, feature flags)

ArgoCD applies these CRDs checked into Git; dedicated Ameide operators reconcile them into Deployments, Services, HPAs, Temporal workers, CNPG schemas, ServiceMonitors, and other low-level objects. This keeps Git as the single source of truth while giving SREs a first-class object model for primitives. See [474-ameide-refactoring.md](474-ameide-refactoring.md) for how the platform converges on this primitive + CRD model.

### 2.7 Contract → Implementation → Runtime chain

Every primitive travels the same three-layer chain. We make the links explicit, versioned, and machine-checkable so that a change in the proto contract propagates deterministically into Go code and ultimately into the GitOps manifests that Argo CD applies.

#### Proto (contract)

* Protos live in dedicated Buf modules per bounded context under `packages/ameide_core_proto/` (mirrored into Go modules such as `github.com/ameide/apis/domains/sales`).
* CI runs Buf breaking-change checks against the main branch before publishing a new module tag (e.g. `apis/domains/sales v1.5.0`).[^buf]
* Generated Go (and TS/Python) SDKs are published as versioned modules so downstream services can pin to an exact contract.

#### Implementation (Go service)

* Each Domain/Process/Agent implementation is its own Go module (or sub-module) that **imports the generated proto module** in `go.mod`:

  ```go
  module github.com/ameide/sales-domain

  require github.com/ameide/apis/domains/sales v1.5.0
  ```

* CI enforces:

  * `go mod tidy` cleanliness (no transitive drift).
  * Tests against the imported proto version.
  * Docker images tagged with `git sha` + semantic version (e.g. `ghcr.io/ameide/sales-domain:1.12.3`).
  * Optional OCI labels capturing proto module + version:

    ```bash
    docker build \
      --label 'ameide.io/proto-module=github.com/ameide/apis/domains/sales' \
      --label 'ameide.io/proto-version=v1.5.0' \
      -t ghcr.io/ameide/sales-domain:1.12.3 .
    ```

#### Runtime (GitOps + Argo CD)

* GitOps manifests in the platform repo (and tenant repos) reference the exact image tag and annotate the relationship:

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: sales-domain
    annotations:
      ameide.io/proto-module: github.com/ameide/apis/domains/sales
      ameide.io/proto-version: v1.5.0
      ameide.io/image-digest: sha256:…
  spec:
    template:
      spec:
        containers:
        - name: sales-domain
          image: ghcr.io/ameide/sales-domain:1.12.3
  ```

* Argo CD best practices already recommend separating application source repos from environment/GitOps repos; we follow that model so manifests reflect the latest *published* image, not whatever is on a developer branch.[^argo]
* Rolling out a new proto or service version therefore becomes:

  1. Merge proto PR → Buf module tag.
  2. Update Go module import + release image.
  3. Update GitOps manifest (fast-forward PR) to new image tag.

#### Service descriptor & catalog cross-link

To make the chain easy to query, each primitive carries a Backstage `catalog-info.yaml` (or `service.yaml`) describing:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: sales-domain
  annotations:
    ameide.io/proto-module: github.com/ameide/apis/domains/sales
    ameide.io/proto-version: v1.5.0
    ameide.io/gitops-path: gitops/platform/domains/sales-domain
spec:
  type: domain
  lifecycle: production
  owner: platform/sales
```

Backstage ingests these descriptors so humans and agents can answer “which proto version is live in prod?” without scraping clusters.[^backstage] Automation (e.g. the `PrimitiveImplementationDraft` flow in [478-ameide-extensions.md](478-ameide-extensions.md)) uses the same metadata to open the right PRs in source and GitOps repos. `478` goes deeper on folder layout and template scaffolding; this section documents why the linkage is required.

[^buf]: Buf modules and breaking change detection – [buf.build/docs/breaking](https://buf.build/docs/breaking/?utm_source=chatgpt.com)
[^argo]: Argo CD best practices for separating source vs config repos – [argo-cd.readthedocs.io](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/?utm_source=chatgpt.com)
[^backstage]: Backstage Software Catalog descriptors – [backstage.io/docs/features/software-catalog](https://backstage.io/docs/features/software-catalog/?utm_source=chatgpt.com)

---

### 2.8 Proto Semantic Layers (Entities / Interfaces / Events)

Proto serves as Ameide's single schema language, but carries **three distinct semantics**:

| Semantic | Purpose | Proto construct | Package convention |
|----------|---------|-----------------|-------------------|
| **Entity** | Data that lives in the DB | `message` | `ameide.{domain}.v1` |
| **Interface** | Operations callers invoke | `service` | `ameide.{domain}.v1` |
| **Event** | Facts that happened | `message` | `ameide.{domain}.events.v1` |

These are differentiated via **conventions + lightweight options**, not a separate metadata system. This keeps us aligned with the "code-first, thin metadata" stance (470 §5).

#### 2.8.1 Entities

Entities are the "business objects" a Domain primitive owns. They map conceptually to tables or aggregates, but we don't force 1:1 with DB tables in proto (DB schema stays an implementation detail).

**Conventions:**

* Live in the domain package for that bounded context
* Can be tagged with `(ameide.entity_kind)` for tooling: `AGGREGATE`, `PROJECTION`, `LOOKUP`

```proto
package ameide.sales.v1;

message Opportunity {
  string id = 1;
  string tenant_id = 2;
  string organization_id = 3;
  string customer_id = 4;
  string name = 5;
  string stage = 6;
}

// Projection entity (like a read model / view)
message OpportunityListItem {
  option (ameide.entity_kind) = PROJECTION;
  string id = 1;
  string customer_name = 2;
  string stage = 3;
}
```

#### 2.8.2 Interfaces

Interfaces are the `service` definitions – what UIs, agents, or other domains actually call. We typically split into:

* **Domain APIs** – internal, rich operations
* **Integration APIs** – external, stable contracts

```proto
service SalesDomainService {
  option (ameide.interface_kind) = DOMAIN;

  // Commands
  rpc CreateOpportunity(CreateOpportunityRequest) returns (Opportunity);
  rpc UpdateOpportunityStage(UpdateOpportunityStageRequest) returns (Opportunity);

  // Queries
  rpc GetOpportunity(GetOpportunityRequest) returns (Opportunity);
  rpc ListOpportunities(ListOpportunitiesRequest) returns (ListOpportunitiesResponse);
}

service SalesIntegrationService {
  option (ameide.interface_kind) = INTEGRATION;

  rpc UpsertOpportunityProjection(OpportunityProjection) returns (UpsertResult);
  rpc StreamOpportunityChanges(ChangesRequest) returns (stream OpportunityChangeEvent);
}
```

#### 2.8.3 Events

Events are messages representing facts that already happened. They live in a separate `events` package and are tagged by kind:

* `BUSINESS` – external integration events (stable contracts)
* `DOMAIN_INTERNAL` – internal domain events (implementation detail)

```proto
package ameide.sales.events.v1;

message OpportunityWon {
  option (ameide.event_kind) = BUSINESS;
  string id = 1;
  string tenant_id = 2;
  string customer_id = 3;
  double amount = 4;
  string currency = 5;
  google.protobuf.Timestamp won_at = 6;
}

message OpportunityStageChanged {
  option (ameide.event_kind) = DOMAIN_INTERNAL;
  string id = 1;
  string from_stage = 2;
  string to_stage = 3;
}
```

Events are published by Domain/Process primitives via Watermill (§3.3.1) and consumed by other domains or projections.

#### 2.8.4 Typical Domain Layout

```text
ameide/
  sales/
    v1/
      sales_entities.proto      # Opportunity, Customer, Quote, ...
      sales_service.proto       # SalesDomainService
      sales_integration.proto   # SalesIntegrationService (optional)
    events/
      v1/
        sales_events.proto      # OpportunityWon, InvoicePosted, ...
```

#### 2.8.5 Design Heuristic

When deciding where something belongs:

> * *Do I need to store this as domain state?* → **Entity** (`message` in domain package)
> * *Is this an operation a caller invokes?* → **Interface** (`service`)
> * *Is this a fact that already happened?* → **Event** (`message` in `events/v1`)

For mappings to D365 and SAP concepts, see [470a-ameide-vision-vs-d365.md §7.7](470a-ameide-vision-vs-d365.md) and [470b-ameide-vision-vs-saps4.md §6.7](470b-ameide-vision-vs-saps4.md).

#### 2.8.6 CQRS Recipe (Commands / Queries / Events)

This is the standard pattern for implementing a Domain primitive.

##### Command/Event Naming Discipline

**Principle**: No API should say "set this field to X". It should say "do this business thing".

| Pattern | Allowed Examples | Forbidden Examples |
|---------|------------------|-------------------|
| **Commands** | `PlaceOrder`, `CancelOrder`, `ApproveQuote`, `AssignOwner` | `UpdateOrder`, `SetStatus`, `PatchOrder`, `ModifyQuote` |
| **Events** | `OrderPlaced`, `OrderCancelled`, `QuoteApproved`, `OwnerAssigned` | `OrderUpdated`, `StatusChanged`, `OrderModified` |
| **Queries** | `GetOrder`, `ListOrders`, `FindOrdersByCustomer` | (any that mutate state) |

**Rules**:
1. **Commands are business verbs** (imperatives that mean something to the business), not CRUD operations
2. **Events are past-tense facts** (what actually happened), not field changes
3. **Queries are read-only** and never have side effects
4. **Only Domain primitives write to DB**; Process/Agent/UISurface call Domain APIs

**Proto enforcement** (validated by `ameide primitive verify`):
- Domain/Process services: Allowed prefixes include `Create/Place/Cancel/Approve/Reject/Assign/Submit/Start/Complete/Archive/...`
- Forbidden generics: `Update/Set/Delete/Patch/Modify` (too vague, doesn't express business intent)
- Exception: `Delete` is acceptable for hard-delete semantics when business intent is clear (e.g., `DeleteDraftOrder`)

**Event emission rule**:
- Every command that changes business state **must** emit at least one domain event
- The DB row is a **projection** of events, not something callers manipulate directly
- This enables audit trails, event-driven integrations, and eventual consistency

##### Standard Patterns

| Pattern | Proto Construct | Naming Convention | Publishing |
|---------|-----------------|-------------------|------------|
| **Command** | RPC that mutates state | Business verb: `PlaceOrder`, `ApproveQuote` | Synchronous call; emits Event |
| **Query** | RPC that reads state | `GetFoo`, `ListFoos`, `SearchFoos` | Synchronous call; no side effects |
| **Event** | Message representing a fact | Past-tense: `OrderPlaced`, `QuoteApproved` | Published via Watermill CQRS |

**Typical flow:**

```text
[UISurface / Agent]
    → Command RPC (CreateOpportunity)
    → Domain primitive validates & persists
    → Domain primitive publishes Event (OpportunityCreated)
    → Event subscribers (other domains, projections, Graph)
```

**Example service with Commands and Queries:**

```proto
service OpportunityService {
  // Commands (mutate state, emit events) - business verbs, not CRUD
  rpc CreateOpportunity(CreateOpportunityRequest) returns (Opportunity);
  rpc QualifyOpportunity(QualifyOpportunityRequest) returns (Opportunity);  // Not "UpdateStage"
  rpc WinOpportunity(WinOpportunityRequest) returns (Opportunity);
  rpc LoseOpportunity(LoseOpportunityRequest) returns (Opportunity);
  rpc ReassignOpportunity(ReassignOpportunityRequest) returns (Opportunity);  // Not "SetOwner"

  // Queries (read-only, no side effects)
  rpc GetOpportunity(GetOpportunityRequest) returns (Opportunity);
  rpc ListOpportunities(ListOpportunitiesRequest) returns (ListOpportunitiesResponse);
  rpc SearchOpportunities(SearchOpportunitiesRequest) returns (SearchOpportunitiesResponse);
}
```

Note: `QualifyOpportunity` instead of `UpdateOpportunityStage` - the command expresses **what the business is doing**, not which field is changing. The domain decides internally what "qualify" means (stage change, validation, notifications, etc.).

**Event publishing (Go example with Transactional Outbox):**

```go
func (s *OpportunityService) WinOpportunity(ctx context.Context, req *pb.WinOpportunityRequest) (*pb.Opportunity, error) {
    // Start transaction - aggregate update + outbox insert must be atomic
    tx, err := s.db.BeginTx(ctx, nil)
    if err != nil { return nil, err }
    defer tx.Rollback()

    // 1. Validate and update aggregate
    opp, err := s.repo.GetForUpdate(ctx, tx, req.Id)
    if err != nil { return nil, err }
    opp.Stage = "won"
    opp.WonAt = timestamppb.Now()

    // 2. Persist aggregate (within transaction)
    if err := s.repo.Save(ctx, tx, opp); err != nil { return nil, err }

    // 3. Write event to outbox (same transaction - NO dual write!)
    event := &events.OpportunityWon{
        Id:         opp.Id,
        TenantId:   opp.TenantId,
        CustomerId: opp.CustomerId,
        Amount:     opp.Amount,
        WonAt:      opp.WonAt,
    }
    if err := s.outbox.Insert(ctx, tx, "sales.events.v1.OpportunityWon", event); err != nil {
        return nil, err
    }

    // 4. Commit transaction - aggregate + event are now atomically persisted
    if err := tx.Commit(); err != nil { return nil, err }

    return opp, nil
}
```

**Important**: Never call `publisher.Publish()` directly from domain code. The outbox dispatcher (a separate background process) reads the outbox table and publishes to the broker, handling retries and delivery guarantees. See §3.3.1.1 for full outbox pattern details.

This pattern ensures:
- Commands are the only way to mutate state
- Queries never have side effects
- **No dual writes**: aggregate + event are in the same transaction
- Events provide eventual consistency across domains
- The outbox pattern (§3.3.1.1) ensures reliable, at-least-once event delivery

### 2.9 Primitive Code Generation Philosophy

The CLI scaffolds **technical shape**, not business meaning. This follows patterns established by industry tools:

| Tool | What It Generates | Ameide Equivalent |
|------|-------------------|-------------------|
| **Buf/Connect** | Types, clients, server handler interfaces | Domain primitive skeletons |
| **Backstage Templates** | Repo structure, Docker, CI, docs | All primitive scaffolds |
| **Kubebuilder** | CRD types, controller skeleton, RBAC | Operator scaffolds only |
| **Temporal examples** | Worker main, workflow/activity signatures | Process primitive skeletons |

#### 2.9.1 What Gets Generated per Primitive Kind

| Primitive | Generated (Shape) | Not Generated (Meaning) |
|-----------|-------------------|------------------------|
| **Domain** | `cmd/main.go`, handler files with signatures, repository interfaces, test harness | Handler implementations, invariants, SQL migrations |
| **Process** | Worker main, workflow/activity stubs, Temporal registration | Workflow logic, compensations, branching |
| **Agent** | Agent harness, tool skeletons, prompt config files | Prompts, reasoning logic, tool implementations |
| **UISurface** | Next.js app skeleton, SDK wrapper, E2E test scaffold | Pages, components, UI workflows |

#### 2.9.2 Generation Rules

1. **Idempotent and additive**: Only create files that don't exist; never overwrite hand-edited code
2. **Bootstrap, not serving layer**: After scaffold, all changes are normal editing
3. **Proto as single source**: Handler signatures derived from proto via Buf, not a second DSL
4. **One-shot**: Scaffold only when folder doesn't exist; refuse to regenerate
5. **Backstage alignment**: CLI templates match Backstage Software Templates for the same primitives

#### 2.9.3 Anti-Pattern: JHipster-Style Full-Stack Generation

Avoid generating:
- Full CRUD stacks from domain models
- Entity structs beyond proto definitions
- Any "businessy" validation or aggregation
- DSLs deeper than proto + Transformation artifacts

This would recreate the "metadata universe" problem where generated code becomes untouchable and domain invariants get smudged into templates.

**The CLI is 70% observer/validator** (`describe`, `verify`, `impact`) **and 30% scaffolder of boring wiring** (`scaffold` for structure + tests).

For CLI command details, see [484-ameide-cli.md §4](484-ameide-cli.md).

---

## 3. Information Architecture

### 3.1 Multi-tenant Data Layout (conceptual)

At the information level the platform is **logically multi-tenant**, regardless of how physical databases are configured in the tech layer.

> **Security**: For the full security model and implementation gaps (RLS, `organization_id` columns, API authorization), see [476-ameide-security-trust.md](476-ameide-security-trust.md).

Core concepts:

* **Tenant** (Platform domain)

  * Owns org structure, users, environments, configuration.
* **Tenant isolation**

  * Every domain, process, agent, artifact row carries `tenant_id`.
  * Cross-tenant access only allowed in platform/operations contexts.

We keep **three main categories of data**:

1. **Operational domain data**

   * e.g. leads, quotes, orders, invoices, backlog items, initiatives.
   * Authoritative per domain.

2. **Process execution data**

   * e.g. L2O instances, O2C instances, transformation sprints.
   * Lives in process engines and is projected into Ameide for analytics.

3. **Design / knowledge artifacts**

   * Transformation design tooling content: BPMN, ArchiMate, Markdown docs, etc.
   * Projection graph: cross-domain graph of entities and relations.
   * Backstage catalog entities: domain/process/agent templates and instances.

### 3.2 Domain Data

Each domain specifies:

* **Canonical aggregates** (e.g. Opportunity, Quote, Order, Invoice, Initiative, BacklogItem).
* **Invariants & state machines** embedded in its APIs (e.g. Invoice can only move from `Draft` → `Posted` via a validated posting command).
* **Tenant-scoped reporting views** (read models) that can be:

  * Direct SQL views
  * Materialized views
  * Projections into Graph

Older IPA architecture already codifies the pattern of “pure domain + persistence + builder/runtime”; we reuse the same layering for domains, but replace “IPA” as an aggregate with explicit Domain/Process definitions.

### 3.3 Process Data

For each process type (L2O, O2C, T2C, etc.) we define:

* **ProcessDefinition** = BPMN-compliant artifact (from custom React Flow modeller) + metadata
* **Process variables schema** = typed variables bound to domain identifiers
* **SLA and policy metadata** (e.g. L2O should close within 30 days)

This mirrors Temporal's distinction between workflow definitions and workflow executions: ProcessDefinitions are design-time artifacts; Process primitives execute them at runtime.

We will:

* Store **ProcessDefinitions** in **Transformation design tooling / Transformation domain** as versioned artifacts.
* Store **execution projections** in Ameide's own DB for:

  * process analytics
  * SLA monitoring
  * process-first UI views.

### 3.3.1 Event-driven CQRS runtime (Watermill)

Many Process primitives (especially those written in Go) are implemented with Watermill’s CQRS component so they can emit and consume commands/events as plain Go structs while relying on Watermill for serialization, routing, and pub/sub plumbing.

* **Commands** – intent to change state (e.g. `RegisterUser`, `CreateInvoice`). Process primitives send commands via `CommandBus`, which Watermill serializes and routes through the configured pub/sub backend (Kafka, RabbitMQ, Postgres, etc.). Command handlers mutate the write model and/or emit domain events.
* **Events** – facts that happened (e.g. `UserRegistered`). Domain primitives publish events through `EventBus`; read models and integrations subscribe via `EventProcessor` handlers to update projections or trigger side effects.
* **Queries** – read models (Postgres, Elastic, etc.) stay optimized for queries and are updated by event handlers.

Watermill’s CQRS layer gives us:

- Type-safe command/event handlers (`func(ctx context.Context, cmd *RegisterUser) error`) without hand-written topic or encoding code.
- Pluggable pub/sub backends, consistent middleware (logging, metrics, retries).
- A router abstraction that makes it easy to compose Process primitives from command and event handlers.

When describing Process/Domain implementations, reference Watermill CQRS so teams standardize on the same event-driven plumbing; see [`github.com/ThreeDotsLabs/watermill`](https://github.com/ThreeDotsLabs/watermill?utm_source=chatgpt.com) and the [Watermill CQRS docs](https://watermill.io/docs/cqrs/?utm_source=chatgpt.com). For lighter weight use cases (e.g., TypeScript agents) we still follow the same CQRS concepts even if Watermill is not involved.

#### 3.3.1.1 Transactional Outbox Pattern (Required)

**Problem**: Updating DB + publishing event is a **dual write**; one can succeed and the other fail, leading to inconsistent state.

**Solution**: In the same DB transaction as the business change, write an outbox row. A background dispatcher reads the outbox and publishes to the broker, retrying until success.

```go
func (s *OrdersService) CancelOrder(ctx context.Context, req *pb.CancelOrderRequest) (*pb.Order, error) {
    tx, _ := s.db.BeginTx(ctx, nil)
    defer tx.Rollback()

    // 1. Update aggregate
    order, _ := s.repo.GetForUpdate(ctx, tx, req.Id)
    order.Status = "cancelled"
    order.CancelledAt = time.Now()
    s.repo.Save(ctx, tx, order)

    // 2. Write event to outbox (same transaction)
    event := &events.OrderCancelled{
        Id:       order.Id,
        TenantId: order.TenantId,
        Reason:   req.Reason,
    }
    s.outbox.Insert(ctx, tx, "orders.events.v1.OrderCancelled", event)

    tx.Commit()
    return order, nil
}
```

**Rules**:
- Never call `publisher.Publish()` directly from domain code
- All event writes go through the outbox table
- The outbox dispatcher handles retries, ordering, and delivery guarantees

**CLI enforcement**: `verify` should warn if domain code publishes events without outbox.

**References**: [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html), [AWS Outbox Pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)

### 3.3.2 Idempotency & Inbox Pattern

Most brokers + outbox give **at-least-once** delivery. Consumers **must** be idempotent.

**Inbox Pattern**: Track processed event IDs to guarantee exactly-once processing per aggregate:

```go
func (h *OrderEventHandler) HandleOrderCreated(ctx context.Context, event *events.OrderCreated) error {
    // 1. Check inbox - already processed?
    if h.inbox.Exists(ctx, event.EventId) {
        return nil // Idempotent: skip duplicate
    }

    // 2. Process event (in transaction with inbox insert)
    tx, _ := h.db.BeginTx(ctx, nil)
    defer tx.Rollback()

    h.projection.UpdateFromOrder(ctx, tx, event)
    h.inbox.Insert(ctx, tx, event.EventId)

    tx.Commit()
    return nil
}
```

**Rules**:
- Every event handler must be idempotent given the same event ID
- Use Inbox table or derive idempotency from natural keys
- Single transaction per "consume event → update state" operation
- This aligns with the "no imperative writes" principle: applying `OrderCancelled` twice must yield the same state

**References**: [Inbox Pattern](https://event-driven.io/en/outbox_inbox_patterns_and_delivery_guarantees_explained/)

### 3.3.3 Go Concurrency Patterns for Events

**Principles**:
1. **One responsibility per goroutine**: reader, processor, dispatcher are separate
2. **Always propagate `context.Context`**: for cancellation, timeouts, and tracing
3. **Bounded worker pools**: over goroutine-per-message for backpressure control

**Standard event consumer shape** (this pattern is scaffolded by CLI):

```go
func (c *EventConsumer) Run(ctx context.Context) error {
    msgs := make(chan *Message, c.bufferSize)

    // Reader goroutine
    go c.readFromBroker(ctx, msgs)

    // Bounded worker pool
    var wg sync.WaitGroup
    for i := 0; i < c.workerCount; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for msg := range msgs {
                c.processWithRetry(ctx, msg)
            }
        }()
    }

    <-ctx.Done()
    close(msgs)
    wg.Wait()
    return ctx.Err()
}
```

**Anti-patterns to avoid**:
- Goroutine-per-message without bounds (leads to OOM under load)
- Forgetting to propagate context (breaks cancellation and tracing)
- Blocking the reader goroutine with slow processing

**References**: [Event-Driven Architecture in Go](https://blog.devops.dev/event-driven-architecture-in-go-golang-ab46f23bf9a8)

### 3.3.4 EDA Principles Checklist

For **every Domain/Process/Agent primitive** in Go, these principles apply:

| # | Principle | Enforcement |
|---|-----------|-------------|
| 1 | **Commands, not CRUD** | All writes via proto commands, not field setters (§2.8.6) |
| 2 | **Event as fact** | Every business change emits an event type in proto |
| 3 | **Outbox required** | DB + event writes in single transaction (§3.3.1.1) |
| 4 | **Consumers idempotent** | Inbox pattern or natural key idempotency (§3.3.2) |
| 5 | **Schema-first** | All events in Buf proto modules; no ad-hoc JSON (§2.7) |
| 6 | **Clean boundaries** | Domain logic doesn't import broker clients directly (§2.9) |
| 7 | **Explicit concurrency** | Bounded worker pools + contexts (§3.3.3) |
| 8 | **Observable** | Correlation IDs, structured logs, metrics per event |
| 9 | **12-factor** | Stateless services, config via env, GitOps for runtime |

CLI `verify` enforces principles 1, 3, 4, 5 via static analysis; principles 7-9 are checked in code review and observability dashboards.

### 3.3.5 Event Schema Versioning

**Principle**: Event schemas evolve over time. Use Buf proto conventions for backwards compatibility:

| Rule | Description |
|------|-------------|
| **Additive only** | Add new optional fields; never remove or rename existing fields |
| **Reserved fields** | Use `reserved` for deprecated field numbers to prevent reuse |
| **Package versioning** | Breaking changes require a new package (`events.v2`) |

```proto
package ameide.sales.events.v1;

message OpportunityWon {
    string id = 1;
    string tenant_id = 2;
    double amount = 3;
    google.protobuf.Timestamp won_at = 4;

    // v1.1 addition (backwards compatible)
    string currency = 5;

    reserved 6; // was: deprecated_field
}
```

**Consumer compatibility rules**:
- Handle missing optional fields gracefully (use proto3 defaults or explicit presence checks)
- Never fail on unknown fields (proto3 default behavior)
- When breaking changes are unavoidable, publish to `events.v2` and run parallel consumers during migration

**CLI enforcement**: `verify` warns when event messages violate Buf breaking change rules.

### 3.3.6 Dead Letter Queues & Error Handling

Events that repeatedly fail processing should be routed to a dead-letter queue (DLQ) for manual inspection:

| Scenario | Handling |
|----------|----------|
| **Transient failure** | Retry with exponential backoff (Watermill middleware) |
| **Poison pill** | After N retries, route to DLQ, continue processing |
| **Schema mismatch** | Log error, route to DLQ, alert team |

**Observability requirements**:

| Metric | Purpose |
|--------|---------|
| `events_processed_total{domain, event_type, outcome}` | Track success/failure/dlq rates |
| `consumer_lag_seconds{domain, consumer_group}` | Monitor processing delays |
| `dlq_depth{domain}` | Alert on accumulating failures |

**Alerting thresholds**:
- Consumer lag > 5 minutes → Warning
- DLQ depth > 0 → Alert (requires manual investigation)

### 3.3.7 Multi-Tenant Event Isolation

Events carry `tenant_id` and consumers **must** enforce tenant scoping:

```go
func (h *Handler) HandleEvent(ctx context.Context, event *events.OpportunityWon) error {
    // Validate tenant context matches event
    ctxTenant := middleware.TenantFromContext(ctx)
    if event.TenantId != ctxTenant {
        return fmt.Errorf("tenant mismatch: event=%s, context=%s", event.TenantId, ctxTenant)
    }
    // ... process event
}
```

**Topic partitioning strategy**:
- Partition by `tenant_id` to ensure ordering within a tenant
- Allows parallel processing across tenants
- Prevents cross-tenant message leakage

**Multi-tenant isolation rules**:
1. Every event message **must** include `tenant_id` field
2. Consumers validate `tenant_id` against execution context
3. RLS policies in downstream projections enforce tenant isolation
4. Event streams are never shared across tenants without explicit cross-tenant integration

### 3.4 Knowledge Graph (Read-side only)

> **Core Definitions** (see [470-ameide-vision.md §0](470-ameide-vision.md)):
> - **Graph** is a read-only knowledge layer that projects selected data from any primitive into a graph database.
> - All writes go through primitives; Graph is never a source of truth.

The Knowledge Graph provides a **cross-domain read-only model**:

* Entities: Customer, Product, Opportunity, ProcessInstance, Initiative, System, API, etc.
* Relationships: `CUSTOMER->HAS_OPPORTUNITY`, `OPPORTUNITY->FLOWS_THROUGH_L2O`, `SERVICE->IMPLEMENTS_API`.

This is *not* an authoritative store:

* Write path: Domain/Process primitives project to graph as part of their post-commit flow.
* Read path: agents and analytics use it to understand topology and behavior (e.g. "show me all L2O paths where margin < 10%").

**Design artifacts** (BPMN, architecture diagrams, Markdown) are persisted by the **Transformation Domain**, using an artifact + revision pattern. Graph can project these artifacts or references to them for cross-domain queries, but the canonical records remain in the Transformation Domain.

### 3.5 Transformation / Transformation design tooling Data

> **Core Definitions** (see [470-ameide-vision.md §0](470-ameide-vision.md)):
> - **Transformation Domain** owns all design-time artifacts (ProcessDefinitions, AgentDefinitions, BPMN, diagrams, Markdown).
> - **Transformation design tooling** is the set of modelling UIs (BPMN editor, diagram editor, Markdown editor) that call Transformation Domain APIs—Transformation design tooling has no independent storage.

Transformation is a **Domain primitive like any other**, owning:

* **Initiatives & Backlog Items**

  * Standard domain entities (SQL) representing epics, features, tasks.
* **Design-time artifacts** (stored in Transformation Domain, modelled via Transformation design tooling UIs)

  * **ProcessDefinitions** – BPMN-compliant process models (produced by custom React Flow modeller).
  * **AgentDefinitions** – Declarative agent specs (tools, policies, risk tiers).
  * **ExtensionDefinitions** – WASM extension specs for process/domain/agent hooks.
  * ArchiMate, Markdown, etc., optionally event-sourced internally with commands and snapshots.
* **Governance**

  * Promotion status of artifacts (draft, in review, promoted).
  * Links from artifacts to runtime primitives (e.g. "ProcessDefinition L2O_v3 is executed by Process primitive l2o-process-svc").

> **Important**: There is no separate "Transformation design tooling service" in the runtime. The event-sourced artifact store is part of the Transformation Domain. Transformation design tooling is the *client* (modelling experience) that sends commands to Transformation.

Transformation Agent primitives operate primarily on this domain:

* Read backlog + design artifacts (ProcessDefinitions, AgentDefinitions) from Transformation Domain.
* Generate or modify Backstage templates and Domain primitive/Process primitive/Agent primitive configurations.
* Propose changes that are then applied via standard deployment workflows.

---

## 4. Backstage as the Application Catalog & Template Engine

> **Important:** Backstage is an **internal factory** for Ameide engineers and transformation agents; **tenants never see Backstage**. Tenant-facing UX is delivered through UISurface primitives (Next.js apps), not through Backstage.

We standardize on **Backstage** as the catalog and templating layer for:

* Domain primitives
* Process primitives
* Agents
* UI components / microfrontends

This aligns with Backstage’s ecosystem modeling and software templates features.

### 4.1 Catalog modeling

**Backstage entities** (examples):

* `Domain` (custom kind)

  * Represents a bounded context such as *Sales*, *Billing*, *Transformation*.
* `Component` (service)

  * Domain primitive service (e.g. `sales-domain-svc`).
  * Process primitive service (e.g. `l2o-process-svc`).
* `API`

  * Proto-defined API surface of a domain or process.
* `Resource`

  * External dependencies (DB cluster, Kafka topics, Camunda cluster, Temporal cluster).
* `Template`

  * Domain/Process/Agent/UI templates used by the scaffolder.

**Mapping to Ameide concepts**

| Ameide concept     | Backstage kind                     |
| ------------------ | ---------------------------------- |
| Domain primitive   | `Component` (+ custom `Domain`)    |
| Process primitive  | `Component` + BPMN artifact link   |
| Agent              | `Component` or custom `Agent` kind |
| UI workspace       | `Component` (frontend)             |
| Domain API         | `API`                              |
| Process API        | `API` (process control / queries)  |
| Transformation design tooling artifact       | `Resource` or custom `Artifact`    |

This keeps all “things that exist” in the platform visible and discoverable via the catalog.

### 4.2 Templates for Domain / Process / Agent / UI

We provide **four base templates** (each parameterized per tenant):

1. **Domain Template**

   * Inputs: domain name, initial aggregates, tenant scope, persistence plan.
   * Generates:

     * Proto definitions & service skeletons (aligned with proto-based API guidelines).
     * Base repository & migrations.
     * Domain events schema.
     * Backstage `Component` + `Domain` entities.
   * Vendor-aligned with Backstage’s Software Templates “skeleton + publish” model.

2. **Process primitive Template**

   * Inputs: process type (L2O, O2C, custom), participating domains, SLA target.
   * Generates:

     * Initial ProcessDefinition (BPMN-compliant, from React Flow modeller) with lane structure (Sales, Billing, Logistics, etc.).
     * Temporal workflow skeletons (service tasks bound to Domain primitive APIs).
     * Process primitive service skeleton.
     * Backstage `Component` entity + links to domain APIs.

3. **Agent primitive Template**

   * Inputs: agent purpose, allowed domains/processes, risk tier.
   * Generates:

     * AgentDefinition (prompt, tools, scopes, policies) stored in Transformation Domain.
     * Proto definition for the agent configuration & invocation.
     * Agent primitive service skeleton.
     * Backstage `Component`/`Agent` entity with RBAC metadata for who can use this template.

4. **UISurface primitive Template**

   * Inputs: target domain/process, view style (workspace vs process view), routing (hosts/paths), required auth scopes.
   * Generates:

     * Next.js app skeleton with Ameide TS SDK integration.
     * UISurface CRD declaring image, routing, auth scopes, and dependencies on Domain/Process/Agent primitives.
     * Backstage `Component` entry of type `frontend`.
     * GitOps skeleton (`values.yaml`, `kustomization.yaml`) matching the UISurface CRD shape.
   * See [477-primitive-stack.md §3.4](477-primitive-stack.md) for UISurface operator reconciliation details.

Backstage templates remain **the control plane** for agents too: a transformation agent manipulates template parameters and triggers scaffolding rather than directly creating low-level services.

> Templates are strictly **Tier 2** tooling for full primitives. Tier 1 WASM extensions follow the `ExtensionDefinition` + shared runtime path from 479/480 and do not create new services via Backstage.

Every primitive-oriented template emits both a repository skeleton (code, proto, CI) and the corresponding Domain/Process/Agent/UISurface CR so GitOps/Argo can apply the runtime object. Human engineers and agents therefore work with the same declarative surface (the CR specs) regardless of who authored the primitive.

---

## 5. Example: L2O Application & Information Flow

To make this less abstract, here’s how **Lead-to-Opportunity (L2O)** looks in this architecture.

### 5.1 Involved building blocks

* Domain primitives:

  * `SalesDomain` (Lead, Contact, Opportunity)
  * `ProductDomain` (Products, Price Lists)
  * `BillingDomain` (for credit check / pre-invoice)
* Process primitive:

  * `L2OProcess` – BPMN definition + process service.
* Agents:

  * `SalesCoachAgent` (helps sales with next actions)
  * `TransformationAgent` (helps the org refine L2O version)
* UI:

  * `SalesWorkspace` (classic ERP-style)
  * `L2OProcessView` (timeline / swimlane)

### 5.2 Data lifecycle

1. **Lead created**

   * `SalesDomain` persists `Lead` row with `tenant_id`.
   * Domain emits `LeadCreated` event → Graph projects `CustomerCandidate` node.

2. **L2O instance started**

   * `L2OProcess primitive` is started with `leadId` (Temporal).
   * Process instance state is stored in Temporal; Ameide stores a projection (e.g. "L2O phase = Qualification").

3. **Quote created**

   * `L2OProcess primitive` reaches "Prepare Quote" task.
   * For manual: Sales user opens `SalesWorkspace`, sees tasks from `L2OProcessView` and creates a Quote via `SalesDomain primitive` APIs.
   * For automated: an Agent primitive or automated task calls `SalesDomain primitive::CreateQuote` using its proto API.

4. **Approval**

   * `L2OProcess primitive` orchestrates approval steps, maybe invoking:

     * `TransformationAgent primitive` to suggest better pricing.
     * `SalesCoachAgent primitive` to highlight risk.
   * Agent primitives read from Graph & domain APIs but write back via domain commands only.

5. **Handover to O2C**

   * Once L2O is "Won", `L2OProcess primitive`:

     * Calls `BillingDomain primitive::CreateProformaInvoice`.
     * Emits `OpportunityWon` → `O2CProcess primitive` starter.

Information picture:

* **Domain tables**

  * `sales_leads`, `sales_opportunities`, `sales_quotes`, `billing_preinvoices`.
* **Process tables / engine**

  * Temporal internal tables for instance execution.
  * Ameide `process_instances` projection for analytics and UI.
* **Graph**

  * Nodes for Customer, Lead, Opportunity, Quote, L2OInstance.
  * Edges for flows (Lead → L2O → Opportunity → O2C).

Agents “see” the world via:

* Graph queries (structure + history).
* Domain APIs (truth).
* Transformation design tooling artifacts describing L2O design (as the process improves).

---

## 6. Alignment with Existing Specs

This section is just to show that we’re not inventing yet another stack, but reusing patterns from existing Ameide backlogs.

1. **IPA Architecture (043)**

   * IPA = "composition of workflows + agents + tools from DB, compiled to runtimes".
   * New view:

     * *Domain primitives* ≈ "domain and storage" part of IPA.
     * *Process primitives* (executing ProcessDefinitions) ≈ "workflows" part.
     * *Agent primitives* (executing AgentDefinitions) ≈ "agents/tools" part.
   * The layered approach (Domain → Storage → Builder → Runtime) remains valid, but the *unit* moves from "IPA" to explicit ProcessDefinitions and AgentDefinitions (design-time) executed by primitives (runtime).

2. **Proto-based APIs (044)**

   * All domain/process/agent services are still proto-first, with gRPC + REST surfaces and consistent error handling.

3. **North-Star (064)**

   * Design vs Deploy vs Runtime separation is kept:

     * Design: Backstage + Transformation design tooling + transformation domain.
     * Deploy: primitives & process engines.
     * Runtime: domain & process services.

4. **Transformation design tooling (067)**

   * The Transformation design tooling *pattern* (artifacts, revisions, promotions) is implemented inside the **Transformation Domain**.
   * Transformation design tooling as a UI surfaces those artifacts; the primitive remains the source of truth.
   * Instead of a generic "IPA designer", the Transformation Domain (with Transformation design tooling UIs) is used to generate Backstage templates & primitive config.

5. **BPMN-compliant definitions (063)**

   * BPMN semantics (stages, gateways, lanes) remain valid; we use a **custom React Flow modeller** (not Camunda/bpmn-js) to produce ProcessDefinitions stored in the Transformation Domain (via Transformation design tooling UIs) and executed by Temporal-backed Process primitives.

6. **TypeScript SDK (050)**

   * The SDK acts as the canonical way UI and other services call Domain/Process/Agent APIs, keeping the application boundary clean.

---

## 8. Cross-References

This Application & Information Architecture should be read with:

| Document | Relationship | Alignment |
|----------|-------------|-----------|
| [470‑ameide‑vision](470-ameide-vision.md) | Parent vision & principles | Strong ✅ |
| [471‑ameide‑business‑architecture](471-ameide-business-architecture.md) | Business context & journeys | Strong ✅ |
| [473‑ameide‑technology](473-ameide-technology.md) | Technology implementation | Strong ✅ |
| [475‑ameide‑domains](475-ameide-domains.md) | Domain portfolio & patterns | Strong ✅ |
| [300‑ameide‑metamodel](300-ameide-metamodel.md) | Element graph foundation | See §8.1 |
| [305‑workflow](305-workflow.md) | Process primitive implementation | See §8.2 |
| [310‑agents‑v2](310-agents-v2.md) | Agent primitive implementation | See §8.3 |
| [370‑event‑sourcing](370-event-sourcing.md) | Event sourcing exploration | See §8.4 |
| [478‑ameide‑extensions](478-ameide-extensions.md) | Extension namespace topology | See §8.5 |
| [479‑ameide‑extensibility‑wasm](479-ameide-extensibility-wasm.md) | Tier 1 + Tier 2 extensibility model | See §2.5, §8.5 |
| [480‑ameide‑extensibility‑wasm‑service](480-ameide-extensibility-wasm-service.md) | Shared runtime implementation | See §2.5 |

### 8.1 Metamodel alignment (300)

The Element graph (300) and Application Architecture (472) coexist:

* **Graph = read-side projection**: Cross-domain knowledge for agents, analytics, and transformation. Not authoritative for operational data.
* **Domain stores = write-side authority**: Each Domain primitive owns its data; graph receives projections.
* **Design artifacts are projected**: BPMN, ArchiMate, Markdown stored in **Transformation Domain** and **projected** as versioned nodes in the graph.

> **Important**: Transformation remains the source of truth for transformation artifacts; the graph stores references and read-optimised views, not the authoritative records.

This means: Transformation domain entities project to `graph.elements`, but the canonical records stay in the Transformation Domain. Process primitive runtime state stays in Temporal.

### 8.2 Workflow alignment (305)

305 describes platform workflow infrastructure implementing **Process primitives** from this document:

* "Platform Workflows" = implementation detail of Process primitives (§2.2)
* Temporal is the runtime; customers see "process stages" and "process boards"
* ProcessDefinitions (BPMN-compliant, from custom React Flow modeller) stored in Transformation Domain (via Transformation design tooling UIs); compiled to Temporal workflows

### 8.3 Agent alignment (310)

310 describes Agent primitive implementation:

* AgentDefinitions stored in Transformation Domain (modelled via Transformation design tooling UIs); Agent primitives execute them
* Key constraint: Agent primitives invoked via domain/process APIs; never become source of truth

### 8.4 Event sourcing alignment (370)

472 §3.5 describes Transformation Domain with "optionally event-sourced" artifacts:

* Event contracts per domain, outbox + relay, projections from streams
* The event-sourced pattern is an internal implementation detail of the Transformation Domain, not a separate service

### 8.5 Extension alignment (478)

478 describes how tenant-specific primitives are cataloged and deployed:

* **Platform repo** contains `service_catalog/` with base primitive structure.
* **Tenant repos** (`tenant-{id}-controllers`, `tenant-{id}-gitops`) hold custom code and manifests.
* Backstage catalog uses **Locations** pointing at tenant repos, so tenant primitives appear in the internal portal.
* Namespace topology by SKU determines where primitives deploy (§4 of 478).

The catalog entity mappings in §4.1 extend to tenant primitives:

| Ameide Concept | Backstage Kind | Repository |
|----------------|----------------|------------|
| Platform Domain primitive | `Component` | Platform repo |
| Tenant Domain primitive | `Component` | `tenant-{id}-controllers` |
| Platform Process primitive | `Component` | Platform repo |
| Tenant Process primitive | `Component` | `tenant-{id}-controllers` |
| Platform Agent primitive | `Component` | Platform repo |
| Tenant Agent primitive | `Component` | `tenant-{id}-controllers` |

Only the **Transformation Domain** manipulates primitive definitions (ProcessDefinitions, AgentDefinitions, ExtensionDefinitions, PrimitiveImplementationDrafts). Backstage templates and GitOps keep platform vs tenant concerns isolated while letting us enforce guardrails (namespaces, secrets, rollout waves).

---

## 9. Terminology

| Legacy Term | 472 Term |
|-------------|----------|
| IPA (Intelligent Process Automation) | Domain primitive + Process primitive + Agent primitive bundle |
| Platform Workflows (305) | Process primitive (executes ProcessDefinitions) |
| AgentRuntime (310) | Agent primitive (executes AgentDefinitions) |
| "Graph service" | Knowledge Graph (read projection) |
| DomainService | Domain primitive |
| BPMN definition | ProcessDefinition (from custom React Flow modeller) |
