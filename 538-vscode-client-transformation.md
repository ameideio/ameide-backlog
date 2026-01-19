# 538 — VSCode Client for Transformation (IDE-Based Agentic Access)

**Status:** Draft (platform-side MCP adapter exists; VSCode client not implemented)
**Parent:** [527-transformation-capability.md](527-transformation-capability.md)
**Audience:** Platform engineering, UX/UI, agent teams
**Scope:** Define a VSCode extension as an **Application Component (client)** that bridges platform architecture context to user-subscribed AI tools (Claude Code, Copilot, Cursor, etc.).

**Use with:**
- Authentication spec: `backlog/538-vscode-client-auth.md`
- Extension implementation: `backlog/538-vscode-client-extension.md`
- MCP adapter pattern: `backlog/534-mcp-protocol-adapter.md`
- Read optimizations (semantic search): `backlog/535-mcp-read-optimizations.md`
- Write optimizations (agent-friendly commands): `backlog/536-mcp-write-optimizations.md`
- **Proto contract notes (v6)**: `backlog/527-transformation-proto-v6.md`
- UISurface posture (v6): `backlog/527-transformation-uisurface-v6.md`
- Primitives stack: `backlog/520-primitives-stack-v6.md`

---

## Implementation progress (current)

Repo status (today):
- [x] Platform-side Transformation MCP adapter primitive exists: `primitives/integration/transformation-mcp-adapter` (passes `ameide primitive verify --kind integration --name transformation-mcp-adapter --mode repo`, with warnings about missing transport/security tests).
- [x] Type-safe client SDKs exist (e.g., TS SDK under `packages/ameide_sdk_ts`) suitable for extension UI query calls.
- [ ] No VSCode extension package/repo is implemented in-tree yet (no `packages/*vscode*`, no extension build pipeline, no marketplace packaging).

Next milestones (checklist):
- [ ] Create a dedicated extension package/repo layout (where it lives, how it versions, how it releases).
- [ ] Implement auth flow for the extension UI (PKCE + SecretStorage) per `backlog/538-vscode-client-auth.md`.
- [ ] Implement first read-only vertical slice: browse elements + views via Projection query services, with caching + pagination.
- [ ] Decide whether to ship an optional local MCP server for `vscode.*` context tools in v0 (and add conformance tests if yes).

## Clarification requests (next steps)

Decide/confirm:
- [ ] Where the extension code lives (monorepo `packages/` vs dedicated `ameide-vscode` repo) and what the release process is (CI, signing, marketplace publishing).
- [ ] The default “platform URL” per environment (dev/staging/prod) and how the extension discovers it (settings vs well-known discovery).
- [ ] Whether the extension ever calls the platform MCP adapter (tooling) or strictly uses typed SDKs for UI and leaves MCP to external AI tools.
- [ ] What the minimum supported client set is for v0 (VS Code stable only vs include `github.dev`/Codespaces/Gitpod).

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (client), Application Interface (MCP relay, Query Services), User Interface.
- **Component classification:** **Application Component (client)** — the VSCode extension is a *developer client application* that consumes Transformation Projection query services and Domain command services. It is NOT a UISurface primitive under 520 (which implies CRD/operator/GitOps deployment).
- **Out-of-scope layers:** Strategy/Business (capability design), Technology (deployment specifics beyond extension packaging).

> **Classification rationale:** The 520 primitives stack defines UISurface primitives as cluster-deployed, operator-managed runtimes (Next.js, CRD-backed). A VSCode extension is a developer workstation client — it *consumes* capability services but is not part of the six-primitive runtime set. This keeps the primitive taxonomy strict and avoids drift.

---

## Architecture Decisions (ADRs)

| ADR | Decision | Rationale |
|-----|----------|-----------|
| **ADR-01** | **Extension UI uses SDK, not MCP** | MCP is for AI tool-calling; extension UI benefits from typed SDK with caching, pagination, and streaming. See 538-vscode-client-extension.md. |
| **ADR-02** | **No token brokering** | Each MCP client handles its own OAuth. Extension does NOT proxy tokens. Security boundary stays clear. |
| **ADR-03** | **Direct remote OAuth for AI tools** | AI tools connect to platform MCP adapter directly via Streamable HTTP + OAuth 2.1. No relay/proxy architecture. Industry-aligned (GitHub, Port, Spotify). |
| **ADR-04** | **Local MCP server is optional, vscode.* tools only** | Optional local server provides IDE context (files, selection, pinned elements). Does NOT proxy platform tools. |
| **ADR-05** | **Not a UISurface primitive** | Per 520, this is an Application Component (client), not an operator-deployed primitive. Lives under `packages/`, not `primitives/`. |

---

## 0) Problem

**API costs are prohibitive** for heavy AI-assisted architecture work:
- Platform agents (AmeidePO, AmeideSA, AmeideCoder) incur API rates that users won't pay for exploratory/interactive work
- Users already have subscriptions to Claude Code, Copilot, Cursor, etc.
- These tools lack access to platform context (ArchiMate models, capability definitions, transformation state)

**Result:** Users either pay API rates or work without architecture context — neither is acceptable.

---

## 1) Solution

**Industry-aligned architecture:** Direct remote HTTP MCP connection (like GitHub, Port, Spotify).

### 1.1 Platform MCP adapter (Integration primitive)

The platform exposes Transformation tools/resources via **MCP over Streamable HTTP** with **OAuth 2.1** (per 534):
- Deployed as a cluster service (`transformation-mcp-adapter`)
- Implements MCP Authorization spec (OAuth 2.1 Resource Server)
- AI tools (Claude Code, VS Code, Cursor) connect directly and handle OAuth themselves
- Routes commands → Domain, queries → Projection

### 1.2 VSCode extension (Application Component)

The extension is a **UI client**, NOT an MCP server or token broker:
- Provides visual browser for Transformation artifacts (tree views, search, webviews)
- Consumes Projection Query Services directly (gRPC/HTTP via TS SDK)
- Does NOT manage OAuth for external tools (each MCP client handles its own auth)
- Optionally exposes local MCP server for `vscode.*` context tools only

```
┌─────────────────────────────────────────────────────────────────────┐
│  AI Tools (Claude Code, Cursor, VS Code Copilot)                    │
│                                                                     │
│  ┌─────────────────────┐    ┌─────────────────────────────────┐    │
│  │ MCP Client          │    │ MCP Client                      │    │
│  │ (remote HTTP)       │    │ (local stdio, optional)         │    │
│  └──────────┬──────────┘    └──────────┬──────────────────────┘    │
│             │                          │                            │
│             │ Streamable HTTP          │ stdio                      │
│             │ + OAuth 2.1              │                            │
└─────────────┼──────────────────────────┼────────────────────────────┘
              │                          │
              ▼                          ▼
┌─────────────────────────────┐  ┌─────────────────────────────────┐
│  Platform MCP Adapter       │  │  VS Code Extension              │
│  (cluster service)          │  │  (Application Component)        │
│                             │  │                                 │
│  transformation.*           │  │  • UI: Element browser, search  │
│  tools/resources            │  │  • Optional: vscode.* MCP tools │
│                             │  │                                 │
│  OAuth 2.1 Resource Server  │  │  Consumes Query Services        │
│  Keycloak = Auth Server     │  │  directly (not via MCP)         │
│                             │  │                                 │
│  primitives/integration/    │  │  packages/ameide_vscode_ext     │
│  transformation-mcp-adapter │  └─────────────────────────────────┘
└──────────────┬──────────────┘
               │
     ┌─────────┴─────────┐
     ▼                   ▼
┌────────────────┐ ┌────────────────┐
│  Projection    │ │  Domain        │
│  (query)       │ │  (write)       │
└────────────────┘ └────────────────┘
```

**Industry alignment:**
| Vendor | Architecture |
|--------|--------------|
| **GitHub** | Remote MCP at `https://api.github.com/mcp` |
| **Port** | Remote MCP at `https://mcp.port.io/v1` |
| **Claude Code** | Direct HTTP connection, OAuth via `/mcp` command |
| **VS Code** | Built-in OAuth support for remote MCP servers |

**Key insight:** All major vendors use **direct remote HTTP connections** — no local proxy binary needed. MCP clients handle OAuth themselves.

---

## 2) Value proposition

| Aspect | Platform Agents (API-billed) | VSCode Client (User Sub) |
|--------|------------------------------|--------------------------|
| **LLM Cost** | Platform pays API rates | User's existing subscription |
| **Context** | Full platform context | Full platform context (via Query Services + MCP) |
| **Governance** | AgentDefinition + service account | User auth + role permissions |
| **Use Case** | Automated workflows, headless | Interactive development, exploration |
| **Triggers** | Facts, schedules, signals | Human conversation |

**Key insight:** The platform's value is the **structured context** (ArchiMate model, capability definitions, transformation state), not the LLM. Users bring their own LLM subscription; the platform provides the intelligence.

---

## 3) Architecture

### 3.1 Extension as Application Component

The VSCode extension is an **Application Component (client)** (TypeScript):
- Presents Transformation data via VSCode UI (tree views, webviews, panels)
- Does NOT embed its own LLM — relies on user's Claude Code / Copilot / Cursor
- UI panels consume **Projection Query Services** directly (gRPC/HTTP via TS SDK)
- Does NOT broker OAuth tokens for external tools (each MCP client handles its own auth)

**Critical:** The extension does NOT manage authentication for AI tools. Each MCP client (Claude Code, VS Code Copilot, Cursor) handles OAuth with the platform MCP adapter directly.

### 3.2 Platform MCP adapter as primary surface

AI tools connect directly to the platform MCP adapter via **Streamable HTTP**:

```
Claude Code ────── Streamable HTTP + OAuth 2.1 ──────► Platform MCP Adapter
                                                              │
                                                              ├── Validates token against Keycloak
                                                              ├── Routes queries → Projection
                                                              ├── Routes commands → Domain
                                                              └── Returns responses
```

**Why direct remote connection (no local proxy)?**
1. **Industry standard** — GitHub, Port, Spotify all use remote HTTP MCP servers
2. **Simpler UX** — No binary to install, no token file management
3. **Better security** — No tokens written to disk, MCP clients handle OAuth directly
4. **Works everywhere** — Same configuration for VS Code, Cursor, Claude Code

### 3.3 Tool surface (exposed by platform MCP adapter)

**Proto contract mapping (see [527-transformation-proto.md](527-transformation-proto.md)):**

| MCP Tool | Proto Service | Intent/Query Message |
|----------|---------------|----------------------|
| `transformation.listElements` | `ArchiMateQueryService.ListElements` | Query RPC |
| `transformation.getView` | `ArchiMateQueryService.GetView` | Query RPC |
| `transformation.semanticSearch` | `SemanticQueryService.Search` | Query RPC (see [535](535-mcp-read-optimizations.md)) |
| `transformation.submitArchitectureIntent` | `TransformationArchitectureWriteService.SubmitIntent` | `TransformationArchitectureDomainIntent` |
| `transformation.createBaseline` | `BaselineWriteService.CreateBaseline` | Intent via `transformation.domain.intents.v1` |

**Write optimization patterns (see [536-mcp-write-optimizations.md](536-mcp-write-optimizations.md)):**
- `validateWrite` — dry-run validation before commit
- `client_request_id` — idempotency key
- Simplified inputs (no UUIDs, no versioning)

**Platform tools (via platform MCP adapter):**
```
transformation.listElements      → Platform Projection
transformation.getView           → Platform Projection
transformation.semanticSearch    → Platform Projection
transformation.submitArchitectureIntent → Platform Domain
transformation.createBaseline    → Platform Domain
```

**IDE tools (optional, via local extension MCP server):**
```
vscode.getOpenFiles             → List open editor tabs
vscode.getSelection             → Get current selection
vscode.getWorkspaceContext      → Active workspace, folder, git branch
vscode.getPinnedContext         → Pinned architecture elements
```

**Note:** `vscode.*` tools are served by the VS Code extension's local MCP server (optional). Users can configure their AI tool with both the remote platform server and the local VS Code server.

---

## 4) Authentication flow

> **Full specification:** See `backlog/538-vscode-client-auth.md` for Keycloak OIDC client configuration.

### 4.1 MCP clients handle OAuth directly (primary)

Per MCP Authorization spec, **MCP clients handle OAuth themselves**. No token brokering needed:

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. User configures MCP server URL in their AI tool              │
│    └── e.g., VS Code mcp.json, Claude Code settings, Cursor     │
│ 2. AI tool discovers OAuth metadata from MCP adapter            │
│    └── GET https://mcp.transformation.ameide.io/.well-known/... │
│ 3. AI tool runs OAuth flow (browser redirect with PKCE)         │
│    └── User authenticates to Keycloak                           │
│ 4. AI tool stores/refreshes tokens automatically                │
│ 5. AI tool calls MCP adapter with Authorization: Bearer <token> │
└─────────────────────────────────────────────────────────────────┘
```

**Per-client auth support:**

| Client | OAuth Support | Configuration |
|--------|---------------|---------------|
| **VS Code (Copilot)** | Yes (DCR or client credentials) | `.vscode/mcp.json` |
| **Claude Code** | Yes (`/mcp` command) | `~/.claude.json` or `.mcp.json` |
| **Cursor** | Env var fallback | `${AMEIDE_TOKEN}` header |

### 4.2 VS Code extension auth (for UI only)

The VS Code extension authenticates **for its own UI** (tree views, search), NOT for external AI tools:

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. User opens VSCode, extension activates                       │
│ 2. Extension checks for stored token (SecretStorage)            │
│    ├── Token exists and valid → Use it                          │
│    └── No token / expired → Prompt OAuth login                  │
│ 3. OAuth flow (browser redirect with PKCE)                      │
│    └── User authenticates to Keycloak                           │
│ 4. Extension stores token in SecretStorage                      │
│ 5. Extension uses token to call Query Services directly         │
│    └── NOT shared with external AI tools                        │
└─────────────────────────────────────────────────────────────────┘
```

**Key requirements:**
- **Public client with PKCE** — `ameide-vscode` client (native apps cannot store secrets)
- **Redirect URIs** — See 538-vscode-client-auth.md
- **Required scopes** — profile, email, roles, groups, organizations, tenant
- **Token storage** — VSCode's `SecretStorage` (OS-level encryption)
- **Status bar** — Shows auth state (logged in / expired)

### 4.3 Keycloak redirect URLs for MCP clients

Per vendor documentation, Keycloak must allow these redirect URLs:

| Client | Required Redirect URLs |
|--------|------------------------|
| **VS Code** | `http://127.0.0.1:33418`, `https://vscode.dev/redirect` |
| **VS Code Extension** | `vscode://ameide.transformation/auth/callback`, `http://127.0.0.1:3000/auth/callback` |
| **Claude Code** | Per Anthropic OAuth spec (TBD) |

---

## 5) UI components

### 5.1 Element browser (tree view)

```
TRANSFORMATION
├── Models
│   └── platform-architecture
│       ├── Strategy
│       │   ├── Capabilities (12)
│       │   └── Value Streams (4)
│       ├── Business
│       │   └── Processes (8)
│       ├── Application
│       │   ├── Components (15)
│       │   └── Services (22)
│       └── Technology
│           └── Nodes (6)
├── Views
│   ├── Capability Map
│   ├── Application Landscape
│   └── Integration Overview
└── Baselines
    ├── 2024-Q4 (promoted)
    └── 2025-Q1-draft
```

**Interactions:**
- Click element → Show detail panel (properties, relationships)
- Right-click → "Add to context" (pins element for AI tools)
- Drag to editor → Insert element reference

### 5.2 Context panel

Shows currently pinned context for AI tools:

```
CONTEXT (3 items)
├── Orders Domain [ApplicationComponent]
├── Capability Map [View]
└── 2024-Q4 Baseline [Baseline]

[Clear All]  [Copy as MCP Resources]
```

AI tools can access pinned context via `vscode.getPinnedContext` tool.

### 5.3 Semantic search panel

```
┌─────────────────────────────────────────────────┐
│ 🔍 Search architecture...                       │
├─────────────────────────────────────────────────┤
│ Query: "order processing capabilities"          │
│ Baseline: 2024-Q4 (rev: abc123)                 │
│                                                 │
│ Results:                                        │
│ ├── Orders Capability (0.92) [Capability]       │
│ │   rev: abc123 | baseline: 2024-Q4             │
│ ├── Order Processing [Process]                  │
│ │   rev: abc123 | baseline: 2024-Q4             │
│ ├── Orders Domain [ApplicationComponent]        │
│ │   rev: def456 | baseline: 2024-Q4             │
│ └── Order Service [ApplicationService]          │
│     rev: def456 | baseline: 2024-Q4             │
└─────────────────────────────────────────────────┘
```

Backed by `transformation.semanticSearch` (535).

**Citation discipline:** All search results include:
- `repository_id` — Repository scope
- `element_id` + `version_id` — Immutable citation to canonical truth (303/656)
- `read_context` — The effective selector (published/head/baseline/as_of)

This enables AI tools to cite specific versions and supports audit trails.

---

## 6) MCP integration

### 6.1 Direct remote connection (primary)

Users configure their AI tool to connect directly to the platform MCP adapter. **No local binary needed.**

### 6.2 Claude Code configuration

```bash
# Add remote MCP server (OAuth mode - best UX)
claude mcp add transformation --url https://mcp.transformation.ameide.io

# Or with explicit Bearer token (fallback for CI/scripts)
claude mcp add transformation \
  --transport http \
  --url https://mcp.transformation.ameide.io \
  --header "Authorization: Bearer ${AMEIDE_TOKEN}"
```

**Resulting configuration (`~/.claude.json` or `.mcp.json`):**
```json
{
  "mcpServers": {
    "transformation": {
      "type": "http",
      "url": "https://mcp.transformation.ameide.io"
    }
  }
}
```

Claude Code handles OAuth via the `/mcp` command when the server requires authentication.

### 6.3 VS Code MCP configuration

For VS Code's native MCP support (Copilot Agent Mode):

```json
// .vscode/mcp.json
{
  "servers": {
    "transformation": {
      "type": "http",
      "url": "https://mcp.transformation.ameide.io"
    }
  }
}
```

VS Code handles OAuth automatically (DCR or client credentials).

### 6.4 Optional: VS Code extension local MCP server

For `vscode.*` tools (IDE context), the extension can expose a **local** MCP server:

```json
// .vscode/mcp.json (multi-server config)
{
  "servers": {
    "transformation": {
      "type": "http",
      "url": "https://mcp.transformation.ameide.io"
    },
    "vscode-context": {
      "type": "stdio",
      "command": "node",
      "args": ["${userHome}/.vscode/extensions/ameide.transformation/dist/mcp-server.js"]
    }
  }
}
```

The local server provides only `vscode.*` tools — all platform tools come from the remote server.

### 6.5 Cursor configuration

```json
// Cursor MCP config (env var for auth)
{
  "mcpServers": {
    "transformation": {
      "url": "https://mcp.transformation.ameide.io",
      "headers": {
        "Authorization": "Bearer ${AMEIDE_TOKEN}"
      }
    }
  }
}
```

### 6.6 Cline configuration (compatibility note)

**⚠️ Cline users:** Cline has [documented issues with Streamable HTTP transport](https://github.com/cline/cline/issues/6767). Until resolved, Cline users have two options:

**Option 1: Use stdio with local proxy (if available)**
```json
{
  "mcpServers": {
    "transformation": {
      "type": "stdio",
      "command": "ameide-mcp",
      "args": ["--capability", "transformation"]
    }
  }
}
```

**Option 2: Wait for Cline fix or use alternative client**

The platform prioritizes Streamable HTTP per MCP spec 2025-11-25. Claude Code and VS Code Copilot work reliably with remote HTTP connections.

See `534-mcp-protocol-adapter.md` Section 2.7 for full client transport compatibility matrix.

---

## 7) Phased implementation

### Phase 1: Read-only browser + context injection

**Scope:**
- Element/view browser (tree view)
- OAuth authentication flow
- MCP server exposing query tools only:
  - `transformation.listElements`
  - `transformation.getView`
  - `transformation.listRelationships`
  - `transformation.semanticSearch`
- Context pinning for AI tools
- VSCode tools: `vscode.getOpenFiles`, `vscode.getPinnedContext`

**Acceptance:**
- User can browse ArchiMate model in VSCode
- Claude Code can query platform context via MCP
- No write operations

### Phase 2: Write with Process-enforced approval

**Scope:**
- Command tools submit to Domain with `proposed_by: "claude-code"` metadata
- Domain creates **ArchitectureProposal** (Process primitive state machine):
  - State: `draft` → `pending_review` → `approved` / `rejected`
  - Approval transitions governed by Process primitive, NOT client UI
- Extension UI shows pending proposals and approval status (read-only display)
- Baseline creation/promotion follows same Process governance

**Architecture:**
```
Claude Code → submitArchitectureIntent → Domain
                                           │
                                           ▼
                              ArchitectureProposal (Process)
                              state: pending_review
                                           │
                                           ▼
                              Architect reviews in Web Portal
                              (or any authorized UI)
                                           │
                                           ▼
                              approveProposal / rejectProposal
                              (Domain command with role check)
```

**Acceptance:**
- User can propose changes via Claude Code
- Proposals enter Process-governed state machine (not client-side approval)
- Approval requires authorized role via any authenticated client
- Audit trail shows "proposed by claude-code (user@example.com), approved by architect@example.com"

### Phase 3: Governed autonomous

**Scope:**
- Direct write for users with appropriate roles (Architect)
- Read-only for users without write roles (Analyst)
- Tool access governed by user's platform permissions

**Acceptance:**
- Architect via Claude Code can write what Architect can write in browser UI
- Analyst via Claude Code can read what Analyst can read in browser UI
- Permission denied errors surface clearly in Claude Code

---

## 8) Non-goals

| What | Why |
|------|-----|
| Embedding LLM in extension | Users bring their own subscription |
| Replacing web portal | Different UISurface, same backend |
| Offline-first architecture | Platform is source of truth; caching is optimization only |
| Multi-capability aggregation | One extension per capability (may compose later) |

---

## 9) Dependencies

| Dependency | Purpose |
|------------|---------|
| **transformation-mcp-adapter** | Platform MCP endpoint (534) |
| **Projection** | Query services for elements, views, search |
| **Domain** | Write services for intents |
| **Keycloak** | OAuth/OIDC authentication |
| **Claude Code / Copilot** | AI tools that consume MCP |

---

## 10) Acceptance criteria

1. VSCode extension is classified as **Application Component (client)**, NOT a primitive.
2. Extension UI consumes **Projection Query Services** directly (not MCP).
3. **Platform MCP adapter** is the primary MCP surface (Streamable HTTP + OAuth 2.1).
4. AI tools (Claude Code, VS Code, Cursor) connect **directly** to platform MCP adapter — no local proxy binary.
5. MCP clients handle OAuth themselves per MCP Authorization spec.
6. Extension manages its own OAuth for UI only — does NOT broker tokens for external tools.
7. Extension optionally exposes local MCP server for `vscode.*` tools only.
8. Phase 1 delivers read-only access; writes require Phase 2 with Process governance.
9. Tool access respects user's platform role permissions.
10. All query results include citation metadata (repository_id, element_id, version_id, read_context).
11. Write operations route through Process primitives; no client-side approval gates.

---

## 11) Extension packaging

### 11.1 VS Code extension

```
packages/
  ameide_vscode_extension/
    package.json              # VSCode extension manifest
    src/
      extension.ts            # Activation, lifecycle
      auth/
        oauth.ts              # OAuth flow (for extension UI only)
        tokenStorage.ts       # SecretStorage wrapper
      platform/
        client.ts             # Query Services client (SDK)
      ui/
        elementBrowser.ts     # Tree view provider
        contextPanel.ts       # Pinned context panel
        searchPanel.ts        # Semantic search webview
      mcp/
        server.ts             # Optional local MCP server for vscode.* tools
        tools/
          vscode.ts           # vscode.* tool implementations
    test/
      auth.test.ts            # OAuth flow tests
      mcp.test.ts             # Local MCP server tests
```

### 11.2 Platform MCP adapter (Integration primitive)

Located at `primitives/integration/transformation-mcp-adapter/` — scaffolded via:

```bash
ameide scaffold integration mcp-adapter --capability transformation
```

See `backlog/534-mcp-protocol-adapter.md` for implementation details.

---

## 12) Example usage

### 12.1 Setup flow

```
1. User adds MCP server to Claude Code:
   $ claude mcp add transformation --url https://mcp.transformation.ameide.io

2. Claude Code prompts for authentication (OAuth flow)
   → User logs in via browser → Keycloak
   → Claude Code stores/refreshes tokens automatically

3. Claude Code now has access to transformation.* tools
```

No binary installation, no token file management, no VS Code extension required for MCP access.

### 12.2 Query flow

```
User in Claude Code: "What capabilities does the platform have?"

Claude Code: [calls transformation.listElements via Streamable HTTP]
             [platform MCP adapter validates OAuth token]
             [platform MCP adapter returns capability list]

"The platform has 12 capabilities including Orders, Fulfillment,
Inventory, and Transformation. Would you like details on any of these?"

User: "Show me the Orders capability and its relationships"

Claude Code: [invokes transformation.listRelationships]
             "Orders capability realizes 3 value streams and is realized by
             the Orders Domain application component..."
```

### 12.3 Write flow (with Process governance)

```
User: "I want to add a new Fulfillment capability"

Claude Code: [invokes transformation.submitArchitectureIntent]
             [Platform Domain creates ArchitectureProposal in pending_review state]

"I've submitted a proposal to create the Fulfillment capability.
The proposal is now pending review (ID: prop-abc123).
An authorized architect will need to approve it via the web portal
or any authorized client."

[Later, architect reviews in web portal and approves]

User: "What's the status of my Fulfillment proposal?"

Claude Code: [invokes transformation.getProposal]
             "The Fulfillment capability proposal has been approved
             and is now visible in the Capability Map view."
```

### 12.4 VS Code extension usage (optional, for UI)

```
1. User installs Ameide extension in VS Code
2. User runs "Ameide: Login" → OAuth flow → token stored in SecretStorage
3. Extension UI shows tree view of architecture elements
4. User can browse, search, pin elements for context
5. Pinned context available via vscode.getPinnedContext tool (if local MCP server enabled)
```

The VS Code extension is purely for UI — it does NOT manage authentication for Claude Code or other AI tools.

---

## 13) Security & data handling

### 13.1 Token security

| Concern | Mitigation |
|---------|------------|
| Token storage | VSCode `SecretStorage` (OS-level encryption) |
| Token transmission | TLS only; Bearer token in Authorization header |
| Token refresh | Automatic refresh before expiry; no user interaction |
| Token revocation | Logout clears local tokens; server-side revocation via Keycloak |

### 13.2 Data classification

| Data | Classification | Handling |
|------|----------------|----------|
| Architecture elements | Platform-confidential | Fetched on-demand, not persisted locally |
| Search results | Platform-confidential | Displayed only, not cached to disk |
| Pinned context | Session-scoped | In-memory only, cleared on extension deactivate |
| User preferences | User-scoped | VSCode settings (non-sensitive) |

### 13.3 AI tool data flow

```
┌─────────────────────────────────────────────────────────────────────┐
│ Data flows through AI tools (Claude Code, Copilot, Cursor):         │
│                                                                     │
│ 1. Extension → AI tool: Platform data (elements, relationships)     │
│    - Subject to AI provider's data policies                         │
│    - User is responsible for their AI subscription terms            │
│                                                                     │
│ 2. AI tool → Extension: Tool calls (queries, commands)              │
│    - Authenticated with user's platform token                       │
│    - Audited in platform (user, tool, timestamp)                    │
│                                                                     │
│ IMPORTANT: Platform data transmitted to user's AI subscription      │
│ is governed by that subscription's terms, not platform terms.       │
└─────────────────────────────────────────────────────────────────────┘
```

### 13.4 Audit trail

All MCP tool calls are logged:
- `user_id` — Authenticated user
- `tool_name` — Tool invoked
- `client_type` — "vscode-extension"
- `ai_client` — "claude-code" / "copilot" / "cursor" (if detectable)
- `timestamp` — ISO 8601
- `request_id` — Correlation ID

---

## 14) Future considerations

- **Multi-capability extension**: Single extension exposing multiple capability MCP adapters
- **Collaborative context**: Share pinned context with team members
- **View rendering**: Render ArchiMate views as interactive diagrams in VSCode webview
- **Git integration**: Link architecture elements to code locations in workspace
