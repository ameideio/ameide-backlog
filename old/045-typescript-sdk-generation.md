# 045: TypeScript SDK Generation for Public Access ✅ COMPLETED

**Status**: ✅ COMPLETED (2025-08-14)
**Implementation**: See `packages/ameide_sdk_ts` and `packages/ameide_core_proto_web`

## Overview

Generate and publish a TypeScript SDK from proto definitions to enable external developers to integrate with the Ameide platform APIs. The SDK should provide type-safe, developer-friendly access to all public API endpoints.

## Current State

**✅ COMPLETED IMPLEMENTATION:**

Based on the proto-based API implementation (044), we now have:
- ✅ Proto definitions in `packages/ameide_core_proto`
- ✅ Connect-ES TypeScript code generation pipeline
- ✅ SDK packaging with Bazel and tsup
- ✅ Client-side authentication via interceptors
- ✅ Comprehensive documentation and examples
- ✅ Mock transport for development

What was implemented:
- ✅ Buf + Connect-ES generation pipeline
- ✅ React Query hooks with method symbols
- ✅ Transport abstraction (browser/Node.js)
- ✅ Mock transport with one-flip switching
- ✅ SSR utilities for Next.js
- ✅ Full TypeScript type safety

## Requirements

### 1. Code Generation

**Proto to TypeScript Generation**
- Use `ts-proto` or `protobuf-ts` for type generation
- Generate both gRPC-Web and REST clients
- Preserve proto field behaviors and validations
- Support streaming for real-time updates

**Generated Code Structure**
```typescript
// Generated from users.proto
export interface User {
  id: string;
  name: string;
  email?: string;
  roles: Role[];
  isActive: boolean;
  metadata: Record<string, string>;
  createdAt?: Timestamp;
  updatedAt?: Timestamp;
}

export class UserServiceClient {
  constructor(private transport: Transport);
  
  async createUser(request: CreateUserRequest): Promise<CreateUserResponse>;
  async getUser(request: GetUserRequest): Promise<GetUserResponse>;
  async listUsers(request: ListUsersRequest): Promise<ListUsersResponse>;
  // ... other methods
}
```

### 2. SDK Architecture

```
@ameide/sdk/
├── src/
│   ├── generated/           # Proto-generated code
│   │   ├── users/
│   │   ├── workflows/
│   │   ├── agents/
│   │   └── ipas/
│   ├── client/             # High-level client
│   │   ├── AmeideClient.ts
│   │   ├── auth/
│   │   ├── errors/
│   │   └── transport/
│   ├── utils/              # Helper utilities
│   │   ├── retry.ts
│   │   ├── pagination.ts
│   │   └── streaming.ts
│   └── index.ts            # Public API exports
├── examples/               # Usage examples
├── docs/                   # API documentation
└── package.json
```

### 3. Client Implementation

**High-Level Client Wrapper**
```typescript
export class AmeideClient {
  private userService: UserServiceClient;
  private workflowsService: WorkflowServiceClient;
  private agentService: AgentServiceClient;
  private ipaService: IPAServiceClient;
  
  constructor(config: AmeideConfig) {
    const transport = this.createTransport(config);
    this.userService = new UserServiceClient(transport);
    // ... initialize other services
  }
  
  // Convenience methods
  async getCurrentUser(): Promise<User> {
    return this.userService.getCurrentUser({});
  }
  
  async executeWorkflow(workflowsId: string, inputs: Record<string, any>): Promise<WorkflowExecution> {
    return this.workflowsService.executeWorkflow({
      workflowsId,
      inputs
    });
  }
}
```

**Authentication Handling**
```typescript
export interface AuthProvider {
  getAccessToken(): Promise<string>;
  refreshToken(): Promise<string>;
}

export class OAuth2Provider implements AuthProvider {
  constructor(private config: OAuth2Config) {}
  
  async getAccessToken(): Promise<string> {
    // Handle token refresh automatically
    if (this.isTokenExpired()) {
      await this.refreshToken();
    }
    return this.accessToken;
  }
}
```

### 4. Developer Experience

**TypeScript Features**
- Full type safety with generated interfaces
- IntelliSense support in IDEs
- Discriminated unions for proto oneofs
- Async/await API
- Tree-shakeable exports

**Error Handling**
```typescript
export class AmeideError extends Error {
  constructor(
    public code: ErrorCode,
    public message: string,
    public details?: any
  ) {
    super(message);
  }
}

export enum ErrorCode {
  NOT_FOUND = 'NOT_FOUND',
  INVALID_ARGUMENT = 'INVALID_ARGUMENT',
  PERMISSION_DENIED = 'PERMISSION_DENIED',
  // ... mapped from proto/grpc codes
}
```

**Retry and Circuit Breaker**
```typescript
export interface RetryConfig {
  maxAttempts: number;
  backoffMultiplier: number;
  retryableErrors: ErrorCode[];
}

export class RetryableTransport implements Transport {
  constructor(
    private baseTransport: Transport,
    private config: RetryConfig
  ) {}
  
  async unary<Req, Res>(method: Method<Req, Res>, request: Req): Promise<Res> {
    return retry(
      () => this.baseTransport.unary(method, request),
      this.config
    );
  }
}
```

## Implementation Plan

### Phase 1: Code Generation Pipeline (Week 1)

**Day 1-2: Proto to TypeScript Setup**
- [ ] Evaluate ts-proto vs protobuf-ts vs grpc-web
- [ ] Configure code generation with Buf
- [ ] Setup generation scripts in Bazel
- [ ] Generate initial TypeScript interfaces

**Day 3-4: Client Generation**
- [ ] Generate service clients from proto
- [ ] Add transport abstraction layer
- [ ] Implement gRPC-Web transport
- [ ] Implement REST transport fallback

**Day 5: Build Pipeline**
- [ ] Create npm package structure
- [ ] Setup TypeScript compilation
- [ ] Configure tree-shaking with Rollup
- [ ] Add source maps for debugging

### Phase 2: SDK Implementation (Week 2)

**Day 1-2: High-Level Client**
- [ ] Implement AmeideClient wrapper
- [ ] Add convenience methods
- [ ] Implement connection pooling
- [ ] Add request/response interceptors

**Day 3-4: Authentication & Errors**
- [ ] Implement OAuth2Provider
- [ ] Add API key authentication
- [ ] Create error mapping from proto
- [ ] Add retry logic with backoff

**Day 5: Developer Experience**
- [ ] Add JSDoc comments to generated code
- [ ] Create type guards and helpers
- [ ] Implement pagination helpers
- [ ] Add streaming utilities

### Phase 3: Documentation & Publishing (Week 3)

**Day 1-2: Documentation**
- [ ] Generate API documentation with TypeDoc
- [ ] Create getting started guide
- [ ] Write authentication guide
- [ ] Add code examples for each service

**Day 3: Testing & Examples**
- [ ] Unit tests for SDK components
- [ ] Integration tests against mock server
- [ ] Create example applications
- [ ] Add React/Next.js examples

**Day 4-5: Publishing Infrastructure**
- [ ] Setup npm publishing pipeline
- [ ] Configure GitHub releases
- [ ] Add version management
- [ ] Setup CDN distribution

## Technical Considerations

### 1. Proto to TypeScript Mapping

**Recommended Tool: buf + connect-es**
```yaml
# buf.gen.yaml
version: v1
plugins:
  - plugin: es
    out: src/generated
    opt:
      - target=ts
      - import_extension=.js
  - plugin: connect-es
    out: src/generated
    opt:
      - target=ts
      - import_extension=.js
```

### 2. Transport Layer

**Multiple Transport Support**
```typescript
export interface Transport {
  unary<Req, Res>(method: Method<Req, Res>, request: Req): Promise<Res>;
  stream<Req, Res>(method: Method<Req, Res>, request: Req): AsyncIterable<Res>;
}

// gRPC-Web for internal use
export class GrpcWebTransport implements Transport {
  constructor(private baseUrl: string, private auth: AuthProvider) {}
}

// REST for broader compatibility
export class RestTransport implements Transport {
  constructor(private baseUrl: string, private auth: AuthProvider) {}
}
```

### 3. Bundle Size Optimization

**Tree-Shaking Strategy**
- Separate imports per service
- Dynamic imports for large features
- Minimal runtime dependencies
- Proto lite runtime option
- Dual ESM/CJS builds for maximum compatibility

### 4. Browser Compatibility

**Requirements**
- ES2017+ (async/await)
- Fetch API or polyfill
- WebSocket for streaming (optional)
- Works in Node.js and browsers

## Versioning Strategy

### SDK Versioning
- Follow semver (major.minor.patch)
- Major version tracks proto breaking changes
- Minor version for new features
- Patch for bug fixes

### API Compatibility
```typescript
// Version negotiation
export class AmeideClient {
  constructor(config: AmeideConfig) {
    this.apiVersion = config.apiVersion || 'v1';
    // Validate compatibility
    this.validateApiVersion();
  }
}
```

## Security Considerations

1. **Token Storage**
   - Never store tokens in localStorage
   - Use secure cookie storage in browsers
   - Implement token refresh automatically

2. **CORS Handling**
   - SDK should handle CORS preflight
   - Support credentials in requests
   - Configurable allowed origins

3. **Request Signing**
   - Optional request signing for high-security
   - HMAC-SHA256 with timestamp
   - Replay attack prevention

## Distribution

### NPM Package
```json
{
  "name": "@ameide/sdk",
  "version": "1.0.0",
  "main": "dist/index.js",
  "module": "dist/index.esm.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.esm.js",
      "require": "./dist/index.js",
      "types": "./dist/index.d.ts"
    },
    "./users": {
      "import": "./dist/users/index.esm.js",
      "require": "./dist/users/index.js",
      "types": "./dist/users/index.d.ts"
    }
  }
}
```

### CDN Distribution
- Publish to unpkg/jsdelivr
- Provide minified UMD build
- Include source maps
- Support subpath imports

## Success Criteria

1. **Code Quality**
   - 100% type coverage
   - No any types in public API
   - Comprehensive JSDoc comments
   - Pass strict TypeScript checks

2. **Developer Experience**
   - < 5 minutes to first API call
   - IntelliSense for all methods
   - Clear error messages
   - Helpful examples

3. **Performance**
   - < 50KB gzipped base bundle
   - < 100ms SDK initialization
   - Efficient proto serialization
   - Minimal memory footprint

4. **Compatibility**
   - Node.js 14+
   - All modern browsers
   - React Native support
   - Deno compatibility

## Dependencies on 044

This SDK generation depends on:
- Stable proto definitions
- Completed API implementation (044)
- Published API documentation
- CORS configuration on APIs
- Public API endpoint URLs

## Future Enhancements

1. **Framework Integrations**
   - React hooks (`@ameide/sdk-react`)
   - Vue composables
   - Angular services
   - Svelte stores

2. **Advanced Features**
   - Offline support with IndexedDB
   - Request caching
   - Optimistic updates
   - Real-time subscriptions

3. **Developer Tools**
   - VS Code extension
   - Chrome DevTools extension
   - Mock server for testing
   - SDK playground

## References

- [buf connect-es](https://buf.build/docs/connect/connect-protocol)
- [ts-proto](https://github.com/stephenh/ts-proto)
- [protobuf-ts](https://github.com/timostamm/protobuf-ts)
- [gRPC-Web](https://github.com/grpc/grpc-web)
- [TypeDoc](https://typedoc.org/)
- [API Styleguide](https://google.github.io/styleguide/tsguide.html)

## Architectural Review Findings (2025-07-26)

### Key Concerns Identified

1. **Generator not chosen** – `ts-proto` vs `protobuf-ts` defers every downstream decision
   - Impact: Blocks all SDK development
   - Recommendation: **ts-proto** for wider ecosystem support, Connect-ES ready
   - Action: Select ts-proto and scaffold SDK CI (Day 0-5)

2. **Auth weak** – Only bearer token, no refresh-token or custom provider interface
   - Risk: Token expiry causes poor DX
   - Action: Implement OAuth2Provider with automatic refresh logic
   - Add support for custom auth providers (API key, mTLS)

3. **Streaming** – Browser vs Node transport not solved
   - Connect-ES provides unified transport for both environments
   - Action: Use Connect-ES transport abstraction for streaming

4. **Version coupling** – SDK semver to proto major not stated
   - Risk: Breaking changes without major version bump
   - Action: Define clear policy: proto major version → SDK major version

### Missing Dependencies

- Stable proto definitions (blocked on 048)
- Completed API implementation (044)
- Published API documentation
- CORS configuration on APIs
- Public API endpoint URLs

### Immediate Actions (P1)

- [ ] Select ts-proto for wider ecosystem, Connect-ES compatibility (Day 0-5)
- [ ] Implement OAuth2Provider with token refresh
- [ ] Define SDK versioning policy tied to proto versions
- [ ] Setup tree-shaking and dual ESM/CJS builds
- [ ] Create bundle size CI check (< 50KB gzipped base)

### Implementation Recommendations

1. **Code Generation Pipeline**
   - Use buf + connect-es for optimal DX
   - Generate both gRPC-Web and REST clients
   - Add JSDoc from proto comments automatically

2. **Transport Layer**
   - Primary: Connect-ES for unified experience
   - Fallback: REST for broader compatibility
   - Auto-detect best transport based on environment

3. **Bundle Optimization**
   - Separate entry points per service for tree-shaking
   - Dynamic imports for large features
   - Proto lite runtime for smaller bundles

4. **Authentication**
   ```typescript
   export interface AuthProvider {
     getAccessToken(): Promise<string>;
     refreshToken(): Promise<string>;
     onTokenExpired?: (retry: () => Promise<void>) => void;
   }
   ```

### Success Metrics

- < 5 minutes to first API call
- < 50KB gzipped base bundle
- 100% type coverage
- Works in Node.js 14+ and all modern browsers