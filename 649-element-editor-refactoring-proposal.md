---
title: "649 – Element Editor refactor: real modal host + plugin sessions + BPMN editor"
status: draft
owners:
  - platform-ui
  - platform-devx
created: 2026-01-12
---

# 649 – Element Editor refactor: real modal host + plugin sessions + BPMN editor

## 0) Intent

Make the **Element Editor** a real, reliable platform surface:

- A real **modal host** (not a placeholder) that is the single entry point for editing elements.
- A **plugin + session** model that standardizes load/save and keeps editors isolated.
- A first-class **BPMN view editor** that is “Camunda-like” in UX (bpmn.io / `bpmn-js`), while remaining Ameide-owned and profile/verify-driven.

This backlog is explicitly “no band-aids”: we remove drift (docs/tests/code) rather than shimming around it.

## 1) Problem statement (current reality)

Today, the platform has the right building blocks, but they don’t meet each other:

- The modal route is wired, but `ElementEditorModal` renders nothing (placeholder), so the UI surface is effectively missing.
- We already have a plugin interface and working plugins (Document, ArchiMate), but no real modal host that selects and renders them.
- Tests assume a real modal DOM contract (test IDs, buttons, loading/error text), but the UI doesn’t implement it.
- The metamodel declares BPMN element kinds, but the UI has no BPMN editor implementation; docs reference a non-existent `features/bpmn/` implementation, creating confusion.

## 2) Goals (definition of done)

1. **Element editor modal is real**
   - Modal renders consistently with stable test IDs.
   - Opens via element route, loads element metadata, shows loading/error states, and renders an editor plugin.
2. **Plugin + session contract is enforced**
   - Every editor uses a session to load/save state through the Element API.
   - Modal host owns session lifecycle and error handling; plugins stay focused on editor UI.
3. **BPMN view editing exists**
   - `ELEMENT_KIND_BPMN_VIEW` opens a BPMN modeler with import/export of BPMN XML stored in `Element.body`.
   - BPMN editing is compatible with Ameide’s BPMN profile and uses “verify” as a first-class guardrail (not optional).
4. **Docs and tests match reality**
   - Remove/replace stale documentation that points at missing code.
   - Playwright tests validate the modal and at least one editor happy path end-to-end.

## 3) Non-goals (explicit)

- Embedding or re-hosting **Camunda 8 Web Modeler** itself.
- Multi-user real-time BPMN collaboration (CRDT/OT) in v1 of this refactor.
- Full BPMN execution semantics in the web app (this is an editor surface; runtime semantics remain the Process primitive).

## 4) Taxonomy (so we stop mixing terms)

- **Element**: Ameide’s canonical editable object (`metamodel.v1.Element`), identified by `elementId` and typed by `ElementKind`.
- **ElementKind**: drives editor selection (e.g., `DOCUMENT_MD`, `ARCHIMATE_VIEW`, `BPMN_VIEW`).
- **Element Editor Modal**: UI shell that loads an element and renders the correct editor plugin for it.
- **EditorPlugin**: UI implementation for one “kind family” (Document, ArchiMate, BPMN).
- **EditorSession**: standardized load/save bridge between a plugin and the backend (Element API).
- **Artifact**: legacy/UI wording that should be retired or treated as a synonym for Element (pick one term and make it consistent in code/tests).

## 5) Target architecture (platform-owned)

### 5.1 Modal host responsibilities (single place)

The modal host owns:

- **Open/close** lifecycle (route ↔ store ↔ modal)
- **Element fetch** (title/kind/metadata) and status UI
- **Plugin resolution** (by `ElementKind`)
- **Session creation** (`plugin.openSession({ client, elementId })`)
- **Error boundaries** (load/save failures surfaced consistently)
- **Stable DOM/test contract** (test IDs, chrome buttons, header title)

Plugins own only editor UX and editor-specific state.

### 5.2 Plugin resolution rules

- Select exactly one plugin by `acceptsElementKind(element.kind)`.
- If no plugin matches: show “unsupported kind” with the numeric/string kind value.
- If multiple plugins match: fail fast (developer error) and require tightening plugin predicates.

### 5.3 Persistence contract (Element.body `Any`)

Store editor bodies in `Element.body` with a clear `typeUrl` per content kind:

- Document editor: `typeUrl: text/html` (already implemented pattern).
- BPMN view editor: `typeUrl: application/bpmn+xml` (store BPMN XML bytes).
- ArchiMate view editor: define/confirm a `typeUrl` (likely JSON) and standardize it before implementing save.

Save uses `elementService.updateElement` with an `updateMask` targeting only what changed (`body`, `description`, etc.).

## 6) Refactoring plan (work packages)

### 6.1 WP0 — Align terminology and contracts

- Decide whether the UI surface calls this **Element** everywhere (recommended) and remove “artifact” wording in tests and UI labels.
- Define the modal’s DOM/test contract (test IDs, button semantics) as normative.

### 6.2 WP1 — Implement the real `ElementEditorModal`

- Implement modal chrome and route/store wiring.
- Add: loading state (“fetching …”), error state (“failed to load …”), header title, close/fullscreen/open-in-new-tab actions.
- Resolve plugin and create session; render plugin output inside a stable host (`EditorPluginHost`).

### 6.3 WP2 — Fix docs drift

- Update `services/www_ameide_platform/features/README.md` to reflect the plugin approach (or reintroduce the referenced code if it is intended to exist).
- Remove references to missing `features/bpmn/*` paths and replace with the new plugin location.

### 6.4 WP3 — Add BPMN editor plugin (bpmn.io / `bpmn-js`)

- Implement `element-editor-bpmn` plugin + session.
- Load/save BPMN XML from `Element.body`.
- Provide minimal UX:
  - canvas + properties panel
  - save on explicit action + optional debounce
  - “Verify” button that runs Ameide BPMN verification and blocks publish/save-to-mainline flows when invalid.

### 6.5 WP4 — Verifications (must be real)

- **Unit/integration**: plugin resolution, session load/save contract, modal load/error rendering.
- **Playwright**: modal opens by URL, renders header/title, closes; at least one editor (Document) proves persistence via save/reload.
- **BPMN smoke**: open BPMN view, import a valid BPMN file, save, reload, and verify passes.

## 7) Acceptance criteria

- The modal is visible and functional in CI and local dev (no “renders null” placeholders).
- All existing editor plugins render through the same host without bespoke wiring.
- BPMN view editing is available via `ELEMENT_KIND_BPMN_VIEW`.
- No stale docs point to missing codepaths; the implementation’s folder structure matches documentation.
- Playwright has at least one stable end-to-end test for the modal and one editor.

## 8) Open decisions (must be closed early)

1. **Terminology**: “Element” vs “Artifact” in UI/tests/code (pick one; migrate the other away).
2. **BPMN verification UX**: on-save verify vs explicit verify vs both (default should be explicit + required before publish).
3. **Autosave policy**: whether editors autosave by default or only on explicit save (make it consistent across plugins).
4. **ArchiMate save format**: define the canonical `typeUrl` + schema for ArchiMate view bodies before implementing save.

