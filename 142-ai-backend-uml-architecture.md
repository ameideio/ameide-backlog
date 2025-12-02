# AI Backend Architecture - UML & Dual-Plane Overview

**Invariant: All AI requests go through LangGraph.** No direct model API calls. Tools are allow-listed per agent; budgets and rate limits enforceable mid-stream.

## Executive Summary

UML architecture diagram visualizing the AI backend POC from backlog item 141, showing the multi-layer architecture with Connect protocol streaming, control plane separation, and GraphPatch semantic operations.

## Component Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    <<system>>                                       │
│                               AMEIDE AI Platform                                    │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐ │
│  │                            <<layer>>                                          │ │
│  │                         Presentation Layer                                    │ │
│  ├──────────────────────────────────────────────────────────────────────────────┤ │
│  │                                                                               │ │
│  │  ┌─────────────────────┐   ┌─────────────────────┐   ┌──────────────────┐  │ │
│  │  │   <<component>>      │   │   <<component>>      │   │  <<component>>   │  │ │
│  │  │   React Chat UI      │   │   BPMN Editor UI    │   │  ArchiMate UI    │  │ │
│  │  │  ───────────────     │   │  ────────────────    │   │ ──────────────   │  │ │
│  │  │ + useInferenceStream │   │ + GraphPatchReview   │   │ + RefactorView   │  │ │
│  │  │ + ChatMessages       │   │ + OperationCard      │   │ + LayerSelector  │  │ │
│  │  │ + ChatInput          │   │ + PreviewPanel       │   │ + ViewpointEdit  │  │ │
│  │  │                     │   │ + Projection Review  │   │                   │  │ │
│  │  └──────────┬───────────┘   └──────────┬───────────┘   └────────┬─────────┘  │ │
│  │             │                           │                        │            │ │
│  │             └───────────────────────────┼────────────────────────┘            │ │
│  │                                         │                                     │ │
│  │                            <<uses>>     ▼     <<protocol>>                    │ │
│  │                         ┌────────────────────────────────┐                    │ │
│  │                         │    Connect-Web v2 Transport    │                    │ │
│  │                         │  • HTTP/1.1 & HTTP/2 Support   │                    │ │
│  │                         │  • Binary/JSON Serialization   │                    │ │
│  │                         │  • Server-Sent Events (SSE)    │                    │ │
│  │                         └────────────────────────────────┘                    │ │
│  └──────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐ │
│  │                            <<layer>>                                          │ │
│  │                        Infrastructure Layer                                   │ │
│  ├──────────────────────────────────────────────────────────────────────────────┤ │
│  │                                                                               │ │
│  │  ┌────────────────────────────────────────────────────────────────────────┐  │ │
│  │  │                        <<component>>                                    │  │ │
│  │  │                      Envoy Gateway (Port 8443)                         │  │ │
│  │  │  ───────────────────────────────────────────────────────────────────   │  │ │
│  │  │  + JWT Authentication           : SecurityPolicy                       │  │ │
│  │  │  + Rate Limiting                : BackendTrafficPolicy                 │  │ │
│  │  │  + CORS Headers                 : SecurityPolicy                       │  │ │
│  │  │  + Stream Timeouts              : ClientTrafficPolicy                  │  │ │
│  │  │  + Load Balancing               : HTTPRoute                           │  │ │
│  │  │  + Circuit Breaking             : BackendTrafficPolicy                 │  │ │
│  │  │  + Telemetry & Tracing         : ClientTrafficPolicy                  │  │ │
│  │  └────────────────────────────────┬───────────────────────────────────────┘  │ │
│  │                                    │                                          │ │
│  └────────────────────────────────────┼──────────────────────────────────────────┘ │
│                                       ▼                                             │
│  ┌──────────────────────────────────────────────────────────────────────────────┐ │
│  │                            <<layer>>                                          │ │
│  │                          Control Plane                                        │ │
│  ├──────────────────────────────────────────────────────────────────────────────┤ │
│  │                                                                               │ │
│  │  ┌────────────────────────────────────────────────────────────────────────┐  │ │
│  │  │                        <<component>>                                    │  │ │
│  │  │                   Go Gateway Service (Port 6003)                       │  │ │
│  │  │  ───────────────────────────────────────────────────────────────────   │  │ │
│  │  │  <<interface>> ControlPlane                                            │  │ │
│  │  │  + ProtoTranslation()          : Connect → REST/SSE                   │  │ │
│  │  │  + RequestForwarding()         : Route to execution engine            │  │ │
│  │  │  + HealthCheck()               : Service monitoring                   │  │ │
│  │  │                                                                        │  │ │
│  │  │  <<interface>> PolicyEnforcement (Future)                              │  │ │
│  │  │  + RedisBuffering()            : Stream resume support                │  │ │
│  │  │  + TenantRateLimit()           : Per-tenant quotas                    │  │ │
│  │  │  + TokenBudget()               : Usage tracking                       │  │ │
│  │  │  + StreamHeartbeat()           : Keep-alive mechanism                 │  │ │
│  │  ├────────────────────────────────────────────────────────────────────────┤  │ │
│  │  │                     <<component>>                                       │  │ │
│  │  │                  GraphPatch Service                                    │  │ │
│  │  │  ───────────────────────────────────────────────────────────────────   │  │ │
│  │  │  + ProposePatch()              : Generate semantic operations          │  │ │
│  │  │  + ReviewPatch()               : Human approval workflows              │  │ │
│  │  │  + ApplyPatch()                : Execute approved changes             │  │ │
│  │  │  + ValidatePatch()             : Domain rule validation               │  │ │
│  │  │  + CompileToOperations()       : Convert to UI commands               │  │ │
│  │  ├────────────────────────────────────────────────────────────────────────┤  │ │
│  │  │                     <<component>>                                       │  │ │
│  │  │                  Projection Service (ProjectionTool)                   │  │ │
│  │  │  ───────────────────────────────────────────────────────────────────   │  │ │
│  │  │  + GetProjection()             : Token-budgeted context               │  │ │
│  │  │  + WatchProjection()           : Stream delta updates                 │  │ │
│  │  │  + ListReferences()            : Cross-model traceability             │  │ │
│  │  └────────────────────────────────┬───────────────────────────────────────┘  │ │
│  │                                    │                                          │ │
│  └────────────────────────────────────┼──────────────────────────────────────────┘ │
│                                       ▼                                             │
│  ┌──────────────────────────────────────────────────────────────────────────────┐ │
│  │                            <<layer>>                                          │ │
│  │                         Execution Engine                                      │ │
│  ├──────────────────────────────────────────────────────────────────────────────┤ │
│  │                                                                               │ │
│  │  ┌────────────────────────────────────────────────────────────────────────┐  │ │
│  │  │                        <<component>>                                    │  │ │
│  │  │                 Python FastAPI Service (Port 8000)                     │  │ │
│  │  │  ───────────────────────────────────────────────────────────────────   │  │ │
│  │  │  + AgentSelection()            : Route by agent_id                    │  │ │
│  │  │  + LLMIntegration()            : Direct API calls                     │  │ │
│  │  │  + SSEStreaming()              : Real-time responses                  │  │ │
│  │  │  + ContextManagement()         : Conversation state                   │  │ │
│  │  │  + ToolExecution()             : Agent capabilities                   │  │ │
│  │  │  + PromptEngineering()         : Template management                  │  │ │
│  │  └──────────┬─────────────────────────────────────────────────────────────┘  │ │
│  │             │                                                                 │ │
│  │             ▼                                                                 │ │
│  │  ┌────────────────────────────────────────────────────────────────────────┐  │ │
│  │  │                        <<component>>                                    │  │ │
│  │  │                    LangGraph Agent Registry                            │  │ │
│  │  │  ───────────────────────────────────────────────────────────────────   │  │ │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                │  │ │
│  │  │  │<<agent>>     │  │<<agent>>     │  │<<agent>>     │                │  │ │
│  │  │  │simple-threads   │  │react-agent   │  │code-agent    │                │  │ │
│  │  │  │─────────────│  │─────────────│  │─────────────│                │  │ │
│  │  │  │+ basic conv │  │+ tools       │  │+ generation  │                │  │ │
│  │  │  │+ no tools   │  │+ reasoning   │  │+ validation  │                │  │ │
│  │  │  └──────────────┘  └──────────────┘  └──────────────┘                │  │ │
│  │  │                                                                        │  │ │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                │  │ │
│  │  │  │<<agent>>     │  │<<agent>>     │  │<<agent>>     │                │  │ │
│  │  │  │bpmn-agent    │  │archimate     │  │graphpatch    │                │  │ │
│  │  │  │─────────────│  │─────────────│  │─────────────│                │  │ │
│  │  │  │+ BPMN ops   │  │+ refactor    │  │+ semantic    │                │  │ │
│  │  │  │+ validation │  │+ layers      │  │+ operations  │                │  │ │
│  │  │  └──────────────┘  └──────────────┘  └──────────────┘                │  │ │
│  │  └────────────────────────────────────────────────────────────────────────┘  │ │
│  └──────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐ │
│  │                            <<layer>>                                          │ │
│  │                          Data & State Layer                                   │ │
│  ├──────────────────────────────────────────────────────────────────────────────┤ │
│  │                                                                               │ │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                 │ │
│  │  │ <<datastore>>  │  │ <<datastore>>  │  │ <<datastore>>  │                 │ │
│  │  │  PostgreSQL    │  │     Redis      │  │     Neo4j      │                 │ │
│  │  │ ──────────────│  │ ──────────────│  │ ──────────────│                 │ │
│  │  │ • LangGraph   │  │ • Caching      │  │ • Graph data   │                 │ │
│  │  │   state       │  │ • Sessions     │  │ • Traceability │                 │ │
│  │  │ • Checkpoints │  │ • Rate limits  │  │ • Relations    │                 │ │
│  │  │ • Patches     │  │ • Resume tokens│  │                │                 │ │
│  │  └────────────────┘  └────────────────┘  └────────────────┘                 │ │
│  └──────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## Dual-Plane Architecture

```
[Canvas: CommandStack] --(SubmitChangeSet)--> [Repo/Events]
[LangGraph Agent] --(GraphPatchTool)-------> [Compiler→Repo/Events]
                                   \--(ProjectionTool)--> [Projection Envelope]
```

## UML Support Scope

> **UML-Lite** initially: **Class + Component** in diagram-js (custom renderers/rules, JSON I/O). **Sequence/XMI** are *deferred*. BPMN stays on bpmn-js. AI edits use **GraphPatch** across notations; compiler maps ops to per-notation mutations. *(The data layer's canonical graph backs all notations.)*

## Sequence Diagrams

### 1. Basic Chat Flow

```
     User           React UI        Envoy         Go Gateway      FastAPI       LangGraph
       │                │              │              │              │              │
       │  Send Message  │              │              │              │              │
       ├───────────────▶│              │              │              │              │
       │                │              │              │              │              │
       │                │  Connect     │              │              │              │
       │                │  Request     │              │              │              │
       │                ├─────────────▶│              │              │              │
       │                │              │              │              │              │
       │                │              │ Authenticate │              │              │
       │                │              ├─────────────▶│              │              │
       │                │              │              │              │              │
       │                │              │              │ Forward to   │              │
       │                │              │              │   FastAPI    │              │
       │                │              │              ├─────────────▶│              │
       │                │              │              │              │              │
       │                │              │              │              │ Select Agent │
       │                │              │              │              ├─────────────▶│
       │                │              │              │              │              │
       │                │              │              │              │  Process &   │
       │                │              │              │              │  Generate    │
       │                │              │              │              │◄─────────────┤
       │                │              │              │              │              │
       │                │              │              │ SSE Stream   │              │
       │                │              │              │◄─────────────┤              │
       │                │              │              │              │              │
       │                │              │ Connect      │              │              │
       │                │              │ Stream       │              │              │
       │                │              │◄─────────────┤              │              │
       │                │              │              │              │              │
       │                │ Update UI    │              │              │              │
       │                │◄──────────────              │              │              │
       │                │              │              │              │              │
       │  Show Response │              │              │              │              │
       │◄───────────────┤              │              │              │              │
       │                │              │              │              │              │
```

### 2. GraphPatch Review Flow

```
     User         BPMN Editor    Go Gateway    GraphPatch     LangGraph    PostgreSQL
       │               │             │            Service         │            │
       │  Request AI   │             │              │             │            │
       │  Suggestion   │             │              │             │            │
       ├──────────────▶│             │              │             │            │
       │               │             │              │             │            │
       │               │ Get AI      │              │             │            │
       │               │ Suggestions │              │             │            │
       │               ├────────────▶│              │             │            │
       │               │             │              │             │            │
       │               │             │ Generate     │             │            │
       │               │             │ GraphPatch   │             │            │
       │               │             ├─────────────▶│             │            │
       │               │             │              │             │            │
       │               │             │              │ Execute     │            │
       │               │             │              │ Agent       │            │
       │               │             │              ├────────────▶│            │
       │               │             │              │             │            │
       │               │             │              │  GraphPatch │            │
       │               │             │              │◄────────────┤            │
       │               │             │              │             │            │
       │               │             │              │ Store as    │            │
       │               │             │              │ PROPOSED    │            │
       │               │             │              ├────────────────────────▶│
       │               │             │              │             │            │
       │               │             │ Return Patch │             │            │
       │               │             │◄─────────────┤             │            │
       │               │             │              │             │            │
       │               │ Display      │              │             │            │
       │               │ Review UI    │              │             │            │
       │               │◄─────────────              │             │            │
       │               │              │              │             │            │
       │  Review &     │              │              │             │            │
       │  Select Ops   │              │              │             │            │
       │◄──────────────┤              │              │             │            │
       │               │              │              │             │            │
       │  Approve      │              │              │             │            │
       ├──────────────▶│              │              │             │            │
       │               │              │              │             │            │
       │               │ Review Patch│              │             │            │
       │               ├─────────────▶│              │             │            │
       │               │              │ Apply       │             │            │
       │               │              ├─────────────▶│             │            │
       │               │              │              │             │            │
       │               │              │              │ Compile to  │            │
       │               │              │              │ Operations  │            │
       │               │              │              ├────────────────────────▶│
       │               │              │              │             │            │
       │               │              │              │  Success    │            │
       │               │              │              │◄────────────────────────┤
       │               │              │              │             │            │
       │               │              │ Applied     │             │            │
       │               │              │◄─────────────┤             │            │
       │               │              │              │             │            │
       │               │ Update Model │              │             │            │
       │               │◄──────────────              │             │            │
       │               │              │              │             │            │
       │  Show Changes │              │              │             │            │
       │◄──────────────┤              │              │             │            │
       │               │              │              │             │            │
```

## Class Diagrams

### GraphPatch Domain Model

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            <<class>>                                    │
│                           GraphPatch                                    │
├─────────────────────────────────────────────────────────────────────────┤
│ - meta: GraphPatchMeta                                                  │
│ - ops: List<GraphOperation>                                            │
│ - status: PatchStatus                                                   │
│ - review: ReviewMetadata                                                │
├─────────────────────────────────────────────────────────────────────────┤
│ + validate(): ValidationResult                                          │
│ + compile(): List<UICommand>                                           │
│ + apply(model: Model): Model                                           │
│ + serialize(): bytes                                                    │
└─────────────────────────────────────────────────────────────────────────┘
                                │
                                │ 1
                                │
                                │ *
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          <<class>>                                      │
│                       GraphOperation                                    │
├─────────────────────────────────────────────────────────────────────────┤
│ - id: string                                                            │
│ - operation: OperationType                                              │
│ - status: OperationStatus                                               │
│ - rationale: string                                                     │
│ - preview: PreviewData                                                  │
├─────────────────────────────────────────────────────────────────────────┤
│ + execute(context: Context): Result                                     │
│ + validate(model: Model): bool                                          │
│ + generatePreview(): PreviewData                                        │
└─────────────────────────────────────────────────────────────────────────┘
                                △
                                │
                ┌───────────────┼───────────────┬──────────────┐
                │               │               │              │
┌───────────────────┐ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│    <<class>>      │ │   <<class>>    │ │   <<class>>    │ │   <<class>>    │
│    AddNodeOp      │ │  UpdateNodeOp  │ │   ConnectOp    │ │ RemoveNodeOp   │
├───────────────────┤ ├────────────────┤ ├────────────────┤ ├────────────────┤
│ - parent: Selector│ │ - target: Sel. │ │ - from: Sel.   │ │ - target: Sel. │
│ - node: Node      │ │ - updates: Map │ │ - to: Selector │ │ - cascade: bool│
├───────────────────┤ ├────────────────┤ │ - edge: Edge   │ ├────────────────┤
│ + validate()      │ │ + validate()   │ ├────────────────┤ │ + validate()   │
│ + execute()       │ │ + execute()    │ │ + validate()   │ │ + execute()    │
└───────────────────┘ └────────────────┘ │ + execute()    │ └────────────────┘
                                         └────────────────┘
```

### Agent Hierarchy

```
                        ┌─────────────────────────┐
                        │      <<abstract>>       │
                        │       BaseAgent         │
                        ├─────────────────────────┤
                        │ # tools: List<Tool>     │
                        │ # llm: LLMProvider      │
                        │ # memory: Memory        │
                        ├─────────────────────────┤
                        │ + process(): Result     │
                        │ + validate(): bool      │
                        │ # selectTools(): List   │
                        └───────────┬─────────────┘
                                    │
                ┌───────────────────┼───────────────────┐
                │                   │                   │
    ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐
    │    <<class>>      │ │    <<class>>      │ │    <<class>>      │
    │   SimpleChatAgent │ │    BPMNAgent      │ │  GraphPatchAgent  │
    ├───────────────────┤ ├───────────────────┤ ├───────────────────┤
    │ - temperature: f  │ │ - validator: Val. │ │ - compiler: Comp. │
    │ - maxTokens: int  │ │ - rules: Rules    │ │ - selector: Sel.  │
    ├───────────────────┤ ├───────────────────┤ ├───────────────────┤
    │ + threads(): Stream  │ │ + suggest(): Ops  │ │ + generate(): GP  │
    │ + complete(): Msg │ │ + fix(): Commands │ │ + validate(): bool│
    └───────────────────┘ └───────────────────┘ └───────────────────┘
```

## State Diagram

### GraphPatch Lifecycle

```
                           ┌─────────────┐
                           │   Created   │
                           └──────┬──────┘
                                  │
                                  ▼
                           ┌─────────────┐
                    ┌─────▶│   PROPOSED  │◀─────┐
                    │      └──────┬──────┘      │
                    │             │              │
                    │             ▼              │
                    │      ┌─────────────┐      │
                    │      │   Review    │      │
                    │      │   Pending   │      │
                    │      └──────┬──────┘      │
                    │             │              │
                    │      ┌──────┴──────┐      │
                    │      │             │      │
                    │      ▼             ▼      │
                    │ ┌─────────┐ ┌─────────────┐
                    │ │REJECTED │ │  APPROVED    │
                    │ └─────────┘ └──────┬──────┘
                    │                     │
                    │                     ▼
                    │            ┌─────────────┐
                    │            │   Applying  │
                    │            └──────┬──────┘
                    │                    │
                    │             ┌──────┴──────┐
                    │             │             │
                    │             ▼             ▼
                    │      ┌─────────┐   ┌─────────┐
                    └──────┤  Failed │   │ APPLIED │
                           └─────────┘   └─────────┘
```

## Deployment Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          <<node>>                                           │
│                     Kubernetes Cluster (k3d)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    <<namespace>>                                     │   │
│  │                         ameide                                      │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │                                                                      │   │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐  │   │
│  │  │   <<pod>>        │  │    <<pod>>        │  │   <<pod>>       │  │   │
│  │  │ envoy-gateway    │  │  go-gateway       │  │   fastapi       │  │   │
│  │  │ ────────────────│  │ ─────────────────│  │ ───────────────│  │   │
│  │  │ • 2 replicas    │  │ • 2 replicas      │  │ • 2 replicas   │  │   │
│  │  │ • 443:443       │  │ • 6003:6003       │  │ • 8000:8000    │  │   │
│  │  │ • 1CPU/2GB      │  │ • 1CPU/512MB      │  │ • 2CPU/4GB     │  │   │
│  │  └──────────────────┘  └──────────────────┘  └─────────────────┘  │   │
│  │                                                                      │   │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐  │   │
│  │  │   <<pod>>        │  │    <<pod>>        │  │   <<pod>>       │  │   │
│  │  │   postgresql     │  │     redis         │  │    neo4j        │  │   │
│  │  │ ────────────────│  │ ─────────────────│  │ ───────────────│  │   │
│  │  │ • 4 clusters    │  │ • 1 instance      │  │ • 1 instance    │  │   │
│  │  │ • 5432:5432     │  │ • 6379:6379       │  │ • 7687:7687     │  │   │
│  │  │ • SSD storage   │  │ • Memory cache    │  │ • Graph DB      │  │   │
│  │  └──────────────────┘  └──────────────────┘  └─────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    <<namespace>>                                     │   │
│  │                    cert-manager                                     │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │  • TLS Certificate automation                                       │   │
│  │  • Self-signed issuer for local dev                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    <<namespace>>                                     │   │
│  │                       operators                                     │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │  • CloudNativePG - PostgreSQL management                            │   │
│  │  • Strimzi - Kafka cluster operator                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Key Architecture Benefits

1. **Clean Separation**: Each layer has distinct responsibilities
2. **Protocol Safety**: Type-safe Connect protocol throughout
3. **Scalability**: Horizontal scaling at each layer
4. **Resilience**: Circuit breaking, rate limiting, retry logic
5. **Observability**: Full tracing and telemetry support
6. **Flexibility**: Multiple agent types for different use cases
7. **Human-in-the-Loop**: GraphPatch review workflows for safety

## Acceptance Criteria

- Diagrams: layered component view, streaming sequence, GraphPatch review flow, and deployment topology are present and coherent.
- Invariants: “All AI requests go through LangGraph” and per-agent tool allow-lists/budget controls are explicitly called out.
- Interfaces: Protocols labeled as Connect-Web v2/gRPC-Web where applicable; streaming and resume semantics depicted.
- Responsibilities: Each layer’s responsibilities/non-responsibilities are clear and non-overlapping.
- Cross-references: Links to POC (141) and migration plan (146) provided and accurate.

## Migration Path

```
Phase 1: POC (Current)
├── Connect protocol streaming ✅
├── Basic threads agent ✅
└── Kubernetes deployment ✅

Phase 2: Control Plane
├── Redis buffering for resume
├── Per-tenant rate limiting
└── Token budget enforcement

Phase 3: GraphPatch Integration
├── Semantic operations
├── Review workflows UI
└── Domain compilers

Phase 4: Domain Integration
├── BPMN agent with operations
├── ArchiMate refactoring
└── Code generation agents

Phase 5: Production
├── Multi-region deployment
├── Advanced monitoring
└── Agent marketplace
```

## Related Documents

- [141-ai-backend-poc.md](./141-ai-backend-poc.md) - Original POC specification
- [146-ai-backend-migration.md](./146-ai-backend-migration.md) - Migration plan
- [Connect-ES Documentation](https://connectrpc.com/docs/web/) - Protocol reference
- [LangGraph Documentation](https://github.com/langchain-ai/langgraph) - Agent framework
