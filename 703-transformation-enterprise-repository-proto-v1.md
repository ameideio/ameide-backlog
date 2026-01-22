---
title: "703 — Transformation Enterprise Repository Proto (v1 proposal: Git tree browsing + governed Git writes)"
status: implemented
owners:
  - transformation
  - platform
created: 2026-01-19
updated: 2026-01-22
related:
  - 496-eda-principles-v6.md
  - 694-elements-gitlab-v6.md
  - 701-repository-ui-enterprise-repository-v6.md
  - 527-transformation-proto-v6.md
---

# 703 — Transformation Enterprise Repository Proto (v1 proposal: Git tree browsing + governed Git writes)

This proposes a clean v1 proto seam for “Enterprise Repository = Git tree” browsing and governed edits.

## Status (2026-01-20)

This v1 seam is now implemented in the platform codebase and should be treated as the canonical contract for:

- Git tree reads via Projection (path + read_context + SHA citations)
- Governed writes via Domain (MR-backed `EnsureChange`/`CreateCommit`/`PublishChange`)

## GitLab product analogies (how to interpret the contract)

- Projection `ListTree/GetContent` ≈ GitLab “browse repository” at a ref, but always citation-grade (commit SHA + path + optional anchor).
- Domain `EnsureChange` ≈ create/reuse a branch + Merge Request (proposal container).
- Domain `CreateCommit` ≈ GitLab Commit API on the MR branch (batched file actions, guarded by `last_commit_id`).
- Domain `PublishChange` ≈ merge the MR into `main` and record immutable audit pointers (MR IID/URL, pipeline id/status if applicable, merge/squash SHA).

## Normative constraints (must hold)

From `backlog/496-eda-principles-v6.md`:

- Inter-primitive communication is contract-first (public APIs/event payloads defined in Protobuf; transport may be RPC and/or events).
- Owner-only writes: only the owning Domain performs canonical Git writes.
- Facts are emitted only after commit (Git-backed: commit/merge/tag + durable audit pointers recorded).
- Idempotency and traceability are mandatory.

From `backlog/694-elements-gitlab-v6.md` and `backlog/701-repository-ui-enterprise-repository-v6.md`:

- GitLab is an internal storage subsystem; tenants do not use GitLab directly.
- Repository hierarchy in the UI is the Git file tree (folders + files), served via Projection (read) and Domain (write).

## What this seam is (and is not)

- **Is:** UISurface/Agents/Process call Domain/Projection via this proto seam for repository browsing and governed edits.
- **Is not:** a “GitLab proxy API contract” for inter-primitive usage. Domain/Projection call GitLab REST directly as an internal storage adapter.

## Package and scope

Proposed package: `io.ameide.transformation.enterprise_repository.v1`.

All requests and responses must carry explicit scope:

- `tenant_id`
- `organization_id`
- `repository_id`

There is no separate “workspace id” for repository navigation.

## Read model (Projection): Git tree hierarchy

### Core query: list directory children

`ListTree(ref, path)` returns the repository’s Git tree entries at a specific ref:

- `ref` is resolved to a commit SHA for audit-grade reads (`read_context.resolved.commit_sha`).
- `path` is repo-relative (empty/`/` means root).

Each returned node should minimally include:

- `path`
- `name`
- `kind`: `FILE | DIRECTORY | GITLINK`
- `size_bytes` (optional, for files where cheaply available)
- `content_type` / `extension` hints (optional)
- `citations[]` anchored to `{repository_id, commit_sha, path[, anchor]}` where applicable

**Submodules:** a Git submodule is represented as `kind=GITLINK` with the pinned object id from the tree. Do not attempt to resolve it to a second `repository_id` as part of browsing.

### Read file content

`GetContent(ref, path)` returns file bytes/text plus type hints and citations anchored to the resolved commit SHA.

### Search (optional v1)

If search is included, it must be a Projection concern and must return `read_context` + citations anchored to the resolved commit SHA.

## Write model (Domain): governed Git operations

All writes occur against a change (branch + MR) owned by the Domain.

### Ensure a change (branch + MR)

`EnsureChange(...)` creates or reuses a Domain-tracked change and returns:

- change id (platform identity)
- GitLab project id
- branch name/ref
- merge request IID
- head commit SHA (for concurrency control)

### Commit actions (batching; idempotent)

`CreateCommit(change_id, actions[], expected_last_commit_id, idempotency_key)`:

- Uses GitLab’s Commit API semantics (multiple file actions in one commit).
- Must be concurrency-safe: reject if `expected_last_commit_id` does not match the current branch head.
- Must be idempotent with the provided idempotency key.

Minimum action kinds:

- `UPSERT_FILE {path, content_bytes, content_encoding}`
- `DELETE_PATH {path}` (file delete)
- `MOVE_PATH {from_path, to_path}` (file move/rename)

**Folder operations:** Git has no first-class empty directories. “Move folder” is a prefix rename implemented as a set of `MOVE_PATH` operations computed by the Domain (or by the client if it has enumerated the tree). “Create folder” is implicit (created when the first file is written under the path).

### Publish (merge)

`PublishChange(change_id, expected_mr_head_sha)` merges the MR into `main` only if:

- platform governance is satisfied, and
- the MR head SHA matches `expected_mr_head_sha` (SHA-safe publish).

The Domain records durable audit pointers, minimally:

- GitLab project/repository id
- MR IID
- pipeline id + status (if applicable)
- resulting commit SHA on `main` (merge or squash strategy)
- optional tag

Only after durable recording does the Domain emit facts.

## Facts (CloudEvents `type`) emitted by the Enterprise Repository owner (Domain)

Per `backlog/496-eda-principles-v6.md`, fact semantic identities are normative (`io.ameide.<context>.fact.<name>.v<major>`). For this seam, lock the initial fact types (names only; payloads remain protobuf-governed):

**Owner facts (Domain):**

* `io.ameide.transformation.fact.enterprise_repository.repo_mapping_upserted.v1`
* `io.ameide.transformation.fact.enterprise_repository.change_committed.v1`
* `io.ameide.transformation.fact.enterprise_repository.change_published.v1`
* `io.ameide.transformation.fact.enterprise_repository.validation_failed.v1` (only if modeled as a durable audit outcome recorded by Domain)

**Derived facts (Projection; optional):**

* `io.ameide.transformation.fact.enterprise_repository.typed_item_indexed.v1`

Emission rules:

* Emit facts only after the corresponding Git outcome is durable and Domain has recorded the audit pointers it will rely on (MR IID, commit SHA(s), pipeline id/status if applicable).
* Emit `change_published` only after the publish anchor (`target_head_sha`) is recorded (Git-backed outbox-equivalent).

## Migration posture

This seam intentionally does not reuse “workspace tree/node” concepts (nodes, node ids, node move facts). Those older APIs may remain for historical initiatives/workspace UI, but **Enterprise Repository browsing is path + ref** only.
