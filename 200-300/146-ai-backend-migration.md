# AI Backend Migration: Semantic Tool Layer & Workspace Integration

Status: In Execution  
Priority: Critical  
Complexity: XL (12–16 weeks)  
Related: 145-threads-artifact.md, 147-artifact-first-routes.md

## Executive Summary

Move all AI capabilities from frontend API routes to a backend tool layer that exposes semantic graph operations. AI will read and write through Projection + GraphPatch contracts (compact, validated), not UI commands. The workspace UI remains typed‑event driven: AI streams typed suggestions, the editor previews them, and acceptance applies an atomic patch. This eliminates API key exposure, enables rate/budget controls, and yields reliable, auditable edits.

## Problem Statement

### Critical Security & Architecture Issues

1. **API Key Exposure**: OpenAI/Anthropic keys required in frontend environment variables
2. **No Semantic Operations**: AI operates on UI commands instead of graph primitives
3. **Token Inefficiency**: Verbose UI command sequences instead of compact semantic operations  
4. **No Rate Limiting**: Cannot control AI usage per user or tenant
5. **No Cost Tracking**: Cannot monitor or limit AI spending
6. **No Audit Trail**: AI interactions not centrally logged
7. **No Resume Capability**: Lost streams cannot be recovered
8. **Vercel AI SDK Lock-in**: Deep coupling to proprietary streaming format

### Why a Tool Layer is Necessary

Current approaches force AI to emit dozens of order-sensitive UI commands to make simple changes. This is:
- **Brittle**: UI commands break with interface changes
- **Verbose**: 50+ commands for what should be 3-5 semantic operations
- **Hard to validate**: No pre/post conditions or semantic rule checking
- **Difficult to merge**: Order-dependent commands conflict easily

## Solution Architecture: Semantic Tool Layer

### Core Design Principle

**Invariant: All AI requests go through LangGraph.** No direct model API calls. Tools are allow-listed per agent; budgets and rate limits enforceable mid-stream.

**Expose modeling operations as semantic graph edits, not UI commands**. *UI commands are not used by AI.* AI writes **only** via GraphPatch; humans keep CommandStack. The compiler produces repo deltas (+ optional UI commands for visualization). Streams emit **semantic deltas**, not UI replay. The AI operates on a graph abstraction while humans continue using the familiar diagram-js interface. The backend compiles between these two planes.

```
┌──────────────┐          ┌─────────────────┐          ┌──────────────┐
│  AI Agent    │─────────▶│   Tool Layer    │─────────▶│  Model Store │
│ (Semantic)   │  Graph   │   (Compiler)    │  Events  │  (Source of  │
└──────────────┘  Patches └─────────────────┘          │    Truth)    │
                                │                       └──────────────┘
                                │ Compiles to                    ▲
                                ▼                                │
                          ┌─────────────────┐                   │
                          │  UI Commands    │───────────────────┘
                          │ (for humans)    │    Updates
                          └─────────────────┘
```

### Tool Contracts

#### A. ProjectionTool (Read Operations)

Purpose: Provide AI with compact, token-efficient views of architecture for reasoning.

```protobuf
service ProjectionService {
  // Get a projection optimized for AI reasoning
  rpc GetProjection(GetProjectionRequest) returns (ProjectionEnvelope);
  
  // Watch projection changes (for real-time awareness)
  rpc WatchProjection(WatchProjectionRequest) returns (stream ProjectionDelta);
  
  // List available model references
  rpc ListRefs(ListRefsRequest) returns (ListRefsResponse);
}

message GetProjectionRequest {
  string ref = 1;              // Model reference (branch/version)
  string intent = 2;            // What AI wants to do ("analyze", "modify", "validate")
  int32 token_budget = 3;       // Max tokens for response (AI-friendly)
  ProjectionOptions options = 4;
}

message ProjectionOptions {
  bool include_metrics = 1;     // Include performance metrics
  bool include_rules = 2;       // Include validation rules
  bool include_hotspots = 3;    // Include complexity hotspots
  int32 max_depth = 4;         // Graph traversal depth
  repeated string focus_types = 5;  // Element types to prioritize
}

message ProjectionEnvelope {
  string ref = 1;
  uint64 version = 2;           // Monotonic version for optimistic locking
  ProjectionGraph graph = 3;    // Synthetic, optimized graph
  repeated ValidationRule rules = 4;
  map<string, Metric> metrics = 5;
  repeated Hotspot hotspots = 6;
  int32 token_count = 7;        // Actual tokens used
  ProjectionMeta meta = 8;      // Context metadata
  repeated Scenario scenarios = 9;  // Top execution paths
  Delta delta = 10;             // Changes vs baseline
  Evidence evidence = 11;       // Supporting commits/explanations
}

message ProjectionGraph {
  repeated Node nodes = 1;      // Simplified nodes (pruned, collapsed)
  repeated Edge edges = 2;      // Relationships
  map<string, string> metadata = 3;
}

// Projection Envelope standard format:
// {
//   "meta": {"system":"...", "branch":"...", "as_of_version":58, "intent":"impact"},
//   "nodes":[{"id":"A1","type":"Activity","name":"Validate Order","score":0.82}],
//   "edges":[{"id":"E9","type":"ControlFlow","src":"A1","tgt":"G3"}],
//   "scenarios":[{"id":"P1","label":"Create Order","path":["Start","A1","G3","End"],"weight":0.61}],
//   "delta":{"added":[...],"removed":[...],"changed":[...]},
//   "hotspots":[{"id":"APP2","why":["fan_in","churn"]}],
//   "evidence":{"commits":["c1a2f","d9e0b"],"explain":["Collapsed A3,A4→M1"]}
// }

// Budget-fit pipeline: prune → collapse chains/SCC → focus hops → type quotas → scenario extraction
```

#### B. GraphPatchTool (Write Operations)

Purpose: Let AI propose semantic graph mutations with validation and compilation to graph changes.

```protobuf
service GraphPatchService {
  // Propose and validate a patch
  rpc ProposePatch(ProposePatchRequest) returns (ProposePatchResponse);
  
  // Apply a validated patch
  rpc ApplyPatch(ApplyPatchRequest) returns (ApplyPatchResponse);
  
  // Compare two model versions
  rpc Diff(DiffRequest) returns (DiffResponse);
  
  // Create a working branch
  rpc Branch(BranchRequest) returns (BranchResponse);
}

message ProposePatchRequest {
  string ref = 1;
  repeated GraphOp operations = 2;
  bool dry_run = 3;             // Validate without applying
}

message GraphOp {
  string id = 1;                // Unique ID for this operation
  string correlation_id = 2;    // Idempotency key
  
  enum OpType {
    OP_TYPE_UNSPECIFIED = 0;
    ADD_NODE = 1;
    REMOVE_NODE = 2;
    UPDATE_NODE = 3;
    CONNECT = 4;
    DISCONNECT = 5;
    REPLACE_SUBGRAPH = 6;      // Macro refactors
    RENAME = 7;
    SET_LIFECYCLE = 8;         // planned/current/retiring/retired
    MOVE = 9;
    GROUP = 10;
  }
  
  OpType type = 1;
  Selector selector = 2;        // What to operate on
  google.protobuf.Struct params = 3;  // Operation parameters
  repeated Condition pre_conditions = 4;   // Must be true before
  repeated Condition post_conditions = 5;  // Must be true after
}

message Selector {
  oneof selector {
    string node_id = 1;         // Direct ID reference
    string node_key = 2;        // Semantic key (e.g., "process.order-validation")
    TypeSelector by_type = 3;   // Select by type + filter
    NeighborSelector by_neighbor = 4;  // Select relative to other nodes
    PathSelector by_path = 5;   // Select along paths
  }
}

message ApplyPatchRequest {
  string ref = 1;
  repeated GraphOp operations = 2;
  uint64 expected_version = 3;  // Optimistic concurrency control
  string correlation_id = 4;    // Idempotency key
  string user_id = 5;
  string comment = 6;           // Commit message
}

message ApplyPatchResponse {
  bool success = 1;
  uint64 new_version = 2;
  string commit_id = 3;
  repeated Diagnostic diagnostics = 4;
  CompilationResult compilation = 5;  // How it compiled to UI commands
}

message CompilationResult {
  repeated UICommand ui_commands = 1;  // For diagram-js
  repeated RepositoryDelta repo_deltas = 2;  // File changes
  map<string, string> metadata = 3;
}
```

## Client Workspace Integration Plan (UI + Bus)

This section maps the backend migration to the existing workspace shell (typed bus, Coordinator, editor plugins), ensuring a drop‑in migration path.

### Current UI Contracts (kept)

- URL‑as‑state with a single writer (Coordinator) and a lease to suppress echo during URL→editor application.
- Editors are plugins: render view‑only, emit intents; domain writes go through a boundary (currently legacy command bus; target GraphPatch).
- Typed event bus carries UI intents and suggestion previews (suggest.createElement, suggest.connect, suggest.patch).

### Migration Phases for UI

1) Stream Gateway (SSE pass‑through)
- Server proxies backend typed suggestion events over existing `/api/threads/{id}/stream` SSE.
- Parser simply passes through backend‑typed `suggest.*` events (no heuristics required), with legacy fallback kept behind a flag.

2) Context & Projection
- Chat POST attaches workspace context (artifactId, selected elements, viewpoint, optional model snapshot).
- Backend ProjectionService returns compact graphs for LLMs; frontend consumes typed events only.

3) Atomic Accept via GraphPatch
- Add `useGraphPatch.apply(ops)` in UI that calls backend `ApplyPatch` (feature‑flagged).
- Editors switch Accept from multi-endpoint calls to single ApplyPatch; previews remain unchanged.

4) Decommission Frontend Keys
- All AI model keys removed from frontend; only backend endpoints used.
- Budget/rate limits enforced by the backend; typed telemetry events emitted to the client on limit hits.

### Acceptance Criteria (UI)

- Typed stream emits `suggest.graphPatch` (or compatible `suggest.patch`) consumed by the workspace parser with no schema drift.
- Accept on a suggestion applies atomically via GraphPatch; projection confirms.
- Lease still prevents selection loops; back/forward works; URL remains the single writer.
- Large models remain responsive (layout worker + virtualization in place).

### Telemetry & Resume

- Record `suggestion.accepted|rejected` with correlation IDs; log GraphPatch `commit_id`.
- Honor backend resume tokens for SSE; client parser remains idempotent.

### Backend Implementation Architecture

```
  +------------------------------+
  |  Router (Auth / Rate / Budg) |
  +---------------+--------------+
                  v
  +---------------+--------------+
  |   Execution Control Plane    |
  |   (Resume, Audit, Telemetry) |
  +---------+------------+-------+
            |            |
            v            v
  +---------+--+   +-----+-----------+
  | Projection |   |  GraphPatch     |
  |  Service   |   |    Service      |
  +-----+------+   +--------+--------+
        \                 /
         \               /
          v             v
         +---------------+
         |  Model Store  |
         +---------------+
```

## Diagrams

### Architecture Overview (ASCII)

```
CLIENT (Workspace)
  - @threads Panel
  - Stream Parser
  - Typed Bus
  - Editor Plugin (ArchiMate/BPMN)
  - Coordinator (URL single writer)
  - useGraphPatch.apply (acceptor)

BACKEND (AI Service)
  - Router (Auth / Rate / Budget)
  - Execution Control Plane (Resume / Audit)
  - ProjectionTool / GraphPatchTool
  - Model Store

Flow:
  Chat POST /api/threads + context -> Router -> Control Plane -> Tools
  Tools -> (SSE typed suggest.*) -> Chat UI -> Parser -> Bus -> Editor
  Editor Accept -> useGraphPatch.apply -> GraphPatchTool -> Model Store
  Model Store -> Projection/Delta -> Client (refresh)
```

### Streaming & Preview Sequence (Text)

1) User types a prompt in @threads.
2) Client POST /api/threads with workspace context (artifactId, selection, viewpoint).
3) Gateway forwards to AI backend (LangGraph pipeline).
4) Backend streams SSE parts (typed `suggest.*`).
5) Client parser converts parts → typed events.
6) Typed bus delivers events to the editor.
7) Editor renders ghost previews (create/connect/patch).

### Atomic Accept (GraphPatch) Sequence (Text)

1) User clicks Accept in the editor.
2) Editor calls useGraphPatch.apply(ops[], expectedVersion, correlationId).
3) Backend validates and applies the patch atomically (OCC).
4) Backend returns success (newVersion, commitId, diagnostics).
5) Editor clears ghost previews.
6) Projection refresh confirms the change and updates the canvas.

## Client Workspace Integration Plan (UI + Bus)

This section maps the backend migration to the existing workspace shell (typed bus, Coordinator, editor plugins), ensuring we can land the backend without breaking UI flows.

### Current UI Contracts (kept)

- URL-as-state with a single writer (Coordinator) and a lease to suppress echo during URL→editor application.
- Editors are plugins: render view-only, emit intents; domain writes go through a boundary (currently legacy command bus; target GraphPatch).
- Typed event bus carries UI intents and suggestion previews (suggest.createElement, suggest.connect, suggest.patch).

### Migration Phases for UI

1) Stream Gateway (SSE pass-through)
- Server proxies backend typed suggestion events over existing `/api/threads/{id}/stream` SSE.
- Parser simply passes through backend-typed `suggest.*` events (no heuristics required), with legacy fallback kept behind a flag.

2) Context & Projection
- Chat POST attaches workspace context (artifactId, selected elements, viewpoint, optional model snapshot).
- Backend ProjectionService returns compact graphs for LLMs; frontend carries only typed events.

3) Atomic Accept via GraphPatch
- Add `useGraphPatch.apply(ops)` in UI that calls backend `ApplyPatch` (feature-flagged).
- Editors switch Accept from multi-endpoint calls to single ApplyPatch; previews remain unchanged.

4) Decommission Frontend Keys
- All AI model keys removed from frontend; only backend endpoints used.
- Budget/rate limits enforced by the backend; typed telemetry events emitted to the client on limit hits.

### Acceptance Criteria (UI)

- Typed stream emits `suggest.graphPatch` (or compatible `suggest.patch`) consumed by the workspace parser with no schema drift.
- Accept on a suggestion applies atomically via GraphPatch; projection confirms.
- Lease still prevents selection loops; back/forward works; URL remains the single writer.
- Large models remain responsive (layout worker + virtualization in place).

### Telemetry & Resume

- Record `suggestion.accepted|rejected` with correlation IDs; log GraphPatch `commit_id`.
- Honor backend resume tokens for SSE; client parser remains idempotent.

## Frontend Action Items (Incremental)

- Add `useGraphPatch.apply()` hook (feature-flag) with a fallback that compiles patch ops to the legacy command bus (temporary shim).
- Route Accept in editors through GraphPatch hook (keeps preview logic unchanged).
- Move typed suggestion emit to the backend tools; keep client parser pass-through first.
- Add tests: lease suppression, typed preview → accept, SSE resume idempotency.


## Diagrams

### Architecture Overview (ASCII)

```
CLIENT (Workspace)
  - @threads Panel
  - Stream Parser
  - Typed Bus
  - Editor Plugin (ArchiMate/BPMN)
  - Coordinator (URL single writer)
  - useGraphPatch.apply (acceptor)

BACKEND (AI Service)
  - Router (Auth / Rate / Budget)
  - Execution Control Plane (Resume / Audit)
  - ProjectionTool / GraphPatchTool
  - Model Store

Flow:
  Chat POST /api/threads + context -> Router -> Control Plane -> Tools
  Tools -> (SSE typed suggest.*) -> Chat UI -> Parser -> Bus -> Editor
  Editor Accept -> useGraphPatch.apply -> GraphPatchTool -> Model Store
  Model Store -> Projection/Delta -> Client (refresh)
```

### Streaming & Preview Sequence (Text)

1) User types a prompt in @threads.
2) Client POST /api/threads with workspace context (artifactId, selection, viewpoint).
3) Gateway forwards to AI backend (LangGraph pipeline).
4) Backend streams SSE parts (typed `suggest.*`).
5) Client parser converts parts → typed events.
6) Typed bus delivers events to the editor.
7) Editor renders ghost previews (create/connect/patch).

### Atomic Accept (GraphPatch) Sequence (Text)

1) User clicks Accept in the editor.
2) Editor calls useGraphPatch.apply(ops[], expectedVersion, correlationId).
3) Backend validates and applies the patch atomically (OCC).
4) Backend returns success (newVersion, commitId, diagnostics).
5) Editor clears ghost previews.
6) Projection refresh confirms the change and updates the canvas.

### Workspace Integration Map (ASCII)

```
@threads -> Parser -> Typed Bus -> (Editor, Coordinator)
Editor -> selection.intent -> Bus
Coordinator -> debounced push -> URL (searchParams)
URL -> (lease apply) -> Editor
```

### Alternative View (ASCII Components)

```
CLIENT
  Chat UI -> Stream Parser -> Typed Bus -> Editor, Coordinator
  Editor -> useGraphPatch.apply

BACKEND
  Router/Rate/Budget -> Control Plane -> (ProjectionTool, GraphPatchTool) -> Model Store

Data Paths
  POST /api/threads (context) -> Backend
  SSE typed suggest.* -> Client Parser -> Bus -> Editor
  ApplyPatch -> Backend -> Model Store -> Projection -> Client
```


### Tool Service Implementation

#### ProjectionTool Service

```python
class ProjectionToolService:
    def __init__(self, model_store, rule_engine):
        self.model_store = model_store
        self.rule_engine = rule_engine
        self.tokenizer = tiktoken.get_encoding("cl100k_base")
    
    async def get_projection(self, request: GetProjectionRequest) -> ProjectionEnvelope:
        """Build AI-optimized projection within token budget."""
        
        # Load full model
        model = await self.model_store.get_model(request.ref)
        
        # Build projection based on intent
        projection_builder = ProjectionBuilder(
            intent=request.intent,
            token_budget=request.token_budget,
            options=request.options
        )
        
        # Intelligent sampling based on intent
        if request.intent == "analyze":
            # Focus on metrics and hotspots
            graph = await self._build_analysis_projection(model, projection_builder)
        elif request.intent == "modify":
            # Include neighboring context for changes
            graph = await self._build_modification_projection(model, projection_builder)
        elif request.intent == "validate":
            # Include rules and constraints
            graph = await self._build_validation_projection(model, projection_builder)
        
        # Compress to fit token budget
        compressed = await self._compress_to_budget(
            graph, 
            request.token_budget,
            preserve=request.options.focus_types
        )
        
        return ProjectionEnvelope(
            ref=request.ref,
            version=model.version,
            graph=compressed,
            rules=await self.rule_engine.get_relevant_rules(model),
            metrics=await self._calculate_metrics(model),
            hotspots=await self._identify_hotspots(model),
            token_count=self._count_tokens(compressed)
        )
    
    async def _compress_to_budget(self, graph, budget, preserve):
        """Intelligently compress graph to fit token budget."""
        
        current_tokens = self._count_tokens(graph)
        if current_tokens <= budget:
            return graph
        
        # Progressive compression strategies
        strategies = [
            lambda g: self._remove_annotations(g),
            lambda g: self._collapse_similar_nodes(g),
            lambda g: self._summarize_descriptions(g),
            lambda g: self._remove_low_importance_edges(g),
            lambda g: self._create_abstract_nodes(g)
        ]
        
        for strategy in strategies:
            graph = strategy(graph)
            if self._count_tokens(graph) <= budget:
                break
        
        # Ensure preserved types remain
        graph = self._restore_preserved_types(graph, preserve)
        
        return graph
    
    async def watch_projection(self, request: WatchProjectionRequest):
        """Stream projection deltas as model changes."""
        
        last_version = request.since_version
        stream_id = generate_id()
        
        async for event in self.model_store.watch(request.ref):
            if event.version <= last_version:
                continue
            
            # Calculate minimal delta
            delta = await self._calculate_delta(
                last_version,
                event.version,
                request.ref
            )
            
            # Only send if significant
            if delta.change_count > 0:
                yield ProjectionDelta(
                    stream_id=stream_id,
                    from_version=last_version,
                    to_version=event.version,
                    added_nodes=delta.added_nodes,
                    removed_nodes=delta.removed_nodes,
                    modified_nodes=delta.modified_nodes,
                    added_edges=delta.added_edges,
                    removed_edges=delta.removed_edges
                )
            
            last_version = event.version
```

#### GraphPatchTool Service

```python
class GraphPatchToolService:
    def __init__(self, model_store, rule_engine, compiler):
        self.model_store = model_store
        self.rule_engine = rule_engine
        self.compiler = compiler  # Compiles to UI commands
    
    async def propose_patch(self, request: ProposePatchRequest) -> ProposePatchResponse:
        """Validate patch without applying."""
        
        model = await self.model_store.get_model(request.ref)
        
        # Resolve selectors to concrete IDs
        resolved_ops = []
        for op in request.operations:
            resolved = await self._resolve_selector(op.selector, model)
            resolved_ops.append(GraphOp(
                type=op.type,
                target_ids=resolved,
                params=op.params,
                pre_conditions=op.pre_conditions,
                post_conditions=op.post_conditions
            ))
        
        # Validate operations
        diagnostics = []
        for op in resolved_ops:
            # Check pre-conditions
            for condition in op.pre_conditions:
                if not await self._check_condition(condition, model):
                    diagnostics.append(Diagnostic(
                        severity=Diagnostic.ERROR,
                        message=f"Pre-condition failed: {condition}",
                        operation=op
                    ))
            
            # Validate against notation rules (BPMN/UML/ArchiMate)
            rule_violations = await self.rule_engine.validate_op(op, model)
            diagnostics.extend(rule_violations)
        
        # Simulate application
        if request.dry_run and not diagnostics:
            simulated = await self._simulate_patch(model, resolved_ops)
            
            # Check post-conditions on simulated result
            for op in resolved_ops:
                for condition in op.post_conditions:
                    if not await self._check_condition(condition, simulated):
                        diagnostics.append(Diagnostic(
                            severity=Diagnostic.WARNING,
                            message=f"Post-condition would fail: {condition}",
                            operation=op
                        ))
        
        # Compile to UI commands for preview
        compilation = await self.compiler.compile_ops(resolved_ops, model)
        
        return ProposePatchResponse(
            valid=len([d for d in diagnostics if d.severity == Diagnostic.ERROR]) == 0,
            diagnostics=diagnostics,
            preview=compilation
        )
    
    async def apply_patch(self, request: ApplyPatchRequest) -> ApplyPatchResponse:
        """Apply validated patch with optimistic locking."""
        
        # Check idempotency
        existing = await self.model_store.get_by_correlation_id(request.correlation_id)
        if existing:
            return ApplyPatchResponse(
                success=True,
                new_version=existing.version,
                commit_id=existing.commit_id,
                diagnostics=[],
                compilation=existing.compilation
            )
        
        # Load and check version
        model = await self.model_store.get_model(request.ref)
        if model.version != request.expected_version:
            return ApplyPatchResponse(
                success=False,
                diagnostics=[Diagnostic(
                    severity=Diagnostic.ERROR,
                    message=f"Version mismatch: expected {request.expected_version}, got {model.version}"
                )]
            )
        
        # Validate (same as propose)
        validation = await self.propose_patch(ProposePatchRequest(
            ref=request.ref,
            operations=request.operations,
            dry_run=False
        ))
        
        if not validation.valid:
            return ApplyPatchResponse(
                success=False,
                diagnostics=validation.diagnostics
            )
        
        # Apply operations
        for op in request.operations:
            model = await self._apply_operation(op, model)
        
        # Compile to graph changes
        compilation = await self.compiler.compile_to_graph(
            request.operations,
            model
        )
        
        # Commit with correlation ID
        commit = await self.model_store.commit(
            ref=request.ref,
            model=model,
            user_id=request.user_id,
            comment=request.comment,
            correlation_id=request.correlation_id,
            compilation=compilation
        )
        
        # Broadcast change event for UI updates
        await self._broadcast_change(commit)
        
        return ApplyPatchResponse(
            success=True,
            new_version=commit.version,
            commit_id=commit.id,
            diagnostics=[],
            compilation=compilation
        )
    
    async def _apply_operation(self, op: GraphOp, model: Model) -> Model:
        """Apply a single graph operation."""
        
        if op.type == GraphOp.ADD_NODE:
            return await self._add_node(model, op.params)
        elif op.type == GraphOp.REMOVE_NODE:
            return await self._remove_node(model, op.selector)
        elif op.type == GraphOp.UPDATE_NODE:
            return await self._update_node(model, op.selector, op.params)
        elif op.type == GraphOp.CONNECT:
            return await self._connect_nodes(model, op.params)
        elif op.type == GraphOp.DISCONNECT:
            return await self._disconnect_nodes(model, op.params)
        elif op.type == GraphOp.REPLACE_SUBGRAPH:
            return await self._replace_subgraph(model, op.selector, op.params)
        # ... other operations
```

### LangGraph Agent Integration

Agents use these tools through the standard LangGraph tool interface:

```python
from langchain_core.tools import tool
from typing import List, Dict, Any

class ModelingTools:
    def __init__(self, projection_client, patch_client):
        self.projection = projection_client
        self.patch = patch_client
    
    @tool
    async def understand_model(
        self,
        artifact_id: str,
        intent: str = "analyze",
        focus: List[str] = None
    ) -> Dict[str, Any]:
        """Get a token-efficient view of the model for reasoning.
        
        Args:
            artifact_id: The model/artifact to understand
            intent: Your intent - 'analyze', 'modify', or 'validate'
            focus: Optional list of element types to focus on
        
        Returns:
            Optimized model projection with metrics and hotspots
        """
        response = await self.projection.get_projection(
            GetProjectionRequest(
                ref=f"artifacts/{artifact_id}/latest",
                intent=intent,
                token_budget=4000,  # Leave room for reasoning
                options=ProjectionOptions(
                    include_metrics=True,
                    include_hotspots=True,
                    focus_types=focus or []
                )
            )
        )
        
        return {
            "version": response.version,
            "nodes": len(response.graph.nodes),
            "edges": len(response.graph.edges),
            "graph": self._summarize_graph(response.graph),
            "hotspots": [h.description for h in response.hotspots[:5]],
            "metrics": dict(response.metrics),
            "rules": [r.description for r in response.rules[:10]]
        }
    
    @tool
    async def modify_model(
        self,
        artifact_id: str,
        operations: List[Dict[str, Any]],
        comment: str = "AI-assisted modification"
    ) -> Dict[str, Any]:
        """Apply semantic modifications to the model.
        
        Args:
            artifact_id: The model to modify
            operations: List of operations, each with:
                - type: 'add_node', 'remove_node', 'update_node', 'connect', etc.
                - selector: How to find the target (id, key, type, etc.)
                - params: Operation-specific parameters
            comment: Description of the changes
        
        Returns:
            Result with new version and any diagnostics
        """
        # First get current version for optimistic locking
        projection = await self.projection.get_projection(
            GetProjectionRequest(
                ref=f"artifacts/{artifact_id}/latest",
                token_budget=100  # Just need version
            )
        )
        
        # Convert dict operations to protobuf
        graph_ops = []
        for op_dict in operations:
            graph_ops.append(self._dict_to_graph_op(op_dict))
        
        # Apply patch
        response = await self.patch.apply_patch(
            ApplyPatchRequest(
                ref=f"artifacts/{artifact_id}/latest",
                operations=graph_ops,
                expected_version=projection.version,
                correlation_id=generate_id(),
                comment=comment
            )
        )
        
        if response.success:
            return {
                "success": True,
                "new_version": response.new_version,
                "commit_id": response.commit_id,
                "ui_commands": len(response.compilation.ui_commands),
                "message": f"Successfully applied {len(operations)} operations"
            }
        else:
            return {
                "success": False,
                "errors": [d.message for d in response.diagnostics if d.severity == "ERROR"],
                "warnings": [d.message for d in response.diagnostics if d.severity == "WARNING"]
            }

# Create BPMN specialist agent with modeling tools
def create_bpmn_agent():
    from langchain_openai import ChatOpenAI
    from langgraph.prebuilt import create_react_agent
    
    # Initialize tool clients
    projection_client = ProjectionServiceClient(grpc_channel)
    patch_client = GraphPatchServiceClient(grpc_channel)
    
    # Create tools
    modeling_tools = ModelingTools(projection_client, patch_client)
    
    # Additional BPMN-specific tools
    @tool
    def validate_bpmn(xml: str) -> str:
        """Validate BPMN XML against specification."""
        # Call BPMN validation service
        return "Validation results..."
    
    model = ChatOpenAI(model="gpt-4", temperature=0.3)
    
    return create_react_agent(
        model=model,
        tools=[
            modeling_tools.understand_model,
            modeling_tools.modify_model,
            validate_bpmn
        ],
        messages_modifier="""You are a BPMN process expert. You help users understand and improve their business processes.
        
        When analyzing processes, first use understand_model to get the current state, then provide insights.
        When making changes, use modify_model with semantic operations - the system will handle the UI updates.
        
        Always validate BPMN after modifications."""
    )
```

## Control Plane & Governance

### Unified Execution Through LangGraph

All AI operations flow through our existing control plane:

```
┌────────────┐     ┌─────────────┐     ┌──────────────┐     ┌──────────┐
│   Browser  │────▶│Envoy Gateway│────▶│  AI Backend  │────▶│ LangGraph│
│            │     │   (Auth,    │     │  (Router)    │     │ (Agents) │
│            │     │   CORS,     │     │              │     │          │
│            │     │   Rate)     │     │              │     │   Tools: │
└────────────┘     └─────────────┘     └──────────────┘     │  - Proj. │
     SSE                Connect             Connect          │  - Patch │
                                                             └──────────┘
```

### Streaming Implementation with Control

```python
class AIBackendService:
    async def ProcessChat(self, request, context):
        """Main threads endpoint with tool support."""
        
        # Rate limiting
        if not await self.rate_limiter.check_allowed(request.user_id):
            yield StreamEvent(
                type=StreamEvent.Type.ERROR,
                error=Error(code="RATE_LIMITED", message="Rate limit exceeded")
            )
            return
        
        # Get threads and determine agent
        threads = await self.threads_service.get_threads(request.threads_id)
        agent = self.agents.get(threads.agent_id, self.agents['general'])
        
        # Build context with artifact access
        context_data = {
            'threads_id': threads.id,
            'artifact_id': threads.artifact_id,
            'user_id': request.user_id,
            'tenant_id': request.tenant_id
        }
        
        # Stream from agent with budget enforcement
        stream_id = generate_id()
        event_index = 0
        cumulative_cost = 0.0
        
        async for chunk in agent.astream(
            {'messages': request.messages, 'context': context_data},
            stream_mode=['messages', 'updates', 'custom']
        ):
            event_index += 1
            
            # Track costs
            if chunk_cost := self._calculate_chunk_cost(chunk):
                cumulative_cost += chunk_cost
                
                # Check budget mid-stream
                if not await self.budget_guard.check_budget(
                    request.user_id,
                    cumulative_cost
                ):
                    yield StreamEvent(
                        type=StreamEvent.Type.ERROR,
                        error=Error(
                            code="BUDGET_EXCEEDED",
                            message="Daily budget limit reached"
                        )
                    )
                    break
            
            # Transform and yield
            event = self._transform_to_stream_event(chunk, stream_id, event_index)
            
            # Buffer for resume
            await self.stream_buffer.append(event)
            
            yield event
        
        # Send completion
        yield StreamEvent(
            type=StreamEvent.Type.COMPLETE,
            meta=StreamEventMeta(
                stream_id=stream_id,
                event_index=event_index + 1
            )
        )
```

### Envoy Gateway Configuration

```yaml
# Gateway configuration for streaming with proper timeouts
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: ClientTrafficPolicy
metadata:
  name: ai-streaming-policy
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: ai-backend
  timeout:
    http:
      requestReceivedTimeout: "0s"  # No timeout for streaming
  http1:
    http1Settings:
      enableTrailers: true  # For gRPC-Web
  http2:
    initialConnectionWindowSize: 1048576  # 1MB window
```

## Acceptance Criteria

### Tool Layer Success Metrics

- [ ] AI can read model projections in ≤4000 tokens
- [ ] AI can apply 3-10 semantic operations in single commit
- [ ] Operations compile to correct UI commands for live viewers
- [ ] Conflicts produce actionable diagnostics AI can fix
- [ ] All operations flow through auth/rate/budget control plane
- [ ] Resume works without duplicates or gaps
- [ ] Observability shows full trace: tenant → agent → tool → result

### Dual-Plane Success Criteria

- [ ] ✅ Agents never emit UI commands; only GraphPatch
- [ ] ✅ Compiler can generate optional UI commands server-side for animation
- [ ] ✅ Streams to clients carry **one** semantic delta per commit (or `full_reload`)

### Security & Governance

- [ ] No AI API keys in frontend environment
- [ ] All AI calls authenticated via JWT
- [ ] Rate limiting enforced per user/tier
- [ ] Budget enforcement with mid-stream cutoff
- [ ] Cost tracking accurate to $0.01
- [ ] Full audit trail with correlation IDs
- [ ] Tool access controlled by agent registry

### Developer Experience

- [ ] Tools documented with clear examples
- [ ] Mock implementations for testing
- [ ] Debugging UI shows tool calls and results
- [ ] Performance metrics for tool execution
- [ ] Error messages actionable by developers

## Migration Strategy

### Phase 1: Foundation (Weeks 1-4)
- [ ] Implement ProjectionTool service with Projection Envelope
- [ ] Implement GraphPatchTool service (+ compiler & validators)
- [ ] Create model compiler (semantic → UI commands)
- [ ] Add rule validation engine
- [ ] Set up version control and optimistic locking (expected_version)
- [ ] Connect-Web v2 wiring with proper streaming
- [ ] Envoy policies (`request: "0s"`, `streamIdleTimeout: "600s"`, heartbeats)
- [ ] Create comprehensive test suite

### Phase 2: Dual-Plane Integration (Weeks 5-6)
- [ ] Canvas optimistic CommandStack for humans
- [ ] AI via GraphPatch only (enforce at gateway)
- [ ] Resume tokens with Redis XADD (MaxLen≈1000, TTL≈5m)
- [ ] Usage frames + budget cut-off (Error then CANCELLED)
- [ ] Integrate with LangGraph agent runtime
- [ ] Add auth/rate/budget interceptors
- [ ] Create observability dashboard

### Phase 3: Agent Development (Weeks 7-9)
- [ ] BPMN agent using Projection/GraphPatch
- [ ] ArchiMate agent using Projection/GraphPatch
- [ ] **UML-Lite (Class+Component)** on diagram-js (JSON I/O)
- [ ] Canonical graph mappers
- [ ] Risk Analysis agent
- [ ] Code Generation agent
- [ ] Decision (DMN) agent

### Phase 4: Frontend Integration (Weeks 10-12)
- [ ] Frontend review UI for Projection & Patch
- [ ] Update TypeScript SDK with tool services
- [ ] Create /threads-v2 UI with agent selection
- [ ] Implement streaming event handlers
- [ ] Add tool invocation visualization
- [ ] Create error recovery flows
- [ ] Playwright acceptance suites (resume, heartbeats, budgets, conflicts, branching/baselines)

### Phase 5: Hardening (Weeks 13-16)
- [ ] Performance optimization
- [ ] Chaos testing
- [ ] Security audit
- [ ] Tracing dashboards
- [ ] Canary rollout
- [ ] Documentation and training
- [ ] Feature flag deployment
- [ ] A/B testing metrics
- [ ] Gradual traffic migration
- [ ] Production monitoring
- [ ] Documentation and training
- [ ] Feedback collection

## Risk Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| Token budget exceeded | High | Progressive compression, sampling strategies |
| Conflicting edits | Medium | Optimistic locking, clear diagnostics |
| Rule violations | Medium | Pre-validation in propose_patch |
| Performance degradation | High | Caching, incremental projections |
| Complex selectors | Low | Multiple selector types, fallback to IDs |

## Implementation Notes

### Why Semantic Operations Win

1. **Idempotent**: Same patch produces same result
2. **Mergeable**: Non-overlapping patches can combine
3. **Validatable**: Pre/post conditions ensure correctness
4. **Rebaseable**: Can replay on new versions
5. **Auditable**: Clear intent in operation history

### Compiler Design

The compiler translates between semantic operations and UI commands:

```python
class OperationCompiler:
    def compile_add_node(self, op, model):
        # Semantic: add_node(type="Task", name="Validate Order")
        # Compiles to diagram-js commands:
        return [
            UICommand(
                type="shape.create",
                context={
                    "shape": {
                        "type": "bpmn:Task",
                        "businessObject": {"name": "Validate Order"}
                    },
                    "position": self._calculate_position(model)
                }
            ),
            UICommand(
                type="element.updateLabel",
                context={"element": "{{created}}", "newLabel": "Validate Order"}
            )
        ]
```

### Projection Optimization

Projections use intelligent sampling to fit token budgets:

1. **Hotspot Detection**: Identify complex areas needing attention
2. **Progressive Detail**: More detail near focus, less at periphery  
3. **Type Prioritization**: Include all instances of focus_types
4. **Relationship Pruning**: Keep only significant edges
5. **Annotation Stripping**: Remove verbose descriptions if needed

## Why Semantic Operations Win

**Dual-plane payoff:** Order-insensitive ops, deterministic merges, lower token use, clear audits, and simpler conflict handling than long UI command chains. This is now a platform rule.

- **Token Efficiency**: 70% reduction vs UI commands
- **Reliability**: Idempotent operations with correlation IDs
- **Merge Safety**: Semantic ops are order-independent
- **Debugging**: Clear semantic intent vs opaque command sequences
- **Validation**: Pre/post conditions enforced at semantic level
- **Evolution**: UI can change without breaking AI agents

## Related Documents

- [145-threads-artifact.md](./145-threads-artifact.md) - Chat/Artifact gRPC backend
- [113-canvas-platform-integration-artifact-centric-graph.md](./113-canvas-platform-integration-artifact-centric-graph.md) - Canvas/UI command integration
- [144-bpmn-lifecycle.md](./144-bpmn-lifecycle.md) - Model lifecycle management
