# 529 — ArchiMate Alignment for 470+ Backlogs (vocabulary + layering + relationships)

**Status:** Draft (tracking + coordination)  
**Audience:** Architecture, Transformation, platform engineering, doc owners  
**Scope:** Align Ameide documentation (`470+`) to **ArchiMate standard language** while preserving Ameide-specific implementation mapping (primitives, operators, EDA).

## 0) Goal

Make the `470+` backlog set readable and consistent in ArchiMate terms:

- consistent **layering** (Business / Application / Technology / Implementation & Migration),
- consistent **nouns** (Capability, Value Stream, Business Process, Application Service, Application Component, etc.),
- consistent **relationship verbs** (realizes, supports, uses, triggers),
- preserve Ameide-specific language as a **crosswalk** (e.g., primitives = application components).

## 1) Why now

We are introducing “Capability” as a first-class concept (`528-capability-definition.md`) and defining concrete capabilities (e.g., Transformation in `527-transformation-capability.md`, Commerce in `523-commerce*`). Without ArchiMate-aligned language:

- “process” is ambiguous (Business Process vs Process primitive),
- “domain” is overloaded (business domain vs bounded context vs Domain primitive),
- “capability decomposes into primitives” reads as same-layer decomposition (it is cross-layer support/realization).

## 2) Canonical crosswalk (repo-wide standard)

### 2.1 ArchiMate → Ameide mapping

- **Capability (Business)** → Ameide **Capability** (`523`, `527`, `528`)
- **Value Stream / Business Process (Business)** → value stream maps and golden paths (capability process sections)
- **Application Service / Interface / Event (Application)** → proto contracts (RPCs, topics, envelopes)
- **Application Component (Application)** → Ameide primitives:
  - Domain / Process / Projection / Integration / UISurface / Agent
- **Technology Service / Node / System Software (Technology)** → K8s, operators, brokers, DBs, gateways
- **Work Package / Deliverable / Plateau / Gap (Implementation & Migration)** → rollout plans, fit/gap, migration phases

### 2.2 Relationship verbs (required phrasing)

- Capability **is realized by** Value Streams / Business Processes
- Capability **is supported by** Application Services
- Application Service **is realized by** Application Components (primitives)
- Application Components **use** Technology Services
- Application Events **trigger** processes/workflows (Business or Application as appropriate)

**Rule:** avoid “capability decomposes into primitives” in normative docs; use the chain above.

## 3) Scope (documents to align)

Primary focus: all backlogs numbered `470+`, especially:

- `470-ameide-vision.md` (anchor)
- `471-ameide-business-architecture.md`
- `472-ameide-information-application.md`
- `473-ameide-technology.md`
- `477-primitive-stack.md` and `520-primitives-stack-v2.md`
- EDA + proto conventions: `496-eda-principles.md`, `496-eda-protobuf-ameide.md`, `509-proto-naming-conventions.md`
- Capability docs: `523-commerce*`, `527-transformation-capability.md`, `528-capability-definition.md`

## 4) Work plan (phased, low-churn)

### Phase A — Define the anchor language in 470 (minimal edits)

Update `470-ameide-vision.md` to include:

- ArchiMate layer model and the crosswalk in §2.
- Terminology rules:
  - “Business Process” vs “Process primitive”
  - “bounded context” vs “Domain primitive”
  - “Application Component” label for primitives
- Relationship verbs guidance.

### Phase B — Normalize 471/472/473 around the anchor

- `471-*`: ensure it uses Business-layer language (capabilities, value streams, policies) and points to application/tech where needed.
- `472-*`: ensure it uses Application-layer language (services, interfaces, events, data objects) and treats EDA contracts as application interfaces/events.
- `473-*`: ensure it uses Technology-layer language (nodes, tech services, operators, runtimes).

### Phase C — Normalize primitives + EDA docs (477/520/496/509)

- Label primitives taxonomy as **Application Components**.
- Label EDA/proto conventions as **Application Interfaces + Events** conventions.
- Remove accidental “business layer” terms from these docs unless explicitly doing traceability.

### Phase D — Normalize capability docs (523/527/528/524)

Adopt a consistent template:

1) Business layer: capability brief, value streams, policies, identity axes  
2) Application layer: services/events/interfaces + component (primitive) realization  
3) Technology layer: topology constraints (cloud/edge/offline), operator/runtime assumptions  
4) Implementation & Migration: phases and fit/gap

## 5) Acceptance criteria

1. `470-ameide-vision.md` defines ArchiMate-aligned vocabulary and is referenced as the canonical language anchor.
2. `471/472/473` each cleanly map to their intended layers and avoid overloaded terms.
3. “Process primitive” vs “Business Process” is consistently disambiguated across `470+`.
4. Capability docs consistently use “supported by / realized by” language (no same-layer decomposition ambiguity).
5. A short glossary exists in `470-ameide-vision.md` covering the overloaded terms: capability, domain, process, service, component.

## 6) Non-goals

- Rewriting historical research sections beyond terminology cleanup.
- Enforcing ArchiMate diagram syntax or tool selection in this backlog (only language alignment).
- Introducing a new primitive kind named “Capability”.

## 7) References

- `470-ameide-vision.md` (to be updated to host the canonical vocabulary section)
- `528-capability-definition.md` (capability concept)
- `524-transformation-capability-decomposition.md` (method: capability → EDA contracts → primitives)
- `527-transformation-capability.md` (capability decomposition example)
- `523-commerce*.md` (capability decomposition example)

