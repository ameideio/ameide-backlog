# Backend & Data — Save-Time Versioning, Drafts, Projections

**Status**: Proto & SDK Complete, Platform Integration Done, Backend Handlers Pending (2025-08-21)
**Note**: Original REST endpoints shown for reference, actual implementation uses gRPC
**Progress**: 
- ✅ BPMNEditorService proto finalized with all operations
- ✅ TypeScript SDK complete with full test coverage and BpmnJSAdapter
- ✅ Mock transport provides realistic BPMN XML and streaming responses
- ✅ Platform integration complete in www_ameide_platform with dynamic imports
- ✅ TransportProvider and auth interceptor properly configured
- ⏳ Python service handlers pending in core_grpc package
- ⏳ Storage layer integration (PostgreSQL event store)

## API Surface

### gRPC Service Implementation

The backend now implements `BPMNEditorService` defined in `bpmn_editor_service.proto`:

```python
from ameide_core_grpc import BPMNEditorServiceServicer
from ameide_core_proto import Operation, SaveDraftResponse

class BPMNEditorServiceImpl(BPMNEditorServiceServicer):
    async def SaveDraft(self, request, context):
        # Apply operations to current state
        for op in request.operations:
            await self.apply_operation(op)
        
        # Check for conflicts
        if request.expected_version != self.current_version:
            return SaveDraftResponse(
                conflict=ConflictInfo(
                    server_version=self.current_version,
                    conflicting_operations=self.get_conflicts()
                )
            )
        
        # Save to event store
        await self.event_store.append(request.operations)
        return SaveDraftResponse(
            version=self.current_version + 1,
            etag=self.generate_etag()
        )
    
    async def WatchDraft(self, request, context):
        # Stream events to client
        async for event in self.draft_events:
            yield DraftEvent(operation_applied=event)
    
    async def GetAISuggestions(self, request, context):
        # Stream AI operations progressively
        yield AISuggestionEvent(header=SuggestionHeader(...))
        
        async for op in self.generate_ai_operations(request.prompt):
            yield AISuggestionEvent(operation=op)
        
        yield AISuggestionEvent(complete=SuggestionComplete(...))
```

### Original REST Endpoints (Reference)

#### POST /api/diagrams/{id}/save (maps to CreateSnapshot RPC)
```typescript
interface SaveRequest {
  xml: string;
  clientVersion?: number;  // For conflict detection
  checksum?: string;       // Client-computed SHA256
  metadata?: {
    name?: string;
    description?: string;
    tags?: string[];
  };
}

interface SaveResponse {
  version: number;         // New version number
  checksum: string;        // Server-computed
  warnings?: LintWarning[];
  size: number;           // Bytes
  savedAt: string;        // ISO timestamp
}

// Implementation
async function save(req: SaveRequest): SaveResponse {
  // 1. Validate XSD
  const xsdResult = await validateXSD(req.xml);
  if (!xsdResult.valid) {
    throw new ValidationError(400, xsdResult.errors);
  }
  
  // 2. Check version conflict
  if (req.clientVersion) {
    const current = await getLatestVersion(diagramId);
    if (req.clientVersion < current) {
      throw new ConflictError(409, { 
        serverVersion: current,
        clientVersion: req.clientVersion 
      });
    }
  }
  
  // 3. Persist new version
  const version = await db.transaction(async tx => {
    const nextVersion = await getNextVersion(tx, diagramId);
    await tx.insert('diagram_versions', {
      diagram_id: diagramId,
      version: nextVersion,
      xml: req.xml,
      checksum: sha256(req.xml),
      size: Buffer.byteLength(req.xml),
      actor_id: req.actorId,
      created_at: new Date()
    });
    return nextVersion;
  });
  
  // 4. Update read models (async)
  await queue.publish('diagram.saved', { 
    diagramId, 
    version, 
    xml: req.xml 
  });
  
  // 5. Optional lint warnings
  const warnings = config.lintOnSave 
    ? await lintBPMN(req.xml) 
    : undefined;
  
  return { version, warnings, ... };
}
```

#### PUT /api/diagrams/{id}/draft
```typescript
interface DraftRequest {
  xml: string;
  etag?: string;  // For conflict detection
}

interface DraftResponse {
  etag: string;   // New ETag for next update
  savedAt: string;
}

// Last-write-wins draft storage
async function saveDraft(req: DraftRequest): DraftResponse {
  const etag = generateETag(req.xml);
  
  // Optional ETag check
  if (req.etag) {
    const current = await getDraftETag(diagramId, userId);
    if (req.etag !== current) {
      // Draft conflict - but we still save (LWW)
      console.warn('Draft ETag mismatch, overwriting');
    }
  }
  
  await db.upsert('diagram_drafts', {
    diagram_id: diagramId,
    user_id: userId,
    xml: req.xml,
    etag,
    updated_at: new Date()
  });
  
  return { etag, savedAt: new Date().toISOString() };
}
```

#### GET /api/diagrams/{id}/snapshot
```typescript
interface SnapshotParams {
  version?: number;  // Specific version or latest
}

interface SnapshotResponse {
  xml: string;
  version: number;
  checksum: string;
  createdAt: string;
  author: string;
}
```

#### POST /api/ai/suggest
```typescript
interface AIRequest {
  intent: string;           // User's request
  context: {
    selection?: string[];   // Selected element IDs
    lintIssues?: LintIssue[];
    xmlExcerpt?: string;    // Bounded context
  };
  limits?: {
    maxCommands: number;    // Default: 50
    maxTokens: number;      // Default: 2000
  };
}

interface AIResponse {
  suggestions: BpmnCommand[];  // Structured commands
  explanation?: string;         // Human-readable
  confidence: number;           // 0-1
}

// Server-side LLM proxy
async function suggestAI(req: AIRequest): AIResponse {
  // Build prompt with schema
  const prompt = buildPrompt(req.intent, req.context);
  
  // Call LLM with structured output
  const response = await llm.generate({
    prompt,
    responseFormat: BPMN_COMMAND_SCHEMA,
    maxTokens: req.limits?.maxTokens || 2000
  });
  
  // Validate commands against schema
  const commands = validateCommands(response.commands);
  
  // Additional safety checks
  if (commands.length > (req.limits?.maxCommands || 50)) {
    commands = commands.slice(0, req.limits.maxCommands);
  }
  
  return { 
    suggestions: commands,
    explanation: response.explanation,
    confidence: response.confidence 
  };
}
```

## Database Schema

### PostgreSQL Tables

```sql
-- Core diagram metadata
CREATE TABLE diagrams (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  created_by UUID NOT NULL,
  is_deleted BOOLEAN DEFAULT FALSE
);

-- Immutable version history
CREATE TABLE diagram_versions (
  diagram_id UUID NOT NULL REFERENCES diagrams(id),
  version INTEGER NOT NULL,
  xml TEXT NOT NULL,  -- Or BYTEA with compression
  checksum VARCHAR(64) NOT NULL,
  size_bytes INTEGER NOT NULL,
  actor_id UUID NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  metadata JSONB,  -- Optional: tags, comments
  PRIMARY KEY (diagram_id, version)
);

-- User drafts (optional, last-write-wins)
CREATE TABLE diagram_drafts (
  diagram_id UUID NOT NULL REFERENCES diagrams(id),
  user_id UUID NOT NULL,
  xml TEXT NOT NULL,
  etag VARCHAR(64),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  PRIMARY KEY (diagram_id, user_id)
);

-- Indexes
CREATE INDEX idx_versions_created ON diagram_versions(created_at DESC);
CREATE INDEX idx_drafts_updated ON diagram_drafts(updated_at DESC);
```

### Read Model Projections (Updated on Save)

```sql
-- Denormalized element view
CREATE TABLE bpmn_elements (
  diagram_id UUID NOT NULL,
  version INTEGER NOT NULL,
  element_id VARCHAR(255) NOT NULL,
  element_type VARCHAR(100),
  name TEXT,
  parent_id VARCHAR(255),
  properties JSONB,
  PRIMARY KEY (diagram_id, version, element_id)
);

-- Flow relationships
CREATE TABLE bpmn_flows (
  diagram_id UUID NOT NULL,
  version INTEGER NOT NULL,
  flow_id VARCHAR(255) NOT NULL,
  source_id VARCHAR(255),
  target_id VARCHAR(255),
  condition_expr TEXT,
  PRIMARY KEY (diagram_id, version, flow_id)
);

-- Aggregated statistics
CREATE TABLE bpmn_statistics (
  diagram_id UUID PRIMARY KEY,
  latest_version INTEGER,
  element_count INTEGER,
  flow_count INTEGER,
  gateway_count INTEGER,
  pool_count INTEGER,
  last_updated TIMESTAMP
);
```

## Implementation Status

### Completed Frontend Integration (2025-08-21)
- ✅ BpmnJSAdapter bridges bpmn-js with SDK operations
- ✅ Dynamic imports prevent SSR/chunk loading issues
- ✅ Mock transport enables full development without backend
- ✅ Auto-save with configurable intervals
- ✅ Conflict resolution with operation rebasing
- ✅ AI suggestions streaming with progressive UI updates
- ✅ Event listener cleanup to prevent memory leaks

## MVP Implementation (Backend)

### Week 2: Core Services (Pending)
```python
# services/bpmn_save/main.py
from fastapi import FastAPI, HTTPException
from lxml import etree
import hashlib

app = FastAPI()

@app.post("/api/diagrams/{diagram_id}/save")
async def save_diagram(diagram_id: str, request: SaveRequest):
    # 1. XSD validation
    try:
        schema = load_bpmn_xsd()
        doc = etree.fromstring(request.xml.encode())
        schema.assertValid(doc)
    except etree.XMLSyntaxError as e:
        raise HTTPException(400, detail=str(e))
    
    # 2. Version management
    async with db.transaction() as tx:
        current = await tx.fetchval(
            "SELECT MAX(version) FROM diagram_versions WHERE diagram_id = $1",
            diagram_id
        )
        new_version = (current or 0) + 1
        
        # 3. Check conflict
        if request.client_version and request.client_version < current:
            raise HTTPException(409, detail={
                "error": "version_conflict",
                "server_version": current
            })
        
        # 4. Save
        await tx.execute("""
            INSERT INTO diagram_versions 
            (diagram_id, version, xml, checksum, size_bytes, actor_id)
            VALUES ($1, $2, $3, $4, $5, $6)
        """, diagram_id, new_version, request.xml, 
            hashlib.sha256(request.xml.encode()).hexdigest(),
            len(request.xml.encode()), request.actor_id)
    
    # 5. Trigger projections (async)
    await message_queue.publish("diagram.saved", {
        "diagram_id": diagram_id,
        "version": new_version,
        "xml": request.xml
    })
    
    return SaveResponse(
        version=new_version,
        checksum=checksum,
        size=len(request.xml.encode()),
        saved_at=datetime.now().isoformat()
    )
```

### Week 3: Projections
```python
# workers/projection_updater.py
async def handle_diagram_saved(event):
    diagram_id = event["diagram_id"]
    version = event["version"]
    xml = event["xml"]
    
    # Parse BPMN
    elements, flows = parse_bpmn_xml(xml)
    
    # Update read models
    async with db.transaction() as tx:
        # Clear old projections
        await tx.execute(
            "DELETE FROM bpmn_elements WHERE diagram_id = $1 AND version = $2",
            diagram_id, version
        )
        
        # Insert elements
        for elem in elements:
            await tx.execute("""
                INSERT INTO bpmn_elements 
                (diagram_id, version, element_id, element_type, name, properties)
                VALUES ($1, $2, $3, $4, $5, $6)
            """, diagram_id, version, elem.id, elem.type, elem.name, elem.props)
        
        # Update statistics
        await tx.execute("""
            INSERT INTO bpmn_statistics 
            (diagram_id, latest_version, element_count, flow_count, last_updated)
            VALUES ($1, $2, $3, $4, NOW())
            ON CONFLICT (diagram_id) DO UPDATE SET
                latest_version = $2,
                element_count = $3,
                flow_count = $4,
                last_updated = NOW()
        """, diagram_id, version, len(elements), len(flows))
```

### Week 4: AI Proxy
```python
# services/ai_proxy/main.py
@app.post("/api/ai/suggest")
async def suggest_ai(request: AIRequest):
    # Build prompt
    prompt = f"""
    You are a BPMN expert. The user wants to: {request.intent}
    
    Current selection: {request.context.selection}
    Lint issues: {request.context.lint_issues}
    
    Generate BPMN commands as JSON following this schema:
    {COMMAND_SCHEMA}
    
    Maximum {request.limits.max_commands} commands.
    """
    
    # Call LLM
    response = await openai.create_completion(
        model="gpt-4",
        prompt=prompt,
        response_format={"type": "json_object"}
    )
    
    # Validate and return
    commands = validate_commands(response.choices[0].message.content)
    return AIResponse(
        suggestions=commands[:request.limits.max_commands],
        confidence=0.8
    )
```

## Performance & Scaling

### Optimization Strategies
1. **XML Compression**: Store compressed in DB (zlib/gzip)
2. **Projection Caching**: Redis for frequently accessed diagrams
3. **Async Processing**: Queue-based projection updates
4. **CDN for Reads**: Cache immutable versions at edge

### Capacity Planning
- Average diagram: 50KB XML
- After compression: ~10KB
- 10K diagrams × 100 versions = 1M records = ~10GB
- Read models add ~2x storage