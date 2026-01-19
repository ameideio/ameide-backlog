> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# 304: Systematic Context & Knowledge System for AI Agent

> **Status (2026-01): HISTORICAL / SUPERSEDED**
>
> This document predates the current “organizational memory” contract and uses legacy terminology (notably `graph`/`graph_id`, and “folders as elements”).
>
> **Use instead (authoritative):**
> - Canonical storage + scoping: `backlog/300-400/303-elements.md` (repository scope is `{tenant_id, organization_id, repository_id}`; workspace tree is separate from elements)
> - Agentic organizational memory contract: `backlog/656-agentic-memory-v6.md`
> - Memory implementation increments: `backlog/656-agentic-memory-implementation-mvp-v6.md`
> - Retrieval + citation discipline: `backlog/527-transformation-proto.md` (read_context + citations), `backlog/535-mcp-read-optimizations.md` (projection-owned retrieval), `backlog/534-mcp-protocol-adapter.md` (MCP surfaces)
>
> **How to read legacy terms here:**
> - “graph” ⇒ `repository_id`
> - “knowledge base / memory” ⇒ governed element graph + baselines (not a vector store)
> - “folders are elements” ⇒ **not canonical** in 303; repository navigation is the workspace tree

## Overview

Design and implement a comprehensive context and knowledge system that:
1. Systematically provides rich context from UI to agent (tenant, graph, element, diagram)
2. Enables agents to access database/graph projections through LangGraph tools
3. Allows agents to query and hydrate knowledge from the backend
4. Captures active user identity and capabilities alongside UI context
5. **Tailors prompts, agent choice, and tool access based on user role and permissions**

## Key Design Principles

### Context Adapts to UI Location

**Different pages send different context** - the system is designed to handle varying levels of context from the start:

- **Home page**: `{ tenant, user }` - General architecture threads
- **Repository browser**: `{ tenant, user, graph }` - Repository-aware threads with query tools
- **Diagram viewer**: `{ tenant, user, graph, element }` - Element-focused context (element = diagram being viewed)
- **Diagram editor**: `{ tenant, user, graph, element, selection }` - Full context with selected element on canvas

The context schema is **flexible by design**, but implementation is **incremental** (start with tenant + user + graph, expand to element + selection as needed).

**Aligned with 303-elements.md unified element model:**
- Everything is an element (diagrams, documents, folders)
- No separate "artifacts" - unified into elements table
- Element kinds: `ARCHIMATE_VIEW`, `BPMN_VIEW`, `DOCUMENT_MD`, `FOLDER`, etc.

### Context Flow by Page

```
┌─────────────────┬──────────────────────┬─────────────────────┬──────────────────────┐
│   Home Page     │  Repository Browser  │   Diagram Viewer    │   Diagram Editor     │
│                 │                      │                     │   (with selection)   │
├─────────────────┼──────────────────────┼─────────────────────┼──────────────────────┤
│ useChatContext()│  useChatContext()    │  useChatContext()   │useDiagramEditor      │
│                 │                      │                     │    Context()         │
├─────────────────┼──────────────────────┼─────────────────────┼──────────────────────┤
│ Context Sent:   │  Context Sent:       │  Context Sent:      │  Context Sent:       │
│ • tenant        │  • tenant            │  • tenant           │  • tenant            │
│ • user          │  • user              │  • user             │  • user              │
│                 │  • graph        │  • graph       │  • graph        │
│                 │                      │  • element          │  • element           │
│                 │                      │    (VIEW type)      │    (VIEW type)       │
│                 │                      │                     │  • selection         │
│                 │                      │                     │  • ui.page           │
├─────────────────┼──────────────────────┼─────────────────────┼──────────────────────┤
│ Enriched:       │  Enriched:           │  Enriched:          │  Enriched:           │
│ • tenant name   │  • tenant name       │  • tenant name      │  • tenant name       │
│ • user display  │  • user display      │  • user display     │  • user display      │
│ • user role     │  • user role         │  • user role        │  • user role         │
│                 │  • repo name         │  • repo name        │  • repo name         │
│                 │  • element count     │  • element count    │  • element count     │
│                 │                      │  • element.title    │  • element.title     │
│                 │                      │  • element.kind     │  • element.kind      │
│                 │                      │                     │  • selection.title   │
│                 │                      │                     │  • selection.type    │
├─────────────────┼──────────────────────┼─────────────────────┼──────────────────────┤
│ Agent Mode:     │  Agent Mode:         │  Agent Mode:        │  Agent Mode:         │
│ General threads    │  Repository query    │  View-aware query   │  Editor-focused      │
│ Base agent      │  Repo analyst agent  │  View analyst agent │  Editor agent        │
│ No tools        │  + 4 query tools     │  + 4 query tools    │  + 4 query tools     │
│                 │  + repo agent prompt │  + view agent prompt│  + editor agent prompt│
│                 │                      │  + view context     │  + view context      │
│                 │                      │                     │  + selection context │
└─────────────────┴──────────────────────┴─────────────────────┴──────────────────────┘
```

**Key terminology (from 303-elements.md):**
- ❌ "Artifact" - Removed (unified into elements)
- ✅ "Element" - Universal atomic unit (diagrams, documents, folders, nodes, etc.)
- ✅ "VIEW" elements - Diagrams (`ARCHIMATE_VIEW`, `BPMN_VIEW`, `C4_VIEW`)
- ✅ "Selection" - Element selected within a VIEW (e.g., clicking a box on a diagram)

## Architectural Principles

**Critical Design Constraints:**

### 1. **Stateless Inference with Domain Abstractions**
Route all knowledge queries through graph service abstractions (`ameide_core_domain/repositories`) instead of direct SQL in agents. Keep the inference layer stateless and focused on orchestration.

**Why:**
- Separation of concerns: inference orchestrates, domain layer owns data access
- Testability: mock graph interfaces vs mocking database
- Maintainability: schema changes isolated to domain layer
- Consistency: business logic stays in one place

**Reference:** `packages/ameide_core_domain/src/ameide_ameide_core_domain/repositories.py:225`

### 2. **Context Enrichment in Chat Service**
Store structured context (tenant, graph, user metadata) with the threads thread in the threads service. Inference reads enriched thread context instead of rehydrating per request.

**Why:**
- Performance: context hydrated once per thread vs per message
- Consistency: all messages in thread share the same tenant/graph/user data
- Simplicity: inference doesn't need to understand context assembly
- Personalization: stored user attributes enable role-aware prompts without extra lookups
- Auditability: context (including user identity) is part of thread history

**Reference:** `services/threads/src/threads/service.ts:613`

### 3. **Incremental Implementation**
Start with lightweight typed context envelope (tenant + user + graph IDs only). Validate end-to-end flow. Expand to element + selection context once UI data sources are available.

**Why:**
- Faster iteration: simpler = faster to implement and test
- Risk reduction: prove architecture before adding complexity
- Clear milestones: each increment adds measurable value
- Easier debugging: fewer moving parts

**References:**
- `services/www_ameide_platform/features/threads/hooks/useChatContext.ts:49`
- `services/inference/main.py:492`

## Current State

### ✅ What We Have (Updated 2025-10-25)

**Context Collection & Metadata:**
- ✅ Flexible URL-based context extraction (`useChatContextMetadata`)
  - Supports both `/repo/` and `/graph/` URL patterns
  - Supports both `/transformation/` and `/transformations/` URL patterns
  - Detects settings pages (`/settings`)
  - Extracts tenant, graph, transformation, element IDs from URLs
- ✅ Enhanced context envelope with:
  - `tenant` - Organization context
  - `user` - User identity, role, permissions
  - `graph` - Repository context (from `/repo/` or `/graph/`)
  - `transformation` - Initiative context (from `/transformations/`)
  - `element` - Element being viewed/edited
  - `selection` - Selected element within diagrams
  - `settings` - Settings page context
  - `ui` - Page type (`'home' | 'graph' | 'editor' | 'diagram-editor' | 'settings'`)
- ✅ Metadata propagation includes:
  - `context.version`, `context.envelope` (full JSON)
  - `context.tenantId`, `context.graphId`, `context.transformationId`
  - `context.elementId`, `context.elementKind`, `context.selectionId`
  - `context.isSettings`, `context.uiPage`, `context.userRole`

**Client-Side Context Enrichment (NEW - Completed 2025-10-25):**
- ✅ `contextCacheStore` - Zustand store for caching enriched context
  - Stores: Repository, Initiative, Organization context
  - Populated automatically when pages load data
  - Cleared automatically when navigating away
- ✅ Context cache integration:
  - Organization layout populates organization context
  - Repository page populates graph context (name, type, elementCount, updatedAt)
  - Initiative page populates transformation context (title, status, owner, targetDate)
- ✅ `useChatContextMetadata` reads from cache stores:
  - Enriches tenant with organization name and type
  - Enriches graph with name, type, description, elementCount, lastModified
  - Enriches transformation with title, description, status, owner, targetDate
- ✅ Benefits achieved:
  - No duplicate database queries (reuses UI-loaded data)
  - Always fresh (auto-updates when pages load)
  - Type-safe enrichment with full TypeScript support
- ✅ Test coverage:
  - 10 unit tests for `contextCacheStore` (100% coverage)
  - 9 unit tests for `useChatContext` with enrichment scenarios
  - E2E test created for end-to-end context enrichment flow

**UI Integration:**
- ✅ Global `ChatProvider` wraps all pages
- ✅ `ChatFooter` available globally for all pages
- ✅ `ChatPanel` integrated into:
  - Settings pages (organization, graph, transformation)
  - Element editors (via `WorkspaceFrame`)
- ✅ Shared `SettingsLayout` component with threads panel support
- ✅ Responsive sidebar that hides when threads is active

**Infrastructure:**
- ✅ LangGraph agent infrastructure (`ameide_core_inference_agents`)
- ✅ Storage abstractions (`core_storage` with graph, sql, vector stores)
- ✅ Domain layer with graph abstractions (`ameide_core_domain/repositories`)
- ✅ Chat service with thread management (`services/threads`)
- ✅ Example tools in SimpleChatGraph (weather, calculator)

### 🔧 Critical Gap Fixes (2025-10-25)

After initial context enrichment implementation, several critical gaps were identified and addressed:

#### ✅ Fixed
1. **E2E Test Metadata Mismatch** - Test expected metadata in HTTP headers with version '1.0', but client sends metadata in request body with version '2024-12-01'
   - Fixed: [context-enrichment.spec.ts](services/www_ameide_platform/features/threads/__tests__/e2e/context-enrichment.spec.ts)
   - Now correctly validates `requestBody.metadata` instead of `headers`

2. **Enriched Names Ignored in Prompts** - Inference service `build_prompt()` was showing "Repository: primary-architecture" instead of using enriched names
   - Fixed: [context.py](services/inference/context.py:16-165,187-238)
   - Updated `ParsedContext` to include: `tenant_name`, `graph_name`, `graph_type`, `graph_description`, `element_count`, `transformation_title`, etc.
   - Prompts now show: "Repository: Primary Architecture (archimate) - 156 elements"

3. **Repository Service HTTP REST API** - Inference tools called REST endpoints that didn't exist (404 errors)
   - Fixed: Created dual-protocol graph service
     - HTTP/1.1 REST on port **8080** → [http-api.ts](services/graph/src/http-api.ts)
     - HTTP/2 gRPC on port **8081** (existing)
   - REST endpoints: `/api/repositories/:id`, `/api/repositories/:id/elements`, `/api/repositories/:id/elements/:elementId`, `/api/repositories/:id/elements/:elementId/neighborhood`
   - Updated Kubernetes service to expose both ports
   - Updated inference to call `http://graph:8080`

4. **Diagram Selection Propagation** - Diagram editor now syncs `selectedElement` and `selectionKind` query params and threads metadata falls back to legacy `selected` format
   - Fixed: [EditorUrlSync.tsx](services/www_ameide_platform/features/editor/navigation/EditorUrlSync.tsx), [useChatContext.ts](services/www_ameide_platform/features/threads/hooks/useChatContext.ts)
   - `context.selectionId` and `context.selectionKind` flow to inference, enabling editor-aware agents

5. **Agent Telemetry & Tool Allowlisting** - Metrics now tag requests with agent, role, and page, and tool allowlists intersect user role + agent policy
   - Fixed: [threads-metrics.ts](services/www_ameide_platform/lib/threads-metrics.ts), [threads/stream route](services/www_ameide_platform/app/api/threads/stream/route.ts), [context.py](services/inference/context.py:197-258)
   - Ensures unauthorized agents cannot invoke graph tools while observability dashboards correlate behaviour

6. **Thread Pre-Allocation** - Threads are now persisted before the first message to eliminate zero UUID fallbacks and survive refreshes
   - Fixed: `/api/threads/threads` endpoint (services/www_ameide_platform/app/api/threads/threads/route.ts) ensures threads via ChatService with empty append
   - ChatProvider pre-allocates threads on mount (services/www_ameide_platform/features/threads/providers/ChatProvider.tsx)
   - ChatService append flow allows zero messages for idempotent thread creation (services/threads/src/threads/service.ts)

7. **Role-Aware System Prompts** - Inference prompts now append persona-specific guidance for architects, developers, business analysts, stakeholders, and admins
   - Fixed: [context.py](services/inference/context.py:40-208)
   - Ensures assistants mirror the user's role expectations before adding contextual bullets

#### ⏳ Remaining
1. **End-to-end context verification** - Need automated check that enriched envelope reaches inference in E2E environment
2. **Agent specialization rollout** - Define additional specialized agents (settings, transformations) once telemetry confirms usage patterns

### ❌ What's Still Missing
- **LangGraph tools fully integrated** (REST API exists, but inference doesn't use it yet)
- **Agent registry & routing** (map context → specialized LangGraph agent)
- **Role-based prompt evaluation** (agent must prove it tailors responses to distinct personas)

## Architecture Design

### 1. Flexible Context Envelope

**Design for expansion, implement incrementally**: Context schema supports all UI locations, but pages send only what they know.

```typescript
interface ChatContext {
  // Always present (from session/auth)
  tenant: {
    id: string;
  };
  user: {
    id: string;
    displayName?: string;
    role?: UserRole;
    permissions?: string[];
  };

  // Repository context (graph browser, element editor)
  graph?: {
    id: string;
  };

  // Element context (viewing/editing a specific element)
  // This represents the PRIMARY element being viewed/edited:
  // - ARCHIMATE_VIEW / BPMN_VIEW / C4_VIEW = diagram being edited
  // - DOCUMENT_MD / DOCUMENT_PDF = document being viewed
  // - FOLDER = folder being browsed
  // - ARCHIMATE_ELEMENT / BPMN_ELEMENT = standalone element being edited
  element?: {
    id: string;
    kind?: ElementKind;  // ARCHIMATE_VIEW, DOCUMENT_MD, BPMN_VIEW, FOLDER, etc.
  };

  // Selection context (element selected WITHIN a diagram)
  // When editing a diagram, this is the element selected on the canvas
  // Example: editing "Application Architecture" diagram (element), user clicks "Customer Portal" box (selection)
  selection?: {
    elementId: string;  // ID of selected element within the diagram/container
  };

  // UI state (any page)
  ui?: {
    page: 'home' | 'graph' | 'editor' | 'diagram-editor';
    path: string;
  };
}

type ElementKind =
  // Documents
  | 'DOCUMENT_MD' | 'DOCUMENT_PDF' | 'DOCUMENT_DOCX'
  // ArchiMate
  | 'ARCHIMATE_VIEW' | 'ARCHIMATE_ELEMENT'
  // BPMN
  | 'BPMN_VIEW' | 'BPMN_ELEMENT'
  // C4
  | 'C4_VIEW' | 'C4_ELEMENT'
  // UML
  | 'UML_VIEW' | 'UML_ELEMENT'
  // Organizational
  | 'FOLDER';

type UserRole = 'architect' | 'developer' | 'business_analyst' | 'stakeholder' | 'admin';
```

**Alignment with 303-elements.md:**
- ✅ Unified element model (no separate artifacts)
- ✅ ElementKind enum matches new schema (`ARCHIMATE_VIEW`, `DOCUMENT_MD`, etc.)
- ✅ Diagrams are elements with `ARCHIMATE_VIEW` / `BPMN_VIEW` / etc.
- ✅ Documents are elements with `DOCUMENT_MD` / `DOCUMENT_PDF` / etc.
- ✅ Separation between primary element (being edited) and selection (within that element)

**Context by UI Location:**

| Page | Context Provided |
|------|------------------|
| Home page | `{ tenant, user }` |
| Repository browser | `{ tenant, user, graph }` |
| Diagram viewer | `{ tenant, user, graph, element: { kind: 'ARCHIMATE_VIEW' } }` |
| Diagram editor (with selection) | `{ tenant, user, graph, element: { kind: 'ARCHIMATE_VIEW' }, selection: { elementId } }` |
| Document viewer | `{ tenant, user, graph, element: { kind: 'DOCUMENT_MD' } }` |
| Folder browser | `{ tenant, user, graph, element: { kind: 'FOLDER' } }` |

**Implementation approach:**
- Define full schema upfront (extensible design)
- Implement Phase 1 with `tenant`, `user`, and `graph`
- Other pages progressively adopt as needed (element, selection, ui metadata)

### 2. Client-Side Context Enrichment (Using Zustand State)

**Use data already loaded in Zustand stores** instead of duplicate server queries.

**Key Insight:** When a user navigates to a graph/transformation page, the UI already fetches and caches this data in Zustand stores for rendering. We should reuse this data for context enrichment instead of making duplicate server queries.

```typescript
// Client-side enrichment using existing Zustand stores
export function useChatContextMetadata(): ChatContextMetadata {
  const pathname = usePathname();
  const { data: session } = useSession();

  // Read enriched data from Zustand stores (already loaded for UI)
  const graph = useRepositoryStore(state => state.currentRepository);
  const transformation = useInitiativeStore(state => state.currentInitiative);
  const organization = useOrganizationStore(state => state.organization);

  const envelope: ChatContextEnvelope = {
    tenant: {
      id: organization?.id ?? 'default',
      name: organization?.name,              // ✅ From Zustand
      type: organization?.type,
    },
    graph: graph ? {
      id: graph.id,
      name: graph.name,                 // ✅ From Zustand
      type: graph.type,
      elementCount: graph.stats?.elementCount,
      lastModified: graph.updatedAt,
    } : undefined,
    transformation: transformation ? {
      id: transformation.id,
      title: transformation.title,               // ✅ From Zustand
      status: transformation.status,
      owner: transformation.owner?.name,
    } : undefined,
    // ...
  };

  return {
    'context.envelope': JSON.stringify(envelope),
    // ...
  };
}
```

**Benefits:**
- ✅ No duplicate database queries (data already loaded for UI)
- ✅ No server-side enrichment endpoint needed
- ✅ Always fresh (re-read from Zustand on every message)
- ✅ Simpler architecture (less moving parts)
- ✅ Instant enrichment (no API round-trip)

**Trade-off:**
- Slightly larger message payload (few KB of enriched JSON)
- Context sent with every message instead of stored once

### 3. Repository Service HTTP API

**Expose domain repositories as REST endpoints** for LangGraph tools to call.

```
GET  /api/repositories/:id
     → Repository metadata and stats

GET  /api/repositories/:id/elements/:elementId
     → Element with properties and relationships

GET  /api/repositories/:id/elements?search=...&type=...&layer=...
     → Search elements with filters

GET  /api/repositories/:id/elements/:elementId/neighborhood?depth=2
     → N-hop graph neighborhood
```

These endpoints delegate to `ameide_core_domain/repositories.py` abstractions.

### 4. Knowledge Access Layer (LangGraph Tools)

**Tools call graph service HTTP API** instead of direct SQL.

```python
@tool
async def query_graph(graph_id: str) -> dict:
    """Fetch graph metadata and statistics."""
    # Call graph service via HTTP
    response = await http_client.get(f"/api/repositories/{graph_id}")
    return response.json()

@tool
async def query_element(graph_id: str, element_id: str) -> dict:
    """Fetch element with properties and direct relationships."""
    response = await http_client.get(
        f"/api/repositories/{graph_id}/elements/{element_id}"
    )
    return response.json()

@tool
async def search_elements(graph_id: str, query: str, filters: dict) -> list:
    """Full-text search across elements with filters."""
    response = await http_client.get(
        f"/api/repositories/{graph_id}/elements",
        params={"search": query, **filters}
    )
    return response.json()

@tool
async def get_element_neighborhood(
    graph_id: str,
    element_id: str,
    depth: int = 2
) -> dict:
    """Get element with N-hop neighborhood in graph."""
    response = await http_client.get(
        f"/api/repositories/{graph_id}/elements/{element_id}/neighborhood",
        params={"depth": depth}
    )
    return response.json()
```

**Benefits:**
- Inference layer stays stateless (no database connection)
- Domain logic stays in one place (graph service)
- Easy to test (mock HTTP responses)
- Clear separation of concerns

### 5. Role-Based Prompting & Tool Access

#### User Roles

Different user roles have different needs, expertise levels, and permissions:

| Role | Expertise | Primary Focus | Tool Access | Prompt Style |
|------|-----------|---------------|-------------|--------------|
| **Architect** | Expert | System design, patterns, compliance | Full (all tools) | Technical, detailed, ArchiMate-focused |
| **Developer** | Intermediate | Implementation, APIs, components | Read-heavy, limited write | Practical, code-focused, examples |
| **Business Analyst** | Beginner-Intermediate | Business processes, requirements | Business-layer only | Plain language, business-focused |
| **Stakeholder** | Beginner | High-level overview, decisions | Read-only, summary tools | Executive summary, simple explanations |
| **Admin** | Expert | Repository management, permissions | Full + admin tools | Administrative, configuration-focused |

#### Role-Based Tool Filtering

```python
class ToolAccessControl:
    """Filter tools based on user role and permissions."""

    ROLE_TOOL_MAP = {
        'architect': [
            'query_graph',
            'query_element',
            'search_elements',
            'get_element_neighborhood',
            'create_element',
            'update_element',
            'delete_element',
            'analyze_patterns',
            'validate_compliance'
        ],
        'developer': [
            'query_graph',
            'query_element',
            'search_elements',
            'get_element_neighborhood',
            'find_apis',
            'get_implementation_details'
        ],
        'business_analyst': [
            'query_graph',
            'search_elements',  # Business layer only
            'query_business_process',
            'trace_requirements'
        ],
        'stakeholder': [
            'query_graph',
            'get_graph_summary',
            'get_high_level_view'
        ],
        'admin': [
            # All tools from architect plus:
            'manage_permissions',
            'audit_changes',
            'configure_graph'
        ]
    }

    @staticmethod
    def filter_tools(all_tools: list, user_role: str, permissions: list[str]) -> list:
        """Filter tools based on role and explicit permissions."""
        allowed_tool_names = ToolAccessControl.ROLE_TOOL_MAP.get(user_role, [])

        # Filter tools by name
        filtered = [t for t in all_tools if t.name in allowed_tool_names]

        # Further filter by explicit permissions if needed
        # e.g., if 'write:elements' not in permissions, remove create/update/delete
        if 'write:elements' not in permissions:
            write_tools = ['create_element', 'update_element', 'delete_element']
            filtered = [t for t in filtered if t.name not in write_tools]

        return filtered
```

#### Role-Based System Prompts

```python
class RoleBasedPrompts:
    """Generate role-specific system prompts."""

    @staticmethod
    def get_prompt_for_role(role: str, expertise: str, context) -> str:
        """Generate tailored system prompt based on user role."""

        base_prompts = {
            'architect': """You are an expert enterprise architecture assistant specializing in ArchiMate.

You're helping a professional architect who understands:
- ArchiMate graph and viewpoints
- TOGAF ADM methodology
- Architecture patterns and best practices

Communication style:
- Use technical ArchiMate terminology
- Provide detailed relationship explanations
- Suggest patterns and compliance checks
- Reference ArchiMate specification when relevant

When analyzing architecture:
- Identify missing relationships or elements
- Suggest viewpoint applications
- Validate against ArchiMate rules
- Propose refactoring opportunities""",

            'developer': """You are a practical architecture assistant helping developers understand system design.

You're helping a developer who needs to:
- Understand system components and APIs
- Find implementation details
- Trace dependencies

Communication style:
- Focus on practical implementation
- Provide code-level insights
- Explain technical relationships simply
- Give concrete examples

When analyzing architecture:
- Highlight relevant APIs and interfaces
- Explain component dependencies
- Suggest implementation approaches
- Connect architecture to code""",

            'business_analyst': """You are a business-focused architecture assistant.

You're helping a business analyst who needs to:
- Understand business processes
- Trace business requirements
- Analyze business capabilities

Communication style:
- Avoid technical jargon
- Focus on business value
- Explain in plain language
- Use business terminology

When analyzing architecture:
- Focus on business layer elements
- Explain business process flows
- Connect to business outcomes
- Highlight business capabilities""",

            'stakeholder': """You are an executive-focused architecture assistant.

You're helping a stakeholder who needs:
- High-level understanding
- Strategic insights
- Decision support

Communication style:
- Executive summaries
- Simple, clear language
- Focus on business impact
- Avoid technical details

When analyzing architecture:
- Provide overview and key points
- Highlight strategic alignment
- Explain business implications
- Summarize complexity simply""",

            'admin': """You are an administrative assistant for graph management.

You're helping an administrator who manages:
- Repository configuration
- User permissions
- Audit and compliance

Communication style:
- Administrative focus
- Security-aware
- Process-oriented
- Configuration details

When analyzing architecture:
- Include administrative metadata
- Highlight permission issues
- Suggest configuration improvements
- Provide audit insights"""
        }

        base = base_prompts.get(role, base_prompts['stakeholder'])

        # Add expertise-specific guidance
        if expertise == 'beginner':
            base += "\n\nNote: This user is new to architecture. Explain concepts clearly and provide educational context."
        elif expertise == 'expert':
            base += "\n\nNote: This user is highly experienced. You can use advanced terminology and skip basic explanations."

        # Add current context
        base += f"\n\n{context.to_system_message()}"

        return base
```

#### Example: Context-Aware Responses

**Same question, different roles:**

**Question**: "What is connected to the Customer Management system?"

**Architect Response**:
```
The Customer Management application component has the following relationships:

Composition (contains):
- Customer Data Service (application service)
- CRM Interface (application interface)

Used by (serving):
- Sales Process (business process) via Serving relationship
- Marketing Campaign (business process) via Serving relationship

Accesses (data):
- Customer Database (data object) via Access relationship

Realizes:
- Customer Management capability (business capability)

This follows a standard layered pattern. Consider adding an explicit Assignment relationship to the CRM Team business actor if not already modeled.
```

**Developer Response**:
```
Customer Management system connects to:

APIs it exposes:
- CRM Interface - REST API at /api/crm
- Customer Data Service

APIs it consumes:
- Customer Database

Used by these processes:
- Sales Process
- Marketing Campaign

If you're implementing a feature, you'll likely interact with the CRM Interface. Check the API documentation in the graph docs.
```

**Business Analyst Response**:
```
The Customer Management system supports:

Business Processes:
- Sales Process - helps sales team manage customer relationships
- Marketing Campaign - enables marketing to target customers

It manages:
- Customer data and information

This system is central to how we interact with customers across sales and marketing.
```

**Stakeholder Response**:
```
The Customer Management system supports two key business processes:
1. Sales Process
2. Marketing Campaign

It stores and manages customer information that both departments rely on. This is a critical system for customer-facing operations.
```

### 6. Agent Routing & Specialization

**Why multiple agents?**
- Different UI surfaces map to distinct problem domains (graph browsing vs. diagram editing)
- Role-specific prompts become easier to reason about when attached to a dedicated agent graph
- Tool exposure can be scoped per agent and audited independently

**Agent catalog (initial proposal):**
- `general-threads` – lightweight default agent used on home page; conversational prompt, minimal tools.
- `graph-analyst` – optimized for graph browser context; attaches graph query/search tools and graph-focused system prompt.
- `view-analyst` – activated on diagram/document viewers; extends graph tools with element neighborhood access and modeling guidance prompt.
- `editor-copilot` – used inside diagram editor when a selection is present; richer planning prompt, write-capable tools (e.g., update element) gated by permissions.
- `admin-console` – optional agent for tenant/graph administrators; focuses on governance tooling, surfaced only when `context.user.role === 'admin'`.

**Routing rules:**
1. Determine user role & permissions from `context.user`.
2. Determine UI mode from `context.ui.page`.
3. Select agent by `(page, role)` pair using a routing table with fallbacks (e.g., business analyst in diagram editor → `view-analyst`, read-only toolset).
4. Inject role-aligned prompt fragment + filtered tool list into the chosen agent factory.

**LangGraph structure variations:**
- `general-threads`: single LLM node, optional knowledge lookup stub.
- `graph-analyst`: ReAct loop with graph tool node.
- `view-analyst`: LangGraph branch adding element neighborhood retrieval before answer synthesis.
- `editor-copilot`: Planner/executor split with guardrails; uses context + permissions to decide when to attempt write operations.
- `admin-console`: Adds policy-check node ensuring commands comply with audit requirements.

**Operational considerations:**
- Central agent registry mapping agent id → prompt, tool list, graph factory.
- Telemetry events should include `agentId` to measure adoption and quality.
- Feature flag gating allows rolling out specialized agents incrementally per tenant or role.

### 7. Graph Projection System

#### PostgreSQL → Graph Projection

```python
class GraphProjection:
    """Project relational data into graph structure for agent queries."""

    async def project_element_neighborhood(
        self,
        element_id: str,
        depth: int = 2
    ) -> dict:
        """
        Returns:
        {
          "element": {...},
          "neighbors": {
            "outgoing": [{"relation": "composition", "target": {...}}, ...],
            "incoming": [{"relation": "used-by", "source": {...}}, ...]
          },
          "path_to_root": [...],
          "statistics": {...}
        }
        """
```

## Implementation Plan

### Phase 1: Context Collection (Frontend)

**Goal**: Flexible context hook that works for all pages, each page sends what it knows.

#### Task 1.1: Create Flexible Context Hook

**File**: `services/www_ameide_platform/features/threads/hooks/useChatContext.ts`

**Implementation**:

> Assumes `useSession` from `next-auth/react` exposes `user.role` and `user.permissions` claims.

```typescript
export interface ChatContext {
  tenant: {
    id: string;
  };
  user: {
    id: string;
    displayName?: string;
    role?: UserRole;
    permissions?: string[];
  };
  graph?: {
    id: string;
  };
  element?: {
    id: string;
    kind?: ElementKind;
  };
  selection?: {
    elementId: string;
  };
  ui?: {
    page: 'home' | 'graph' | 'editor' | 'diagram-editor';
    path: string;
  };
}

type ElementKind =
  | 'DOCUMENT_MD' | 'DOCUMENT_PDF' | 'DOCUMENT_DOCX'
  | 'ARCHIMATE_VIEW' | 'ARCHIMATE_ELEMENT'
  | 'BPMN_VIEW' | 'BPMN_ELEMENT'
  | 'C4_VIEW' | 'C4_ELEMENT'
  | 'UML_VIEW' | 'UML_ELEMENT'
  | 'FOLDER';

type UserRole = 'architect' | 'developer' | 'business_analyst' | 'stakeholder' | 'admin';

/**
 * Base hook: Extract context from URL.
 * Works for all pages, provides what's available in the URL.
 */
export function useChatContext(): ChatContext {
  const pathname = usePathname();
  const { data: session } = useSession();

  return useMemo(() => {
    const segments = pathname.split('/').filter(Boolean);

    // Extract tenant (always present or default)
    const orgIndex = segments.indexOf('org');
    const tenantId = orgIndex >= 0 && segments[orgIndex + 1]
      ? segments[orgIndex + 1]
      : 'default-tenant';

    // Extract graph (if in URL)
    const repoIndex = segments.indexOf('repo');
    const graphId = repoIndex >= 0 && segments[repoIndex + 1]
      ? segments[repoIndex + 1]
      : undefined;

    // Extract element (if in URL)
    // Pattern: /org/[orgId]/repo/[repoId]/element/[elementId]
    const elementIndex = segments.indexOf('element');
    const elementId = elementIndex >= 0 && segments[elementIndex + 1]
      ? segments[elementIndex + 1]
      : undefined;

    // Determine page type from URL
    let page: ChatContext['ui']['page'] = 'home';
    if (pathname.includes('/element/') && pathname.includes('/editor')) {
      page = 'diagram-editor';
    } else if (pathname.includes('/element/')) {
      page = 'editor';
    } else if (graphId) {
      page = 'graph';
    }

    const userId = session?.user?.id ?? 'anonymous';
    const role = session?.user?.role ?? undefined;
    const permissions = Array.isArray(session?.user?.permissions)
      ? session?.user?.permissions
      : undefined;

    return {
      tenant: { id: tenantId },
      user: {
        id: userId,
        displayName: session?.user?.name ?? undefined,
        role,
        permissions,
      },
      graph: graphId ? { id: graphId } : undefined,
      element: elementId ? { id: elementId } : undefined,
      ui: {
        page,
        path: pathname
      }
    };
  }, [pathname]);
}

/**
 * Enhanced hook for diagram editor: Includes selected element on canvas.
 * Use this in diagram editor pages.
 */
export function useDiagramEditorContext(): ChatContext {
  const baseContext = useChatContext();
  const selectedElement = useSelectedElement(); // From editor store

  return useMemo(() => ({
    ...baseContext,
    // Base element is the diagram itself (ARCHIMATE_VIEW, BPMN_VIEW, etc.)
    element: baseContext.element ? {
      ...baseContext.element,
      kind: baseContext.element.kind || 'ARCHIMATE_VIEW' // Default, should come from data
    } : undefined,
    // Selection is the element selected within the diagram
    selection: selectedElement ? { elementId: selectedElement.id } : undefined
  }), [baseContext, selectedElement]);
}
```

**Usage in different pages:**

```typescript
// Home page
const HomePage = () => {
  const context = useChatContext();
  // context = { tenant: { id: "acme" }, ui: { page: "home", path: "/" } }
};

// Repository browser
const RepositoryPage = () => {
  const context = useChatContext();
  // context = { tenant, graph: { id: "main" }, ui: { page: "graph", ... } }
};

// Element viewer (e.g., document viewer)
const ElementViewer = () => {
  const context = useChatContext();
  // context = { tenant, graph, element: { id: "doc-123", kind: "DOCUMENT_MD" }, ui: { page: "editor", ... } }
};

// Diagram editor with selection
const DiagramEditor = () => {
  const context = useDiagramEditorContext();
  // context = {
  //   tenant,
  //   graph,
  //   element: { id: "view-456", kind: "ARCHIMATE_VIEW" },
  //   selection: { elementId: "elem-789" },
  //   ui: { page: "diagram-editor", ... }
  // }
};
```

**Why flexible:**
- Same hook works everywhere, extracts what's available
- Specialized hooks (like `useDiagramEditorContext`) layer on additional context
- Pages provide progressively richer context as user navigates deeper
- No need to refactor when adding new context fields

**Alignment with 303-elements.md:**
- URL pattern: `/org/[orgId]/repo/[repoId]/element/[elementId]`
- Element can be a diagram (`ARCHIMATE_VIEW`), document (`DOCUMENT_MD`), or folder (`FOLDER`)
- No separate `/artifact/` routes - everything goes through `/element/`

#### Task 1.2: Update Chat Provider to Use Flexible Context

**File**: `services/www_ameide_platform/features/threads/providers/ChatProvider.tsx`

**Changes**:

```typescript
// Use the appropriate context hook based on where threads is used
// Default to base hook, can be overridden via props
const defaultContext = useChatContext();
const context = props.context ?? defaultContext;

const {
  messages,
  sendMessage,
  // ...
} = useInferenceStream({
  threadsId: resolvedChatId,
  initialMessages,
  parameters: {
    context: JSON.stringify(context), // Flexible: varies by page
    context_version: '1.0'
  },
});
```

**Result**:
- Home page sends `{ tenant }`
- Repository page sends `{ tenant, graph }`
- Element viewer sends `{ tenant, graph, element }`
- Diagram editor sends `{ tenant, graph, element, selection }`
- Agent gets progressively richer context as user navigates deeper

### Phase 2: Client-Side Context Enrichment (Using Zustand)

**Goal**: Use data already loaded in UI state stores instead of duplicate server queries.

**Architecture Change:** Instead of server-side enrichment with database queries, read enriched data from Zustand stores that are already populated for UI rendering.

#### Task 2.1: Identify Existing Zustand Stores

**Goal:** Find the Zustand stores that already hold graph, transformation, and organization data.

**Files to check:**
- Repository store: `features/graph/store/*` or `features/common/stores/*`
- Initiative store: `features/transformations/store/*`
- Organization store: `features/organization/store/*`

**What to extract:**
- Repository: `id`, `name`, `type`, `elementCount` (or `stats.elementCount`), `updatedAt`
- Initiative: `id`, `title`, `status`, `owner`
- Organization: `id`, `name`, `type`

#### Task 2.2: Update useChatContextMetadata Hook

**File**: `services/www_ameide_platform/features/threads/hooks/useChatContext.ts`

**Changes:**

```typescript
import { useRepositoryStore } from '@/features/graph/store';  // Find actual path
import { useInitiativeStore } from '@/features/transformations/store'; // Find actual path
import { useOrganizationStore } from '@/features/organization/store'; // Find actual path

export function useChatContextMetadata(): ChatContextMetadata {
  const pathname = usePathname();
  const searchParams = useSearchParams();
  const { data: session } = useSession();

  // Read enriched data from Zustand stores
  const graph = useRepositoryStore(state => state.currentRepository);
  const transformation = useInitiativeStore(state => state.currentInitiative);
  const organization = useOrganizationStore(state => state.organization);

  return useMemo(() => {
  const enriched: any = {};

  // Always enrich tenant
  const tenant = await this.db.query(
    'SELECT id, name, type FROM tenants WHERE id = $1',
    [context.tenantId]
  );
  enriched.tenant = tenant.rows[0];

  // Enrich graph if provided
  if (context.graphId) {
    const graph = await this.db.query(
      `SELECT id, name, type,
              (SELECT COUNT(*) FROM elements WHERE graph_id = $1) as element_count,
              updated_at as last_modified
       FROM repositories WHERE id = $1`,
      [context.graphId]
    );
    enriched.graph = graph.rows[0];
  }

  // Enrich element if provided
  // Element = primary item being viewed (diagram, document, folder, etc.)
  if (context.elementId) {
    const element = await this.db.query(
      `SELECT id, title, element_kind, type_key, description
       FROM elements WHERE id = $1`,
      [context.elementId]
    );
    enriched.element = element.rows[0];
  }

  // Enrich selection if provided
  // Selection = element selected within a diagram
  if (context.selectionId) {
    const selection = await this.db.query(
      `SELECT id, title, element_kind, type_key
       FROM elements WHERE id = $1`,
      [context.selectionId]
    );
    enriched.selection = selection.rows[0];
  }

  // Store UI context as-is
  if (context.ui) {
    enriched.ui = context.ui;
  }

  // Store enriched context with thread
  await this.db.query(
    'UPDATE threads_threads SET context = $1 WHERE id = $2',
    [JSON.stringify(enriched), threadId]
  );

  return enriched;
}
```

**Alignment with 303-elements.md:**
- ✅ Query from `elements` table (not `artifacts`)
- ✅ Use `element_kind` field (e.g., `ARCHIMATE_VIEW`, `DOCUMENT_MD`)
- ✅ Use `type_key` for subtype (e.g., `business-actor`, `adr`)
- ✅ Use `title` field (not `name`)
- ✅ Support both element (primary) and selection (within element)

**API Route**: `POST /api/threads/threads/:threadId/context`

```typescript
// services/www_ameide_platform/app/api/threads/threads/[threadId]/context/route.ts
export async function POST(
  req: Request,
  { params }: { params: { threadId: string } }
) {
  const context = await req.json();

  const enrichedContext = await threadsService.enrichThreadContext(
    params.threadId,
    context
  );

  return Response.json({ context: enrichedContext });
}
```

**Result**:
- Home page context (`{ tenant }`) → Enriches tenant only
- Repository page context (`{ tenant, graph }`) → Enriches both
- Element viewer context (`{ tenant, graph, element }`) → Enriches all three
- Diagram editor context (`{ tenant, graph, element, selection }`) → Enriches all four
- Enrichment adapts to what's provided

#### Task 2.2: Update Chat Provider to Enrich on Create

**File**: `services/www_ameide_platform/features/threads/providers/ChatProvider.tsx`

```typescript
const createThread = useCallback(async () => {
  const context = useChatContext(); // Or useDiagramEditorContext() for editor

  // Create thread
  const newThreadId = generateUUID();

  // Build enrichment request from context
  const enrichmentData = {
    tenantId: context.tenant.id,
    graphId: context.graph?.id,
    elementId: context.element?.id,  // Primary element (diagram, document, folder, etc.)
    selectionId: context.selection?.elementId,  // Selected element within diagram (if applicable)
    ui: context.ui
  };

  // Enrich thread with whatever context is available
  await fetch(`/api/threads/threads/${newThreadId}/context`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(enrichmentData)
  });

  return newThreadId;
}, []);
```

**Result**: Enrichment request adapts to current page - sends only what's available.

#### Task 2.3: Flexible Context Parser (Inference)

**File**: `services/inference/lib/context.py`

**Implementation**:

```python
from dataclasses import dataclass
from typing import Optional
import json

@dataclass
class ChatContext:
    """Flexible context envelope - adapts to different UI pages."""
    tenant_id: str
    graph_id: Optional[str] = None
    element_id: Optional[str] = None  # Primary element (diagram, document, folder, etc.)
    selection_id: Optional[str] = None  # Selected element within diagram (if applicable)
    page: Optional[str] = None  # 'home', 'graph', 'editor', 'diagram-editor'

    @classmethod
    def from_metadata(cls, metadata: dict) -> "ChatContext":
        """Parse flexible context from request metadata."""
        context_json = metadata.get("context")
        if not context_json:
            return cls(tenant_id="default")

        try:
            data = json.loads(context_json)
            return cls(
                tenant_id=data.get("tenant", {}).get("id", "default"),
                graph_id=data.get("graph", {}).get("id"),
                element_id=data.get("element", {}).get("id"),
                selection_id=data.get("selection", {}).get("elementId"),
                page=data.get("ui", {}).get("page")
            )
        except (json.JSONDecodeError, AttributeError):
            return cls(tenant_id="default")

    def has_graph_context(self) -> bool:
        """Check if user is in graph context."""
        return self.graph_id is not None

    def has_element_context(self) -> bool:
        """Check if user is viewing/editing a specific element."""
        return self.element_id is not None

    def has_selection_context(self) -> bool:
        """Check if user has selected an element within a diagram."""
        return self.selection_id is not None
```

**Alignment with 303-elements.md:**
- ✅ Removed `artifact_id` (unified into `element_id`)
- ✅ Added `selection_id` for diagram canvas selections
- ✅ Helper methods reflect new terminology

**Why flexible:**
- Parses whatever context is provided
- Provides helper methods to check context level
- Inference can adapt agent behavior based on context richness

#### Task 2.4: Update Inference to Build Context-Aware Prompts

**File**: `services/inference/main.py`

**Changes**:

```python
from lib.context import ChatContext

@app.post("/agents/stream")
async def stream_agent(request: AgentRequest):
    """Stream agent responses with context-aware prompting."""

    # Parse context (flexible: adapts to different pages)
    context = ChatContext.from_metadata(request.metadata or {})

    # Fetch enriched thread context from threads service
    thread_response = await threads_client.get(f"/threads/{request.thread_id}")
    thread = thread_response.json()
    enriched = thread.get("context", {})

    # Build context-aware system prompt
    system_prompt = build_context_prompt(context, enriched)

    # Create agent with appropriate tools based on context
    if context.has_graph_context():
        # Repository or editor page - provide graph tools
        repo_tools = RepositoryTools(context.graph_id)
        tools = repo_tools.get_all_tools()
        agent = create_react_agent(
            model=llm,
            tools=tools,
            state_modifier=SystemMessage(content=system_prompt)
        )
        logger.info(f"Repository agent | page={context.page} | tools={len(tools)}")
    else:
        # Home page - general threads without tools
        agent = create_react_agent(
            model=llm,
            tools=[],
            state_modifier=SystemMessage(content=system_prompt)
        )
        logger.info(f"General threads agent | page={context.page}")

    # Stream...

def build_context_prompt(context: ChatContext, enriched: dict) -> str:
    """Build system prompt based on context level."""

    # Home page: General architecture assistant
    if not context.has_graph_context():
        tenant_name = enriched.get("tenant", {}).get("name", "your organization")
        return f"""You are a helpful enterprise architecture assistant for {tenant_name}.

Help users understand architecture concepts, best practices, and methodologies.
When they navigate to a graph, you'll have access to specific architecture data."""

    # Repository page: Repository-aware assistant
    repo = enriched.get("graph", {})
    repo_name = repo.get("name", "this graph")
    element_count = repo.get("element_count", 0)

    if not context.has_element_context():
        return f"""You are an expert ArchiMate architecture assistant.

Current Repository: {repo_name} ({element_count} elements)

You have access to tools to query this graph:
- query_graph: Get graph metadata and statistics
- query_element: Get element details with relationships
- search_elements: Search for elements by name/type/layer
- get_element_neighborhood: Explore graph neighborhoods

Help users understand and navigate the architecture in this graph."""

    # Element editor/viewer page: Element-focused assistant
    element = enriched.get("element", {})
    element_title = element.get("title", "this element")
    element_kind = element.get("element_kind", "")

    # Determine if it's a view (diagram) or other element type
    is_view = "VIEW" in element_kind
    element_type_label = "diagram" if is_view else "element"

    prompt = f"""You are an expert ArchiMate architecture assistant.

Current Context:
- Repository: {repo_name}
- {element_type_label.capitalize()}: {element_title} ({element_kind})
"""

    # If viewing a diagram with selection
    if context.has_selection_context():
        selection = enriched.get("selection", {})
        selection_title = selection.get("title", "selected element")
        selection_type = selection.get("type_key", "unknown")
        prompt += f"- Selected on Canvas: {selection_title} ({selection_type})\n"

    prompt += """
You have access to graph query tools. When users ask about elements:
1. Use tools to fetch accurate, up-to-date information
2. Consider the current diagram/element and any selection
3. Provide context-relevant insights
"""

    return prompt
```

**Alignment with 303-elements.md:**
- ✅ Changed `has_artifact_context()` to `has_element_context()`
- ✅ Use `element.title` (not `name`)
- ✅ Use `element.element_kind` (e.g., `ARCHIMATE_VIEW`)
- ✅ Use `selection.type_key` for element type
- ✅ Different prompts for VIEWs (diagrams) vs other elements
- ✅ Handle selection within diagrams

**Result**:
- Home page → General architecture threads (no tools)
- Repository page → Repository-aware threads with query tools
- Element viewer → Element-focused threads (documents, folders, etc.)
- Diagram editor → Diagram-focused threads with selection awareness
- Prompts adapt to user's current location and element type

### Phase 3: Repository Service HTTP API

**Goal**: Expose domain repositories as REST endpoints for tools to call.

#### Task 3.1: Create Repository Service API Routes

**File**: `services/graph/src/api/routes.py` (or similar)

**Implementation**:

```python
from fastapi import APIRouter, HTTPException
from ameide_core_domain.repositories import RepositoryReader, ElementReader

router = APIRouter(prefix="/api/repositories")

@router.get("/{graph_id}")
async def get_graph(graph_id: str):
    """Fetch graph metadata and statistics."""
    repo_reader = RepositoryReader(db_pool)
    repo = await repo_reader.get_by_id(graph_id)

    if not repo:
        raise HTTPException(status_code=404, detail="Repository not found")

    # Get statistics
    stats = await repo_reader.get_statistics(graph_id)

    return {
        "id": repo.id,
        "name": repo.name,
        "type": repo.type,
        "description": repo.description,
        "element_count": stats.element_count,
        "artifact_count": stats.artifact_count,
        "last_modified": repo.updated_at
    }

@router.get("/{graph_id}/elements/{element_id}")
async def get_element(graph_id: str, element_id: str):
    """Fetch element with properties and direct relationships."""
    element_reader = ElementReader(db_pool)
    element = await element_reader.get_by_id(element_id, graph_id)

    if not element:
        raise HTTPException(status_code=404, detail="Element not found")

    # Get relationships
    relationships = await element_reader.get_relationships(element_id)

    return {
        "id": element.id,
        "name": element.name,
        "type": element.type,
        "layer": element.layer,
        "description": element.description,
        "properties": element.properties,
        "relationships": {
            "outgoing": [
                {
                    "type": r.type,
                    "target_id": r.target_id,
                    "target_name": r.target_name,
                    "target_type": r.target_type
                }
                for r in relationships.outgoing
            ],
            "incoming": [
                {
                    "type": r.type,
                    "source_id": r.source_id,
                    "source_name": r.source_name,
                    "source_type": r.source_type
                }
                for r in relationships.incoming
            ]
        }
    }

@router.get("/{graph_id}/elements")
async def search_elements(
    graph_id: str,
    search: str = "",
    type: str | None = None,
    layer: str | None = None,
    limit: int = 10
):
    """Search elements by name/description with optional filters."""
    element_reader = ElementReader(db_pool)

    results = await element_reader.search(
        graph_id=graph_id,
        query=search,
        element_type=type,
        layer=layer,
        limit=limit
    )

    return [
        {
            "id": el.id,
            "name": el.name,
            "type": el.type,
            "layer": el.layer,
            "description": el.description
        }
        for el in results
    ]

@router.get("/{graph_id}/elements/{element_id}/neighborhood")
async def get_element_neighborhood(
    graph_id: str,
    element_id: str,
    depth: int = 2
):
    """Get N-hop neighborhood of an element in the graph."""
    element_reader = ElementReader(db_pool)

    neighborhood = await element_reader.get_neighborhood(
        element_id=element_id,
        graph_id=graph_id,
        depth=min(max(depth, 1), 3)  # Clamp to 1-3
    )

    return {
        "nodes": [
            {
                "id": node.id,
                "name": node.name,
                "type": node.type,
                "layer": node.layer,
                "depth": node.depth
            }
            for node in neighborhood.nodes
        ],
        "edges": [
            {
                "source_id": edge.source_id,
                "target_id": edge.target_id,
                "type": edge.type
            }
            for edge in neighborhood.edges
        ],
        "root": element_id,
        "depth": depth
    }
```

**Note**: These endpoints use existing `ameide_core_domain/repositories.py` abstractions, keeping SQL in the domain layer.

#### Task 3.2: Create LangGraph Tools (HTTP Client)

**File**: `packages/ameide_core_inference_agents/src/ameide_core_inference_agents/tools/graph_tools.py`

**Implementation**:

```python
from langchain_core.tools import tool
from typing import Any
import httpx

# Repository service base URL (from environment)
REPOSITORY_SERVICE_URL = os.getenv("REPOSITORY_SERVICE_URL", "http://graph:8080")

class RepositoryTools:
    """Tools that call graph service HTTP API."""

    def __init__(self, graph_id: str):
        self.graph_id = graph_id
        self.client = httpx.AsyncClient(base_url=REPOSITORY_SERVICE_URL)

    @tool
    async def query_graph(self) -> dict[str, Any]:
        """
        Fetch graph metadata including artifact count, element statistics.

        Returns:
            Dictionary with graph details, statistics, and metadata
        """
        response = await self.client.get(f"/api/repositories/{self.graph_id}")
        response.raise_for_status()
        return response.json()

    @tool
    async def query_element(self, element_id: str) -> dict[str, Any]:
        """
        Fetch element with its properties and direct relationships.

        Args:
            element_id: The element identifier

        Returns:
            Element details with incoming and outgoing relationships
        """
        response = await self.client.get(
            f"/api/repositories/{self.graph_id}/elements/{element_id}"
        )
        response.raise_for_status()
        return response.json()

    @tool
    async def search_elements(
        self,
        query: str,
        element_type: str | None = None,
        layer: str | None = None,
        limit: int = 10
    ) -> list[dict[str, Any]]:
        """
        Search elements by name/description with optional filters.

        Args:
            query: Search text
            element_type: Filter by ArchiMate type
            layer: Filter by layer (business, application, technology, etc.)
            limit: Maximum results to return

        Returns:
            List of matching elements
        """
        params = {"search": query, "limit": limit}
        if element_type:
            params["type"] = element_type
        if layer:
            params["layer"] = layer

        response = await self.client.get(
            f"/api/repositories/{self.graph_id}/elements",
            params=params
        )
        response.raise_for_status()
        return response.json()

    @tool
    async def get_element_neighborhood(
        self,
        element_id: str,
        depth: int = 2
    ) -> dict[str, Any]:
        """
        Get N-hop neighborhood of an element in the graph.

        Args:
            element_id: Starting element
            depth: How many hops to traverse (1-3)

        Returns:
            Graph structure with nodes and edges
        """
        response = await self.client.get(
            f"/api/repositories/{self.graph_id}/elements/{element_id}/neighborhood",
            params={"depth": depth}
        )
        response.raise_for_status()
        return response.json()

    def get_all_tools(self):
        """Return list of all tools for agent."""
        return [
            self.query_graph,
            self.query_element,
            self.search_elements,
            self.get_element_neighborhood
        ]
```

**Benefits:**
- No database connection in inference layer
- All SQL stays in domain layer via graph service
- Easy to test with mocked HTTP responses
- Inference stays stateless

### Phase 4: Integration & Wiring

#### Task 4.1: Update Inference Service to Use Repository Tools

**File**: `services/inference/main.py`

**Changes**:

```python
from ameide_core_inference_agents.tools.graph_tools import RepositoryTools
from lib.context import MinimalContext
from langgraph.prebuilt import create_react_agent
from langchain_core.messages import SystemMessage
import httpx

# HTTP client for threads service (to fetch enriched context)
threads_client = httpx.AsyncClient(
    base_url=os.getenv("CHAT_SERVICE_URL", "http://threads-service:8000")
)

@app.post("/agents/stream")
async def stream_agent(request: AgentRequest):
    """Stream agent responses with graph tools."""

    # Parse minimal context (just IDs)
    context = MinimalContext.from_metadata(request.metadata or {})

    # Fetch enriched context from threads service
    thread_response = await threads_client.get(f"/threads/{request.thread_id}")
    thread = thread_response.json()
    enriched_context = thread.get("context", {})

    logger.info(
        f"Thread: {request.thread_id} | "
        f"Repository: {context.graph_id or 'none'}"
    )

    # Build system message from enriched context
    system_lines = []
    if enriched_context.get("tenant"):
        tenant = enriched_context["tenant"]
        system_lines.append(f"Organization: {tenant.get('name')} (ID: {tenant.get('id')})")

    if enriched_context.get("graph"):
        repo = enriched_context["graph"]
        system_lines.append(f"Repository: {repo.get('name')} ({repo.get('type')})")
        system_lines.append(f"  └─ {repo.get('element_count', 0)} elements")
        system_lines.append(f"  └─ Last modified: {repo.get('last_modified', 'unknown')}")

    # Create agent with graph tools if in repo context
    if context.graph_id:
        # Initialize tools for this graph
        repo_tools = RepositoryTools(context.graph_id)
        tools = repo_tools.get_all_tools()

        system_prompt = f"""You are an expert ArchiMate architecture assistant.

Current Context:
{chr(10).join(system_lines)}

You have access to tools to query the architecture graph:
- query_graph: Get graph metadata and statistics
- query_element: Get detailed element information with relationships
- search_elements: Search for elements by name/type/layer
- get_element_neighborhood: Explore graph neighborhoods

When users ask about elements, relationships, or the architecture:
1. Use the tools to fetch accurate, up-to-date information
2. Explain ArchiMate concepts clearly
3. Suggest relevant elements or relationships based on context
4. Provide insights about the architecture structure
"""

        agent = create_react_agent(
            model=llm,
            tools=tools,
            state_modifier=SystemMessage(content=system_prompt)
        )

        logger.info(f"Created graph agent with {len(tools)} tools")
    else:
        # Use simple threads agent for non-graph contexts
        system_prompt = f"""You are a helpful architecture assistant.

{chr(10).join(system_lines) if system_lines else 'No specific context available.'}

Help the user understand enterprise architecture concepts and best practices.
"""

        agent = create_react_agent(
            model=llm,
            tools=[],  # No tools for general threads
            state_modifier=SystemMessage(content=system_prompt)
        )

        logger.info("Created simple threads agent (no graph context)")

    # Stream with agent...
    # (existing streaming logic continues here)
```

**Key Changes:**
- Removed database connection from inference service
- Fetch enriched context from threads service instead of building from metadata
- Use RepositoryTools that call HTTP API instead of direct SQL
- Inference stays stateless (no DB, just HTTP calls)

#### Task 4.2: Add Environment Configuration

**File**: `infra/kubernetes/charts/platform/inference/values.yaml`

**Add**:

```yaml
env:
  # Service URLs (no database credentials needed)
  - name: CHAT_SERVICE_URL
    value: http://threads-service:8000
  - name: REPOSITORY_SERVICE_URL
    value: http://graph-service:8000
```

**File**: `infra/kubernetes/charts/platform/graph/values.yaml`

**Add** (graph service needs DB access):

```yaml
env:
  - name: DB_HOST
    value: postgres-platform-rw
  - name: DB_NAME
    value: ameide_platform
  - name: DB_USER
    value: postgres
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: postgres-credentials
        key: password
```

**Result**: Only graph service has database access. Inference just calls HTTP APIs.

### Phase 5: Agent Routing & Specialization

**Goal**: Route threads requests to role/page-specific LangGraph agents with tailored prompts and toolsets.

#### Task 5.1: Define agent registry
- Create configuration describing agent id, prompt template, tool allowlist, and LangGraph factory
- Store registry in `packages/ameide_core_inference_agents` for reuse across services

#### Task 5.2: Implement agent router
- Add router in inference entrypoint (or threads service) that maps `(context.user.role, context.ui.page)` to an agent id
- Provide safe fallbacks (default to `general-threads` when context incomplete)

#### Task 5.3: Wire specialized agents
- Instantiate `graph-analyst`, `view-analyst`, `editor-copilot`, and `admin-console` agents using the new registry
- Connect role-specific prompt templates and tool filters defined earlier in this document

#### Task 5.4: Add telemetry & tests
- Emit agent selection metrics (`agentId`, role, page)
- Unit tests for routing matrix and permission-based tool filtering
- Integration tests that confirm correct agent selection when switching UI surfaces

## Testing Strategy

### Unit Tests

1. **Frontend Minimal Context Hook**
 - Test context extraction from URL patterns (`/org/[orgId]/repo/[repoId]`)
 - Test fallback to default tenant when no org in URL
 - Test undefined graph when no repo in URL
  - Test default user when session info missing
  - Test memoization

2. **Chat Service Context Enrichment**
  - Test fetching tenant details from database
  - Test mapping user role/permissions from auth claims
  - Test fetching graph details and stats
  - Test storing enriched context with thread
  - Test error handling when tenant/repo not found

3. **Backend Context Parser**
  - Test parsing minimal context (tenant + user + repo IDs)
  - Test fallback to default tenant when context missing
  - Test error handling for malformed JSON

4. **Repository Tools (HTTP Client)**
   - Mock HTTP responses from graph service
   - Test query_graph tool
   - Test query_element with relationships
   - Test search_elements with filters
   - Test get_element_neighborhood with depth
   - Test error handling for 404/500 responses

### Integration Tests

1. **End-to-End Context Flow**
   - Send message from UI at `/org/acme/repo/main-arch`
  - Verify minimal context (tenant + user + repo IDs) sent in metadata
  - Verify threads service enriches thread with tenant/repo/user details
   - Verify inference fetches enriched context from thread
   - Verify agent receives correct graph context

2. **Repository Service API**
   - Test graph endpoints with real database
   - Verify element queries return correct relationships
   - Test search with various filters
   - Test graph traversal with actual element data

3. **Tool Execution via HTTP**
   - Test tools calling graph service endpoints
   - Verify responses parsed correctly
   - Test error handling when service unavailable

### E2E Tests

1. **Agent Conversations in Repository Context**
   - Navigate to `/org/acme/repo/main-arch`
   - Ask: "What elements are in this graph?"
   - Verify agent uses query_graph tool
   - Verify response includes element count from context

2. **Element Queries**
   - Ask: "Show me relationships for element X"
   - Verify agent uses query_element tool
   - Verify response includes incoming/outgoing relationships

3. **Search and Discovery**
   - Ask: "Find all application components"
   - Verify agent uses search_elements tool with layer filter
   - Verify results filtered to application layer

4. **Graph Exploration**
   - Ask: "What is connected to element X?"
   - Verify agent uses get_element_neighborhood tool
   - Verify response includes neighbor elements and relationships

## Success Criteria

**Phase 1 (Context Collection - MOSTLY COMPLETE ✅):**
- [x] UI extracts tenant + user + repo IDs from URL/session
- [x] Support both `/repo/` and `/graph/` URL patterns
- [x] Support both `/transformation/` and `/transformations/` URL patterns
- [x] Detect settings pages (`/settings`)
- [x] Extract transformation context from URLs
- [x] Context envelope includes: tenant, user, graph, transformation, element, selection, settings, ui
- [x] Metadata includes: tenantId, graphId, transformationId, elementId, isSettings, uiPage, userRole
- [x] Chat provider passes context to inference
- [x] Chat panel integrated into settings pages (organization, graph, transformation)
- [x] Shared SettingsLayout component with threads support
- [x] Server allocates/persists a thread identifier before streaming so the first user message survives refresh/navigation
- [x] E2E test validates thread persistence (`features/threads/__tests__/e2e/thread-persistence.spec.ts`)

**Phase 2 (Client-Side Context Enrichment - ✅ COMPLETE):**
- [x] Create `contextCacheStore` for caching enriched context data
- [x] Update useChatContextMetadata to read from context cache stores
- [x] Include enriched graph data (name, type, elementCount, description, lastModified) in envelope
- [x] Include enriched transformation data (title, status, owner, targetDate) in envelope
- [x] Include enriched organization data (name, type) in envelope
- [x] Organization layout populates organization context on mount
- [x] Repository page populates graph context when data loads
- [x] Initiative page populates transformation context when data loads
- [x] Context cleared when navigating away from pages
- [x] Unit tests for contextCacheStore (10 tests, 100% coverage)
- [x] Unit tests for useChatContext enrichment (9 tests)
- [x] E2E test for end-to-end context enrichment flow
- [ ] Verify enriched context arrives at inference service (pending E2E test environment fix)
- [x] Update inference to use enriched context in system prompts (blocked by above)

**Phase 3 (Repository Service API - ✅ COMPLETE):**
- [x] Repository service exposes REST endpoints on port 8080
- [x] Dual-protocol architecture: HTTP/1.1 REST (8080) + HTTP/2 gRPC (8081)
- [x] REST API endpoints implemented:
  - `GET /api/repositories/:id` - Repository metadata and statistics
  - `GET /api/repositories/:id/elements` - Search/list elements with filters
  - `GET /api/repositories/:id/elements/:elementId` - Element details
  - `GET /api/repositories/:id/elements/:elementId/neighborhood` - Graph neighborhood
- [x] Endpoints use gRPC handlers (delegates to domain layer, no direct SQL)
- [x] Kubernetes service updated to expose both ports
- [x] Inference configured to call `http://graph:8080`
- [x] Express server with error handling and Connect error code mapping
- [ ] Integration testing with inference tools (pending inference update)

**Phase 4 (Integration - 🟡 PARTIAL):**
- [x] Inference `ParsedContext` includes all enriched fields (names, counts, descriptions)
- [x] System prompts use enriched names: "Repository: Primary Architecture (archimate) - 156 elements"
- [x] Repository HTTP REST API ready for tool integration
- [x] Inference tools updated to call graph HTTP API (context-driven HTTP client)
- [ ] Agent can answer "What graph am I in?" with correct name from enriched context
- [ ] Agent can answer "What transformation am I working on?" with enriched details
- [ ] Agent can query elements using graph tools
- [ ] Agent can search elements with filters
- [ ] Agent can explore graph neighborhoods
- [ ] Tool queries execute in < 500ms for typical graphs
- [x] Inference configured for HTTP-only communication (no database connection needed)
- [x] Context propagation works across graph pages
- [x] Context propagation works across transformation pages
- [x] Context propagation works across settings pages

**Phase 5 (Agent Routing - 🟡 PARTIAL):**
- [ ] Router selects agent id based on `context.user.role` + `context.ui.page`
- [x] Tool allowlists enforced per agent (no unauthorized tool usage)
- [x] Telemetry captures `agentId`, role, and page for observability
- [ ] Settings-specific agent for configuration assistance
- [ ] Initiative-specific agent for transformation tracking

## Future Enhancements

**Context Expansion (Phase 1+):**
1. **User Context Enhancements**: Capture user expertise level, team, and feature flags in context
2. **Element Kind Detection**: Auto-detect element_kind from URL or data
3. **Selection Context**: Include selected element(s) within diagrams
4. **UI State**: Include view mode, filters, zoom level

**Role-Based Features (Phase 2+):**
5. **Role-Based Prompts**: Tailor response style per user role
6. **Role-Based Tool Access**: Filter tools by role and permissions
7. **Expertise Levels**: Adjust technical detail based on user experience
8. **Addressable Personas**: Expose `context.user.displayName` in agent prompts so assistants can greet users by name

**Advanced Knowledge (Phase 3+):**
8. **Vector Search**: Semantic search over element descriptions
9. **Multi-Repository**: Cross-graph queries and comparisons
10. **Temporal Queries**: "What changed in the last week?"
11. **Recommendation Engine**: "Elements similar to X"
12. **Graph Analytics**: Centrality, clustering, impact analysis
13. **Export Tools**: Generate reports, diagrams from agent queries

## Dependencies

**Frontend:**
- Next.js routing for URL pattern extraction
- React hooks (usePathname, useMemo)

**Chat Service:**
- PostgreSQL access (for tenant/graph lookups)
- Thread storage with context column

**Repository Service:**
- `ameide_core_domain/repositories.py` abstractions
- PostgreSQL with pg_trgm extension for similarity search
- FastAPI for REST endpoints

**Inference Service:**
- LangGraph for agent orchestration
- httpx for HTTP client (threads service, graph service)
- No database connection required

## Timeline Estimate

**Phase 1: Minimal Context (Frontend)**
- Task 1.1: Create minimal context hook (useMinimalChatContext) - 1 day
- Task 1.2: Update threads provider to send context - 1 day
- Testing: URL extraction, context serialization - 0.5 days
**Subtotal: 2-3 days**

**Phase 2: Context Enrichment (Chat Service)**
- Task 2.1: Add context enrichment endpoint - 2 days
- Task 2.2: Update threads provider to enrich threads - 1 day
- Task 2.3: Create minimal context parser (inference) - 0.5 days
- Task 2.4: Update inference to read thread context - 1 day
- Testing: Enrichment flow, database queries - 1 day
**Subtotal: 5-6 days**

**Phase 3: Repository Service API**
- Task 3.1: Create graph REST endpoints - 3 days
  - GET /repositories/:id
  - GET /repositories/:id/elements/:elementId
  - GET /repositories/:id/elements (search)
  - GET /repositories/:id/elements/:elementId/neighborhood
- Task 3.2: Create LangGraph tools (HTTP client) - 2 days
- Testing: API endpoints, tool execution - 2 days
**Subtotal: 6-8 days**

**Phase 4: Integration**
- Task 4.1: Update inference to use graph tools - 2 days
- Task 4.2: Environment configuration (Helm values) - 0.5 days
- Testing: End-to-end flow, agent conversations - 2 days
**Subtotal: 4-5 days**

**Phase 5: Agent Routing & Specialization**
- Task 5.1: Agent registry definitions - 1 day
- Task 5.2: Router implementation + fallbacks - 1.5 days
- Task 5.3: Wire specialized agents & prompts - 1.5 days
- Task 5.4: Telemetry + routing tests - 1 day
**Subtotal: 5-6 days**

**Total: 22-28 days (4.5-5.5 sprints)**

**Comparison to original estimate:**
- Original (single agent, artifact-centric context, direct SQL): 20-28 days
- Incremental plan (structured context, graph abstraction, agent routing): 22-28 days
- **Net change**: +2-4 days while adding safer data access and multi-agent routing capabilities

## Notes

**Implementation Strategy:**
- Start with Phase 1 & 2 to validate context flow end-to-end
- Phase 3 can begin in parallel once Phase 1 is done (graph service is independent)
- Implement tools incrementally (start with query_graph, then query_element)
- Use feature flags to test in production gradually
- Coordinate with auth team to expose `role` and `permissions` claims for `context.user`

**Architecture Principles Applied:**
- ✅ Inference stateless (no database connection)
- ✅ Context enriched once per thread (not per message)
- ✅ Domain logic in graph service (not inference)
- ✅ Minimal context envelope (tenant/user/repo IDs only)
- ✅ Incremental implementation (validate flow before expanding)

**Performance Considerations:**
- Monitor graph service query performance
- Add database indexes as needed (especially for search and graph traversal)
- Consider caching for frequently accessed repositories
- Thread context enrichment happens once (amortized cost)

**Future Expansion:**
- User/role context can be added to context envelope later
- Selection context can be refined as diagram editors mature
- Role-based prompts can be layered on top of existing infrastructure
- All expansion follows same pattern: UI → context → enrichment → inference

**Alignment with 303-elements.md:**
- ✅ No separate artifacts table - everything is an element
- ✅ Context uses `element_kind` enum (`ARCHIMATE_VIEW`, `DOCUMENT_MD`, etc.)
- ✅ Queries use `elements` table with `title`, `element_kind`, `type_key` fields
- ✅ Supports both primary element (being viewed) and selection (within element)
- ✅ URL pattern: `/org/[orgId]/repo/[repoId]/element/[elementId]`
