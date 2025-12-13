# 481 – Service Catalog

**Status:** In Progress
**Priority:** High
**Complexity:** Large
**Created:** 2025-12-07
**Updated:** 2025-12-07

> **Cross-References (Vision Suite)**:
>
> | Document | Relationship |
> |----------|-------------|
> | [470-ameide-vision.md](470-ameide-vision.md) | §0 – Six primitives model |
> | [472-ameide-information-application.md](472-ameide-information-application.md) | §2 – Primitive definitions |
> | [473-ameide-technology.md](473-ameide-technology.md) | §2.3 – Backstage as factory for primitives |
> | [475-ameide-domains.md](475-ameide-domains.md) | §3 – Domain portfolio |
> | [467-backstage.md](467-backstage.md) | §10 – Software Templates for primitives |

## Grounding & contract alignment

- **Catalog view of primitives:** Applies the primitive definitions and information-flow model from `470-ameide-vision.md`, `471-ameide-business-architecture.md`, `472-ameide-information-application.md`, and `475-ameide-domains.md` to a Backstage/service‑catalog folder layout for Domain/Process/Agent primitives.  
- **Alignment with placement:** Mirrors the placement and GitOps conventions in `477-primitive-stack.md`, ensuring that Service Catalog structure, primitive code in `primitives/*`, and GitOps manifests remain coherent.  
- **Scrum & transformation anchoring:** Shows where the Transformation domain and its Agile/Scrum Process primitives appear in the catalog (e.g., `domains/transformation`, `processes/agile`), which the Scrum stack (`300-400/367-1-scrum-transformation.md`, `506-scrum-vertical-v2.md`, `508-scrum-protos.md`) and agent backlogs (`505-agent-developer-v2*.md`) can reference without redefining structure.

---

## 1. Executive Summary

This document defines the **Service Catalog** architecture: a flat structure for organizing Ameide primitives aligned with Backstage, Buf, and Temporal patterns.

**Key concepts:**
- **service_catalog/** – Single folder containing all primitive types
- **_primitive/** – Base definitions (contracts + base types) under each primitive type
- **{name}/** – Concrete implementations alongside `_primitive`

This structure enables Backstage to scaffold new primitives via Software Templates while keeping definitions and implementations co-located. The architectural rationale for the proto → implementation → runtime chain lives in [472-ameide-information-application.md §2.7](472-ameide-information-application.md); this document focuses on how that chain maps to source folders and GitOps repos. Remember that proto contracts live in `packages/ameide_core_proto/` (the service catalog merely references those modules), so the catalog is an implementation detail rather than a new architecture center of gravity.

---

## 2. Folder Structure

```
ameide/                                    # github.com/ameideio/ameide
├── service_catalog/
│   ├── domains/                        # Domain primitives
│   │   ├── _primitive/                 # Base contract + helpers
│   │   ├── transformation/             # Transformation Domain primitive
│   │   ├── platform/                   # Platform Domain primitive
│   │   ├── sales/                      # Sales Domain primitive
│   │   └── ...
│   ├── processes/                      # Process primitives
│   │   ├── _primitive/                 # Base contract + helpers
│   │   ├── togaf_adm/                  # TOGAF ADM Process primitive
│   │   ├── agile/                      # Agile/Scrum Process primitive
│   │   ├── l2o/                        # Lead-to-Opportunity
│   │   └── ...
│   └── agents/                         # Agent primitives
│       ├── _primitive/                 # Base contract + helpers
│       ├── architect/                  # Architect Agent primitive
│       ├── sales_clerk/                # Sales Clerk Agent primitive
│       └── ...
│
├── services/                           # LEGACY + non-primitive workloads
│   ├── www_ameide/                     # Next.js UI (NOT a primitive)
│   ├── www_ameide_platform/            # Platform UI (NOT a primitive)
│   └── ...
│
└── packages/                           # Shared SDKs (unchanged)
    └── ameide_core_proto/              # Existing proto structure stays as-is
```

---

## 3. Base Definitions (_primitive/)

**Purpose:** Contracts and thin base abstractions for each primitive type.

### 3.1 What goes here (future)

- Buf proto contracts (`ameide.primitives.domain.v1`, etc.)
- Base interfaces/helpers (`Domain primitiveBase`, `Process primitiveBase`, `Agent primitiveBase`)
- Common middleware (auth, tenant context, logging)

### 3.2 What does NOT go here

- Heavy business/runtime logic
- Domain-specific messages (Order, Invoice, etc.)
- Design-time artifacts (ProcessDefinitions, AgentDefinitions)

### 3.3 Future proto structure

```
service_catalog/
├── domains/_primitive/
│   └── v1/
│       └── primitive.proto        # GetPrimitiveInfo, HealthCheck, ListCapabilities
├── processes/_primitive/
│   └── v1/
│       └── primitive.proto        # StartProcess, AbortProcess, QueryProcess
└── agents/_primitive/
    └── v1/
        └── primitive.proto        # InvokeAgent, StreamAgentResponse
```

---

## 4. Primitive Implementations

**Purpose:** Concrete primitive implementations scaffolded via Backstage.

### 4.1 Primitive types

| Type | Responsibility | Runtime |
|------|---------------|---------|
| **Domain primitive** | Own entities, CRUD, business rules | gRPC service + Postgres |
| **Process primitive** | Execute ProcessDefinitions | Temporal workflows |
| **Agent primitive** | Execute AgentDefinitions | LangGraph/LLM runtime |

### 4.2 Future primitive folder structure

Each primitive folder will contain:

```
service_catalog/domains/{name}/
├── src/                            # Service source code
├── proto/                          # Domain-specific protos (optional)
├── Dockerfile.dev
├── Dockerfile.release
├── catalog-info.yaml               # Backstage Component registration
├── Chart.yaml                      # Helm chart (or reference to charts/)
└── README.md
```

---

## 5. Proto Strategy

### 5.1 Current state (unchanged)

- `packages/ameide_core_proto/` – Central platform protos
- `packages/ameide_sdk_*` – Generated SDKs (TS, Go, Python)

### 5.2 Open question: Tenant-specific protos

Primitive-specific protos may diverge per tenant. If tenants customize their Domain primitives (e.g., custom Sales fields), they get different API surfaces.

**Options to explore:**

| Option | Description | Trade-offs |
|--------|-------------|------------|
| All protos stay central | Primitives extend base types, no tenant variation | Simple but inflexible |
| Primitive protos in service_catalog/ | Each primitive owns its proto, SDK generated per primitive | More flexible but complex generation |
| Tenant-specific SDKs | BSR modules per tenant, tenant-specific SDK packages | Maximum flexibility but operational complexity |

This decision is deferred until we understand tenant customization requirements better.

---

## 6. Backstage Integration

### 6.1 Software Templates

Backstage templates scaffold into `service_catalog/`. Templates are implemented in `_primitive/` directories:

```
service_catalog/
├── domains/_primitive/
│   ├── template.yaml              # Backstage template definition
│   └── skeleton/                  # Scaffolding files
│       ├── catalog-info.yaml      # Component registration (Jinja)
│       ├── README.md              # Documentation (Jinja)
│       ├── Dockerfile.dev         # Dev container (Jinja)
│       ├── Dockerfile.release     # Production container (Jinja)
│       ├── src/.gitkeep
│       └── migrations/.gitkeep
├── processes/_primitive/
│   ├── template.yaml
│   └── skeleton/
│       ├── catalog-info.yaml
│       ├── README.md
│       ├── Dockerfile.dev
│       ├── Dockerfile.release
│       └── src/{workflows,activities}/.gitkeep
└── agents/_primitive/
    ├── template.yaml
    └── skeleton/
        ├── catalog-info.yaml
        ├── README.md
        ├── Dockerfile.dev
        ├── Dockerfile.release
        ├── pyproject.toml
        └── src/{tools,prompts}/.gitkeep
```

#### Template parameters

| Template | Key Parameters |
|----------|----------------|
| **Domain primitive** | `domain_id`, `owner`, `language` (ts/go), `database_plan` (shared/dedicated) |
| **Process primitive** | `process_id`, `owner`, `language` (ts/go), `temporal_task_queue`, `process_definition_ref` |
| **Agent primitive** | `agent_id`, `owner`, `default_model`, `agent_definition_ref` |

#### Template actions

All templates use the same action sequence:
1. `fetch:template` – Copy skeleton with Jinja substitution
2. `publish:github:pull-request` – Create PR to `ameideio/ameide`
3. `catalog:register` – Register Component in Backstage catalog

### 6.2 Catalog entities

Each primitive registers as a Backstage Component with appropriate type:

| Primitive Type | Backstage Component Type |
|----------------|--------------------------|
| Domain primitive | `domain-primitive` (future; currently `domain-controller`) |
| Process primitive | `process-primitive` (future; currently `process-controller`) |
| Agent primitive | `agent-primitive` (future; currently `agent-controller`) |

---

## 7. Relationship to Existing Concepts

| Concept | Location | Notes |
|---------|----------|-------|
| ProcessDefinitions | Transformation Domain primitive | Design-time artifacts (BPMN from React Flow) |
| AgentDefinitions | Transformation Domain primitive | Design-time artifacts (tools, policies) |
| UAF | UI layer | Calls Transformation APIs, no own storage |
| Graph | Read-only projection | Fed by primitive events |
| Domain-specific protos | TBD | Central vs per-primitive decision pending |

---

## 8. Relationship to services/

The existing `services/` folder contains legacy services that predate this architecture:

| Current Service | Future Location | Notes |
|-----------------|-----------------|-------|
| `services/platform/` | `service_catalog/domains/platform/` | Migrate when ready |
| `services/agents/` | `service_catalog/domains/agents/` | Agents control plane |
| `services/workflows_runtime/` | Process primitive infrastructure | Temporal worker |
| `services/www_ameide/` | Stays in `services/` | UI, not a primitive |
| `services/www_ameide_platform/` | Stays in `services/` | UI, not a primitive |

**Non-primitives stay in `services/`:**
- Next.js UIs
- Backstage itself
- Infrastructure helpers
- Utility services

---

## 9. Migration Plan

### Phase 1: Foundation (complete)
- [x] Create folder structure with `.gitkeep` files
- [x] Create this spec document
- [x] Implement Backstage templates for all primitive types:
  - [x] Domain primitive template (`service_catalog/domains/_primitive/`)
  - [x] Process primitive template (`service_catalog/processes/_primitive/`)
  - [x] Agent primitive template (`service_catalog/agents/_primitive/`)

### Phase 2: Base contracts
- [ ] Define base proto contracts in `service_catalog/{domains,processes,agents}/_primitive/`
- [ ] Update Buf workspace
- [ ] Generate base SDK types

### Phase 3: First primitive
- [ ] Implement Transformation Domain primitive in `service_catalog/domains/transformation/`
- [ ] Create Backstage template for Domain primitive
- [ ] Validate GitOps deployment pattern

### Phase 4: Migrate existing
- [ ] Migrate `services/platform/` → `service_catalog/domains/platform/`
- [ ] Migrate `services/agents/` → `service_catalog/domains/agents/`
- [ ] Migrate `services/workflows_runtime/` → Process primitive infrastructure

### Phase 5: Expand
- [ ] Create Process primitive for L2O/O2C
- [ ] Create Agent primitive templates
- [ ] Deprecate legacy `services/` paths

---

## 10. Open Questions

1. **Tenant-specific protos/SDKs**: If tenants customize primitives, do they need tenant-specific SDKs? How does this interact with BSR?

2. **Proto location**: Should primitive-specific protos live in `service_catalog/` or stay centralized in `packages/ameide_core_proto/`?

3. **SDK generation**: Per-primitive SDK generation vs monolithic SDK with all primitives?

4. **Backstage template versioning**: How do we version primitive templates as the base contracts evolve?

---

## 11. Non-Goals

- Not redefining UAF/Transformation/Graph concepts (see [472](472-ameide-information-application.md))
- Not moving design-time artifacts out of Transformation Domain primitive
- Not killing `services/` for non-primitive workloads (UIs, Backstage, infra)
- Not implementing primitives in this phase (scaffolding only)

---

## 12. Decision Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2025-12-07 | Flat structure: `service_catalog/{domains,processes,agents}/` with `_primitive/` for base definitions | Simpler hierarchy; co-locates definitions with implementations |
| 2025-12-07 | Keep existing `packages/ameide_core_proto/` unchanged | Minimize disruption; proto strategy TBD |
| 2025-12-07 | Non-primitives stay in `services/` | UIs and infrastructure don't fit primitive model |
| 2025-12-07 | Backstage templates implemented | Domain primitive, Process primitive, Agent primitive templates with skeleton/ directories |
| 2025-12-07 | Agent primitive is Python-only | LangGraph requirement; Domain primitive/Process primitive support TS or Go |
