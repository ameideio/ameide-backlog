# 046: Portal Backend Integration Requirements

## Overview

Enable the `www-ameide-portal` web application to interact seamlessly with backend APIs, providing a complete user experience for managing IPAs, workflows, agents, and monitoring executions.

## Current State

Based on existing infrastructure:
- TypeScript SDK generation planned (045)
- Proto-based APIs being implemented (044)
- Portal service exists but integration unclear
- Authentication via JWT/OIDC available

What's missing for portal integration:
- Next.js/React specific SDK integration
- Server-side rendering (SSR) authentication
- Real-time updates via WebSocket/SSE (transport abstraction needed)
- State management integration (data-fetch/cache layer)
- File upload/download handling (strategy undefined)
- Session management
- SSR hydration strategy
- Token scope/audience rules

## Requirements

### 1. Next.js SDK Integration

**Server Components Support**
```typescript
// app/lib/ameide-server.ts
import { AmeideClient } from '@ameide/sdk';
import { cookies } from 'next/headers';

export async function getServerClient() {
  const token = cookies().get('auth-token')?.value;
  
  return new AmeideClient({
    endpoint: process.env.AMEIDE_API_URL!,
    auth: {
      type: 'bearer',
      token: token || ''
    },
    // Server-side specific config
    cache: 'force-cache',
    next: { revalidate: 60 }
  });
}
```

**Client Components Hook**
```typescript
// app/hooks/useAmeide.ts
import { useAuth } from '@/contexts/AuthContext';
import { useMemo } from 'react';
import { AmeideClient } from '@ameide/sdk';

export function useAmeideClient() {
  const { token } = useAuth();
  
  return useMemo(() => {
    return new AmeideClient({
      endpoint: '/api/proxy', // Proxy through Next.js
      auth: {
        type: 'bearer',
        token: token || ''
      }
    });
  }, [token]);
}
```

### 2. Authentication Flow

**NextAuth.js Integration**
```typescript
// app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth';
import { JWT } from 'next-auth/jwt';

export const authOptions = {
  providers: [
    {
      id: 'ameide',
      name: 'Ameide',
      type: 'oauth',
      authorization: {
        url: process.env.AUTH0_ISSUER_BASE_URL + '/authorize',
        params: {
          scope: 'openid profile email offline_access',
          audience: process.env.AMEIDE_API_AUDIENCE
        }
      },
      token: process.env.AUTH0_ISSUER_BASE_URL + '/oauth/token',
      userinfo: process.env.AUTH0_ISSUER_BASE_URL + '/userinfo',
      client: {
        id: process.env.AUTH0_CLIENT_ID,
        secret: process.env.AUTH0_CLIENT_SECRET
      }
    }
  ],
  callbacks: {
    async jwt({ token, account }) {
      if (account) {
        token.accessToken = account.access_token;
        token.refreshToken = account.refresh_token;
        token.expiresAt = account.expires_at;
      }
      
      // Refresh token if expired
      if (Date.now() > token.expiresAt * 1000) {
        return refreshAccessToken(token);
      }
      
      return token;
    },
    async session({ session, token }) {
      // Never expose refresh token to client
      session.accessToken = token.accessToken;
      session.user.id = token.sub;
      session.tokenScope = token.scope;
      session.tokenAudience = token.audience;
      return session;
    }
  }
};
```

### 3. Real-time Updates

**Server-Sent Events for Executions**
```typescript
// app/api/sse/executions/[id]/route.ts
export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const stream = new ReadableStream({
    async start(controller) {
      const client = await getServerClient();
      
      // Subscribe to execution updates
      const subscription = client.executions.subscribe(params.id);
      
      for await (const update of subscription) {
        const data = `data: ${JSON.stringify(update)}\n\n`;
        controller.enqueue(new TextEncoder().encode(data));
      }
    }
  });
  
  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive'
    }
  });
}
```

**React Hook for Real-time Data**
```typescript
// app/hooks/useExecutionStream.ts
import { RealtimeClient } from '@/lib/realtime';

export function useExecutionStream(executionId: string) {
  const [execution, setExecution] = useState<Execution>();
  const [status, setStatus] = useState<'connecting' | 'connected' | 'error'>('connecting');
  const realtimeClient = useRealtimeClient();  // Abstract transport
  
  useEffect(() => {
    // Abstract transport - could be SSE, WebSocket, or polling
    const subscription = realtimeClient.subscribe(
      `executions/${executionId}`,
      {
        onMessage: (update) => {
          setExecution(update);
          setStatus('connected');
        },
        onError: () => setStatus('error'),
        transport: 'auto'  // Let client choose best transport
      }
    );
    
    return () => subscription.unsubscribe();
  }, [executionId, realtimeClient]);
  
  return { execution, status };
}
```

### 4. State Management

**Zustand Store with SSR Hydration**
```typescript
// app/stores/ameide-store.ts
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

interface AmeideState {
  currentWorkspace: Workspace | null;
  workflows: Workflow[];
  agents: Agent[];
  ipas: IPA[];
  
  // Actions
  setWorkspace: (workspace: Workspace) => void;
  fetchWorkflows: () => Promise<void>;
  fetchAgents: () => Promise<void>;
  fetchIPAs: () => Promise<void>;
  
  // SSR helpers
  _hasHydrated: boolean;
  setHasHydrated: (state: boolean) => void;
}

export const useAmeideStore = create<AmeideState>()(
  devtools(
    persist(
      (set, get) => ({
        currentWorkspace: null,
        workflows: [],
        agents: [],
        ipas: [],
        
        setWorkspace: (workspace) => set({ currentWorkspace: workspace }),
        
        fetchWorkflows: async () => {
          const client = getClient();
          const workflows = await client.workflows.list({
            workspaceId: get().currentWorkspace?.id
          });
          set({ workflows });
        },
        
        // ... other actions
      }),
      {
        name: 'ameide-storage',
        partialize: (state) => ({ currentWorkspace: state.currentWorkspace })
      }
    )
  )
);
```

### 5. File Handling

**Upload Strategy with Signed URLs**
```typescript
// app/lib/upload.ts
export class FileUploadService {
  async uploadLargeFile(file: File, metadata: FileMetadata) {
    // Get signed upload URL from API
    const { uploadUrl, fileId } = await this.getSignedUploadUrl({
      fileName: file.name,
      contentType: file.type,
      size: file.size,
      metadata
    });
    
    // Direct upload to S3/storage
    await this.uploadToSignedUrl(uploadUrl, file, {
      onProgress: (progress) => this.onUploadProgress?.(progress)
    });
    
    // Confirm upload completion
    return this.confirmUpload(fileId);
  }
}

// app/components/FileUpload.tsx
export function WorkflowUpload() {
  const uploadService = useFileUploadService();
  const [uploading, setUploading] = useState(false);
  
  const handleUpload = async (file: File) => {
    setUploading(true);
    
    try {
      // For large files, use signed URL upload
      if (file.size > 10 * 1024 * 1024) {  // 10MB
        const result = await uploadService.uploadLargeFile(file, {
          type: 'workflows',
          format: 'bpmn'
        });
        router.push(`/workflows/${result.workflowsId}`);
      } else {
        // Small files can go through API proxy
        const content = await file.text();
        const workflows = await client.workflows.createFromBPMN({
          name: file.name.replace('.bpmn', ''),
          bpmnXml: content
        });
        router.push(`/workflows/${workflows.id}`);
      }
      
      toast.success('Workflow uploaded successfully');
    } catch (error) {
      toast.error('Failed to upload workflows');
    } finally {
      setUploading(false);
    }
  };
  
  return (
    <UploadDropzone
      accept=".bpmn,.bpmn20.xml"
      onUpload={handleUpload}
      uploading={uploading}
      maxSize={100 * 1024 * 1024}  // 100MB limit
    />
  );
}
```

### 6. API Proxy Configuration

**Next.js API Routes as Proxy**
```typescript
// app/api/proxy/[...path]/route.ts
export async function handler(req: NextRequest) {
  const session = await getServerSession(authOptions);
  
  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }
  
  const path = req.nextUrl.pathname.replace('/api/proxy/', '');
  const url = `${process.env.AMEIDE_API_URL}/${path}${req.nextUrl.search}`;
  
  const response = await fetch(url, {
    method: req.method,
    headers: {
      'Authorization': `Bearer ${session.accessToken}`,
      'Content-Type': 'application/json',
      'X-Request-ID': crypto.randomUUID()
    },
    body: req.body
  });
  
  return response;
}
```

### 7. UI Components Library

**Ameide-specific Components**
```typescript
// packages/ui-components/
├── WorkflowDesigner/      # BPMN editor wrapper
├── AgentBuilder/          # Visual agent editor
├── IPAComposer/          # IPA composition UI
├── ExecutionMonitor/     # Real-time execution view
├── MetricsChart/         # Performance metrics
└── LogViewer/            # Structured log display
```

### 8. Error Boundary & Fallbacks

```typescript
// app/components/ErrorBoundary.tsx
export function APIErrorBoundary({ children }: { children: React.ReactNode }) {
  return (
    <ErrorBoundary
      fallback={<APIErrorFallback />}
      onError={(error) => {
        if (error instanceof AmeideError) {
          switch (error.code) {
            case 'UNAUTHORIZED':
              router.push('/auth/login');
              break;
            case 'RATE_LIMITED':
              toast.error('Too many requests. Please try again later.');
              break;
            default:
              toast.error(error.message);
          }
        }
      }}
    >
      {children}
    </ErrorBoundary>
  );
}
```

## Implementation Plan

### Phase 1: Authentication & Session (Week 1)

**Day 1-2: NextAuth Setup**
- [ ] Configure NextAuth with Auth0/Keycloak
- [ ] Implement token refresh logic
- [ ] Setup secure session storage
- [ ] Add auth middleware

**Day 3-4: API Client Integration**
- [ ] Create server-side client helper
- [ ] Implement client-side hooks
- [ ] Setup API proxy routes
- [ ] Add request interceptors

**Day 5: Testing**
- [ ] Test auth flows
- [ ] Verify token refresh
- [ ] Test API calls
- [ ] Error handling

### Phase 2: Real-time & State (Week 2)

**Day 1-2: Real-time Infrastructure**
- [ ] Design RealtimeClient transport abstraction
- [ ] Implement SSE transport adapter
- [ ] Consider WebSocket for bidirectional needs
- [ ] Create subscription hooks
- [ ] Add reconnection logic
- [ ] Handle connection errors

**Day 3-4: State Management**
- [ ] Evaluate TanStack Query vs SWR for data-fetch layer
- [ ] Setup Zustand stores with TanStack Query
- [ ] Implement data fetching patterns
- [ ] Add optimistic updates
- [ ] Cache management strategy
- [ ] SSR hydration boundaries

**Day 5: UI Integration**
- [ ] Connect components to store
- [ ] Add loading states
- [ ] Implement error boundaries
- [ ] Test real-time updates

### Phase 3: Advanced Features (Week 3)

**Day 1-2: File Handling**
- [ ] Implement signed URL upload flow
- [ ] Add direct S3 upload for large files
- [ ] Create upload service abstraction
- [ ] Add drag-and-drop support
- [ ] Handle large files (>10MB)
- [ ] Progress tracking with resumable uploads

**Day 3-4: UI Components**
- [ ] Create workflows designer wrapper
- [ ] Build execution monitor
- [ ] Add metrics visualization
- [ ] Implement log viewer

**Day 5: Polish**
- [ ] Performance optimization
- [ ] Accessibility audit
- [ ] Mobile responsiveness
- [ ] Documentation

## Environment Configuration

```env
# .env.local
NEXT_PUBLIC_APP_URL=http://localhost:3000
AMEIDE_API_URL=http://localhost:8000
AMEIDE_GRPC_URL=http://localhost:50051

# Auth
AUTH0_ISSUER_BASE_URL=https://your-tenant.auth0.com
AUTH0_CLIENT_ID=your-client-id
AUTH0_CLIENT_SECRET=your-client-secret
AUTH0_AUDIENCE=https://api.ameide.io
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=your-secret-key

# Feature Flags
NEXT_PUBLIC_ENABLE_WEBSOCKET=true
NEXT_PUBLIC_ENABLE_OFFLINE=false
```

## Performance Considerations

1. **API Call Optimization**
   - Implement request deduplication
   - Use React Query or SWR for caching
   - Batch API requests where possible
   - Implement virtual scrolling for lists

2. **Bundle Size**
   - Dynamic imports for heavy components
   - Tree-shake unused SDK methods
   - Lazy load visualization libraries
   - Use Next.js Image optimization

3. **SSR vs Client Rendering**
   - SSR for initial page load
   - Client-side for interactive features
   - Streaming SSR for large data
   - Edge runtime for auth checks

## Security Requirements

1. **Token Management**
   - Store tokens in httpOnly cookies
   - Implement CSRF protection
   - Use secure flag in production
   - Regular token rotation

2. **Content Security Policy**
   ```typescript
   // next.config.js
   const cspHeader = `
     default-src 'self';
     script-src 'self' 'unsafe-eval' 'unsafe-inline';
     style-src 'self' 'unsafe-inline';
     img-src 'self' blob: data:;
     font-src 'self';
     object-src 'none';
     base-uri 'self';
     form-action 'self';
     frame-ancestors 'none';
     block-all-mixed-content;
     upgrade-insecure-requests;
   `;
   ```

3. **API Security**
   - Validate all inputs
   - Implement rate limiting
   - Use API proxy to hide endpoints
   - Add request signing for sensitive ops

## Monitoring & Analytics

1. **Application Monitoring**
   - Sentry for error tracking
   - Performance monitoring
   - User analytics (privacy-compliant)
   - API call metrics

2. **User Experience Metrics**
   - Page load times
   - Time to interactive
   - API response times
   - Error rates by feature

## Success Criteria

1. **Authentication**
   - Seamless login/logout
   - Automatic token refresh (server-side only)
   - Session persistence
   - Multi-tab sync
   - Clear token scope/audience handling
   - No refresh token exposure in browser

2. **Performance**
   - < 3s initial page load
   - < 100ms API response (cached)
   - Smooth real-time updates
   - No memory leaks

3. **User Experience**
   - Intuitive navigation
   - Clear error messages
   - Responsive design
   - Offline indication

4. **Developer Experience**
   - Type-safe API calls
   - Hot module replacement
   - Clear documentation
   - Easy local setup

## Dependencies

- TypeScript SDK (045)
- Proto-based APIs (044)
- NextAuth.js
- Zustand or Redux Toolkit
- React Query or SWR
- Socket.io or native EventSource

## Future Enhancements

1. **Progressive Web App**
   - Service worker for offline
   - Push notifications
   - Background sync
   - Install prompt

2. **Collaboration Features**
   - Real-time collaboration
   - Presence indicators
   - Conflict resolution
   - Change history

3. **Advanced Visualizations**
   - 3D workflows visualization
   - AR/VR support
   - Custom dashboards
   - Export capabilities

## Architectural Review Findings (2025-07-26)

### Key Concerns Identified

1. **Data layer undefined** – Zustand, React-Query, SWR mentioned; choose one to avoid hydration bugs
   - Risk: Inconsistent state management, SSR hydration issues
   - Recommendation: **TanStack Query** + Zustand for optimal DX
   - Action: Standardize on TanStack Query for server state, Zustand for client state

2. **One-way realtime** – SSE limits bidirectional features like designer collaboration
   - Current: Only SSE implementation shown
   - Action: Design `RealtimeClient` abstraction supporting SSE, WebSocket, and polling
   - Future: Enable bidirectional communication for collaborative features

3. **Token exposure** – Refresh tokens never leave server, but examples mix client/server imports
   - Security risk: Accidental token leakage to browser
   - Action: Clear separation of server-only auth code
   - Add lint rules to prevent server imports in client components

4. **File upload** – Strategy only half-written (signed URL vs proxy)
   - Missing: Resumable uploads, progress tracking, large file handling
   - Action: Implement signed URL strategy for files > 10MB
   - Add upload progress with resumable support

5. **SSR hydration** – Zustand partial-persist may clash with RSC
   - Risk: Hydration mismatches, flash of incorrect content
   - Action: Define clear hydration boundaries
   - Use TanStack Query for server-fetched data

### Missing Design Decisions

- Transport abstraction for real-time (SSE vs WebSocket vs polling)
- Data fetching/caching strategy (React Query vs SWR)
- State management approach (Zustand vs Redux Toolkit)
- File upload architecture for large files
- Token refresh strategy (server-side only)
- SSR hydration boundaries

### Immediate Actions (P1)

- [ ] Choose TanStack Query for data fetching layer (Day 0-5)
- [ ] Design RealtimeClient abstraction with multiple transports (Week 2)
- [ ] Implement secure token handling with server/client separation
- [ ] Complete file upload strategy with resumable uploads
- [ ] Define SSR hydration boundaries and patterns

### Implementation Recommendations

1. **State Management Architecture**
   ```typescript
   // Server state: TanStack Query
   const { data: workflows } = useQuery({
     queryKey: ['workflows', workspaceId],
     queryFn: () => client.workflows.list({ workspaceId })
   });
   
   // Client state: Zustand
   const { currentWorkspace, setWorkspace } = useAmeideStore();
   ```

2. **Realtime Transport Abstraction**
   ```typescript
   interface RealtimeTransport {
     subscribe(channel: string, handler: MessageHandler): Subscription;
     unsubscribe(subscription: Subscription): void;
   }
   
   class RealtimeClient {
     constructor(private transports: RealtimeTransport[]) {}
     
     subscribe(channel: string, options: SubscribeOptions) {
       // Auto-select best transport (WebSocket → SSE → Polling)
       const transport = this.selectTransport(options);
       return transport.subscribe(channel, options.onMessage);
     }
   }
   ```

3. **Secure Auth Pattern**
   ```typescript
   // app/lib/auth/server.ts (server-only)
   export async function refreshAccessToken(refreshToken: string) {
     // Server-only refresh logic
   }
   
   // app/lib/auth/client.ts (client-safe)
   export function useAuth() {
     // Only exposes access token, never refresh token
     return { accessToken, user };
   }
   ```

### Performance Optimizations

- Implement request deduplication with TanStack Query
- Use React.lazy() for heavy visualization components
- Add virtual scrolling for large lists
- Optimize bundle with dynamic imports
- Use Next.js ISR for semi-static content

### Security Requirements

- httpOnly cookies for refresh tokens
- Secure flag in production
- CSRF protection via double-submit cookies
- Strict CSP headers
- API endpoint hiding via Next.js proxy