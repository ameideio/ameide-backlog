# 478 – Service Catalog

**Status:** In Progress
**Priority:** High
**Complexity:** Large
**Created:** 2025-12-07
**Updated:** 2025-12-07

> **Cross-References (Vision Suite)**:
>
> | Document | Relationship |
> |----------|-------------|
> | [470-ameide-vision.md](470-ameide-vision.md) | §5 – Controller model vision |
> | [472-ameide-information-application.md](472-ameide-information-application.md) | §2 – DomainController, ProcessController, AgentController definitions |
> | [473-ameide-technology.md](473-ameide-technology.md) | §2.3 – Backstage as factory for controllers |
> | [475-ameide-domains.md](475-ameide-domains.md) | §3 – Domain portfolio |
> | [477-backstage.md](477-backstage.md) | §10 – Software Templates for controllers |

---

## 1. Executive Summary

This document defines the **Service Catalog** architecture: a flat structure for organizing Ameide controllers aligned with Backstage, Buf, and Temporal patterns.

**Key concepts:**
- **service_catalog/** – Single folder containing all controller types
- **_controller/** – Base definitions (contracts + base types) under each controller type
- **{name}/** – Concrete implementations alongside _controller

This structure enables Backstage to scaffold new controllers via Software Templates while keeping definitions and implementations co-located.

---

## 2. Folder Structure

```
ameide-core/
├── service_catalog/
│   ├── domains/                        # DomainControllers
│   │   ├── _controller/                # Base contract + helpers
│   │   ├── transformation/             # Transformation DomainController
│   │   ├── platform/                   # Platform DomainController
│   │   ├── sales/                      # Sales DomainController
│   │   └── ...
│   ├── processes/                      # ProcessControllers
│   │   ├── _controller/                # Base contract + helpers
│   │   ├── togaf_adm/                  # TOGAF ADM ProcessController
│   │   ├── agile/                      # Agile/Scrum ProcessController
│   │   ├── l2o/                        # Lead-to-Opportunity
│   │   └── ...
│   └── agents/                         # AgentControllers
│       ├── _controller/                # Base contract + helpers
│       ├── architect/                  # Architect AgentController
│       ├── sales_clerk/                # Sales Clerk AgentController
│       └── ...
│
├── services/                           # LEGACY + non-controller workloads
│   ├── www_ameide/                     # Next.js UI (NOT a controller)
│   ├── www_ameide_platform/            # Platform UI (NOT a controller)
│   └── ...
│
└── packages/                           # Shared SDKs (unchanged)
    └── ameide_core_proto/              # Existing proto structure stays as-is
```

---

## 3. Base Definitions (_controller/)

**Purpose:** Contracts and thin base abstractions for each controller type.

### 3.1 What goes here (future)

- Buf proto contracts (`ameide.controllers.domain.v1`, etc.)
- Base interfaces/helpers (`DomainControllerBase`, `ProcessControllerBase`, `AgentControllerBase`)
- Common middleware (auth, tenant context, logging)

### 3.2 What does NOT go here

- Heavy business/runtime logic
- Domain-specific messages (Order, Invoice, etc.)
- Design-time artifacts (ProcessDefinitions, AgentDefinitions)

### 3.3 Future proto structure

```
service_catalog/
├── domains/_controller/
│   └── v1/
│       └── controller.proto        # GetControllerInfo, HealthCheck, ListCapabilities
├── processes/_controller/
│   └── v1/
│       └── controller.proto        # StartProcess, AbortProcess, QueryProcess
└── agents/_controller/
    └── v1/
        └── controller.proto        # InvokeAgent, StreamAgentResponse
```

---

## 4. Controller Implementations

**Purpose:** Concrete controller implementations scaffolded via Backstage.

### 4.1 Controller types

| Type | Responsibility | Runtime |
|------|---------------|---------|
| **DomainController** | Own entities, CRUD, business rules | gRPC service + Postgres |
| **ProcessController** | Execute ProcessDefinitions | Temporal workflows |
| **AgentController** | Execute AgentDefinitions | LangGraph/LLM runtime |

### 4.2 Future controller folder structure

Each controller folder will contain:

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

Controller-specific protos may diverge per tenant. If tenants customize their DomainControllers (e.g., custom Sales fields), they get different API surfaces.

**Options to explore:**

| Option | Description | Trade-offs |
|--------|-------------|------------|
| All protos stay central | Controllers extend base types, no tenant variation | Simple but inflexible |
| Controller protos in service_catalog/ | Each controller owns its proto, SDK generated per-controller | More flexible but complex generation |
| Tenant-specific SDKs | BSR modules per tenant, tenant-specific SDK packages | Maximum flexibility but operational complexity |

This decision is deferred until we understand tenant customization requirements better.

---

## 6. Backstage Integration

### 6.1 Software Templates

Backstage templates scaffold into `service_catalog/`. Templates are implemented in `_controller/` directories:

```
service_catalog/
├── domains/_controller/
│   ├── template.yaml              # Backstage template definition
│   └── skeleton/                  # Scaffolding files
│       ├── catalog-info.yaml      # Component registration (Jinja)
│       ├── README.md              # Documentation (Jinja)
│       ├── Dockerfile.dev         # Dev container (Jinja)
│       ├── Dockerfile.release     # Production container (Jinja)
│       ├── src/.gitkeep
│       └── migrations/.gitkeep
├── processes/_controller/
│   ├── template.yaml
│   └── skeleton/
│       ├── catalog-info.yaml
│       ├── README.md
│       ├── Dockerfile.dev
│       ├── Dockerfile.release
│       └── src/{workflows,activities}/.gitkeep
└── agents/_controller/
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
| **DomainController** | `domain_id`, `owner`, `language` (ts/go), `database_plan` (shared/dedicated) |
| **ProcessController** | `process_id`, `owner`, `language` (ts/go), `temporal_task_queue`, `process_definition_ref` |
| **AgentController** | `agent_id`, `owner`, `default_model`, `agent_definition_ref` |

#### Template actions

All templates use the same action sequence:
1. `fetch:template` – Copy skeleton with Jinja substitution
2. `publish:github:pull-request` – Create PR to `ameide-core`
3. `catalog:register` – Register Component in Backstage catalog

### 6.2 Catalog entities

Each controller registers as a Backstage Component with appropriate type:

| Controller Type | Backstage Component Type |
|-----------------|--------------------------|
| DomainController | `domain-controller` |
| ProcessController | `process-controller` |
| AgentController | `agent-controller` |

---

## 7. Relationship to Existing Concepts

| Concept | Location | Notes |
|---------|----------|-------|
| ProcessDefinitions | Transformation DomainController | Design-time artifacts (BPMN from React Flow) |
| AgentDefinitions | Transformation DomainController | Design-time artifacts (tools, policies) |
| UAF | UI layer | Calls Transformation APIs, no own storage |
| Graph | Read-only projection | Fed by controller events |
| Domain-specific protos | TBD | Central vs per-controller decision pending |

---

## 8. Relationship to services/

The existing `services/` folder contains legacy services that predate this architecture:

| Current Service | Future Location | Notes |
|-----------------|-----------------|-------|
| `services/platform/` | `service_catalog/domains/platform/` | Migrate when ready |
| `services/agents/` | `service_catalog/domains/agents/` | Agents control plane |
| `services/workflows_runtime/` | ProcessController infrastructure | Temporal worker |
| `services/www_ameide/` | Stays in `services/` | UI, not a controller |
| `services/www_ameide_platform/` | Stays in `services/` | UI, not a controller |

**Non-controllers stay in `services/`:**
- Next.js UIs
- Backstage itself
- Infrastructure helpers
- Utility services

---

## 9. Migration Plan

### Phase 1: Foundation (complete)
- [x] Create folder structure with `.gitkeep` files
- [x] Create this spec document
- [x] Implement Backstage templates for all controller types:
  - [x] DomainController template (`service_catalog/domains/_controller/`)
  - [x] ProcessController template (`service_catalog/processes/_controller/`)
  - [x] AgentController template (`service_catalog/agents/_controller/`)

### Phase 2: Base contracts
- [ ] Define base proto contracts in `service_catalog/{domains,processes,agents}/_controller/`
- [ ] Update Buf workspace
- [ ] Generate base SDK types

### Phase 3: First controller
- [ ] Implement Transformation DomainController in `service_catalog/domains/transformation/`
- [ ] Create Backstage template for DomainController
- [ ] Validate GitOps deployment pattern

### Phase 4: Migrate existing
- [ ] Migrate `services/platform/` → `service_catalog/domains/platform/`
- [ ] Migrate `services/agents/` → `service_catalog/domains/agents/`
- [ ] Migrate `services/workflows_runtime/` → ProcessController infrastructure

### Phase 5: Expand
- [ ] Create ProcessController for L2O/O2C
- [ ] Create AgentController templates
- [ ] Deprecate legacy `services/` paths

---

## 10. Open Questions

1. **Tenant-specific protos/SDKs**: If tenants customize controllers, do they need tenant-specific SDKs? How does this interact with BSR?

2. **Proto location**: Should controller-specific protos live in `service_catalog/` or stay centralized in `packages/ameide_core_proto/`?

3. **SDK generation**: Per-controller SDK generation vs monolithic SDK with all controllers?

4. **Backstage template versioning**: How do we version controller templates as the base contracts evolve?

---

## 11. Non-Goals

- Not redefining UAF/Transformation/Graph concepts (see [472](472-ameide-information-application.md))
- Not moving design-time artifacts out of Transformation DomainController
- Not killing `services/` for non-controller workloads (UIs, Backstage, infra)
- Not implementing controllers in this phase (scaffolding only)

---

## 12. Decision Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2025-12-07 | Flat structure: `service_catalog/{domains,processes,agents}/` with `_controller/` for base definitions | Simpler hierarchy; co-locates definitions with implementations |
| 2025-12-07 | Keep existing `packages/ameide_core_proto/` unchanged | Minimize disruption; proto strategy TBD |
| 2025-12-07 | Non-controllers stay in `services/` | UIs and infrastructure don't fit controller model |
| 2025-12-07 | Backstage templates implemented | DomainController, ProcessController, AgentController templates with skeleton/ directories |
| 2025-12-07 | AgentController is Python-only | LangGraph requirement; DomainController/ProcessController support TS or Go |
