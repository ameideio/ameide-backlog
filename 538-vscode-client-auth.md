# 538 — VSCode Client Authentication (Keycloak OIDC Integration)

**Status:** Draft (design only; GitOps not implemented)
**Parent:** [538-vscode-client-transformation.md](538-vscode-client-transformation.md)
**Audience:** Platform engineering, identity/auth, extension developers
**Scope:** Define the Keycloak OIDC client configuration and authentication flow for the VSCode client extension.

**Use with:**
- VSCode Client: `backlog/538-vscode-client-transformation.md`
- MCP adapter: `backlog/534-mcp-protocol-adapter.md`
- Keycloak GitOps: `gitops/ameide-gitops/sources/charts/foundation/operators-config/keycloak_realm/values.yaml`
- Keycloak OIDC client reconciliation: `backlog/485-keycloak-oidc-client-reconciliation.md`
- Keycloak OIDC scopes: `backlog/460-keycloak-oidc-scopes.md`

---

## Implementation progress (current)

Repo status (today):
- [x] Authentication design is specified here (public client + PKCE, separate clients per consumer type, no token brokering).
- [ ] No Keycloak GitOps configuration detected for `ameide-vscode` / `ameide-mcp-*` clients yet (no occurrences in `gitops/ameide-gitops/sources/`).
- [ ] No end-to-end auth smoke test exists yet (extension UI login → token storage → successful query call).

Next milestones (checklist):
- [ ] Add Keycloak realm GitOps for `ameide-vscode` public client + redirect URIs (and decide which redirect URIs are allowed in v0).
- [ ] Add Keycloak client scopes `mcp:tools` and `mcp:resources` (or confirm they already exist elsewhere) and assign to the relevant `ameide-mcp-*` clients.
- [ ] Implement “whoami”/token introspection debug path for extension troubleshooting (claims + tenant selection visibility).
- [ ] Add a security review checklist: token lifetimes, offline_access policy, allowed origins, and logout semantics.

## Clarification requests (next steps)

Decide/confirm:
- [ ] Whether MCP clients use DCR (Dynamic Client Registration) or pre-provisioned Keycloak clients (and which clients are required at v0).
- [ ] The canonical claim(s) for tenant selection (`tenantId` vs `tenant_id` vs org/group mapping) and what happens when a user belongs to multiple tenants.
- [ ] Whether `offline_access` is ever allowed for interactive VSCode sessions, or only for device/CLI flows.
- [ ] How role/group claims map to capability permissions for MCP (coarse scope check vs fine-grained tool authorization).

## Layer header (Application / Technology)

- **Primary ArchiMate layer(s):** Application (client), Technology (identity infrastructure).
- **Primary element types used:** Application Interface (OAuth flow), Technology Service (Keycloak IdP).
- **Out-of-scope layers:** Strategy/Business (capability design).

---

## Architecture Decisions (ADRs)

| ADR | Decision | Rationale |
|-----|----------|-----------|
| **ADR-01** | **Extension UI uses SDK, MCP clients use MCP adapter** | Extension UI benefits from typed SDK; AI tools need MCP for tool-calling. Separate auth clients. |
| **ADR-02** | **No token brokering** | Extension does NOT broker tokens for MCP clients. Each authenticates independently. |
| **ADR-03** | **Public client + PKCE S256** | Native apps cannot store secrets; PKCE required per OAuth 2.1 best practice. |
| **ADR-04** | **Separate Keycloak clients per consumer type** | `ameide-vscode` (UI), `ameide-mcp-*` (AI tools). Independent revocation, scope assignment. |
| **ADR-05** | **No offline_access by default** | Short-lived tokens preferred; offline only when explicitly required. |

---

## 0) Problem

The VSCode client extension (538) requires OAuth/OIDC authentication. Two separate authentication concerns exist:

1. **Extension UI** — Authenticates to call Transformation Query Services (SDK/Connect) for browsing/search
2. **External AI tools** — Authenticate to call platform MCP adapter (Claude Code, Cursor, etc.)

**ADR-01 alignment:** Extension UI uses SDK (Connect/gRPC), NOT MCP. Only external AI tools use MCP.

**Current state:**
- Keycloak realm `ameide` is configured with clients for: `platform-app`, `argocd`, `kubernetes-dashboard`
- All existing clients are **confidential** (require client secret)
- No client exists for native/desktop applications using PKCE

**Gap:**
- VSCode extensions are native apps and cannot securely store client secrets
- A **public client with PKCE** is required for secure OAuth flow
- Separate Keycloak clients needed for extension UI vs MCP tool access

---

## 1) Required OIDC Client Configuration

### 1.1 Client specification

```json
{
  "clientId": "ameide-vscode",
  "name": "Ameide VSCode Extension",
  "description": "Application Component (client) for Claude Code/Copilot/Cursor MCP integration",
  "enabled": true,
  "publicClient": true,
  "clientAuthenticatorType": "client-secret",
  "standardFlowEnabled": true,
  "implicitFlowEnabled": false,
  "directAccessGrantsEnabled": false,
  "serviceAccountsEnabled": false,
  "protocol": "openid-connect",
  "redirectUris": [
    "vscode://ameide.transformation/auth/callback",
    "vscode-insiders://ameide.transformation/auth/callback",
    "https://*.github.dev/redirect",
    "https://*.gitpod.io/redirect",
    "http://127.0.0.1:3000/auth/callback"
  ],
  "webOrigins": [],
  "attributes": {
    "pkce.code.challenge.method": "S256"
  },
  "defaultClientScopes": [
    "profile",
    "email",
    "roles",
    "groups",
    "organizations",
    "tenant"
  ],
  "optionalClientScopes": [
    "offline_access"
  ]
}
```

### 1.2 Configuration rationale

| Setting | Value | Rationale |
|---------|-------|-----------|
| `publicClient` | `true` | Native apps cannot securely store client secrets |
| `pkce.code.challenge.method` | `S256` | PKCE is required for public clients (RFC 7636) |
| `standardFlowEnabled` | `true` | Authorization Code flow with PKCE |
| `implicitFlowEnabled` | `false` | Implicit flow is deprecated for security |
| `directAccessGrantsEnabled` | `false` | Resource Owner Password grant is insecure |
| `serviceAccountsEnabled` | `false` | No service account needed for user-facing extension |

### 1.3 Redirect URIs

| URI Pattern | Purpose | Environment |
|-------------|---------|-------------|
| `vscode://ameide.transformation/auth/callback` | VSCode URI handler (desktop) | VSCode Desktop |
| `vscode-insiders://ameide.transformation/auth/callback` | VSCode Insiders variant | VSCode Insiders |
| `https://*.github.dev/redirect` | Codespaces browser editor | github.dev |
| `https://*.gitpod.io/redirect` | Gitpod browser editor | Gitpod |
| `http://127.0.0.1:3000/auth/callback` | Fixed port loopback (dev only) | Local dev |

**Critical:** Keycloak does not support wildcard ports in loopback URIs (`http://127.0.0.1:*/...`). Use fixed ports or prefer the `vscode://` scheme.

### 1.4 MCP-specific redirect URIs (for MCP OAuth 2.1)

VS Code's built-in MCP support (Copilot/Agent Mode) and Claude Code use their own loopback redirect URIs. These are **separate from** the extension redirect URIs and must be registered in Keycloak for MCP OAuth 2.1 to work.

See `534-mcp-protocol-adapter.md` Section 5.2 for the MCP-specific Keycloak clients.

| URI Pattern | Purpose | MCP Client |
|-------------|---------|------------|
| `http://127.0.0.1:33418` | VS Code MCP loopback (fixed port) | VS Code Copilot/Agent Mode |
| `https://vscode.dev/redirect` | VS Code Web MCP callback | VS Code Web |
| `http://127.0.0.1:*` | Claude Code loopback (dynamic port) | Claude Code `/mcp` |

**Note:** These are registered in **separate** Keycloak clients (`ameide-mcp-vscode`, `ameide-mcp-claude`) from the extension client (`ameide-vscode`). This separation ensures:

1. MCP clients authenticate directly with Keycloak (no token brokering via extension)
2. Credential scope is limited (MCP clients only get MCP-relevant scopes)
3. Token revocation is independent (revoking MCP access doesn't affect extension auth)

**VSCode URI scheme handling:**

The extension MUST use `vscode.env.uriScheme` to construct redirect URIs dynamically, not hardcoded `vscode://`:

```typescript
// Correct: dynamic scheme (handles insiders, codespaces)
const redirectUri = `${vscode.env.uriScheme}://${context.extension.id}/auth/callback`;

// For Remote/Codespaces, use asExternalUri to get a web-accessible callback
const callbackUri = await vscode.env.asExternalUri(
  vscode.Uri.parse(`${vscode.env.uriScheme}://${context.extension.id}/auth/callback`)
);
```

**Why `asExternalUri` matters:** In VSCode Web (github.dev, Codespaces browser editor), the `vscode://` scheme does not work. `asExternalUri` returns an https:// URI that routes back to the extension.

---

## 2) Token claims and scopes

The extension and MCP clients rely on Keycloak scopes already configured in the realm:

### 2.1 Standard scopes (existing)

| Scope | Claims | Usage |
|-------|--------|-------|
| `profile` | `preferred_username`, `given_name`, `family_name` | Display user identity |
| `email` | `email`, `email_verified` | User identification |
| `roles` | `realm_access.roles`, `resource_access.*.roles` | Role-based authorization |
| `groups` | `groups` | Group membership |
| `organizations` | `org_groups` | Organization membership |
| `tenant` | `tenantId` | Multi-tenant isolation |
| `offline_access` | (refresh token) | Token refresh without re-auth (see policy below) |

### 2.2 MCP-specific scopes (new)

Per MCP Authorization spec, define these scopes for MCP tool access:

| Scope | Purpose | Required by |
|-------|---------|-------------|
| `mcp:tools` | Invoke MCP tools (`tools/call`) | All MCP tool invocations |
| `mcp:resources` | Read MCP resources (`resources/read`) | MCP resource access |

**Keycloak configuration:**
```yaml
# Create these client scopes in Keycloak
clientScopes:
  - name: mcp:tools
    description: "Invoke MCP tools"
    protocol: openid-connect
    attributes:
      include.in.token.scope: "true"
  - name: mcp:resources
    description: "Read MCP resources"
    protocol: openid-connect
    attributes:
      include.in.token.scope: "true"
```

**Scope assignment:**
- `ameide-vscode` (extension UI): Does NOT need `mcp:*` scopes (calls Query Services directly)
- `ameide-mcp-vscode` (VS Code MCP): Needs `mcp:tools`, `mcp:resources`
- `ameide-mcp-claude` (Claude Code MCP): Needs `mcp:tools`, `mcp:resources`
- `ameide-mcp-cli` (Device flow): Needs `mcp:tools`, `mcp:resources`

**`offline_access` policy:**

`offline_access` is **optional** and should only be requested when necessary:

- Offline tokens can outlive normal SSO session constraints and survive logout
- They have longer lifetimes (30+ days) which increases exposure window
- Only enable for users who need "long-lived" refresh (e.g., CI/CD integrations)

**Default behavior:** Do NOT request `offline_access` for interactive desktop sessions. Standard refresh tokens (8-24 hours) are sufficient for most users.

### 2.3 Token usage by component

```
┌─────────────────────────────────────────────────────────────────────┐
│ VSCode Extension receives token with claims:                        │
│ ├── sub (user ID)                                                   │
│ ├── preferred_username                                              │
│ ├── tenantId ────────────────► Injected into MCP requests           │
│ ├── realm_access.roles ──────► Authorization decisions              │
│ └── groups ──────────────────► Group-based permissions              │
│                                                                     │
│ Extension → MCP Adapter (Streamable HTTP)                           │
│ └── Authorization: Bearer <access_token>                            │
│                                                                     │
│ MCP Adapter validates token:                                        │
│ ├── Verifies signature (Keycloak public key)                        │
│ ├── Extracts tenantId → gRPC metadata                               │
│ ├── Extracts roles → Authorization check                            │
│ └── Forwards to Domain/Projection                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3) Authentication flow

### 3.1 Flow diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. User activates extension                                         │
│    └── Extension checks SecretStorage for valid token               │
│                                                                     │
│ 2. If no token or expired:                                          │
│    ├── Extension generates PKCE code_verifier + code_challenge      │
│    ├── Opens browser: Keycloak /auth?client_id=ameide-vscode&...    │
│    └── User authenticates in browser                                │
│                                                                     │
│ 3. Keycloak redirects to vscode://ameide.transformation/auth/callback│
│    └── VSCode handles URI, routes to extension                      │
│                                                                     │
│ 4. Extension exchanges code for tokens:                             │
│    ├── POST /token with code + code_verifier                        │
│    ├── Receives access_token + refresh_token                        │
│    └── Stores in SecretStorage (encrypted)                          │
│                                                                     │
│ 5. Extension connects to Query Services (SDK):                      │
│    └── Bearer token in Authorization header (Connect transport)     │
│    └── Note: Extension uses SDK, NOT MCP (see ADR-01)               │
│                                                                     │
│ 6. Token refresh (background):                                      │
│    ├── Extension monitors token expiry                              │
│    ├── Uses refresh_token before expiry                             │
│    └── Updates SecretStorage                                        │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 PKCE parameters

```typescript
import * as vscode from 'vscode';
import * as crypto from 'crypto';

// Extension generates PKCE values
const codeVerifier = crypto.randomBytes(32).toString('base64url');
const codeChallenge = crypto
  .createHash('sha256')
  .update(codeVerifier)
  .digest('base64url');

// Build redirect URI using VSCode APIs (handles desktop/insiders/web)
const baseRedirectUri = vscode.Uri.parse(
  `${vscode.env.uriScheme}://${context.extension.id}/auth/callback`
);
// asExternalUri handles Remote/Codespaces (returns https:// URI in web contexts)
const redirectUri = await vscode.env.asExternalUri(baseRedirectUri);

// Authorization request includes:
const authUrl = new URL('https://auth.ameide.io/realms/ameide/protocol/openid-connect/auth');
authUrl.searchParams.set('client_id', 'ameide-vscode');
authUrl.searchParams.set('response_type', 'code');
authUrl.searchParams.set('redirect_uri', redirectUri.toString());
// NOTE: Do NOT include offline_access by default (see policy in Section 2)
authUrl.searchParams.set('scope', 'openid profile email roles groups organizations tenant');
authUrl.searchParams.set('code_challenge', codeChallenge);
authUrl.searchParams.set('code_challenge_method', 'S256');
authUrl.searchParams.set('state', state); // CSRF protection
```

### 3.3 Token exchange

```typescript
// Exchange authorization code for tokens
// IMPORTANT: redirect_uri must match exactly what was used in the auth request
const tokenResponse = await fetch(
  'https://auth.ameide.io/realms/ameide/protocol/openid-connect/token',
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'authorization_code',
      client_id: 'ameide-vscode',
      code: authorizationCode,
      redirect_uri: redirectUri.toString(), // Same URI used in auth request
      code_verifier: codeVerifier,
    }),
  }
);

const { access_token, refresh_token, expires_in } = await tokenResponse.json();
```

### 3.4 Device Authorization Grant (CLI / headless)

For CLI tools and headless environments where browser redirect is impractical, use the **Device Authorization Grant** (RFC 8628). This powers `ameide auth token`.

**Keycloak client configuration:**
```yaml
# CLI/Device flow client
- clientId: ameide-mcp-cli
  protocol: openid-connect
  publicClient: true
  standardFlowEnabled: false
  directAccessGrantsEnabled: false
  attributes:
    oauth2.device.authorization.grant.enabled: "true"
  defaultClientScopes:
    - openid
    - profile
    - mcp:tools
    - mcp:resources
```

**Device Authorization flow:**
```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. CLI requests device code                                          │
│    POST /protocol/openid-connect/auth/device                         │
│    └── client_id=ameide-mcp-cli&scope=openid mcp:tools mcp:resources │
│                                                                     │
│ 2. Keycloak returns device_code + user_code + verification_uri      │
│    { device_code: "...", user_code: "ABCD-EFGH",                    │
│      verification_uri: "https://auth.ameide.io/device" }             │
│                                                                     │
│ 3. CLI displays: "Enter code ABCD-EFGH at https://auth.ameide.io/device"
│    └── User opens browser, enters code, authenticates                │
│                                                                     │
│ 4. CLI polls token endpoint until user completes auth               │
│    POST /protocol/openid-connect/token                               │
│    └── grant_type=urn:ietf:params:oauth:grant-type:device_code       │
│                                                                     │
│ 5. Keycloak returns access_token + refresh_token                     │
└─────────────────────────────────────────────────────────────────────┘
```

**CLI implementation (`ameide auth token`):**
```go
// Request device code
resp, _ := http.PostForm(deviceEndpoint, url.Values{
    "client_id": {"ameide-mcp-cli"},
    "scope":     {"openid profile mcp:tools mcp:resources"},
})
var deviceResp struct {
    DeviceCode      string `json:"device_code"`
    UserCode        string `json:"user_code"`
    VerificationURI string `json:"verification_uri"`
    Interval        int    `json:"interval"`
}
json.NewDecoder(resp.Body).Decode(&deviceResp)

// Display to user
fmt.Printf("Open %s and enter code: %s\n", deviceResp.VerificationURI, deviceResp.UserCode)

// Poll for token
for {
    time.Sleep(time.Duration(deviceResp.Interval) * time.Second)
    tokenResp, _ := http.PostForm(tokenEndpoint, url.Values{
        "client_id":   {"ameide-mcp-cli"},
        "grant_type":  {"urn:ietf:params:oauth:grant-type:device_code"},
        "device_code": {deviceResp.DeviceCode},
    })
    // Check for authorization_pending, slow_down, or success
}
```

**Usage:**
```bash
# Interactive: opens browser for authentication
$ ameide auth token
Open https://auth.ameide.io/device and enter code: ABCD-EFGH
Waiting for authentication...
eyJhbGciOiJSUzI1NiIsInR5cCI...

# Quiet mode: outputs token only (for scripts)
$ export AMEIDE_TOKEN=$(ameide auth token --quiet)

# Use with Claude Code
$ claude mcp add transformation \
    --transport http \
    --url https://mcp.transformation.ameide.io \
    --header "Authorization: Bearer ${AMEIDE_TOKEN}"
```

---

## 4) Token storage (VSCode SecretStorage)

### 4.1 Storage keys

| Key | Content | Sensitivity |
|-----|---------|-------------|
| `ameide.accessToken` | JWT access token | High (short-lived) |
| `ameide.refreshToken` | Refresh token | Critical (long-lived) |
| `ameide.tokenExpiry` | Expiry timestamp | Low |
| `ameide.idToken` | ID token (optional) | Medium |

### 4.2 Implementation

```typescript
import * as vscode from 'vscode';

class TokenStorage {
  constructor(private secrets: vscode.SecretStorage) {}

  async storeTokens(tokens: {
    accessToken: string;
    refreshToken: string;
    expiresIn: number;
  }) {
    const expiry = Date.now() + tokens.expiresIn * 1000;
    await Promise.all([
      this.secrets.store('ameide.accessToken', tokens.accessToken),
      this.secrets.store('ameide.refreshToken', tokens.refreshToken),
      this.secrets.store('ameide.tokenExpiry', expiry.toString()),
    ]);
  }

  async getAccessToken(): Promise<string | undefined> {
    const expiry = await this.secrets.get('ameide.tokenExpiry');
    if (expiry && Date.now() > parseInt(expiry) - 60000) {
      // Token expired or expiring soon, refresh
      return this.refreshAccessToken();
    }
    return this.secrets.get('ameide.accessToken');
  }

  async clearTokens() {
    await Promise.all([
      this.secrets.delete('ameide.accessToken'),
      this.secrets.delete('ameide.refreshToken'),
      this.secrets.delete('ameide.tokenExpiry'),
    ]);
  }
}
```

---

## 5) Extension status bar

The extension displays authentication status in the VSCode status bar:

| State | Icon | Text | Action on Click |
|-------|------|------|-----------------|
| Authenticated | ✓ | `Ameide: user@example.com` | Show user menu |
| Expired | ⚠ | `Ameide: Session expired` | Trigger re-auth |
| Offline | ○ | `Ameide: Offline` | Show connection info |
| Not authenticated | ✗ | `Ameide: Sign in` | Trigger auth flow |

---

## 6) GitOps integration

### 6.1 Add client to Keycloak realm import

Update `gitops/ameide-gitops/sources/charts/foundation/operators-config/keycloak_realm/values.yaml`:

```yaml
# In realmImport.ameide-realm.json.clients array, add:
{
  "clientId": "ameide-vscode",
  "name": "Ameide VSCode Extension",
  "description": "Application Component (client) for Claude Code/Copilot/Cursor MCP integration",
  "enabled": true,
  "publicClient": true,
  "standardFlowEnabled": true,
  "implicitFlowEnabled": false,
  "directAccessGrantsEnabled": false,
  "protocol": "openid-connect",
  "redirectUris": [
    "vscode://ameide.transformation/auth/callback",
    "vscode-insiders://ameide.transformation/auth/callback",
    "https://*.github.dev/redirect",
    "https://*.gitpod.io/redirect",
    "http://127.0.0.1:3000/auth/callback"
  ],
  "webOrigins": [],
  "attributes": {
    "pkce.code.challenge.method": "S256"
  },
  "defaultClientScopes": ["profile", "email", "roles", "groups", "organizations", "tenant"],
  "optionalClientScopes": ["offline_access"]
}
```

**Note:** The `https://*.github.dev` and `https://*.gitpod.io` patterns are required for VSCode Web support. Remove if you don't need browser-based editor support.

### 6.2 Client reconciliation (if realm already exists)

If the Keycloak realm was already imported, use the client reconciliation mechanism:

```yaml
# In clientPatcher.clientReconciliation.clients array, add:
clientReconciliation:
  enabled: true
  clients:
    # Extension client (UI only)
    - clientId: ameide-vscode
      realm: ameide
      spec:
        name: "Ameide VSCode Extension"
        protocol: openid-connect
        publicClient: true
        standardFlowEnabled: true
        redirectUris:
          - "vscode://ameide.transformation/auth/callback"
          - "vscode-insiders://ameide.transformation/auth/callback"
          - "https://*.github.dev/redirect"
          - "https://*.gitpod.io/redirect"
          - "http://127.0.0.1:3000/auth/callback"
        attributes:
          pkce.code.challenge.method: "S256"
        defaultClientScopes:
          - profile
          - email
          - roles
          - groups
          - organizations
          - tenant
        optionalClientScopes:
          - offline_access
```

### 6.3 MCP-specific Keycloak clients

Add separate clients for MCP OAuth 2.1 (see `534-mcp-protocol-adapter.md` Section 5.2):

```yaml
# VS Code MCP client (built-in MCP support)
- clientId: ameide-mcp-vscode
  realm: ameide
  spec:
    name: "Ameide MCP (VS Code)"
    description: "MCP OAuth client for VS Code Copilot/Agent Mode"
    protocol: openid-connect
    publicClient: true
    standardFlowEnabled: true
    directAccessGrantsEnabled: false
    redirectUris:
      - "http://127.0.0.1:33418"       # VS Code MCP loopback (fixed port)
      - "https://vscode.dev/redirect"   # VS Code Web
    webOrigins:
      - "vscode-webview://*"
    attributes:
      pkce.code.challenge.method: "S256"
    defaultClientScopes:
      - profile
      - email
      - roles
      - tenant

# Claude Code MCP client
- clientId: ameide-mcp-claude
  realm: ameide
  spec:
    name: "Ameide MCP (Claude Code)"
    description: "MCP OAuth client for Claude Code /mcp authentication"
    protocol: openid-connect
    publicClient: true
    standardFlowEnabled: true
    directAccessGrantsEnabled: false
    redirectUris:
      - "http://127.0.0.1:*"            # Claude Code loopback (dynamic port)
    attributes:
      pkce.code.challenge.method: "S256"
    defaultClientScopes:
      - profile
      - email
      - roles
      - tenant
```

**Design rationale:**

| Client | Purpose | Redirect URIs |
|--------|---------|---------------|
| `ameide-vscode` | Extension UI (tree views, status bar) | `vscode://` scheme |
| `ameide-mcp-vscode` | VS Code built-in MCP (Copilot/Agent Mode) | `http://127.0.0.1:33418` |
| `ameide-mcp-claude` | Claude Code MCP | `http://127.0.0.1:*` |

This separation ensures MCP clients authenticate directly with Keycloak without token brokering.

---

## 7) Security considerations

### 7.1 PKCE is mandatory

Public clients MUST use PKCE (RFC 7636). The Keycloak client configuration enforces `S256` challenge method.

### 7.2 State parameter

The authorization request MUST include a `state` parameter to prevent CSRF attacks:

```typescript
const state = crypto.randomBytes(16).toString('base64url');
// Store state locally, verify on callback
```

### 7.3 Secure token storage

**VSCode SecretStorage** is the correct choice (backed by OS keychain/credential manager), but it has limitations:

| Concern | Reality | Mitigation |
|---------|---------|------------|
| OS-level encryption | Yes, backed by macOS Keychain / Windows Credential Manager / libsecret | Good baseline security |
| Cross-extension isolation | Partial; extensions share the same credential namespace | Use unique key prefixes (`ameide.*`) |
| Malicious extension access | A malicious extension in the same workspace could potentially read secrets | Rely on marketplace review; short token TTLs |
| Local admin/root access | Secrets are accessible to processes with sufficient OS privileges | Accept as out-of-scope threat |

**Mitigations:**
- Short access token TTLs (5-15 minutes) limit exposure window
- Do NOT store offline tokens unless explicitly required
- Never log tokens or include in error messages
- Clear tokens on explicit logout

### 7.4 Token lifetime

| Token | Recommended Lifetime | Rationale |
|-------|---------------------|-----------|
| Access token | 5-15 minutes | Short-lived, limits exposure |
| Refresh token | 8-24 hours | Balances UX with security |
| Offline refresh | 30 days (optional) | Only for CI/CD; avoid for interactive use |

### 7.5 Redirect URI security

**Minimize redirect URIs:** Keycloak maintainers warn against broad wildcards. Keep the allowlist minimal and exact:

| Pattern | Risk | Recommendation |
|---------|------|----------------|
| `http://127.0.0.1:*/...` | Keycloak doesn't support; may fail | Use fixed port |
| `https://*.example.com/...` | Subdomain takeover risk | Avoid in production |
| `vscode://...` | Safe for desktop | Primary redirect |

**Production policy:** Only register redirect URIs for environments you actively support. Remove unused URIs.

### 7.6 VSCode Web / Codespaces authentication

VSCode Web (github.dev, Codespaces browser editor) cannot use `vscode://` scheme redirects. The extension MUST handle this:

```typescript
async function initiateAuth(context: vscode.ExtensionContext): Promise<void> {
  const baseUri = vscode.Uri.parse(
    `${vscode.env.uriScheme}://${context.extension.id}/auth/callback`
  );

  // asExternalUri returns an https:// URI in web contexts
  const redirectUri = await vscode.env.asExternalUri(baseUri);

  // Check if we're in a web context (redirectUri will be https://)
  const isWebContext = redirectUri.scheme === 'https';

  if (isWebContext) {
    // Web context: open auth URL in same browser (no external browser launch)
    // Callback will route through the web host (github.dev proxy)
    await vscode.env.openExternal(vscode.Uri.parse(authUrl.toString()));
  } else {
    // Desktop context: standard external browser flow
    await vscode.env.openExternal(vscode.Uri.parse(authUrl.toString()));
  }
}
```

**Fallback for unsupported environments:** If `asExternalUri` fails or returns an invalid URI, display a manual token entry option (paste token from browser).

---

## 8) Acceptance criteria

1. Keycloak client `ameide-vscode` is configured as public client with PKCE (S256).
2. Client uses dynamic redirect URIs via `vscode.env.uriScheme` + `vscode.env.asExternalUri`.
3. All required scopes (profile, email, roles, groups, organizations, tenant) are assigned.
4. `offline_access` is NOT requested by default (policy-controlled).
5. Extension implements PKCE authorization code flow per RFC 7636.
6. Tokens are stored in VSCode SecretStorage with unique key prefixes.
7. Access token TTL is short (5-15 minutes); refresh happens automatically.
8. Extension status bar shows authentication state.
9. GitOps configuration includes the client (realm import or reconciliation).
10. Extension handles VSCode Web (github.dev, Codespaces) via `asExternalUri` fallback.
11. Redirect URIs are minimal and exact (no wildcard ports).
12. **MCP OAuth clients**: Separate Keycloak clients exist for MCP authentication (`ameide-mcp-vscode`, `ameide-mcp-claude`).
13. **MCP redirect URIs**: VS Code MCP loopback (`http://127.0.0.1:33418`) and Claude Code loopback (`http://127.0.0.1:*`) are registered.
14. **No token brokering**: Extension does NOT broker tokens for MCP clients; each MCP client authenticates directly.

---

## 9) Dependencies

| Dependency | Purpose |
|------------|---------|
| **Keycloak** | OIDC identity provider |
| **VSCode SecretStorage API** | Secure token storage |
| **VSCode URI Handler API** | OAuth callback handling |
| **538 VSCode Client** | Parent extension spec |
| **534 MCP Adapter** | Token consumption endpoint |
