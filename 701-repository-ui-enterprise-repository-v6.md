---
title: "701 — Repository UI (v6: GitLab-backed Enterprise Repository view)"
status: draft
owners:
  - platform
  - ui
  - transformation
created: 2026-01-19
related:
  - 694-elements-gitlab-v6.md
  - 695-gitlab-configuration-gitops.md
  - 520-primitives-stack-v6.md
  - 496-eda-principles-v6.md
  - 527-transformation-v6-index.md
  - 706-transformation-v6-coordinated-implementation-plan.md
---

# 701 — Repository UI (v6: GitLab-backed Enterprise Repository view)

## Functional story (what users do)

This backlog aligns the Ameide Platform UI “Repository” experience with the v6 posture:

- The Enterprise Repository is **canonical Git content**, backed by **in-cluster GitLab** (`backlog/694-elements-gitlab-v6.md`).
- GitLab is a **private platform subsystem**; users do not log into GitLab directly.
- The UI is the governance/product surface: users browse, edit, propose, approve, and publish changes using the platform.
- The UI presents repository content as an **Enterprise Repository hierarchy** (folders + files), derived from the Git file tree (no separate canonical workspace tree).

### GitLab product analogies (keep UX intent grounded)

- The repository browser is analogous to GitLab’s repository file browser, except reads/writes are mediated by Projection/Domain (users do not use GitLab directly).
- “Start change / propose edits / publish” is analogous to GitLab branch + MR + merge, except approvals/policy truth lives in the platform UI/DB.
- Citations in the UI are analogous to GitLab permalinks (file-at-commit anchors), so reads are reproducible.

## Normative rules (v6)

These are the non-negotiables this backlog is written to:

- **Contract-first boundaries:** inter-primitive interfaces are contract-first per `backlog/496-eda-principles-v6.md` (public APIs/event payloads defined in Protobuf; transport may be RPC and/or events). GitLab REST is an internal storage adapter used by owner/projection primitives.
- **Owner-only writes:** the browser/UI does not mutate Git; it calls Domain commands.
- **Relationships are inline-only:** no relationship CRUD, no relationship sidecar artifacts.
- **GitLab CE only:** no Premium/Ultimate dependencies; governance truth is platform-owned; GitLab enforces only via CE primitives (protected branches, MR-only, pipeline gating as configured).
- **Hierarchy equals Git tree:** repository navigation is the Git file tree at a selected `read_context`.
- **Submodules are Git-level:** treat as Git tree `gitlink` entries; do not resolve or flatten into other platform repositories during browsing.

Operational clarifications (v6):

- Truth model (content vs governance vs view truth) and allowed GitLab CE primitives: `backlog/694-elements-gitlab-v6.md`.
- Projection contract (rebuildable, incremental, citation-grade, eventually consistent): `backlog/656-agentic-memory-v6.md`.
- “Inline-only relationships” must be implemented so every derived relationship view (backlinks/impact) can show citations back to `{repository_id, commit_sha, path[, anchor]}` and can surface unresolved/broken references deterministically.
- Metadata blocks are mandatory: canonical files must embed frontmatter/metadata (in-file) carrying stable `id` and `refs`. UI editors must preserve it (and scaffold it when creating new artifacts) so references can be authored consistently.

### Primary user journey (happy path)

1) User opens an organization and selects an Enterprise Repository (a single underlying Git repo).
2) UI shows the repository as a Git-backed folder/file hierarchy at a selected ref (`published` baseline by default).
3) User browses and opens an **element** (document/model/process) from the hierarchy.
4) UI opens the correct editor based on the element’s file type (e.g., Markdown, ArchiMate, BPMN).
5) User edits and proposes changes; the platform creates/updates a change branch and MR; checks run.
6) Approvals happen in the UI (governance truth in platform DB).
7) Platform merges the MR into `main` and records immutable audit pointers (project/repository id, MR IID, pipeline id + status (if applicable), and the resulting commit SHA on `main`; optionally tag as a baseline).
8) Projections refresh (index/search/graph/timeline) from owner facts and/or baseline change signals.

Vendor-correct refinements (GitLab):

- The Domain should prefer GitLab’s **Commit API** for “editor save” operations (single commit with multiple file actions on a branch).
- Idempotent updates must be concurrency-safe: use GitLab commit actions guarded by `last_commit_id` (avoid lost updates when two writers race).
- MR state is eventually consistent: the Domain must tolerate asynchronous population of mergeability/diff/pipeline fields.
- Merge should be performed using GitLab’s merge endpoint keyed by merge request **IID** (not an internal DB id), and record merge/squash commit SHAs according to strategy.
- “Checks run” means GitLab MR pipelines (GitLab CI).

### ProcessDefinitions in the Enterprise Repository

Under v6, ProcessDefinitions (aka “workflow definitions”) are **design-time governed artifacts stored as Git-backed elements** in the Enterprise Repository (not in a Definition Registry, and not in a separate custom workflow-definition subsystem).

They are authored as BPMN and later deployed to the selected BPMN runtime(s) (Zeebe/Camunda 8 and Flowable).

Vendor-correct definitions:

- `process_key` should equal the BPMN `<process id="...">` (runtime “key” semantics) and is the canonical identifier used for validation, deployment, and worker binding.
- “Major version” (`v<major>/`) is a platform semantic layered over vendor deployment versions. Recommended posture:
  - a **major bump requires a new BPMN process id** (new `process_key`) to avoid mixing breaking changes into one runtime version chain.
  - instance migration policy (moving running instances to a new version) is explicitly **TBD**.

```text
processes/<module>/<process_key>/v<major>/process.bpmn
```

Promoting a ProcessDefinition (at a high level):

- change the ProcessDefinition files in Git,
- validate against the target runtime (Zeebe/Camunda 8 or Flowable),
- deploy the Process primitive worker implementation that provides the BPMN task side-effects,
- roll out safely and observe (progress/timeline).

Vendor-correct deployment notes:

- Camunda 8 deploys BPMN resources via a deployment endpoint (atomic resource deployment; size limits apply).
- Flowable typically deploys via BAR/ZIP (multipart), scanning BPMN resources within the archive.
- Cross-runtime portability is a goal but not guaranteed; runtime-specific extensions may require constraints or variants.

Placement in the repository hierarchy: ProcessDefinitions are normal files in the repo (typically under `processes/**` by convention) and appear where they live in the folder tree.

Multi-tenancy details for process catalogs are **explicitly out of scope here**; capture as a separate backlog.

## Non-goals (v6)

- Direct GitLab UI usage as the product surface.
- UI access to GitLab credentials or Git operations.
- Treating GitLab as optional: v6 assumes GitLab is part of the default platform deployment.
- Finalizing the “agent memory model” (IDs/citations/semantic overlays); keep as TBD (projection concern).

## Current implementation reality (what exists today)

The current “Repository” UI is largely **element/graph-shaped**, not Git-backed.

### “Transformation” in the current UI (as-is)

Today, the platform already has a “change-as-a-process” experience, but it is not yet GitLab/MR-native:

- **R2R UI is element-based**:
  - Repository R2R list: `services/www_ameide_platform/app/(app)/org/[orgId]/repo/[repositoryId]/r2r/page.tsx`
  - Change workspace: `services/www_ameide_platform/app/(app)/org/[orgId]/repo/[repositoryId]/r2r/change/[changeId]/page.tsx`
  - A “change” is an Element with `typeKey = "ameide:change"`; the UI lists and creates these elements.
- **Governance anchors are versioned element references**:
  - Required anchors: `ref:requirement` and `ref:deliverables_root` relationships with `metadata.target_version_id`.
  - Helper: `services/www_ameide_platform/features/r2r/lib/anchors.ts` (`baselineIdForChange`, `getTargetVersionId`).
- **Runs are started via a platform Workflow runtime (not BPMN ProcessDefinitions)**:
  - `services/www_ameide_platform/features/r2r/lib/runs.ts` shows:
    - `R2R_WORKFLOW_DEFINITION_ID = "transformation-process"`
    - `R2R_WORKFLOW_TASK_QUEUE = "transformation-tq"`
    - `workflowsType` is chosen by methodology (e.g., `transformation.r2r.governance.scrum.v1`).
  - The UI starts runs via `workflowsClient.startRun(...)` and lists them via `workflowsClient.listRuns(...)`:
    - `services/www_ameide_platform/app/(app)/org/[orgId]/repo/[repositoryId]/r2r/change/[changeId]/page.tsx`
  - The current run-start dialog requires **manual** `RepoURL` + `CommitSHA` + `Workdir`, which indicates the current workflow execution is not yet “GitLab MR as the governance surface”.
- **Timeline is already wired to process facts**:
  - Repo timeline: `services/www_ameide_platform/app/(app)/org/[orgId]/repo/[repositoryId]/timeline/page.tsx`
  - Reads `transformationProcessQuery.listProcessFacts(...)` (projection-backed).

### Process facts ingestion (as-is)

The projection already supports a “process evidence timeline” ingestion posture, but it currently includes a Temporal-based process primitive:

- Process facts are read from the Transformation Projection:
  - gRPC server: `primitives/projection/transformation/cmd/main.go`
  - handler: `primitives/projection/transformation/internal/handlers/handlers.go` (`ListProcessFacts`)
- MVP ingestion exists as a **DB relay**, tailing:
  - Domain outbox (`transformation.domain_outbox`) and
  - Process outbox (`transformation_process.process_outbox`)
  - Relay: `primitives/projection/transformation/cmd/relay/main.go`
- The “current process primitive” that emits `process_outbox` is Temporal-based:
  - worker: `primitives/process/transformation/cmd/worker/main.go`
  - ingress router: `primitives/process/transformation/internal/ingress/router.go`

v6 target posture (this backlog) is to pivot workflow definitions to **BPMN ProcessDefinitions stored in the Enterprise Repository** (`processes/**`) and run them on Zeebe/Flowable; the above is the as-is evidence wiring that must be migrated.

### GitLab integration in the UI service (as-is; prototype)

The web app contains server-side GitLab adapter endpoints implemented via `@gitbeaker/rest`:

- Client: `services/www_ameide_platform/lib/gitlab/client.ts` (`GITLAB_URL`, `GITLAB_TOKEN`)
- Routes: `services/www_ameide_platform/app/api/gitlab/repositories/**` (tree/files/commits; includes “create commit”)

These endpoints are not currently the canonical write path (they bypass Domain governance) and are not wired into the repository UX as the primary surface. Under v6, GitLab operations should be mediated by the owning Domain primitive and audited, not performed ad-hoc by the UI layer.

### UI routes and what they do today

- Repository list: `services/www_ameide_platform/app/(app)/org/[orgId]/repo/page.tsx`
  - language is “graphs/elements”, not “repo files”.
  - uses `useRepositories(orgId)` which calls `GET /api/repositories` (see `services/www_ameide_platform/lib/api/hooks.ts` and `services/www_ameide_platform/app/api/repositories/route.ts`).
- Repository root: `services/www_ameide_platform/app/(app)/org/[orgId]/repo/[repositoryId]/page.tsx`
  - lists “ArchiMate views” by calling `transformationKnowledgeQuery.listElements` with `ElementKind.ARCHIMATE_VIEW`.
  - creates a “New view” via `transformationKnowledgeCommand.createElement` (DB-canonical element posture).
- ArchiMate view editor: `services/www_ameide_platform/app/(app)/org/[orgId]/repo/[repositoryId]/archimate/views/[viewId]/page.tsx`
  - reads and writes ArchiMate content via element/version/relationship APIs.
- Definition Registry: `services/www_ameide_platform/app/(app)/org/[orgId]/repo/[repositoryId]/registry/page.tsx`
  - reads `transformationRegistryQuery.listDefinitions` and renders Definition Registry entries.
  - conflicts with v6: ProcessDefinitions should be files under `processes/**`.
- “Open Element” browser dialog: `services/www_ameide_platform/features/elements/graph/ElementRepositoryBrowser.tsx`
  - browses repository “contents” via `useRepositoryData(...)`, which ultimately reads `GET /api/repositories/:id`.
- “Workflow definitions” settings UI: `services/www_ameide_platform/app/(app)/org/[orgId]/settings/workflows/[workflowId]/page.tsx`
  - authors JSON workflow definitions via a separate `workflows` runtime API (not BPMN process definitions stored in Git).

### ConnectRPC proxy posture (already aligned)

The UI calls services via a Connect proxy that keeps the internal gateway private:

- Connect proxy: `services/www_ameide_platform/app/api/proto/[...path]/route.ts`

This is consistent with the GitLab posture: the browser should not talk to GitLab directly either; it should call owner/projection APIs.

## Target-state UI decomposition (v6)

The repository UI becomes “Enterprise Repository hierarchy (file tree) + Git-backed elements + projections + governance”.

### Repository navigation model (Git tree; projection-derived)

The UI should present the repository’s folder/file hierarchy as the primary navigation model.

Key rule: the UI shows and edits **elements** (files), but the hierarchy itself is the Git file tree (directories + files) at the selected `read_context`.

Classification (e.g., “this is a ProcessDefinition”) is derived from file content/type (extensions/MIME) and repository conventions, but hierarchy itself is not remapped into a separate taxonomy.

Submodules: a directory entry may be a Git submodule (a `gitlink` entry in the Git tree). Submodules are handled at the Git level; the Projection/UI should treat them as Git tree entries (e.g., `GITLINK`) and must not attempt to “resolve” them into other platform repositories as part of browsing.

Implementation note (UISurface): stay aligned with the existing Radix-based component posture; prefer reusing internal primitives/patterns over introducing a new third-party tree library unless scale proves it necessary.

### Editors (extension-driven)

Editors are selected by the element’s underlying file extension / MIME:

- Markdown/text editors for `*.md`, `*.txt`, etc.
- ArchiMate editor for ArchiMate element files (format TBD by convention).
- BPMN editor for `processes/**/process.bpmn`.

These editors are not different persistence models: they are different frontends for modifying Git-backed elements.

### Relationships (authored as references; reconstructed by projections)

Relationships are **only inline references inside element content** (links/identifiers embedded in Markdown, model files, BPMN, etc.).

- There is no `relationships/**` folder and no relationship sidecar artifacts.
- The platform does not support canonical “relationship CRUD” as a write surface.
- Projections derive backlinks/impact/graphs only from inline references (and optionally code indexing, if present), never from separate relationship artifacts.

### Owner vs projection read/write rules (UI contract)

- UI writes always go to the owning Domain primitive (commands); Domain is the only canonical writer (including Git operations).
- UI reads should prefer Projection query APIs for:
  - repository hierarchy browsing (folders + files), search, graph traversal, impact analysis,
  - timelines and derived views.
- UI should not require GitLab APIs; GitLab remains a private storage adapter behind Domain/Projection.

### GitLab capability constraint (v6)

- We rely only on GitLab Community Edition capabilities (no Premium/Ultimate dependencies).
- The platform implements governance features (approvals/policy/publish eligibility) as platform truth and uses GitLab for storage + CI plus basic enforcement primitives (protected `main`, merge via MR, pipeline gating, restricted push/merge roles as configured).
- CI gating depends on the chosen runner posture; track and keep aligned with `backlog/695-gitlab-configuration-gitops.md`.

### Contract-first boundaries (v6; per 496)

- Inter-primitive communication (UISurface ↔ Projection/Domain, Process ↔ Domain, Agent ↔ Domain) remains contract-first per `backlog/496-eda-principles-v6.md` (public APIs/event payloads defined in Protobuf; transport may be RPC and/or events).
- GitLab REST APIs are used directly only inside the owner/projection primitives as the Git storage adapter; the browser does not talk to GitLab directly.

## Required backend seams to support the UI (high-level)

This backlog does not define the full proto set, but it defines the UI’s minimal needs.

### Projection query surface (read path)

Minimum queries needed to build an Enterprise Repository browser:

- browse repository hierarchy (list directory children; return file/directory/gitlink nodes)
- fetch element content (path + ref → bytes/text + type hints)
- baseline selection support (published baseline commit SHA; optional tags)
- search (full-text + structured as available)
- backlinks/relationships view (derived graph from references; code graphs optional)

### Domain command surface (write/govern path)

Minimum commands needed for “edit elements with governance”:

- begin/ensure a change (branch + MR) scoped to a repository
- commit/update element content on the change branch (idempotent; use Commit API + `last_commit_id` guards)
- move/rename paths (files and folders) on the change branch (idempotent; guard with `last_commit_id`)
- request/record approvals (governance truth)
- merge/publish (merge MR → `main`, record audit pointers, emit facts)
- merge/publish is allowed only if platform governance is satisfied; GitLab is configured so protected `main` cannot be pushed directly, and merges occur via MR with CI gating as configured.

Vendor-aligned notes:

- Treat MR readiness as eventually consistent; automation must sometimes wait for MR processing to stabilize before acting.
- Merge/publish should be SHA-safe: merge only the MR head commit SHA that was evaluated; stale head → refuse and re-evaluate.

## Deprecations / replacements implied by v6

This backlog makes the following direction explicit:

- **Definition Registry is deprecated** for ProcessDefinitions:
  - replace `repo/<id>/registry` with ProcessDefinitions stored as files in the Enterprise Repository (typically under `processes/**` by convention) and browsed via the repository hierarchy.
- The custom “workflow definitions” subsystem is superseded for BPMN-governed processes:
  - pivot to BPMN authoring and runtime validation for Zeebe/Flowable.
- “Enterprise Repository UI == element graph browser (DB substrate)” is superseded:
  - the repository root view should be hierarchy-first (Git file tree), with graph/search as derived projections.
- ArchiMate and other editors must move from “DB element CRUD” to “Git-backed element editing” (with projections to index/relate).

## Incremental delivery plan (minimize disruption)

1) Add an Enterprise Repository hierarchy view to the repository page that can:
   - browse folders and open elements read-only from the published baseline.
2) Add “Edit → propose” flow:
   - edit element in UI → Domain creates/updates MR → UI shows checks/governance state.
3) Replace Definition Registry view:
   - stop treating process definitions as registry entries; author/browse BPMN ProcessDefinitions in-repo.
4) Migrate existing editors:
   - ArchiMate view editor persists to Git-backed elements (with projection indexing for search/graph).
5) Switch the repository landing page:
   - default to the repository hierarchy view (not “ArchiMate views”), with derived navigation layered on top.

## Open questions / explicit TBDs

- Multi-tenancy boundaries for process catalogs and any shared template repos (default is tenant-owned repos; details TBD).
- Agent memory model and citations under Git-first (“memory is projection” is accepted; specifics TBD).
- Level of code graph indexing (reuse external graph indexers vs build in-house); treated as projection implementation detail.
- GitLab identity strategy:
  - service-account commits + platform DB as authoritative audit trail (default), vs
  - per-user authorship/approvals in GitLab (requires delegated identity/tokens and must reconcile with commit signing/approval eligibility).
- Enforcement posture:
  - GitLab as “storage + CI substrate” only, vs
  - GitLab as “community-grade enforcement substrate” using protected branches + merge requests + required pipelines, with publish eligibility enforced by platform governance.
