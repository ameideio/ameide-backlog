---
title: 641 – Camunda Integration (Platform App): BPMN Element Editor Plugin
status: draft
owners:
  - platform
created: 2026-01-12
updated: 2026-01-13
---

## Summary

Deliver a **Camunda-like BPMN authoring experience** in the AMEIDE Platform by implementing a **BPMN element editor plugin** using the open modeling toolkit (`bpmn.io` / `bpmn-js`) and wiring it into the existing editor-plugin/session architecture.

This is *not* “embed Camunda 8 Web Modeler UI”. It is:

- A real `ElementEditorModal` host (modal chrome + plugin selection + session lifecycle).
- A new BPMN editor plugin (`element-editor-bpmn`) that edits **raw BPMN XML** stored in `Element.body`.
- A first-class **Verify** action that calls the same verification logic/endpoint as CLI/back-end so UI cannot drift.

This integration uses **open-source libraries only** and does not require any Camunda commercial components.

## Relationship to Camunda Web Modeler (separate concern)

Camunda Web Modeler is tracked separately as a GitOps concern in `backlog/642-camunda-web-modeler-identity-gitops.md` (and is intended to be **dev-only** unless explicitly enabled elsewhere).

This backlog (641) remains about the **AMEIDE Platform UI** editor plugin. Do not conflate “use Web Modeler in the browser” with “build a BPMN editor into the platform app”.

**Where Web Modeler lives (dev AKS):**

- Web Modeler (UI): `https://modeler.dev.ameide.io/`
- Web Modeler (REST API): `https://modeler-api.dev.ameide.io/`
- Web Modeler (WebSockets): `wss://modeler-ws.dev.ameide.io/`
- Camunda Operate/Tasklist (Orchestration apps): `https://camunda.dev.ameide.io/`

The AMEIDE Platform BPMN editor plugin is a **separate experience** embedded in the platform UI:

- It uses `bpmn-js` to edit BPMN XML stored in `Element.body`.
- It is governed by the AMEIDE BPMN profile + Verify semantics.

Both experiences can coexist: Web Modeler for users who want the Camunda-native UI, and the platform BPMN editor for the AMEIDE-native element workflow.

## What We Learn From “Camunda Web Modeler” (And What We Don’t)

Useful:

- BPMN editing UX is a first-class product surface; BPMN is treated as “just another element editor” (like ArchiMate).
- The best “portable base” is the open toolkit (`bpmn-js` + properties panel + moddle extensions), not a vendor UI embed.

Not applicable:

- **Embedding** Camunda 8 Web Modeler (self-managed) inside our platform app is not a drop-in integration; it is a separate app with its own auth/session model and infrastructure assumptions.
- The “Helm combined ingress” docs are about deploying Camunda’s services, not about implementing an editor inside our platform UI.

## Context

- The platform already has the right architectural seam:
  - “editor plugin + session model” exists (see the editor plugin/session interfaces referenced by the platform).
  - Working editors exist (Document + ArchiMate).
- BPMN is already a first-class element kind in the metamodel (`ELEMENT_KIND_BPMN_VIEW = 31`).
- There is no BPMN editor implementation folder today.
- Platform tests already assume a real modal shell via the Playwright POM (modal test id + close/fullscreen controls).
- Repo-level BPMN profile work exists and informs UI semantics:
  - `backlog/511-process-primitive-scaffolding-bpmn-extension.md`
  - `backlog/511-process-primitive-scaffolding-bpmn.xsd`

## Code reality (today)

The element editor route exists but renders nothing:

- The “element editor modal” route is wired, but the modal returns `null`, so the UI is blank.
- The Playwright POM expects `data-testid="editor-modal"` with close/fullscreen, etc., so UI and tests are currently mismatched.
- Docs reference a non-existent BPMN editor folder/path; docs are updated to match the real implementation.

## Goals

1. Implement a real `ElementEditorModal` host that renders the modal shell expected by tests and users.
2. Load element metadata and select the correct editor plugin by `element.kind`.
3. Add a BPMN editor plugin:
   - Edit BPMN using `bpmn-js` Modeler.
   - Save/load BPMN XML to/from `Element.body`.
   - Provide a properties panel surface for AMEIDE extensions (`ameide:*`) as part of the modeling experience.
4. Add a **Verify** button that calls the existing verification logic/endpoint (same semantic contract as CLI).
5. Remove or update stale BPMN docs so they point at the real plugin path and behavior.

## Non-goals

- Embedding Camunda 8 Web Modeler as-is.
- Implementing every Camunda feature (collaboration, deployments, connector catalogs, etc.).
- Defining a brand new BPMN semantics profile in the UI; the UI implements the existing profile and verification.
- Making this backlog depend on deploying Camunda 8 in-cluster; BPMN editing works independently of a running Camunda stack.

## Implementation (minimal, honest, scalable)

### A) Implement `ElementEditorModal` host (platform app)

Requirements:

- Render modal chrome with:
  - `data-testid="editor-modal"`
  - close button
  - fullscreen toggle
- Fetch/resolve:
  - element `title/name`
  - element `kind` (to choose plugin)
- Plugin routing:
  - `kind=Document` → existing Document plugin/session
  - `kind=ArchiMate` → existing ArchiMate plugin/session
  - `kind=BPMN_VIEW (31)` → new BPMN plugin/session
- Session lifecycle:
  - create session on open
  - track dirty state
  - block-close (or confirm) on unsaved changes (consistent with other editors)

### B) Create BPMN plugin package/folder

Add:

- `services/www_ameide_platform/features/element-editor-bpmn/`
  - `BpmnEditorPlugin.tsx` (plugin entry)
  - `BpmnSession.ts` (session: load/save XML; dirty tracking; verify action)
  - `bpmn/` moddle extensions + property provider (for `ameide:*`)

Behavior:

- **Storage**: `Element.body` stores the BPMN XML text (source of truth).
  - Load XML from `Element.body` into `bpmn-js` on open.
  - Export XML from `bpmn-js` back into `Element.body` on save.
- **Verify**:
  - A “Verify” action in the modal toolbar calls the existing verification endpoint/logic.
- UI displays results (pass/fail + actionable errors) without inventing a second verifier.

### C) Properties panel for AMEIDE BPMN extensions

Use the existing AMEIDE BPMN extension namespace:

- `https://ameide.io/schema/bpmn/extensions/v1` (see `backlog/511-process-primitive-scaffolding-bpmn.xsd`)

Implementation direction:

- Add a `bpmn-moddle` extension descriptor for `ameide:*`.
- Provide a properties panel that can edit all `ameide:*` fields required by the v1 execution/profile constraints documented in:
  - `backlog/511-process-primitive-scaffolding-bpmn-extension.md`
  - `backlog/511-process-primitive-scaffolding-refactoring.md`
  and can render validation errors returned by Verify.

At minimum, it must edit:
  - element name/id/documentation
  - `ameide:*` extension fields required for a “Verify: pass” outcome on conformant diagrams

## Test plan

- Playwright:
  - Existing POM `element-editor-modal` passes (modal renders, can close, can toggle fullscreen).
  - BPMN editor loads for BPMN elements and doesn’t crash on first open.
- Unit/integration:
  - Session load/save roundtrip preserves XML (string-in/string-out).
  - Verify button calls the correct endpoint and displays results (mock).

## Documentation updates

- Update/remove stale BPMN references in the platform docs/README so they:
  - point to `services/www_ameide_platform/features/element-editor-bpmn/`, and
  - describe “BPMN editing via bpmn-js + AMEIDE profile + Verify”, not “Camunda Web Modeler embedded”.

## Work items

- [ ] Implement modal shell (`ElementEditorModal`) to satisfy Playwright POM contract.
- [ ] Implement plugin selection by element kind and session creation.
- [ ] Add BPMN plugin folder + session with load/save of BPMN XML in `Element.body`.
- [ ] Wire `bpmn-js` Modeler (canvas) and basic UX affordances (zoom, fit, undo/redo).
- [ ] Add properties panel + moddle extension for `ameide:*`.
- [ ] Add Verify action wired to existing backend verification logic.
- [ ] Fix stale docs referencing non-existent BPMN editor path(s).

## Acceptance criteria

- Opening the element editor route shows a non-empty modal with `data-testid="editor-modal"` and working close/fullscreen controls.
- BPMN elements open in the BPMN editor (bpmn-js canvas visible).
- Saving updates `Element.body` with valid BPMN XML and re-opening shows the same diagram.
- “Verify” calls the existing verification logic/endpoint and renders results; UI and CLI verify semantics match.
- Playwright modal tests pass without changing their contract.
