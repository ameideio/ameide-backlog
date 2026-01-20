---
title: "694 – Elements / Enterprise Repository in GitLab (v6: Git-backed canonical content)"
status: draft
owners:
  - platform
  - transformation
created: 2026-01-18
supersedes:
  - 694-elements-gitlab.md
related:
  - 496-eda-principles-v6.md
  - 520-primitives-stack-v6.md
  - 527-transformation-v6-index.md
  - 695-gitlab-configuration-gitops.md
  - 701-repository-ui-enterprise-repository-v6.md
---

# 694 – Elements / Enterprise Repository in GitLab (v6: Git-backed canonical content)

This v6 file exists to keep the v6 “Git-backed Enterprise Repository” posture stable and easy to reference from other backlogs.

## Normative rules (v6)

- **Owner-only writes:** only the owning Domain performs canonical Git operations; other primitives route through Domain commands (`backlog/496-eda-principles-v6.md`).
- **Contract-first inter-primitive seams:** protos govern primitive↔primitive interactions; GitLab REST is an internal storage adapter used by owner/projection primitives (`backlog/496-eda-principles-v6.md`).
- **Relationships are inline-only:** no `relationships/**` folder and no relationship sidecar artifacts.
- **GitLab CE only:** no Premium/Ultimate dependencies; enforcement uses Community primitives (protected `main`, MR-only, pipeline gating) with governance truth in the platform.

## Truth model (v6: three kinds of truth)

Use this triad consistently to avoid accidentally making projections canonical:

- **Content truth** = Git files + commit graph (immutable anchors are commit SHAs; tags are optional).
- **Governance truth** = platform-owned state (approvals, attestations, policy outcomes, workflow state, audit pointers).
- **View truth** = derived projections (indexes/graphs/search/timelines), rebuildable from Git + owner audit pointers, citation-addressable.

This is consistent with the “projection-owned memory” posture in `backlog/656-agentic-memory-v6.md`.

## Inline-only relationships (operational v0)

“Inline-only relationships” is a normative posture, but it must be implementable:

- **What counts as “inline”**: a relationship is authored inside canonical files (Markdown, BPMN, model formats, and optionally structured metadata blocks within those files). There is no separate relationship artifact and no relationship CRUD API.
- **Allowed encodings (v0)**: the platform may standardize conventions per file type, but projections must at minimum support extracting references from:
  - Markdown links to repo-relative paths (optionally with `#anchor` fragments),
  - optional frontmatter/metadata blocks (e.g., YAML) containing stable IDs and reference lists,
  - model-native identifiers/links where the format already supports them (e.g., BPMN ids/extension properties).
- **Directionality**: relationships are declared **single-sided**; backlinks are always derived by projections.
- **Identity and citations**: a relationship must be resolvable (at read time) to audit-grade citations `{repository_id, commit_sha, path[, anchor]}`. Stable IDs are allowed as a file convention, but projections must always be able to cite concrete file locations at a specific commit SHA.
- **Rename/move semantics**: rename/move must not silently create false edges. If a reference cannot be resolved at the selected commit SHA, it is surfaced as an explicit “broken/unresolved reference” (and governance/validation decides whether that blocks publication).
- **Referential integrity**: dangling references are allowed as drafts, but they must be machine-detectable and citation-addressable (so validators and UI can explain what is missing).

## Allowed GitLab CE primitives (in-scope vs out-of-scope)

**In-scope (we do rely on these):**

- Git repositories (projects/groups), branches and tags.
- Merge Requests as the collaboration substrate (discussion + proposed diffs).
- GitLab CI pipelines as validation/publishing automation.
- Protected default branch (`main`), “merge via MR” enforcement, and pipeline gating as configured.
- GitLab REST APIs as **internal adapters** used by owner/projection primitives (not by UI/agents/processes).

**Out-of-scope (we must not design a dependency on these):**

- GitLab Premium/Ultimate planning surfaces (epics/roadmaps/iterations) as platform UX primitives.
- Any “GitLab is the governance UI” assumption (governance truth is platform-owned).
- Paid-tier approval/status-check frameworks as required correctness dependencies (platform policy decides; GitLab enforcement stays CE-grade).

## GitLab product analogies (keep the mental model aligned)

This repo treats GitLab CE as a subsystem, but using GitLab’s own product concepts as analogies helps keep intent unambiguous:

- **Enterprise Repository (Ameide)** ≈ **GitLab Project repository** (files + commit graph; served by Gitaly).
- **Published baseline (Ameide)** ≈ **`main` HEAD commit SHA** (optionally a tag/release marker).
- **Citation `{repo_id, commit_sha, path[, anchor]}` (Ameide)** ≈ **GitLab permalink to a file at a specific commit** (plus an optional in-file anchor like a heading or line range).
- **Change / proposal (Ameide)** ≈ **Merge Request** (branch + diff + discussion), but **governance truth** lives in our platform DB.
- **Governed write (Ameide Domain)** ≈ **GitLab Commit API + MR merge** performed by a bot identity under policy.
- **Projection graph/search (Ameide)** ≈ **GitLab-derived views** (search/code intelligence/Knowledge Graph): useful for UX and AI, but still derived from repository content.

## Git provider posture (v6)

In the current platform posture, “Git-backed” concretely means:

- **GitLab is deployed in-cluster** as a platform component and managed by GitOps.
- GitLab is **not optional**: it is part of the default platform deployment.
- GitLab is **internal/private** to the platform: tenants do not log into GitLab; platform services use a bot identity to operate Git.

To avoid losing detail, the full decision text (including the “How this evolves prior work” paragraphs and external links) is preserved in:

- `backlog/694-elements-gitlab.md` (historical record; superseded by this v6 pointer)

Normative cross-references for the v6 story:

- Integration/EDA posture: `backlog/496-eda-principles-v6.md`
- Runtime posture: `backlog/520-primitives-stack-v6.md`
- Transformation v6 spine: `backlog/527-transformation-v6-index.md`

## Default tenant Enterprise Repository layout (v6)

This is a recommended, minimal repository layout that keeps “canonical truth is files” concrete without forcing a single file format:

```text
<tenant-enterprise-repo>/
  elements/                         # canonical authored artifacts (docs/diagrams/config)
  processes/                        # design-time governed ProcessDefinitions (BPMN as files)
    <module>/<process_key>/v<major>/
      process.bpmn
      bindings.yaml                 # optional
      README.md                     # optional
```

Vendor-correct naming note: `process_key` SHOULD match the BPMN `<process id="...">` so runtime deployment/keying semantics align across Zeebe/Flowable.

Normative rule: **Relationships are expressed only as inline references within element content** (links/identifiers embedded in Markdown, model files, BPMN, etc.). There is no `relationships/**` folder and no relationship sidecar artifacts.

Normative rule: **`main` is the published baseline**, and “published” is a commit SHA on `main` (optionally tagged).

- GitLab deployment (GitOps) tracking: `backlog/695-gitlab-configuration-gitops.md`

## Alignment note (694 ↔ 695)

694 defines the **governance/tenancy posture**: GitLab is a platform-owned subsystem and the platform remains the enforcement surface (tenants do not get arbitrary GitLab accounts by default).

695 defines the **workload configuration and operations posture** (GitOps wiring, endpoints, secrets, smokes). If external hostnames exist (e.g., `gitlab.<env>.ameide.io`), they are for **operators/platform services** and should be network-restricted (private DNS/internal Gateway and/or allowlists + SSO), consistent with the “private subsystem” stance.

Progress note: GitLab is deployed as a standard GitOps component (`platform-gitlab`) and verified via PostSync smoke jobs (`platform-gitlab-smoke`), per `backlog/695-gitlab-configuration-gitops.md`.

Operational note: GitLab object storage currently uses the shared in-cluster MinIO (`data-minio`) as a temporary posture; the long-term target is external object storage / scoped credentials (tracked in `backlog/695-gitlab-configuration-gitops.md`).

## Governance enforcement surface (GitLab controls)

694’s governance posture assumes GitLab is configured so that “raw Git pushes” cannot bypass platform policy. Record (and keep consistent) the concrete GitLab controls relied upon, such as:

- Protected default branch (`main`): restrict who can push/merge.
- Require merge requests (no direct pushes) for protected branches.
- Require pipeline success (status checks) prior to merge.
- Limit who can merge (bot/governance role vs operators), as applicable.

## Audit linkage contract (platform ↔ Git evidence)

Record the minimal immutable evidence pointers the platform stores for each accepted change, for example:

- Repository/project id + MR IID/URL (human traceability)
- Pipeline id + status (CI evidence)
- Merge commit SHA (immutable content anchor)
- Optional tag/release marker (published/baseline anchor)
