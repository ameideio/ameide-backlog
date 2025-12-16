# 478 – Ameide Extensions & Tenant Primitive Model

**Status:** Draft v1
**Owner:** Architecture / Platform
**Audience:** Platform engineers, domain architects, solution architects

> **Cross-References (Vision Suite)**:
>
> | Document | Purpose |
> |----------|---------|
> | [470-ameide-vision.md](470-ameide-vision.md) | Vision, rationale, principles |
> | [471-ameide-business-architecture.md](471-ameide-business-architecture.md) | Business concepts, tenant journey |
> | [472-ameide-information-application.md](472-ameide-information-application.md) | Application architecture, Backstage catalog |
> | [473-ameide-technology.md](473-ameide-technology.md) | Technology stack, deployment |
> | [475-ameide-domains.md](475-ameide-domains.md) | Domain portfolio, primitive patterns |
> | [476-ameide-security-trust.md](476-ameide-security-trust.md) | Security principles, agent governance |
> | [467-backstage.md](467-backstage.md) | Backstage implementation |
> | [479-ameide-extensibility-wasm.md](479-ameide-extensibility-wasm.md) | Tier 1 + Tier 2 extensibility framing |
> | [480-ameide-extensibility-wasm-service.md](480-ameide-extensibility-wasm-service.md) | Tier 1 runtime implementation |
>
> **Related Implementation**:
> - [443-tenancy-models.md](443-tenancy-models.md) – SKU definitions (Shared/Namespace/Private)
> - [465-applicationset-architecture.md](465-applicationset-architecture.md) – GitOps deployment model
> - [447-waves-v3-cluster-scoped-operators.md](447-waves-v3-cluster-scoped-operators.md) – Rollout phases

---

## 1. Purpose

This document describes **how tenant-specific primitives are created, deployed, and isolated** within the Ameide platform. It operationalizes the architecture principles from 470-477 into concrete workflows.

**Scope note:** This document covers **Tier 2 primitive-based extensions** (Domain/Process/Agent primitives owned by tenants). Tier 1 WASM extensions stay inside the shared runtime described in [479](479-ameide-extensibility-wasm.md)/[480](480-ameide-extensibility-wasm-service.md); they only appear here when referencing shared namespace/security invariants.

Key questions answered:

* How does Backstage fit into the extension model?
* Where does platform code vs tenant code live?
* How are namespaces structured by SKU?
* What is the E2E flow from requirement to running a new primitive?
* What security invariants must hold?

---

## 2. What Backstage Is in Ameide

### 2.1 Positioning

Backstage is the **internal factory + map** for primitives:

* Domain primitives
* Process primitives
* Agent primitives

It is **never tenant-facing**. Tenants only see the Ameide platform UI.

Backstage pulls truth from:

* `service_catalog/` in the platform repo (which primitives exist, how they're wired)
* Tenant extension repos (for custom primitives)
* Runtime systems (ArgoCD, K8s, observability)

> **Think of it as**: "where platform engineers go to create & reason about primitives", not "where customers click buttons".
>
> Tier 1 WASM extensions do **not** use Backstage; they are stored as `ExtensionDefinition` artifacts and executed by the shared runtime (479/480).
>
> Tier 2 templates emit primitive repositories **and** the six primitive CRDs (Domain/Process/Agent/UISurface/Projection/Integration) that GitOps applies to realise runtime primitives.

### 2.2 Backstage Capabilities Used

| Backstage Feature | Ameide Usage |
|-------------------|--------------|
| Software Catalog | Registry of all Domain/Process/Agent primitives and APIs |
| Software Templates | Scaffolder templates for creating new primitives |
| TechDocs | Technical documentation aggregation |
| Integrations | GitHub, Keycloak, ArgoCD plugins |

See [467-backstage.md](467-backstage.md) for implementation details.

---

## 3. Repository Strategy

### 3.1 Platform Repository

Contains:

* `service_catalog/` with primitive structure and base implementations
* Backstage app + plugins
* **Backstage templates**:
  * Domain primitive template
  * Process primitive template
  * Agent primitive template
* Shared libraries / SDKs, infra charts, etc.

**Only Ameide-owned code lives here. No tenant custom code.**

### 3.2 Tenant-Side Repositories

Per tenant that has customisation:

| Repository | Purpose |
|------------|---------|
| `tenant-{id}-controllers` | Tenant-specific or tenant-scoped primitive code |
| `tenant-{id}-gitops` | Helm/Argo manifests for those primitives |

Backstage catalog has **Locations** pointing at these repos, so tenant primitives show up in the internal portal, but code and manifests stay out of the platform repo.

### 3.3 Repository Ownership Matrix

| Code Origin | Repository | Who Can Commit |
|-------------|------------|----------------|
| Platform | `ameide-platform` | Ameide engineers only |
| Tenant (custom) | `tenant-{id}-controllers` | Tenant engineers + approved agents |
| GitOps (platform) | `ameide-gitops` | Ameide engineers + CI |
| GitOps (tenant) | `tenant-{id}-gitops` | Tenant engineers + approved agents |

---

## 4. Namespace Strategy by SKU

For each environment (`dev`, `staging`, `prod`), namespaces are structured based on SKU.

### 4.1 Shared Platform Plane

* `ameide-{env}`
  * Multi-tenant platform primitives (shared product)
  * RLS + tenantId/orgId for data isolation
  * **Code must come from platform repos only**
  * **No tenant-specific/custom code ever**
  * Tier 1 WASM modules run here only as data executed by the platform-owned runtime (see 479/480); they are not services.

### 4.2 Tenant Planes by SKU

#### Shared SKU

* Shared product runs in: `ameide-{env}`
* If the tenant has custom code, they get:
  * `tenant-{id}-{env}-cust`
    * Tenant/agent-generated primitives
    * Calls **into** `ameide-{env}` only via well-defined APIs (HTTP/gRPC) with `tenantId` in auth
    * No direct access to shared DBs or internal services
  * No `tenant-{id}-{env}-base` (reserved for dedicated namespace/cluster SKUs)

#### Namespace SKU

On a shared cluster with dedicated namespace:

* `tenant-{id}-{env}-base`
  * Primary product runtime **dedicated to that tenant** (their "base" namespace)
  * Platform-managed primitives specialised for that tenant
* Optionally `tenant-{id}-{env}-cust`
  * Tenant/agent-owned code, kept separate from the "product" base

#### Private SKU

On a dedicated cluster:

* `tenant-{id}-{env}-base`
  * Primary product runtime on dedicated infrastructure
* Optionally `tenant-{id}-{env}-cust`
  * Custom extensions

### 4.3 Namespace Naming Convention

```
ameide-{env}                    # Shared platform (Shared SKU product)
tenant-{id}-{env}-base          # Dedicated product (Namespace/Private SKU)
tenant-{id}-{env}-cust          # Custom extensions (all SKUs)
```

Where:
* `{env}` = `dev` | `staging` | `prod`
* `{id}` = tenant identifier (e.g., `acme`, `contoso`)

### 4.4 Namespace Decision Matrix

| SKU | Product Runtime | Custom Code |
|-----|-----------------|-------------|
| Shared | `ameide-{env}` | `tenant-{id}-{env}-cust` |
| Namespace | `tenant-{id}-{env}-base` | `tenant-{id}-{env}-cust` |
| Private | `tenant-{id}-{env}-base` | `tenant-{id}-{env}-cust` |

**Invariant**: Custom services always run in a dedicated tenant namespace, never in the shared `ameide-*` namespaces. Tier 1 WASM extensions are the single, governed exception and execute as data within the shared runtime per [479](479-ameide-extensibility-wasm.md)/[480](480-ameide-extensibility-wasm-service.md).

---

## 5. Backstage Templates

### 5.1 Golden Templates

We maintain 3–4 **golden templates** in the platform repo:

| Template | Purpose |
|----------|---------|
| `Domain primitive` | Create a new domain with proto APIs, DB schema, Helm chart |
| `Process primitive` | Create a process runtime with Temporal workers |
| `Agent primitive` | Create an agent runtime with tool registry |
| `UIWorkspace` (optional) | Create a Next.js microfrontend |

### 5.2 Template Inputs

Each template asks for:

| Input | Description |
|-------|-------------|
| `tenant_id` | Optional if platform-only |
| `sku` | Shared / Namespace / Private |
| `primitive_name` | Name of the primitive |
| `primitive_type` | Domain / Process / Agent |
| `target_repo` | Platform vs tenant repo |

### 5.3 Template Outputs

The template computes the **target namespace** per env based on:

* `tenant_id`
* SKU (Shared / Namespace / Private)
* Origin (`platform` vs `tenant`)

Then scaffolds:

* Primitive code skeleton (with SDK, RLS hooks, telemetry, auth middleware)
* Dockerfile / CI skeleton
* Helm chart / K8s manifests
* `catalog-info.yaml` for Backstage

The **Backstage Scaffolder** always publishes into the **right repo** (platform or tenant) and generates manifests pointing at the **right namespace** (`ameide-*`, `tenant-{id}-*-base`, or `tenant-{id}-*-cust`), according to the rules in §4.

### 5.4 Namespace Calculation Logic

```
function calculateNamespace(tenant_id, sku, origin, env):
  if origin == "platform":
    if sku == "Shared":
      return "ameide-{env}"
    else:
      return "tenant-{tenant_id}-{env}-base"
  else:  # origin == "tenant" or "agent"
    return "tenant-{tenant_id}-{env}-cust"
```

---

## 6. E2E Flow: From Idea to Running Primitive

### Step 0 – Requirement Appears

* A human or AI agent sees a need: "Tenant X needs capability Y."
* This is captured through the **Transformation domain** (your "change-the-business" API).

### Step 1 – Design First (Config/Definitions)

* The agent/human works in **Transformation**:
  * Defines / updates `ProcessDefinition` (L2O variant, workflows, SLAs)
  * Defines / updates `AgentDefinition` (tools, prompts, policies)
  * Attaches them to `tenantId`/`orgId`
* If the requirement can be met by **definitions + existing primitives**, it ends here: **no new code, no Backstage**.

### Step 2 – Decide If New Code Is Needed

If existing primitives and definitions can't implement the behaviour:

* Transformation (or a human) decides: "we need a new [Domain/Process/Agent] primitive."
* Transformation validates:
  * Tenant SKU
  * Allowed isolation mode
  * Whether custom code is permitted at all for this tenant

### Step 3 – Create a PrimitiveImplementationDraft

Agent calls a high-level API:

```
CreatePrimitiveImplementationDraft(
  tenant_id,
  primitive_type,
  primitive_id,
  template_id,
  code_origin   = "platform" | "tenant",
  target_repo   = "...",
  design_refs   = [ProcessDefinition/AgentDefinition IDs]
)
```

Transformation:

* Looks up tenant SKU & execution plan
* Calculates proper namespaces per env:
  * Shared + platform code → `ameide-{env}`
  * Any custom/tenant code → `tenant-{id}-{env}-cust`
  * Namespace/private base → `tenant-{id}-{env}-base` (for platform-owned dedicated primitives)
* Creates a `PrimitiveImplementationDraft` artifact with all that info

### Step 4 – Deterministic Worker Calls Backstage

A backend worker (Process primitive) reacts to the draft and:

* Calls Backstage Scaffolder with the right template and values
* Backstage:
  * Generates code & manifests
  * Publishes to the **target repo** (platform or tenant)
  * Opens a branch/PR

The draft is updated with **links to the repo/PR** and Backstage task logs.

### Step 5 – Agent "Writes Code" via the Draft, Not Git

The agent never gets Git or K8s credentials.

Instead, it uses tools like:

| Tool | Purpose |
|------|---------|
| `GetPrimitiveDraftFiles(draft_id)` | Snapshot of files from the feature branch |
| `UpdatePrimitiveDraftFiles(draft_id, patches)` | Apply patches via deterministic worker |

The deterministic worker:

* Applies patches
* Runs lint/tests/security checks
* Commits & pushes to the feature branch

All this is:

* Audited (who/what changed what, for which tenant)
* Logged as **write-intent** events in Transformation

### Step 6 – "Put This in Test" → Promote to Dev

Agent (or human) calls:

```
RequestPromotion(draft_id, env="dev")
```

Transformation worker:

1. Runs CI on the branch (tests, lint, SAST/secret scan, etc.)
2. If green, updates **`tenant-{id}-gitops`** (or platform GitOps repo if platform-owned code) with:
  * Argo `Application`/`HelmRelease` pointing to the primitive
   * Appropriate namespace:
     * Platform code → `ameide-dev`
     * Tenant custom code → `tenant-{id}-dev-cust`
     * Dedicated base → `tenant-{id}-dev-base`
3. Opens a GitOps PR, which may auto-merge for `dev` under policy

ArgoCD's ApplicationSet for that env syncs:

* Controller is deployed into the **dedicated namespace** dictated by the rules
* No custom images in `ameide-dev` ever

### Step 7 – Observe & Iterate

* Agent / human uses Backstage + observability to see:
  * Health, logs, metrics
  * Catalog relations (which definitions use this primitive, which tenants are affected)
* If changes needed, repeat Step 5–6

### Step 8 – Promote to Staging / Prod

Same API, stricter policy:

```
RequestPromotion(draft_id, env="staging")
RequestPromotion(draft_id, env="prod")
```

* Reuses the **same built artifact** (image/tag) that ran in dev
* Only changes GitOps config to:
  * `ameide-staging` vs `ameide-prod`, or
  * `tenant-{id}-staging-{base/cust}` vs `tenant-{id}-prod-{base/cust}`
* Requires approvals appropriate to risk tier and tenant

Backstage & ArgoCD show the rollout status per environment.

---

## 7. Security Invariants

No matter how fancy the flow gets, these hold:

### 7.1 No Custom Code in Shared Namespaces

* Any primitive whose origin is `tenant` (or "untrusted") **cannot** target `ameide-*`
* Validated in Transformation before scaffolding, and in GitOps/Argo policy

### 7.2 Dedicated Namespaces for All Custom Services

* Shared SKU → custom code in `tenant-{id}-{env}-cust`
* Namespace/Private → product runtime in `tenant-{id}-{env}-base`, optional custom in `*-cust`

### 7.3 Only Deterministic Services Touch Git, Backstage, Argo

* Agents only call high-level APIs in Transformation
* Everything side-effectful (scaffold, commit, deploy) happens in predictable primitives

### 7.4 All Calls from Tenant Code via APIs with Tenant-Aware Auth

* No direct DB or internal network access into shared plane
* NetworkPolicies and mTLS + JWT enforce "API/tenantId only"

### 7.5 Alignment with 476 Security Principles

| 476 Principle | 478 Implementation |
|---------------|-------------------|
| §2.1 Tenant isolation first | §7.1, §7.2 namespace isolation |
| §2.2 Deterministic boundary | §7.3 agents don't touch Git directly |
| §2.3 Zero trust between services | §7.4 API-only access with auth |
| §7.3 Agent-generated code governance | §6 Step 5 via draft APIs, not direct Git |

---

## 8. PrimitiveImplementationDraft Artifact

### 8.1 Definition

A `PrimitiveImplementationDraft` is a first-class artifact in the **Transformation Domain** that tracks the lifecycle of creating a new primitive.

### 8.2 Schema

```protobuf
message PrimitiveImplementationDraft {
  string id = 1;
  string tenant_id = 2;
  string primitive_type = 3;  // "domain" | "process" | "agent"
  string primitive_id = 4;
  string template_id = 5;
  string code_origin = 6;      // "platform" | "tenant"
  string target_repo = 7;
  repeated string design_refs = 8;  // ProcessDefinition/AgentDefinition IDs

  // Computed by Transformation
  map<string, string> target_namespaces = 9;  // env -> namespace
  string sku = 10;

  // State
  DraftState state = 11;
  string branch_name = 12;
  string pr_url = 13;
  repeated PromotionRecord promotions = 14;
}

enum DraftState {
  DRAFT_STATE_UNSPECIFIED = 0;
  PENDING = 1;
  SCAFFOLDED = 2;
  IN_REVIEW = 3;
  PROMOTED_DEV = 4;
  PROMOTED_STAGING = 5;
  PROMOTED_PROD = 6;
  REJECTED = 7;
}
```

### 8.3 APIs

| API | Purpose |
|-----|---------|
| `CreatePrimitiveImplementationDraft` | Create a new draft |
| `GetPrimitiveDraftFiles` | Get current files from feature branch |
| `UpdatePrimitiveDraftFiles` | Apply patches via deterministic worker |
| `RequestPromotion` | Promote to an environment |
| `GetDraftStatus` | Get current state and audit trail |

---

## 9. NetworkPolicy Patterns

### 9.1 Shared Namespace (`ameide-{env}`)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-tenant-custom-ingress
  namespace: ameide-dev
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    # Allow from same namespace
    - from:
        - podSelector: {}
    # Allow from gateway
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: argocd
    # Deny from tenant-*-cust namespaces (implicit)
```

### 9.2 Tenant Custom Namespace (`tenant-{id}-{env}-cust`)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-platform-api
  namespace: tenant-acme-dev-cust
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    # Allow to platform API gateway only
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: argocd
      ports:
        - protocol: TCP
          port: 443
    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

---

## 10. Cross-References

### 10.1 Vision Suite Alignment

| Document | Section | Relationship |
|----------|---------|--------------|
| [470-ameide-vision](470-ameide-vision.md) | §4.3 | Backstage as factory – 478 operationalizes |
| [471-ameide-business-architecture](471-ameide-business-architecture.md) | §4, §6 | Business lifecycle – 478 details primitive creation |
| [472-ameide-information-application](472-ameide-information-application.md) | §4 | Backstage catalog – 478 adds tenant repos |
| [473-ameide-technology](473-ameide-technology.md) | §6 | Multi-tenancy – 478 defines namespace topology |
| [475-ameide-domains](475-ameide-domains.md) | §4.6 | Agent primitives – 478 adds tenant isolation |
| [476-ameide-security-trust](476-ameide-security-trust.md) | §8 | Agent governance – 478 adds draft pattern |
| [467-backstage](467-backstage.md) | §10 | Templates – 478 adds namespace calculation |

### 10.2 Implementation Alignment

| Document | Relationship |
|----------|--------------|
| [443-tenancy-models](443-tenancy-models.md) | SKU definitions used in §4 |
| [465-applicationset-architecture](465-applicationset-architecture.md) | GitOps model extended for tenant repos |
| [310-agents-v2](310-agents-v2.md) | Agent primitive runtime for draft APIs |

---

## 11. Open Questions

1. **Tenant repo provisioning**
   * How is `tenant-{id}-controllers` and `tenant-{id}-gitops` created initially?
   * Backstage template? Manual? Transformation API?

2. **Image registry isolation**
   * Do tenant custom images go to a tenant-specific registry path?
   * `ghcr.io/ameideio/tenant-{id}/controller-name:tag`?

3. **Cost allocation**
   * How are compute costs for `tenant-{id}-*-cust` namespaces attributed?

4. **Draft expiry**
   * How long do unpromoted drafts persist before cleanup?

---

## 12. Implementation Status

| Component | Status | Notes |
|-----------|--------|-------|
| Namespace convention | ✅ Defined | §4 |
| Backstage templates | ⏳ Planned | [467-backstage.md](467-backstage.md) §10.1 |
| PrimitiveImplementationDraft | ⏳ Not implemented | §8 schema defined |
| Draft APIs | ⏳ Not implemented | §8.3 |
| NetworkPolicies | ⏳ Not implemented | §9 patterns defined |
| Tenant repo provisioning | ❓ Open question | §11.1 |
