# AI Backend Architecture: Connect Protocol with Multi-Layer Control Plane

## Architectural Overview

### System Architecture Principles

The AI backend implements a **4-layer architecture** with clear separation of concerns, designed for enterprise-scale AI workloads:

1. **Protocol Layer** (Connect/gRPC-Web) - Type-safe, bidirectional streaming protocol
2. **Control Plane** (Go Gateway) - Policy enforcement, routing, and orchestration
3. **Execution Layer** (Python/LangGraph) - AI model execution and agent orchestration
4. **Infrastructure Layer** (Envoy Gateway) - Edge proxy, authentication, and observability

### Key Architectural Decisions

| Decision | Rationale | Impact |
|----------|-----------|--------|
| **Connect Protocol over REST** | Type safety, streaming efficiency, native proto support | Reduced latency, better error handling |
| **Go Control Plane** | High-performance concurrency, strong typing, cloud-native | 10K+ concurrent streams support |
| **Python Execution** | AI/ML ecosystem maturity, LangGraph integration | Flexible agent development |
| **Event-Driven State** | Resilience, auditability, replay capability | Stream resume, debugging |
| **Semantic Operations** | AI-friendly abstractions over low-level commands | 70% token reduction |
| **Dual-Plane Editing** | Humans use CommandStack, AI uses GraphPatch semantic ops | Predictable merges, fewer tokens, clearer audit |
| **Projection Envelope** | Token-budgeted projection for LLM context | Stable, small prompts; deterministic grounding |

## POC Deliverables

- Running Connect-Web v2 streaming endpoint with resume tokens and tracing.
- Go control plane gateway enforcing auth, rate limits, and per-tenant budget guards.
- Python/LangGraph execution path with at least one specialized agent wired via tools.
- Event store capturing conversation events and checkpoints with replay support.
- GraphPatch semantic ops skeleton and compilation pipeline stub (validate/preview paths).
- Projection Envelope service returning token-bounded projections with deltas.

## System Architecture

### Layered Architecture Pattern

```
┌──────────────────────────────────────────────────────────────┐
│                     Client Applications                      │
│  (React/Connect-Web v2, Native Apps/Connect, CLI/gRPC)      │
└────────────────────┬─────────────────────────────────────────┘
                     │ Connect Protocol (HTTP/2)
┌────────────────────▼─────────────────────────────────────────┐
│              Infrastructure Layer (Envoy Gateway)            │
│ ┌───────────────────────────────────────────────────────────┐│
│ │ • Edge Proxy: TLS termination, HTTP/2 multiplexing       ││
│ │ • Authentication: JWT validation, OAuth2/OIDC             ││
│ │ • Rate Limiting: Token bucket, sliding window             ││
│ │ • Circuit Breaking: Failure detection, auto-recovery     ││
│ │ • Load Balancing: Consistent hashing, health checks      ││
│ │ • Observability: Distributed tracing, metrics, logs      ││
│ └───────────────────────────────────────────────────────────┘│
└────────────────────┬─────────────────────────────────────────┘
                     │ Internal gRPC/HTTP
┌────────────────────▼─────────────────────────────────────────┐
│                Control Plane Layer (Go Gateway)              │
│ ┌───────────────────────────────────────────────────────────┐│
│ │ Service Mesh Integration                                  ││
│ │ ├─ Service Discovery: Consul/K8s endpoints               ││
│ │ ├─ Traffic Management: Canary, blue-green, A/B           ││
│ │ └─ Resilience: Retries, timeouts, bulkheads             ││
│ ├───────────────────────────────────────────────────────────┤│
│ │ Policy Engine                                            ││
│ │ ├─ RBAC: Role-based access control                       ││
│ │ ├─ Budget Control: Token/cost limits per tenant         ││
│ │ ├─ Quotas: Request/concurrent stream limits             ││
│ │ └─ Content Filtering: PII detection, safety checks      ││
│ ├───────────────────────────────────────────────────────────┤│
│ │ Stream Management                                        ││
│ │ ├─ Connection Pooling: Reusable HTTP/2 streams          ││
│ │ ├─ Buffer Management: Redis-backed event buffering       ││
│ │ ├─ Resume Tokens: Checkpoint-based recovery             ││
│ │ └─ Heartbeat: Keep-alive for long streams              ││
│ ├───────────────────────────────────────────────────────────┤│
│ │ Agent Registry                                           ││
│ │ ├─ Dynamic Registration: Hot-reload agents               ││
│ │ ├─ Version Management: A/B testing, gradual rollout      ││
│ │ └─ Tenant Customization: Per-tenant agent configs       ││
│ └───────────────────────────────────────────────────────────┘│
└────────────────────┬─────────────────────────────────────────┘
                     │ Agent Protocol (HTTP/SSE)
┌────────────────────▼─────────────────────────────────────────┐
│              Execution Layer (Python/LangGraph)              │
│ ┌───────────────────────────────────────────────────────────┐│
│ │ Agent Orchestration                                       ││
│ │ ├─ Graph Execution: State machines, conditional flows    ││
│ │ ├─ Tool Integration: Sandboxed execution environment     ││
│ │ ├─ Memory Management: Context windows, RAG               ││
│ │ └─ Checkpointing: PostgreSQL state persistence          ││
│ ├───────────────────────────────────────────────────────────┤│
│ │ Model Abstraction                                        ││
│ │ ├─ Provider Routing: OpenAI, Anthropic, Llama, etc.     ││
│ │ ├─ Fallback Chains: Automatic provider failover         ││
│ │ ├─ Cost Optimization: Model selection by complexity      ││
│ │ └─ Response Caching: Semantic similarity matching       ││
│ ├───────────────────────────────────────────────────────────┤│
│ │ Domain Agents                                            ││
│ │ ├─ BPMN Agent: Process modeling assistance              ││
│ │ ├─ ArchiMate Agent: Architecture refactoring            ││
│ │ ├─ Code Agent: Generation from models                   ││
│ │ └─ Review Agent: Change validation and approval         ││
│ └───────────────────────────────────────────────────────────┘│
└────────────────────┬─────────────────────────────────────────┘
                     │ Storage Layer
┌────────────────────▼─────────────────────────────────────────┐
│                    Persistence & State                       │
│ ┌───────────────────────────────────────────────────────────┐│
│ │ • PostgreSQL: Checkpoints, audit logs, configurations    ││
│ │ • Redis: Stream buffers, rate limits, session cache      ││
│ │ • S3/MinIO: Large artifacts, model outputs               ││
│ │ • Kafka: Event streaming, async processing               ││
│ └───────────────────────────────────────────────────────────┘│
└───────────────────────────────────────────────────────────────┘
```

## Service Boundaries and Contracts

### Invariant
**All AI requests go through LangGraph.** No direct model API calls. Tools are allow-listed per agent; budgets and rate limits enforceable mid-stream.

### Service Decomposition

**Protocol Layer Note:** We standardize on **Connect-Web v2** (not gRPC-Web). Streams carry **Usage** and **Heartbeat** frames; Envoy sets `request: "0s"` and `streamIdleTimeout: "600s"`. Clients persist **resume_token** and can **resume** without gaps or dupes. Mid-stream **Budget** checks may close the stream cleanly with `Error(BUDGET_EXCEEDED)` then `Status(CANCELLED)`.

#### 1. Envoy Gateway (Infrastructure Boundary)
**Responsibilities:**
- TLS termination and certificate management
- HTTP/2 connection management
- JWT token validation
- Global rate limiting
- Request routing and load balancing
- Telemetry collection

**Not Responsible For:**
- Business logic
- Agent selection
- Token counting
- Conversation state

#### 2. Control Plane Gateway (Business Boundary)
**Responsibilities:**
- Tenant-aware routing
- Budget enforcement
- Agent lifecycle management
- Stream coordination
- Protocol translation
- Audit logging

**Not Responsible For:**
- Model execution
- Prompt engineering
- Tool execution
- Response generation

#### 3. Execution Engine (AI Boundary)
**Responsibilities:**
- Agent graph execution
- LLM API calls
- Tool invocation
- Context management
- Response streaming

**Not Responsible For:**
- Authentication
- Rate limiting
- Cost tracking
- Network resilience

### Dual-Plane Editing & Tools (Humans vs. AI)

**Dual-Plane Editing.** Humans edit via the canvas **CommandStack** with optimistic UX and undo on NACK; AI edits via **GraphPatch** semantic operations (idempotent, pre/post-validated). The backend compiles GraphPatch to graph mutations (and optional UI commands) and broadcasts a single **semantic delta** with a monotonic **version** for all subscribers.

* **Human plane:** UI → CommandStack (optimistic); SubmitChangeSet; undo on NACK
* **AI plane:** **ProjectionTool** (read) + **GraphPatchTool** (write). Backend validates, compiles to repo changes, bumps **version**, broadcasts one semantic delta
* **Invariant:** All AI calls traverse **LangGraph** with per-tenant agent/tool allow-lists, rate limits, budgets, tracing

### Contract Specifications

#### Client → Gateway Contract (Connect Protocol)
```proto
service InferenceService {
  // Streaming inference with resume support
  rpc Generate(GenerateRequest) returns (stream StreamEvent) {
    option idempotency_level = IDEMPOTENT;
    option (connect.timeout) = "600s";
  }
  
  // Batch inference for non-streaming use cases
  rpc GenerateBatch(BatchRequest) returns (BatchResponse) {
    option idempotency_level = IDEMPOTENT;
  }
}

message GenerateRequest {
  repeated Message messages = 1;
  string agent_id = 2;           // Agent routing
  Options options = 3;            // Model parameters
  string resume_token = 4;        // Stream recovery
  string correlation_id = 5;      // Idempotency
}

message StreamEvent {
  oneof event {
    TokenEvent token = 1;         // Incremental content
    ToolCallEvent tool_call = 2;  // Tool invocation
    StatusEvent status = 3;       // Stream metadata
    ErrorEvent error = 4;         // Error handling
  }
  StreamMetadata meta = 10;       // Resume tokens, usage
}
```

#### Gateway → Execution Contract (Agent Protocol)
```python
@dataclass
class AgentRequest:
    thread_id: str              # Conversation thread
    messages: List[Message]     # Input messages
    agent_config: AgentConfig   # Agent-specific config
    checkpoint_id: Optional[str] # Resume from checkpoint
    tools: List[Tool]           # Available tools
    
@dataclass
class AgentResponse:
    events: AsyncIterator[AgentEvent]  # Streaming events
    final_state: GraphState            # Terminal state
    checkpoint_id: str                 # For resume
```

## Data Architecture

### State Management Patterns

#### 1. Conversation State (Checkpointed)
```python
# PostgreSQL-backed conversation state
class ConversationState:
    thread_id: str           # Unique conversation ID
    messages: List[Message]  # Message history
    metadata: Dict           # User context, preferences
    checkpoints: List[Checkpoint]  # Resumable states
    
    def persist(self, store: PostgresStore):
        """Atomic state persistence with versioning"""
        with store.transaction() as tx:
            tx.save_checkpoint(self.to_checkpoint())
            tx.increment_version()
```

#### 2. Stream State (Ephemeral)
```go
type StreamState struct {
    StreamID      string
    Buffer        *CircularBuffer  // Recent events for replay
    LastEventID   string           // For resume
    TokensUsed    int              // Running count
    Checkpoints   []Checkpoint     // Recovery points
}

func (s *StreamState) AddEvent(event *pb.StreamEvent) {
    s.Buffer.Push(event)
    s.LastEventID = event.Id
    s.TokensUsed += event.Usage.Tokens
    
    // Checkpoint every N tokens
    if s.TokensUsed % 1000 == 0 {
        s.CreateCheckpoint()
    }
}
```

#### 3. Agent State (Versioned)
```go
type AgentRegistry struct {
    agents map[string]AgentVersion
    mu     sync.RWMutex
}

type AgentVersion struct {
    ID          string
    Version     string
    Config      AgentConfig
    Endpoints   []Endpoint
    HealthState HealthState
    Metrics     AgentMetrics
}

// Atomic agent updates with version control
func (r *AgentRegistry) UpdateAgent(id string, update AgentUpdate) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    current := r.agents[id]
    next := current.ApplyUpdate(update)
    
    // Validate compatibility
    if err := r.validateTransition(current, next); err != nil {
        return err
    }
    
    r.agents[id] = next
    r.notifyWatchers(id, next)
    return nil
}
```

### Projection Envelope

**Projection Envelope.** The standard, token-budgeted context for agents: meta (branch/baseline/version/intent), pruned nodes/edges, top scenarios, minimal delta vs. baseline, hotspots, and evidence (commits + explain of reductions). It is deterministic and sized to fit the provided budget; clients can **WatchProjection** for small deltas while modeling continues.

```json
{
  "meta": {"system":"...", "branch":"...", "as_of_version":58, "intent":"impact"},
  "nodes":[{"id":"A1","type":"Activity","name":"Validate Order","score":0.82}],
  "edges":[{"id":"E9","type":"ControlFlow","src":"A1","tgt":"G3"}],
  "scenarios":[{"id":"P1","label":"Create Order","path":["Start","A1","G3","End"],"weight":0.61}],
  "delta":{"added":[...],"removed":[...],"changed":[...]},
  "hotspots":[{"id":"APP2","why":["fan_in","churn"]}],
  "evidence":{"commits":["c1a2f","d9e0b"],"explain":["Collapsed A3,A4→M1"]}
}
```

### Event Sourcing Architecture

```go
// All state changes as events
type AIEvent interface {
    EventType() string
    AggregateID() string
    Timestamp() time.Time
    Version() int64
}

type ConversationStarted struct {
    ThreadID  string
    TenantID  string
    AgentID   string
    Timestamp time.Time
}

type TokenGenerated struct {
    ThreadID  string
    Token     string
    Position  int
    Usage     TokenUsage
}

type CheckpointCreated struct {
    ThreadID     string
    CheckpointID string
    State        []byte
    Version      int64
}

// Event store for audit and replay
type EventStore interface {
    Append(ctx context.Context, events []AIEvent) error
    Load(ctx context.Context, aggregateID string, fromVersion int64) ([]AIEvent, error)
    Subscribe(ctx context.Context, filter EventFilter) <-chan AIEvent
}
```

## Scalability Patterns

### Horizontal Scaling Strategy

#### 1. Control Plane Scaling
```yaml
# Kubernetes HPA configuration
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: inference_gateway
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: inference_gateway
  minReplicas: 3
  maxReplicas: 100
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: concurrent_streams
      target:
        type: AverageValue
        averageValue: "1000"  # 1K streams per pod
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Percent
        value: 100  # Double capacity
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # 5 min stabilization
```

#### 2. Execution Layer Scaling
```python
# LangGraph Cloud configuration
LANGGRAPH_CONFIG = {
    "deployment": {
        "min_instances": 2,
        "max_instances": 50,
        "scale_factor": {
            "cpu": 0.7,
            "memory": 0.8,
            "queue_depth": 100,
            "response_time_p99": 5000  # 5s
        },
        "instance_types": [
            {"type": "cpu", "count": 40},      # General agents
            {"type": "gpu", "count": 10},      # Complex models
        ]
    }
}
```

### Load Distribution Patterns

#### 1. Agent-Based Sharding
```go
// Consistent hashing for agent distribution
type AgentRouter struct {
    ring     *hashring.HashRing
    backends map[string]*Backend
}

func (r *AgentRouter) RouteRequest(req *Request) *Backend {
    // Shard by tenant + agent for locality
    key := fmt.Sprintf("%s:%s", req.TenantID, req.AgentID)
    backend := r.ring.GetNode(key)
    return r.backends[backend]
}
```

#### 2. Adaptive Load Balancing
```go
// P2C (Power of Two Choices) with least connections
type AdaptiveBalancer struct {
    backends []*Backend
    rng      *rand.Rand
}

func (b *AdaptiveBalancer) SelectBackend() *Backend {
    // Sample two random backends
    a := b.backends[b.rng.Intn(len(b.backends))]
    b := b.backends[b.rng.Intn(len(b.backends))]
    
    // Choose least loaded
    if a.ActiveStreams() < b.ActiveStreams() {
        return a
    }
    return b
}
```

## Resilience Patterns

### Circuit Breaker Implementation
```go
type CircuitBreaker struct {
    state           State
    failures        int
    successes       int
    lastFailureTime time.Time
    config          CircuitConfig
}

type CircuitConfig struct {
    FailureThreshold   int           // Open circuit after N failures
    SuccessThreshold   int           // Close circuit after N successes
    Timeout           time.Duration  // Half-open timeout
    BucketSize        time.Duration  // Time window for failures
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    if !cb.CanExecute() {
        return ErrCircuitOpen
    }
    
    err := fn()
    cb.RecordResult(err)
    return err
}
```

### Bulkhead Pattern
```go
// Isolate resources per tenant/agent
type BulkheadManager struct {
    bulkheads map[string]*Bulkhead
}

type Bulkhead struct {
    semaphore chan struct{}
    queue     chan Request
    workers   int
}

func (b *Bulkhead) Execute(ctx context.Context, fn func() error) error {
    select {
    case b.semaphore <- struct{}{}:
        defer func() { <-b.semaphore }()
        return fn()
    case <-ctx.Done():
        return ErrTimeout
    default:
        return ErrBulkheadFull
    }
}
```

### Retry Strategy with Backoff
```go
type RetryPolicy struct {
    MaxAttempts     int
    InitialDelay    time.Duration
    MaxDelay        time.Duration
    Multiplier      float64
    Jitter          float64
    RetryableErrors []error
}

func (p *RetryPolicy) Execute(fn func() error) error {
    delay := p.InitialDelay
    
    for attempt := 0; attempt < p.MaxAttempts; attempt++ {
        err := fn()
        if err == nil {
            return nil
        }
        
        if !p.isRetryable(err) {
            return err
        }
        
        if attempt < p.MaxAttempts-1 {
            jitter := p.addJitter(delay)
            time.Sleep(jitter)
            delay = time.Duration(float64(delay) * p.Multiplier)
            if delay > p.MaxDelay {
                delay = p.MaxDelay
            }
        }
    }
    
    return ErrMaxRetriesExceeded
}
```

## Security Architecture

### Defense in Depth

#### Layer 1: Edge Security (Envoy)
```yaml
# SecurityPolicy configuration
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: SecurityPolicy
metadata:
  name: ai-security
spec:
  targetRef:
    kind: HTTPRoute
    name: inference
  
  # TLS enforcement
  tls:
    minimumVersion: "1.3"
    cipherSuites:
    - TLS_AES_256_GCM_SHA384
    - TLS_CHACHA20_POLY1305_SHA256
  
  # Authentication
  jwt:
    providers:
    - name: keycloak
      issuer: https://auth.dev.ameide.io/realms/ameide
      audiences: ["ameide-ai"]
      remoteJWKS:
        uri: https://auth.dev.ameide.io/realms/ameide/certs
        cacheDuration: 600s
  
  # Rate limiting
  rateLimit:
    global:
      rules:
      - dimensions:
        - requestHeader:
            name: X-User-ID
        limit:
          requests: 100
          unit: Minute
  
  # WAF rules
  waf:
    rules:
    - name: sql-injection
      pattern: "(?i)(union|select|insert|update|delete|drop)"
    - name: xss-prevention
      pattern: "<script|javascript:|onerror="
```

#### Layer 2: Application Security (Control Plane)
```go
// Multi-tenant isolation
type TenantIsolation struct {
    rbac      *RBACEngine
    quotas    *QuotaManager
    budget    *BudgetController
    sanitizer *InputSanitizer
}

func (t *TenantIsolation) ValidateRequest(ctx context.Context, req *Request) error {
    // Extract tenant context
    tenant := ExtractTenant(ctx)
    
    // RBAC check
    if !t.rbac.Can(tenant, req.Action, req.Resource) {
        return ErrUnauthorized
    }
    
    // Quota check
    if !t.quotas.Allow(tenant, req.Size) {
        return ErrQuotaExceeded
    }
    
    // Budget check
    estimated := t.budget.EstimateCost(req)
    if !t.budget.CanAfford(tenant, estimated) {
        return ErrBudgetExceeded
    }
    
    // Input sanitization
    sanitized, err := t.sanitizer.Sanitize(req)
    if err != nil {
        return ErrMaliciousInput
    }
    
    *req = *sanitized
    return nil
}
```

#### Layer 3: Execution Security (Sandbox)
```python
# Tool execution sandboxing
class ToolSandbox:
    def __init__(self):
        self.container_runtime = "gvisor"  # or firecracker
        self.resource_limits = {
            "memory": "512Mi",
            "cpu": "0.5",
            "disk": "100Mi",
            "network": False,
            "timeout": 30,
        }
    
    def execute(self, tool: Tool, args: Dict) -> Any:
        with self.create_sandbox() as sandbox:
            # Copy tool code to sandbox
            sandbox.copy_in(tool.code)
            
            # Execute with limits
            result = sandbox.run(
                tool.entrypoint,
                args,
                timeout=self.resource_limits["timeout"]
            )
            
            # Validate output
            if not self.validate_output(result):
                raise SecurityError("Suspicious output detected")
            
            return result
```

### Data Protection

#### PII Detection and Masking
```go
type PIIDetector struct {
    patterns map[string]*regexp.Regexp
    ml       *MLDetector  // ML-based detection
}

func (d *PIIDetector) ScanAndMask(text string) (string, []PIIMatch) {
    matches := []PIIMatch{}
    masked := text
    
    // Pattern-based detection
    for name, pattern := range d.patterns {
        for _, match := range pattern.FindAllStringIndex(text, -1) {
            matches = append(matches, PIIMatch{
                Type:  name,
                Start: match[0],
                End:   match[1],
            })
            masked = maskRange(masked, match[0], match[1])
        }
    }
    
    // ML-based detection for complex PII
    mlMatches := d.ml.Detect(text)
    matches = append(matches, mlMatches...)
    
    return masked, matches
}
```

## Performance Architecture

### Caching Strategy

#### Multi-Level Cache
```go
// L1: In-memory cache (process-local)
// L2: Redis cache (shared)
// L3: CDN cache (edge)

type CacheManager struct {
    l1    *lru.Cache
    l2    *redis.Client
    l3    *cdn.Client
    stats *CacheStats
}

func (c *CacheManager) Get(ctx context.Context, key string) ([]byte, error) {
    // L1 check
    if val, ok := c.l1.Get(key); ok {
        c.stats.L1Hit()
        return val.([]byte), nil
    }
    
    // L2 check
    val, err := c.l2.Get(ctx, key).Bytes()
    if err == nil {
        c.stats.L2Hit()
        c.l1.Add(key, val)
        return val, nil
    }
    
    // L3 check (for public content)
    if c.isPublicContent(key) {
        val, err := c.l3.Get(ctx, key)
        if err == nil {
            c.stats.L3Hit()
            c.warmLowerLevels(ctx, key, val)
            return val, nil
        }
    }
    
    c.stats.Miss()
    return nil, ErrCacheMiss
}
```

### Connection Pooling

```go
// HTTP/2 connection multiplexing
type ConnectionPool struct {
    connections map[string]*http2.ClientConn
    maxStreams  int
    mu          sync.RWMutex
}

func (p *ConnectionPool) GetConnection(endpoint string) (*http2.ClientConn, error) {
    p.mu.RLock()
    conn, exists := p.connections[endpoint]
    p.mu.RUnlock()
    
    if exists && conn.CanTakeNewRequest() {
        return conn, nil
    }
    
    // Create new connection or wait
    return p.createOrWaitForConnection(endpoint)
}
```

### Stream Optimization

```go
// Adaptive buffering for streams
type AdaptiveBuffer struct {
    buffer      []byte
    size        int
    minSize     int
    maxSize     int
    flushTimer  *time.Timer
    metrics     *BufferMetrics
}

func (b *AdaptiveBuffer) Adapt() {
    latency := b.metrics.AverageLatency()
    throughput := b.metrics.Throughput()
    
    if latency > TargetLatency && b.size > b.minSize {
        b.size = int(float64(b.size) * 0.9)  // Reduce buffer
    } else if throughput < TargetThroughput && b.size < b.maxSize {
        b.size = int(float64(b.size) * 1.1)  // Increase buffer
    }
}
```

## Observability Architecture

### Distributed Tracing

```go
// OpenTelemetry integration
func (s *InferenceService) Generate(
    ctx context.Context,
    req *pb.GenerateRequest,
) (<-chan *pb.StreamEvent, error) {
    // Start span
    ctx, span := tracer.Start(ctx, "inference.generate",
        trace.WithAttributes(
            attribute.String("tenant_id", req.TenantId),
            attribute.String("agent_id", req.AgentId),
            attribute.String("thread_id", req.ThreadId),
            attribute.String("user_id", req.UserId),
            attribute.Int("message_count", len(req.Messages)),
        ),
    )
    defer span.End()
    
    // Propagate context through layers
    headers := propagator.Extract(ctx)
    
    // Create child spans for each layer
    gatewaySpan := tracer.StartSpan("gateway.process", trace.ChildOf(span))
    executionSpan := tracer.StartSpan("execution.run", trace.ChildOf(gatewaySpan))
    
    return s.processWithTracing(ctx, req, span)
}
```

### Metrics Architecture

```go
// Prometheus metrics
var (
    streamDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "ai_stream_duration_seconds",
            Buckets: []float64{0.1, 0.5, 1, 2, 5, 10, 30, 60},
        },
        []string{"agent", "tenant", "status"},
    )
    
    tokenUsage = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "ai_tokens_total",
        },
        []string{"agent", "tenant", "model"},
    )
    
    concurrentStreams = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "ai_concurrent_streams",
        },
        []string{"agent"},
    )
)
```

### Structured Logging

```go
// Contextual logging with correlation
type Logger struct {
    base *zap.Logger
}

func (l *Logger) ForStream(streamID string) *Logger {
    return &Logger{
        base: l.base.With(
            zap.String("stream_id", streamID),
            zap.String("correlation_id", GetCorrelationID()),
            zap.String("trace_id", GetTraceID()),
        ),
    }
}

func (l *Logger) LogStreamEvent(event *StreamEvent) {
    l.base.Info("stream_event",
        zap.String("type", event.Type),
        zap.Int("tokens", event.Tokens),
        zap.Duration("latency", event.Latency),
        zap.Any("metadata", event.Metadata),
    )
}
```

## Integration Patterns

### GraphPatch: Semantic Operations for AI

The GraphPatch pattern provides a higher-level abstraction for AI agents to modify domain models, replacing brittle command sequences with semantic operations.

#### Architecture

```proto
// Semantic operation specification
message GraphPatch {
  GraphPatchMeta meta = 1;        // System, version, idempotency
  repeated GraphOperation ops = 2; // Semantic operations
  PatchStatus status = 3;          // Lifecycle state
  ReviewMetadata review = 4;       // Human approval tracking
}

message GraphOperation {
  string id = 1;                   // For selective approval
  oneof operation {
    AddNodeOp add_node = 2;       // Semantic node creation
    UpdateNodeOp update_node = 3;  // Property updates
    ConnectOp connect = 4;         // Relationship creation
    ReplaceSubgraphOp replace = 5; // Complex transformations
  }
  string rationale = 9;            // AI explanation
  PreviewData preview = 10;        // Visual diff
}
```

#### Compilation Pipeline

```go
// GraphPatch compiler architecture
type CompilationPipeline struct {
    stages []CompilationStage
}

type CompilationStage interface {
    Process(context.Context, *GraphPatch) (*GraphPatch, error)
}

// Pipeline stages
var defaultPipeline = []CompilationStage{
    &ValidationStage{},      // Domain rule validation
    &ResolutionStage{},      // Selector → ID resolution
    &OptimizationStage{},    // Operation merging/reordering
    &CompilationStage{},     // Generate mutations
    &PreviewStage{},         // Generate visual preview
}

func (p *CompilationPipeline) Compile(
    ctx context.Context, 
    patch *GraphPatch,
) (*CompiledPatch, error) {
    current := patch
    
    for _, stage := range p.stages {
        next, err := stage.Process(ctx, current)
        if err != nil {
            return nil, fmt.Errorf("%s: %w", stage.Name(), err)
        }
        current = next
    }
    
    return &CompiledPatch{
        Patch:     current,
        Mutations: extractMutations(current),
        Commands:  extractCommands(current),
    }, nil
}
```

### Platform Integration Pattern

Each domain service exposes AI capabilities through its own proto contract while delegating to shared infrastructure:

```go
// Domain service integration
type DomainAIAdapter struct {
    aiPlatform AIExecutor
    compiler   DomainCompiler
    validator  DomainValidator
}

func (a *DomainAIAdapter) ProcessAIRequest(
    ctx context.Context,
    domainReq DomainRequest,
) (DomainResponse, error) {
    // Transform to platform request
    aiReq := &AIRequest{
        AgentID:  a.selectAgent(domainReq),
        Context:  a.marshalContext(domainReq),
        TenantID: extractTenant(ctx),
    }
    
    // Execute through platform
    result := a.aiPlatform.Execute(ctx, aiReq)
    
    // Transform to domain response
    return a.transformResponse(result)
}
```

## Deployment Architecture

### Container Orchestration

```yaml
# Kubernetes deployment strategy
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inference_gateway
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero-downtime deployment
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - topologyKey: kubernetes.io/hostname
      containers:
      - name: gateway
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        readinessProbe:
          httpGet:
            path: /ready
            port: health
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: health
          initialDelaySeconds: 30
          periodSeconds: 10
```

### Service Mesh Configuration

```yaml
# Istio service mesh integration
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: inference
spec:
  hosts:
  - inference.dev.ameide.io
  http:
  - match:
    - headers:
        x-experiment:
          exact: canary
    route:
    - destination:
        host: inference-canary
        port:
          number: 8000
      weight: 10  # 10% canary traffic
  - route:
    - destination:
        host: inference
        port:
          number: 8000
      weight: 90  # 90% stable traffic
```

## Architecture Decision Records

### ADR-001: Connect Protocol over REST
**Status**: Accepted  
**Context**: Need efficient, type-safe streaming protocol  
**Decision**: Use Connect protocol (Connect-RPC)  
**Consequences**: 
- ✅ Native proto support with type safety
- ✅ Efficient binary encoding
- ✅ Built-in streaming
- ❌ Less familiar than REST
- ❌ Requires Connect-aware clients

### ADR-002: Go Control Plane + Python Execution
**Status**: Accepted  
**Context**: Need high-performance control with flexible AI execution  
**Decision**: Go for control plane, Python for AI execution  
**Consequences**:
- ✅ Go provides excellent concurrency for stream management
- ✅ Python has mature AI/ML ecosystem
- ✅ Clear separation of concerns
- ❌ Cross-language complexity
- ❌ Additional serialization overhead

### ADR-003: GraphPatch for AI Operations
**Status**: Accepted  
**Context**: AI agents need robust way to modify domain models  
**Decision**: Semantic GraphPatch operations instead of UI commands  
**Consequences**:
- ✅ 70% reduction in token usage
- ✅ Idempotent operations
- ✅ Human review capability
- ✅ Becomes the **only** AI write interface; UI commands are human-only
- ❌ Additional compilation layer
- ❌ Migration effort for existing agents

### ADR-004: Event-Driven State Management
**Status**: Accepted  
**Context**: Need resilient, auditable state management  
**Decision**: Event sourcing for all state changes  
**Consequences**:
- ✅ Complete audit trail
- ✅ Stream resume capability
- ✅ Time-travel debugging
- ❌ Additional storage requirements
- ❌ Eventually consistent reads

### ADR-005: Dual-Plane Editing
**Status**: Accepted  
**Context**: Need clear separation between human and AI editing models  
**Decision**: Split human vs AI surfaces; enforce via gateways/tool allow-lists  
**Consequences**:
- ✅ Simpler reasoning about edits
- ✅ Cleaner audit trail
- ✅ Predictable merge semantics
- ❌ Compiler layer required
- ❌ Two distinct editing paths to maintain

### ADR-006: Projection Envelope
**Status**: Accepted  
**Context**: Need standardized, token-efficient LLM context  
**Decision**: Standard token-budgeted projection format for all agents  
**Consequences**:
- ✅ Deterministic context generation
- ✅ Token-bounded prompts
- ✅ Cross-notation consistency
- ❌ Additional projection service
- ❌ Budget-fitting complexity

## Success Metrics

### Technical Metrics
- **Latency**: P50 < 200ms, P99 < 2s for first token
- **Throughput**: 10K concurrent streams per cluster
- **Availability**: 99.95% uptime (4.38 hours/year)
- **Token Efficiency**: 70% reduction with GraphPatch
- **Error Rate**: < 0.1% for successful requests

### Business Metrics
- **Agent Utilization**: 80% requests use specialized agents
- **Cost Reduction**: 40% via intelligent model routing
- **Developer Velocity**: 50% faster agent development
- **User Satisfaction**: > 4.5/5 rating for AI assistance

## Acceptance Criteria

- End-to-end streaming works via Connect-Web v2 with first-token latency within targets and resume after simulated disconnects.
- Control plane enforces per-tenant budgets and rate limits with observable mid-stream cutoff behavior.
- Event sourcing persists conversation, token usage, and checkpoint events; replay restores state deterministically.
- GraphPatch requests are validated and produce a preview/diagnostics; compiler stub emits structured mutations.
- Projection Envelope API fits within token budgets and supports incremental deltas; watch endpoints stream small updates.

## Migration Strategy

### Phase 1: Foundation (Weeks 1-2)
- Deploy core infrastructure
- Implement basic threads agent
- Validate streaming pipeline

### Phase 2: Control Plane (Weeks 3-4)
- Budget enforcement
- Stream resume
- Rate limiting
- Observability

### Phase 3: Domain Integration (Weeks 5-6)
- BPMN agent with GraphPatch
- ArchiMate agent
- Code generation agent

### Phase 4: Production Hardening (Weeks 7-8)
- Performance optimization
- Security hardening
- Chaos testing
- Documentation

## Risk Assessment

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| **LLM Provider Outage** | Medium | High | Multi-provider fallback chains |
| **Token Budget Overrun** | High | Medium | Hard limits with graceful degradation |
| **Streaming Interruption** | Medium | Medium | Checkpoint-based resume |
| **Security Breach** | Low | Critical | Defense in depth, sandboxing |
| **Performance Degradation** | Medium | High | Auto-scaling, circuit breakers |

## Conclusion

This architecture provides a robust, scalable foundation for AI capabilities across the AMEIDE platform. The separation of concerns between infrastructure, control, and execution layers enables independent scaling and evolution while maintaining system coherence. The GraphPatch pattern significantly improves AI reliability and token efficiency, while the event-driven architecture ensures resilience and auditability.

The investment in this architecture extends beyond threads, establishing patterns and infrastructure that will accelerate AI adoption across all platform services while maintaining enterprise-grade security, performance, and reliability requirements.
