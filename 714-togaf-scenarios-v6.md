---
title: "714 — v6 Scenario Slices (Product ladder + incremental cross‑primitive implementation & test spec)"
status: draft
owners:
  - platform
  - transformation
  - ui
  - repository
  - agents
created: 2026-01-21
updated: 2026-01-22
supersedes:
  - 713-v6-togaf-functional-scenarios-e2e-tests.md
related:
  - 496-eda-principles-v6.md
  - 520-primitives-stack-v6.md
  - 537-primitive-testing-discipline.md
  - 590-capabilities.md
  - 591-capabilities-tests.md
  - 656-agentic-memory-v6.md
  - 694-elements-gitlab-v6.md
  - 701-repository-ui-enterprise-repository-v6.md
  - 703-transformation-enterprise-repository-proto-v1.md
  - 704-v6-enterprise-repository-memory-e2e-implementation-plan.md
  - 705-transformation-agentic-alignment-v6.md
  - 717-ameide-agents-v6.md
  - 710-gitlab-api-token-contract.md
  - 714-togaf-scenarios-v6-00.md
  - 714-togaf-scenarios-v6-05.md
---

Below is a **re-imagined, coordinated incremental ladder** that treats your **six primitives as equal** and ensures **every increment advances every primitive** (variable depth is OK, but never “null”). It also bakes in your constraints:

* **Git/GitLab is a vendor substrate** (accessed by Domain internals via `gitlab.com/gitlab-org/api/client-go`; no wrappers), never a canonical platform API surface.
* **Domain owns canonical writes** (owner-only writes) and is the **only** owner of the **Evidence Spine** record/schema.
* **Projection owns all platform reads** and is **purely derived** (rebuildable, citeable) by consuming **Domain facts** (Kafka/outbox). Projection does not call GitLab.
* **Process orchestrates** (durable gates/timers) and must not become a writer.
* **Agent reasons and proposes** under grants/risk tiers; **Memory is an agent workflow concern** (Agent uses Projection for citeable context; Memory never becomes a platform truth store).
* **UISurface is the human product**, calling the same seams as Agent/Process.
* **Integration owns adapters** for external exchange (MCP server = Integration primitive exposing Projection queries to coding agents; imports/exports; etc.) and never owns business semantics.
* **GitLab client standard:** Domain internals use `gitlab.com/gitlab-org/api/client-go` (pin a single major version; currently v1), and GitLab client types MUST NOT leak across platform contracts.
* **MCP direction:** MCP reads route to **Projection** queries; MCP writes route to **Domain** commands.

This ladder is compatible with the shape/discipline you already codified in 714 Slice 0/Scenario A/Scenario B: contract-pass harness, identity/read_context/citations, MR-based publish, derived backlinks with “why linked,” etc.   

---

# Recap: platform principles and objectives

## Principles

1. **Git-first canonical truth**

* Canonical artifacts are Git files at immutable SHAs (baseline truth = commit SHA).
* GitLab is a storage/collaboration substrate, not a platform semantic surface.
* Vocabulary:
  * **Canonical** = `main` (the `main@sha` baseline).
  * **Transformation initiative** = branch (an `initiative_branch@sha`, typically wrapped by an MR).
* Relationship authoring is intentionally hybrid (Git-first + agent-friendly):
  * inline links/refs are allowed (prefer frontmatter `links:`; body `ref:` may remain as compatibility input),
  * path-based links are allowed as locators for agents (`target_path`), but stable identity is still `id`,
  * ArchiMate/BPMN relationships can be first-class items (`archimate.relationship`, `bpmn.relationship`) when the relationship itself carries attributes and must satisfy metamodel rules.
* “Things” the platform validates/indexes as EA knowledge are typed items (YAML frontmatter with `id`, `scheme`, `type`). Implementation artifacts (code/config/GitOps manifests) may remain untyped and are linked/traceable via `target_path` plus citations at `{repo, sha}`.

### Travelling requirement overlay (Inc 0 → Inc 5)

All increments reuse one stable requirement id so the trace story accumulates instead of resetting each slice:

* `REQ-TRAVEL-001` is referenced explicitly in every increment’s capability test narrative.

2. **Owner-only writes**

* Only **Domain** performs canonical writes (MR/branch/commit/merge) and holds write credentials.
* Everyone else requests writes via Domain commands.

3. **CQRS: Domain commands, Projection queries**

* Platform rule: clients (UI/Agent/Process) perform **reads only via Projection**.
* Domain is **command-only** at the platform seam; Domain reads **only** from Projection when it must validate preconditions (e.g., “approval recorded”), and reads GitLab internally to execute commands and emit facts.
* Projection reconstructs read models (canonical repository browse/open + derived IDs/backlinks/graphs/search) by consuming **Domain facts** from Kafka/outbox.
* Derived outputs are **rebuildable** and **citeable**.

### Enterprise Repository fact types (496 `io.ameide.*` naming)

Lock down the semantic identities (CloudEvents `type` strings) used for Enterprise Repository propagation, per `backlog/496-eda-principles-v6.md` (`io.ameide.<context>.fact.<name>.v<major>`).

**Owner facts (Domain; emitted only after commit):**

* `io.ameide.transformation.fact.enterprise_repository.repo_mapping_upserted.v1`
* `io.ameide.transformation.fact.enterprise_repository.change_committed.v1`
* `io.ameide.transformation.fact.enterprise_repository.change_published.v1`
* `io.ameide.transformation.fact.enterprise_repository.validation_failed.v1` (only if modeled as a durable audit outcome recorded by Domain)

**Derived facts (Projection; optional):**

* `io.ameide.transformation.fact.enterprise_repository.typed_item_indexed.v1`

**Emission rule (Git-backed outbox-equivalent):**

* Facts are emitted only after the Git outcome is durable and auditable:
  * `change_committed` after the commit SHA is created and Domain has durably recorded the audit pointer(s),
  * `change_published` after MR merge (or squash) is complete and Domain has recorded the publish anchor (`target_head_sha`) and other required audit pointers.

4. **Evidence Spine is Domain-owned**

* The Evidence Spine is Domain’s internal audit record for governed changes/publishes.
* The Evidence Spine schema is also used as a generic “run evidence envelope”; publish runs have additional required fields (e.g., `mr_iid`, `target_head_sha`).
* Other primitives **must not define competing “evidence spine” schemas**; they render/consume Domain’s view model.

5. **Process orchestrates, never writes canonical**

* Process coordinates long-running workflows (gates, timers, approvals) through commands and events; it never becomes “a writer.”
* GitLab guardrails can block a merge, but Process guardrails are what make governance **platform-owned and non-bypassable across channels** (UISurface + MCP): Process holds the durable state machine (restart-safe) and records approvals/evidence packets; Domain commands can validate required gate state by reading Projection views.
* Process is therefore “earned” in scenarios where the platform requires prerequisites GitLab cannot natively enforce in your semantics (e.g., “impact preview packet exists”, “approval snapshot is tied to the exact proposal head SHA”, timeouts/escalations).

6. **Agent proposes, under grants**

* Agent produces proposals/evidence, requests actions via Domain/Process seams, and operates under grants/risk tiers.
* **Memory is an Agent workflow concern**: memory stores/recalls *citeable* context bundles and decisions, but it never becomes canonical truth.

7. **UISurface is first-class**

* Human UX is non-optional: browse/edit/review/approve all must exist, even if minimal early.

### 2.6.1 Agent-first deliverables (interface-neutral)

Scenario Slices are written **agent-first**:

- Each slice must ship an **agent-usable capability** (architect/developer/human-in-loop).
- UI and Process are **first-class consumers** of the same contracts; they must not introduce parallel semantics.

Tool interface neutrality:

- In-platform agents should use SDK-backed capability tools/resources.
- External/devtool consumption can be served via MCP adapters (optional transport binding per `backlog/534-mcp-protocol-adapter.md`).
- The slice definition must specify **tool semantics**, not a specific transport.

### 2.7 Element-editor-first UX (no “raw file viewer” as the product surface)

8. **Integration is adapters only**

* MCP server (Integration primitive exposing Projection queries to coding agents), imports/exports, vendor APIs all live here.
* GitLab is a canonical-storage substrate and is accessed by **Domain internals** only (never directly by UI/Agent/Process/Projection).
* Integration does not define business semantics; it exposes **protocol surfaces** (e.g., MCP) that call platform seams.
* Pattern (apply everywhere): **MCP server = Integration primitive that exposes Projection queries to coding agents; it never calls GitLab and never invents semantics.**

## Objectives

* Deliver incremental end-to-end capability where **humans and agents both succeed**.
* Every increment has:

  * **one capability-level user story**
  * **one capability-level contract-pass test** (seedless; fresh repo mapping, deterministic outputs)
  * **primitive-level tests** proving each primitive’s contribution
* Maintain strict boundaries so you don’t drift into:

  * “GitLab API becomes a platform API”
  * “Projection writes truth”
  * “Process writes truth”
  * “Agent invents uncited facts”
  * “UI bypasses Domain/Projection”

---

# Program ladder overview

Each increment is a **vertical slice**: Domain + Projection + Process + Agent + UISurface + Integration all advance.

1. **Increment 0 — Onboard + Explore (Baseline read)**
2. **Increment 1 — Governed Change + Publish (Evidence Spine real)**
3. **Increment 2 — Typed EA Items + Derived Backlinks (Explainable traceability)**
4. **Increment 3 — Proposal vs Published Review Gate (Impact preview + approvals)**
5. **Increment 4 — Work Package delivery trace (EA ↔ code ↔ GitOps ↔ process assets)**
6. **Increment 5 — Schemes + metamodel validation (TOGAF/ArchiMate/BPMN/Markdown) + no-sidecar views**

This ladder intentionally “absorbs” your 714 scaffolding + Scenario A + Scenario B, but makes **UI/Agent/Process/Integration non-optional** in every increment.   

### 3.2 Agentic deliverables template (required for every slice)

Every slice must explicitly state the **incremental agentic capability** delivered by completing that slice, using the same structure each time:

- **Architect agent deliverables**
  - What new questions the architect agent can answer *directly from Git + projection* at a `read_context`.
  - What new **derived views** it can explain with “why” evidence (origin citations).
  - How it supervises the developer’s active work (review tick cadence; escalation rules) per `backlog/717-ameide-agents-v6.md`.
- **Developer agent deliverables**
  - What new execution it can perform safely (repo writes, verification front doors, evidence capture).
  - What new guardrails are enforced (scope/path/policy boundaries; fail-fast behavior).
- **Human-in-the-loop deliverables**
  - What new checkpoint(s) exist and how the decision is recorded into the evidence spine.

---

# Increment 0 — Onboard + Explore baseline

## 1) Capability user story

“As a human (architect) and as an agent, we can onboard a repository into a tenant scope and **browse/open citeable content at the Canonical (`main@sha`) baseline**.”

## Capability-level testing objective

A single contract-pass scenario proves, from **empty state**:

* repo mapping created for `{tenant_id, org_id, repo_id}`
* `read_context=published` resolves to a deterministic SHA
* browse tree + open one file returns citeable content
* Process run starts (no-op orchestration is fine, but durable instance id must exist)
* Agent produces a short proposal/summary that includes **only citeable references**
* Domain emits an **EvidenceSpineViewModel** (fields may be sparse here, but schema is consistent) 

**Implementation status (as of 2026-01-22)**

* ✅ Domain + Projection seams exist for Enterprise Repository (Git remote mapping; `ListTree`/`GetContent` with `published` → resolved SHA).
* ✅ Platform UI can browse Canonical (`main@sha`) and open files through proto seams (no direct GitLab routes).
* ✅ Capability test coverage exists for `0/OnboardExploreBaseline`.
* ⏳ Process/Agent portions of Increment 0 are not yet wired end-to-end in the same capability test.
* ⏳ Target-state refactor pending: Projection should serve reads from Domain facts (Kafka/outbox) (no GitLab calls).

## 2) Primitive goals, build items, and primitive-level tests

### Domain

**Goal**

* Own onboarding and the “audit surface” shape from day one.

**Build**

* `UpsertRepositoryGitRemote(scope, provider=gitlab, remote_id|remote_path)`
* Emit EvidenceSpine facts (Domain-owned schema; Projection serves queries)

**Test**

* Unit: rejects any write command that includes vendor credentials (boundary enforcement)
* Unit: emits EvidenceSpine facts with identity + mapping (even if empty proposal/publish fields)

### Projection

**Goal**

* Canonical read plane at resolved SHA with citations.

**Build**

* `ListTree(scope, read_context, path)` (response includes `read_context.resolved.commit_sha`)
* `GetContent(scope, read_context, path)` returns citeable content `{repo_id, sha, path[, anchor]}` (line-range anchors can come later)

**Test**

* Unit: deterministic NotFound behavior for missing path
* Unit: citations always include resolved sha and path

### Process

**Goal**

* Orchestration shell exists and carries identity/correlation.

**Build**

* `StartProcess(scope, process_key="repo.explore", inputs)` → `process_instance_id`
* Minimal persisted process state (even if no actions yet)

**Test**

* Unit: starting a process records correlation metadata and scope identity

### Agent

**Goal**

* Agent can consume citeable context and produce citeable proposals. Memory exists as workflow state, not platform truth.

**Build**

* `Agent.Propose(scope, goal, context_bundle)` where `context_bundle` is Projection-derived citations/excerpts
* Memory store: store `{citations, rationale}` for the run (no uncited facts)

**Test**

* Unit: fails if proposal contains statements without citations (enforce “cite-only” discipline)
* Unit: memory entries are only (citations + agent notes), never “facts”

### UISurface

**Goal**

* Human can browse and open content at Canonical (`main@sha`) baseline (non-optional).

**Build**

* Repository page:

  * show “Canonical main @ <sha>” (i.e., `read_context=published`)
  * tree browse and file open (opens the element editor)
  * show citation block

* Element editor (modal) — **target-state shell + composition** (reused by all increments)

  ```text
  Route (Next.js modal):
    /org/{orgId}/repo/{repositoryId}/element/{elementId}
      ↳ app/(app)/org/[orgId]/repo/[repositoryId]/@modal/(.)element/[elementId]/page.tsx
          ├─ <ElementRouteBootstrap elementId />
          └─ <ElementEditorModal />

  <ElementEditorModal />
    └─ <Dialog open={isOpen} onOpenChange=...>
        └─ <DialogPortal>
            ├─ <DialogOverlay />
            └─ <DialogPrimitive.Content data-testid="editor-modal">
                └─ <ElementEditorRoot elementId isFullscreen ...>
                    └─ <EditorModalChrome title subtitle kindBadge actions>
                        └─ Layout (flex)
                            ├─ Main (left)
                            │   ├─ Tabs: Document | Properties | Derived | Evidence
                            │   ├─ <EditorPluginHost elementId activeTab />
                            │   └─ <ModalChatFooter />
                            └─ Right sidebar (desktop)
                                └─ <RightSidebarTabs>
                                    └─ <ModalChatPanel elementId />
  ```

  ```text
  Target-state wiring (GitLab-backed Enterprise Repository; no DB element CRUD):
    - Canonical writes (initiative branch/MR to main)      → Domain (EnterpriseRepositoryCommand: EnsureChange/CreateCommit/PublishChange) → GitLab (client-go)
    - All reads (canonical + derived)                      → Projection (EnterpriseRepositoryQuery + derived views; derived from Domain facts via Kafka/outbox; no GitLab)
  ```

**Test**

* UI integration test: UI reads via Projection seams only (canonical + derived) (no GitLab direct calls)

### Integration

**Goal**

* Centralize all vendor adapters and protocol adapters.

**Build**

* `MCPAdapter` (MCP server = Integration primitive exposing Projection queries to coding agents) (optional protocol, but **adapter presence is non-optional**): expose at least read tools backed by Projection

**Test**

* Unit: MCP tool calls (Integration primitive exposing Projection queries to coding agents) map to internal Projection APIs

---

# Increment 1 — Governed change + publish (Evidence Spine real)

This is your Scenario A in “every primitive participates” form. 

## 1) Capability user story

“As an architect (human) and as an agent, we can **propose and publish** an Architecture Vision as Canonical (`main@sha`) content, with audit-grade evidence anchored to the **target head SHA after publish**.”

## Capability-level testing objective

From empty repo:

* Domain creates MR proposal, commits changes to:

  * `architecture/vision.md`
  * `architecture/statement-of-work.md`
* Domain publishes (merges) with SHA guards
* Projection reads `published` and resolves to new `target_head_sha`
* EvidenceSpineViewModel contains minimum publish fields (MR iid, proposal head SHA best-effort, target head SHA required) 

## 2) Primitive goals, build items, and primitive-level tests

### Domain

**Build**

* `EnsureChange → CreateCommit(actions[]) → PublishChange(expected_mr_head_sha)`
* Record Evidence Spine (canonical anchor = `target_head_sha`) 

**Test**

* Unit: publish fails fast on SHA mismatch
* Unit: evidence spine always records `target_head_sha` even if merge/squash SHA differs

### 5.5 Agent DoD (required in every slice)

Every slice must declare and satisfy agent-focused acceptance criteria:

- **Architect agent DoD**
  - can answer the slice’s primary questions using citeable reads at a resolved `read_context`,
  - can explain derived results (“why”) using origin citations,
  - can supervise the developer’s active work via periodic review ticks (`review_interval_minutes`) and either continue/adjust/escalate (`backlog/717-ameide-agents-v6.md`).
- **Developer agent DoD**
  - can execute the slice’s change or verification tasks and attach evidence (default front door: `./ameide test` for platform-code work).
- **Human-in-loop DoD**
  - there is at least one explicit checkpoint that requires a human decision, recorded into the evidence spine.

### Projection

**Build**

* Strong convergence rule: after publish, `published` resolves to `target_head_sha` (no stale cache)
* Reads are citeable at that SHA

**Test**

* Unit: published ref resolution updates deterministically after publish

### Process

**Build**

* Process `arch.publish_vision` orchestrates:

  * request draft (human/agent)
  * call Domain commands
  * wait for validation signal (even if basic)
  * then publish

**Test**

* Unit: process never calls GitLab; only Domain/Projection

### Agent

**Build**

* Agent can draft vision text and propose a change (no write creds)
* Memory stores “what citations were used to draft”

**Test**

* Unit: agent can propose file edits but cannot publish; must request Domain/Process

### UISurface

**Build**

* “Propose change” UX (MR-backed)
* “Save draft” -> Domain CreateCommit
* “Publish” -> Domain PublishChange
* Evidence view: show MR + target_head_sha 

**Test**

* UI: publish flow uses Domain commands; evidence displayed uses the Domain-owned EvidenceSpine schema

### Integration

**Build**

* No new deliverables (must remain compatible):

  * GitLab operations are implemented inside Domain internals (not an Integration primitive).
  * MCP reads route to Projection; MCP writes route to Domain.
  * Integration focuses on external exchange surfaces (MCP server = Integration primitive exposing Projection queries to coding agents) when/if required.

**Test**

* Unit: token scope least privilege; Domain enforces “no main direct writes”

---

# Increment 2 — Typed EA items + derived backlinks (explainable traceability)

This is Scenario B generalized and made non-optional across all primitives. 

## 1) Capability user story

“As a human and agent, we can create a requirement and later navigate **derived impact/backlinks with ‘why linked’ evidence**, without relationship CRUD.”

## Capability-level testing objective

From empty state:

* Publish `requirements/REQ-TRAVEL-001.md` with stable ID
* Publish `architecture/standards/standard-a.md` referencing `REQ-TRAVEL-001`
* Projection builds ID index + backlinks and returns origin citations
* UI shows backlinks and “why linked?” highlight
* Agent answers “what depends on REQ-TRAVEL-001?” with citations only 

## 2) Primitive goals, build items, and primitive-level tests

### Domain

**Build**

* Add publish-time validation hook (reject EA files missing stable ID if they’re in a governed EA path set)
* Evidence spine includes list of changed paths + target_head_sha

**Test**

* Unit: publish rejects missing stable IDs for eligible EA items (policy-driven)

### Projection

**Build**

* Parse stable IDs and inline ref grammar MVP (`ref: <ID>`, e.g. `ref: REQ-TRAVEL-001`) 
* Index:

  * `id -> path` (duplicate IDs produce deterministic “ambiguous” error)
  * backlinks with origin citation `{repo, sha, path, anchor}`

**Test**

* Unit: duplicate ID detection produces deterministic error + candidate citations
* Unit: “why linked” anchor is stable across rebuild at same sha

### Process

**Build**

* Process `req.intake`:

  * draft requirement (human/agent)
  * publish via Domain
  * trigger projection rebuild (via command/event; projection remains rebuildable)

**Test**

* Unit: process orchestration uses commands and observes facts; no writes outside Domain

### Agent

**Build**

* Agent “impact explainer” tool:

  * calls Projection backlinks
  * returns citeable summary
* Memory stores the context bundle used and the produced answer.

**Test**

* Unit: agent output must include backlinks origin citations (no uncited claims)

### UISurface

**Build**

* Requirement detail view:

  * canonical content tab (path + citation visible)
  * derived backlinks tab with “why linked” and broken/ambiguous surfaced explicitly 

**Test**

* UI: “why linked” opens cited location (anchor/line highlight) at immutable SHA

### Integration

**Build**

* MCP server expands (Integration primitive exposing Projection queries to coding agents):

  * `resolve_id`
  * `list_backlinks`
  * `open_origin` (why-linked snippet)
* Still calls Projection APIs; Integration contains protocol, not semantics.

**Test**

* Unit: MCP tool outputs (Integration primitive exposing Projection queries to coding agents) are strictly citeable; no data invented in Integration

---

# Increment 3 — Proposal vs published review gate (impact preview + approvals)

## 1) Capability user story

“As governance (human) and as an agent assistant, we can review a proposal vs Canonical (`main@sha`), see blast radius, approve, and publish—without bypassing Domain.”

## Capability-level testing objective

From empty state:

* Publish Canonical (`main`) with `principles/P-001.md` and one referencing artifact
* Create proposal MR changing P-001
* Projection can answer impact for:

  * published SHA
  * proposal head SHA
* Process enforces gate:

  * cannot publish until impact preview exists and approval recorded
* Domain publishes and EvidenceSpineViewModel records the publish anchor

## 2) Primitive goals, build items, and primitive-level tests

### Domain

**Build**

* Add `ReadContext` support for “proposal head” (either by change_id or MR iid resolved internally)
* Publish still SHA-guarded; Evidence Spine remains the single source for audit record

**Test**

* Unit: `PublishChange` refuses if proposal head advanced since approval snapshot

### Projection

**Build**

* Projection read APIs accept `read_context` that can address:
  * `published` (canonical `main@sha`)
  * explicit commit SHA
  * `proposal_ref` (initiative / MR head)
* Backlinks/impact computed at arbitrary SHA (published or proposal), derived from Domain facts (Kafka/outbox).

**Test**

* Unit: impact answers differ by SHA deterministically when content differs

### Process

**Build**

* `governance.review_gate`:

  * request impact preview (Agent)
  * collect approval (UI)
  * then authorize publish (Domain)

**Test**

* Unit: process state machine persists and can resume after restart (durable orchestration)

### Agent

**Build**

* Agent generates “impact preview” packet:

  * cites origin edges/snippets
  * includes deltas between published and proposal contexts
* Memory stores both contexts and citations used.

**Test**

* Unit: impact preview output is entirely citation-backed

### UISurface

**Build**

* Review screen:

  * “Published vs Proposal” context selector
  * impact preview panel (from Agent output)
  * approval button (records decision into Process/governance state, not into Git)

**Test**

* UI: approval cannot directly publish; it advances Process which then calls Domain

### Integration

**Build**

* MCP server exposes “impact preview” tool as read (Integration primitive exposing Projection queries to coding agents)

**Test**

* Unit: Integration does not interpret approvals; it just fetches substrate signals

---

# Increment 4 — Work package delivery trace (EA ↔ code ↔ GitOps ↔ process assets)

## 1) Capability user story

“As a planner and delivery team, we can turn a requirement into a work package, implement changes in code/GitOps/process assets, and see traceability and validation evidence.”

## Capability-level testing objective

From empty state:

* Publish `REQ-TRAVEL-001`
* Publish `WP-001` referencing `REQ-TRAVEL-001`
* Make a delivery change that touches:

  * one code location
  * one GitOps manifest
* Projection derives:

  * WP ↔ changed paths
  * GitOps resource identity (kind/name/namespace) for touched manifests
  * (deferred) code symbol neighborhood and BPMN/DMN element graphs (not required for Increment 4)
* Process runs “work package execution” orchestration, capturing validation trace pointer
* UI shows “Delivered by” derived panel; Agent produces delivery summary with citations

## 2) Primitive goals, build items, and primitive-level tests

### Domain

**Build**

* Evidence Spine enriched (still Domain-owned) with:

  * pipeline pointer or validation trace ref (not a competing schema elsewhere)
  * changed paths list

**Test**

* Unit: evidence spine always ties validation evidence to the same `target_head_sha`

### Projection

**Build**

* Extend derived models:

  * `TOUCHES`: commit/publish → paths
  * GitOps manifest parsing to resource IDs
  * (deferred) code symbol extraction and BPMN/DMN element extraction (not required for Increment 4)

**Test**

* Unit: derived code symbols/manifest resources are rebuildable from Git at SHA
* Unit: trace view returns origin citations to the exact files/sections

### Process

**Build**

* `wp.execute` orchestration:

  * ensure WP exists and is approved to start
  * wait for publish evidence spine
  * record “execution complete” state with pointers to Domain evidence

**Test**

* Unit: cannot mark “done” without a Domain evidence spine publish anchor

### Agent

**Build**

* Agent “delivery summarizer”:

  * uses Projection trace queries
  * produces citeable summary (changed code snippets + manifest snippets)
* Memory stores citations used.

**Test**

* Unit: summary includes citations to code/manifest line ranges or anchors

### UISurface

**Build**

* Work Package page:

  * “Implements REQ-TRAVEL-001” (derived)
  * “Delivered by: commits/paths/resources” (derived)
  * “Evidence” tab shows Domain Evidence Spine (no UI-owned schema)

**Test**

* UI: all traceability panels are explicitly labeled “Derived” and include “why” citations

### Integration

**Build**

* Add adapters as needed for:

  * pipeline/test report retrieval (read-only)
  * MCP server tools for trace queries (Integration primitive exposing Projection queries to coding agents) (so vendor agents can ask “what does WP-001 touch?”)

**Test**

* Unit: pipeline retrieval is adapter-only; business meaning is interpreted in Projection/Domain, not Integration

---

# Increment 5 — Schemes + metamodel validation (TOGAF/ArchiMate/BPMN/Markdown) + no-sidecar views

## 1) Capability user story

“As enterprise architecture, we can author and govern TOGAF artifacts and model ArchiMate/BPMN as Git-first items (elements + relationships + views), with deterministic schema + metamodel validation gated on commit/MR, and with citeable, rebuildable derived graphs for navigation and impact.”

## Capability-level testing objective

From empty state:

* Publish a minimal registry for schemes/types/rules (Git-authored, governed publish):

  * schemes: `markdown`, `togaf`, `archimate`, `bpmn`
  * types: at least one per scheme
  * rules: at least one metamodel rule set (ArchiMate or BPMN)
* Publish a minimal ArchiMate model as three item kinds (Git files with YAML frontmatter; no sidecars):

  * `archimate.element` (element with `element_kind`)
  * `archimate.relationship` (relationship with `rel_kind`, `from`, `to`, and optional attributes)
  * `archimate.view` (view with membership + layout stored in-file)
* Projection derives and serves:

  * a typed item index (`id → {scheme,type,path,citation}`)
  * a graph view across inline links + relationship-items with origin citations
  * impact/backlinks that include “why linked?” origin citations
* Domain enforces deterministic validation gates on publish:

  * YAML frontmatter parse
  * required fields present (`id`, `scheme`, `type`)
  * metamodel constraint checks for ArchiMate/BPMN relationship-items (and view membership must reference existing ids)
* Process orchestrates a review gate for “model change publish” (impact preview + approval + publish).
* Agent produces a citeable “model change impact preview” (what elements/relationships/views were added/removed/affected) and attaches it to the Process instance.
* UISurface can browse items by scheme/type, open an element/relationship/view, render a view diagram (from in-file layout), and show validation errors deterministically when violated.

## 2) Primitive goals, build items, and primitive-level tests

### Domain

**Build**

* Governed publish for registry + model items (MR-based) with evidence spine.
* Validation on publish (commit/MR gate) for:

  * frontmatter parse and required fields
  * scheme/type existence in registry
  * ArchiMate/BPMN relationship-item endpoint validity (type constraints) and metamodel constraints (rule set)
  * view membership references must exist at the `proposal_head_sha` being published
* Evidence spine includes `changed_paths[]`, `target_head_sha`, and (optionally) declared item ids touched (still Domain-owned).

**Test**

* Unit: publish rejects invalid frontmatter (deterministic error).
* Unit: publish rejects invalid ArchiMate/BPMN relationship-items (deterministic metamodel error).
* Unit: publish produces evidence spine with `target_head_sha` + `changed_paths[]`.

### Projection

**Build**

* Typed item index:

  * `id → {scheme, type, path, citation}`
  * deterministic “duplicate id” errors
* Derived graph across:

  * inline links (frontmatter `links:` and/or body `ref:` compatibility)
  * relationship-items (e.g., `archimate.relationship`, `bpmn.relationship`)
  * view membership edges (e.g., `archimate.view` includes elements/relationships; layout is data, not a sidecar)
* Impact/backlinks queries with origin citations (why linked).

**Test**

* Unit: duplicate id and broken ref behaviors remain deterministic and citeable.
* Unit: relationship-items produce edges with metamodel validation outcomes surfaced (valid/invalid with reasons).
* Unit: view membership references resolve or fail deterministically.
* Rebuildability: indexing the same SHA yields identical derived graphs and impact results.

### Process

**Build**

* `model.change_review` orchestration:

  * draft → validation preview → request agent impact preview → approval → publish

**Test**

* Unit: process gate requires agent impact preview + human approval before Domain publish.

### Agent

**Build**

* “Model change analyst”:

  * validates the proposal context (reads via Projection) and explains failures citeably
  * summarizes what changed in the model (elements/relationships/views) with citations
  * produces impact preview packets for Process review (citations only)

**Test**

* Unit: impact preview contains only citeable claims and includes “why linked?” origin citations.

### UISurface

**Build**

* Type-driven UI surfaces:

  * browse by scheme/type (TOGAF/ArchiMate/BPMN/Markdown)
  * open item (element/relationship/view) and show frontmatter-backed widgets
  * view renderer reads layout from the view item (no sidecars)
  * show validation errors as first-class outcomes (not hidden)

**Test**

* UI: model open/render/impact outputs are citeable and anchored to `main@sha` (or proposal head SHA for review flows).

### Integration

**Build**

* MCP server tools for model navigation and impact (Integration primitive exposing Projection queries to coding agents), e.g.:

  * `item.get(id|path, read_context)`
  * `graph.impact(id, read_context)`
  * `view.get(id, read_context)`

**Test**

* Unit: MCP tools are pass-through to Projection/Domain seams (no invented semantics), and outputs remain citeable.

---

# Global “no-optionals” Definition of Done per increment

Every increment is “Done” only if:

1. **Capability-level contract-pass test exists** under `capabilities/<cap>/__tests__/integration/…` (as you already set up in Slice 0 discipline). 
2. **Each primitive has at least one new behavior + one primitive-level test** proving it.
3. **UI and Agent are both exercised** in the capability test:

   * UI path: at least one UI-level flow (smoke integration) that calls the same seams
   * Agent path: at least one agent invocation producing citeable output
4. **Integration is never bypassed**:

  * no GitLab REST calls from Projection/UI/Agent directly (Domain only)
5. **Evidence spine is Domain-owned**:

   * any “report/run output” uses Domain’s EvidenceSpineViewModel; other primitives never invent a competing schema. 
