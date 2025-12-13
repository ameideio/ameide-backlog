# 470.D – Ameide Code‑First / AI‑First ERP vs. Odoo (Modular OSS ERP: Python/ORM + Views + Configuration)

**Status:** Draft  
**Owner:** Architecture / Product  
**Companion:** [470-ameide-vision.md](470-ameide-vision.md)  
**Related:** [520-primitives-stack-v2.md](520-primitives-stack-v2.md) (normative primitives stack v2)

## 1. Purpose

This document describes the objectives of the **Ameide architecture** relative to **Odoo’s modular ERP model** (modules/addons, Python + ORM, view definitions, and functional configuration).

We are **not** trying to “containerize Odoo and call it next-gen ERP”. Instead, we want:

* Odoo-class modularity and end-to-end coherence,
* but with a **proto‑centric, multi-service primitives architecture** that is **AI‑operated**,
* and a strict separation between **sources of truth** and **projections/integrations**.

---

## 2. Two paradigms in one sentence each

**Odoo:**

> A modular, model-driven ERP where business objects are Python ORM models, UI is heavily driven by view definitions and conventions, and many implementations combine configuration with custom modules (addons).

**Ameide:**

> A **code‑first, proto‑centric platform** where business capability is implemented as code in a single repo, operated via **six primitives** as Kubernetes CRDs: **Domain / Process / Agent / UISurface / Projection / Integration**, with an **AI agent** as the primary developer.

---

## 3. Where Odoo excels (what we want to preserve in spirit)

1. **Modularity by default**
   * Coherent “apps” that can be installed/extended, with explicit dependencies.

2. **End-to-end business coverage**
   * Common enterprise workflows feel integrated because the platform ships a broad catalog of business capabilities.

3. **Fast adaptation loop**
   * A lot can be achieved quickly via configuration and lightweight customization.

---

## 4. What Ameide intentionally does differently

### 4.1 Source of truth: Domain primitives + events (not ORM models as the global nucleus)

**Odoo**

* The ORM model layer is the center of gravity for data + business behavior.
* Custom modules frequently touch shared models and views, which can increase upgrade friction.

**Ameide**

* Sources of truth are explicit **Domain primitives** with proto-first APIs and event emission (outbox + idempotent consumers).
* Read models and UI convenience queries move into **Projections**, not into “the source domain database plus extra computed fields”.

---

### 4.2 Process: explicit orchestration vs implicit workflow glue

**Odoo**

* Many business processes emerge from model methods, state fields, automated actions, and module conventions; long-running orchestration is often “implicit”.

**Ameide**

* Orchestration is explicit as a `Process` primitive executing versioned **ProcessDefinitions** stored in the **Transformation Domain** (Temporal-backed execution).
* Process behavior is designed for observability, retries, compensation, and versioned evolution.

---

### 4.3 UI: view-driven forms vs UISurfaces as code

**Odoo**

* Large parts of UX are defined by view/config patterns (lists/forms/search, etc.) and framework conventions.

**Ameide**

* UISurfaces are code-first applications (e.g., Next.js) consuming generated SDKs; the platform favors standard web development patterns.
* UI is kept close to typed APIs rather than being derived from a global view engine.

---

### 4.4 Extensibility: module ecosystem vs explicit extension points + manifests

**Odoo**

* Extensions are modules that can add/override models, views, and behaviors; power comes with coupling risk.

**Ameide**

* Extensions are code modules + manifests that target explicit extension points (interfaces/events/hooks).
* Guardrails are enforced by generation + CI (regen-diff, import policy, breaking checks), not by “best effort” conventions.

---

### 4.5 Integration: connectors vs Integration primitives (flows-as-code, operator managed)

**Odoo**

* Integrations are typically connector modules, custom integrations, or external middleware; deployments vary widely.

**Ameide**

* Integrations are first-class primitives (`Integration`) with proto-declared ports/contracts and operator-managed day-2 operations (drift detection, schedules, rollout).
* Environment bindings and secrets remain outside proto and are injected via CRDs/runtime config.

---

## 5. Bottom line

Odoo optimizes for **modularity and speed via a unified app framework** (ORM + views + modules), which is excellent for rapid implementations but often couples customization deeply into the framework’s model layer.

Ameide optimizes for **coherence via code-first contracts + a small set of operational CRDs**, making sources of truth, orchestration, UI surfaces, projections, and integrations explicit and AI-operable at scale.
