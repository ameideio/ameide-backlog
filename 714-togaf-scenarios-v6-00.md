# Increment 0 — Onboard and Explore Baseline

## Outcome

A human and an agent can **onboard a repository into a tenant scope**, resolve `published` to a deterministic commit SHA, browse the Git tree, open citeable content, start a process instance, and produce an agent proposal that is strictly citation-backed—while enforcing the boundaries (no GitLab access from UISurface/Agent/Process; owner-only writes via Domain; reads via Projection).

This is “Slice 0” made **non-optional across all six primitives** (UI and Agent included) and with **Memory moved into Agent workflow**, while keeping the contract-pass scaffolding posture. 

## GitLab adapter standard (applies to all increments)

* **Single GitLab Go client:** `gitlab.com/gitlab-org/api/client-go` (pin a single major version; currently v1).
* **No wrapper clients:** Domain implementation code may call `client-go` services directly (the SDK *is* the adapter).
* **No leaking types:** GitLab client types never cross platform contracts (proto/SDK/view models).
* **Testing posture:**
  * **Primitive unit tests:** Domain uses `gitlab.com/gitlab-org/api/client-go/testing` (gomock) to validate handler logic without HTTP; Projection tests validate outbox→derived apply (no GitLab client imports).
  * **Capability/integration tests:** use gomock-based GitLab **mocks** (vendor `client-go/testing` or equivalent) and deterministic outbox→projection apply; no real GitLab dependency.

## Validator standard (applies to all increments)

Define exactly one deterministic typed-item validator (library + CLI) and reuse it in:

* local developer hook (best-effort)
* CI merge request job (hard gate)
* Domain `PublishChange` (authoritative; Domain always re-validates)

---

## Implementation progress (as of 2026-01-22)

* ✅ Enterprise Repository Domain command seam implemented for Inc 0 scaffolding (Git remote mapping + change/commit/publish flows) using `client-go` directly.
* ✅ Enterprise Repository Projection query seam implemented (`ListTree`, `GetContent`) including `published` → resolved SHA.
* ✅ Platform repository page wired to proto seams for “Canonical (main@sha)” + tree + file open (no direct GitLab routes).
* ✅ Capability integration coverage added for v6 scenario `0/OnboardExploreBaseline`.
* ✅ Domain primitive unit tests added using `gitlab.com/gitlab-org/api/client-go/testing` (gomock).
* ⚠️ Current deviation (implementation): Projection read seams still call GitLab; target state is derived reads from Domain facts only (Kafka/outbox).

## 1) Capability user story and capability-level testing objective

### User story (human + agent, equally important)

* **As a human (architect)**, I can onboard a repository, browse the canonical tree at `Canonical main@<sha>` (i.e., `read_context=published`), open a file, and see its citation.
* **As an agent** (via the platform Agent seam), I can retrieve a citeable context bundle for that same file and generate a short proposal/summary that contains **only citeable statements**.

### Capability-level testing objective (contract-pass)

## Agentic deliverables (Slice 0)

Slice 0 is still “plumbing”, but it must already be usable by agents as an execution substrate:

- **Architect agent deliverables**
  - Can retrieve a citeable context bundle for a selected artifact at a resolved `read_context`.
  - Can produce a citeable proposal that references returned citations (no uncited claims).
  - Can supervise a developer run at a very low fidelity: the harness can emit at least one “review tick” record into the evidence spine (even if the tick is a stub that only verifies “evidence exists”).
- **Developer agent deliverables**
  - Can run the minimal verification front door used by the harness and attach the results to evidence (even if the command is a stub in Slice 0).
- **Human-in-the-loop deliverables**
  - The runner exposes one explicit “stop/continue” decision point (even if simulated in the harness) and records it as evidence.

## EDA alignment (496)

A single **capability-owned integration test** (run by `ameide test`, Phase 0/1/2) proves from empty state:

1. **Identity creation**: `{tenant_id, organization_id, repository_id}` is used everywhere.
2. **Onboarding**: Domain records a repo mapping for this identity (local/in-memory provider is acceptable for Inc 0).
3. **Read context resolution**: Projection resolves `published` to a deterministic `resolved.commit_sha`. 
4. **Canonical reads**: Projection lists tree + reads `requirements/REQ-TRAVEL-001.md` (typed item with YAML frontmatter `id: REQ-TRAVEL-001`, `scheme: togaf`, `type: togaf.requirement`); responses include citations `{repository_id, commit_sha, path[, anchor]}`. 
5. **Process instance**: Process starts a durable process instance `repo.explore` (or equivalent) and returns `process_instance_id`. 
6. **Agent proposal**: Agent consumes Projection-provided citeable context and produces a proposal with citations (no uncited facts). 
7. **Run report**: Projection serves an `EvidenceSpineViewModel` for the run using the **Domain-owned schema** (fields not applicable in Inc 0 may be empty). 

**Negative assertions (must pass):**

* UI/Agent/Process/Projection do not call GitLab directly; only Domain calls `client-go`, and GitLab types never cross the proto seam.
* Missing path yields a deterministic error outcome (no silent empty lists). 

## Travelling requirement overlay (all increments)

All v6 increments use one stable requirement id to make end-to-end traceability mechanically hard to break:

* `REQ-TRAVEL-001` (typed item) exists at `published@sha` from Increment 0 onward.
* Increment 0’s explored/opened artifact is `requirements/REQ-TRAVEL-001.md` (not a generic README).

### Placement

* The test lives under the **capability composition boundary** (as Slice 0 already mandates) and exercises all primitives end-to-end. 

---

## 2) Primitive-by-primitive plan (build + individual tests)

Below, “build” means “new code/API/behavior introduced by Increment 0”, and “primitive tests” are what each primitive must prove independently.

---

## Domain — canonical mapping + boundary + evidence schema presence

### Increment 0 goals

1. Domain owns the **canonical repository mapping** (platform truth for which vendor remote a repository_id points to).
2. Domain enforces the **owner-only write boundary at interfaces** (nobody supplies vendor credentials; Domain is the only place that could ever hold them). 
3. Domain **owns the EvidenceSpine schema** (as a contract type), even if Inc 0 populates only identity/mapping/citations summary.

### Build (Increment 0)

* **Command:** `UpsertRepositoryGitRemote(scope, provider, remote_id|remote_path)`
* **Evidence shape availability:** `EvidenceSpineViewModel` type is available in Domain contracts; Domain emits EvidenceSpine facts (outbox) for runs.

### Domain individual tests

* **Unit test: mapping round-trip**

  * Upsert mapping → Get mapping returns same data (scope + provider + remote_id|remote_path)
  * Identity must match exactly.
* **Unit test: credential boundary**

  * Any command that attempts to include vendor credentials in its payload is rejected (compile-time or runtime validation).
* **Contract test: EvidenceSpine facts minimal emission**

  * For a read-only run, Domain emits EvidenceSpine facts containing:

    * identity
    * repository mapping reference
    * citations list (passed-through)
    * summary string
  * “publish/proposal” fields may be empty in Inc 0, but schema is stable (important for later increments). 

---

## Projection — derived read models from Domain facts (outbox→Kafka)

### Increment 0 goals

1. Projection is the **only owner of read APIs** at the platform seam (CQRS).
2. Projection reconstructs the read models by consuming **Domain facts** (outbox→Kafka), not by calling GitLab.
3. Every read is **citation-grade**.

### Build (Increment 0)

* **Query:** `ListTree(scope, read_context, path)` → nodes + citations
* **Query:** `GetContent(scope, read_context, path)` → bytes/text + citation
* **Query:** `GetRepositoryGitRemote(scope, provider)` (or equivalent)
* **Query:** `GetEvidence(run_id|scope)` returns EvidenceSpineViewModel (Domain-owned schema) from derived state
* **Implementation detail:** Projection serves these reads from **its derived state** built from Domain facts (commit actions, publish anchors, etc.). Projection does not call GitLab.
* Deterministic errors:

  * unknown mapping
  * unknown path
  * invalid read_context 

> Note: Inc 0 can keep citations at `{repo_id, sha, path[, anchor]}` exactly as Slice 0 currently specifies. Line-range anchors can arrive later (but don’t break this shape). 

### Projection individual tests

* **Unit test: apply Domain facts**

  * Given a minimal fact stream (mapping + published SHA + one file upsert), the derived read model can list the tree and return file content.
* **Unit test: list tree**

  * Root listing returns deterministic nodes and node kinds.
* **Unit test: get content**

  * Returns content + citation `{repository_id, resolved_sha, path}`.
* **Unit test: deterministic 404 semantics**

  * Missing path → explicit NotFound error outcome (not empty file content).
* **Unit test: rebuildability (Inc 0 local)**

  * Re-resolve same SHA and re-read same file → identical results.

---

## Process — durable orchestration skeleton with identity and correlation

### Increment 0 goals

1. Process exists as a durable orchestration primitive; it can start a process instance and persist minimal state. 
2. Process can coordinate via commands/events later; in Inc 0 it must at least prove:

   * identity propagation
   * correlation metadata presence
   * stable instance id return

### Why Process is first-class even in Increment 0

Increment 0 establishes the platform’s **durable run handle** (`process_instance_id`) so later increments can make governance and publish flows non-bypassable and restart-safe across UISurface + MCP clients. This is not something GitLab provides for the platform: GitLab governs merges, but the platform still needs a resumable workflow instance to coordinate multi-step actions (agent evidence → approvals → publish) and to expose progress/status as a first-class, queryable view.

### Build (Increment 0)

* **Command:** `StartProcess(scope, process_key, inputs)` → `process_instance_id`

  * Use a single key for Inc 0: `repo.explore` (or similar).
* Optional (but still Inc 0 scope): `GetProcessInstance(process_instance_id)`.

### Process individual tests

* **Unit test: start persists instance**

  * StartProcess returns id; GetProcessInstance returns status `RUNNING|STARTED`.
* **Unit test: identity propagation**

  * Instance record includes `{tenant, org, repo}`.
* **Unit test: idempotency/correlation metadata**

  * If StartProcess takes `idempotency_key`, repeated calls produce deterministic outcomes (either same instance id or explicit AlreadyStarted).

---

## Agent — cite-only proposal + Memory as workflow concern

### Increment 0 goals

1. Agent can consume citeable context bundles (from Projection) and produce a **proposal/evidence artifact** that is citation-backed. 
2. “Memory” exists inside Agent as workflow state:

   * store what was read (citations)
   * store what was produced (proposal + citations)
   * never store uncited “facts” as platform truth.

### Build (Increment 0)

* **Command-like entrypoint:** `Agent.Propose(scope, goal, context_bundle)` → `proposal {text, citations_used}` 
* **Agent memory store (internal):**

  * `Memory.append(run_id, {citations, notes})`
  * `Memory.list(run_id)` (optional, used for debugging/tests)
* A strict “cite-only” policy:

  * Proposal must include citations or be rejected (you can start with a simple rule like “if any factual claim sentence exists, it must reference at least one citation”).

### Agent individual tests

* **Unit test: cite-only enforcement**

  * Given empty context_bundle → proposal is either rejected or returns only “I don’t have sources,” with zero factual claims.
* **Unit test: proposal contains citations**

  * Given one cited file excerpt → proposal must echo citations_used.
* **Unit test: memory only stores citations**

  * Attempt to store “fact-only memory” fails schema validation (only citations + agent notes allowed).

---

## UISurface — minimal human workflow, no bypass

### Increment 0 goals

1. UI is **not optional**: it must let a human onboard + browse baseline + open file.
2. UI must call:

   * Domain for onboarding/mapping
   * Projection for reads
   * Process to start a process instance
   * Agent to request a proposal
3. UI must never call GitLab directly (GitLab adapter boundary).

### Build (Increment 0)

A minimal “Repository Explore” surface:

* **Repository Onboarding panel**

  * Inputs: provider (gitlab), remote_id or remote_path
  * Calls Domain.UpsertRepositoryGitRemote
* **Repository Browser**

  * Shows `Published @ <resolved_sha>`
  * Tree browse + file open
  * Displays citation block `{repo_id, sha, path[, anchor]}`
* **Action: Start Explore**

  * Calls Process.StartProcess(`repo.explore`)
* **Action: Ask Agent**

  * Calls Projection to assemble a context bundle for the open file (or calls Agent with citations + excerpt), then calls Agent.Propose
  * Displays proposal + citations

This is consistent with the Slice 0 “repo tree + element editor shell” concept, except now it is **mandatory**, not optional. 

### UISurface individual tests

* **UI integration test: calls correct seams**

  * Mock Domain/Projection/Process/Agent SDKs
  * Assert UI never imports GitLab SDK packages nor calls GitLab endpoints.
* **UI e2e smoke (local)**

  * Onboard → browse → open file → ask agent produces proposal

---

## Integration — external protocol adapters and imports/exports

### Increment 0 goals

1. Provide at least one external protocol adapter now: **an MCP server (Integration primitive) that exposes Projection queries to vendor coding agents**.
2. Keep protocol adapters semantics-free: forward to Domain/Projection/Process/Agent; do not re-implement business logic.
3. Enforce vendor substrate hygiene: GitLab remains a substrate; no direct GitLab access from UISurface/Agent/Process.

### Build (Increment 0)

* **MCP Adapter (Integration primitive exposing Projection queries to coding agents) (minimal)**

  * Expose read-only tools that call Projection queries:

    * `repo.list_tree(scope, read_context, path)`
    * `repo.get_content(scope, read_context, path)`
  * Pattern (apply everywhere): **MCP server = Integration primitive that exposes Projection queries to coding agents; it never calls GitLab and never invents semantics.**
  * This lets vendor agents connect immediately without you building an agent CLI.

### Integration individual tests

* **Unit test: MCP tool routing (Integration primitive exposing Projection queries to coding agents)**

  * MCP (Integration primitive) `repo.get_content` routes to Projection.GetContent and returns citations unchanged.
* **Static boundary test (Phase 0 gate)**

  * A repo-wide check that disallows importing `gitlab.com/gitlab-org/api/client-go` from UISurface/Agent/Process/Projection layers (only Domain may import it), and disallows importing MCP protocol libs (Integration primitive) outside Integration.
  * This is one of the highest leverage “no drift” protections you can add in Inc 0.

---

# Increment 0 contract-pass scenario definition

This is the single “vertical slice” scenario executed by `ameide test` (Phase 0/1/2), aligned to your existing Slice 0 runner steps but updated to “no optionals” and “Memory=Agent workflow.” 

### Scenario steps

1. Create identity `{tenant_id, organization_id, repository_id}`.
2. Domain: `UpsertRepositoryGitRemote(identity, provider=gitlab, remote_id=<mockProjectId|path>)`.
3. Projection: `ListTree(identity, published, "/")` → response includes `read_context.resolved.commit_sha = resolved_sha`.
4. Projection: `GetContent(identity, published, "README.md")` (or a known file) → citation includes `{repo_id, resolved_sha, path}`.
5. Process: `StartProcess(identity, "repo.explore", inputs={resolved_sha, path})` → `process_instance_id`.
6. Agent:

   * Build `context_bundle` from Projection response (citation + excerpt).
   * `Agent.Propose(identity, goal="Summarize README", context_bundle)` → proposal + citations.
7. Projection: `GetEvidence(identity, run_id|scope)` returns the run report using the **Domain-owned EvidenceSpineViewModel schema**:

   * includes identity
   * mapping
   * resolved_sha
   * citations
   * process_instance_id
   * proposal summary (and citations used)
   * publish fields empty (Inc 0)

### Assertions

* Every call carries identity (directly or via request context).
* Every content-bearing output is citeable to `{repo_id, resolved_sha, path[, anchor]}`. 
* UI smoke test passes for the same scenario (can run separately but is mandatory for Inc 0 deliverable).
* MCP call test (Integration primitive exposing Projection queries to coding agents) passes for `repo.get_content` and returns the same citation.
  * MCP is an Integration primitive that exposes Projection queries to coding agents.

---

# Increment 0 “Done means done” checklist

**For Inc 0 to be complete:**

* ✅ 1 capability-level contract-pass test proving the end-to-end slice (all primitives touched)
* ✅ Each primitive has:

  * at least one new API/behavior,
  * at least one primitive-owned test that proves it
* ✅ UI can onboard + browse + open + ask agent (even if minimal)
* ✅ GitLab adapter boundary enforced by at least one static check
* ✅ EvidenceSpineViewModel schema exists as Domain-owned and is used in the run report (no competing schemas) 

---
