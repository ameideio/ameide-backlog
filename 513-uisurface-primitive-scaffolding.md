# 513 – UISurface Primitive Scaffolding (TypeScript, opinionated)

**Status:** Draft  
**Audience:** UI engineers, AI agents, CLI implementers  
**Scope:** Exact scaffold shape and patterns for **UISurface** primitives. One opinionated SDK‑first pattern, aligned with `514-primitive-sdk-isolation.md` (SDK-only, self-contained primitives). No new UISurface-specific CLI parameters beyond the canonical `ameide primitive scaffold` flags.

---

## Primitive/operator alignment

- **Operator responsibilities (495, 501, 497):** The UISurface operator reconciles `UISurface` CRDs into Deployments, Services, HTTPRoutes (Gateway API), Keycloak clients, ConfigMaps, HPAs, and monitoring resources. It is responsible for how traffic reaches a UISurface and how auth/observability are wired, but it does not implement UI routes or call backend SDKs.  
- **Primitive responsibilities (this backlog, SDK_USAGE):** The UISurface scaffold owns the **application layer**: HTTP server/Connect endpoints, route handlers, and SDK-based calls into Domain/Process/Agent primitives. It lives inside the image referenced by the UISurface CRD and encapsulates UI behavior and composition, independent of operator internals.  
- **Boundary:** Operators are the **control plane for hosting and routing**; UISurface primitives are the **frontend applications** that use SDKs to talk to backends. 513’s scaffold shape is designed so the operator only needs an image and a few config fields (host/path/auth), while the primitive remains fully in charge of UX and SDK wiring, consistent with `495-ameide-operators.md`, `497-operator-implementation-patterns.md`, and `501-uisurface-operator.md`.

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
- **Primitive/operator contract:** `495-ameide-operators.md`, `497-operator-implementation-patterns.md`.  
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

---

## 6. Implementation progress (CLI & scaffold)

This section describes the current implementation status of 513 in the CLI (`packages/ameide_core_cli`) and repo scaffolds. It is descriptive; the rest of 513 remains the target spec.

### 6.1 Scaffolder behavior for UISurface primitives

**Status:** Partially implemented; current UISurface scaffolds use a generic TS handler pattern, not the HTTP/SDK structure described here.

- `ameide primitive scaffold --kind uisurface`:
  - Uses the generic TypeScript path in `primitive_scaffold.go`:
    - Creates `primitives/uisurface/<name>` with:
      - `package.json`, `tsconfig.json`, `jest.config.cjs`,
      - `Dockerfile`,
      - `src/handlers.ts` (class with methods per RPC),
      - `tests/handlers.test.ts` (RED tests calling the handler).
    - Adds GitOps manifests under `gitops/primitives/uisurface/<name>` when `--include-gitops` is set.
  - Does **not** currently emit:
    - `src/server.ts`,
    - `src/routes/<name>.ts`,
    - `src/proto/index.ts` or any SDK wiring file.

- Templates vs inline strings:
  - There are no UISurface-specific templates under `templates/`.
  - TS scaffolding logic (handlers, tests, tsconfig, Dockerfile) is generated from inline string builders, not from templates as 513 envisions.

### 6.2 SDK-only and HTTP surface behavior

**Status:** Planned in 513, but not enforced or scaffolded yet.

- Runtime shape:
  - The spec’s HTTP/Connect server + routes + SDK clients pattern is not reflected in current scaffolds.
  - Generated TS code is a simple handler class, with no HTTP server or route definitions.
- SDK usage:
  - The current scaffold does not reference `@ameideio/ameide-sdk-ts`.
  - There is no TS-specific import policy in `primitive verify` to enforce:
    - SDK-only imports,
    - Avoiding `@ameide/core-proto` in runtime code.

### 6.3 Verify behavior for UISurface primitives

**Status:** Generic checks only; UISurface-specific rules are not implemented.

- `primitive verify --kind uisurface --name <name>`:
  - Runs generic repo checks (naming, security, tests, etc.).
  - Does **not** currently check:
    - Presence of `src/server.ts` or `src/routes/**`.
    - That `src/proto/index.ts` exists or uses SDK-based imports.
    - That runtime code avoids `@ameide/core-proto`.

### 6.4 Known gaps and next steps

- Scaffold:
  - Introduce a UISurface-specific scaffold path that:
    - Generates `src/server.ts`, `src/routes/<name>.ts`, and `src/proto/index.ts`.
    - Wires SDK clients via `@ameideio/ameide-sdk-ts` in `src/proto/index.ts`.
    - Uses UISurface templates for README and code comments (similar to Domain/Agent templates).
- Verify:
  - Add TS import checks for UISurface primitives to ensure:
    - No `@ameide/core-proto` imports in runtime code.
    - SDK-only outbound calls via `@ameideio/ameide-sdk-ts` or approved barrels.
  - Shape checks for presence of server/routes/proto files.
