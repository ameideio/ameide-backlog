# Artifact-Centric Graph Architecture - Human-in-the-Loop Change Review

## Core Design Principles

The ChangeSet system enables reviewable proposals while maintaining:
- **Event Store as single source of truth**
- **Graph as read-only projection**
- **Repository security boundary**
- **No graph writes outside projector**

## ChangeSet Aggregate Model

### Aggregate Structure
```
Stream: ChangeSetAggregate:<tenant>/<graph>/<changesetId>

State:
- id, tenant_id, graph_id
- status: draft | pending_review | approved | rejected | conflicted | committed
- proposer_id (user or agent)
- created_at, updated_at
- title, description, rationale
- risk_summary, confidence_score (for AI-generated)
- touches: [{stream_id, expected_version}]
- proposed_events: [ProposedEvent]
```

### ProposedEvent Structure
```json
{
  "target_stream_id": "ArtifactAggregate:<artifactId>",
  "expected_version": 12,
  "event_type": "ArtifactNameUpdated",
  "event_payload": { /* domain payload */ },
  "intent": { /* original command for rebase */ }
}
```

## Lifecycle Flow

### 1. Create (Draft)
```python
async def create_changeset(command: CreateChangeSetCommand) -> str:
    changeset_id = generate_uuid()
    event = ChangeSetCreated(
        changeset_id=changeset_id,
        tenant_id=command.tenant_id,
        graph_id=command.graph_id,
        title=command.title,
        proposer_id=command.proposer_id,
        ai_generated=command.ai_generated
    )
    await event_store.append(
        f"ChangeSetAggregate:{tenant}/{repo}/{changeset_id}",
        [event]
    )
    return changeset_id
```

### 2. Populate with Proposed Events
```python
async def populate_changeset(changeset_id: str, intents: list[Intent]):
    proposed = []
    for intent in intents:
        # Rehydrate aggregate to current state
        agg = await rehydrate(intent.stream_id)
        expected_version = agg.version
        
        # Dry-run to produce events without committing
        events = agg.dry_run(intent.command)
        
        for event in events:
            proposed.append(ProposedEvent(
                target_stream_id=intent.stream_id,
                expected_version=expected_version,
                event_type=event.type,
                event_payload=event.payload,
                intent=intent  # Store for rebase
            ))
            expected_version += 1
    
    # Add to changeset
    await event_store.append(
        f"ChangeSetAggregate:{changeset_id}",
        [ProposedEventsAdded(events=proposed)]
    )
```

### 3. Submit for Review
```python
async def submit_for_review(changeset_id: str):
    changeset = await load_changeset(changeset_id)
    
    # Validate all target streams still at expected versions
    for touch in changeset.touches:
        current_version = await get_stream_version(touch.stream_id)
        if current_version != touch.expected_version:
            raise ConflictError(f"Stream {touch.stream_id} has changed")
    
    # Update status
    await event_store.append(
        f"ChangeSetAggregate:{changeset_id}",
        [ChangeSetSubmittedForReview()]
    )
    
    # Notify reviewers
    await notify_reviewers(changeset.graph_id)
```

### 4. Preview Without Graph Writes
```python
async def preview_changeset(changeset_id: str) -> Preview:
    changeset = await load_changeset(changeset_id)
    preview = Preview()
    
    # Apply proposed events in memory
    for proposed in changeset.proposed_events:
        agg = await rehydrate(proposed.target_stream_id)
        
        # Apply event to in-memory aggregate
        preview_state = agg.apply_preview(proposed.event_payload)
        preview.add_diff(
            stream_id=proposed.target_stream_id,
            before=agg.state,
            after=preview_state
        )
    
    # Generate graph overlay (in memory only)
    preview.graph_overlay = compute_graph_overlay(changeset.proposed_events)
    
    return preview
```

### 5. Approve and Commit
```python
async def approve_changeset(changeset_id: str, reviewer_id: str):
    changeset = await load_changeset(changeset_id)
    
    # Atomic multi-stream append
    result = await event_store.append_batch(
        graph_id=changeset.graph_id,
        batch=[
            {
                'stream_id': pe.target_stream_id,
                'expected_version': pe.expected_version,
                'events': [pe.event_payload]
            }
            for pe in changeset.proposed_events
        ]
    )
    
    if result.success:
        # Mark committed
        await event_store.append(
            f"ChangeSetAggregate:{changeset_id}",
            [ChangeSetCommitted(reviewer_id=reviewer_id)]
        )
        # Events flow through outbox → Kafka → projector → graph
    else:
        # Mark conflicted
        await event_store.append(
            f"ChangeSetAggregate:{changeset_id}",
            [ChangeSetConflicted(conflicts=result.conflicts)]
        )
```

## Atomic Multi-Stream Append

### PostgreSQL Implementation
```sql
-- Atomic append_batch implementation
CREATE FUNCTION append_batch(
    p_graph_id TEXT,
    p_batch JSONB
) RETURNS TABLE(success BOOLEAN, conflicts JSONB) AS $$
DECLARE
    v_item JSONB;
    v_current_version BIGINT;
    v_expected_version BIGINT;
    v_conflicts JSONB := '[]'::JSONB;
BEGIN
    -- Start transaction
    SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    
    -- Check all versions first
    FOR v_item IN SELECT * FROM jsonb_array_elements(p_batch)
    LOOP
        SELECT version INTO v_current_version
        FROM event_streams
        WHERE stream_id = v_item->>'stream_id'
        FOR UPDATE;  -- Lock the row
        
        v_expected_version := (v_item->>'expected_version')::BIGINT;
        
        IF v_current_version != v_expected_version THEN
            v_conflicts := v_conflicts || jsonb_build_object(
                'stream_id', v_item->>'stream_id',
                'expected', v_expected_version,
                'actual', v_current_version
            );
        END IF;
    END LOOP;
    
    -- If any conflicts, rollback
    IF jsonb_array_length(v_conflicts) > 0 THEN
        RETURN QUERY SELECT FALSE, v_conflicts;
        RETURN;
    END IF;
    
    -- No conflicts, append all events
    FOR v_item IN SELECT * FROM jsonb_array_elements(p_batch)
    LOOP
        -- Insert events
        INSERT INTO events (stream_id, version, event_type, event_data)
        SELECT 
            v_item->>'stream_id',
            (v_item->>'expected_version')::BIGINT + ord,
            e->>'event_type',
            e->'event_payload'
        FROM jsonb_array_elements(v_item->'events') WITH ORDINALITY AS t(e, ord);
        
        -- Insert outbox entries (same transaction)
        INSERT INTO outbox (
            graph_id, stream_id, event_type, event_payload, 
            trace_parent, correlation_id
        )
        SELECT 
            p_graph_id,
            v_item->>'stream_id',
            e->>'event_type',
            e->'event_payload',
            current_setting('app.trace_parent', true),
            current_setting('app.correlation_id', true)
        FROM jsonb_array_elements(v_item->'events') AS e;
        
        -- Update stream version
        UPDATE event_streams 
        SET version = version + jsonb_array_length(v_item->'events')
        WHERE stream_id = v_item->>'stream_id';
    END LOOP;
    
    RETURN QUERY SELECT TRUE, '[]'::JSONB;
END;
$$ LANGUAGE plpgsql;
```

## Conflict Resolution & Rebase

### Automatic Rebase
```python
async def rebase_changeset(changeset_id: str):
    changeset = await load_changeset(changeset_id)
    new_proposed = []
    
    for proposed in changeset.proposed_events:
        # Rehydrate to current head
        current_agg = await rehydrate(proposed.target_stream_id)
        
        # Re-apply original intent
        if proposed.intent:
            new_events = current_agg.dry_run(proposed.intent.command)
            
            for event in new_events:
                new_proposed.append(ProposedEvent(
                    target_stream_id=proposed.target_stream_id,
                    expected_version=current_agg.version,
                    event_type=event.type,
                    event_payload=event.payload,
                    intent=proposed.intent
                ))
                current_agg.version += 1
        else:
            # Manual rebase required
            raise ManualRebaseRequired(proposed.target_stream_id)
    
    # Update changeset with rebased events
    await event_store.append(
        f"ChangeSetAggregate:{changeset_id}",
        [ChangeSetRebased(
            old_events=changeset.proposed_events,
            new_events=new_proposed
        )]
    )
```

## API Design

### Proto Definitions
```proto
message ProposedEvent {
  string target_stream_id = 1;
  int64 expected_version = 2;
  string event_type = 3;
  google.protobuf.Any event_payload = 4;
  google.protobuf.Struct intent = 5;  // For rebase
}

message CreateChangeSetRequest {
  CommandHeader header = 1;  // tenant_id, graph_id
  string title = 2;
  string description = 3;
  string rationale = 4;
  bool ai_generated = 5;
  string ai_model = 6;
  double confidence_score = 7;
}

message PopulateChangeSetRequest {
  CommandHeader header = 1;
  string changeset_id = 2;
  repeated ProposedEvent events = 3;
}

message PreviewChangeSetRequest {
  CommandHeader header = 1;
  string changeset_id = 2;
  bool include_graph_overlay = 3;
}

message PreviewChangeSetResponse {
  repeated ArtifactDiff diffs = 1;
  GraphOverlay graph_overlay = 2;
  repeated ImpactAssessment impacts = 3;
}

message ApproveChangeSetRequest {
  CommandHeader header = 1;
  string changeset_id = 2;
  string comment = 3;
}

message RebaseChangeSetRequest {
  CommandHeader header = 1;
  string changeset_id = 2;
  bool auto_submit = 3;  // Auto-submit after successful rebase
}
```

### REST API Endpoints
```python
# Create changeset
POST /api/tenants/{tenant_id}/repositories/{graph_id}/changesets

# Add proposed events
PUT /api/tenants/{tenant_id}/repositories/{graph_id}/changesets/{changeset_id}/events

# Preview without committing
GET /api/tenants/{tenant_id}/repositories/{graph_id}/changesets/{changeset_id}/preview

# Submit for review
POST /api/tenants/{tenant_id}/repositories/{graph_id}/changesets/{changeset_id}/submit

# Approve and commit
POST /api/tenants/{tenant_id}/repositories/{graph_id}/changesets/{changeset_id}/approve

# Reject
POST /api/tenants/{tenant_id}/repositories/{graph_id}/changesets/{changeset_id}/reject

# Rebase
POST /api/tenants/{tenant_id}/repositories/{graph_id}/changesets/{changeset_id}/rebase
```

## Security & Authorization

### Role-Based Access
```python
CHANGESET_PERMISSIONS = {
    'graph.viewer': ['read', 'preview'],
    'graph.contributor': ['read', 'preview', 'create', 'populate'],
    'graph.reviewer': ['read', 'preview', 'create', 'populate', 'approve', 'reject'],
    'graph.owner': ['*']
}

async def check_changeset_permission(
    user_id: str, 
    graph_id: str, 
    action: str
) -> bool:
    user_roles = await get_user_graph_roles(user_id, graph_id)
    
    for role in user_roles:
        if action in CHANGESET_PERMISSIONS.get(role, []):
            return True
        if '*' in CHANGESET_PERMISSIONS.get(role, []):
            return True
    
    return False
```

## Graph Projection Behavior

### Projector Rules
```python
class GraphProjector:
    async def process_event(self, record: ConsumerRecord):
        event = json.loads(record.value)
        
        # IGNORE changeset events - they don't affect the graph
        if event['stream_id'].startswith('ChangeSetAggregate:'):
            logger.debug("Skipping changeset event", event_type=event['type'])
            return
        
        # Only process committed artifact events
        if event['stream_id'].startswith('ArtifactAggregate:'):
            await self.apply_projection(event)
            
        await self.consumer.commit()
```

## UI/UX Components

### Review Queue Component
```typescript
interface ChangeSetReviewQueue {
  filters: {
    status: ChangeSetStatus[];
    proposer: string[];
    aiGenerated: boolean;
    myReviews: boolean;
  };
  
  columns: [
    { field: 'title', sortable: true },
    { field: 'proposer', sortable: true },
    { field: 'created_at', sortable: true },
    { field: 'touches', render: (items) => `${items.length} artifacts` },
    { field: 'risk_score', render: RiskBadge },
    { field: 'status', render: StatusChip }
  ];
  
  actions: {
    onPreview: (id: string) => void;
    onApprove: (id: string) => void;
    onReject: (id: string) => void;
    onRebase: (id: string) => void;
  };
}
```

### Diff Viewer
```typescript
interface ChangeSetDiffViewer {
  changeset: ChangeSet;
  mode: 'unified' | 'split' | 'graph';
  
  renderDiff(): JSX.Element {
    switch(this.mode) {
      case 'unified':
        return <UnifiedDiff before={} after={} />;
      case 'split':
        return <SplitDiff left={} right={} />;
      case 'graph':
        return <GraphOverlay base={} overlay={} />;
    }
  }
}
```

## Observability

### Metrics
```python
# Prometheus metrics
changeset_created = Counter('changeset_created_total', ['graph_id', 'ai_generated'])
changeset_status = Gauge('changeset_by_status', ['graph_id', 'status'])
changeset_approval_time = Histogram('changeset_approval_seconds', ['graph_id'])
changeset_conflict_rate = Counter('changeset_conflicts_total', ['graph_id'])
changeset_rebase_success = Counter('changeset_rebase_success_total', ['graph_id'])
```

### Tracing
```python
# OpenTelemetry spans
with tracer.start_as_current_span("changeset.lifecycle") as span:
    span.set_attribute("changeset.id", changeset_id)
    span.set_attribute("graph.id", graph_id)
    span.set_attribute("proposed.events.count", len(proposed_events))
    
    # Link to approval span
    with tracer.start_as_current_span("changeset.approve") as approve_span:
        approve_span.add_link(span.get_span_context())
        
        # Link to batch append
        with tracer.start_as_current_span("eventstore.append_batch") as batch_span:
            batch_span.set_attribute("streams.count", len(unique_streams))
```

### Dashboards
```yaml
# Grafana dashboard config
panels:
  - title: ChangeSet Queue Depth
    query: changeset_by_status{status="pending_review"}
    
  - title: Approval Latency (p95)
    query: histogram_quantile(0.95, changeset_approval_seconds)
    
  - title: Conflict Rate
    query: rate(changeset_conflicts_total[5m])
    
  - title: AI vs Human Proposals
    query: sum by (ai_generated) (rate(changeset_created_total[1h]))
```

## Acceptance Criteria

- [ ] **No uncommitted changes** reach the graph projection
- [ ] **Atomic multi-stream commits** with all-or-nothing semantics
- [ ] **Preview without side effects** - no graph writes
- [ ] **Conflict detection** via expected_version on approval
- [ ] **Automatic rebase** when possible, manual when not
- [ ] **Repository-scoped** all operations
- [ ] **Role-based review** with audit trail
- [ ] **Full observability** from proposal to commit
- [ ] **Graph remains read-only** projection of committed events only

## Implementation Phases

### Phase 0: Core ChangeSet
- ChangeSet aggregate and events
- Basic create, populate, approve flow
- Atomic append_batch in PostgreSQL

### Phase 1: Preview & UI
- In-memory preview computation
- Review queue UI
- Diff viewer components

### Phase 2: Advanced Features
- Automatic rebase
- AI confidence scoring
- Batch approvals
- Scheduled commits

### Phase 3: Graph Overlay (Optional)
- Separate overlay projector
- Union queries for preview
- Performance optimization for large diffs