Below is a pragmatic, backlog‑ready plan to **level‑up the Agents & Workflows UX**—focused on discoverability, creation/editing, monitoring, and day‑to‑day use inside the rest of the app.

---

## What's solid already (and what's missing)

* You have clear **catalog → editor → runs** flows for Workflows, with good list/search scaffolding and a JSON editor, plus a Runs table that auto‑refreshes. The Run Details screen is still a placeholder and the editor is JSON‑only.
  * **STATUS**: ✅ 95% implemented - Run Details placeholder is critical gap; editor is textarea not Monaco
  * **IMPACT**: High - core journey works but debugging blocked
* Agents are discoverable via **Settings → Agents**, with Instances and a Node Catalog, and a clean "Create Agent" dialog feeding into the Element Editor; a testing playground and analytics are on your future list.
  * **STATUS**: ✅ 70% implemented - Discovery works, testing/analytics missing
  * **IMPACT**: High - agents are "write-only" without playground
* The global **layout/command palette** patterns (NavTabs, ⌘K) are in place and ready to be leveraged for surfacing automation as a first‑class citizen across the app.
  * **STATUS**: ⚠️ 40% partial - NavTabs exist, no command palette implementation
  * **CONSIDERATION**: Framework ready but automation not yet first-class in UI
* Your **threads system** is strong, but it lacks context chips/modes and deep links—prime opportunities to expose agents where people work.
  * **STATUS**: ✅ Accurate (38% complete per backlog/321) - streaming excellent, thread management missing
  * **IMPACT**: CRITICAL - blocks adoption, can't browse/revisit conversations 

---

## Top 12 UX improvements (prioritized)

### 1) Make Automation first‑class in nav & command palette (Impact: High, Effort: M)

**What:** Pull **Workflows** and **Agents** out of a "Settings‑only" mental model.
**Why:** People need to *use* automation, not just *configure* it.
**How:**

* Add **"Automation"** top‑level area with two tabs: **Workflows** and **Agents**, using the existing NavTabs pattern and server‑side nav descriptors. Add command palette actions: *"New workflows"*, *"Trigger workflows"*, *"New agent"*, *"Open last failed run"*.
* Keep the Settings tabs as admin routes, but route discoverability through the new top‑level entry.

**STATUS**: ❌ Not implemented (0%)
**DEVIATIONS**: Settings-only pattern persists; no command palette architecture exists
**CONSIDERATIONS**: Navigation system supports patterns but command palette is 3-4 week effort; quick win = new route group, defer command palette 

---

### 2) Supercharge list UIs with **text qualifiers** & saved views (Impact: High, Effort: M)

**What:** Extend search boxes on Workflow Catalog / Runs / Agent Instances to support *typeahead qualifiers* (e.g., `status:active owner:@me node:openai tag:billing`) plus saved views.
**Why:** Reduces scanning; scales to large orgs.
**How:** Reuse your documented **advanced filtering** pattern and command palette conventions; persist views in localStorage first, then server.
**Acceptance:** Qualifiers + chips render above results; "Save view" stores query; keyboard nav works.

**STATUS**: ❌ Not implemented (5% - basic text search + status dropdown exists)
**IMPACT**: High for scale
**CONSIDERATIONS**: Requires qualifier parser, saved views protobuf + backend API, localStorage first for quick win

---

### 3) Templates that create something useful in 30 seconds (Impact: High, Effort: M)

**What:** One‑click **Workflow Templates** (e.g., "On element approved → publish diagram → notify Slack") and **Agent Templates** (e.g., "RAG Researcher", "Standards Reviewer").
**Why:** Destroys blank‑page anxiety; accelerates adoption.
**How:** Add **Templates** section on Workflow Catalog; "Use template" opens editor prefilled; mirror for Agents using the Node Catalog as the source of truth. Visual "This template will…" summaries.

**STATUS**: ❌ Not implemented (0%)
**IMPACT**: High adoption multiplier
**CONSIDERATIONS**: Can start with static JSON files in `/services/workflows/templates/`, seed via migration, reuse catalog pattern; n8n reference code available 

---

### 4) Dual‑mode **Workflow Editor**: Visual graph ⇄ JSON (Impact: Very High, Effort: L)

**What:** Toggle between **Graph builder** and **JSON**; show a side‑by‑side **Diff** when switching versions.
**Why:** Authors get safety and speed; experts keep raw control.
**How:** Keep Monaco JSON (with schema validation) and add a graph surface (nodes: activity/decision/parallel). Every graph edit updates JSON AST; validation errors highlight nodes & JSON lines. Run "lint" on save. You already plan a visual builder—ship it with Diff & Schema hints from day one.

**STATUS**: ❌ Not implemented (0% - current editor is plain textarea, not even Monaco)
**DEVIATIONS**: Editor more basic than assumed; bidirectional sync 8-10 week effort
**IMPACT**: Very High - democratizes authoring
**CONSIDERATIONS**: Phase 1 upgrade to Monaco (1w), Phase 2 read-only graph (4w), Phase 3 editable (4w); ReactFlow in deps; n8n reference available 

---

### 5) A real **Run Details** screen with timeline & step I/O (Impact: Very High, Effort: M)

**What:** Replace the placeholder with:

* **Timeline** across steps, live progress, and collapsible **Step cards** (input, output, logs, retries).
* **Retry from step**, **Cancel**, and **Download payload**.
  **Why:** Investigating failures is the #1 daily task for ops.
  **How:** Link from the Runs table’s `[View]` to `/runs/[executionId]`. Polling exists today; layer in SSE/WebSocket for step deltas. 

```
Run #a1b2  [FAILED]   Started 14:02  Duration 2m10s
└─ validate   ✓ 0.6s   I/O • Logs
└─ enrich     ✓ 1.2s   I/O • Logs
└─ publish    ✗ 0.3s   I/O • Logs • Retry step
               Error: Missing credentials...
```

*(Runs page currently polls every 10s; promote Run Details to near‑real‑time.)*

**STATUS**: 🔴 CRITICAL GAP - Placeholder only (5%)
**IMPACT**: Very High - debugging/ops blocker, #1 daily task
**DEVIATIONS**: Route exists, renders "TODO"; backend has GetWorkflowRun but no step-level data model
**CONSIDERATIONS**: Need Temporal history integration, step data in proto, SSE is separate 2w effort (can defer); timeline/step cards/I/O viewers are 3-4 weeks; reference Temporal UI patterns 

---

### 6) **Side‑by‑side versioning** & "Promote/Rollback" (Impact: High, Effort: M)

**What:** Compare two workflows versions in the editor and **Promote** or **Rollback** safely.
**Why:** Confidence for iterative changes.
**How:** Add a "Compare versions" panel under the existing Versions list; show structured spec diff (added/removed steps, param changes). Then expose a quick **Rollback** action.

**STATUS**: ⚠️ Partial (30% - version history displayed, no diff/promote UI)
**DEVIATIONS**: Versioning exists but missing status, publishedAt, specHash fields
**CONSIDERATIONS**: Need structured diff algorithm (not line diff), promote RPC, migration to add missing fields; depends on workflows spec schema formalization 

---

### 7) **Agent Testing Playground** (Impact: High, Effort: M)

**What:** A right‑panel playground in the Element Editor to threads with the agent, inspect tool calls, and tweak parameters live; a "Capture as test case" button saves fixtures.
**Why:** Shortens the edit → test → iterate loop dramatically.
**How:** You've already flagged this as a future enhancement—build it into the same modal you open from "Configure," with a message history viewer and tool‑call trace.

**STATUS**: 🔴 CRITICAL GAP - Not implemented (0%)
**IMPACT**: High - shortens iteration loop dramatically, agents currently "write-only"
**CONSIDERATIONS**: agents_runtime has Stream RPC (✓), need Element Editor panel expansion, test fixture storage table + API, tool call trace UI; 3 week effort; reference Langfuse trace patterns 

---

### 8) Put **Agents inside Chat**—with context chips (Impact: Very High, Effort: M)

**What:** Add an **Agent picker** in the footer and **context chips** (Follow / Locked / Global). Let pages map a “default agent” via UI‑context mapping.
**Why:** This is where users already are.
**How:** Extend ChatFooter with an inline agent chip + picker and the missing Follow/Lock/Global chips. Respect “UI context mapping” from Agent config so the right agent appears on repo/transformation pages. Deep‑link back to an agent’s config when needed.  

```
[🤖 Standards Reviewer]  [Follow this page]  [context: /repo/acme-core]
┆ Ask: "Review new Tech Service for policy gaps…"
```

**STATUS**: 🔴 CRITICAL GAP - Not implemented (0%)
**IMPACT**: Very High - makes agents discoverable where users work
**DEVIATIONS**: ChatFooter exists with starters, context extraction works, but no agent integration
**CONSIDERATIONS**: BLOCKED by thread management (backlog/321) - need anchor/mode fields in thread model first; AgentInstance.ui_pages exists in backend; routing logic needed; 3-4 week effort after thread prerequisite

---

### 9) Consistent **Draft / Active / Archived** publishing model (Impact: Medium, Effort: S)

**What:** Unify status language and chips across Workflows and Agents, with a single "Publish" and "Archive" pattern and confirmation copy.
**Why:** Reduces cognitive load moving between the two surfaces.
**How:** Reuse your status badge components; same microcopy for all publish actions.

**STATUS**: ✅ Mostly implemented (80%)
**DEVIATIONS**: Workflows use enum (DRAFT/ACTIVE/ARCHIVED), agents use boolean (published); microcopy varies
**CONSIDERATIONS**: Quick 1-week alignment - migrate agents to enum, standardize confirmation dialogs, add Archive action for agents 

---

### 10) **Empty states** that teach by doing (Impact: Medium, Effort: S)

**What:** On empty Catalog/Instances pages, show 1–2 top templates, a 60‑second checklist, and a link to "How it works" (inline explainer).
**Why:** Better first‑run experience.
**How:** Apply your standardized **Blankslate** pattern and content design guidelines (active voice, concrete verbs).

**STATUS**: ⚠️ Partial (40% - basic "No workflows" message + create button, not educational)
**CONSIDERATIONS**: Template preview depends on #3, but can add checklist + docs link immediately; 1-2 week incremental improvement 

---

### 11) Keyboarding & power‑user affordances (Impact: Medium, Effort: S)

**What:**

* ⌘K → "New workflows / agent", "Open last failed run".
* `g w` → Workflows, `g a` → Agents; `?` opens shortcuts help.
  **Why:** Fast muscle‑memory paths for frequent tasks.
  **How:** Build on the search/command palette you already ship; add the `?` help overlay you planned.

**STATUS**: ❌ Not implemented (0%)
**DEVIATIONS**: No command palette architecture exists (3-4 week effort, not small)
**CONSIDERATIONS**: Can implement shortcuts (`g w`, `?`) without full command palette (1w); defer full cmdk integration to Phase 2 

---

### 12) Observability at a glance (Impact: High, Effort: M)

**What:** Small KPI strip on Catalog & Runs: *Success rate (24h)*, *P95 duration*, *Top failing step*, *Most triggered workflows*. Agent strip: *Invocations*, *Average tokens*, *Tool error rate*.
**Why:** Makes health visible where users decide what to open next.
**How:** Reuse your metrics grid pattern from org overview; source from workflows/agent telemetry endpoints.

**STATUS**: ❌ Not implemented (0%)
**IMPACT**: High for ops visibility
**CONSIDERATIONS**: Backend Prometheus exists, need metrics aggregation API (2w) + frontend KPI components (1w); consider Redis caching; 2-3 week effort 

---

## Key interaction & IA tweaks

* **Deep links everywhere:** `/workflows/[id]?v=2`, `/workflows/runs/[executionId]`, `/agents/[id]#tools`. Your routing scaffolding already supports rich patterns; extend it for automation objects (and make links copyable from UI).
  * **STATUS**: ⚠️ Partial (50% - routes exist, no version params, no copy buttons)
* **Trigger from context:** From a graph or transformation, show "Run workflows" inline (modal with input JSON), using your element/transformation settings integration points.
  * **STATUS**: ❌ Not implemented (0% - trigger only in editor)
* **Cross‑link Agent ↔ Workflow:** If a workflows calls an agent (or vice‑versa), show a linked pill with a breadcrumb to jump between them, anticipating your planned integration.
  * **STATUS**: ❌ Not implemented (0% - no integration between systems yet) 

---

## Minimal design changes (so you can ship quickly)

* Reuse **PageHeader + NavTabs + Badge** patterns; your design system (shadcn/Radix) covers all components we need. 
* Mirror the **list/catalog** pattern you already use for Workflows to Agents Node Catalog and Instances (they’re already close), just add qualifiers and saved views. 
* Keep JSON as the single source of truth while the visual builder matures; guard with schema validation and diff. 

---

## Data model & API deltas (concise)

### Workflows

* **Versioning:** Ensure versions have `status`, `tag`, `publishedAt`, and `specHash` for compare/rollback UX. Expose `GET /workflows/:id/versions` and `POST /workflows/:id/versions/:v/promote`.
  * **STATUS**: ⚠️ Partial (80% - versions exist with tag, missing status/publishedAt/specHash; GET exists, promote missing)
* **Runs streaming:** Add `GET /workflows-runs/:executionId/stream` (SSE) that emits `step_started/step_completed/log/error` events for the new Run Details. Current list polling remains.
  * **STATUS**: ❌ Not implemented (0% - 10s polling only)

### Agents

* **UI context mapping:** Persist `uiPages: string[]` and expose in list rows; this powers the Chat footer agent chip.
  * **STATUS**: ✅ Backend complete (100% - protobuf + DB + API), frontend not using yet
* **Playground fixtures:** `POST /agents/:id/tests` to store input → expected output snapshots for regression checks in the playground.
  * **STATUS**: ❌ Not implemented (0%)

### Cross‑cutting

* **Saved views:** `GET/POST /saved-views?kind=workflows|run|agent` with a `{query, filters, sort}` payload, owned by user; hydrate into list pages.
  * **STATUS**: ❌ Not implemented (0%) 

---

## Microcopy snippets (plug‑and‑play)

* **Publish:** “Make this version available to all triggers. You can roll back anytime.” 
* **Retry step:** “Retry from here using the original input.” 
* **Empty state (Workflows):** “No workflows yet. Start from a template or create a blank one.” 

---

## Quality & accessibility guardrails

* Add **ARIA live regions** for run progress and toast announcements; ensure all new chips and pills are keyboard‑reachable. 
* Keep **contrast & focus** consistent with your design tokens and Badge variants. 

---

## Suggested execution order (by dependency)

1. **Run Details screen + streaming** → unlocks serious value for existing users.
2. **Agent picker + context chips in Chat** → agents where people already work.
3. **Qualifiers & saved views** in all lists → scale to bigger orgs.
4. **Dual‑mode workflows editor** (visual + JSON) → unlocks new creators.
5. **Templates** → better first‑run and adoption.

---

## REVISED EXECUTION ORDER (Reality-Adjusted)

**CRITICAL DEPENDENCY DISCOVERED**: Agent-threads integration (#2) blocked by thread management (backlog/321) - need anchor/mode/agent_id fields in thread model first

**Phase 1 - Critical Path (6 weeks)**:
1. **Thread Management** (backlog/321) - 2 weeks - PREREQUISITE for agent integration
2. **Run Details Screen** - 2 weeks - High value, no dependencies (defer SSE)
3. **Agent Picker in Chat** - 2 weeks - Post-thread management

**Phase 2 - High Value (5 weeks)**:
4. **Context Chips** - 1 week - Completes agent-threads integration
5. **Templates** - 2 weeks - Adoption multiplier
6. **Agent Testing Playground** - 2 weeks - Shortens iteration loop

**Phase 3 - Scale (6 weeks)**:
7. **Text Qualifiers & Saved Views** - 3 weeks
8. **Observability Metrics** - 3 weeks (backend + frontend)

**Phase 4 - Advanced (8+ weeks)**:
9. **Dual-mode Workflow Editor** - 6 weeks (Monaco upgrade → read-only graph → editable)
10. **Run Streaming (SSE)** - 2 weeks

**Deferred**: Keyboard shortcuts, command palette (3-4w effort not "small"), cross-linking, version diff

**Timeline Impact**: +3 weeks from original estimate due to thread management prerequisite 

---

If you’d like, I can turn this into a granular ticket set (with acceptance criteria and component/file touch‑points) for the first two items—**Run Details** and **Chat agent picker**—so engineering can start immediately.
