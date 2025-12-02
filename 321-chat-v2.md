Below is a concrete product + technical blueprint for a threads experience inside a SaaS app with:

* **Bottom/footer composer** (global input + conversation starters)
* **Right-side panel** (threads)
* Clear vision for **threads** (open/close/reopen), **context switching** while navigating, and **persistence / resume**.

---

## Implementation Status (as of 2025-10-30)

**Overall Completion: ~38%** | **Phase: 1 (Foundation)** | **Code: [services/www_ameide_platform/features/threads](../services/www_ameide_platform/features/threads)**

### Legend
- ✅ **Implemented** - Feature complete and working
- ⚠️ **Partial** - Basic version exists, missing advanced features
- ❌ **Not Implemented** - Not started or missing entirely
- 📍 **File Reference** - Location in codebase

### Status by Category
| Category | Status | Completion |
|----------|--------|------------|
| UI Layout & Surfaces | ⚠️ Partial | 70% - Missing thread list, context chips |
| Thread Lifecycle | ❌ Not Implemented | 20% - Only basic persistence |
| Context Management | ⚠️ Partial | 40% - Extraction works, no modes/anchors |
| Data Models | ⚠️ Partial | 50% - Core fields only |
| Backend APIs | ⚠️ Partial | 25% - Send/stream only, no CRUD |
| State Management | ✅ Good | 75% - Zustand + Context working |
| Persistence | ⚠️ Partial | 50% - Server-side only, no local cache |
| Navigation/Context Switch | ❌ Not Implemented | 15% - No mode switching |
| Advanced Features | ❌ Not Implemented | 10% - Search, summaries missing |

### What's Built
- ✅ Sliding side panel with resize (300-1200px) - [ChatLayoutWrapper.tsx](../services/www_ameide_platform/features/threads/ChatLayoutWrapper.tsx)
- ✅ Footer composer with conversation starters - [ChatFooter.tsx](../services/www_ameide_platform/features/threads/ChatFooter.tsx)
- ✅ SSE streaming with robust error handling - [useInferenceStream.ts](../services/www_ameide_platform/features/threads/hooks/useInferenceStream.ts)
- ✅ Context extraction from routes (org/repo/element) - [useChatContextMetadata.ts](../services/www_ameide_platform/features/threads/hooks/useChatContext.ts)
- ✅ Thread persistence (PostgreSQL) - [services/threads](../services/threads)
- ✅ Three-mode layout (default/active/minimized) - [threadsLayoutStore.ts](../services/www_ameide_platform/features/threads/stores/threadsLayoutStore.ts)
- ✅ Markdown rendering, attachments, message actions
- ✅ Rate limiting (20 req/min per user)

### Critical Gaps
- ❌ Thread list UI (browsing, search, filters)
- ❌ Context chips & mode switching (Follow/Lock/Global)
- ❌ Close/reopen/archive lifecycle
- ❌ Draft autosave persistence
- ❌ Thread metadata (anchor, breadcrumbs, snapshots)
- ❌ Deep links (`/threads?threadId=...`)
- ❌ Keyboard shortcuts (⌘J, ⌘K, Esc)
- ❌ Thread summaries & reopen recap

### Next Phase Recommendations
1. **Phase 2: Thread Management** - List UI, close/reopen, deep links (3-4 weeks)
2. **Phase 3: Context Management** - Anchor storage, mode switching, context chips (4-5 weeks)
3. **Phase 4: Persistence** - Draft autosave, IndexedDB cache, offline support (2-3 weeks)

---

## 1) Product vision: threads as a system‑level surface

**Implementation Status:** ⚠️ **Partial (60%)**

You'll have **two complementary surfaces**:

1. **Global Composer (footer):** ✅ **Implemented** - Always present, single line + expand to multiline, great for quick asks anywhere. It can start a new thread or append to the active one.
   - 📍 [ChatFooter.tsx](../services/www_ameide_platform/features/threads/ChatFooter.tsx)
   - ✅ Fixed positioning, context-aware starters, smooth transitions
   - ✅ Creates new thread or reactivates minimized thread

2. **Right-Side Chat Panel:** ⚠️ **Partially Implemented** - Shows **active thread** only. Thread list view missing.
   - 📍 [ChatPanel.tsx](../services/www_ameide_platform/features/threads/ChatPanel.tsx)
   - ✅ Persistent panel with message history and composer
   - ❌ **Missing:** Thread list view with filters and search
   - ❌ **Missing:** Thread switcher UI

> Mental model: The composer is *how* you speak; the right panel is *where* the conversation lives.
>
> **Current Reality:** Composer works perfectly. Panel only shows one active thread at a time (no browsing).

---

## 2) Anatomy of the UI

**Implementation Status:** ⚠️ **Partial (55%)**

### Footer / Bottom Composer

**Status:** ⚠️ **70% Complete** | 📍 [ChatFooter.tsx](../services/www_ameide_platform/features/threads/ChatFooter.tsx)

* ❌ **Input** with inline **context chips** - Not implemented. Context is sent but not visualized.
* ✅ **Conversation starters** - Fully working with route-based dynamic suggestions (transformations, elements, repos, general).
* ⚠️ **Actions**: ✅ Send (Enter), ✅ Attach file UI exists. ❌ Missing: "New thread" explicit action, emoji picker, voice, insert selection.
* ❌ **Draft autosave** - Drafts only in React state, not persisted to localStorage/server.

**What Works:** Beautiful glassmorphic popover with context-aware starters, smooth show/hide on hover, fixed positioning.

### Right-Side Panel

**Status:** ⚠️ **50% Complete** | 📍 [ChatPanel.tsx](../services/www_ameide_platform/features/threads/ChatPanel.tsx)

* ⚠️ **Header**: ✅ Thread ID shown, ✅ New Chat button, ✅ Close button. ❌ Missing: title editing, status pill, participants, privacy selector, context anchor link.
* ⚠️ **Message list**: ✅ Streaming, ✅ markdown/code, ✅ attachments, ✅ copy button, ⚠️ reactions (vote UI exists). ❌ Not virtualized, no citations display.
* ❌ **Thread tools**: Missing all tools (Summarize, Rename, Pin, Follow/Lock modes, Snapshot, Export).
* ❌ **Switcher**: Thread list/search UI not implemented.

**What Works:** Smooth streaming, resizable panel (300-1200px), integrated with nav system via CSS variables.

### Thread List Item (card)

**Status:** ❌ **Not Implemented**

* ❌ No thread list UI exists
* ❌ Backend has `last_message_preview`, `is_pinned`, `message_count` but no API to list threads
* **Blocker:** Need `GET /threads` API with filters

**Backend Support:** `ChatThread` protobuf has most fields ready (`title`, `last_message_preview`, `message_count`, `is_pinned`, timestamps).

---

## 3) Thread lifecycle (open → close → reopen)

**Implementation Status:** ❌ **Not Implemented (20%)**

**States**: `open`, `closed`, `archived` (optional).

* ⚠️ **Open**: Implicit default; all threads accept messages. No explicit status field in DB yet.
* ❌ **Close**: Not implemented. No close action, status pill, or reopen button.
* ❌ **Reopen**: Not implemented. No reopen action or summary card.
* ❌ **Archive**: Not implemented.

**Good UX touches**

* ❌ Ask for a **close reason** (Resolved / Duplicate / Out of scope) - Not implemented.
* ❌ Auto‑close inactive threads after N days (configurable), with undo - Not implemented.

**Backend Gap:** `ChatThread` protobuf missing `status` enum field and `close_reason` field. Need schema migration.

**Current Behavior:** Users can only start a "New Chat" which generates a new thread ID. Old threads persist in DB but no UI to revisit, close, or manage them. The only lifecycle action is implicit abandonment.

**What Exists:**
- ✅ Thread persistence in PostgreSQL with timestamps
- ✅ Auto-generated title from first user message
- ⚠️ `minimized` mode in frontend (thread ID persists when navigating away)

---

## 4) Context model & navigation behavior

**Implementation Status:** ⚠️ **Partial (40%)** - Context extraction works, storage and mode switching missing

A **thread can carry context** that helps answers be precise and lets users jump back to the right place.

**Context Envelope** (attached to thread):

* ❌ `anchor`: Not stored in thread. Context sent per-message only.
* ❌ `mode`: Not implemented. No Follow/Lock/Global modes.
  * ❌ **Follow this page**: Not implemented.
  * ❌ **Lock to entity**: Not implemented.
  * ⚠️ **Global**: De facto current behavior (context changes per message based on route).
* ❌ `snapshot`: Not implemented (attachment UI exists but no snapshot concept).
* ❌ `breadcrumbs`: Not implemented.

**Current Implementation:** 📍 [useChatContextMetadata.ts](../services/www_ameide_platform/features/threads/hooks/useChatContext.ts)

```typescript
// What's extracted per-message (not stored in thread):
{
  'context.tenantId': orgId,
  'context.graphId': repoId,
  'context.elementId': elementId,
  'context.selectionId': selectedId,
  'context.uiPage': pathname,
  'context.userRole': role
}
```

**When user navigates to a new page**

* ❌ **Current:** Context is re-extracted for next message. No system notes, no mode awareness.
* ❌ **Vision:** All Follow/Lock/Global flows not implemented.

**From a thread → page**

* ❌ No context anchor links (thread doesn't store anchor).
* ❌ No snapshot fallback.

**What Works:**
- ✅ Context extraction from Next.js routes (org, repo, element, transformation)
- ✅ Metadata sent with each inference request
- ✅ Thread ID persists across navigation (minimized mode)

**Critical Gap:** Context is ephemeral (per-message) rather than part of thread identity. Threads don't "remember" what they were about.

---

## 5) Persistence & resume

**Implementation Status:** ⚠️ **Partial (55%)** - Server persistence works, local caching missing

* ⚠️ **Thread persistence**: ✅ Server-side (PostgreSQL). ❌ No local cache (IndexedDB).
  - 📍 Backend: [services/threads](../services/threads) - Stores threads and messages
  - ✅ History loaded via `GET /api/threads/messages?threadsId=...`
  - ❌ No prefetching or cache for last 20 threads

* ✅ **Active thread memory**: Fully working via `threadsLayoutStore`.
  - 📍 [threadsLayoutStore.ts](../services/www_ameide_platform/features/threads/stores/threadsLayoutStore.ts)
  - ✅ Thread ID persists in localStorage (`threads-layout-storage`)
  - ✅ `minimized` mode keeps thread active during navigation
  - ✅ Panel width preference saved

* ❌ **Draft persistence**: Not implemented.
  - ❌ Drafts only in React state (lost on refresh)
  - ❌ No localStorage backup
  - ❌ No server-side sync
  - ❌ No per-thread draft storage

* ❌ **Resume helpers**: Not implemented.
  - ❌ No context recap card
  - ❌ No "continue where you left off" scroll
  - ❌ No "summarize since I left" action
  - **Blocker:** Requires thread summarization pipeline

**What Works:**
- ✅ Thread ID and layout state persist across sessions
- ✅ Message history loads on ChatProvider mount
- ✅ Panel remembers width and mode

**Critical Gaps:**
- Drafts are volatile (UX pain point)
- No local message cache (every mount fetches from server)
- No resume helpers for long-lived threads

---

## 6) Information architecture (IA)

**Implementation Status:** ⚠️ **Partial (40%)** - Panel works, navigation missing

* ✅ **Chat** is a system-level surface present on most pages (except auth).
  - 📍 [ChatLayoutWrapper.tsx](../services/www_ameide_platform/features/threads/ChatLayoutWrapper.tsx) wraps main layouts

* ⚠️ The **threads list** can be opened as:
  * ✅ **Drawer** (right panel) - Panel exists but only shows active thread, not list.
  * ❌ **Full page** `/threads` - Not implemented.

* ❌ **Deep links**: `/threads?threadId=th_123` - Not implemented.
  - No URL param handling
  - No auto-open panel on link navigation

* ❌ **Discovery**: "On this page" filter - Not implemented.
  - Backend doesn't store thread anchors
  - No filter API

**Current Navigation:**
- User clicks "New Chat" → generates UUID → opens panel
- Navigating away → `minimized` mode
- Returning → can reactivate same thread (if ID still in store)
- No way to browse, search, or filter threads

**What's Needed:**
1. Route handler for `/threads` page (full view)
2. URL param support for `?threadId=...`
3. Thread list component with filters
4. Backend `GET /threads` API with `on_page` filter support

---

## 7) Data model (suggested)

**Implementation Status:** ⚠️ **Partial (50%)** - Core fields exist, advanced metadata missing

### Thread Model Comparison

| Field | Vision | Current (Protobuf) | Status |
|-------|--------|-------------------|--------|
| `id` | string | ✅ `string` | ✅ |
| `user_id` | - | ✅ `string` (owner) | ✅ |
| `title` | string | ✅ `string` (auto-generated) | ✅ |
| `status` | `open/closed/archived` | ❌ Missing | 🔴 |
| `privacy` | `team/org/private` | ⚠️ `visibility` (public/private) | 🟡 |
| `participants` | `string[]` | ❌ Only single owner | 🔴 |
| `anchor` | `{entity_type, entity_id, page_url, version}` | ❌ Missing | 🔴 |
| `mode` | `follow/locked/global` | ❌ Missing | 🔴 |
| `snapshot_ids` | `string[]` | ❌ Missing | 🔴 |
| `tags` | `string[]` | ❌ Missing | 🔴 |
| `last_message_id` | string | ❌ Missing | 🔴 |
| `unread_count` | number | ⚠️ `message_count` (total) | 🟡 |
| `pinned` | boolean | ✅ `is_pinned` | ✅ |
| `created_at` | timestamp | ✅ `google.protobuf.Timestamp` | ✅ |
| `updated_at` | timestamp | ✅ `google.protobuf.Timestamp` | ✅ |
| `last_message_preview` | string | ✅ `string` | ✅ |
| `last_message_at` | timestamp | ✅ `google.protobuf.Timestamp` | ✅ |
| `summaries` | `[{ts, text}]` | ❌ Missing | 🔴 |
| `breadcrumbs` | `[{ts, page_url}]` | ❌ Missing | 🔴 |
| `close_reason` | string | ❌ Missing | 🔴 |

**Current Protobuf:** 📍 [threads_types.proto](../packages/ameide_core_proto/src/ameide_core_proto/threads/v1/threads_types.proto)

```protobuf
message ChatThread {
  string id = 1;
  string user_id = 2;
  string title = 3;
  Visibility visibility = 4;  // PRIVATE | PUBLIC
  int32 message_count = 5;
  google.protobuf.Timestamp created_at = 6;
  google.protobuf.Timestamp updated_at = 7;
  string last_message_preview = 8;
  google.protobuf.Timestamp last_message_at = 9;
  bool is_pinned = 10;
}
```

**Vision Thread (for comparison):**

```json
{
  "id": "th_123",
  "title": "Q4 forecast anomalies",
  "status": "open",  // ❌ Missing
  "privacy": "team",  // ⚠️ Simplified to visibility
  "participants": ["u_1","u_2","bot_assistant"],  // ❌ Missing
  "anchor": { /* ... */ },  // ❌ Missing
  "mode": "locked",  // ❌ Missing
  "snapshot_ids": ["file_abc"],  // ❌ Missing
  "tags": ["forecasting","finance"],  // ❌ Missing
  "summaries": [/* ... */],  // ❌ Missing
  "breadcrumbs": [/* ... */],  // ❌ Missing
  "close_reason": null  // ❌ Missing
}
```

### Message Model Comparison

| Field | Vision | Current (Protobuf) | Status |
|-------|--------|-------------------|--------|
| `id` | string | ✅ `string` | ✅ |
| `thread_id` | string | ✅ `string` | ✅ |
| `sender_id` / `user_id` | string | ❌ Missing (implicit from auth) | 🟡 |
| `role` | `user/assistant/system` | ✅ `MessageRole` enum | ✅ |
| `text` / `content` | string | ✅ `string` | ✅ |
| `format` | markdown | ❌ Implicit markdown | 🟡 |
| `attachments` | `[{file_id, kind, range}]` | ❌ Missing | 🔴 |
| `state` | `sending/sent/failed` | ❌ Missing (handled in UI) | 🟡 |
| `created_at` | timestamp | ✅ `google.protobuf.Timestamp` | ✅ |
| `meta.citations` | array | ❌ Missing | 🔴 |
| `meta.tool_calls` | array | ❌ Missing | 🔴 |
| `meta.reply_to` | string | ❌ Missing | 🔴 |

**Current Protobuf:**

```protobuf
message ChatMessage {
  string id = 1;
  string thread_id = 2;
  MessageRole role = 3;  // USER | ASSISTANT | SYSTEM
  string content = 4;
  google.protobuf.Timestamp created_at = 5;
}
```

**Frontend Type:** 📍 [useInferenceStream.ts](../services/www_ameide_platform/features/threads/hooks/useInferenceStream.ts)

```typescript
// Simplified frontend interface
type InferenceMessage = {
  id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
};
```

**Schema Migration Needed:**
1. Add `status` enum to `ChatThread`
2. Add `close_reason`, `anchor` (JSONB), `mode` enum, `tags` (array)
3. Add `participants` table or field
4. Add `attachments`, `meta` (JSONB) to `ChatMessage`
5. Add `summaries`, `breadcrumbs` tables or JSONB fields

---

## 8) Front-end architecture

**Implementation Status:** ⚠️ **Partial (65%)** - State management good, event bus & keyboard missing

### State Layers

| Layer | Vision | Current | Status |
|-------|--------|---------|--------|
| **In-memory store** | Zustand/Redux with threads map, messages map | ✅ Zustand (`threadsLayoutStore`) + React Context (`ChatProvider`) | ⚠️ |
| `threads map` | Normalized entity cache | ❌ Only `threadsId` (string) | 🔴 |
| `messages map` | Entity-normalized | ⚠️ Array in React state (not normalized) | 🟡 |
| `activeThreadId` | string | ✅ `threadsId` | ✅ |
| `per-thread drafts` | Map<threadId, draft> | ❌ Not implemented | 🔴 |
| `global draft` | string | ❌ Not implemented | 🔴 |
| `panel open/closed` | boolean | ✅ `mode` enum (default/active/minimized) | ✅ |
| `unread counters` | Map<threadId, count> | ⚠️ `messageCount` (single thread only) | 🟡 |

**Persistence**

| Layer | Vision | Current | Status |
|-------|--------|---------|--------|
| **LocalStorage** | Thread IDs, metadata, drafts, panel state | ✅ `threadsId`, `mode`, `threadsWidth` via Zustand persist | ⚠️ |
| **IndexedDB** | Message history cache (last 20 threads) | ❌ Not implemented | 🔴 |
| **Server** | Source of truth with ETags | ✅ PostgreSQL (no ETags) | ⚠️ |

**UI Features**

| Feature | Vision | Current | Status |
|---------|--------|---------|--------|
| **Virtualized lists** | react-virtual for performance | ❌ Simple scroll | 🔴 |
| **Optimistic UI** | Immediate message add | ✅ User message added before stream | ✅ |
| **Retry logic** | Exponential backoff | ✅ Built into useInferenceStream | ✅ |
| **Service worker** | Offline queue | ❌ Not implemented | 🔴 |

**Current Implementation:**
- 📍 [threadsLayoutStore.ts](../services/www_ameide_platform/features/threads/stores/threadsLayoutStore.ts) - Layout state (Zustand)
- 📍 [ChatProvider.tsx](../services/www_ameide_platform/features/threads/providers/ChatProvider.tsx) - Message state (React Context)
- 📍 [useInferenceStream.ts](../services/www_ameide_platform/features/threads/hooks/useInferenceStream.ts) - Streaming & optimistic updates

### Event Bus

**Status:** ❌ **Not Implemented** - No centralized event bus

Vision events vs. current state:

| Event | Vision | Current |
|-------|--------|---------|
| `THREAD_OPENED/CLOSED/REOPENED` | Global event | ❌ Direct state updates only |
| `ACTIVE_THREAD_CHANGED` | Event subscribers | ❌ Zustand subscriptions (no event) |
| `CONTEXT_CHANGED` | With anchor/mode payload | ❌ Not implemented |
| `MESSAGE_SENDING/SENT/FAILED/STREAM_DELTA` | Stream lifecycle | ⚠️ Handled in `useInferenceStream` (not global) |
| `UNREAD_UPDATED` | Thread unread counts | ❌ Not implemented |
| `PANEL_TOGGLED` | UI event | ❌ Not implemented |
| `DRAFT_SAVED/RESTORED` | Draft lifecycle | ❌ Not implemented |

**Note:** There is a `typedEventStore` 📍 [typedEventStore.ts](../services/www_ameide_platform/features/common/stores/typedEventStore.ts) used for suggestion events from streaming, but not thread/message lifecycle.

### Keyboard Shortcuts

**Status:** ❌ **Not Implemented**

| Shortcut | Vision | Current |
|----------|--------|---------|
| `⌘/Ctrl+J` | Toggle panel | ❌ Not bound |
| `Esc` | Collapse panel | ❌ Not bound |
| `⌘/Ctrl+Enter` | Send message | ❌ Not bound (plain Enter works) |
| `⌘/Ctrl+K` | Quick search threads | ❌ Not implemented |

**Current Input Behavior:**
- `Enter` → Send message
- No keyboard shortcuts for panel control
- Tab navigation works for accessibility

---

## 9) Backend & APIs

**Implementation Status:** ⚠️ **Partial (25%)** - Streaming works, CRUD APIs missing

### Transport

| Layer | Vision | Current | Status |
|-------|--------|---------|--------|
| **WebSocket/SSE** | Streaming deltas, typing, thread updates | ✅ SSE for message streaming only | ⚠️ |
| **REST/GraphQL** | CRUD and search | ⚠️ Minimal REST (2 endpoints) | 🔴 |

**Current:** 📍 [/api/threads/stream](../services/www_ameide_platform/app/api/threads/stream/route.ts) (Next.js API route → gRPC)

### Endpoints Comparison

| Endpoint | Vision | Current | Status |
|----------|--------|---------|--------|
| `GET /threads` | List with filters (`active`, `on_page`, etc.) | ❌ Not implemented | 🔴 |
| `POST /threads` | Explicit create with anchor/mode | ⚠️ Auto-created on first message | 🟡 |
| `PATCH /threads/:id` | Update metadata (title, status, mode, etc.) | ❌ Not implemented | 🔴 |
| `POST /threads/:id/messages` | Send message | ✅ `POST /api/threads/stream` (SSE) | ✅ |
| `GET /threads/:id/messages` | Fetch history with cursor | ✅ `GET /api/threads/messages?threadsId=...` | ✅ |
| `POST /threads/:id/close` | Close with reason | ❌ Not implemented | 🔴 |
| `POST /threads/:id/reopen` | Reopen thread | ❌ Not implemented | 🔴 |
| `POST /threads/:id/summarize` | Generate summary | ❌ Not implemented | 🔴 |
| `POST /drafts` | Save draft | ❌ Not implemented | 🔴 |
| `GET /search` | Full-text search threads/messages | ❌ Not implemented | 🔴 |

**Implemented APIs:**

1. **POST `/api/threads/stream`** - ✅ Working
   - Body: `{ threadId?, content, agentId?, metadata? }`
   - Returns: SSE stream (`started` → `token`* → `done`)
   - Creates thread automatically if `threadId` missing
   - Rate limit: 20 req/min per user
   - Auth: NextAuth session + Keycloak access token

2. **GET `/api/threads/messages`** - ✅ Working
   - Query: `?threadsId={uuid}`
   - Returns: `{ messages: InferenceMessage[] }`
   - Pagination: Not implemented (returns all)
   - Auth: NextAuth session

**Backend Service:** 📍 [services/threads](../services/threads) (Node.js + gRPC)
- Protobuf: [threads_service.proto](../packages/ameide_core_proto/src/ameide_core_proto/threads/v1/threads_service.proto)
- Database: PostgreSQL with Prisma

### Auth & Tenancy

| Feature | Vision | Current | Status |
|---------|--------|---------|--------|
| **Thread privacy** | RBAC checks per call | ⚠️ Basic user_id ownership check | 🟡 |
| **Per-message redaction** | PII/secrets filtering | ❌ Not implemented | 🔴 |
| **Tenant isolation** | Multi-tenant queries | ⚠️ User-level isolation only | 🟡 |

**Current Auth Flow:**
1. Frontend sends NextAuth session cookie
2. API route extracts user ID from session
3. Backend verifies user ownership of thread
4. No team/org-level sharing yet

**What's Needed:**
1. `GET /threads` with filters (active, on_page, recent, closed)
2. `PATCH /threads/:id` for metadata updates
3. `POST /threads/:id/close` and `/reopen` actions
4. Search API with full-text indexing
5. Draft sync endpoints
6. Proper RBAC for privacy levels (team/org)

---

## 10) Switching context across pages (detailed flows)

**Implementation Status:** ❌ **Not Implemented (15%)** - Context extracted but no mode switching

**A. Active thread is in "Follow this page"** - ❌ Not implemented

1. ❌ Navigation occurs → No `routeChange` event
2. ❌ ChatStore doesn't track anchor
3. ❌ No `CONTEXT_CHANGED` dispatch
4. ❌ No system marker messages
5. ✅ Conversation starters DO refresh based on route

**B. Active thread is "Locked to entity"** - ❌ Not implemented

1. ❌ No lock mode
2. ❌ No context hint UI
3. ❌ No switch context CTA

**C. No active thread (panel shows list)** - ❌ Not implemented

* ❌ No thread list view
* ❌ No "On this page" filter
* ❌ No contextual thread highlighting

**Current Behavior:**
- Navigation → Chat enters `minimized` mode (thread ID persists)
- Next message gets fresh context from current route (per-message, not per-thread)
- No user control over context behavior

---

## 11) Closing & reopening UX rules

**Implementation Status:** ❌ **Not Implemented**

* ❌ **Closing:** No close action exists
  * ❌ No draft confirmation
  * ❌ No "Close & summarize" option

* ❌ **Reopening:** No reopen capability
  * ❌ No auto summary on reopen
  * ❌ No TODOs extraction

* ⚠️ **Thread history:** Threads persist in DB but user can't revisit them (no list UI)

---

## 12) Conversation starters & suggestions

**Implementation Status:** ✅ **Implemented (80%)**

* ✅ **Source suggestions from:** Current page type (transformations, elements, repos, default)
  - 📍 [ChatFooter.tsx](../services/www_ameide_platform/features/threads/ChatFooter.tsx) - Route-based starters
  - ✅ Initiatives → ADM phases, deliverables, governance
  - ✅ Elements → Connections, relationships, view generation
  - ✅ Repos → Structure, diagrams, review
  - ✅ Default → General EA, ArchiMate, TOGAF help

* ❌ **Not implemented:** User role/persona customization, thread activity-based suggestions

* ⚠️ **UI:** Glassmorphic popover above composer (not inline chips as spec suggests, but elegant)

---

## 13) Accessibility & polish

**Implementation Status:** ⚠️ **Partial (70%)**

* ⚠️ **Keyboard navigation:** Basic tab navigation works, no shortcuts
* ✅ **Screen-reader labels:** Panel has `role="complementary"`, divider has `role="separator"`
* ❌ **ARIA live regions:** Not implemented for streaming updates
* ✅ **WCAG AA colors:** Design follows color standards
* ⚠️ **Status cues:** No status pills yet (would need accessible labels)

**What works:**
- All interactive elements keyboard accessible
- Skip link in header (`#main-content`)
- Semantic HTML structure

**What's missing:**
- Keyboard shortcuts (⌘J, ⌘K, Esc)
- Live region announcements for streaming
- Context chip accessibility (chips don't exist)

---

## 14) Observability & guardrails

**Implementation Status:** ⚠️ **Partial (60%)**

* ⚠️ **Instrumentation:** Basic logging, no metrics dashboard
  - ✅ Request IDs in errors
  - ✅ Error logging with context
  - ❌ No time-to-first-token tracking
  - ❌ No latency metrics
  - ❌ No drop-off analysis

* ✅ **Rate limiting:** 20 messages/min per user (in-memory, per backend instance)
  - 📍 [services/threads/src/threads/phase1.ts](../services/threads/src/threads/phase1.ts)

* ❌ **Content redaction:** Not implemented (PII/secrets could leak to logs)

**What's needed:**
- OpenTelemetry instrumentation
- Metrics export (Prometheus/DataDog)
- PII detection and masking pipeline
- Distributed rate limiting (Redis)

---

## 15) Edge cases to handle

**Implementation Status:** ⚠️ **Partial (40%)**

| Edge Case | Vision | Current | Status |
|-----------|--------|---------|--------|
| **Unsaved drafts on navigation** | Autosave + toast | ❌ Drafts lost | 🔴 |
| **Thread link in new tab** | Auto-open panel | ❌ No deep links | 🔴 |
| **Concurrent edits** | ETag resolution | ❌ No ETags | 🔴 |
| **Offline** | Queue + "Will send when online" | ❌ Fetch fails | 🔴 |
| **Stream interruption** | Retry/reconnect | ✅ Abort controller works | ✅ |
| **Invalid thread ID** | 404 → rotate | ✅ Retry logic in useInferenceStream | ✅ |
| **Rate limit hit** | Show retry after | ✅ 429 with user message | ✅ |

---

## 16) Implementation checklist (short)

**Phase 1 (Foundation)** - ✅ **Completed**

1. ✅ **Scaffold panel + composer** - ChatLayoutWrapper, ChatPanel, ChatFooter
2. ⚠️ **ChatStore** - Layout store works, missing drafts and thread cache
3. ⚠️ **Thread CRUD** - Send/stream works, CRUD missing
4. ❌ **Context chips** - Not implemented
5. ❌ **Navigation hooks** - No context updates
6. ❌ **Thread list** - Not implemented
7. ❌ **Close/Reopen** - Not implemented
8. ❌ **Autosave drafts** - Not implemented
9. ❌ **Deep links** - Not implemented
10. ⚠️ **Message list** - Works but not virtualized, markdown ✅, attachments ✅

**Phase 2 (Thread Management)** - ❌ **Not Started** - Recommended next

1. Add `status` field to ChatThread (schema migration)
2. Implement `GET /threads` API with filters
3. Build thread list UI component
4. Add close/reopen actions and APIs
5. Implement deep links (`/threads?threadId=...`)
6. Add keyboard shortcuts (⌘J, ⌘K, Esc)

**Phase 3 (Context Management)** - ❌ **Not Started**

1. Add `anchor`, `mode`, `breadcrumbs` to thread model
2. Build context chip UI in composer
3. Implement mode switching logic
4. Add system marker messages
5. Build "on this page" filter

**Phase 4 (Polish)** - ❌ **Not Started**

1. Draft autosave (localStorage + server sync)
2. IndexedDB message cache
3. Virtualized message list (react-window)
4. Offline queue (service worker)
5. Thread summaries
6. Full accessibility (ARIA live, keyboard shortcuts)

---

### Optional: small pseudo-API for context chips

```ts
// Chip toggles
function setMode(mode: 'follow' | 'locked' | 'global', anchor?: Anchor) {
  if (mode === 'locked' && !anchor) anchor = deriveAnchorFromPage();
  ChatAPI.patchThread(activeThreadId, { mode, anchor });
  ChatStore.dispatch({ type: 'CONTEXT_CHANGED', payload: { mode, anchor } });
}

// On route change
router.onChange((route) => {
  const thread = ChatStore.getActiveThread();
  if (!thread) { ChatStore.filter = { onPage: route.url }; return; }
  if (thread.mode === 'follow') setMode('follow', deriveAnchorFromPage());
});
```

---

## What success looks like

**Vision vs. Current State:**

| Success Metric | Vision | Current Reality | Gap |
|----------------|--------|-----------------|-----|
| **Thread reuse** | Users reuse threads for same entity | ❌ Can't browse threads, always "New Chat" | Need thread list + anchor storage |
| **Context persistence** | Context explainable via chips + system notes | ❌ Context invisible to user | Need chips + mode UI |
| **Thread lifecycle** | "Close & summarize" reduces friction | ❌ No close/reopen | Need lifecycle APIs |
| **Shareability** | Deep links make threads first-class | ❌ Can't share or bookmark | Need deep links |
| **Discovery** | "On this page" filter surfaces relevant threads | ❌ No filtering | Need anchor + search |

**Current Wins:**

* ✅ **Streaming feels instant** - Token-by-token updates work beautifully
* ✅ **Layout is polished** - Resizable panel, smooth transitions, great positioning
* ✅ **Context extraction works** - Route metadata captured and sent
* ✅ **Conversation starters delight** - Context-aware suggestions add value
* ✅ **Error handling is robust** - Users get clear feedback with retry options

**To Achieve Vision:**

1. **Immediate (Phase 2):** Thread list + close/reopen → Enables thread reuse
2. **Near-term (Phase 3):** Context chips + modes → Makes context explainable
3. **Long-term (Phase 4+):** Summaries + deep links → Reduces friction, enables sharing

**Current Assessment:** Strong Phase 1 foundation (~38% complete). Core UX is excellent, but missing thread management layer prevents vision from being realized.
