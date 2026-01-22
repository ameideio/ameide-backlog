# Increment 2 — Requirements and derived backlinks with explainable “why linked”

This increment implements **Scenario B** end‑to‑end, but aligned to your **six primitives** (no separate “Memory primitive”). It delivers the first real **EA knowledge graph derivation**: stable IDs + inline refs → **derived backlinks/impact** with **origin citations** and deterministic edge cases (broken refs, duplicate IDs). 

It builds directly on Increment 1’s governed publish loop and Evidence Spine semantics (MR is proposal unit; canonical publish anchor is the **target branch head SHA after publish**). 

---

## 1) Capability user story and capability-level testing objective

### User story

* **As a human (architect)**, I can create a requirement (`REQ-001`) and a standard that references it, publish both through governed changes, and then open `REQ-001` and see:

  * a **Derived Backlinks/Impact** panel (what depends on this requirement)
  * a **“why linked?”** affordance showing the precise origin location of each ref
  * deterministic handling for broken/ambiguous cases (not silent) 
* **As an agent**, I can answer “what depends on REQ-001?” and produce a citeable impact summary by calling Projection queries and reading cited snippets; I cannot create relationships in a DB (because there is no canonical relationship store), and I cannot write without Domain.

### Capability-level testing objective (contract-pass)

One **capability-owned integration test pack** proves from empty state:

## Agentic deliverables (Scenario B)

Scenario B is where agentic “graph reasoning over Git” becomes real: the platform must return deterministic, citeable graph answers at a commit SHA, suitable for architect supervision and human review.

- **Architect agent deliverables**
  - Can answer “what depends on REQ-001?” by calling derived backlink/impact queries and returning origin-cited results (“why linked?”).
  - Can produce a citeable context bundle for `REQ-001` (and its immediate neighbors) that is reproducible at the resolved `read_context`.
  - Can supervise a developer working on requirements/standards by verifying that:
    - derived edges are explainable via origin citations,
    - broken/ambiguous references are surfaced (not auto-fixed),
    - and evidence is attached for any proposed remediation change.
- **Developer agent deliverables**
  - Can implement remediations as governed changes (fix broken refs, resolve duplicate IDs) and attach evidence; never “fix” derived views directly.
- **Human-in-the-loop deliverables**
  - Ambiguous ID and broken reference cases always produce an explicit human decision point (“choose canonical ID / fix reference”) recorded as evidence.

## EDA alignment (496)

1. Onboard mapping for `{tenant_id, organization_id, repository_id}`.
2. Publish two artifacts via **Domain** governed write loop:

   * `requirements/REQ-001.md` containing a stable ID `REQ-001`
   * `architecture/standards/standard-a.md` containing an inline reference `ref: REQ-001` 
3. Projection resolves `published → resolved.commit_sha` and indexes the published baseline.
4. Projection answers:

   * `GetElementById("REQ-001")` returns a single path + citation (or deterministic “duplicate id” error if applicable).
   * `ListBacklinks("REQ-001")` returns at least one backlink (standard-a) and every edge includes an **origin citation** (where the `ref:` token lived). 
5. Process instance exists for the intake flow (durable orchestration state recorded).
6. Agent produces an “impact summary” that is strictly citeable (citations refer back to the origin citations + target content).
7. UISurface can:

   * browse derived requirements index
   * open `REQ-001`
   * show backlinks + “why linked?” reveal
   * show “Broken ref” / “Duplicate ID” errors explicitly when those cases are triggered 
8. Integration exposes the same derived views via protocol adapter: **an MCP server (Integration primitive) that exposes Projection queries to coding agents**, without embedding semantics.

#### Required negative cases (same increment, same test pack)

* **Broken ref:** a file contains `ref: REQ-404` but no element claims `id: REQ-404` → Projection returns backlinks with target marked unresolved and origin citations; UI shows “Broken reference” deterministically. 
* **Duplicate ID:** two files claim `id: REQ-001` at the same SHA → `GetElementById` fails with explicit “duplicate id” error and includes candidate citations; UI does not pick a winner. 

> Note on “Memory”: Scenario B describes “Memory/context” as projection-owned and citeable. 
> Under your six-primitive model, we implement **context bundle retrieval as a Projection query**, and treat “memory” as **Agent workflow storage of citeable bundles** (no separate primitive, no uncited facts).

---

## 2) Primitive-by-primitive goals, what to build, and what to test

Below: every primitive advances (variable scope, never null).

---

## Domain — governed writes + publish-time artifact policy

### Increment 2 goals

1. Domain continues to be the **only canonical writer** (MR-backed publish).
2. Domain enforces minimum authoring policy for participating EA artifacts:

   * stable IDs required for eligible “EA item” files
   * (optional in the doc; **mandatory in this increment**) because we need deterministic graph semantics. 
3. Domain continues to be the **only owner of Evidence Spine**; Projection/Process/UI/Agent do not invent their own “evidence schema.”

### Build

* Extend publish workflow (from Increment 1) with an artifact validation step:

  * On `CreateCommit` or `PublishChange` (choose one consistently), validate:

    * if path is under `requirements/**` then it must contain stable ID field (per Scenario B MVP rule)
    * if path is under `architecture/standards/**` then stable ID is recommended, but at minimum refs are parseable (you can enforce IDs here too if you want uniformity)
* Evidence Spine enrichment:

  * include `changed_paths[]` and the `target_head_sha` for each publish
  * do not add any derived backlinks/graph results (those belong to Projection) 

### Domain individual tests

* **Unit:** publish rejects `requirements/REQ-001.md` if stable ID missing (explicit error, deterministic).
* **Unit:** publish accepts `requirements/REQ-001.md` when ID present and well-formed.
* **Unit:** EvidenceSpineViewModel remains Domain-owned and contains:

  * identity
  * MR iid
  * `target_head_sha`
  * changed paths list
* **Boundary/static:** UISurface/Agent/Process/Projection do not import `gitlab.com/gitlab-org/api/client-go` (Domain may); GitLab types never cross proto contracts.

---

## Projection — derived ID index + backlinks + “why linked” anchors (from Domain facts)

### Increment 2 goals

1. Projection owns:

   * canonical reads at SHA
   * derived, rebuildable read models (ID index, backlinks index) 
   * canonical+derived reads are served from projection state derived from Domain facts (outbox→Kafka); Projection does not call GitLab.
2. Derived edges are **explainable**:

   * every backlink includes an **origin citation** (path + anchor at the immutable SHA)
3. Edge cases are deterministic:

   * missing target IDs yield “unresolved reference” outcomes (with origin citations)
   * duplicate IDs yield explicit ambiguous errors (with candidate citations) 

### Build

Minimum derived model, exactly per Scenario B:

* **Stable identity extraction**

  * Parse `id: REQ-001` from frontmatter/metadata section in `requirements/REQ-001.md` 
  * Index: `element_id → {path, citation}` (citation anchored to `{repo_id, sha, path[, anchor]}`)

* **Inline reference extraction (MVP grammar)**

  * Parse `ref: REQ-001` tokens in body/content
  * Emit an origin anchor:

    * `fm:<field>` for frontmatter locations
    * `L<line>:C<col>` for body token positions (Scenario B MVP anchor strategy) 

* **Backlinks index**

  * `target_id → [ {source_path, origin_citation, ref_kind} ]`
  * Rebuildable from Git at SHA

* **Query surface (minimum)**

  * `GetElementById(scope, read_context, element_id)`
  * `ListBacklinks(scope, read_context, target_element_id)`
  * Optional but useful for UI/agent: `GetOriginSnippet(scope, read_context, origin_citation)` (returns a small excerpt around the anchor)

### Projection individual tests

* **Unit:** `GetElementById("REQ-001")` returns path + citation at published SHA.
* **Unit:** duplicate ID at same SHA → explicit “duplicate id” error with candidate citations (no arbitrary winner). 
* **Unit:** `ListBacklinks("REQ-001")` returns `standard-a.md` and includes origin citations (anchor points to the ref token).
* **Unit:** broken ref behavior is deterministic and surfaced as unresolved target, not silently ignored. 
* **Rebuildability:** indexing twice at same SHA yields identical backlinks output.

---

## Process — requirement intake orchestration (durable, non-writing)

### Increment 2 goals

1. Process owns a durable workflow for “requirement intake + publish + derive”.
2. Process does not write canonical artifacts; it coordinates by:

   * invoking Domain commands
   * invoking Projection rebuild/index commands (if you model “index now” as a command) or waiting for post-commit facts later.

### Build

A minimal durable process `req.intake` (and a second one `standard.update`, or one combined flow), with states like:

1. `STARTED`
2. `DRAFT_READY` (human or agent draft attached)
3. `CHANGE_OPENED` (Domain EnsureChange done)
4. `COMMITTED` (Domain CreateCommit done)
5. `PUBLISHED` (Domain PublishChange done; `target_head_sha` recorded)
6. `INDEXED` (Projection confirms index built for `target_head_sha`)
7. `DONE`

### Process individual tests

* **Unit:** workflow persists and resumes (durability) mid-flight.
* **Unit:** process cannot advance to `INDEXED` without Projection acknowledging the indexed SHA equals Domain’s `target_head_sha`.
* **Boundary:** process cannot access GitLab adapters; it can only call Domain/Projection seams.

---

## Agent — citeable impact summary + workflow memory

### Increment 2 goals

1. Agent can answer impact questions (“what depends on REQ-001?”) by:

   * calling Projection backlinks
   * reading cited content snippets
   * producing a citeable answer
2. Memory is treated as Agent workflow concern:

   * store the context bundle citations used to produce the output
   * never store uncited “facts”

### Build

* `Agent.ExplainImpact(scope, read_context, target_element_id)`:

  1. call `Projection.ListBacklinks(target_element_id)`
  2. optionally call `Projection.GetContent`/`GetOriginSnippet` for each backlink origin
  3. produce `impact_summary {text, citations_used}`

* Agent memory store records:

  * selected citations
  * the final output + citations

### Agent individual tests

* **Unit:** output includes citations used; reject/flag any claim without citations.
* **Unit:** agent can handle “broken refs” deterministically (reports “Unresolved: REQ-404” with origin citations).
* **Unit:** agent can handle “duplicate id” deterministically (reports ambiguity and includes candidate citations; does not pick one).

---

## UISurface — derived requirements UX + “why linked” explainability

### Increment 2 goals

1. UI makes “Derived vs Canonical” explicit:

   * Derived requirements list is projection-backed
   * Canonical editor opens the Git file at SHA
2. UI supports “open by element ID” (REQ-001), not path identity.
3. UI provides:

   * backlinks/impact panel
   * “why linked?” reveal (origin citation anchors)
   * explicit broken ref and duplicate ID experiences 

### Build

Minimum UI routes/components (aligned with Scenario B):

* **Derived Requirements index**

  * list `REQ-*` from Projection ID index
  * shows resolved SHA
* **Requirement detail modal/editor**

  * shows canonical file path + citation
  * derived backlinks panel:

    * each backlink row has “why linked?” opening the origin anchor
  * show deterministic error panels for:

    * duplicate ID
    * broken ref 

### UISurface individual tests

* **UI integration:** requirement index loads from Projection; no GitLab calls.
* **UI e2e smoke:** open REQ-001 → backlinks visible → “why linked?” opens cited location.
* **UI negative:** duplicate ID error UI renders candidate citations; broken ref UI renders origin citations.

---

## Integration — protocol tools + substrate isolation

### Increment 2 goals

1. Integration extends outward-facing protocols (MCP server = Integration primitive exposing Projection queries to coding agents) so vendor agents can access:

   * resolve id
   * backlinks
   * open origin snippet
2. Integration remains semantics-free:

   * it forwards to Projection/Domain; it doesn’t implement graph logic or ID rules.
3. GitLab remains a vendor substrate accessed by Domain internals (no direct GitLab access from UISurface/Agent/Process/Projection).

### Build

* Extend MCP server tool surface (Integration primitive exposing Projection queries to coding agents) (minimum):

  * `ea.resolve_id(element_id, read_context)`
  * `ea.backlinks(element_id, read_context)`
  * `ea.open_origin(origin_citation, read_context)`
* Ensure all tools return citeable outputs (citations unchanged from Projection).

### Integration individual tests

* **Unit:** MCP tools (Integration primitive exposing Projection queries to coding agents) route to Projection and return deterministic outputs (including errors for duplicate IDs).
* **Boundary/static:** UISurface/Agent/Process/Projection do not import `gitlab.com/gitlab-org/api/client-go`; only Domain imports GitLab clients; only Integration imports MCP protocol libs (Integration primitive exposing Projection queries to coding agents).

---

## Increment 2 “Done means done”

Increment 2 is complete only when:

* ✅ One capability-level contract-pass test demonstrates:

  * two publishes through Domain
  * Projection derived backlinks with origin citations
  * Agent impact summary is citeable
  * UI shows derived backlinks + “why linked?”
  * Integration exposes tools for vendor agents
* ✅ Each primitive has at least one new behavior + at least one primitive-owned test.
* ✅ No competing evidence spine schemas exist outside Domain.
* ✅ No relationship CRUD exists; all relationships are inline refs and derived backlinks. 

---

If you reply “next”, I’ll do **Increment 3**: “Proposal vs Published” review gate (impact preview across SHAs + human approval + process gate + agent-produced evidence packet), keeping the same structure and making sure **every primitive advances** again.
