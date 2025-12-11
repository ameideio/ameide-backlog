Below is a consolidated **target architecture** for your modelling stack now that a single “ETML” abstraction has been ruled out. Everything is organised around four *self‑contained* notations, each persisted natively in Apache AGE but still able to reference objects from its siblings.

---

## 1 . Repository layout in Apache AGE

| Graph name (schema) | Purpose                                  | Node labels                                                                                 | Edge labels                                              | Import / Export                                                                     |
| ------------------- | ---------------------------------------- | ------------------------------------------------------------------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `archimate`         | Enterprise‑wide structure & capabilities | *Exactly* the ArchiMate 3.2 class names (`BusinessProcess`, `ApplicationComponent`, …)      | ArchiMate relationship names (`TRIGGERS`, `REALIZES`, …) | `.archimate` exchange XML round‑trip (no code generation)                           |
| `bpmn`              | Deterministic process logic              | `StartEvent`, `Task`, `UserTask`, `ExclusiveGateway`, `SequenceFlow` *(as relationship)*, … | `SEQUENCE_FLOW` *(edge)*                                 | `.bpmn` XML <br>→ **Camunda** deployment package<br>→ **Temporal.io** workflows code |
| `wfdesc`            | Non‑deterministic/AI agent orchestration | `Process`, `Step`, `Input`, `Output`, `Agent` *(WF4Ever vocabulary)*                        | `EXECUTES`, `DEPENDS_ON`, …                              | `.wfdesc.json` <br>→ **LangGraph** Python module                                    |
| `uml`               | Logical/data model                       | UML class names (`Class`, `Association`, `Attribute`, …)                                    | `ASSOCIATION`, `GENERALIZATION`, …                       | XMI in/out <br>→ Delta‑Lake / Databricks DDL & PySpark stubs                        |

**Why separate graphs?**

* Strong label‑level constraints, fast label scans, and minimal refactoring when any standard evolves.
* Cross‑notation queries stay possible because AGE lets you address multiple graphs in the same Cypher session.

---

## 2 . Cross‑notation linkage strategy

| Link edge (global) | Semantics                                                                    | Created by                                              |
| ------------------ | ---------------------------------------------------------------------------- | ------------------------------------------------------- |
| `:REALISES`        | ArchiMate element *realised by* BPMN (e.g., `:BusinessProcess` → `:Process`) | Manual curation or heuristic matcher                    |
| `:DECOMPOSES_TO`   | BPMN `Task` *decomposed into* a `wfdesc:Process` (AI agent flow)             | BPMN modeller adds `implementation="wfdesc"` annotation |
| `:IMPLEMENTS`      | UML `Class` *implements or stores data for* a BPMN `DataObject`              | UML importer or data architect                          |
| `:MONITORS`        | wfdesc `Agent` *emits metrics for* ArchiMate `ApplicationComponent`          | SRE / platform team                                     |

All linkages go into a **separate, tiny graph** `meta_link` so the domain graphs remain pristine.

---

## 3 . Import / export & transpilation pipelines

```text
┌──────────────┐      Importer         ┌──────────────┐
│ order.bpmn   │──┐  (Python       )   │  bpmn graph  │
└──────────────┘  │                   └──────────────┘
                  │                       │
                  │  Transpiler           ▼
                  │  (Jinja → TS/Go)  Camunda XML pkg
                  │                 Temporal Workflow
                  ▼
        ┌────────────────┐
        │ code‑gen / ci  │
        └────────────────┘
```

### 3.1 BPMN → Camunda / Temporal

1. **Importer** (`BpmnImporter`) populates `bpmn` graph exactly as today.
2. **Annotate tasks** with custom properties:

| Property                     | Meaning                                                                         |
| ---------------------------- | ------------------------------------------------------------------------------- |
| `impl="camunda"` *(default)* | included unchanged in Camunda package                                           |
| `impl="temporal"`            | rendered into deterministic Temporal workflows code                              |
| `impl="wfdesc"`              | generates a Temporal *activity stub* + `DECOMPOSES_TO` edge to a wfdesc process |

3. **Transpiler** traverses the graph and renders:

* Camunda BPMN **unchanged** (you can still open it in Modeler).
* Temporal **TypeScript/Go** workflows + activity stubs.

### 3.2 wfdesc → LangGraph

* Importer reads WF‑METS/WF‑Desc JSON and writes into `wfdesc` graph.
* Transpiler walks each `Process` node and emits:

```python
from langgraph.graph import StateGraph, START, END
# generated nodes …
graph = builder.compile()
```

* The generated module is packaged as a Docker image.
* Temporal activity stubs call it via gRPC/HTTP.

### 3.3 UML → Databricks / Fabric

* Importer ingests XMI into `uml` graph.
* Transpiler produces:

  * Delta‑Lake table DDL (`CREATE TABLE …`)
  * PySpark or Fabric notebooks with ORM‑style classes that map UML attributes to Spark schema.

---

## 4 . Governance processes (TOGAF ADM / Scrum)

* Model **ADM phases** or **Scrum ceremonies** as ***BPMN processes*** inside `bpmn` graph.
* Tasks requiring AI assistance (e.g., “Draft Architecture Vision”, “Refine Product Backlog”) are annotated `impl="wfdesc"`.
* Importer auto‑links a skeleton wfdesc agent flow; architects can flesh out individual `Step`s later.
* **Transpilation result:** one **Temporal Workflow** driving deterministic ADM/Scrum flow, invoking **LangGraph agents** at the annotated points.

```
[Start] ──> "Evaluate Architecture" (impl=temporal)
      │
      ├─> "Generate Alternatives" (impl=wfdesc) ──► LangGraph
      │
      └─> "Board Approval" (impl=temporal signal) ──► End
```

---

## 5 . Practical conventions & guidelines

| Topic           | Recommendation                                                                                                                                 |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **Identifiers** | Preserve source `xml_id` / `xmi:id` as property `src_id`. Generate a canonical `uuid5(namespace, src_id)` for cross‑graph joins.               |
| **Versioning**  | Stick to *notation‑own* versions (ArchiMate 3.2, BPMN 2.0 …). Increment a separate `release_tag` when you publish a new snapshot of any graph. |
| **Indexing**    | Unique index on `(label, src_id)` inside each graph; index on `uuid` in `meta_link` graph.                                                     |
| **Validation**  | Keep the original XSD/Ecore constraints; add Cypher constraints where the standard is vague (e.g., only `SequenceFlow` may link flow nodes).   |
| **CI checks**   | *Round‑trip* test each importer/exporter, *compile* transpiled artefacts, and *dry‑run* generated Temporal/LangGraph workflows.                |
| **Security**    | Use AGE row‑level security to expose only needed graphs to each team (EA, Process, Data, ML).                                                  |

---

## 6 . Benefits & trade‑offs of this layout

### Benefits

* **Zero refactor of existing importers** – they continue to dump straight into their native graphs.
* **Clear separation of concerns** – each team works on the notation it already knows.
* **Deterministic / non‑deterministic split** materialises naturally (BPMN ↔ wfdesc).
* **Tool‑friendly** – Camunda Modeler, Archi, StarUML, etc. read the exported files unchanged.

### Trade‑offs

* **Cross‑notation queries** are a little more verbose (`GRAPH_PATH` or explicit graph names in AGE).
* **Schema sprawl** – four graphs, four transpilers, four validation suites.
* **Link maintenance** – heuristics or manual curation required to keep `meta_link` edges accurate.

---

### **Bottom line**

Keep **ArchiMate, BPMN, wfdesc and UML in separate, faithfully modelled graphs**; use lightweight **link edges** in a dedicated `meta_link` graph to bind them together.
This lets each notation evolve at its own speed, keeps existing tooling intact, and still delivers the *hybrid execution model* you need:

* **Temporal** drives deterministic BPMN logic.
* **LangGraph** executes AI agents modelled in wfdesc.
* **Camunda** remains an option for pure BPMN deployments.
* **Databricks / Fabric** receive code‑generated schemas and ETL pipelines straight from UML.

You avoid a big‑bang meta‑ontology refactor, yet retain full interoperability and forward compatibility for future notations or execution targets.

---

## ✅ IMPLEMENTATION UPDATE (December 2025)

### Completed:
- **2 of 4 graphs implemented**: `archimate` and `bpmn` fully operational in Apache AGE
- **Cross-model linking**: Implemented via `link_archimate_to_bpmn()` function in `meta_link` pattern
- **Import/Export**: Full XML round-trip for both ArchiMate and BPMN formats
- **Visualization**: PlantUML diagrams generated from graph data (component, activity, sequence)
- **Services**: All model management services containerized with descriptive names
- **✅ Transpilers**: BPMN → Temporal workflows code generation (Python) implemented in `packages/ameide_core-transpilers/`
- **✅ Provenance tracking**: Runtime-to-provenance integration hooks with `EXECUTED_AS` links
- **✅ Graph abstraction**: Generalized GraphProvider protocol supporting AGE, Neo4j, and MockProvider

### Pending:
- **wfdesc graph**: For AI agent orchestration (partial support via `impl="wfdesc"` tasks)
- **uml graph**: For logical/data models (Databricks/Fabric integration)
- **Transpilers**: TypeScript code generation (templates exist), Go generation (not implemented)
