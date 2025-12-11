# Text Editor V3 Implementation Gaps Analysis

## Executive Summary

The text editor implementation currently uses **ProseMirror** (rich text) and **CodeMirror** (code) with **fully optimistic** updates and auto-save. Moving to v3's **server-gated pessimistic** architecture requires fundamental changes to transaction handling, introducing operational transformation (OT) complexity beyond BPMN's discrete commands.

**Critical Finding**: Text editing presents **unique challenges** for pessimistic control due to high-frequency character-level operations. Current implementation has **zero server validation** and **no collaborative infrastructure**.

## Gap Severity Classification

- 🔴 **CRITICAL**: Blocks entire v3 architecture
- 🟠 **MAJOR**: Core functionality missing
- 🟡 **MINOR**: Enhancement or optimization needed
- 🟢 **READY**: Existing infrastructure can be leveraged

## 1. Frontend Editor Gaps

### 🔴 CRITICAL: No Transaction Gating
**Current ProseMirror**:
```typescript
// Current: Immediate application
const newState = editorRef.current.state.apply(transaction);
editorRef.current.updateState(newState);
```
**Required**: Block transactions until server approval
**Challenge**: Character-by-character blocking would be unusable

### 🔴 CRITICAL: No Command Extraction
**Current**: Raw ProseMirror/CodeMirror transactions
**Required**: Convert to typed DocumentCommands (insert, delete, format)
**Complexity**: ProseMirror transactions contain complex steps

### 🟠 MAJOR: No Optimistic Concurrency Control
**Current**: Last-write-wins with debounced saves
**Required**: Version tracking per transaction
**Files**: `lib/editor/config.ts`, `components/text-editor.tsx`

### 🟠 MAJOR: No Pending State Visualization
**Current**: Immediate visual updates
**Required**: Shadow DOM or overlay for unconfirmed text
**UX Impact**: Critical for perceived performance

### 🟡 MINOR: No Operational Transform
**Current**: Single-user editing only
**Future**: OT for concurrent editing
**Note**: Not required for v3 single-user, but architecture should allow it

## 2. Transport Layer Gaps

### 🔴 CRITICAL: No Document Command Service
**Current**: Direct artifact save via API
**Required**: DocumentCommandService with batched operations
**Proto exists**: `documents/v1/document_commands.proto`

### 🟠 MAJOR: No Transaction Batching
**Current**: Individual auto-saves with basic debouncing
**Required**: Intelligent operation batching with 2-second debounce
```typescript
// Required pattern with 2-second batching
interface DocumentTransaction {
  operations: DocumentOperation[];  // Accumulated over 2 seconds
  baseVersion: number;
  transactionId: string;
  accumulationTime: number;  // Time spent collecting operations
  operationCount: number;    // Pre-coalescence count for metrics
}
```
**Impact**: 2-second debounce reduces server calls by ~90% during active editing

### 🟠 MAJOR: No Conflict Resolution
**Current**: No conflict detection
**Required**: Three-way merge or rebase strategies
**Complexity**: Text conflicts harder than BPMN element conflicts

## 3. Backend Service Gaps

### 🔴 CRITICAL: No Document Command Handler
**Current**: No service implementation
**Required**: Service to process DocumentCommandEnvelope
**Location**: `services/core_document_command/` (doesn't exist)

### 🔴 CRITICAL: No Text Validation Pipeline
**Current**: No validation
**Required Options**:
- Grammar checking (LanguageTool)
- Spell checking
- Format validation (Markdown/HTML)
- Security (XSS prevention)

### 🟠 MAJOR: No Character-Level Event Store
**Current**: Full document saves
**Required**: Operation-based storage
```proto
message DocumentOperation {
  oneof op {
    InsertOp insert = 1;
    DeleteOp delete = 2;
    FormatOp format = 3;
  }
  int64 version = 4;
  string userId = 5;
}
```

### 🟠 MAJOR: No Snapshot Optimization
**Current**: Full content every save
**Required**: Delta compression with periodic snapshots
**Storage Impact**: 100x reduction possible

## 4. Unique Text Editor Challenges

### 🔴 Character-Level Latency
**Problem**: Each keystroke can't wait 300ms
**Solution Options**:
1. **Local Permits**: Pre-approve character ranges
2. **Optimistic with Reconciliation**: Apply locally, reconcile on conflict
3. **Hybrid Mode**: Optimistic for typing, pessimistic for formatting

### 🔴 Operational Transform Complexity
**ProseMirror Steps**:
```typescript
// ProseMirror transaction contains complex steps
transaction.steps.forEach(step => {
  // ReplaceStep, ReplaceAroundStep, AddMarkStep, etc.
  // Each needs conversion to domain commands
});
```

### 🟠 Cursor Position Management
**Current**: Local cursor only
**Required**: Transform cursor positions through operations
```typescript
// Position transformation through operations
newPosition = transformPosition(oldPosition, operations);
```

### 🟠 Selection Preservation
**Current**: Selection lost on content replacement
**Required**: Maintain selection through server round-trips

## 5. Architectural Comparison: Text vs BPMN

| Aspect | BPMN Editor | Text Editor |
|--------|-------------|-------------|
| **Operation Frequency** | Low (drag/drop) | High (keystrokes) |
| **Latency Tolerance** | 300ms acceptable | <50ms required |
| **Command Granularity** | Element-level | Character-level |
| **Conflict Resolution** | ID-based | Position-based |
| **Validation Complexity** | XSD/schema | Grammar/spelling |
| **State Size** | Small (elements) | Large (documents) |

## 6. Implementation Roadmap

### Phase 1: Command Extraction (1 week)
**Goal**: Convert editor operations to domain commands
- [ ] ProseMirror step → DocumentCommand converter
- [ ] CodeMirror change → DocumentCommand converter
- [ ] Command batching logic
- [ ] Version tracking

### Phase 2: Local Buffering (2 weeks)
**Goal**: Optimistic editing with server sync
- [ ] Local operation buffer
- [ ] Shadow DOM for pending changes
- [ ] Reconciliation on server response
- [ ] Conflict detection

### Phase 3: Server Integration (2 weeks)
**Goal**: Document command service
- [ ] DocumentCommandService implementation
- [ ] Operation validation pipeline
- [ ] Event store with deltas
- [ ] Snapshot management

### Phase 4: Smart Gating with 2-Second Batching (3 weeks)
**Goal**: Hybrid optimistic/pessimistic with intelligent debouncing
- [ ] Implement 2-second debounce timer
- [ ] Operation coalescing during batch window
- [ ] Local permits for typing
- [ ] Server gates for formatting
- [ ] Predictive pre-authorization
- [ ] Adaptive latency hiding
- [ ] Batch size optimization based on activity

### Phase 5: Advanced Features (2 weeks)
**Goal**: Production optimizations
- [ ] Operation compression
- [ ] Cursor transformation
- [ ] Selection preservation
- [ ] Undo/redo with versions

## 7. Proposed Hybrid Architecture

### Three-Layer Approach
```typescript
// Layer 1: Immediate (no gate)
const immediateOps = ['insert-char', 'delete-char'];

// Layer 2: Optimistic (reconcile later)
const optimisticOps = ['format', 'paste', 'cut'];

// Layer 3: Pessimistic (wait for server)
const pessimisticOps = ['delete-paragraph', 'reformat-document'];
```

### Smart Batching Strategy with 2-Second Debouncing
```typescript
class OperationBatcher {
  private static readonly DEBOUNCE_DELAY = 2000; // 2 seconds
  private static readonly MAX_BATCH_SIZE = 100;
  private static readonly FORCE_FLUSH_SIZE = 500; // Safety limit
  
  // Combine sequential inserts
  coalesceInserts(ops: InsertOp[]): InsertOp;
  
  // Merge adjacent deletes
  mergeDeletes(ops: DeleteOp[]): DeleteOp;
  
  // 2-second debounce with safety limits
  shouldFlush(): boolean {
    // Force flush on large batches to prevent memory issues
    if (ops.length >= FORCE_FLUSH_SIZE) return true;
    
    // Standard 2-second debounce for normal editing
    if (timeSinceLastOp > DEBOUNCE_DELAY) return true;
    
    // Flush on explicit save action or blur
    if (userTriggeredSave || editorLostFocus) return true;
    
    return false;
  }
}
```

### Benefits of 2-Second Debouncing
- **Reduced server load**: ~90% fewer requests during active typing
- **Better operation coalescing**: More opportunities to merge operations
- **Natural pause detection**: Aligns with thinking/reading pauses
- **Conflict reduction**: Larger atomic transactions reduce mid-edit conflicts

## 8. Migration Path from Current Implementation

### Phase-Compatible Changes
1. **Add Command Extraction** - Works with current optimistic flow
2. **Introduce Version Tracking** - Shadow state alongside current
3. **Build Server Service** - Test with subset of operations
4. **Gradual Gating** - Feature flag per operation type
5. **Full Cutover** - Remove optimistic path

### Rollback Strategy
- Keep current auto-save as fallback
- Dual-write during transition
- Version compatibility checks

## 9. Quick Wins & Incremental Delivery

### Week 1 Quick Wins
1. ✅ Add version field to saves (1 hour)
2. ✅ Extract insert/delete commands (4 hours)
3. ✅ Create DocumentCommandService stub (2 hours)
4. ✅ Log operations to console (30 min)

### MVP Scope
- Single-user editing only
- Insert/delete commands only
- No formatting commands
- 500ms latency acceptable
- No conflict resolution

## 10. Performance Requirements

### Latency Budgets
- **Keystroke → Visual**: <16ms (one frame) - unchanged with batching
- **Operation → Local Buffer**: <1ms - instant accumulation
- **Batch → Server**: 2000ms debounce + <100ms network
- **Server → Confirmation**: <200ms processing
- **Conflict → Resolution**: <500ms

### Throughput Targets with 2-Second Batching
- **Operations/second**: 20 (fast typing) accumulated locally
- **Batch size**: 40-100 operations (2 seconds of typing)
- **Server requests/minute**: ~10-15 (down from 100+)
- **Compression ratio**: 20:1 (better with larger batches)
- **Memory usage**: <1MB buffer per editor instance

## 11. Alternative Approaches Considered

### A. Full Optimistic with Periodic Sync
**Pros**: Simple, performant
**Cons**: Conflicts harder to resolve
**Verdict**: Not aligned with v3 architecture

### B. CRDT-Based (Yjs/Automerge)
**Pros**: Automatic conflict resolution
**Cons**: Different architecture, large library
**Verdict**: Consider for Phase 7+

### C. Server-Side Rendering
**Pros**: True single source of truth
**Cons**: Latency unacceptable for typing
**Verdict**: Rejected

## 12. Risk Assessment

### Critical Risks
1. **Typing Latency**: Users abandon if sluggish
   - **Mitigation**: Hybrid architecture with local permits
   
2. **Complex Conflicts**: Text merging is hard
   - **Mitigation**: Start with last-write-wins
   
3. **Storage Explosion**: Every keystroke stored
   - **Mitigation**: Aggressive compression and cleanup

### Technical Debt
- ProseMirror plugins may need rewriting
- CodeMirror extensions must be adapted
- Current auto-save must coexist during transition

## 13. Success Metrics

### User Experience
- Typing latency <50ms perceived
- No lost keystrokes
- Conflict resolution <2% of edits
- Undo/redo works across sessions

### Technical Metrics
- Operation throughput >100 ops/sec
- Storage efficiency >10:1 compression
- Server validation <50ms average
- Snapshot generation <100ms

## 14. Key Differences from BPMN Implementation

### Why Text is Harder
1. **Volume**: 1000x more operations
2. **Latency**: 10x stricter requirements  
3. **Conflicts**: Position-based vs ID-based
4. **State Size**: Kilobytes vs megabytes

### Why Text is Easier
1. **Linear Structure**: Sequential operations
2. **Well-Studied**: OT algorithms exist
3. **Graceful Degradation**: Can fall back to full saves
4. **User Expectations**: Some latency accepted for rich features

## Conclusion

Text editing in a server-gated architecture presents **fundamentally different challenges** than BPMN editing. The high frequency and latency sensitivity of text operations require a **hybrid approach** rather than pure pessimistic gating.

**Recommendation**: Implement a **three-layer architecture** with immediate local operations, optimistic formatting, and pessimistic structural changes. **Critical: Use 2-second debouncing** to dramatically reduce server load while maintaining responsive user experience.

**Key Insight**: 2-second debouncing transforms the problem:
- **Before**: 100+ server calls/minute during active editing
- **After**: 10-15 server calls/minute with better coalescing
- **User Impact**: Imperceptible - typing remains instant, only sync is delayed

**Timeline**: 10-12 weeks for full implementation, but **MVP possible in 3 weeks** with limited scope.

**Critical Success Factor**: Perceived performance must match current optimistic implementation through local buffering and 2-second intelligent debouncing, not aggressive server communication.