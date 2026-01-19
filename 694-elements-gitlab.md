> **Superseded:** replaced by `backlog/694-elements-gitlab-v6.md` (Git-backed canonical content posture; maintained as a stable v6 decision set).

Here’s a reworked, **cleaner set of architecture decisions** that matches **“GitLab hidden, platform‑enforced tenancy”** and your new framing (Enterprise Repository, Transformation, governance, relationships).

---

## Decision 1: Tenancy is owned by the platform, GitLab is a private subsystem

**Decision**

* Tenants do **not** have accounts in GitLab.
* GitLab is reachable only from platform services (network policy / private ingress).
* GitLab is deployed and operated as an **in-cluster platform component** (GitOps-managed), not a tenant-facing SaaS surface.
* The platform uses a **service identity** (bot) to create/manage GitLab projects, branches, MRs, tags.

**Why**

* Keeps tenancy boundaries and UX entirely in your product.
* Avoids coupling tenant RBAC to GitLab RBAC.

**Consequence**

* GitLab is “internal infrastructure” (operators/admins can see everything, but tenants cannot).

**How this evolves prior work (superseding direction)**

This keeps the long‑standing platform posture that **tenancy boundaries live in Ameide**, not in third‑party subsystems (identity, RBAC, governance), but makes the “git provider” a strictly internal dependency. It is compatible with the general primitives boundary language in `backlog/470-ameide-vision.md`, and it reframes older repository/graph docs (e.g. `backlog/200-300/211-repository-architecture.md`) as *platform UX/service boundaries* rather than “tenant logs into GitLab”. In other words: GitLab becomes an implementation detail behind platform-owned tenancy, instead of a tenant-facing tool.

---

## Decision 2: Enterprise Repository equals a GitLab Project repository

**Decision**

* **Enterprise Repository (TOGAF repository equivalent) = one GitLab Project repo**.
* All architecture/requirement artifacts are **just files** (any type).

**Why**

* Git already provides immutable history + collaboration + diffs.
* GitLab provides storage, browsing, MRs, CI hooks.

**Consequence**

* You avoid inventing an “element model” unless you later *choose* to add metadata conventions.

**How this evolves prior work (superseding direction)**

This is a deliberate pivot away from the `303/527` posture where **Elements/Relationships/Versions in Postgres are canonical** (`backlog/300-400/303-elements.md`, `backlog/527-transformation-domain.md`). Under this direction, canonical “repository truth” is **the GitLab project repository** (one GitLab project per tenant repository), and “element/relationship” semantics are expressed as **files and conventions** rather than database-first schemas. Postgres remains for tenant identity + governance/workflow state only (the minimum needed to enforce policy, approvals, sequencing, and audit pointers). Any indexing/search/graph traversal becomes a derived read model over the Git repository contents, not a second source of truth.

---

## Decision 3: Baseline equals `main`; transformations are branching off `main`

**Decision**

* **Baseline = `main` branch**.
* **Transformation = one or more branches from `main`**, merged back via MR(s).

**Why**

* This maps directly to Git’s strengths:

  * baseline is what’s accepted
  * transformations are proposed change sets

**Consequence**

* “Published” can mean “merged to `main`”.
* If you need immutable audit anchors (often you will), add tags later (“published snapshot”), but baseline remains `main`.

**How this evolves prior work (superseding direction)**

This replaces the older “baseline = `{element_id, version_id}` pointer set” concept from the Elements-in-Postgres approach (`backlog/300-400/303-elements.md`, `backlog/527-transformation-domain.md`) with a Git-native baseline: **baseline/published == commit SHA on `main` (and optionally tags)**. It also reshapes the “draft → review → approve → publish” journey framing (`backlog/200-300/220-ci-automation-patterns.md`) into Git terms: draft = branch, review = MR, publish = merge to `main` + optional tag. The platform still owns the semantics of “what counts as a baseline” and “what is allowed to merge”, but the canonical artifact history is the Git DAG.

---

## Decision 4: Transformation is a platform-level object (TOGAF “Request for Architecture Work”)

**Decision**

* **Transformation** is a **first-class object in your platform DB**, not a GitLab primitive.
* A Transformation:

  * points to 1+ Enterprise Repositories (GitLab projects)
  * spans multiple workstreams across the ADM cycle
  * owns the lifecycle + status of those workstreams

**GitLab mapping**

* Each Transformation creates **multiple branches/MRs** over time, e.g.:

  * Phase A MR (Vision)
  * Phase B MR (Business)
  * Phase C MR(s) (Data/App)
  * Phase D MR (Technology)
  * etc.

**Why**

* GitLab has “Epics” and “Requirements”, but they are **licensed features**:

  * Epics: **Premium/Ultimate** ([GitLab Docs][1])
  * Requirements management: **Ultimate** ([GitLab Docs][2])
* You don’t want your core product model to depend on GitLab tiers if GitLab is hidden/internal.

**Consequence**

* Optionally, for internal operator visibility, you *can mirror* a Transformation to a GitLab Issue/Epic/Requirement **when available**, but canonical truth stays in your platform.

**How this evolves prior work (superseding direction)**

This is consistent with the broader direction that Transformation is a platform-owned bounded context (initiative/workstream lifecycle, governance state, sequencing), but it changes what the Transformation Domain “owns” as canonical content. In the older 527/303 posture, Transformation Domain is the single-writer for the element substrate (`backlog/527-transformation-domain.md`); in this new posture, Transformation is still the system-of-record for *governance and lifecycle*, while GitLab holds the canonical artifact corpus (files). GitLab’s planning features (epics/requirements) remain optional mirrors because they are tiered/licensed; the platform’s Transformation model cannot depend on them for correctness.

---

## Decision 5: Governance is reimplemented in the platform; GitLab is the enforcement surface

**Decision**

* You implement a **governance policy engine** in your platform (tailored to TOGAF + your personas).
* Enforcement uses GitLab primitives:

  * merge rights restricted to a bot / governance role
  * pipeline must pass before merge
  * MR metadata (labels, templates, discussions) as evidence

**Why**

* On GitLab Free, MR “approvals” are **optional** and do not block merges. ([GitLab Docs][3])
* You want a consistent governance model regardless of GitLab tier.

**“Follow GitLab EE design patterns” (without copying EE code)**
What to take as patterns (conceptual):

* Explicit **feature identifiers** and **guards** (a feature is usable only if policy says so)
* Central registry of capabilities / checks (like GitLab’s licensed-feature registry concept) ([GitLab Docs][4])
* Test helpers/stubs around “feature available” decisions (so policies are testable)

**Consequence**

* Your platform controls merges by making “policy success” the gate, not GitLab Premium approval rules.

**How this evolves prior work (superseding direction)**

This directly supersedes older governance designs that assumed “domain-owned lifecycle + domain-owned canonical content” (`backlog/200-300/151-enterprise-repository-functional.md`, `backlog/200-300/154-enterprise-repository-implementation-plan.md`). The *intent* stays the same (platform-defined policies, evidence, required checks), but the enforcement point becomes **MR merge eligibility** and **pipeline gating**, with Postgres storing the governance decisions, policies, and audit references (MR IDs, commit SHAs, tag refs) rather than storing the full element graph. This also aligns with the “pipelines mirror production paths” concept (`backlog/200-300/220-ci-automation-patterns.md`) while removing dependence on GitLab tier features like required approvals or MR dependencies for correctness.

---

## Decision 6: Relationships are content hyperlinks + derived graph + LLM consumption

**Decision**

* **No first-class “relationship object” in GitLab.**
* Relationships are represented as:

  1. **hyperlinks inside artifacts** (primarily Markdown), and
  2. a **derived graph** computed by your platform indexer
  3. optionally enriched by an LLM agent (semantic linking/suggestions)

**GitLab support for hyperlinks**

* GitLab Flavored Markdown supports **relative links to repo files** and paths. ([GitLab Docs][5])
* GitLab UI supports **permalinks** to files/dirs/line ranges/anchors (useful when you want stable references). ([GitLab Docs][6])

**Derived graph**

* Platform indexer parses:

  * Markdown links
  * file paths + conventions
  * optional metadata if you introduce it later
* Store graph in platform DB for:

  * impact analysis
  * traceability views
  * navigation in your business UI

**LLM agentic consumption**

* The LLM consumes:

  * artifacts (retrieval)
  * derived graph (context)
  * MR diffs (change intent)
* Output: summaries, missing-link suggestions, “what changed and what it impacts”, etc.

**Consequence**

* Relationships are lightweight and author-friendly, while still enabling a “graph experience” in your frontend.

**How this evolves prior work (superseding direction)**

This reframes “relationships” away from being primarily database rows (as in the 303/527 element substrate: `backlog/300-400/303-elements.md`, `backlog/527-transformation-domain.md`) into a Git-first representation: **relationships live as text** (both inline links inside element files *and* standalone relationship files such as `<id>.yaml`), and the platform derives navigation/traceability/impact views by indexing those files. The earlier “Graph is read-only projection” posture (`backlog/470-ameide-vision.md`) still applies to the *derived graph experience*, but the canonical source is now the Git repository, not a domain database. This also fits the semantic retrieval direction in `backlog/535-mcp-read-optimizations.md`: agents consume artifacts + a derived graph/index as context, but canonical authoring happens in Git.

---

## Decision 7: ADM “DAG” is orchestrated by the platform; GitLab dependencies are optional

**Decision**

* The platform models ADM sequencing (“Phase E cannot publish before B/C/D complete”).
* If you have GitLab Premium/Ultimate you can optionally use MR dependencies, but you don’t rely on them.

**Why**

* GitLab MR dependencies are a **Premium feature**, and Free projects can’t be marked dependent. ([GitLab Docs][7])

**Consequence**

* Your platform is the source of truth for sequencing, even if GitLab can’t represent it natively.

**How this evolves prior work (superseding direction)**

This keeps the “platform owns workflow semantics” idea found throughout the Transformation specs (e.g., TOGAF ADM flows in `backlog/527-transformation-scenario-togaf-adm.md`), but rebinds the units of work to Git-native primitives (branches/MRs/commits) rather than element-version promotion sets. Sequencing (phase gates, required evidence, cross-repo dependencies) remains a platform-level concern stored in Postgres and enforced at merge time; GitLab dependency features remain optional sugar when present, but correctness does not depend on GitLab tier features.

---

If you want, next step is to write this as 6–10 short **ADRs** (each: decision, rationale, consequences, non-goals) plus a minimal “Transformation → branches/MRs” state machine (statuses + allowed transitions) that your policy engine will enforce.

[1]: https://docs.gitlab.com/user/group/epics/manage_epics/?utm_source=chatgpt.com "Manage epics"
[2]: https://docs.gitlab.com/user/project/requirements/?utm_source=chatgpt.com "Requirements management | GitLab Docs"
[3]: https://docs.gitlab.com/user/project/merge_requests/approvals/?utm_source=chatgpt.com "Merge request approvals"
[4]: https://docs.gitlab.com/development/ee_features/?utm_source=chatgpt.com "Guidelines for implementing Enterprise Edition features"
[5]: https://docs.gitlab.com/user/markdown/?utm_source=chatgpt.com "GitLab Flavored Markdown (GLFM)"
[6]: https://docs.gitlab.com/user/project/repository/files/?utm_source=chatgpt.com "File management"
[7]: https://docs.gitlab.com/user/project/merge_requests/dependencies/?utm_source=chatgpt.com "Merge request dependencies"
