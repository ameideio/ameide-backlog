---
title: "714 — v6 Scenario Slices v6-05 (Increment 5 — Schemes + metamodel validation + no-sidecar views)"
status: draft
owners:
  - platform
  - transformation
  - ui
  - repository
  - agents
created: 2026-01-22
updated: 2026-01-22
related:
  - 496-eda-principles-v6.md
  - 520-primitives-stack-v6.md
  - 714-togaf-scenarios-v6.md
  - 714-togaf-scenarios-v6-00.md
  - 714-togaf-scenarios-v6-01.md
  - 714-togaf-scenarios-v6-02.md
  - 714-togaf-scenarios-v6-03.md
  - 714-togaf-scenarios-v6-04.md
---

# Increment 5 — Schemes + metamodel validation (TOGAF/ArchiMate/BPMN/Markdown) + no-sidecar views

This increment standardizes **typed “things”** as Git files with YAML frontmatter (minimum: `id`, `scheme`, `type`) and introduces **scheme/type registries** plus **metamodel validation** for modeling schemes where relationships carry semantics (ArchiMate and BPMN).

Key constraints preserved:

* Strict CQRS: UI/Agent/Process read via **Projection**; Domain is command-only at the platform seam.
* Domain is the only primitive that calls GitLab (Domain internals use `gitlab.com/gitlab-org/api/client-go`; no wrappers).
* Projection is purely derived by consuming Domain facts (Kafka/outbox); Projection does not call GitLab.
* MCP reads route to Projection; MCP writes route to Domain/Process (publish remains non-bypassable).
* No sidecars: views store membership + layout in the same Git file.
* “Relationship authoring” is hybrid:
  * inline links/refs for TOGAF/Markdown traceability,
  * relationship-as-item files for ArchiMate/BPMN where the relationship itself has attributes and metamodel constraints.

## Travelling requirement overlay (all increments)

Increment 5 converges the travelling requirement into typed graph + view semantics:

* `requirements/REQ-TRAVEL-001.md` remains the stable TOGAF requirement node.
* At least one model/view includes it explicitly:
  * as an `archimate.element` (Requirement) linked to `REQ-TRAVEL-001`, and
  * as a node in an `archimate.view` (no sidecars; layout stored in-file).

---

## 1) Capability user story and capability-level testing objective

### User story (human + agent)

* **As a human (architect)**, I can author and publish:
  * TOGAF artifacts (`scheme: togaf`) and general documentation (`scheme: markdown`) as typed items,
  * ArchiMate and BPMN models as **elements + relationships + views** (all typed items),
  and the platform blocks merges/publishes that violate schema or metamodel constraints.
* **As an agent**, I can propose model changes and produce citeable impact previews across items/relationships/views, but I cannot bypass validation or governance gates.

### Capability-level testing objective (contract-pass vertical slice)

One capability-owned integration test proves, from empty state:

1. **Publish registry (governed)**
   * Publish a minimal scheme/type registry into Git (typed items) supporting:
     * schemes: `markdown`, `togaf`, `archimate`, `bpmn`
     * types: at least one per scheme
     * widgets: `type → widgets[]` (type-driven UI composition)
     * rules: at least one metamodel ruleset (ArchiMate or BPMN)
2. **Publish a minimal ArchiMate model (governed)**
   * Publish:
     * `requirements/REQ-TRAVEL-001.md` as a typed TOGAF requirement (stable id; links are allowed)
     * one `archimate.element` representing the requirement (and linking back to `REQ-TRAVEL-001` via `links:`)
     * one `archimate.relationship` (`from`, `to`, `rel_kind`, optional attributes)
     * one `archimate.view` referencing those ids (including the requirement element), with layout stored in-file
3. **Validation gate is real**
   * A publish that introduces an invalid relationship (metamodel violation) fails deterministically (Domain gate).
4. **Projection derives a typed graph**
   * Projection answers:
     * `GetItemById` for element/relationship/view
     * `ListBacklinks/Impact` with origin citations (“why linked?”) across:
       * inline links/refs, and
       * relationship-items and view membership edges
5. **Process gate is exercised**
   * A model-change review workflow runs:
     * agent impact preview attached
     * human approval recorded
     * publish executed by Process calling Domain
6. **UISurface renders the model**
   * UI can browse by scheme/type, open items, render a view diagram, and show validation failures as first-class outcomes.
7. **Integration exposes the same reads via MCP**
   * MCP tools (Integration primitive) expose model reads/impact by routing to Projection queries (citeable outputs unchanged).

---

## 2) Primitive-by-primitive goals, build, and individual tests

### Domain

#### Increment 5 goals

1. Continue owner-only governed writes (MR-backed publish).
2. Make validation a deterministic publish gate for typed items:
   * schema (frontmatter parse + required fields),
   * registry lookup (scheme/type existence),
   * metamodel constraints for ArchiMate/BPMN relationship-items and view membership references.
3. Keep Evidence Spine Domain-owned (audit record for publishes; no competing schemas).

#### Build

* Validation is enforced by **one** deterministic validator (library + CLI) reused in:
  * local hook (best-effort),
  * CI merge request job (hard gate),
  * Domain publish gate (`PublishChange`, authoritative; always re-validates).
* Publish-time validation (commit/MR gate) includes:
  * YAML frontmatter parse for typed items and required fields: `id`, `scheme`, `type`
  * registry checks: `scheme/type` must be recognized
  * ArchiMate/BPMN relationship-item constraints:
    * endpoints exist
    * endpoints are of allowed types (e.g., element kinds)
    * `(from_kind, rel_kind, to_kind)` permitted by metamodel ruleset
  * view membership references exist (elements/relationships referenced by a view must exist at the proposal head SHA)

#### Domain tests

* **Unit:** invalid frontmatter fails deterministically.
* **Unit:** unknown scheme/type fails deterministically.
* **Unit:** invalid ArchiMate/BPMN relationship-item fails deterministically with a stable error code.
* **Unit:** EvidenceSpineViewModel includes `target_head_sha` + `changed_paths[]`.

---

### Projection

#### Increment 5 goals

1. Provide typed-item reads and derived views for agents and UI:
   * `id → {scheme,type,path,citation}`
   * rebuildable derived graph at a SHA
2. Treat relationship-items and views as first-class graph nodes/edges in derived queries.

#### Build

* Typed item index:
  * parse `{id, scheme, type}` from frontmatter
  * deterministic duplicate id errors
* Graph derivation:
  * inline links/refs (frontmatter `links:` preferred, body `ref:` compatibility)
  * relationship-items (e.g., `archimate.relationship`, `bpmn.relationship`)
  * view membership edges (view includes nodes/edges) and layout remains data stored in-file
* Query surface includes (names illustrative; align with proto):
  * `GetItemById`
  * `ListBySchemeType`
  * `Impact/Backlinks` + `GetOriginSnippet`

#### Projection tests

* **Unit:** typed item index produces `{scheme,type,path}` and citations at `main@sha`.
* **Unit:** relationship-items produce derived edges and “why linked?” origins are citeable.
* **Unit:** view membership references resolve deterministically (or error deterministically).
* **Rebuildability:** same SHA yields identical derived outputs.

---

### Process

#### Increment 5 goals

1. Orchestrate model-change review and ensure publish is non-bypassable.
2. Record approvals and agent evidence packets as durable workflow state.

#### Build

* `model.change_review` workflow (durable):
  * draft → validation preview → agent impact preview → approval → publish → indexed → done

#### Process tests

* **Unit:** cannot publish without agent impact preview + approval.
* **Unit:** restart-resume mid-process is deterministic.

---

### Agent

#### Increment 5 goals

1. Produce citeable model change impact previews (elements/relationships/views) and attach to Process.
2. Use Projection reads only; no uncited facts.

#### Build

* `Agent.GenerateModelImpactPreview(scope, read_context_published, read_context_proposal, target_ids[])`:
  * reads typed items and derived graph from Projection
  * emits an impact preview packet containing only citeable claims

#### Agent tests

* **Unit:** impact preview output contains citations only.
* **Unit:** validation failures are surfaced as citeable errors, not “auto-fixed” claims.

---

### UISurface

#### Increment 5 goals

1. Provide type-driven UI for typed items across schemes:
   * browse by scheme/type
   * open element/relationship/view
   * render view diagram from in-file layout (no sidecar)
2. Use the type registry as the source of truth for widget composition (ElementEditor/plugin host):
   * `type → widgets[]` controls which panels render (e.g., Properties, Derived/Impact, Evidence, Diagram Renderer, Validation Results)
   * `togaf.requirement` enables: Properties + Derived/Allocation + Evidence/Trace + Governance/Validation
3. Show deterministic validation outcomes and governance state.

#### UISurface tests

* **UI smoke:** open a view and render nodes/edges from the view file.
* **UI negative:** invalid relationship shows a deterministic validation failure (from Domain/Process/Projection seams).

---

### Integration

#### Increment 5 goals

1. Provide protocol adapters for external/devtool agent clients:
   * MCP reads → Projection queries
   * MCP writes → Domain/Process commands (publish remains non-bypassable)

#### Integration tests

* **Unit:** MCP tools are pass-through (no semantics invented) and outputs remain citeable.
