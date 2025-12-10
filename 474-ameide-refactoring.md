# 474 – Refactoring & Migration Plan (Domain / Process / Agent / UISurface)

**Status:** Draft v1
**Audience:** Platform & product engineering, architecture, internal agents

This guide describes **how we move from the current service topology and legacy concepts to the Domain / Process / Agent / UISurface primitives and their CRDs**, while staying **code-first and AI-first**. It complements the vision and architecture docs in the 470–480 range.

> **Core Invariants**: See [470-ameide-vision.md §0 "Ameide Core Invariants"](470-ameide-vision.md) for the canonical list (four primitives, Graph read-only, Transformation as domain, proto chain, tenant isolation, Backstage internal).
>
> **Cross-References**:
> - [471 §4](471-ameide-business-architecture.md) – Backstage as the platform factory (business view)
> - [472 §2](472-ameide-information-application.md) – Primitive definitions (Domain/Process/Agent/UISurface)
> - [472 §2.7](472-ameide-information-application.md) – Contract → Implementation → Runtime chain
> - [473 §4](473-ameide-technology.md) – Backstage implementation as factory
> - [475 §4](475-ameide-domains.md) – Domain portfolio and E2E processes
> - [477 §5.2](477-primitive-stack.md) – AI coder workflow with human review gates
> - [478](478-ameide-extensions.md) / [479](479-ameide-extensibility-wasm.md) / [480](480-ameide-extensibility-wasm-service.md) – Tier 1/Tier 2 extensibility model

> **Reviewer Checklist – Deprecated Terms**
>
> Push back on PRs/docs that introduce these legacy terms:
>
> | Deprecated | Use Instead |
> |------------|-------------|
> | IPA (Intelligent Process Automation) | Domain + Process + Agent bundle |
> | IDC (IntelligentDomainController) | Domain CRD |
> | IPC (IntelligentProcessController) | Process CRD |
> | IAC (IntelligentAgentController) | Agent CRD |
> | AgentRuntime | Agent primitive |
> | WorkflowService | Process primitive |
> | DomainService | Domain primitive |
> | "Transformation tooling service" | Transformation design tooling (UI only) |
> | "Agent domain" | Transformation Domain (owns AgentDefinitions) |
> | "component domain" | Deployment category |

---

## 1. Objectives

### 1.1 What we’re aiming for

1. **Code-first, AI-first**

   * The **canonical representation** of the system is **plain code**: modules, functions, types, tests.
   * AI and humans both operate on this code directly (with help from design artifacts), not on a giant metadata layer.
   * Metadata exists only where it helps AI and operations; it never replaces code.

2. **Four primitives only**

   We converge on **four application primitives**:

   * **Domain** – owns business state and rules for a bounded context.
   * **Process** – orchestrates multiple Domains and Agents into an end-to-end outcome.
   * **Agent** – non-deterministic worker using typed tools.
   * **UISurface** – user-facing entry point (workspace or process view).

   Each primitive has two faces:

   * A **Primitive** – the code: packages, modules, tests.
   * A **CRD** – a declarative runtime object on Kubernetes (Domain CRD, Process CRD, Agent CRD, UISurface CRD).

3. **CRDs for runtime wiring, not for modeling the business**

   * App-level CRDs exist **only** for the four primitives and a small number of infra objects (tenant, DB, etc.).
   * We **never** introduce CRDs for tables, fields, forms, or other fine-grained constructs.

4. **Not a traditional metadata-driven platform**

   * We **deliberately avoid** the pattern where a giant metadata system is the “real” model and code is just glue written by human developers.
   * Here, **code *is* the model**. Design-time artifacts and CRDs are hints that keep AI and ops sane, not a meta-runtime.

5. **Backward-compatible, incremental migration**

   * We keep systems running while we gradually re-shape them.
   * Existing services are wrapped and renamed where possible, not rewritten from scratch on day one.

---

## 2. Target model (quick recap)

This section restates the target model in the new vocabulary so the rest of the guide can be very literal. For full definitions, see [472 §2](472-ameide-information-application.md) and [475 §4](475-ameide-domains.md).

### 2.1 Primitives

**Domain primitive**

* Code that owns:

  * A bounded business concept (e.g. Orders, Transformation, Product).
  * Its persistence schema and migrations.
  * Its invariants and events.
* Exposes a stable API (proto or equivalent) for other primitives and surfaces.

**Process primitive**

* Code that orchestrates:

  * Calls to Domain primitives.
  * Interactions with Agent primitives.
  * Lifecycle of a business instance (e.g. “this opportunity from lead to order”).

**Agent primitive**

* Code that:

  * Accepts typed requests.
  * Uses tools that correspond to Domain and Process APIs.
  * Emits typed proposals or decisions.

**UISurface primitive**

* Code for user-facing apps:

  * Workspaces (entity-centric).
  * Process views (stage/board/timeline).
* Uses generated SDKs / typed clients for Domain, Process, Agent primitives.

### 2.2 CRDs

For each primitive, there is **at most one CRD type**. For operator design, see [473 §3](473-ameide-technology.md).

* **Domain CRD** – describes how a Domain primitive is run:

  * image, config, resources, DB bindings, observability.
* **Process CRD** – describes how a Process primitive is run:

  * image, process type, queue/topic bindings, timeouts.
* **Agent CRD** – describes how an Agent primitive is run:

  * image, model config, tool grants, risk tier.
* **UISurface CRD** – describes how a UISurface primitive is run:

  * image, routing, auth scopes, feature flags.

Operators reconcile these CRDs into Deployments/Services/etc., but **the CRDs never describe tables, fields or layout**.

### 2.3 Contract → Implementation → Runtime chain

Every primitive participates in the contract chain described in [472 §2.7](472-ameide-information-application.md):

1. **Proto module** (`packages/ameide_core_proto`) defines the API contract.
2. **Generated SDKs** (Go/TS/Python) are produced from proto via Buf.
3. **Service implementation** pins to proto version via `go.mod` / `package.json`.
4. **Docker image** tagged and labelled with proto version metadata.
5. **GitOps manifest** references exact image tag + proto annotations.
6. **Backstage catalog** links service to proto module/version.

Refactors must preserve this chain: proto version bumps trigger SDK regen, image rebuilds, and GitOps manifest updates.

---

## 3. Current state (very high-level)

Today we have:

* A set of **domain-ish services**:

  * own their own schema and APIs (e.g. platform, identity, transformation, graph, various business services).
* A set of **workflow-ish services**:

  * implement long-running flows in code, but are not yet framed as Process primitives.
* A set of **agent-ish services**:

  * provide LLM-driven helpers using internal tool stacks.
* **UI apps**:

  * the main platform UI and adjunct tools.
* A number of **legacy concepts** (IPA, “platform workflows”, legacy controller naming) still present in code and docs.

The goal is to **relabel and reshape** these into the four primitives + CRDs without pausing the platform.

---

## 4. Refactoring principles

These rules should be applied ruthlessly across code, docs and tooling.

### 4.1 Naming

1. **Use `{Domain/Process/Agent/UISurface} primitive` for code**

   * Example: “Sales Domain primitive”, “Onboarding Process primitive”, “Transformation Agent primitive”.

2. **Use `{Domain/Process/Agent/UISurface} CRD` for runtime objects**

   * Example: “Sales Domain CRD”, “L2O Process CRD”, “Transformation Agent CRD”, “Sales UISurface CRD”.

3. **Do not use old controller names in new work**

   * Avoid `DomainController`, `ProcessController`, `AgentController`.
   * Avoid `Intelligent*`, `IDC`, `IPC`, `IAC`.
   * When touching old code, move it incrementally to the new naming.

### 4.2 Code-first, AI-first

> **AI coder workflow**: See [477-primitive-stack.md §5.2](477-primitive-stack.md) for the canonical AI coder sequence with human review gates (research → human approval → proto → scaffold → implement → UAT).

4. **Plain code is the source of truth**

   * Each primitive is a **module or package** with:

     * Types/entities.
     * Public API.
     * Tests.
   * Design-time diagrams, processes, and specs are commentary around the code, not executable metadata.

5. **AI operates on the same code as humans**

   * Refactoring and extension work is expressed as:

     * "Change this Domain primitive in these ways."
     * "Introduce a new Process primitive between Sales and Orders."
   * We avoid any separate DSL or meta language that only AI understands.
   * AI coders follow the human-gated workflow in 477 §5.2: research standard solutions, present findings for approval, then proceed with implementation.

### 4.3 Minimal metadata

6. **Use CRDs only for runtime configuration**

   * CRDs should contain:

     * References to the code artifact (image/tag, or build ref).
     * Operational settings (resources, SLOs, rollout strategy).
   * CRDs should **not** try to encode:

     * Entity schemas.
     * Field properties.
     * UI layouts.
     * Business rules.

7. **Design artifacts are guidance, not runtime**

   * BPMN diagrams, architecture diagrams, written specs:

     * Live in the Transformation area (or equivalent).
     * Help humans and agents understand the system.
     * Never act as the runtime model of truth.

### 4.4 Gradual, reversible changes

8. **Wrap before rewrite**

   * First give each area a clear primitive name (Domain, Process, Agent, UISurface).
   * Introduce CRDs that point at the existing deployments.
   * Only then start splitting or reshaping code.

9. **Keep behavior identical per step**

   * A refactor step is “done” only when:

     * External APIs are unchanged.
     * Observability and SLA coverage are at least as good as before.

---

## 5. Mapping old to new (conceptual)

We avoid naming legacy types explicitly, but we still need a mental map:

* **Any service that owns business state + rules + schema**
  → becomes a **Domain primitive**, with a **Domain CRD** for runtime.

* **Any service that orchestrates cross-domain flows over time**
  → becomes a **Process primitive**, with a **Process CRD** for runtime.

* **Any service that wraps an LLM with tools and policies**
  → becomes an **Agent primitive**, with an **Agent CRD** for runtime.

* **Any UI app that presents workspaces or process views**
  → becomes a **UISurface primitive**, with a **UISurface CRD** for runtime.

The rest of this guide assumes we can identify current services that match these shapes and rename them accordingly.

---

## 6. Step-by-step refactoring plan

> **Extensibility context**: Phases 0–2 establish the primitive foundation. Tenant-specific extensions (Tier 1 WASM, Tier 2 primitives) layer on top of this foundation—see [478](478-ameide-extensions.md), [479](479-ameide-extensibility-wasm.md), [480](480-ameide-extensibility-wasm-service.md) for how extensions interact with the primitives being refactored here.

### 6.1 Phase 0 – Foundations

**Goal:** Create the new vocabulary and minimal tooling without moving any behavior.

1. **Introduce primitive types in the codebase**

   * Create top-level packages/modules for:

     * `domain/`
     * `process/`
     * `agent/`
     * `uisurface/`
   * Move *only* type definitions and interfaces at first; keep implementations where they are.

2. **Introduce the four CRD kinds**

   * Define:

     * `Domain` CRD.
     * `Process` CRD.
     * `Agent` CRD.
     * `UISurface` CRD.
   * For now, fields are minimal: name, image, version, environment, references to infra.

3. **Backfill CRDs for existing deployments**

   * For each running service:

     * Create the appropriate `{Primitive} CRD` pointing to the existing image/tag.
     * Deploy CRDs side-by-side with current manifests.
   * Operators should reconcile CRDs **without** changing behavior yet (or run in “observe only” mode).

**Exit criteria:**

* Four CRD types live in the cluster and are applied through Git.
* Each important service is represented by exactly one primitive CRD.

---

### 6.2 Phase 1 – Code alignment: Domain primitives

**Goal:** Make domain services explicit Domain primitives.

1. **Create Domain primitive modules**

   * For each domain-ish service:

     * Add a `domain/<name>/` module.
     * Move or alias:

       * Entities and value objects.
       * Business rule functions.
       * Public API types.

2. **Tie Domain CRD to Domain primitive**

   * Ensure each Domain CRD has:

     * A `spec.primitive` field referencing the Domain primitive (e.g. by package name or logical id).
     * A reference to its database or schema.

3. **Standardise patterns across Domains**

   * Consistent:

     * Logging, tracing.
     * Error semantics.
     * Tenant/org scoping.

4. **Update docs & ADRs**

   * All references to “domain services” or legacy naming become **Domain primitive + Domain CRD** language.

**Exit criteria:**

* Each core business domain has:

  * A `domain/<name>` module.
  * A Domain CRD that is the authoritative deployment description.

---

### 6.3 Phase 2 – Code alignment: Process primitives

**Goal:** Elevate orchestration logic into explicit Process primitives.

1. **Identify candidate processes**

   * L2O, O2C, onboarding, transformation cycles, etc.
   * For each, list the domain interactions.

2. **Create Process primitive modules**

   * Add `process/<name>/` modules.
   * Move orchestration code here:

     * Sequences/calls across domains.
     * State models and transitions (if any, outside the engine).

3. **Tie Process CRDs to Process primitives**

   * For each Process primitive:

     * Create a Process CRD specifying:

       * Which Process primitive it runs.
       * How it is scheduled (queues/topics).
       * Timeouts, retry policies.

4. **Align UI**

   * Update UISurfaces to treat process instances as first-class:

     * Process boards, timelines, SLA views.

**Exit criteria:**

* Key flows run through named Process primitives, configured via Process CRDs.
* Orchestration logic is no longer hidden in ad-hoc application services.

---

### 6.4 Phase 3 – Code alignment: Agent primitives

**Goal:** Treat LLM workers as first-class Agent primitives.

1. **Create Agent primitive modules**

   * Add `agent/<name>/` modules.
   * Centralise:

     * Prompts and system instructions.
     * Tool definitions (typed wrappers over Domain/Process APIs).
     * Policies and risk tiers.

2. **Tie Agent CRDs to Agent primitives**

   * Agent CRD contains:

     * Reference to the Agent primitive.
     * Model + provider config.
     * Allowed tools (by Domain/Process).

3. **Refactor call sites**

   * Replace ad-hoc HTTP/gRPC calls to “some agent service” with:

     * A typed client for the specific Agent primitive.
   * This ensures:

     * Clear boundaries between deterministic code and AI-driven behavior.

**Exit criteria:**

* All AI helpers are catalogued as Agent primitives, with Agent CRDs and clear tool grants.

---

### 6.5 Phase 4 – Code alignment: UISurfaces

**Goal:** Make the UI surfaces explicit UISurface primitives.

1. **Identify UISurfaces**

   * Main platform app, plus any separate portals.
   * Group them by:

     * Process views.
     * Domain workspaces.

2. **Create UISurface primitive modules**

   * Add `uisurface/<name>/` modules.
   * Centralise:

     * Navigation structure.
     * Data hooks (SDK usage).
     * Authorization model.

3. **Introduce UISurface CRDs**

   * For each UISurface primitive:

     * Create UISurface CRD with:

       * Routing (paths, domains).
       * Required scopes/permissions.
       * References to the Domain/Process/Agent primitives it depends on.

4. **Align user journeys**

   * Ensure the “process-first, workspaces-second” experience is visible in the code structure:

     * Process views live under Process-oriented UISurfaces.
     * Entity tables/forms live under domain-oriented UISurfaces.

**Exit criteria:**

* Each major UI is represented by a UISurface primitive and CRD.
* Platform navigation is obviously aligned with Domain and Process primitives.

---

### 6.6 Phase 5 – Clean-up & de-legacy

After the primitives exist, we can remove older concepts.

1. **Remove legacy names from the codebase**

   * Delete or rename:

     * IPA terminology.
     * Old “Workflow” or “AgentRuntime” types that duplicate Process/Agent.
   * Update package paths and imports.

2. **Update documentation**

   * Replace:

     * Legacy diagrams and labels with Domain/Process/Agent/UISurface primitives.
   * Ensure 470–475 and related docs reference the new names consistently.

3. **Simplify CRDs**

   * Once operators are stable, review CRD schemas:

     * Remove fields that leak implementation detail.
     * Keep only what truly belongs in runtime configuration.

4. **Tighten boundaries**

   * Enforce:

     * “Domains don’t orchestrate long-running flows” (that’s Process).
     * “Agents never own durable state” (state belongs to Domain/Process).
     * “UISurfaces never bypass primitives to hit storage directly.”

---

## 7. Refactoring workstreams

To keep the plan manageable, organize work as parallel streams:

1. **Domain stream**

   * Enumerate domains.
   * Create Domain primitives and CRDs.
   * Standardize patterns.

2. **Process stream**

   * Prioritize key processes (Onboarding, L2O, O2C, Transformation).
   * Create Process primitives and CRDs.
   * Align UISurfaces and metrics.

3. **Agent stream**

   * Catalog existing agents.
   * Create Agent primitives and CRDs.
   * Harden policies and tooling.

4. **UISurface stream**

   * Catalog UI apps.
   * Create UISurface primitives and CRDs.
   * Align journeys with primitives.

5. **Ops & CRD stream**

   * Build operators and pipelines around the new CRDs.
   * Ensure Git is the source of truth for all CRDs.

6. **Documentation & language stream**

   * Sweep docs, diagrams, and onboarding materials.
   * Enforce the “four primitives + CRD” vocabulary everywhere.

---

## 8. Risks & mitigations

* **Risk:** Fragmented terminology persists in the codebase and docs.  
  **Mitigation:** Introduce a linter/checklist for naming, and make PR review enforce primitive + CRD vocabulary.

* **Risk:** Over-eager metadata creep into CRDs.  
  **Mitigation:** Aggressively review CRD schemas; any attempt to encode tables/fields/forms into CRDs must be treated as an architecture decision.

* **Risk:** AI tooling implicitly re-creates a metadata universe.  
  **Mitigation:** Keep AI agents working on **code + small CRDs**, not on a separate model. If an internal DSL appears, document why and how it stays small.

* **Risk:** Refactors stall due to fear of behavior changes.  
  **Mitigation:** Always start by **relabeling and wrapping** existing behavior as primitives, then only change internals with strong tests and metrics.

---

## 9. Success criteria

We consider this refactor “done enough” when:

1. **Every major service is a primitive**

   * Each has a clear classification: Domain, Process, Agent, or UISurface.
   * Each is configured via exactly one `{Primitive} CRD`.

2. **Code, docs, and CRDs share the same language**

   * No stray references to legacy controller names.
   * Internal onboarding materials explain the platform purely in terms of the four primitives.

3. **AI agents can evolve the system via primitives and CRDs**

   * Agents can:

     * Propose changes to Domain/Process/Agent/UISurface primitives in code.
     * Propose changes to corresponding CRDs for deployment.
   * There is no separate metadata layer they have to learn.

4. **We clearly differentiate from traditional metadata-driven systems**

   * Internally and externally we can say:

     * “Our model is code, not a giant metadata repository. Metadata and CRDs help us operate it, but they are not the application.”

---
