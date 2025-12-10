Here’s a version of the same “vision” but framed explicitly **vs SAP S/4HANA** instead of D365.

---

# 470.B – Ameide Code-First / AI-First ERP vs. SAP S/4HANA Metadata- & Config-Driven ERP

**Status:** Draft  
**Owner:** Architecture / Product  
**Companion:** [470-ameide-vision.md](470-ameide-vision.md)

## 1. Purpose

This document describes the objectives of the **Ameide architecture** relative to **SAP S/4HANA’s metadata- and configuration-driven model** (ABAP Data Dictionary, CDS, Fiori annotations, Customizing/IMG, extensibility frameworks).

We are **not** trying to rebuild S/4HANA’s DDIC + CDS + Customizing stack inside Kubernetes. Instead we want:

* ERP-class capabilities and coherence,
* but with a **code-first, AI-operated** core,
* and a **thin, infra-oriented metadata layer**, not a huge metadata/configuration universe.

---

## 2. Two paradigms in one sentence each

**SAP S/4HANA:**

> A mix of **metadata repositories and configuration frameworks** (ABAP Data Dictionary, CDS views with annotations, Fiori elements, Customizing/IMG, key-user and developer extensibility) plus ABAP code; humans primarily adapt the system via configuration and metadata, and extend it via ABAP and BAdIs.

**Ameide:**

> A **code-first, proto-centric system** where business logic lives in Go/Temporal/TS in a single repo, with **Domain / Process / Agent / Surface** as coarse-grained runtime primitives modeled as Kubernetes CRDs; an **AI agent** is the primary “developer” that edits code and specs.

---

## 3. Ameide Approach – High-Level Objectives (recap)

1. **Code-first, AI-friendly representation**

   * Business logic and structure are primarily **code + proto contracts**, not dense metadata/config layers.
   * We optimize for **LLMs navigating and editing code**, not humans working in SE11/SE80 or Fiori config apps.

2. **Thin operational metadata layer (CRDs only where infra matters)**

   * We introduce CRDs only for **infra-relevant** units:

     * `Domain` – bounded context with data + rules
     * `Process` – long-running orchestration
     * `Agent` – AI worker
     * `Surface` – UI shell / entry point

   * No CRDs for “table”, “field”, “form”, or “UI annotation”.

3. **Central, coherent system graph derived from code**

   * A **single codebase + proto descriptors + CRDs** is the system’s “model store”.
   * Tools (and AI) query this graph to understand domains, processes, agents, surfaces, and their relationships.

4. **Extensions are code modules, not configuration trees**

   * Partners ship **code + a small manifest**, not IMG projects, Customizing variants, or DDIC artifacts.
   * They work by **chatting with AI**, which edits code and specs against well-defined extension points.

5. **Preserve ERP-class coherence, without replicating S/4’s config/meta bulk**

   * We still care about:

     * cross-reference and impact analysis,
     * upgrade-safe extensions (“clean core” in spirit),
     * security + data classification,
   * but we achieve it by analyzing **code+proto+CRDs**, not via a huge Customizing layer or DDIC.

---

## 4. Quick overview: S/4HANA’s metadata & config stack

To clarify what we’re deliberately *not* recreating, here’s a rough picture of the S/4 world.

### 4.1 ABAP Data Dictionary (DDIC)

* Central repository of metadata: tables, views, structures, **domains, data elements, search helps**, etc.
* Defines data types, lengths, check tables, foreign-key relations, technical settings.
* All ABAP programs use the DDIC to interact with the DB; it enforces global integrity.

### 4.2 CDS views, annotations, and Fiori Elements

* **Core Data Services (CDS)** views define semantically rich data models (associations, aggregations, etc.).
* **UI annotations** on CDS/OData specify:

  * which fields appear in lists/forms,
  * UI facets, value helps, filters, semantics.
* SAP Fiori Elements uses these annotations to **auto-generate UI**.

S/4 uses CDS+annotations as a **metadata-driven UI and API layer** on top of DDIC.

### 4.3 Customizing / IMG / Business Configuration

* Huge configuration tree via **Implementation Guide (IMG/SPRO)** for finance, logistics, etc.
* Customizing data stored in control tables; grouped into BC Sets, transported between systems.
* This configuration drives system behaviour without changing code.

### 4.4 Extensibility frameworks

* **In-app / key-user extensibility**:

  * Custom fields & logic, custom CDS views, custom business objects, etc. via Fiori apps.
* **Developer extensibility / ABAP**:

  * BAdIs, enhancement framework, side-by-side or “clean core”-compliant ABAP code.

So S/4 is *heavily* metadata/config driven, with ABAP as the programmable escape valve.

---

## 5. Ameide vs S/4HANA – Objective-Level Comparison

### 5.1 Source of truth: metadata/config vs code + proto

**S/4HANA**

* System behaviour emerges from:

  * DDIC (tables, domains, data elements, views),
  * CDS views and annotations (OData, UI semantics),
  * Customizing (IMG, configuration tables),
  * plus ABAP code and BAdI implementations.

* There are multiple overlapping repositories of truth:

  * Data model in DDIC,
  * UI semantics in CDS annotations,
  * behaviour in ABAP,
  * configuration in IMG tables.

**Ameide**

* **Code + proto are the primary source of truth**:

  * Proto packages define entities, commands, queries, events.
  * Go/Temporal/TS implement behaviour.
* CRDs (`Domain`, `Process`, `Agent`, `Surface`) describe **how to run** that code in the cluster, not business semantics.
* Business configuration is expressed mainly as:

  * structured config objects in code or proto,
  * not as a separate, massive Customizing tree.

**Objective:**
Avoid multiple divergent metadata/config layers. Keep a *single* modelable graph based on **code + proto + a thin set of CRDs**, so AI and tools can reason about the whole system without reconciling DDIC+CDS+IMG.

---

### 5.2 Object model granularity

**S/4HANA**

* Very fine-grained metadata in DDIC:

  * domains, data elements, tables, fields, search helps, views.
* CDS layer adds a second modeling space (entities, associations, annotations).
* Fiori, analytics, and extensibility layers reference these meta-objects.

**Ameide**

* Infra-level object model is **intentionally coarse**:

  * A `Domain` might own dozens of entities, but infra doesn’t know them individually.
* “EDT-like” concepts, forms, and queries live as:

  * code patterns and proto message types,
  * optional annotations/options in proto,
  * not as separate runtime objects.

**Objective:**
Keep infra simple; avoid an explosion of runtime metadata entities. Low-level semantics are **code patterns** (which AI finds natural) rather than first-class runtime objects.

---

### 5.3 Development model: key-user / config vs AI developer

**S/4HANA**

* Heavy emphasis on **configuration and key-user tools**:

  * Custom Fields & Logic, Custom CDS Views, Custom Business Objects, etc.
* ABAP developers extend via released APIs and BAdIs, often guided by “clean core” principles.
* Tools are designed for **humans** to click/configure in Fiori or SE11/SE80.

**Ameide**

* Designed for **AI as the primary developer**:

  * AI reads/writes Go/Temporal/TS and proto,
  * manipulates Domain/Process/Agent/Surface CRDs.
* Humans primarily:

  * express requirements,
  * review diffs,
  * approve changes / rollouts.

**Objective:**
Treat S/4’s rich key-user tooling as human-heavy and not needed in this context. Instead, use **plain code + simple metadata** as the substrate for LLM-driven development and refactoring.

---

### 5.4 UI & semantics: CDS + Fiori elements vs Surfaces as code

**S/4HANA**

* CDS views with UI annotations define **UI-relevant metadata directly in the data model**; Fiori Elements generates list reports, object pages, etc. automatically.

**Ameide**

* Surfaces are **Next.js apps / micro-portals**:

  * They consume typed SDKs based on proto,
  * but we don’t have a full metadata-driven form engine like Fiori Elements.
* Optional: we may add “form/board/workspace patterns” in code, but still code-centric.

**Objective:**
Rely less on metadata-driven UI generation and more on **conventional, AI-crafted React**. Keep UI semantics as close to standard web development as possible so both AI and web developers are productive.

---

### 5.5 Extensibility & “clean core”

**S/4HANA**

* Extensibility strategy:

  * Key-user in-app extensibility (fields, logic, CDS, BOs),
  * developer extensibility via BAdIs, ABAP on S/4HANA, and side-by-side BTP apps,
  * all under a **clean core** philosophy (stay within released interfaces, avoid core modifications).

**Ameide**

* Extensibility = **code modules + manifest**:

  * A module declares:

    * which Domains/Processes/Surfaces it extends,
    * which extension points it hooks into.
  * AI helps partners write those modules.

* “Clean core” is enforced by:

  * explicit extension interfaces,
  * linting and static analysis (no imports into forbidden packages),
  * impact analysis based on the module manifest and code graph.

**Objective:**
Keep the **spirit** of S/4’s clean core (upgrade-safe, interface-based extensions) while making the mechanics code-centric and AI-operated rather than config/metadata-heavy.

---

### 5.6 Business configuration

**S/4HANA**

* Business behaviour is heavily influenced by **Customizing / IMG**:

  * chart of accounts, document types, number ranges, posting rules, etc. managed via transactions & Fiori apps, stored in config tables and BC Sets.

**Ameide**

* Business configuration is:

  * structured **config data** owned by Domains (e.g., config tables in Postgres, feature flags),
  * exposed via typed proto messages and admin Surfaces,
  * versioned and moved via normal app mechanisms (migrations, seed data, domain services),
  * not a separate IMG universe.

**Objective:**
Avoid a separate “config world” with its own UI, transport, and logic. Treat configuration as part of the **domain model** and code, which AI can also understand and evolve.

---

### 5.7 Runtime introspection

**S/4HANA**

* You can introspect:

  * DDIC (tables, elements, domains),
  * CDS views & annotations,
  * Customizing tables,
  * registered BAdIs and implementations.

* SAP tooling (SE11/SE80, Fiori apps, Extensibility Explorer, etc.) gives a rich view of metadata and extensibility options.

**Ameide**

* We provide introspection via a **graph service** that:

  * scans proto descriptors,
  * indexes Go/Temporal/TS code,
  * reads Domain/Process/Agent/Surface CRDs and module manifests.

* Queries like:

  * “Show me all processes involving Invoices.”
  * “Which extensions hook into Domain ‘Sales’?”
  * “Where is PII stored?”

  are answered by analyzing this graph, not by traversing DDIC/IMG/CDS.

**Objective:**
Provide S/4-like or better “what’s in my system?” answers, but over **modern code + proto + k8s resources** rather than multiple legacy metadata/config repositories.

---

## 6. Non-Goals vs S/4HANA

Ameide explicitly does **not** aim to:

* Recreate the **ABAP Data Dictionary** as a separate metadata system.
* Provide a **CDS + Fiori Elements** style metadata-driven UI generator.
* Build an **IMG / Customizing** universe with BC Sets and dedicated config transports.
* Replicate key-user tools for non-technical users to mold the system.

Instead:

* We assume **developers + AI** as the primary shaping forces.
* We use standard web & cloud-native primitives (Go, TS, Kubernetes, Postgres, Temporal).
* We keep metadata **thin and infra-oriented**, not a parallel modeling universe.

---

## 7. Summary

Compared to **SAP S/4HANA’s metadata- and configuration-driven architecture**, Ameide:

* Chooses **code-first, AI-first** over config-first and key-user tooling.
* Replaces DDIC + CDS + IMG + BAdI frameworks with:

  * a **single codebase**,
  * **proto contracts**,
  * **Domain/Process/Agent/Surface CRDs**,
  * and **extension modules + manifests**.
* Keeps the *goals* of ERP metadata systems (coherence, extensibility, clean core, introspection), but:

  * implements them via **code analysis, proto descriptors, and k8s primitives**,
  * in a shape that’s friendlier for LLMs and cloud-native operations.

In short:
**Ameide is what you get if you take the core *intent* of S/4HANA (structured ERP, clean core, extensible) but redesign it under the assumption that “the main developer is an AI that loves code, not a human that loves configuration screens.”**
