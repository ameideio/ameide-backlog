# 470.C – Ameide Code‑First / AI‑First ERP vs. Oracle Fusion Cloud ERP (SaaS + Configuration‑Driven)

**Status:** Draft  
**Owner:** Architecture / Product  
**Companion:** [470-ameide-vision.md](470-ameide-vision.md)  
**Related:** [520-primitives-stack-v2.md](520-primitives-stack-v2.md) (normative primitives stack v2)

## 1. Purpose

This document describes the objectives of the **Ameide architecture** relative to **Oracle Fusion Cloud ERP’s SaaS, configuration‑driven model** (setup/configuration, constrained extension points, vendor‑managed upgrades, and integration via suite APIs and integration tooling).

We are **not** trying to rebuild Oracle Fusion inside Kubernetes. Instead, we want:

* ERP‑class coherence and “one system” discoverability,
* with a **code‑first, proto‑centric, AI‑operated** core,
* and a **thin operational metadata layer** focused on deployment/runtime only.

---

## 2. Two paradigms in one sentence each

**Oracle Fusion Cloud ERP:**

> A vendor‑managed SaaS suite where much of “implementation” is **configuration + constrained extensions** over a closed core, with upgrades delivered continuously by the vendor and adaptation happening through setup tasks, personalization, supported extensibility, and integrations.

**Ameide:**

> A **code‑first, proto‑centric platform** where business capability is implemented as code in a single repo, operated via **six primitives** as Kubernetes CRDs: **Domain / Process / Agent / UISurface / Projection / Integration**; an **AI agent** is the primary developer that edits code/specs and generates deployable artifacts.

---

## 3. Ameide approach – high‑level objectives (recap)

1. **Code‑first, AI‑friendly representation**
   * Business structure and behavior are primarily **code + proto contracts**, not a configuration universe.

2. **Thin operational metadata layer**
   * CRDs exist only for infra‑relevant units (`Domain`, `Process`, `Agent`, `UISurface`, `Projection`, `Integration`).
   * CRDs are **operational only** (operators reconcile lifecycle/config); behavior remains **code + proto**, with `buf generate` as the canonical generator runner and CI “regen-diff” guardrails.

3. **Derived system graph for “what is running?”**
   * The Knowledge Graph is a **read‑only projection** computed from code/proto/CRDs, not a hand‑maintained model store.

4. **Extensions are code modules**
   * Extensions ship as code + a small manifest and follow explicit extension points (interfaces/events/hooks), rather than a large configuration tree.

---

## 4. Quick overview: Oracle Fusion’s adaptation surfaces (what we’re not recreating)

Oracle Fusion implementations commonly rely on:

* **Configuration / setup**: functional setup tasks and business configuration that shape behavior.
* **Personalization**: user/role personalization of pages and experiences (within policy limits).
* **Supported extensibility**: extending objects and behaviors via vendor-supported mechanisms rather than changing core code.
* **Workflow / approvals**: configurable approval rules and processes for business documents.
* **Integrations**: suite APIs plus integration tooling/adapters to connect finance, procurement, HR, CRM, and external systems.
* **Vendor‑managed upgrades**: the core is upgraded on a schedule; customizations must survive upgrades within supported boundaries.

---

## 5. Oracle Fusion vs Ameide – objective‑level comparison

### 5.1 Source of truth

**Oracle Fusion**

* Behavior emerges from vendor core + configuration, with extensions living in supported mechanisms and integrations.
* “What is running” is distributed across tenant configuration, extension artifacts, and integration definitions.

**Ameide**

* **Code + proto** are the source of truth; Git carries desired state via primitive CRDs.
* “What is running?” is discoverable via derived graphs from repo + cluster state.

---

### 5.2 Change workflow: configuration projects vs code + generated contracts

**Oracle Fusion**

* Change is largely configuration-first; deeper behavior changes often become side-by-side extensions and integrations.

**Ameide**

* Default change mechanism is **edit proto/code → `buf generate` → implement until tests pass → GitOps rollout**.
* Integrations are an explicit primitive (`Integration`), not a separate “integration product category”.

---

### 5.3 Extensibility & upgrade safety

**Oracle Fusion**

* Upgrade safety comes from staying within supported extension boundaries; the vendor manages the core.

**Ameide**

* Upgrade safety comes from explicit extension points, generated contracts, import policies, and impact analysis in CI.
* The “clean core” principle is enforced by repository and operator guardrails rather than a closed vendor core.

---

### 5.4 Process & workflow

**Oracle Fusion**

* Provides workflow/approvals oriented around suite objects and configured rule frameworks.

**Ameide**

* Processes are first‑class (`Process` primitive) executing versioned **ProcessDefinitions** stored in the **Transformation Domain**.
* Long‑running orchestration is explicit and observable (Temporal‑backed), and process changes are promoted like code.

---

### 5.5 Integration

**Oracle Fusion**

* Integrations are implemented via suite APIs and integration tooling, with environment-specific configuration and credentials managed outside “business logic”.

**Ameide**

* Integrations are first‑class primitives (`Integration`) with proto‑declared ports/contracts and operator‑managed day‑2 operations (drift, rollout, schedules).
* Domain logic stays isolated from broker/client details (outbox + idempotent consumers); environment bindings live in CRDs/runtime config.

---

### 5.6 Analytics & reporting

**Oracle Fusion**

* Built-in reporting and analytics surfaces are provided by the suite, with customization within supported boundaries.

**Ameide**

* Analytics/read models are explicit primitives (`Projection`) derived from facts/events and exposed via query APIs optimized for consumption.
* Projections are replaceable and evolvable without conflating them with sources of truth.

---

## 6. Bottom line

Oracle Fusion optimizes for **suite coherence via vendor‑managed SaaS + configuration and controlled extensibility**.

Ameide optimizes for **coherence via code‑first contracts + a small set of operational CRDs**, with AI‑operated development and explicit primitives for domains, processes, agents, UIs, projections, and integrations.
