---
title: "655 – Agentic coding — external executor profile: Dynamics 365 Finance & Operations (D365FO)"
status: draft
owners:
  - platform-devx
  - erp
created: 2026-01-12
suite: "650-agentic-coding"
source_of_truth: false
---

# 655 – Agentic coding — external executor profile: Dynamics 365 Finance & Operations (D365FO)

> This backlog is an **external executor profile** within the 650 suite.
>
> It intentionally deviates from the “execution is in-cluster” *default* because D365FO development is vendor-locked to a Windows dev VM + Visual Studio toolchain.
>
> Normative references:
> - `backlog/650-agentic-coding-overview-v6.md` (model + exception rule)
> - `backlog/654-agentic-coding-cli-surface-coder.md` (CLI front doors + profiles)

Below is a **vendor‑doc–aligned spec** for the architecture you described:

* **Orchestration only in-cluster** (Coder workspace / Coder Task runs Codex)
* **Execution only in the D365 F&O Windows dev VM** (VS + your custom add‑in)
* **All repo writes + git ops happen in the VM** via tool calls (including “commit on successful sync”)
* **Coder/Codex still has outside visibility** of the branch for reasoning (by consuming the pushed branch state)

I’m going to be explicit about decisions so we don’t leave “open questions”.

---

# Ameide D365 F&O Agentic Development Spec

**Codex (in-cluster) orchestrates → VM executes via VS Add-in → Git commits/pushes → Coder mirrors branch for reasoning**

## 0) How this fits the 650 suite (alignment)

This profile keeps the 650 invariants:

- Orchestration runs in-cluster (Coder workspace/task).
- No Telepresence.
- The CLI remains the cognitive-load absorber and the agent runs `ameide` front doors.
- Templates are agent profiles with scoped `AGENTS.md` guardrails.

This profile adds one explicit exception:

- Execution runs on an external executor substrate (Windows dev VM), accessed only through an allowlisted tool contract.

## 1. Vendor constraints we must respect

### 1.1 D365 F&O development is a VM + Visual Studio workflow

Microsoft defines a “Developer” as someone who develops code through Visual Studio and explicitly states that a developer requires **Remote Desktop access to the development VM**.
This anchors the design: the “execution environment” is a Windows dev VM with the D365 VS tooling installed.

### 1.2 We can build custom developer tooling using the D365 Add-ins infrastructure

Microsoft provides an **Add-ins infrastructure** in Visual Studio for Dynamics 365 development tools and explicitly notes you can create your own add-ins by selecting the **Dynamics Developer Tools Add-in** project type.
So “create table via VS custom extension” is vendor-aligned.

### 1.3 D365 F&O source is stored as metadata XML and is version-controllable

Microsoft documents that the **Metadata folder contains source XML files organized by packages and models**.
This is what makes “AOT object creation → Git-tracked code” viable: if our add‑in produces/updates the metadata XML under the model/package structure, Git sees it.

Microsoft also defines models/packages: a **model belongs to a package**, and a package is a compilation/deployment unit with metadata/binaries/resources.

### 1.4 Deep links for forms require menu items

Microsoft docs state that access to forms is controlled through **Menu items** (security enforced there) and deep links only work for menu items that allow root navigation.
So “return a URL to view the form” must go through creating (or using) a Menu Item.

### 1.5 Coder Tasks + MCP + Codex are compatible

Coder Tasks:

* run agents “powered by Coder workspaces”
* **each task runs inside its own Coder workspace**
* **any terminal-based agent that supports MCP can be integrated**

Codex:

* supports MCP in CLI and IDE
* supports **STDIO servers** and **streamable HTTP servers**
* config lives in `~/.codex/config.toml`
* MCP servers can be managed with `codex mcp …`

Coder also supports Windows access via RDP and provides modules for web-based RDP on Windows workspaces.

---

# 2. Goals and non-goals

## Goals

1. **Codex runs only inside Coder in-cluster** (workspace for humans, tasks for agents).
2. **All D365 execution happens in the Windows dev VM** (create table/form via your VS add‑in, build/sync, etc.).
3. **All Git write operations happen in the VM** via tool calls:

   * checkout branch
   * apply changes
   * build/sync
   * commit & push (default: commit on successful sync)
4. **Coder/Codex has branch visibility externally** (mirrors/pulls from remote Git) so the agent can reason about the current state even though edits are executed in VM via AOT.
5. Provide **a deterministic return artifact** for “create table + form”:

   * commit SHA
   * branch name
   * form deep link URL

## Non-goals

* Running Telepresence or similar “network intercept” techniques in-cluster (not needed here).
* Replacing Argo CD / preview-env testing for Kubernetes services (this spec is for D365 FO devbox workflow).
* Making Visual Studio UI automation the only path; we will structure the add‑in so it’s callable deterministically.

---

# 3. High-level architecture

## 3.1 Components

### A) Coder (in-cluster)

* **Human**: Coder workspace (browser code-server) used to run Codex and inspect the Git branch state.
* **Agent**: Coder Task workspace runs Codex non-interactively.

### B) “FO MCP Bridge” (inside the Coder image)

A small process packaged into the Coder workspace image that exposes tools to Codex via MCP (STDIO).

* Codex calls MCP tools.
* The bridge translates tool calls into HTTPS RPC calls to the VM service.

This is vendor-aligned because Codex supports STDIO MCP servers started locally by command.

### C) FO Executor Service (Windows VM)

A Windows service (or long-lived daemon) running on each dev VM that:

* manages sessions (working directories / solution instances)
* runs git operations in the VM
* calls into Visual Studio add‑in functionality to create artifacts (table/form/menu item)
* runs build and DB sync
* enforces “commit on successful sync” policy
* returns structured results (JSON)

### D) Visual Studio Add-in (your custom extension)

A D365 Developer Tools add‑in project that implements:

* “create table”
* “create form”
* “create menu item”
* optionally “add to project”
* optionally “run build / sync” if you want to keep everything in add-in land

Vendor-aligned because Microsoft provides the add-ins infrastructure and project type.

## 3.2 Data flow (who reads/writes what)

**Write authority:** VM working copy only
**Read authority:** Both VM (tools) and Coder workspace (mirror checkout)

* VM is the *only place* where code is mutated and committed.
* Coder workspace keeps a read-only mirror of the branch (pulled from remote) so Codex can reason using normal file context.

This is how we satisfy: “Coder should have visibility on the branch from the outside”.

---

# 4. Source control and branch policy

## 4.1 Repository layout assumptions (vendor-aligned)

* D365 metadata source lives in “Metadata folder with source XML organized by packages/models”.
* The dev VM’s local model store is typically under something like `K:\AOSService\PackagesLocalDirectory`.
  The add‑in/tooling must write the “source XML” into the mapped model/package path so it is version-controllable.

## 4.2 Branch naming

* Base FO branch: `fo/main` (or your chosen stable branch)
* Feature branches: `fo/agent/<user-or-task>/<yyyyMMddHHmm>-<slug>`

## 4.3 Git operations MUST be performed in the VM

We implement git through MCP tools that call the VM executor:

* `fo.git.checkout_base`
* `fo.git.create_branch`
* `fo.git.status`
* `fo.git.commit_push`

**Decision:** Cluster/Coder workspace may do **read-only fetch/clone** to mirror branch state, but may not push.

## 4.4 Commit policy (explicit decision)

**Default:** `commit_on_successful_sync = true`

Meaning:

* After `fo.db_sync` returns success, VM executor:

  1. stages allowlisted paths (metadata + relevant project files)
  2. commits with deterministic message
  3. pushes to remote

If sync fails:

* **No automatic commit**
* Codex can still inspect the failing state via VM file-read tools, and can optionally call an explicit `fo.git.commit_push` to checkpoint WIP.

**Rationale:** matches your “commit at every successful sync” requirement without polluting history with known-bad states.

## 4.5 “Visibility from outside” mechanism

After a successful sync+commit+push:

* VM returns `{branch, commit_sha, changed_files}`
* Coder workspace mirror runs `git fetch && git reset --hard <commit_sha>`
* Codex uses the mirrored repo as its primary “code context”

This makes the agent able to reason on the exact state that VM produced via AOT operations.

---

# 5. MCP tool contract (what Codex sees)

Codex supports MCP servers via `codex mcp …` and shared `~/.codex/config.toml`.
Coder Tasks supports MCP-capable terminal agents.

We expose one MCP server named `fo`.

## 5.1 Session tools

### `fo.session.open`

Creates/attaches a VM working directory.

**Input**

* `repo_url`
* `base_branch`
* `feature_branch` (optional; if omitted, VM creates one)
* `user_context` (Coder user id / task id)

**Output**

* `session_id`
* `repo_root_vm`
* `branch`
* `head_sha`

**VM behavior**

* clones repo if missing
* checks out branch
* ensures D365 metadata mapping paths exist

### `fo.session.close`

Cleans up, releases locks.

## 5.2 File visibility tools (to support reasoning even before commit)

These are for *reading* and *diffing* the VM working copy, including uncommitted changes.

* `fo.fs.read(session_id, path)`
* `fo.fs.list(session_id, path, glob)`
* `fo.git.diff(session_id, base_ref)`  → returns unified diff
* `fo.git.show_file_at_ref(ref, path)` → optional convenience

## 5.3 AOT/VS Add-in tools

These tools must execute via your VS add-in (as required).

### `fo.vs.create_table`

**Input**: `table_spec` (name, fields, indexes, groups, labels)
**Output**: `{created_objects, created_files, warnings}`

### `fo.vs.create_form`

**Input**: `form_spec` (name, datasource, design template, controls)
**Output**: `{created_objects, created_files, warnings}`

### `fo.vs.create_menu_item_display`

**Input**: `menu_item_spec` (name, form_name)
**Output**: `{menu_item_name, created_files}`

**Decision:** the agent must always create/ensure a menu item because form deep links and security flow through menu items.

## 5.4 Build and sync tools

### `fo.build(session_id, scope)`

Runs compile/build appropriate for the FO dev box.

### `fo.db_sync(session_id, mode)`

Runs DB sync.

**Success side effect (per policy):**

* if `commit_on_successful_sync` is true, VM stages/commits/pushes.

**Output**

* `build_ok`, `sync_ok`
* `commit_sha` (if committed)
* `branch`
* `logs_uri` (pointer to stored logs)

## 5.5 Git tools (VM writes only)

* `fo.git.status`
* `fo.git.commit_push(message, paths_allowlist?)`
* `fo.git.create_branch(base, name)`
* `fo.git.push`

## 5.6 “Return a URL to open the form”

### `fo.form.deep_link`

**Input**: `company` (cmp), `menu_item_name`
**Output**: `url`

**Rule:** use menu item deep link because that’s how form access/security is enforced.

---

# 6. End-to-end orchestration flows

## 6.1 Agent Task: “Create a table and a form”

**Goal:** produce a PR-ready branch + a URL to view the form.

1. `fo.session.open(repo_url, base_branch=fo/main)`
2. `fo.git.create_branch(...)` (if not already created)
3. `fo.vs.create_table(table_spec)`
4. `fo.vs.create_form(form_spec)`
5. `fo.vs.create_menu_item_display(menu_item_spec)`
6. `fo.build(scope=model-or-package)`
7. `fo.db_sync(mode=full)`

   * if success → VM auto commits & pushes (policy)
8. `fo.form.deep_link(cmp=USMF, mi=<menu item>)` → returns URL
9. (Optional) `fo.github.create_pr(...)` or return branch for external PR creation
10. `fo.session.close`

**Outputs (task result contract)**

* `branch`
* `commit_sha` (from sync commit)
* `form_url`
* summary of created objects/files
* build/sync logs link

## 6.2 Human interactive: “Ask Codex, then inspect”

Human runs Codex in their Coder workspace:

* same tool calls
* after commit, the Coder workspace mirror fetches and the code-server view updates
* human can:

  * review diff in code-server
  * open the returned `form_url`
  * optionally RDP into VM for debugging (Coder supports RDP access patterns).

---

# 7. How “Coder has visibility” works in practice

## 7.1 Mirror repo in Coder workspace

* Coder workspace contains a local clone at `/workspaces/fo-repo`
* It is **read-mostly**
* After each successful VM push, Codex (or the MCP bridge automatically) runs:

  * `git fetch origin <branch>`
  * `git reset --hard <commit_sha>`

This ensures the agent’s local context matches the actual VM-produced state.

## 7.2 Why this works even though edits happen via AOT

Because D365 source is persisted as **metadata XML** under the metadata folder layout (packages/models).
So “AOT object creation” translates to changes in those XML files, which are committed and pulled like normal code.

---

# 8. VM executor implementation requirements (non-negotiable)

## 8.1 Determinism + audit

Every tool call must:

* be idempotent when possible
* emit a correlation id (`task_id`, `session_id`, `operation_id`)
* log:

  * who invoked it (Coder identity)
  * repo + branch + head SHA before/after
  * created objects/files
  * build/sync results

## 8.2 Concurrency control

**Decision:** one active session per `{devbox, branch}` to avoid VS / AOT collisions.

Implement:

* VM-local lock (file lock or mutex) keyed on branch.

## 8.3 Staging allowlist for commits

We must prevent committing binaries or volatile outputs.

**Decision:** stage only:

* `Metadata/**`
* `Projects/**` (if relevant)
* model descriptor files where applicable
* exclude `bin/**`, `XppMetadata/**`, logs, build outputs

(Exact allowlist depends on your repo layout, but the principle is: commit only source.)

---

# 9. Security model

## 9.1 Auth between Coder workspace and VM executor

**Decision:** mTLS with short-lived client certs issued per Coder user/task.

* MCP bridge holds no long-lived secrets.
* VM executor validates:

  * cert issuer (your CA)
  * subject (Coder user/task id)
  * allowed operations (RBAC policy)

## 9.2 Git credentials inside the VM

**Decision:** VM executor uses a **scoped Git identity** (GitHub App installation token preferred; short-lived) to push to repo.
No PATs embedded in templates.

## 9.3 Principle of least privilege

* VM executor can only operate on configured repos/orgs.
* Operations are allowlisted:

  * create table/form/menu item
  * build/sync
  * git commit/push
* No arbitrary command execution tool.

---

# 10. Testing plan (Coder-native)

Coder’s own recommended posture is that Tasks are first-class and run inside workspaces, and MCP-capable agents can be integrated.
So the **most honest E2E test** is “run a Coder Task that uses the tools”.

## 10.1 Unit tests (fast, per commit)

* MCP bridge:

  * schema validation of tool inputs/outputs
  * auth token/cert injection
  * error mapping
* VM executor:

  * git operations (in temp repo)
  * session locks
  * staging allowlist

## 10.2 Integration tests (VM required)

Run nightly or gated:

1. Create a scratch feature branch
2. `fo.vs.create_table` + `fo.vs.create_form` + `fo.vs.create_menu_item_display`
3. build + sync
4. assert:

   * build/sync success
   * commit exists and pushed
   * metadata files changed in expected directories
5. cleanup branch

## 10.3 End-to-end test (full chain)

A single Coder Task:

* triggers the flow above
* then in the *Coder workspace mirror*:

  * fetches branch
  * verifies files exist
  * verifies the returned `form_url` matches expected `cmp/mi` structure (and optionally curls it if reachable)

(We do not rely on UI automation; we validate the returned URL contract, and optionally validate reachability.)

## 10.4 CLI phase mapping (required)

For the `365fo` agent profile, the CLI surface maps as follows (see `backlog/654-agentic-coding-cli-surface-coder.md`):

- Phase 0/1/2 front door: `ameide test`
  - delegates execution to the FO tool surface (MCP bridge → VM executor)
  - emits JUnit evidence under the run root in the orchestrator workspace/task
- Phase 3 front door: `ameide test cluster`
  - validates the full chain described in §10.3 (including mirror checkout validation)

---

# 11. Operational model (what we run where)

## 11.1 In cluster

* Coder control plane (already in your GitOps)
* Linux Coder templates for human workspaces + tasks
* MCP bridge binary in the workspace image (STDIO server)
* Network policy/egress to reach VM executor endpoints

## 11.2 In VM

* D365 environment + VS tooling (vendor requirement)
* VS Add-in (your extension) (vendor-aligned add-in infrastructure)
* VM executor service
* Git client + credentials integration

---

# 12. Summary of key decisions (no open questions)

1. **Codex runs only in-cluster** (Coder workspace/task).
2. **Execution runs only on the D365 dev VM** (developer requires RDP/VM/VS).
3. **Artifact creation uses your VS add-in** (Microsoft-supported add-ins infrastructure).
4. **VM performs all git writes** (commit/push); default policy is **commit after successful DB sync**.
5. **Coder has outside code visibility** by mirroring the pushed branch; D365 source is XML metadata under packages/models so it’s commit-friendly.
6. **Form URL is returned via menu item deep link**, because menu items enforce form access/security.

---

If you want, I can turn this into a concrete interface document (JSON schemas for each MCP tool + expected outputs), plus a minimal “reference implementation outline” for the VM executor (session store, lock strategy, staging allowlist, and the VS add‑in IPC contract).
