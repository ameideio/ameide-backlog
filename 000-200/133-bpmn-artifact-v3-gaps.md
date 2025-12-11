# V3 Implementation Gaps Analysis

## Executive Summary

The v3 architecture proposes a **fundamental shift** from optimistic to pessimistic editing, requiring server validation before any canvas mutations. Current implementation has **no server integration**, operating as a standalone browser editor with console logging only.

**Critical Finding**: ~80% of v3 architecture is unimplemented. The shift from optimistic to pessimistic control affects every layer.

## Gap Severity Classification

- 🔴 **CRITICAL**: Blocks entire v3 architecture
- 🟠 **MAJOR**: Core functionality missing
- 🟡 **MINOR**: Enhancement or optimization needed
- 🟢 **READY**: Existing infrastructure can be leveraged

## 1. Frontend Layer Gaps

### 🔴 CRITICAL: No Server Gating
**Current**: bpmn-js applies changes immediately with optimistic updates
**Required**: Rules Gate blocks all local execution until server permits
**Impact**: Fundamental architecture change affecting all user interactions

### 🔴 CRITICAL: No Command Batching
**Current**: Individual events logged to console
**Required**: Transaction-based command batching for atomic operations
**Files**: `services/www_ameide_platform/artifacts/bpmn/client.tsx`

### 🟠 MAJOR: No Pending State UX
**Current**: Immediate canvas updates
**Required**: Ghost/preview visualization while awaiting server
**Effort**: New UI states and visual feedback system

### 🟠 MAJOR: No Programmatic Apply
**Current**: User-driven canvas manipulation only
**Required**: Server-driven canvas updates via modeling API
**Complexity**: ID remapping, coordinate transformation

## 2. Transport Layer Gaps

### 🔴 CRITICAL: No gRPC-Web Client
**Current**: Mock transport only, no server communication
**Required**: gRPC-Web client with Connect-ES v2
**Files**: Missing transport configuration in SDK usage

### 🔴 CRITICAL: No Envoy gRPC-Web Filter
**Current**: Envoy configured for HTTP/HTTPS only
**Required**: gRPC-Web filter for browser→backend communication
**Files**: `gitops/ameide-gitops/sources/charts/apps/gateway/`

### 🟠 MAJOR: No Request/Response Correlation
**Current**: Fire-and-forget logging
**Required**: Transaction ID tracking for async operations

## 3. Backend Service Gaps

### 🔴 CRITICAL: No Command Service Implementation
**Current**: Proto definitions exist, no service
**Required**: `BpmnCommandService` with `Propose` RPC
**Location**: `services/core_bpmn_command/` (doesn't exist)

### 🔴 CRITICAL: No Validation Pipeline
**Current**: No validation infrastructure
**Required**: XSD + bpmnlint + domain validation chain
**Components**:
- Python XSD validator (not implemented)
- Node.js bpmnlint sidecar (not implemented)
- Validation orchestrator (not implemented)

### 🔴 CRITICAL: No Event Store Integration
**Current**: No event persistence
**Required**: Dual stream (semantic/layout) event sourcing
**Missing**:
- Stream management
- Version tracking
- Optimistic locking

### 🟠 MAJOR: No Moddle Projection Service
**Current**: No server-side BPMN model manipulation
**Required**: bpmn-moddle sidecar for deterministic projection
**Complexity**: Node.js sidecar with gRPC interface

### 🟠 MAJOR: No Snapshot Management
**Current**: No state persistence
**Required**: XML snapshots with version tracking
**Storage**: Redis or PostgreSQL with TTL

## 4. Data Layer Gaps

### 🟠 MAJOR: No Read Model Projections
**Current**: No denormalized views
**Required**: PostgreSQL + Graph projections from events
**Tables**: `bpmn_elements`, `bpmn_flows`, `bpmn_statistics`

### 🟠 MAJOR: No Graph Database Integration
**Current**: Neo4j deployed but unused
**Required**: Graph projections for traversal/compliance
**Use Cases**: Path analysis, impact assessment, agent context

### 🟡 MINOR: No Cross-Model References
**Current**: BPMN isolation
**Required**: ArchiMate→BPMN traceability
**Impact**: Reduced value proposition

## 5. Infrastructure Gaps

### 🟢 READY: PostgreSQL Clusters
**Status**: Already deployed via CloudNativePG
**Services**: `postgres-event-rw`, `postgres-platform-rw`
**Action**: Just needs schema migration

### 🟢 READY: Envoy Gateway
**Status**: Deployed and routing HTTP/HTTPS
**Need**: Add gRPC-Web filter configuration
**Files**: `gitops/ameide-gitops/sources/charts/apps/gateway/`

### 🟢 READY: Redis Cache
**Status**: Deployed and available
**Use**: Snapshot storage, session state

### 🟡 MINOR: No Dedicated BPMN Database
**Current**: Shared platform database
**Better**: Separate BPMN event store for isolation

## 6. Architectural Mismatches

### 🔴 Optimistic vs Pessimistic Control
**Current Architecture**:
```javascript
// Current: Optimistic
eventBus.on('shape.create', (e) => {
  // Canvas already updated
  console.log('Command:', e);  // Log after the fact
});
```

**V3 Architecture**:
```javascript
// Required: Pessimistic
ruleProvider.canExecute('shape.create', () => {
  return pendingPermits.has(context.transactionId);  // Block without permit
});
```

### 🔴 Command Structure Mismatch
**Current**: Raw bpmn-js events with circular references
**V3**: Typed domain commands with clean serialization

### 🔴 Version Control Philosophy
**Current**: No versioning, last-write-wins
**V3**: Strict versioning with conflict detection

## 7. Implementation Roadmap

### Phase 1: Tracer Bullet (1 week)
**Goal**: Prove end-to-end flow with minimal implementation
- [ ] Simple Rules Gate (block one command type)
- [ ] Basic gRPC-Web echo service
- [ ] Always-accept validation stub
- [ ] In-memory command log
- [ ] Manual canvas update

### Phase 2: Frontend Gating (2 weeks)
**Goal**: Complete pessimistic control implementation
- [ ] Full RuleProvider implementation
- [ ] Permit management system
- [ ] Ghost state visualization
- [ ] Command batching with transactions
- [ ] ID remapping logic

### Phase 3: Transport & Service (2 weeks)
**Goal**: Real client-server communication
- [ ] gRPC-Web Envoy configuration
- [ ] BpmnCommandService skeleton
- [ ] Request/response correlation
- [ ] Error handling and retries
- [ ] Transaction management

### Phase 4: Validation Pipeline (3 weeks)
**Goal**: Full BPMN 2.0 compliance
- [ ] Python XSD validator service
- [ ] Node.js bpmnlint sidecar
- [ ] bpmn-moddle projection service
- [ ] Validation orchestration
- [ ] Error aggregation and reporting

### Phase 5: Event Sourcing (3 weeks)
**Goal**: Complete event store implementation
- [ ] Dual stream storage design
- [ ] Version management system
- [ ] Event replay mechanism
- [ ] Snapshot generation
- [ ] Optimistic concurrency control

### Phase 6: Read Models (2 weeks)
**Goal**: Query optimization and analytics
- [ ] PostgreSQL projections
- [ ] Graph database sync
- [ ] Query APIs
- [ ] Analytics endpoints
- [ ] Agent context builders

## 8. Risk Assessment

### High Risk Items
1. **Performance**: Pessimistic gating may feel sluggish
   - **Mitigation**: Local permits for rapid operations
   
2. **Complexity**: ID remapping and coordinate transformation
   - **Mitigation**: Start with simple 1:1 mapping
   
3. **Browser Compatibility**: gRPC-Web support varies
   - **Mitigation**: Fallback to REST for older browsers

### Technical Debt Considerations
- Current optimistic implementation needs complete replacement
- Proto definitions may need revision for v3 patterns
- Testing strategy must account for async validation

## 9. Quick Wins & Incremental Path

### Week 1 Quick Wins
1. ✅ Add gRPC-Web filter to Envoy (30 min)
2. ✅ Create stub command service (2 hours)
3. ✅ Simple canExecute override (1 hour)
4. ✅ Basic command batching (4 hours)

### Incremental Delivery
1. **Alpha**: Single-user with always-accept validation
2. **Beta**: XSD validation only
3. **RC**: Full validation pipeline
4. **GA**: Complete with read models and analytics

## 10. Resource Requirements

### Team Composition
- **Frontend Developer**: Rules Gate, UX states (4 weeks)
- **Backend Developer**: Command service, validation (6 weeks)
- **DevOps Engineer**: gRPC-Web, deployment (1 week)

### Infrastructure Needs
- Python service for XSD validation
- Node.js sidecar for moddle/lint
- Additional PostgreSQL schema
- Redis for snapshots

### External Dependencies
- bpmn-moddle npm package
- lxml Python package
- bpmnlint npm package

## 11. Migration Strategy

### From Current to V3
1. **Parallel Implementation**: Keep current optimistic path during development
2. **Feature Flag**: Toggle between optimistic/pessimistic modes
3. **Gradual Rollout**: Start with non-critical operations
4. **Full Cutover**: Remove optimistic path after validation

### Data Migration
- No existing persisted data (console logging only)
- Clean slate for event store design

## 12. Success Metrics

### Technical Metrics
- Command acceptance latency < 300ms (p95)
- XSD validation < 80ms average
- Zero data loss on version conflicts
- 100% command traceability

### User Experience Metrics
- No perceived lag for simple operations
- Clear feedback on validation failures
- Smooth ghost state transitions
- Predictable undo/redo behavior

## Conclusion

The v3 architecture requires **substantial implementation effort** with ~13-15 weeks for full implementation. The fundamental shift from optimistic to pessimistic control affects every layer of the stack.

**Recommendation**: Start with Phase 1 tracer bullet to validate the architecture, then proceed with incremental phases based on user feedback and performance metrics.

**Critical Path**: Frontend Rules Gate → gRPC-Web Transport → Command Service → Validation Pipeline

Without these four components, the v3 architecture cannot function.
