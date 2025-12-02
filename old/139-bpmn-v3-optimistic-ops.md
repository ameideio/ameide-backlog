# Ops, Config, Risks & MVP Success Criteria

**Status**: Configuration & Integration Updated (2025-08-21)
**Note**: Configuration adapted for gRPC/Connect architecture, SDK integration complete

## Configuration

### Server Configuration
```yaml
# config/production.yaml
autosnapshot:
  remoteDrafts: true       # Enabled via gRPC streaming
  maxDraftAge: 7d
  cleanupCron: "0 2 * * *"  # 2 AM daily
  operationBatchSize: 100  # Operations per draft save

save:
  xsdEnabled: true
  xsdTimeoutMs: 5000
  lintOnSave: true         # Return warnings only
  maxMessageBytes: 4_000_000  # gRPC limit - chunking is app-side policy (see SDK README)
  blobStorageThreshold: 1_000_000  # Use content_ref above 1MB
  compressionEnabled: true # zlib in DB

ai:
  enabled: true
  provider: "openai"       # or "vertex", "azure", "anthropic"
  model: "gpt-4"
  maxContextBytes: 100_000
  maxOperations: 100       # Operations, not commands
  streamingEnabled: true   # Progressive operation delivery
  timeout: 10000
  temperature: 0.7
  autopilotAllowlist:      # Future, not MVP
    - "add-labels"
    - "set-default-flow"
    - "fix-disconnected"

readModels:
  sql: true               # Enable SQL projections
  graph: false            # Post-MVP
  asyncUpdate: true       # Queue-based
  
versioning:
  retentionDays: 365      # Keep versions for 1 year
  maxVersionsPerDiagram: 1000
```

### Client Configuration
```typescript
// config/editor.ts
import { createConnectTransport } from '@connectrpc/connect-web';

export const EDITOR_CONFIG = {
  transport: createConnectTransport({
    baseUrl: '/rpc',
    useBinaryFormat: true  // Efficient protobuf encoding
  }),
  
  autosave: {
    intervalMs: 30000,      // Every 30 seconds
    idleMs: 5000,          // After 5s idle
    remoteDrafts: true,    // Enabled via SaveDraft RPC
    maxOperationBuffer: 500, // Max operations before force flush
    debounceMs: 2000       // 2s debounce for operation batching
  },
  
  ai: {
    enabled: true,
    mode: 'HITL',          // Human-in-the-loop
    maxOperations: 50,     // Per suggestion
    contextLimit: 75000,
    debounceMs: 500,       // Debounce AI requests
    streaming: true,       // Use streaming RPC
    abortOnNavigation: true // Cancel stream on page change
  },
  
  lint: {
    active: true,
    profile: 'recommended',
    showOverlay: true,
    throttleMs: 1000       // Throttle lint runs
  },
  
  save: {
    prelintOnSave: false,  // Don't block on lint
    confirmOverwrite: true,
    showVersionNumber: true
  }
};
```

## SLOs & Performance Targets

### Service Level Objectives
| Operation | Target (p95) | Max |
|-----------|-------------|------|
| Local edit | <16ms | 50ms |
| Autosnapshot | <100ms | 500ms |
| Save (<100KB) | <400ms | 1s |
| Save (<1MB) | <800ms | 2s |
| AI suggestion | <2s | 5s |
| Draft recovery | <200ms | 500ms |
| XSD validation | <200ms | 5s |

### Availability
- Editor functionality: 100% (works offline)
- Save endpoint: 99.9% uptime
- AI suggestions: 99% uptime (graceful degradation)

## Transport & Protocols

### gRPC/Connect Configuration
```typescript
// Connection setup
const transport = createConnectTransport({
  baseUrl: process.env.RPC_URL || '/rpc',
  interceptors: [
    authInterceptor,      // Add JWT token
    retryInterceptor,     // Exponential backoff
    deadlineInterceptor,  // 30s default timeout
    tracingInterceptor    // OpenTelemetry
  ],
  // Support both protocols
  useBinaryFormat: true,  // Protobuf for efficiency
  acceptCompression: ['gzip', 'br']
});

// Streaming configuration
const streamOptions = {
  signal: abortController.signal,  // Cancellation
  headers: {
    'x-client-version': CLIENT_VERSION,
    'x-session-id': sessionId
  }
};
```

## Observability

### Metrics
```typescript
// Client metrics (sent via telemetry)
metrics.track('editor.operation', { 
  type: operationType,
  origin: 'user' | 'ai',
  latency: ms 
});

metrics.track('autosave.snapshot', { 
  size: bytes,
  duration: ms,
  storage: 'local' | 'remote' 
});

metrics.track('save.attempt', {
  version: newVersion,
  size: bytes,
  duration: ms,
  status: 'success' | 'conflict' | 'validation_error'
});

// Server metrics
prometheus.histogram('bpmn_save_duration_ms', duration);
prometheus.counter('bpmn_save_total', { status });
prometheus.counter('xsd_validation_errors', { error_type });
prometheus.histogram('ai_suggestion_latency_ms', duration);
```

### Logging
```typescript
// Structured logs
logger.info('diagram.saved', {
  diagram_id: id,
  version: v,
  actor_id: userId,
  size: bytes,
  checksum: sha256,
  warnings: lintWarnings.length
});

logger.error('xsd.validation.failed', {
  diagram_id: id,
  errors: xsdErrors,
  xml_excerpt: xml.substring(0, 500)
});
```

### Tracing
```typescript
// OpenTelemetry spans
const span = tracer.startSpan('save.diagram');
span.setAttributes({
  'diagram.id': id,
  'diagram.size': bytes,
  'version.client': clientVersion,
  'version.new': newVersion
});
```

## Risks & Mitigations

### Critical Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| **Data loss from no autosave** | High | Medium | IndexedDB snapshots every 30s |
| **Invalid BPMN saved** | High | Low | XSD validation blocks save |
| **AI generates bad commands** | Medium | Medium | HITL mode, validation, undo |
| **Version conflicts** | Medium | Low | Clear conflict UI, retry logic |
| **Large diagram performance** | Medium | Low | 5MB limit, compression, streaming |

### Security Considerations
1. **XSD Bombs**: Limit entity expansion, timeout validation
2. **AI Prompt Injection**: Sanitize context, validate outputs
3. **Draft Tampering**: Signed drafts in IndexedDB
4. **Version History Access**: Check permissions on read

## MVP Success Criteria (4 Weeks)

### Proto/SDK Foundation (✅ COMPLETE - 2025-08-21)
- ✅ BPMNEditorService proto with streaming support
- ✅ Operation-based delta system defined
- ✅ TypeScript SDK with Connect-ES v2 integration
- ✅ Complete mock transport with realistic BPMN XML
- ✅ BpmnJSAdapter bridge implementation
- ✅ Platform integration with TransportProvider
- ✅ Dynamic imports for SSR compatibility
- ✅ Test suite with streaming and conflict scenarios

### Week 1 Deliverables
- ✅ bpmn-js editor loads and works (dynamic imports)
- ✅ Basic operation capture via BpmnJSAdapter
- ✅ Auto-save with configurable intervals (30s default)
- [ ] bpmnlint shows inline errors
- [ ] IndexedDB persistence for drafts
- [ ] Manual save exports XML locally

### Week 2 Deliverables
- [ ] Save endpoint deployed and working
- [ ] XSD validation passes for valid BPMN
- [ ] Version increments on save
- [ ] Conflict detection returns 409
- [ ] Basic error handling UI

### Week 3 Deliverables
- [ ] Draft recovery modal on page load
- [ ] Version number displayed in UI
- [ ] Save success/error toasts
- [ ] Loading states during save
- [ ] Keyboard shortcuts (Ctrl+S)

### Week 4 Deliverables
- [ ] AI context extraction works
- [ ] AI proxy endpoint deployed
- [ ] At least 3 AI operations work:
  - [ ] Add labels to unnamed elements
  - [ ] Fix disconnected elements
  - [ ] Add default flows to gateways
- [ ] AI suggestions are undoable
- [ ] AI operations show in command log

### User Acceptance Criteria
1. **Can create** a new BPMN diagram from scratch
2. **Can edit** with immediate visual feedback
3. **Can save** and see version number increment
4. **Won't lose work** due to autosnapshots
5. **Can recover** draft after browser crash
6. **Can get AI help** for basic fixes
7. **Can undo** any operation including AI

### Performance Criteria
- Typing/editing feels instant (<50ms perceived)
- Save completes in <1 second for typical diagrams
- AI suggestions appear within 3 seconds
- No memory leaks after 1 hour of editing

### Quality Criteria
- Zero data loss scenarios identified
- All critical paths have error handling
- 80% code coverage on critical functions
- Manual testing on Chrome, Firefox, Safari

## Test Strategy

### Unit Tests
```typescript
describe('AutoSnapshot', () => {
  it('saves to IndexedDB after 30 seconds');
  it('saves on 5 second idle');
  it('skips if content unchanged');
  it('handles IndexedDB quota exceeded');
});

describe('Save', () => {
  it('sends correct payload');
  it('handles 409 conflict');
  it('handles validation errors');
  it('updates version on success');
});
```

### Integration Tests
```typescript
describe('E2E Save Flow', () => {
  it('creates diagram, edits, saves, increments version');
  it('recovers draft after refresh');
  it('handles concurrent edits (multiple tabs)');
});

describe('AI Integration', () => {
  it('extracts context correctly');
  it('applies valid suggestions');
  it('rejects invalid commands');
  it('maintains undo history');
});
```

### Load Tests
- 100 concurrent save operations
- 10MB diagram save
- 1000 element diagram with AI
- 24 hour editing session

## Rollout Plan

### Phase 1: Internal Testing (Week 4)
- Deploy to staging
- Internal team testing
- Fix critical bugs

### Phase 2: Beta Users (Week 5)
- 10 selected users
- Monitor metrics closely
- Gather feedback

### Phase 3: General Availability (Week 6)
- Release to all users
- Monitor for 2 weeks
- Plan v2 features based on usage

## Post-MVP Roadmap

### Near Term (Weeks 5-8)
- Remote draft synchronization
- Export formats (SVG, PNG, PDF)
- Diagram templates
- Advanced AI operations
- Collaborative features prep

### Medium Term (Months 2-3)
- Graph database projections
- Multi-tenant isolation
- Authentication/authorization
- Audit logging
- Performance optimizations

### Long Term (Months 4-6)
- Real-time collaboration
- Version branching/merging
- Process simulation
- Custom lint rules
- Plugin system