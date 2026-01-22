# Increment 4 — Requirement → Work Package → Delivered change trace (EA ↔ code ↔ GitOps) with governed publish

This increment is the first time we make **delivery** visible and explainable end‑to‑end:

* A **Requirement** is planned into a **Work Package** (both are canonical Git-authored EA artifacts, not DB rows).
* A governed change is published that touches **code** and **GitOps** (no frontmatter required in code/YAML).
* The platform derives and serves a **Work Package delivery trace** that is:

  * **rebuildable** (Projection-owned)
  * **citeable** (every item anchored to `{repo_id, sha, path[, anchor]}`)
  * **evidence-linked** (Domain Evidence Spine is the authoritative audit record for the publish)

This increment builds directly on:

* Slice 0 discipline: identity/read_context/citations/evidence view model. 
* Scenario A: governed write loop + MR proposal unit + publish anchor is **target branch head SHA after publish**. 
* Scenario B: stable IDs in files + inline refs + derived backlinks/impact are projection-owned and deterministic. 

---

## 1) Capability user story and capability-level testing objectives

### User story (human and agent equally important)

* **As a human (planner/architect)**, I can create a Work Package `WP-001` that references Requirement `REQ-001`, publish it, then execute the work package by publishing code + GitOps changes through the governed publish loop, and finally see “what this work package delivered” with citations and evidence.
* **As an agent**, I can:

  1. propose an implementation plan for `WP-001` grounded in citeable repository context, and
  2. produce a citeable “delivery summary” after publish, grounded in the delivery trace + Evidence Spine—without writing directly.

### Capability-level testing objective (contract-pass vertical slice)

One capability-owned integration test proves, from empty state:

1. **Publish planning artifacts (governed)**

   * Publish `requirements/REQ-001.md` with stable `id: REQ-001` (Scenario B rule). 
   * Publish `work-packages/WP-001.md` with stable `id: WP-001` and inline refs:

     * `ref: REQ-001` (WP implements requirement)
     * optional `refs:` list for BPC later (not required yet)

2. **Execute a governed delivery change associated with the work package**

   * Start a durable process instance `wp.execute` for `WP-001`.
   * Agent produces a citeable execution plan (which files to touch, checks to run).
   * Domain opens a change and commits edits to (at minimum):

     * one **code file** (e.g., `src/example.py`) — **no frontmatter**
     * one **GitOps manifest** (e.g., `gitops/app.yaml`) — **no frontmatter**
   * Domain publishes (merges) with SHA-safe semantics, producing `target_head_sha` as the canonical anchor. 

3. **Evidence Spine is authoritative**

   * Domain’s EvidenceSpineViewModel for the publish includes:

     * `mr_iid`
     * `target_head_sha` (**required**)
     * `changed_paths[]`
     * `declared_work_item_ids[] = ["WP-001"]` (new in this increment; still Domain-owned, still not a competing schema)

4. **Projection produces Work Package delivery trace (derived, citeable)**

   * Projection can answer `GetWorkPackageTrace(WP-001, read_context=published)` returning:

     * publish evidence pointer (mr iid + target_head_sha) *as references to Domain evidence*
     * changed paths (citeable to the published SHA)
     * derived GitOps resource identities for changed manifests (kind/name/namespace) with origin citations (path+anchor)
     * (optional in future increments) derived code symbol spans; in this increment, file-level trace is sufficient as long as it is citeable and rebuildable

5. **UISurface shows the end-to-end story**

   * WP detail screen shows:

     * Implements: `REQ-001` (derived from inline refs)
     * Deliveries: list of publishes tied to `WP-001` (from Domain evidence spine, rendered; not reinvented)
     * Delivered artifacts/resources: derived from Projection trace, with “why included” citations

6. **Agent produces a citeable delivery summary**

   * Agent calls Projection trace + reads cited snippets and produces a “delivery summary” with citations only.
   * Agent stores the citeable context bundle and summary in its internal workflow memory (no uncited facts).

7. **Integration exposes the trace through adapters** (MCP server = Integration primitive exposing Projection queries to coding agents)

   * At least one protocol adapter: MCP server (Integration primitive exposing Projection queries to coding agents) exposes `wp.trace(WP-001)` and returns the same citeable trace objects produced by Projection (no semantics invented in Integration).

#### Required negative assertions

* If the delivery publish is attempted without `declared_work_item_ids` including `WP-001`, then `GetWorkPackageTrace(WP-001)` must **not** “guess” by scanning commits; it returns “no deliveries recorded” deterministically (no heuristics yet).
* CQRS rule: UI/Agent/Process read via **Projection**; Domain is **command-only** at the platform seam.
* UI/Agent/Process/Projection never call GitLab APIs directly; only Domain uses `gitlab.com/gitlab-org/api/client-go`.
* Publish still fails fast on stale MR head SHA. 

---

## 2) Primitive-by-primitive goals, build, and individual tests

### Domain

#### Increment 4 goals

1. Keep owner-only writes and governed publish semantics (Scenario A). 
2. Make **work package association** an explicit, auditable part of the governed change (without inventing a second evidence schema elsewhere).
3. Continue to be the **only owner of the Evidence Spine**.

#### Build

* Extend the change lifecycle commands to accept **declared work item linkage**:

  * `EnsureChange(scope, idempotency_key, declared_work_item_ids[])` or
  * `CreateCommit(change_id, actions[], ..., declared_work_item_ids_delta)`
    Choose one and standardize; the goal is: by the time Publish happens, the Evidence Spine records the declared association.
* EvidenceSpineViewModel additions (Domain-owned):

  * `declared_work_item_ids[]`
  * `changed_paths[]` (already started in Inc 2/3)
  * keep `target_head_sha` as the canonical anchor for publish. 

> Important boundary: Domain records declared linkage and publish evidence; it does **not** compute “trace views.” That remains Projection-owned.

#### Domain tests

* **Unit:** Evidence spine includes `declared_work_item_ids=["WP-001"]` when provided.
* **Unit:** `changed_paths[]` recorded equals the set of commit actions (create/update/move/delete) that Domain executed.
* **Unit:** publish produces `target_head_sha` and remains SHA-safe. 
* **Policy test:** commands reject callers attempting to attach arbitrary “evidence spine fields” not owned by Domain (prevents schema drift).

---

### Projection

#### Increment 4 goals

1. Own a **rebuildable delivery trace read model** for Work Packages that joins:

   * EA semantics (WP ↔ REQ via inline refs) and
   * Delivery evidence (publishes linked to WP via Domain evidence spine) and
   * Implementation artifacts (changed paths, GitOps resources)
2. Serve the trace as citeable outputs at an immutable SHA (Scenario A/B citation rules).  
3. Do this without requiring metadata inside code/YAML files.

#### Build

Projection adds three derived indexes (all rebuildable):

1. **EA index enhancements**

   * Treat `work-packages/**` as eligible “element files” with stable IDs (same collision rules as Scenario B). 
   * Parse inline refs in `WP-001.md` to find `REQ-001` (already have ref parser from Inc 2).

2. **Delivery evidence join**

   * Consume Domain Evidence Spine records (or the post-commit fact that points to them) to build:

     * `work_item_id → [ publish_refs… ]` where each publish_ref contains `{mr_iid, target_head_sha, changed_paths[]}`.
   * This is explicitly not “Projection owns evidence”; it is “Projection builds a derived view over Domain-owned evidence.”

3. **Implementation artifact derivation**

   * For each `changed_path` at `target_head_sha`:

     * If under `gitops/**` (or matching yaml/k8s patterns), parse K8s resource identity (kind/name/namespace) and attach an origin citation to the manifest location (anchor may be “fm:” for YAML fields or line/col for token position, consistent with Scenario B’s anchor strategy). 
     * For code files: in this increment, derive “touched file” nodes with citations. (Symbol-level parsing can arrive in Increment 5; not required to ship delivery trace value now, but the trace must be correct and citeable.)

**Query surface (minimum)**

* `GetWorkPackageById(scope, read_context, "WP-001")` → `{path, citation}`
* `GetWorkPackageTrace(scope, read_context, "WP-001")` →

  * `implements[]` (requirements) with citations
  * `deliveries[]` each with:

    * `evidence_ref` (mr_iid + target_head_sha)
    * `changed_paths[]` with citations at `target_head_sha`
    * `gitops_resources[]` with citations (if applicable)

#### Projection tests

* **Unit:** WP ↔ REQ derived edge exists from inline refs and is deterministic. 
* **Unit:** given a Domain evidence spine record that declares `WP-001` and changed paths, `GetWorkPackageTrace(WP-001)` returns the publish in deliveries.
* **Unit:** GitOps manifest parsing returns expected resource identity with an origin citation.
* **Rebuildability:** rebuilding the trace for the same `target_head_sha` yields identical outputs.
* **Negative:** if no Domain evidence declares `WP-001`, trace returns “no deliveries recorded” (no heuristic inference yet).

---

### Process

#### Increment 4 goals

1. Provide a durable **work package execution workflow** that coordinates:

   * plan generation (Agent)
   * governed changes (Domain)
   * projection convergence (Projection)
2. Ensure “Done” for a work package execution is tied to **Domain’s publish evidence** (not UI state).

#### Build

Process workflow `wp.execute` (durable states, minimal but real):

1. `STARTED` (inputs: `work_package_id=WP-001`)
2. `PLAN_READY` (agent plan attached)
3. `CHANGE_OPENED` (Domain EnsureChange done; recorded `change_id`, `mr_iid`)
4. `COMMITTED` (Domain CreateCommit done)
5. `PUBLISH_REQUESTED` (approval state can be minimal here; stricter gates can come later)
6. `PUBLISHED` (Domain PublishChange returns `target_head_sha`)
7. `INDEXED` (Projection confirms trace/index built for `target_head_sha`)
8. `DONE`

Key rule:

* Process cannot mark DONE until:

  * it has Domain publish evidence (`target_head_sha`) and
  * Projection has indexed/confirmed trace availability for that SHA.

#### Process tests

* **Unit:** cannot reach DONE without a Domain publish response containing `target_head_sha`. 
* **Unit:** cannot reach DONE until Projection acknowledges indexing for the same SHA.
* **Durability:** restart-resume test mid-process.

---

### Agent

#### Increment 4 goals

1. Agent contributes product value in two places:

   * **Execution plan** (before delivery)
   * **Delivery summary** (after publish)
2. Memory is internal workflow state:

   * store citeable context bundles used for both outputs
   * store the produced artifacts, each with citations
   * never store uncited facts as “truth”

#### Build

Two agent capabilities:

1. `Agent.PlanWorkPackageExecution(scope, read_context, wp_id)`

   * reads WP file via Projection (citeable)
   * reads impacted requirements via Projection (citeable)
   * proposes a plan that identifies candidate files/areas to change (citeable where possible)

2. `Agent.SummarizeWorkPackageDelivery(scope, read_context, wp_id)`

   * calls `Projection.GetWorkPackageTrace`
   * opens cited snippets for changed files/resources (Projection reads)
   * produces a citeable summary: “What changed, what GitOps resources affected, what requirement it implements.”

#### Agent tests

* **Unit:** both plan and summary outputs must be citation-backed (reject or flag uncited claims).
* **Unit:** handles missing deliveries deterministically (“no deliveries recorded for WP-001” with explanation).
* **Unit:** memory entries store only `{citations, notes, outputs}`.

---

### UISurface

#### Increment 4 goals

1. Provide a **Work Package product surface**:

   * create/view WP
   * see “implements” requirements (derived)
   * run/track execution (process state)
   * see deliveries and evidence (Domain evidence + Projection trace)
2. Ensure UI never bypasses ownership:

   * publish goes through Process → Domain
   * reads go through Projection

#### Build

Minimum screens:

1. **Work Packages index (Derived)**

   * projection-backed list of `WP-*` IDs and titles, resolved at `published @ sha`.

2. **Work Package detail**

   * Canonical tab: open `work-packages/WP-001.md` with citation
   * Derived tab:

     * Implements: list of `REQ-*` from inline refs (with “why linked?” origin citations)
     * Deliveries: list of publishes associated with WP (from Projection trace, linked to Domain evidence spine refs)
     * Delivered artifacts/resources: changed paths + parsed GitOps resources, each citeable

3. **Execute Work Package**

   * start process `wp.execute`
   * show process state progression
   * show agent plan (with citations)
   * provide a minimal “edit files” flow (text editor sufficient) that results in Domain commit actions on a change branch (no direct git writes)
   * publish action triggers Process publish request

#### UISurface tests

* **UI smoke:** create/view WP → execute → see deliveries after publish.
* **UI integration:** “Deliveries” view renders Domain Evidence Spine data by reference; UI does not invent an evidence schema.
* **Boundary test:** UI never imports GitLab clients.

---

### Integration

#### Increment 4 goals

1. Extend substrate adapter support to include any minimal signals needed to:

   * create commits with multiple file actions (already needed for Scenario A) 
   * (if used) fetch pipeline status pointer for Domain to record in evidence (read-only, adapter-only)
2. Provide at least one external protocol surface that exposes WP trace to vendor tools (adapter-only; semantics remain Projection/Domain).

#### Build

* GitLab adapter enhancements (if not already):

  * commit actions support for updating code and YAML files in one commit (Domain calls adapter; adapter calls GitLab commit API). 
  * optional: pipeline status fetch (so Domain can store a pointer in Evidence Spine; still Domain-owned schema)

* MCP server tools (Integration primitive exposing Projection queries to coding agents) (minimum)

  * `wp.get(wp_id, read_context)` → routes to Projection
  * `wp.trace(wp_id, read_context)` → routes to Projection
  * `wp.status(process_instance_id)` → routes to Process

#### Integration tests

* **Unit:** MCP tool routing (Integration primitive exposing Projection queries to coding agents) is pass-through to Projection/Process (no semantics implemented in Integration).
* **Boundary/static:** only Domain imports GitLab client libs; only Integration imports MCP libs (Integration primitive exposing Projection queries to coding agents).
* **Contract:** adapter honors SHA-guard behaviors for merge/publish (already required), and supports the multi-file commit action set.

---

## Increment 4 “Done means done”

Increment 4 is complete only when:

* ✅ One capability-level contract-pass test proves:

  * REQ → WP planning
  * WP execution orchestrated by Process
  * governed publish associated to WP via Domain-declared linkage
  * Projection trace shows delivered changed paths + GitOps resources, citeable
  * Agent produces citeable plan + citeable delivery summary
  * UI renders WP page with implements + deliveries + evidence
  * Integration exposes WP trace through protocol adapter
* ✅ Each primitive shipped at least one new behavior and has at least one primitive-owned test.
* ✅ Evidence Spine remains Domain-owned; others only reference/render it. 
* ✅ No frontmatter is required in code/YAML files; traceability comes from:

  * canonical EA artifacts + inline refs (Scenario B) 
  * Domain-recorded declared linkage + changed paths in Evidence Spine (Scenario A anchor rules) 
  * Projection-derived parsing of manifests/resources at immutable SHAs.
