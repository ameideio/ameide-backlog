# **Diagram Editor North Star Alignment — Event-Sourced Command Stack for BPMN & ArchiMate**

> **Audience** Engineering Team
> **Status** Draft v2.0
> **Objective** Align the existing BPMN and ArchiMate editing experience with the North Star vision by implementing an event-sourced command stack that enables governed design-time modeling, comprehensive audit trails, and seamless deployment.

---

## Executive Summary

The North Star vision (064) describes a Camunda SaaS-style experience with client-side command stacks, immutable artifact storage, and governance gates. Our current implementation has strong foundations (event sourcing infrastructure, diagram-js with both bpmn-js and archimate-js, Camunda generation) but uses traditional CRUD patterns instead of the event-sourced command architecture.

This backlog item outlines a focused MVP that delivers governed save-and-deploy capabilities for both BPMN and ArchiMate diagrams, with real-time collaboration deferred as a future enhancement.

---

## Current State Analysis

### What We Have ✅

1. **Frontend (Portal Canvas)**
   - Full diagram-js integration supporting both bpmn-js v17.2.1 and archimate-js
   - Access to CommandStack with change detection for both diagram types
   - TypeScript SDK with proper types
   - React Query + Zustand state management

2. **Backend Infrastructure**
   - Robust event sourcing system (EventStore protocol)
   - Complete BPMN 2.0 parsing and ArchiMate model support
   - Comprehensive Camunda BPMN/Java generation
   - Business event tracking via decorators
   - gRPC service definitions for BPMN and ArchiMate operations

3. **What's Missing ❌**
   - Command interception and persistence
   - Command-based API endpoints with validation
   - Command history UI
   - Idempotency and conflict resolution
   - Local persistence queue for offline resilience

---

## Re-Sequenced Implementation Plan (MVP-First)

### Phase A: Command Infrastructure + Validation (4 weeks)

**Goal**: Capture, validate, and persist diagram-js commands for both BPMN and ArchiMate with pre-commit validation

1. **Extend Proto for Unified Commands** (`packages/ameide_core_proto/proto/ameide_core_proto/common/v1/diagram_commands.proto`):
   ```proto
   message ModelingCommand {
     string id = 1;  // e.g., "element.updateProperties"
     bytes raw_context = 2;  // gzipped JSON for stability
     google.protobuf.Timestamp timestamp = 3;
     string user_id = 4;
     string command_uuid = 5;  // UUIDv7 for idempotency
     uint64 sequence = 6;  // monotonic counter per document
     string diagram_type = 7;  // "bpmn" or "archimate"
     uint32 version = 8;  // command schema version
   }
   
   message ValidationResult {
     bool valid = 1;
     repeated string errors = 2;
     repeated string warnings = 3;
   }
   
   service DiagramCommandService {
     // Validate then execute with idempotency
     rpc ExecuteCommand(ExecuteCommandRequest) returns (ExecuteCommandResponse);
     rpc GetCommandHistory(GetCommandHistoryRequest) returns (stream ModelingCommand);
   }
   ```

2. **Create Unified Command Service with Validation** (`packages/ameide_core-services/src/ameide_core_services/diagram/command_service.py`):
   ```python
   @traced
   class DiagramCommandService(ServiceBase):
       def __init__(
           self, 
           event_store: EventStore,
           validators: Dict[str, CommandValidator]
       ):
           self.event_store = event_store
           self.validators = validators  # {"bpmn": BPMNValidator, "archimate": ArchiMateValidator}
       
       @business_operation(BusinessEvent.DIAGRAM_COMMAND_EXECUTE)
       async def execute_command(
           self,
           document_id: str,
           command: ModelingCommand,
           expected_sequence: Optional[int] = None
       ) -> CommandResult:
           # Idempotency check
           if await self._command_exists(document_id, command.command_uuid):
               return await self._get_command_result(document_id, command.command_uuid)
           
           # Pre-commit validation
           validator = self.validators[command.diagram_type]
           validation = await validator.validate_command(command)
           if not validation.valid:
               return CommandResult(
                   success=False,
                   errors=validation.errors,
                   warnings=validation.warnings
               )
           
           # Persist with sequence check
           stream_id = f"diagram-commands-{document_id}"
           event = Event(
               type="ModelingCommandExecuted",
               data={
                   "command_id": command.id,
                   "raw_context": command.raw_context,  # Already gzipped
                   "sequence": command.sequence,
                   "diagram_type": command.diagram_type
               },
               metadata={
                   "user_id": command.user_id,
                   "command_uuid": command.command_uuid
               }
           )
           
           await self.event_store.append(
               stream_id, 
               [event], 
               expected_version=expected_sequence
           )
           
           return CommandResult(
               success=True, 
               sequence=command.sequence,
               command_uuid=command.command_uuid
           )
   ```

3. **Portal Command Interceptor with Local Queue** (`packages/ameide_core-sdk-ts/src/services/diagram/commandInterceptor.ts`):
   ```typescript
   import Dexie from 'dexie';
   import { v7 as uuidv7 } from 'uuid';
   import pako from 'pako';
   
   class CommandQueue extends Dexie {
     commands!: Dexie.Table<QueuedCommand, string>;
     
     constructor() {
       super('DiagramCommands');
       this.version(1).stores({
         commands: 'uuid, documentId, sequence, synced'
       });
     }
   }
   
   export class DiagramCommandInterceptor {
     private queue = new CommandQueue();
     private sequence = 0;
     
     constructor(
       private client: AmeideClient,
       private documentId: string,
       private diagramType: 'bpmn' | 'archimate'
     ) {}
     
     async install(modeler: any) {
       const eventBus = modeler.get('eventBus');
       
       // Load last sequence
       this.sequence = await this.getLastSequence();
       
       // Intercept only executed events to avoid double-fire
       eventBus.on('commandStack.*.executed', async (event: any) => {
         const { command, context } = event;
         
         const queuedCommand = {
           uuid: uuidv7(),
           documentId: this.documentId,
           sequence: ++this.sequence,
           command: {
             id: command,
             raw_context: pako.deflate(JSON.stringify(context)),
             timestamp: new Date().toISOString(),
             user_id: this.client.currentUser.id,
             command_uuid: uuidv7(),
             sequence: this.sequence,
             diagram_type: this.diagramType,
             version: 1
           },
           synced: false
         };
         
         // Save to IndexedDB first
         await this.queue.commands.add(queuedCommand);
         
         // Async sync attempt
         this.syncCommand(queuedCommand).catch(err => 
           console.warn('Command sync failed, will retry:', err)
         );
       });
       
       // Start sync worker
       this.startSyncWorker();
     }
     
     private async syncCommand(cmd: QueuedCommand) {
       const result = await this.client.diagram.executeCommand({
         documentId: cmd.documentId,
         command: cmd.command,
         expectedSequence: cmd.sequence - 1
       });
       
       if (result.success) {
         await this.queue.commands.update(cmd.uuid, { synced: true });
       } else if (result.errors?.includes('SEQUENCE_MISMATCH')) {
         // Handle conflict - needs OT or retry strategy
         throw new Error('Sequence conflict');
       }
     }
     
     private async startSyncWorker() {
       setInterval(async () => {
         const unsynced = await this.queue.commands
           .where('synced').equals(0)
           .toArray();
         
         for (const cmd of unsynced) {
           await this.syncCommand(cmd).catch(() => {});
         }
       }, 5000);
     }
   }
   ```

4. **Validation Strategies** (`packages/ameide_core-services/src/ameide_core_services/diagram/validators.py`):
   ```python
   class BPMNCommandValidator:
       async def validate_command(self, command: ModelingCommand) -> ValidationResult:
           context = json.loads(gzip.decompress(command.raw_context))
           
           # BPMN-specific rules
           if command.id == "shape.create":
               element_type = context.get("shape", {}).get("type")
               if element_type not in VALID_BPMN_ELEMENTS:
                   return ValidationResult(
                       valid=False,
                       errors=[f"Invalid BPMN element type: {element_type}"]
                   )
           
           # Camunda extension validation
           if "camunda:" in str(context):
               return await self._validate_camunda_extensions(context)
           
           return ValidationResult(valid=True)
   
   class ArchiMateCommandValidator:
       async def validate_command(self, command: ModelingCommand) -> ValidationResult:
           context = json.loads(gzip.decompress(command.raw_context))
           
           # ArchiMate layer rules
           if command.id == "connection.create":
               source_layer = self._get_element_layer(context["source"])
               target_layer = self._get_element_layer(context["target"])
               if not self._is_valid_relationship(source_layer, target_layer):
                   return ValidationResult(
                       valid=False,
                       errors=["Invalid cross-layer relationship"]
                   )
           
           return ValidationResult(valid=True)
   ```

### Phase B: Snapshots & Fast Replay (2 weeks)

**Goal**: Content-addressed storage and sub-second replay for 1000+ commands

1. **Snapshot Service** (`packages/ameide_core-services/src/ameide_core_services/diagram/snapshot_service.py`):
   ```python
   class DiagramSnapshotService:
       def __init__(self, blob_store: BlobStore, cache: CacheStore):
           self.blob_store = blob_store
           self.cache = cache
       
       async def create_snapshot(
           self,
           document_id: str,
           commands: List[ModelingCommand],
           diagram_type: str
       ) -> Snapshot:
           # Replay commands to generate XML/JSON
           if diagram_type == "bpmn":
               content = await self._replay_bpmn_commands(commands)
           else:
               content = await self._replay_archimate_commands(commands)
           
           # Content-addressed storage
           snapshot_sha = hashlib.sha256(content.encode()).hexdigest()
           key = f"snapshots/{diagram_type}/{snapshot_sha}"
           
           # Store with metadata
           await self.blob_store.put(
               key,
               gzip.compress(content.encode()),
               metadata={
                   "document_id": document_id,
                   "command_count": len(commands),
                   "diagram_type": diagram_type
               }
           )
           
           # Cache for fast retrieval
           await self.cache.set(
               f"snapshot:{document_id}:latest",
               snapshot_sha,
               ttl=3600
           )
           
           return Snapshot(
               sha=snapshot_sha,
               command_count=len(commands),
               size_bytes=len(content)
           )
       
       async def load_with_replay(
           self,
           document_id: str,
           target_sequence: Optional[int] = None
       ) -> DiagramState:
           # Check cache first
           cached_sha = await self.cache.get(f"snapshot:{document_id}:latest")
           
           if cached_sha and not target_sequence:
               # Fast path: return cached snapshot
               return await self._load_snapshot(cached_sha)
           
           # Replay from event store
           commands = await self._load_commands(document_id, target_sequence)
           
           # Performance guard: snapshot if > 100 commands
           if len(commands) > 100 and not target_sequence:
               snapshot = await self.create_snapshot(
                   document_id, 
                   commands,
                   commands[0].diagram_type
               )
           
           return await self._replay_to_state(commands)
   ```

### Phase C: Command History UI (1 week)

**Goal**: Timeline view with replay capabilities and diff visualization

1. **Command History Component** (`services/www-ameide-portal-canvas/src/components/CommandHistory.tsx`):
   ```typescript
   export const CommandHistory: React.FC<{ 
     documentId: string;
     diagramType: 'bpmn' | 'archimate';
   }> = ({ documentId, diagramType }) => {
     const { data: commands } = useQuery({
       queryKey: ['diagram', 'commands', documentId],
       queryFn: () => sdk.diagram.getCommandHistory({ 
         documentId,
         limit: 100  // Pagination for performance
       })
     });
     
     const replayToSequence = async (sequence: number) => {
       // Server-side replay
       const state = await sdk.diagram.replayToSequence({
         documentId,
         targetSequence: sequence
       });
       
       // Update modeler
       if (diagramType === 'bpmn') {
         await modeler.importXML(state.content);
       } else {
         await modeler.importJSON(state.content);
       }
     };
     
     return (
       <div className="command-history">
         <h3>Version History</h3>
         <div className="timeline">
           {commands?.map((cmd) => (
             <TimelineItem
               key={cmd.sequence}
               command={cmd}
               onReplay={() => replayToSequence(cmd.sequence)}
               icon={getCommandIcon(cmd.id)}
               timestamp={cmd.timestamp}
               user={cmd.userId}
             />
           ))}
         </div>
       </div>
     );
   };
   ```

### Phase D: Collaboration Spike (2 week time-box)

**Goal**: Feasibility assessment and cost-benefit analysis for real-time collaboration

1. **Research Tasks**:
   - Evaluate OT libraries (Yjs vs ShareJS vs custom)
   - Prototype conflict resolution for diagram-js commands
   - Measure WebSocket infrastructure costs
   - Test with 10 concurrent users on complex diagrams

2. **Deliverables**:
   - Working prototype (if feasible)
   - Performance metrics and infrastructure costs
   - Go/No-go recommendation with ROI analysis
   - Alternative approaches (e.g., session-based locking)


---

## Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Command capture rate | 100% of modeling operations | Commands in event store / Total diagram-js commands |
| Command replay accuracy | 100% fidelity | Replayed content == Original content |
| Initial load time | < 1s @ 1000 commands | Time to replay on MacBook Air M1 |
| Validation performance | < 100ms p95 | Pre-commit validation time |
| Storage efficiency | < 2KB per command (gzipped) | Average command size in event store |
| Offline resilience | 100% local capture | Commands saved during network outage |
| Idempotency | Zero duplicates | Unique command_uuid enforcement |

---

## Migration Strategy

1. **Feature Flag Rollout**:
   ```typescript
   const FEATURES = {
     COMMAND_PERSISTENCE: false,      // Phase A
     VALIDATION_GATES: false,         // Phase A
     SNAPSHOT_OPTIMIZATION: false,    // Phase B
     COMMAND_HISTORY_UI: false,       // Phase C
     COLLABORATION_SPIKE: false       // Phase D (experimental)
   };
   ```

2. **Backward Compatibility**:
   - Keep existing CRUD endpoints operational
   - Dual-write to both systems during migration
   - Legacy XML import creates single "document.import" command

3. **Data Migration**:
   ```python
   async def migrate_existing_documents():
       """Convert existing documents to command streams with clear attribution"""
       for doc in await document_repo.get_all():
           # Single import command preserves original create time
           command = ModelingCommand(
               id="document.import",
               raw_context=gzip.compress(json.dumps({
                   "content": doc.content,
                   "type": doc.diagram_type,
                   "legacy": True
               }).encode()),
               timestamp=doc.created_at,
               user_id="system:migration",
               command_uuid=uuidv7(),
               sequence=1,
               diagram_type=doc.diagram_type
           )
           
           await command_service.execute_command(
               doc.id, 
               command,
               skip_validation=True  # Legacy content already validated
           )
           
           logger.info(
               "Migrated document",
               document_id=doc.id,
               diagram_type=doc.diagram_type
           )
   ```

---

## Technical Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Command size explosion | High storage costs | Gzip compression + automatic snapshots at 100 commands |
| Sequence conflicts | Inconsistent state | Monotonic sequence counter with expected_sequence checks |
| Network partitions | Lost commands | IndexedDB queue with background sync worker |
| Performance degradation | Poor UX | Snapshot-based fast load + command pagination |
| Validation blocks user | Frustrating UX | Pre-commit validation with clear error messages |
| Command schema evolution | Breaking changes | Version field + proto oneof for backward compatibility |
| BPMN-js API changes | Integration breaks | Pin versions, schedule upgrade windows |
| PII in commands | Compliance risk | Scrub sensitive data before persistence |

---

## Dependencies

- **Existing packages to enhance**:
  - `core-storage` (event store, blob store, cache)
  - `core-services` (add diagram command service)
  - `core-sdk-ts` (command interceptor)
  - `www-ameide-portal-canvas` (UI integration)
  - `core-domain` (extend for command models)

- **New dependencies**:
  - `dexie` (IndexedDB wrapper for offline queue)
  - `pako` (gzip compression in browser)
  - `uuid` (UUIDv7 for command IDs)

- **External dependencies**:
  - diagram-js (base for both bpmn-js and archimate-js)
  - bpmn-js v17+ (already in use)
  - archimate-js (already integrated)

---

## Acceptance Criteria

### Phase A (Command Infrastructure + Validation)
- [ ] Every diagram-js operation creates an event in the event store
- [ ] Commands are validated before persistence
- [ ] Invalid commands return clear error messages
- [ ] Idempotent command execution via UUID
- [ ] Offline commands queue in IndexedDB
- [ ] Background sync recovers from network failures

### Phase B (Snapshots & Fast Replay)
- [ ] Snapshots auto-created at 100+ commands
- [ ] Content-addressed storage with SHA-256
- [ ] Initial load < 1s for 1000 commands
- [ ] Cache integration for latest snapshots

### Phase C (Command History UI)
- [ ] Timeline view shows all commands
- [ ] User attribution on each command
- [ ] Replay to any point in history
- [ ] Visual diff between versions

### Overall
- [ ] Existing documents migrated without data loss
- [ ] Both BPMN and ArchiMate fully supported
- [ ] Performance metrics dashboard operational
- [ ] Zero regression in existing functionality

---

## Next Steps

1. **Technical Spike** (3 days):
   - Prototype command interception for both bpmn-js and archimate-js
   - Test event store performance with 10k compressed commands
   - Validate IndexedDB sync approach with network simulation

2. **Architecture Review**:
   - Present focused MVP design to architecture board
   - Confirm deferral of real-time collaboration
   - Align with Q1-Q2 roadmap priorities

3. **Implementation**:
   - Start with Phase A (Command Infrastructure + Validation)
   - Weekly demos focusing on governance benefits
   - Incremental rollout with comprehensive monitoring

---

## References

- North Star Vision: `/backlog/064-north-star.md`
- Current BPMN Architecture: `/backlog/063-bpmn-camunda.md`
- Event Store Protocol: `/packages/ameide_core-storage/src/ameide_core_storage/eventstore/base.py`
- Portal Canvas: `/services/www_ameide-portal-canvas/`
- BPMN Proto: `/packages/ameide_core_proto/proto/ameide_core_proto/bpmn/v1/bpmn.proto`
- ArchiMate Proto: `/packages/ameide_core_proto/proto/ameide_core_proto/archimate/v1/archimate_core.proto`
- diagram-js Documentation: https://github.com/bpmn-io/diagram-js

---

## Summary of Changes from v1.0

Based on comprehensive feedback, this v2.0 revision:

1. **Refocused on MVP**: Moved real-time collaboration to optional Phase D spike
2. **Added ArchiMate**: Unified command infrastructure for both diagram types
3. **Enhanced Technical Depth**:
   - Command idempotency via UUIDv7
   - Local persistence with IndexedDB/Dexie
   - Pre-commit validation (moved from Phase 4 to Phase A)
   - Gzipped command storage
   - Performance guardrails (1s load @ 1k commands)
4. **Addressed Risks**: Added 8 specific mitigations for identified risks
5. **Corrected Event Handling**: Using `*.executed` events to avoid double-fire
6. **Clarified Migration**: Single import command preserves attribution