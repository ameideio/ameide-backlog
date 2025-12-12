# 513 – UISurface Primitive Scaffolding (TypeScript, opinionated)

**Status:** Draft  
**Audience:** UI engineers, AI agents, CLI implementers  
**Scope:** Exact scaffold shape and patterns for **UISurface** primitives. One opinionated SDK‑first pattern, aligned with `514-primitive-sdk-isolation.md` (SDK-only, self-contained primitives). No new UISurface-specific CLI parameters beyond the canonical `ameide primitive scaffold` flags.

---

## General implementation guidelines (for CLI scaffolder)

- Maintain a single, opinionated scaffold per primitive kind; **do not introduce new CLI flags** beyond the existing `ameide primitive scaffold` parameters.  
- All scaffold instructions for UISurfaces must be expressed in **template files** (README/templates, comment templates), not inline string literals in the CLI codebase.  
- Generated README and code comments must be **exhaustive and self‑contained**, with **no references to external backlogs**; implementers should not need to consult design docs to understand the scaffold.  

---

## Grounding & cross‑references

- **Primitive stack:** `477-primitive-stack.md` (UISurface primitives in `primitives/uisurface/{name}`).  
- **SDK usage:** `services/www_ameide_platform/backlog/SDK_USAGE.md`.  
- **CLI overview & scaffold:** `484-ameide-cli-overview.md`, `484a-ameide-cli-primitive-workflows.md`, `484f-ameide-cli-scaffold-implementation.md`.  
- **UISurface operator:** `501-uisurface-operator.md`.  

---

## 1. Canonical scaffold command (UISurface / TypeScript)

For UISurface primitives we assume a Node/TS surface that **always** talks to Domain/Process via SDKs, not raw protos:

```bash
ameide primitive scaffold \
  --kind uisurface \
  --name <name> \
  --include-gitops \
  --include-test-harness
```

This creates a UISurface primitive under:

```text
primitives/uisurface/<name>/
gitops/primitives/uisurface/<name>/
```

---

## 2. Generated structure (UISurface / TS)

```text
primitives/uisurface/<name>/
├── README.md                            # Usage, SDK wiring notes
├── catalog-info.yaml                    # Backstage component
├── package.json                         # Node/TS project
├── tsconfig.json                        # TypeScript config
├── jest.config.cjs                      # Jest test config
├── Dockerfile                           # Production image
├── src/
│   ├── server.ts                        # HTTP/Connect server entrypoint
│   ├── routes/
│   │   └── <name>.ts                    # Example route(s)
│   └── proto/
│       └── index.ts                     # SDK-based client wiring (no @ameide/core-proto imports)
└── tests/
    └── server.test.ts                   # Failing tests for routes/server
```

GitOps with `--include-gitops`:

```text
gitops/primitives/uisurface/<name>/
├── values.yaml                          # UISurface Deployment/service config
├── component.yaml                       # Argo CD component descriptor
└── kustomization.yaml                   # Kustomize stub
``>

---

## 3. Opinionated SDK‑first pattern

Scaffolded UISurface primitives must:

- Use the **published SDKs** (`@ameideio/ameide-sdk-ts`) for all backend calls.  
- Avoid importing `@ameide/core-proto` directly in runtime code (consistent with `SDK_USAGE.md`).  
- Provide a `src/proto/index.ts` that:
  - wraps SDK proto clients or high‑level SDK helpers;  
  - becomes the **only** place proto types are referenced in the UISurface layer.

Routes/handlers under `src/routes/**` should:

- Accept HTTP/Connect requests,  
- Call SDK clients,  
- Return JSON/Connect responses,  
- Never deal with raw `*_pb.ts` types directly.

---

## 4. Test semantics (UISurface / TS)

Scaffolded tests:

- Live in `tests/server.test.ts`.  
- Are RED by default:
  - start the `src/server.ts` app in test mode,  
  - hit one or more example routes,  
  - assert a placeholder failure with TODO comments.

Agents/humans are expected to:

1. Implement real routes and map them to SDK calls (e.g., Scrum views using `ScrumQueryService` via SDK).  
2. Replace placeholder assertions with real ones (status codes, payload shapes).  
3. Ensure no direct proto imports leak into route code.

---

## 5. Verification expectations

`ameide primitive verify --kind uisurface --name <name>` should check:

- Presence of `src/server.ts` and `src/routes/**`.  
- `src/proto/index.ts` imports from SDK (`@ameideio/ameide-sdk-ts`), not `@ameide/core-proto`, in line with `SDK_USAGE.md`.  
- Tests exist under `tests/` and initially fail.  
- GitOps manifests are valid when present.

Behavioral details (what routes exist, what UX looks like) remain in vertical slices and UX backlogs; this document defines only the **scaffold shape and SDK usage pattern** for UISurface primitives.
