# 477 – Primitive Stack, Operators & GitOps Layout

**Status:** Draft v1
**Audience:** Platform engineering, architecture, internal agents, AI coders
**Goal:** Make it unambiguous *what lives where* for the Ameide six primitives (Domain, Process, Agent, UISurface, Projection, Integration), their operators, and GitOps CRDs – so humans and AI agents don't fight the repo layout.

> **Context:**
>
> * Platform vision & primitives: 470 / 4xx combined
> * Information & apps (CRDs, proto chain): 472
> * Tech stack & operators: 473
> * Refactoring to primitives + CRDs: 474
> * CLI & scaffolding: 484
> * Vendor transformation methodology contrast: `526-transformation-vendor-methodology-comparison.md`
> * IT4IT value-stream mapping (R2D/R2F/D2C lens): `525-it4it-value-stream-mapping.md`

---

## Grounding & contract alignment

- **Placement contract:** Concretizes the design/deploy/runtime split from `470-ameide-vision.md`, `472-ameide-information-application.md`, `473-ameide-technology.md`, and `475-ameide-domains.md` into a precise repo and GitOps layout for primitives, operators, and tenant repos.  
- **Operator & CLI dependencies:** Provides the layout that operator/vertical-slice backlogs (`495-ameide-operators.md`, `497-operator-implementation-patterns.md`, `498-domain-operator.md`, `499-process-operator.md`, `500-agent-operator.md`, `501-uisurface-operator.md`, `502-domain-vertical-slice.md`, `503-operators-helm-chart.md`, `504-agent-vertical-slice.md`) and CLI backlogs (`484a-484f`) assume for code, CRDs, and GitOps.  
- **Scrum/agent grounding:** Defines where the Transformation Scrum domain, Scrum Process primitives, and AmeidePO/AmeideSA/AmeideCoder artifacts live in the repos, which the Scrum stack (`367-1-scrum-transformation.md`, `506-scrum-vertical-v2.md`, `508-scrum-protos.md`, `505-agent-developer-v2*.md`, `507-scrum-agent-map.md`) relies on when describing Stage 0–3 behavior.

## 1. Design ≠ Deploy ≠ Runtime (recap)

We standardise on the same 3-step chain everywhere:

1. **Design-time (Transformation Domain)**

   * ProcessDefinitions, AgentDefinitions, ExtensionDefinitions, Backstage template configs.
   * Stored as domain data in the **Transformation Domain primitive**, surfaced via transformation design tooling UIs.

2. **Deployment-time (Git & factories)**

   * Backstage templates and `buf generate` turn design artifacts/contracts into:

     * Primitive **code** (Domain/Process/Agent/UISurface/Projection/Integration) in the core monorepo.
     * Primitive **CRs** (`Domain`, `Process`, `Agent`, `UISurface`, `Projection`, `Integration`) committed to GitOps repos.

3. **Runtime (Kubernetes + operators)**

   * Ameide operators watch primitive CRDs and reconcile them into Deployments, Services, Temporal workers, DB schemas, etc.

**Key invariant:**
*App-level CRDs exist **only** for the primitive kinds; they describe **how to run** code, not the business model itself.*

---

## 2. Where things live – high level

### 2.1 Core monorepo (`ameideio/ameide`)

Holds everything that is *code*, owned by Ameide:

```text
ameide/                               # github.com/ameideio/ameide
├── packages/
│   ├── ameide_core_proto/            # Buf module, proto contracts
│   ├── ameide_core_cli/              # 484: codegen/verification CLI
│   ├── ameide_sdk_go/                # Generated SDKs
│   ├── ameide_sdk_ts/
│   └── ameide_sdk_python/
│
├── primitives/
│   ├── domain/
│   ├── process/
│   ├── agent/
│   ├── uisurface/
│   ├── projection/
│   └── integration/
│
├── operators/
│   ├── domain-operator/
│   ├── process-operator/
│   ├── agent-operator/
│   ├── uisurface-operator/            # see §3.4
│   ├── projection-operator/           # v2 (planned)
│   └── integration-operator/          # v2 (planned)
│
└── service_catalog/
    └── backstage/                    # internal factory templates
```

* Proto modules live under `packages/ameide_core_proto`.
* Primitive code lives under `primitives/*` (Domain/Process/Agent/UISurface/Projection/Integration).
* Operators are normal Go/TS services built & released like any other platform component.

### 2.2 Platform GitOps repo (`ameideio/ameide-gitops`)

Git is **only** desired state here – no application code, no operator implementation.

```text
ameide-gitops/                        # github.com/ameideio/ameide-gitops
├── clusters/
│   ├── prod/
│   │   ├── apps/                     # ArgoCD ApplicationSets
│   │   └── infra/                    # CNPG, ingress, etc.
│   └── staging/
│       └── ...
└── primitives/
    ├── domain/
    ├── process/
    ├── agent/
    └── uisurface/
```

* `primitives/` holds **primitive CRs** and Helm/Kustomize overlays, mirroring the `primitives/*` code tree.
* Argo CD syncs this repo into `ameide-{env}` namespaces; operators reconcile CRDs into workloads.

### 2.3 Tenant GitOps repos

Per-tenant repos hold custom primitives and config:

```text
tenant-{id}-gitops/
└── primitives/
    ├── domain/
    ├── process/
    ├── agent/
    ├── uisurface/
    ├── projection/
    └── integration/
```

* Platform repo **never** contains tenant-specific code; it links to `tenant-{id}-controllers` + `tenant-{id}-gitops` via Backstage locations.
* Namespace/SKU strategy for tenant planes is defined in 478.

---

## 3. Primitive vs operator vs GitOps – responsibility split

For each primitive kind we keep a hard boundary:

> **Primitive code** = business logic
> **Operator** = infra automation + rollout rules
> **CRD / GitOps** = environment- and tenant-specific runtime config

### 3.1 Domain primitive

**Domain primitive (code, in `ameideio/ameide: primitives/domain/{name}`)**

* Owns bounded context data, invariants, migrations, events.
* Exposes proto-first APIs via generated SDKs.

**Domain operator (code, in `operators/domain-operator`)**

* Watches `Domain` CRDs.
* Reconciles into:

  * Deployments, Services, HPAs.
  * CNPG schemas/users, migration Jobs.
  * NetworkPolicies, ServiceMonitors, standard sidecars/env vars.
* Enforces namespace/SKU placement and image policies.

**Domain CRD (YAML, in `*-gitops/primitives/domain/{name}`)**

* Declares:

  * `spec.image`, resources, env, secrets references.
  * DB binding (cluster, schema).
  * Observability & policy toggles.
* **Never** describes tables/fields; that stays in migrations & code.

### 3.2 Process primitive

**Process primitive (code, `primitives/process/{name}`)**

* Temporal workflows + activities implementing a ProcessDefinition.

**Process operator (code, `operators/process-operator`)**

* Watches `Process` CRDs.
* Reconciles into:

  * Temporal worker Deployments.
  * ConfigMaps mapping BPMN ids → workflow/activities.
  * ServiceMonitors, optional CronJobs for compensations.
* Validates references to ProcessDefinitions and Temporal namespaces.

**Process CRD (YAML, `*-gitops/primitives/process/{name}`)**

* Declares:

  * Image & version.
  * Temporal namespace and queue.
  * Linked ProcessDefinition version.
  * Timeouts, retries, rollout policy.

### 3.3 Agent primitive

**Agent primitive (code, `primitives/agent/{name}`)**

* Implements an AgentDefinition: tools, prompts, orchestration graph, risk tier.

> **V2 alignment:** [505-agent-developer-v2.md](505-agent-developer-v2.md) defines three standard Agent primitives that live in this tree:
> - **AmeidePO** (`runtime_role=product_owner`) – product decisions only, no repo/tool access.
> - **AmeideSA** (`runtime_role=solution_architect`) – technical decomposition + A2A client.
> - **AmeideCoder** (`runtime_role=a2a_server`) – devcontainer service exposing the A2A REST binding (`/v1/message:send`, `/v1/message:stream`, `/v1/tasks/*`).
> Process primitives remain separate (Temporal workers), but they orchestrate PO/SA via EDA events and never host agents themselves.

**Agent operator (code, `operators/agent-operator`)**

* Watches `Agent` CRDs.
* Reconciles into:

  * Agent runtime Deployments.
  * Secrets for LLM/model providers + tool credentials.
  * Tool grant policies, logging/metrics wiring.

**Agent CRD (YAML, `*-gitops/primitives/agent/{name}`)**

* Declares:

  * Image & model/provider config.
  * Allowed tool set (Domain/Process APIs).
  * Risk tier and policy set.
* Links back to an AgentDefinition id in the Transformation Domain.

### 3.4 UISurface primitive

**UISurface primitive (code, `primitives/uisurface/{name}`)**

* Next.js app (workspaces + process views) using Ameide SDKs.

**UISurface operator (code, `operators/uisurface-operator`)**

We extend the "three operator" picture to four, matching all primitives:

* Watches `UISurface` CRDs.
* Reconciles into:

  * Next.js Deployment/Service, HPA.
  * Ingress/Gateway routes (domains/paths).
  * Env config to point at the correct Domain/Process/Agent endpoints.
  * Auth scopes → Keycloak/IdP client configs (via infra CRs).

This is a **thin infra operator**, *not* a layout engine:

* It doesn't understand UI components or forms.
* It only ensures that a given UISurface image is reachable and correctly wired to primitives.

**UISurface CRD (YAML, `*-gitops/primitives/uisurface/{name}`)**

* Declares:

  * Image, env, and rollout settings.
  * Routing (hosts, paths).
  * Required scopes/permissions.
  * References to Domain/Process/Agent primitives it depends on.

---

## 4. Where primitive CRDs live (direct answer)

**Short version:**

* Primitive CRDs live **only in GitOps repos** (platform or tenant).
* Operators live in the **core monorepo** (`ameideio/ameide`) and are deployed via GitOps charts, but their *implementation* never lives in GitOps.

### 4.1 Platform primitives

* Multi-tenant platform primitives (Identity, Transformation, L2O, etc.) have CRDs in `ameideio/ameide-gitops: primitives/*`.
* Namespaces:

  * Shared platform plane: `ameide-{env}` for shared multi-tenant services.
  * SKU/tenant planes per 478 for isolating heavy or custom workloads.

### 4.2 Tenant primitives

* Tenant-specific primitives have CRDs in `tenant-{id}-gitops/primitives/*` (Domain/Process/Agent/UISurface/Projection/Integration).
* Backed by tenant-specific controllers (`tenant-{id}-controllers`) when custom code is needed.

**No CRDs live in the core monorepo.** The core repo only references CRD *schemas* and sample manifests in docs/tests.

---

## 5. CLI + TDD as guardrails for the stack

Per 484, the **Ameide CLI** is the AI/human guardrail for primitives:

### 5.1 What CLI scaffolds

For each primitive kind, CLI emits *paired* outputs:

```text
primitives/domain/{name}/           # code
gitops/primitives/domain/{name}/    # runtime config (values.yaml/kustomization)
```

(and the same for `process/`, `agent/`, `uisurface/`).

Each scaffold includes:

* **Code skeletons** (handlers, workflows, agent tools, Next.js app).
* **Failing tests** (TDD-style) that describe expected behavior in machine-readable names/messages.
* **GitOps skeleton** (`values.yaml`, `kustomization.yaml`) that match the CRD shape for that primitive.

### 5.2 How AI coders use it

* AI agents run inside a **devcontainer sandbox**, interacting only with the filesystem and the CLI (no live cluster required).
* Typical sequence:

  1. Agent loads **Transformation Domain** context (ProcessDefinitions, AgentDefinitions, etc.).
  2. Agent **researches standard solutions** – searches for industry patterns, existing libraries, and well-established approaches that solve similar problems; normalizes the implementation approach to avoid reinventing the wheel.
  3. **Human review gate** – agent presents research findings and proposed approach for human approval before proceeding with proto changes.
  4. Agent proposes/edits proto contracts (informed by approved research).
  5. Agent runs `buf` to regenerate SDKs.
  6. Agent uses Ameide CLI to scaffold/refresh primitive code + GitOps skeletons.
  7. Agent implements business logic until tests pass.
  8. Agent updates GitOps CR/values to point to the new image/tag.
  9. **Human UAT** – human verifies the implementation meets requirements and accepts the deliverable.
* CLI output is deliberately **descriptive** so that:

  * The agent can treat "list of created files + tests + TODOs" as a plan.
  * The same text can be pasted into prompts as context.

---

## 6. Mapping table (one glance)

| Concern                         | Primitive code (core repo)                      | Operators (core repo)                                                | CRDs & GitOps (platform/tenant repos)             |
| ------------------------------- | ----------------------------------------------- | -------------------------------------------------------------------- | ------------------------------------------------- |
| Business rules & invariants     | ✅ Domain                                        | ❌ (no business logic)                                                | ❌                                                 |
| Long-running orchestration      | ✅ Process (Temporal workflows)                  | ❌                                                                    | Ref to ProcessDefinition + Temporal ns            |
| AI reasoning / LLM loops        | ✅ Agent                                         | ✅ Enforce tool grants + secrets                                      | Risk tier & runtime config                        |
| UI layout & navigation          | ✅ UISurface (Next.js)                           | ❌ (only routes & auth)                                               | Routing + auth scopes                             |
| DB schemas & migrations         | ✅ Migrations in domain code                     | ✅ Run migrations, create schemas/users                               | DB binding & environment                          |
| Multi-tenant placement/quotas   | ❌                                               | ✅ Schedules workloads to proper namespace/SKU; sets resource classes | Tenant/SKU flags on CRDs                          |
| Image tags & rollout strategies | ❌                                               | ✅ Enforce policies (e.g., canary, pause on `MigrationFailed`)        | Exact images & rollout policy (Git is truth)      |
| Environment differences         | ❌ (same code)                                   | ✅ Reacts to different CR specs per env                               | Per-env overlays (prod/staging/etc.)              |
| Tenant-specific behavior        | Tenant controllers / WASM extensions (Tier 1/2) | ✅ Enforce isolation & quotas                                         | Tenant-specific CRs, namespaces, extension wiring |

---

## 7. Open questions / next iterations

1. **UISurface operator implementation detail**

   * Ensure 473 §3.3.5 covers all primitive kinds (including Projection/Integration) to match this layout doc.

2. **Repo layout vs multi-repo reality**

   * 484 shows `primitives/` and `gitops/` in a single tree for simplicity; in production we map those logical paths to separate physical repos (`ameideio/ameide` vs `ameideio/ameide-gitops`).

3. **AmeideTenant CRD & primitive CRs**

   * Ensure `AmeideTenant` (infra plane) composes cleanly with primitive CRs and namespace/SKU strategy from 478.

---

## 8. Cross-references

| Backlog | Relationship |
|---------|--------------|
| [467-backstage](467-backstage.md) | Backstage templates scaffold primitives via this stack |
| [472-information-and-apps](472-information-and-apps.md) | CRD chain and proto contracts |
| [473-ameide-technology](473-ameide-technology.md) | Operator implementation details |
| [474-refactoring-to-primitives](474-refactoring-to-primitives.md) | Migration plan to this layout |
| [478-ameide-extensions](478-ameide-extensions.md) | Tenant namespace/SKU strategy |
| [484-ameide-cli](484-ameide-cli.md) | CLI commands that scaffold primitives |
| [495-ameide-operators](495-ameide-operators.md) | Operator CRD shapes, reconciliation logic, publishing & deployment |
