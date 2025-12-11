# Chat and Artifact gRPC Architecture

## Status: PLANNING
**Created**: 2025-01-21
**Priority**: High
**Complexity**: Large
**Related**: [146-ai-backend-migration.md](./146-ai-backend-migration.md)

## Problem Statement

The current implementation has critical UX and architectural issues:

1. **URL Instability**: Navigating to `/artifact/[id]` triggers redirects or client-side URL rewrites
2. **Lost Context**: Refreshing artifact pages loses state and redirects to threads URLs
3. **Non-Atomic Operations**: Sending first message from artifact requires multiple API calls (create threads, then send)
4. **Confusing Navigation**: Artifacts aren't truly first-class - they're always subordinate to threadss
5. **Race Conditions**: Multi-tab usage or network retries can create duplicate threadss
6. **Direct Database Access**: Frontend directly queries PostgreSQL, violating microservices principles
7. **No Audit Trail**: No event sourcing or proper history tracking

## Solution Overview

Make artifacts true first-class citizens with stable URLs and atomic threads creation, eliminating all URL manipulation and race conditions. All data operations go through gRPC backend services accessed via the SDK.

### Core Requirements

1. **Stable URLs**: `/artifact/[id]` and `/threads/[id]` never change during interaction
2. **Atomic Operations**: Single API call to create-threads-and-send-message from artifact page
3. **Bidirectional Navigation**: Both entities can reference and navigate to each other
4. **No URL Tricks**: Zero client-side URL manipulation (no replaceState, pushState, or redirects)
5. **Audit Trail**: Maintain a full event history for state changes
6. **Multi-tenancy**: Support for tenant isolation throughout

### Frontend Requirements

To support the artifact-first UX, the frontend must:
- Call the atomic `EnsureChatForArtifactAndSendMessage` RPC when sending messages from artifact pages
- Maintain stable URLs without client-side mutations
- Handle streaming with resume tokens for reconnection

Detailed routing patterns are documented in [147-artifact-first-routes.md](./147-artifact-first-routes.md).

## Technical Design

### 1. Chat Service Proto Definitions

Create new protobuf definitions at `/packages/ameide_core_proto/proto/ameide_core_proto/threads/v1/`:

```protobuf
// threads_service.proto
service ChatCommandService {
  // Atomic operation for artifact-first UX - THE KEY TO STABLE URLs
  rpc EnsureChatForArtifactAndSendMessage(EnsureChatForArtifactAndSendMessageRequest) 
    returns (EnsureChatForArtifactAndSendMessageResponse);
  
  rpc CreateChat(CreateChatRequest) returns (CreateChatResponse);
  rpc SendMessage(SendMessageRequest) returns (SendMessageResponse);
  rpc UpdateChatTitle(UpdateChatTitleRequest) returns (UpdateChatTitleResponse);
  rpc DeleteChat(DeleteChatRequest) returns (DeleteChatResponse);
  rpc AssociateArtifact(AssociateChatArtifactRequest) returns (AssociateChatArtifactResponse);
  rpc UpdateVisibility(UpdateChatVisibilityRequest) returns (UpdateChatVisibilityResponse);
}

service ChatQueryService {
  rpc GetChat(GetChatRequest) returns (GetChatResponse);
  rpc ListChats(ListChatsRequest) returns (ListChatsResponse);
  rpc GetMessages(GetMessagesRequest) returns (GetMessagesResponse);
  rpc GetChatArtifacts(GetChatArtifactsRequest) returns (GetChatArtifactsResponse);
  rpc SearchChats(SearchChatsRequest) returns (SearchChatsResponse);
  rpc StreamChatUpdates(StreamChatRequest) returns (stream ChatUpdate);
}

// Atomic operation request - critical for artifact-first UX
message EnsureChatForArtifactAndSendMessageRequest {
  string artifact_id = 1;        // Required: the artifact we're discussing
  string threads_id = 2;            // Optional: if user selected existing threads
  ChatMessageInput message = 3;  // The message to send
  string idempotency_key = 4;    // For safe retries
  ChatVisibility visibility = 5; // Optional visibility override
  string user_id = 6;            // Filled by auth layer
  string tenant_id = 7;          // For multi-tenancy
}

message EnsureChatForArtifactAndSendMessageResponse {
  string threads_id = 1;            // The threads used (created or existing)
  ChatMessage persisted_message = 2;  // The saved message
  bool threads_created = 3;         // Whether a new threads was created
}

// Pagination for all list operations
message ListChatsRequest { 
  int32 page_size = 1;     // Default: 20, Max: 100
  string page_token = 2;    // From previous response
  string user_id = 3;
  string tenant_id = 4;
  ChatFilter filter = 5;   // Optional filters
}

message ChatFilter {
  string query = 1;              // Full-text search
  ChatVisibility visibility = 2;
  repeated string artifact_ids = 3;
  google.protobuf.Timestamp created_after = 4;
  google.protobuf.Timestamp created_before = 5;
}

message ListChatsResponse { 
  repeated Chat threadss = 1; 
  string next_page_token = 2;  // Empty if no more pages
  int32 total_count = 3;       // Total matching records
}

message GetMessagesRequest {
  string threads_id = 1;
  int32 page_size = 2;     // Default: 50, Max: 200
  string page_token = 3;    // For pagination
  string tenant_id = 4;
}

message GetMessagesResponse {
  repeated ChatMessage messages = 1;
  string next_page_token = 2;
  bool has_more = 3;
  int32 total_count = 4;  // Total messages in threads
}

message SearchChatsRequest {
  string query = 1;              // Search query
  int32 page_size = 2;           // Default: 20, Max: 100
  string page_token = 3;         // For pagination
  string user_id = 4;
  string tenant_id = 5;
  ChatFilter filter = 6;         // Additional filters
}

message SearchChatsResponse {
  repeated Chat threadss = 1;
  string next_page_token = 2;
  int32 total_count = 3;
}

message GetChatRequest {
  string threads_id = 1;
  string tenant_id = 2;
}

message GetChatResponse {
  Chat threads = 1;
}

message GetChatArtifactsRequest {
  string threads_id = 1;
  int32 page_size = 2;
  string page_token = 3;
  string tenant_id = 4;
}

message GetChatArtifactsResponse {
  repeated ArtifactReference artifacts = 1;
  string next_page_token = 2;
}

message ArtifactReference {
  string artifact_id = 1;
  string title = 2;
  string kind = 3;
  RelationType relation_type = 4;
  google.protobuf.Timestamp linked_at = 5;
}

// Streaming with resume tokens
message StreamChatRequest {
  string threads_id = 1;
  string resume_token = 2;  // Resume from checkpoint
  string tenant_id = 3;
}

message ChatUpdate {
  oneof update {
    ChatMessage message = 1;
    ChatMetadata metadata = 2;
    ChatArtifactRelation relation = 3;
  }
  string checkpoint_token = 4;  // Save for resume
  google.protobuf.Timestamp timestamp = 5;
}

// threads_types.proto
message Chat {
  string id = 1;
  string title = 2;
  string user_id = 3;
  
  reserved 4;  // artifact_ids was removed - reserve to prevent reuse
  
  google.protobuf.Timestamp created_at = 5;
  google.protobuf.Timestamp updated_at = 6;
  ChatVisibility visibility = 7;
  ChatMetadata metadata = 8;
  string tenant_id = 9;          // For multi-tenancy
  string agent_id = 10;          // Configured LangGraph assistant
}

message ChatMessage {
  string id = 1;
  string threads_id = 2;
  string role = 3;  // user, assistant, system
  repeated MessagePart parts = 4;
  repeated Attachment attachments = 5;
  google.protobuf.Timestamp created_at = 6;
  string artifact_id = 7;  // Explicit artifact reference (not ambiguous "context")
  string tenant_id = 8;     // Multi-tenancy support
  string idempotency_key = 9;  // For safe retries
}

// Input model for composing a message (no ids/timestamps)
message ChatMessageInput {
  string role = 1;                    // "user" | "assistant" | "system"
  repeated MessagePart parts = 2;     
  repeated Attachment attachments = 3;
  string artifact_id = 4;             // Optional context
}

message MessagePart {
  oneof kind {
    TextPart text = 1;
    ImagePart image = 2;
    CodePart code = 3;
    // Extensible for future types
  }
}

message TextPart {
  string text = 1;
}

message ImagePart {
  // One of url or data must be set
  string url = 1;
  bytes data = 2;
  string mime_type = 3;
}

message CodePart {
  string language = 1; // e.g., "python", "sql"
  string content = 2;
}

message Attachment {
  string id = 1;
  string name = 2;
  string mime_type = 3;
  string uri = 4;      // pointer to blob storage
  int64 size_bytes = 5;
}

message ChatArtifactRelation {
  string threads_id = 1;
  string artifact_id = 2;
  RelationType type = 3;  // created, referenced, modified
  google.protobuf.Timestamp created_at = 4;
  string created_by = 5;
}

enum RelationType {
  RELATION_TYPE_UNSPECIFIED = 0;
  RELATION_TYPE_CREATED = 1;      // Artifact created in this threads
  RELATION_TYPE_REFERENCED = 2;   // Artifact referenced/viewed
  RELATION_TYPE_MODIFIED = 3;     // Artifact modified from threads
}
```

### 2. Enhanced Artifact Services

Update existing artifact proto to support threads relationships:

```protobuf
// In artifacts.proto
message ArtifactView {
  // ... existing fields ...
  repeated string threads_ids = 10;        // All threadss that reference this
  string primary_threads_id = 11;          // Chat where created
  repeated ChatReference threads_refs = 12; // Detailed threads relationships
}

message ChatReference {
  string threads_id = 1;
  string threads_title = 2;
  RelationType relation_type = 3;
  google.protobuf.Timestamp linked_at = 4;
}

// New RPCs in ArtifactCommandService
rpc CreateArtifact(CreateArtifactRequest) returns (CreateArtifactResponse);
rpc AssociateChat(AssociateArtifactChatRequest) returns (AssociateArtifactChatResponse);
rpc DissociateChat(DissociateArtifactChatRequest) returns (DissociateArtifactChatResponse);

// New RPCs in ArtifactQueryService  
rpc GetArtifactChats(GetArtifactChatsRequest) returns (GetArtifactChatsResponse);
rpc GetArtifactHistory(GetArtifactHistoryRequest) returns (GetArtifactHistoryResponse);
rpc StreamArtifactUpdates(StreamArtifactRequest) returns (stream ArtifactUpdate);

message StreamArtifactRequest {
  string artifact_id = 1;
  string resume_token = 2;  // For resuming after disconnect
  string tenant_id = 3;
}

message GetArtifactChatsRequest {
  string artifact_id = 1;
  int32 page_size = 2;
  string page_token = 3;
  string tenant_id = 4;
}

message GetArtifactChatsResponse {
  repeated ChatReference threadss = 1;
  string next_page_token = 2;
}

message ArtifactUpdate {
  oneof update {
    ChatReference threads_added = 1;
    ArtifactContent content_updated = 2;
    ArtifactMetadata metadata_changed = 3;
  }
  string checkpoint_token = 10;
  google.protobuf.Timestamp timestamp = 11;
}

message ArtifactContent { 
  string mime_type = 1; 
  bytes body = 2; 
}

message ArtifactMetadata { 
  string title = 1; 
  map<string,string> labels = 2; 
}

// Basic CRUD Request/Response messages for ChatCommandService
message CreateChatRequest {
  string title = 1;
  ChatVisibility visibility = 2;
  string artifact_id = 3;  // Optional: if creating from artifact context
  ChatMetadata metadata = 4;
  string user_id = 5;      // Filled by auth layer
  string tenant_id = 6;
  string agent_id = 7;     // Optional: override default agent selection
}

message CreateChatResponse {
  Chat threads = 1;
}

message SendMessageRequest {
  string threads_id = 1;
  ChatMessageInput message = 2;
  string idempotency_key = 3;
  string user_id = 4;      // Filled by auth layer
  string tenant_id = 5;
}

message SendMessageResponse {
  ChatMessage message = 1;
}

message UpdateChatTitleRequest {
  string threads_id = 1;
  string title = 2;
  string user_id = 3;
  string tenant_id = 4;
}

message UpdateChatTitleResponse {
  Chat threads = 1;
}

message DeleteChatRequest {
  string threads_id = 1;
  string user_id = 2;
  string tenant_id = 3;
}

message DeleteChatResponse {
  bool success = 1;
}

message AssociateChatArtifactRequest {
  string threads_id = 1;
  string artifact_id = 2;
  RelationType relation_type = 3;
  string user_id = 4;
  string tenant_id = 5;
}

message AssociateChatArtifactResponse {
  ChatArtifactRelation relation = 1;
}

message UpdateChatVisibilityRequest {
  string threads_id = 1;
  ChatVisibility visibility = 2;
  string user_id = 3;
  string tenant_id = 4;
}

message UpdateChatVisibilityResponse {
  Chat threads = 1;
}

// Supporting types
enum ChatVisibility {
  CHAT_VISIBILITY_UNSPECIFIED = 0;
  CHAT_VISIBILITY_PRIVATE = 1;
  CHAT_VISIBILITY_TEAM = 2;
  CHAT_VISIBILITY_PUBLIC = 3;
}

message ChatMetadata {
  map<string, string> labels = 1;
  string description = 2;
  repeated string tags = 3;
}

message ChatUpdatedEvent {
  string threads_id = 1;
  string field = 2;       // What changed: "title", "visibility", etc.
  string old_value = 3;
  string new_value = 4;
}

message EventMetadata {
  string event_id = 1;
  google.protobuf.Timestamp timestamp = 2;
  string user_id = 3;
  int64 sequence = 4;
}

// Artifact service request/response messages
message CreateArtifactRequest {
  string title = 1;
  string kind = 2;       // e.g., "bpmn"
  string user_id = 3;
  string tenant_id = 4;
  map<string,string> metadata = 5;
  string primary_threads_id = 6; // optional
}

message CreateArtifactResponse {
  string artifact_id = 1;
}

message AssociateArtifactChatRequest {
  string artifact_id = 1;
  string threads_id = 2;
  RelationType relation_type = 3;
  string user_id = 4;
  string tenant_id = 5;
}

message AssociateArtifactChatResponse {
  ChatReference threads_reference = 1;
}

message DissociateArtifactChatRequest {
  string artifact_id = 1;
  string threads_id = 2;
  string user_id = 3;
  string tenant_id = 4;
}

message DissociateArtifactChatResponse {
  bool success = 1;
}

message GetArtifactHistoryRequest {
  string artifact_id = 1;
  int32 page_size = 2;
  string page_token = 3;
  string tenant_id = 4;
}

message GetArtifactHistoryResponse {
  repeated ArtifactHistoryEntry entries = 1;
  string next_page_token = 2;
}

message ArtifactHistoryEntry {
  string event_type = 1;  // "created", "modified", "threads_linked", etc.
  google.protobuf.Timestamp timestamp = 2;
  string user_id = 3;
  string threads_id = 4;      // If relevant
  map<string, string> metadata = 5;
}
```

### 3. Event Sourcing Model

```protobuf
// Event definitions for threads audit trail
message ChatAggregate {
  string id = 1;
  string tenant_id = 2;
  repeated ChatEvent events = 3;
  int64 version = 4;
}

message ChatEvent {
  oneof event {
    ChatCreatedEvent created = 1;
    MessageSentEvent message_sent = 2;
    ArtifactAssociatedEvent artifact_associated = 3;
    ChatUpdatedEvent updated = 4;
  }
  EventMetadata metadata = 10;
}

message ChatCreatedEvent {
  string threads_id = 1;
  string user_id = 2;
  string title = 3;
  ChatVisibility visibility = 4;
  string artifact_id = 5;  // If created from artifact
  string agent_id = 6;      // Configured assistant
}

message MessageSentEvent {
  string message_id = 1;
  string threads_id = 2;
  ChatMessage message = 3;
}

message ArtifactAssociatedEvent {
  string threads_id = 1;
  string artifact_id = 2;
  RelationType relation_type = 3;
}
```

### 4. Resume Token Semantics

**Checkpoint/Resume Token Specifications:**
- **Scope**: Tokens are scoped by `{tenant_id, stream_key}` where stream_key is threads_id or artifact_id
- **Format**: Opaque string, monotonically increasing within a stream
- **TTL**: 24 hours (configurable) - tokens expire after this period
- **Storage**: Redis Streams with circular buffer (last 1000 events per stream)
- **Guarantee**: At-least-once delivery with in-order checkpoints for a single stream
- **Idempotency**: Clients must dedupe using event_index or checkpoint_token
- **Coalescing**: Service may coalesce events during high load; checkpoint ensures no loss

### 5. SDK Client Extensions

Update `/packages/ameide_sdk_ts/src/client.ts`:

```typescript
// SDK helper for atomic operations
export class ChatArtifactHelper {
  constructor(private client: AmeideClient) {}
  
  async sendMessageFromArtifact(
    artifactId: string,
    message: string,
    threadsId?: string
  ): Promise<{ threadsId: string; messageId: string; isNewChat: boolean }> {
    const response = await this.client.threadsCommands.ensureChatForArtifactAndSendMessage({
      artifactId,
      threadsId,
      message: { role: 'user', parts: [{ text: message }] },
      idempotencyKey: crypto.randomUUID(),
    });
    
    return {
      threadsId: response.threadsId,
      messageId: response.persistedMessage.id,
      isNewChat: response.threadsCreated,
    };
  }
}

export class AmeideClient {
  // Existing services
  readonly artifactCommands: Client<typeof ArtifactCommandService>;
  readonly artifactQueries: Client<typeof ArtifactQueryService>;
  readonly bpmnEditor: Client<typeof BPMNEditorService>;
  
  // New threads services
  readonly threadsCommands: Client<typeof ChatCommandService>;
  readonly threadsQueries: Client<typeof ChatQueryService>;
  
  // Helper utilities
  readonly threadsArtifact: ChatArtifactHelper;
  
  constructor(transport: Transport) {
    // ... existing initialization ...
    this.threadsCommands = createClient(ChatCommandService, this.transport);
    this.threadsQueries = createClient(ChatQueryService, this.transport);
    this.threadsArtifact = new ChatArtifactHelper(this);
  }
}
```

### 6. Frontend Integration

**Note**: Frontend routing patterns and implementation details are documented in [147-artifact-first-routes.md](./147-artifact-first-routes.md).

**Key Frontend Requirements:**
- Use the atomic `EnsureChatForArtifactAndSendMessage` RPC to prevent race conditions
- Maintain stable URLs (`/artifact/[id]` and `/threads/[id]`) without client-side mutations
- Implement bidirectional navigation between threadss and artifacts
- Handle real-time updates via gRPC streaming with resume tokens

## Data Flow Examples

### Creating an Artifact from Chat

```typescript
// User in /threads/123 asks to create a BPMN diagram
const { artifactId } = await client.artifactCommands.createArtifact({
  title: "Order Process",
  kind: "bpmn",
  userId: session.user.id,
  tenantId: session.user.tenantId,
  metadata: { primaryChatId: "123" }
});

// Associate with current threads
await client.threadsCommands.associateArtifact({
  threadsId: "123",
  artifactId,
  relationType: RelationType.CREATED
});
```

### Viewing an Artifact

```typescript
// User navigates to /artifact/artifact-456  (singular route)
const artifact = await client.artifactQueries.getArtifact({
  artifactId: "artifact-456",
  includeChats: true
});

// Load associated threadss with pagination
const threadss = await client.artifactQueries.getArtifactChats({
  artifactId: "artifact-456",
  pageSize: 20
});

// User sends message about artifact - ATOMIC operation
const response = await client.threadsCommands.ensureChatForArtifactAndSendMessage({
  artifactId: "artifact-456",
  threadsId: selectedChatId,  // Optional - if user selected existing threads
  message: {
    role: 'user',
    parts: [{ text: userMessage }]
  },
  idempotencyKey: crypto.randomUUID(),
});

// UI stays on /artifact/artifact-456
// No URL change, just update sidebar with new/updated threads
```

### Real-time Collaboration

Use gRPC streaming for live updates:

```typescript
// Watch artifact for changes with resume capability
let resumeToken: string | null = localStorage.getItem('artifact-resume-token');

const stream = client.artifactQueries.streamArtifactUpdates({
  artifactId: "artifact-456",
  resumeToken
});

for await (const update of stream) {
  // Save checkpoint for resume
  if (update.checkpointToken) {
    resumeToken = update.checkpointToken;
    localStorage.setItem('artifact-resume-token', resumeToken);
  }
  
  switch (update.type) {
    case 'CHAT_ADDED':
      // New threads associated
      addChatToSidebar(update.threads);
      break;
    case 'CONTENT_UPDATED':
      // Artifact content changed
      updateArtifactContent(update.content);
      break;
    case 'METADATA_CHANGED':
      // Title, etc. changed
      updateMetadata(update.metadata);
      break;
  }
}
```

## Backend Service Implementation

### Chat Command Service

```python
# In services/threads/main.py
class ChatCommandService(ChatCommandServiceServicer):
    def __init__(self, event_store: EventStore):
        self.event_store = event_store
    
    async def EnsureChatForArtifactAndSendMessage(self, request, context):
        """Atomic operation for artifact-first UX"""
        
        # Check idempotency
        existing = await self.check_idempotency(request.idempotency_key)
        if existing:
            return existing
        
        threads_id = request.threads_id
        threads_created = False
        
        # Create threads if needed
        if not threads_id:
            threads_id = generate_uuid()
            threads_created = True
            
            # Determine agent based on artifact type
            artifact = await self.artifact_service.get_artifact(request.artifact_id)
            agent_id = self.select_agent_for_artifact(artifact.kind)
            
            # Create threads event with configured agent
            create_event = ChatCreatedEvent(
                threads_id=threads_id,
                user_id=request.user_id,
                title=f"Discussion about {artifact.title}",
                visibility=request.visibility or ChatVisibility.PRIVATE,
                artifact_id=request.artifact_id,
                agent_id=agent_id  # Configure assistant for this threads
            )
            
            # Note: For true atomicity, batch both events in a single append
            # or use a transaction with outbox pattern
            events = [
                create_event,
                ArtifactAssociatedEvent(
                    threads_id=threads_id,
                    artifact_id=request.artifact_id,
                    relation_type=RelationType.REFERENCED
                )
            ]
            
            await self.event_store.append(
                aggregate_id=threads_id,
                events=events,
                expected_version=0
            )
        
        # Send message
        message_id = generate_uuid()
        message_event = MessageSentEvent(
            message_id=message_id,
            threads_id=threads_id,
            message=ChatMessage(
                id=message_id,
                threads_id=threads_id,
                role=request.message.role,
                parts=request.message.parts,
                artifact_id=request.artifact_id,
                tenant_id=request.tenant_id,
                idempotency_key=request.idempotency_key
            )
        )
        
        await self.event_store.append(
            aggregate_id=threads_id,
            events=[message_event]
        )
        
        # Store idempotency result
        result = EnsureChatForArtifactAndSendMessageResponse(
            threads_id=threads_id,
            persisted_message=message_event.message,
            threads_created=threads_created
        )
        
        # Idempotency storage: keyed by {tenant_id, user_id, idempotency_key}
        # with 24-48h TTL to prevent unbounded growth
        await self.store_idempotency(
            key=(request.tenant_id, request.user_id, request.idempotency_key),
            value=result,
            ttl_hours=24
        )
        
        return result
```

## Implementation Phases

### Phase 1: Backend Proto & Services (Week 1-2)
- [ ] Define threads service proto files
- [ ] Update artifact proto with threads relationships
- [ ] Implement threads command service for write operations
- [ ] Implement threads query service for read operations
- [ ] Add threads relationship support to artifact services
- [ ] Set up event sourcing for threadss
- [ ] Implement idempotency layer

### Phase 2: SDK Updates (Week 2)
- [ ] Regenerate proto types with buf
- [ ] Add threads services to AmeideClient
- [ ] Implement ChatArtifactHelper utility
- [ ] Update mock transport with threads data
- [ ] Add integration tests for threads operations
- [ ] Document new SDK methods

### Phase 3: Frontend Data Layer (Week 3)
- [ ] Create useChat hook for threads operations
- [ ] Create useArtifact hook with gRPC backend
- [ ] Build ChatProvider context
- [ ] Build ArtifactProvider context
- [ ] Remove direct database queries
- [ ] Add error handling and retry logic
- [ ] Implement resume token persistence

### Phase 4: Artifact Viewer Page (Week 3-4)
- [ ] Create `/artifact/[artifactId]/page.tsx`  (singular route)
- [ ] Build ChatSidebar component for associated threadss
- [ ] Implement atomic message sending with threads creation
- [ ] Add threads switching functionality
- [ ] Ensure URL stability (no mutations)
- [ ] Add loading and error states

### Phase 5: Enhanced Chat Page (Week 4)
- [ ] Update threads page to use gRPC
- [ ] Build ArtifactNavigator component
- [ ] Implement artifact switching in threads view
- [ ] Add artifact creation tracking
- [ ] Update message context with artifacts

### Phase 6: Real-time & Polish (Week 5)
- [ ] Implement streaming updates for artifacts
- [ ] Implement streaming updates for threadss
- [ ] Add presence indicators
- [ ] Optimize performance with caching
- [ ] Add comprehensive error handling
- [ ] Write end-to-end tests

## Migration Strategy

### Step 1: Parallel Operation
- Keep existing database for backward compatibility
- New operations go through gRPC
- Shadow write to both systems

### Step 2: Read Migration
- Start reading from gRPC services
- Fall back to database if not found
- Log discrepancies for debugging

### Step 3: Write Migration
- All writes go through gRPC
- Database becomes read-only
- Monitor for issues

### Step 4: Data Migration
- Migrate existing threadss to backend
- Migrate artifact metadata
- Establish relationships from history

### Step 5: Cleanup
- Remove database queries from frontend
- Remove local database tables
- Update documentation

## Success Metrics

- **URL Stability**: 100% of refreshes maintain URL
- **Performance**: <200ms p95 for data operations
- **Relationships**: All artifacts trackable to creating threads
- **Real-time**: <100ms for update propagation
- **Migration**: Zero data loss during migration
- **Idempotency**: Zero duplicate threadss from retries

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Backend service downtime | High | Implement circuit breakers, local caching |
| Data migration errors | High | Comprehensive backup, rollback plan |
| Performance regression | Medium | Load testing, query optimization |
| Breaking changes | Medium | Versioned APIs, gradual rollout |
| Complex relationships | Low | Clear data model, good documentation |
| Race conditions | High | Idempotency keys, atomic operations |

## Dependencies

- Backend team implements threads service endpoints
- Proto definitions approved by architecture team
- SDK v2 with Connect-ES fully deployed
- Infrastructure supports gRPC streaming
- Redis for checkpoint storage
- Event store with optimistic concurrency
- Inference service with LangGraph agents (Week 1 Day 3-4 completed in backlog/141)

## Open Questions

1. Should we support offline mode with local caching?
2. How to handle large threads histories (pagination vs streaming)?
3. Should artifacts support versioning from day one?
4. What's the retention policy for threads-artifact relationships?
5. How to handle artifact deletion with existing threadss?
6. Buffer size for streaming replay after disconnects?

## Future Enhancements

- **Artifact Versioning**: Track all changes with version history
- **Collaborative Editing**: Real-time multi-user artifact editing
- **Access Control**: Granular permissions for artifacts and threadss
- **Templates**: Create artifacts from templates with threads context
- **Export/Import**: Backup and restore threadss with artifacts
- **Search**: Full-text search across threadss and artifacts
- **Analytics**: Usage patterns and insights
- **Offline Support**: Local-first with sync

## Acceptance Criteria

- [ ] Users can navigate to `/artifact/[id]` and stay there (singular)
- [ ] Users can navigate to `/threads/[id]` and stay there  
- [ ] NO client-side URL mutations (no replaceState, no router.replace)
- [ ] Atomic threads creation prevents race conditions
- [ ] Pagination works for all list operations  
- [ ] Resume tokens enable reliable streaming
- [ ] Multi-tenancy enforced throughout
- [ ] Artifacts show all associated threadss
- [ ] Chats show all associated artifacts
- [ ] Creating artifact from threads maintains relationship
- [ ] Refreshing any page maintains state
- [ ] All data operations go through SDK
- [ ] No direct database access from frontend
- [ ] Real-time updates work across tabs
- [ ] Comprehensive test coverage
- [ ] Idempotency prevents duplicate operations

## Related Documents

- [146-ai-backend-migration.md](./146-ai-backend-migration.md) - AI backend migration with LangGraph integration
- [147-artifact-first-routes.md](./147-artifact-first-routes.md) - Frontend routing patterns and URL stability
- [144-bpmn-lifecycle.md](./144-bpmn-lifecycle.md) - BPMN draft management
- [113-canvas-platform-integration-artifact-centric-graph.md](./113-canvas-platform-integration-artifact-centric-graph.md) - Canvas/editor integration layer
