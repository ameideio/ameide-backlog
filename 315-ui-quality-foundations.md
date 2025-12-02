# 315 – UI Quality Foundations & Consistency

## Summary
The current UI delivers feature depth, but inconsistent loading states, ad-hoc error handling, and divergent layouts erode user trust. This transformation establishes a cohesive design and interaction baseline before layering on additional functionality. The outcome should be a predictable interface where navigation, feedback, and CRUD experiences behave the same across settings, repositories, transformations, and agent surfaces.

## Objectives
1. **Shared Data & Mutation Flow**
   - Normalize how client components fetch data, execute mutations, and force revalidation. Replace “local setState mocks” with reusable `useMutation` helpers that coordinate API calls, optimistic updates, and toast feedback.
   - Ensure every feature exposes an imperative `refresh()` backed by SWR/React Query or custom hooks so views can resync after edits without full page reloads.
   - Centralize error parsing (ConnectError → user-friendly message) and map gRPC/REST codes to UI status.
2. **Consistent Loading & Empty States**
   - Standardize skeleton loaders, placeholders, and empty-state messaging via dedicated components and design tokens. Eliminate bespoke markup per feature.
   - Provide content guidelines (iconography, copy tone) so user expectations are consistent.
3. **Accessible, Responsive Layouts**
   - Define a polished layout system: page shell, sidebar behavior, spacing, typography, responsive breakpoints, and focus management.
   - Ensure accessibility from the start: keyboard navigation, ARIA attributes, and focus return on modals/tabs.
4. **Actionable Error Recovery**
   - Replace console logging with visible banners/toasts, retry options, and offline detection.
   - Document a severity hierarchy (blocking vs. informational) and ensure each severity has a consistent visual treatment.

## Success Criteria
- All CRUD flows (settings tabs, agent management, transformation workspace) call centralized API helpers and trigger UI refresh hooks rather than mutating local state.
- A single “list state” primitive wraps graph, transformation, agent, and workflows lists and produces cohesive loading skeletons, empty messaging, and error presentation.
- Page-shell, sidebar behavior, spacing, and typography converge on documented design tokens; the `SettingsShell` wrapper is used by every settings-like route.
- Each destructive action (delete/archive) uses a shared confirmation dialog with consistent copy and toast feedback. Mutation helpers propagate success/error results automatically.
- Accessibility checklist (focus trap, keyboard navigation, ARIA labels, color contrast) passes for modals, tabs, menus, and primary buttons. Accessibility acceptance criteria are added to CI smoke tests.

## Scope

### In Scope
- Refactors within `app/(app)/org/[orgId]/...` and adjacent feature modules (`features/agents`, `features/editor-*`, `features/workflows`, `features/navigation`) to adopt shared primitives.
- Shared React primitives: `useMutation`, `useListState`, `ActionDialog`, `ToastProvider`.
- UI tokenization (spacing, typography, breakpoints) and layout wrappers.
- Updating developer docs (`docs/ui`, `docs/frontend/styles.md`), onboarding guides, and tests (`tests/integration/README.md`) to reflect new patterns.
- Light backend adjustments needed to surface consistent responses/messages (e.g., ensure all relevant endpoints return structured error payloads).

### Out of Scope
- Brand-level visual redesign (color palette, icon set changes beyond consistency fixes).
- Net-new feature domains (e.g., new dashboards) beyond supporting UI foundations.
- Deep backend refactors unrelated to consistency (e.g., new services, data models).

## Milestones

### Phase 1 – Data Flow Harmonization
- [ ] **Audit** existing fetching/mutation patterns (Org settings, Agents, Initiatives, Workflows) and document gaps (local state updates, missing refresh hooks).
- [x] **Implement `useAppMutation` hook** that wraps async operations, exposes `{ execute, isLoading, error }`, integrates with toast notifications, and rolls back state on failure. _(Landed in `features/common/hooks/useAppMutation.ts` with shared `ToastProvider` and error mapping, adopted by the user profile flow for first production usage.)_
- [ ] **Normalize API wrappers** (`api/repositories`, `api/transformations`, etc.) so they return typed payloads and map Connect errors to UI messages using a shared parser.
- [ ] **Refactor key flows** (graph create/update/delete, transformation create/update/archive, agent create/update) to use the new helper and call `refresh()` hooks.
  - [x] User profile, org settings repositories/transformations/feature toggles, and agent instance creation flows now use shared hooks (`useAppMutation`, React Query) instead of bespoke fetch + reload patterns.
- [ ] **Document** the required lifecycle for new mutations (optimistic update, toast, refresh) in `docs/frontend/mutations.md`.

### Phase 2 – Feedback Components & UX Guidelines
- [ ] **Design and implement `ListState` component** supporting the four canonical states (`loading`, `empty`, `error`, `content`) with slots for custom content and default copy.
- [ ] **Create `FeedbackBanner` and `Toast` primitives** with severity levels (info/success/warning/error) and mandate their use over raw `div` or console logs.
  - [x] `ToastProvider` + severity-aware `useToast` rolled out; `FeedbackBanner` component still to do.
- [ ] **Introduce `ActionDialog` component** encapsulating destructive confirmation copy, keyboard traps, and toast integration.
- [ ] **Update empty-state copy** across repositories, transformations, agents to follow content guidelines (title, description, CTA).
- [ ] **Add storybook/docs entries** for each new component so designers/developers have a single source of truth.

### Phase 3 – Layout, Navigation, and Tokens
- [ ] **Extract `SettingsShell` layout** that manages sidebar visibility (`useChatLayout` overrides), sticky headers, and consistent gutters. Adopt it in all settings-like pages.
- [ ] **Define design tokens** (spacing scale, typographic ramp, z-index constants, breakpoints) in `styles/tokens.ts` or Tailwind config extension; update components to use tokens.
- [ ] **Standardize skeleton loaders** across cards, tables, and detail panels; replace bespoke markup with tokenized skeletons.
- [ ] **Audit responsive behavior**: simulate key breakpoints (320, 768, 1024, 1440) and update CSS to avoid overflow/wrap issues in sidebar and card layouts.
- [ ] **Document layout conventions** (section header spacing, card padding, toolbar heights) in `docs/frontend/layout.md`.

### Phase 4 – Accessibility, Testing & Tooling
- [ ] **Accessibility pass**: ensure modals trap focus, `DialogTrigger` returns focus on close, keyboard navigation works (enter/space, ESC).
- [ ] **Add Playwright scenarios** covering keyboard-only flows for dialogs, tabs, and list interactions; integrate into CI.
- [ ] **Introduce ESLint rules/config** to block new `any` and unused variables, with suppression guidelines for legitimate cases.
- [ ] **Update Jest/React Testing Library suites** to cover the new components and ensure semantics (e.g., `aria-live` on toast region).
- [ ] **Update QA checklist** to include UI quality gate: load states, error recovery, accessibility, responsive layout.

## Risks & Mitigations
- **Risk:** Refactors destabilize production features (e.g., agents, workflows).  
  **Mitigation:** Roll out per surface (repositories → transformations → agents) with feature flags and Playwright smoke tests.
- **Risk:** Design alignment stalls without stakeholder input or unified guidelines.  
  **Mitigation:** Coordinate weekly checkpoints with design/PM; track open feedback in a shared Confluence/Notion page.
- **Risk:** New shared components conflict with existing notification systems or styles.  
  **Mitigation:** Inventory current usage, provide compatibility shims, and communicate migration plan before removing old patterns.
- **Risk:** Performance regressions from added abstraction layers.  
  **Mitigation:** Include perf budget (bundle size, interaction timing) in acceptance criteria; profile key flows after changes.
- **Risk:** Accessibility fixes take longer than expected due to systemic issues.  
  **Mitigation:** Prioritize highest-impact screens first (settings modals, lists) and engage accessibility experts early.

## Open Questions
- Should lint hardening (ban `any`, no unused vars) land at the start of the transformation or after main refactors to avoid churn?
- Which design token strategy best fits our stack (Tailwind theme extension, CSS variables, TypeScript constants)?
- How do we integrate the shared `SettingsShell` with threads layout overrides (`useChatLayout.shouldHideSidebars`) without regressing the threads experience?
- What are the minimum responsive targets (mobile, tablet, desktop) and do we need to support legacy browsers?
- Who owns long-term stewardship of the UI system (e.g., appoint a design system guild)?
- How should we version and document shared components (Storybook deployment, design tokens repo)?
