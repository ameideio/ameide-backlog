# 470.A – Ameide Code‑First / AI‑First ERP vs. D365 AOT Metadata‑Driven ERP

**Status:** Draft  
**Owner:** Architecture / Product  
**Companion:** [470-ameide-vision.md](470-ameide-vision.md)

## 1. Purpose

This document defines the objectives of the **Ameide architecture** relative to the **Dynamics 365 Finance & Operations (D365FO) metadata / AOT–driven model**.

We explicitly **do not** aim to recreate the D365 AOT stack in Kubernetes. Instead, we:

* Keep some *systemic* benefits AOT gives (coherence, discoverability, extensibility),
* While **intentionally replacing** the metadata‑first paradigm with a **code‑first, AI‑operated design**.

---

## 2. Two paradigms in one sentence each

* **D365 AOT model:**
  The application is a **metadata tree** (AOT) of tables, EDTs, enums, forms, menu items, security, etc. – a compiler turns that into runtime artifacts. Humans primarily edit metadata in Visual Studio.

* **Ameide model:**
  The application is **code (Go / TS / Temporal) plus proto contracts**, living in a **single codebase**, with **Domain / Process / Agent / Surface** as operational primitives (CRDs). An **AI agent**, not humans in a designer, is the main developer.

---

## 3. Ameide Approach – High‑Level Objectives

1. **Code‑first, AI‑friendly representation**

   * Business logic and structure are primarily expressed as **normal code** and **protos**, not heavyweight metadata trees.
   * We optimize for **LLMs understanding and editing code** over humans dragging boxes in a designer.

2. **Thin, operational metadata layer**

   * **CRDs** exist only for *infra‑relevant* units:

     * `Domain`
     * `Process`
     * `Agent`
     * `Surface`
   * We do **not** create CRDs for low‑level concepts like EDTs, fields, or individual forms.

3. **Central, coherent system graph derived from code**

   * A **single repository** + proto descriptors + CRDs form a **global model** of the system.
   * Tools and AI can ask: “what domains exist?”, “which processes touch SalesOrder?”, “which agents can issue command X?”

4. **Extension model is code, not metadata**

   * ISV / partner extensions are **modules of code** (plus a thin manifest), developed by **chatting with AI**.
   * We do **not** ask partners to understand or modify a giant metadata tree; they focus on code and well‑defined extension points.

5. **Preserve ERP‑class coherence without AOT overhead**

   * We still care about:

     * cross‑reference analysis,
     * upgrade safety,
     * security and data classification,
     * discoverable APIs and entities,
   * but we achieve it with **code analysis + proto + minimal manifests**, not with a dedicated AOT storage and designer.

---

## 4. D365 AOT vs Ameide – Objective‑Level Comparison

### 4.1 Source of truth

**D365 AOT**

* **Metadata‑first.** AOT is the canonical definition of tables, forms, security, labels, etc.
* Build pipeline compiles AOT/X++ into IL and packages.
* If it’s not in AOT, it’s not really part of the app.

**Ameide**

* **Code‑first, metadata‑guided.**
* The canonical definition of behavior is **Go / TS / Temporal code + proto contracts**.
* CRDs and design artifacts **operate** this code (configure, deploy, orchestrate), they don’t replace it.
* We still maintain a **derived “system graph”** for tooling, but that graph is *computed from* code/protos/CRDs instead of being hand‑maintained metadata.

**Objective:**
Maximize expressiveness and AI‑operability by treating **code as the primary model**, and metadata as a thin operational and analytical layer.

---

### 4.2 Object model granularity

**D365 AOT**

* Very fine‑grained application object model:

  * Tables, fields, relations, EDTs, enums, queries, views, forms, menu items, workspaces, security roles/duties/privileges, label files, etc.
* All of these are first‑class metadata objects.

**Ameide**

* **Infra‑level model is intentionally coarse:**

  * `Domain` (bounded context with data + rules)
  * `Process` (orchestration / saga)
  * `Agent` (AI worker)
  * `Surface` (UI entry point)
* Lower‑level concepts (e.g. “EDT”, “form”, “query”) are:

  * **code**, patterns, and conventions,
  * optionally described via **proto options**, type wrappers, or lightweight manifests,
  * but **not** CRDs and not infra objects.

**Objective:**
Keep infra simple and composable; avoid running “infrastructure for an EDT”. Treat low‑level semantics as **code patterns** AI can understand, not as separate runtime entities.

---

### 4.3 Development model

**D365 AOT**

* Designed around **human developers**:

  * Application Explorer, designers for forms, security, and data entities.
  * Metadata exists partly to provide rich design‑time UX.

**Ameide**

* Designed around **AI as the primary developer**:

  * AI navigates **code and proto**, not heavyweight designers.
  * Humans interact mostly through:

    * conceptual docs and requirements,
    * reviewing diffs,
    * guiding the AI.

**Objective:**
Treat “metadata‑for‑Visual Studio” as unnecessary overhead. Instead, shape the system around what LLMs understand best today: **code + simple, structured metadata**.

---

### 4.4 Extensibility & ISV model

**D365 AOT**

* Extensions are mainly **metadata extensions**:

  * extension classes, event handlers, extensible enums, etc.
* ISVs work in the same AOT tree, within a constrained extensibility model.

**Ameide**

* Extensions are **code modules + manifests**, developed by chatting with AI:

  * Add new Go packages / Temporal workflows / Surfaces.
  * Register against explicit extension points (interfaces, events, hooks).
* A **small manifest** (not a whole AOT layer) declares:

  * module name, version, dependencies,
  * which Domains/Processes/Surfaces it touches.

**Objective:**
Make extensibility **AI‑operated and code‑centric**, but still track dependency and impact information via a thin manifest so we can reason about upgrades and conflicts.

---

### 4.5 Runtime representation & introspection

**D365 AOT**

* Model store is used both at design‑time and runtime.
* Rich introspection: tools can walk the tree of objects, search, do cross‑reference analysis, generate security and impact reports.

**Ameide**

* Runtime consists of:

  * **Pods/Deployments** (Domains/Processes/Agents/Surfaces),
  * **Databases** (Postgres),
  * **Workflows** (Temporal), etc.
* Introspection is built by combining:

  * **proto descriptors**,
  * **code index** (AST / symbols),
  * **CRDs** and module manifests.

**Objective:**
Provide **equivalent or better “what’s in the system?” answers** via a unified graph service over code+proto+CRDs, without forcing all structure through a dedicated AOT store.

---

## 5. What We Intentionally *Don’t* Replicate from D365 AOT

1. **Full metadata‑first object tree**

   * No attempt to replicate every AOT concept (EDT, Form, Query, LabelFile, etc.) as first‑class metadata entities.

2. **Visual designers**

   * No heavy VS‑style designers; we rely on:

     * conventional patterns in React/Next.js,
     * code and schema the AI can operate on.

3. **Metadata‑driven UI layout**

   * We don’t have a general “form engine” that renders UI purely from metadata.
   * We accept **hand/AI‑written Surfaces** as code, with optional hints in metadata.

4. **Model store as only source of truth**

   * We do not enforce that *everything* goes through a metadata store before existing.
   * Code is allowed to be primary, as long as it remains analyzable and mapped into the system graph.

---

## 6. What We *Do* Preserve in Spirit

Even with the differences above, we aim to preserve several **AOT benefits** in a new form:

1. **Coherent system graph**

   * One place (graph service) to ask:

     * “Which domains exist?”
     * “Which processes depend on Domain X?”
     * “Which modules extend Order lifecycle?”
   * Implemented via analysis of **code + proto + CRDs + manifests**.

2. **Upgrade & impact analysis**

   * When changing a proto or Domain, we want to know:

     * which modules and Surfaces are impacted,
     * which extension points are affected.
   * Achieved with static analysis and module manifests rather than metadata trees.

3. **Security and data classification visibility**

   * Even though security and data semantics are mainly expressed in code/proto,
   * We standardize annotations (e.g. proto options, tags) so tools can:

     * find PII,
     * list privileged operations,
     * answer “who can do what where?” questions.

4. **Extensibility discipline**

   * Even if extensions are code, we define:

     * explicit, stable extension points,
     * linting/validation that enforces “don’t patch core internals directly”.

---

## 7. Non‑Goals

To avoid confusion, these are **explicit non‑goals** of the Ameide approach:

* Recreating the **D365 AOT designers** or development UX.
* Providing **metadata for every low‑level object** (EDT, form control, etc.).
* Forcing partners to learn a bespoke metadata language or tree.
* Making design‑time metadata the single, authoritative source of truth for everything.

Our focus is:

> **ERP‑class capabilities, with AI‑operable code as the core, and just enough metadata to keep the system coherent and introspectable.**

---

## 8. Summary

Compared to D365’s AOT‑driven, metadata‑first ERP development:

* Ameide is **code‑first, AI‑first**.
* We keep only a **thin operational metadata layer** (CRDs for Domains/Processes/Agents/Surfaces + small manifests).
* We deliberately move concepts like EDTs, forms, and extensions **into code and proto**, where AI can work most effectively.
* We rebuild the **AOT benefits** (coherence, introspection, extensibility discipline) via:

  * a central repo,
  * proto descriptors,
  * CRDs,
  * and analyzers over that combined graph.

This gives us an architecture that respects what made AOT powerful, while aligning with a future where **AI, not Visual Studio, is the primary development environment.**
