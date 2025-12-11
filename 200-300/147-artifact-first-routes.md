# Artifact-First Route Architecture

## Status: Planning
**Created**: 2025-01-22
**Priority**: High
**Complexity**: Medium
**Related**: 
- [145-threads-artifact.md](./145-threads-artifact.md) - Backend gRPC architecture
- [146-ai-backend-migration.md](./146-ai-backend-migration.md) - AI backend integration

## Problem Statement

Users lose context when navigating between artifacts and threadss due to:
1. URL instability - Pages redirect or rewrite URLs
2. Lost state on refresh - Artifact pages redirect to threads URLs
3. Confusing navigation - Artifacts subordinate to threadss
4. Race conditions - Multiple tabs can create duplicate threadss

## Solution: Stable URL Architecture

### Core Principle: The "No URL Mutations" Rule

**CRITICAL INVARIANT**: Once a user navigates to a URL, that URL NEVER changes in the browser bar until they explicitly navigate away.

```typescript
// ❌ FORBIDDEN - These patterns are banned:
router.replace(`/threads/${threadsId}`);
window.history.replaceState(...);
router.push(`/threads/${threadsId}`, { replace: true });

// ✅ ALLOWED - Only explicit navigation:
<Link href={`/threads/${threadsId}`}>View Chat</Link>
router.push(`/threads/${threadsId}`);  // New navigation only
```

### URL Contract

**Routes** (singular, permanent):
- `/artifact/[artifactId]` - Artifact viewer with integrated threads panel
- `/threads/[threadsId]` - Chat viewer with artifact navigator

**Hard Invariants**:
- URL displayed in browser NEVER changes while interacting on a page
- Sending first message from `/artifact/[id]` creates threads without URL change
- All cross-entity navigation uses explicit links (new tab or user click)
- Server-side 308 redirect from `/artifacts/*` → `/artifact/*` (migration only)

## Frontend Implementation

### Artifact Viewer Page

`/app/artifact/[artifactId]/page.tsx`:

```typescript
export default async function ArtifactPage({ 
  params 
}: { 
  params: { artifactId: string } 
}) {
  const { artifactId } = params;  // params is synchronous in Next.js
  const session = await auth();
  
  if (!session) {
    redirect('/login');
  }
  
  return (
    <ArtifactProvider artifactId={artifactId}>
      <ChatProvider>
        <ArtifactViewer 
          artifactId={artifactId}
          session={session}
        />
      </ChatProvider>
    </ArtifactProvider>
  );
}
```

**Key Features:**
- Load artifact from gRPC backend via SDK
- Show sidebar with associated threads sessions
- Create new threads atomically when sending first message
- **Stay on `/artifact/[id]` URL always** (no mutations)
- Support switching between related threadss

### Chat Page

`/app/(threads)/threads/[id]/page.tsx`:

```typescript
export default async function ChatPage({ 
  params 
}: { 
  params: { id: string } 
}) {
  const { id } = params;  // params is synchronous in Next.js
  const session = await auth();
  
  if (!session) {
    redirect('/login');
  }
  
  return (
    <ChatProvider threadsId={id}>
      <ArtifactProvider>
        <ChatViewer 
          threadsId={id}
          session={session}
        />
      </ArtifactProvider>
    </ChatProvider>
  );
}
```

**Key Features:**
- Load threads and messages from gRPC backend
- Show artifact navigator with all related artifacts
- Support switching between artifacts
- **Stay on `/threads/[id]` URL always** (no mutations)
- Real-time updates via streaming

## Atomic Chat Creation from Artifact

The key to stable URLs is the atomic RPC that creates a threads and sends the first message in one operation:

```typescript
// In ArtifactViewer component
const sendFirstMessage = async (content: string) => {
  // This atomic operation prevents race conditions
  const response = await client.threadsCommands.ensureChatForArtifactAndSendMessage({
    artifactId,
    message: {
      role: 'user',
      parts: [{ text: content }]
    },
    idempotencyKey: crypto.randomUUID(),
  });
  
  // Update UI to show new threads in sidebar
  // BUT DO NOT CHANGE THE URL
  setSidebarChatId(response.threadsId);
  
  if (response.threadsCreated) {
    // Add new threads to sidebar list
    addChatToSidebar({
      id: response.threadsId,
      title: `Discussion: ${artifact.title}`
    });
  }
};
```

## Navigation Patterns

### Cross-Entity Navigation

```typescript
// From artifact to threads - explicit user action
<Link 
  href={`/threads/${threadsId}`}
  target="_blank"  // Optional: open in new tab
>
  Open Full Chat
</Link>

// From threads to artifact - explicit user action
<Link 
  href={`/artifact/${artifactId}`}
  target="_blank"
>
  View Artifact
</Link>
```

### Chat/Artifact Switching

```typescript
// Switch between threadss in artifact view
const switchChat = (threadsId: string) => {
  // Update displayed messages
  setActiveChatId(threadsId);
  // DO NOT change URL
};

// Switch between artifacts in threads view
const switchArtifact = (artifactId: string) => {
  // Update artifact preview
  setActiveArtifactId(artifactId);
  // DO NOT change URL
};
```

## Implementation Guidelines

### DO:
- Use the atomic `ensureChatForArtifactAndSendMessage` RPC
- Keep URLs stable throughout user interaction
- Use explicit navigation for route changes
- Store UI state (active threads/artifact) separately from URL

### DON'T:
- Use `router.replace()` or `window.history.replaceState()`
- Redirect after creating a threads
- Mutate URLs based on user actions
- Mix URL state with UI state

## Migration Strategy

### Phase 1: Fix Artifact Pages
1. Remove redirect from `/artifacts/[artifactId]/page.tsx`
2. Implement proper artifact viewer with threads sidebar
3. Use atomic RPC for threads creation

### Phase 2: Update Chat Pages
1. Add artifact navigator to threads view
2. Enable artifact switching without URL change
3. Implement bidirectional relationships

### Phase 3: Clean Up
1. Remove all `router.replace()` calls
2. Audit for `replaceState` usage
3. Add server-side redirects for old URLs

### Server Redirects (Example)

Implement a single 308 from legacy plural paths to singular in `middleware.ts`:

```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const url = new URL(request.url)
  if (url.pathname.startsWith('/artifacts/')) {
    url.pathname = url.pathname.replace('/artifacts/', '/artifact/')
    return NextResponse.redirect(url, { status: 308 })
  }
  return NextResponse.next()
}
```

## Success Metrics

- **URL Stability**: 100% of page refreshes maintain same URL
- **No Lost Context**: Refresh always shows same content
- **No Duplicates**: Zero duplicate threadss from race conditions
- **User Satisfaction**: Reduced confusion from URL changes

## Technical Dependencies

- Backend atomic RPC: `EnsureChatForArtifactAndSendMessage`
- SDK support for atomic operations
- Bidirectional threads-artifact relationships
- Idempotency keys for safe retries

## Related Documents

- [145-threads-artifact.md](./145-threads-artifact.md) - Backend gRPC architecture
- [146-ai-backend-migration.md](./146-ai-backend-migration.md) - AI backend integration

## Acceptance Criteria

- Artifact page (`/artifact/[id]`) never mutates the URL during use, including first message send.
- Chat page (`/threads/[id]`) never mutates the URL during use.
- Atomic RPC `EnsureChatForArtifactAndSendMessage` is used from artifact view to prevent race conditions.
- No usages of `router.replace` or `window.history.replaceState` in pages/components.
- Server 308 redirect exists for legacy `/artifacts/*` to `/artifact/*`.
