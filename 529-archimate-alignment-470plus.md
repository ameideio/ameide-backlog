# 529 — ArchiMate alignment for 470+ docs

**Status:** Draft  
**Audience:** Architecture + platform engineering  
**Scope:** Make the `470+` document set internally consistent using a single ArchiMate vocabulary, relationship verbs, and layer boundaries.

This backlog is about language and conceptual correctness (not implementation).

## Layer header (Implementation & Migration / governance)

- **Primary ArchiMate layer(s):** Implementation & Migration (rules/checklists for changing docs) with cross-layer vocabulary constraints.
- **Primary element types used:** Capability, Value Stream, Business Process, Application Service/Interface/Event, Application Component, Technology Service/Node/System Software, Work Package/Deliverable/Plateau/Gap.
- **Out-of-scope layers:** none (this doc defines how to write across layers).
- **Secondary layers referenced:** Strategy/Business/Application/Technology (only to define correct terms and verbs).
- **Allowed nouns:** layer, relationship verb, capability/value stream/business process, application component/service/interface/event, technology service/node, work package/deliverable.
- **Prohibited unless qualified:** process, service, domain, event (must be qualified per `470-ameide-vision.md` §0.2).

## 0) Baseline

Use **ArchiMate 3.2** as the baseline vocabulary.

Within Ameide docs, the canonical language anchor is `470-ameide-vision.md` §0.1–§0.2.

## 0.1) Layer model (explicit)

- **Motivation (optional):** principles, drivers, constraints.
- **Strategy:** capability, value stream, course of action, resource.
- **Business:** business process, roles/actors, business services.
- **Application:** application components, application services/interfaces/events, data objects.
- **Technology:** nodes, system software, technology services.
- **Implementation & Migration:** work package, deliverable, plateau, gap.

**Layer labeling rule (common source of drift):**

- **Capability and Value Stream are Strategy-layer elements.** Do not place them under “Business” headings/diagrams. If a document spans layers, keep the Strategy section separate and have Business sections describe the business processes that realize the value stream.

## 0.2) ArchiMate as an official design-time language (not just prose)

In Ameide, **ArchiMate 3.2 notation** is the official design-time language for architecture and capability models (not just a writing vocabulary):

- **Where it lives:** ArchiMate models/views are **design-time artifacts** persisted by the **Transformation Domain** and edited via **Transformation design tooling**.
- **Where it is used:** capability design and cross-layer architecture docs (`470+`) use ArchiMate vocabulary/verbs, and when diagrams are used they should be ArchiMate-consistent.
- **How to apply it:** use the capability worksheet `530-ameide-capability-design-worksheet.md` to keep layer boundaries and verbs consistent.

Companion design-time languages:

- **BPMN (BPMN-compliant)**: ProcessDefinitions (design-time orchestration intent).
- **Markdown**: narrative specs/decision records; links to ArchiMate/BPMN artifacts.
- **Protobuf (Buf modules)**: runtime application boundary (Application Services/Interfaces/Events and Data Objects).

## 1) Canonical relationships (required verbs)

Use this chain (default model):

1. **Capability serves Value Stream**
2. **Value Stream is realized by Business Processes**
3. **Business Processes use (are served by) Application Services**
4. **Application Services are realized by Application Components**
5. **Application Components use Technology Services**
6. **Implementation & Migration elements** (Work Package / Deliverable / Plateau / Gap) describe *change work* and trace to the above; they are not runtime architecture elements.

Event relationship note:

- **Application Events (facts) are triggered by Application Components and may trigger other behavior**, but do not replace the “service boundary” in narrative: facts describe state changes; services define exposed behavior.

## 2) Canonical mapping (ArchiMate ↔ Ameide)

- **Application Components:** Ameide primitives (Domain / Process / Projection / Integration / UISurface / Agent).
- **Application Services / Interfaces / Events:** proto contracts.
  - RPC services and read-only query services are **Application Services**.
  - Topic families / subjects (plus protocol bindings like gRPC/HTTP) are **Application Interfaces**.
  - Facts (domain facts and process facts) are **Application Events** (state changes; name as facts).

## 3) EDA taxonomy mapping (stop arguing about “intent vs event”)

Keep Ameide’s EDA taxonomy, but model it consistently:

- **Fact** → **Application Event** (state change).
- **Intent / command** → request to invoke an **Application Service** (often carried asynchronously over an **Application Interface**).
- **Query** → read-only **Application Service** (often realized by a Projection primitive).

## 4) Disambiguation rules (mandatory phrasing)

Overloaded words must be qualified unless the doc’s primary layer already implies it:

- “process” → **Business Process** or **Process primitive**
- “service” → **Business Service** or **Application Service** or **Technology Service**
- “domain” → **business domain** or **bounded context** or **Domain primitive**
- “event” → **Application Event** (fact) vs “message” (generic transport)

## 5) Doc header requirement (layer boundary guardrail)

Each of these docs should include a small “Layer header” near the top (after scope/audience):

- Primary layer (Strategy/Business/Application/Technology/Implementation & Migration)
- Secondary layers referenced (if any)
- Allowed nouns (for the doc)
- Prohibited overloaded words unless qualified

## 6) Acceptance criteria (alignment)

- No normative doc defines a capability as “decomposing into primitives”.
- `471` stays Business/Strategy-first and pushes implementation detail into clearly marked “realization notes”.
- `472` stays Application-first and maps intents/facts/queries to services/interfaces/events correctly.
- `473` stays Technology-first and treats brokers/DBs/workflow engines/operators as Technology Services/Nodes/System Software.
- `528` uses the canonical relationship chain and the 4-layer deliverables template.

## 7) Why this matters

Without consistent vocabulary, we repeatedly blur:

- capability (business intent) vs primitive (application component),
- proto contracts (application services/interfaces/events) vs runtime topology (K8s/operators/brokers),
- “domain” as a business area vs “Domain primitive” as a bounded context implementation unit.

This causes drift between the narrative backlogs and the proto-driven implementation work.
