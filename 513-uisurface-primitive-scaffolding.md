# 513 – UISurface Primitive Scaffolding (TypeScript, opinionated)

This backlog defines the **canonical target scaffold** for **UISurface** primitives.

- **Audience:** UI engineers, AI agents, CLI implementers
- **Scope:** One opinionated TypeScript-first web app pattern (Next.js or equivalent), aligned with `backlog/520-primitives-stack-v2.md` and `services/www_ameide_platform/backlog/SDK_USAGE.md` (SDK-only calls, projection-driven reads). The CLI orchestrates repo/GitOps wiring; `buf generate` + plugins handle deterministic generation (SDKs, generated-only glue).

> **Update (2026-01, 670): CI-owned GitOps wiring**
>
> - GitOps wiring is authored in `ameide-gitops` via the CI-owned scaffolding workflow (workflow → PR → merge). See `backlog/670-gitops-authoritative-write-path-for-scaffolding.md`.
> - UISurface scaffolding in the core repo creates runtime code + tests; it does not directly write GitOps files as the canonical path.

---

## Primitive/operator alignment

- **Operator responsibilities (495, 501, 497):** The UISurface operator reconciles `UISurface` CRDs into Deployments, Services, HTTPRoutes (Gateway API), Keycloak clients, ConfigMaps, HPAs, and monitoring resources. It is responsible for how traffic reaches a UISurface and how auth/observability are wired, but it does not implement UI routes or call backend SDKs.  
- **Primitive responsibilities (this backlog, SDK_USAGE):** The UISurface scaffold owns the **application layer**: HTTP server/Connect endpoints, route handlers, and SDK-based calls into Domain/Process/Agent primitives. It lives inside the image referenced by the UISurface CRD and encapsulates UI behavior and composition, independent of operator internals.  
- **Boundary:** Operators are the **control plane for hosting and routing**; UISurface primitives are the **frontend applications** that use SDKs to talk to backends. 513’s scaffold shape is designed so the operator only needs an image and a few config fields (host/path/auth), while the primitive remains fully in charge of UX and SDK wiring, consistent with `495-ameide-operators.md`, `497-operator-implementation-patterns.md`, and `501-uisurface-operator.md`.

UISurface progress views (Kanban/timelines):

- UISurfaces MUST render “business progress” (e.g., Kanban boards, timelines) from **Projection/query APIs** built from domain facts and process progress facts, not from infra internals (Temporal visibility, KEDA/Job status, broker offsets).
- Temporal visibility/Search Attributes may be linked as an engineering debug surface, but are not a product UI data source.

---

## Scope exclusion – client applications

> **Not UISurface primitives:** IDE extensions (VSCode, JetBrains), browser extensions, CLI tools, and mobile apps are **Application Components (clients)**, not UISurface primitives. They consume primitive services but are not deployed via operators, do not use `ameide primitive scaffold`, and do not live under `primitives/`. See `backlog/538-vscode-client-transformation.md` for the VSCode client pattern.

**Key distinction:**
| Aspect | UISurface Primitive | Client Application |
|--------|--------------------|--------------------|
| **Location** | `primitives/uisurface/{name}/` | `packages/{name}/` |
| **Deployment** | Operator-managed (CRD/GitOps) | Marketplace/manual install |
| **Scaffold** | `ameide primitive scaffold --kind uisurface` | Manual or extension tooling |
| **Example** | Platform dashboard, workspace UI | VSCode extension (538), CLI |

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
- **Capability implementation DAG:** `backlog/533-capability-implementation-playbook.md` (UISurface scaffold + implementation nodes).
- **Contrast – client applications:** `538-vscode-client-transformation.md`, `538-vscode-client-extension.md` (VSCode extension is NOT a UISurface primitive; see "Scope exclusion" above).

---

## 1. Canonical scaffold command (UISurface / TypeScript)

UISurface primitives are deployed web applications. The target scaffold is a TypeScript-first web app (Next.js or equivalent) with SDK-only backend calls and projection-driven reads (Kanban/timelines).

```bash
ameide primitive scaffold \
  --kind uisurface \
  --name <name> \
  --include-test-harness
```

This creates a UISurface primitive under:

```text
primitives/uisurface/<name>/
```

---

## 2. Generated structure (UISurface / TypeScript)

The target UISurface scaffold is a TypeScript-first web app/BFF that:

- serves the UI (Next.js or equivalent) and optionally a Connect/BFF layer,
- uses Ameide SDK clients for all calls (SDK-only rule),
- renders Kanban/timeline views from Projection query APIs (domain facts + process progress facts),
- keeps generated-only artifacts under generated roots (never mixed with implementation-owned UI code).

```text
primitives/uisurface/<name>/
├── README.md
├── catalog-info.yaml
├── package.json
├── tsconfig.json
├── next.config.js
├── Dockerfile
├── src/
│   ├── app/                             # Next.js app router (pages/canvases)
│   ├── components/                      # Widgets + design system integration
│   ├── proto/                           # SDK-only client wiring (no @ameide/core-proto)
│   │   └── index.ts
│   └── server/                          # Optional BFF/Connect routes
│       ├── server.ts
│       └── routes/
├── tests/
│   └── *.test.ts
└── internal/gen/                         # Generated-only artifacts (if any)
```

GitOps wiring (canonical, 670):

```text
environments/_shared/components/apps/primitives/uisurface-<name>-<version>/component.yaml
sources/values/_shared/apps/uisurface-<name>-<version>.yaml
environments/_shared/components/apps/primitives/uisurface-<name>-<version>-smoke/component.yaml   # optional
sources/values/_shared/apps/uisurface-<name>-<version>-smoke.yaml                                 # optional
```

---

## 3. Verification expectations

Phase 0 of `ameide test` is expected to check:

- Presence of `src/server/server.ts` and `src/server/routes/**`.  
- `src/proto/index.ts` imports from SDK (`@ameideio/ameide-sdk-ts`), not `@ameide/core-proto`, in line with `SDK_USAGE.md`.  
- Tests exist under `tests/` and initially fail.  
- GitOps manifests are valid when present.

Behavioral details (what routes exist, what UX looks like) remain in vertical slices and UX backlogs; this document defines only the **scaffold shape and SDK usage pattern** for UISurface primitives.
