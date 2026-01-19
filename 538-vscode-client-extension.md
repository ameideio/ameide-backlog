# 538 — VSCode Client Extension Implementation

**Status:** Draft (no repo implementation yet)
**Parent:** [538-vscode-client-transformation.md](538-vscode-client-transformation.md)
**Audience:** Extension developers, platform engineering
**Scope:** VSCode extension implementation details — APIs, lifecycle, UI components, optional local MCP server for `vscode.*` tools.

**Use with:**
- Authentication spec: `backlog/538-vscode-client-auth.md`
- Parent spec: `backlog/538-vscode-client-transformation.md`
- MCP adapter: `backlog/534-mcp-protocol-adapter.md`

**Architecture note:** The VS Code extension is a **UI client** (Application Component) that consumes Query Services directly. It does NOT broker authentication for external AI tools. It optionally exposes a local MCP server for `vscode.*` tools only (IDE context). Per `backlog/520-primitives-stack-v6.md`, this is a client application, **not a primitive** — primitives are server-side components deployed via operators.

**Industry alignment:** AI tools (Claude Code, VS Code Copilot, Cursor) connect directly to the platform MCP adapter via Streamable HTTP with OAuth 2.1. No local proxy binary needed.

---

## Implementation progress (current)

Repo status (today):
- [x] This doc defines a target extension architecture (typed SDK for UI; optional local MCP server for `vscode.*` only).
- [ ] No VSCode extension implementation detected yet (no `package.json` extension manifest, no `src/extension.ts`, no packaging pipeline).

Build-out checklist:
- [ ] Decide repo/package location for the extension (monorepo vs dedicated `ameide-vscode` repo) and publishing target (Marketplace / OpenVSX / internal).
- [ ] Implement manifest + activation events + configuration (platform URL, realm URL, cache TTL, enable local MCP).
- [ ] Implement auth UX (sign-in/out) using the referenced auth spec (`backlog/538-vscode-client-auth.md`) and store tokens securely.
- [ ] Implement minimal UI surfaces (tree views for elements/views/baselines; details panel; search; pin/unpin context).
- [ ] Implement caching + pagination + retry strategy for QueryService calls (avoid chatty graph traversals).
- [ ] Implement optional local MCP server for `vscode.*` tools only (open files, selection, pinned context), explicitly not proxying platform MCP.
- [ ] Add integration tests for auth + basic queries and smoke tests for local MCP server behavior.

## Clarification requests (next steps)

Decide/confirm:
- [ ] What is v1 extension scope: “read-only browser + pin context” vs “also authoring/proposals” (which would require governance/process integration).
- [ ] Canonical platform URLs per environment (dev/staging/prod) and how users select tenant/workspace/model.
- [ ] Token storage approach (SecretStorage vs OS keychain wrappers) and offline/session-expiry UX.
- [ ] Whether the extension should talk to gRPC directly or via a REST/Connect gateway (browser compatibility, TLS, proxies).
- [ ] Whether local MCP server is required for v1, and if so which tool set is in-scope (`vscode.openFile`, `vscode.getSelection`, etc.).

## Architecture Decisions (ADRs)

| ADR | Decision | Rationale |
|-----|----------|-----------|
| **ADR-01** | **Extension UI uses SDK, not MCP** | MCP is for tool-calling AI agents; extension UI is a client that benefits from typed SDK with proper caching, pagination, and error handling. |
| **ADR-02** | **No token brokering** | Each MCP client handles its own OAuth. Extension does NOT proxy tokens for external AI tools. Security boundary remains clear. |
| **ADR-03** | **Local MCP server is optional, `vscode.*` tools only** | Local MCP server provides IDE context (open files, selection, pinned elements) for AI tools that support it. It does NOT proxy platform tools. |
| **ADR-04** | **Direct remote OAuth for AI tools** | AI tools connect to platform MCP adapter directly via Streamable HTTP + OAuth 2.1. No relay/proxy architecture. |

## Implementation progress (repo)

- [x] Server-side extensions execution runtime exists (`services/extensions-runtime`) with an SDK helper (`packages/ameide_sdk_go/extensions_runtime.go`).
- [ ] VSCode extension source tree is not implemented in this repo yet (no extension package, build pipeline, or tests).
- [ ] No local MCP server implementation exists for `vscode.*` tools (IDE context).
- [ ] No VSCode auth integration is implemented (PKCE/login, token storage, refresh, tenant selection).
- [ ] Document has an internal mismatch: ADR-01 says “UI uses SDK, not MCP”, but §4 is “Platform client (MCP over Streamable HTTP)” and the sample manifest description references MCP.

## Clarifications requested (next steps)

- [ ] Decide whether the extension talks to the platform via typed SDK/gRPC or via MCP (pick one for v1), then align §4 and the manifest copy accordingly.
- [ ] Decide where the extension lives (monorepo path + repo split), and the build/test toolchain (pnpm/esbuild/vitest vs something else).
- [ ] Decide whether the optional local MCP server is required for v1 and what the minimum `vscode.*` tool set is (files, selection, diagnostics, terminal, Git).
- [ ] Confirm the authentication approach for VSCode (Keycloak PKCE, device code, or brokered login) and the token storage mechanism.
- [ ] Define caching/offline expectations (TTL defaults, eviction, and “refresh” UX) and whether pinned AI context persists across workspaces.

---

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Component (VSCode extension), Application Interface (Query Services, optional local MCP).
- **Out-of-scope layers:** Technology (packaging/distribution specifics beyond extension manifest).

---

## 1) Extension manifest (package.json)

### 1.1 Core manifest

```json
{
  "name": "ameide-transformation",
  "displayName": "Ameide Transformation",
  "description": "Browse and interact with Ameide platform architecture via MCP",
  "version": "0.1.0",
  "publisher": "ameide",
  "license": "Apache-2.0",
  "repository": {
    "type": "git",
    "url": "https://github.com/ameide/ameide-vscode"
  },
  "engines": {
    "vscode": "^1.85.0"
  },
  "categories": [
    "Other"
  ],
  "keywords": [
    "architecture",
    "archimate",
    "mcp",
    "ai",
    "transformation"
  ],
  "activationEvents": [
    "onStartupFinished"
  ],
  "main": "./dist/extension.js",
  "contributes": {
    "commands": [],
    "viewsContainers": {},
    "views": {},
    "configuration": {},
    "menus": {}
  }
}
```

### 1.2 Commands

```json
{
  "contributes": {
    "commands": [
      {
        "command": "ameide.signIn",
        "title": "Sign In to Ameide",
        "category": "Ameide"
      },
      {
        "command": "ameide.signOut",
        "title": "Sign Out from Ameide",
        "category": "Ameide"
      },
      {
        "command": "ameide.refresh",
        "title": "Refresh",
        "category": "Ameide",
        "icon": "$(refresh)"
      },
      {
        "command": "ameide.pinToContext",
        "title": "Pin to AI Context",
        "category": "Ameide",
        "icon": "$(pin)"
      },
      {
        "command": "ameide.unpinFromContext",
        "title": "Unpin from AI Context",
        "category": "Ameide",
        "icon": "$(pinned)"
      },
      {
        "command": "ameide.clearContext",
        "title": "Clear AI Context",
        "category": "Ameide"
      },
      {
        "command": "ameide.copyAsReference",
        "title": "Copy as Reference",
        "category": "Ameide"
      },
      {
        "command": "ameide.showElement",
        "title": "Show Element Details",
        "category": "Ameide"
      },
      {
        "command": "ameide.search",
        "title": "Search Architecture",
        "category": "Ameide",
        "icon": "$(search)"
      }
    ]
  }
}
```

### 1.3 Views and containers

```json
{
  "contributes": {
    "viewsContainers": {
      "activitybar": [
        {
          "id": "ameide",
          "title": "Ameide",
          "icon": "resources/ameide-icon.svg"
        }
      ]
    },
    "views": {
      "ameide": [
        {
          "id": "ameide.elements",
          "name": "Elements",
          "contextualTitle": "Transformation Elements"
        },
        {
          "id": "ameide.views",
          "name": "Views",
          "contextualTitle": "Architecture Views"
        },
        {
          "id": "ameide.context",
          "name": "AI Context",
          "contextualTitle": "Pinned Context for AI Tools"
        },
        {
          "id": "ameide.baselines",
          "name": "Baselines",
          "contextualTitle": "Promoted Baselines"
        }
      ]
    }
  }
}
```

### 1.4 Configuration

```json
{
  "contributes": {
    "configuration": {
      "title": "Ameide",
      "properties": {
        "ameide.platformUrl": {
          "type": "string",
          "default": "https://mcp.transformation.ameide.io",
          "description": "URL of the Ameide MCP adapter"
        },
        "ameide.authUrl": {
          "type": "string",
          "default": "https://auth.ameide.io/realms/ameide",
          "description": "Keycloak realm URL for authentication"
        },
        "ameide.defaultModel": {
          "type": "string",
          "default": "",
          "description": "Default ArchiMate model ID to load"
        },
        "ameide.localMcpServer.enabled": {
          "type": "boolean",
          "default": false,
          "description": "Enable local MCP server for vscode.* tools (IDE context)"
        },
        "ameide.cache.ttlSeconds": {
          "type": "number",
          "default": 300,
          "description": "Cache TTL for architecture data (seconds)"
        }
      }
    }
  }
}
```

### 1.5 Context menus

```json
{
  "contributes": {
    "menus": {
      "view/title": [
        {
          "command": "ameide.refresh",
          "when": "view =~ /^ameide\\./",
          "group": "navigation"
        },
        {
          "command": "ameide.search",
          "when": "view == ameide.elements",
          "group": "navigation"
        }
      ],
      "view/item/context": [
        {
          "command": "ameide.pinToContext",
          "when": "view =~ /^ameide\\./ && viewItem != pinned",
          "group": "inline"
        },
        {
          "command": "ameide.unpinFromContext",
          "when": "view == ameide.context",
          "group": "inline"
        },
        {
          "command": "ameide.copyAsReference",
          "when": "view =~ /^ameide\\./",
          "group": "9_cutcopypaste"
        },
        {
          "command": "ameide.showElement",
          "when": "view == ameide.elements",
          "group": "navigation"
        }
      ],
      "commandPalette": [
        {
          "command": "ameide.signIn",
          "when": "!ameide.authenticated"
        },
        {
          "command": "ameide.signOut",
          "when": "ameide.authenticated"
        }
      ]
    }
  }
}
```

---

## 2) Extension lifecycle

### 2.1 Activation

```typescript
// src/extension.ts
import * as vscode from 'vscode';
import { AuthProvider } from './auth/provider';
import { PlatformClient } from './platform/client';
import { ElementTreeProvider } from './ui/elementTree';
import { ViewTreeProvider } from './ui/viewTree';
import { ContextTreeProvider } from './ui/contextTree';
import { BaselineTreeProvider } from './ui/baselineTree';
import { StatusBarManager } from './ui/statusBar';
import { LocalMcpServer } from './mcp/server';

let localMcpServer: LocalMcpServer | undefined;

export async function activate(context: vscode.ExtensionContext) {
  // 1. Initialize auth provider (for extension UI only, NOT for external AI tools)
  const authProvider = new AuthProvider(context.secrets);

  // 2. Initialize platform client (Query Services via SDK)
  const platformClient = new PlatformClient(authProvider);

  // 3. Initialize tree data providers
  const elementTree = new ElementTreeProvider(platformClient);
  const viewTree = new ViewTreeProvider(platformClient);
  const contextTree = new ContextTreeProvider();
  const baselineTree = new BaselineTreeProvider(platformClient);

  // 4. Register tree views
  context.subscriptions.push(
    vscode.window.registerTreeDataProvider('ameide.elements', elementTree),
    vscode.window.registerTreeDataProvider('ameide.views', viewTree),
    vscode.window.registerTreeDataProvider('ameide.context', contextTree),
    vscode.window.registerTreeDataProvider('ameide.baselines', baselineTree),
  );

  // 5. Initialize status bar
  const statusBar = new StatusBarManager(authProvider);
  context.subscriptions.push(statusBar);

  // 6. Register commands
  registerCommands(context, authProvider, platformClient, contextTree, elementTree);

  // 7. Optional: Start local MCP server for vscode.* tools
  const config = vscode.workspace.getConfiguration('ameide');
  if (config.get<boolean>('localMcpServer.enabled', false)) {
    localMcpServer = new LocalMcpServer(contextTree);
    await localMcpServer.start();
  }

  // 8. Restore auth state
  await authProvider.restoreSession();

  // 9. Set context for menu visibility
  vscode.commands.executeCommand(
    'setContext',
    'ameide.authenticated',
    authProvider.isAuthenticated
  );
}

export function deactivate() {
  if (localMcpServer) {
    localMcpServer.stop();
  }
}
```

### 2.2 Command registration

```typescript
// src/commands.ts
function registerCommands(
  context: vscode.ExtensionContext,
  authProvider: AuthProvider,
  platformClient: PlatformClient,
  contextTree: ContextTreeProvider,
  elementTree: ElementTreeProvider,
) {
  context.subscriptions.push(
    // Auth commands
    vscode.commands.registerCommand('ameide.signIn', async () => {
      await authProvider.signIn();
      vscode.commands.executeCommand('setContext', 'ameide.authenticated', true);
      elementTree.refresh();
    }),

    vscode.commands.registerCommand('ameide.signOut', async () => {
      await authProvider.signOut();
      vscode.commands.executeCommand('setContext', 'ameide.authenticated', false);
      elementTree.refresh();
    }),

    // Context commands
    vscode.commands.registerCommand('ameide.pinToContext', (item: ElementTreeItem) => {
      contextTree.pin(item.element);
      vscode.window.showInformationMessage(`Pinned "${item.element.name}" to AI context`);
    }),

    vscode.commands.registerCommand('ameide.unpinFromContext', (item: ContextTreeItem) => {
      contextTree.unpin(item.element.id);
    }),

    vscode.commands.registerCommand('ameide.clearContext', () => {
      contextTree.clear();
    }),

    // Navigation commands
    vscode.commands.registerCommand('ameide.showElement', async (item: ElementTreeItem) => {
      const panel = vscode.window.createWebviewPanel(
        'ameide.element',
        item.element.name,
        vscode.ViewColumn.One,
        { enableScripts: true }
      );
      panel.webview.html = await renderElementDetail(item.element, platformClient);
    }),

    vscode.commands.registerCommand('ameide.refresh', () => {
      elementTree.refresh();
    }),

    // Copy command
    vscode.commands.registerCommand('ameide.copyAsReference', (item: ElementTreeItem) => {
      const ref = `transformation://element/${item.element.id}`;
      vscode.env.clipboard.writeText(ref);
      vscode.window.showInformationMessage(`Copied reference: ${ref}`);
    }),

    // Search command
    vscode.commands.registerCommand('ameide.search', async () => {
      const query = await vscode.window.showInputBox({
        prompt: 'Search architecture elements',
        placeHolder: 'e.g., orders, capability, domain',
      });
      if (query) {
        const results = await platformClient.semanticSearch(query);
        // Show results in quick pick or dedicated view
        showSearchResults(results);
      }
    }),
  );
}
```

---

## 3) Tree data providers

### 3.1 Element tree provider

```typescript
// src/ui/elementTree.ts
import * as vscode from 'vscode';
import { PlatformClient, ArchiMateElement, ArchiMateLayer } from '../mcp/platformClient';

export class ElementTreeProvider implements vscode.TreeDataProvider<ElementTreeItem> {
  private _onDidChangeTreeData = new vscode.EventEmitter<ElementTreeItem | undefined>();
  readonly onDidChangeTreeData = this._onDidChangeTreeData.event;

  private cache: Map<string, ArchiMateElement[]> = new Map();

  constructor(private client: PlatformClient) {}

  refresh(): void {
    this.cache.clear();
    this._onDidChangeTreeData.fire(undefined);
  }

  getTreeItem(element: ElementTreeItem): vscode.TreeItem {
    return element;
  }

  async getChildren(element?: ElementTreeItem): Promise<ElementTreeItem[]> {
    if (!this.client.isAuthenticated) {
      return [new ElementTreeItem({
        id: 'signin',
        name: 'Sign in to view elements',
        type: 'info',
        layer: 'none',
      }, vscode.TreeItemCollapsibleState.None)];
    }

    if (!element) {
      // Root: show models
      const models = await this.client.listModels();
      return models.map(m => new ElementTreeItem(
        { id: m.id, name: m.name, type: 'model', layer: 'none' },
        vscode.TreeItemCollapsibleState.Collapsed
      ));
    }

    if (element.element.type === 'model') {
      // Model: show layers
      return Object.values(ArchiMateLayer).map(layer => new ElementTreeItem(
        { id: `${element.element.id}/${layer}`, name: layer, type: 'layer', layer },
        vscode.TreeItemCollapsibleState.Collapsed
      ));
    }

    if (element.element.type === 'layer') {
      // Layer: show element types
      const [modelId] = element.element.id.split('/');
      const elements = await this.getCachedElements(modelId, element.element.layer);

      // Group by type
      const byType = new Map<string, ArchiMateElement[]>();
      for (const el of elements) {
        const list = byType.get(el.type) || [];
        list.push(el);
        byType.set(el.type, list);
      }

      return Array.from(byType.entries()).map(([type, els]) => new ElementTreeItem(
        { id: `${element.element.id}/${type}`, name: `${type} (${els.length})`, type: 'type-group', layer: element.element.layer },
        vscode.TreeItemCollapsibleState.Collapsed
      ));
    }

    if (element.element.type === 'type-group') {
      // Type group: show individual elements
      const [modelId, layer, type] = element.element.id.split('/');
      const elements = await this.getCachedElements(modelId, layer as ArchiMateLayer);
      return elements
        .filter(e => e.type === type)
        .map(e => new ElementTreeItem(e, vscode.TreeItemCollapsibleState.None));
    }

    return [];
  }

  private async getCachedElements(modelId: string, layer: ArchiMateLayer): Promise<ArchiMateElement[]> {
    const key = `${modelId}/${layer}`;
    if (!this.cache.has(key)) {
      const elements = await this.client.listElements(modelId, layer);
      this.cache.set(key, elements);
    }
    return this.cache.get(key)!;
  }
}

export class ElementTreeItem extends vscode.TreeItem {
  constructor(
    public readonly element: ArchiMateElement,
    collapsibleState: vscode.TreeItemCollapsibleState,
  ) {
    super(element.name, collapsibleState);

    this.id = element.id;
    this.tooltip = `${element.type}: ${element.name}`;
    this.description = element.type !== 'model' && element.type !== 'layer' && element.type !== 'type-group'
      ? element.type
      : undefined;

    // Set icon based on layer/type
    this.iconPath = this.getIcon(element);

    // Set context value for menus
    this.contextValue = element.type === 'info' ? 'info' : 'element';
  }

  private getIcon(element: ArchiMateElement): vscode.ThemeIcon {
    switch (element.layer) {
      case 'strategy': return new vscode.ThemeIcon('star');
      case 'business': return new vscode.ThemeIcon('briefcase');
      case 'application': return new vscode.ThemeIcon('server');
      case 'technology': return new vscode.ThemeIcon('database');
      default: return new vscode.ThemeIcon('symbol-class');
    }
  }
}
```

### 3.2 Context tree provider (pinned items)

```typescript
// src/ui/contextTree.ts
import * as vscode from 'vscode';
import { ArchiMateElement } from '../mcp/platformClient';

export class ContextTreeProvider implements vscode.TreeDataProvider<ContextTreeItem> {
  private _onDidChangeTreeData = new vscode.EventEmitter<ContextTreeItem | undefined>();
  readonly onDidChangeTreeData = this._onDidChangeTreeData.event;

  private pinnedElements: Map<string, ArchiMateElement> = new Map();

  pin(element: ArchiMateElement): void {
    this.pinnedElements.set(element.id, element);
    this._onDidChangeTreeData.fire(undefined);
  }

  unpin(elementId: string): void {
    this.pinnedElements.delete(elementId);
    this._onDidChangeTreeData.fire(undefined);
  }

  clear(): void {
    this.pinnedElements.clear();
    this._onDidChangeTreeData.fire(undefined);
  }

  getPinnedElements(): ArchiMateElement[] {
    return Array.from(this.pinnedElements.values());
  }

  getTreeItem(element: ContextTreeItem): vscode.TreeItem {
    return element;
  }

  async getChildren(): Promise<ContextTreeItem[]> {
    if (this.pinnedElements.size === 0) {
      return [new ContextTreeItem({
        id: 'empty',
        name: 'Pin elements to add AI context',
        type: 'info',
        layer: 'none',
      })];
    }

    return Array.from(this.pinnedElements.values()).map(
      el => new ContextTreeItem(el)
    );
  }
}

export class ContextTreeItem extends vscode.TreeItem {
  constructor(public readonly element: ArchiMateElement) {
    super(element.name, vscode.TreeItemCollapsibleState.None);

    this.id = element.id;
    this.tooltip = `${element.type}: ${element.name}`;
    this.description = element.type;
    this.contextValue = element.type === 'info' ? 'info' : 'pinned';
    this.iconPath = new vscode.ThemeIcon('pinned');
  }
}
```

---

## 4) Platform client (MCP over Streamable HTTP)

### 4.1 Client implementation

```typescript
// src/mcp/platformClient.ts
import * as vscode from 'vscode';
import { AuthProvider } from '../auth/provider';

export type ArchiMateLayer = 'strategy' | 'business' | 'application' | 'technology' | 'implementation';

export interface ArchiMateElement {
  id: string;
  name: string;
  type: string;
  layer: ArchiMateLayer | 'none';
  description?: string;
  properties?: Record<string, string>;
}

export interface ArchiMateModel {
  id: string;
  name: string;
  description?: string;
}

export interface SearchResult {
  element: ArchiMateElement;
  score: number;
}

/**
 * PlatformClient uses the Transformation SDK (Connect/gRPC) for extension UI operations.
 *
 * ADR-01: Extension UI uses SDK, not MCP.
 * - MCP is for AI tool-calling agents (Claude Code, Cursor, etc.)
 * - Extension UI benefits from typed SDK with proper caching, pagination, and error handling
 * - SDK provides better TypeScript types and streaming support
 */
export class PlatformClient {
  private queryClient: ArchiMateQueryServiceClient;
  private semanticClient: SemanticQueryServiceClient;
  private cache: Map<string, { data: any; expiry: number }> = new Map();
  private cacheTtl: number;

  constructor(private authProvider: AuthProvider) {
    const config = vscode.workspace.getConfiguration('ameide');
    const baseUrl = config.get<string>('platformUrl', 'https://api.transformation.ameide.io');
    this.cacheTtl = config.get<number>('cache.ttlSeconds', 300) * 1000;

    // Create Connect transport with auth interceptor
    const transport = createConnectTransport({
      baseUrl,
      interceptors: [this.authInterceptor()],
    });

    // Initialize SDK clients (generated from proto)
    this.queryClient = createClient(ArchiMateQueryService, transport);
    this.semanticClient = createClient(SemanticQueryService, transport);
  }

  private authInterceptor(): Interceptor {
    return (next) => async (req) => {
      const token = await this.authProvider.getAccessToken();
      if (token) {
        req.header.set('Authorization', `Bearer ${token}`);
      }
      return next(req);
    };
  }

  get isAuthenticated(): boolean {
    return this.authProvider.isAuthenticated;
  }

  // Cache helper
  private async cached<T>(key: string, fetcher: () => Promise<T>): Promise<T> {
    const cached = this.cache.get(key);
    if (cached && cached.expiry > Date.now()) {
      return cached.data as T;
    }
    const data = await fetcher();
    this.cache.set(key, { data, expiry: Date.now() + this.cacheTtl });
    return data;
  }

  // Public API — uses SDK, not MCP
  async listModels(): Promise<ArchiMateModel[]> {
    return this.cached('models', async () => {
      const response = await this.queryClient.listModels({});
      return response.models;
    });
  }

  async listElements(modelId: string, layer?: ArchiMateLayer): Promise<ArchiMateElement[]> {
    const cacheKey = `elements:${modelId}:${layer || 'all'}`;
    return this.cached(cacheKey, async () => {
      const response = await this.queryClient.listElements({
        modelId,
        filter: layer ? { layer: layerToProto(layer) } : undefined,
      });
      return response.elements.map(protoToElement);
    });
  }

  async getElement(elementId: string): Promise<ArchiMateElement> {
    const cacheKey = `element:${elementId}`;
    return this.cached(cacheKey, async () => {
      const response = await this.queryClient.getElement({ elementId });
      return protoToElement(response.element!);
    });
  }

  async listRelationships(elementId: string): Promise<ArchiMateRelationship[]> {
    const cacheKey = `relationships:${elementId}`;
    return this.cached(cacheKey, async () => {
      const response = await this.queryClient.listRelationships({ elementId });
      return response.relationships;
    });
  }

  async semanticSearch(query: string, limit: number = 10): Promise<SearchResult[]> {
    // Semantic search uses SemanticQueryService (see 535-mcp-read-optimizations.md)
    const response = await this.semanticClient.search({
      query,
      topK: limit,
    });
    return response.results.map(r => ({
      element: protoToElement(r.element!),
      score: r.score,
    }));
  }

  async getView(viewId: string): Promise<ArchiMateView> {
    const cacheKey = `view:${viewId}`;
    return this.cached(cacheKey, async () => {
      const response = await this.queryClient.getView({ viewId });
      return response.view!;
    });
  }

  async listViews(modelId: string): Promise<ArchiMateView[]> {
    const cacheKey = `views:${modelId}`;
    return this.cached(cacheKey, async () => {
      const response = await this.queryClient.listViews({ modelId });
      return response.views;
    });
  }

  async listBaselines(): Promise<Baseline[]> {
    return this.cached('baselines', async () => {
      const response = await this.queryClient.listBaselines({});
      return response.baselines;
    });
  }

  clearCache(): void {
    this.cache.clear();
  }
}

// Helper functions for proto ↔ TypeScript conversion
function layerToProto(layer: ArchiMateLayer): number {
  const layerMap: Record<ArchiMateLayer, number> = {
    strategy: 1,
    business: 2,
    application: 3,
    technology: 4,
    implementation: 5,
  };
  return layerMap[layer] ?? 0;
}

function protoToElement(proto: any): ArchiMateElement {
  return {
    id: proto.id,
    name: proto.name,
    type: proto.type,
    layer: protoToLayer(proto.layer),
    description: proto.description,
    properties: proto.properties,
  };
}

function protoToLayer(layer: number): ArchiMateLayer | 'none' {
  const layerMap: Record<number, ArchiMateLayer | 'none'> = {
    0: 'none',
    1: 'strategy',
    2: 'business',
    3: 'application',
    4: 'technology',
    5: 'implementation',
  };
  return layerMap[layer] ?? 'none';
}
```

---

## 5) Optional local MCP server (for vscode.* tools)

The extension can optionally expose a **local MCP server** for `vscode.*` tools (IDE context). This is **disabled by default** since most users only need the platform MCP adapter.

**When to enable:**
- User wants AI tools to access pinned architecture context from VS Code
- User wants AI tools to see current selection, open files, workspace info

**Architecture:**
```
┌─────────────────────────────────────────────────────────────────────┐
│  AI Tool (Claude Code / VS Code Copilot)                            │
│                                                                     │
│  ┌─────────────────────┐    ┌─────────────────────────────────┐    │
│  │ MCP Client          │    │ MCP Client                      │    │
│  │ (remote HTTP)       │    │ (local stdio)                   │    │
│  └──────────┬──────────┘    └──────────┬──────────────────────┘    │
│             │                          │                            │
│             │ transformation.*         │ vscode.* tools             │
│             │ tools                    │                            │
└─────────────┼──────────────────────────┼────────────────────────────┘
              │                          │
              ▼                          ▼
┌─────────────────────────────┐  ┌─────────────────────────────────┐
│  Platform MCP Adapter       │  │  VS Code Extension              │
│  (cluster)                  │  │  (local MCP server)             │
│                             │  │                                 │
│  OAuth 2.1 + Streamable HTTP│  │  vscode.getSelection            │
│                             │  │  vscode.getPinnedContext        │
└─────────────────────────────┘  │  vscode.getWorkspaceContext     │
                                 └─────────────────────────────────┘
```

### 5.1 Local MCP server implementation

```typescript
// src/mcp/server.ts
import * as vscode from 'vscode';
import { ContextTreeProvider } from '../ui/contextTree';

interface JsonRpcRequest {
  jsonrpc: '2.0';
  id: number | string;
  method: string;
  params?: any;
}

interface JsonRpcResponse {
  jsonrpc: '2.0';
  id: number | string;
  result?: any;
  error?: { code: number; message: string };
}

export class LocalMcpServer {
  private running = false;

  constructor(private contextTree: ContextTreeProvider) {}

  async start(): Promise<void> {
    if (this.running) return;
    this.running = true;

    // stdio MCP server: reads JSON-RPC from stdin, writes to stdout
    process.stdin.on('data', async (data) => {
      try {
        const request: JsonRpcRequest = JSON.parse(data.toString());
        const response = await this.handleRequest(request);
        process.stdout.write(JSON.stringify(response) + '\n');
      } catch (err: any) {
        // Ignore parse errors on stdin
      }
    });
  }

  stop(): void {
    this.running = false;
  }

  private async handleRequest(request: JsonRpcRequest): Promise<JsonRpcResponse> {
    switch (request.method) {
      case 'initialize':
        return this.handleInitialize(request);
      case 'tools/list':
        return this.handleToolsList(request);
      case 'tools/call':
        return this.handleToolsCall(request);
      default:
        return this.error(request.id, -32601, `Unknown method: ${request.method}`);
    }
  }

  private handleInitialize(request: JsonRpcRequest): JsonRpcResponse {
    return {
      jsonrpc: '2.0',
      id: request.id,
      result: {
        protocolVersion: '2025-03-26',  // Current MCP spec version
        capabilities: { tools: {} },
        serverInfo: { name: 'ameide-vscode-context', version: '0.1.0' },
      },
    };
  }

  private handleToolsList(request: JsonRpcRequest): JsonRpcResponse {
    return {
      jsonrpc: '2.0',
      id: request.id,
      result: {
        tools: [
          {
            name: 'vscode.getOpenFiles',
            description: 'List currently open files in VS Code',
            inputSchema: { type: 'object', properties: {} },
          },
          {
            name: 'vscode.getSelection',
            description: 'Get the current text selection in the active editor',
            inputSchema: { type: 'object', properties: {} },
          },
          {
            name: 'vscode.getPinnedContext',
            description: 'Get pinned architecture elements for AI context',
            inputSchema: { type: 'object', properties: {} },
          },
          {
            name: 'vscode.getWorkspaceContext',
            description: 'Get current workspace, folder, and git branch',
            inputSchema: { type: 'object', properties: {} },
          },
        ],
      },
    };
  }

  private async handleToolsCall(request: JsonRpcRequest): Promise<JsonRpcResponse> {
    const { name } = request.params;
    let result: any;

    switch (name) {
      case 'vscode.getOpenFiles':
        result = vscode.window.tabGroups.all
          .flatMap(g => g.tabs)
          .filter(t => t.input instanceof vscode.TabInputText)
          .map(t => (t.input as vscode.TabInputText).uri.fsPath);
        break;

      case 'vscode.getSelection':
        const editor = vscode.window.activeTextEditor;
        result = editor ? {
          file: editor.document.uri.fsPath,
          selection: editor.document.getText(editor.selection),
          range: {
            start: { line: editor.selection.start.line, character: editor.selection.start.character },
            end: { line: editor.selection.end.line, character: editor.selection.end.character },
          },
        } : { file: null, selection: null };
        break;

      case 'vscode.getPinnedContext':
        result = this.contextTree.getPinnedElements();
        break;

      case 'vscode.getWorkspaceContext':
        const workspaceFolder = vscode.workspace.workspaceFolders?.[0];
        result = {
          workspace: workspaceFolder?.name || null,
          folder: workspaceFolder?.uri.fsPath || null,
        };
        break;

      default:
        return this.error(request.id, -32602, `Unknown tool: ${name}`);
    }

    return {
      jsonrpc: '2.0',
      id: request.id,
      result: { content: [{ type: 'text', text: JSON.stringify(result, null, 2) }] },
    };
  }

  private error(id: number | string, code: number, message: string): JsonRpcResponse {
    return { jsonrpc: '2.0', id, error: { code, message } };
  }
}
```

### 5.2 Configuration for multi-server setup

Users who want both platform tools and VS Code context tools configure two MCP servers:

```json
// .vscode/mcp.json
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

**Note:** The VS Code extension must be configured with `"ameide.localMcpServer.enabled": true` for this to work.

---

## 6) Status bar

```typescript
// src/ui/statusBar.ts
import * as vscode from 'vscode';
import { AuthProvider } from '../auth/provider';

export class StatusBarManager implements vscode.Disposable {
  private statusBarItem: vscode.StatusBarItem;
  private disposables: vscode.Disposable[] = [];

  constructor(private authProvider: AuthProvider) {
    this.statusBarItem = vscode.window.createStatusBarItem(
      vscode.StatusBarAlignment.Right,
      100
    );
    this.statusBarItem.command = 'ameide.signIn';
    this.disposables.push(this.statusBarItem);

    // Update on auth changes
    this.authProvider.onDidChangeAuthState(() => this.update());

    this.update();
    this.statusBarItem.show();
  }

  private update(): void {
    if (this.authProvider.isAuthenticated) {
      const user = this.authProvider.currentUser;
      this.statusBarItem.text = `$(check) Ameide: ${user?.email || 'Connected'}`;
      this.statusBarItem.tooltip = 'Click to sign out';
      this.statusBarItem.command = 'ameide.signOut';
      this.statusBarItem.backgroundColor = undefined;
    } else if (this.authProvider.isExpired) {
      this.statusBarItem.text = '$(warning) Ameide: Session expired';
      this.statusBarItem.tooltip = 'Click to sign in again';
      this.statusBarItem.command = 'ameide.signIn';
      this.statusBarItem.backgroundColor = new vscode.ThemeColor('statusBarItem.warningBackground');
    } else {
      this.statusBarItem.text = '$(sign-in) Ameide: Sign in';
      this.statusBarItem.tooltip = 'Click to sign in to Ameide';
      this.statusBarItem.command = 'ameide.signIn';
      this.statusBarItem.backgroundColor = undefined;
    }
  }

  dispose(): void {
    this.disposables.forEach(d => d.dispose());
  }
}
```

---

## 7) Automatic context injection

The extension provides **automatic context** to AI tools without requiring explicit tool calls. This is similar to how Claude Code automatically injects workspace context, git status, and file information into conversations.

### 7.1 Context injection mechanism

MCP supports **context injection** via the `sampling` capability and **system prompt augmentation**. The extension uses two complementary approaches:

1. **MCP Resources** — AI tools can read pinned elements as resources
2. **System prompt injection** — Extension provides context that gets prepended to conversations

```
┌─────────────────────────────────────────────────────────────────────┐
│  AI Tool starts conversation                                        │
│                                                                     │
│  1. AI calls resources/list → Extension returns pinned elements     │
│  2. AI calls resources/read for each → Gets element details         │
│  3. Context is injected into conversation automatically             │
│                                                                     │
│  OR (for tools supporting system prompts):                          │
│                                                                     │
│  1. Extension provides system prompt via initialize response        │
│  2. System prompt includes current architecture context             │
│  3. AI uses context without explicit tool calls                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 Automatic context sources

The extension automatically collects and surfaces these context types:

| Context Type | Source | Injection Method |
|--------------|--------|------------------|
| **Pinned elements** | User pins in tree view | MCP resources + system prompt |
| **Active model** | Last browsed/selected model | System prompt |
| **Workspace mapping** | Git repo → ArchiMate component | System prompt |
| **Recent searches** | Last 5 semantic searches | System prompt |
| **Current baseline** | Active baseline for tenant | System prompt |

### 7.3 System prompt template

The extension constructs a system prompt that AI tools can consume:

```typescript
// src/context/systemPrompt.ts
export function buildSystemPrompt(context: ContextState): string {
  const sections: string[] = [];

  // Architecture context header
  sections.push(`<architecture-context>`);

  // Active model
  if (context.activeModel) {
    sections.push(`## Active ArchiMate Model`);
    sections.push(`Model: ${context.activeModel.name} (${context.activeModel.id})`);
    sections.push(`Description: ${context.activeModel.description || 'N/A'}`);
    sections.push('');
  }

  // Pinned elements
  if (context.pinnedElements.length > 0) {
    sections.push(`## Pinned Architecture Elements (${context.pinnedElements.length})`);
    for (const el of context.pinnedElements) {
      sections.push(`- **${el.name}** [${el.type}] (${el.layer})`);
      if (el.description) {
        sections.push(`  ${el.description}`);
      }
      sections.push(`  ID: ${el.id}`);
    }
    sections.push('');
  }

  // Workspace mapping
  if (context.workspaceMapping) {
    sections.push(`## Workspace → Architecture Mapping`);
    sections.push(`Repository: ${context.workspaceMapping.repoName}`);
    sections.push(`Maps to: ${context.workspaceMapping.component.name} [${context.workspaceMapping.component.type}]`);
    sections.push('');
  }

  // Current baseline
  if (context.currentBaseline) {
    sections.push(`## Current Baseline`);
    sections.push(`Name: ${context.currentBaseline.name}`);
    sections.push(`Status: ${context.currentBaseline.status}`);
    sections.push(`Created: ${context.currentBaseline.createdAt}`);
    sections.push('');
  }

  sections.push(`</architecture-context>`);

  return sections.join('\n');
}
```

### 7.4 MCP initialize response with context

The MCP server includes context in the `initialize` response:

```typescript
private handleInitialize(request: JsonRpcRequest): JsonRpcResponse {
  // Build current context
  const contextState = this.buildContextState();
  const systemPrompt = buildSystemPrompt(contextState);

  return {
    jsonrpc: '2.0',
    id: request.id,
    result: {
      protocolVersion: '2024-11-05',
      capabilities: {
        tools: {},
        resources: {},
        // Indicate we support context injection
        experimental: {
          contextInjection: true,
        },
      },
      serverInfo: {
        name: 'ameide-transformation',
        version: '0.1.0',
      },
      // System prompt for AI tools that support it
      instructions: systemPrompt,
    },
  };
}
```

### 7.5 Dynamic resource listing

Resources are dynamically generated based on pinned context:

```typescript
private handleResourcesList(request: JsonRpcRequest): JsonRpcResponse {
  const resources: McpResource[] = [];

  // Pinned elements as resources
  for (const el of this.contextTree.getPinnedElements()) {
    resources.push({
      uri: `transformation://element/${el.id}`,
      name: el.name,
      description: `${el.type} (${el.layer}) - ${el.description || 'No description'}`,
      mimeType: 'application/json',
    });
  }

  // Active model as resource
  const activeModel = this.getActiveModel();
  if (activeModel) {
    resources.push({
      uri: `transformation://model/${activeModel.id}`,
      name: `[Active Model] ${activeModel.name}`,
      description: activeModel.description || 'Current working model',
      mimeType: 'application/json',
    });
  }

  // Workspace mapping as resource (if exists)
  const workspaceMapping = this.getWorkspaceMapping();
  if (workspaceMapping) {
    resources.push({
      uri: `transformation://element/${workspaceMapping.component.id}`,
      name: `[This Repo] ${workspaceMapping.component.name}`,
      description: `Architecture component for ${workspaceMapping.repoName}`,
      mimeType: 'application/json',
    });
  }

  return {
    jsonrpc: '2.0',
    id: request.id,
    result: { resources },
  };
}
```

### 7.6 Workspace-to-architecture mapping

The extension can automatically map the current VSCode workspace to an ArchiMate component:

```typescript
// src/context/workspaceMapping.ts
export class WorkspaceMapper {
  constructor(private platformClient: PlatformClient) {}

  async findMapping(): Promise<WorkspaceMapping | undefined> {
    const workspaceFolder = vscode.workspace.workspaceFolders?.[0];
    if (!workspaceFolder) return undefined;

    // Try to find git remote
    const gitExtension = vscode.extensions.getExtension('vscode.git')?.exports;
    const git = gitExtension?.getAPI(1);
    const repo = git?.repositories[0];
    const remoteUrl = repo?.state.remotes[0]?.fetchUrl;

    if (!remoteUrl) return undefined;

    // Extract repo name from URL
    const repoName = this.extractRepoName(remoteUrl);

    // Search for matching ArchiMate component
    // Convention: components have property `sourceRepository` = repo URL or name
    const results = await this.platformClient.semanticSearch(
      `repository:${repoName} OR sourceRepository:${repoName}`,
      5
    );

    // Find Application Component with matching repo
    const match = results.find(r =>
      r.element.layer === 'application' &&
      (r.element.type === 'ApplicationComponent' || r.element.type === 'Domain')
    );

    if (match) {
      return {
        repoName,
        remoteUrl,
        component: match.element,
      };
    }

    return undefined;
  }

  private extractRepoName(url: string): string {
    // Handle git@github.com:org/repo.git and https://github.com/org/repo.git
    const match = url.match(/[\/:]([^\/]+\/[^\/]+?)(?:\.git)?$/);
    return match?.[1] || url;
  }
}

export interface WorkspaceMapping {
  repoName: string;
  remoteUrl: string;
  component: ArchiMateElement;
}
```

### 7.7 Context refresh and caching

Context is refreshed on specific triggers:

```typescript
// src/context/contextManager.ts
export class ContextManager {
  private contextState: ContextState;
  private refreshInterval: NodeJS.Timeout | undefined;

  constructor(
    private platformClient: PlatformClient,
    private contextTree: ContextTreeProvider,
    private workspaceMapper: WorkspaceMapper,
  ) {
    this.contextState = this.buildInitialContext();

    // Refresh on pin/unpin
    this.contextTree.onDidChangeTreeData(() => this.refresh());

    // Refresh on workspace change
    vscode.workspace.onDidChangeWorkspaceFolders(() => this.refreshWorkspaceMapping());

    // Periodic refresh (every 5 minutes)
    this.refreshInterval = setInterval(() => this.refresh(), 5 * 60 * 1000);
  }

  async refresh(): Promise<void> {
    this.contextState = {
      pinnedElements: this.contextTree.getPinnedElements(),
      activeModel: await this.getActiveModel(),
      workspaceMapping: await this.workspaceMapper.findMapping(),
      currentBaseline: await this.getCurrentBaseline(),
      recentSearches: this.getRecentSearches(),
      timestamp: Date.now(),
    };
  }

  getContext(): ContextState {
    return this.contextState;
  }

  dispose(): void {
    if (this.refreshInterval) {
      clearInterval(this.refreshInterval);
    }
  }
}
```

### 7.8 Configuration options

Users can control automatic context behavior:

```json
{
  "contributes": {
    "configuration": {
      "properties": {
        "ameide.context.autoInject": {
          "type": "boolean",
          "default": true,
          "description": "Automatically inject architecture context into AI conversations"
        },
        "ameide.context.includeWorkspaceMapping": {
          "type": "boolean",
          "default": true,
          "description": "Include workspace-to-architecture mapping in context"
        },
        "ameide.context.includeRecentSearches": {
          "type": "boolean",
          "default": false,
          "description": "Include recent semantic searches in context"
        },
        "ameide.context.maxPinnedElements": {
          "type": "number",
          "default": 10,
          "description": "Maximum pinned elements to include in context"
        }
      }
    }
  }
}
```

---

## 8) Project structure

```
packages/
  ameide_vscode_extension/
    package.json                    # Extension manifest
    tsconfig.json                   # TypeScript config
    webpack.config.js               # Bundle config
    .vscodeignore                   # Publish ignore list
    resources/
      ameide-icon.svg               # Activity bar icon
    src/
      extension.ts                  # Activation, lifecycle
      commands/
        index.ts                    # Command registration
      auth/
        provider.ts                 # AuthProvider (for UI only)
        oauth.ts                    # OAuth flow implementation
      platform/
        client.ts                   # Platform query services client (SDK)
      mcp/
        server.ts                   # Optional local MCP server for vscode.* tools
      context/
        contextManager.ts           # Context state and refresh
        workspaceMapping.ts         # Git repo → ArchiMate mapping
      ui/
        elementTree.ts              # Element browser tree
        viewTree.ts                 # Views tree
        contextTree.ts              # Pinned context tree
        baselineTree.ts             # Baselines tree
        statusBar.ts                # Status bar manager
        elementDetail.ts            # Element detail webview
        searchResults.ts            # Search results quick pick
    test/
      extension.test.ts             # Extension tests
      mcp.test.ts                   # Local MCP server tests
      context.test.ts               # Context tests
```

---

## 9) Acceptance criteria

1. Extension manifest defines all commands, views, and configuration.
2. Tree views display models, layers, element types, and elements.
3. Context tree allows pinning/unpinning elements for AI context.
4. Platform client communicates with query services via SDK (not MCP).
5. Extension manages its own OAuth for UI only — does NOT broker tokens for external AI tools.
6. Optional local MCP server exposes `vscode.*` tools for IDE context (disabled by default).
7. Status bar shows authentication state.
8. Commands are registered and accessible via command palette.
9. Extension activates on startup and restores auth session.
10. **Workspace mapping** automatically links git repo to ArchiMate component.
11. Context refreshes on pin/unpin and workspace change.

---

## 10) Dependencies

| Dependency | Purpose |
|------------|---------|
| **VSCode Extension API** | All UI and lifecycle |
| **538-auth** | OAuth/OIDC authentication (for extension UI) |
| **Transformation SDK** | Query services client |
| **Platform MCP Adapter** | Remote MCP surface (AI tools connect directly) |
