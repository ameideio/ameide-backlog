### Where **form‑js** Sits in the Semantic Stack

| Layer                                                | Role of *form‑js*                                                                                                                                                                          | Why it Belongs Here                                                                                            |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------- |
| **2 · Deterministic Behaviour** (BPMN)               | Renders / edits the **User Task** or **Manual Task** UI that lets a human supply or approve data.                                                                                          | A form is part of the *prescribed* control‑flow—every time the task fires the same questions must be answered. |
| **3 · Adaptive / AI Behaviour** (wfdesc → LangGraph) | Surfaces **human‑in‑the‑loop (HITL)** checkpoints: e.g. an LLM proposes a decision; a reviewer approves via a form.                                                                        | Keeps the “AI improvise → human validate” loop explicit and auditable.                                         |
| **4 · Provenance**                                   | On *submit*, the form runtime can emit a `prov:Activity` (the **submission**) that **used** the question set (`prov:Entity` form schema) and **wasAssociatedWith** the human `prov:Agent`. | Gives end‑to‑end traceability: who answered what, when, and in which workflows run.                             |
| **5 · Agency Semantics** (OASIS Agent)               | A *form-js* session materialises a **delegation** (“User must approve within 24 h”) that maps straight to OASIS `Commitment` / `Delegation`.                                               | Enables SLA checks (“task overdue”) without hard‑coding timers in BPMN.                                        |

---

## Concrete Integration Points

| Graph Edge (→ `meta_link` schema)                     | Example                                            | Created By                                                             |
| ----------------------------------------------------- | -------------------------------------------------- | ---------------------------------------------------------------------- |
| `(:bpmn__UserTask)-[:REQUIRES_FORM]->(:form__Schema)` | “Approve Order” → *order‑approval‑v3.json*         | BPMN importer (reads `camunda:formRef` or custom `formRef` attribute). |
| `(:wfdesc__Step)-[:HITL_FORM]->(:form__Schema)`       | `review_diagnosis` step → *diagnosis‑review\.json* | wfdesc importer or LangGraph transpiler.                               |
| `(:prov__Activity)-[:USED_FORM]->(:form__Schema)`     | `Activity {id=taskRun‑123}`                        | Temporal/Camunda interceptor in provenance‑collector.                  |

**Storage choice**

* Keep *form-js* **schema JSON** as a lightweight `form` graph (labels `form__Schema`, `form__Field`) so that form evolution is version‑controlled and queryable (e.g. *“which processes still ask for SSN?”*).

---

## Repository Placement

```
ameide-core/
├─ importers/
│   └─ form-js/             # NEW: parses JSON schema → AGE `form` graph
├─ transpilers/
│   ├─ bpmn/temporal/
│   │   └─ ...              # already proposed IR / emitter
│   └─ hitl/                # NEW: code that wires UserTask ↔ form runtime
│       ├─ temporal/        # generates TS/Go activity stubs awaiting form data
│       └─ camunda/         # copies schema into deployment ZIP
├─ ui/
│   └─ form-js/             # embedded viewer/editor, React wrapper, tests
└─ services/
    └─ form-gateway/        # small Node service hosting rendered forms, authenticating users
```

*The `form-gateway` service can be reused by both Temporal and LangGraph workers: they create a **task token** and hand the browser a `/forms/start?token=…` URL; on submission the gateway signals/patches the workflows.*

---

## Runtime Flow (Temporal Example)

```text
1. BPMN UserTask has camunda:formRef="order-approval-v3".
2. Transpiler generates:
   • TypeScript activity `await hitl.waitFor("order-approval-v3", order);`
   • metadata edge :REQUIRES_FORM to Form Schema node UUID_x.
3. Worker calls form-gateway → gets public URL; sends e‑mail/slack to assignee.
4. User fills the @bpmn-io/form-js form ➜ gateway
   • validates payload (feel-js expressions)
   • signals workflows with data
   • POSTs to provenance-collector:
     (:prov:Activity {uuid=run‑123})-[:USED_FORM]->(:form__Schema {uuid=UUID_x})
     (:prov:Activity)-[:wasAssociatedWith]->(:prov:Agent {userId})
5. Workflow continues; Grafana panel can pivot from prov node to Tempo trace.
```

---

## Benefits Delivered

* **Consistent UX** – Same JSON schema drives Camunda, Temporal HITL steps, and form‑js playground.
* **Schema Governance** – Versioned in Git, visible in the knowledge graph, and diff‑able in PRs.
* **Full Audit Chain** – Each submission is tied to *who/what/when* via PROV‑O; compliance teams can trace a decision back to the exact form version.
* **Loose Coupling** – Swapping Temporal ↔ Camunda does **not** affect form definitions; only the runtime adapter changes.

---

## Minimal Path to Prove Value (≈2 sprints)

1. **Importer**: parse one sample `.form.json`; create `form` graph and `:REQUIRES_FORM` edge for a BPMN UserTask.
2. **Gateway+Viewer**: embed form-js viewer in a simple Express app; return JSON on submit.
3. **Temporal Stub**: `waitForForm(schemaId)` activity that blocks on gateway callback.
4. **Provenance Hook**: interceptor attaches `USED_FORM` in collector.
5. **Demo**: run end‑to‑end “Approve Order” workflows with UI, show provenance node in AGE.

Once this slice is operational you have closed the loop from **design → human action → audit trail**, proving the stack’s multi‑layer story.
