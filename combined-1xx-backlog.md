# Combined Backlog (1xx Series)

Generated on: 2025-12-05 15:47:49 UTC

---

## 141-ai-backend-poc.md

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

---

## 142-ai-backend-uml-architecture.md

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

---

## 142-tilt-helm-north-star-v2.md

> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# Tilt + Helm North Star v2

**STATUS**: Partially Implemented (Infrastructure/Application Separation Complete)
**Last Updated**: 2025-08-29

This proposal formalizes a clean separation of concerns for local development:

## Current State (Implemented)

### Infrastructure/Application Separation ✅
- **Infrastructure**: Deployed exclusively via Helmfile (`helmfile -e local sync`)
- **Applications**: Deployed via Tilt for fast development iteration
- **No Conflicts**: Tilt no longer deploys infrastructure, preventing resource ownership conflicts

### What Was Fixed
- Removed all infrastructure `helm_resource()` calls from Tiltfile
- Enhanced infrastructure health checks to verify all components
- Fixed certificate deletion issue caused by dual Helm ownership
- Added `ops-deploy-infrastructure` recovery tool for missing infrastructure

## Next Phase (To Be Implemented)

### Digest-Based Deployments
- Tilt handles file watching, rebuilding images, and publishing them to the local registry
- Helm remains authoritative for all Kubernetes resources and image selection
- Deployments use immutable image digests, never ambiguous floating tags like `:dev`

## Goals

- Deterministic, reproducible deploys during local dev.
- Fast inner loop (Tilt) without coupling to Kubernetes YAML templating.
- Helm as the single source of truth for manifests and values (even in local).
- Remove ambiguity and drift caused by multiple `:dev` tags.

## Summary

- Tilt builds an image and pushes it to the local registry using a content-based tag (`$EXPECTED_REF`).
- Tilt resolves the pushed image digest and writes a small Helm values file per service:
  - `image.graph: <registry>/<service>`
  - `image.digest: sha256:...`
- Tilt calls `helm_resource()` with base values + environment values + the Tilt‑generated “tilt-values” file.
- Helm templates the Deployment using the digest (digest-first), then applies it to the cluster.

## Prerequisites

### Current Setup (Working)
- k3d cluster with integrated registry exposed via Envoy at `docker.io/ameide`
- Infrastructure deployed via Helmfile before starting Tilt
- Separation of concerns: Helmfile owns infrastructure, Tilt owns applications

### For Digest Implementation (Future)
- Local registry reachable by both host and cluster at `docker.io/ameide`
  - Containerd mirror + DNS wiring now ships automatically via the `registry-mirror` Helm component (see backlog 337)
- Tilt and Docker available on the dev machine
- `crane` (gcr.io/go-containerregistry/crane) for resolving digests

## Helm Chart Requirements

Deployments must prefer digest when provided:

```yaml
{{- $repo := .Values.image.graph -}}
{{- if .Values.image.digest }}
image: "{{ $repo }}@{{ .Values.image.digest }}"
{{- else if .Values.image.tag }}
image: "{{ $repo }}:{{ .Values.image.tag }}"
{{- else }}
image: "{{ $repo }}"
{{- end }}
```

- Inference chart already supports digest-first:
  - `infra/kubernetes/charts/platform/inference/templates/deployment.yaml:1`
- Update any tag‑only charts (e.g., inference_gateway) to digest-first using the snippet above:
  - `infra/kubernetes/charts/platform/inference_gateway/templates/deployment.yaml:1`

## Tiltfile Pattern (per service)

### Build + Publish + Write Digest Values

**IMPORTANT**: Use `local_resource` instead of `custom_build` for reliable builds. The `custom_build` approach may not trigger if nothing directly references the image name.

```python
# Build inference image and generate digest file
local_resource(
    'inference-build',
    cmd='bash -c "set -euo pipefail; \
    REGISTRY=docker.io/ameide; \
    REF=\\"\\$REGISTRY/inference:tilt-\\$(date +%s)\\"; \
    docker build -f services/inference/Dockerfile -t \\"\\$REF\\" .; \
    docker push \\"\\$REF\\"; \
    DIGEST=\\$(crane digest --insecure \\"\\$REF\\"); \
    mkdir -p infra/kubernetes/environments/local/tilt-values; \
    {
      echo \\"image:\\"; \
      echo \\"  graph: \\$REGISTRY/inference\\"; \
      echo \\"  digest: \\$DIGEST\\"; \
    } > infra/kubernetes/environments/local/tilt-values/inference.values.yaml; \
    echo \\"Built and pushed \\$REF with digest \\$DIGEST\\""',
    deps=['services/inference', 'packages/ameide_core_inference_agents'],
    labels=['build.inference']
)
```

Notes:
- `local_resource` ensures the build always executes when dependencies change
- **Must use `bash -c`** instead of `sh` for `pipefail` support
- Use escaped quotes and variables for proper shell execution
- Use echo commands instead of heredocs for simpler escaping
- Generate unique tags with timestamp to avoid conflicts
- Echo status for debugging

### Deploy with Helm + Include Tilt Values

```python
# Sentinel resource that signals digest file is ready
local_resource(
    'inference-digest-ready',
    cmd='test -f infra/kubernetes/environments/local/tilt-values/inference.values.yaml && echo "Digest ready" || echo "Waiting for digest"',
    deps=['infra/kubernetes/environments/local/tilt-values/inference.values.yaml'],
    resource_deps=[]  # CRITICAL: No deps to avoid dependency cycles
)

# Deploy using helm_resource - NO image linking, only digest file
helm_resource(
    'inference',
    'infra/kubernetes/charts/platform/inference',
    namespace='ameide',
    flags=[
        '--values=infra/kubernetes/values/platform/inference.yaml',
        '--values=infra/kubernetes/environments/local/platform/inference.yaml',
        '--values=infra/kubernetes/environments/local/tilt-values/inference.values.yaml',
        '--wait',
        '--atomic',
    ],
    deps=[
        'infra/kubernetes/charts/platform/inference',
        'infra/kubernetes/values/platform/inference.yaml',
        'infra/kubernetes/environments/local/platform/inference.yaml',
        'infra/kubernetes/environments/local/tilt-values/inference.values.yaml',
    ],
    image_keys=[],  # CRITICAL: Disable automatic image injection
    labels=['inference'],
    resource_deps=['infra-health-check', 'inference-digest-ready']
)
```

**Critical settings:**
- `image_keys=[]` - Prevents helm_resource from injecting `--set image.*` flags
- NO `image_deps` - Avoid linking builds to deploys which triggers name mangling
- Sentinel resource without `resource_deps` to avoid circular dependencies
- `--wait --atomic` flags for reliable deployments

### For Services Without Digest Support

- Preferred: patch the chart to support digest-first (see snippet above).
- Interim: write `image.tag` in the Tilt-generated file instead of `image.digest`.

```python
# Interim for a tag-only chart (not recommended long-term)
custom_build(
    'docker.io/ameide/inference_gateway',
'''docker build -f services/inference_gateway/Dockerfile.release -t $EXPECTED_REF services/inference_gateway && \
    docker push $EXPECTED_REF && \
    TAG=${EXPECTED_REF##*:} && \
    mkdir -p infra/kubernetes/environments/local/tilt-values && \
    cat > infra/kubernetes/environments/local/tilt-values/inference_gateway.values.yaml <<EOF
image:
  graph: docker.io/ameide/inference_gateway
  tag: $TAG
EOF
    ''',
    deps=['services/inference_gateway'],
    skips_local_docker=False,
)
```

## Directory Layout

- Base values: `infra/kubernetes/values/platform/<service>.yaml`
- Env overrides: `infra/kubernetes/environments/local/platform/<service>.yaml`
- Tilt‑generated overrides: `infra/kubernetes/environments/local/tilt-values/<service>.values.yaml`

## Migration Steps

- Add digest-first support to any tag‑only charts (e.g., inference‑gateway) using the template above.
- Update Tiltfile for each app service:
  - Add custom_build with build → push → digest → write tilt-values.
  - Update `helm_resource` to include the tilt-values file and remove any `--set=image.graph=…` overrides (tilt-values now sets repo+digest).
- Keep infrastructure Helmfile flows as-is for non-local environments.

## Verification

### Current Setup Verification
- Check infrastructure is deployed:
  ```bash
  helm list -n ameide | grep -E "(gateway|cert-manager|redis|postgres)"
  kubectl get gateway ameide -n ameide
  kubectl get secret ameide-wildcard-tls -n ameide
  ```
- Verify HTTPS access:
  ```bash
  curl -k https://www.dev.ameide.io/
  curl -k https://platform.dev.ameide.io/
  ```
- Check Tilt is not deploying infrastructure:
  ```bash
  grep "helm_resource" Tiltfile | grep -v "^#" | wc -l  # Should be 0 (except load statement)
  ```

### Future Digest Verification
- Confirm DNS + registry wiring before builds:
  ```bash
  scripts/infra/helm-ops.sh preflight dns docker.io
  ```
- After a build, confirm Tilt generated values:
  - `ls -l infra/kubernetes/environments/local/tilt-values/`
  - `cat infra/kubernetes/environments/local/tilt-values/inference.values.yaml`
- Confirm Deployment uses digest:
  - `kubectl -n ameide get deploy inference -o jsonpath='{.spec.template.spec.containers[0].image}'`
  - Expect: `docker.io/ameide/inference@sha256:…`

## Configuration

### Timeout Settings

```python
# At the top of your Tiltfile
update_settings(
    k8s_upsert_timeout_secs=600,     # 10 minutes for complex Helm deployments
    suppress_unused_image_warnings='*'  # optional, keeps UI quiet for local-only builds
)
```

The default 30-second timeout is insufficient when using `helm --wait` for complex deployments. We found 600 seconds (10 minutes) works reliably for all services.

## Current Implementation Details

### Infrastructure Components (Helmfile-managed)
```yaml
# All deployed via: helmfile -e local sync
- PostgreSQL clusters (CloudNativePG operator)
- Kafka cluster (Strimzi operator)
- Redis, MinIO, Neo4j, Keycloak, Temporal
- Prometheus, Grafana, Loki, Tempo (observability)
- Envoy Gateway and Gateway configuration
- Cert-Manager with TLS certificates
```

### Application Services (Tilt-managed)
```python
# Pattern used in current Tiltfile:
docker_build(
    'docker.io/ameide/service-name',
    '.',
    dockerfile='services/service_name/Dockerfile',
)

yaml = helm(
    'infra/kubernetes/charts/platform/service-name',
    name='service-name',
    namespace='ameide',
    values=['...'],
    set=['image.graph=docker.io/ameide/service-name'],
)
k8s_yaml(yaml, allow_duplicates=True)

k8s_resource('service-name',
    labels=['app.category'],
    resource_deps=['infra-health-check']
)
```

## Troubleshooting

- HTTP response to HTTPS client on registry:
  - Envoy exposes `docker.io/ameide`. If using HTTPS locally, keep the listener in sync; otherwise rely on the Helm-managed containerd mirror for HTTP.
- ErrImagePull (connection refused):
  - Registry not reachable from cluster network; ensure it's started and routed.
- Helm error "Kubernetes cluster unreachable": 
  - Check kubeconfig/context Tilt/Helm are using.
- No tilt-values file:
  - Ensure the local_resource ran and the path matches the helm_resource flag.
- Helm deployment timeout:
  - Increase `k8s_upsert_timeout_secs` in Tiltfile (default 30s is often too short)
- Dependency cycle detected:
  - Check sentinel resources don't have `resource_deps` pointing back to helm_resource
- Build not triggering:
  - Use `local_resource` instead of `custom_build` for guaranteed execution
- Image name mangling (docker.io_5000_):
  - Ensure `image_keys=[]` is set on helm_resource to disable injection
- Shell syntax errors ("set: Illegal option -o pipefail"):
  - Use `bash -c` instead of `sh` as pipefail is bash-specific
- Bazel proto path errors ("glob pattern 'proto/**/*.proto' didn't match"):
  - Update BUILD.bazel to use `src/proto/**/*.proto` if protos are in src directory

## Practical Notes From Implementation

- Avoid Tilt image rewriting when using digests
  - Set `image_keys=[]` on every `helm_resource` that consumes tilt‑values (this explicitly disables the extension’s image injection and prevents `--set image.*` from being added to the Helm command).
  - Do not use `image_deps` with `helm_resource` in this pattern (it links builds to deploys and triggers image injection/mangling).
  - Using automatic image linking can rewrite names (e.g., `docker.io_5000_service`) and override your digest values. Keep Helm in control via tilt‑values.
  - Use `resource_deps` (or a dummy `local_resource` sentinel that depends on the generated values file) to order Helm after the build without exposing image maps.

### Key Implementation Lessons

1. **Use `local_resource` not `custom_build`**:
   - `custom_build` may not trigger if the image name isn't directly referenced
   - `local_resource` guarantees execution when dependencies change
   - Clean names (like 'inference') work better than suffixed names ('inference-img')

2. **Prevent helm_resource image injection completely**:
   - Set `image_keys=[]` on every helm_resource
   - Never use `image_deps` - it triggers name mangling
   - The extension is aggressive about injecting `--set image.*` flags

3. **Avoid dependency cycles**:
   - Sentinel resources must have `resource_deps=[]`
   - Use sentinels to order Helm after builds without creating cycles

4. **Registry routing**:
   - Envoy now fronts the k3d registry at `docker.io/ameide` for both host pushes and in-cluster pulls.
   - The Helm-managed `registry-mirror` DaemonSet keeps containerd mirrors + trust settings in sync automatically.
   - Always populate tilt-values with the Envoy host (`docker.io/ameide/...`) to avoid drift.

### Complete Working Example

```python
# 1. Build and push with local_resource
local_resource(
    'inference-build',
    cmd='bash -c "set -euo pipefail; \
    REGISTRY=docker.io/ameide; \
    REF=\\"\\$REGISTRY/inference:tilt-\\$(date +%s)\\"; \
    docker build -f services/inference/Dockerfile -t \\"\\$REF\\" .; \
    docker push \\"\\$REF\\"; \
    DIGEST=\\$(crane digest --insecure \\"\\$REF\\"); \
    mkdir -p infra/kubernetes/environments/local/tilt-values; \
    {
      echo \\"image:\\"; \
      echo \\"  graph: \\$REGISTRY/inference\\"; \
      echo \\"  digest: \\$DIGEST\\"; \
    } > infra/kubernetes/environments/local/tilt-values/inference.values.yaml; \
    echo \\"Built and pushed \\$REF with digest \\$DIGEST\\""',
    deps=['services/inference', 'packages/ameide_core_inference_agents'],
    labels=['build.inference']
)

# 2. Sentinel to signal digest readiness
local_resource(
    'inference-digest-ready',
    cmd='test -f infra/kubernetes/environments/local/tilt-values/inference.values.yaml && echo "Ready"',
    deps=['infra/kubernetes/environments/local/tilt-values/inference.values.yaml'],
    resource_deps=[]  # No deps to avoid cycles
)

# 3. Deploy with Helm - clean, no image injection
helm_resource(
    'inference',
    'infra/kubernetes/charts/platform/inference',
    namespace='ameide',
    flags=[
        '--values=infra/kubernetes/values/platform/inference.yaml',
        '--values=infra/kubernetes/environments/local/platform/inference.yaml',
        '--values=infra/kubernetes/environments/local/tilt-values/inference.values.yaml',
        '--wait', '--atomic',
    ],
    deps=[
        'infra/kubernetes/charts/platform/inference',
        'infra/kubernetes/environments/local/tilt-values/inference.values.yaml',
    ],
    image_keys=[],  # Critical: disable image injection
    resource_deps=['infra-health-check', 'inference-digest-ready'],
)
```

- Registry routing (local)
  - Host and cluster traffic use `docker.io/ameide`
  - The `registry-mirror` DaemonSet configures containerd for HTTP automatically; no manual mirror script required
  - Ensure Envoy/TLS listeners stay aligned if you opt into HTTPS locally

- Generated files hygiene
  - Add `infra/kubernetes/environments/local/tilt-values/` to `.gitignore` so per‑developer digests don’t get committed.

## Common Pitfalls to Avoid

1. **DON'T use custom_build with clean names** - It won't trigger unless something references the image
2. **DON'T add `-img` suffixes** - Keep names clean and simple
3. **DON'T forget `image_keys=[]`** - The helm_resource extension is aggressive about injection
4. **DON'T create dependency cycles** - Sentinels must not depend on the resources they guard
5. **DON'T bypass the Envoy registry host** - Always use `docker.io/ameide` in tilt-values
6. **DON'T skip the timeout increase** - Helm --wait needs more than 30 seconds

## Migration Checklist

### Phase 1: Infrastructure Separation (Completed 2025-08-29)
- [x] Removed all infrastructure `helm_resource()` from Tiltfile
- [x] Infrastructure deployed exclusively via Helmfile
- [x] Enhanced health checks for all infrastructure components
- [x] Fixed certificate ownership conflicts
- [x] Application services use `docker_build()` + `helm()` + `k8s_yaml()`

### Phase 2: Digest-Based Deployments (To Do)
- [ ] Charts render digest‑first (inference, inference‑gateway, others as needed)
- [ ] Tiltfile per‑service using `local_resource`: build → push → `crane digest` → write tilt‑values
- [ ] Helm consumes: base values + env values + tilt‑values
- [ ] Disabled image injection: `image_keys=[]` on helm_resource; no `image_deps`
- [ ] Use sentinel `local_resource` for ordering without creating cycles
- [ ] `.gitignore` excludes `infra/kubernetes/environments/local/tilt-values/`
- [x] Increased timeout: `update_settings(k8s_upsert_timeout_secs=600)`
- [ ] Verified Deployments reference `repo@sha256:…` with clean names (no prefix mangling)

## FAQ

- Can we still use live_update for frontends?
  - Yes. This pattern targets backend images; frontends can keep `docker_build` + live_update for hot reload.

- What about CI/CD?
  - CI builds and pushes, resolves digests, and Helmfile deploys with digest overrides — the same contract as local.

- Do we need to keep `:dev` tags?
  - No. They are not required when using digest‑based deploys. Keep them only if your tooling still references tags.

## CI/CD Alignment

- In CI, build and push images, then resolve digests:
  - `docker buildx build --push -t $IMAGE:$GIT_SHA …`
  - `DIGEST=$(crane digest $IMAGE:$GIT_SHA)`
- Helmfile deploys with `image.graph` + `image.digest` (no tags), enabling rollback by promoting previous digests.

## Decision Log

### 2025-08-29: Infrastructure/Application Separation
- **Problem**: Tilt was deploying infrastructure via `helm_resource()`, causing ownership conflicts
- **Example**: Gateway deployment by Tilt deleted cert-manager certificates created by Helmfile
- **Solution**: Removed all infrastructure deployments from Tilt, kept only application services
- **Result**: Clean separation - Helmfile owns infrastructure, Tilt owns applications

### Future: Digest-Based Deployments
- Remove floating tags (`:dev`) in favor of digests to avoid ambiguity
- Keep Helm authoritative for all cluster changes; Tilt only prepares immutable inputs
- Adopt a per-service tilt-values layer to keep local dev changes encapsulated

---

## 143-ai-backend-archimate-diagram.md

# AI Backend Architecture - ArchiMate Enterprise View

## Executive Summary

ArchiMate enterprise architecture diagrams for the AI backend POC, focusing on Business, Information, and Technology layers to show how AI capabilities support business processes through structured information flows and technology infrastructure.

## ArchiMate Layer Architecture

### Complete Three-Layer View

```
╔═══════════════════════════════════════════════════════════════════════════════════════════╗
║                                    BUSINESS LAYER                                         ║
╠═══════════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                           ║
║  ┌─────────────┐        ┌─────────────────────┐        ┌──────────────────┐            ║
║  │  <<Actor>>  │        │   <<Business        │        │  <<Business      │            ║
║  │             │═══════▶│    Process>>        │═══════▶│   Service>>      │            ║
║  │  End User   │        │ Model Enhancement   │        │ AI Assistance    │            ║
║  └─────────────┘        └─────────────────────┘        └──────────────────┘            ║
║         │                       │                              │                         ║
║         │                       │                              │                         ║
║         ▼                       ▼                              ▼                         ║
║  ┌─────────────┐        ┌─────────────────────┐        ┌──────────────────┐            ║
║  │  <<Actor>>  │        │   <<Business        │        │  <<Business      │            ║
║  │  Business   │═══════▶│    Process>>        │═══════▶│   Service>>      │            ║
║  │  Analyst    │        │ Review & Approve    │        │ Change Review    │            ║
║  └─────────────┘        └─────────────────────┘        └──────────────────┘            ║
║         │                       │                              │                         ║
║         │                       └──────────────────────────────┘                         ║
║         │                                   │                                            ║
║         │                    ┌──────────────▼───────────────┐                           ║
║         │                    │    <<Business Function>>     │                           ║
║         │                    │   AI-Driven Modeling         │                           ║
║         │                    └──────────────────────────────┘                           ║
║         │                                   │                                            ║
║  ┌──────▼────────────────────────────────────────────────────────────────┐             ║
║  │                     <<Business Collaboration>>                         │             ║
║  │                  Human-AI Collaborative Design                         │             ║
║  └─────────────────────────────────────────────────────────────────────────┘             ║
║                                                                                           ║
╠═══════════════════════════════════════════════════════════════════════════════════════════╣
║                                   INFORMATION LAYER                                       ║
╠═══════════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                           ║
║  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐                   ║
║  │ <<Data Object>> │     │ <<Data Object>> │     │ <<Data Object>> │                   ║
║  │                 │────▶│                 │────▶│                 │                   ║
║  │ Conversation    │     │   GraphPatch    │     │ Domain Model    │                   ║
║  │    Context      │     │   Operations    │     │   Updates       │                   ║
║  └─────────────────┘     └─────────────────┘     └─────────────────┘                   ║
║          │                        │                        │                             ║
║          │                        │                        │                             ║
║          ▼                        ▼                        ▼                             ║
║  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐                   ║
║  │<<Representation>│     │<<Representation>│     │<<Representation>│                   ║
║  │                 │     │                 │     │                 │                   ║
║  │ Message Stream  │     │ Semantic Ops    │     │  BPMN/ArchiMate │                   ║
║  │   (Connect)     │     │    (Proto)      │     │     (XML)       │                   ║
║  └─────────────────┘     └─────────────────┘     └─────────────────┘                   ║
║                                                                                           ║
║  ┌───────────────────────────────────────────────────────────────────┐                  ║
║  │                    <<Data Flow Architecture>>                      │                  ║
║  │  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐   │                  ║
║  │  │  Input   │───▶│  Process │───▶│ Validate │───▶│  Output  │   │                  ║
║  │  │  Stream  │    │  Stream  │    │  Stream  │    │  Stream  │   │                  ║
║  │  └──────────┘    └──────────┘    └──────────┘    └──────────┘   │                  ║
║  └───────────────────────────────────────────────────────────────────┘                  ║
║                                                                                           ║
╠═══════════════════════════════════════════════════════════════════════════════════════════╣
║                                   TECHNOLOGY LAYER                                        ║
╠═══════════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                           ║
║  ┌───────────────────────────────────────────────────────────────────────────────────┐  ║
║  │                        <<Technology Collaboration>>                                │  ║
║  │                         Kubernetes Container Platform                              │  ║
║  ├───────────────────────────────────────────────────────────────────────────────────┤  ║
║  │                                                                                     │  ║
║  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                   │  ║
║  │  │   <<Node>>      │  │   <<Node>>      │  │   <<Node>>      │                   │  ║
║  │  │                 │  │                 │  │                 │                   │  ║
║  │  │  Load Balancer  │══│  API Gateway    │══│ Service Mesh    │                   │  ║
║  │  │  (Envoy Proxy)  │  │  (Connect-Go)   │  │   (Istio)       │                   │  ║
║  │  └─────────────────┘  └─────────────────┘  └─────────────────┘                   │  ║
║  │           ║                    ║                    ║                              │  ║
║  │           ╚════════════════════╬════════════════════╝                              │  ║
║  │                                ║                                                    │  ║
║  │  ┌─────────────────────────────╬──────────────────────────────────────────────┐   │  ║
║  │  │          <<System Software>>║                                               │   │  ║
║  │  │              Container Runtime (containerd)                                 │   │  ║
║  │  ├──────────────────────────────────────────────────────────────────────────┤   │  ║
║  │  │                                                                            │   │  ║
║  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                   │   │  ║
║  │  │  │  <<Device>>  │  │  <<Device>>  │  │  <<Device>>  │                   │   │  ║
║  │  │  │              │  │              │  │              │                   │   │  ║
║  │  │  │  Inference   │  │  LangGraph   │  │   FastAPI    │                   │   │  ║
║  │  │  │   Gateway    │  │   Engine     │  │   Service    │                   │   │  ║
║  │  │  │  (Go/6003)   │  │  (Python)    │  │  (8000)      │                   │   │  ║
║  │  │  └──────────────┘  └──────────────┘  └──────────────┘                   │   │  ║
║  │  │         │                  │                 │                            │   │  ║
║  │  │         └──────────────────┴─────────────────┘                            │   │  ║
║  │  │                            │                                               │   │  ║
║  │  │  ┌──────────────────────────────────────────────────────────────────┐    │   │  ║
║  │  │  │              <<Infrastructure Service>>                           │    │   │  ║
║  │  │  │                  Data Persistence Layer                          │    │   │  ║
║  │  │  ├──────────────────────────────────────────────────────────────────┤    │   │  ║
║  │  │  │                                                                  │    │   │  ║
║  │  │  │  ┌────────────┐  ┌────────────┐  ┌────────────┐               │    │   │  ║
║  │  │  │  │<<Artifact>>│  │<<Artifact>>│  │<<Artifact>>│               │    │   │  ║
║  │  │  │  │            │  │            │  │            │               │    │   │  ║
║  │  │  │  │ PostgreSQL │  │   Redis    │  │   Neo4j    │               │    │   │  ║
║  │  │  │  │  Database  │  │   Cache    │  │   Graph    │               │    │   │  ║
║  │  │  │  └────────────┘  └────────────┘  └────────────┘               │    │   │  ║
║  │  │  └──────────────────────────────────────────────────────────────────┘    │   │  ║
║  │  └─────────────────────────────────────────────────────────────────────────────┘   │  ║
║  └───────────────────────────────────────────────────────────────────────────────────┘  ║
╚═══════════════════════════════════════════════════════════════════════════════════════════╝
```

## Detailed Layer Views

### Business Layer - Process and Service Architecture

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                         BUSINESS CAPABILITY MODEL                              │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │                   Enterprise Architecture Management                   │    │
│  ├──────────────────────────────────────────────────────────────────────┤    │
│  │                                                                        │    │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐         │    │
│  │  │  <<Capability>>│  │  <<Capability>>│  │  <<Capability>>│         │    │
│  │  │                │  │                │  │                │         │    │
│  │  │    Modeling    │  │   Analysis     │  │  Governance    │         │    │
│  │  │  Automation    │  │  Assistance    │  │   Support      │         │    │
│  │  └────────────────┘  └────────────────┘  └────────────────┘         │    │
│  │         │                    │                    │                   │    │
│  │         ▼                    ▼                    ▼                   │    │
│  │  ┌────────────────────────────────────────────────────────┐         │    │
│  │  │              AI-Enhanced Business Services              │         │    │
│  │  ├────────────────────────────────────────────────────────┤         │    │
│  │  │                                                          │         │    │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │         │    │
│  │  │  │<<Service>>   │  │<<Service>>   │  │<<Service>>   │ │         │    │
│  │  │  │              │  │              │  │              │ │         │    │
│  │  │  │ BPMN Design  │  │ ArchiMate   │  │ Code Gen    │ │         │    │
│  │  │  │ Assistant    │  │ Refactoring  │  │ Assistant   │ │         │    │
│  │  │  └──────────────┘  └──────────────┘  └──────────────┘ │         │    │
│  │  └────────────────────────────────────────────────────────┘         │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │                        Value Stream                                  │    │
│  ├──────────────────────────────────────────────────────────────────────┤    │
│  │                                                                        │    │
│  │    Request ──▶ Analyze ──▶ Generate ──▶ Review ──▶ Apply ──▶ Monitor │    │
│  │                                                                        │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────────────────────┘
```

### Information Layer - Data Architecture

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                           DATA ARCHITECTURE VIEW                               │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │                     Core Information Entities                         │    │
│  ├──────────────────────────────────────────────────────────────────────┤    │
│  │                                                                        │    │
│  │  ┌─────────────┐      realizes      ┌─────────────┐                 │    │
│  │  │<<Meaning>>  │◄───────────────────│<<Data       │                 │    │
│  │  │             │                     │ Object>>    │                 │    │
│  │  │ AI Intent   │                     │ GraphPatch  │                 │    │
│  │  └─────────────┘                     └─────────────┘                 │    │
│  │         │                                   │                         │    │
│  │         │ expressed in                     │ composed of             │    │
│  │         ▼                                   ▼                         │    │
│  │  ┌─────────────┐                     ┌─────────────┐                 │    │
│  │  │<<Data       │                     │<<Data       │                 │    │
│  │  │ Object>>    │                     │ Object>>    │                 │    │
│  │  │             │                     │             │                 │    │
│  │  │  Message    │                     │ Operation   │                 │    │
│  │  └─────────────┘                     └─────────────┘                 │    │
│  │         │                                   │                         │    │
│  │         │ aggregated by                    │ transforms              │    │
│  │         ▼                                   ▼                         │    │
│  │  ┌─────────────┐                     ┌─────────────┐                 │    │
│  │  │<<Data       │                     │<<Data       │                 │    │
│  │  │ Object>>    │────────────────────▶│ Object>>    │                 │    │
│  │  │             │     produces        │             │                 │    │
│  │  │Conversation │                     │Domain Model │                 │    │
│  │  └─────────────┘                     └─────────────┘                 │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │                    Information Flow Patterns                          │    │
│  ├──────────────────────────────────────────────────────────────────────┤    │
│  │                                                                        │    │
│  │   Streaming Pattern:                                                  │    │
│  │   ┌────────┐ ══▶ ┌────────┐ ══▶ ┌────────┐ ══▶ ┌────────┐         │    │
│  │   │ Event  │     │ Buffer │     │Process │     │ Output │         │    │
│  │   └────────┘     └────────┘     └────────┘     └────────┘         │    │
│  │                                                                        │    │
│  │   Batch Pattern:                                                      │    │
│  │   ┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐         │    │
│  │   │Collect │────▶│ Batch  │────▶│Process │────▶│Deliver │         │    │
│  │   └────────┘     └────────┘     └────────┘     └────────┘         │    │
│  │                                                                        │    │
│  │   Review Pattern:                                                     │    │
│  │   ┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐         │    │
│  │   │Propose │────▶│ Review │────▶│Approve │────▶│ Apply  │         │    │
│  │   └────────┘     └────────┘     └────────┘     └────────┘         │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────────────────────┘
```

### Technology Layer - Infrastructure Components

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                        TECHNOLOGY INFRASTRUCTURE                               │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │                    Network Infrastructure                             │    │
│  ├──────────────────────────────────────────────────────────────────────┤    │
│  │                                                                        │    │
│  │   Internet ═══════════════════════════════════════════════════════    │    │
│  │       ║                                                                │    │
│  │       ▼                                                                │    │
│  │  ┌─────────────────────────────────────────────────────────┐         │    │
│  │  │            <<Communication Network>>                      │         │    │
│  │  │                 Edge Network                              │         │    │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │         │    │
│  │  │  │  CDN    │  │  WAF    │  │  DDoS   │  │  Load   │   │         │    │
│  │  │  │         │  │         │  │  Protect│  │ Balance │   │         │    │
│  │  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘   │         │    │
│  │  └─────────────────────────────────────────────────────────┘         │    │
│  │                                ║                                       │    │
│  │                                ▼                                       │    │
│  │  ┌─────────────────────────────────────────────────────────┐         │    │
│  │  │         <<Communication Network>>                         │         │    │
│  │  │            Service Mesh (Istio)                          │         │    │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │         │    │
│  │  │  │ Ingress │══│ Virtual │══│ Dest.   │══│ Circuit │   │         │    │
│  │  │  │ Gateway │  │ Service │  │ Rules   │  │ Breaker │   │         │    │
│  │  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘   │         │    │
│  │  └─────────────────────────────────────────────────────────┘         │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │                    Compute Infrastructure                             │    │
│  ├──────────────────────────────────────────────────────────────────────┤    │
│  │                                                                        │    │
│  │  ┌───────────────────┐     ┌───────────────────┐                    │    │
│  │  │  <<Node Pool>>    │     │  <<Node Pool>>    │                    │    │
│  │  │   CPU Optimized   │     │   GPU Enabled     │                    │    │
│  │  ├───────────────────┤     ├───────────────────┤                    │    │
│  │  │                   │     │                   │                    │    │
│  │  │ ┌─────────────┐  │     │ ┌─────────────┐  │                    │    │
│  │  │ │   Node 1    │  │     │ │   Node 1    │  │                    │    │
│  │  │ │ (4CPU/16GB) │  │     │ │ (8CPU/32GB  │  │                    │    │
│  │  │ │             │  │     │ │  + T4 GPU)  │  │                    │    │
│  │  │ └─────────────┘  │     │ └─────────────┘  │                    │    │
│  │  │                   │     │                   │                    │    │
│  │  │ ┌─────────────┐  │     │ ┌─────────────┐  │                    │    │
│  │  │ │   Node 2    │  │     │ │   Node 2    │  │                    │    │
│  │  │ │ (4CPU/16GB) │  │     │ │ (8CPU/32GB  │  │                    │    │
│  │  │ │             │  │     │ │  + T4 GPU)  │  │                    │    │
│  │  │ └─────────────┘  │     │ └─────────────┘  │                    │    │
│  │  └───────────────────┘     └───────────────────┘                    │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │                    Storage Infrastructure                             │    │
│  ├──────────────────────────────────────────────────────────────────────┤    │
│  │                                                                        │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │    │
│  │  │<<Storage>>   │  │<<Storage>>   │  │<<Storage>>   │              │    │
│  │  │              │  │              │  │              │              │    │
│  │  │ Block Store  │  │Object Store  │  │ File System  │              │    │
│  │  │   (SSD)      │  │  (S3/MinIO)  │  │    (NFS)     │              │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────────────────────┘
```

## Cross-Layer Relationships

### Service Realization Matrix

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         SERVICE REALIZATION VIEW                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   Business Service          Information           Technology Component          │
│   ────────────────          ────────────          ───────────────────          │
│                                                                                  │
│   AI Assistance  ─────────▶ Message Stream ──────▶ Inference Gateway           │
│                             GraphPatch Ops         Go Service (6003)            │
│                                                                                  │
│   Change Review  ─────────▶ Review State   ──────▶ GraphPatch Service          │
│                             Approval Data          PostgreSQL Storage           │
│                                                                                  │
│   Model Enhancement ──────▶ Domain Models  ──────▶ LangGraph Engine            │
│                             BPMN/ArchiMate         Python Executor              │
│                                                                                  │
│   Governance Support ─────▶ Audit Trail    ──────▶ Event Store                 │
│                             Event Stream           Kafka/PostgreSQL             │
│                                                                                  │
│   ┌──────────────────────────────────────────────────────────────────┐         │
│   │                     Traceability Matrix                          │         │
│   ├──────────────────────────────────────────────────────────────────┤         │
│   │                                                                   │         │
│   │  Business ←────── realizes ──────▶ Application                  │         │
│   │     ▲                                    │                       │         │
│   │     │                                    │                       │         │
│   │  serves                              uses│                       │         │
│   │     │                                    │                       │         │
│   │     │                                    ▼                       │         │
│   │  Stakeholder ←──── accesses ────▶ Technology                    │         │
│   │                                                                   │         │
│   └──────────────────────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Motivation Layer Integration

### Goal and Principle Alignment

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            MOTIVATION ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌────────────────┐     ┌────────────────┐     ┌────────────────┐             │
│  │   <<Driver>>   │     │   <<Driver>>   │     │   <<Driver>>   │             │
│  │                │     │                │     │                │             │
│  │  Efficiency    │     │   Quality      │     │  Innovation    │             │
│  └────────────────┘     └────────────────┘     └────────────────┘             │
│           │                      │                      │                       │
│           └──────────────────────┴──────────────────────┘                       │
│                                  │                                              │
│                                  ▼                                              │
│                      ┌────────────────────┐                                    │
│                      │     <<Goal>>       │                                    │
│                      │                    │                                    │
│                      │  AI-Augmented      │                                    │
│                      │  Architecture      │                                    │
│                      │   Management       │                                    │
│                      └────────────────────┘                                    │
│                                  │                                              │
│                 ┌────────────────┼────────────────┐                           │
│                 │                │                │                           │
│                 ▼                ▼                ▼                           │
│     ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                    │
│     │<<Principle>> │  │<<Principle>> │  │<<Principle>> │                    │
│     │              │  │              │  │              │                    │
│     │ Semantic     │  │ Human-in-    │  │ Event       │                    │
│     │ Operations   │  │ the-Loop     │  │ Sourcing    │                    │
│     └──────────────┘  └──────────────┘  └──────────────┘                    │
│             │                 │                 │                             │
│             │                 │                 │                             │
│             ▼                 ▼                 ▼                             │
│     ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                    │
│     │<<Requirement>│  │<<Requirement>│  │<<Requirement>│                    │
│     │              │  │              │  │              │                    │
│     │ GraphPatch   │  │ Review       │  │ Audit       │                    │
│     │ Support      │  │ Workflow     │  │ Trail       │                    │
│     └──────────────┘  └──────────────┘  └──────────────┘                    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Implementation Roadmap View

### Migration Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          IMPLEMENTATION ROADMAP                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  PHASE 1: Foundation (Current State)                                            │
│  ┌──────────────────────────────────────────────────────────────────┐          │
│  │  Business:  Basic Chat Service                                    │          │
│  │  Information: Message Streams                                     │          │
│  │  Technology: Connect Protocol + FastAPI                           │          │
│  └──────────────────────────────────────────────────────────────────┘          │
│                              │                                                  │
│                              ▼                                                  │
│  PHASE 2: Control Plane (Target State)                                          │
│  ┌──────────────────────────────────────────────────────────────────┐          │
│  │  Business:  + Review Workflows                                    │          │
│  │  Information: + GraphPatch Operations                             │          │
│  │  Technology: + Go Gateway + Redis Buffer                          │          │
│  └──────────────────────────────────────────────────────────────────┘          │
│                              │                                                  │
│                              ▼                                                  │
│  PHASE 3: Domain Integration (Future State)                                     │
│  ┌──────────────────────────────────────────────────────────────────┐          │
│  │  Business:  + BPMN/ArchiMate Assistance                          │          │
│  │  Information: + Domain Model Transformations                      │          │
│  │  Technology: + Specialized Agents                                │          │
│  └──────────────────────────────────────────────────────────────────┘          │
│                              │                                                  │
│                              ▼                                                  │
│  PHASE 4: Platform Scale (Vision)                                               │
│  ┌──────────────────────────────────────────────────────────────────┐          │
│  │  Business:  Full AI-Augmented EA Management                      │          │
│  │  Information: Cross-Model Traceability                           │          │
│  │  Technology: Multi-Region, Auto-Scaling                          │          │
│  └──────────────────────────────────────────────────────────────────┘          │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Key ArchiMate Relationships

### Relationship Types Used

| Symbol | Relationship | Description |
|--------|-------------|-------------|
| `───▶` | Flow | Information or control flow |
| `═══▶` | Realization | Implementation relationship |
| `┅┅┅▶` | Serving | Service provision |
| `╌╌╌▶` | Access | Data access relationship |
| `────` | Association | Generic relationship |
| `════` | Assignment | Allocation to resource |

### Layer Interactions

```
Business Layer
     │
     │ serves
     ▼
Information Layer  
     │
     │ realized by
     ▼
Technology Layer
```

## Architecture Principles

1. **Separation of Concerns**: Each layer has distinct responsibilities
2. **Service Orientation**: Business capabilities exposed as services
3. **Event-Driven**: All state changes captured as events
4. **Human-in-the-Loop**: Critical decisions require human review
5. **Semantic Operations**: High-level abstractions for AI interactions
6. **Platform Thinking**: Shared infrastructure for all AI capabilities

## Value Streams

### AI-Assisted Modeling Value Stream

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Ideate  │───▶│  Design  │───▶│ Generate │───▶│  Review  │───▶│  Deploy  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
     │               │               │               │               │
   User          AI Agent       GraphPatch      Human Review    Production
  Request       Processing      Generation       & Approval     Application
```

## Compliance and Governance View

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         GOVERNANCE ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌────────────────┐     ┌────────────────┐     ┌────────────────┐             │
│  │  <<Contract>>  │     │  <<Contract>>  │     │  <<Contract>>  │             │
│  │                │     │                │     │                │             │
│  │  Data Privacy  │     │  AI Ethics     │     │  Security      │             │
│  │  (GDPR/CCPA)   │     │  Guidelines    │     │  Standards     │             │
│  └────────────────┘     └────────────────┘     └────────────────┘             │
│           │                      │                      │                       │
│           └──────────────────────┼──────────────────────┘                       │
│                                  │                                              │
│                                  ▼                                              │
│                    ┌──────────────────────────┐                               │
│                    │    <<Assessment>>        │                               │
│                    │   Compliance Framework   │                               │
│                    └──────────────────────────┘                               │
│                                  │                                              │
│                 ┌────────────────┼────────────────┐                           │
│                 │                │                │                           │
│                 ▼                ▼                ▼                           │
│     ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                    │
│     │  <<Control>> │  │  <<Control>> │  │  <<Control>> │                    │
│     │              │  │              │  │              │                    │
│     │ PII Masking  │  │ Human Review │  │ Audit Trail  │                    │
│     └──────────────┘  └──────────────┘  └──────────────┘                    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Summary

This ArchiMate diagram illustrates how the AI backend architecture spans across:

- **Business Layer**: Providing AI-augmented enterprise architecture services
- **Information Layer**: Managing semantic operations and domain transformations
- **Technology Layer**: Leveraging cloud-native infrastructure for scalability

The architecture ensures clear separation between business capabilities, information flows, and technical implementation while maintaining traceability and governance across all layers.

## Acceptance Criteria

- Business, Information, and Technology layers are all represented with clear relationships and service boundaries.
- Human-in-the-loop review and auditability are visible in the diagrams.
- Technology layer shows edge, mesh, compute, and storage components relevant to AI streaming.
- Governance: controls for PII, audit trail, and human review appear and map to platform capabilities.
- Cross-references: Links to POC (141) and UML (142) are accurate.

## Related Documents

- [141-ai-backend-poc.md](./141-ai-backend-poc.md) - Original POC specification
- [142-ai-backend-uml-architecture.md](./142-ai-backend-uml-architecture.md) - UML diagrams
- [ArchiMate 3.2 Specification](https://pubs.opengroup.org/architecture/archimate32-doc/) - Standard reference

---

## 145-chat-artifact.md

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

---

## 146-ai-backend-migration.md

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

---

## 147-artifact-first-routes.md

# Artifact-First Route Architecture

## Status: Planning
**Created**: 2025-01-22
**Priority**: High
**Complexity**: Medium
**Related**: 
- [145-threads-artifact.md](./145-threads-artifact.md) - Backend gRPC architecture
- [146-ai-backend-migration.md](./146-ai-backend-migration.md) - AI backend integration

## Problem Statement

Users lose context when navigating between artifacts and threadss due to:
1. URL instability - Pages redirect or rewrite URLs
2. Lost state on refresh - Artifact pages redirect to threads URLs
3. Confusing navigation - Artifacts subordinate to threadss
4. Race conditions - Multiple tabs can create duplicate threadss

## Solution: Stable URL Architecture

### Core Principle: The "No URL Mutations" Rule

**CRITICAL INVARIANT**: Once a user navigates to a URL, that URL NEVER changes in the browser bar until they explicitly navigate away.

```typescript
// ❌ FORBIDDEN - These patterns are banned:
router.replace(`/threads/${threadsId}`);
window.history.replaceState(...);
router.push(`/threads/${threadsId}`, { replace: true });

// ✅ ALLOWED - Only explicit navigation:
<Link href={`/threads/${threadsId}`}>View Chat</Link>
router.push(`/threads/${threadsId}`);  // New navigation only
```

### URL Contract

**Routes** (singular, permanent):
- `/artifact/[artifactId]` - Artifact viewer with integrated threads panel
- `/threads/[threadsId]` - Chat viewer with artifact navigator

**Hard Invariants**:
- URL displayed in browser NEVER changes while interacting on a page
- Sending first message from `/artifact/[id]` creates threads without URL change
- All cross-entity navigation uses explicit links (new tab or user click)
- Server-side 308 redirect from `/artifacts/*` → `/artifact/*` (migration only)

## Frontend Implementation

### Artifact Viewer Page

`/app/artifact/[artifactId]/page.tsx`:

```typescript
export default async function ArtifactPage({ 
  params 
}: { 
  params: { artifactId: string } 
}) {
  const { artifactId } = params;  // params is synchronous in Next.js
  const session = await auth();
  
  if (!session) {
    redirect('/login');
  }
  
  return (
    <ArtifactProvider artifactId={artifactId}>
      <ChatProvider>
        <ArtifactViewer 
          artifactId={artifactId}
          session={session}
        />
      </ChatProvider>
    </ArtifactProvider>
  );
}
```

**Key Features:**
- Load artifact from gRPC backend via SDK
- Show sidebar with associated threads sessions
- Create new threads atomically when sending first message
- **Stay on `/artifact/[id]` URL always** (no mutations)
- Support switching between related threadss

### Chat Page

`/app/(threads)/threads/[id]/page.tsx`:

```typescript
export default async function ChatPage({ 
  params 
}: { 
  params: { id: string } 
}) {
  const { id } = params;  // params is synchronous in Next.js
  const session = await auth();
  
  if (!session) {
    redirect('/login');
  }
  
  return (
    <ChatProvider threadsId={id}>
      <ArtifactProvider>
        <ChatViewer 
          threadsId={id}
          session={session}
        />
      </ArtifactProvider>
    </ChatProvider>
  );
}
```

**Key Features:**
- Load threads and messages from gRPC backend
- Show artifact navigator with all related artifacts
- Support switching between artifacts
- **Stay on `/threads/[id]` URL always** (no mutations)
- Real-time updates via streaming

## Atomic Chat Creation from Artifact

The key to stable URLs is the atomic RPC that creates a threads and sends the first message in one operation:

```typescript
// In ArtifactViewer component
const sendFirstMessage = async (content: string) => {
  // This atomic operation prevents race conditions
  const response = await client.threadsCommands.ensureChatForArtifactAndSendMessage({
    artifactId,
    message: {
      role: 'user',
      parts: [{ text: content }]
    },
    idempotencyKey: crypto.randomUUID(),
  });
  
  // Update UI to show new threads in sidebar
  // BUT DO NOT CHANGE THE URL
  setSidebarChatId(response.threadsId);
  
  if (response.threadsCreated) {
    // Add new threads to sidebar list
    addChatToSidebar({
      id: response.threadsId,
      title: `Discussion: ${artifact.title}`
    });
  }
};
```

## Navigation Patterns

### Cross-Entity Navigation

```typescript
// From artifact to threads - explicit user action
<Link 
  href={`/threads/${threadsId}`}
  target="_blank"  // Optional: open in new tab
>
  Open Full Chat
</Link>

// From threads to artifact - explicit user action
<Link 
  href={`/artifact/${artifactId}`}
  target="_blank"
>
  View Artifact
</Link>
```

### Chat/Artifact Switching

```typescript
// Switch between threadss in artifact view
const switchChat = (threadsId: string) => {
  // Update displayed messages
  setActiveChatId(threadsId);
  // DO NOT change URL
};

// Switch between artifacts in threads view
const switchArtifact = (artifactId: string) => {
  // Update artifact preview
  setActiveArtifactId(artifactId);
  // DO NOT change URL
};
```

## Implementation Guidelines

### DO:
- Use the atomic `ensureChatForArtifactAndSendMessage` RPC
- Keep URLs stable throughout user interaction
- Use explicit navigation for route changes
- Store UI state (active threads/artifact) separately from URL

### DON'T:
- Use `router.replace()` or `window.history.replaceState()`
- Redirect after creating a threads
- Mutate URLs based on user actions
- Mix URL state with UI state

## Migration Strategy

### Phase 1: Fix Artifact Pages
1. Remove redirect from `/artifacts/[artifactId]/page.tsx`
2. Implement proper artifact viewer with threads sidebar
3. Use atomic RPC for threads creation

### Phase 2: Update Chat Pages
1. Add artifact navigator to threads view
2. Enable artifact switching without URL change
3. Implement bidirectional relationships

### Phase 3: Clean Up
1. Remove all `router.replace()` calls
2. Audit for `replaceState` usage
3. Add server-side redirects for old URLs

### Server Redirects (Example)

Implement a single 308 from legacy plural paths to singular in `middleware.ts`:

```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const url = new URL(request.url)
  if (url.pathname.startsWith('/artifacts/')) {
    url.pathname = url.pathname.replace('/artifacts/', '/artifact/')
    return NextResponse.redirect(url, { status: 308 })
  }
  return NextResponse.next()
}
```

## Success Metrics

- **URL Stability**: 100% of page refreshes maintain same URL
- **No Lost Context**: Refresh always shows same content
- **No Duplicates**: Zero duplicate threadss from race conditions
- **User Satisfaction**: Reduced confusion from URL changes

## Technical Dependencies

- Backend atomic RPC: `EnsureChatForArtifactAndSendMessage`
- SDK support for atomic operations
- Bidirectional threads-artifact relationships
- Idempotency keys for safe retries

## Related Documents

- [145-threads-artifact.md](./145-threads-artifact.md) - Backend gRPC architecture
- [146-ai-backend-migration.md](./146-ai-backend-migration.md) - AI backend integration

## Acceptance Criteria

- Artifact page (`/artifact/[id]`) never mutates the URL during use, including first message send.
- Chat page (`/threads/[id]`) never mutates the URL during use.
- Atomic RPC `EnsureChatForArtifactAndSendMessage` is used from artifact view to prevent race conditions.
- No usages of `router.replace` or `window.history.replaceState` in pages/components.
- Server 308 redirect exists for legacy `/artifacts/*` to `/artifact/*`.

---

## 148-azure-managed-offering.md

# Azure Managed Application Offering

## Context
- We want to package the AMEIDE control-plane cluster as an Azure Marketplace Managed Application that tenants can deploy into their subscriptions while we retain day-2 management.
- All control-plane UIs (Argo CD, Grafana, Prometheus, Alertmanager, Loki, Tempo, Temporal Web, pgAdmin, Kubernetes Dashboard) now ship Azure AD configurations in Git; by default every environment points at the shared `ameide-aks-admins` group while dedicated read-only groups are evaluated, and rollout remains gated on supplying final client IDs/secrets per environment.
- External Secrets targets Azure Key Vault with per-environment overlays; secret keys for every control-plane client are declared, but production/staging vaults still need to be populated and automated rotation defined. Local development now reads from the Ameide-Local vault (`ameidelocalkv`). A helper script (`scripts/azure/seed-control-plane-secrets.sh`) can seed vault entries from a JSON payload when needed.
- Marketplace ingestion requires an ARM/Bicep deployment package (template + `createUiDefinition`) plus operations collateral; Terraform can help internally but is not a supported packaging format for the listing.

## Goals
- Ship a single Bicep-based Managed Application that can deploy either the staging or production footprint (parameterised `environment`).
- Move control-plane SSO (Argo CD, Grafana, Kubernetes Dashboard proxy, observability UIs) to Azure AD with per-app registrations and group-based RBAC.
- Store runtime secrets exclusively in Azure Key Vault for staging/production and surface them to the clusters via External Secrets with workload identity.
- Document and automate day-2 operations (patching, telemetry, backups, support) to meet Marketplace requirements.
- Produce partner-ready collateral (runbooks, SLA, support model, pricing inputs) accompanying the technical artefacts.

## Non-Goals
- Migrating tenant-facing applications away from Keycloak (they still use federated directories).
- Shipping a public multi-tenant PaaS offer; this scope is limited to managed clusters we operate for customers.
- Refactoring existing Terraform inventories beyond what is required for DNS or shared services.

## Current State Snapshot
- **Identity**
  - Argo CD now ships an Azure AD OIDC profile (`infra/kubernetes/values/common/argocd.yaml`) with per-environment overrides; `ameide-aks-admins` is wired everywhere as the interim admin/read-only group while the need for distinct read-only roles is assessed.
  - Grafana consumes `auth.azuread` settings and Prometheus / Alertmanager / Loki / Tempo are fronted by dedicated oauth2-proxy releases; secrets are sourced from Azure Key Vault across all environments (local uses `ameidelocalkv`).
  - Azure AD app registrations remain provisional—client IDs, secrets, and group assignments must be captured in automation or documentation before launch.
- **Secrets**
  - Global `azure-keyvault-secrets` chart now exposes client/ cookie secrets for Argo CD, Prometheus, Alertmanager, Loki, Tempo, Temporal, and pgAdmin; staging/production overlays set explicit vault names and workload identity is enabled.
  - Local development relies on the Ameide-Local Key Vault (`ameidelocalkv`) with the same secret names; automation is still required to populate customer Key Vault entries for managed environments.
- **Marketplace Packaging**
  - Initial Bicep scaffolding (`infra/bicep/managed-application/`) defines AKS, user-assigned identity, Key Vault, Log Analytics, and optional DNS plus a `createUiDefinition.json`; pipelines still need to compile and ship the artefacts.
  - CI/CD does not yet bundle marketplace-ready artefacts.

## Scope
- Environments: staging and production (customer Key Vaults) plus local development (Ameide-Local Key Vault).
- Resource coverage: AKS cluster, ACR, Azure Key Vault, DNS wiring, Log Analytics, and per-environment Managed Identities needed for runtime.
- Deliver Managed Application definition with tenant onboarding/offboarding automation.

## Deliverables
- Bicep template + `createUiDefinition.json` (tracked under `infra/bicep/managed-application/`).
- Azure AD app registration definitions (Terraform/Bicep module or documented CLI) with group mappings and secret rotation process.
- Updated Helm/Helmfile values for Argo CD, Grafana, Kubernetes Dashboard proxy, Prometheus/Alertmanager, Loki, Tempo to consume Azure AD + Key Vault secrets.
- GitOps updates for Argo CD App-of-Apps pointing to this graph.
- Operational documentation: runbooks, support workflows, SLA summary, compliance notes, cost model inputs.
  - Initial runbook captured in `docs/operations/managed-application-runbook.md`.
- CI/CD pipeline segment that bundles and publishes the Managed Application artefacts for Partner Center ingestion.

## Latest Progress
- [x] Switched Argo CD to a Helmfile-managed OCI release with Azure AD OIDC and documented the new bootstrap flow.
- [x] Introduced oauth2-proxy releases for Prometheus, Alertmanager, Loki, and Tempo with Azure AD wiring across local/staging/production.
- [x] Extended Azure Key Vault manifest to surface client credentials for all control-plane UIs and set explicit vault names per environment.
- [x] Added marketplace Bicep scaffold (modules, parameters, `createUiDefinition.json`) to unblock packaging automation.
- [x] Switched local Kubernetes Dashboard to Azure AD so every control-plane surface uses Microsoft Entra ID.
- [x] Created packaging helper (`infra/bicep/managed-application/package.sh`) and day-2 operations runbook (`docs/operations/managed-application-runbook.md`).
- [x] Adopted `ameide-aks-admins` as the default control-plane group across environment overlays and added a Key Vault seeding script for oauth2-proxy credentials.
- [ ] Publish automation to mint Azure AD app registrations and rotate secrets.
- [ ] Add CI packaging step that lints, what-ifs, and uploads Managed Application artefacts.
- [ ] Resolve remaining Bicep lint warnings (Key Vault identity metadata, AKS admin username parameterisation, optional DNS module outputs) before Partner Center submission.

## Execution Plan
### 1. Identity Alignment (Azure AD)
- [x] Capture Azure AD settings for every control-plane surface in Helm (redirect URIs, scopes, logout URLs, CLI flows).
- [ ] Automate Azure AD app registrations and ongoing group management (ensuring `ameide-aks-admins` membership is tracked and future read-only splits are supported) and feed the resulting object IDs into environment values.
- [ ] Document emergency access (temporary group membership, admin re-enable) and make it part of the operated runbooks.

### 2. Secret Management Hardening
- [x] Parameterise Key Vault settings and secret names across environments (Helmfile + ExternalSecrets).
- [ ] Seed staging/production vaults with the declared secrets and verify ExternalSecret reconciliation.
- [ ] Implement automated secret provisioning/rotation (CLI/Terraform/Bicep) for every control-plane client.

### 3. Managed Application Packaging
- [x] Scaffold `infra/bicep/managed-application/` (modules, `main.bicep`, parameters, `createUiDefinition.json`, packaging helper).
- [ ] Extend modules for customer networking options and document Managed Resource Group RBAC expectations.
- [ ] Add CI validation/publishing (Bicep lint/what-if, artefact bundling via `package.sh` for Partner Center submissions).
- [ ] Gate CI on zero Bicep warnings or capture justifiable suppressions for outstanding SDK gaps.

### 4. Operations & Compliance Readiness
- [x] Draft managed-application runbook (`docs/operations/managed-application-runbook.md`).
- [ ] Finalise monitoring/alerting hand-off (Log Analytics, Azure Monitor, ticketing integration) and codify it in the runbook.
- [ ] Define upgrade cadence & support collateral (SLAs/SLOs, pricing, data handling, escalation paths).
- [ ] Assemble the marketplace submission package (offer description, legal artefacts, telemetry statements).

### 5. Publishing & Partner Center Intake
- [ ] Stand up a packaging pipeline that versions the Managed Application artefacts, uploads them to storage, and generates the Partner Center submission bundle.
- [ ] Dry-run the Partner Center submission flow with a sandbox offer to validate metadata and policy requirements.
- [ ] Capture ongoing update cadence (e.g., monthly patch rollup) and align with Partner Center update process.

## Validation
- End-to-end dry run in a sandbox subscription deploying the Managed Application (both staging and production variants).
- Authentication journey tests for each control-plane app using Azure AD accounts and groups.
- Secret rotation drill (Key Vault -> External Secrets -> pod reload) in staging.
- Bicep CI pipeline must lint, run what-if, and attach artefacts before tagging a release.

## Dependencies
- Azure AD tenant admin to create app registrations, groups, and grant admin consent.
- Access to Partner Center with permissions to create Managed Application offers.
- Managed identity/role assignments for AKS and automation principals.

## Open Questions
- Where should Azure AD app registrations live in IaC (Terraform vs Bicep vs manual)? Need decision for long-term drift control.
- Do we bundle customer networking (VNet, subnet, private DNS) or assume pre-existing infrastructure provided as parameters?
- Is there a requirement for customer-managed keys or bring-your-own Key Vault that the template must support?
- Which telemetry stack (Azure Monitor, Application Insights, Grafana) must be surfaced to customers vs kept internal?

## Risks & Mitigations
| Risk | Impact | Mitigation |
| --- | --- | --- |
| Azure AD registrations drift outside IaC | Marketplace deployments fail auth | Adopt Terraform/Bicep module, automate secret rotation, document verification checklist |
| Key Vault secret seeding incomplete | Managed Application fails post-deploy | Include pre-flight validation script, add health probes, fail fast in Helmfile |
| Partner Center rejection due to artefact gaps | Launch delayed | Establish lint/what-if gates, run sandbox submission, maintain compliance checklist |
| Customer networking assumptions wrong | Deployment fails in customer subscription | Parameterise VNet/peering options, document prerequisites, include validation step |

## Timeline (Tentative)
- **Phase 1 (Weeks 1–2):** Finalise Azure AD automation, seed Key Vault, close identity deliverables.
- **Phase 2 (Weeks 3–4):** Harden Bicep modules, resolve lint warnings, integrate CI packaging.
- **Phase 3 (Weeks 5–6):** Sandbox deployment validation, Partner Center submission dry-run, publish operations collateral.

---

## 150-enterprise-repository-business-scenario.md

# Enterprise Repository Business Scenarios (PO-first)

## Orientation
The Enterprise Architecture graph is the tenant-wide system of record for approved architecture assets, structured according to TOGAF 9.2 (Architecture Capability, Metamodel, Landscape Baseline/Target/Transition, Standards Information Base, Reference Library, Requirements Repository, and Governance Log). Repository configuration now includes:
- **Governance Bundles**: `GovernancePolicy`, `PlacementRule`, and `RequiredCheck` cataloged once per graph.
- **Access Fabric**: `RepositoryAccessRole` + `RepositoryMembership` supply least-privilege access and auto-link stakeholders to the right `GovernanceOwnerGroup`.
- **Operational Telemetry**: `RepositorySyncJob` publishes changes to knowledge graph, search, and analytics endpoints; `RepositoryStatisticSnapshot` captures queue, approval, and SLA health.

The graph still operates in a three-context model:
1. **Enterprise Repository** – Organization-level TOGAF structure containing approved, governed assets, including releases and sync telemetry.
2. **Initiative Workspace** – ADM phase structure (Preliminary, A–H, Requirements Management) for work-in-progress deliverables.
3. **Initiative Repository View** – Filtered, read-only references showing which approved assets an transformation consumes, including release versions and retention flags.

Governance—`GovernanceOwnerGroup`, approval rules, required checks, routing rules, promotion queue—lives at graph/tenant scope and applies to every artifact review. Transformation transformations consume graph truth (assignments, effective windows, releases, governance outputs) without redefining policy. The scenarios below describe the PO-ready playbook for operating the graph with the updated governance and telemetry fabric.

---

## Scenario A1. Govern Once, Enforce Everywhere
- **Trigger:** New graph or governance refresh.
- **Owner:** Governance Lead (configures); Product Owner (consumes outcomes).
- **Flow:** Provision `RepositoryAccessRole` templates → assign memberships → configure `GovernancePolicy` (rules, playbook, retention, SLA) → link `GovernanceOwnerGroup` and `PlacementRule` → publish governance profile banner. From this point, any artifact requesting review automatically inherits guardrails, routing rules, and SLA expectations.
- **PO Value:** Initiative boards surface the same required checks/approvals with zero per-transformation setup; graph dashboard exposes governance readiness via `RepositoryStatisticSnapshot`.
- **KPIs:** % artifacts blocked by missing owner group; approval cycle time P50/P95; required check pass rate; SLA breach count.
- **Spec references:** FR‑12..16, FR‑20; Repository Stage 3 (Governance Deep Dive).

## Scenario A2. Draft → In Review → Approved (Artifact-centric)
- **Trigger:** Architect requests review on an artifact tied to an transformation.
- **Owner:** Artifact Owner (executes); PO monitors status.
- **Flow:** `Request review` → reviewers auto-assigned from `GovernanceOwnerGroup` membership (fed by `RepositoryAccessRole`) → required checks execute with `CheckRun` logging → approvals collected via `ApprovalDecision` → promotion queue validates `PlacementRule` and SLA → `LifecycleEvent` set to `approved`; `RepositorySyncJob` publishes AssetVersion updates.
- **PO Value:** Board cards display a single chip summarizing checks, approvals, queue position, and sync status (search/graph).
- **KPIs:** Approval cycle time P50/P95; queue wait P95; % artifacts with `release_package_id`; sync success rate.
- **Spec references:** FR‑11, FR‑13, FR‑14, FR‑15; Repository Stages 2–4.

## Scenario A3. Time Windows & Scenarios at the Source
- **Trigger:** EA prepares As-Is/To-Be or scenario planning data.
- **Owner:** Enterprise Architect (records data); PO consumes toggles in transformations.
- **Flow:** EA sets `effective_from`, `effective_to`, `scenario_tag`, and `visibility` on `ClassificationAssignment`; smart folders expose the metadata; transformations consume via `InitiativeReference`. `RepositorySyncJob` emits `AssetSyncPayload` so transformation dashboards update automatically.
- **PO Value:** Initiative scenario toggles are trustworthy because the graph is the canonical store.
- **KPIs:** % transformation-linked assignments with windows; variance between As-Is/To-Be counts; scenario toggle latency (graph → transformation sync); % assignments auto-validated by `PlacementRule`.
- **Spec references:** FR‑18; graph DDL (`graph_assignments`); Stage 5 Time-Aware Modeling.

## Scenario A4. Release the Decision
- **Trigger:** Artifact reaches `approved` state.
- **Owner:** Architect (publishes release); PO shares readiness.
- **Flow:** Publish Release (notes + assets) → protected tags generated → `AssetVersion` flagged with release metadata → `RepositorySyncJob` dispatches release payload to search/analytics → transformations pull release data into readiness reports with synced status badges.
- **PO Value:** Release readiness reports draw directly from graph releases, required checks, and sync telemetry to confirm dissemination.
- **KPIs:** Releases per milestone; % cards released; download counts per release asset; release sync success latency.
- **Spec references:** FR‑15, governance dashboards, Stage 4 collaboration.

## Scenario A5. Data Quality & Recertification
- **Trigger:** Missing metadata or policy change.
- **Owner:** Governance Lead; PO tracks risk.
- **Flow:** Data-quality surveys enter queue with `RepositoryStatisticSnapshot` context → responses feed `GovernanceMetricSnapshot` → recertification jobs push approved items back to `in_review`, logging `AssetRetentionEvent` and `SlaBreachEvent` where needed → downstream systems updated via `RepositorySyncJob`.
- **PO Value:** Repository flags “Needs data” or “Recertified” so transformations can adjust plans.
- **KPIs:** Data-quality score trend; overdue surveys; recertification cycle time; # recertified artifacts synced within SLA.
- **Spec references:** FR‑20; Stage 4.

---

## Edge Cases & Playbooks
- **Scenario Collision:** Setting overlapping windows for the same artifact/scenario triggers validation (optional `btree_gist` exclusion). Queue refuses promotion until one window is adjusted. *Owners:* PO + Governance Lead.
- **Queue Contention:** Multiple artifacts with identical priority remain queued; graph queue shows FIFO position and responsible owners. PO coordinates priority adjustments with Governance Lead.

---

## Acceptance Tests (Repository-backed, PO-centric)
1. **Blockers Funnel (PO view):** Given mixed governance states, when a PO filters “Needs approvals” via transformation board, then graph events update card chips and funnel counts within 30 s, including deep links to artifact timelines and latest `RepositorySyncJob` status. *(FR‑12/13; Repository Stage 3)*
2. **Recertification Trigger:** Given a policy change, when recertification pushes an approved artifact back to `in_review`, then graph marks the artifact, logs the reason, emits `AssetRetentionEvent`, and transformations display the risk flag and sync timestamp in readiness reports. *(FR‑20; Stage 4)*

---

## KPIs & Telemetry Checklist
- Search P95 < 2.5 s; tree fetch P95 < 300 ms.
- Approval cycle time P50/P95; queue wait P95; SLA breach count (per `SlaBreachEvent`).
- % artifacts with pinned releases; release sync latency P95; data-quality score ≥ threshold.
- Collision count/time-to-resolution; survey completion rate; RepositorySyncJob success rate.

---

## Why This Fits the Specs
- Governance configured once at graph scope (FR‑12..16) applies everywhere.
- Lifecycle remains artifact-level; discussions and review requests stay artifact-specific.
- Initiatives consume graph truth (assignments, windows, releases, governance outputs) without duplicating policies.
- Metrics and acceptance tests tie directly to implementation stages (2–5), providing ready-made definition of done for POs and engineering.

---

## 151-enterprise-repository-functional.md

# Executive Summary
Enterprise graph management in Ameide enables each tenant to curate a structured knowledge base of architecture assets—standards, reference architectures, and authoritative current-state baselines following TOGAF 9.2 structure. Each graph now bundles governance policies, access roles, and telemetry sinks so transformations inherit consistent rules, routing, and KPIs. The system operates in three distinct contexts:

1. **Enterprise Repository** (`/org/{orgId}/graph`) - TOGAF-structured organization-level view containing approved, governed architecture assets organized into Architecture Capability, Metamodel, Landscape (Baseline/Target/Transition), Standards Information Base (SIB), Reference Library, Requirements Repository, and Governance Log.

2. **Initiative Workspace** (`/org/{orgId}/transformations/{id}/workspace`) - ADM phase structure (Preliminary, Phases A-H, Requirements Management) for work-in-progress deliverables where teams collaborate on architecture development following the TOGAF ADM cycle.

3. **Initiative Repository View** (`/org/{orgId}/transformations/{id}/graph`) - Filtered, read-only view showing which approved enterprise graph assets a specific transformation references and consumes.

Repositories are tenant resources that anchor the state of the enterprise architecture. Initiatives layer planning views over graph assignments, but governance remains defined once at graph/tenant scope and distributed through `RepositoryAccessRole`, `GovernanceOwnerGroup`, and `RepositorySyncJob` orchestration. This specification focuses on graph capabilities: multi-tenant provisioning, hierarchical navigation, artifact assignment, governance-aware discovery, access control, telemetry, and artifact-centric collaboration. Initiative-specific flows are defined in `backlog/161-transformation-transformations-functional.md`.

# What Problem Does This Solve?
- Fragmented architecture assets dispersed across diagrams, documents, and spreadsheets slow down architectural governance and transformation execution.
- Tenants lack a consistent taxonomy that maps artifacts to TOGAF phases, ArchiMate layers, and governance checkpoints.
- Architects and developers cannot easily discover vetted artifacts or understand their lineage, causing duplication and compliance gaps.
- Governance leads lack transparency into graph usage, preventing KPI tracking and compliance auditing.

# Key Capabilities
- Multi-tenant graph provisioning with `tenant_id` isolation and `(tenant_id, key)` uniqueness guarantees.
- Hierarchical navigation backed by materialized paths for tenant taxonomies, including folders, governance phases, and smart folders (virtual nodes with filter expressions).
- Artifact curation: repositories own the canonical catalog of tenant artifacts (BPMN processes, ArchiMate models, UML diagrams, DMN tables, strategies, documents, code assets). Each artifact carries metadata about kind, lifecycle, owners, and referenced transformations.
- Artifact assignment with version pinning and semantic roles to support governance and reuse.
- gRPC-based CRUD, tree manipulation, search, and stats services consumable via `@ameideio/ameide-sdk-ts`.
- Template-driven graph creation (e.g., Standards Catalog, Reference Architecture, Transformation Initiative collateral) with configurable node types and policies.
- Cross-graph search and smart folder surfacing for knowledge discovery.
- Governance access fabric via `RepositoryAccessRole` and `RepositoryMembership` with automatic governance owner enrollment and SLA targets.
- Sync and telemetry services (`RepositorySyncJob`, `RepositoryIntegrationEndpoint`, `RepositoryStatisticSnapshot`) pushing deltas to knowledge graph, search, and analytics pipelines.
- Retention and data-quality workflows (`AssetRetentionEvent`, survey orchestration) to uphold compliance.
- Audit-ready telemetry: assignment history, queue latency, SLA breaches, graph stats, and governance checkpoints.

# User Personas & Scenarios
- **Enterprise Architect (EA)**: Seeds and curates tenant repositories (standards catalog, reference architecture, transformation collateral), tailors taxonomies, and monitors progress across transformation transformations.
- **Solution Architect (SA)**: Browses graph trees to clone reusable architecture building blocks, verifies dependencies, and aligns new designs with standards.
- **Governance Lead (GL)**: Audits compliance by reviewing phase-specific assignments, checks pinned versions, and exports graph stats for steering committees.
- **Developer (DEV)**: Quickly searches smart folders (e.g., “Phase C approved APIs”) to access implementation-ready artifacts and confirm ownership.

# Functional Requirements
- **FR-1 Repository Lifecycle**: Support create, read, update, delete, and list operations per tenant via `RepositoryAdminService` (proto `graph_service.proto`). Accept template IDs, enforce optimistic locking, and emit audit events.
- **FR-2 Template Application**: Allow EA to initialize repositories from predefined templates (standards catalog, reference architecture, transformation transformation collateral) with default node definitions and config JSON (see `graph.proto` `RepositoryConfig.default_tree`).
- **FR-3 Tenant Isolation**: Every operation must supply `tenant_id` in the request context; backend repositories (`TenantRepository`) must refuse cross-tenant data access except for explicitly authorized admin flows.
- **FR-4 Tree Navigation**: Provide create, update, move, delete, list-children, and get-tree operations via `RepositoryTreeService`, preserving adjacency (`parent_id`) and recomputing materialized `path` (`NodeRepo.compute_path`).
- **FR-5 Smart Folder Support**: Enable virtual nodes with `is_virtual=true` and evaluable `filter_expression` so that governance filters can project artifacts dynamically without duplication.
- **FR-6 Artifact Assignment**: Allow assigning/unassigning artifacts to nodes through `RepositoryAssignmentService` with optional `pinned_version` and `role`, enforcing uniqueness of `(artifact_id, node_id)` per graph and honoring optimistic locking.
- **FR-7 Version Pinning & Audit**: Persist and surface pinned versions; if absent, services must resolve latest artifact state at read time while recording assignment updates for audit logs.
- **FR-8 Cross-Repository Search**: Expose `RepositoryQueryService` endpoints for keyword, attribute, and filter-based queries across repositories, including smart-folder semantics and pagination.
- **FR-9 UI Integration**: The Next.js `RepositoryBrowser` must switch between tree/list/grid views, fetch data via `@ameideio/ameide-sdk-ts`, support drag-and-drop moves, and load artifact details using `artifactQueries` SDK calls. Transformation transformation workspaces layer As-is / To-be perspectives on top of graph assignments.
- **FR-10 Governance Reporting**: Provide stats endpoints (`GetRepositoryStats`) returning node/assignment counts and template-specific aggregations (e.g., by governance phase or catalog family) so Governance Leads can assess adoption and compliance.
- **FR-11 Artifact Lifecycle**: Expose lifecycle state management for artifacts (`draft → in_review → approved → deprecated → archived`). Implement state transitions via the shared artifact command bus described in `backlog/120-artifacts-north-star.md`, so humans and agents issue the same typed commands, optional CRDT collab can feed state changes, and GitHub-style interactions (labels, assignees, subscribers, comments) trigger workflows/webhook automation and governance checks before promotion.
- **FR-12 Node Owners & Approval Rules**: Allow repositories to declare ownership and approval policies (NodeOWNERS) at node and graph scope. Transitions to `approved` must honor rule settings (minimum approvals, owner-required, segregation-of-duties) with audit trails. Governance policies apply tenant-wide across repositories.
- **FR-13 Required Checks & Check Runs**: Register named required checks per graph/node against a canonical check registry and block lifecycle transitions until all check runs succeed. Expose APIs/UI to view, re-run, and diagnose check results akin to GitHub status checks.
- **FR-14 Transition Queue**: Serialize conflicting lifecycle transitions through a priority-aware queue that revalidates approvals and required checks before finalizing state changes; enforce idempotency keys/duplicate guards and surface queue status + SLA metrics in the UI.
- **FR-15 Artifact Releases & Protected Tags**: Support publishing immutable releases with notes and downloadable assets when artifacts reach `approved`. Auto-generate protected semantic tags and enforce retention policies through rulesets.
- **FR-16 Rulesets & Policy Bundles**: Provide reusable rulesets that bundle approval rules, required checks, label policies, and tag protections at tenant, template, or graph scope. Version rulesets and record which version approved each transition.
- **FR-17 Discussions & Decision Records**: Offer threaded discussions, ADR templates, and decision timelines that are anchored to individual artifacts (with links back to nodes/transformations as context), mirroring GitHub Discussions patterns.
- **FR-18 Time-Scoped Assignments & Impact Views**: Capture `effective_from`/`effective_to` windows and scenario tags on assignments to enable as-is/to-be comparisons, roadmap overlays, and impact analysis.
- **FR-19 Initiative Planning Boards**: Deliver table/board/roadmap views for transformation transformations that pull graph assignments, lifecycle state, risk scores, and linked checks into LeanIX-style planning surfaces.
- **FR-20 Surveys & Data Quality**: Provide survey workflows to collect missing metadata from owners, compute data quality scores, and flag overdue updates within governance dashboards.

# Key Workflows
## Create Repository from Tenant Template
```mermaid
sequenceDiagram
    participant EA as Enterprise Architect
    participant FE as Platform Web App
    participant Admin as RepositoryAdminService
    participant Tree as RepositoryTreeService
    participant Store as Event Store/DB
    EA->>FE: Initiate "Create Repository" (select tenant template)
    FE->>Admin: CreateRepository(context{tenant}, graph{template_id=selected})
    Admin->>Store: Persist graph (attributes/config, template metadata)
    Admin-->>FE: CreateRepositoryResponse(graph_id)
    FE->>Tree: GetTree(graph_id) // default tree from template
    Tree->>Store: Load node definitions and materialize nodes
    Tree-->>FE: Tree nodes (folders, governance phases, or transformation views)
    FE-->>EA: Show graph scaffold ready for customization
```

## Assign Artifact to Taxonomy Node
```mermaid
sequenceDiagram
    participant SA as Solution Architect
    participant FE as Platform Web App
    participant Tree as RepositoryTreeService
    participant Assign as RepositoryAssignmentService
    participant Artifact as ArtifactQueryService
    SA->>FE: Drag artifact onto "Governance Phase B"
    FE->>Tree: ListChildren(graph_id, parent=Phase B) // ensure node exists
    Tree-->>FE: Node metadata (id/path)
    FE->>Assign: CreateAssignment(context{tenant}, graph_id, node_id, artifact_id, pinned_version)
    Assign->>Store: Persist assignment (order, role)
    Assign-->>FE: Assignment confirmation
    FE->>Artifact: GetArtifact(artifact_id) // hydrate UI details
    FE-->>SA: Updated node with artifact card and pinned label
```

## Cross-Repository Search
```mermaid
sequenceDiagram
    participant DEV as Developer
    participant FE as Platform Web App
    participant Query as RepositoryQueryService
    participant Search as Search Index/Read Model
    DEV->>FE: Search smart folder "Phase C APIs"
    FE->>Query: SearchRepositories(context{tenant}, filter{phase=C, kind=api})
    Query->>Search: Execute query across graph read models
    Search-->>Query: Matching nodes and assignments with metadata
    Query-->>FE: Paginated results (graph, node, artifact summaries)
    FE-->>DEV: Display list/grid with quick actions and deep links
```

# Artifact Lifecycle & Governance Triggers
- **State model**: Each artifact carries a lifecycle state (`draft`, `in_review`, `approved`, `deprecated`, `archived`). Transitions require explicit actions (e.g., “Request Review”, “Approve”, “Archive”) and may be conditioned on workflows checks (tests, architectural guardrails).
- **Command bus alignment**: State transitions originate from a unified command bus (human UI, automation, or agents). Each transition packages base version, origin, and validation status to keep graph assignments consistent.
- **GitHub-like interactions**: Artifacts support assignees, reviewers, labels, watchers, and threaded comments. Label updates (e.g., `needs-compliance`, `blocked`) and reviewer requests fire domain events similar to GitHub issue/pull request timelines.
- **Automation hooks**: State changes publish events (`artifact.lifecycle.transitioned`) consumed by workflows engines. Required checks (policy evaluators, CI results) must report status before `approved` transitions succeed; failure reverts to `in_review` with blocking metadata.
- **Governance actions**: Approved artifacts auto-pin their version in critical graph nodes, notify Governance Leads, and schedule periodic recertification tasks. Deprecated artifacts trigger notifications to dependent repositories and flag transformation transformations linked through bidirectional labels.
- **Audit trail**: Maintain chronological activity feeds per artifact (comments, assignments, state transitions, workflows results) to support compliance audits and rollbacks.

# Artifact Taxonomy & Ownership
- **Repository ownership**: A graph is the authoritative home for a tenant’s artifact catalog. Each artifact belongs to exactly one primary graph (e.g., `reference-architecture`), yet can appear in multiple repositories as linked references via assignments and smart folders. Occupancy is tracked via `graph_id` + `artifact_id` pairs, ensuring lineage back to the owning graph.
- **Artifact kinds**: Supported kinds include (non-exhaustive):
  - `bpmn::process`, `bpmn::collaboration`, `bpmn::decision`
  - `archimate::capability`, `archimate::application_component`, `archimate::value_stream`
  - `uml::class_diagram`, `uml::sequence_diagram`
  - `dmn::decision_table`
  - `doc::policy`, `doc::standard`, `doc::playbook`
  - `code::api_contract`, `code::infrastructure_module`
  - `strategy::transformation_brief`, `strategy::roadmap`
- **Metadata envelopes**: Each artifact kind carries standardized metadata: identity (`artifact_id`, `graph_id`, `kind`), lifecycle state, owners, tags (e.g., architectural layer, governance phase), linked transformation transformations (for proposed/WIP context), and cross-references to other artifacts. The metadata schema follows the `ArtifactView` contract in `packages/ameide_core_proto/src/ameide_core_proto/artifacts/v1`.
- **Behavioral contracts**: Artifact kind determines command validation (e.g., BPMN schema validation vs. Markdown lint), preview rendering, and automation hooks. Repositories must store allowed kinds per node (`NodeType.allowed_artifact_kinds`) to enforce taxonomy rules.
- **Initiative links**: Transformation transformations (see `backlog/161-transformation-transformations-functional.md`) maintain bidirectional labels to artifacts. Repositories expose APIs to fetch artifacts by transformation tag and to update transformation status when related artifacts change lifecycle state.

# Governance & Collaboration Enhancements
- **NodeOWNERS & approval flows**: Node-level owner declarations automatically assign reviewers when artifacts move to `in_review`. The UI surfaces owner chips, outstanding approvals, and segregation-of-duties exceptions.
- **Required checks dashboard**: Artifacts show a status banner summarizing required checks (policy evaluations, architecture linters, SAST scans). Users can re-run failed checks or drill into details before approvals.
- **Transition queue visibility**: Lifecycle actions enqueue transitions, and stakeholders see queue position, pending checks, and auto-promotion ETA. Queues also power batch promotions during release windows.
- **Releases tab**: Each artifact exposes a releases tab with immutable notes, downloadable assets (documents, exports, SBOMs), and protected semantic tags.
- **Discussions & ADRs**: Discussions support Q&A, decision threads, and ADR templates. Accepted ADRs link to specific graph nodes and surface in transformation dashboards.
- **Time travel & scenarios**: Repository surfaces effective windows on assignments; transformation workspaces provide the “As-is / To-be / Scenario” toggles and roadmap visualizations (see `backlog/161-transformation-transformations-functional.md`).
- **Initiative integrations**: Repository exposes smart folders, APIs, and governance events that transformation boards/roadmaps consume for lifecycle status, risks, and dependencies.
- **Survey prompts & data quality**: Owners receive survey prompts when required metadata is stale; data quality scores contribute to governance KPIs and can block promotions if thresholds drop.
- **Tenant governance policies**: Governance rules (approval thresholds, required checks, tag protections) are defined at graph/tenant scope. Transformation transformations inherit these policies; transformations may add context-specific visibility but cannot weaken graph-level guardrails.
- **Governance explainability**: Ruleset Inspector panels surface matched policies, required checks, and ruleset version/hash; “Request missing owners” shortcuts auto-assign outstanding reviewers; saved governance views (“Failing checks”, “Needs owners”, “Queued for approval”) behave like smart folders. Governance policies are managed once at graph/tenant layer and transformations consume the outcomes.

# UI Architecture
## Overview
The enterprise graph experience is delivered as a multi-tenant Next.js 15 SPA (React 19, shadcn/ui) consuming gRPC/ConnectRPC services through the `@ameideio/ameide-sdk-ts` client. Repository and transformation modules share the platform shell, design system, and provider stack described earlier. This section captures how the UI implements the functional behaviors above.

## Existing Platform Context
- **App shell**: `app/(app)/layout.tsx` wires `Header`, `AppSidebar`, `SearchProvider`, `TooltipProvider`, `DataStreamProvider`, `UrlSyncGate`, and `DebugProvider`. Repository/transformation routes integrate with this shell instead of redefining providers.
- **Header**: `features/navigation/components/Header.tsx` renders breadcrumbs, scope tabs, notifications, and search. Routes register descriptors via `resolveHeaderDescriptor` to expose relevant tabs/actions.
- **Sidebar**: `features/navigation/components/AppSidebar.tsx` supplies navigation, history, and quick-create modals (e.g., `ArtifactRepositoryBrowser`). Repository and transformation views extend these menus rather than creating parallel sidebars.
- **Providers & telemetry**: Hooks like `useDebug`, `useSearch`, `useSidebar` plus observability utilities are reused. Additional context (Realtime, Scenario state) is composed under `DataStreamProvider` for consistent lifecycle handling.
- **Design system**: shadcn/ui primitives, Lucide icons, and Tailwind tokens remain the styling baseline. Repository-specific components (timeline, governance banners, releases) live in the shared component library for reuse across modules.
- **Routing**: Repository paths `/graph/:graphId` and transformation paths `/transformations/:transformationId` align with `UrlSyncGate` and analytics expectations.

## Component Topology (Repository-Focused)
```
====================================================================================================================
| AppShell (app/(app)/layout.tsx)                                                                                  |
| ---------------------------------------------------------------------------------------------------------------- |
|  Providers: DebugProvider → TooltipProvider → SearchProvider → DataStreamProvider → UrlSyncGate → SDKClient      |
|  Shared UI: Header (breadcrumbs, scope tabs, actions) • AppSidebar (nav, history, quick-create modals)           |
|  Main Content: Routed area for Repository Browser, Governance Dashboards, Artifact Editor Host                   |
====================================================================================================================
          │
          │ layout.tsx renders children here (with --app-header-h spacing)
          ▼
+------------------------------------------------------------------------------------------------------------------+
| Repository Browser ( /graph/:graphId )                                                                 |
|------------------------------------------------------------------------------------------------------------------|
|  ┌────────── Tree Pane ──────────┐  ┌────────────── Main Pane ──────────────┐  ┌───── Governance Sidebar ─────┐ |
|  │ • Virtualized tree            │  │ ┌────────── Assignment List ───────┐ │  │ • NodeOWNERS card             │ |
|  │ • Breadcrumb trail            │  │ │ • Filters: state/tags/init       │ │  │ • Approvals progress          │ |
|  │ • NodeOWNERS badges           │  │ │ • Inline pills (lifecycle/check) │ │  │ • Required checks banner       │ |
|  │ • Drag/drop reorder           │  │ │ • Bulk actions (move/review)     │ │  │ • Queue status + ETA          │ |
|  │ • Filters (type/state)        │  │ │ • Saved views (Failing/Needs/Queued) ││  │ • Data quality score           │ |
|  │ • Effective window indicators │  │ └──────────────────────────────────┘ │  │ • Ruleset inspector link       │ |
|  └───────────────────────────────┘  │ ┌──────────── Artifact Drawer ─────────────┐│                             │ |
                                       │ │ Tabs: Details | Releases | Discussions | ││                             │ |
                                       │ │       Timeline                            ││                             │ |
                                       │ │ • Details → metadata, lifecycle, tags     ││                             │ |
                                       │ │ • Releases → versions, assets, tags       ││                             │ |
                                       │ │ • Discussions → ADR threads, comments     ││                             │ |
                                       │ │ • Timeline → events (assign/check/approve ││                             │ |
                                       │ │ • Actions → request review, approve, etc. ││                             │ |
                                       │ │ • Effective window summary (read-only)    ││                             │ |
                                       │ └───────────────────────────────────────────┘│                             │ |
+------------------------------------------------------------------------------------------------------------------+
          │ React Query cache keys: repositories, nodes, assignments, artifactView, owners, checks, queue, releases |
          │ Invalidation via SDK mutations + SSE events + Temporal callbacks                                      |
          ▼
+------------------------------------------------------------------------------------------------------------------+
| Governance Dashboards ( /dashboard → Governance tab )                                                            |
|------------------------------------------------------------------------------------------------------------------|
|  • KPI tiles for adoption, coverage, check success, queue latency, ruleset compliance, survey completion         |
|  • Charts and drilldowns for overdue approvals, failing checks, low data quality nodes                           |
|  • Reuses existing dashboard card/grid patterns                                                                  |
+------------------------------------------------------------------------------------------------------------------+
          │
          ▼
+------------------------------------------------------------------------------------------------------------------+
| Artifact Editor Host ( /artifact/:artifactId )                                                                   |
|------------------------------------------------------------------------------------------------------------------|
|  • Embeds domain editors (BPMN, ArchiMate, Markdown) with shared command bus                                     |
|  • Lifecycle toolbar (state, request review, approve, queue indicator)                                           |
|  • Required checks panel, NodeOWNER info, links back to graph nodes                                         |
|  • Yjs presence for collaborative editing; timeline of edits aligned with graph events                      |
+------------------------------------------------------------------------------------------------------------------+

Legend:
- Outer shell uses existing platform Header/Sidebar providers.
- Repository view includes tree, main pane, and governance sidebar with rich interaction.
- Governance dashboards and artifact editor reuse the same shell; transformation workspace topology lives in the transformation transformation UI spec.
```

## Data Flow & State Management
- **Server interactions**: `RepositoryTreeService.GetTree`, `RepositoryAssignmentService.ListNodeAssignments`, lifecycle command helpers, `GovernanceService` (owners/rulesets), `CheckRunService`, releases/discussions services.
- **React Query caches**: repositories, nodes, assignments, artifactView, governance owners/checks, queue status, releases, discussions; keys scoped by tenant+graph; invalidate via mutations and SSE/WebSocket events.
- **Local state**: view mode (`tree|list|grid`), artifact selection, filters, timeline filters, scenario badges stored in component state or lightweight stores for cross-pane updates.
- **Realtime updates**: SSE/WebSockets push `graph.assignment.*`, `governance.check_run.*`, `graph.transition.*`; optimistic cache updates follow. Yjs manages presence inside collaborative editors.

## Module Responsibilities
- **Repository Browser**: Tree pane (hierarchy, owners, drag/drop); assignment list/grid (filters, saved views, lifecycle chips); artifact drawer (details, releases, discussions, timeline, actions, effective window summary); governance sidebar (approvals, required checks, queue status, data quality); transformation cross-links.
- **Initiative Workspace**: Kanban/table/roadmap views driven by graph assignments; cards surface role assignments, governance status (checks, approvals, queue position), risks, upcoming releases; scenario overlays and As-is/To-be toggles leverage assignment effective windows.
- **Governance Dashboards**: KPIs (adoption, coverage, check success, queue latency, ruleset compliance) and drilldowns (overdue approvals, failing checks, low data quality) reuse platform dashboard components.
- **Artifact Editor Host**: Embeds domain editors with lifecycle toolbar, governance panels, and navigation back to graph nodes via the shared command bus (see backlog/120).

## UI Patterns & Components
- Tree & drag-and-drop (React Virtuoso), tabs/drawers (shadcn/ui), timeline (events with filters), discussions/ADR editors (Lexical/Tiptap), releases list, required-check banners, queue indicator, sidebar integration, Ruleset Inspector slide-over, saved governance views as smart folders.

## Governance UX Considerations
- Automatic reviewer assignment from NodeOWNERS; approval dialogs highlight required roles and segregation warnings.
- Ruleset Inspector communicates blocking policies (version/hash, matched rules, required checks) and remediation guidance.
- Data-quality prompts trigger surveys; completion updates scores. Accessibility (keyboard navigation, ARIA labels, color-safe palettes) remains mandatory.

## Performance & Offline Strategy
- Virtualized lists, cached queries, incremental pagination (`infiniteQuery`); prefetch sibling nodes when expanding tree items to keep perceived latency <100 ms.
- Use Web Workers for graph impact analysis; gracefully fall back to polling when SSE/WebSockets unavailable; coordinate skeleton states with `--app-header-h` to avoid layout shift.

## Telemetry & Instrumentation
- Track node navigation, assignment search, lifecycle transition attempts, check reruns, release downloads, discussion posts.
- Monitor TTFB, TTI, tree render time, assignment grid render, timeline update latency, approval cycle time, check resolution, queue wait time.

## Feature Flagging & Rollout
- Feature flags by stage (`repo_lifecycle`, `repo_governance`, `repo_releases`, `repo_time_travel`) gate features per tenant. Roll out gradually (internal → pilot → GA) while collecting adoption metrics.
- Components honor flags to hide/show panels without redeployments.

## Cross-Cutting Concerns
- Localization readiness, tenant theming via design tokens, RBAC-aware rendering, sanitization for user inputs, component/integration testing (Playwright, Vitest, Chromatic).

## Roadmap Alignment
- Stage mapping:
  1. Repository browser core (tree/assignment UI)
  2. Lifecycle controls, timeline basics, watchers
  3. Governance sidebar, required-check banners, queue indicators
  4. Releases tab, discussions, survey prompts
  5. Initiative time-travel controls, scenario overlays, role-aware boards/roadmaps
  6+. Notifications, external webhooks visualization, AI-assisted suggestions

## Next Steps
- Register graph and transformation routes with contextual navigation descriptors so the header renders correct breadcrumbs/tabs.
- Update sidebar navigation to include graph entry points alongside modal behaviors (e.g., `ArtifactRepositoryBrowser`).
- Catalogue shared components (timeline, releases list, governance banners) and add UI kit stories/tests.

# Success Metrics
| Metric | Target | Persona | Notes |
| --- | --- | --- | --- |
| Repository adoption | ≥ 75% of active tenants create ≥1 graph within 90 days | EA | Track via `repositories` table by tenant. |
| Assignment coverage | ≥ 60% of governance-critical nodes contain ≥1 approved artifact | SA/GL | Use `graph_assignments` counts filtered by `role='primary'` and status. |
| Search efficiency | < 2.5 s P95 for cross-graph searches | DEV | Measure `RepositoryQueryService` latency with telemetry; ensure indexed queries. |
| Governance compliance | 100% of pinned artifacts have explicit version metadata | GL | Validate via nightly audit job scanning `pinned_version` columns. |
| User satisfaction | ≥ 4.2/5 CSAT for graph UI interactions | All | Collect from in-app surveys post key workflows. |

# Decisions & Proposed Approaches
- **Template source & customization**: Manage baseline templates in configuration (`graph.config` JSON) stored alongside code, versioned via migrations. Allow tenants to clone templates into tenant-specific configs persisted in `repositories.config`, and track divergence via template version metadata. Migration tooling should provide diff-and-merge when baseline templates evolve.
- **Smart-folder expression language**: Launch with a constrained JSON filter schema (field + operator + value) validated server-side to avoid arbitrary CEL risks. Roadmap CEL support behind a feature flag once governance validation and sandboxing are in place.
- **Soft delete exposure**: Extend `BaseEntity` with `deleted_at`/`deleted_by` fields and add `List...` filters to include/exclude soft-deleted rows; surface delete endpoints as soft delete by default, with hard delete restricted to admin tooling and captured via audit events.
- **RBAC model**: Integrate with tenant IAM where graph permissions are granted through roles (`graph.admin`, `graph.editor`, `graph.viewer`). Editors manage nodes and assignments; admins manage templates and lifecycle; viewers have read access. Enforce via request context before invoking storage operations.
- **Change notifications & workflows**: Emit domain events (e.g., `graph.assignment.created`) to the platform event bus so downstream services (notifications, workflows automation) can subscribe. In the short term, integrate with existing notification service to alert stakeholders when governance KPIs breach thresholds.

---

## 152-enterprise-repository-information-architecture.md

# ER Diagram
```mermaid
erDiagram
    Tenant ||--o{ Repository : "1..N (logical)"
    Repository {
        string id PK
        string tenant_id FK
        string key
        string name
        text description
        string template_id
        jsonb attributes
        jsonb config
        timestamptz created_at
        timestamptz updated_at
        int version
    }
    Repository ||--o{ RepositoryNode : "1..N"
    RepositoryNode {
        string id PK
        string tenant_id FK
        string graph_id FK
        string parent_id FK
        string type_key
        string name
        string slug
        text path
        int order
        boolean is_virtual
        text filter_expression
        jsonb attributes
        timestamptz created_at
        timestamptz updated_at
        int version
    }
    RepositoryNode ||--o{ RepositoryNode : "1..N self"
    RepositoryNode ||--o{ RepositoryAssignment : "1..N"
    RepositoryAssignment {
        string id PK
        string tenant_id FK
        string graph_id FK
        string node_id FK
        string artifact_id
        string artifact_type
        int pinned_version
        string role
        int order
        jsonb attributes
        timestamptz created_at
        timestamptz updated_at
        int version
    }
```

# Schema DDL
The Alembic migration `migrations/versions/platform/010_graph.py` provisions the graph schema. Field sizes align with SQLAlchemy model metadata in `packages/ameide_core_storage/src/ameide_core_storage/sql/models/graph.py`.

## repositories
```sql
CREATE TABLE repositories (
    id           VARCHAR(255) PRIMARY KEY,
    tenant_id    VARCHAR(255) NOT NULL,
    deleted_at   TIMESTAMPTZ,
    deleted_by   VARCHAR(255),
    legal_hold   BOOLEAN      NOT NULL DEFAULT FALSE,
    created_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at   TIMESTAMPTZ,
    version      INTEGER      NOT NULL DEFAULT 0,
    key          VARCHAR(128) NOT NULL,
    name         VARCHAR(255) NOT NULL,
    description  TEXT,
    template_id  VARCHAR(255),
    attributes   JSONB,
    config       JSONB
);
```

## graph_nodes
```sql
CREATE TABLE graph_nodes (
    id                VARCHAR(255) PRIMARY KEY,
    tenant_id         VARCHAR(255) NOT NULL,
    deleted_at        TIMESTAMPTZ,
    deleted_by        VARCHAR(255),
    legal_hold        BOOLEAN      NOT NULL DEFAULT FALSE,
    created_at        TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ,
    version           INTEGER      NOT NULL DEFAULT 0,
    graph_id     VARCHAR(255) NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    parent_id         VARCHAR(255) REFERENCES graph_nodes(id) ON DELETE CASCADE,
    type_key          VARCHAR(128) NOT NULL,
    name              VARCHAR(255) NOT NULL,
    slug              VARCHAR(128),
    path              VARCHAR(2048),
    path_ltree        LTREE,
    "order"           INTEGER      NOT NULL DEFAULT 0,
    is_virtual        BOOLEAN      NOT NULL DEFAULT FALSE,
    filter_expression TEXT,
    attributes        JSONB
);
-- Requires `CREATE EXTENSION IF NOT EXISTS ltree;` in the migration.
```

## graph_assignments
```sql
CREATE TABLE graph_assignments (
    id             VARCHAR(255) PRIMARY KEY,
    tenant_id      VARCHAR(255) NOT NULL,
    deleted_at     TIMESTAMPTZ,
    deleted_by     VARCHAR(255),
    legal_hold     BOOLEAN      NOT NULL DEFAULT FALSE,
    created_at     TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at     TIMESTAMPTZ,
    version        INTEGER      NOT NULL DEFAULT 0,
    graph_id  VARCHAR(255) NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    node_id        VARCHAR(255) NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    artifact_id    VARCHAR(255) NOT NULL,
    artifact_type  VARCHAR(255) NOT NULL,
    pinned_version INTEGER,
    effective_from TIMESTAMPTZ,
    effective_to   TIMESTAMPTZ,
    scenario_tag   VARCHAR(64),
    role           VARCHAR(64),
    "order"        INTEGER      NOT NULL DEFAULT 0,
    attributes     JSONB
);
ALTER TABLE graph_assignments
  ADD CONSTRAINT uq_assign_per_node UNIQUE (graph_id, node_id, artifact_id);
ALTER TABLE graph_assignments
  ADD CONSTRAINT chk_pinned_positive CHECK (pinned_version IS NULL OR pinned_version > 0);
ALTER TABLE graph_assignments
  ADD CONSTRAINT chk_artifact_type_not_empty CHECK (length(artifact_type) > 0);
ALTER TABLE graph_assignments
  ADD CONSTRAINT chk_assignment_effective
    CHECK (
      (effective_from IS NULL AND effective_to IS NULL)
      OR (effective_from IS NOT NULL AND (effective_to IS NULL OR effective_from < effective_to))
    );
-- Optional: requires btree_gist extension
-- ALTER TABLE graph_assignments
--   ADD CONSTRAINT ex_assignment_window
--   EXCLUDE USING gist (
--     graph_id WITH =,
--     node_id       WITH =,
--     artifact_id   WITH =,
--     COALESCE(scenario_tag,'') WITH =,
--     tstzrange(effective_from, effective_to) WITH &&
--   ) WHERE (effective_from IS NOT NULL OR effective_to IS NOT NULL);
```

# Governance & Collaboration Tables

## graph_node_owners
```sql
CREATE TABLE graph_node_owners (
    id             VARCHAR(255) PRIMARY KEY,
    tenant_id      VARCHAR(255) NOT NULL,
    graph_id  VARCHAR(255) NOT NULL REFERENCES repositories(id) ON DELETE CASCADE,
    node_id        VARCHAR(255) NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    principal_id   VARCHAR(255) NOT NULL,
    principal_type VARCHAR(32)  NOT NULL, -- user, group, team, service
    pattern        VARCHAR(255),
    created_at     TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    attributes     JSONB
);
CREATE INDEX idx_node_owner_node ON graph_node_owners (graph_id, node_id);
```

## approval_rules
```sql
CREATE TABLE approval_rules (
    id                    VARCHAR(255) PRIMARY KEY,
    tenant_id             VARCHAR(255) NOT NULL,
    scope_type            VARCHAR(32)  NOT NULL, -- graph, node, template
    scope_id              VARCHAR(255) NOT NULL,
    min_approvals         INTEGER      NOT NULL DEFAULT 1,
    require_owners        BOOLEAN      NOT NULL DEFAULT TRUE,
    allow_self_approval   BOOLEAN      NOT NULL DEFAULT FALSE,
    segregation_rules     JSONB,
    created_at            TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    created_by            VARCHAR(255),
    version               INTEGER      NOT NULL DEFAULT 0
);
CREATE INDEX idx_approval_scope ON approval_rules (scope_type, scope_id);
```

## required_checks & check_runs
```sql
CREATE TABLE required_checks (
    id          VARCHAR(255) PRIMARY KEY,
    tenant_id   VARCHAR(255) NOT NULL,
    scope_type  VARCHAR(32)  NOT NULL, -- graph, node, ruleset
    scope_id    VARCHAR(255) NOT NULL,
    name        VARCHAR(255) NOT NULL,
    registry_id VARCHAR(255),
    run_on      VARCHAR(32)  NOT NULL, -- assign, transition, periodic
    attributes  JSONB,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);
ALTER TABLE required_checks
  ADD CONSTRAINT fk_required_checks_registry
  FOREIGN KEY (registry_id) REFERENCES check_registry(id);

CREATE TABLE check_runs (
    id            VARCHAR(255) PRIMARY KEY,
    tenant_id     VARCHAR(255) NOT NULL,
    subject_type  VARCHAR(32)  NOT NULL, -- artifact, assignment, node
    subject_id    VARCHAR(255) NOT NULL,
    name          VARCHAR(255) NOT NULL,
    status        VARCHAR(32)  NOT NULL, -- queued, in_progress, completed
    conclusion    VARCHAR(32),
    details_url   TEXT,
    payload_json  JSONB,
    started_at    TIMESTAMPTZ,
    completed_at  TIMESTAMPTZ,
    created_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_check_runs_subject ON check_runs (subject_type, subject_id);
```

## check_registry
```sql
CREATE TABLE check_registry (
    id          VARCHAR(255) PRIMARY KEY,
    tenant_id   VARCHAR(255) NOT NULL,
    name        VARCHAR(255) NOT NULL UNIQUE,
    source      VARCHAR(32)  NOT NULL, -- opa | ci | manual | external
    description TEXT,
    metadata    JSONB,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    retired_at  TIMESTAMPTZ
);
```

## transition_queue
```sql
CREATE TABLE transition_queue (
    id             VARCHAR(255) PRIMARY KEY,
    tenant_id      VARCHAR(255) NOT NULL,
    graph_id  VARCHAR(255) NOT NULL,
    artifact_id    VARCHAR(255) NOT NULL,
    node_id        VARCHAR(255),
    desired_state  VARCHAR(32)  NOT NULL,
    enqueued_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    status         VARCHAR(32)  NOT NULL DEFAULT 'queued',
    priority       INTEGER      NOT NULL DEFAULT 0,
    attempts       INTEGER      NOT NULL DEFAULT 0,
    last_attempt_at TIMESTAMPTZ,
    idempotency_key VARCHAR(255) NOT NULL,
    metadata       JSONB
);
CREATE INDEX idx_transition_queue_repo ON transition_queue (graph_id, status);
CREATE UNIQUE INDEX uq_transition_request
  ON transition_queue (graph_id, artifact_id, desired_state)
  WHERE status IN ('queued','in_progress');
CREATE UNIQUE INDEX uq_transition_idempotency ON transition_queue (idempotency_key);
```

## artifact_releases & release_assets
```sql
CREATE TABLE artifact_releases (
    id            VARCHAR(255) PRIMARY KEY,
    tenant_id     VARCHAR(255) NOT NULL,
    artifact_id   VARCHAR(255) NOT NULL,
    version       INTEGER      NOT NULL,
    notes         TEXT,
    is_immutable  BOOLEAN      NOT NULL DEFAULT TRUE,
    created_by    VARCHAR(255),
    created_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    metadata      JSONB
);
CREATE INDEX idx_releases_artifact ON artifact_releases (artifact_id, version);

CREATE TABLE release_assets (
    id             VARCHAR(255) PRIMARY KEY,
    release_id     VARCHAR(255) NOT NULL REFERENCES artifact_releases(id) ON DELETE CASCADE,
    name           VARCHAR(255) NOT NULL,
    content_type   VARCHAR(255),
    storage_uri    TEXT NOT NULL,
    sha256         VARCHAR(128),
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## graph_rulesets
```sql
CREATE TABLE graph_rulesets (
    id         VARCHAR(255) PRIMARY KEY,
    tenant_id  VARCHAR(255) NOT NULL,
    scope      VARCHAR(32)  NOT NULL, -- tenant, template, graph
    scope_id   VARCHAR(255),
    status     VARCHAR(32)  NOT NULL DEFAULT 'active',
    def_json   JSONB        NOT NULL,
    created_at TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    version    INTEGER      NOT NULL DEFAULT 1
);
CREATE INDEX idx_ruleset_scope ON graph_rulesets (scope, scope_id, status);
```

## discussions & posts
```sql
CREATE TABLE discussions (
    id            VARCHAR(255) PRIMARY KEY,
    tenant_id     VARCHAR(255) NOT NULL,
    subject_type  VARCHAR(32)  NOT NULL, -- artifact, node, transformation
    subject_id    VARCHAR(255) NOT NULL,
    title         VARCHAR(255) NOT NULL,
    body          TEXT,
    state         VARCHAR(32)  NOT NULL DEFAULT 'open',
    created_by    VARCHAR(255),
    created_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    attributes    JSONB
);

CREATE TABLE discussion_comments (
    id             VARCHAR(255) PRIMARY KEY,
    discussion_id  VARCHAR(255) NOT NULL REFERENCES discussions(id) ON DELETE CASCADE,
    body           TEXT NOT NULL,
    created_by     VARCHAR(255),
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    attributes     JSONB
);
```

## model_edges
```sql
CREATE TABLE model_edges (
    id             VARCHAR(255) PRIMARY KEY,
    tenant_id      VARCHAR(255) NOT NULL,
    src_type       VARCHAR(32)  NOT NULL,
    src_id         VARCHAR(255) NOT NULL,
    relationship   VARCHAR(64)  NOT NULL,
    dst_type       VARCHAR(32)  NOT NULL,
    dst_id         VARCHAR(255) NOT NULL,
    attributes     JSONB,
    effective_from TIMESTAMPTZ,
    effective_to   TIMESTAMPTZ
);
CREATE INDEX idx_edges_src ON model_edges (src_type, src_id);
CREATE INDEX idx_edges_dst ON model_edges (dst_type, dst_id);
```

## surveys & responses
```sql
CREATE TABLE metadata_surveys (
    id            VARCHAR(255) PRIMARY KEY,
    tenant_id     VARCHAR(255) NOT NULL,
    subject_type  VARCHAR(32)  NOT NULL,
    subject_id    VARCHAR(255) NOT NULL,
    schema_json   JSONB        NOT NULL,
    state         VARCHAR(32)  NOT NULL DEFAULT 'open',
    created_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    due_at        TIMESTAMPTZ,
    created_by    VARCHAR(255)
);

CREATE TABLE survey_responses (
    id             VARCHAR(255) PRIMARY KEY,
    survey_id      VARCHAR(255) NOT NULL REFERENCES metadata_surveys(id) ON DELETE CASCADE,
    responder_id   VARCHAR(255) NOT NULL,
    payload_json   JSONB        NOT NULL,
    submitted_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);
```

# Indices & Constraints
- `repositories`: `uq_repo_tenant_key` enforces unique `(tenant_id, key)`; `idx_repo_tenant_name` accelerates lookups by tenant and name. Soft-delete columns (`deleted_at`, `deleted_by`, `legal_hold`) support tombstoning and legal hold workflows.
- `graph_nodes`: `idx_node_repo_parent` supports child listings; `idx_node_repo_path` and the optional `idx_node_repo_path_ltree` (requires `ltree`) optimize subtree queries; `idx_node_tenant_type` and `idx_node_repo_type` enable filtering by type. Path updates must run in a single transaction to keep `path`/`path_ltree` aligned.
- `graph_assignments`: `idx_assign_repo_node` and `idx_assign_artifact` accelerate navigation; `idx_assign_repo_type_state` supports artifact-type filtering. `uq_assign_per_node` (graph, node, artifact) prevents duplicates; `chk_pinned_positive` and `chk_artifact_type_not_empty` enforce data hygiene. When `btree_gist` is available, the optional exclusion constraint on `tstzrange(effective_from, effective_to)` blocks overlapping time windows per scenario.
- `required_checks`: `chk_check_names_nonempty` ensures non-empty names; `registry_id` ties checks to `check_registry`, preventing drift.
- `check_registry`: unique constraint on `name` prevents duplicate registrations and enables soft retirement (`retired_at`).
- `transition_queue`: `idx_transition_queue_repo` feeds per-graph workers; `uq_transition_request` (queued/in-progress) avoids duplicate queue entries; `uq_transition_idempotency` enforces global idempotency per request.
- `check_runs`: subject index supports timeline retrieval; store `started_at`/`completed_at` for SLA analysis.
- All tables inherit optimistic locking via the `version` column (`__mapper_args__` in `BaseEntity`) and capture timestamps for auditing.
- Extensions: migrations should enable `ltree` and `btree_gist` where supported to back subtree queries and time-window exclusion constraints.

# Data Patterns
- **Materialized paths**: `graph_nodes.path` stores a dot-delimited lineage; `NodeRepo.compute_path` recomputes descendants during moves. `path_ltree` mirrors `path` for efficient subtree queries using GIST indexes.
- **Adjacency plus hierarchy**: Combining `parent_id`, `path`, and `path_ltree` allows simple CRUD with performant reads. Redis caches invalidate by path prefix whenever nodes move.
- **Virtual nodes (smart folders)**: `is_virtual + filter_expression` provide saved queries; enforcement occurs in services using ArtifactView filters.
- **Optimistic concurrency**: `version` columns across entities ensure API consumers detect conflicting writes.
- **Soft delete & legal hold**: `deleted_at`, `deleted_by`, and `legal_hold` enable reversible deletes, legal retention, and audit trails; default queries exclude soft-deleted rows.
- **Artifact ownership**: Repositories remain the system of record; assignments reference artifacts by ID/type while respecting per-node allowed kinds.
- **Time-scoped assignments**: `effective_from`, `effective_to`, and `scenario_tag` deliver first-class roadmap support. Exclusion constraints (with `btree_gist`) prevent overlapping windows; scenario tags let tenants compare future plans. Transformation transformations reference these canonical fields via assignment identifiers rather than duplicating the data in transformation tables.
- **Initiative tagging**: Store transformation links in assignment attributes or dedicated tables to power transformation boards; updates propagate to transformation transformation services.
- **Ownership resolution cache**: Combine `graph_node_owners` inheritance with graph-level defaults to fetch approvers quickly; cached outputs feed governance sidebar.
- **Ruleset inheritance**: `graph_rulesets` define approvals, checks, tag protections. Transitions log ruleset `id+version+hash` to guarantee provenance.
- **Canonical check registry**: `check_registry` centralizes allowed check names/sources. `required_checks.registry_id` must reference active entries; retiring a check prevents new associations.
- **Check run lineage**: `check_runs` store workflows correlation IDs, details URLs, and payloads for audit. Failed runs tie back to governance dashboards.
- **Transition queue semantics**: Queue entries record desired state, priority, attempts, last attempt timestamp, idempotency key, and metadata for diagnostics. Per-graph workers process entries FIFO while respecting priority and idempotency guards, with dead-letter handling for exceeded attempts.
- **Release immutability**: `artifact_releases` + `release_assets` ensure versioned artifacts remain immutable; SHA-256 hashes verify downloads; protected tags managed via rulesets.
- **Graph edges & impact analysis**: `model_edges` represent relationships with effective windows. Event consumers refresh edges to support impact queries.
- **Survey lifecycle**: `metadata_surveys` gather missing data; responses adjust assignment attributes and data quality scores; governance dashboards highlight outstanding surveys.
- **Database isolation**: PostgreSQL Row-Level Security (optional) enforces tenant boundaries at the database layer, complementing application-level guards.

# Multi-Repository Support
- **Tenant isolation**: Application-level repositories (`TenantRepository` in `packages/ameide_core_storage/.../tenant_graph.py`) inject tenant scopes into every query, preventing cross-tenant leakage even without database-level tenant FKs.
- **Key uniqueness per tenant**: `(tenant_id, key)` uniqueness lets tenants host multiple repositories (ADM cycle, Standards, Reference Architecture) with predictable slugs while avoiding collisions across tenants.
- **Cascade semantics**: `ON DELETE CASCADE` on `graph_id` and `parent_id` keeps hierarchy integrity when repositories or branches are removed. Assignments follow nodes automatically, avoiding orphaned rows.
- **Cross-graph references**: `assignment.artifact_id` intentionally omits an FK so that artifacts managed by other bounded contexts (Archimate, BPMN, documents) can be linked without circular migrations. The combination of `artifact_id`, `artifact_type`, and optional `pinned_version` enables cross-graph reuse with deterministic version pinning.
- **Admin operations**: `TenantRepository.get_cross_tenant` and `.migrate_to_tenant` enable support for managed service operations (e.g., moving a graph to a new tenant) while default services enforce isolation.
- **Row-level security (optional)**: PostgreSQL RLS policies can be enabled per tenant (`app.tenant_id`) for regulated deployments; application-level guards remain to provide defense in depth.
- **Soft delete defaults**: Services filter on `deleted_at IS NULL` by default; purge and legal-hold operations are limited to governance administrators and emit explicit audit events.

# Sample Data Examples
- **Seed graph (Alembic 011_graph_seed)**:
  | Entity | ID | Key Fields |
  |--------|----|-----------|
  | Repository | `repo-demo` | tenant `demo-tenant`, key `default`, name `Default ADM Repository` |
  | Node | `n-cycle-2024` | root folder `ADM Cycle 2024`, path `n-cycle-2024`, order `0` |
  | Node | `n-phase-a` | child of `n-cycle-2024`, type `phase`, slug `phase-a`, path `n-cycle-2024.n-phase-a` |
  | Assignment | `a-seed-1..3` | node `n-phase-a`, artifacts `artifact-1..3`, types `doc::document`, `archimate::capability`, `bpmn::process` |

- **Smart folder filter expression**:
  ```json
  {
    "is_virtual": true,
    "filter_expression": "artifact.phase == 'A' && artifact.layer == 'Business'",
    "attributes": {
      "adm_phase": "A",
      "archimate_layer": "Business"
    }
  }
  ```
  Store alongside a node to surface all business-layer artifacts tagged to ADM Phase A, regardless of their physical placement.

- **Version-pinned assignment**:
  ```json
  {
    "artifact_id": "arch-vision-2025",
    "artifact_type": "archimate::architecture_vision",
    "pinned_version": 7,
    "effective_from": "2025-01-01T00:00:00Z",
    "scenario_tag": "baseline",
    "role": "primary",
    "order": 0
  }
  ```
  Ensures graph reviews always open the approved version—even if the artifact aggregate advances.

- **Cross-graph reference map**:
  Using the optional `config` JSON, tenants can declare graph types:
  ```json
  {
    "graph_type": "standards-catalog",
    "default_tree": [
      {"type_key": "standard", "name": "Security Standards"},
      {"type_key": "standard", "name": "Data Governance Standards"}
    ]
  }
  ```
  Services can inspect `config.graph_type` to apply tenant-specific business rules (e.g., only allow standards repositories to expose smart folders scoped by compliance domain).
- **Artifact metadata view**:
  Artifact services expose union views keyed by `artifact_id` and `artifact_type`. Sample representation:
  ```json
  {
    "artifact_id": "arch-vision-2025",
    "graph_id": "repo-reference-arch",
    "kind": "archimate::capability",
    "lifecycle_state": "approved",
    "owners": ["emma.wilson"],
    "tags": ["archimate:business", "governance:phase-b"],
    "linked_transformations": ["init-supply-chain"],
    "last_revision": {
      "version": 7,
      "updated_at": "2025-02-14T09:28:00Z"
    }
  }
  ```
  Repository read models denormalize key fields (state, owners, tags) into assignment `attributes` for fast filtering while relying on the artifacts bounded context for authoritative content.
- **Check run timeline**:
  ```json
  [
    {
      "name": "policy:security-controls",
      "status": "completed",
      "conclusion": "success",
      "detailsUrl": "https://governance.internal/runs/abc",
      "completedAt": "2025-02-15T10:02:00Z"
    },
    {
      "name": "lint:bpmn-schema",
      "status": "completed",
      "conclusion": "failure",
      "detailsUrl": "https://ci.example/job/123",
      "payload": {"validationErrors": ["Missing lane owner"]}
    }
  ]
  ```
- **Ruleset definition snippet**:
  ```json
  {
    "name": "standards-catalog-default",
    "approvalRules": {
      "minApprovals": 2,
      "requireOwners": true,
      "segregationOfDuties": {"blockSelfApproval": true}
    },
    "requiredChecks": [
      {"name": "policy:compliance", "runOn": "transition"},
      {"name": "security:vulnerability-scan", "runOn": "periodic"}
    ],
    "tagProtections": [{"pattern": "standards/*", "roles": ["graph.admin"]}]
  }
  ```
- **Transition queue entry**:
  ```json
  {
    "artifactId": "arch-vision-2025",
    "desiredState": "approved",
    "queuedAt": "2025-02-16T08:15:00Z",
    "requiredChecks": {"policy:compliance": "pending", "security:vulnerability-scan": "success"},
    "approvals": ["emma.wilson", "li.chen"],
    "rulesetVersion": 3,
    "priority": 10,
    "attempts": 2,
    "idempotencyKey": "req-8c1d23"
  }
  ```
- **Scenario overlay**:
  ```json
  {
    "artifact_id": "arch-vision-2025",
    "scenario_tag": "future-state-q3",
    "effective_from": "2025-07-01T00:00:00Z",
    "effective_to": "2025-09-30T23:59:59Z"
  }
  ```
  Supports transformation As-is / To-be toggles without graph traversal for simple filters.

---

## 153-enterprise-repository-technology-architecture.md

# Technology Matrix
| Layer | Choice | Rationale | Integration Notes |
| --- | --- | --- | --- |
| Backend runtime | Python 3.11 (FastAPI + gRPC via ConnectRPC) | Matches existing Ameide core services; async SQLAlchemy support | New `services/core_graph` service reuses shared middleware for auth, tenancy, tracing |
| Data store | PostgreSQL 15 | JSONB + materialized path queries; tenant-safe isolation | Alembic migrations enable `ltree` + `btree_gist`; optional Row-Level Security per tenant |
| Messaging | NATS JetStream / Kafka (existing event bus) | Artifact lifecycle & assignment events | Emit `graph.assignment.*`, `governance.check_run.*`, `graph.transition.*`, `ruleset.updated` |
| Object cache | Redis (clustered) | Read-model caching (tree, stats) and workflows throttling | Cache tree snapshots, NodeOWNER inheritance docs, and ruleset digests |
| Frontend | Next.js 15 + React 19 + shadcn/ui | Aligns with platform SPA; SSR where needed | `RepositoryBrowser` and transformation surfaces consume SDK clients |
| Client SDK | `@ameideio/ameide-sdk-ts` | Typed clients generated from protobuf; consistent error handling | Expand to include graph services + lifecycle commands |
| Collaboration | Yjs (optional, per artifact) | Real-time editing for text/code artifacts | Agents and users share command bus origins; integrates with `ArtifactView` updates |
| Workflow automation | Temporal / Argo Workflows (existing) | Enforce lifecycle gates & recertification jobs | `TransitionQueue` worker evaluates approvals/checks before committing |
| Policy engine | OPA / Rego bundles + check registry | Externalize governance checks | Bundles versioned per tenant; registry locks check names/sources referenced by rulesets |
| Graph analytics | Neo4j or PostgreSQL `model_edges` table | Impact analysis & scenario planning | Event consumers populate edges with effective windows |

# Architecture Overview
```mermaid
flowchart LR
    subgraph Frontend
        UI[Next.js Repository Browser]
        InitiativesUI[Transformation Initiatives UI]
    end
    subgraph SDK
        TSClient[@ameideio/ameide-sdk-ts]
    end
    subgraph Services
        RepoSvc[Repository Service<br/>(Admin/Tree/Assignment/Query)]
        GovSvc[Governance Service<br/>(Owners/Approvals/Rulesets)]
        CheckSvc[Check Run Service<br/>(OPA / CI / SAST orchestration)]
        QueueSvc[Transition Queue Worker<br/>(Temporal)]
        ArtifactSvc[Artifact Service<br/>(event-driven + CRDT hybrids)]
    end
    subgraph Data
        PG[(PostgreSQL Repository Schema)]
        Redis[(Redis Cache)]
    end
    subgraph Bus
        NATS[(Event Bus)]
    end
    UI --> TSClient --> RepoSvc
    InitiativesUI --> TSClient
    RepoSvc --> PG
    RepoSvc --> Redis
    RepoSvc <--> ArtifactSvc
    RepoSvc --> NATS
    GovSvc --> NATS
    CheckSvc --> NATS
    QueueSvc --> NATS
    ArtifactSvc --> NATS
    NATS --> GovSvc
    NATS --> CheckSvc
    NATS --> QueueSvc
    GovSvc --> RepoSvc
    CheckSvc --> RepoSvc
    QueueSvc --> RepoSvc
```

# Backend Architecture
- **Service boundaries**:
  - `RepositoryService` keeps Admin/Tree/Assignment/Query APIs generated from `packages/ameide_core_proto/src/ameide_core_proto/graph/v1`.
  - `GovernanceService` manages NodeOWNERS, approval rules, check registry, rulesets, and resolves approver sets per transition.
  - `CheckRunService` schedules and records required checks, invoking OPA, CI, SAST, or manual attestations.
  - `TransitionQueueWorker` (Temporal) serializes lifecycle transitions, revalidates approvals/checks, then commits state.
  - `ArtifactService` continues owning artifact storage and lifecycle events consumed by graph projections.
- **Command bus alignment**: Lifecycle commands flow through a shared schema (`backlog/120-artifacts-north-star.md`). Governance and queue services subscribe to command events, update state, and publish outcomes (`graph.transition.completed`).
- **Storage implementation**: Async SQLAlchemy repositories cover core tables plus governance tables (owners, approvals, checks, releases, discussions, rulesets, check registry). Soft-delete fields (`deleted_at`, `legal_hold`) are enforced by default scopes. Transactions wrap node moves (updating `path`/`path_ltree`), check run persistence, release publication, and queue operations with idempotency keys.
- **Read models**: Redis caches tree snapshots, NodeOWNER inheritance, ruleset digests, check dashboards, and queue heads. Optional graph projections populate `model_edges` or Neo4j for impact analysis.
- **Event publishing**: Services emit `graph.assignment.*`, `graph.node.moved`, `graph.template.applied`, `governance.node_owner.*`, `governance.check_run.*`, `graph.transition.*`, `ruleset.updated`, and forwarded `artifact.lifecycle.transitioned`. Events include tenant context, ruleset version, approval sets, and diff payloads.
- **Database isolation**: Support optional PostgreSQL Row-Level Security policies keyed by `app.tenant_id` to satisfy regulated tenants while retaining application-level tenancy checks.
- **Protocol policy**: Repository surfaces only ConnectRPC/gRPC APIs (and their SDK bindings). Aggregated GraphQL read endpoints exist solely in the Transformation Initiatives workspace; graph write paths remain gRPC-only. Downstream consumers (including transformations) access graph data via the SDK.

# API Design Considerations
- **Context propagation**: All gRPC requests include `RequestContext` (`tenant_id`, `user_id`, `request_id`, `impersonation`). Governance APIs return the `ruleset_version` used, enabling audit replay.
- **Pagination & filtering**: Query APIs accept pagination, include filters for node type, artifact kind, lifecycle, transformation tags, effective windows, scenario tags, ruleset, approval state, check status, and queue state. Smart folders persist structured filters server-side.
- **Validation & errors**: Tree mutations guard against invalid node configurations. Lifecycle actions emit `FAILED_PRECONDITION` for unmet required checks or approvals, `ABORTED` if queue revalidation fails, and `PERMISSION_DENIED` when RBAC or rulesets block the actor.
- **Idempotency & provenance**: Lifecycle commands carry idempotency keys so duplicate submitters don’t create duplicate queue items. Responses include ruleset `{id, version, hash}` and check registry references to satisfy audit replays.
- **Webhooks & subscriptions**: Tenants can register webhooks for lifecycle, governance, release, and discussion events. SSE/WebSocket channels stream activity timelines, check run updates, queue status, and discussion threads.
- **Graph traversal APIs**: `RepositoryQueryService` exposes endpoints for impact analysis using `model_edges`, supporting queries like “systems impacted by capability X” or “transformations referencing deprecated artifacts.”
- **Check registry APIs**: Governance service publishes CRUD endpoints to register, retire, and audit canonical check definitions; `required_checks` references registry IDs and rejects unknown names.

# Frontend Architecture
- **Component hierarchy**: `RepositoryBrowser` orchestrates tree/list/grid views, assignment panels, releases, discussions, and governance sidebars. React Query fetches tree snapshots, NodeOWNER/resolution docs, required check status, and queue position concurrently.
- **Artifact detail surface**: Artifact drawer shows lifecycle state, owners, approvals, required check banner, queue status, releases tab (assets/downloads), discussions/ADR tab, and timeline of events (comments, check runs, approvals, releases).
- **Discussions & ADRs**: Markdown editors (Lexical/Tiptap) power discussions. ADR templates prefill decision records; accepted ADRs surface badges across graph nodes.
- **Time travel & scenarios**: Initiative workspace offers As-is/To-be/Scenario toggles that filter assignments by effective windows and scenario tags, with timeline scrubber for roadmap visualization.
- **Initiative integration**: Initiatives expose table/board/roadmap views (GitHub Projects style). Cards display linked artifacts, lifecycle, risk score, outstanding checks, and NodeOWNERS.
- **Collaboration cues**: Presence and comment threads use Yjs/WebSockets. Activity feed merges human and agent actions with origin metadata; watchers receive notifications via subscription service.

# Performance & Reliability
- **Indices**: Maintain composite indexes from migration `010_graph` plus governance tables (`idx_node_owner_node`, `idx_approval_scope`, `idx_check_runs_subject`, `idx_transition_queue_repo`). Evaluate GIST/ltree for `path` and partial indexes for queue lookups.
- **Caching strategy**: Cache tree structures, ownership inheritance, ruleset digests, check dashboards, and queue heads (60s TTL). Invalidate caches via events and background refresh.
- **Ruleset cache**: Propagate ruleset updates through Redis pub/sub to invalidate per-tenant caches immediately.
- **Scaling**: Repository, governance, and check services scale horizontally with asyncio + uvloop. Queue workers shard by graph and priority to avoid head-of-line blocking; graph projection consumers run separately to protect core throughput.
- **Resilience**: Enforce idempotency keys (request IDs, queue item IDs) and duplicate guards (unique queue index). Circuit breakers guard artifact-service, OPA, CI integrations. Queue workers retry with exponential backoff; dead-letter queue captures failed transitions for manual remediation.

# Security & Compliance
- **RBAC enforcement**: Middleware maps `RequestContext` roles to fine-grained permissions (`graph.admin`, `graph.editor`, `governance.reviewer`, `ruleset.manager`). High-risk operations (ruleset changes, queue overrides, release deletions) require MFA-verified sessions.
- **Audit logging**: Immutable logs capture lifecycle transitions, approvals, check runs (with results), ruleset versions, release publications, discussion decisions, and survey responses. Logs replicate to centralized compliance store.
- **Data residency**: Tenant region drives routing to localized clusters; ruleset bundles and OPA policies mirror regionally. Releases stored in region-specific object storage with encryption-at-rest.
- **Secrets & token management**: Governance/check services store integration secrets (CI tokens, OPA bundle credentials) in Vault; rotation policies enforced via Temporal workflows.
- **Row-level security & soft delete**: Optional PostgreSQL RLS policies enforce `tenant_id` isolation for regulated tenants; APIs respect `deleted_at`/`legal_hold` tombstones with admin-only purge flows.

# Deployment & DevOps
- **Packaging**: Containers built via `uv`/`poetry`; Helm charts configure Postgres DSN, Redis, event topics, OPA bundle URIs, Temporal endpoints, and ruleset cache TTLs.
- **Local development**: Tilt spins up graph, governance, check services, Temporal test worker, OPA dev server, Neo4j (optional), and Next.js frontend. Migration bundle includes new governance tables and seed rulesets/owners.
- **CI/CD**: Pipelines run schema diff checks, proto compatibility, policy tests (OPA), synthetic approval flows, and queue integration tests. ArgoCD rolls out charts with canary rules for governance services.
- **Observability**: OTLP tracing, Prometheus metrics (API latency, queue depth, check run durations, ruleset cache hit rate), Loki logs, Grafana dashboards. Alerting on failed check runs, queue SLA breaches, and ruleset cache miss spikes.
- **Rollout order**:
  1. Required checks + CheckRun service
  2. NodeOWNERS & Governance service
  3. Transition queue worker
  4. Artifact releases & protected tags
  5. Rulesets + policy bundles
  6. Graph read model & time-scoped assignments
  7. Discussions, surveys, transformation planning views

---

## 154-enterprise-repository-implementation-plan.md

# Enterprise Repository – Staged Implementation Plan

This plan sequences delivery from an MVP that unlocks essential graph curation to a fully governed, collaborative platform aligned with documents 150 (functional), 151 (data), and 152 (tech stack). Each stage lists goals, scope, success criteria, dependencies, and rollout guidance.

---

## Stage 0 – Foundations & Enablement
**Goals:** Stand up core scaffolding and ensure environments can host the new services.

- **Scope**
  - Generate protobuf clients (`RepositoryAdminService`, `RepositoryTreeService`, `RepositoryAssignmentService`, `RepositoryQueryService`).
  - Create initial Alembic migrations (`010_graph`) and seed script (`011_graph_seed`).
  - Provision Postgres schemas, Redis cache, event topics, Temporal namespace (if missing).
  - Enable required Postgres extensions (`ltree`, `btree_gist`) and baseline RLS policies (feature-flagged).
  - Add SDK plumbing in `@ameideio/ameide-sdk-ts` and Next.js app skeleton for Repository Browser.
- **Success criteria**
  - Local Tilt environment runs services with seed data.
  - Frontend loads mock tree via SDK (even if backend returns stub data).
  - Observability baseline (tracing, metrics) wired for new service(s).
- **Dependencies**: None beyond existing infrastructure.

---

## Stage 1 – MVP Repository Curation
**Goals:** Allow tenants to create repositories, manage hierarchy, and assign artifacts with optimistic locking.

- **Scope**
  - Implement Repository service CRUD, tree operations, assignment APIs, and tenant isolation (FR-1 through FR-8).
  - Wire Next.js `RepositoryBrowser` for tree/list/grid, assignment lists, artifact detail drawer (read-only metadata via `artifactQueries`).
  - Backfill initial templates (standards catalog, reference architecture) and import existing seed graph.
  - MVP search (tree traversal + server-side filters) and basic stats (node counts, assignment counts).
- **Success criteria**
  - Tenant can create graph from template, add/move nodes, assign artifacts, and browse via UI.
  - P95 tree fetch < 300 ms, assignment list < 500 ms on seeded data.
  - Audit events (`graph.assignment.*`, `graph.node.*`) emitted to event bus.
  - Operational readiness: cache stampede test at 5× seed load keeps P95 tree fetch under 300 ms; idempotent retries verified via chaos tests.
- **Dependencies**: Stage 0.

---

## Stage 2 – Lifecycle & Governance Lite
**Goals:** Introduce artifact lifecycle states and governance surfaces without complex automation.

- **Scope**
  - Integrate shared command bus for lifecycle transitions (draft → in_review → approved).
  - Add `FR-11` behaviors: labels, assignees, watchers, comments, activity timeline.
  - Implement optimistic check for pinned versions (auto-pin on approve, notify on stale references).
  - Provide governance dashboards (basic counts, pinned artifacts coverage) and success metrics instrumentation.
- **Success criteria**
  - Lifecycle transitions persist state changes, notify watchers, update timeline.
  - Governance metrics (adoption, coverage) populate dashboards with daily refresh.
  - Approved artifacts auto-pin versions and surface in UI.
- **Dependencies**: Stage 1; command bus per backlog/120.

---

## Stage 3 – Governance Deep Dive (Owners, Checks, Queue)
**Goals:** Deliver NodeOWNERS, required checks, transition queue, and rulesets to achieve GitHub-style gated approvals (FR-12 to FR-16).

- **Scope**
  - Add NodeOWNERS table + APIs for owner resolution and approval tracing.
  - Implement required checks & check run orchestration via CheckRun service (OPA, CI integration).
  - Introduce Transition Queue (Temporal worker) with revalidation and queue UI.
  - Stand up check registry + validation, enforce idempotency keys and duplicate guards on queue entries.
  - Launch rulesets (tenant/template scope) with versioning and policy bundles.
- **Success criteria**
  - Transition to `approved` blocked until minimum approvals + required checks satisfied.
  - Queue ensures serialized transitions; queue UI shows position/status.
  - Ruleset updates propagate to caches; audit logs capture ruleset version per transition.
  - Operational readiness: queue SLA (P95 wait < target minutes) observed under load, SoD exception workflows audited, OPA bundle rollback tested, idempotency/duplicate guards verified with chaos tests.
- **Dependencies**: Stage 2; Temporal/OPA infra; team alignment on policies.

---

## Stage 4 – Releases, Discussions, & Advanced Collaboration
**Goals:** Improve consumption and collaboration through releases, discussions, ADRs, and enhanced UI.

- **Scope**
  - Implement `artifact_releases` + `release_assets`, protected tags, release tab UI.
  - Add discussions/ADR tables, REST/gRPC endpoints, Next.js discussion views with Markdown editors.
  - Extend timeline with check run results, releases, approvals, discussions.
  - Introduce survey service (metadata prompts) and data quality scoring.
- **Success criteria**
  - Releases published with immutable assets; protected tags enforced.
  - Discussions/ADRs active with notifications; decisions link to nodes and transformations.
  - Data quality scores appear in governance dashboards; surveys close loop on missing metadata.
  - Operational readiness: release asset hash verification on download; protected tag rules exercised in staging.
- **Dependencies**: Stage 3; S3/Blob storage for assets; Markdown editor integration.

---

## Stage 5 – Time-Aware Modeling & Graph Analytics
**Goals:** Enable as-is/to-be analysis, scenario planning, and impact analysis (FR-18, graph edges).

- **Scope**
  - Extend assignments with `effective_from/to`, scenario tags, graph projection (`model_edges`).
  - Build graph consumer to populate edges, integrate with Neo4j or Postgres table.
  - Enhance transformation workspace with As-is/To-be toggles, scenario overlays, and extended boards (table/board/roadmap) driven by graph assignment metadata.
    - Story: Deliver transformation roadmap toggle that switches As-is/To-be/Scenario views using assignment effective windows.
    - Story: Extend transformation board/table cards to surface role assignments, governance status (checks/approvals/queue), and scenario badge filters.
    - Story: Provide transformation-level impact view powered by `model_edges` that highlights affected systems with role-based filters.
  - Implement impact analysis APIs leveraging graph queries.
- **Success criteria**
  - Users switch between as-is/to-be scenarios; UI updates assignments accordingly.
  - Impact queries (e.g., “affected systems”) return results under 2 seconds on seeded dataset.
  - Initiative boards show lifecycle status, risk scores, dependency counts.
  - Operational readiness: graph projection load test validates P95 impact query < 2 s on 95th percentile tenant size; scenario cache warmers verified.
- **Dependencies**: Stage 4; graph infrastructure; design for scenario UX. Coordinate with Transformation Initiatives Stage 3 delivery (Stage 3-lite may launch with tag-based scenarios until this stage lands).

---

## Stage 6 – Integrations & Continuous Governance
**Goals:** Mature integrations, automation, and monitoring to production-grade standards.

- **Scope**
  - Add webhook/websocket subscriptions; integrate with external notification service.
  - Enforce policy-as-code bundle deployment pipeline (OPA) with testing and rollback.
  - Implement automated recertification workflows, queue SLA alerts, ruleset cache monitoring.
  - Expand API surface for external consumers (webhooks for releases, discussions, governance events).
- **Success criteria**
  - Real-time notifications for check failures, approvals, releases.
  - Policy bundles CI pipeline with automated validation; immediate cache invalidation on deploy.
  - SLOs defined and monitored (queue depth, check run duration, ruleset cache hit rate).
- **Dependencies**: Stages 1–5; coordination with platform automation teams.

---

## Stage 7 – Full Enterprise Rollout & Continuous Improvement
**Goals:** Harden for enterprise adoption, ensure compliance, and plan iterative enhancements.

- **Scope**
  - Conduct pilot with representative tenants; gather feedback on governance UX.
  - Execute load tests and failover drills (queue worker, check service).
  - Document governance playbooks, NodeOWNER onboarding guides, and support runbooks.
  - Plan backlog enhancements (Git provider integrations, AI-assisted reviews, advanced analytics).
- **Success criteria**
  - Pilot sign-off from governance/legal/security stakeholders.
  - Platform meets targeted KPIs (adoption, coverage, check latency, queue SLA).
  - Hand-off materials and training delivered; backlog prioritized for next phase.
- **Dependencies**: Completion of previous stages, cross-functional alignment (support, compliance).

---

## Rollout Guidance
- **Incremental releases:** Each stage should ship behind feature flags per tenant; maintain backward-compatible APIs and migrations.
- **Change management:** For governance-heavy stages, run workshops with architects/governance leads; pilot on limited tenants before global rollout.
- **Telemetry-first:** Instrument metrics/logs from Stage 1 onward to ensure observability underpinning later guardrails.
- **Documentation:** Update specs and runbooks after each stage; highlight changes in release notes and internal wikis.

This staged path balances quick wins (graph curation) with structural investments (governance, policy, analytics) that make the enterprise graph resilient and auditable. Subsequent roadmap items (e.g., external source-control artifact integration, AI automation) can slot after Stage 7.

---

## 160-transformation-initiatives-business-scenario.md

# Transformation Initiative Business Scenarios (PO-first)

## Orientation
Transformation transformations coordinate people, plans, and scenarios across repositories. Governance (owners, approvals, required checks, rulesets, promotion queue) is owned by the graph; transformations consume governance outputs, surface blockers, and drive Go/No-Go decisions. The Product Owner (PO) is the primary consumer—they need clear views of blockers, readiness, and scenario impact.

---

## Scenario B1. Spin Up an Initiative with a PO
- **Trigger:** Executive kickoff for “Payments Modernization 2025.”
- **Owners:** Sponsor & Program Manager (create); PO assigned at creation.
- **Flow:** Create transformation from template (milestones) → link graph → assign roles (Sponsor, PM, **PO**, Lead Architect, Reviewers, Contributors, optional AI reviewers).
- **PO Value:** PO becomes default owner for board filters, readiness gates, and notifications.
- **KPIs:** Time to fully staff transformation; role coverage (% roles filled).
- **Spec references:** Initiative business persona, functional objectives (roles), IA tables (transformations, milestones, role assignments), Stage 1 lifecycle & roles.

## Scenario B2. Scope the Work from the Repository
- **Trigger:** Planning sprint (week 0–2).
- **Owners:** Lead Architect + PO.
- **Flow:** Link existing graph artifacts (current state) to milestones; create drafts for proposed work (graph assignments carry lifecycle windows/scenarios); transformation board surfaces “Existing vs New.”
- **PO Value:** Clear visibility into baseline vs new scope without shadow artefacts.
- **KPIs:** % scope linked to existing artifacts; # new drafts created; graph coverage ratio.
- **Spec references:** Initiative artifact links, graph FR‑6/18.

## Scenario B3. As-Is / To-Be / Scenario Toggle (PO Demo View)
- **Trigger:** Steering committee review.
- **Owner:** PO.
- **Flow:** PO toggles As-Is/To-Be/Scenario; workspace queries graph assignment windows & scenario tags via `assignment_id`; dependency overlays use `model_edges`; results cached for <300 ms perceived latency (lite) / <2 s P95 with full windows.
- **PO Value:** One click shows what changes, when, and who is affected.
- **KPIs:** Toggle latency (steady/P95), impacted system counts, unresolved dependency count.
- **Spec references:** FR‑18, graph Stage 5, transformation Stages 3 & 5.

## Scenario B4. Approval & Checks Triage (PO Blockers Funnel)
- **Trigger:** Weekly board review.
- **Owner:** PO.
- **Flow:** Filter “Needs approvals”, “Failing checks”, “In queue”; graph events refresh chip counts; PO assigns owners or reorders queue priority via governance lead.
- **PO Value:** Actionable list of who must act next; supports stand-ups and executive updates.
- **KPIs:** Time-to-unblock; approvals outstanding per week; % checks passing after rerun.
- **Spec references:** Repository FR‑12/13/14 outputs; transformation board UI module; Stage 3 (boards & roadmaps).

## Scenario B5. Release Readiness (Go/No-Go)
- **Trigger:** Milestone approaching Launch.
- **Owner:** PO (decision facilitator).
- **Flow:** Initiative workspace compiles release readiness report: approved & released versions, required check status, approvals list, queue history, release notes; export PDF/CSV.
- **PO Value:** Evidence-based Go/No-Go in two clicks.
- **KPIs:** % scope released; # critical checks remaining; documentation completeness.
- **Spec references:** Repository FR‑15; transformation KPI pane; Stage 4.

---

## Future & Portfolio Scenarios
### B6. Directory Change & Role Reassignment (Future)
- **Trigger:** Product Owner changes in corporate directory.
- **Owners:** Program Manager + incoming PO.
- **Flow:** Directory sync updates `transformation_role_assignments`; workspace flags gaps; notifications prompt acceptance; board filters switch to new PO.
- **PO Value:** No loss of accountability mid-stream.
- **Spec references:** Initiative directory integration (Stage 5), IA role assignments.

### B7. Portfolio Heatmap (Multi-Initiative View)
- **Trigger:** Quarterly planning.
- **Owners:** Sponsor/PMO; POs provide context.
- **Flow:** Portfolio dashboard aggregates milestone health, blocker counts, approval cycle time, readiness per transformation; drill-down links to blockers funnel.
- **PO Value:** Prioritization and capacity planning with governance-backed metrics.
- **Spec references:** Initiative Stage 5 portfolio views, governance event consumption.

---

## Acceptance Test (PO Blockers Funnel)
- **Scenario:** Given a board with mixed governance states, when the PO filters “Needs approvals,” then the cards show owner chips, deep links to artifact timelines, and counts update within 30 s of graph events. *(Repository FR‑12/13, Initiative Stage 3)*

---

## KPIs & Telemetry Checklist
- Board column load / scenario toggle <300 ms perceived; P95 <500 ms under 95th percentile tenant load.
- Approval cycle time P50/P95; queue wait P95; blockers resolved per week.
- Release readiness completeness (% artifacts released, checks cleared).
- Role coverage, directory sync resolution time.
- Impact delta (As-Is vs To-Be) per scenario.

---

## Why This Fits the Specs
- Initiatives consume graph truth (assignments, releases, governance outputs) without altering guardrails.
- POs are first-class: blockers, readiness, scenario toggles, and portfolio views are designed for decision support.
- Each scenario maps to functional requirements, IA structures, and staged delivery (Initiative Stages 1–5, Repository Stage 5), giving engineering clear definition-of-done.

---

## 161-transformation-initiatives-functional.md

# Executive Summary
Transformation Initiatives represent tenant-led change programs (product naming for legacy “ADM cycles”) that orchestrate repositories and artifacts in Ameide. Initiatives coordinate proposed/WIP architectures, delivery progress, and roles, while governance (owners, approvals, required checks, rulesets, promotion queue) remains defined once at graph/tenant scope. This document outlines the functional scope required to plan, execute, and track transformations while reusing the graph infrastructure defined in `backlog/150-enterprise-graph-functional.md`.

# Objectives
- Provide a tenant-scoped portfolio view of transformations, their phases, and status.
- Link transformations bidirectionally with graph artifacts, standards, and assignments (including effective windows and scenario tags for proposed/WIP designs).
- Surface progress dashboards and transformation-specific views (boards/roadmaps) that reflect graph data while honoring graph-level governance outcomes (checks, approvals, queue status).
- Enable collaboration across Enterprise Architects, Initiative Sponsors, Initiative Leads, Delivery Teams, and Reviewers with artifact-centric discussions and review workflows.

# Key Capabilities (Draft)
- Initiative lifecycle management (create, plan, launch, close) with template-driven phase definitions.
- Association of repositories, nodes, and artifacts to transformation milestones (including effective windows and scenario tags for proposed/WIP architectures).
- Initiative workspace views (table/board/roadmap) with As-is / To-be / Scenario toggles derived from graph assignments.
- Status tracking and KPI dashboards leveraging graph stats, assignment metadata, and graph-level governance outputs (checks, approvals, queue status).
- Integration hooks for notifications, workflows automation, and change logs.
- People & roles (and agents) assignment (future integration with company directory and agent registry) to capture transformation sponsors, program managers, architects, delivery leads, reviewers, contributors, and AI reviewers/assistants, including RBAC alignment and availability.
- Release readiness and blocker reporting for Product Owners, aggregating artifact approvals, required checks, queue status, and upcoming releases.

# Personas & Roles (Draft)
- **Initiative Sponsor**: Executive accountable for outcomes; sets KPIs, approves stage gates.
- **Program Manager**: Coordinates execution, updates timelines, manages risks and communications.
- **Product Owner**: Curates transformation backlog, defines acceptance criteria, runs release readiness checks, and aligns business stakeholders; partners with Lead Architect/Governance Lead to ensure quality gates are met.
- **Lead Architect**: Aligns transformation scope with graph architecture; curates artifacts and ensures governance adherence.
- **Delivery Lead**: Oversees implementation teams, links workstreams to transformation milestones.
- **Contributors**: Cross-functional specialists (engineers, analysts) assigned to specific milestones or artifacts.
- **AI Reviewer/Assistant**: Automated agent that performs preliminary checks, generates summaries, or proposes recommendations prior to human approval; participates in review workflows with clearly scoped permissions.

# Role Management & Directory Integration (Draft)
- Support assignment of people and AI agents to transformations with role tags (sponsor, program manager, architect, delivery lead, contributor, ai_reviewer).
- Store source-of-truth references (e.g., company directory GUID, agent registry ID, email/webhook) for each principal; allow sync from corporate directory/agent registry (future integration).
- Provide RBAC mapping so transformation roles inherit appropriate graph permissions (e.g., lead architect retains governance approval rights) and constrain AI agents to approved scopes (read-only, check execution, advisory comments).
- Surface ownership and accountability in transformation dashboards (e.g., cards display assigned leads or agent assignments, empty slots flagged for action).
- Expose APIs to list/search transformation members and agents, update roles, and emit events when assignments change (for downstream notifications or agent orchestration).

# Artifact-Centric Collaboration & Governance Alignment
- Discussions, ADRs, and request-for-review workflows remain anchored to artifacts in the graph; transformation workspaces deep-link to those timelines rather than duplicating threads.
- Initiatives surface consolidated status for artifacts (e.g., checks pending, approvals required, queue position, AI reviewer findings) by consuming graph governance outputs.
- Promotion decisions and guardrails stay governed at graph/tenant scope; transformations reflect outcomes for planning and reporting without redefining approval logic.

# UI Architecture
## Overview
Transformation transformation screens live inside the existing Next.js 15 SPA, sharing the AppShell, design system, and provider stack used by repositories. The workspace layers planning views (boards, tables, roadmaps, KPIs) atop graph assignments and governance outputs without duplicating governance logic.

## Existing Platform Context
- **App shell**: `app/(app)/layout.tsx` provides Header, Sidebar, Search, Tooltip, DataStream, UrlSyncGate, and Debug providers. Initiative routes integrate with this shell rather than redefining providers.
- **Header**: Contextual breadcrumbs/tabs rendered via `resolveHeaderDescriptor` expose Overview, Board, Roadmap, KPIs, and Settings for each transformation.
- **Sidebar**: `AppSidebar` gains transformation navigation entries and “New Initiative” quick action; history panel captures recently viewed transformations.
- **Providers & telemetry**: Hooks (`useSearch`, `useSidebar`, `useDebug`) and observability utilities are reused. Scenario/Realtime contexts compose inside `DataStreamProvider` for consistent lifecycle handling.
- **Routing**: Initiative workspace lives under `/transformations/:transformationId`, coexisting with graph paths. Feature flags (e.g., `transformation_boards`, `transformation_roadmap`) guard rollout per tenant.

## Component Topology
```
====================================================================================================================
| AppShell (app/(app)/layout.tsx)                                                                                  |
| ---------------------------------------------------------------------------------------------------------------- |
|  Providers: DebugProvider → TooltipProvider → SearchProvider → DataStreamProvider → UrlSyncGate → SDKClient      |
|  Shared UI: Header (breadcrumbs, scope tabs, actions) • AppSidebar (nav, history, quick-create modals)           |
|  Main Content: Routed Initiative Workspace                                                                      |
====================================================================================================================
          │
          │ layout.tsx renders transformation routes here (header spacing via --app-header-h)
          ▼
+------------------------------------------------------------------------------------------------------------------+
| Initiative Workspace ( /transformations/:transformationId )                                                              |
|------------------------------------------------------------------------------------------------------------------|
|  ┌─────────────── Overview Pane ───────────────┐  ┌────────────── Board/Table View ───────────────┐             |
|  │ • Status summary cards                      │  │ • Columns grouped by lifecycle / role          │             |
|  │ • Upcoming milestones (timeline preview)    │  │ • Cards: title, role owner, governance badges   │             |
|  │ • Role roster & directory sync status       │  │ • Drag-and-drop with optimistic updates         │             |
|  │ • Recently updated artifacts (deep-link)    │  │ • Filters: role, lifecycle, risk, scenario      │             |
|  └─────────────────────────────────────────────┘  │ • Saved views (e.g., “Needs approvals”)         │             |
|                                                   │ • Quick actions: link artifact, assign owner    │             |
|                                                   └─────────────────────────────────────────────────┘             |
|                                                                                                                  |
|  ┌────────────── Roadmap / Scenario View ───────────────┐  ┌────────────── KPI / Dashboard Pane ───────────────┐ |
|  │ • Timeline bars (planned vs actual)                  │  │ • Charts: milestone completion %, cycle time     │ |
|  │ • Toggles: As-is | To-be | Scenario (Plan A/B)       │  │ • Check success rate, queue latency, data quality │ |
|  │ • Dependency overlays from graph `model_edges`  │  │ • Role coverage heatmap, survey completion        │ |
|  │ • Hover tooltips: affected systems, owners           │  │ • Export/report shortcuts                          │ |
|  └─────────────────────────────────────────────────────┘  └─────────────────────────────────────────────────────┘ |
|                                                                                                                  |
|  ┌────────────── Settings / Admin Drawer ──────────────┐                                                         |
|  │ • Milestone template management                     │                                                         |
|  │ • Role assignment (directory search + manual)       │                                                         |
|  │ • Notification & survey preferences                 │                                                         |
|  │ • Scenario definitions (tags, descriptions)         │                                                         |
|  └─────────────────────────────────────────────────────┘                                                         |
+------------------------------------------------------------------------------------------------------------------+

Legend:
- Overview anchors summary/KPIs; Board/Table manage execution; Roadmap handles scenario planning; KPI pane aggregates governance/delivery metrics; Settings configures milestones, roles, notifications.
```

## Data Flow & State Management
- **Server interactions**: `InitiativeService` (transformation/milestone/role CRUD), `InitiativeWorkspaceService` (board/table/roadmap datasets), graph governance APIs (checks, approvals, queue), notification/survey endpoints.
- **React Query caches**: transformations, milestones, role assignments, artifact links, board columns, roadmap datasets, governance snapshots. Keys scoped by tenant+transformation; invalidated on mutations and `transformation.*`/`graph.*` events.
- **Local state**: view mode (board vs table), active filters (role, lifecycle, risk), scenario selection, saved view definitions, column/order preferences.
- **Realtime updates**: SSE/WebSockets stream `transformation.role.updated`, `transformation.milestone.updated`, governance events; board/roadmap re-render optimistically. Presence indicators (optional) for concurrent users editing milestones.

## Module Responsibilities
- **Initiative Overview**: Summaries (status, KPIs, upcoming milestones), role roster, latest releases, quick links into graph artifacts/discussions.
- **Board/Table View**: Kanban columns, list layout, drag-and-drop progression, per-card governance badges (checks, approvals, queue), inline actions (assign role/agent, link artifact), saved views for Product Owners (e.g., Release Readiness, Blocked checks, Needs approvals).
- **Roadmap & Scenario View**: Timeline visualization with As-is/To-be toggles, scenario overlays, dependency highlights from graph graph; supports zoom, grouping by workstream.
- **KPI & Dashboard Pane**: Charts/tables for milestone progress, approval cycle time, check success rate, data-quality scores, release readiness metrics; filters by role, phase, scenario.
- **Settings Drawer**: Manage milestones, assign roles (people + AI agents via directory/registry search), configure notifications, define scenarios, manage surveys, scope AI agent permissions (read-only, comment, check-run).
- **Product Owner Views**: Release readiness reports (artifact, version, required checks, approvals, queue status), blockers funnel ranked by outstanding approvals/check failures, exportable Go/No-Go packet.

## UI Patterns & Components
- Reuse shadcn/ui primitives (Tabs, Drawer, Dialog, Dropdowns) plus custom components: transformation-card, governance-badge, timeline-bar, scenario-badge, role-roster list.
- Boards use virtualized lists for large datasets; roadmap uses D3/visx timeline component with tooltip overlays.
- Quick links deep-link to graph artifact drawer/timeline (new tab or side panel); AI agent summaries or recommendations display alongside human comments when available.

## Governance & Collaboration UX
- Cards display governance badges (checks pending, approvals outstanding, queue position) derived from graph events; tooltips list required actions and responsible owners.
- Discussions remain artifact-centric—board cards provide “Open discussion” CTA that deep-links to graph artifact timeline (human + AI agent contributions distinguished via badges).
- Governance badges differentiate human approvals vs AI advisory checks to maintain transparency and trust.
- Role avatars indicate assignment health (e.g., missing role highlighted). Directory sync warnings appear inline when principal changes are detected.

## Performance & Offline Strategy
- Incremental data fetch per board column (lazy loading) with optimistic UI updates on drag-and-drop.
- Prefetch artifact/governance details on hover to minimize drawer latency.
- Offline indicators when mutations fail; queued actions retried automatically with exponential backoff.
- SLO alignment: board column load and scenario toggle interactions target <300 ms perceived latency; P95 <500 ms under 95th percentile tenant load (matching graph SLO wording).

## Telemetry & Feature Flags
- Track board interactions, scenario toggle usage, KPI filter changes, role assignment edits, roadmap zoom actions.
- Feature flags: `transformation_board_view`, `transformation_table_view`, `transformation_roadmap`, `transformation_kpi_dashboards`, `transformation_directory_sync` control rollout per tenant.

# Next Steps
- Validate personas and workflows with product stakeholders.
- Detail transformation data model and API contracts.
- Align transformation templates with graph templates for consistent taxonomy.
- Engage identity team to design directory integration API contract for people/role assignments.
- Plan transformation workspace Stage 5 stories for As-is / To-be / Scenario toggles and role-driven boards leveraging graph assignment metadata and governance outputs.

---

## 162-transformation-initiatives-information-architecture.md

# Transformation Initiatives Information Architecture

## ER Diagram
```mermaid
erDiagram
    Tenant ||--o{ TransformationInitiative : "1..N"
    TransformationInitiative {
        string id PK
        string tenant_id FK
        string graph_id FK
        string name
        text description
        timestamptz start_date
        timestamptz target_date
        jsonb metadata
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
        boolean legal_hold
    }
    TransformationInitiative ||--o{ InitiativeMilestone : "1..N"
    InitiativeMilestone {
        string id PK
        string transformation_id FK
        string title
        text description
        timestamptz planned_start
        timestamptz planned_end
        timestamptz actual_start
        timestamptz actual_end
        text status
        jsonb metadata
    }
    TransformationInitiative ||--o{ InitiativeRoleAssignment : "1..N"
    InitiativeRoleAssignment {
        string id PK
        string transformation_id FK
        string person_id
        string role_key
        jsonb attributes
        timestamptz assigned_at
        timestamptz unassigned_at
    }
    TransformationInitiative ||--o{ InitiativeArtifactLink : "1..N"
    InitiativeArtifactLink {
        string id PK
        string transformation_id FK
        string assignment_id FK
        string graph_id
        string artifact_id
        string node_id
        jsonb metadata
    }
    TransformationInitiative ||--o{ InitiativeChecklist : "1..N"
    InitiativeChecklist {
        string id PK
        string transformation_id FK
        string name
        text description
        jsonb items
        text status
        timestamptz created_at
    }
```

## Schema Highlights
- **transformation_transformations**: Master record per transformation. Stores graph linkage, lifecycle dates, descriptive metadata, soft-delete/legal-hold state.
- **transformation_milestones**: Template-driven milestones (Discover, Design, Build, Launch) with planned/actual windows and status.
- **transformation_role_assignments**: Mapping of people to roles (sponsor, transformation_lead, lead_architect, product_owner, reviewer, contributor). `person_id` references corporate directory GUID when available; attributes include email, department, availability.
- **transformation_artifact_links**: Junction table that associates transformations with graph assignments. Stores the canonical `assignment_id` (which already carries effective windows/scenario tags), optional milestone references, and transformation-level metadata.
- **transformation_checklists**: Optional per-transformation checklists (e.g., compliance gates, funding approvals) stored as JSON arrays with individual item status.
- **Derived views**: Materialized views for dashboards (e.g., milestone completion %, approval cycle time, check run aggregations) fed from graph governance events.

## Indices & Constraints
- Unique `(tenant_id, name)` per transformation to prevent duplicates.
- Foreign keys back to repositories ensure transformations cannot reference artifacts outside the linked graph.
- Role assignments enforce `(transformation_id, person_id, role_key, unassigned_at IS NULL)` uniqueness to avoid overlapping active roles.
- Artifact links enforce `(transformation_id, assignment_id)` uniqueness to avoid duplicate references. Effective windows and scenario tags remain canonical on the graph assignment record.
- Soft-delete columns (`deleted_at`, `legal_hold`) propagate through queries; views default to active records only.

## Data Flows
1. **Initiative creation**: Inserts record, optional milestones from template, default role placeholders.
2. **Role assignment**: Upserts into `transformation_role_assignments`; emits events for notifications and directory sync.
3. **Artifact linking**: Adds entries into `transformation_artifact_links` referencing graph assignments (via `assignment_id`) and optional milestone association.
4. **Governance consumption**: Jobs subscribe to graph governance events to update transformation dashboards (check outcomes, approvals, queue status) without duplicating governance logic.
5. **Reporting**: Read models aggregate milestone progress, role coverage, risk flags, and scenario impacts (joins with `model_edges` for dependency graphs).

## Soft Delete & Legal Hold
- Initiatives, milestones, role assignments, and artifact links support soft-delete. Legal hold prevents purge operations and keeps historical data available for audit.
- Background jobs periodically purge soft-deleted records outside legal-hold scope after retention thresholds.

## Directory Integration Considerations
- `person_id` stores canonical GUID from corporate directory; attributes include snapshot of display name and email to cover offline scenarios.
- Sync jobs reconcile assigned roles with directory changes (e.g., replacement of Sponsor) and emit events when conflicts arise.

## Governance Alignment
- Initiatives do not redefine governance rules; instead, `transformation_artifact_links` store `assignment_id` references and cached governance snapshots (latest approval status, required check completion). Dashboards fetch authoritative status from graph APIs/events.

## Next Steps
- Finalize JSON schemas for `transformation_checklists` and milestone templates.
- Define API contracts for role assignment CRUD, artifact linking, and milestone updates.
- Model transformation-level KPIs (cycle times, risk scoring) in analytical views fed from this schema plus graph events.

---

## 163-transformation-initiatives-technology-architecture.md

# Transformation Initiatives Technology Architecture

## Technology Matrix
| Layer | Choice | Rationale | Integration Notes |
| --- | --- | --- | --- |
| Backend runtime | Python 3.11 (FastAPI + gRPC via ConnectRPC) | Aligns with graph services; shared middleware | New `services/transformation_transformations` service reuses auth, tenancy, tracing stack |
| Data store | PostgreSQL 15 | Reuses tenant DB, supports JSONB + temporal queries | Tables defined in `162-transformation-transformations-information-architecture.md`; references graph assignment IDs for scenarios |
| Messaging | NATS JetStream / Kafka | React to graph governance & artifact events | Subscribe to `graph.*`, emit `transformation.*` updates (role changes, milestone status) |
| Cache | Redis (clustered) | Store transformation dashboard snapshots, role resolution, scenario filters | Keys scoped by tenant and transformation; invalidated on mutation events |
| Frontend | Next.js 15 + React 19 | Shares SPA shell/components with graph | Initiative workspace screens live under `/transformations/:transformationId` |
| Client SDK | `@ameideio/ameide-sdk-ts` | Typed ConnectRPC clients | Extend SDK with transformation service stubs (`InitiativeService`, `InitiativeWorkspaceService`) |
| Workflow automation | Temporal / Argo Workflows | Drives periodic reminders, survey prompts, recertification flows | Workers consume transformation events and graph governance outcomes |
| Directory integration | Future Corporate Directory API | Source of truth for people & roles | Sync job maps directory principals to transformation role assignments |

## Service Architecture
- **InitiativeService (gRPC)**: CRUD for transformations, milestones, role assignments, artifact links; enforces tenancy, soft-delete rules, and directory lookups.
- **InitiativeWorkspaceService (gRPC for writes, GraphQL for aggregated reads)**: Provides board/table/roadmap datasets by joining transformation data with graph APIs. GraphQL is restricted to read-only fan-in queries; mutations stay on gRPC.
- **InitiativeEventsProcessor**: Listens to graph governance events (check runs, approvals, releases) and updates transformation read models. Emits `transformation.status.updated`, `transformation.role.assignment.updated`, `transformation.milestone.progressed`.
- **Reminder/Survey Worker**: Temporal workers trigger survey prompts, role reminders, recertification tasks based on milestone dates and governance status.

## API Design Considerations
- All RPCs receive `RequestContext` (tenant_id, user_id, request_id); operations validate graph ownership relationship.
- Role assignment endpoints enforce uniqueness and consult directory data when available; respond with normalized principal info.
- Artifact linking endpoints validate artifact/node existence via graph APIs and persist the canonical `assignment_id` (effective windows/scenario tags remain on graph assignments); transformation-specific metadata lives alongside the link.
- Workspace queries support As-is/To-be/Scenario filters by projecting graph assignment windows via the stored `assignment_id` and transformation metadata. Impacts rely on Graph API (graph `model_edges`).
- Read-only endpoints return governance status fields (required checks pending, approvals outstanding, queue position) retrieved from graph services.

## Data Synchronization
- Repository governance events (`governance.check_run.*`, `graph.transition.*`, `graph.release.published`) drive transformation status updates and refresh linked assignment metadata (checks, approvals, releases).
- Directory sync job resolves principal metadata; conflicts (e.g., missing user) trigger alert events and mark assignments as stale.
- Scheduled jobs recompute KPI aggregates (cycle time, completion %) for dashboards; results cached in Redis.

## Security & Compliance
- RBAC middleware ensures only authorized tenant members access transformation data; fine-grained scopes (transformations.admin / transformations.editor / transformations.viewer) map to graph permissions.
- Auditing captures all role assignments, milestone changes, and artifact link modifications with actor, timestamp, and diff.
- Soft-delete with legal-hold flags ensures regulatory retention; purge operations gated behind governance admin role.
- Sensitive integrations (directory, notifications) store credentials in Vault; rotation orchestrated via Temporal workflows.

## Observability
- OpenTelemetry instrumentation traces transformation RPCs, event processors, and Temporal workflows.
- Metrics: board query latency, milestone throughput, role assignment churn, event processing lag, survey completion rate.
- Logs: structured JSON with tenant_id, transformation_id; log forwarding to central logging stack (Loki or Elastic).

## Deployment
- Service packaged via `uv`/`poetry` container; Helm chart configures Postgres DSN, Redis, event topics, Temporal namespace, directory API endpoint.
- Local dev (Tilt) spins up transformation service, graph mocks, Temporal worker, directory stub, Next.js client.
- CI/CD: schema diff checks, API contract tests, event consumer integration tests, UI e2e flows (Playwright) verifying board/roadmap behavior.

## Dependencies & Interfaces
- **Repository Service**: artifact lookup, governance status, release info.
- **Directory API** (future): principal metadata, group membership.
- **Notification Service**: deliver role-based reminders, release announcements.
- **Analytics pipeline**: consumes transformation events for portfolio-level dashboards.

## Next Steps
- Finalize gRPC proto definitions for `InitiativeService` and `InitiativeWorkspaceService`.
- Define event schemas (CloudEvents or custom) for transformation notifications.
- Prototype Temporal workflows for reminder/survey automation.
- Establish directory sync contract with identity team (batch vs webhook).

---

## 164-transformation-initiatives-implementation-plan.md

# Transformation Initiatives Implementation Plan

## Stage 0 – Foundations
**Goals:** Establish baseline schema, service scaffolding, and SDK contracts.
- Provision database tables (`transformation_transformations`, milestones, role assignments, artifact links).
- Generate proto stubs for `InitiativeService` and `InitiativeWorkspaceService`; extend SDK.
- Configure Tilt environment with transformation service, graph mock, Temporal worker, directory stub.
- Success: CRUD prototype for transformations and milestones; basic board view renders mock data; observability baseline in place.

## Stage 1 – Initiative Lifecycle & Roles
**Goals:** Deliver core transformation management.
- Implement create/update/delete for transformations and milestones (template-driven).
- Build role assignment UX (manual selection + directory lookup placeholder) and API enforcement of uniqueness.
- Emit events on role/milestone changes; update Redis caches.
- Success: Initiatives can be created, templated milestones instantiated, roles assigned with audit trail.

## Stage 2 – Artifact Linking & Governance Consumption
**Goals:** Connect transformations to graph artifacts and governance state.
- Implement APIs/UI to link transformations to graph assignments (store `assignment_id`, milestone association, transformation metadata).
- Subscribe to graph governance events (check runs, approvals, releases) and persist status snapshots.
- Display governance banners (pending checks, approvals, queue) within transformation workspace cards.
- Success: Initiative workspace reflects governance status in real time; artifact links reference canonical assignment data.

## Stage 3 – Boards & Roadmaps (Scenario-lite)
**Goals:** Deliver transformation workspace views while Repository Stage 5 is still in progress.
- Ship board/table views grouped by lifecycle, role owner, risk flags.
- Implement roadmap timeline with As-is / To-be / Scenario toggles using tag-based scenarios (upgrade to assignment windows/`model_edges` once Repository Stage 5 lands).
- Provide role-based filters and saved views (e.g., “Needs approvals”, “Blocked checks”).
- Success: Initiative leads can manage delivery via boards/roadmaps; scenario toggles refresh tag-based overlays under 300 ms perceived latency. Full window/impact overlays ship in Stage 3+ (after Repository Stage 5).

## Stage 4 – Notifications, Surveys, KPIs
**Goals:** Automate reminders and quality tracking.
- Integrate with notification service to alert role owners (pending reviews, overdue milestones).
- Launch survey prompts for missing metadata; collect responses and update data-quality scores.
- Produce KPI dashboards (milestone completion %, approval cycle time, check success rate).
- Success: Notification/survey workflows operational; dashboards show live KPIs.

## Stage 5 – Scenario Upgrade, Directory Integration & Portfolio Views
**Goals:** Upgrade scenario capabilities, connect to corporate directory, and deliver portfolio insights.
- Integrate graph Stage 5 outputs (assignment windows, `model_edges`) into roadmap/impact views; enable full As-is/To-be/Scenario overlays with dependency highlights.
- Implement directory sync (batch or webhook) to reconcile role assignments with source of truth.
- Handle reassignment flows (e.g., departing sponsor) and surface conflicts.
- Provide tenant-level portfolio view combining multiple transformations (status heatmaps, capacity planning).
- Success: Scenario overlays reflect authoritative assignment windows/impacts; role data stays in sync with directory; portfolio dashboard supports executive reporting.
- Dependencies: Repository Stage 5 (time-aware modeling & graph analytics), identity integration readiness.

## Stage 6 – Operational Hardening
**Goals:** Ensure production readiness.
- Load test board/roadmap queries with 95th percentile tenant data; verify P95 < 500 ms.
- Chaos tests for event lag, directory outages, notification failures.
- Document runbooks, escalation paths, and retention policies for soft-deleted transformations.
- Success: SLOs agreed and met; incident response playbooks ready; compliance requirements (audit trail, legal hold) validated.

## Rollout Guidance
- Feature flag transformations per tenant; start with design partners.
- Coordinate messaging with enterprise architects/governance leads; provide training on role assignments and scenario planning.
- Gather feedback after each stage to adjust future work (e.g., additional KPIs, integration needs).

---

## 102-keycloak-config.md (old)

> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# Keycloak Configuration and Setup

## Overview
Keycloak is deployed as the identity and access management solution for the AMEIDE platform, providing OAuth2/OIDC authentication and authorization.

## Current Configuration

### Deployment Details
- **Helm Chart**: bitnami/keycloak v17.3.6
- **Keycloak Version**: 22.0.5
- **Namespace**: ameide
- **Service Type**: LoadBalancer
- **External Port**: 4000
- **Internal Port**: 8080

### Database Configuration
- **Type**: PostgreSQL (CloudNativePG)
- **Host**: postgres-keycloak-rw.ameide.svc.cluster.local
- **Database**: keycloak
- **User**: keycloak

### Access Points
- **Admin Console**: http://localhost:4000
- **OIDC Discovery**: http://localhost:4000/realms/ameide/.well-known/openid-configuration
- **Token Endpoint**: http://localhost:4000/realms/ameide/protocol/openid-connect/token
- **Internal Service URL**: http://keycloak.ameide.svc.cluster.local:4000

### Realms Configuration

#### Master Realm
- **Purpose**: Keycloak administration
- **Admin User**: admin
- **Admin Password**: changeme
- **Client**: admin-cli (for API access)

#### Ameide Realm
- **Purpose**: Application authentication
- **Configured Clients**:
  - `web-app`: Main application client
  - `account`: User account management
  - `account-console`: Account management UI
  - `admin-cli`: Administrative CLI
  - `broker`: Identity brokering
  - `realm-management`: Realm administration
  - `security-admin-console`: Security administration

### Test Credentials
- **Standard User**: user@ameide.io / password123
- **Admin User**: admin@ameide.io / password123
- **Test User**: testuser@ameide.local / testpass
- **Test Client**: web-app (confidential client with secret)

## Known Issues and Resolutions

### 1. Liveness Probe Configuration Issue (FIXED)
**Problem**: StatefulSet fails to create pods due to invalid liveness probe configuration with both `httpGet` and `tcpSocket` handlers.

**Root Cause**: The Bitnami Keycloak Helm chart v17.3.6 can merge probe configurations incorrectly during upgrades if the probe type is not explicitly specified in values.

**Error Message**:
```
Pod "keycloak-0" is invalid: spec.containers[0].livenessProbe.tcpSocket: 
Forbidden: may not specify more than 1 handler type
```

**Prevention (IMPLEMENTED)**:
- Explicitly specified `httpGet` probe type in `/workspace/infra/kubernetes/values/infrastructure/keycloak.yaml`
- Created validation script at `/workspace/infra/kubernetes/scripts/validate-keycloak.sh`
- Probes now explicitly define handler type to prevent merge conflicts

**Resolution if it occurs**:
1. Run validation script: `/workspace/infra/kubernetes/scripts/validate-keycloak.sh`
2. Delete the problematic StatefulSet: `kubectl delete statefulset keycloak -n ameide`
3. Reinstall with correct configuration using Helm

### 2. Stuck Helm Release
**Problem**: Helm release stuck in `pending-upgrade` or `pending-rollback` state.

**Resolution**:
```bash
# Check release status
helm status keycloak -n ameide

# Rollback to previous version
helm rollback keycloak -n ameide

# Verify status
helm list -n ameide --pending
```

### 3. Authentication Flow Configuration Issue (FIXED)
**Problem**: Keycloak authentication flow references non-existent "organization" authenticator.

**Error Message**:
```
Unable to find factory for AuthenticatorFactory: organization
```

**Root Cause**: Imported realm configuration included custom authentication flows with unavailable authenticators.

**Resolution**: 
- Reset browser flow to default via Admin API
- Updated client from public to confidential with secret
- Fixed authentication flow to use standard username/password form

## Testing Authentication

### 1. Obtain Admin Token (Master Realm)
```bash
curl -X POST http://172.20.0.3:4000/realms/master/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin" \
  -d "password=changeme" \
  -d "grant_type=password" \
  -d "client_id=admin-cli"
```

### 2. Obtain User Token (Ameide Realm)
```bash
# Test with standard user
curl -X POST http://localhost:4000/realms/ameide/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=testuser" \
  -d "password=testpass" \
  -d "grant_type=password" \
  -d "client_id=web-app"

# Test with admin user
curl -X POST http://localhost:4000/realms/ameide/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin" \
  -d "password=adminpass" \
  -d "grant_type=password" \
  -d "client_id=web-app"
```

### 3. Create New User via API
```bash
# Get admin token
TOKEN=$(curl -s -X POST http://172.20.0.3:4000/realms/master/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin" \
  -d "password=changeme" \
  -d "grant_type=password" \
  -d "client_id=admin-cli" | jq -r '.access_token')

# Create user
curl -X POST http://172.20.0.3:4000/admin/realms/ameide/users \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "newuser",
    "enabled": true,
    "emailVerified": true,
    "credentials": [{
      "type": "password",
      "value": "password123",
      "temporary": false
    }]
  }'
```

## Monitoring and Troubleshooting

### Check Pod Status
```bash
kubectl get pods -n ameide | grep keycloak
kubectl logs keycloak-0 -n ameide --tail=50
```

### Check Events
```bash
kubectl get events -n ameide --field-selector involvedObject.name=keycloak-0
kubectl describe statefulset keycloak -n ameide
```

### Verify Service Endpoints
```bash
# Check service
kubectl get svc keycloak -n ameide

# Test connectivity
curl -s http://172.20.0.3:4000/realms/ameide/.well-known/openid-configuration
```

## Configuration Files

### Main Values
- `/workspace/infra/kubernetes/values/infrastructure/keycloak.yaml`
- Contains base configuration including database settings, probes, and realm import

### Local Environment Overrides
- `/workspace/infra/kubernetes/environments/local/infrastructure/keycloak.yaml`
- Overrides service type to LoadBalancer for local development

### Realm Configuration
- Mounted from ConfigMap: `keycloak-realm-config`
- Path: `/opt/bitnami/keycloak/data/import/ameide-realm.json`
- Auto-imported on startup with `KEYCLOAK_EXTRA_ARGS=--import-realm`

## NextAuth Integration

### Configuration for www-ameide-canvas

The frontend applications use NextAuth.js with the Keycloak provider:

#### Environment Variables (Updated for Production-Ready Setup)
```yaml
AUTH_URL: http://localhost:3001
AUTH_SECRET: development-secret-change-in-production
AUTH_TRUST_HOST: true  # For development
KEYCLOAK_CLIENT_ID: web-app
KEYCLOAK_CLIENT_SECRET: development-client-secret-change-in-production
# Dual URL configuration for internal vs public access
KEYCLOAK_ISSUER: http://keycloak:4000/realms/ameide  # Internal server-side
KEYCLOAK_ISSUER_PUBLIC: http://localhost:4000/realms/ameide  # Browser OAuth
```

#### Authentication Flow
1. User visits protected page
2. Middleware redirects to `/login` (custom login page)
3. User clicks "Sign in with SSO" button
4. NextAuth initiates OAuth flow with Keycloak
5. Redirected to Keycloak login page at port 4000
6. User enters credentials (e.g., user@ameide.io / password123)
7. After successful auth, redirected back with session

#### Testing Authentication
```bash
# Check available providers
curl http://localhost:3001/api/auth/providers

# Should return:
{
  "keycloak": {
    "id": "keycloak",
    "name": "Keycloak",
    "type": "oidc",
    "signinUrl": "http://localhost:3001/api/auth/signin/keycloak",
    "callbackUrl": "http://localhost:3001/api/auth/callback/keycloak"
  }
}
```

## Current OAuth Integration Status

### ✅ Completed (2025-08-08)
- ✅ Keycloak deployed and accessible at port 4000
- ✅ Test users configured in realm (user@ameide.io, admin@ameide.io, testuser@ameide.local)
- ✅ Client updated from public to confidential with secret
- ✅ SSO button added to login page alongside email/password option
- ✅ Dual URL configuration for internal/external access
- ✅ OAuth flow initiates successfully and redirects to Keycloak
- ✅ Keycloak login form displays correctly
- ✅ Fixed authentication flow "organization" authenticator issue
- ✅ Fixed OAuth callback configuration error
- ✅ NextAuth properly configured with separate public/internal URLs
- ✅ Session creation working after successful authentication
- ✅ User data properly extracted from Keycloak (ID, email, name)

### Working Configuration
The final working configuration uses:
- **Public issuer** as base for browser-facing discovery
- **Public URL** for authorization endpoint (browser redirect)  
- **Internal URL** for token and userinfo endpoints (server-side calls)
- **Profile mapping** to extract user data from Keycloak claims

### Helm Configuration
All OAuth configuration is now fully declarative in Helm:

#### www-ameide-canvas values (`/workspace/infra/kubernetes/values/platform/www-ameide-canvas.yaml`)
```yaml
auth:
  secret: "development-secret-change-in-production"

keycloak:
  clientId: "web-app"
  clientSecret: "development-client-secret-change-in-production"
  issuerInternal: "http://keycloak:4000/realms/ameide"  # Server-side calls
  issuerPublic: "http://localhost:4000/realms/ameide"   # Browser redirects
```

#### Keycloak Realm ConfigMap (`/workspace/infra/kubernetes/charts/platform/keycloak-realm-configmap.yaml`)
- Defines `simple-browser` authentication flow to avoid "organization" authenticator issues
- Configures web-app client as confidential with secret
- Sets up test users with credentials
- Includes all redirect URIs for ports 3000-3003

#### Secret Template (`/workspace/infra/kubernetes/charts/platform/www-ameide-canvas/templates/secret.yaml`)
```yaml
data:
  KEYCLOAK_CLIENT_SECRET: {{ .Values.keycloak.clientSecret | b64enc | quote }}
  AUTH_SECRET: {{ .Values.auth.secret | b64enc | quote }}
```

#### ConfigMap Template (`/workspace/infra/kubernetes/charts/platform/www-ameide-canvas/templates/configmap.yaml`)
```yaml
data:
  KEYCLOAK_CLIENT_ID: "{{ .Values.keycloak.clientId }}"
  KEYCLOAK_ISSUER: "{{ .Values.keycloak.issuerInternal | default .Values.keycloak.issuer }}"
  KEYCLOAK_ISSUER_PUBLIC: "{{ .Values.keycloak.issuerPublic | default .Values.keycloak.issuer }}"
```

### Implementation Notes

#### Client Configuration via Admin API
```bash
# Update client to confidential
TOKEN=$(kubectl exec -n ameide statefulset/keycloak -- curl -s -X POST \
  http://localhost:8080/realms/master/protocol/openid-connect/token \
  -d "client_id=admin-cli" -d "username=admin" -d "password=changeme" \
  -d "grant_type=password" | grep -o '"access_token":"[^"]*' | cut -d'"' -f4)

kubectl exec -n ameide statefulset/keycloak -- curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:8080/admin/realms/ameide/clients/<CLIENT_ID> \
  -d '{
    "publicClient": false,
    "secret": "development-client-secret-change-in-production"
  }'
```

#### Playwright E2E Tests
Tests created to verify OAuth flow:
- `/workspace/services/www_ameide_canvas/tests/e2e/sso-button.test.ts` - Verifies SSO button exists
- `/workspace/services/www_ameide_canvas/tests/e2e/keycloak-debug.test.ts` - Debug Keycloak redirect
- `/workspace/services/www_ameide_canvas/tests/e2e/full-oauth-login.test.ts` - Complete login flow
- `/workspace/services/www_ameide_canvas/tests/e2e/oauth-session.test.ts` - Verifies session creation

#### Application Code Changes
- **Login Page** (`/workspace/services/www_ameide_canvas/app/(auth)/login/page.tsx`): Added SSO button
- **Auth Config** (`/workspace/services/www_ameide_canvas/app/(auth)/auth.ts`): Configured Keycloak provider with dual URLs
- **API Route** (`/workspace/services/www_ameide_canvas/app/(auth)/api/auth/[...nextauth]/route.ts`): Exports NextAuth handlers

## ⚠️ Critical Configuration Limitation

**The current dual-URL OAuth configuration only works in k3d/Docker Desktop** due to their special networking where `localhost:4000` is accessible from inside pods via LoadBalancer services.

### The Issuer URL Problem
- Keycloak puts the accessed URL in the JWT's `iss` claim
- NextAuth must validate this matches the configured issuer
- If browser uses `localhost:4000` but pod uses `keycloak:4000`, tokens won't validate
- See `/workspace/infra/kubernetes/charts/platform/www-ameide-canvas/IMPORTANT-OAUTH-NOTES.md` for details

## Production Deployment Checklist

For production, you MUST solve the issuer URL problem first:

### Option A: Single URL (Recommended)
Configure DNS/ingress so `https://auth.yourdomain.com` is accessible from both browser and pods:
```yaml
keycloak:
  issuer: "https://auth.yourdomain.com/realms/ameide"
  # issuerPublic not needed - same URL works everywhere
```

### Option B: Configure Keycloak Frontend URL
Force Keycloak to use a consistent issuer:
```yaml
extraEnvVars:
  - name: KC_HOSTNAME_URL
    value: "https://auth.yourdomain.com"
  - name: KC_HOSTNAME_STRICT
    value: "false"
```

### Then Complete Setup:

1. **Generate Secure Secrets**:
   ```bash
   # Generate AUTH_SECRET
   openssl rand -base64 32
   
   # Generate Keycloak client secret in Keycloak Admin Console
   ```

2. **Update Helm Values**:
   ```yaml
   # production/www-ameide-canvas.yaml
   auth:
     secret: "<generated-secret>"
   
   keycloak:
     clientSecret: "<keycloak-generated-secret>"
     issuerPublic: "https://auth.yourdomain.com/realms/ameide"
   ```

3. **Update Keycloak Configuration**:
   - Add production redirect URIs: `https://app.yourdomain.com/*`
   - Add production web origins: `https://app.yourdomain.com`
   - Enable HTTPS required
   - Set production mode: true

4. **Use Secret Management**:
   - Consider External Secrets Operator
   - Or Sealed Secrets for GitOps
   - Never commit real secrets to git

## Future Improvements

1. **Enhanced Security**:
   - Enable HTTPS/TLS
   - Set `production: true` in Helm values
   - Configure proper hostnames
   - Enable strict hostname checking

2. **High Availability**:
   - Increase replica count
   - Configure Infinispan clustering
   - Set up session replication

3. **Security Hardening**:
   - Change default admin password
   - Configure password policies
   - Enable MFA/2FA
   - Set up proper CORS policies

4. **Integration**:
   - Configure LDAP/AD integration
   - Set up social login providers
   - Implement fine-grained authorization

5. **Monitoring**:
   - Enable metrics endpoint
   - Configure Prometheus scraping
   - Set up alerts for authentication failures

## Related Documentation
- [Bitnami Keycloak Chart](https://github.com/bitnami/charts/tree/main/bitnami/keycloak)
- [Keycloak Documentation](https://www.keycloak.org/documentation)
- [OIDC Specification](https://openid.net/connect/)

---
---

## 103-tilt-nextjs-build-optimization.md (old)

# Tilt NextJS Build Optimization

**Issue**: Every change to NextJS services triggers a full production build with optimization, causing significant delays (60-90s per change).

**Root Cause**: The current Tiltfile uses production Dockerfiles that run `npm run build`, which includes:
- Production optimizations (minification, tree-shaking)
- Static site generation
- Full bundling for production deployment

## Current Setup Problems

1. **Production Builds in Development**: 
   - Dockerfile runs `npm run build` on every change
   - Next.js production optimizer runs unnecessarily
   - No hot module replacement available

2. **No Live Updates**:
   - Live update is commented out in Tiltfile (lines 192-195)
   - Every change triggers full Docker rebuild
   - No file syncing for quick iterations

3. **Inefficient Layer Caching**:
   - Source files copied before build
   - Any source change invalidates build cache
   - Dependencies rebuilt unnecessarily

## Proposed Solutions

### Solution 1: Development Dockerfiles (Recommended - Fastest)

Create separate `Dockerfile.dev` for each NextJS service that uses development mode:

```dockerfile
# services/www-ameide/Dockerfile.dev
FROM node:18-alpine
WORKDIR /app

# Install dependencies (cached layer)
COPY package.json ./
RUN corepack enable pnpm && pnpm install --no-frozen-lockfile

# Copy source (changes frequently)
COPY . .

# Run development server with Turbopack
ENV NODE_ENV=development
ENV NEXT_TELEMETRY_DISABLED=1
EXPOSE 3000
CMD ["pnpm", "dev"]
```

**Benefits**:
- Instant hot reload with Turbopack
- No production build overhead
- Changes reflect immediately
- ~10x faster iteration

**Implementation**:
1. Create Dockerfile.dev for each service
2. Update Tiltfile `docker_build` to use Dockerfile.dev
3. Enable live_update for source syncing

### Solution 2: Enable Live Updates

Re-enable and optimize the commented live_update configuration:

```python
docker_build(
    repo,
    service_path,
    dockerfile=service_path + '/Dockerfile.dev',  # Use dev dockerfile
    live_update=[
        sync(service_path + '/src', '/app/src'),
        sync(service_path + '/public', '/app/public'),
        sync(service_path + '/app', '/app/app'),  # Next.js app directory
        run('cd /app && pnpm install --no-frozen-lockfile', trigger=[service_path + '/package.json']),
    ]
)
```

**Benefits**:
- Sub-second updates for code changes
- Only rebuilds on dependency changes
- Works with dev server hot reload

### Solution 3: Conditional Build Mode

Modify existing Dockerfiles to support both dev and production:

```dockerfile
ARG BUILD_MODE=production

# ... dependency stages ...

FROM node:18-alpine AS runner
WORKDIR /app

# Copy built app or source based on mode
ARG BUILD_MODE
RUN if [ "$BUILD_MODE" = "development" ]; then \
      echo "Development mode - skipping build"; \
    else \
      pnpm build; \
    fi

CMD [ "$BUILD_MODE" = "development" ] && pnpm dev || npm start
```

Then in Tiltfile:
```python
docker_build(
    repo,
    service_path,
    dockerfile=service_path + '/Dockerfile',
    build_args={'BUILD_MODE': 'development'}
)
```

### Solution 4: Optimize Production Dockerfile Caching

Improve the existing production Dockerfile for better caching:

```dockerfile
# Better layer caching
FROM node:18-alpine AS deps
WORKDIR /app
# Dependencies only - cached unless package files change
COPY package.json ./
RUN --mount=type=cache,target=/root/.pnpm-store \
    corepack enable pnpm && pnpm install --no-frozen-lockfile

FROM node:18-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
# Source files - separate layer
COPY . .
# Build with cache mount
RUN --mount=type=cache,target=/app/.next/cache \
    pnpm build
```

## Performance Comparison

| Approach | Initial Build | Code Change | Pros | Cons |
|----------|--------------|-------------|------|------|
| Current (Production) | 60-90s | 60-90s | Production-ready | Very slow iteration |
| Dev Dockerfile | 10-15s | <1s (hot reload) | Fastest iteration | Different from prod |
| Live Update | 60-90s | <1s | Minimal rebuilds | Complex setup |
| Conditional Build | 15-20s | 15-20s | Single Dockerfile | Still rebuilds |
| Optimized Caching | 60-90s | 20-30s | Production builds | Still slow |

## Recommended Implementation Plan

### Phase 1: Quick Win (1 hour)
1. Create Dockerfile.dev for all 4 NextJS services
2. Update Tiltfile to use dev dockerfiles
3. Test hot reload functionality

### Phase 2: Optimize (30 mins)
1. Enable live_update in Tiltfile
2. Configure proper sync patterns
3. Test file syncing

### Phase 3: Production Parity (Optional)
1. Add development vs production profiles to Tiltfile
2. Use production builds for staging/demo
3. Keep dev builds for local development

## Implementation Status

### ✅ Completed
- [x] Create services/www_ameide_platform/Dockerfile.dev (formerly www-ameide_canvas)
- [x] Update Tiltfile `deploy_nextjs_app` function to use Dockerfile.dev
- [x] Make solution generic - ANY service with Dockerfile.dev gets dev mode
- [x] Enable live_update with comprehensive directory syncing
- [x] Test hot reload with www-ameide-platform (<1s updates working!)

### Clean Architecture Achieved
The refactored solution is now:
- **Generic**: Any NextJS service with `Dockerfile.dev` automatically gets dev mode
- **No hardcoding**: Removed service-specific conditions
- **Consistent**: All dev-mode services get the same hot reload setup
- **Simple to extend**: Just add `Dockerfile.dev` to any service to enable

### To Enable Dev Mode for Other Services
Simply create a `Dockerfile.dev` in the service directory:
- [x] Create services/www-ameide/Dockerfile.dev
- ~~[ ] Create services/www-ameide_portal/Dockerfile.dev~~ (service deleted)
- ~~[ ] Create services/www-ameide_portal_canvas/Dockerfile.dev~~ (service deleted)

No Tiltfile changes needed - it will automatically detect and use dev mode!

## Expected Outcomes

**Before**:
- 60-90s per change
- Full production builds
- No hot reload
- Slow feedback loop

**After**:
- <1s for most changes
- Instant hot reload
- Fast feedback loop
- 10-100x faster iteration

## Notes

- Next.js 15 includes Turbopack which is significantly faster than Webpack
- The package.json already has `"dev": "next dev --turbopack"` configured
- Development mode skips optimizations that aren't needed locally
- Can still do production builds for final testing before deployment

## Related Files

- /workspace/Tiltfile (lines 176-232 for NextJS deployment)
- /workspace/services/www_*/Dockerfile (current production builds)
- /workspace/backlog/092-skaffold-nextjs-integration.md (similar optimization for Skaffold)
---

## 104-keycloak-config-v2.md (old)

> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# Discovery-First OAuth Configuration

✅ **COMPLETED**: 2025-01-24 (Updated 2025-01-05)

## Implementation Summary

Successfully implemented HTTPS-only Keycloak OAuth2/OIDC authentication with:
- Declarative Helm-based realm configuration (no CLI commands)
- HTTPS access through Envoy Gateway with proper redirect URIs
- Fixed invalid_scope error by removing unsupported offline_access scope
- Working authentication flow with admin/admin and user/user test accounts
- **CRITICAL FIX (2025-01-05)**: Implemented split endpoints pattern for K8s

## Key Fixes Applied
- Removed `offline_access` scope from both Keycloak realm and app configuration
- Set up HTTPRoute for auth.dev.ameide.io through Gateway
- Configured proper HTTPS redirect URIs for platform-app client
- Native Keycloak realm import via ConfigMap (replaced keycloak-config-cli)
- **Split Endpoints Pattern**: External HTTPS for browser, internal HTTP for server-to-server
- **Redis Master Fix**: Fixed authentication failures due to read-only replica
- **Keycloak Hostname Configuration**: Added KC_HOSTNAME_BACKCHANNEL_DYNAMIC=true

## Design Principles

1. **OIDC Discovery**: Standards-compliant, no manual endpoints
2. **Single Issuer**: One source of truth
3. **Security**: PKCE + state + nonce, proper TLS
4. **Minimal JWT**: < 4KB cookies
5. **Observable**: Metrics and logging

## Core Configuration Schema

```yaml
# values.yaml - Configuration only, no secrets
auth:
  url: "https://app.ameide.io"  # AUTH_URL for callbacks
  trustHost: true  # AUTH_TRUST_HOST - trust proxy headers
  redirectProxyUrl: ""  # AUTH_REDIRECT_PROXY_URL for Codespaces/preview envs
  
  session:
    maxAge: 86400  # 24 hours
    updateAge: 3600  # Update every hour
    
  # Cookie configuration for cross-subdomain sharing
  cookies:
    domain: ".ameide.io"  # Production: share across subdomains
    # domain: ""  # Local dev: no domain restriction
    sameSite: "lax"  # Use 'lax' for same-site, 'none' only for cross-site
    secure: true  # Always true in production (HTTPS required)
    httpOnly: true  # Prevent XSS attacks
    
keycloak:
  issuer: "https://auth.ameide.io/realms/ameide"  # Full issuer URL with realm
  httpTimeout: 10000  # Milliseconds
  tlsVerify: "true"  # Set to "false" only in dev with self-signed certs
```

## Cookie Strategy

### Production (Subdomain Sharing)
```javascript
// Production: Share cookies across *.ameide.io
cookies: {
  sessionToken: {
    name: '__Secure-authjs.session-token',
    options: {
      httpOnly: true,
      sameSite: 'lax',  // Same-site protection
      secure: true,     // HTTPS required
      path: '/',
      domain: '.ameide.io'  // Share across subdomains
    }
  }
}
```

### Local Development
```javascript
// Local: No domain restriction for k3d/localhost
cookies: {
  sessionToken: {
    name: 'authjs.session-token',
    options: {
      httpOnly: true,
      sameSite: 'lax',
      secure: false,  // Allow HTTP in local dev
      path: '/',
      // No domain - cookie only for exact host
    }
  }
}
```

### Cross-Site (Codespaces/Gitpod)
```javascript
// Cross-site: Different domains require SameSite=None
cookies: {
  sessionToken: {
    name: '__Secure-authjs.session-token',
    options: {
      httpOnly: true,
      sameSite: 'none',  // Required for cross-site
      secure: true,      // Required with SameSite=None
      path: '/',
      // No domain - each service gets its own cookie
    }
  }
}
```

**Key Points:**
- **Production**: `domain=.ameide.io` enables SSO across subdomains
- **SameSite=lax**: Default for same-site protection
- **SameSite=none**: Only for true cross-site scenarios (Codespaces)
- **Secure=true**: Always in production, required with SameSite=none

## Implementation in auth.ts

### Authentication Method

- **`client_secret_post`**: Credentials in request body
- **`client_secret_basic`**: Credentials in Authorization header

Both are equally secure over TLS. Use `client_secret_post` (Keycloak's default). The refresh helper below handles both methods - just set `AUTH_KEYCLOAK_TOKEN_METHOD` consistently.

```typescript
// app/(auth)/auth.ts - Discovery-first implementation with Redis
import NextAuth from 'next-auth';
import Keycloak from 'next-auth/providers/keycloak';
import https from 'https';
import crypto from 'crypto';
import { authAttempts, tokenRefreshes, sessionActive } from '@/lib/metrics';
import { 
  storeTokens, 
  getTokens, 
  acquireRefreshLock, 
  releaseRefreshLock,
  deleteTokens,
  mapSidToSessionId
} from '@/lib/session-store';
import { refreshAccessToken } from '@/lib/keycloak';

import { decodeJwt } from 'jose'; // NextAuth already depends on this

// Runtime must be Node.js for https.Agent support
export const runtime = 'nodejs';

// Auth.js v5 auto-discovers these environment variables
// AUTH_KEYCLOAK_ID and AUTH_KEYCLOAK_SECRET are picked up automatically
const keycloakConfig = {
  // Full issuer URL including realm - discovery handles everything else
  issuer: process.env.AUTH_KEYCLOAK_ISSUER!,
  clientId: process.env.AUTH_KEYCLOAK_ID!,
  clientSecret: process.env.AUTH_KEYCLOAK_SECRET!,
};

// Use the standard Keycloak provider with full security
const keycloakProvider = Keycloak({
  ...keycloakConfig,
  
  // Full security checks
  checks: ['state', 'pkce', 'nonce'],
    
  // Request offline tokens for automatic refresh
  authorization: {
    params: { 
      scope: 'openid profile email offline_access',
    },
  },
    
  // Authentication method - consistent with refresh code
  client: {
    token_endpoint_auth_method: process.env.AUTH_KEYCLOAK_TOKEN_METHOD || 'client_secret_post',
  },
    
  // HTTP options (use function form for dynamic config)
  httpOptions: () => ({
    timeout: Number(process.env.AUTH_KEYCLOAK_HTTP_TIMEOUT ?? 10000),
    // Dev only: self-signed certs
    ...(process.env.NODE_ENV !== 'production' && process.env.KEYCLOAK_TLS_VERIFY === 'false' && {
      agent: new https.Agent({
        rejectUnauthorized: false,
      }),
    }),
  }),
    
  // Profile mapping - minimal data
  profile(profile) {
    return {
      id: profile.sub,
      name: profile.name ?? profile.preferred_username,
      email: profile.email,
      image: profile.picture,
    };
  },
});

export const { auth, signIn, signOut, handlers } = NextAuth({
  providers: [keycloakProvider],
  
  secret: process.env.AUTH_SECRET,
  
  // Session configuration
  session: {
    strategy: 'jwt',
    maxAge: Number(process.env.AUTH_SESSION_MAX_AGE ?? 86400),
    updateAge: Number(process.env.AUTH_SESSION_UPDATE_AGE ?? 3600),
  },
  
  callbacks: {
    async jwt({ token, account, user, profile }) {
      // Initial sign in
      if (account && user) {
        // Store tokens in Redis, keep only pointer in JWT
        const sessionId = crypto.randomUUID();
        
        // Extract Keycloak SID from id_token for back-channel logout
        const idTokenDecoded = decodeJwt(account.id_token!) as KeycloakIdToken;
        const sid = idTokenDecoded.sid || idTokenDecoded.session_state;
        
        // Calculate refresh token expiry (Keycloak typically uses 30 days)
        const refreshExpiresAt = account.refresh_expires_in 
          ? Date.now() + account.refresh_expires_in * 1000
          : Date.now() + 30 * 24 * 60 * 60 * 1000; // Default 30 days
        
        await storeTokens(sessionId, {
          accessToken: account.access_token,
          refreshToken: account.refresh_token,
          idToken: account.id_token,
          expiresAt: account.expires_at! * 1000,
          refreshExpiresAt,
          sid: sid, // Store SID with tokens for cleanup
        });
        
        // Map Keycloak SID to our sessionId for back-channel logout
        // Use same TTL as tokens to ensure mapping persists as long as session is valid
        if (sid) {
          const ttlSeconds = Math.ceil((refreshExpiresAt - Date.now()) / 1000);
          await mapSidToSessionId(sid, sessionId, ttlSeconds);
        }
        
        // JWT only contains pointer and minimal data (< 1KB)
        token.sessionId = sessionId;
        token.expiresAt = account.expires_at! * 1000;
        
        // Extract roles from access token with type safety
        const decoded = decodeJwt(account.access_token!) as KeycloakAccessToken;
        const roles = decoded.realm_access?.roles ?? [];
        const clientId = process.env.AUTH_KEYCLOAK_ID!;
        const clientRoles = decoded.resource_access?.[clientId]?.roles ?? [];
        token.isAdmin = roles.includes('admin') || clientRoles.includes('admin');
        return token;
      }
      
      // Check if token needs refresh
      const SKEW_MS = 10_000;
      if (Date.now() + SKEW_MS >= (token.expiresAt as number) && token.sessionId) {
        // Prevent thundering herd with distributed lock
        const hasLock = await acquireRefreshLock(token.sessionId as string);
        if (!hasLock) {
          // Another request is refreshing, skip
          return token;
        }
        
        try {
          const stored = await getTokens(token.sessionId as string);
          if (!stored?.refreshToken) throw new Error('No refresh token');
          
          const startTime = Date.now();
          const refreshed = await refreshAccessToken(stored.refreshToken);
          const duration = (Date.now() - startTime) / 1000;
          tokenRefreshLatency?.observe(duration);
          
          // Update refresh expiry if provided
          const refreshExpiresAt = refreshed.refresh_expires_in 
            ? Date.now() + refreshed.refresh_expires_in * 1000
            : stored.refreshExpiresAt; // Preserve existing if not provided
          
          // Update stored tokens with proper TTL
          await storeTokens(token.sessionId as string, {
            accessToken: refreshed.access_token,
            refreshToken: refreshed.refresh_token ?? stored.refreshToken,
            idToken: refreshed.id_token ?? stored.idToken,
            expiresAt: Date.now() + refreshed.expires_in * 1000,
            refreshExpiresAt,
            sid: stored.sid, // Preserve SID
          });
          
          // Extend SID mapping TTL on successful refresh
          if (stored.sid) {
            const ttlSeconds = Math.ceil((refreshExpiresAt - Date.now()) / 1000);
            await mapSidToSessionId(stored.sid, token.sessionId as string, ttlSeconds);
          }
          
          token.expiresAt = Date.now() + refreshed.expires_in * 1000;
          tokenRefreshes?.labels({ status: 'success', error_code: '' }).inc();
        } catch (e: any) {
          token.error = 'RefreshAccessTokenError';
          
          // Log specific error for debugging
          const errorCode = e?.error || e?.code || 'unknown';
          tokenRefreshes?.labels({ status: 'error', error_code: errorCode }).inc();
          
          console.error('[AUTH] Token refresh failed:', {
            error: errorCode,
            description: e?.error_description || e?.message,
            sessionId: token.sessionId,
          });
          
          // Clean up Redis on permanent failure (e.g., invalid_grant)
          if (errorCode === 'invalid_grant' || errorCode === 'invalid_client') {
            await deleteTokens(token.sessionId as string);
          }
        } finally {
          await releaseRefreshLock(token.sessionId as string);
        }
      }
      
      return token;
    },
    
    async session({ session, token }) {
      // Minimal session data to keep cookies small
      session.user.id = token.sub!;
      session.user.isAdmin = token.isAdmin as boolean;
      // Surface auth errors to the client
      if (token.error) {
        (session as any).error = token.error;
      }
      // Don't expose tokens in session - fetch when needed
      
      return session;
    },
  },
  
  // Event logging with metrics
  events: {
    async signIn({ user, account }) {
      console.log('[AUTH] User signed in', {
        userId: user.id,
        provider: account?.provider,
        timestamp: new Date().toISOString(),
      });
      authAttempts?.labels({ provider: 'keycloak', status: 'success' }).inc();
      sessionActive?.inc();
    },
    async signOut({ token }) {
      console.log('[AUTH] User signed out', {
        userId: token?.sub,
        timestamp: new Date().toISOString(),
      });
      sessionActive?.dec();
    },
    async error(error) {
      console.error('[AUTH] Authentication error', error);
      authAttempts?.labels({ provider: 'keycloak', status: 'error' }).inc();
    },
  },
  
  // Custom pages
  pages: {
    signIn: '/auth/signin',
    error: '/auth/error',
  },
  
  // Cookie security
  cookies: {
    sessionToken: {
      name: process.env.NODE_ENV === 'production' 
        ? '__Secure-authjs.session-token' 
        : 'authjs.session-token',
      options: {
        httpOnly: true,
        // Cross-site cookie configuration
        // Use 'none' for Codespaces/Gitpod where auth UI is on different domain
        sameSite: process.env.AUTH_CROSS_SITE === 'true' ? 'none' : 'lax',
        path: '/',
        // Must be true when sameSite is 'none'
        secure: process.env.NODE_ENV === 'production' || process.env.AUTH_CROSS_SITE === 'true',
      }
    }
  },
});
```

## NextAuth Route Handler

```typescript
// app/api/auth/[...nextauth]/route.ts
import { handlers } from '@/app/(auth)/auth';

export const runtime = 'nodejs'; // Required for Node.js APIs

export const { GET, POST } = handlers;
```

## Back-Channel Logout

```typescript
// app/api/auth/backchannel-logout/route.ts - Clear sessions when Keycloak notifies
import { createRemoteJWKSet, jwtVerify } from 'jose';
import { deleteTokens, getSessionIdBySid } from '@/lib/session-store';
import { sessionActive } from '@/lib/metrics';

const JWKS = createRemoteJWKSet(
  new URL(`${process.env.AUTH_KEYCLOAK_ISSUER}/protocol/openid-connect/certs`)
);

export async function POST(req: Request) {
  try {
    const formData = await req.formData();
    const logoutToken = formData.get('logout_token') as string | null;
    
    if (!logoutToken) {
      return new Response('Ignored', { status: 204 });
    }
    
    const { payload } = await jwtVerify(logoutToken, JWKS, {
      issuer: process.env.AUTH_KEYCLOAK_ISSUER,
      audience: process.env.AUTH_KEYCLOAK_ID,
    });
    
    // Verify this is a logout event token
    const isLogout = (payload as any)?.events?.['http://schemas.openid.net/event/backchannel_logout'] !== undefined;
    if (!isLogout) {
      return new Response('Ignored', { status: 204 });
    }
    
    // Look up our sessionId using Keycloak's SID
    const sid = payload.sid as string | undefined;
    if (sid) {
      const sessionId = await getSessionIdBySid(sid);
      if (sessionId) {
        await deleteTokens(sessionId);
        // For back-channel logout, we DO need to decrement since signOut event won't fire
        sessionActive?.dec();
      }
    }
    
    return new Response('OK');
  } catch (e) {
    console.error('[AUTH] Backchannel logout error', e);
    // Best-effort semantics: don't break Keycloak with 500s
    return new Response('Ignored', { status: 204 });
  }
}
```

Configure in Keycloak: Backchannel Logout URL = `https://app.ameide.io/api/auth/backchannel-logout`

## Federated Logout Implementation

```typescript
// app/api/auth/federated-logout/route.ts
import { auth, signOut } from '@/app/(auth)/auth';
import { getToken } from 'next-auth/jwt';
import { getTokens, deleteTokens } from '@/lib/session-store';
import { sessionActive } from '@/lib/metrics';
import type { NextRequest } from 'next/server';

export const runtime = 'nodejs';

export async function GET(req: NextRequest) {
  try {
    // Don't call auth() - it may redirect if session is expired
    const jwt = await getToken({ req, secret: process.env.AUTH_SECRET });
    
    if (!jwt) {
      // User not authenticated, just redirect home
      return Response.redirect(process.env.AUTH_URL || '/');
    }
    
    const issuer = process.env.AUTH_KEYCLOAK_ISSUER;
    const authUrl = process.env.AUTH_URL || '/';
    const clientId = process.env.AUTH_KEYCLOAK_ID;
    const clientSecret = process.env.AUTH_KEYCLOAK_SECRET;
    
    if (!issuer || !authUrl || !clientId) {
      await signOut({ redirect: false });
      return Response.redirect(authUrl);
    }
    
    // Fetch tokens from Redis using sessionId (not from JWT!)
    let refreshToken: string | undefined;
    let idTokenHint: string | undefined;
    if (jwt?.sessionId) {
      const stored = await getTokens(jwt.sessionId as string);
      refreshToken = stored?.refreshToken;
      idTokenHint = stored?.idToken;
    }
    
    // Best-effort revoke refresh token
    if (refreshToken && clientSecret) {
      try {
        const revokeUrl = `${issuer}/protocol/openid-connect/revoke`;
        await fetch(revokeUrl, {
          method: 'POST',
          headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
          body: new URLSearchParams({
            client_id: clientId,
            client_secret: clientSecret,
            token: refreshToken,
            token_type_hint: 'refresh_token',
          }),
        });
      } catch {}
    }
    
    const endSessionUrl = `${issuer}/protocol/openid-connect/logout`;
    const params = new URLSearchParams({
      post_logout_redirect_uri: authUrl,
      client_id: clientId,
    });
    if (idTokenHint) params.set('id_token_hint', idTokenHint);
    
    // Clear local session + Redis
    await signOut({ redirect: false });  // This triggers events.signOut which decrements sessionActive
    if (jwt?.sessionId) {
      await deleteTokens(jwt.sessionId as string);
      // Don't decrement sessionActive here - signOut event already does it
    }
    
    return Response.redirect(`${endSessionUrl}?${params}`);
  } catch {
    await signOut({ redirect: false });
    return Response.redirect(process.env.AUTH_URL || '/');
  }
}
```

## Session Storage for <4KB Cookies (Redis or Database)

Real Keycloak tokens exceed 4KB cookie limits. Use Redis (shown here) or database sessions to store tokens server-side:

```typescript
// lib/session-store.ts
import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL);

export async function storeTokens(sessionId: string, tokens: any) {
  // Always use refresh token expiry for TTL to ensure tokens remain available
  const refreshExpiresAt = tokens.refreshExpiresAt || Date.now() + 30 * 24 * 60 * 60 * 1000;
  const ttl = Math.ceil((refreshExpiresAt - Date.now()) / 1000);
  
  // Cap at 30 days to prevent indefinite storage
  const finalTtl = Math.min(30 * 86400, ttl);
  
  await redis.setex(`session:${sessionId}`, finalTtl, JSON.stringify(tokens));
}

export async function getTokens(sessionId: string) {
  const data = await redis.get(`session:${sessionId}`);
  return data ? JSON.parse(data) : null;
}

// Map Keycloak SID to our sessionId for back-channel logout
export async function mapSidToSessionId(sid: string, sessionId: string, ttlSec: number) {
  await redis.setex(`sid:${sid}`, ttlSec, sessionId);
}

export async function getSessionIdBySid(sid: string) {
  return redis.get(`sid:${sid}`);
}

// Prevent refresh thundering herd
export async function acquireRefreshLock(sessionId: string): Promise<boolean> {
  // Lock TTL should be longer than HTTP timeout to prevent concurrent refreshes
  const httpMs = Number(process.env.AUTH_KEYCLOAK_HTTP_TIMEOUT ?? 10000);
  const ttlSec = Math.ceil(httpMs / 1000) + 2; // e.g., 12s for 10s timeout
  const result = await redis.set(`session:${sessionId}:refreshLock`, '1', 'EX', ttlSec, 'NX');
  return result === 'OK';
}

export async function releaseRefreshLock(sessionId: string) {
  await redis.del(`session:${sessionId}:refreshLock`);
}

// Clean up stored tokens on logout or permanent failure
export async function deleteTokens(sessionId: string) {
  // Also clean up any SID mappings
  const tokens = await getTokens(sessionId);
  if (tokens?.sid) {
    await redis.del(`sid:${tokens.sid}`);
  }
  await redis.del(`session:${sessionId}`, `session:${sessionId}:refreshLock`);
}

// lib/keycloak.ts - Shared token refresh helper
export async function refreshAccessToken(refreshToken: string) {
  const tokenEndpoint = `${process.env.AUTH_KEYCLOAK_ISSUER!}/protocol/openid-connect/token`;
  const method = process.env.AUTH_KEYCLOAK_TOKEN_METHOD ?? 'client_secret_post';

  const headers: Record<string,string> = { 'Content-Type':'application/x-www-form-urlencoded' };
  const body = new URLSearchParams({ grant_type:'refresh_token', refresh_token: refreshToken });

  if (method === 'client_secret_basic') {
    headers.Authorization = 'Basic ' + Buffer
      .from(`${process.env.AUTH_KEYCLOAK_ID!}:${process.env.AUTH_KEYCLOAK_SECRET!}`)
      .toString('base64');
  } else {
    body.set('client_id', process.env.AUTH_KEYCLOAK_ID!);
    body.set('client_secret', process.env.AUTH_KEYCLOAK_SECRET!);
  }

  // Add timeout to prevent hanging
  const ac = new AbortController();
  const timeout = setTimeout(() => ac.abort(), Number(process.env.AUTH_KEYCLOAK_HTTP_TIMEOUT ?? 10000));
  
  try {
    const res = await fetch(tokenEndpoint, { 
      method: 'POST', 
      headers, 
      body, 
      signal: ac.signal 
    });
    const json = await res.json();
    
    if (!res.ok) {
      // Log www-authenticate header for debugging
      const wwwAuth = res.headers.get('www-authenticate');
      if (wwwAuth) {
        console.error('[AUTH] Refresh failed with WWW-Authenticate:', wwwAuth);
      }
      throw json;
    }
    
    return json; // { access_token, refresh_token, id_token, expires_in, ... }
  } finally {
    clearTimeout(timeout);
  }
}

// lib/access-token.ts - ALWAYS use this helper to get valid access tokens
// Never access Redis directly from API routes or server actions!
import { getTokens, storeTokens } from './session-store';
import { tokenRefreshes } from './metrics';
import { refreshAccessToken } from './keycloak';

export async function withAccessToken(sessionId: string, fn: (token: string) => Promise<any>) {
  const tokens = await getTokens(sessionId);
  if (!tokens) throw new Error('Session missing');
  
  let accessToken = tokens.accessToken;
  
  // Proactively refresh if near expiry
  const SKEW_MS = 10_000;
  if (Date.now() + SKEW_MS >= tokens.expiresAt && tokens.refreshToken) {
    const refreshed = await refreshAccessToken(tokens.refreshToken);
    
    // Update refresh expiry if provided
    const refreshExpiresAt = refreshed.refresh_expires_in 
      ? Date.now() + refreshed.refresh_expires_in * 1000
      : tokens.refreshExpiresAt;
    
    await storeTokens(sessionId, {
      ...tokens,
      accessToken: refreshed.access_token,
      refreshToken: refreshed.refresh_token ?? tokens.refreshToken,
      idToken: refreshed.id_token ?? tokens.idToken,
      expiresAt: Date.now() + refreshed.expires_in * 1000,
      refreshExpiresAt,
    });
    
    // Extend SID mapping if present
    if (tokens.sid) {
      const ttlSeconds = Math.ceil((refreshExpiresAt - Date.now()) / 1000);
      await mapSidToSessionId(tokens.sid, sessionId, ttlSeconds);
    }
    
    accessToken = refreshed.access_token;
    tokenRefreshes?.labels({ status: 'success', error_code: '' }).inc();
  }
  
  return fn(accessToken);
}

// Example usage in API routes:
// export async function GET(req: Request) {
//   const session = await auth();
//   const jwt = await getToken({ req, secret: process.env.AUTH_SECRET });
//   
//   return withAccessToken(jwt.sessionId, async (token) => {
//     // Use the fresh access token
//     const response = await fetch('https://api.example.com/data', {
//       headers: { Authorization: `Bearer ${token}` }
//     });
//     return response.json();
//   });
// }

// app/(auth)/auth.ts - pointer session approach
callbacks: {
  async jwt({ token, account, user }) {
    if (account && user) {
      // Store tokens in Redis, not JWT
      const sessionId = crypto.randomUUID();
      await storeTokens(sessionId, {
        accessToken: account.access_token,
        refreshToken: account.refresh_token,
        idToken: account.id_token,
        expiresAt: account.expires_at! * 1000,
      });
      
      // JWT only contains pointer and minimal data
      token.sessionId = sessionId;
      token.isAdmin = /* decode and check */;
      // Cookie stays under 1KB
    }
    return token;
  },
}
```

## Observability Implementation

```typescript
// lib/metrics.ts - Shared metrics instance
import { Counter, Histogram, Gauge, collectDefaultMetrics, register } from 'prom-client';

// Initialize once
if (!(global as any).__metricsInit) {
  collectDefaultMetrics();
  (global as any).__metricsInit = true;
}

export const authAttempts = new Counter({
  name: 'auth_attempts_total',
  help: 'Total number of authentication attempts',
  labelNames: ['provider', 'status'],
});

export const tokenRefreshes = new Counter({
  name: 'auth_token_refreshes_total',
  help: 'Total number of token refresh attempts',
  labelNames: ['status', 'error_code'],
});

export const tokenRefreshLatency = new Histogram({
  name: 'auth_token_refresh_duration_seconds',
  help: 'Token refresh latency in seconds',
  buckets: [0.1, 0.5, 1, 2, 5, 10],
});

export const sessionActive = new Gauge({
  name: 'auth_sessions_active',
  help: 'Number of active sessions',
});

// app/api/metrics/route.ts
import { register } from 'prom-client';
import '@/lib/metrics'; // Initialize metrics

export const runtime = 'nodejs'; // Required for prom-client
export const dynamic = 'force-dynamic';

export async function GET(req: Request) {
  // Protect metrics endpoint (use network policy or basic auth)
  // Example: Check for internal network or monitoring service
  const auth = req.headers.get('authorization');
  if (process.env.NODE_ENV === 'production' && !auth?.startsWith('Basic ')) {
    return new Response('Unauthorized', { status: 401 });
  }
  
  const metrics = await register.metrics();
  return new Response(metrics, {
    headers: {
      'Content-Type': register.contentType,
      'Cache-Control': 'no-store',
    },
  });
}

// app/(auth)/auth.ts
import { authAttempts, tokenRefreshes, tokenRefreshLatency } from '@/lib/metrics';
```

## Gateway API Configuration (Replaces Ingress)

```yaml
# templates/httproute.yaml - Application route
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: {{ include "www-ameide-canvas.fullname" . }}
  namespace: {{ .Release.Namespace }}
spec:
  parentRefs:
    - name: ameide
      namespace: ameide  # Central Gateway namespace
  hostnames:
    - app.ameide.io
  rules:
    - backendRefs:
        - name: {{ include "www-ameide-canvas.fullname" . }}
          port: {{ .Values.service.port | default 3004 }}
---
# If app and Gateway are in different namespaces, enable cross-namespace
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-gateway-to-{{ include "www-ameide-canvas.fullname" . }}
  namespace: {{ .Release.Namespace }}
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: ameide
  to:
    - group: ""
      kind: Service
      name: {{ include "www-ameide-canvas.fullname" . }}
```

### Keycloak HTTPRoute

```yaml
# Keycloak service route (separate chart)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: keycloak
  namespace: {{ .Release.Namespace }}  # Where Keycloak runs
spec:
  parentRefs:
    - name: ameide
      namespace: ameide  # Central Gateway
  hostnames:
    - auth.ameide.io
  rules:
    - backendRefs:
        - name: keycloak
          port: 80  # or 8080 depending on your service
---
# Enable cross-namespace if needed
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-gateway-to-keycloak
  namespace: {{ .Release.Namespace }}
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: ameide
  to:
    - group: ""
      kind: Service
      name: keycloak
```

## Helm Templates

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "www-ameide-canvas.fullname" . }}-config
data:
  # Auth.js v5 configuration
  AUTH_URL: {{ .Values.auth.url | quote }}
  AUTH_TRUST_HOST: {{ .Values.auth.trustHost | quote }}
  {{- if .Values.auth.redirectProxyUrl }}
  AUTH_REDIRECT_PROXY_URL: {{ .Values.auth.redirectProxyUrl | quote }}
  {{- end }}
  
  # Keycloak provider configuration
  AUTH_KEYCLOAK_ISSUER: {{ .Values.keycloak.issuer | quote }}
  # Note: AUTH_KEYCLOAK_ID and AUTH_KEYCLOAK_SECRET come from Secret
  
  # HTTP timeout for discovery calls
  AUTH_KEYCLOAK_HTTP_TIMEOUT: {{ .Values.keycloak.httpTimeout | default "10000" | quote }}
  # Token endpoint authentication method
  AUTH_KEYCLOAK_TOKEN_METHOD: {{ .Values.keycloak.tokenAuthMethod | default "client_secret_post" | quote }}
  
  # Session configuration
  AUTH_SESSION_MAX_AGE: {{ .Values.auth.session.maxAge | quote }}
  AUTH_SESSION_UPDATE_AGE: {{ .Values.auth.session.updateAge | quote }}
  
  {{- if .Values.keycloak.tlsVerify }}
  KEYCLOAK_TLS_VERIFY: {{ .Values.keycloak.tlsVerify | quote }}
  {{- end }}
```

```yaml
# templates/externalsecret.yaml - Declarative secret management
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ include "www-ameide-canvas.fullname" . }}-auth
spec:
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: {{ include "www-ameide-canvas.fullname" . }}-auth
  data:
    - secretKey: AUTH_SECRET
      remoteRef:
        key: ameide/auth
        property: auth_secret
    - secretKey: AUTH_KEYCLOAK_ID
      remoteRef:
        key: ameide/keycloak
        property: client_id
    - secretKey: AUTH_KEYCLOAK_SECRET
      remoteRef:
        key: ameide/keycloak
        property: client_secret
```

## Environment-Specific Values

### Local Development (k3d)
```yaml
# environments/local/values.yaml
# Requires: echo "127.0.0.1 auth.ameide.io app.ameide.io" >> /etc/hosts
keycloak:
  clientId: "web-app"
  issuer: "http://auth.ameide.io:8080/realms/ameide"  # Same hostname as prod, HTTP for local
  httpTimeout: 10000
  tokenAuthMethod: "client_secret_post"  # Keycloak default
  tlsVerify: "true"  # Local uses real certs or HTTP

auth:
  url: "http://app.ameide.io:3004"  # Consistent domain, different port
  trustHost: true
  session:
    maxAge: 86400
    updateAge: 3600
```

### Dev Container
```yaml
# environments/devcontainer/values.yaml
# Requires extra_hosts in docker-compose (see below)
keycloak:
  clientId: "web-app"
  issuer: "http://auth.ameide.io:8080/realms/ameide"  # Same as local dev
  tlsVerify: "false"  # Only for dev
  httpTimeout: 10000
  tokenAuthMethod: "client_secret_post"

auth:
  url: "http://app.ameide.io:3004"
  trustHost: true
  session:
    maxAge: 86400
    updateAge: 3600
```

### GitHub Codespaces
```yaml
# environments/codespaces/values.yaml
keycloak:
  issuer: "${CODESPACE_KEYCLOAK_URL}/realms/ameide"  # e.g., https://username-repo-8080.githubpreview.dev/realms/ameide
  httpTimeout: 10000

auth:
  url: "${CODESPACE_APP_URL}"  # e.g., https://username-repo-3004.githubpreview.dev
  trustHost: true  # Trust Codespace proxy headers
  redirectProxyUrl: "https://auth-proxy.ameide.io"  # Stable URL for OAuth callbacks
  session:
    maxAge: 86400
    updateAge: 3600
```

### Production
```yaml
# environments/production/values.yaml
keycloak:
  clientId: "web-app"
  issuer: "https://auth.ameide.io/realms/ameide"  # Full issuer URL
  httpTimeout: 10000
  tokenAuthMethod: "client_secret_post"  # Be consistent with refresh code
  tlsVerify: "true"  # Always verify in production

auth:
  url: "https://app.ameide.io"
  trustHost: false  # Enforce proper URLs
  session:
    maxAge: 28800  # 8 hours
    updateAge: 1800  # 30 minutes
```

## Keycloak Server Configuration

```yaml
# Keycloak Helm values for consistent issuer
extraEnvVars:
  # Hostname configuration
  - name: KC_HOSTNAME
    value: "auth.ameide.io"
  - name: KC_HOSTNAME_ADMIN
    value: "auth.ameide.io"
  - name: KC_HOSTNAME_STRICT
    value: "true"
  - name: KC_PROXY
    value: "edge"  # TLS terminated at Gateway
  - name: KC_PROXY_HEADERS
    value: "xforwarded"  # Gateway sets X-Forwarded-* headers
  
  # Always HTTP behind Gateway (TLS at Gateway layer)
  - name: KC_HTTP_ENABLED
    value: "true"  # Gateway handles TLS, Keycloak runs HTTP
```

## Gateway API Migration Considerations

### TLS Strategy
- **TLS Termination**: Always at Gateway level using wildcard certificate
- **Backend Protocol**: Services run HTTP behind Gateway (simpler, no double TLS)
- **Certificate Management**: Single `ameide-wildcard-tls` secret in Gateway namespace

### Required Changes for Gateway API

1. **Replace Ingress with HTTPRoute**:
   - All services need HTTPRoute resources
   - ReferenceGrants for cross-namespace routing
   - No more NGINX-specific annotations

2. **Gateway Configuration** (already in helmfile):
   ```yaml
   # Central Gateway in ameide namespace
   - name: gateway
     chart: ./charts/platform/gateway
     needs:
       - ameide/envoy-gateway
   ```

3. **Service Updates**:
   - Remove Ingress templates
   - Add HTTPRoute templates
   - Update service type to ClusterIP (no LoadBalancer needed)

4. **Environment Variables**:
   - Keep using service DNS for internal communication
   - Public URLs unchanged (auth.ameide.io, app.ameide.io)
   - AUTH_KEYCLOAK_ISSUER uses public URL for browser flows

### Refactoring Checklist

#### Immediate Actions Required

- [ ] **Update all Helm charts** to use HTTPRoute instead of Ingress:
  - `www-ameide-canvas` → HTTPRoute template
  - `www-ameide` → HTTPRoute template  
  - `www-ameide-portal` → HTTPRoute template
  - `www-ameide-portal-canvas` → HTTPRoute template
  - `keycloak` → HTTPRoute template

- [ ] **Create Gateway chart** (`charts/platform/gateway/`):
  ```yaml
  # templates/gateway.yaml
  apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
    name: ameide
    namespace: ameide
  spec:
    gatewayClassName: envoy-gateway
    listeners:
      - name: http
        protocol: HTTP
        port: 80
      - name: https
        protocol: HTTPS
        port: 443
        tls:
          mode: Terminate
          certificateRefs:
            - name: ameide-wildcard-tls
  ```

- [ ] **Update cert-manager configuration**:
  - Certificate issued to Gateway namespace
  - Single wildcard cert for all subdomains
  - Remove per-service TLS configs

#### Testing Strategy

1. **Local k3d Testing**:
   ```bash
   # Deploy Gateway first
   helmfile -e local apply -l component=gateway
   
   # Then deploy services with HTTPRoutes
   helmfile -e local apply -l layer=application
   
   # Verify routing
   curl -H "Host: auth.ameide.io" http://localhost/realms/ameide/.well-known/openid-configuration
   curl -H "Host: app.ameide.io" http://localhost/
   ```

2. **Migration Path**:
   - Keep Ingress resources temporarily (mark deprecated)
   - Deploy Gateway + HTTPRoutes in parallel
   - Test with Gateway endpoints
   - Remove Ingress after validation

### Benefits of Gateway API for Auth

1. **Unified TLS Management**: Single wildcard cert at Gateway
2. **Better Observability**: Built-in metrics from Envoy Gateway
3. **Advanced Features**: 
   - Request/response transformations
   - Fine-grained traffic policies
   - Native gRPC-Web support (no separate Envoy needed)
4. **Security**: 
   - Automatic HSTS headers
   - Built-in rate limiting
   - OAuth2 filter support (future enhancement)

### No Code Changes Required

The beauty of this migration is that **zero application code changes** are needed:
- Auth.js configuration remains identical
- Keycloak issuer URLs unchanged
- Cookie and session handling unaffected
- Only infrastructure (Helm) templates change

## Dev Container Setup

```yaml
# .devcontainer/docker-compose.yml
services:
  app:
    build: .
    volumes:
      - ..:/workspace:cached
    extra_hosts:
      # Map ameide domains to host
      - "auth.ameide.io:host-gateway"
      - "app.ameide.io:host-gateway"
    environment:
      AUTH_KEYCLOAK_ISSUER: "http://auth.ameide.io:8080/realms/ameide"
      AUTH_KEYCLOAK_ID: "web-app"
      AUTH_KEYCLOAK_SECRET: "dev-secret"
      AUTH_URL: "http://app.ameide.io:3004"
      AUTH_SECRET: "dev-secret-at-least-32-chars-long"
      AUTH_TRUST_HOST: "true"
      KEYCLOAK_TLS_VERIFY: "false"
    
  keycloak:
    image: quay.io/keycloak/keycloak:26.1.0  # Pin version to avoid surprises
    ports:
      - "8080:8080"  # Standard Keycloak port
    environment:
      KC_HOSTNAME: "auth.ameide.io"
      KC_HOSTNAME_PORT: "8080"
      KC_HTTP_ENABLED: "true"
      KC_HEALTH_ENABLED: "true"
      KEYCLOAK_ADMIN: "admin"
      KEYCLOAK_ADMIN_PASSWORD: "admin"
    command: start-dev
```

```json
// .devcontainer/devcontainer.json
{
  "name": "AMEIDE Canvas Dev",
  "dockerComposeFile": "docker-compose.yml",
  "service": "app",
  "workspaceFolder": "/workspace",
  
  "forwardPorts": [3004, 8080],
  
  "postCreateCommand": "pnpm install --no-frozen-lockfile",
  
  "remoteEnv": {
    // Override for Codespaces
    "AUTH_KEYCLOAK_ISSUER": "${CODESPACE_NAME:+https://$CODESPACE_NAME-8080.githubpreview.dev/realms/ameide}",
    "AUTH_URL": "${CODESPACE_NAME:+https://$CODESPACE_NAME-3004.githubpreview.dev}"
  }
}
```

## Cross-Site Cookie Configuration

For environments where the authentication UI and application run on different domains (e.g., GitHub Codespaces, Gitpod), you need to configure cross-site cookies:

### Environment Detection

```typescript
// Detect if running in cross-site environment
const isCrossSite = () => {
  // Codespaces: Different preview domains for each port
  if (process.env.CODESPACE_NAME) return true;
  
  // Gitpod: Different workspace URLs
  if (process.env.GITPOD_WORKSPACE_URL) return true;
  
  // Custom cross-site flag
  if (process.env.AUTH_CROSS_SITE === 'true') return true;
  
  return false;
};
```

### Cookie Configuration

```yaml
# Environment variables for cross-site scenarios
AUTH_CROSS_SITE: "true"  # Enable for Codespaces/Gitpod
```

```typescript
// Dynamic cookie configuration based on environment
cookies: {
  sessionToken: {
    name: process.env.NODE_ENV === 'production' 
      ? '__Secure-authjs.session-token' 
      : 'authjs.session-token',
    options: {
      httpOnly: true,
      sameSite: isCrossSite() ? 'none' : 'lax',
      secure: process.env.NODE_ENV === 'production' || isCrossSite(),
      path: '/',
      // Optional: Set domain for cookie sharing across subdomains
      ...(process.env.COOKIE_DOMAIN && { domain: process.env.COOKIE_DOMAIN }),
    }
  }
}
```

### Important Notes

- **SameSite=None requires HTTPS**: Must set `secure: true`
- Only use for cross-site scenarios (Codespaces/Gitpod)
- Some browsers block third-party cookies by default

## TypeScript Types

```typescript
// types/auth.d.ts
import 'next-auth';
import 'next-auth/jwt';

declare module 'next-auth' {
  interface Session {
    user: {
      id: string;
      name?: string | null;
      email?: string | null;
      image?: string | null;
      isAdmin: boolean;
    };
    error?: string; // Surface authentication errors to the client
  }
}

declare module 'next-auth/jwt' {
  interface JWT {
    sub: string;
    sessionId?: string;  // Pointer to Redis session
    expiresAt?: number;  // Access token expiry
    isAdmin: boolean;
    error?: string;
  }
}

// Type-safe token decoding
interface KeycloakAccessToken {
  sub: string;
  iss: string;
  aud: string | string[];
  exp: number;
  iat: number;
  realm_access?: {
    roles: string[];
  };
  resource_access?: {
    [clientId: string]: {
      roles: string[];
    };
  };
}

interface KeycloakIdToken {
  sub: string;
  sid?: string;
  session_state?: string;
  name?: string;
  preferred_username?: string;
  email?: string;
  picture?: string;
}

// Type guards
function isKeycloakAccessToken(token: unknown): token is KeycloakAccessToken {
  return typeof token === 'object' && token !== null && 'sub' in token && 'iss' in token;
}

function isKeycloakIdToken(token: unknown): token is KeycloakIdToken {
  return typeof token === 'object' && token !== null && 'sub' in token;
}
```

## Key Principles

1. **Pure OIDC Discovery**: Single issuer URL, no manual endpoint configuration
2. **Full Issuer URLs**: Always include `/realms/{realm}` in the issuer
3. **Minimal JWT Sessions**: Store only user ID and essential flags
4. **Complete Security**: PKCE + state + nonce checks always enabled
5. **Database Sessions for Production**: Avoid JWT size limits entirely
6. **Standard Ports**: Keycloak on 8080, apps on their designated ports
7. **AUTH_URL**: Auth.js v5 naming convention
8. **Error Resilience**: Graceful fallbacks when dependencies unavailable

## Deployment

```bash
# Deploy with Helm - secrets created by ExternalSecret
helm upgrade --install www-ameide-canvas ./charts/platform/www-ameide-canvas \
  -f environments/production/values.yaml \
  --namespace ameide

# Verify discovery endpoint
curl https://auth.ameide.io/realms/ameide/.well-known/openid-configuration
```

## Benefits

1. **True Standards Compliance**: Pure OIDC discovery, no custom logic
2. **Reduced Complexity**: Single issuer URL, no endpoint juggling
3. **Better Performance**: Smaller JWT cookies, faster auth
4. **Production Ready**: Proper TLS, security, and observability
5. **Developer Friendly**: Works identically in all environments

## Error Page Implementation

```typescript
// app/auth/error/page.tsx
'use client';

import { useSearchParams } from 'next/navigation';
import Link from 'next/link';

export default function AuthError() {
  const searchParams = useSearchParams();
  const error = searchParams.get('error');
  
  const errorMessages: Record<string, string> = {
    Configuration: 'There is a problem with the server configuration.',
    AccessDenied: 'You do not have permission to sign in.',
    Verification: 'The verification token has expired or has already been used.',
    OAuthSignin: 'Error occurred trying to sign in with OAuth provider.',
    OAuthCallback: 'Error occurred during OAuth callback.',
    Default: 'An unexpected error occurred during authentication.',
  };
  
  return (
    <div className="flex min-h-screen items-center justify-center">
      <div className="mx-auto max-w-md space-y-6 p-6">
        <h1 className="text-2xl font-bold text-red-600">Authentication Error</h1>
        <p className="text-gray-600">
          {errorMessages[error || 'Default']}
        </p>
        <div className="space-x-4">
          <Link href="/auth/signin" className="text-blue-600 hover:underline">
            Try Again
          </Link>
          <Link href="/" className="text-gray-600 hover:underline">
            Go Home
          </Link>
        </div>
      </div>
    </div>
  );
}
```

### Handling Refresh Errors in UI Components

```typescript
// components/auth-error-handler.tsx
'use client';

import { useSession, signIn } from 'next-auth/react';
import { useEffect } from 'react';

export function AuthErrorHandler({ children }: { children: React.ReactNode }) {
  const { data: session } = useSession();
  
  useEffect(() => {
    // Check for refresh token errors
    if ((session as any)?.error === 'RefreshAccessTokenError') {
      // Token refresh failed - force re-authentication
      signIn('keycloak');
    }
  }, [session]);
  
  return <>{children}</>;
}

// Wrap your app or sensitive pages with this component:
// <AuthErrorHandler>
//   <YourApp />
// </AuthErrorHandler>
```

## Troubleshooting

### Common Issues and Solutions

1. **"Invalid redirect_uri" error**
   - Ensure Keycloak client has correct redirect URIs configured
   - Check that AUTH_URL matches your actual URL
   - For dev: `http://app.ameide.io:3004/*`
   - For production: `https://app.ameide.io/*`

2. **"OIDC discovery failed" error**
   - Verify issuer URL is accessible from your server
   - Check network connectivity: `curl ${KEYCLOAK_ISSUER_URL}/.well-known/openid-configuration`
   - For containers, ensure proper host resolution

3. **Cookie size exceeds 4KB**
   - Switch to database sessions
   - Reduce data stored in JWT callbacks
   - Remove unnecessary claims from tokens

4. **Token refresh fails**
   - Ensure `offline_access` scope is requested
   - Verify client has refresh token enabled in Keycloak
   - Check token expiration settings

5. **Federated logout not working**
   - Verify post_logout_redirect_uri is configured in Keycloak
   - Ensure id_token is being stored and passed
   - Check Keycloak logout endpoint is accessible

### Debug Mode

```typescript
// Enable debug logging in development
export const { auth, signIn, signOut, handlers } = NextAuth({
  debug: process.env.NODE_ENV === 'development',
  // ... rest of config
});
```


## Identity Federation via Keycloak

Keep Keycloak as broker to Azure AD/Auth0. **Zero app refactoring** - issuer stays `auth.ameide.io`.

```
App → Keycloak → [Azure AD | Auth0 | SAML | Social]
```

**Benefits:**
- App config unchanged (same issuer/discovery)
- Add/swap IdPs in Keycloak UI, not code
- Unified token schema regardless of upstream

**Quick setup:**
1. Add IdP: Azure `login.microsoftonline.com/<tenant>/v2.0`, Auth0 `<custom-domain>/`
2. Map claims: Upstream roles → your `isAdmin` 
3. Configure logout: Backchannel URLs for each IdP
4. Enable `kc_idp_hint` for auto-routing by email domain

**Gotchas:** Azure group overage, logout quirks, test cookie size.


## Security Checklist

- ✅ PKCE + state + nonce enabled
- ✅ TLS verification in production
- ✅ Secure cookie configuration  
- ✅ Manual token refresh in jwt callback
- ✅ Refresh timeout with AbortController
- ✅ Federated logout support
- ✅ Redis pointer sessions < 1KB cookies
- ✅ Structured logging
- ✅ Prometheus metrics
- ✅ Full issuer URLs with realm

## Keycloak Configuration Requirements

For offline refresh to work properly:

**Client Settings:**
- Standard Flow: **ON**
- Client Authentication: **ON** 
- Valid Redirect URIs: 
  - `http://app.ameide.io:*/*` (dev)
  - `https://app.ameide.io/*` (prod)
  - `https://auth-proxy.ameide.io/*` (if using AUTH_REDIRECT_PROXY_URL for preview envs)
- Web Origins: `+` (allows same as redirect URIs)
- Allowed scopes: Must include `offline_access`

**Realm Token Settings:**
- SSO Session Idle: 30 minutes (or your preference)
- SSO Session Max: 10 hours (or your preference)
- Offline Session Idle: 30 days (align with Redis TTL computation)
- Offline Session Max: 365 days (maximum lifetime)

The Redis TTL in the code already aligns with these by computing based on refresh token expiry.

## Final Checklist

### Discovery-First Implementation
- ✅ Single issuer URL configuration
- ✅ No manual endpoint overrides
- ✅ Full OIDC discovery compliance
- ✅ Consistent token auth method (client_secret_post)

### Security
- ✅ PKCE + state + nonce checks
- ✅ Proper TLS verification
- ✅ Token refresh with error handling
- ✅ Federated logout with id_token_hint
- ✅ No access tokens in client session

### Performance
- ✅ Minimal JWT payload
- ✅ 10-second clock skew (not 60s)
- ✅ Timeout cleanup in finally block
- ✅ Metrics without duplicate registration

### Gateway API Compatibility
- ✅ Works with both Ingress and Gateway API
- ✅ TLS termination at edge (KC_PROXY=edge)
- ✅ Consistent hostname (auth.ameide.io) across all environments
- ✅ /etc/hosts or extra_hosts for local domain resolution

## Operational Checklist

### Pre-Deployment Verification
```bash
# 1. Verify Keycloak is accessible
curl -sS "${AUTH_KEYCLOAK_ISSUER}/.well-known/openid-configuration" | jq '.issuer'

# 2. Check Redis connectivity
redis-cli -u "${REDIS_URL}" ping

# 3. Verify client credentials
curl -X POST "${AUTH_KEYCLOAK_ISSUER}/protocol/openid-connect/token" \
  -d "grant_type=client_credentials" \
  -d "client_id=${AUTH_KEYCLOAK_ID}" \
  -d "client_secret=${AUTH_KEYCLOAK_SECRET}"

# 4. Test Gateway routing (if using Gateway API)
curl -H "Host: auth.ameide.io" http://gateway-ip/realms/ameide/.well-known/openid-configuration
```

### Post-Deployment Health Checks
```bash
# 1. Auth endpoints responding
curl http://app.ameide.io/api/auth/providers
curl http://app.ameide.io/api/auth/session

# 2. Metrics available
curl http://app.ameide.io/api/metrics | grep auth_

# 3. Redis sessions being created
redis-cli -u "${REDIS_URL}" keys "session:*" | wc -l

# 4. No errors in logs
kubectl logs -n ameide deployment/www-ameide-canvas | grep ERROR
```

### Monitoring Alerts

```yaml
# Prometheus alert rules
groups:
  - name: auth
    rules:
      - alert: HighTokenRefreshFailureRate
        expr: rate(auth_token_refreshes_total{status="error"}[5m]) > 0.1
        for: 5m
        annotations:
          summary: "High token refresh failure rate"
          
      - alert: AuthSessionsDropped
        expr: delta(auth_sessions_active[1h]) < -10
        annotations:
          summary: "Significant drop in active sessions"
          
      - alert: KeycloakUnreachable
        expr: up{job="keycloak"} == 0
        for: 2m
        annotations:
          summary: "Keycloak is not responding"
```

### Troubleshooting Guide

| Issue | Check | Fix |
|-------|-------|-----|
| "Invalid redirect_uri" | Keycloak client config | Add callback URL to Valid Redirect URIs |
| Cookie > 4KB | DevTools → Application | Implement Redis sessions (done) |
| Token refresh fails | Check `offline_access` scope | Ensure scope requested in auth config |
| CORS errors | Browser console | Add origin to Web Origins in Keycloak |
| "READONLY" Redis error | `redis-cli INFO replication` | Connect to master node, not replica |
| Gateway routing fails | `kubectl get httproutes` | Check ReferenceGrants for cross-namespace |


Awesome—let’s turn your design into a crisp, incremental plan. Each step is small, deployable, and has a concrete “How to test” so you know when it’s done. I’ve also baked in the fixes from the review (shared refresh helper, lock TTL, dynamic client roles, JWKS robustness, TTL alignment, no double `dec()`).

# Step‑by‑step, each step testable

## 1) Prove OIDC discovery works (issuer is the single source of truth)

**Do**

* Stand up Keycloak (your devcontainer or k3d).
* Create realm `ameide`, client `web-app` with:

  * Standard Flow **ON**
  * Client Authentication **ON**
  * Redirect URIs covering your env (dev + prod + proxy if used)
  * Web Origins: `+`
  * Scope includes `offline_access`
* Set `AUTH_KEYCLOAK_ISSUER` to the **full** issuer URL including `/realms/ameide`.

**How to test**

* `curl -sS "$AUTH_KEYCLOAK_ISSUER/.well-known/openid-configuration" | jq '.issuer,.authorization_endpoint,.token_endpoint,.jwks_uri'`
* Verify `issuer` exactly matches `AUTH_KEYCLOAK_ISSUER` and endpoints resolve.

**Done when**: All endpoints respond and `issuer` is identical.

### ✅ Step 1 Complete

**Actions taken:**
1. Added `/etc/hosts`: `127.0.0.1 auth.ameide.io app.ameide.io`
2. Found Keycloak service: `keycloak:4000` (internal), realm `ameide` exists
3. Updated `/workspace/infra/kubernetes/environments/local/platform/www-ameide-canvas.yaml`:
   - Added `auth:` section with `url`, `trustHost`, `session` config
   - Changed to `keycloak.issuer: "http://keycloak:4000/realms/ameide"`
4. Fixed `/workspace/infra/kubernetes/charts/platform/www-ameide-canvas/templates/configmap.yaml`:
   - Replaced `KEYCLOAK_CLIENT_ID` → `AUTH_KEYCLOAK_ID`
   - Replaced `KEYCLOAK_ISSUER` → `AUTH_KEYCLOAK_ISSUER`
   - Added `AUTH_SESSION_MAX_AGE`, `AUTH_SESSION_UPDATE_AGE`
   - Fixed YAML syntax (quote escaping)
5. Fixed `/workspace/infra/kubernetes/charts/platform/www-ameide-canvas/templates/secret.yaml`:
   - `KEYCLOAK_CLIENT_SECRET` → `AUTH_KEYCLOAK_SECRET`
6. Verified from pod: `wget http://keycloak:4000/realms/ameide/.well-known/openid-configuration` works

**Next:** Update auth.ts to use new env var names

---

## 2) Minimal NextAuth sign‑in/out (happy path, discovery‑only)

**Do**

* Keep `runtime = 'nodejs'`.
* Configure Keycloak provider with checks `['state','pkce','nonce']`.
* Use your custom `profile()` mapper (minimal fields).
* Add custom sign‑in and error pages (you have them).

**How to test**

* Visit `/auth/signin` → choose Keycloak → sign in.
* Hit `/api/auth/session` to confirm a session exists.
* Sign out via `/api/auth/signout`.

**Done when**: You can sign in and out successfully with zero manual endpoints in code.

### ✅ Step 2 Complete

**Actions taken:**
1. Updated `/workspace/services/www_ameide_canvas/app/(auth)/auth.ts`:
   - Added `export const runtime = 'nodejs'`
   - Changed to Auth.js v5 env vars: `AUTH_KEYCLOAK_ID`, `AUTH_KEYCLOAK_SECRET`, `AUTH_KEYCLOAK_ISSUER`
   - Removed manual endpoint overrides, using pure discovery
   - Added security checks: `['state', 'pkce', 'nonce']`
   - Added `offline_access` scope for refresh tokens
   - Fixed profile mapper to include `type: 'keycloak'`
2. Updated `/workspace/services/www_ameide_canvas/app/(auth)/api/auth/[...nextauth]/route.ts`:
   - Added `export const runtime = 'nodejs'`
3. Verified endpoints:
   - `/api/auth/providers` returns Keycloak provider
   - `/api/auth/session` works (returns null when not logged in)
   - Callback URL: `http://localhost:3001/api/auth/callback/keycloak`

**Current state:** NextAuth configured with discovery-only, ready for sign-in testing

---

## 3) Pointer sessions in Redis (<4KB cookies) (COMPLETED)

**Do**

* Implement `lib/session-store.ts` (your version).
* In the `jwt` callback on first sign‑in:

  * Generate `sessionId`, store tokens in Redis.
  * Put only `{ sessionId, expiresAt, isAdmin }` in JWT.
  * Extract `sid` from `id_token` and map `sid → sessionId`.
  * **Capture `account.refresh_expires_in`** and compute `refreshExpiresAt`; store it.
* Switch `session.strategy = 'jwt'` with small payload.

**How to test**

* Sign in, open DevTools → Application → Cookies.
* Confirm cookie name, flags (httpOnly, secure, sameSite).
* Ensure each `Set-Cookie` header line length is small (cookies total well < 4KB).
* `redis-cli -u "$REDIS_URL" keys 'session:*'` → you see your `session:{id}`.
* `redis-cli get "sid:{sid}"` returns your `sessionId`.

**Done when**: Cookies are small; Redis has a session + `sid` mapping.

### ✅ Step 3 Complete

**Actions taken:**
1. Created TypeScript cache abstraction in `/workspace/services/www_ameide_canvas/lib/cache/types.ts`
   - Mirrors Python's CacheStorage protocol for consistency
   - Provides swappable cache implementations
2. Implemented Redis adapter in `/workspace/services/www_ameide_canvas/lib/cache/redis.ts`
   - Connection pooling with global singleton
   - Namespace support for key organization
   - Full feature set including SETNX for distributed locks
3. Added session store in `/workspace/services/www_ameide_canvas/lib/session-store.ts`
   - Token storage with dynamic TTL based on refresh token expiry
   - SID to sessionId mapping for backchannel logout
   - Distributed lock helpers for refresh coordination
4. Updated auth.ts JWT callback:
   - Stores full tokens in Redis on initial sign-in
   - Returns only sessionId pointer in JWT cookie
   - Extracts and stores Keycloak SID for logout handling
   - Computes isAdmin from realm and client roles
5. Configured infrastructure:
   - Added `redis.url` to Helm values
   - Added `REDIS_URL` to ConfigMap template
   - Redis available at `redis://redis:6379/0`
   - Added AUTH_SECRET to kubernetes secret
6. Added `jose` package for JWT decoding (required for extracting SID and roles)
7. Created comprehensive E2E tests in `tests/e2e/redis-session.test.ts`
8. Results:
   - Cookie size reduced from ~4KB to <1KB
   - Full tokens safely stored server-side
   - Auth.js v5 endpoints verified and working
   - Ready for token refresh implementation

**Important Auth.js v5 Note:**
- Direct GET to `/api/auth/signin/provider` returns "Configuration" error - this is expected
- OAuth flow must be initiated via client-side `signIn('keycloak')` or POST with CSRF token
- The login page SSO button correctly uses `signIn()` from next-auth/react

---

## 4) Shared token refresh helper + dynamic roles (COMPLETED)

**Do**

* Move `refreshAccessToken` into `lib/oidc.ts` and **import** it from both `auth.ts` and `withAccessToken`.
* In `jwt` callback, keep your "near‑expiry" refresh with `SKEW_MS = 10s`.
* **Dynamic client roles**: read `clientId = process.env.AUTH_KEYCLOAK_ID` when checking `resource_access[clientId].roles`.

**How to test**

* Temporarily set a very short token lifetime in Keycloak (e.g., 1–2 minutes).
* Sign in, wait until \~5–10s before expiry, load any server route.
* Expect a single refresh (see logs).
* Confirm your `tokenRefreshes_total{status="success"}` increases.

**Done when**: Access token refreshes automatically and `isAdmin` works regardless of client ID.

### ✅ Step 4 Complete

**Actions taken:**
1. Created shared Keycloak utilities in `/workspace/services/www_ameide_canvas/lib/keycloak.ts`:
   - `refreshAccessToken()` - Shared between auth.ts and withAccessToken
   - Handles both client_secret_post and client_secret_basic methods
   - Timeout protection with AbortController
   - Returns structured error responses
   - `extractRoles()` - Dynamically reads client ID from environment
   - `revokeToken()` - Best-effort token revocation for logout

2. Implemented access token helper in `/workspace/services/www_ameide_canvas/lib/access-token.ts`:
   - `withAccessToken()` - Ensures fresh tokens for API calls
   - Automatic refresh with 10-second buffer
   - Distributed lock to prevent concurrent refreshes
   - Waits for in-progress refresh if lock unavailable
   - Updates Redis and extends SID mapping TTL

3. Added comprehensive TypeScript types in `/workspace/services/www_ameide_canvas/types/auth.d.ts`:
   - Keycloak token interfaces (AccessToken, IdToken, TokenResponse)
   - Type guards for runtime validation
   - NextAuth module augmentation

4. Updated auth.ts with complete token refresh logic:
   - Proactive refresh with SKEW_MS = 10 seconds
   - Distributed lock acquisition
   - Proper refresh_expires_in tracking
   - Re-extracts roles after refresh
   - Cleans up session on permanent errors (invalid_grant, invalid_client)

5. Created test API route `/workspace/services/www_ameide_canvas/app/api/test/userinfo/route.ts`:
   - Demonstrates withAccessToken usage
   - Calls Keycloak userinfo endpoint
   - Handles various error scenarios

6. Comprehensive E2E tests in `/workspace/services/www_ameide_canvas/tests/e2e/token-refresh.test.ts`:
   - All 7 tests passing
   - Verifies configuration, lock TTL, role extraction
   - Tests error handling scenarios

**Results:**
- Shared refresh helper eliminates code duplication
- Dynamic role extraction works with any client ID
- Lock prevents thundering herd on refresh
- Tests confirm correct behavior
- Build successful with all new code

---

## 5) Refresh lock TTL safe against timeouts

**Status: COMPLETED ✅**

**Do**

* In `acquireRefreshLock`, set TTL >= HTTP timeout + 2s buffer (derive from `AUTH_KEYCLOAK_HTTP_TIMEOUT`).
* Keep `releaseRefreshLock` as is.

**How to test**

* Set `AUTH_KEYCLOAK_HTTP_TIMEOUT=10000`.
* Put the token near expiry; hit a protected route with concurrency (e.g., 20 concurrent requests).
* Inspect logs: exactly **one** refresh request should hit Keycloak.
* `tokenRefreshes_total` should increment by 1, not 20.

**Done when**: No refresh stampede under load.

### Implementation Complete

**Lock TTL Calculation** (`lib/session-store.ts` lines 67-69):
```typescript
const httpTimeoutMs = Number(process.env.AUTH_KEYCLOAK_HTTP_TIMEOUT ?? 10000);
const ttlSec = Math.ceil(httpTimeoutMs / 1000) + 2; // buffer
```

**Key Safety Features**:
- Lock TTL = HTTP timeout + 2 second buffer
- Prevents deadlocks if refresh request times out
- Lock auto-expires even if process crashes
- AbortController ensures clean timeout handling

### Test Results

Created comprehensive test suite in `tests/e2e/lock-ttl-safety.test.ts`:
- ✅ Lock TTL exceeds HTTP timeout by at least 2 seconds
- ✅ Timeout scenarios handled correctly
- ✅ Error handling preserves lock safety
- ✅ Optimal TTL calculations for different configurations
- ✅ Concurrent refresh prevention verified
- ✅ Lock cleanup on session deletion confirmed

**Production Recommendations**:
- Low-latency network: 5s timeout, 7s lock TTL
- Standard cloud: 10s timeout, 12s lock TTL (current)
- High-latency: 30s timeout, 35s lock TTL
- Mobile/edge: 60s timeout, 70s lock TTL

---

## 6) `withAccessToken` + a test route (proves server can call OIDC‑protected APIs)

**Status: COMPLETED ✅**

**Do**

* Implement `withAccessToken(sessionId, fn)` (you have it).
* Add an example route `/api/oidc/userinfo` that:

  * Retrieves `sessionId` from JWT.
  * Calls `${issuer}/protocol/openid-connect/userinfo` with the fresh access token.

**How to test**

* Hit `/api/oidc/userinfo` after sign‑in; you should get user claims.
* Wait for token near expiry and call again; still works (refresh happens).

**Done when**: The route always returns userinfo without 401s during normal operation.

### Implementation Complete

**Access Token Helper** (`lib/access-token.ts`):
```typescript
export async function withAccessToken<T>(
  sessionId: string,
  fn: (accessToken: string) => Promise<T>
): Promise<T>
```

**Features**:
- Auto-refreshes tokens near expiry (10s buffer)
- Prevents concurrent refreshes with distributed lock
- Waits for in-progress refresh if locked
- Extends SID mapping TTL on refresh
- Proper error handling and cleanup

**Test Route** (`app/api/test/userinfo/route.ts`):
- Demonstrates calling Keycloak userinfo endpoint
- Uses withAccessToken for automatic token management
- Returns userinfo + session data + metadata
- Handles all error cases appropriately

### Test Results

Created comprehensive test suite in `tests/e2e/access-token-helper.test.ts`:
- ✅ Test route protected and working
- ✅ Behavior matrix for all token states verified
- ✅ Concurrent refresh prevention confirmed
- ✅ Error handling validated
- ✅ Token freshness guarantees verified
- ✅ SID mapping extension on refresh confirmed
- ✅ Production patterns documented
- ✅ Anti-patterns identified

**Production Usage Patterns**:
1. Simple API calls with automatic refresh
2. Proper error handling with user-friendly messages
3. Batch multiple API calls with single refresh

**Anti-Patterns to Avoid**:
- ❌ Storing tokens in component state
- ❌ Manually refreshing tokens
- ❌ Passing tokens to client components
- ❌ Not handling refresh failures
- ❌ Refreshing on every request

---

## 7) Federated logout (no double gauge decrement)

**Do**

* Keep your `federated-logout` route: revoke refresh token (best‑effort), build `end_session` URL, call `signOut({redirect:false})`, delete Redis tokens.
* **Remove** any extra `sessionActive.dec()` outside the NextAuth `events.signOut`.

**How to test**

* While signed in, open `/api/auth/federated-logout`.
* You are redirected to Keycloak logout then back to `AUTH_URL`.
* `redis-cli` shows your `session:{id}` and `sid:{sid}` are gone.
* `auth_sessions_active` decreased by exactly one.

**Done when**: Local and upstream sessions both clear, and metrics gauge stays correct.

---

## 8) Back‑channel logout (robust, idempotent)

**Do**

* Hardening:

  * Wrap verification in `try/catch` and return `204` on errors/ignored events.
  * Use JWKS via `createRemoteJWKSet`.
  * Use `sid → sessionId` mapping; delete tokens on match.
  * Align `sid` TTL with `refresh_expires_in` captured at sign‑in (not a fixed 30d cap).

**How to test**

* Configure Keycloak client: Backchannel Logout URL → `/api/auth/backchannel-logout`.
* Sign in, then in Keycloak Admin, force logout that user/session.
* Your app should clear Redis entries without any browser action.
* Calling the endpoint twice should be harmless (idempotent).

**Done when**: Admin‑triggered logouts nuke the app session immediately; no 500s in logs.

---

## 9) Observability & metrics

**Do**

* Keep `authAttempts`, `tokenRefreshes`, `auth_sessions_active`.
* Add (optional) `Histogram` for refresh latency.
* Protect `/api/metrics` (basic auth or network policy).

**How to test**

* Call `/api/metrics` without auth in prod → `401`.
* With auth → metrics text shows:

  * `auth_attempts_total{status="success"}` increments on sign‑in.
  * `auth_token_refreshes_total` reflects refresh attempts.
  * `auth_sessions_active` never goes negative.

**Done when**: You can graph a successful login and a refresh in Prometheus.

---

## 10) Cross‑site cookie mode (Codespaces/Gitpod)

**Do**

* Implement `isCrossSite()` helper and use it consistently in cookie options.
* Set `AUTH_CROSS_SITE="true"` and `AUTH_REDIRECT_PROXY_URL` for preview envs.

**How to test**

* In Codespaces/Gitpod, sign in via provider.
* Inspect cookies: `SameSite=None; Secure` present.
* Complete the auth flow (no “blocked third‑party cookie” errors).

**Done when**: Login works end‑to‑end in cross‑site environments.

---

## 11) Helm & Ingress rollout (dev → prod)

**Do**

* Ship `ConfigMap` and `ExternalSecret` templates.
* Ingress uses TLS and `KC_PROXY=edge` on Keycloak.
* Set `AUTH_TRUST_HOST=false` in prod unless you need proxy trust.

**How to test**

* `helm upgrade --install ...` into a dev namespace.
* `curl https://app.ameide.io/api/auth/signin` returns HTML (200).
* `curl https://auth.ameide.io/realms/ameide/.well-known/openid-configuration` works over HTTPS.
* Perform the full sign‑in/out and userinfo tests from earlier.

**Done when**: The same flow works outside local, behind Ingress + TLS.

---

## 12) RBAC: verify `isAdmin` via realm or client roles

**Do**

* Use the dynamic client ID when reading `resource_access`.
* Surfacing `session.user.isAdmin` only.

**How to test**

* Assign your user the `admin` realm role **or** client role.
* After re‑login, check `/api/auth/session` → `user.isAdmin: true`.
* Remove the role; re‑login → `false`.

**Done when**: Role toggles reflect in session on next login.

---

## 13) Failure drills (resilience)

**Do**

* **invalid\_grant**: revoke the refresh token in Keycloak; next request should log refresh failure, delete Redis tokens, and force re‑auth.
* **Network slowness**: raise `AUTH_KEYCLOAK_HTTP_TIMEOUT` and lower it to trigger aborts; confirm timeouts don’t wedge the app.
* **JWKS outage**: temporarily block JWKS URL; back‑channel route returns 204 (ignored), no 500s.

**How to test**

* Watch logs + metrics:

  * `tokenRefreshes_total{status="error"}` increments appropriately.
  * Redis keys are cleaned on permanent failures.

**Done when**: The app fails safe and recovers cleanly in each scenario.

---

# Lightweight automation you can add quickly

* **Playwright e2e**

  * `login.spec.ts`: hits `/auth/signin`, completes Keycloak form, verifies session cookie and `/api/oidc/userinfo`.
  * `refresh.spec.ts`: short token lifetime; waits until T‑10s; hits `/api/oidc/userinfo`; asserts no 401 and that `auth_token_refreshes_total` increased.
  * `logout.spec.ts`: calls `/api/auth/federated-logout`; asserts redirect and cleared cookies.
* **Load test for refresh lock** (optional): use `hey`/`autocannon` to hammer a protected route around expiry; assert a single refresh in logs.
* **CI smoke**: `curl` checks for discovery, `/api/auth/signin` (200), `/api/metrics` (401 in prod), and a basic Redis key existence after a headless login (if you wire a service token).

# Quick reference: patch set to include during implementation

* **Shared refresh helper** in `lib/oidc.ts`; import in `auth.ts` and `withAccessToken`.
* **Lock TTL >= HTTP timeout + 2s**.
* **Dynamic client roles** for `isAdmin`.
* **Back‑channel logout** wrapped in `try/catch`, return 204 on ignore/error.
* **TTL alignment** for `sid → sessionId` using `refresh_expires_in`.
* **Remove duplicate `sessionActive.dec()`** from federated logout.

---

If you want, I can turn this into a checklist PR template (GitHub issue with boxes) or a tiny Playwright starter that covers "login → refresh → logout".

---

## Critical Fixes Applied (Post-Implementation)

### Redis Master Connection Issue
**Status: FIXED ✅**

**Problem**: 
- Redis deployed with master-slave replication using Sentinel
- The `redis` service was routing to read-only replica
- Error: `READONLY You can't write against a read only replica`

**Solution**:
- Updated Redis URL to connect directly to master node
- Changed from: `redis://redis:6379/0`
- To: `redis://redis-node-1.redis-headless.ameide.svc.cluster.local:6379/0`
- Updated in:
  - `/workspace/infra/kubernetes/values/platform/www-ameide-canvas.yaml`
  - `/workspace/infra/kubernetes/environments/local/platform/www-ameide-canvas.yaml`
  - ConfigMap template to use master URL

**Verification**:
```bash
# Check which node is master
kubectl exec -n ameide redis-node-1 -- redis-cli INFO replication | head -5
# Shows: role:master
```

### Missing Dependencies
**Status: FIXED ✅**

**Problem**: 
- `ioredis` package was not installed
- Caused Redis connection failures during JWT callback

**Solution**:
```bash
pnpm add ioredis
pnpm add -D @types/ioredis
```

### Missing Home Page Component
**Status: FIXED ✅**

**Problem**:
- Empty `app/page.tsx` file
- Error: "The default export is not a React Component in "/page""

**Solution**:
- Created complete home page component with:
  - Session information display
  - User details (ID, email, name, admin status)
  - OAuth implementation status tracker
  - Sign out button
  - Test API link
  - Auto-redirect to login if unauthenticated

---

## Authentication Flow Summary

### Working Features ✅
1. **SSO Login via Keycloak** - Full OAuth/OIDC flow
2. **Redis Session Storage** - Tokens stored server-side, <1KB cookies
3. **Automatic Token Refresh** - With distributed locks
4. **Role Extraction** - Dynamic client role detection
5. **Protected API Routes** - Using `withAccessToken` helper
6. **Session Management** - Proper cleanup and TTL management

### Login Methods
- **SSO Button** ✅ - Uses Keycloak OAuth (WORKING)
- **Username/Password Form** ❌ - Needs credentials provider (NOT CONFIGURED)

---

## Testing Instructions

### Manual Login Test (From Host Machine)

1. **Prerequisites**:
   - Add to `/etc/hosts`: `127.0.0.1 keycloak auth.ameide.io app.ameide.io`
   - Ensure services are running: `kubectl get pods -n ameide`

2. **Access Points**:
   - Login Page: http://localhost:3001/login
   - Keycloak Admin: http://localhost:4000/admin (admin/C1tr0n3lla!)
   - Session Check: http://localhost:3001/api/auth/session
   - Protected API: http://localhost:3001/api/test/userinfo
   - Home Page: http://localhost:3001/ (after login)

3. **Login Flow**:
   - Navigate to http://localhost:3001/login
   - Click "Sign in with SSO" (NOT the username/password form)
   - Redirected to Keycloak login page
   - Enter credentials (create user in Keycloak admin if needed)
   - Redirected back to app with session established
   - Home page shows user information

4. **Create Test User in Keycloak**:
   - Access Keycloak Admin: http://localhost:4000/admin
   - Login: admin/C1tr0n3lla!
   - Navigate to realm "ameide" → Users → Add User
   - Set username and email
   - Go to Credentials tab → Set Password
   - Turn OFF "Temporary" toggle

5. **Verify Success**:
   - Home page displays user information
   - Session cookie is set (check DevTools)
   - API endpoints return user data
   - Sign out button works

### Known Issues
- Username/password form doesn't work (no credentials provider configured)
- Only SSO/OAuth flow is functional
- Must use master Redis node for writes (already fixed in Helm values)

---

## 105-domain-strategy-ameide-io.md (old)

# Domain Strategy: Unified ameide.io Architecture

## Current Implementation Status

**Local Development**: ✅ Using `.test` domains (platform.dev.ameide.io)
**TLS Management**: ✅ cert-manager with self-signed certificates
**Gateway**: ✅ Envoy Gateway v1.5.0 supporting both .test and .io domains
**DevContainer**: ✅ Fully configured with .test domains and port mappings

## Executive Summary

Implement consistent domain naming across all environments. Local development uses `.test` domains for safety, while staging/production use `.io` domains. This ensures OAuth flows work correctly in each environment.

## Domain Mapping

### Service to Domain Assignment

| Service | Domain | Purpose |
|---------|--------|---------|
| `www-ameide` | `ameide.io` | Main application |
| `www-ameide` | `www.ameide.io` | Main application (redirect to apex) |
| `www-ameide_canvas` | `platform.ameide.io` | Platform/Canvas application |
| `keycloak` | `auth.ameide.io` | Authentication/SSO |
| `envoy` | `api.ameide.io` | API Gateway (gRPC-Web) |
| `grafana` | `metrics.ameide.io` | Observability dashboard |

## Environment Strategy

### 1. Local Development (k3d)

**DNS Resolution:**
- Uses `.test` domains for local development (not production `.io` domains)
- Developers add entries to `/etc/hosts` or use dnsmasq for `*.dev.ameide.io`
- DevContainer automatically configures `/etc/hosts` entries
- No real DNS required

**Access:**
- `http://platform.dev.ameide.io:8080` (via Gateway API, k3d maps 8080→80)
- `https://platform.dev.ameide.io` (with self-signed cert, k3d maps 443→443)
- `http://auth.dev.ameide.io:8080` (Keycloak via Gateway API)
- `http://api.dev.ameide.io:8080` (gRPC-Web via Gateway API)

**SSL/TLS:**
- Self-signed certificates for local HTTPS testing
- Generated by cert-manager with self-signed issuer
- Certificate secret: `ameide-wildcard-tls`
- Browser warnings expected (add exception for local dev)

**Configuration:**
```yaml
domains:
  root: "dev.ameide.io"  # Using .test for local development
  platform: "platform.dev.ameide.io"
  auth: "auth.dev.ameide.io"
protocol: "https"  # HTTPS with self-signed cert
tls:
  enabled: true
  issuer: "self-signed"  # cert-manager ClusterIssuer
  secretName: "ameide-wildcard-tls"  # Wildcard certificate for *.dev.ameide.io
```

### 2. DevContainer (Debian on macOS Host)

**DNS Resolution:**
- Container runs Debian, host machine is macOS
- DevContainer's `postCreateCommand.sh` automatically adds `.test` domains to `/etc/hosts`
- For macOS host: run `manage-host-domains.sh setup` to configure dnsmasq for `*.dev.ameide.io`
- k3d cluster runs inside DevContainer, not on host

**Special Considerations:**
- DevContainer includes full k3d cluster (Docker-in-Docker)
- Ports 8080/8443 are mapped for Gateway access
- TLS certificates managed by cert-manager (not mkcert)

**Configuration:**
```bash
# /etc/hosts entries added by postCreateCommand.sh
127.0.0.1 dev.ameide.io www.dev.ameide.io
127.0.0.1 auth.dev.ameide.io
127.0.0.1 platform.dev.ameide.io
127.0.0.1 portal.dev.ameide.io
127.0.0.1 api.dev.ameide.io
127.0.0.1 metrics.dev.ameide.io
```

**Environment Variables:**
```yaml
# Services use .test domains for local development
- AUTH_URL=https://platform.dev.ameide.io
- AUTH_KEYCLOAK_ISSUER=http://auth.dev.ameide.io:4000/realms/ameide
```

**macOS Host Setup (Optional - for accessing from host browser):**
```bash
# Run manage-host-domains.sh to set up dnsmasq for *.dev.ameide.io
./.devcontainer/manage-host-domains.sh setup

# This configures:
# - dnsmasq to resolve *.dev.ameide.io to 127.0.0.1
# - macOS resolver to use dnsmasq for .test domains
# - No TLS certificates (handled by cert-manager in k8s)

# To check status:
./.devcontainer/manage-host-domains.sh status

# To remove:
./.devcontainer/manage-host-domains.sh cleanup
```

**Note:** The k3d cluster runs inside the DevContainer with port mappings:
- Port 8080 → 80 (HTTP)
- Port 443 → 443 (HTTPS)
These are automatically configured by `postCreateCommand.sh`

### 3. Azure Test Environment

**DNS Resolution:**
- Azure DNS Zone: `test.ameide.io`
- Real DNS with A/CNAME records
- Points to Gateway LoadBalancer IP

**Access:**
- `https://platform.test.ameide.io`
- `https://auth.test.ameide.io`
- `https://api.test.ameide.io`

**Configuration:**
```yaml
domains:
  root: "test.ameide.io"
  platform: "platform.test.ameide.io"
  auth: "auth.test.ameide.io"
protocol: "https"
tls:
  enabled: true
  issuer: "letsencrypt-staging"
```

### 4. Azure Production

**DNS Resolution:**
- Azure DNS Zone: `ameide.io`
- Production DNS records
- Points to Gateway LoadBalancer IP

**Access:**
- `https://platform.ameide.io`
- `https://auth.ameide.io`
- `https://api.ameide.io`

**Configuration:**
```yaml
domains:
  root: "ameide.io"
  platform: "platform.ameide.io"
  auth: "auth.ameide.io"
protocol: "https"
tls:
  enabled: true
  issuer: "letsencrypt-prod"
```

## Implementation Plan

### Phase 1: Local k3d Setup (✅ IMPLEMENTED)

#### 1.1 k3d cluster creation (✅ DONE)
```bash
# Actual implementation in postCreateCommand.sh
k3d cluster create ameide \
  --port "8080:80@loadbalancer" \    # Port 8080 maps to 80
  --port "8443:443@loadbalancer" \   # Port 443 maps to 443
  --k3s-arg "--disable=traefik@server:0" \
  --registry-create docker.io/ameide
```

#### 1.2 Envoy Gateway Installation (✅ DONE)

```bash
# Installed via Helmfile
helmfile -e local sync -l name=gateway

# Gateway supports both .test and .io domains
# Configuration in environments/local/platform/gateway.yaml
```

#### 1.3 Configure CoreDNS (Optional - Only for OIDC Discovery)

**When to use CoreDNS overrides:**
- ✅ OIDC discovery requires public URLs (e.g., `AUTH_KEYCLOAK_ISSUER=http://auth.ameide.io`)
- ✅ Services need to validate JWT tokens against public issuer URL
- ❌ Regular pod-to-pod communication (use `*.svc.cluster.local` instead)

**Preferred approach for pod-to-pod:**
```yaml
# Use cluster-internal DNS for pod-to-pod communication
env:
  # Public URLs for OIDC/OAuth (requires CoreDNS override)
  - name: AUTH_KEYCLOAK_ISSUER
    value: "http://auth.ameide.io:4000/realms/ameide"
  
  # Internal URLs for direct service calls (no CoreDNS needed)
  - name: KEYCLOAK_INTERNAL_URL
    value: "http://keycloak.ameide.svc.cluster.local:4000"
  - name: API_INTERNAL_URL
    value: "http://envoy.ameide.svc.cluster.local:8000"
```

**If CoreDNS override is required:**
```bash
# Get the Gateway's ClusterIP
GATEWAY_IP=$(kubectl get svc -n ameide ameide-gateway -o jsonpath='{.spec.clusterIP}')

# Apply CoreDNS custom config ONLY if needed for OIDC
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  ameide.override: |
    ameide.io:53 {
      hosts {
        $GATEWAY_IP ameide.io
        $GATEWAY_IP www.ameide.io
        $GATEWAY_IP platform.ameide.io
        $GATEWAY_IP auth.ameide.io
        $GATEWAY_IP api.ameide.io
      }
    }
EOF

# Restart CoreDNS
kubectl rollout restart -n kube-system deployment/coredns
```

#### 1.4 Create HTTPRoutes
```yaml
# platform-httproute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: platform
  namespace: ameide
spec:
  parentRefs:
    - name: ameide
  hostnames:
    - platform.ameide.io
  rules:
    - backendRefs:
        - name: www-ameide-canvas
          port: 3001
```

#### 1.5 Update Service Configurations
- Replace all `localhost:PORT` references with proper domains
- Update OAuth callback URLs
- Configure Keycloak with consistent issuer URL

### Phase 2: DevContainer Support (Week 2)

#### 2.1 Update .devcontainer/docker-compose.yml
```yaml
version: '3.8'
services:
  devcontainer:
    build: 
      context: .
      dockerfile: Dockerfile
    image: mcr.microsoft.com/devcontainers/base:debian
    extra_hosts:
      # On macOS, host-gateway maps to host.docker.internal
      # This allows Debian container to reach k3d on macOS host
      # Keep in sync with /etc/hosts (use setup-local-domains.sh)
      - "ameide.io:host-gateway"
      - "www.ameide.io:host-gateway"
      - "platform.ameide.io:host-gateway"
      - "auth.ameide.io:host-gateway"
      - "api.ameide.io:host-gateway"
    environment:
      - AUTH_URL=http://platform.ameide.io  # NextAuth v5 standard env var
      - AUTH_KEYCLOAK_ISSUER=http://auth.ameide.io:4000/realms/ameide  # Keycloak URL with port 4000
      # DevContainer-specific settings
      - DOCKER_HOST=unix:///var/run/docker-host.sock
```

#### 2.2 macOS Host Configuration
```bash
# On macOS host machine
# 1. Run the setup script (same as Phase 1)
./setup-local-domains.sh

# Or manually update /etc/hosts
sudo bash -c 'echo "127.0.0.1 ameide.io www.ameide.io platform.ameide.io auth.ameide.io api.ameide.io" >> /etc/hosts'

# 2. Ensure k3d cluster is running with exposed ports
k3d cluster create ameide \
  --port "80:80@loadbalancer" \
  --port "443:443@loadbalancer" \
  --k3s-arg "--disable=traefik@server:0" \
  --k3s-arg "--disable=servicelb@server:0"

# 3. Verify from macOS host
curl http://platform.ameide.io  # Should reach k3d ingress
```

#### 2.3 DevContainer Network Validation
```bash
# Inside DevContainer (Debian)
# Test connectivity to macOS host
ping host.docker.internal

# Test domain resolution (nslookup doesn't use /etc/hosts)
getent hosts platform.ameide.io
# Should resolve to host-gateway IP

# Alternative: use ping which respects /etc/hosts
ping -c 1 platform.ameide.io
# Should reach host.docker.internal

# Test k3d cluster access
curl http://platform.ameide.io
# Should reach the application through k3d on macOS
```

### Phase 3: Azure Test Deployment (Week 3)

#### 3.1 Create Azure DNS Zone
```bash
# Create the subzone
az network dns zone create \
  --resource-group ameide-rg \
  --name test.ameide.io

# Get the nameservers for the new zone
TEST_NS=$(az network dns zone show \
  --resource-group ameide-rg \
  --name test.ameide.io \
  --query nameServers -o tsv)

# Delegate from parent zone (ameide.io) to subzone (test.ameide.io)
for ns in $TEST_NS; do
  az network dns record-set ns add-record \
    --resource-group ameide-rg \
    --zone-name ameide.io \
    --record-set-name test \
    --nsdname $ns
done
```

#### 3.2 Configure DNS Records
```bash
# Get Gateway IP
GATEWAY_IP=$(kubectl get svc -n ameide ameide-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Create A record for apex domain with short TTL during rollout
az network dns record-set a add-record \
  --resource-group ameide-rg \
  --zone-name test.ameide.io \
  --record-set-name "@" \
  --ipv4-address $GATEWAY_IP \
  --ttl 300  # 5 minutes during rollout, increase to 3600 after stability

# Create CNAME records pointing to apex (recommended approach)
for subdomain in platform auth api www; do
  az network dns record-set cname create \
    --resource-group ameide-rg \
    --zone-name test.ameide.io \
    --name $subdomain \
    --ttl 300  # Short TTL during rollout
  
  az network dns record-set cname set-record \
    --resource-group ameide-rg \
    --zone-name test.ameide.io \
    --record-set-name $subdomain \
    --cname test.ameide.io  # All subdomains point to apex
done

# After successful rollout, increase TTLs for production
# az network dns record-set a update --ttl 3600 ...
# az network dns record-set cname update --ttl 3600 ...
```

#### 3.3 Install cert-manager (✅ DONE FOR LOCAL)
```bash
# cert-manager is installed via Helmfile
helmfile -e local sync -l name=cert-manager

# Self-signed issuer is automatically configured for local development
kubectl get clusterissuer self-signed -o yaml
```

#### 3.4 Configure Issuers (✅ SELF-SIGNED IMPLEMENTED)

##### Self-Signed Issuer (Local Development - ✅ ACTIVE)
```yaml
# Automatically created by cert-manager-config chart
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: self-signed
spec:
  selfSigned: {}

# Certificate created for *.dev.ameide.io
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ameide-wildcard
  namespace: ameide
spec:
  secretName: ameide-wildcard-tls
  dnsNames:
    - "*.dev.ameide.io"
    - "dev.ameide.io"
  issuerRef:
    name: self-signed
    kind: ClusterIssuer
```

##### Let's Encrypt Staging (Test Environments)
```yaml
# letsencrypt-staging.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # Staging server for testing (higher rate limits, untrusted certs)
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: admin@ameide.io
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
      # HTTP-01 challenge for domain validation
      - http01:
          ingress:
            gatewayHTTPRoute:
              parentRefs:
                - name: ameide
                  namespace: ameide
                  kind: Gateway
      # DNS-01 challenge for wildcard certificates (optional)
      - dns01:
          azureDNS:
            clientID: ${AZURE_CLIENT_ID}
            clientSecretSecretRef:
              name: azuredns-config
              key: client-secret
            subscriptionID: ${AZURE_SUBSCRIPTION_ID}
            tenantID: ${AZURE_TENANT_ID}
            resourceGroupName: ameide-rg
            hostedZoneName: test.ameide.io
```

##### Let's Encrypt Production
```yaml
# letsencrypt-prod.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # Production server (trusted certificates, strict rate limits)
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@ameide.io
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      # HTTP-01 for individual domains
      - http01:
          ingress:
            gatewayHTTPRoute:
              parentRefs:
                - name: ameide
                  namespace: ameide
                  kind: Gateway
      # DNS-01 for wildcard certificates (using Workload Identity)
      - dns01:
          azureDNS:
            subscriptionID: ${AZURE_SUBSCRIPTION_ID}
            resourceGroupName: ameide-rg
            hostedZoneName: ameide.io
            # Using AKS Workload Identity (no client secret)
            managedIdentity:
              clientID: ${AZURE_MANAGED_IDENTITY_CLIENT_ID}
```

##### AKS Workload Identity Setup
```bash
# 1. Enable OIDC issuer on AKS cluster
az aks update \
  --resource-group ameide-rg \
  --name ameide-cluster \
  --enable-oidc-issuer

# Get the OIDC issuer URL
AKS_OIDC_ISSUER=$(az aks show \
  --resource-group ameide-rg \
  --name ameide-cluster \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)

# 2. Create managed identity for cert-manager
az identity create \
  --name cert-manager-dns-identity \
  --resource-group ameide-rg

# Get identity details
IDENTITY_CLIENT_ID=$(az identity show \
  --name cert-manager-dns-identity \
  --resource-group ameide-rg \
  --query clientId -o tsv)

# 3. Grant DNS Zone Contributor role
IDENTITY_ID=$(az identity show --name cert-manager-dns-identity --resource-group ameide-rg --query principalId -o tsv)
DNS_ZONE_ID=$(az network dns zone show --name ameide.io --resource-group ameide-rg --query id -o tsv)
az role assignment create \
  --role "DNS Zone Contributor" \
  --assignee $IDENTITY_ID \
  --scope $DNS_ZONE_ID

# 4. Create federated credential for cert-manager service account
az identity federated-credential create \
  --name cert-manager-credential \
  --identity-name cert-manager-dns-identity \
  --resource-group ameide-rg \
  --issuer $AKS_OIDC_ISSUER \
  --subject system:serviceaccount:cert-manager:cert-manager

# 5. Annotate cert-manager service account with client ID
kubectl annotate serviceaccount cert-manager \
  -n cert-manager \
  azure.workload.identity/client-id=$IDENTITY_CLIENT_ID \
  --overwrite

# 6. Label the service account for workload identity
kubectl label serviceaccount cert-manager \
  -n cert-manager \
  azure.workload.identity/use=true \
  --overwrite
```

#### 3.5 HTTPRoute with Automatic TLS
```yaml
# platform-httproute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: platform
  namespace: ameide
spec:
  parentRefs:
    - name: ameide
  hostnames:
    - platform.test.ameide.io
  rules:
    - backendRefs:
        - name: www-ameide-canvas
          port: 3001
```

**Note**: TLS is handled by the Gateway with cert-manager annotations. HTTPS redirect is configured via a separate HTTPRoute.

### Phase 4: Production Deployment (Week 4)

#### 4.1 DNS Configuration Best Practices

**TTL Strategy:**
- **During Rollout**: Use 300 seconds (5 minutes) for quick changes
- **After Stability**: Increase to 3600 seconds (1 hour) for better caching
- **Emergency Changes**: Can temporarily reduce to 60 seconds if needed

**DNS Records Structure:**
```bash
# Production DNS setup
# A record for apex
ameide.io.           A      300  <GATEWAY_IP>

# CNAME records for all subdomains
www.ameide.io.       CNAME  300  ameide.io.
platform.ameide.io.  CNAME  300  ameide.io.
auth.ameide.io.      CNAME  300  ameide.io.
api.ameide.io.       CNAME  300  ameide.io.
metrics.ameide.io.   CNAME  300  ameide.io.
```

**Benefits of CNAME approach:**
- Single IP to update (apex A record only)
- Consistent routing for all subdomains
- Easier SSL certificate management
- Better for CDN/proxy integration

#### 4.2 WWW Subdomain Handling

```yaml
# HTTPRoute for apex domain
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: main-app
  namespace: ameide
spec:
  parentRefs:
    - name: ameide
  hostnames:
    - ameide.io
  rules:
    - backendRefs:
        - name: www-ameide
          port: 3000

---
# HTTPRoute for www redirect to apex
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: www-redirect
  namespace: ameide
spec:
  parentRefs:
    - name: ameide
  hostnames:
    - www.ameide.io
  rules:
    - filters:
        - type: RequestRedirect
          requestRedirect:
            hostname: ameide.io
            statusCode: 301
```

**SEO and User Experience:**
- Redirects preserve search rankings
- Users typing "www" reach the correct site
- Consistent URL structure for bookmarks
- Compatible with HSTS includeSubDomains

#### 4.3 Blue-Green Deployment
- Deploy to staging slot
- Test OAuth flows with short TTLs
- Switch DNS to new deployment (wait for TTL expiry)
- Monitor for issues
- Increase TTLs after 24 hours of stability

## Configuration Templates

### Helm Values Structure

**IMPORTANT**: Helm does NOT template values files. Use helpers in templates instead.

#### Values Files (Static Configuration Only)

```yaml
# values-base.yaml - NO TEMPLATING HERE
global:
  domain:
    root: ""  # Will be overridden by environment-specific values
    protocol: ""  # Will be overridden
    
  # Static internal URLs (these don't change)
  internal:
    auth: "http://keycloak.ameide.svc.cluster.local:4000"
    api: "http://envoy.ameide.svc.cluster.local:8000"
    cache: "redis://redis-node-1.redis-headless.ameide.svc.cluster.local:6379"

# values-local.yaml
environment:
  name: "local"
  domain: "ameide.io"
  protocol: "http"

# values-test.yaml
environment:
  name: "test"
  domain: "test.ameide.io"
  protocol: "https"

# values-prod.yaml
environment:
  name: "production"
  domain: "ameide.io"
  protocol: "https"
```

#### Helm Template Helpers

```yaml
# templates/_helpers.tpl
{{- define "ameide.domain" -}}
{{- .Values.environment.domain | default "ameide.io" -}}
{{- end -}}

{{- define "ameide.protocol" -}}
{{- .Values.environment.protocol | default "http" -}}
{{- end -}}

{{- define "ameide.url" -}}
{{- $proto := include "ameide.protocol" . -}}
{{- $domain := include "ameide.domain" . -}}
{{- printf "%s://%s" $proto $domain -}}
{{- end -}}

{{- define "ameide.subdomainUrl" -}}
{{- $proto := include "ameide.protocol" . -}}
{{- $domain := include "ameide.domain" . -}}
{{- $subdomain := .subdomain -}}
{{- printf "%s://%s.%s" $proto $subdomain $domain -}}
{{- end -}}

{{- define "ameide.platformUrl" -}}
{{- include "ameide.subdomainUrl" (dict "Values" .Values "subdomain" "platform") -}}
{{- end -}}

{{- define "ameide.authUrl" -}}
{{- include "ameide.subdomainUrl" (dict "Values" .Values "subdomain" "auth") -}}
{{- end -}}

{{- define "ameide.apiUrl" -}}
{{- include "ameide.subdomainUrl" (dict "Values" .Values "subdomain" "api") -}}
{{- end -}}
```

#### Using Helpers in Templates

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "ameide.fullname" . }}-config
data:
  # Use template helpers for computed values
  AUTH_URL: {{ include "ameide.platformUrl" . | quote }}
  AUTH_KEYCLOAK_ISSUER: {{ printf "%s/realms/ameide" (include "ameide.authUrl" .) | quote }}
  API_PUBLIC_URL: {{ include "ameide.apiUrl" . | quote }}
  
  # Static internal URLs from values
  API_INTERNAL_URL: {{ .Values.global.internal.api | quote }}
  REDIS_URL: {{ .Values.global.internal.cache | quote }}
```

### Service Configuration in Templates

```yaml
# templates/deployment.yaml (example for www-ameide_canvas)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "ameide.fullname" . }}-canvas
spec:
  template:
    spec:
      containers:
        - name: canvas
          env:
            # Computed URLs using helpers
            - name: AUTH_URL
              value: {{ include "ameide.platformUrl" . | quote }}
            - name: AUTH_KEYCLOAK_ISSUER
              value: {{ printf "%s/realms/ameide" (include "ameide.authUrl" .) | quote }}
            - name: NEXT_PUBLIC_API_URL
              value: {{ include "ameide.apiUrl" . | quote }}
            
            # Static internal URLs from values
            - name: API_INTERNAL_URL
              value: {{ .Values.global.internal.api | quote }}
            - name: REDIS_URL
              value: {{ printf "%s/0" .Values.global.internal.cache | quote }}
            
            # Static config from values
            - name: AUTH_KEYCLOAK_ID
              value: {{ .Values.keycloak.clientId | default "platform-app" | quote }}
            - name: AUTH_TRUST_HOST
              value: {{ .Values.auth.trustHost | default "true" | quote }}
```

### Alternative: Using ConfigMap Reference

```yaml
# templates/deployment.yaml (cleaner approach)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "ameide.fullname" . }}-canvas
spec:
  template:
    spec:
      containers:
        - name: canvas
          envFrom:
            - configMapRef:
                name: {{ include "ameide.fullname" . }}-config
          env:
            # Secrets from external secret
            - name: AUTH_KEYCLOAK_SECRET
              valueFrom:
                secretKeyRef:
                  name: {{ include "ameide.fullname" . }}-auth
                  key: client-secret
```

## Keycloak Configuration (Actual Platform Implementation)

### Overview

The platform uses Bitnami's Keycloak Helm chart with CloudNativePG for database backend.

### Base Configuration

```yaml
# charts/values/infrastructure/keycloak.yaml
auth:
  adminUser: admin
  adminPassword: changeme  # Override with secrets in production

# External PostgreSQL (CloudNativePG)
postgresql:
  enabled: false  # Use external database

externalDatabase:
  host: postgres-keycloak-rw.ameide.svc.cluster.local
  port: 5432
  user: keycloak
  password: keycloak  # Override with secrets
  database: keycloak

# Service configuration
service:
  type: ClusterIP  # Default type, overridden per environment
  ports:
    http: 4000    # Platform standard port
    https: 8443

# Environment settings
production: false  # Set to true in production
proxy: edge       # Behind Ingress/proxy

extraEnvVars:
  - name: KEYCLOAK_ENABLE_HEALTH_ENDPOINTS
    value: "true"
  - name: KEYCLOAK_ENABLE_STATISTICS
    value: "true"
  - name: KC_HOSTNAME_STRICT
    value: "false"  # Set to "true" in production for stable issuer
  - name: KC_HTTP_ENABLED
    value: "true"   # Keep true even in production (TLS terminated at Ingress)
  - name: KEYCLOAK_EXTRA_ARGS
    value: "--import-realm"

# Local development override (environments/local/infrastructure/keycloak.yaml)
service:
  type: LoadBalancer
  httpPort: 4000  # Exposed on host
```

### Realm Configuration

The platform automatically imports the `ameide` realm on startup:

```yaml
# Realm import via ConfigMap
extraVolumes:
  - name: realm-config
    configMap:
      name: keycloak-realm-config
      items:
        - key: "ameide-realm.json"
          path: "ameide-realm.json"

extraVolumeMounts:
  - name: realm-config
    mountPath: "/opt/bitnami/keycloak/data/import"
    readOnly: true

extraEnvVars:
  - name: KEYCLOAK_EXTRA_ARGS
    value: "--import-realm"
```

### Access URLs by Environment

| Environment | Admin Console | OIDC Issuer |
|------------|---------------|-------------|
| Local k3d | `http://auth.ameide.io:4000/admin` | `http://auth.ameide.io:4000/realms/ameide` |
| DevContainer | `http://auth.ameide.io:4000/admin` | `http://auth.ameide.io:4000/realms/ameide` |
| Azure Test | `https://auth.test.ameide.io/admin` | `https://auth.test.ameide.io/realms/ameide` |
| Azure Prod | `https://auth.ameide.io/admin` | `https://auth.ameide.io/realms/ameide` |

### Client Configuration

The platform creates these OAuth clients in the `ameide` realm:

1. **platform-app** - For www-ameide_canvas
   - Client ID: `platform-app`
   - Client Protocol: `openid-connect`
   - Access Type: `confidential`
   - Valid Redirect URIs:
     - `http://platform.ameide.io/*`
     - `https://platform.ameide.io/*`
   - Web Origins: `+` (same as redirect URIs)

2. **admin-cli** - For administrative tasks
   - Client ID: `admin-cli`
   - Access Type: `public`

### Health Checks

```yaml
# Keycloak health endpoints
startupProbe:
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 30
  failureThreshold: 30

livenessProbe:
  httpGet:
    path: /
    port: http

readinessProbe:
  httpGet:
    path: /realms/master
    port: http
```

### Environment Variables

```yaml
extraEnvVars:
  - name: KC_HOSTNAME
    value: "{{ .Values.keycloak.publicHostname }}"  # e.g., https://auth.test.ameide.io
  - name: KC_HOSTNAME_STRICT
    value: "true"   # Enforce stable issuer URL (critical for production)
  - name: KC_HTTP_ENABLED
    value: "true"   # Always true - TLS is handled by Ingress, not Keycloak
  - name: KC_PROXY
    value: "edge"   # Behind proxy/ingress
  - name: KC_PROXY_HEADERS
    value: "xforwarded"  # Trust X-Forwarded-* headers
  - name: KEYCLOAK_ENABLE_HEALTH_ENDPOINTS
    value: "true"
```

**Note on KC_HTTP_ENABLED:**
- Keep this `true` even in production
- TLS termination happens at the Ingress controller
- Keycloak communicates with Ingress over HTTP internally
- The public-facing URL is still HTTPS (configured via KC_HOSTNAME)

### Keycloak Ingress Configuration

```yaml
# templates/keycloak-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: keycloak
  namespace: ameide
  annotations:
    cert-manager.io/cluster-issuer: "{{ .Values.certManager.issuer }}"
    # ssl-redirect: "true"
    # force-ssl-redirect: "{{ .Values.ingress.forceSSL | default \"false\" }}"
    # Increase buffers for large headers/tokens
    # proxy-buffer-size: "128k"
    # proxy-body-size: "8m"
    # proxy-read-timeout: "180s"
    # Backend protocol
    # backend-protocol: "HTTP"
spec:
  # Using Gateway API - no ingressClassName needed
  tls:
    - hosts:
        - {{ .Values.keycloak.hostname }}
      secretName: ameide-wildcard-tls  # Use single wildcard cert
  rules:
    - host: {{ .Values.keycloak.hostname }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: keycloak
                port:
                  number: 4000
```

### Envoy Gateway Configuration

```yaml
# Gateway configuration with automatic header handling
spec:
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      # Envoy automatically handles X-Forwarded-* headers
      # Large headers support configured via BackendTrafficPolicy
```

### NextAuth Configuration for Stable URLs

```typescript
// app/(auth)/auth.ts
export const { auth, signIn, signOut, handlers } = NextAuth({
  providers: [keycloakProvider],
  
  // Trust proxy headers for correct URLs
  trustHost: true,
  
  // Ensure stable URLs
  callbacks: {
    async redirect({ url, baseUrl }) {
      // Always use the public URL (NextAuth v5 uses AUTH_URL)
      const publicUrl = process.env.AUTH_URL || baseUrl;
      if (url.startsWith("/")) return `${publicUrl}${url}`;
      return url;
    }
  }
});

// middleware.ts (if using Express)
app.set('trust proxy', 1);  // Trust first proxy
```

### Environment-Specific Values

```yaml
# values-local.yaml
keycloak:
  publicHostname: "http://auth.ameide.io:4000"
  hostname: "auth.ameide.io"
certManager:
  issuer: "self-signed"

# values-test.yaml  
keycloak:
  publicHostname: "https://auth.test.ameide.io"
  hostname: "auth.test.ameide.io"
certManager:
  issuer: "letsencrypt-staging"

# values-prod.yaml
keycloak:
  publicHostname: "https://auth.ameide.io"
  hostname: "auth.ameide.io"
certManager:
  issuer: "letsencrypt-prod"

# Note: KC_HTTP_ENABLED is always "true" - TLS termination at Ingress
```

## Let's Encrypt SSL/TLS Strategy

### Certificate Management Overview

Let's Encrypt provides free, automated SSL/TLS certificates with 90-day validity. We use cert-manager to automatically request, renew, and manage these certificates across all environments.

### Environment-Specific Approach

| Environment | Issuer | Certificate Type | Validation Method | Auto-Renewal |
|------------|--------|-----------------|-------------------|--------------|
| Local k3d | self-signed | Self-signed | N/A | Yes |
| DevContainer | self-signed | Self-signed | N/A | Yes |
| Azure Test | letsencrypt-staging | Untrusted (staging) | HTTP-01/DNS-01 | Yes |
| Azure Prod | letsencrypt-prod | Trusted | HTTP-01/DNS-01 | Yes |

### Rate Limits (Production)

Let's Encrypt enforces rate limits to ensure fair usage:

- **Certificates per Registered Domain**: 50 per week
- **Duplicate Certificate**: 5 per week
- **Failed Validation**: 5 failures per account, per hostname, per hour
- **New Orders**: 300 per 3 hours

**Mitigation Strategy:**
1. Use staging environment for all testing
2. Request wildcard certificates where possible (`*.ameide.io`)
3. Monitor certificate events and failures
4. Implement retry logic with exponential backoff

### Wildcard vs Individual Certificates

**Wildcard Certificate (`*.ameide.io`):**
- Pros: Single certificate for all subdomains, fewer rate limit concerns
- Cons: Requires DNS-01 challenge, needs Azure DNS API access
- Use for: Production environment

**Individual Certificates:**
- Pros: Simple HTTP-01 challenge, no DNS API needed
- Cons: Multiple certificates to manage, rate limit considerations
- Use for: Test environments, specific subdomains

### Wildcard Certificate Strategy

**IMPORTANT**: Wildcard `*.ameide.io` does NOT cover the apex `ameide.io`. Always include both.

```yaml
# Single wildcard certificate for all production ingresses
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ameide-wildcard
  namespace: ameide
spec:
  secretName: ameide-wildcard-tls  # All ingresses use this secret
  dnsNames:
    - "*.ameide.io"  # Covers www, platform, auth, api, metrics, and any future subdomains
    - "ameide.io"    # Must explicitly include apex domain
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  duration: 2160h    # 90 days
  renewBefore: 720h  # Renew 30 days before expiry
```

**Coverage includes:**
- `www.ameide.io` - SEO/user-friendly URL
- `platform.ameide.io` - Canvas application
- `auth.ameide.io` - Keycloak SSO
- `api.ameide.io` - gRPC-Web gateway
- `metrics.ameide.io` - Grafana dashboards
- Any future subdomains automatically covered

### Using the Wildcard Certificate

```yaml
# All production ingresses reference the same secret
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: platform
spec:
  tls:
    - hosts:
        - platform.ameide.io
      secretName: ameide-wildcard-tls  # Same secret for all
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth
spec:
  tls:
    - hosts:
        - auth.ameide.io
      secretName: ameide-wildcard-tls  # Same secret for all
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
spec:
  tls:
    - hosts:
        - api.ameide.io
      secretName: ameide-wildcard-tls  # Same secret for all
```

### Staging Before Production

```bash
# 1. ALWAYS test with staging first
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ameide-wildcard-staging
  namespace: ameide
spec:
  secretName: ameide-wildcard-staging-tls
  dnsNames:
    - "*.ameide.io"
    - "ameide.io"
  issuerRef:
    name: letsencrypt-staging  # STAGING FIRST
    kind: ClusterIssuer
EOF

# 2. Verify staging certificate works
kubectl describe certificate ameide-wildcard-staging -n ameide

# 3. Only then switch to production
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ameide-wildcard
  namespace: ameide
spec:
  secretName: ameide-wildcard-tls
  dnsNames:
    - "*.ameide.io"
    - "ameide.io"
  issuerRef:
    name: letsencrypt-prod  # PRODUCTION
    kind: ClusterIssuer
EOF
```

### Monitoring and Alerts

```yaml
# ServiceMonitor for cert-manager metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: cert-manager
  namespace: cert-manager
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: cert-manager
  endpoints:
    - port: tcp-prometheus-servicemonitor
      path: /metrics
```

Key metrics to monitor:
- `certmanager_certificate_ready_time_seconds` - Time until certificate expires
- `certmanager_certificate_renewal_errors_total` - Failed renewal attempts
- `certmanager_http01_challenge_errors_total` - HTTP challenge failures

### Troubleshooting Common Issues

**Certificate Not Issuing:**
```bash
# Check certificate status
kubectl describe certificate platform-tls -n ameide

# Check cert-manager logs
kubectl logs -n cert-manager deploy/cert-manager

# Check challenge status
kubectl get challenges -n ameide
```

**DNS-01 Challenge Failures:**
```bash
# Verify DNS propagation
nslookup -type=txt _acme-challenge.platform.ameide.io

# Check Azure DNS permissions
kubectl logs -n cert-manager deploy/cert-manager-webhook
```

## API Gateway Configuration

### Overview

The platform uses **standalone Envoy proxy** as the API gateway, providing:
- gRPC-Web translation for browser clients
- Service mesh routing
- Load balancing
- Circuit breaking
- Observability
- Authentication/authorization
- Rate limiting

### Architecture

```
Browser → api.ameide.io → Envoy Proxy → Backend Services
         (gRPC-Web)        (port 8000)      (gRPC)
```

### Envoy Proxy Configuration (Actual Platform Implementation)

The platform uses standalone Envoy v1.27 for simplicity and consistency:
```yaml
# charts/values/platform/envoy.yaml
image:
  graph: envoyproxy/envoy
  tag: "v1.27-latest"

service:
  type: ClusterIP
  port: 8000
  targetPort: 8000

# Upstream services with actual platform ports
upstreamServices:
  - name: core-cqrs-command
    host: core-cqrs-command
    port: 6000
  - name: core-cqrs-query
    host: core-cqrs-query
    port: 6001
  - name: core-cqrs-subscription
    host: core-cqrs-subscription
    port: 6002

config:
  adminPort: 9901
  enableGrpcWeb: true
  enableCors: true
  corsOrigins:
    - "*"  # Configure appropriately for production

# Local environment override (LoadBalancer for direct access)
# environments/local/infrastructure/envoy.yaml
service:
  type: LoadBalancer
  port: 8000  # Consistent port everywhere
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                codec_type: AUTO
                stat_prefix: ingress_http
                access_log:
                  - name: envoy.access_loggers.stdout
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: backend
                      domains: ["*"]
                      routes:
                        - match:
                            prefix: "/ameide.core.cqrs.command.v1.CommandService/"
                          route:
                            cluster: command_service
                            timeout: 30s
                        - match:
                            prefix: "/ameide.core.cqrs.query.v1.QueryService/"
                          route:
                            cluster: query_service
                            timeout: 30s
                        - match:
                            prefix: "/ameide.core.cqrs.subscription.v1.SubscriptionService/"
                          route:
                            cluster: subscription_service
                            timeout: 0s  # No timeout for streaming
                      cors:
                        allow_origin_string_match:
                          - prefix: "http://localhost"
                          - prefix: "http://ameide.io"
                          - prefix: "https://ameide.io"
                          - prefix: "https://www.ameide.io"
                          - prefix: "http://platform.ameide.io"
                          - prefix: "https://platform.ameide.io"
                          - prefix: "https://test.ameide.io"
                          - prefix: "https://www.test.ameide.io"
                          - prefix: "https://platform.test.ameide.io"
                        allow_methods: GET, PUT, DELETE, POST, OPTIONS
                        allow_headers: keep-alive,user-agent,cache-control,content-type,content-transfer-encoding,x-accept-content-transfer-encoding,x-accept-response-streaming,x-user-agent,x-grpc-web,grpc-timeout,authorization
                        expose_headers: grpc-status,grpc-message
                        max_age: "1728000"
                http_filters:
                  - name: envoy.filters.http.grpc_web
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.grpc_web.v3.GrpcWeb
                  - name: envoy.filters.http.cors
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.Cors
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
    - name: command_service
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      http2_protocol_options: {}
      load_assignment:
        cluster_name: command_service
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: core-cqrs-command.ameide.svc.cluster.local
                      port_value: 6000  # Actual platform port
      health_checks:
        - timeout: 1s
          interval: 10s
          unhealthy_threshold: 2
          healthy_threshold: 2
          grpc_health_check: {}

    - name: query_service
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      http2_protocol_options: {}
      load_assignment:
        cluster_name: query_service
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: core-cqrs-query.ameide.svc.cluster.local
                      port_value: 6001  # Actual platform port

    - name: subscription_service
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      http2_protocol_options: {}
      load_assignment:
        cluster_name: subscription_service
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: core-cqrs-subscription.ameide.svc.cluster.local
                      port_value: 6002  # Actual platform port
```

#### Kubernetes Deployment
```yaml
# envoy-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: envoy
  namespace: ameide
spec:
  replicas: 2
  selector:
    matchLabels:
      app: envoy
  template:
    metadata:
      labels:
        app: envoy
    spec:
      containers:
        - name: envoy
          image: envoyproxy/envoy:v1.27-latest  # Platform uses v1.27
          ports:
            - containerPort: 8000
              name: grpc-web
            - containerPort: 9901
              name: admin
          volumeMounts:
            - name: config
              mountPath: /etc/envoy
          livenessProbe:
            httpGet:
              path: /ready
              port: 9901
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 9901
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
      volumes:
        - name: config
          configMap:
            name: envoy-config
---
apiVersion: v1
kind: Service
metadata:
  name: envoy
  namespace: ameide
spec:
  type: ClusterIP
  ports:
    - port: 8000
      targetPort: 8000
      name: grpc-web
    - port: 9901
      targetPort: 9901
      name: admin
  selector:
    app: envoy
```

#### Ingress for Grafana Metrics Dashboard
```yaml
# metrics-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: metrics
  namespace: ameide
  annotations:
    cert-manager.io/cluster-issuer: "{{ .Values.certManager.issuer }}"
    # ssl-redirect: "true"
    # Security headers for metrics dashboard
    # configuration-snippet: |
      more_set_headers "X-Frame-Options: SAMEORIGIN";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";
spec:
  # Using Gateway API - no ingressClassName needed
  tls:
    - hosts:
        - metrics.ameide.io
      secretName: ameide-wildcard-tls
  rules:
    - host: metrics.ameide.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grafana
                port:
                  number: 3000
---
# api-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  namespace: ameide
  annotations:
    cert-manager.io/cluster-issuer: "{{ .Values.certManager.issuer }}"
    # ssl-redirect: "true"
    # backend-protocol: "HTTP"  # Envoy handles gRPC-Web
    # Disable buffering for streaming
    # proxy-buffering: "off"
    # proxy-request-buffering: "off"
    # Increase body size for gRPC messages
    # proxy-body-size: "10m"
    # Extended timeouts for streaming
    # proxy-read-timeout: "3600"
    # proxy-send-timeout: "3600"
    # proxy-connect-timeout: "60"
    # Choose one: sticky hashing OR ewma latency-aware balancing
    # nginx.ingress.kubernetes.io/upstream-hash-by: "$binary_remote_addr"
    # nginx.ingress.kubernetes.io/load-balance: "ewma"
spec:
  # Using Gateway API - no ingressClassName needed
  tls:
    - hosts:
        - api.ameide.io
      secretName: ameide-wildcard-tls  # Use single wildcard cert
  rules:
    - host: api.ameide.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: envoy
                port:
                  number: 8000
```

### Client Configuration

#### TypeScript/JavaScript Client
```typescript
// gRPC-Web client configuration
import { GrpcWebFetchTransport } from '@protobuf-ts/grpcweb-transport';

const transport = new GrpcWebFetchTransport({
  baseUrl: process.env.NEXT_PUBLIC_API_URL || 'https://api.ameide.io',
  format: 'binary',
  credentials: 'include',
  headers: {
    'Authorization': `Bearer ${accessToken}`,
  },
});

// Use with generated service clients
const commandClient = new CommandServiceClient(transport);
const queryClient = new QueryServiceClient(transport);
```

#### Authentication Flow
```typescript
// Middleware to add auth headers
class AuthInterceptor implements RpcInterceptor {
  async interceptUnary(next: UnaryCall, method: MethodInfo, input: any, options: RpcOptions): Promise<UnaryCall> {
    const token = await getAccessToken();
    options.meta = {
      ...options.meta,
      'Authorization': `Bearer ${token}`,
    };
    return next(method, input, options);
  }
}
```

### Monitoring and Observability

#### Envoy Admin Interface
```bash
# Port-forward to access Envoy admin
kubectl port-forward -n ameide deploy/envoy 9901:9901

# Access admin interface
open http://localhost:9901

# Key endpoints:
# /stats - Metrics
# /clusters - Upstream health
# /config_dump - Current configuration
# /ready - Health check
```

#### Prometheus Metrics
```yaml
# ServiceMonitor for Envoy
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: envoy
  namespace: ameide
spec:
  selector:
    matchLabels:
      app: envoy
  endpoints:
    - port: admin
      path: /stats/prometheus
```

### Rate Limiting Configuration (Future Enhancement)

Rate limiting can be added when needed by deploying a rate limit service (e.g., Envoy's ratelimit service) and configuring the appropriate filter. For now, rate limiting is handled at the application level.

### Circuit Breaker Configuration
```yaml
# Add to cluster configuration
outlier_detection:
  consecutive_5xx: 5
  interval: 30s
  base_ejection_time: 30s
  max_ejection_percent: 50
  enforcing_consecutive_5xx: 100
  enforcing_success_rate: 100
  success_rate_minimum_hosts: 2
  success_rate_request_volume: 10
```

### Security Considerations

1. **TLS Termination**: Handled at Ingress level
2. **Authentication**: JWT validation in Envoy or backend services
3. **Authorization**: Service-level RBAC
4. **Rate Limiting**: Per-user and per-IP limits
5. **DDoS Protection**: CloudFlare or Azure Front Door in production

## Technical Considerations

### Cookie Configuration

```typescript
// Set domain for cross-subdomain cookies
const rootDomain =
  process.env.ENVIRONMENT_NAME === 'test' ? '.test.ameide.io' :
  process.env.NODE_ENV === 'production' ? '.ameide.io' : 
  undefined;  // No domain for local dev

cookies: {
  sessionToken: {
    options: {
      domain: rootDomain,
      sameSite: 'lax',
      secure: process.env.NODE_ENV !== 'development',  // HTTPS in test/prod
      httpOnly: true,
    }
  }
}
```

### CORS Configuration

```typescript
// API Gateway CORS settings - derive from environment
const PUBLIC_DOMAIN = process.env.PUBLIC_DOMAIN || 'ameide.io'; // e.g., 'test.ameide.io'
const PROTOCOL = process.env.NODE_ENV === 'development' ? 'http' : 'https';

const corsOptions = {
  origin: [
    `${PROTOCOL}://${PUBLIC_DOMAIN}`,
    `${PROTOCOL}://www.${PUBLIC_DOMAIN}`,
    `${PROTOCOL}://platform.${PUBLIC_DOMAIN}`,
    `${PROTOCOL}://auth.${PUBLIC_DOMAIN}`,
    `${PROTOCOL}://api.${PUBLIC_DOMAIN}`,
  ],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization', 'x-grpc-web'],
};
```

### HTTPS Enforcement and Security Headers

```yaml
# Development/Test environment headers
metadata:
  annotations:
    # ssl-redirect: "true"
    # configuration-snippet: |
      more_set_headers "Strict-Transport-Security: max-age=86400";  # 1 day, no preload, no includeSubDomains
      more_set_headers "X-Frame-Options: SAMEORIGIN";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";
      # Permissive CSP for development
      more_set_headers "Content-Security-Policy: default-src 'self' https://*.ameide.io; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';";

# Production environment headers
metadata:
  annotations:
    # ssl-redirect: "true"
    # force-ssl-redirect: "true"
    # configuration-snippet: |
      # HSTS with includeSubDomains (covers www.ameide.io)
      # Note: Only add 'preload' after submission to hstspreload.org
      more_set_headers "Strict-Transport-Security: max-age=31536000; includeSubDomains";
      more_set_headers "X-Frame-Options: SAMEORIGIN";  # Allow same-origin embedding (Grafana, etc.)
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "Referrer-Policy: strict-origin";
      # Strict CSP for production (replace with nonces/hashes)
      more_set_headers "Content-Security-Policy: default-src 'self' https://*.ameide.io; script-src 'self'; style-src 'self'; img-src 'self' data: https:; font-src 'self' data:; connect-src 'self' https://*.ameide.io; frame-ancestors 'self';";
```

**HSTS Considerations with www.ameide.io:**
- `includeSubDomains` ensures www.ameide.io is also HTTPS-only
- Prevents SSL stripping attacks on all subdomains
- Users can bookmark either www or apex safely
- Consider HSTS preload submission after production stability

### CSP Migration Strategy

```yaml
# Phase 1: Report-only mode to identify violations
Content-Security-Policy-Report-Only: default-src 'self'; script-src 'self'; report-uri /api/csp-report;

# Phase 2: Add nonces to inline scripts
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-${NONCE}'; style-src 'self' 'nonce-${NONCE}';

# Phase 3: Production-ready CSP
Content-Security-Policy: default-src 'self' https://*.ameide.io; script-src 'self'; style-src 'self'; upgrade-insecure-requests;
```

### OAuth Redirect URIs

Keycloak client configuration must include exact redirect URIs:

**Local (k3d/DevContainer with .test domains):**
```
http://platform.dev.ameide.io:8080/*
https://platform.dev.ameide.io/*
http://dev.ameide.io:8080/*
https://dev.ameide.io/*
```

**Test Environment:**
```
https://platform.test.ameide.io/*
https://test.ameide.io/*
```

**Production:**
```
https://platform.ameide.io/*
https://ameide.io/*
```

**Note**: Using wildcard `/*` at the end allows all paths under the domain. For stricter security, use exact paths:
- `/api/auth/callback/keycloak` - OAuth callback
- `/api/auth/signin` - Sign-in page
- `/api/auth/signout` - Sign-out redirect

## Testing Strategy

### Smoke Tests

Run these commands to verify the setup is working correctly:

```bash
# 1. Verify Keycloak issuer is stable and publicly accessible
curl -s https://auth.test.ameide.io/realms/ameide/.well-known/openid-configuration | jq -r .issuer
# Expected: https://auth.test.ameide.io/realms/ameide

# 2. Verify TLS certificate SANs (Subject Alternative Names)
openssl s_client -connect platform.ameide.io:443 -servername platform.ameide.io </dev/null 2>/dev/null \
  | openssl x509 -noout -text | grep -A1 "Subject Alternative Name"
# Expected: DNS:*.ameide.io, DNS:ameide.io

# 3. Test gRPC-Web API streaming (verify buffering is disabled)
curl -i https://api.ameide.io/ameide.core.cqrs.query.v1.QueryService/Health \
  -H 'x-grpc-web: 1' \
  -H 'content-type: application/grpc-web+proto' \
  --data-binary ''
# Expected: HTTP/2 200, grpc-status header, streaming response

# 4. Verify www redirect to apex
curl -I https://www.ameide.io
# Expected: HTTP/301 or 308 redirect to https://ameide.io

# 5. Test HSTS headers
curl -I https://platform.ameide.io | grep -i strict-transport
# Expected: Strict-Transport-Security: max-age=31536000; includeSubDomains

# 6. Verify DNS resolution (all domains point to same IP)
for domain in ameide.io www.ameide.io platform.ameide.io auth.ameide.io api.ameide.io; do
  echo "$domain: $(dig +short $domain)"
done
# Expected: All domains resolve to the same ingress IP

# 7. Test OAuth discovery endpoint
curl -s https://auth.ameide.io/realms/ameide/.well-known/openid-configuration | jq '.authorization_endpoint, .token_endpoint'
# Expected: Valid OAuth endpoints

# 8. Verify CoreDNS (from within cluster)
kubectl run -it --rm debug --image=nicolaka/netshoot --restart=Never -- \
  nslookup auth.ameide.io
# Expected: Resolves to ingress controller IP (if CoreDNS override is active)
```

### Local Testing Checklist
- [ ] `/etc/hosts` file updated with .test domains (DevContainer)
- [ ] `manage-host-domains.sh` script executed for macOS host
- [ ] All services accessible via dev.ameide.io domains on ports 8080/8443
- [ ] OAuth flow works with proper redirects
- [ ] Cookies set with correct domain (`.dev.ameide.io` for local)
- [ ] API calls work across subdomains
- [ ] No CORS errors in browser console
- [ ] cert-manager certificates created (`ameide-wildcard-tls`)

### Integration Testing
- [ ] DNS records resolve correctly (use `dig` or `nslookup`)
- [ ] SSL certificates valid (check with `openssl` commands)
- [ ] Certificate SANs include wildcard and apex
- [ ] OAuth flow works end-to-end
- [ ] Session persistence across subdomains
- [ ] API authentication works with JWT tokens
- [ ] gRPC-Web streaming works without buffering

### Production Validation
- [ ] DNS propagation complete (check with `dig @8.8.8.8`)
- [ ] SSL certificates active and trusted
- [ ] OAuth callbacks working with production URLs
- [ ] Monitoring shows healthy traffic
- [ ] No mixed content warnings in browser
- [ ] HSTS headers present with `includeSubDomains`
- [ ] Performance metrics acceptable (TTFB, SSL handshake time)

## Benefits

1. **Consistency**: Same URLs across all environments
2. **Security**: Proper SSL/TLS in test/production
3. **Simplicity**: No port numbers in URLs
4. **OAuth**: Simplified callback configuration
5. **Debugging**: Easier to trace issues
6. **Production-Ready**: Local mimics production

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| DNS propagation delays | Users can't access | Use low TTLs during migration |
| SSL certificate issues | Browser warnings | Test with staging certs first |
| Cookie domain mismatch | Auth failures | Test cookie settings thoroughly |
| CORS errors | API calls fail | Configure CORS properly |
| Ingress misconfiguration | Services unreachable | Test with curl before DNS switch |

## Success Criteria

- [ ] All environments use ameide.io domains
- [ ] OAuth flows work identically everywhere
- [ ] No hardcoded localhost references
- [ ] SSL/TLS working in test/production
- [ ] Developers can run locally with ameide.io
- [ ] Documentation updated

Love it. Here’s a clean, incremental rollout where every step is independently testable and has a clear “done” signal. I’ve grouped by phase; each step has Goal → Do → Test → Rollback.

# Phase 0 — Repo hygiene (day 0)

### Step 0.1 — Helm helpers + env naming

**Goal:** Deterministic URLs across envs; no templating in values files.
**Do:** Add `_helpers.tpl` (your `ameide.*` helpers). Standardize to `NEXTAUTH_URL` (keep `PLATFORM_URL` as fallback).
**Test:**

```bash
helm template charts/ . -f values-local.yaml | grep -E 'https?://(auth|platform|api)\.'
# should show fully-rendered URLs
```

**Rollback:** Revert helpers; charts still work with literal values.

---

# Phase 1 — Local k3d (week 1)

### Step 1.1 — Cluster + Envoy Gateway (✅ IMPLEMENTED)

**Goal:** Reach a Gateway from `platform.dev.ameide.io`.
**Status:** ✅ Complete - k3d cluster with Envoy Gateway v1.5.0
**Test:**

```bash
kubectl get gateway -n ameide
curl -I http://platform.dev.ameide.io:8080  # Via Gateway on port 8080
```

### Step 1.2 — Local domains (✅ IMPLEMENTED)

**Goal:** Hostname resolution for .test domains.
**Status:** ✅ Complete - DevContainer uses /etc/hosts, macOS uses dnsmasq
**Test:**

```bash
# In DevContainer
cat /etc/hosts | grep dev.ameide.io
# On macOS (if configured)
./.devcontainer/manage-host-domains.sh status
```

### Step 1.3 — Platform Ingress (HTTP) (✅ IMPLEMENTED)

**Goal:** Serve applications at `http://platform.dev.ameide.io:8080`.
**Do:** Apply `platform-ingress.yaml`; deploy `www-ameide-canvas` Service.
**Test:**

```bash
curl -I http://platform.ameide.io
```

**Rollback:** `kubectl delete ingress platform -n ameide`.

### Step 1.4 — Keycloak local (✅ IMPLEMENTED)

**Goal:** Keycloak reachable at `auth.dev.ameide.io`.
**Status:** ✅ Complete - Keycloak deployed with stable issuer
**Configuration:**
* Uses both `.test` for local and `.io` for consistency
* `KC_HOSTNAME_STRICT=false` for development flexibility
* `KC_HTTP_ENABLED=true`

**Test:**
```bash
curl -s http://auth.dev.ameide.io:4000/realms/ameide/.well-known/openid-configuration | jq -r .issuer
# Returns appropriate issuer based on access
```

### Step 1.5 — Envoy (gRPC-Web) + API (✅ IMPLEMENTED)

**Goal:** gRPC-Web reachable at `http://api.dev.ameide.io:8080`.
**Status:** ✅ Complete - Envoy proxy deployed on port 8000
**Test:**

```bash
curl -i http://api.dev.ameide.io:8080/ameide.core.cqrs.query.v1.QueryService/Health \
  -H 'x-grpc-web: 1' -H 'content-type: application/grpc-web+proto' --data-binary ''
```

### Step 1.6 — Local OAuth round-trip (✅ IMPLEMENTED)

**Goal:** Full login flow using .test domains.
**Status:** ✅ Complete - OAuth working with .test domains
**Configuration:**
* `NEXTAUTH_URL=https://platform.dev.ameide.io`
* `AUTH_KEYCLOAK_ISSUER=http://auth.dev.ameide.io:4000/realms/ameide`

### Step 1.7 — Local TLS (✅ IMPLEMENTED)

**Goal:** HTTPS with self-signed certificates.
**Status:** ✅ Complete - cert-manager with self-signed ClusterIssuer
**Test:**

```bash
curl -I https://platform.dev.ameide.io -k   # self-signed cert
kubectl get certificate -n ameide ameide-wildcard
```

---

# Phase 2 — DevContainer networking (week 2)

### Step 2.1 — Host gateway mapping

**Goal:** Container reaches host k3d via host-gateway.
**Do:** Add `extra_hosts` and envs (you updated already).
**Test (inside container):**

```bash
getent hosts platform.ameide.io
curl http://platform.ameide.io
```

**Rollback:** Remove `extra_hosts`.

### Step 2.2 — End-to-end auth from container

**Goal:** Login works from inside DevContainer UI/browser.
**Do:** Ensure the same envs; cookies not scoped to domain in dev (no `domain=`).
**Test:** Sign in and hit API from app running in container; no CORS errors.
**Rollback:** Revert envs.

---

# Phase 3 — Azure TEST (week 3)

### Step 3.1 — DNS subzone + delegation

**Goal:** Public resolution for `*.test.ameide.io`.
**Do:** Create zone, add NS delegation in parent.
**Test:**

```bash
dig NS test.ameide.io +short @8.8.8.8
```

**Rollback:** Remove NS at parent; delete subzone.

### Step 3.2 — Ingress + cert-manager (staging)

**Goal:** Valid (staging) cert issued via HTTP-01.
**Do:** Install cert-manager v1.15.x; create a dummy Ingress with `cert-manager.io/cluster-issuer: letsencrypt-staging`.
**Test:**

```bash
kubectl describe certificate dummy -n ameide | grep -i "Ready"
```

**Rollback:** Delete Certificate/Ingress.

### Step 3.3 — Keycloak on TEST

**Goal:** `https://auth.test.ameide.io` with strict hostname.
**Do:** Deploy with `KC_HOSTNAME=https://auth.test.ameide.io`, `KC_HTTP_ENABLED=true`; Ingress TLS (staging).
**Test:**

```bash
curl -s https://auth.test.ameide.io/realms/ameide/.well-known/openid-configuration | jq -r .issuer
# expect: https://auth.test.ameide.io/realms/ameide
```

**Rollback:** Scale down; remove ingress.

### Step 3.4 — Platform on TEST

**Goal:** `https://platform.test.ameide.io` login works.
**Do:** Set `NEXTAUTH_URL=https://platform.test.ameide.io`. Update Keycloak client redirects to TEST URLs.
**Test:** Browser login success; session cookie scoped to `.test.ameide.io`.
**Rollback:** Remove TEST redirect URIs; redeploy app with previous envs.

### Step 3.5 — Envoy/API on TEST

**Goal:** `https://api.test.ameide.io` streams (buffering off), CORS OK.
**Do:** Deploy Envoy + API Ingress (staging cert). Add apex + platform origins in CORS.
**Test:**

```bash
curl -i https://api.test.ameide.io/ameide.core.cqrs.query.v1.QueryService/Health \
  -H 'x-grpc-web: 1' -H 'content-type: application/grpc-web+proto' --data-binary ''
```

**Rollback:** Delete API Ingress/Service.

### Step 3.6 — Grafana on TEST (metrics. subdomain)

**Goal:** `https://metrics.test.ameide.io` with security headers.
**Do:** Add Ingress + staging TLS + headers (no `DENY` if you embed).
**Test:**

```bash
curl -sI https://metrics.test.ameide.io | grep -Ei 'strict-transport|content-security-policy'
```

**Rollback:** Delete Ingress.

---

# Phase 4 — Prod hardening (week 4)

### Step 4.1 — DNS-01 + wildcard (dry-run)

**Goal:** Prove wildcard issuance path before prod.
**Do:** Configure Workload Identity + `letsencrypt-staging` DNS-01 for `*.ameide.io` in a **temporary** Certificate (different secret).
**Test:**

```bash
kubectl describe certificate ameide-wildcard-staging -n ameide | grep Ready
```

**Rollback:** Delete temp Certificate; no prod impact.

### Step 4.2 — Prod wildcard certificate

**Goal:** Real `ameide-wildcard-tls` in prod.
**Do:** Switch issuer to `letsencrypt-prod` DNS-01; SANs: `*.ameide.io` and `ameide.io`.
**Test:**

```bash
openssl s_client -connect platform.ameide.io:443 -servername platform.ameide.io </dev/null 2>/dev/null \
 | openssl x509 -noout -text | grep -A1 "Subject Alternative Name"
```

**Rollback:** Point Ingress back to per-host secrets (if any), or pause cert-manager.

### Step 4.3 — Prod Gateway + security headers

**Goal:** Uniform HTTPS + sane headers.
**Do:** Apply prod annotations (HSTS 1y with `includeSubDomains`, no `preload` yet).
**Test:**

```bash
curl -sI https://ameide.io | grep -i strict-transport
```

**Rollback:** Reapply previous annotations.

### Step 4.4 — Prod DNS (short TTL rollout)

**Goal:** All prod hostnames resolve to Ingress.
**Do:** Apex A → Ingress IP; subdomains CNAME → apex; TTL 300.
**Test:**

```bash
for d in ameide.io www.ameide.io platform.ameide.io auth.ameide.io api.ameide.io; do
  echo "$d: $(dig +short $d @8.8.8.8)"
done
```

**Rollback:** Restore previous IPs/records.

### Step 4.5 — Prod OAuth round-trip

**Goal:** Login works at scale; sessions valid across subdomains.
**Do:** Set `NEXTAUTH_URL=https://platform.ameide.io`; cookie domain `.ameide.io`.
**Test:** Browser login; cross-subdomain API call succeeds; no mixed content/CORS.
**Rollback:** Flip Ingress backends to old revision (blue/green).

---

# Phase 5 — Nice-to-haves (post-cutover)

### Step 5.1 — www redirect

**Goal:** Consistent canonical URLs.
**Do:** Add redirect filter in HTTPRoute for www to apex.
**Test:** `curl -I https://www.ameide.io` → 301/308 to apex.
**Rollback:** Remove annotation.

### Step 5.2 — Rate limiting (only when ready)

**Goal:** Add protection without breaking traffic.
**Do:** Deploy ratelimit service, add Envoy cluster, then enable filter with low enforcement.
**Test:** Synthetic burst → 429 surfaced, normal traffic OK.
**Rollback:** Remove filter (no restart needed for other paths).

---

## Lightweight automation (optional but helpful)

* Makefile targets: `make k3d-up`, `make local-tls`, `make smoke-local`, `make smoke-test`, `make smoke-prod`.
* CI job runs `helm template` and a k3d smoke test (curl + grpc-web check).

If you want, I can package the above into a checklist + `make smoke-*` scripts so every step has a one-liner verifier.

## Dependencies

- Azure DNS zone ownership
- SSL certificates (Let's Encrypt)
- Gateway controller (Envoy Gateway)
- cert-manager for automatic SSL
- DNS update permissions

## Related Documents

- [104-keycloak-config-v2.md](./104-keycloak-config-v2.md) - OAuth configuration
- [100-port-numbering-forward.md](./100-port-numbering-forward.md) - Port assignments
- [Infrastructure README](../infra/kubernetes/README.md) - K8s setup

---

## 107-gateway-api-migration.md (old)

# Gateway API Migration Strategy

## Executive Summary

Migrate from Ingress v1 to Gateway API for cleaner, more maintainable routing configuration that better aligns with our domain strategy and microservices architecture.

## Current State (✅ MIGRATED - August 2025, Enhanced January 2025)

### Problems with Ingress v1 (RESOLVED)
1. ~~Template Complexity~~ → ✅ Clean HTTPRoute templates
2. ~~Non-Standard Values~~ → ✅ Standard Gateway API resources
3. ~~Maintenance Burden~~ → ✅ Simple, declarative configs
4. ~~Poor Separation~~ → ✅ Gateway (infra) vs HTTPRoute (app)

### What We Have Now
- ✅ Envoy Gateway v1.5.0 (replaced NGINX Ingress)
- ✅ Unified Envoy for all traffic (HTTP, HTTPS, gRPC-Web)
- ✅ cert-manager integrated with Gateway API
- ✅ Domain strategy with .test for local, .io for production
- ✅ **HTTPS-only architecture** with HTTP→HTTPS redirects
- ✅ **Split endpoints for OAuth** - External HTTPS, internal HTTP
- ✅ **X-Forwarded headers** properly configured for proxies

## Gateway API Benefits

### Clean Architecture
```yaml
# Before: Ingress v1 (mixed concerns)
apiVersion: networking.k8s.io/v1
kind: Ingress
spec:
  rules:
    - host: platform.ameide.io
      http:
        paths:
          - path: /
            backend:  # Routing + backend mixed
              service:
                name: www-ameide-canvas
                port:
                  number: 3001

# After: Gateway API (separated concerns)
---
# Infrastructure team owns Gateway
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ameide
spec:
  gatewayClassName: envoy
  listeners:
    - name: http
      port: 80
      protocol: HTTP
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        certificateRefs:
          - name: ameide-wildcard-tls
---
# App team owns HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: platform
spec:
  parentRefs:
    - name: ameide
  hostnames:
    - platform.ameide.io
  rules:
    - backendRefs:
        - name: www-ameide-canvas
          port: 3001
```

### Perfect Domain Strategy Alignment

Each service gets its own HTTPRoute (aligned with port spec):

**Local Development (.test domains on ports 8080/8443):**
- `platform.dev.ameide.io` → HTTPRoute → www-ameide-canvas (port 3001) ✅
- `dev.ameide.io` → HTTPRoute → www-ameide (port 3000) ✅
- `portal.dev.ameide.io` → HTTPRoute → www-ameide-portal (port 3002) ✅
- `portal-canvas.dev.ameide.io` → HTTPRoute → www-ameide-portal-canvas (port 3003) ✅
- `api.dev.ameide.io` → HTTPRoute → envoy (port 8000 for gRPC-Web) ✅
- `auth.dev.ameide.io` → HTTPRoute → keycloak (port 4000) ⏳
- `metrics.dev.ameide.io` → HTTPRoute → grafana (port 9001) ⏳

**Production (.io domains on standard ports):**
- Same services on `.io` domains with standard HTTPS (443)

**Port Alignment**: All services follow `backlog/100-port-numbering-forward.md`

### Native gRPC Support

```yaml
apiVersion: gateway.networking.k8s.io/v1  # GA since Gateway API v1.1.0
kind: GRPCRoute
metadata:
  name: api
spec:
  parentRefs:
    - name: ameide
  hostnames: [api.ameide.io]
  rules:
    - backendRefs:
        - name: envoy
          port: 8000
```

## Implementation Plan

### Direct Adoption Approach (Selected)
**Decision**: Skip Ingress v1 fixes, go directly to Gateway API

### ✅ Phase 1: Infrastructure Setup (COMPLETE)
**Goal**: Deploy Gateway API infrastructure

1. **Envoy Gateway Configuration** ✅
   - Configured in helmfile.yaml using official OCI chart v1.5.0
   - Provides unified Envoy control plane for all traffic types

2. **Gateway Resources Created** ✅
   - Gateway chart at `charts/platform/gateway/` with:
     - GatewayClass for Envoy controller
     - Gateway with HTTP, HTTPS, and gRPC listeners
     - HTTP to HTTPS redirect via HTTPRoute
     - Namespace labeling for access control

3. **Cert-Manager Integration** ✅
   - Gateway annotated with `cert-manager.io/cluster-issuer`
   - Automatic certificate provisioning in Gateway namespace
   - Environment-specific issuers (self-signed, staging, prod)

4. **Policy Framework** ✅
   - BackendTLSPolicy templates for mTLS to services
   - BackendTrafficPolicy for retries, timeouts, circuit breakers
   - Environment-specific configurations (dev, staging, prod)

5. **Cross-Namespace Support** ✅
   - ReferenceGrant templates for cross-namespace access
   - Selector-based namespace access with `gateway-access: allowed` label

### ✅ Phase 2: Service Migration (COMPLETE - August 11, 2025)
**Goal**: Convert all services to use HTTPRoute

1. **HTTPRoute Template Created** ✅
   - Created flexible template at `www-ameide-canvas/templates/httproute.yaml`
   - Supports single or multiple hostnames
   - Optional custom rules and filters
   - Route-level timeouts (Gateway API v1.1+)
   - Cross-namespace gateway references

2. **Services Migrated** ✅:
   - ✅ www-ameide-canvas → platform.dev.ameide.io, platform.ameide.io
   - ✅ www-ameide → dev.ameide.io, www.dev.ameide.io
   - ✅ www-ameide-portal → portal.dev.ameide.io
   - ✅ www-ameide-portal-canvas → portal-canvas.dev.ameide.io
   - ✅ API routes → api.dev.ameide.io, api.ameide.io (via HTTPRoute with CORS)
   - ⏳ keycloak → auth.dev.ameide.io (pending final migration)
   - ⏳ grafana → metrics.dev.ameide.io (pending final migration)

2. **Create GatewayClass**
   ```yaml
   apiVersion: gateway.networking.k8s.io/v1
   kind: GatewayClass
   metadata:
     name: envoy
   spec:
     controllerName: gateway.envoyproxy.io/gatewayclass-controller
   ```

3. **Deploy Gateway**
   ```yaml
   apiVersion: gateway.networking.k8s.io/v1
   kind: Gateway
   metadata:
     name: ameide
     namespace: ameide
   spec:
     gatewayClassName: envoy
     listeners:
       - name: http
         port: 80
         protocol: HTTP
         allowedRoutes:
           namespaces:
             from: Same
       - name: https
         port: 443
         protocol: HTTPS
         tls:
           mode: Terminate
           certificateRefs:
             - name: ameide-wildcard-tls
         allowedRoutes:
           namespaces:
             from: Same
   ```

4. **Create HTTPRoutes for Each Service**
   - Start with non-critical service
   - Test thoroughly
   - Roll out to remaining services

### ✅ Phase 3: Migration (COMPLETE - August 2025)
**Goal**: Complete cutover to Gateway API

1. **DNS/Load Balancer Updated** ✅
   - Gateway service exposed on ports 8080/8443 for local
   - k3d loadbalancer configured for port mappings

2. **Ingress Resources Deprecated** ✅
   - No Ingress objects remain
   - All routing via HTTPRoute/GRPCRoute

3. **NGINX Ingress Controller Removed** ✅
   - Replaced entirely by Envoy Gateway
   - All resources cleaned up

## TLS Certificate Ownership (✅ IMPLEMENTED)

**Single Source of Truth: The Gateway owns all TLS certificates.**

- **Wildcard certificate** (`*.dev.ameide.io` for local, `*.ameide.io` for prod) ✅
- **cert-manager** provisions/renews automatically ✅
- **Self-signed ClusterIssuer** for local development ✅
- **HTTPRoutes/GRPCRoutes** are TLS-agnostic - they only specify hostnames ✅
- **Services** speak plain HTTP internally (no TLS configuration needed) ✅

This eliminates:
- Certificate distribution complexity
- Cross-namespace secret references
- Service-level TLS configuration
- Certificate renewal coordination

## Helm Chart Updates

### Current (Broken)
```yaml
# values.yaml - Non-standard structure
ingress:
  hosts:
    - host: platform.ameide.io
      paths:
        - path: /
          pathType: Prefix
          # No backend specification!
```

### With Gateway API
```yaml
# values.yaml - Simple, clear structure
httproute:
  enabled: true
  gateway: ameide
  hostname: platform.ameide.io
  service:
    name: www-ameide-canvas  # Optional, can default
    port: 3001
```

### Template Simplification
```yaml
# templates/httproute.yaml - No complex logic needed
{{- if .Values.httproute.enabled }}
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: {{ include "chart.fullname" . }}
spec:
  parentRefs:
    - name: {{ .Values.httproute.gateway }}
  hostnames:
    - {{ .Values.httproute.hostname }}
  rules:
    - backendRefs:
        - name: {{ .Values.httproute.service.name | default (include "chart.fullname" .) }}
          port: {{ .Values.httproute.service.port | default .Values.service.port }}
{{- end }}
```

## Unified Envoy Stack (✅ ACHIEVED)

### Previous: Two Separate Systems
1. ~~NGINX Ingress → Web traffic~~
2. ~~Standalone Envoy Proxy → gRPC-Web translation~~

### Current: Single Envoy Gateway ✅
```
Envoy Gateway v1.5.0
  ├── HTTP/HTTPS → Web services ✅
  ├── gRPC → Native gRPC services ✅
  └── gRPC-Web → Browser clients ✅
```

Achieved Benefits:
- ✅ Single control plane
- ✅ Consistent configuration
- ✅ Better observability
- ✅ Reduced complexity

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Gateway API maturity | Use stable v1 features only |
| Team familiarity | Training and documentation |
| Migration disruption | Parallel run with gradual cutover |
| Rollback complexity | Backup Gateway configuration before changes |

## Success Criteria

- ✅ All services accessible via Gateway API
- ✅ No Ingress v1 resources remaining
- ✅ Simplified Helm templates with HTTPRoute
- ✅ Unified Envoy stack (v1.5.0)
- ✅ Documentation updated
- ⏳ Team training on Gateway API (ongoing)

## Timeline (✅ COMPLETE)

- ✅ **Phase 1**: Gateway API infrastructure deployed (August 2025)
- ✅ **Phase 2**: Services migrated to HTTPRoute (August 2025)
- ✅ **Phase 3**: Verification and documentation (August 2025)

## Implementation Summary

**Successfully adopted Gateway API with Envoy Gateway v1.5.0**

Achieved:
- ✅ No Ingress v1 resources remain
- ✅ Clean, modern architecture implemented
- ✅ Unified Envoy stack for all traffic types
- ✅ Future-proof with Kubernetes Gateway API v1.0+
- ✅ Environment-specific configurations (.test for local, .io for prod)
- ✅ cert-manager integration with self-signed certificates

## Remaining Gaps

### Minor Pending Items
1. **Keycloak HTTPRoute**: Currently accessible but needs formal HTTPRoute definition
2. **Grafana HTTPRoute**: Metrics dashboard needs HTTPRoute for metrics.dev.ameide.io
3. **Production TLS**: Configure Let's Encrypt issuers for staging/production

### Future Enhancements
1. **GRPCRoute**: Consider native GRPCRoute for API services (currently using HTTPRoute)
2. **Traffic Policies**: Implement rate limiting, circuit breakers via BackendTrafficPolicy
3. **Observability**: Add Gateway metrics to Prometheus/Grafana

## References

- [Gateway API Documentation](https://gateway-api.sigs.k8s.io/)
- [Envoy Gateway](https://gateway.envoyproxy.io/)
- [Gateway API vs Ingress](https://gateway-api.sigs.k8s.io/guides/migrating-from-ingress/)
- [Cert-manager Gateway API Support](https://cert-manager.io/docs/usage/gateway/)

---

## 109-local-development-domain-access.md (old)

> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# 109: Local Development Domain Access

## Problem Statement

When developing AMEIDE Core platform locally using DevContainer + k3d, accessing services from the macOS host requires proper domain resolution for authentication and routing to work correctly. The platform uses domain-based routing (e.g., `auth.ameide.io` for Keycloak), but adding production domains to `/etc/hosts` would break access to the actual production site hosted on Azure.

## North Star Goals

* Click a URL, auth works, routes resolve, no "but on my machine..."
* No collisions with production DNS
* Minimal sudo (one-time only), no privileged ports during day-to-day
* Works on macOS + VS Code Dev Container + k3d

## Recommended Solution: The Opinionated Path

### Core Design Decisions

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **Domains** | `*.dev.ameide.io` | Reserved TLD (RFC 6761), no mDNS conflicts |
| **DNS** | `dnsmasq` wildcard → 127.0.0.1 | No manual `/etc/hosts` maintenance |
| **TLS** | cert-manager with self-signed issuer | Kubernetes-native cert management |
| **Cluster** | k3d | Lightweight, fast, production-like |
| **Routing** | Gateway API + Envoy Gateway | Production parity, host-based routing |
| **Host Ports** | 8080/8443 | Non-privileged, no daily sudo needed |
| **Auth** | Keycloak with edge proxy config | Proper HTTPS redirects, secure cookies |

### Why This Over Alternatives

#### Domain Choice: `.test` vs `.local` vs Others

| Domain | Pros | Cons | Verdict |
|--------|------|------|---------|
| `.local` | Common in tutorials | mDNS conflicts on macOS, VPN issues, no wildcards | ❌ Avoid |
| `.test` | Reserved for testing, no conflicts, wildcard-friendly | Requires DNS setup | ✅ Recommended |
| `.localhost` | Auto-loopback in browsers | Some tools don't support it | ⚠️ Limited |
| `.127.0.0.1.nip.io` | Zero setup | Ugly URLs, external dependency | 🔧 Quick demos only |

#### Port Strategy: Why 8080/8443

* **Ports < 1024 are privileged** - require sudo on macOS/Linux
* **VS Code won't forward privileged ports** from DevContainers
* **Industry standard** - Most K8s local dev uses 8080/8443
* **Clean separation** - Different from production, clear it's local

## Implementation Guide

### One-Time Setup (5-10 minutes)

```bash
# 1. Install tools
brew install k3d kubectl dnsmasq jq helm helmfile

# 2. Configure wildcard DNS for *.dev.ameide.io -> 127.0.0.1
CONF_DIR="$(brew --prefix)/etc/dnsmasq.d"
sudo mkdir -p "$CONF_DIR"
echo 'address=/.dev.ameide.io/127.0.0.1' | sudo tee "$CONF_DIR/ameide.conf" >/dev/null
sudo brew services start dnsmasq
sudo mkdir -p /etc/resolver
echo 'nameserver 127.0.0.1' | sudo tee /etc/resolver/dev.ameide.io >/dev/null

# Note: TLS certificates are now managed by cert-manager in Kubernetes
# No need for mkcert - self-signed certificates are automatically generated

# 4. Create k3d registry (one-time, outside cluster)
k3d registry create ameide-reg --port 32000

# 5. Create k3d cluster with registry config (use config file for repeatability)
cat > k3d-ameide.yaml <<EOF
apiVersion: k3d.io/v1alpha5
kind: Simple
metadata:
  name: ameide
servers: 1
agents: 1
ports:
  - port: 8080:80@loadbalancer
  - port: 8443:443@loadbalancer
registries:
  use:
    - k3d-ameide-reg:32000
options:
  k3s:
    extraArgs:
      - arg: --disable=traefik
        nodeFilters:
          - server:*
EOF

k3d cluster create --config k3d-ameide.yaml

# Note: For Docker Desktop, add to Settings → Docker Engine:
# "insecure-registries": ["docker.io/ameide"]

# 5. Install Gateway API CRDs and Envoy Gateway (idempotent)
kubectl get gatewayclasses.gateway.networking.k8s.io >/dev/null 2>&1 || \
  kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/latest/download/standard-install.yaml

kubectl get ns envoy-gateway-system >/dev/null 2>&1 || \
  kubectl apply -f https://github.com/envoyproxy/gateway/releases/latest/download/install.yaml

# Verify installation
kubectl wait --for=condition=Ready -n envoy-gateway-system deployment/envoy-gateway --timeout=60s
kubectl get gatewayclass
kubectl get crd | grep gateway.networking.k8s.io

# 6. Create and configure namespace
kubectl create ns ameide --dry-run=client -o yaml | kubectl apply -f -
kubectl label ns ameide gateway-access=allowed --overwrite

# 7. Install cert-manager and configure self-signed issuer
# This is handled automatically by Helmfile deployment
helmfile -e local sync -l name=cert-manager
helmfile -e local sync -l name=cert-manager-config

# Certificate will be automatically created as ameide-wildcard-tls
```

### Gateway Configuration

```yaml
# gitops/ameide-gitops/sources/values/local/apps/platform/gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ameide
  namespace: ameide
spec:
  gatewayClassName: envoy
  listeners:
    - name: http
      hostname: "*.dev.ameide.io"
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              gateway-access: allowed
    - name: https
      hostname: "*.dev.ameide.io"
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: ameide-wildcard-tls
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              gateway-access: allowed
```

### HTTPRoute Examples

```yaml
# Keycloak
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: keycloak
  namespace: ameide
spec:
  parentRefs:
    - name: ameide
  hostnames: ["auth.dev.ameide.io"]
  rules:
    - backendRefs:
        - name: keycloak
          port: 8080

---
# Main Application
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: www-ameide
  namespace: ameide
spec:
  parentRefs:
    - name: ameide
  hostnames: ["dev.ameide.io", "www.dev.ameide.io"]
  rules:
    - backendRefs:
        - name: www-ameide
          port: 3000

---
# Platform Application
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: www-ameide-platform
  namespace: ameide
spec:
  parentRefs:
    - name: ameide
  hostnames: ["platform.dev.ameide.io"]
  rules:
    - backendRefs:
        - name: www-ameide-platform
          port: 3001

---
# Grafana Metrics Dashboard
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: grafana
  namespace: ameide
spec:
  parentRefs:
    - name: ameide
  hostnames: ["metrics.dev.ameide.io"]
  rules:
    - backendRefs:
        - name: observability-dashboard-grafana
          port: 3000
          namespace: ameide
```

### Keycloak Configuration

```yaml
# Chart values (in environments/local/infrastructure/keycloak.yaml)
extraEnvVars:
  - name: KC_PROXY
    value: "edge"  # Behind Gateway/proxy
  - name: KC_HOSTNAME
    value: "auth.dev.ameide.io"
  - name: KC_HOSTNAME_PORT
    value: "8443"  # HTTPS port users access
  - name: KC_HTTP_RELATIVE_PATH
    value: "/"  # Root path for Keycloak
  - name: KC_HTTP_ENABLED
    value: "true"  # Allow HTTP internally (Gateway handles TLS)
```

**Note**: Envoy Gateway automatically forwards `X-Forwarded-Proto` and `X-Forwarded-Host` headers, which Keycloak uses when `KC_PROXY=edge` is set.

```bash
# Configure realm redirect URIs
kcadm.sh config credentials \
  --server https://auth.dev.ameide.io \
  --realm master --user admin --password 'changeme'

# Get client ID
CID=$(kcadm.sh get clients -r ameide --fields id,clientId \
  | jq -r '.[] | select(.clientId=="ameide-app").id')

# Update redirect URIs and web origins
kcadm.sh update clients/$CID -r ameide \
  -s 'redirectUris=["https://dev.ameide.io/*","https://platform.dev.ameide.io/*","https://portal.dev.ameide.io/*"]' \
  -s 'webOrigins=["https://dev.ameide.io","https://platform.dev.ameide.io","https://portal.dev.ameide.io"]'

# Quick iteration option: use webOrigins=["+"] then lock down later
```

### DevContainer Configuration

```json
// .devcontainer/devcontainer.json
{
  // Mount Docker socket so k3d can work inside the container
  "mounts": [
    "source=/var/run/docker.sock,target=/var/run/docker.sock,type=bind"
  ],
  
  // Use Docker from the host (not Docker-in-Docker)
  "features": {
    "ghcr.io/devcontainers/features/docker-outside-of-docker:1": {}
  },
  
  "forwardPorts": [8080, 8443, 3000, 3001, 3002, 4000, 8000, 9001],
  
  "portsAttributes": {
    "8080": { "label": "Gateway HTTP", "onAutoForward": "notify" },
    "8443": { "label": "Gateway HTTPS", "onAutoForward": "notify" },
    "3000": { "label": "Main App (direct debug)", "onAutoForward": "silent" },
    "3001": { "label": "Canvas (direct debug)", "onAutoForward": "silent" },
    "3002": { "label": "Portal (direct debug)", "onAutoForward": "silent" },
    "4000": { "label": "Keycloak (direct debug)", "onAutoForward": "silent" },
    "8000": { "label": "API Gateway (direct debug)", "onAutoForward": "silent" },
    "9001": { "label": "Grafana (direct debug)", "onAutoForward": "silent" }
  }
}
```

### k3d Cluster Creation (DevContainer)

```bash
# .devcontainer/postCreateCommand.sh
k3d cluster create ameide \
  --api-port 45099 \
  --servers 1 \
  --agents 1 \
  --port "8080:80@loadbalancer" \
  --port "8443:443@loadbalancer" \
  --port "3000:3000@loadbalancer" \  # Keep for direct debug
  --port "3001:3001@loadbalancer" \
  --port "3002:3002@loadbalancer" \
  --port "4000:4000@loadbalancer" \
  --port "8000:8000@loadbalancer" \
  --registry-create docker.io/ameide \
  --k3s-arg "--disable=traefik@server:0" \
  --wait
```

## CORS Configuration for API

The API Gateway at `api.dev.ameide.io` requires proper CORS headers for browser-based gRPC-Web clients. This is configured at the Gateway level:

```yaml
# HTTPRoute with CORS (in gateway/templates/httproute-api.yaml)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: ameide-api
  namespace: ameide
spec:
  parentRefs:
    - name: ameide
  hostnames:
    - api.ameide.io
    - api.dev.ameide.io
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      filters:
        # Forward proxy headers
        - type: RequestHeaderModifier
          requestHeaderModifier:
            add:
              - name: X-Forwarded-Proto
                value: https
        # Add CORS headers
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: Access-Control-Allow-Origin
                value: "*"  # Specify actual origins in production
              - name: Access-Control-Allow-Methods
                value: "GET, POST, PUT, DELETE, OPTIONS, PATCH"
              - name: Access-Control-Allow-Headers
                value: "Authorization, Content-Type, X-Requested-With, X-GRPC-Web"
              - name: Access-Control-Expose-Headers
                value: "grpc-status, grpc-message"
```

For production, use the `BackendTrafficPolicy` with specific allowed origins:

```yaml
# Advanced CORS policy (in gateway/templates/cors-policy.yaml)
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: BackendTrafficPolicy
metadata:
  name: api-cors-policy
  namespace: ameide
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: ameide-api
  cors:
    allowOrigins:
      - type: Exact
        value: "https://dev.ameide.io"
      - type: Exact
        value: "https://platform.dev.ameide.io"
      - type: Exact
        value: "https://portal.dev.ameide.io"
    allowMethods: [GET, POST, PUT, DELETE, OPTIONS, PATCH, HEAD]
    allowHeaders: [Authorization, Content-Type, X-Requested-With, X-GRPC-Web]
    exposeHeaders: [grpc-status, grpc-message, grpc-status-details-bin]
    maxAge: 86400s
    allowCredentials: true
```

**Benefits of Gateway-level CORS:**
- No need to configure CORS in each backend service
- Consistent CORS policy across all APIs
- Handles preflight OPTIONS requests at the edge
- Better performance (no backend roundtrip for preflight)

## Access Points

### Production-Like Access (via Gateway)
**Preferred (HTTPS):**
- https://dev.ameide.io - Main application
- https://auth.dev.ameide.io - Keycloak authentication
- https://platform.dev.ameide.io - Canvas platform
- https://portal.dev.ameide.io - Portal application
- https://api.dev.ameide.io - API Gateway
- https://metrics.dev.ameide.io - Grafana dashboards

**Also Works (HTTP):**
- Same hosts on port 8080

### Direct Service Access (for debugging)
- http://localhost:3000 - Main app direct
- http://localhost:3001 - Canvas direct
- http://localhost:3002 - Portal direct
- http://localhost:4000 - Keycloak direct
- http://localhost:8000 - API Gateway direct
- http://localhost:9001 - Grafana direct (if port-forwarded)

## Testing & Verification

### Smoke Tests

```bash
# 1. DNS resolution
dscacheutil -q host -a name auth.dev.ameide.io
# Should return: 127.0.0.1

# 2. Gateway routing to Keycloak
curl -sS https://auth.dev.ameide.io/realms/master/.well-known/openid-configuration | jq .issuer
# Should return: "https://auth.dev.ameide.io/realms/master"

# 3. Application health checks
curl -I https://platform.dev.ameide.io/healthz
curl -I https://dev.ameide.io/

# 4. Test with Host header (if ports not forwarded)
curl -sS -H "Host: auth.dev.ameide.io" https://127.0.0.1/realms/master/.well-known/openid-configuration -k | jq .issuer
```

### Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| `auth.dev.ameide.io` doesn't resolve | DNS not configured | Check dnsmasq and `/etc/resolver/dev.ameide.io` |
| Port 8443 connection refused | Port not forwarded | Check VS Code Ports panel, ensure forwarding active |
| Certificate warnings | Self-signed cert | Expected for local dev, click "Advanced" → "Proceed" |
| Keycloak redirects to wrong URL | KC_HOSTNAME misconfigured | Verify KC_HOSTNAME and KC_HOSTNAME_PORT envs |
| Gateway not responding | Service not exposed | Check `kubectl -n envoy-gateway-system get svc` |
| VPN breaks resolution | `.test` intercepted | Disconnect VPN or use IP-based nip.io domains |

## Optional Enhancements

### Pretty URLs (ports 80/443)

If you want clean URLs without port numbers:

#### Option 1: Caddy Reverse Proxy
```bash
# Install Caddy
brew install caddy

# Caddyfile
cat > Caddyfile <<EOF
*.dev.ameide.io {
  tls /path/to/ameide-test.crt /path/to/ameide-test.key
  reverse_proxy localhost:443
}
EOF

# Run with sudo (port 443)
sudo caddy run
```

#### Option 2: macOS pf redirect
```bash
# /etc/pf.anchors/ameide
rdr pass on lo0 inet proto tcp from any to 127.0.0.1 port 80 -> 127.0.0.1 port 8080
rdr pass on lo0 inet proto tcp from any to 127.0.0.1 port 443 -> 127.0.0.1 port 8443

# Enable
sudo pfctl -ef /etc/pf.anchors/ameide
```

### Makefile Automation

```makefile
# Makefile
.PHONY: setup up down trust-certs doctor clean

# Version pinning for stability
GATEWAY_API_VERSION := v1.2.1
ENVOY_GATEWAY_VERSION := v1.2.2

setup: ## One-time setup (DNS, certs)
	@echo "Setting up DNS and certificates..."
	@./scripts/setup-dns.sh
	@./scripts/setup-certs.sh

up: ## Start cluster and platform
	@echo "Creating k3d cluster..."
	@k3d cluster create ameide --config k3d-config.yaml
	
	@echo "Installing Gateway API CRDs..."
	@kubectl get gatewayclasses.gateway.networking.k8s.io >/dev/null 2>&1 || \
		kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/$(GATEWAY_API_VERSION)/standard-install.yaml
	
	@echo "Installing Envoy Gateway..."
	@kubectl get ns envoy-gateway-system >/dev/null 2>&1 || \
		kubectl apply -f https://github.com/envoyproxy/gateway/releases/download/$(ENVOY_GATEWAY_VERSION)/install.yaml
	
	@echo "Waiting for Envoy Gateway..."
	@kubectl wait --for=condition=Ready -n envoy-gateway-system deployment/envoy-gateway --timeout=60s
	
	@echo "Creating namespace with labels..."
	@kubectl create ns ameide --dry-run=client -o yaml | kubectl apply -f -
	@kubectl label ns ameide gateway-access=allowed --overwrite
	
	@echo "Installing platform..."
	@helm install platform ./infra/kubernetes/charts/platform

down: ## Stop cluster
	@k3d cluster delete ameide

trust-certs: ## Check cert-manager certificates
	@kubectl get certificate -n ameide
	@kubectl get secret ameide-wildcard-tls -n ameide

doctor: ## Check prerequisites
	@./scripts/doctor.sh
	@echo "Checking Gateway API..."
	@kubectl get gatewayclass 2>/dev/null || echo "⚠️  Gateway API not installed"
	@kubectl get crd | grep gateway.networking.k8s.io | wc -l | xargs -I {} test {} -gt 0 || echo "⚠️  Gateway CRDs missing"

clean: ## Remove all local setup
	@k3d cluster delete ameide
	@sudo rm -f /etc/resolver/dev.ameide.io
	@sudo rm -f "$$(brew --prefix)/etc/dnsmasq.d/ameide.conf"
	@sudo brew services restart dnsmasq
```

## Rollback Plan

Complete cleanup if needed:

```bash
# Delete cluster
k3d cluster delete ameide

# Remove DNS configuration
sudo rm -f /etc/resolver/dev.ameide.io
sudo rm -f "$(brew --prefix)/etc/dnsmasq.d/ameide.conf"
sudo brew services restart dnsmasq

# Remove certificates (optional)
rm -f ameide-test.crt ameide-test.key
```

## Critical Setup Requirements

### Gateway API & Envoy Gateway
- **Gateway API CRDs**: Required for HTTPRoute, GatewayClass resources
- **Envoy Gateway**: Implements the Gateway API specification
- **Version Pinning**: Pin versions in production scripts for stability
- **Idempotent Install**: Check existence before applying to avoid errors

### Namespace Configuration
- **Label Required**: `gateway-access=allowed` for allowedRoutes selector
- **Apply with `--overwrite`**: Ensures label is always present

### DevContainer Docker Access
- **Docker Socket Mount**: Required for k3d to create clusters
- **docker-outside-of-docker**: Better than Docker-in-Docker for performance
- **Not docker-in-docker**: Avoids nested virtualization issues

## Implementation Status

### ✅ Phase 1: Research & Planning (Completed)
- Researched industry best practices
- Identified `.local` mDNS conflicts
- Determined `.test` as optimal domain choice
- Documented port 8080/8443 standard
- Added critical setup requirements (Gateway API, namespace labels, Docker socket)

### ✅ Phase 2: Core Implementation (Completed)
- ✅ Switched all references to `.test` domains
- ✅ Updated Gateway configuration for port 8080/8443
- ✅ HTTPRoutes configured for all services
- ✅ k3d cluster creation script in postCreateCommand.sh
- ✅ DevContainer port forwarding configured
- ✅ cert-manager replaces mkcert for TLS management
- ✅ Self-signed ClusterIssuer configured
- ✅ Wildcard certificate (ameide-wildcard-tls) auto-generated

### ✅ Phase 3: Enhanced Setup (Completed)
- ✅ Created manage-host-domains.sh for dnsmasq setup
- ✅ Automated TLS via cert-manager (no mkcert needed)
- ✅ Helmfile manages deployments
- ✅ Documentation updated

### 🔮 Phase 4: Polish (Future)
- [ ] Add health check dashboard
- [ ] Create troubleshooting script
- [ ] Add performance monitoring
- [ ] Document production migration path

## Files Updated (All Completed)

### Core Configuration
- ✅ `backlog/109-local-development-domain-access.md` - This document (updated)
- ✅ `gitops/ameide-gitops/sources/values/local/apps/platform/gateway.yaml` - Gateway listeners configured
- ✅ `infra/kubernetes/charts/platform/cert-manager/` - cert-manager configuration
- ✅ HTTPRoutes configured via Helmfile deployments

### DevContainer Setup
- ✅ `.devcontainer/postCreateCommand.sh` - k3d cluster creation with .test domains
- ✅ `.devcontainer/devcontainer.json` - Port forwarding configured
- ✅ `.devcontainer/manage-host-domains.sh` - dnsmasq setup for macOS host

### Documentation
- ✅ `.devcontainer/README.md` - Updated with .test domains and cert-manager
- ✅ `README.md` - Quick start section updated
- ✅ `infra/kubernetes/README.md` - Deployment guide updated

## References

- [RFC 6761](https://datatracker.ietf.org/doc/html/rfc6761) - Special-Use Domain Names
- [cert-manager](https://cert-manager.io/) - Kubernetes native certificate management
- [dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html) - DNS forwarder
- [k3d](https://k3d.io/) - k3s in Docker
- [Gateway API](https://gateway-api.sigs.k8s.io/) - Kubernetes Gateway API
- [Envoy Gateway](https://gateway.envoyproxy.io/) - Gateway API implementation

---

## 111-helm-tilt-north-star.md (old)

gotcha—zooming out. here’s the **big picture** so this stays boring and reliable.

# North star

* **Tilt = local dev**: fast inner loop. builds, rewrites image refs, applies **dev manifests**, live-updates. no releases.
* **Helm/Helmfile = test/prod**: deploy **prebuilt** artifacts, **digest-pinned**, with rollback. no builds.

# End-to-end flow

```
DEV (laptop)                             CI/CD                                   CLUSTERS
-------------                            ----------------------------------      -------------------------
Tilt (docker_build/custom_build)  →      Build & push images (Buildx/Bazel) →    Staging/Prod via Helmfile
k8s_yaml/kustomize apply          →      Resolve digest: crane digest       →    Deploy digest-pinned charts
Logs/ports/live-update            →      Store + pass digest/env            →    --wait --atomic, rollback
```

# Contracts & conventions

**Images**

* Base chart values: `image.graph: <service name>` (no registry, no tag).
* **Helmfile (non-local)** must set:

  * `image.graph: <full-registry>/<service>`
  * `image.digest: sha256:...` (prefer digest; tag optional as a label).
* **Tilt (local)** deploys **raw manifests** with `image: <service name>`; Tilt rewrites to `docker.io/ameide/<service>:tilt-…`.

**Charts (release-only)**

* Template logic (digest-first, strict):

  ```yaml
  {{- $repo := required "image.graph is required" .Values.image.graph }}
  {{- if .Values.image.digest }}
  image: "{{ $repo }}@{{ .Values.image.digest }}"
  {{- else }}
  image: "{{ $repo }}:{{ required "image.tag or image.digest is required" .Values.image.tag }}"
  {{- end }}
  ```
* This is fine now because **Tilt no longer calls Helm**.

**Dev manifests**

* Keep simple YAML (or a kustomize overlay) per app:

  ```yaml
  containers:
    - name: www-ameide
      image: www-ameide   # no tag, no registry
      imagePullPolicy: IfNotPresent
  ```
* Tiltfile:

  ```python
  default_registry('docker.io/ameide')
  docker_build('www-ameide', 'services/www-ameide')
  k8s_yaml('manifests/dev/www-ameide.yaml')
  k8s_resource('www-ameide', port_forwards=['3000:3000'])
  ```

# CI/CD gates (make failures loud)

1. **Build & push** → `docker buildx build --push -t $IMAGE:$GIT_SHA …`
2. **Resolve digest** → `DIGEST=$(crane digest $IMAGE:$GIT_SHA)`
3. **Policy checks**

   * refuse `:latest`
   * ensure `DIGEST` is non-empty & pullable
4. **Deploy** → `helmfile -e staging apply --wait --atomic`, passing:

   * `WWW_AMEIDE_REPO=registry.example.com/www-ameide`
   * `WWW_AMEIDE_DIGEST=$DIGEST`
5. **Post-deploy** smoke: readiness, 200 on health, error budget alerts.

# What you’ve already fixed (nice!)

* Base values are clean (service names).
* Local overrides don’t inject registry.
* Helmfile skips app releases for `local`.
* Docs updated with the final pattern.
* Chart templates enforce “digest/tag required” (now safe since Tilt ≠ Helm).

# Remaining tidy-ups (quick wins)

* **Remove Helm from Tiltfile** for app services (use `k8s_yaml`/kustomize only).
* **CI env wiring**: expose `<APP>_DIGEST` to Helmfile `set:` or env values.
* **Rollback story**: `helm rollback` + promote previous digest from CI history.
* **Guardrails**: a pre-commit or CI check that rejects tags in base values and registry prefixes sneaking into them.

# Mental model (one-liners)

* *Dev speed needs mutability → Tilt owns it.*
* *Release safety needs immutability → Helmfile pins by digest.*
* *Charts don’t decide tags/registries; tools/environments do.*

If you want, paste one app’s current **dev manifest** and the **helmfile release** block; I’ll sanity-check them against this blueprint and tighten anything that could still bite later.

---

## 112-ameide-developer.md (old)

Purpose

Build a two-part AI development assistant where Architect (in the Vercel app) turns user intent into a precise plan, and Coder (in a container with repo access) executes the plan via diffs and tests. Focus is on clear thinking, decisive control points, and a smooth human-in-the-loop—not tooling minutiae.

Outcomes

Fast planning → actionable plan JSON users can approve or tweak.

Safe code changes via reviewable diffs and mandatory approvals at risky steps.

Continuous visibility: everything streams (tokens, tool status, test output).

Resumable runs with deterministic checkpoints.

Artifacts-first UX: Architect outputs (stories, epics, ACs, risks, sprint goals) appear as cards in the Next.js Chatbot Artifacts UI, editable inline.

Core Principles

Plan is the contract: Architect produces a machine-readable plan that drives the run.

Interrupts at control points: Human approves/edits before irreversible actions (applying patches, pushing commits).

Stream everything: Status, partial outputs, and logs are surfaced live.

Idempotent + resumable: Each step can be retried or rolled back deterministically.

Least privilege: Coder’s environment has the minimum rights needed for the task.

Scope & Non‑Goals

In scope

Single-user flows, one repo at a time.

Planning, patch generation, application, test run, and result summarization.

Approvals for patch apply and optional push/PR creation.

Not in scope (v0)

Multi-tenant policy, project scaffolding from scratch, full IDE replacement, advanced sandboxing/VM orchestration.

Actors & Responsibilities

User: Provides intent, approves decisions, can edit the plan/patch.

Architect (Vercel): Converts intent → structured plan; emits artifacts (epics, stories, ACs, risks, sprint goal) to the Next.js Chatbot Artifacts UI; clarifies risks; proposes acceptance tests.

Coder (Container): Clones repo, generates patch per task, pauses for approval, applies patch, runs tests, streams logs.

System Overview (solution sketch)

User ──(prompt)──▶ Architect (Vercel)
  ▲                         │
  │        plan JSON        ▼
  └── approve/edit ◀─── UI streams

Architect ──▶ Coder (container)
    plan, repo, thread id
          │
          ├─ clone ▶ generate diff ▶ INTERRUPT(apply?) ▶ apply ▶ test ▶ result
          │                           ▲                     │
          └─────────── stream status/logs ──────────────────┘

Scrum/Agile organization (Architect duties)

Architect outputs are shaped to Scrum/Agile so the plan doubles as a lightweight backlog:

Product goal & Sprint goal: concise, outcome-focused.

Epics: grouped outcomes with rationale.

User stories (INVEST): As a  I want  so that .

Acceptance criteria: Gherkin-style Given/When/Then per story.

Priority & estimate: MoSCoW priority + story points (with confidence).

Dependencies & risks: per story.

Definition of Ready/Done: short checklists per story.

Sprint candidates: suggested set sized to team capacity.

Architect also performs a quick backlog refinement pass: flags ambiguity, adds/adjusts ACs, and proposes small spikes when uncertainty is high.

Architect artifacts in the Next.js Chatbot Artifacts UI

Principle: artifacts are first-class; the plan is a snapshot derived from them.

Artifact types

Plan summary (product goal, sprint goal)

Epic

User story (INVEST)

Acceptance criteria set (Gherkin)

Risk

Sprint selection / capacity

Minimal artifact shape

{ "id":"US-001", "type":"story", "title":"/health endpoint", "body":"As an SRE...", "relations":{"epic_id":"E-001"}, "ac":["Given... When... Then..."], "priority":"Must", "estimate":{"points":1,"confidence":0.8}, "selected_for_sprint": true }

Streaming (Architect → UI)

{ "type":"artifact", "op":"add|update|remove", "item": { /* artifact */ } }

User actions (thinking-focused)

Approve / edit stories and ACs

Prioritize, mark sprint candidates, adjust capacity

Send to Coder → materialize Plan v2 snapshot and start the run

Source of truth

The artifacts store (per thread_id) in the Next.js app is canonical.

Plan v2 and the Coder run payload are deterministic snapshots of that store at send-time.

Data Contracts (thin but decisive)

Plan v2 (Agile/Scrum Plan; Architect → UI/Container)

{
  "product_goal": "Expose health and sum endpoints with tests",
  "sprint": { "goal": "Operational endpoints in place", "capacity_points": 5, "stories_selected": ["US-001","US-002"] },
  "epics": [
    { "id": "E-001", "title": "Observability & Health", "why": "Detect outages early" }
  ],
  "backlog": [
    {
      "id": "US-001",
      "epic_id": "E-001",
      "story": "As an SRE, I want a /health endpoint so that monitoring can detect outages.",
      "acceptance_criteria": [
        "Given the service is running, When I GET /health, Then I receive 200 and body {\"ok\": true}"
      ],
      "priority": "Must",
      "estimate": { "points": 1, "confidence": 0.8 },
      "dependencies": [],
      "risk": "None",
      "definition_of_ready": ["Story unambiguous", "ACs in Gherkin", "Owner set"],
      "definition_of_done": ["Tests pass", "Lint/format", "Endpoint documented"],
      "tasks": [
        { "id": "T1", "desc": "Implement /health in app/main.py", "tests": ["tests/test_health.py"] }
      ],
      "sprint_candidate": true
    },
    {
      "id": "US-002",
      "epic_id": "E-001",
      "story": "As a developer, I want a /sum endpoint so that clients can compute a total on server.",
      "acceptance_criteria": [
        "Given numbers [1,2,3], When I POST /sum, Then response is 6 with 200"
      ],
      "priority": "Should",
      "estimate": { "points": 2, "confidence": 0.7 },
      "dependencies": ["US-001"],
      "risk": "Input validation edge cases",
      "definition_of_done": ["Happy + error paths tested"],
      "tasks": [
        { "id": "T2", "desc": "Implement /sum in app/main.py", "tests": ["tests/test_sum.py"] }
      ],
      "sprint_candidate": true
    }
  ]
}

Notes

Architect validates INVEST and AC quality; if missing, it proposes ACs.

Coder consumes stories_selected → flattens to tasks in order.

Each task maps to exactly one proposed unified diff + minimal tests.

Start Coder Run (Vercel → Container)

{
  "thread_id": "uuid-123",
  "repo": {"git_url":"https://github.com/org/repo","ref":"feature-branch"},
  "plan": { /* Plan v2 snapshot from Artifacts UI */ }
}

Streamed Events (Container → Vercel UI)

{ "type":"status", "stage":"checkout|codegen|apply|test", "message":"..." }
{ "type":"patch",  "task":"T1", "unified_diff":"--- a/..." }
{ "type":"interrupt", "interrupt_id":"abc", "payload": { "kind":"apply_patch", "diff_preview":"..." } }
{ "type":"result", "summary":"ok|failed", "details": { "tests_passed": true, "stats": {"passed": 8, "failed": 0} } }

Resume (UI → Container)

{ "thread_id":"uuid-123", "value":"approve|reject|edit", "edit": {"unified_diff":"..."} }

State & Control Flow (LangGraph in the Coder)

Nodes (linear for v0): checkout → codegen → interrupt(apply) → apply → test → summarize (for selected sprint stories only)

checkout: clone/checkout ref; detect stack; stream repo summary.

codegen: for each selected story’s tasks, emit proposed unified diff aligned to the story’s ACs.

interrupt(apply): block and surface the diff preview per story/task. Options: approve, reject, or edit.

apply: git apply with safety flags; on conflict, surface conflict hunks as an interrupt.

test: run tests; stream lines and a final summary; map pass/fail back to each story’s ACs.

summarize: report sprint goal coverage: which ACs met, which stories remain.

Retry policy

Any node can be retried with the same inputs (idempotent). On repeated failure, escalate to user with diagnosis.

Human-in-the-Loop
 (where thinking matters)

Before code is touched: User sees the plan and can edit tasks/acceptance.

Before apply: Mandatory approval; user can inline-edit the diff.

On failures: The system proposes one minimal follow-up patch; user decides to continue or stop.

Failure Modes & Responses

Patch doesn’t apply: Show conflict hunks; offer auto-rebase attempt or smaller patch split.

Tests fail: Surface failing tests + traceback; propose targeted fix with a new diff; pause.

Ambiguous repo layout: Ask a single clarifying question or fall back to scaffolding minimal files (behind approval).

Security & Trust (essentials only)

Scope: Coder has repo-scoped token and ephemeral workspace; no access to user secrets beyond what’s needed.

Controls: Only whitelisted commands (git, test runner, formatter). No network egress by default (configurable).

Audit: Persist plan, diffs, approvals, and test summaries per thread_id.

Performance Targets (v0)

Planning: < 5s perceived (stream tokens early).

First patch: < 20s for small edits (< 10 files).

Test window: soft cap 90s; stream partial output.

Milestones

MVP: Plan JSON → single patch → approve → apply → run tests → summarize.

v1: Multi-task sequencing, partial applies, better error recovery.

v2: Repo-aware context (indexing), AST edits fallback, PR creation.

Open Questions

Do we enforce a single mandatory apply-approval, or also require approval to run tests (CI load)?

How do we represent cross-file refactors in the plan (task graph vs. flat list)?

Where should edited diffs be stored (temp artifact vs. branch commit)?



---

## 113-0-overview.md (old)

# Artifact-Centric Graph Architecture - Overview

## Executive Summary

Unified graph model where all platform content becomes **Artifacts** with **typed relationships**. Enables cross-model queries, impact analysis, and natural navigation across BPMN processes, ArchiMate models, documents, and custom content types.

**Key Innovation**: Everything is a node in a graph-scoped graph, enabling queries like "What processes implement this capability?" or "Show all artifacts impacted by this change."

## Goals

- **Universal Graph Model**: All content as graph nodes with typed relationships
- **Semantic Relationships**: Rich, queryable relationships between artifacts
- **Flexible Composition**: Artifacts can contain other artifacts (hierarchical/network)
- **Cross-Model Traceability**: Natural links between BPMN, ArchiMate, documents, etc.

## Benefits

- **Graph Queries**: "Find all BPMN processes that implement this ArchiMate capability"
- **Impact Analysis**: "What artifacts depend on this component?"
- **Natural Navigation**: Browse from strategy → capability → process → implementation
- **Semantic Search**: Find by meaning, not just keywords
- **Artifact Reuse**: Same artifact can appear in multiple contexts
- **AI Context**: Graph traversal provides rich context for AI assistants

## Why Graph vs Document-Centric

| Aspect | Document-Centric | Artifact Graph |
|--------|-----------------|----------------|
| Structure | Hierarchical only | Network + hierarchical |
| Relationships | Parent-child | Any typed relationship |
| Cross-references | Manual links | Native graph edges |
| Impact analysis | Limited to tree | Full graph traversal |
| Model integration | Separate silos | Unified graph |
| Query capability | SQL/document queries | Graph pattern matching |
| Traceability | Manual maintenance | Automatic via edges |
| AI context | Limited to document | Rich neighborhood context |

## High-Level Architecture

```
Event Store (Source of Truth)
    ↓
Transactional Outbox
    ↓
Debezium CDC → Kafka
    ↓
GraphProjector Service
    ↓
Neo4j (Read-Only Projection)
    ↓
Query APIs (gRPC/GraphQL/REST)
```

## Key Design Principles

1. **Event Store owns truth** - Graph is disposable projection
2. **Repository boundary** - Security enforced at graph level
3. **Dual edge patterns** - Direct edges for simple, reified nodes for rich relations
4. **Kill switch ready** - Feature flag + rebuild command for safe rollback
5. **No graph writes** - Only projector writes to graph, all changes via commands

## Document Structure

- [Architecture & ADRs](113-1-architecture.md) - Core architecture decisions
- [Domain Model](113-2-domain-model.md) - Artifact kinds and relationships
- [Technical Implementation](113-3-technical-implementation.md) - Kafka, Neo4j, projections
- [API Design](113-4-api-design.md) - Query patterns and security
- [UI/UX Design](113-5-ui-ux-design.md) - User interface and workflows
- [Observability](113-6-observability.md) - Monitoring and operations
- [Implementation Guide](113-7-implementation-guide.md) - Step-by-step instructions
- [Appendices](113-8-appendices.md) - Integration patterns and examples
- [Governance & Operations](113-9-governance-operations.md) - Kind registry, compliance, lifecycle
- [Human-in-the-Loop ChangeSet System](113-10-changeset-hitl.md) - Review and approval workflows
---

## 113-1-architecture.md (old)

# Artifact-Centric Graph Architecture - Core Architecture

## Architecture Decision Records

### ADR-001: Event Store as Single Source of Truth
- **Status**: Accepted
- **Decision**: Event Store owns all truth; graph is read-only projection
- **Consequences**: Graph can be dropped and rebuilt anytime from events

### ADR-002: Repository as Security Boundary
- **Status**: Accepted
- **Decision**: Tenant → Repository is the only security boundary
- **Consequences**: All queries must start from this root; no cross-graph edges

### ADR-003: Kafka Partitioning Strategy
- **Status**: Accepted
- **Decision**: Partition by graph_id; one consumer per partition
- **Consequences**: Per-graph ordering guaranteed; horizontal scaling by graph

### ADR-004: Relationship Uniqueness Strategy
- **Status**: Accepted
- **Decision**: Rely on Kafka ordering + MERGE with edge_key
- **Consequences**: Simpler implementation; depends on proper Kafka configuration

### ADR-005: Typed Edges Behind Feature Flag
- **Status**: Accepted
- **Decision**: Whitelist specific relationships; control via ENABLE_TYPED_EDGES
- **Consequences**: Performance gains for hot paths; fallback to RELATES if disabled

### ADR-006: Outbox Pattern Over Direct Publishing
- **Status**: Accepted
- **Decision**: Use transactional outbox + Debezium CDC
- **Consequences**: Guaranteed delivery; no dual-write problem; slight latency increase

## Core Architecture

### Single Source of Truth
- Event Store is the only truth
- Graph is a read-only projection (can be dropped and rebuilt)
- No graph write APIs - only the projector writes to graph
- Every node/edge carries `origin_stream`, `origin_seq`, `lastEventSeq`

### Security Model
- Tenant owns Repositories (no separate Workspace concept)
- Repository is the security boundary
- All queries anchor at `(Tenant)->(Repository)`
- Service layer validates JWT → tenant/repo claims

### Consistency Strategy
- Transactional Outbox pattern for dual-write safety
- Idempotent projections with `lastEventSeq` guards
- Eventually consistent with SLO: p95 lag ≤ 5s
- **Effectively-once semantics**: At-least-once delivery with idempotent processing
- Kafka offsets committed AFTER successful Neo4j transaction

### Rollback Guarantees
- Feature flag: `GRAPH_READS_ENABLED` (kill switch)
- Rebuild command: `graphctl rebuild --graph=<id>`
- Dark reads: Shadow execution in staging
- Graph queries return 501 when disabled

## Graph Model

### Node Structure
```cypher
:Tenant {id, name}
:Repository {id, name, visibility, createdAt}
:Artifact {id, kind, name, lastEventSeq, origin_stream, origin_seq, deleted, deletedAt, deletedBy}
:Relation {id, kind, context, valid_from, valid_to}  // Reified when needed
```

### Temporal Semantics for Nodes

#### Soft Delete Pattern
```cypher
// Soft delete marks nodes as deleted without removal
(:Artifact {
  id: "artifact-123",
  deleted: true,              // Soft delete flag
  deletedAt: datetime(),       // When deleted
  deletedBy: "user-456",       // Who deleted
  deletedReason: "Obsolete",   // Optional reason
  lastEventSeq: 42            // Points to delete event
})

// Queries must filter deleted nodes
MATCH (a:Artifact)
WHERE coalesce(a.deleted, false) = false
RETURN a
```

#### Temporal Validity Pattern
```cypher
// Nodes can have temporal validity windows
(:Artifact {
  id: "artifact-789",
  validFrom: datetime("2024-01-01T00:00:00Z"),
  validTo: datetime("2024-12-31T23:59:59Z"),
  supersededBy: "artifact-890",  // Link to replacement
  version: 2
})

// Query for artifacts valid at specific time
MATCH (a:Artifact)
WHERE datetime($asOf) >= a.validFrom 
  AND (a.validTo IS NULL OR datetime($asOf) <= a.validTo)
  AND coalesce(a.deleted, false) = false
RETURN a
```

#### Version Chain Pattern
```cypher
// Version chain using edges
(:Artifact {id: "v1"})-[:NEXT_VERSION]->(:Artifact {id: "v2"})-[:NEXT_VERSION]->(:Artifact {id: "v3"})

// Find latest version
MATCH (a:Artifact {id: $rootId})
MATCH path = (a)-[:NEXT_VERSION*0..]->(:Artifact)
WHERE NOT (last(nodes(path)))-[:NEXT_VERSION]->()
RETURN last(nodes(path)) as latestVersion

// Find version at point in time
MATCH (a:Artifact {id: $rootId})
MATCH path = (a)-[:NEXT_VERSION*0..]->(:Artifact)
WHERE ALL(n IN nodes(path) WHERE n.createdAt <= datetime($asOf))
RETURN last(nodes(path)) as versionAsOf
```

### Edge Patterns

#### Simple Relations
Direct edges with unique key per from/kind/to:
```cypher
(Artifact)-[:RELATES {kind, edge_key}]->(Artifact)
```
- Edge key: `sha256(tenant|repo|from|kind|to)`
- Properties: kind, lastEventSeq, createdAt, createdBy, properties

#### Rich Relations
Reified nodes for multiple, temporal, weighted relations:
```cypher
(Artifact)-[:OUT]->(Relation)-[:IN]->(Artifact)
```
- Multiple relations of same kind with different contexts
- Temporal: valid_from/valid_to for "as of" queries
- Contextual: Different contexts for same from/to/kind triple

### Invariants
- No cross-graph edges between artifacts
- No cross-tenant edges (implied by above)
- All queries start at `(Tenant)->(Repository)`

## Edge Key Canonicalization

```python
def compute_edge_key(tenant_id: str, repo_id: str, from_id: str, kind: str, to_id: str) -> str:
    """Compute canonical edge key for relationship uniqueness."""
    # Normalize all inputs
    tenant_id = tenant_id.strip().lower()
    repo_id = repo_id.strip().lower()
    from_id = from_id.strip().lower()
    kind = kind.strip().lower()  # Always use canonical string
    to_id = to_id.strip().lower()
    
    # Concatenate with pipe separator
    canonical = f"{tenant_id}|{repo_id}|{from_id}|{kind}|{to_id}"
    
    # Compute SHA256
    return hashlib.sha256(canonical.encode('utf-8')).hexdigest()
```

**Requirements:**
- Always lowercase all inputs before hashing
- Always strip whitespace from all inputs
- Kind must be canonical string (e.g., `"bpmn::sequenceflow"` - lowercase)
- Separator is pipe (`|`) character
- Hash is lowercase hex string

## Relationship Uniqueness

Neo4j 5+ doesn't enforce uniqueness on relationships. Our approach:

### Kafka Partitioning + MERGE (Chosen)
- Kafka partitions by `graph_id` ensure sequential processing
- Use `MERGE` with computed `edge_key`
- Simple, leverages existing infrastructure

```cypher
MERGE (from)-[r:RELATES {edge_key: $edge_key}]->(to)
  ON CREATE SET r.kind = $kind, r.properties = $properties
```

## Portability Rules

### Write Path Contract
- **NO APOC procedures** in write operations - plain Cypher only
- **Parameterized queries only** - no string interpolation
- **Reserved properties** that must not be used by applications:
  - `id`, `kind`, `lastEventSeq`
  - `origin_stream`, `origin_seq`
  - `createdAt`, `updatedAt`, `deleted`
  - `createdBy`, `updatedBy`, `deletedBy`

### Provider Compatibility
- Neo4j-specific optimizations (typed edges) must be feature-flagged
- All queries must work with generic RELATES edges
- No vendor-specific functions in core projections

## Performance Strategy

### Typed Edges for Hot Paths
Whitelist of typed edges for 10-50x performance improvement:

```python
TYPED_EDGE_MAPPING = {
    "structural::haspart":      "HAS_PART",
    "doc::haschapter":          "DOC_HAS_CHAPTER",
    "doc::hassection":          "DOC_HAS_SECTION",
    "bpmn::sequenceflow":       "BPMN_SEQUENCE_FLOW",
    "bpmn::messageflow":        "BPMN_MESSAGE_FLOW",
    "archimate::realizes":      "ARCHIMATE_REALIZES",
    "archimate::serves":        "ARCHIMATE_SERVES",
}
```

When `ENABLE_TYPED_EDGES=true` and kind is whitelisted, create both:
1. Generic `RELATES` edge with full properties
2. Typed edge for performance-critical queries
---

## 113-10-changeset-hitl.md (old)

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
---

## 113-2-domain-model.md (old)

# Artifact-Centric Graph Architecture - Domain Model

## Core Interfaces

```typescript
interface Artifact {
  id: string;                    // UUID
  kind: string;                   // e.g., "common::Document", "bpmn::Process"
  name: string;
  description?: string;
  createdAt: Date;
  createdBy: string;              // User ID
  lastEventSeq: number;           // For causality tracking
  properties: Record<string, any>; // Kind-specific properties
}

interface ArtifactRelation {
  id: string;
  fromId: string;                  // Source artifact
  toId: string;                    // Target artifact
  kind: string;                    // e.g., "structural::hasPart", "bpmn::triggers"
  properties: Record<string, any>; // Relation-specific properties
  createdAt: Date;
  createdBy: string;
}
```

## Kind Taxonomy

### Common Core Kinds (`common::`)

```yaml
# Basic structural artifacts
- kind: common::folder
  description: Container for organizing artifacts
  properties: [name, description, order]

- kind: common::document
  description: Text-based content (markdown, rich text)
  properties: [title, content, format, version]

- kind: common::image
  description: Visual assets (PNG, SVG, etc)
  properties: [title, url, mimeType, dimensions]

- kind: common::table
  description: Structured tabular data
  properties: [name, columns[], rows[]]

- kind: common::codeblock
  description: Source code snippets
  properties: [language, content, filename]

# Annotations and metadata
- kind: common::note
  description: Annotation on any artifact
  properties: [content, author, timestamp]

- kind: common::tag
  description: Categorization label
  properties: [name, color, category]

- kind: common::comment
  description: Discussion thread
  properties: [content, author, timestamp, resolved]
```

### Structural Relations (`structural::`)

```yaml
# Composition (parent owns child lifecycle)
- kind: structural::haspart
  description: Composite aggregation (UML-style)
  symmetric: false
  properties: [order, role]

# Dependency
- kind: structural::dependson
  description: Requires for completeness
  symmetric: false
  properties: [type, strength]

# Reference (loose coupling)
- kind: structural::references
  description: Mentions or links to
  symmetric: false
  properties: [context, anchor]

# Derivation
- kind: structural::derivedfrom
  description: Generated or transformed from
  symmetric: false
  properties: [method, timestamp]
```

### Document Structure Kinds (`doc::`)

```yaml
# Document hierarchy
- kind: doc::book
  description: Top-level document container
  properties: [title, author, isbn, publishedDate]
  children: [doc::chapter, doc::frontmatter, doc::backmatter]

- kind: doc::chapter
  description: Major document division
  properties: [number, title, abstract]
  children: [doc::section, doc::figure, doc::table]

- kind: doc::section
  description: Sub-division of chapter
  properties: [number, title, level]
  children: [doc::subsection, doc::paragraph, doc::list]

- kind: doc::paragraph
  description: Block of text
  properties: [content, style]
  children: [doc::sentence, doc::inlineimage, doc::footnote]

# Document elements
- kind: doc::figure
  description: Embedded illustration
  properties: [caption, source, number]
  relations: [references -> common::image]

- kind: doc::table
  description: Embedded data table
  properties: [caption, number]
  relations: [references -> common::table]

# Specialized document types
- kind: doc::spreadsheet
  description: Excel-like document
  children: [doc::sheet]

- kind: doc::presentation
  description: Slide deck
  children: [doc::slide]
```

### View/Diagram Kinds (`view::`)

```yaml
# Canvas and diagrams
- kind: view::canvas
  description: 2D drawing surface
  properties: [width, height, viewBox, zoom]
  children: [view::shape, view::connector]

- kind: view::diagram
  description: Specialized canvas with semantics
  properties: [type, notation, version]
  subtypes: [view::bpmndiagram, view::archimatediagram, view::umldiagram]

# Visual elements
- kind: view::shape
  description: Visual node/box
  properties: [x, y, width, height, style]
  relations: [represents -> artifact]

- kind: view::connector
  description: Visual edge/line
  properties: [sourceX, sourceY, targetX, targetY, points[], style]
  relations: [represents -> relation]
```

### ArchiMate Kinds (`archimate::`)

```yaml
# Strategy Layer
- kind: archimate::capability
  properties: [name, description, maturity]
  relations: [realizes -> archimate::outcome]

- kind: archimate::resource
  properties: [name, type]

- kind: archimate::courseofaction
  properties: [name, description, priority]

# Business Layer
- kind: archimate::businessactor
  properties: [name, type, external]

- kind: archimate::businessrole
  properties: [name, description]

- kind: archimate::businessprocess
  properties: [name, description]
  relations: [realizes -> archimate::businessservice]

# Application Layer
- kind: archimate::applicationcomponent
  properties: [name, version, vendor]

- kind: archimate::applicationservice
  properties: [name, description, external]

- kind: archimate::dataobject
  properties: [name, type, format]

# Technology Layer
- kind: archimate::node
  properties: [name, type, location]

- kind: archimate::systemsoftware
  properties: [name, version, vendor]

# Relations
- kind: archimate::realizes
- kind: archimate::serves
- kind: archimate::assignment
- kind: archimate::access
- kind: archimate::triggering
- kind: archimate::flow
```

### BPMN Kinds (`bpmn::`)

```yaml
# Process elements
- kind: bpmn::process
  properties: [name, executable, type]
  children: [bpmn::task, bpmn::event, bpmn::gateway]

- kind: bpmn::task
  properties: [name, type] # User, Service, Script, etc.

- kind: bpmn::event
  properties: [name, type, trigger] # Start, End, Intermediate

- kind: bpmn::gateway
  properties: [name, type] # Exclusive, Parallel, Inclusive

# Flow objects
- kind: bpmn::sequenceflow
  properties: [name, condition]
  relations: [from -> flownode, to -> flownode]

- kind: bpmn::messageflow
  properties: [name, message]
  relations: [from -> participant, to -> participant]

# Containers
- kind: bpmn::lane
  properties: [name, actor]
  parent: bpmn::pool

- kind: bpmn::pool
  properties: [name, participant]
  children: [bpmn::lane]
```

## Data Modeling Best Practices

### Promote Hot Fields for Indexing

```typescript
// ❌ BAD: Nested properties can't be indexed
{
  kind: "archimate::ApplicationComponent",
  properties: {
    status: string;      // Can't index efficiently
    approved: boolean;   // Can't query directly
    priority: number;    // Slow to filter
  }
}

// ✅ GOOD: Top-level indexed fields
{
  kind: "archimate::ApplicationComponent",
  
  // Indexed for queries
  status: "production",    // CREATE INDEX artifact_status
  approved: true,          // CREATE INDEX artifact_approved
  priority: 1,             // CREATE INDEX artifact_priority
  
  // Non-indexed metadata in properties
  properties: {
    documentation: "...",
    notes: "...",
    customFields: {}
  }
}
```

### Search Field Pattern

```typescript
// Combine searchable text for full-text index
{
  kind: "bpmn::Task",
  name: "Validate Order",
  description: "Check inventory and pricing",
  
  // Computed field for full-text search
  search: "validate order check inventory pricing validation task",
  
  // CREATE FULLTEXT INDEX artifact_search
  // FOR (a:Artifact) ON EACH [a.search]
}
```

### Binary/Large Content Storage

```typescript
// Store references, not content
{
  kind: "doc::Figure",
  name: "System Architecture",
  
  // Reference to S3/MinIO
  contentUrl: "s3://artifacts/diagrams/sys-arch-v2.png",
  mimeType: "image/png",
  sizeBytes: 245823,
  
  // Thumbnail for quick preview
  thumbnailUrl: "s3://artifacts/thumbs/sys-arch-v2-thumb.jpg"
}
```

### Message Thread Modeling

```typescript
// Message threads as artifacts with edges
{
  thread: {
    kind: "message::Thread",
    subject: "Design Review",
    participants: ["user1", "user2"],
    lastActivity: "2024-01-15T10:30:00Z"
  },
  
  messages: [
    {
      kind: "message::Post",
      content: "Please review the attached design",
      author: "user1",
      timestamp: "2024-01-15T09:00:00Z"
    }
  ],
  
  // Edges connect messages to thread
  edges: [
    "(Thread)-[:HAS_MESSAGE {order: 1}]->(Post1)",
    "(Thread)-[:HAS_MESSAGE {order: 2}]->(Post2)"
  ]
}
```

## Change Proposal Domain Model

### ChangeSet Aggregate

```typescript
interface ChangeSet {
  id: string;                      // UUID
  tenantId: string;
  graphId: string;
  status: ChangeSetStatus;
  proposerId: string;               // User or agent ID
  title: string;
  description: string;
  rationale?: string;               // Why this change
  aiGenerated?: boolean;
  aiModel?: string;
  confidenceScore?: number;          // For AI proposals
  riskSummary?: string;
  createdAt: Date;
  updatedAt: Date;
  touches: TouchedStream[];         // Streams affected
  proposedEvents: ProposedEvent[];  // Events to apply
}

enum ChangeSetStatus {
  DRAFT = 'draft',
  PENDING_REVIEW = 'pending_review',
  APPROVED = 'approved',
  REJECTED = 'rejected',
  CONFLICTED = 'conflicted',
  COMMITTED = 'committed'
}

interface TouchedStream {
  streamId: string;
  expectedVersion: number;
}

interface ProposedEvent {
  targetStreamId: string;
  expectedVersion: number;
  eventType: string;
  eventPayload: any;
  intent?: Intent;                  // For rebase capability
}

interface Intent {
  commandType: string;
  commandPayload: any;
  streamId: string;
}
```

### ChangeSet Events

```typescript
// Domain events for ChangeSet lifecycle
interface ChangeSetCreated {
  changesetId: string;
  tenantId: string;
  graphId: string;
  title: string;
  proposerId: string;
  aiGenerated: boolean;
  createdAt: Date;
}

interface ProposedEventsAdded {
  changesetId: string;
  events: ProposedEvent[];
  addedAt: Date;
}

interface ChangeSetSubmittedForReview {
  changesetId: string;
  submittedAt: Date;
  submittedBy: string;
}

interface ChangeSetApproved {
  changesetId: string;
  approvedAt: Date;
  approvedBy: string;
  comment?: string;
}

interface ChangeSetRejected {
  changesetId: string;
  rejectedAt: Date;
  rejectedBy: string;
  reason: string;
}

interface ChangeSetCommitted {
  changesetId: string;
  committedAt: Date;
  committedBy: string;
  affectedStreams: string[];
}

interface ChangeSetConflicted {
  changesetId: string;
  conflicts: StreamConflict[];
  detectedAt: Date;
}

interface ChangeSetRebased {
  changesetId: string;
  oldEvents: ProposedEvent[];
  newEvents: ProposedEvent[];
  rebasedAt: Date;
  rebasedBy: string;
}

interface StreamConflict {
  streamId: string;
  expectedVersion: number;
  actualVersion: number;
}
```

### Preview Models

```typescript
interface ChangeSetPreview {
  changesetId: string;
  diffs: ArtifactDiff[];
  graphOverlay?: GraphOverlay;
  impacts: ImpactAssessment[];
}

interface ArtifactDiff {
  streamId: string;
  artifactId: string;
  before: ArtifactState;
  after: ArtifactState;
  changes: FieldChange[];
}

interface ArtifactState {
  name: string;
  kind: string;
  status?: string;
  properties: Record<string, any>;
}

interface FieldChange {
  field: string;
  oldValue: any;
  newValue: any;
  operation: 'add' | 'update' | 'remove';
}

interface GraphOverlay {
  addedNodes: NodePreview[];
  updatedNodes: NodePreview[];
  deletedNodes: string[];
  addedEdges: EdgePreview[];
  updatedEdges: EdgePreview[];
  deletedEdges: string[];
}

interface NodePreview {
  id: string;  // MUST use "preview_" prefix for ephemeral IDs
  kind: string;
  properties: Record<string, any>;
  isEphemeral?: boolean;  // True for uncommitted preview nodes
}

interface EdgePreview {
  fromId: string;  // MUST use "preview_" prefix if ephemeral
  toId: string;    // MUST use "preview_" prefix if ephemeral
  kind: string;
  properties: Record<string, any>;
  isEphemeral?: boolean;  // True for uncommitted preview edges
}

// ID generation for previews
function generatePreviewId(): string {
  // Prevent collision with real UUIDs by using prefix
  return `preview_${uuidv4()}`;
}

function isPreviewId(id: string): boolean {
  return id.startsWith('preview_');
}

interface ImpactAssessment {
  artifactId: string;
  impactType: 'direct' | 'indirect';
  distance: number;
  description: string;
  severity: 'low' | 'medium' | 'high';
}
```
---

## 113-5-ui-ux-design-dashboard.md (old)

# Dashboard View (/dashboard)

## Context

The AMEIDE platform provides two complementary views of the same data:
- **Main page (/)**: Conversational interface for working with artifacts through threads
- **Dashboard page (/dashboard)**: Visual tile-based overview of the same artifacts and processes

Both views access the same underlying data through the `@ameide/core-proto-web` SDK - artifacts, BPMN processes, ArchiMate elements. The dashboard simply presents this data in a monitoring/overview format rather than a conversational format.

## Overview

A **flexible, tile-based dashboard** at `/dashboard` that uses the `@ameide/core-proto-web` SDK directly with mocked data. The dashboard displays the same BPMN processes, ArchiMate elements, and artifacts that users interact with conversationally on the main page, but organized as visual tiles for operational monitoring and status tracking.

## Core Concept

Each organizational role gets a tailored starting dashboard:
- **Enterprise Architect**: Architecture graph, transformation transformations, capability gaps
- **Operations Manager**: Process performance, bottlenecks, operational metrics
- **Product Owner**: Feature delivery, backlogs, customer feedback
- **Finance Controller**: Cost tracking, budget variance, ROI metrics
- **HR Manager**: Capacity, skills gaps, training transformations
- **IT Manager**: System health, incidents, technical debt

Users can then add, remove, or rearrange tiles to suit their needs.

## Design Principles

1. **Start smart, stay flexible** - Role presets that users can modify
2. **Ontology-backed** - TOGAF/ArchiMate concepts structure the data
3. **Plain language surface** - No jargon in the UI
4. **Single container** - All tiles in one unified grid
5. **Repository-centric** - Enterprise graph tile for architecture assets
6. **Visual simplicity** - Clean, scannable tiles with clear metrics

## Component Specifications

### Process Tiles

```
┌───────────────────────────────────────────┐
│ • Order to Cash   🟠 Slow                 │
│   Speed 72h  | Stuck >24h: 47            │
│   Volume: 320 | Trend: ↓ 8h q/q          │
│   [Open queue] [Flag issue]              │
└───────────────────────────────────────────┘
```

**Data points:**
- **Status dot**: 🟢 On track / 🟠 Slow / 🔴 Blocked / ⚪ Unknown
- **Speed**: Average time (e.g., `72h`)
- **Stuck**: Count over threshold (e.g., `>24h: 47`)
- **Volume**: Items processed this period
- **Trend**: Simple arrow vs last period (↑ / ↓ / ~)
- **Actions**: Open queue, Flag issue

### Enterprise Repository Tile

```
┌───────────────────────────────────────────┐
│ ENTERPRISE REPOSITORY               [⚙️]  │
│                                           │
│ 📊 Models: 47                            │
│ 🔧 Applications: 142                     │
│ 📐 Capabilities: 89                      │
│ 🏢 Business Services: 34                 │
│ 📦 Data Objects: 156                     │
│ 🔄 Updates today: 3                      │
│                                           │
│ Recent changes:                          │
│ • Payment Service v2.1 approved          │
│ • Customer Journey map updated           │
│                                           │
│ [Browse] [Search] [Add]                  │
└───────────────────────────────────────────┘
```

**Features:**
- Quick counts of architecture elements
- Recent changes feed
- Direct access to graph browser
- Search across all architecture assets

### Initiative Tiles (CHANGES THIS QUARTER)

```
┌───────────────────────────────────────────┐
│ • Shorten Payment Approval   In progress │
│   Goal: 36h avg by Oct 31                │
│   Expected effect: ~ -6h/order           │
│   Owner: Sarah   Due: Sep 12             │
│   Linked to: Order to Cash               │
│   Risk: 🟠 Medium   Progress: ███░ 65%   │
│   [Next step] [Docs] [Update]            │
└───────────────────────────────────────────┘
```

**Data points:**
- **Title**: Plain language description
- **Stage**: Planned / In progress / Done
- **Goal**: One line with number + date
- **Expected effect**: Rough impact estimate
- **Owner & due date**
- **Linked processes**: Chip tags
- **Risk & progress**: Color indicator + progress bar
- **Actions**: Next step, Docs, Update


## Visual Language

### Color Coding
- **Blue**: Running work/processes
- **Green**: Changes/transformations
- **Red**: Trouble/blocked
- **Orange**: Warning/slow
- **Grey**: Unknown/no data

### Number Styling
- **Solid numbers**: Measured/actual
- **Dotted numbers**: Estimates
- **Bold**: Above threshold
- **Regular**: Normal range

### Layout Rules
- Maximum 3 tiles per column
- Big numbers (24px+)
- Short labels (< 5 words)
- One-line sentences

## SDK Type Mapping

### Tile to Protobuf Type Mapping

- **Process tile** → `BPMNProcess` from BPMNServiceClient
  - Uses: `processId`, `name`, `isExecutable`
  - Extended with: `ProcessMetrics` (to be added)
  
- **Initiative tile** → `ArchiMateElement` where `elementType == ARCHIMATE_ELEMENT_TYPE_WORK_PACKAGE`
  - Uses: `metadata.id`, `name`, `properties` map
  - Extended with: `InitiativeProgress` (to be added)
  
- **Artifact counts tile** → `ArtifactView` from ArtifactQueryServiceClient
  - Uses: `ListArtifactsResponse` with counts by type
  
- **Capability tile** → `ArchiMateElement` where `elementType == ARCHIMATE_ELEMENT_TYPE_CAPABILITY`
  - Uses: `metadata.id`, `name`, `properties` map
  - Extended with: `CapabilityGap` (to be added)

## Data Model (SDK Types)

### Using Protobuf Types from @ameide/core-proto-web

```typescript
import {
  BPMNProcess,
  BPMNElement,
  BpmnElementType,
} from '@ameide/core-proto-web';

import {
  ArchiMateElement,
  ArchiMateElementType,
  ArchiMateLayer,
  ArchiMateRelationship,
} from '@ameide/core-proto-web';

import {
  ResourceMetadata,
  RequestContext,
} from '@ameide/core-proto-web';

// Dashboard extends with runtime metrics
interface ProcessMetrics {
  processId: string;
  throughput: number;
  averageTime: number;
  queueSize: number;
  stuckCount: number;
  trend: 'up' | 'down' | 'stable';
  status: 'healthy' | 'warning' | 'critical';
}

interface InitiativeProgress {
  workPackageId: string;
  phase: 'planning' | 'executing' | 'completed';
  progressPercentage: number;
  targetDate: Date;
  risks: Risk[];
  deliverables: string[];
}

interface CapabilityGap {
  capabilityId: string;
  currentLevel: 1 | 2 | 3 | 4 | 5;
  targetLevel: 1 | 2 | 3 | 4 | 5;
  workPackageIds: string[];
}
```

## Tile Specifications

### Process Tile (using BPMNProcess)
```typescript
interface ProcessTileSettings {
  processId: string              // BPMNProcess.processId
  metricType: 'throughput' | 'latency' | 'errors' | 'queue';
  threshold?: number;            // For status calculation
  timeWindow?: 'hour' | 'day' | 'week';
}

// Data sources:
// - BPMNProcess from BPMNServiceClient
// - ProcessMetrics (to be added to proto)
// Display: name, status indicator, metric value, trend
```

### Initiative Tile (using ArchiMateElement)
```typescript
interface InitiativeTileSettings {
  elementId: string;             // ArchiMateElement.metadata.id
  // Must be ARCHIMATE_ELEMENT_TYPE_WORK_PACKAGE
}

// Data sources:
// - ArchiMateElement from ArchiMateServiceClient
// - InitiativeProgress (to be added to proto)
// Display: name, phase, progress bar, target date
```

### Artifact Count Tile
```typescript
interface ArtifactCountTileSettings {
  artifactTypes?: string[];      // Filter by type
  showRecent?: boolean;
}

// Data sources:
// - ListArtifactsResponse from ArtifactQueryServiceClient
// Display: counts by type, recent changes
```

## Status Rules

### Process Status (calculated from metrics)
- **Healthy**: Metric value within normal range (< threshold)
- **Warning**: Metric value exceeds threshold by < 50%
- **Critical**: Metric value exceeds threshold by > 50%
- **Unknown**: No metric data available

### Initiative Status (from ArchiMate properties)
- **Planning**: Element has property `phase: 'planning'`
- **Executing**: Element has property `phase: 'executing'`
- **Completed**: Element has property `phase: 'completed'`

### Progress Calculation
- Based on custom properties in ArchiMateElement.properties map
- Will be formalized when InitiativeProgress proto is added

## Role Configurations

### Preconfigured Tile Sets

```typescript
const rolePresets = {
  'enterprise-architect': [
    { type: 'artifact-counts', position: { x: 0, y: 0 } },
    { type: 'archimate-capabilities', position: { x: 1, y: 0 } },
    { type: 'archimate-workpackages', position: { x: 2, y: 0 } },
    { type: 'bpmn-processes', position: { x: 0, y: 1 } },
    { type: 'archimate-transformations', position: { x: 1, y: 1 } },
    { type: 'metrics-summary', position: { x: 2, y: 1 } }
  ],
  'operations-manager': [
    { type: 'process-metrics', position: { x: 0, y: 0 } },
    { type: 'process-queue', position: { x: 1, y: 0 } },
    { type: 'process-alerts', position: { x: 2, y: 0 } },
    { type: 'bpmn-processes', position: { x: 0, y: 1 } },
    { type: 'artifact-counts', position: { x: 1, y: 1 } }
  ],
  'product-owner': [
    { type: 'archimate-transformations', position: { x: 0, y: 0 } },
    { type: 'metrics-velocity', position: { x: 1, y: 0 } },
    { type: 'metrics-backlog', position: { x: 2, y: 0 } },
    { type: 'bpmn-processes', position: { x: 0, y: 1 } },
    { type: 'artifact-counts', position: { x: 1, y: 1 } }
  ]
}
```

## Empty States

### No Processes
```
RUNNING TODAY
"Tell us roughly how long one order takes."
[Add number]
```

### No Initiatives
```
CHANGES THIS QUARTER
"Add one change you're starting this month."
[Add change]
```

### No Data
```
WATCHLIST
"Pick 1-3 things to watch. Type numbers now, connect data later."
[Add metric]
```

## Technical Implementation

### SDK Integration

```typescript
import {
  BPMNServiceClient,
  ArchiMateServiceClient,
  ArtifactQueryServiceClient,
} from '@ameide/core-proto-web';

// Initialize clients
const bpmnClient = new BPMNServiceClient('/api');
const archimateClient = new ArchiMateServiceClient('/api');
const artifactClient = new ArtifactQueryServiceClient('/api');

// Mock clients for development
class MockBPMNServiceClient extends BPMNServiceClient {
  async listBPMNProcesses(request: ListBPMNProcessesRequest) {
    return mockBPMNProcesses();
  }
}

class MockArchiMateServiceClient extends ArchiMateServiceClient {
  async listArchiMateElements(request: ListArchiMateElementsRequest) {
    return mockArchiMateElements();
  }
}
```

### 1. Tile API + Registry (Core Contract)

```typescript
// tiles/types.ts
import { z } from "zod";

export type Size = { w: number; h: number };
export type Pos = { x: number; y: number };

export interface TileProps<T> {
  id: string;
  settings: T;
  filters: { area?: string; time?: string; owner?: string };
  mode: "view" | "arrange";
}

export interface TileDefinition<TSettings = unknown> {
  type: string;                           // "process-status", "transformation-card", etc.
  title: string;                          // human label in the catalog
  version: number;                        // for migrations
  settingsSchema: z.ZodSchema<TSettings>; // validates config at runtime
  defaultSize: Size;
  minSize?: Size;
  Component: (p: TileProps<TSettings>) => JSX.Element;
  migrate?: (oldSettings: any, fromVersion: number) => TSettings; // forward compatibility
}

// tiles/registry.ts
export const registry: Record<string, TileDefinition<any>> = {};
export const register = <T>(def: TileDefinition<T>) => (registry[def.type] = def);
```

### 2. Initial Tile Catalog

**BPMN Tiles:**
- `bpmn-process-status` - Shows BPMNProcess with metrics
- `bpmn-process-list` - List of processes from BPMNServiceClient
- `process-metrics` - ProcessMetrics visualization (when proto extended)

**ArchiMate Tiles:**
- `archimate-workpackage` - WorkPackage elements (transformations)
- `archimate-capability` - Capability elements with maturity
- `archimate-elements-list` - List of ArchiMate elements by type

**Artifact Tiles:**
- `artifact-counts` - Counts from ArtifactQueryServiceClient
- `artifact-recent` - Recent artifact changes

**Metric Tiles:**
- `metric-value` - Single KPI display
- `metric-trend` - Trend chart

### 3. Role Presets

```typescript
// presets/roles.ts
export const rolePresets = {
  "enterprise-architect": [
    { 
      type: "artifact-counts", 
      pos: { x:0, y:0 }, size: { w:4, h:3 },
      settings: {}
    },
    { 
      type: "archimate-capability", 
      pos: { x:4, y:0 }, size: { w:4, h:3 },
      settings: { elementType: ArchiMateElementType.ARCHIMATE_ELEMENT_TYPE_CAPABILITY }
    },
    { 
      type: "archimate-workpackage", 
      pos: { x:8, y:0 }, size: { w:4, h:3 },
      settings: { elementType: ArchiMateElementType.ARCHIMATE_ELEMENT_TYPE_WORK_PACKAGE }
    },
    { 
      type: "bpmn-process-list", 
      pos: { x:0, y:3 }, size: { w:6, h:3 },
      settings: {}
    },
  ],
  "operations-manager": [
    { 
      type: "process-metrics", 
      pos: { x:0, y:0 }, size: { w:4, h:3 },
      settings: { metricType: 'queue' }
    },
    { 
      type: "bpmn-process-status", 
      pos: { x:4, y:0 }, size: { w:4, h:3 },
      settings: { metricType: 'throughput', timeWindow: 'day' }
    },
    { 
      type: "metric-value", 
      pos: { x:8, y:0 }, size: { w:4, h:3 },
      settings: { metricId: 'sla-performance' }
    },
  ],
} as const;
```

### 4. Board Configuration Schema

```typescript
// board/schema.ts
export const TileInstance = z.object({
  id: z.string(),
  type: z.string(),
  pos: z.object({ x: z.number(), y: z.number() }),
  size: z.object({ w: z.number(), h: z.number() }),
  settings: z.record(z.any()).default({}),
});
export type TileInstance = z.infer<typeof TileInstance>;

export const Board = z.object({
  id: z.string(),
  title: z.string(),
  filters: z.object({ 
    area: z.string().optional(), 
    time: z.string().optional(), 
    owner: z.string().optional() 
  }),
  tiles: z.array(TileInstance),
  version: z.number().default(1),
});
export type Board = z.infer<typeof Board>;
```

### 5. View vs Arrange Modes

**View Mode:**
- Tiles are interactive (click to drill down)
- Read-only dashboard experience
- Opens drawers/modals for details

**Arrange Mode:**
- Drag & drop tiles with react-grid-layout
- Resize tiles within constraints
- Add/remove from tile catalog
- Edit tile settings in right panel
- Maximum 6-9 tiles visible for clarity

## Plain Language Surface

### UI Labels Stay Simple
- Tile titles: "Process Performance", "Active Changes", "Repository"
- Field labels: "Speed", "Stuck", "Owner", "Progress"
- Actions: "Open", "Flag", "Next Step"

### Ontology in Context
- **Surface**: Shows "Payment Process - 72h average"
- **Context drawer**: Shows full ontology:
  ```
  BusinessProcess: payment-process-001
  Relations:
    - serves → BusinessService: order-fulfillment
    - accesses → DataObject: invoice
    - realizes → Capability: payment-processing (Level 2)
  Requirement: SLA-001 (target: 24h)
  Plateau: Baseline
  ```

### Export/Import
- Always uses ontology IDs and ArchiMate relation names
- Can round-trip between tools that understand ArchiMate

## Implementation Checklist

### Phase 1: SDK Setup & Mock Data
- [ ] Install `@ameide/core-proto-web` package
- [ ] Create mock SDK clients for BPMN and ArchiMate
- [ ] Implement mock data generators matching protobuf types
- [ ] Setup TanStack Query with SDK clients
- [ ] Create dashboard layout with react-grid-layout

### Phase 2: Core Tiles
- [ ] Implement `process-status` tile using BPMNProcess type
- [ ] Implement `transformation-card` tile using ArchiMateElement (WorkPackage)
- [ ] Implement `graph` tile using ArtifactQueryService
- [ ] Create tile registry and plugin system
- [ ] Add View/Arrange mode toggle

### Phase 3: Real SDK Integration
- [ ] Connect to actual gRPC-Web endpoints
- [ ] Implement authentication with bearer tokens
- [ ] Handle streaming updates via subscriptions
- [ ] Add error handling and retry logic

### Phase 4: Proto Extensions
- [ ] Define dashboard-specific proto messages
- [ ] Add ProcessMetrics message type
- [ ] Add WorkPackageProgress message type
- [ ] Regenerate TypeScript SDK with extensions

## Mockup - Enterprise Architect View

```
┌────────────────────────┐ ┌────────────────────────┐ ┌────────────────────────┐
│ ARTIFACT COUNTS        │ │ CAPABILITIES           │ │ WORK PACKAGES         │
│                        │ │                        │ │                        │
│ 📊 BPMN Models: 47    │ │ Payment Processing     │ │ Payment Automation    │
│ 🔧 ArchiMate: 142     │ │ Customer Service       │ │ ███████░░░ 65%        │
│ 📐 Artifacts: 189     │ │ Data Analytics         │ │                        │
│ 🔄 Updates today: 3   │ │                        │ │ API Gateway           │
│                        │ │ [View all]             │ │ ████░░░░░░ 30%        │
│ [Browse]              │ │                        │ │                        │
└────────────────────────┘ └────────────────────────┘ │ [All transformations]      │
                                                      └────────────────────────┘

┌──────────────────────────────────────────────────┐ ┌────────────────────────┐
│ BPMN PROCESSES                                   │ │ METRICS SUMMARY        │
│                                                   │ │                        │
│ Order to Cash         [View] [Edit]              │ │ Throughput: 1.2k/h     │
│ Quote to Order        [View] [Edit]              │ │ Avg Time: 72h          │
│ Procure to Pay        [View] [Edit]              │ │ Queue Size: 234        │
│                                                   │ │                        │
│ [View all processes]                             │ │ [Configure]            │
└──────────────────────────────────────────────────┘ └────────────────────────┘
```

## Mockup - Operations Manager View

```
┌────────────────────────┐ ┌────────────────────────┐ ┌────────────────────────┐
│ PROCESS METRICS        │ │ BPMN PROCESS STATUS    │ │ METRIC VALUE          │
│                        │ │                        │ │                        │
│ Queue Size: 234        │ │ Order Processing       │ │ SLA Performance       │
│ Avg Wait: 3.2h         │ │ 🟠 Warning             │ │                        │
│ Throughput: 450/h      │ │ Throughput: 450/h      │ │ 87%                   │
│                        │ │                        │ │                        │
│ Trend: ↗ worsening     │ │ Payment Processing     │ │ Target: 95%           │
│                        │ │ 🟢 Healthy             │ │                        │
│ [Details]              │ │ Throughput: 1.2k/h     │ │ [Configure]           │
└────────────────────────┘ └────────────────────────┘ └────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│ BPMN PROCESSES LIST                                                        │
│                                                                             │
│ Name                   Status    Throughput    Queue    Actions            │
│ Order to Cash          🟠        450/h         234      [View] [Metrics]   │
│ Quote to Order         🟢        1.2k/h        12       [View] [Metrics]   │
│ Procure to Pay         🟢        320/h         45       [View] [Metrics]   │
│                                                                             │
│ [Refresh]                                                                   │
└────────────────────────────────────────────────────────────────────────────┘
```

## Key Architectural Decisions

### 1. Plugin Architecture
- **Tiles are self-contained plugins** with their own settings schema
- **Single registry** for all tiles - easy to add new ones
- **Runtime validation** with Zod ensures type safety

### 2. Configuration-Driven
- **Board state is just JSON** - easy to persist, share, version
- **Role presets are data** not code - can be loaded from API
- **Settings are validated** at runtime, not just compile time

### 3. Separation of Concerns
- **Tiles don't know about the grid** - just render in their space
- **Grid doesn't know tile internals** - just positions and sizes
- **Filters propagate down** - tiles react to context changes

### 4. Progressive Enhancement
- **Start with 10 core tiles** - cover 80% of use cases
- **Add domain-specific tiles** later (capability-gaps, sla-performance)
- **Custom tiles** can be registered by power users

### 5. User Experience First
- **Maximum 6-9 tiles** - prevent information overload
- **Two modes only** - View (use) vs Arrange (customize)
- **Role presets** - good defaults, full customization

## Technical Stack

- **React** - Component framework
- **react-grid-layout** - Drag & drop grid system
- **Zod** - Runtime type validation
- **Tailwind CSS** - Styling
- **Zustand** - State management for board config
- **React Query** - Data fetching and caching

## Proto Extensions Required

The following message types need to be added to the protobuf definitions:

```proto
// dashboard/v1/metrics.proto
message ProcessMetrics {
  string process_id = 1;
  double throughput = 2;
  double average_time = 3;
  int32 queue_size = 4;
  int32 stuck_count = 5;
  enum Trend {
    TREND_UNSPECIFIED = 0;
    TREND_UP = 1;
    TREND_DOWN = 2;
    TREND_STABLE = 3;
  }
  Trend trend = 6;
  google.protobuf.Timestamp timestamp = 7;
}

message InitiativeProgress {
  string element_id = 1;  // ArchiMate WorkPackage ID
  enum Phase {
    PHASE_UNSPECIFIED = 0;
    PHASE_PLANNING = 1;
    PHASE_EXECUTING = 2;
    PHASE_COMPLETED = 3;
  }
  Phase phase = 2;
  double progress_percentage = 3;
  google.protobuf.Timestamp target_date = 4;
  repeated string deliverable_ids = 5;
  repeated string risk_ids = 6;
}

message CapabilityGap {
  string capability_id = 1;  // ArchiMate Capability ID
  int32 current_level = 2;   // 1-5
  int32 target_level = 3;    // 1-5
  repeated string work_package_ids = 4;
}

// dashboard/v1/dashboard.proto
message DashboardConfig {
  string id = 1;
  string user_id = 2;
  string name = 3;
  repeated TileInstance tiles = 4;
  google.protobuf.Timestamp created_at = 5;
  google.protobuf.Timestamp updated_at = 6;
}

message TileInstance {
  string id = 1;
  string tile_type = 2;
  Position position = 3;
  Size size = 4;
  google.protobuf.Struct settings = 5;
}

message Position {
  int32 x = 1;
  int32 y = 2;
}

message Size {
  int32 width = 1;
  int32 height = 2;
}
```

## Success Metrics

1. **Type Safety**: 100% protobuf type coverage
2. **Mock Coverage**: All tiles work with mock SDK clients
3. **Performance**: Initial load < 2s with mocked data
4. **SDK Integration**: Seamless switch from mock to real clients
5. **Proto Alignment**: All data flows through generated SDK

## Next Steps

1. Install `@ameide/core-proto-web` package
2. Create mock SDK client implementations
3. Implement tile components using protobuf types
4. Setup react-grid-layout for dashboard
5. Add TanStack Query for data fetching
6. Define proto extensions for dashboard-specific needs
7. Generate updated SDK with extensions

## References

- SDK Package: `@ameide/core-proto-web` - Generated gRPC-Web client
- Protobuf Definitions: `/workspace/packages/ameide_core_proto/proto/`
- Design System: Tailwind CSS with shadcn/ui components
- Authentication: Keycloak SSO with bearer tokens for SDK
---

## 113-6-observability.md (old)

# Artifact-Centric Graph Architecture - Observability

## Distributed Tracing with Tempo

```python
from opentelemetry import trace, propagate
from opentelemetry.trace import SpanKind, Link

tracer = trace.get_tracer(__name__)

class GraphProjector:
    async def process_event(self, record: ConsumerRecord):
        # Extract trace context from Kafka headers
        ctx = propagate.extract(carrier=dict(record.headers))
        
        # Create span with link to original command span
        links = [Link(ctx)] if ctx else []
        
        with tracer.start_as_current_span(
            "graph.projection.apply",
            kind=SpanKind.CONSUMER,
            links=links,
            attributes={
                "messaging.system": "kafka",
                "messaging.destination": record.topic,
                "graph.id": record.key,
                "event.type": record.value.get("type"),
                "event.seq": record.value.get("seq")
            }
        ) as span:
            # Process projection
            with tracer.start_as_current_span("neo4j.merge", attributes={
                "db.system": "neo4j",
                "graph.query_template_id": "artifact_upsert"
            }):
                await self.apply_projection(record.key, record.value)
```

## Metrics for Grafana

```python
from prometheus_client import Counter, Histogram, Gauge

# Low-cardinality metrics with exemplars
projection_events = Counter('projection_events_total',
    'Events applied to graph', ['service', 'env'])

projection_latency = Histogram('projection_apply_latency_seconds',
    'Time to apply projection', ['query_template_id'])

projection_lag = Gauge('projection_lag_seconds', 
    'Lag between event and projection')

# Top-N graph lag without cardinality explosion
projection_lag_top = Gauge('projection_lag_top_seconds',
    'Top 10 graph lags', ['rank'])

# ChangeSet-specific metrics
changeset_created = Counter('changeset_created_total',
    'ChangeSets created', ['graph_id', 'ai_generated'])

changeset_by_status = Gauge('changeset_by_status',
    'Current changesets by status', ['graph_id', 'status'])

changeset_approval_time = Histogram('changeset_approval_seconds',
    'Time from submission to approval', ['graph_id'],
    buckets=(60, 300, 900, 1800, 3600, 7200, 14400))  # 1m, 5m, 15m, 30m, 1h, 2h, 4h

changeset_conflict_rate = Counter('changeset_conflicts_total',
    'ChangeSets with conflicts', ['graph_id', 'conflict_type'])

changeset_rebase_success = Counter('changeset_rebase_success_total',
    'Successful rebases', ['graph_id', 'auto_rebase'])

changeset_batch_size = Histogram('changeset_batch_size',
    'Number of streams in batch commit', ['graph_id'],
    buckets=(1, 2, 5, 10, 20, 50, 100))

# Example usage
async def record_metrics(graph_id: str, event: dict, duration: float):
    projection_events.labels(service='graph-projector', env='prod').inc()
    projection_latency.labels(query_template_id='artifact_upsert').observe(duration)
    
    # Update top-N lag metrics
    lag = time.time() - event['timestamp']
    await update_top_n_lag(graph_id, lag)

async def record_changeset_metrics(changeset: ChangeSet, operation: str):
    if operation == 'created':
        changeset_created.labels(
            graph_id=changeset.graph_id,
            ai_generated=str(changeset.ai_generated)
        ).inc()
    elif operation == 'approved':
        approval_time = (changeset.approved_at - changeset.submitted_at).total_seconds()
        changeset_approval_time.labels(graph_id=changeset.graph_id).observe(approval_time)
    elif operation == 'conflicted':
        changeset_conflict_rate.labels(
            graph_id=changeset.graph_id,
            conflict_type='version_mismatch'
        ).inc()
    elif operation == 'rebased':
        changeset_rebase_success.labels(
            graph_id=changeset.graph_id,
            auto_rebase=str(changeset.auto_rebased)
        ).inc()
```

## Structured Logs to Loki

```python
import structlog

logger = structlog.get_logger()

# JSON logs with trace correlation
logger.info("projection_applied",
    trace_id=span.get_span_context().trace_id,
    tenant_id=tenant_id,
    graph_id=graph_id,
    event_seq=event["seq"],
    event_type=event["type"],
    query_template_id="artifact_upsert",
    rows_affected=1
)
```

Example log entry:
```json
{
  "timestamp": "2024-01-20T10:30:45.123Z",
  "level": "info",
  "msg": "projection_applied",
  "trace_id": "7b3a4e2f9c1d5a8b",
  "tenant_id": "tenant-123",
  "graph_id": "repo-456",
  "event.seq": 8421,
  "event.type": "ArtifactCreated",
  "query_template_id": "artifact_upsert",
  "rows_affected": 1
}
```

## Grafana Dashboards

### Projection Overview
- Event processing rate (events/sec)
- Projection lag by graph (p50, p95, p99)
- Error rate by event type
- DLQ accumulation
- Consumer group lag

### Neo4j Health
- Cypher latency by query_template_id
- Connection pool utilization
- Error rates by error code
- Cache hit ratio
- Active transactions

### ChangeSet Dashboard
```yaml
panels:
  - title: ChangeSet Queue Depth
    query: sum by (status) (changeset_by_status)
    visualization: stat
    
  - title: Approval Latency (p95)
    query: histogram_quantile(0.95, rate(changeset_approval_seconds_bucket[5m]))
    unit: seconds
    
  - title: AI vs Human Proposals
    query: sum by (ai_generated) (rate(changeset_created_total[1h]))
    visualization: pie
    
  - title: Conflict Rate
    query: rate(changeset_conflicts_total[5m])
    visualization: timeseries
    
  - title: Rebase Success Rate
    query: |
      sum(rate(changeset_rebase_success_total[1h])) /
      sum(rate(changeset_conflicts_total[1h]))
    unit: percentunit
    
  - title: Batch Commit Size Distribution
    query: histogram_quantile(0.5, rate(changeset_batch_size_bucket[5m]))
    visualization: heatmap
    
  - title: Review Queue Age
    query: time() - min by (graph_id) (changeset_created_timestamp)
    unit: seconds
```

### SLO Board
- Projection lag SLO (p95 ≤ 5s) - current vs target
- Query latency SLO (p95 ≤ 250ms) - burn rate
- Availability (99.9%) - error budget remaining
- Throughput (2k events/sec) - capacity utilization
- ChangeSet approval SLO (p95 ≤ 1h) - current performance

## Alerts with Runbooks

```yaml
alerts:
  - name: projection_lag_high
    expr: histogram_quantile(0.95, projection_lag_seconds) > 10
    for: 5m
    severity: warning
    runbook: |
      1. Check Kafka consumer lag:
         kubectl exec -n ameide deployment/graph-projector -- kafka-consumer-groups.sh --describe
      2. If high: scale consumers
         kubectl scale deployment/graph-projector --replicas=3
      3. If normal: check Neo4j latency
         MATCH (n) RETURN count(n) // warm cache
      4. Inspect traces with high neo4j.merge duration
  
  - name: dlq_spike
    expr: rate(dlq_events_total[5m]) > 10
    for: 5m
    severity: warning
    runbook: |
      1. Sample DLQ messages:
         kafka-console-consumer --topic artifacts.events.dlq --max-messages 5
      2. Check for schema issues or missing entities
      3. Fix event shape or projector logic
      4. Replay from DLQ once patched:
         graphctl replay-dlq --topic artifacts.events.dlq
  
  - name: neo4j_latency_high
    expr: histogram_quantile(0.95, neo4j_query_duration_seconds) > 1
    for: 10m
    severity: critical
    runbook: |
      1. Check query patterns:
         CALL dbms.listQueries() YIELD query, elapsedTimeMillis WHERE elapsedTimeMillis > 1000
      2. Review indexes:
         SHOW INDEXES
      3. Check for missing typed edges on hot paths
      4. Consider query optimization or caching

  - name: changeset_queue_depth_high
    expr: sum(changeset_by_status{status="pending_review"}) > 50
    for: 30m
    severity: warning
    runbook: |
      1. Check reviewer availability:
         SELECT user_id, COUNT(*) FROM changeset_reviews GROUP BY user_id
      2. Identify bottlenecks:
         - Large changesets blocking queue
         - AI-generated changesets needing extra review
      3. Consider auto-approval for low-risk changes
      4. Alert reviewers via Slack/email

  - name: changeset_approval_slo_breach
    expr: histogram_quantile(0.95, changeset_approval_seconds) > 3600
    for: 15m
    severity: warning
    runbook: |
      1. Check for stuck changesets:
         SELECT * FROM changesets WHERE status='pending_review' 
         AND created_at < NOW() - INTERVAL '1 hour'
      2. Look for conflict resolution delays
      3. Review auto-rebase failures
      4. Consider adjusting SLO or adding reviewers

  - name: changeset_conflict_spike
    expr: rate(changeset_conflicts_total[5m]) > 0.1
    for: 10m
    severity: info
    runbook: |
      1. Identify conflicting streams:
         SELECT stream_id, COUNT(*) FROM conflicts GROUP BY stream_id
      2. Check for:
         - Concurrent editors on same artifacts
         - Stale changesets needing rebase
         - Race conditions in batch commits
      3. Review optimistic locking configuration
      4. Consider advisory locks for hot artifacts
```

## Cardinality Guards

**Metrics labels** - Only use:
- `service` (graph-projector, neo4j-client)
- `env` (dev, staging, prod)
- `query_template_id` (artifact_upsert, relation_create)

**Never use as labels:**
- tenant_id
- graph_id
- artifact_id
- user_id

**Loki labels** - Keep minimal:
```yaml
{service="graph-projector", env="prod", level="info"}
```

Dynamic values as fields, not labels.

## Kill Switch & Operational Controls

```python
# Feature flags
GRAPH_READS_ENABLED = os.getenv("GRAPH_READS_ENABLED", "true") == "true"
ENABLE_TYPED_EDGES = os.getenv("ENABLE_TYPED_EDGES", "true") == "true"
PROJECTION_BATCH_SIZE = int(os.getenv("PROJECTION_BATCH_SIZE", "100"))

# Kill switch implementation
async def handle_query(request):
    if not GRAPH_READS_ENABLED:
        return Response(status=501, body="Graph reads temporarily disabled")
    
    # Normal query processing
    return await execute_graph_query(request)
```

## Rebuild Command

```bash
#!/bin/bash
# graphctl rebuild script

graphctl rebuild --tenant <tenant-id> --graph <repo-id>

# Implementation
async def rebuild_graph(tenant_id: str, graph_id: str):
    """Rebuild graph from event store."""
    
    # 1. Mark graph as rebuilding
    await set_rebuild_flag(tenant_id, graph_id, True)
    
    # 2. Clear existing graph data
    await neo4j.run("""
        MATCH (r:Repository {id: $repoId})-[:OWNS]->(n)
        DETACH DELETE n
    """, {'repoId': graph_id})
    
    # 3. Reset projection state
    await reset_projection_state(tenant_id, graph_id)
    
    # 4. Replay all events
    events = await event_store.get_all_events(
        f"graph-{graph_id}-*"
    )
    
    for batch in chunk(events, 1000):
        await projector.apply_batch(batch)
    
    # 5. Clear rebuild flag
    await set_rebuild_flag(tenant_id, graph_id, False)
    
    logger.info("rebuild_complete",
        tenant_id=tenant_id,
        graph_id=graph_id,
        events_processed=len(events)
    )
```

## Dark Reads for Validation

```python
async def validate_graph_consistency():
    """Compare graph results with document store."""
    
    sample_queries = [
        ("get_artifact", {"id": "test-artifact-1"}),
        ("find_by_kind", {"kind": "bpmn::Process"}),
        ("trace_impact", {"id": "test-artifact-2", "depth": 2})
    ]
    
    for query_type, params in sample_queries:
        # Execute in both stores
        graph_result = await graph_store.execute(query_type, params)
        doc_result = await document_store.execute(query_type, params)
        
        # Compare results
        if not results_match(graph_result, doc_result):
            logger.warning("consistency_mismatch",
                query_type=query_type,
                params=params,
                graph_count=len(graph_result),
                doc_count=len(doc_result)
            )
            
            # Record metric
            consistency_mismatches.labels(query_type=query_type).inc()
```

## Performance Monitoring

```sql
-- Slow query log
CREATE OR REPLACE FUNCTION log_slow_queries() RETURNS trigger AS $$
BEGIN
  IF NEW.duration > interval '1 second' THEN
    INSERT INTO slow_query_log (query, duration, timestamp)
    VALUES (NEW.query, NEW.duration, NOW());
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

## CI/CD Validation

```bash
# Check no cross-graph edges
neo4j-admin cypher-shell --database neo4j --non-interactive \
  "MATCH (r1:Repository)-[:OWNS]->(a1:Artifact)-[e:RELATES]-(a2:Artifact)<-[:OWNS]-(r2:Repository) 
   WHERE r1 <> r2 
   RETURN count(e)" | grep "0"

# Verify projection lag
curl -s http://localhost:9090/api/v1/query \
  -d 'query=histogram_quantile(0.95, projection_lag_seconds[5m])' | \
  jq '.data.result[0].value[1]' | \
  awk '{if ($1 > 5) exit 1}'

# Test kill switch
GRAPH_READS_ENABLED=false curl -s http://localhost:8080/api/artifacts/test | \
  grep "501"
```
---

## 113-9-governance-operations.md (old)

# Artifact-Centric Graph Architecture - Governance & Operations

## Kind Registry & Governance

### Canonical Form
All kinds follow strict canonical form:
- **Format**: `namespace::kind` (lowercase)
- **Namespaces**: `common`, `structural`, `doc`, `view`, `archimate`, `bpmn`
- **Single source of truth**: `packages/ameide_core_proto/kinds_registry.yaml`

### Registry Management
```yaml
# kinds_registry.yaml
version: 1.0.0
kinds:
  common::folder:
    status: stable
    since: 2024-01-01
    description: Container for organizing artifacts
    
  bpmn::sequenceflow:  # Always lowercase in registry
    status: stable
    since: 2024-01-01
    description: BPMN sequence flow between nodes
    
  archimate::capability:
    status: beta
    since: 2024-06-01
    deprecates: archimate::business_capability  # Old name
```

### Governance Process
1. **Propose**: Submit PR to `kinds_registry.yaml`
2. **Review**: Architecture team approval required
3. **Deprecation**: 90-day warning period with migration guide
4. **Removal**: Only after all repositories migrated

### CI Enforcement
```bash
# CI check - fail if unregistered kind used
grep -r "kind:" . --include="*.proto" --include="*.py" | \
  while read line; do
    kind=$(echo $line | sed 's/.*kind: *"\([^"]*\)".*/\1/')
    if ! grep -q "$kind" kinds_registry.yaml; then
      echo "ERROR: Unregistered kind: $kind"
      exit 1
    fi
  done
```

## Event Schema Versioning

### Versioning Strategy
```python
# Event envelope with version
class EventEnvelope:
    schema_version: int = 1  # Increment on breaking changes
    event_type: str
    payload: dict
    
    def upcast(self) -> dict:
        """Upgrade old events to current schema."""
        if self.schema_version == 1 and self.event_type == "ArtifactCreated":
            # v1 -> v2: Add graph_id if missing
            if 'graph_id' not in self.payload:
                self.payload['graph_id'] = extract_from_stream(self.stream_id)
        return self.payload
```

### Upcaster Chain
```python
class ProjectorWithUpcasting:
    def __init__(self):
        self.upcasters = {
            1: UpcastV1ToV2(),
            2: UpcastV2ToV3(),
        }
    
    async def process_event(self, event: dict):
        # Apply upcasters in sequence
        current_version = event.get('schema_version', 1)
        while current_version < CURRENT_SCHEMA_VERSION:
            event = self.upcasters[current_version].upcast(event)
            current_version += 1
        
        # Process with current schema
        await self.apply_projection(event)
```

### Contract Tests
```python
async def test_rebuild_across_versions():
    """Ensure rebuild succeeds with mixed event versions."""
    # Insert v1 events
    await insert_v1_events()
    # Insert v2 events
    await insert_v2_events()
    # Rebuild should handle both
    await graphctl_rebuild()
    assert await count_artifacts() == expected_count
```

## Delete & Tombstone Semantics

### Deletion Policy
```cypher
-- Soft delete (default)
MATCH (a:Artifact {id: $id})
SET a.deleted = true,
    a.deletedAt = datetime(),
    a.deletedBy = $userId

-- All queries must exclude soft-deleted
MATCH (a:Artifact)
WHERE coalesce(a.deleted, false) = false
RETURN a

-- Hard delete after retention period
MATCH (a:Artifact)
WHERE a.deleted = true 
  AND a.deletedAt < datetime() - duration('P90D')
DETACH DELETE a
```

### Edge Deletion
```cypher
-- Soft delete edge
MATCH ()-[r:RELATES {edge_key: $edgeKey}]-()
SET r.deleted = true,
    r.deletedAt = datetime()

-- Hard delete edge
MATCH ()-[r:RELATES {edge_key: $edgeKey}]-()
WHERE r.deleted = true
DELETE r
```

### Reified Relation Deletion
```cypher
-- Soft delete reified relation
MATCH (rel:Relation {id: $relationId})
SET rel.deleted = true

-- Remove edges on hard delete
MATCH (rel:Relation {id: $relationId})
WHERE rel.deleted = true
MATCH (rel)-[e]-()
DELETE e, rel
```

## Query Guardrails & Quotas

### Default Limits
```python
QUERY_DEFAULTS = {
    'maxDepth': 5,           # Maximum traversal depth
    'maxNodes': 50_000,      # Maximum nodes returned
    'maxEdges': 150_000,     # Maximum edges traversed
    'queryTimeout': 3000,    # Milliseconds
    'maxResultRows': 10_000, # Maximum result size
}

RATE_LIMITS = {
    'requests_per_second': 100,
    'burst_size': 200,
    'per_graph_rps': 10,
}
```

### Enforcement
```python
async def execute_query(query: str, params: dict, limits: dict = None):
    limits = {**QUERY_DEFAULTS, **(limits or {})}
    
    # Add limits to Cypher
    query = f"""
    CALL apoc.cypher.runTimeboxed(
      '{query}',
      {params},
      {limits['queryTimeout']}
    ) YIELD value
    RETURN value
    LIMIT {limits['maxResultRows']}
    """
    
    # Check result size
    result = await neo4j.run(query)
    if len(result) >= limits['maxResultRows']:
        raise QueryLimitExceeded(413, "Result too large")
    
    return result
```

### Rate Limiting
```python
from asyncio_throttle import Throttler

class RateLimitedAPI:
    def __init__(self):
        self.throttler = Throttler(
            rate_limit=100,  # requests per second
            period=1.0,
            burst_size=200
        )
    
    async def handle_request(self, request):
        async with self.throttler:
            # Check per-graph limit
            repo_key = f"{request.tenant_id}:{request.graph_id}"
            if await check_repo_limit(repo_key):
                return Response(429, headers={'Retry-After': '1'})
            
            return await process_request(request)
```

## Access Control Mapping

### Keycloak to Repository Roles
```python
# Role mapping
ROLE_MAPPING = {
    'graph.viewer': ['read'],
    'graph.editor': ['read', 'write'],
    'graph.owner': ['read', 'write', 'delete', 'admin'],
}

def extract_permissions(jwt_claims: dict) -> dict:
    """Map Keycloak claims to graph permissions."""
    permissions = {}
    
    # Extract graph roles from resource_access
    for repo_id, roles in jwt_claims.get('resource_access', {}).items():
        if repo_id.startswith('repo-'):
            permissions[repo_id] = []
            for role in roles.get('roles', []):
                permissions[repo_id].extend(ROLE_MAPPING.get(role, []))
    
    return permissions
```

### Permission Cache
```python
from cachetools import TTLCache

class AuthzCache:
    def __init__(self):
        self.cache = TTLCache(maxsize=10000, ttl=60)  # 60s TTL
    
    async def check_permission(self, user_id: str, repo_id: str, action: str):
        cache_key = f"{user_id}:{repo_id}"
        
        # Check cache
        if cache_key in self.cache:
            return action in self.cache[cache_key]
        
        # Fetch from Keycloak
        permissions = await fetch_user_permissions(user_id, repo_id)
        self.cache[cache_key] = permissions
        
        return action in permissions
```

## Data Lifecycle & Compliance

### GDPR/PII Handling
```python
# PII fields encryption
PII_FIELDS = ['email', 'phone', 'ssn', 'address']

class PIIEncryptor:
    def encrypt_event(self, event: dict) -> dict:
        for field in PII_FIELDS:
            if field in event:
                event[field] = self.encrypt(event[field])
                event[f"{field}_encrypted"] = True
        return event
    
    def decrypt_for_authorized(self, event: dict, user_permissions: list) -> dict:
        if 'pii_read' not in user_permissions:
            return event
        
        for field in PII_FIELDS:
            if f"{field}_encrypted" in event:
                event[field] = self.decrypt(event[field])
        return event
```

### Right to be Forgotten
```python
async def handle_erasure_request(user_id: str):
    """GDPR Article 17 - Right to erasure."""
    
    # 1. Write compensating events
    await event_store.append(
        stream_id=f"user-{user_id}",
        event=UserErasureRequested(user_id=user_id)
    )
    
    # 2. Anonymize in projections
    await neo4j.run("""
        MATCH (a:Artifact {createdBy: $userId})
        SET a.createdBy = 'ANONYMIZED',
            a.pii_erased = true
    """, {'userId': user_id})
    
    # 3. Schedule hard delete after retention
    await schedule_hard_delete(user_id, days=30)
```

### Retention Schedules
```sql
-- Retention policy table
CREATE TABLE retention_policies (
    artifact_kind TEXT PRIMARY KEY,
    retention_days INTEGER NOT NULL,
    hard_delete BOOLEAN DEFAULT false
);

INSERT INTO retention_policies VALUES
    ('common::log', 90, true),
    ('doc::draft', 180, false),
    ('archimate::model', 2555, false);  -- 7 years

-- Cleanup job
DELETE FROM artifacts
WHERE deleted = true
  AND deletedAt < NOW() - INTERVAL '1 day' * (
    SELECT retention_days FROM retention_policies
    WHERE artifact_kind = artifacts.kind
  );
```

## Operational Limits

### Repository Quotas
```python
REPOSITORY_LIMITS = {
    'max_artifacts': 5_000_000,
    'max_edges_per_node': 50_000,
    'max_blob_size_mb': 25,
    'max_total_storage_gb': 100,
    'warning_threshold': 0.8,  # Warn at 80%
}

async def check_quotas(repo_id: str):
    """Check graph against quotas."""
    stats = await get_repo_stats(repo_id)
    
    for metric, limit in REPOSITORY_LIMITS.items():
        current = stats.get(metric, 0)
        if current > limit:
            raise QuotaExceeded(f"{metric} exceeded: {current}/{limit}")
        
        if current > limit * REPOSITORY_LIMITS['warning_threshold']:
            await notify_admins(f"Repository {repo_id} approaching {metric} limit")
```

## Failure Modes & Recovery

### Retry Strategy
```python
from tenacity import retry, stop_after_attempt, wait_exponential

class ResilientProjector:
    @retry(
        stop=stop_after_attempt(7),
        wait=wait_exponential(multiplier=0.1, max=2)  # 100ms -> 2s
    )
    async def process_with_retry(self, event):
        try:
            await self.apply_projection(event)
        except Neo4jConnectionError:
            # Circuit breaker
            if self.failure_count > 10:
                await self.pause_partition()
                raise CircuitBreakerOpen()
            self.failure_count += 1
            raise
```

### Backpressure Handling
```python
async def handle_backpressure():
    """Auto-scale based on lag."""
    lag = await get_consumer_lag()
    
    if lag > 10000:  # 10k events behind
        await scale_consumers(min(current * 2, MAX_CONSUMERS))
    elif lag < 100 and current > MIN_CONSUMERS:
        await scale_consumers(max(current // 2, MIN_CONSUMERS))
```

## Snapshots & Rebuild Optimization

### Aggregate Snapshots
```python
async def create_snapshot(stream_id: str):
    """Snapshot aggregate state periodically."""
    events = await event_store.get_events(stream_id)
    
    if len(events) > SNAPSHOT_THRESHOLD:
        state = rebuild_aggregate(events)
        await snapshot_store.save(
            stream_id=stream_id,
            version=events[-1].version,
            state=state
        )
```

### Graph Snapshots
```bash
# Nightly graph snapshot for fast rebuilds
neo4j-admin backup \
  --database=neo4j \
  --backup-dir=/backups/$(date +%Y%m%d) \
  --incremental
```

## Feature Flag Management

### Per-Repository Toggles
```sql
CREATE TABLE graph_features (
    graph_id TEXT,
    feature_name TEXT,
    enabled BOOLEAN,
    config JSONB,
    PRIMARY KEY (graph_id, feature_name)
);

-- Override precedence: repo > tenant > global
WITH feature_config AS (
    SELECT enabled, 1 as priority FROM graph_features 
    WHERE graph_id = $1 AND feature_name = $2
    UNION
    SELECT enabled, 2 FROM tenant_features
    WHERE tenant_id = $3 AND feature_name = $2
    UNION
    SELECT enabled, 3 FROM global_features
    WHERE feature_name = $2
)
SELECT enabled FROM feature_config
ORDER BY priority
LIMIT 1;
```

## Projector Upgrade Safety

### Blue-Green Deployment
```yaml
# Deploy new version alongside old
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graph-projector-v2
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: projector
        image: graph-projector:v2
        env:
        - name: CONSUMER_GROUP
          value: graph-projection-v2  # Different group
        - name: MODE
          value: shadow  # Don't commit offsets
```

### Query Template Versioning
```python
QUERY_TEMPLATES = {
    'artifact_upsert_v1': """...""",
    'artifact_upsert_v2': """...""",  # New version
}

def get_query_template(template_id: str, version: int):
    key = f"{template_id}_v{version}"
    return QUERY_TEMPLATES[key]
```
---

## 113-canvas-platform-integration-authentication.md (old)

# Platform Authentication Integration

## Executive Summary

This document details the authentication architecture for the AMEIDE platform, with a unified Keycloak SSO implementation across the authenticated platform service.

## Target Architecture

### Service Structure

```
services/
├── www-ameide/              # Public portal (no authentication)
│   └── Landing page, documentation, marketing
│
└── www_ameide_platform/     # Authenticated platform (Keycloak SSO)
    └── Chat, canvas, BPMN editor, all platform features
```

### Authentication Flow

```mermaid
graph TD
    A[User] -->|Visits| B[www-ameide - Public Portal]
    B -->|Clicks Login| C[Redirect to Platform]
    C -->|No Session| D[Keycloak Login]
    D -->|Authenticates| E[Keycloak]
    E -->|OIDC Callback| F[Platform Auth Handler]
    F -->|Store Tokens| G[Redis Session Store]
    F -->|Create Session| H[NextAuth Session]
    H -->|Access| I[Protected Platform Features]
    
    J[Token Refresh] -->|Before Expiry| K[Refresh Handler]
    K -->|Distributed Lock| L[Prevent Race Conditions]
    L -->|Update Tokens| G
```

## Implementation Details

### www_ameide_platform Service

**Complete Keycloak Integration Features:**
- ✅ NextAuth.js v5 with Keycloak OIDC provider
- ✅ PKCE, state, and nonce security checks
- ✅ Redis-based session storage with distributed locking
- ✅ Automatic token refresh with race condition prevention
- ✅ Role extraction from JWT claims (admin detection)
- ✅ Comprehensive auth logging and metrics
- ✅ Middleware-based route protection
- ✅ Backchannel logout support via SID mapping
- ✅ CSRF protection with secure cookies
- ✅ Multi-tab session synchronization

**File Structure:**
```
services/www_ameide_platform/
├── app/(auth)/
│   ├── auth.ts              # Main NextAuth configuration
│   ├── auth.config.ts       # Base auth config
│   └── login/
│       └── page.tsx         # Auto-redirect to Keycloak
├── lib/
│   ├── keycloak.ts         # Token refresh & role extraction
│   ├── session-store.ts    # Redis session management
│   ├── auth-logger.ts      # Logging utilities
│   └── cache/
│       └── redis.ts         # Redis implementation
├── middleware.ts            # Route protection
└── tests/e2e/
    └── auth-*.test.ts      # Comprehensive auth tests
```

### www-ameide Service (Public Portal)

**No Authentication Required:**
- Public landing page
- Documentation
- Marketing content
- Links to authenticated platform

**File Structure:**
```
services/www-ameide/
├── app/
│   ├── page.tsx            # Public landing page
│   ├── docs/               # Public documentation
│   └── api/
│       └── health/         # Health endpoints
└── (no auth files)         # No authentication needed
```

## Configuration

### Keycloak Setup

```javascript
// Realm: ameide
// Client: platform-app
{
  "clientId": "platform-app",
  "protocol": "openid-connect",
  "publicClient": false,
  "standardFlowEnabled": true,
  "attributes": {
    "pkce.code.challenge.method": "S256",
    "access.token.lifespan": "300",
    "refresh.token.max.reuse": "0"
  },
  "redirectUris": [
    "http://platform.dev.ameide.io:8080/*"
  ]
}
```

### Environment Variables

```bash
# Platform service configuration
AUTH_URL=http://platform.dev.ameide.io:8080
AUTH_SECRET=development-secret-at-least-32-chars
AUTH_TRUST_HOST=true

# Keycloak
AUTH_KEYCLOAK_ISSUER=http://auth.dev.ameide.io:8080/realms/ameide
AUTH_KEYCLOAK_ID=platform-app
AUTH_KEYCLOAK_SECRET=changeme

# Redis sessions (CoreDNS handles resolution)
REDIS_URL=redis://redis-node-1.redis-headless:6379/0

# Database for shadow users (CoreDNS handles resolution)
POSTGRES_URL=postgres://dbuser:dbpassword@postgres-platform-rw:5432/platform

# Session settings
AUTH_SESSION_MAX_AGE=86400
AUTH_SESSION_UPDATE_AGE=3600
```

### Kubernetes Configuration

```yaml
# infra/kubernetes/values/platform/www-ameide-platform.yaml
service:
  name: www-ameide-platform
  port: 3001

auth:
  enabled: true
  
keycloak:
  issuer: "http://auth.dev.ameide.io:8080/realms/ameide"
  clientId: "platform-app"
  clientSecret: "changeme"

redis:
  url: "redis://redis-node-1.redis-headless:6379/0"

# Database for shadow user storage (CoreDNS resolves service names)
database:
  url: "postgres://dbuser:dbpassword@postgres-platform-rw:5432/platform"
```

```yaml
# infra/kubernetes/values/platform/www-ameide.yaml
service:
  name: www-ameide
  port: 3000

# No auth configuration - public service
```

## Security Features

### Token Storage
- Access tokens stored in Redis (never in cookies)
- Refresh tokens in Redis with TTL
- Minimal session pointer in httpOnly cookie

### Session Security
- Distributed lock for token refresh
- Session fixation protection
- Automatic session extension
- Idle timeout handling

### Network Security
- CSRF protection with double-submit cookies
- SameSite=lax cookie attributes
- Origin validation in callbacks
- Secure redirect validation

## Testing

### Integration Tests
```typescript
describe('Platform Authentication', () => {
  test('redirects to Keycloak when not authenticated');
  test('allows access after successful login');
  test('refreshes token before expiry');
  test('handles concurrent refresh attempts');
  test('validates admin roles');
  test('single sign-out works');
});
```

### Security Tests
- CSRF token validation
- XSS prevention
- Token expiry enforcement
- Secure redirect handling

## Deployment

### HTTPRoute Configuration

```yaml
# Platform (authenticated)
hostnames:
  - platform.dev.ameide.io
  - app.dev.ameide.io

# Public portal
hostnames:
  - dev.ameide.io
  - www.dev.ameide.io
```

### Health Checks

Both services expose health endpoints:
- `www-ameide`: `/api/health` (public)
- `www_ameide_platform`: `/api/health` (public endpoint, auth not required)

## Success Metrics

- **Authentication Coverage**: 100% of platform features protected
- **Session Reliability**: 99.9% uptime
- **Token Refresh Success**: >99% success rate
- **Auth Latency**: <100ms p99
- **Security**: Zero auth-related breaches

## Implementation Status

### ✅ Completed - Phase 1: Authentication Infrastructure
- Keycloak OIDC integration with NextAuth.js v5
- Token storage and refresh with Redis
- Middleware-based route protection
- CSRF and security features
- Health check endpoints
- Comprehensive test coverage

### ✅ Completed - Phase 2: Shadow User Pattern
- Database schema updated with Keycloak integration fields
- User upsert on login via `upsertUserFromKeycloak()`
- Local user records synced from Keycloak profile
- Role-based authorization helpers in `lib/auth/authorization.ts`
- UserContext provider for centralized user state
- Removed legacy guest/password authentication code

### ✅ Completed - Phase 3: Test Coverage
- Replaced all legacy authentication tests with Keycloak SSO tests
- Created comprehensive shadow user pattern integration tests
- Added role-based access control test suite
- Removed all guest user and email/password test references
- Documented shadow user pattern benefits and implementation

### ✅ Completed - Phase 4: Database Integration & Fixes (August 2025)
- Fixed missing `POSTGRES_URL` environment variable configuration
- Added database connection to Kubernetes ConfigMap and values
- Fixed role-based entitlements (`getUserEntitlements` function)
- Removed all references to `user.type` in favor of role-based access
- Successfully tested authentication flow with database persistence
- Verified shadow user creation with both `id` and `kcSub` fields

## Implemented Shadow User Pattern

### Architecture Decision
Use the **shadow user pattern** - let Keycloak own identity while maintaining minimal local user records for referential integrity.

**Keycloak owns:**
- Authentication (login, MFA, sessions, password resets)
- Core profile (name, email, picture)
- Roles and group assignments
- Token issuance with claims

**App database owns:**
- Shadow user table keyed by Keycloak `sub`
- App-specific data (preferences, quotas)
- Foreign keys for domain tables (messages, threadss)
- Audit and analytics data

### Implementation Steps

#### 1. Database Migration
```sql
-- Update User table for shadow records
ALTER TABLE "User" 
  ADD COLUMN kc_sub VARCHAR(255) UNIQUE NOT NULL,
  DROP COLUMN password,
  ADD COLUMN name VARCHAR(255),
  ADD COLUMN avatar_url TEXT,
  ADD COLUMN created_at TIMESTAMP DEFAULT NOW(),
  ADD COLUMN updated_at TIMESTAMP DEFAULT NOW();
-- Don't store roles - read from token
```

#### 2. User Sync on Login
```typescript
// In auth.ts callbacks.signIn
const upsertUser = await db.user.upsert({
  where: { kc_sub: profile.sub },
  create: {
    id: generateUUID(),
    kc_sub: profile.sub,
    email: profile.email,
    name: profile.name,
    avatar_url: profile.picture,
  },
  update: {
    email: profile.email,
    name: profile.name,
    avatar_url: profile.picture,
    updated_at: new Date(),
  },
});

// Store local ID in token for FKs
token.userId = upsertUser.id;
token.kc_sub = profile.sub;
```

#### 3. Authorization Helpers
```typescript
// Read roles from JWT, not database
interface SessionUser {
  id: string;        // Local DB ID for FKs
  kc_sub: string;    // Keycloak subject
  email: string;
  roles: string[];   // From JWT claims
}

function hasRole(user: SessionUser, role: string): boolean {
  return user.roles.includes(role);
}
```

#### 4. Cleanup Tasks
- Remove guest user system (`createGuestUser`, guest routes)
- Remove password-based authentication
- Remove hardcoded `UserType = 'keycloak'`
- Update all FK references to use local user.id

#### 5. Migration Path
1. Add `kc_sub` column (nullable initially)
2. Populate on next login for existing users
3. Make `kc_sub` required once populated
4. Remove old auth code

## Benefits
- **Referential integrity**: Local FKs work properly
- **Stable identity**: `kc_sub` never changes
- **Simple sync**: Upsert on login only
- **Clean separation**: Identity vs application data
- **Performance**: No external calls for queries

## Production Hardening

### Secret Management Patterns

#### Pattern 1: Full Connection String in Secret
```yaml
# templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "chart.fullname" . }}-db
stringData:
  POSTGRES_URL: {{ .Values.database.url | quote }}
  
# templates/deployment.yaml
envFrom:
  - secretRef:
      name: {{ include "chart.fullname" . }}-db
```

#### Pattern 2: External Secrets Operator (Recommended)
```yaml
# templates/externalsecret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ include "chart.fullname" . }}-db
spec:
  secretStoreRef:
    name: vault-backend
  target:
    name: {{ include "chart.fullname" . }}-db
  data:
    - secretKey: POSTGRES_URL
      remoteRef:
        key: database/credentials
        property: connection_string
```

#### Pattern 3: Assembled from Components
```yaml
# templates/secret.yaml
stringData:
  DB_HOST: postgres-platform-rw
  DB_NAME: platform
  DB_USER: {{ .Values.database.user }}
  DB_PASSWORD: {{ .Values.database.password }}

# App constructs: postgres://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:5432/${DB_NAME}
```

### Helm Chart Improvements

#### Auto-rollout on Config Changes
```yaml
# templates/deployment.yaml
metadata:
  annotations:
    checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
```

#### Health Probes with Database Check
```yaml
readinessProbe:
  httpGet:
    path: /api/health
    port: http
  initialDelaySeconds: 5
  periodSeconds: 5
livenessProbe:
  httpGet:
    path: /api/health
    port: http
  initialDelaySeconds: 15
  periodSeconds: 10
```

#### Values Schema Validation
```json
// values.schema.json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "properties": {
    "database": {
      "type": "object",
      "required": ["url"],
      "properties": {
        "url": {
          "type": "string",
          "pattern": "^postgres://.*"
        }
      }
    }
  },
  "required": ["database"]
}
```

### Security Best Practices

1. **Never use `${VAR}` syntax in ConfigMaps** - Kubernetes doesn't expand them
2. **Quote all values**: Use `| quote` filter to prevent YAML injection
3. **Use defaults**: `| default ""` prevents nil pointer errors
4. **Separate concerns**: Secrets in Secret objects, config in ConfigMaps
5. **Validate health**: Ensure `/api/health` checks database connectivity

### Testing Commands

```bash
# Verify ConfigMap has the right values
kubectl get cm www-ameide-platform-config -n ameide -o yaml | grep POSTGRES_URL

# Check environment in pod
kubectl exec -n ameide deploy/www-ameide-platform -- printenv | grep -E 'POSTGRES|REDIS'

# Test database connectivity from cluster
kubectl run psql --rm -i -t --image=postgres:16 --restart=Never -- \
  psql "postgres://dbuser:dbpassword@postgres-platform-rw:5432/platform" -c 'SELECT 1;'

# Check deployment annotations
kubectl describe deploy/www-ameide-platform -n ameide | grep checksum/

# Verify probes are working
kubectl get deploy/www-ameide-platform -n ameide -o yaml | grep -A10 Probe
```

## Future Enhancements

1. **User deletion sync**: Subscribe to Keycloak admin events
2. **Profile management UI**: Settings page for preferences
3. **Fine-grained permissions**: Resource-level ACLs if needed
4. **Session analytics**: Track user behavior and metrics
5. **Enhanced health checks**: Include database, Redis, and Keycloak connectivity
---

## 115-keycloak-ssl-update.md (old)

> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# Keycloak SSL/TLS Configuration

**Status**: ✅ Mostly Implemented
**Created**: 2025-01-13
**Updated**: 2025-01-13
**Priority**: Medium (Certificate fix needed)
**Dependencies**: 
- [104-keycloak-config-v2.md](./104-keycloak-config-v2.md) - OAuth configuration
- [105-domain-strategy-ameide-io.md](./105-domain-strategy-ameide-io.md) - Domain strategy
- [106-cert-manager-letsencrypt.md](./106-cert-manager-letsencrypt.md) - Certificate management
- [114-ssl.md](./114-ssl.md) - SSL security implementation

## Executive Summary

Keycloak SSL/TLS is implemented with TLS termination at the Gateway layer. The system uses a split-endpoint strategy: HTTPS for browser-facing URLs and internal HTTP for server-side communication. One critical issue remains: the local certificate covers `*.ameide.io` instead of `*.dev.ameide.io`.

## Current State

### What's Implemented ✅

1. **Gateway TLS Termination**
   - HTTPS listener on port 443 (exposed as 8443 via k3d)
   - Internal services use HTTP (Keycloak on port 4000)
   - HTTPRoute for Keycloak already configured
   - Gateway handles all TLS complexity

2. **Split Endpoint Strategy**
   - Browser requests: `https://auth.dev.ameide.io` (HTTPS)
   - Server-side calls: `http://keycloak:4000` (internal HTTP)
   - Optimizes both security and cluster performance
   - NextAuth v5 configured with separate endpoints

3. **Authentication Working**
   - OAuth flows functional with PKCE + state + nonce
   - JWT session strategy with secure cookies
   - Cookie security properly configured per environment
   - Session persistence across page refreshes

### Critical Issue to Fix ⚠️

**Certificate DNS Mismatch**
- Certificate configured for: `*.ameide.io`
- Local development uses: `*.dev.ameide.io`
- Impact: Browser certificate errors for local development

## Architecture Overview

```mermaid
graph LR
    Browser -->|HTTPS:443| k3d
    k3d -->|port-forward| Gateway
    Gateway -->|HTTP:4000| Keycloak
    Gateway -->|HTTP:3001| Platform
    
    subgraph "k3d Port Mapping"
        k3d[k3d loadbalancer<br/>8443→443]
    end
    
    subgraph "TLS Termination"
        Gateway[Envoy Gateway<br/>listener: https (443)]
    end
    
    subgraph "Internal HTTP"
        Keycloak[Keycloak<br/>keycloak:4000]
        Platform[Platform<br/>www-ameide-platform:3001]
    end
```

## Current Implementation

### Local Development Configuration

**Gateway Configuration:**
```yaml
# gitops/ameide-gitops/sources/values/local/apps/platform/gateway.yaml
listeners:
  - name: https
    port: 443  # Internal port (k3d maps 8443→443)
    protocol: HTTPS
    hostname: "*.dev.ameide.io"
    tls:
      certificateRefs:
        - name: ameide-wildcard-tls
```

**Keycloak HTTPRoute (Already Exists):**
```yaml
# infra/kubernetes/charts/platform/keycloak/templates/httproute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
spec:
  parentRefs:
    - name: ameide
      sectionName: https  # References the 'https' listener
  hostnames:
    - auth.dev.ameide.io
  rules:
    - backendRefs:
        - name: keycloak
          port: 4000
```

**NextAuth Split Endpoint Configuration:**
```typescript
// services/www_ameide_platform/app/(auth)/auth.ts
const keycloakProvider = Keycloak({
  // Browser-facing URL (HTTPS)
  authorization: {
    url: "https://auth.dev.ameide.io/realms/ameide/protocol/openid-connect/auth",
  },
  // Server-side endpoints (internal HTTP)
  token: "http://keycloak:4000/realms/ameide/protocol/openid-connect/token",
  userinfo: "http://keycloak:4000/realms/ameide/protocol/openid-connect/userinfo",
  // ... other endpoints
});
```

### 2. Staging Environment

```yaml
# environments/staging/platform/keycloak.yaml
keycloak:
  issuer: "https://auth.test.ameide.io/realms/ameide"
  
extraEnvVars:
  - name: KC_HOSTNAME
    value: "auth.test.ameide.io"
  - name: KC_HOSTNAME_STRICT
    value: "true"
  - name: KC_PROXY
    value: "edge"  # TLS at Gateway
  - name: KC_HTTP_ENABLED
    value: "true"  # HTTP internally
```

### 3. Production Environment

```yaml
# environments/production/platform/keycloak.yaml
keycloak:
  issuer: "https://auth.ameide.io/realms/ameide"
  
extraEnvVars:
  - name: KC_HOSTNAME
    value: "auth.ameide.io"
  - name: KC_HOSTNAME_STRICT
    value: "true"
  - name: KC_PROXY
    value: "edge"
  - name: KC_HTTP_ENABLED
    value: "true"
```

## Required Fix: Certificate DNS Names

### The Issue
The local development certificate is misconfigured:

```yaml
# Current (WRONG) - infra/kubernetes/environments/local/platform/cert-manager.yaml
certificates:
  - name: ameide-wildcard-local
    secretName: ameide-wildcard-tls
    dnsNames:
      - "*.ameide.io"    # Should be *.dev.ameide.io
      - "ameide.io"      # Should be dev.ameide.io
      - "localhost"
```

### The Fix
```yaml
# Required change
certificates:
  - name: ameide-wildcard-local
    secretName: ameide-wildcard-tls
    dnsNames:
      - "*.dev.ameide.io"  # Correct for local development
      - "dev.ameide.io"     # Correct for local development
      - "localhost"
```

After applying this fix, regenerate the certificate:
```bash
# Delete the old certificate and secret
kubectl delete certificate ameide-wildcard-local -n ameide
kubectl delete secret ameide-wildcard-tls -n ameide

# Reapply cert-manager configuration
helmfile -e local sync --selector name=cert-manager

# Verify new certificate
kubectl get certificate -n ameide
kubectl describe certificate ameide-wildcard-local -n ameide
```

## Certificate Trust (Developer Experience)

### Option A: Install Self-Signed Certificate (Recommended)
```bash
# Extract certificate from cluster
kubectl get secret ameide-wildcard-tls -n ameide \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > /tmp/ameide-test.crt

# macOS
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain /tmp/ameide-test.crt

# Linux
sudo cp /tmp/ameide-test.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates

# Windows (PowerShell as Admin)
Import-Certificate -FilePath C:\tmp\ameide-test.crt \
  -CertStoreLocation Cert:\LocalMachine\Root
```

### Option B: Browser Exception (Quick)
1. Visit https://auth.dev.ameide.io
2. Accept certificate warning
3. Visit https://platform.dev.ameide.io
4. Accept certificate warning

## Validation

### Health Checks
```bash
# 1. Verify Keycloak HTTPS access
curl -k https://auth.dev.ameide.io/realms/ameide/.well-known/openid-configuration

# 2. Check issuer URL in response
curl -k https://auth.dev.ameide.io/realms/ameide/.well-known/openid-configuration \
  | jq -r .issuer
# Returns: https://auth.dev.ameide.io/realms/ameide

# 3. Verify certificate DNS names (after fix)
kubectl get certificate ameide-wildcard-local -n ameide -o yaml | grep -A 3 dnsNames
# Should show *.dev.ameide.io, not *.ameide.io

# 4. Test OAuth flow in browser
# - Visit https://platform.dev.ameide.io/login
# - Click "Sign in with Keycloak"
# - Should redirect to HTTPS Keycloak login
# - After auth, check cookie has Secure flag in DevTools
```

## Security Considerations

### 1. Mixed Content Prevention
- All OAuth URLs must use HTTPS in browser context
- Server-side can use HTTP for internal communication
- Gateway ensures consistent HTTPS for external access

### 2. Cookie Security Matrix

| Environment | Cookie Name | Secure | SameSite | Domain |
|------------|-------------|---------|----------|---------|
| Local Dev | `authjs.session-token` | Yes | Lax | None |
| Staging | `__Secure-authjs.session-token` | Yes | Lax | `.test.ameide.io` |
| Production | `__Secure-authjs.session-token` | Yes | Lax | `.ameide.io` |

### 3. CORS Configuration
```typescript
// Ensure CORS allows HTTPS origins
const corsOrigins = [
  'https://platform.dev.ameide.io',
  'https://auth.dev.ameide.io',
  'https://api.dev.ameide.io'
];
```

## NextAuth v5 Configuration

The current implementation uses a split-endpoint strategy that's already optimized:

```typescript
// services/www_ameide_platform/app/(auth)/auth.ts
const keycloakProvider = Keycloak({
  clientId: process.env.AUTH_KEYCLOAK_ID,
  clientSecret: process.env.AUTH_KEYCLOAK_SECRET,
  checks: ['state', 'pkce', 'nonce'],
  
  // Browser-facing endpoints (HTTPS)
  issuer: "https://auth.dev.ameide.io/realms/ameide",
  authorization: {
    url: "https://auth.dev.ameide.io/realms/ameide/protocol/openid-connect/auth",
  },
  
  // Server-side endpoints (internal HTTP for performance)
  token: "http://keycloak:4000/realms/ameide/protocol/openid-connect/token",
  userinfo: "http://keycloak:4000/realms/ameide/protocol/openid-connect/userinfo",
  wellKnown: "http://keycloak:4000/realms/ameide/.well-known/openid-configuration",
  
  httpOptions: {
    timeout: 10000,
    // Handle self-signed certs in development
    ...(process.env.NODE_ENV !== 'production' && {
      agent: new https.Agent({ rejectUnauthorized: false }),
    }),
  },
});
```

## Testing Checklist

### Prerequisites
- [x] Keycloak HTTPRoute exists and configured
- [x] Gateway HTTPS listener on port 443
- [x] NextAuth split-endpoint configuration
- [ ] Certificate DNS names fixed (*.dev.ameide.io)

### After Certificate Fix
- [ ] Certificate shows `*.dev.ameide.io` in DNS names
- [ ] Keycloak accessible via `https://auth.dev.ameide.io`
- [ ] Platform accessible via `https://platform.dev.ameide.io`
- [ ] OAuth flow completes successfully
- [ ] Cookies have Secure flag set
- [ ] No certificate warnings (after trust installation)
- [ ] Session persists across page refreshes

## Action Items

1. **Immediate**: Fix certificate DNS names configuration
   - Update `infra/kubernetes/environments/local/platform/cert-manager.yaml`
   - Change `*.ameide.io` → `*.dev.ameide.io`
   - Regenerate certificates

2. **Documentation**: Update developer onboarding
   - Add certificate trust installation steps
   - Document the split-endpoint strategy
   - Explain k3d port mapping (8443→443)

3. **Production Path**: 
   - Staging/Production already configured correctly
   - Use Let's Encrypt for production certificates
   - Keep split-endpoint strategy for optimal performance

## Troubleshooting Guide

### Common Issues

#### 1. "Invalid redirect_uri" Error
**Cause**: Keycloak expects HTTPS but receives HTTP callback
**Solution**: 
```yaml
# Update Keycloak client configuration
Valid Redirect URIs:
  - https://platform.dev.ameide.io/*
  - https://platform.dev.ameide.io/*
```

#### 2. Cookie Not Set
**Cause**: Secure cookie on HTTP connection
**Solution**: Ensure all URLs use HTTPS or adjust cookie security for development

#### 3. Certificate Warnings
**Cause**: Self-signed certificate not trusted
**Solution**: Install certificate in system trust store (see Phase 2)

#### 4. Mixed Content Blocked
**Cause**: HTTPS page loading HTTP resources
**Solution**: Update all resource URLs to use HTTPS

## Summary

### Current Status
- ✅ **TLS Termination**: Gateway handles HTTPS on port 443 (exposed as 8443)
- ✅ **Split Endpoints**: HTTPS for browser, HTTP for internal cluster
- ✅ **OAuth Working**: Full authentication flow with secure cookies
- ✅ **HTTPRoute**: Keycloak routing already configured
- ❌ **Certificate Issue**: DNS names show `*.ameide.io` instead of `*.dev.ameide.io`

### Only Action Required
Fix the certificate DNS names in `infra/kubernetes/environments/local/platform/cert-manager.yaml`:
- Change `*.ameide.io` → `*.dev.ameide.io`
- Regenerate the certificate
- Optionally install certificate in system trust store

The rest of the SSL/TLS configuration is already properly implemented and working.

---

## 116-connect-v2-migration-typescript.md (old)

# Connect v2 Migration for TypeScript Packages

**Status:** ✅ Completed  
**Date:** 2025-01-15  
**Packages:** `ameide_core_proto_web`, `ameide_sdk_ts`

## Overview

Successfully migrated TypeScript proto and SDK packages from Connect v1 to Connect v2, following the official migration guide. The migration removes the deprecated `protoc-gen-connect-es` plugin and adopts the new protobuf-es v2 API where service descriptors are now part of the `_pb.js` files.

## Challenge 1: Service Descriptor Location Change

**Problem:** In Connect v1, service descriptors lived in `*_connect.js` files. In v2, they're in `*_pb.js` files alongside message definitions.

**Solution:**
```yaml
# buf.gen.connect.yaml - BEFORE (v1)
plugins:
  - local: protoc-gen-es
    out: ../ameide_core_proto_web/src/generated
    opt: [target=ts]
  - local: protoc-gen-connect-es  # REMOVED in v2
    out: ../ameide_core_proto_web/src/generated
    opt: [target=ts]

# buf.gen.connect.yaml - AFTER (v2)
plugins:
  - local: protoc-gen-es
    out: ../ameide_core_proto_web/src/generated
    opt:
      - target=ts
      - import_extension=js  # Required for NodeNext
```

## Challenge 2: Message API Changes

**Problem:** Connect v2 uses protobuf-es v2's schema-based API instead of classes.

**Before (v1):**
```typescript
import { ArtifactView } from "@ameide/core-proto-web/artifacts/v1/types";
const artifact = new ArtifactView({
  id: "123",
  name: "Test"
});
```

**After (v2):**
```typescript
import { create } from "@bufbuild/protobuf";
import { ArtifactViewSchema } from "@ameide/core-proto-web/artifacts/v1/pb";
const artifact = create(ArtifactViewSchema, {
  id: "123",
  name: "Test"
});
```

## Challenge 3: TypeScript Resolution with Package Exports

**Problem:** TypeScript couldn't resolve re-exported types through package.json `exports` field when using star re-exports (`export *`) with NodeNext module resolution.

**Failed Approach:**
```typescript
// shims/artifacts/v1/pb.ts
export * from '../../../generated/ameide_core_proto/artifacts/v1/artifacts_pb.js';
```

**Working Solution:** Use named re-exports
```typescript
// shims/artifacts/v1/pb.ts
// Service descriptors
export {
  ArtifactCommandService,
  ArtifactQueryService,
} from '../../../generated/ameide_core_proto/artifacts/v1/artifacts_pb.js';

// Schemas (for create() function)
export {
  AggregateSchema,
  ArtifactViewSchema,
  // ... other schemas
} from '../../../generated/ameide_core_proto/artifacts/v1/artifacts_pb.js';

// Types
export type {
  Aggregate,
  ArtifactView,
  // ... other types
} from '../../../generated/ameide_core_proto/artifacts/v1/artifacts_pb.js';
```

## Challenge 4: tsup DTS Mode Failures

**Problem:** tsup's built-in `.d.ts` bundling (`dts: true`) failed with TypeScript errors when dealing with package.json exports + NodeNext + re-exports.

**Solution:** Separate JavaScript bundling from type generation:

```typescript
// tsup.config.ts
export default defineConfig({
  entry: ['src/index.ts', 'src/client.ts', /* ... */],
  format: ['esm', 'cjs'],
  dts: false,  // Let tsc handle declarations
  // ...
});
```

```json
// tsconfig.types.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "declaration": true,
    "emitDeclarationOnly": true,
    "outDir": "./dist",
    "noEmit": false
  }
}
```

```json
// package.json
{
  "scripts": {
    "build": "tsup && tsc -p tsconfig.types.json"
  }
}
```

## Challenge 5: Bazel ESM Detection

**Problem:** Bazel's ts_project rule couldn't detect ESM modules correctly, causing TS1479 errors.

**Attempted Solutions:**
1. Added `"type": "module"` to package.json ✅
2. Included package.json in Bazel data attribute ✅
3. Set `supports_workers = False` in ts_project ✅

**Status:** Known Bazel limitation. Packages work correctly with native tooling (npm/tsc/tsup).

## Migration Checklist

- [x] Remove `@connectrpc/protoc-gen-connect-es` dependency
- [x] Update buf.gen.yaml to use only `protoc-gen-es` v2
- [x] Add `import_extension=js` for NodeNext compatibility
- [x] Update all imports from `/connect` to `/pb`
- [x] Replace `new Message()` with `create(Schema, data)`
- [x] Update shims to use named re-exports
- [x] Configure tsup with `dts: false`
- [x] Add separate tsconfig for type generation
- [x] Clean up old `*_connect.*` files (42 files removed)
- [x] Update package.json exports for dual ESM/CJS

## Final Configuration

### Proto Package (`ameide_core_proto_web`)
```json
{
  "type": "module",
  "exports": {
    "./artifacts/v1/pb": {
      "types": "./dist/shims/artifacts/v1/pb.d.ts",
      "import": "./dist/shims/artifacts/v1/pb.js",
      "default": "./dist/shims/artifacts/v1/pb.js"
    }
  }
}
```

### SDK Package (`ameide_sdk_ts`)
```json
{
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/index.cjs",
      "default": "./dist/index.js"
    }
  },
  "scripts": {
    "build": "tsup && tsc -p tsconfig.types.json"
  }
}
```

## Verification

```bash
# Proto package builds
cd packages/ameide_core_proto_web && npm run build  # ✅

# SDK builds with dual format
cd packages/ameide_sdk_ts && npm run build     # ✅

# Runtime works
node -e "import('@ameide/core-proto-web/artifacts/v1/pb').then(m => 
  console.log('Services:', Object.keys(m).filter(k => k.includes('Service'))))"
# Output: Services: ['ArtifactCommandService', 'ArtifactQueryService', 'CrossReferenceService']

# CJS works
node -e "const sdk = require('@ameideio/ameide-sdk-ts'); 
  console.log('CJS:', Object.keys(sdk))"
# Output: CJS: ['AmeideClient', 'makeTransport', ...]
```

## Root Cause: The Four-Way Fight

The "spaghetti feeling" comes from juggling four opinionated systems that only *almost* agree:

1. **Node ESM rules** - needs real `.js` extensions, `"type": "module"`, package `exports`
2. **TypeScript's resolver** - NodeNext ≠ Node; struggles with `export *` through subpath exports
3. **Codegen (buf/protobuf-es/connect)** - v1→v2 moved services + changed messages to schema types
4. **Bazel** - sandboxes files; ts_project doesn't "see" `package.json` the way Node/TS expect

Each is sensible alone; together, they fight.

## The "No-Surprises" Recipe

### 1. One Source of Truth for Protos
- Use **protoc-gen-es v2** only (no `protoc-gen-connect-es`)
- `buf.gen.yaml`: `clean: true`, `include_imports: true`, `opt: target=ts, import_extension=js`

### 2. Keep Generated Code Out of Your Compiler's Hair
- Generate TS (`*_pb.ts`) then treat it like third-party code
- Don't hand-edit generated files. Put all stability behind **thin shims**

### 3. Make Shims Explicit (No Wildcard Re-exports)
```typescript
// ❌ BAD: TypeScript can't resolve this through subpath exports
export * from '../../../generated/ameide_core_proto/artifacts/v1/artifacts_pb.js';

// ✅ GOOD: Explicit named exports
export {
  ArtifactCommandService,
  ArtifactQueryService,
} from '../../../generated/ameide_core_proto/artifacts/v1/artifacts_pb.js';

export type {
  Aggregate,
  ArtifactView,
  // ... specific types
} from '../../../generated/ameide_core_proto/artifacts/v1/artifacts_pb.js';
```

### 4. Pick One Module Story and Stick to It
- `module: "NodeNext"`, `moduleResolution: "NodeNext"`, real `.js` extensions everywhere
- Add `"lib": ["ES2020","DOM","DOM.Iterable"]` if you touch `Headers.entries()` & friends

### 5. Split JS Bundling from Type Emission
- **tsup**: JS only (esm+cjs), `dts: false`
- **tsc**: declarations only via `tsconfig.types.json` (`emitDeclarationOnly: true`)

### 6. Subpath Exports Are Contracts
```json
// proto pkg package.json
{
  "exports": {
    "./artifacts/v1/pb": {
      "types": "./dist/shims/artifacts/v1/pb.d.ts",
      "import": "./dist/shims/artifacts/v1/pb.js"
    }
    // Don't export old *_connect paths (v2 puts services in *_pb)
  }
}
```

### 7. Bazel: Minimize Friction
- Avoid using `ts_project` to produce publishable JS; let tsup/tsc do that
- If you *do* type-check under Bazel, add `data = ["package.json"]` so NodeNext rules apply

### 8. Pin Versions & Add Smoke Tests
```json
// Pin in package.json
{
  "dependencies": {
    "@bufbuild/protobuf": "2.6.3",  // exact version
    "@connectrpc/connect": "2.0.4"   // exact version
  }
}
```

Add two tiny tests:
- `node` ESM import of `@pkg/artifacts/v1/pb` at runtime
- `tsc --noEmit` that imports the same and references a `...Schema`

## Mental Model

```
protos ──buf/protoc-gen-es v2──▶ *_pb.ts
                              ▲
                              │  (explicit named re-exports)
                         shims/pb.ts ──▶ published subpath export
                                              ▲
                                              │
                                           SDK imports
                            tsup(JS only) + tsc(d.ts only)
```

## Lessons Learned

1. **Named exports > star exports** for TypeScript with package.json exports
2. **Separate type generation from bundling** when using complex module setups
3. **Test both ESM and CJS** imports to ensure dual-format compatibility
4. **Clean old files** - leftover v1 files can cause confusing errors
5. **Follow vendor guides exactly** - the Connect team's migration guide was prescriptive for good reasons
6. **The intersection is the problem** - ESM + TS + codegen + Bazel each work fine alone
7. **Explicit shim exports + split builds** are the big stabilizers

## References

- [Connect v2 Migration Guide](https://connectrpc.com/docs/web/migrating)
- [Protobuf-ES v2 Manual](https://github.com/bufbuild/protobuf-es/blob/v2/MANUAL.md)
- [TypeScript NodeNext Resolution](https://www.typescriptlang.org/docs/handbook/modules/reference.html#node16-nodenext)

---

## Final Simplification (2025-08-17)

### Major Architecture Change: Removed `ameide_core_proto_web`

After successfully migrating to Connect-ES v2, we discovered an even simpler architecture:

**Before**: Three packages with complex dependencies
```
ameide_core_proto → ameide_core_proto_web → ameide_sdk_ts → platform
```

**After**: Two packages with clear boundaries
```
ameide_core_proto → ameide_sdk_ts → platform
```

### Key Changes

1. **Eliminated `ameide_core_proto_web` entirely** (169 files deleted)
   - Proto generation now happens directly in `ameide_core_proto`
   - No more separate web-specific proto package
   - Simpler build process with fewer moving parts

2. **SDK as the single boundary**
   - SDK re-exports ALL proto types: `export * from "@ameide/core-proto"`
   - Platform NEVER imports from proto directly
   - Clean architectural boundary enforced

3. **Simplified build process**
   - Proto: `pnpm run generate` (using buf)
   - SDK: `pnpm run build` (using tsc -b)
   - No more tsup, just TypeScript compilation

4. **Pure ESM with CJS compatibility**
   ```json
   {
     "type": "module",
     "exports": {
       ".": {
         "types": "./dist/index.d.ts",
         "import": "./dist/index.js",
         "default": "./dist/index.js"  // CJS fallback
       }
     }
   }
   ```

### Cleanup Performed

- ✅ Removed entire `packages/ameide_core_proto_web` directory
- ✅ Removed unused dependencies (`@connectrpc/connect-query`)
- ✅ Removed unused buf configs (grpc-web, go, python, local)
- ✅ Removed duplicate `interceptors.ts` file
- ✅ Cleaned up .tgz files from services
- ✅ Updated all documentation references
- ✅ Updated BUILD.bazel to remove proto-web references

### Working pnpm Workspace Configuration

The migration also standardized on pnpm workspaces:

```yaml
# pnpm-workspace.yaml
packages:
  - 'packages/*'
  - 'services/www_ameide_platform'

injectWorkspacePackages: true
```

With proper workspace protocol in dependencies:
```json
{
  "dependencies": {
    "@ameide/core-proto": "workspace:*",
    "@ameideio/ameide-sdk-ts": "workspace:*"
  }
}
```

### Final Architecture Benefits

1. **Simpler mental model**: Just proto → SDK → platform
2. **Faster builds**: One less package to build
3. **Clearer boundaries**: SDK is THE boundary for proto types
4. **Better IDE support**: Direct TypeScript compilation without bundlers
5. **Easier debugging**: Standard tsc output without transformation

### Verification

```bash
# Build everything in workspace
pnpm run build

# Proto builds successfully
cd packages/ameide_core_proto && pnpm run build  # ✅

# SDK builds with proto re-exports
cd packages/ameide_sdk_ts && pnpm run build  # ✅

# Platform uses SDK only
cd services/www_ameide_platform && pnpm run build  # ✅

# Docker build works
docker build -f services/www_ameide_platform/Dockerfile.release .  # ✅

# Kubernetes deployment via Tilt
tilt up  # ✅ Health checks pass
```

### Lessons from Simplification

1. **Question intermediate layers** - `ameide_core_proto_web` added no value
2. **SDK as boundary works** - Clean separation without complexity
3. **Standard tools win** - tsc over tsup reduced configuration
4. **Workspace protocols work** - pnpm's `workspace:*` solved resolution
5. **Less is more** - Removing a package made everything clearer

The Connect-ES v2 migration is now complete with a simpler, cleaner architecture than originally planned.
---

## 117-bazel-north-star.md (old)

# North Star - Bazel Proto Generation

## Status: ✅ IMPLEMENTED (2025-08-16)

Successfully implemented production-ready proto generation and integrated with Tilt/Docker workflows.

### Implementation Challenges & Solutions

#### 1. Tree Artifact Challenge (Bazel)
**Problem**: Genrule cannot output directories, only individual files. Proto generation needs to output entire directory trees.
```python
# This fails - genrule can't declare directories
genrule(
    name = "buf_js",
    outs = ["generated"],  # ERROR: must be specific files
)
```

**Solution**: Created custom Starlark rule that properly declares tree artifacts:
```python
# tools/buf_gen_simple.bzl
def _buf_generate_tree_impl(ctx):
    outdir = ctx.actions.declare_directory("generated")  # Tree artifact
    ctx.actions.run_shell(
        outputs = [outdir],
        inputs = ctx.files.srcs + [ctx.file.buf_yaml, ctx.file.template, ctx.file.buf_bin],
        command = "... buf generate --output {}".format(outdir.path),
    )
    return [DefaultInfo(files = depset([outdir]))]
```

#### 2. Platform-Specific npm Binary Issues
**Problem**: `@bufbuild/buf` npm package pulls platform-specific binaries (Darwin on macOS, fails on Linux ARM64 in CI).

**Solution**: Use Bazel's hermetic buf toolchain + copy_file workaround:
```python
copy_file(
    name = "buf_bin",
    src = "@rules_buf_toolchains//:buf",  # Hermetic, platform-correct
    out = "buf",
    is_executable = True,
)
```

#### 3. Docker Context Mismatch (Tilt)
**Problem**: Dockerfile.dev expected `packages/` directory but Tilt used service directory as context:
```dockerfile
# Dockerfile.dev expected:
COPY packages/ameide_core_proto_web/package.json ./packages/ameide_core_proto_web/
# But Tilt context was: services/www_ameide_platform/
```

**Solution**: Changed Docker build context to workspace root with selective includes:
```python
# Tiltfile
docker_build(
    repo,
    '.',  # Changed from service_path to workspace root
    dockerfile=dev_dockerfile,
    only=[  # Limit what gets sent to Docker
        'package.json',
        'pnpm-workspace.yaml',
        'packages/ameide_core_proto_web/',
        'packages/ameide_sdk_ts/',
        service_path + '/',
    ],
)
```

#### 4. Missing Proto → Package → Docker Chain
**Problem**: Proto changes didn't trigger package rebuilds or Docker image updates. Tilt wasn't watching proto files.

**Solution**: Added local_resource builds with proper dependency chain:
```python
# Build proto package when proto files change
local_resource(
    'proto-web-build',
    cmd='bazel build //packages/ameide_core_proto_web:npm_package',
    deps=['packages/ameide_core_proto/proto', ...],  # Watch proto files
)

# SDK depends on proto-web
local_resource(
    'sdk-build',
    cmd='bazel build //packages/ameide_sdk_ts:npm_package',
    resource_deps=['proto-web-build'],  # Chain dependency
)

# Frontend depends on both packages
k8s_resource(
    name,
    resource_deps=['proto-web-build', 'sdk-build']  # Full chain
)
```

#### 5. Live Update Path Misalignment
**Problem**: Live update synced to `/app/src` but container WORKDIR was `/app/services/www_ameide_platform`.

**Solution**: Fixed sync paths to match container structure:
```python
live_update=[
    sync(service_path + '/src', '/app/' + service_path + '/src'),
    sync('packages/ameide_core_proto_web/generated', '/app/packages/ameide_core_proto_web/generated'),
]
```

#### 6. pnpm Workspace Integration
**Problem**: Switching from `file:` references to `workspace:*` protocol caused lockfile checksum mismatches.

**Solution**: Use `--no-frozen-lockfile` in development Docker builds:
```dockerfile
RUN pnpm install --no-frozen-lockfile  # Allow lockfile updates in dev
```

### What Works
- ✅ `bazel build //packages/ameide_core_proto_web:npm_package` - Proto package builds with tree artifacts
- ✅ `bazel build //packages/ameide_sdk_ts:npm_package` - SDK builds (simplified to source distribution)
- ✅ Proto changes automatically trigger: proto-web → SDK → Docker → Kubernetes
- ✅ Docker builds work with pnpm workspace structure
- ✅ Live updates work for rapid frontend development
- ✅ Remote Buf plugins eliminate platform-specific issues

### Known Limitations
- ⚠️ SDK package simplified to source distribution (avoids ts_project complexity)
- ⚠️ Minor skylib deprecation warnings (harmless)
- ⚠️ pnpm lockfile not strictly enforced in dev (necessary for workspace:* protocol)

## Gap Analysis: Current vs Ideal

### Current State (60-70% Complete)
We have a working system with pragmatic compromises:

| Component | Current Implementation | Ideal Solution | Gap Impact |
|-----------|----------------------|----------------|------------|
| **Proto Generation** | Custom tree artifact rule | Vendor buf_generate rule | Low - works reliably |
| **TypeScript Build** | Source distribution only | Compiled ESM+CJS+types | Medium - consumers must compile |
| **Build Speed** | Sequential, ~15-20s proto→browser | Parallel, <5s with cache | High - affects developer velocity |
| **Type Safety** | 70% (runtime compilation) | 100% pre-compiled | Medium - catches errors late |
| **Docker Integration** | Rebuilds from source | Uses pre-built packages | Low - only affects build time |
| **Testing** | Separate, no contract tests | Integrated with proto changes | Medium - misses breaking changes |

### Why We Made These Trade-offs

1. **ts_project complexity**: The npm_link_package resolution for local packages is notoriously complex in Bazel. Shipping sources avoids this entirely.

2. **Custom tree artifact**: rules_buf doesn't provide buf_generate yet. Our 50-line Starlark rule is simple and maintainable.

3. **pnpm workspace:* protocol**: Required for monorepo but breaks lockfile checksums. The --no-frozen-lockfile compromise keeps development moving.

4. **Sequential builds**: Easier to debug and understand. Parallelization can come later with remote cache.

### Path to 100% Ideal

**Phase 1 - Quick Wins (1 week)**
- [ ] Add proto contract tests with buf breaking
- [ ] Set up Bazel remote cache for CI
- [ ] Document the current patterns for team

**Phase 2 - Type Safety (2-4 weeks)**
- [ ] Get ts_project working with npm_link_package
- [ ] Pre-compile SDK to ESM and CJS
- [ ] Add source maps for debugging

**Phase 3 - Performance (1-2 months)**
- [ ] Implement ibazel for instant rebuilds
- [ ] Optimize Docker layers with pre-built tarballs
- [ ] Parallel build execution with proper dependencies

**Phase 4 - Tooling (3-6 months)**
- [ ] Contribute buf_generate to rules_buf
- [ ] Create Bazel macro for TypeScript SDK pattern
- [ ] Full hermetic toolchain with no external dependencies

### The Pragmatic Reality

**This setup works and ships code.** The gaps are mostly about developer experience and build time, not correctness. Given that:

- Proto changes correctly propagate through the entire stack
- Type definitions are available (even if runtime-compiled)
- The system is deterministic and reproducible (mostly)
- Developers can iterate quickly with Tilt hot reload

**We recommend using this as-is** while incrementally improving. Perfect Bazel setups are a journey, not a destination. The current 60-70% solution is better than being blocked on the perfect 100%.

---

## Original North Star

Bazel = builds only (SDK + proto web packages).

Tilt/Helm = deploy (both www-ameide & www-ameide-platform).

Follow rules_js, rules_ts, rules_buf docs exactly.

What to do (vendor pattern)

Root workspace hygiene (per pnpm & rules_js docs)

Keep a minimal root package.json (no "type": "module", no deps).

pnpm-workspace.yaml lists packages/* and services/*.

No pnpm lockfile; rely on workspace manifests instead.

In MODULE.bazel, configure npm without a translated lock:

npm_translate_lock(name = "npm")

Use use_repo(npm, "npm").

Node modules for actions (rules_js)

In each Bazel package that runs node/tsc/buf add:

npm_link_all_packages(name = "node_modules")

In every ts_project(...) add:

data = [":node_modules", "package.json"]
(This is exactly how rules_js wants you to surface .bin + deps in the sandbox.)

Proto generation (rules_buf + genrule, per vendor guidance)

rules_buf does not provide a codegen rule; vendor docs recommend running buf yourself.

In MODULE.bazel:

buf = use_extension("@rules_buf//buf:extensions.bzl", "buf")

buf.toolchains(version = "<current buf>")

use_repo(buf, "rules_buf_toolchains")

**✅ COMPLETED**: In packages/ameide_core_proto_web/BUILD.bazel, created:
- `copy_file` to stage buf binary as executable (workaround for Linux ARM64)
- Custom Starlark rule `buf_generate_tree` that outputs a proper tree artifact
- Uses remote Buf plugins (`buf.build/bufbuild/es` and `buf.build/connectrpc/es`)
- Generates JS+d.ts directly with `target=js+dts` (no re-transpilation needed)
- Outputs to standardized `generated/` directory

**Implementation:**
```python
# tools/buf_gen_simple.bzl - minimal tree artifact rule
buf_generate_tree = rule(
    implementation = _buf_generate_tree_impl,
    attrs = {
        "srcs": attr.label_list(allow_files = True),
        "buf_yaml": attr.label(allow_single_file = True),
        "template": attr.label(allow_single_file = True),
        "buf_bin": attr.label(allow_single_file = True),
    },
)

# packages/ameide_core_proto_web/BUILD.bazel
copy_file(
    name = "buf_bin",
    src = "@rules_buf_toolchains//:buf",
    out = "buf",
    is_executable = True,
)

buf_generate_tree(
    name = "buf_js",
    srcs = ["//packages/ameide_core_proto:buf_module"],
    buf_yaml = "//packages/ameide_core_proto:proto/buf.yaml",
    template = "//packages/ameide_core_proto:buf.gen.bazel.yaml",
    buf_bin = ":buf_bin",
)
```

TypeScript (rules_ts)

**✅ COMPLETED**: In packages/ameide_core_proto_web:

ts_project(ts_esm) compiles only your handwritten shims (not generated JS).

data = [":buf_js", ":node_modules", "package.json"] so shims can import generated files.

**Implementation:**
```python
ts_project(
    name = "ts_esm",
    srcs = glob(["src/shims/**/*.ts", "src/index.ts"]),
    root_dir = ".",
    out_dir = "dist",
    transpiler = "tsc",
    tsconfig = "tsconfig.json",
    allow_js = True,
    resolve_json_module = True,
    declaration = True,
    data = [":buf_js", "package.json"],
    validate = False,  # Skip validation since generated files come from buf_js
)
```

In packages/ameide_sdk_ts:

Keep two ts_projects if you truly need dual outputs (ESM/CJS); otherwise ESM only (per Connect/ES recommendations).

Add npm_link_package(name = "node_modules/@ameide/core-proto-web", src = "//packages/ameide_core_proto_web:npm_package") so tsc resolves the local built proto web package without hacks.

data = [":node_modules", ":node_modules/@ameide/core-proto-web", "package.json"].

Packaging (rules_js)

**✅ COMPLETED**: For each package, one npm_package(...) that includes:

Proto web: srcs = [":buf_js", ":ts_esm", "package.json", "README.md"] + glob(["src/**/*.ts"])

SDK: srcs = [":ts_esm", ":ts_cjs", "package.json", "README.md"]

**Implementation:**
```python
# packages/ameide_core_proto_web/BUILD.bazel
npm_package(
    name = "npm_package",
    srcs = [
        ":buf_js",  # generated JS + d.ts
        "package.json",
        "README.md",
    ] + glob(["src/**/*.ts"]),  # Include raw shim sources for now
    package = "@ameide/core-proto-web",
)

# Alias for npm_link_all_packages which expects :pkg
alias(
    name = "pkg",
    actual = ":npm_package",
)
```

Do not call npm binaries via @npm//:node_modules/...; vendor docs say rely on :node_modules + PATH.

Services (Tilt/Helm)

Leave the Tiltfile and Helm charts unchanged.

Services consume the workspace packages via pnpm (e.g., workspace:*) or by copying the Bazel npm tarballs if that’s your chosen pattern.

No Bazel targets in service Dockerfiles; Bazel’s job ends at npm artifacts.

Agent-ready checklist (give this to both agents)

Build-agent (Bazel)

 Root package.json minimal; pnpm-workspace.yaml lists packages/*, services/*.

 MODULE.bazel: npm_translate_lock, rules_buf toolchains registered.

 In packages/ameide_core_proto_web & packages/ameide_sdk_ts: npm_link_all_packages(name="node_modules").

 genrule for buf generation as above (tools = @rules_buf_toolchains//:buf, PATH adds :node_modules/.bin).

 ts_project compiles only handwritten TS; generated JS is consumed via data.

 npm_package emits the package tarballs.

 Command:
bazel clean --expunge && bazel build //packages/ameide_core_proto_web:npm_package //packages/ameide_sdk_ts:npm_package

Deploy-agent (Tilt/Helm)

 Do not change Tiltfile/Helm.

 Ensure the service uses the SDK/proto web via pnpm workspace (workspace:*) or pulls the Bazel-built tarballs—pick one and keep it consistent.

 Command: tilt up (same as today).

Pass/Fail acceptance

**✅ PARTIAL SUCCESS**:
- `bazel build //packages/ameide_core_proto_web:npm_package` succeeds with minimal custom rule (tree artifact for directory output)
- Generated proto JS + .d.ts present under the proto web package's Bazel outputs
- Shims import from standardized `generated/` directory
- `bazel build //packages/ameide_sdk_ts:npm_package` fails due to npm dependency resolution (typical npm_link_package complexity)

**Key Achievements:**
- Remote Buf plugins eliminate platform-specific npm binary issues
- `target=js+dts` generates runtime-ready code (no re-transpilation)
- Tree artifact rule properly outputs directory
- Clean separation: Bazel builds artifacts, Tilt/Helm deploy

www-ameide and www-ameide-platform start under Tilt unchanged.

This is the shortest path that mirrors vendor docs:

rules_js for node_modules in actions

rules_ts for TS compile

rules_buf toolchain + genrule for buf generate
No bespoke Skylark beyond that.

You said:
and what are we doing wrong here
ChatGPT said:
Thought for 25s

Here’s what’s been going wrong, point-by-point, and what to do instead:

Using non-existent Bazel rules

Wrong: Expecting @rules_buf to provide buf_image/buf_generate.

Do instead: Use the buf toolchain plus a plain genrule (or run_shell) to run buf generate exactly like the vendor docs show.

Treating npm binaries as Bazel targets

Wrong: Labels like @npm//:node_modules/@bufbuild/buf/bin/buf or plugin labels.

Do instead: Create npm_link_all_packages(name = "node_modules") in each package and rely on PATH="$RUNFILES/node_modules/.bin:$PATH" during actions.

Putting node_modules in the wrong place for tsc

Wrong: Using //:node_modules (root) and expecting tsc to see it.

Do instead: Each package that runs tsc must have a local :node_modules and include it in ts_project(data=[":node_modules","package.json", …]).

Root package.json influencing everything

Wrong: "type": "module" and devDeps at the root breaking Bazel validators.

Do instead: Keep root package.json minimal (workspace only). Let each package declare its own deps and module type.

Mixing dependency worlds for services

Wrong: Flipping between workspace:* and file:*.tgz tarballs in services/....

Do instead: Pick one: either consume workspace packages (workspace:*) or consume the Bazel-built npm tarballs—consistently.

Proto generation outside Bazel / missing edges

Wrong: Running buf via ad-hoc scripts or committing generated sources.

Do instead: Drive buf generate from Bazel (buf toolchain + genrule). Make .proto and the template inputs of that rule; make downstream targets depend on its output dir.

Re-transpiling generated JS

Wrong: Feeding protoc-gen-es output back into ts_project.

Do instead: Treat protoc-gen-es output as final JS + d.ts. Only compile your handwritten shims with ts_project; consume generated files via data.

ESM/CJS mismatch leakage

Wrong: Expecting a CJS build to import an ESM-only proto package.

Do instead: Prefer ESM-only builds for SDK/proto-web (aligns with Connect/ES). If you must ship CJS, keep the boundary clean and don’t import ESM-only code in CJS.

Using deps on ts_project like other Bazel rules

Wrong: Adding npm things to deps.

Do instead: For rules_ts, wire everything through data (node_modules, generated outputs, package.json).

Platform-specific npm binaries

Wrong: Depending on @bufbuild/buf from npm, which pulls a Darwin binary on Linux/ARM and fails in sandbox.

Do instead: Use the buf toolchain from rules_buf as the executable; only plugins (protoc-gen-*) come from node_modules/.bin.

Custom Starlark for buf

Wrong: Writing a bespoke buf_generate_dir rule.

Do instead: Keep it boring: genrule + toolchain buf + PATH to .bin. That’s the supported path.

Version drift in MODULE.bazel

Wrong: Ignoring Bazel’s “requires protobuf@X but got @Y” warnings.

Do instead: Align versions (or explicitly relax checks) so external toolchains and repos resolve cleanly.

Blurring build vs deploy

Wrong: Tweaking Tilt/Helm/K8s to chase build errors.

Do instead: Bazel builds artifacts only. Tilt/Helm untouched. Fix build in Bazel until bazel build //packages/...:npm_package is green.

If you apply those corrections, you get the vendor-blessed flow:

rules_js → npm_translate_lock once, npm_link_all_packages per package, expose .bin via data.

rules_ts → ts_project compiles only shims; generated JS consumed via data.

rules_buf → register toolchain; run buf generate in a genrule with tools = ["@rules_buf_toolchains//:buf"] and PATH extended to the local :node_modules/.bin.

That’s the smallest, most supportable shape—and it keeps Bazel strictly in the “build” lane while Tilt/Helm deploy both services.

---

## 119-auth-review.md (old)

# Authentication System Review and Fixes

## Status: TODO

## Problem Statement

The www-ameide-platform service experiences authentication failures with NextAuth v5 + Keycloak integration:
- **Error**: "state value could not be parsed" during OAuth callback
- **Impact**: Users cannot authenticate, blocking platform access
- **Root Cause**: Unstable AUTH_SECRET causing state JWE decryption failures

## The Real Issue

The error "state value could not be parsed" means **the process handling `/api/auth/callback/*` can't decrypt the `state` JWE** it issued at `/api/auth/signin`. 

### Primary Cause: Unstable AUTH_SECRET
- If `AUTH_SECRET` isn't set, Auth.js generates a random secret at boot
- Any hot reload / pod restart / scale-out means callbacks may land on a process with a *different* secret
- Result: **state cannot be decrypted**

### Secondary Issue: Cookie Security with HTTPS + Dev Mode
- In dev mode, Auth.js uses non-`Secure` cookies
- Accessing via HTTPS (`https://platform.dev.ameide.io`) expects secure cookies
- Cookies may not persist through the OAuth flow

## The Fix (Keeping NODE_ENV=development)

### 1. Make AUTH_SECRET Stable Across All Instances

**First, verify current state:**
```bash
kubectl exec -n ameide deploy/www-ameide-platform -- sh -lc 'printf "AUTH_SECRET=%s\n" "$AUTH_SECRET"'
```

**If empty or inconsistent, create and inject it:**
```bash
# Generate a 32-byte secret
SECRET=$(openssl rand -base64 32 | tr -d '\n')
kubectl -n ameide create secret generic www-ameide-auth --from-literal=AUTH_SECRET="$SECRET"

# Wire it into the deployment
kubectl -n ameide patch deploy/www-ameide-platform --type='json' -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/env/-","value":{
     "name":"AUTH_SECRET",
     "valueFrom":{"secretKeyRef":{"name":"www-ameide-auth","key":"AUTH_SECRET"}}
  }}
]'

# Restart to pick up the secret
kubectl -n ameide rollout restart deploy/www-ameide-platform
```

> With a stable `AUTH_SECRET`, you don't need sticky sessions. Any pod can decrypt the state and session.

### 2. Set Proper Environment Variables

Ensure these are set (via Helm values or patch):
```bash
AUTH_URL=https://platform.dev.ameide.io
AUTH_TRUST_HOST=true
AUTH_DEBUG=true   # temporarily, for debugging
```

**Why:**
- `AUTH_URL` lets Auth.js build correct callback/redirect URLs behind TLS termination
- `AUTH_TRUST_HOST=true` tells Auth.js to honor `X-Forwarded-*` headers from the gateway

### 3. Force Secure Cookies on HTTPS (Keep Development Mode)

**File: `app/(auth)/auth.shared.ts`**

```typescript
// Detect HTTPS even in development
const isHttps =
  (process.env.AUTH_URL ?? '').startsWith('https://') ||
  process.env.COOKIE_SECURE === 'true';

export const sharedAuthConfig = {
  // ... existing config ...
  trustHost: true,
  secret: process.env.AUTH_SECRET, // MUST be set now

  // Keep dev cookie names, but make them secure on HTTPS
  cookies: {
    sessionToken: {
      name: process.env.NODE_ENV === 'production'
        ? '__Secure-authjs.session-token'
        : 'authjs.session-token',
      options: {
        httpOnly: true,
        sameSite: 'lax',
        path: '/',
        secure: isHttps,  // <-- secure even in dev when using HTTPS
      },
    },
    // Harden the OAuth flow cookies too:
    state: {
      name: process.env.NODE_ENV === 'production'
        ? '__Secure-authjs.state'
        : 'authjs.state',
      options: { 
        httpOnly: true, 
        sameSite: 'lax', 
        path: '/', 
        secure: isHttps 
      },
    },
    nonce: {
      name: process.env.NODE_ENV === 'production'
        ? '__Secure-authjs.nonce'
        : 'authjs.nonce',
      options: { 
        httpOnly: true, 
        sameSite: 'lax', 
        path: '/', 
        secure: isHttps 
      },
    },
    pkceCodeVerifier: {
      name: process.env.NODE_ENV === 'production'
        ? '__Secure-authjs.pkce.code_verifier'
        : 'authjs.pkce.code_verifier',
      options: { 
        httpOnly: true, 
        sameSite: 'lax', 
        path: '/', 
        secure: isHttps 
      },
    },
  },
} satisfies AuthConfig;
```

> Don't set a `domain` unless you need cross-subdomain sharing. Host-only cookies are correct here.

### 4. Ensure Edge and Node Use Same Secret

**File: `app/(auth)/auth.edge.ts`**

```typescript
import NextAuth from "next-auth";
import { sharedAuthConfig } from "./auth.shared";

// No need to redefine providers/callbacks here
// Edge runtime only needs to read/verify the session
export const { auth } = NextAuth(sharedAuthConfig);
```

### 5. Gateway Headers Verification

Check that the gateway forwards headers correctly:
```bash
curl -k -sI https://platform.dev.ameide.io/ | sed -n '1,20p'
```

Should see:
- `x-forwarded-proto: https`
- `x-forwarded-host: platform.dev.ameide.io`

## Testing the Fix

### Proper OAuth Flow Test (Use POST, not GET!)

```bash
jar=/tmp/ameide.cookies
rm -f "$jar"

# Start the sign-in (POST initiates OAuth and sets authjs.state cookie)
curl -k -s -D - -c "$jar" -b "$jar" \
  -X POST "https://platform.dev.ameide.io/api/auth/signin/keycloak?callbackUrl=%2F" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data '' | sed -n '1,40p'
```

**Expected in response:**
- `HTTP/2 302`
- `Location: https://auth.dev.ameide.io/realms/ameide/...`
- `Set-Cookie: authjs.state=...; Path=/; SameSite=Lax; Secure; HttpOnly`

With stable `AUTH_SECRET` and secure cookies, the callback should work without "state value could not be parsed" errors.

## Optional Hardening

1. **Keycloak Client Configuration:**
   - Valid Redirect URIs: `https://platform.dev.ameide.io/api/auth/callback/*`
   - Web Origins: `https://platform.dev.ameide.io`

2. **Helm Values Update:**
   Add to the Helm chart values to ensure AUTH_SECRET is always set:
   ```yaml
   auth:
     secret: "{{ .Values.auth.secret | required "auth.secret is required" }}"
   ```

3. **Error Page Fix:**
   Ensure `/api/auth/error` route is properly exported and not wrapped in a throwing layout.

## Key Insights

1. **The root cause is AUTH_SECRET instability**, not the NODE_ENV setting
2. **OAuth requires POST** to `/api/auth/signin/keycloak` to properly initiate the flow
3. **Cookies can be secure in dev mode** by detecting HTTPS from AUTH_URL
4. **Don't need to switch to production mode** - just fix the secret and cookie security

## Success Criteria

- [x] AUTH_SECRET is stable across all pod instances
- [x] No "state value could not be parsed" errors in logs
- [x] OAuth flow completes with POST to signin endpoint
- [x] Sessions persist across pod restarts
- [x] Cookies are secure when using HTTPS

## Implementation Checklist

- [x] Create Kubernetes Secret for AUTH_SECRET (using Helm values)
- [x] Update deployment to use secretKeyRef
- [x] Modify auth.shared.ts to force secure cookies on HTTPS
- [x] Ensure edge config uses shared config
- [x] Test with POST to signin endpoint
- [x] Remove conflicting wellKnown/internalBase configuration
- [x] Fix Redis to point to master node
- [x] Configure Keycloak hostname settings

## References

- [NextAuth v5 OAuth Documentation](https://authjs.dev/reference/core#oauth)
- [NextAuth Cookie Configuration](https://authjs.dev/reference/core#cookies)
- [JWE State Token Explanation](https://authjs.dev/concepts/security)

## Notes

- The earlier hypothesis about "flip to production" was rejected because it's not necessary
- The real fix is ensuring AUTH_SECRET stability and proper cookie security
- GET requests to signin endpoints don't initiate OAuth (explains why cURL tests didn't set cookies)
- This approach maintains development features while fixing the authentication issues
---

## 122-hubspot.md (old)

Is it possible to export hubspot configuration 
ChatGPT said:
Thought for 1m 11s

Short answer: there isn’t a single “export everything” button for a full HubSpot configuration backup. But you can export many pieces (and copy some between portals). Here’s the practical breakdown:

CRM schema

Properties (all objects): Settings → Properties → Export all properties (CSV of internal names, field types, rules, etc.). 
HubSpot Knowledge Base

Pipelines & stages: no UI export of definitions; use the Pipelines API to pull pipeline + stage config (incl. probabilities). 
HubSpot Developers
+1

Automation

Workflows: you can export a spreadsheet view of workflows or a PNG of an individual workflows (not importable). 
HubSpot Knowledge Base

To move workflows/pipelines/config between environments, use Sandboxes (Enterprise): sync object definitions, pipelines, and workflows to a standard sandbox; some actions may be limited. 
HubSpot Knowledge Base

For cross-portal copying, Multi-Account Management (BETA) currently copies automated marketing emails, forms, and lists (definitions, not records). 
HubSpot Knowledge Base

CMS (website assets)

Themes/Templates/Modules: download files with the HubSpot CLI (hs fetch …). 
HubSpot Developers

Pages/Blogs: export all pages & blog posts as HTML from the UI. 
HubSpot Knowledge Base

Users & teams

Export a CSV of users (incl. roles, teams, last login). 
HubSpot Knowledge Base

Other handy exports

Lists (members), KB articles, URL redirects, analytics, records (contacts/companies/deals/tickets) can all be exported from their tools. 
HubSpot Knowledge Base
+2
HubSpot Knowledge Base
+2

Programmatic/backups

Use the Properties API for schema, Exports API for record data, and the Workflows API (v3/v4) to read/manage workflows where supported. 
HubSpot Developers
+2
HubSpot Developers
+2

If you tell me which parts you need (e.g., pipelines, workflows, CMS theme, users), I’ll give exact click-paths or API calls to grab them fast.

You said:
Valid for all license tiers?
ChatGPT said:
Thought for 1m 29s

Short answer: mostly yes for basic exports; some items need higher tiers.

Available on all tiers (incl. Free & Starter)

Export CRM records (contacts/companies/deals/tickets/custom objects). 
HubSpot Knowledge Base

Export all properties (schema) as CSV. 
HubSpot Knowledge Base

Export users (CSV of users/teams/roles). 
HubSpot Knowledge Base

Export website/blog HTML, KB articles, redirects, analytics (if you use those tools in your account). 
HubSpot Knowledge Base

Requires Professional or Enterprise

Workflows (and thus workflows spreadsheet/PNG exports) are only in Pro/Ent across Marketing, Sales, Service, and Operations Hubs. 
HubSpot Knowledge Base

Enterprise only

Standard Sandboxes for copying/testing config (pipelines, workflows, object defs, CMS assets). 
HubSpot

Multi-Account Management (BETA): to set up the org you need Marketing Hub Enterprise; asset copying works between Pro/Ent accounts. 
HubSpot Knowledge Base

APIs

You can use CRM/Exports/Pipelines APIs on all tiers; limits increase with higher tiers. 
HubSpot Developers
+2
HubSpot Developers
+2

CMS/Content Hub code export

The HubSpot CLI (to fetch themes/modules) is available with Content Hub (CMS) Starter+; you just need CMS access. 
HubSpot Developers
+1

If you tell me your exact tier(s), I can map which export paths/APIs you can use right now.

You said:
Which credentials do I need to access the api? 
ChatGPT said:
Thought for 1m 38s

Short version: you’ll use either a Private app access token (single HubSpot portal) or OAuth 2.0 (public/multi-portal). API Keys are sunset and no longer supported. 
HubSpot Developers

Option A — Private app (fastest for your own portal)

You need: a Private app access token created in your HubSpot portal (Super Admins can create these at Settings → Integrations → Private apps; pick scopes, then copy the token). Use it as a Bearer token. 
HubSpot Developers
HubSpot Community

Example: Authorization: Bearer <your_private_app_token> (works exactly like OAuth tokens). 
Postman

Option B — OAuth 2.0 (for apps customers install)

You need: a HubSpot developer account, your app’s Client ID, Client Secret, Redirect URI, and the scopes you’ll request. You’ll exchange an auth code for access and refresh tokens via POST /oauth/v1/token. 
HubSpot Developers
+2
HubSpot Developers
+2

Scopes & permissions (what your token can do)

Scopes control which APIs you can call. Pick scopes that match the endpoints (e.g., crm.objects.contacts.read to read contacts; for properties schema either crm.objects.{objectType}.read or crm.schemas.{objectType}.read). 
HubSpot Developers
HubSpot Community

Some APIs (e.g., Workflows) also require specific scopes like automation / workflows-access-public-api and the portal must have the right subscription. 
HubSpot Developers
HubSpot Community

A token never bypasses account/user permissions—if the user/app lacks rights you’ll see “Request for Integration Permissions.” 
HubSpot Knowledge Base

Quick “which do I pick?”

Internal scripts, server-to-server, one portal → Private app token. 
HubSpot Developers

App for multiple customers / marketplace → OAuth 2.0 (client id/secret + tokens). 
HubSpot Developers

If you tell me your use case (internal vs public, which endpoints), I’ll list the exact scopes you should grant and show ready-to-run curl examples.

You said:
Can I have an internal job / workflows that read internal crm objects and send data externally 
ChatGPT said:
Thought for 57s

Yes—natively in HubSpot you can do this. You’ve got a few solid patterns:

Workflow → “Send a webhook” action (no code).
From a workflows, push selected (or all) properties of the enrolled record to any HTTPS endpoint (POST/GET). You can test calls, add outputs to use later in the workflows, set a rate limit, and HubSpot will retry failed webhooks for up to 3 days. Requires Operations Hub Pro or Enterprise. 
HubSpot Knowledge Base

Workflow → “Custom code” action (programmable automation).
Write JavaScript (and, in beta, Python) that can read any CRM objects/associations via the HubSpot client and then call external APIs using secrets for auth. This is great when you need to look up related records, transform data, or handle complex logic before sending it out. Available in Operations Hub Pro/Enterprise. 
HubSpot Developers
HubSpot

Reusable across portals → Workflow Extensions (custom workflows actions).
If you want a branded “Send to <Your System>” action that teammates can drop into workflows, build a public app action and install it. It can read workflows inputs and send to your service. 
HubSpot Developers

Scheduling options.
You can trigger by changes to records, manual enrollment, or on a set frequency (for “nightly job” vibes). 
HubSpot Knowledge Base

If you tell me your tier and target system, I can sketch the exact steps (and a minimal payload or code snippet) for your use case.

You said:
I want something that works on all tiers and produce an abstract representation of the system in an external graph database
ChatGPT said:
Thought for 1m 5s

Absolutely—here’s a design that works on every HubSpot tier (incl. Free/Starter) and outputs an abstract graph of your HubSpot system into, say, Neo4j (or any graph DB).

1) Architecture (tier-agnostic)

A. Pull sync (simplest, scheduled)
Run a small job (cron/serverless/container) with a Private app token. It reads schema + lightweight record links via HubSpot’s public APIs and writes nodes/edges to your graph DB. Works on all tiers. 
HubSpot Developers

B. Optional near-real-time
Add webhook subscriptions (private app or public OAuth app). HubSpot posts object change events to your endpoint; you upsert the affected nodes/edges. Also works across tiers because webhooks live with your app, not your subscription level. 
HubSpot Developers
+2
HubSpot Developers
+2

2) What to export for an “abstract representation”

Model two layers:

Meta graph (configuration only)

Nodes: ObjectType (contacts, companies, deals, tickets, custom objects…), Property, AssociationType, Pipeline, Stage.

Edges:

(ObjectType)-[:HAS_PROPERTY]->(Property) from Properties API. 
HubSpot Developers
+1

(ObjectType)-[:ASSOCIATES_TO {typeId}]->(ObjectType) from Associations (v4 details or v3 schema). 
HubSpot Developers
+1

(ObjectType)-[:USES_PIPELINE]->(Pipeline)-[:HAS_STAGE]->(Stage) from Pipelines API. 
HubSpot Developers
+1

Thin data graph (optional, lightweight)

Nodes: one per record (just id, createdate, lastmodifieddate or similar).

Edges: actual associations between records (contact→company, deal→company, etc.). Pull via Objects v3 with associations or via Associations endpoints. 
HubSpot Developers
+1

3) HubSpot calls (cheat sheet)

List properties for an object: GET /crm/v3/properties/{objectType} → build (ObjectType)-[:HAS_PROPERTY]->(Property) nodes/edges. 
HubSpot Developers

List association definitions: use Associations v4 details or v3 schema to get types, category, labels. 
HubSpot Developers
+1

List pipelines/stages (deals, tickets, custom objects that support pipelines): GET /crm/v3/pipelines/{objectType}. 
HubSpot Developers

List records + associated IDs: GET /crm/v3/objects/{objectType}?properties=...&associations=companies,contacts,... and paginate with after. Use only minimal props to keep it “abstract.” 
HubSpot Developers

(Optional) Bulk export records: Exports API if you prefer file-based loads. 
HubSpot Developers
+1

Auth & scopes (private app): add read scopes for the objects you touch (e.g., crm.objects.contacts.read, crm.objects.deals.read), plus schema scopes if you’re calling schema endpoints. 
HubSpot Developers
+1

4) Cypher starter (Neo4j)

Meta (schema)

// Object types you decide to include
UNWIND $objectTypes AS o
MERGE (:ObjectType {name: o.name, label: o.label});

// Properties per object
UNWIND $properties AS p
MERGE (ot:ObjectType {name: p.objectType})
MERGE (prop:Property {objectType: p.objectType, name: p.name})
SET prop += {type:p.type, fieldType:p.fieldType, required:p.required, options:p.options}
MERGE (ot)-[:HAS_PROPERTY]->(prop);

// Association definitions
UNWIND $associations AS a
MERGE (from:ObjectType {name:a.from})
MERGE (to:ObjectType {name:a.to})
MERGE (from)-[r:ASSOCIATES_TO {typeId:a.typeId}]->(to)
SET r += {label:a.label, category:a.category, cardinality:a.cardinality};

// Pipelines & stages
UNWIND $pipelines AS pl
MERGE (ot:ObjectType {name:pl.objectType})
MERGE (p:Pipeline {id:pl.id, name:pl.label})
MERGE (ot)-[:USES_PIPELINE]->(p)
WITH p, pl
UNWIND pl.stages AS s
MERGE (st:Stage {id:s.id})
SET st += {label:s.label, order:s.displayOrder, probability:s.metadata.probability}
MERGE (p)-[:HAS_STAGE]->(st);


Thin data (IDs + links)

// Records (lightweight)
UNWIND $records AS r
MERGE (n:Record {objectType:r.objectType, id:r.id})
SET n += {createdAt:r.createdAt, updatedAt:r.updatedAt};

// Edges from association IDs
UNWIND $links AS l
MATCH (a:Record {objectType:l.fromType, id:l.fromId})
MATCH (b:Record {objectType:l.toType,   id:l.toId})
MERGE (a)-[:ASSOCIATED_WITH {typeId:l.typeId}]->(b);

5) Incremental sync pattern (works on Free)

Initial load: pull schema + first pass of IDs/associations for each object.

Deltas: poll objects changed since lastSync using updatedAt (or the object’s last modified property) and re-upsert those nodes/edges. Use pagination with after and respect rate limits. Search endpoints cap at 5 req/s, and global burst/daily limits apply (higher on Pro/Enterprise), so add retries/backoff. 
HubSpot Developers
+2
HubSpot Developers
+2

Optional: switch to webhooks to push changes as they happen; still keep a periodic reconcile job. 
HubSpot Developers

6) Why this is tier-proof

Everything above uses public CRM APIs + private app auth (or optional webhooks), which are available regardless of subscription level. You are not relying on Workflows/Programmable Automation (Pro/Ent), so Free/Starter portals work fine. 
HubSpot Developers

If you want, I can generate a tiny starter script (Node or Python) that: (1) pulls properties/associations/pipelines, and (2) emits the JSON payloads your graph loader expects.

You said:
What I cannot export
ChatGPT said:
Thought for 2m 39s

Here’s the practical answer, based on HubSpot’s current docs and what’s actually in the product.

Not natively exportable (no one-click/bulk export in the UI)

Inbox/Conversations threads & threads transcripts (bulk): no built-in bulk export; people use reports/APIs or marketplace apps. 
HubSpot Knowledge Base
+1
HubSpot Community
+1

Playbooks (content or usage analytics): no export for the content; usage data isn’t exportable as CSV from the Playbooks “Analyze” tab. 
HubSpot Community
+1

Sales email templates & snippets (bulk): no native bulk export (workarounds are manual or third-party). 
HubSpot Community
+1

Chatflows (bot) definitions/diagrams: no export of the flow; only ideas/requests exist. 
HubSpot Community
+1

Attachments on records (bulk): you can’t bulk-download all attachments tied to records; must script via Files API or download item-by-item. 
HubSpot Community
+2
HubSpot Community
+2

Call recordings/transcripts (bulk): no bulk export tool. 
HubSpot Community
+1

Workflow definitions as a portable file: you can export a PNG image or a table view (CSV), but not a re-importable JSON of the logic. 
HubSpot Knowledge Base

Exportable only in a limited way

Activities/engagements (notes, emails, calls, meetings): no global “all activities” CSV; either export per contact or use activity reports / the Engagements API. 
HubSpot Knowledge Base
+1

Ads reporting: you can export campaign and ad set level data; ad (creative/keyword) level isn’t exportable from HubSpot. 
HubSpot Knowledge Base

Social reports: HubSpot doesn’t provide a direct CSV export for the built-in Social reports (share/email/Slack only). 
Coupler.io Blog

Campaigns: you can export campaign details (list view), not all associated assets/content via a single export. 
HubSpot Knowledge Base

Marketing emails: performance data exports are supported; there’s no bulk export of the HTML content itself. (You can save emails as HubSpot templates.) 
HubSpot Knowledge Base
+1

Exportable via API (no/limited UI)

Pipelines & stages (incl. stage probabilities): use the Pipelines API. 
HubSpot Developers
+1

Association types & labels (definitions and usage): Associations v4 API. 
HubSpot Developers
+1

Lists (including filter logic): Lists v3 API can return definitions (includeFilters=true). 
HubSpot Developers

Forms (definitions & submissions): Forms API for definitions; UI supports submissions export. 
HubSpot Developers
HubSpot Knowledge Base

Definitely exportable (for completeness)

CRM records & property history: records export + single-property history export; you can also export all property definitions. 
HubSpot Knowledge Base
+2
HubSpot Knowledge Base
+2

Users (incl. teams/roles/permissions column): Users export. 
HubSpot Knowledge Base

CMS content & files: HTML of pages/blogs and a ZIP of Files; coded assets via CLI/Design Manager. (Note: exported HTML can’t be re-imported as pages.) 
HubSpot Knowledge Base
+1

If you want, I can map each of these to the APIs you’d use to build your external graph model and flag the handful that will need workarounds (e.g., Conversations, Playbooks, templates/snippets, attachments).

You said:
Yes create a table. Also consider workarounds like scraping
ChatGPT said:
Thought for 1m 46s

Here’s a compact “what you can’t (fully) export” matrix + practical workarounds (incl. scraping/automation). I kept this focused on all tiers.

Asset / Area	Native export in UI?	Public API?	Pragmatic workarounds (incl. scraping)	Notes
Conversations Inbox (threads / threads transcripts) — bulk	No single “Export all” in core UI. Some posts recommend reports or manual copy. Marketplace apps add export. 
HubSpot Community
HubSpot Ecosystem
	Yes (beta): Conversations APIs for inboxes, threads, messages. 
HubSpot Developers
	Use Conversations API to iterate threads/messages → CSV/JSON. Or install a marketplace exporter. As a last resort, run headless browser (Playwright/Puppeteer) to open each thread and capture transcript. 
HubSpot Ecosystem
	Coverage differs by channel; ensure you include attachments/metadata if needed.
Sales email templates (one-to-one)	No bulk export. 
HubSpot Community
	No official Templates API for sales templates. 
HubSpot Community
	Headless browser to open each template and save HTML/text. Some community tools/extensions exist (YMMV). 
Reddit
	Distinct from Marketing emails (which have an API).
Snippets	No native export. 
HubSpot Community
+1
	No public Snippets API. 
HubSpot Community
	Chrome extensions or headless browser to enumerate & copy content to CSV. 
snippetpowertools.com
	Keep eye on count/length limits; audit periodically.
Playbooks (content & usage analytics)	No export of content or usage CSV. 
HubSpot Community
+1
	No public Playbooks API. 
HubSpot Community
	Save answers to properties, then export those; take screenshots/PNG; scrape editor pages if you must. 
HubSpot Community
	Great candidate for “meta export” (titles, sections) via scraping for documentation.
Chatflows (bot definitions/diagrams)	No export of flow definition/diagram in core UI. (Long-standing requests.) 
HubSpot Community
	No definition export; Conversations APIs are for messages, not bot flows. 
HubSpot Community
HubSpot Developers
	Document flows via screenshots; or scrape the editor DOM to reconstruct nodes/edges into JSON.	Performance analytics & message logs can be pulled indirectly via Conversations APIs.
Sequences (definitions & perf.)	No direct export from Sequences page; perf export requests are common. 
HubSpot Community
+1
	No public API for exporting sequence definitions.	Build custom reports and export; mine Email Events APIs to reconstruct sends; scrape each step’s content with headless browser.	Treat as content inventory + schedule metadata.
All activities/engagements (global, one click)	No global “export all activities” button; exports are per record or via certain reports. 
HubSpot Knowledge Base
HubSpot Community
	Yes: Engagements APIs (calls, emails, meetings, notes, tasks). 
HubSpot Developers
	Use Engagements APIs to paginate everything to a warehouse/CSV.	Consider rate limits; plan incremental since/after.
Workflow logic as portable file	Only table view CSV and PNG of an individual workflows; not a re-importable JSON of logic. 
HubSpot Knowledge Base
HubSpot Community
xcellimark.com
	Workflows API doesn’t give you a one-click portable export for re-import.	Use CSV + PNG for documentation; optionally scrape editor to build an internal spec doc.	If you’re on Enterprise, sandboxes can copy items (not an “export”).
Call recordings / transcripts (bulk)	No bulk download knob in UI. 
HubSpot Community
	Yes: Calls + Recordings & Transcripts APIs (fetch recording/transcript via IDs). 
HubSpot Developers
+1
	Use Calls API to list call engagements, collect recording IDs/URLs, download in batch. Fallback: automate UI. 
HubSpot Developers
HubSpot Community
	Authenticated URLs and permissions required; older hs_call_recording_url is deprecated. 
HubSpot Developers

Attachments linked to CRM records (bulk)	Not as a single “download all attachments by record”; you can export all Files as a ZIP. 
HubSpot Knowledge Base
	Yes: Files API (list + download); engagements reference attachment IDs. 
HubSpot Developers
+1
HubSpot Community
	Join Engagements (for attachment IDs) → Files API to pull originals. For strict “per-record” bundles, script it; or scrape the record UI.	Files ZIP includes everything in the Files tool (not only CRM-linked).
Campaigns (asset groups)	You can export campaign details (table view), but not all linked asset contents in one export. 
HubSpot Knowledge Base
	No “export whole campaign with all assets” API.	Use APIs per asset type (emails, lists, pages, etc.); or scrape the campaign drilldowns and resolve links to assets.	Treat as an index; export child assets separately.
Scraping/automation playbook (use only when APIs/UI exports fall short)

Prefer official APIs first (stable, faster, safer). If you must scrape, use an authenticated headless browser (Playwright/Puppeteer) with throttling, honor your company’s compliance rules, and expect UI changes to break scripts. Check HubSpot’s Terms and your DPA before scraping at scale.

Typical pattern: enumerate item IDs from a list view → open detail pages → capture HTML/JSON blobs → normalize to your schema (e.g., for Playbooks/Chatflows/Sequences/Templates).

For Conversations, strongly consider the Conversations APIs or a marketplace exporter rather than raw scraping. 
HubSpot Developers
HubSpot Ecosystem

If you want, tell me which rows you care about first and I’ll sketch an export script (API-first, with a scraping fallback) and the exact fields to store for your external graph model.

You said:
 If there are marketplace exporters then it means that we can access the native apis as well think harder
ChatGPT said:
You said:
Ideally the whole exporter should be a marketplace app itself
ChatGPT said:
Thought for 3m 32s

You’re right to push here. If an exporter is listed on HubSpot’s marketplace, it’s (supposed to be) built on public APIs and scopes—not scraping or private endpoints. So let’s tighten the plan and assume we’re shipping a public marketplace app that installs on any tier, requests optional scopes so it never blocks install, and exports an abstract model of the portal into your graph DB.

Below is a pragmatic coverage matrix (updated with the newest APIs), then how to package this as a marketplace app and map it to a graph.

What you can (and can’t) export — with native APIs & marketplace-safe workarounds
Area	API coverage	Key endpoints	Typical scopes (public app)	Tier notes	Marketplace-compliant workaround	Notes / sources
CRM objects (contacts, companies, deals, tickets, custom objects), properties, associations	Full (read)	CRM Objects v3; Properties v3; Associations v4	crm.objects.*.read, crm.schemas.*.read (for custom objects/properties), crm.associations.read	Works on all tiers (objects vary by hub/seat).	Use Exports API for bulk CSVs when you want chunked, resumable dumps.	Objects & props: official refs. Associations v4 incl. labels/custom types. Exports API for bulk. 
HubSpot Developers
+4
HubSpot Developers
+4
HubSpot Developers
+4

Pipelines & stages (deals, tickets, custom)	Full (read)	Pipelines API	crm.pipelines.read	All tiers that have that object.	—	Docs: Pipelines in CRM APIs. 
HubSpot Developers

Users / Owners / Teams	Partial	Owners v3, User Provisioning (SCIM)	crm.objects.owners.read (+ provisioning scopes if you use SCIM)	All tiers (owners). Provisioning is enterprise IT use case.	—	Owners v2 → v3 migration notice; provisioning doc. 
HubSpot Developers
HubSpot Community

Marketing emails (definitions & content)	Good	Marketing Email API	marketing-email-read	Requires Marketing Hub. Mark as optional scope.	Export via UI if scope unavailable.	Email API ref. 
HubSpot Legal

Forms (definitions; submissions)	Good	Forms v3 (defs), Forms v2 submissions	forms (and related)	All tiers that have Forms.	—	v3 for forms, v2 for submissions noted by HubSpot/community. 
HubSpot Developers
+1
HubSpot Community

Campaigns	Good (improving)	Campaigns v3	marketing.campaigns.read	Marketing Hub only → optional scope.	—	July 2025 updates incl. more asset types & business units. 
HubSpot Developers
+1

Workflows	Good (v3)	Automation / Workflows v3	automation	Pro+ hubs have workflows → optional.	UI export for lists/PNGs if API unsuitable.	Workflows API and changelog. 
HubSpot Developers
+2
HubSpot Developers
+2

Sequences (defs & enrollments)	Now available (v4 / Beta)	Sequences v4	automation.sequences.read, automation.sequences.enrollments.write (if you enroll)	Sales/Service Pro+ → optional.	—	Newer API; see docs & Postman notes on scopes expansion. 
HubSpot Developers
Postman

Conversations (inboxes, threads, messages)	Good (Beta)	Conversations v3	conversations.read (and .write only if sending)	Service/Conversations → optional.	—	Inbox/threads/messages APIs & Postman collection. 
HubSpot Developers
Postman

Files / attachments	Good	Files v3	files.read	All tiers	Export all via UI ZIP as fallback.	Files API + KB for “Export all files”. 
HubSpot Developers
HubSpot Knowledge Base

Analytics / reporting data	Partial	Reporting/Analytics API	analytics.reports.read (varies)	All tiers	Dashboard/report visuals export via UI email; no full dashboard JSON.	Reporting API + KB. 
HubSpot Developers
HubSpot Knowledge Base

Knowledge Base articles	No public CRUD	—	—	Service Hub feature	UI export of KB articles (CSV/XLS/XLSX). Public site crawl is ok (your domain).	Community & KB show no public KB API; UI export exists. 
HubSpot Legal
HubSpot Knowledge Base

CTAs (definitions)	No CRUD (widget SDK only)	CTA Widget SDK	—	Marketing/Content Hub	Capture performance via reporting; defs via UI.	SDK is runtime, not config export. 
HubSpot Developers

Playbooks content	No public API	—	—	Sales/Service Hub	Export via UI or include only references (IDs) if surfaced via Campaigns.	No content API; campaigns may reference assets but not return body. 
HubSpot Developers

Sales Email Templates & Snippets	No public API	—	—	Sales Hub	Mirror via Sequences where referenced; else manual export.	Community confirms lack of public endpoints. 
HubSpot Developers
HubSpot Community

Chatflows (bot definitions)	No public API (config)	—	—	All	Use Conversations APIs for messages, not bot configs.	Conversations API ≠ threadsflow config. 
HubSpot Developers

Why this “works on all tiers”: make every non-core scope optional. HubSpot will drop optional scopes that the installing portal doesn’t have, so the app installs everywhere and gracefully narrows features per portal. 
HubSpot Community

“Scraping” vs marketplace rules (important)

Do not scrape HubSpot’s app UI or hit private/undocumented endpoints in a marketplace app. That violates HubSpot’s Developer Terms and Acceptable Use (automated access/robots), and won’t pass listing/certification. Use only public APIs + user-initiated UI exports. 
HubSpot Legal
HubSpot

Allowed alternatives: exporting via official Exports API, built-in UI exports (files, workflows PNG, KB CSV), or crawling your public website to re-harvest published KB pages if you want content parity that APIs don’t provide. 
HubSpot Developers
HubSpot Knowledge Base

How we’d package this as a public marketplace app

Auth & install

OAuth 2.0 public app (client ID/secret, redirect URI). Request a minimal required set (oauth, crm.objects.owners.read) and put the rest as optional so install never fails on lower tiers. HubSpot’s scopes model and install flow are documented here. 
HubSpot Developers
+1

Scopes to include (suggested)

Core graph model: crm.objects.contacts.read, companies.read, deals.read, tickets.read, custom.read, crm.schemas.*.read, crm.associations.read, crm.pipelines.read.

Marketing (optional): marketing-email-read, forms, marketing.campaigns.read.

Automation (optional): automation (workflows), automation.sequences.read (+ .enrollments.write only if you actually enroll). 
HubSpot Developers
+1
Postman

Service/conversations (optional): conversations.read. 
HubSpot Developers

Files (optional): files.read. 
HubSpot Developers

Data ingestion strategy

Initial full sync via paged CRM reads (and Exports API for heavy tables), then delta via updatedAt cursors + Webhooks for near-real-time updates (CRM object changes & conversation events). 
HubSpot Developers
+1

Rate limits/backoff: Honor HubSpot per-app limits; batch where available and retry with jitter per usage guidelines. 
HubSpot Developers

Graph model (abstract representation)

Nodes: each CRM object type (standard + custom), Users/Owners, Campaigns, Emails (marketing), Threads/Messages (conversations), Files.

Edges: Associations v4 types (use labels as edge types), “SENT_EMAIL”, “OPENED”, “REPLIED”, “BELONGS_TO_PIPELINE”, etc. Persist HubSpot IDs as external keys. 
HubSpot Developers

Orchestration

Installer UI → OAuth → write an install record (portalId, granted scopes).

Sync planner schedules object families by priority and size, then hydrates associations last to maintain referential integrity.

Checksum content (e.g., email HTML) to avoid redundant writes to the graph.

Optional UX

A tiny CRM UI extension “Export status” card (no extra scopes needed) to show sync health per portal. 
HubSpot Developers

Security & compliance

Encrypt tokens at rest, rotate refresh tokens, and document data flow for GDPR/DSARs. Respect data residency expectations when writing to your graph DB.

Quick answers to your two points

“If marketplace exporters exist, we can access native APIs, think harder.”
Agreed—and we’ve expanded coverage accordingly: Workflows v3, Sequences v4, Conversations v3, Campaigns v3 all unlock more config export than a couple of years ago. Where APIs still don’t exist (Playbooks, Sales Templates/Snippets, Chatflow definitions, KB CRUD), we rely on UI exports or published-site crawling (KB) rather than scraping HubSpot. 
HubSpot Developers
+3
HubSpot Developers
+3
HubSpot Developers
+3
HubSpot Knowledge Base

“Ideally the exporter is itself a marketplace app.”
Perfect. Build a public app with optional scopes so it installs on all tiers and progressively enables features based on what the portal has. Use Webhooks for change data capture and stream your abstract graph (nodes = objects, edges = association labels + activities) into your graph database. 
HubSpot Community
HubSpot Developers

If you want, I can draft the exact scope manifest (required vs optional), a first-pass graph schema (node/edge types + properties), and a sync plan (per endpoint, pagination, webhooks coverage) based on the specific graph DB you’re using.

You said:
Explain the anatomy of a marketplace app. How the app is deployed , where it runs, access to hubspot platform primitives (only via api rest? ) it would be a connector to an external system. What would be validated by hubspot
ChatGPT said:
Request for GPT-5 Pro
Reasoned for 3m 39s

Great question. Here’s a clear, up‑to‑date picture of how a HubSpot Marketplace (public) app is built, deployed, and validated, with special notes for “connector” style integrations.

1) What a “Marketplace app” actually is

At its core, a Marketplace app is:

A public OAuth app you register in a HubSpot developer account (client ID/secret, redirect URIs, scopes). Users install it into their portals via the OAuth flow. 
HubSpot Developers
+1

Your runtime, hosted wherever you want (AWS, GCP, Azure, etc.), that:

Receives OAuth callbacks and stores tokens.

Calls HubSpot public APIs.

Optionally exposes endpoints for Webhooks (HubSpot → you) and Workflow custom actions (HubSpot calls you during automation). 
HubSpot Developers
+1

Optional UI inside HubSpot (for a better UX):

UI Extensions (React “App Cards”) rendered in HubSpot’s CRM UI; you can fetch data via your own APIs or HubSpot serverless functions bound to a project. HubSpot is sunsetting classic iFrame cards in favor of these React cards. 
HubSpot Developers
+2
HubSpot Developers
+2

Takeaway: HubSpot does not host your integration’s main backend. You host it. HubSpot hosts only the optional UI extension bundle and (if you use them) serverless functions tied to a developer project. 
HubSpot Developers
+2
HubSpot Developers
+2

2) Where each part runs
Piece	Where it runs	What it does
OAuth app (public)	HubSpot manages app registration; your backend handles the OAuth handshake & token storage.	Installation, auth, token lifecycle. 
HubSpot Developers
+1

Your connector backend	Your infra (e.g., AWS Lambda/ECS).	Reads/writes HubSpot via APIs; talks to the external system; processes webhooks; runs schedulers.
Webhooks endpoint	Your infra.	Receives change events (contacts, companies, deals, etc.) from HubSpot. 
HubSpot Developers

Workflow custom actions	Your infra.	HubSpot Workflow calls your endpoint with execution payloads you define. 
HubSpot Developers
+1

UI Extensions (React)	Rendered inside HubSpot; built & deployed via a developer project.	App “cards,” custom tabs/panels on record pages; calls your APIs or HubSpot serverless functions. 
HubSpot Developers
+1

Serverless functions (optional)	Hosted by HubSpot; deployed with the project.	Small Node.js handlers to fetch or proxy data for extensions, without exposing secrets to the browser. 
HubSpot Developers
+1
3) How the app is deployed (end‑to‑end)

Register a public app in your developer account

Name, logo, redirect URIs, required + optional scopes (HubSpot now supports advanced auth settings and truly optional scopes). 
HubSpot Developers
+1

Build your backend

Implement OAuth code→tokens, secure token storage/rotation, your sync logic, and endpoints for webhooks and (optionally) workflows custom actions. 
HubSpot Developers
+1

(Optional) Build UI Extensions

Create a developer project, add your React extension(s), and (if needed) serverless functions; deploy with the CLI (hs project upload). Link the project to your public app. 
HubSpot Developers
+1

App listing & review

Create a Marketplace listing and provide testing instructions for reviewers. After approval, your listing is discoverable; you can later apply for Certification. 
HubSpot Developers

4) How you access HubSpot “platform primitives”

Primary interface: Public REST APIs + events. There’s also GraphQL in specific contexts.

A. REST APIs (read/write):

CRM objects (contacts, companies, deals, tickets, custom objects), properties, pipelines, associations, files, owners, etc. 
HubSpot Developers
+2
HubSpot Developers
+2

Automation & engagement surfaces: Workflows API (definitions/inspection), custom workflows actions (HubSpot calls your API), Sequences (where available), Communications/Meetings, Marketing Emails/Forms/Campaigns, Conversations (inbox/threads/messages). 
HubSpot Developers
+2
HubSpot Developers
+2

Bulk & high‑volume tools: Imports API (fast backfills), batch/async endpoints, Associations v4 for labeled relationships. 
HubSpot Developers
+1

Analytics & events: Web analytics/events and custom/timeline events you can define and attach to records. 
HubSpot Developers
+2
HubSpot Developers
+2

B. Eventing:

Webhooks v3 for data‑change notifications; your app subscribes per object/event type. 
HubSpot Developers

C. In‑product UX surfaces:

UI Extensions (React “App Cards”) with an SDK to access record context and call serverless functions/your APIs; classic CRM cards are being sunset (build new on UI Extensions). 
HubSpot Developers
+1

D. GraphQL (collector):

HubSpot also offers a GraphQL endpoint primarily for CMS and UI Extensions scenarios, with dedicated scopes (collector.graphql_*). For broad “connector” data sync, you’ll still rely mostly on REST + webhooks. 
HubSpot Developers
+1
Postman

E. Limits you must design around:

Public‑app burst limit ≈ 110 requests / 10s; Search API ~ 5 req/s/token; use batch endpoints & backoff. (HubSpot increased several limits in late 2024/2025.) 
HubSpot Developers
+1

5) Connector blueprint (HubSpot ↔ External system)

Install & auth: User installs via OAuth → you store access/refresh tokens. 
HubSpot Developers

Data in (HubSpot → External):

Subscribe to webhooks for the objects you care about (e.g., contact.created/updated).

Resolve associations, enrich, and push to the external API. Use a queue to handle spikes and rate limits. 
HubSpot Developers

Data out (External → HubSpot):

On external events, call HubSpot REST (batch where possible) or the Imports API for big backfills. Set associations with Associations v4. 
HubSpot Developers
+1

In‑HubSpot UX:

UI Extension card on contact/company/deal pages showing external data and actions; call your backend or a HubSpot serverless function to fetch/act. 
HubSpot Developers
+1

Automation hooks:

Provide custom workflows actions (“Send to External”, “Upsert in External”), so admins can push data in real time inside HubSpot automation. 
HubSpot Developers

Advanced telemetry:

Emit Timeline or Custom Events to decorate CRM timelines with external activity. 
HubSpot Developers
+1

6) What HubSpot reviews/validates

There are two levels: listing review (to be in the marketplace) and app certification (optional, higher bar that earns a “Certified” badge).

A) Listing requirements (baseline)

You must own the integration, provide support & documentation, have minimum active installs (3) to list, and agree to Platform Policy/Agreements. 
HubSpot

Your listing must meet content rules and provide test steps so reviewers can validate the install & basic functionality. 
HubSpot Developers

B) Certification requirements (higher trust badge)

HubSpot evaluates security & privacy, reliability & performance, usability & accessibility, and value. (You need ≥60 active installs and typically ≥6 months listed before you can apply.) 
HubSpot Academy
HubSpot Developers

They check scope hygiene (request only what you use). The review team will compare your requested OAuth scopes with what your app actually calls and ask you to reduce scopes if they’re not used. 
HubSpot Developers

Expect scrutiny of rate‑limit handling and error behavior (e.g., exponential backoff, use of batch endpoints)—this is a common theme in developer communications. 
HubSpot Developers
HubSpot Community

You must comply with Acceptable Use and Developer Policy (e.g., no scraping of HubSpot’s UI/private endpoints; protect customer data; follow data‑deletion practices). 
HubSpot Legal
+1

C) Lifecycle & uninstall

You’re responsible for handling uninstall/deauthorization: stop processing, revoke tokens, delete customer data as promised. HubSpot has introduced an Uninstall API for public apps to programmatically remove installs. 
HubSpot Developers
+1

7) Is it “only REST”? (Quick verdict)

For connectors (bidirectional sync, bulk movement, orchestration): Yes—primarily REST, plus Webhooks and Workflow custom actions. That’s the stable, well‑documented surface. 
HubSpot Developers
+2
HubSpot Developers
+2

GraphQL exists but is aimed at CMS/Content and UI Extension scenarios (it requires specific collector scopes). Treat it as a convenience for in‑HubSpot UIs, not your main sync rail. 
HubSpot Developers
Postman

8) Practical “first draft” of your connector app

Scopes: keep core read/write scopes minimal; mark everything else optional so the app installs on all tiers (HubSpot supports optional scopes). 
HubSpot Developers

Ingest: initial backfill via Search + Batch or Imports API; then near‑real‑time via Webhooks. 
HubSpot Developers

Write‑back: batch updates to HubSpot and your external system; model relationships with Associations v4. 
HubSpot Developers

UX: add a UI Extension card to show external status and manual actions. 
HubSpot Developers

Automation: ship a custom Workflow action (“Send to External”) so admins can push data without leaving HubSpot. 
HubSpot Developers

Ops & limits: respect the 110/10s public‑app burst limit and the Search API rate; implement retries & backoff; prefer batch endpoints. 
HubSpot Developers
+1

Compliance: document data usage, retention, and deletion on uninstall; keep scopes to least privilege; follow AUP/Developer Policy. 
HubSpot Legal
+1

Want me to sketch a reference architecture diagram (components, queues, endpoints) and a scope manifest tailored to your target external system? I can draft it directly around your external API’s auth model and objects.
You said:
Yes also consider ameide as the target enterprise architecture platform and hubspot as a source of both architecture components ( all hubspot internals to be published) and process events for process mining. The hubspot app can be a pod in ameide k8s . 
ChatGPT said:
You said:
Continue
ChatGPT said:
Request for GPT-5 Pro
Reasoned for 4m 18s

Great—let’s turn this into a concrete, production‑ready blueprint that treats HubSpot as a source of both enterprise‑architecture components and process‑mining events, and deploys the HubSpot exporter as a Marketplace app that runs as a pod inside Ameide’s Kubernetes.

1) High‑level architecture (runtime inside Ameide)
+--------------------+           +---------------------+         +----------------------+
|  HubSpot Account   |           |  Public App (OAuth) |         |     Ameide K8s       |
|  (any tier)        |  <--->    |  + Webhooks         |  --->   |  Namespace: hs-xport |
|                    |           |  + Redirect URL     |         |  + Ingress (TLS)     |
|  CRM objects,      |           |                     |         |  + Deployment        |
|  pipelines, etc.   |  [HTTPS]  |  (HubSpot pushes    |  [HTTPS]|    - api (ingress)  |
|                    |  events   |   & calls)          |         |    - worker(s)       |
+--------------------+           +---------------------+         |  + Jobs/CronJobs     |
                                                                |  + Secrets/ConfigMap |
                                                                |  + Queue (Kafka/NATS)|
                                                                |  + Cache (Redis)     |
                                                                |  + Timeseries (optional)
                                                                +----------+-----------+
                                                                           |
                                                                           v
                                                     +-----------------------------+
                                                     | Ameide Graph Service        |
                                                     | (e.g., Neo4j/JanusGraph)    |
                                                     |  - schema registry          |
                                                     |  - upsert/merge API         |
                                                     +---------------+-------------+
                                                                     |
                                                                     v
                                                     +-----------------------------+
                                                     | Ameide Process-Mining Store |
                                                     | (events parquet/lakehouse)  |
                                                     +-----------------------------+


Flow in short

Install: Customer installs your public app (OAuth), granting scopes. App registers webhook subscriptions and sets up initial sync schedule. 
HubSpot Developers
+1

Backfill: Pod runs backfill jobs using CRM REST (and GraphQL collector where helpful) with paging & batching. 
HubSpot Developers
+1
Postman

Delta: Real‑time webhooks for CRM CRUD, property changes, association changes, merges/restores, and Conversations events; periodic pulls for email & analytics events. 
HubSpot Developers
+1
Postman

Projection: Workers translate HubSpot concepts → Ameide’s graph schema (components/relations) and event taxonomy (for process mining).

Control plane: a minimal UI Extension card in HubSpot can show sync health/status and link to Ameide. (Optional, but nice.) 
HubSpot Developers
+1

2) Installation, auth, scopes & rate‑limit posture

Public app OAuth: Use the HubSpot OAuth flow; configure required + optional scopes (HubSpot supports optional scopes for public apps). Validate requests with X‑HubSpot‑Signature‑v3 on webhook deliveries. 
HubSpot Developers
+2
HubSpot Developers
+2

Where the app runs: the actual compute runs in Ameide (your pod). Only the OAuth redirect and webhook receivers must be publicly reachable (via your Ingress/TLS).

Rate limits (design for never dropping data):

Public apps: ~110 requests / 10s per account (burst). Search endpoints are 5 req/s per token, 200 records/page. Build an adaptive token bucket + backoff. 
HubSpot Developers
+1

K8s secrets you’ll need

HUBSPOT_CLIENT_ID, HUBSPOT_CLIENT_SECRET, OAUTH_SCOPES, REDIRECT_URI

WEBHOOK_SECRET (for v3 signature verification) 
HubSpot Developers

GRAPH_DB_URI, GRAPH_DB_USER/PASS, QUEUE_BROKER, REDIS_URL

3) Data model: HubSpot → Ameide Graph

Map every “internal” HubSpot primitive to a small, stable set of node & edge types. You can extend later without breaking queries.

HubSpot source	API (primary)	Graph node / edge	Notes
Object Types (contacts, companies, deals, tickets, custom objects, products, line items)	CRM Objects v3	:Object node {type, id, name, ...}	Use GraphQL for wide fan‑out lookups. 
HubSpot Developers
+1
Postman

Properties (schema)	CRM Properties v3	:Property node, (:Object)-[:HAS_PROPERTY]->(:Property)	Carry datatype, validations, options. 
HubSpot Developers
+1

Associations + Labels	Associations v4	(:Object)-[:ASSOCIATED {typeId,label}]->(:Object)	Labels are first‑class. 
HubSpot Developers
+1

Pipelines & Stages (deals/tickets)	Pipelines v3	:Pipeline, :Stage, (:Pipeline)-[:HAS_STAGE]->(:Stage), (:Object)-[:IN_STAGE]->(:Stage)	Stage changes feed events. 
HubSpot Developers
+1

Workflows	Automation v3/v4	:Workflow, :Action nodes; (:Workflow)-[:CONTAINS]->(:Action)	Use v4 where available. 
HubSpot Developers
+1

Sequences	Sequences v4	:Sequence, :SequenceStep	Enrollment events feed process stream. 
HubSpot Developers

Forms	Forms v3	:Form	Submissions become events. 
HubSpot Developers

Campaigns	Campaigns v3	:Campaign	Marketing structure & influence. 
HubSpot Developers

Marketing Emails	Marketing Emails v3	:MarketingEmail	Events via Email Events API. 
HubSpot Developers
+1

Conversations (inboxes, threads, messages)	Conversations v3	:Inbox, :Thread, :Message	New message webhooks exist. 
HubSpot Developers
Postman

Owners/Teams	Owners	:User/:Team	For actor attribution. 
HubSpot Developers
4) Event taxonomy for process mining

Define a single normalized envelope:

{
  "ts": "...", "objectType": "deal", "objectId": "123",
  "eventType": "deal.stage_changed",
  "actor": {"type": "user|system|integration", "id": "..."},
  "source": "hubspot",
  "attributes": {...}  // old/new values, labels, channel, campaign, etc.
}


Event families & how to capture

CRUD & property changes – subscribe to CRM object events via Webhooks (create/update/delete/restore/merge). Include property deltas; for historical rebuilds, use propertiesWithHistory on reads. 
HubSpot Developers
+1
HubSpot Community

Association add/remove + label changes – associationChange webhook + Associations v4 to resolve details/labels. 
HubSpot Developers
+1

Pipeline stage transitions – stage changes appear as property change (dealstage, hs_pipeline_stage). Backfill & validate with property history. 
HubSpot Developers
HubSpot Community

Workflow lifecycle – enrollment/exit can be inferred via Workflows API and/or custom workflows actions that post to your pod on key steps. (HubSpot also supports generic workflows webhooks.) 
HubSpot Developers
knowledge.hubspot.com

Sequences – enrollments and step outcomes via Sequences v4 + email event joins. 
HubSpot Developers

Marketing email events – sends/opens/clicks/bounces via Email Events / Email analytics APIs (time‑window pulls). 
HubSpot Developers

Forms – submissions via Forms API (or a workflows webhook on submission for push). 
HubSpot Developers
knowledge.hubspot.com

Conversations – thread created, assignment/status changes, new messages via Conversations webhooks; content via API. 
Postman

Web analytics – page views & event analytics via Event Analytics API; add Custom Behavioral Events if you want your own semantics. 
HubSpot Developers
+1

5) Sync strategy (backfill + change data capture)

Backfill (idempotent, resumable):

Pull by object in pages of 200, breadth‑first by create date; hydrate associations & owners last. For wide traversals, prefer GraphQL collector to reduce call count. 
HubSpot Developers
Postman

For property histories you need for process baselines (e.g., first‑touch stage), pull with propertiesWithHistory for the small set you care about. 
HubSpot Community

Where exports are much faster/lighter (e.g., one‑off audits), consider the Exports API and load from the returned URL. 
HubSpot Developers

CDC / streaming:

Webhooks drive near‑real‑time changes (batch deliveries, signed with v3); keep an ack queue and dead‑letter to never lose updates. 
HubSpot Developers
+1

For Email Events / Analytics, run incremental time‑window pulls (e.g., every 5 minutes; widen windows to cover late arrivals). 
HubSpot Developers

Rate‑limit safety:

Token‑bucket per account: 110/10s burst; Search at ≤5 r/s; prefer batch & GraphQL joins to reduce calls. 
HubSpot Developers
+1

6) Pod design inside Ameide K8s

Containers (one Deployment, two processes is fine; or split):

api

Serves /oauth/callback, /webhooks, healthz

Verifies X‑HubSpot‑Signature‑v3 and enqueues work items. 
HubSpot Developers

worker

Backfill jobs, retry queue, batch reads, graph upserts, event emission

Periodic “pullers” (email events, analytics)

K8s objects (sketch)

apiVersion: apps/v1
kind: Deployment
metadata: {name: hs-exporter}
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: api
        image: ameide/hs-exporter:latest
        envFrom: [{secretRef: {name: hs-exporter-secrets}}]
        ports: [{containerPort: 8080}]
      - name: worker
        image: ameide/hs-exporter:latest
        args: ["worker"]
        envFrom: [{secretRef: {name: hs-exporter-secrets}}]
---
kind: Service
spec: {type: ClusterIP, ports: [{port: 80, targetPort: 8080}]}
---
kind: Ingress
spec:
  tls: [...]
  rules:
  - host: hubspot-exporter.ameide.example
    http:
      paths:
      - path: /webhooks
      - path: /oauth/callback


Persistence & resilience

Queue (Kafka/NATS/Rabbit) for webhook bursts; Redis for cursors; object storage for backfill checkpoints; idempotency keys (composed of appId+portalId+objectId+eventId).

Observability: per‑account metrics (latency, backlog, 429s, drops). Check HubSpot’s API usage logs when debugging customer installs. 
HubSpot Developers

7) Building as a Marketplace app (and what HubSpot validates)

Listing requirements & validation: correct links, HTTPS, status‑200, name rules, shared data table, etc., are now actively validated during listing. Provide test credentials. 
HubSpot Developers
+2
HubSpot Developers
+2

App certification (optional but recommended): minimum 60 active installs, time‑on‑market, plus security, privacy, reliability, performance, usability, accessibility checks. 
HubSpot Developers
+2
HubSpot Developers
+2

Uninstall API: new public beta endpoint to programmatically uninstall the app from a customer’s portal—use for cleanup flows and to respect customer off‑boarding. 
HubSpot Developers
+1

Access to platform primitives

Your app interacts with HubSpot via REST APIs, Webhooks, Workflow extensions, UI Extensions (React), and optionally GraphQL collector (for efficient reads). You don’t run code inside HubSpot except for serverless functions if you choose their project-based model (not required here since we run in Ameide). 
HubSpot Developers
+2
HubSpot Developers
+2

8) Event & component coverage vs. “works on all tiers”

Design for graceful degradation:

Public app + CRM core: works on Free/Starter—you can read core objects/properties/associations and receive webhooks. 
HubSpot Community

Marketing (emails, campaigns), Sequences, Conversations: some endpoints are product‑gated; your app should detect missing scopes or 403s and mark those streams “inactive.” (Campaigns v3 & Sequences v4 document supported products; Conversations APIs are in beta.) 
HubSpot Developers
+2
HubSpot Developers
+2

9) Minimal UI in HubSpot (optional but helpful)

A UI Extension card on contact/company/deal to show “Ameide Sync Status”, last sync, object URL in Ameide, and a “Resync” button that hits your pod. Build with React SDK; when you need HubSpot data inline, query via /collector/graphql (with the proper scopes). 
HubSpot Developers
+1

10) Security model

Verify every webhook with Signature v3; reject >5‑minute skew. Store per‑portal OAuth tokens (encrypted). 
HubSpot Developers

Scope‑minimize: split required (crm.objects., crm.schemas.read, crm.associations.read, webhooks) vs optional (campaigns.read, marketing‑emails.read, conversations.read, sequences.read, event‑analytics.read, collector.graphql_). The install dialog will show both. 
HubSpot Developers

Protect PII in logs; hash emails in event streams that don’t need them.

11) “What’s hard / edge cases” & our answers

Association churn: use the associationChange webhook to emit precise add/remove events; reconcile with Associations v4 when batches arrive out of order. 
HubSpot Developers

Stage backfills: for historic dealstage timelines, pull property history (selected properties only) rather than brute‑force rescan. 
HubSpot Community

Email & analytics late arrivals: keep rolling windows (e.g., T‑24h, T‑7d) to “heal” gaps; dedupe on eventId. 
HubSpot Developers

Search call limits: use GraphQL queries (contacts/companies + nested associations) to collapse N+1 REST hops. 
HubSpot Developers

12) Deliverables checklist (what we’d implement)

HubSpot Public App: scopes, redirect, webhook subscriptions for objects + associations + conversations. 
HubSpot Developers

Ameide K8s pod: API + workers, Ingress, Secrets, HPA, dashboards.

Graph schema in Ameide: nodes/edges above + upsert semantics.

Event schema + lakehouse tables/partitions by date, portalId, objectType, eventType.

Backfill runners (per object) + CDC consumers.

UI Extension (optional) for visibility. 
HubSpot Developers

Marketplace listing pack (copies, shared data table, privacy/security notes, support contact) & certification plan. 
HubSpot Developers

13) Concrete next steps (zero‑risk pilot)

Spin up an internal (unlisted) public app in a dev portal; enable Webhooks for contacts/companies/deals/tickets + associationChange; set redirect/webhook URLs to the Ameide Ingress. 
HubSpot Developers
+1

Deploy the pod (one replica is fine to start) with a /healthz and /installed endpoint; add dashboards for 429s and webhook lag.

Run a backfill of a single object (e.g., companies) into your graph; turn on webhooks; compare graph vs. HubSpot for 48 hours.

Add one event family (deal stage changes) to the process‑mining store and verify traces.

If you want, I can turn this into a ready‑to‑run starter (folder layout, queue schema, and sample GraphQL queries) in the next message.

You said:
Take in consideration: multitenancy, internal ameide graph standardized on archimate ontology, process mining events near real time actions to be ingested (not the same architecture batch/rest approach) 
ChatGPT said:
Request for GPT-5 Pro
Reasoned for 2m 54s

Perfect—let’s evolve the design around your three constraints:

Multi‑tenancy (one app serving many HubSpot portals)

Ameide’s EA graph standardized on ArchiMate

Process‑mining events as near‑real‑time streams (not batch/pull)

Below I separate control‑plane (schema & backfills) from the data‑plane (event streams), show the tenant model, and give an ArchiMate mapping + event taxonomy that Ameide can ingest with low latency.

A) Multitenant runtime (HubSpot → Ameide)

Tenant = HubSpot portal (portalId). Every install creates a tenant record and an isolated data plane.

Control‑plane components

OAuth handler (per tenant) stores access/refresh tokens (encrypted).

Subscription manager ensures the app has webhook subscriptions for objects you care about (create/update/delete/merge/restore/associationChange). Use the Webhooks API to manage these for the public app. 
HubSpot Developers

Initial backfill (only once per stream) is done with REST (or GraphQL where it collapses multiple reads). This is control‑plane only; the ongoing feed is event‑driven. 
HubSpot Developers
+1

Data‑plane components (real‑time)

Webhook ingress (public HTTPS in Ameide K8s). Validate every delivery with X‑HubSpot‑Signature‑V3 and a shared secret. 
HubSpot Developers

Event queue (Kafka/NATS) with per‑tenant topics (e.g., hs.{portalId}.crm and hs.{portalId}.conv).

Normalizer (stateless) → emits a canonical envelope (see §D) and dedupes by (portalId, eventId).

Projectors:

EA projector → upserts ArchiMate nodes/edges in Ameide’s graph.

Process‑mining projector → writes case events (near‑real‑time) to Ameide’s process store.

Rate‑limit fairness, per tenant
Budget requests per installed account. HubSpot’s limit for public apps is ~110 requests / 10 seconds per installed account (Search endpoints are separately throttled ~5 req/s per token), so maintain a token bucket per tenant and prefer batch/GraphQL to avoid N+1. 
HubSpot Developers
+1

Why this works for “all tiers”: the real‑time feed relies on webhooks (available to public apps) rather than paid workflows actions; polling is only for the initial backfill or the few surfaces that don’t emit events. 
HubSpot Developers

B) Deployment in Ameide K8s (pod pattern)

One deployment, many tenants

api container: /oauth/callback, /webhooks, health checks.

worker container: consumes the queue, calls HubSpot only when needed (e.g., to hydrate association labels), writes to Ameide graph + process store.

Tenancy isolation

Separate topics (and consumer groups) per portalId.

In the graph, either separate databases (if your graph supports multi‑DB) or a tenant key on every node/edge with enforced row‑level security in Ameide’s graph service.

Secrets are namespaced per tenant (tokens, webhook secret).

On uninstall/revocation, stop consumers and purge secrets; HubSpot includes uninstall/deauthorization flows you should handle. 
HubSpot Developers

C) Event‑driven data‑plane (near real time)

What we subscribe to (public app webhooks)

CRUD & propertyChange for CRM objects (contacts, companies, deals, tickets, products, line_items, etc.).

merge and restore events.

associationChange events (add/remove; contact↔company, deal↔company, etc.). 
HubSpot Developers

Where applicable, Conversations (inboxes/threads/messages) via the newer conversations event feeds and APIs. (Stay within documented public surfaces.) 
HubSpot Developers

What the payload gives you

For generic subscriptions, HubSpot includes the objectTypeId and the changed property (for propertyChange) so you can route without extra reads. 
HubSpot Developers
+1

When do we still call REST?

Only to enrich an event (e.g., resolve association labels/types or fetch the minimal property set you standardized), and only if the webhook payload didn’t include it. Use batch endpoints/GraphQL to minimize calls. 
HubSpot Developers

Exactly‑once-ish semantics

Idempotency key = portalId + subscriptionId + eventId (or HubSpot’s event timestamp + object id for older events).

Store last processed time per stream for observability; keep a DLQ for poison events.

D) Canonical process‑mining envelope (streamed, not batch)

This replaces “batch REST” for process mining:

{
  "ts": "2025-08-16T12:34:56Z",
  "tenant": "portalId-12345",
  "objectTypeId": "0-3",          // e.g., deal
  "objectId": "987654321",
  "eventType": "deal.stage_changed",
  "actor": {"type": "user|system|integration", "id": "ownerId"},
  "attrs": { "fromStage": "appointmentscheduled", "toStage": "qualifiedtobuy" },
  "source": "hubspot"
}


How it’s produced (examples)

deal.stage_changed — from propertyChange on dealstage (or hs_pipeline_stage); enrich to resolve stage names. 
HubSpot Developers

association.added|removed — from associationChange → correlates cases across objects. 
HubSpot Developers

ticket.stage_changed, contact.owner_changed, thread.message_created, etc. (Conversations when enabled). 
HubSpot Developers

Case correlation (process mining)

Sales: case = dealId; enrich with company/contact via association events.

Support: case = ticketId; correlate with conversations thread/messages.

Marketing touches: case = contactId (or campaign, if your PM lens is campaign‑centric).

Keep lookup tables in the processor to map objectTypeId (e.g., 0-1 contact, 0-3 deal). 
HubSpot Developers

E) Control‑plane (schema → ArchiMate) — once, then drift‑watch

You still need a thin backfill to publish the structure of HubSpot into Ameide’s EA graph. Do this once per tenant (and then only on drift):

Objects & properties → derive information structure and capabilities.

Pipelines & stages → derive process structure.

Association definitions → derive allowed relations (with labels).
Fetch these via CRM/Properties/Pipelines/Associations APIs, or GraphQL collector where you gain fewer round trips. 
HubSpot Developers
+1

F) ArchiMate‑aligned graph projection

Below is a pragmatic, lose‑nothing mapping. (ArchiMate definitions are standardized by The Open Group; see spec.) 
www.opengroup.org

HubSpot concept	ArchiMate element	Rationale
HubSpot (product) / Hubs	Application Component	A discrete application providing services to business.
CRM Object Type (Deal, Ticket, Contact, Company, Custom)	Data Object (+ optional mapped Business Object)	Information objects; you can also reflect conceptual business objects (e.g., “Customer”) and mark Realization from Business → Data.
Property of an object	Attribute on Data Object	Kept as metadata; expose key properties (ID, owner, stage, timestamps).
Pipeline	Business Process (skeleton)	Represents the designed flow; each Stage can be a sub‑process or a state.
Stage	Business Process (sub‑process) or Business State	Choose one consistently; many EA teams prefer sub‑process.
Workflow / Sequence	Application Process (or Application Function)	Automated logic inside the application layer.
Association Type (e.g., deal↔company)	Association / Flow (depending on semantics)	Use Association for static relations; add Flow where it denotes movement of information.
Owner (user/team)	Business Role / Business Actor	Roles execute processes; actors perform roles.
Conversations (Thread/Message)	Business Interaction + Data Object (Message)	Interaction among actors; messages as exchanged information.

Licensing note: ArchiMate 3.2 is owned by The Open Group; review the licensing terms if you embed icons/notations in Ameide’s UI. 
www.opengroup.org
+1

ArchiMate relationships you’ll use most

Assignment (Role → Process), Realization (Data realizes Business), Serving (Application Service → Business Process), Flow (between processes/interactions), Aggregation/Composition (structural containment), Association (generic link). (See spec reference cards for canonical usage.) 
www.opengroup.org

G) Stream processors (from webhooks to Ameide)

1) EA projector (ArchiMate graph)

On first sight of a portalId, emit: Application Component = HubSpot (+ hub breakdown if useful).

When properties/pipelines/associations are requested (first 24h or on drift timer), pull minimal schema via REST/GraphQL and upsert: Data Objects, Business Processes (pipelines), Association relations. 
HubSpot Developers
+1

2) Process‑mining projector (streaming, no polling)

Consume webhook events; map to activity names & case IDs (see §D).

Enrich with just‑enough lookups (owner name, stage label) using a tenant LRU cache to avoid live calls; on cache miss, use batch REST.

Write to Ameide’s event store (e.g., Parquet/Delta/Kudu) partitioned by date/tenant/objectType, or directly to Ameide’s native streaming API.

H) Cross‑tenant fairness & resilience

Per‑tenant budgets: enforce HubSpot’s per‑install limit; queue bursts and apply backoff on 429. HubSpot’s usage docs and changelogs clarify current limits and the separate Search throttle. 
HubSpot Developers
HubSpot Community

Backpressure: if Ameide’s graph/process stores slow down, the queue absorbs; apply circuit‑breakers so the webhook ingress replies 200 quickly and offloads work to consumers.

Replay: keep a 7–14‑day topic retention; you can rebuild a tenant’s process trace without touching HubSpot again.

I) What still needs a “pull” (and how to keep it near‑real‑time)

A small number of analytics surfaces don’t emit webhooks. For those, run short, incremental pulls (e.g., 1–2‑minute windows) and publish them as events through the same stream so the rest of the pipeline is unchanged. Use batch endpoints/GraphQL to cap cost. 
HubSpot Developers

J) Security, governance, and reviewability

Webhook validation: verify Signature v3 with timestamp skew checks. 
HubSpot Developers

Least‑privilege scopes: make the event scopes required, and schema/marketing/conversations scopes optional, so installs succeed on any tier and features light up when available.

Per‑tenant encryption for tokens; scrub PII from logs.

Audit & observability: per‑tenant dashboards (webhook lag, queue depth, 429s, enrichment calls). HubSpot’s app usage logs are helpful when customers ask “what did you call?” 
HubSpot Developers

K) Deliverables you can stand up next

Webhook ingress + validator (Ameide K8s, TLS) and per‑tenant topic creation. 
HubSpot Developers

Canonical event normalizer with mappings for: contact/company/deal/ticket CRUD, dealstage/ticketstatus changes, associationChange, conversations message created. 
HubSpot Developers
+1

EA projector that emits ArchiMate nodes/edges for objects, pipelines, and association definitions (one‑time pull + drift watcher). 
HubSpot Developers

Process‑mining projector producing case events in near real time (no batch pulls), with correlation rules.

Backfill job (control‑plane only) for historic property histories you need for baselines; then disable ongoing pulls.

Tenant ops: install/uninstall flows, per‑tenant rate‑limiters (110/10s, Search at ~5 r/s), DLQ and replay. 
HubSpot Developers
+1

Quick visual (data‑plane only)
HubSpot Webhooks  ──▶  Ingress (validate v3)  ──▶  Tenant Queue  ──▶  Normalizer
       ▲                                                      │
       │                                                      ├─▶ EA Projector (ArchiMate graph upserts)
   (rare enrich via REST/GraphQL)                              └─▶ PM Projector (case events → Ameide)


If you want, I’ll provide:

a canonical schema (JSON Schema) for the event envelope + a starter topic taxonomy,

an ArchiMate mapping dictionary (object → element, relation → relationship) that Ameide can enforce, and

a K8s manifest with per‑tenant autoscaling knobs.

All three slot straight into this design and keep the process‑mining feed fully push‑based.

You said:
Finally explain in depth how ameide threads interfaces with graph as tool can support hubspot consulting activities 
ChatGPT said:
Request for GPT-5 Pro
Reasoned for 2m 50s

Below is a concrete blueprint for how Ameide Chat interfaces with the Ameide Graph (standardized on ArchiMate) to support HubSpot consulting—from architecture and tool contracts to example queries and day‑to‑day consulting workflows.

1) What “Ameide Chat + Graph” looks like

Ameide Chat is a conversational layer with tools. The most important tools for HubSpot consulting are:

Graph Tool – safe, parameterized access to the ArchiMate‑standardized enterprise graph (HubSpot → ArchiMate projection).

Event Tool – near‑real‑time reads over the process‑mining stream (HubSpot webhooks normalized into an event log).

Knowledge Tool – retrieval over internal docs, runbooks, and HubSpot public docs (optional, for explanations—not for data ground truth).

HubSpot On‑Demand Tool (optional) – scoped, rate‑limited API reads for the rare cases when fresh enrichment is needed that is not already in the graph or the event stream.

Design stance: Chat reads primarily from the graph & the event stream. It only “reaches out” to HubSpot when a field is missing or very fresh enrichment is required.

2) Multi‑tenant and security model

Tenant = HubSpot portal (portalId). Every node/edge/event in the graph carries a tenant tag.

Row‑level security is enforced by the Graph Service; the Chat runtime never sends a Cypher/Gremlin query missing the tenant guard.

Allow‑listed queries (see §4): Chat invokes named query templates; free‑form graph writes are not exposed to Chat.

PII policy: the Graph projection stores only the minimal personal data necessary for consulting (IDs, owner refs, stage names, timestamps). Chat masks emails/phones by default in answers unless the user role allows PII.

3) The data Ameide Graph exposes (ArchiMate projection)

Nodes (examples):

:ApplicationComponent { name: "HubSpot", hub: "Sales|Service|Marketing|Ops" }

:DataObject { objectType: "deal|ticket|contact|company|custom", internalName, label }

:Property { objectType, name, label, type, required, options }

:BusinessProcess { name, pipelineId } and :BusinessProcess { name: stageName }

:ApplicationProcess { name } (Workflows / Sequences)

:BusinessRole { name }, :BusinessActor { userId, teamId }

:Campaign, :Form, :MarketingEmail, :Inbox, :Thread, :Message

Edges (examples):

(:DataObject)-[:ASSOCIATION {typeId, label}]->(:DataObject)

(:BusinessProcess)-[:COMPOSED_OF]->(:BusinessProcess) (pipeline → stage)

(:DataObject)-[:IN_STAGE]->(:BusinessProcess {stageName}) (thin data pointer)

(:ApplicationProcess)-[:REFERENCES_PROPERTY]->(:Property) (workflows/list/filter/form references)

(:BusinessRole)-[:ASSIGNED_TO]->(:BusinessProcess) (owners to processes)

(:ApplicationComponent)-[:SERVES]->(:BusinessProcess)

Process‑mining event log (streamed, not batch)

Event node or external event table with envelope:

tenant, ts, objectType, objectId, eventType, actor, attrs (e.g., fromStage, toStage, label, association), source:"hubspot"

Edges (:Event)-[:NEXT]->(:Event) per case to support dwell‑time calculations.

4) The Graph Tool interface (what Chat can call)

Expose a small set of safe, typed functions (these map to parameterized Cypher under the hood):

Read functions

graph.get_object_types(tenant)

graph.get_properties(tenant, object_type, include_options=false)

graph.find_assets_referencing_property(tenant, object_type, property_name) ➜ (workflows, lists, forms, emails)

graph.find_unassociated_records(tenant, object_type, required_association) ➜ (e.g., deals without company)

graph.pipeline_overview(tenant, object_type) ➜ (pipelines, stages, probabilities)

graph.impact_of_change(tenant, object_type, property_name) ➜ (workflows/lists/CTAs that use it)

graph.schema_diff(tenant, snapshotA, snapshotB) ➜ (what changed since)

Metrics helpers (pre‑computed or derived)

graph.stage_dwell_baseline(tenant, object_type)

graph.property_fill_rates(tenant, object_type) (from a nightly aggregator or materialized view)

The Graph Service owns the Cypher. Chat just supplies arguments. This keeps queries fast, safe, and multi‑tenant‑correct.

Example Cypher behind graph.find_assets_referencing_property
(illustrative—your actual schema names may differ)

MATCH (p:Property {tenant:$tenant, objectType:$objectType, name:$prop})
OPTIONAL MATCH (wf:ApplicationProcess {tenant:$tenant})-[:REFERENCES_PROPERTY]->(p)
OPTIONAL MATCH (f:Form {tenant:$tenant})-[:REFERENCES_PROPERTY]->(p)
OPTIONAL MATCH (l:List {tenant:$tenant})-[:REFERENCES_PROPERTY]->(p)
RETURN p {.*, 
  usedBy: {
    workflows: collect(DISTINCT wf{.name, .id}),
    forms:     collect(DISTINCT f{.name, .id}),
    lists:     collect(DISTINCT l{.name, .id})
  }
}

5) The Event Tool interface (near real‑time process mining)

Expose low‑latency functions backed by the streaming store / event index (or event nodes in graph):

events.case_timeline(tenant, object_type, object_id, since)

events.stage_durations(tenant, object_type, window)

events.bottlenecks(tenant, object_type, window, topN)

events.conformance(tenant, object_type, window) ➜ compares actual sequences to the pipeline model.

events.variance_by_owner(tenant, object_type, stage, window)

events.rework_loops(tenant, object_type, window) ➜ (e.g., stage A → B → A loops)

Example (dwell by stage in last 30 days)

MATCH (e1:Event {tenant:$tenant, objectType:$obj, eventType:'stage_changed'})
WHERE e1.ts >= datetime() - duration('P30D')
MATCH (e1)-[:NEXT]->(e2:Event {tenant:$tenant, objectType:$obj})
WHERE e1.caseId = e2.caseId
RETURN e1.toStage AS stage, avg(duration.between(e1.ts, e2.ts)) AS avg_dwell
ORDER BY avg_dwell DESC
LIMIT 10

6) How this supports HubSpot consulting (concrete workflows)

Below are the top consulting tasks and how Ameide Chat executes them through the tools.

A) Discovery & documentation

“Give me a living map of this portal—objects, properties, pipelines, associations, and which automations depend on what.”

Chat calls graph.get_object_types → shows object families.

For each object: graph.get_properties, graph.pipeline_overview.

graph.find_assets_referencing_property for high‑risk fields (e.g., lifecycle_stage, dealstage).

Outputs a navigable brief: ArchiMate view (Application Component ↔ Business Process ↔ Data Object), with dependency tables.

Value: rapid onboarding, immediate impact analysis before making changes.

B) Change impact analysis

“What breaks if we rename or delete property X?”

graph.impact_of_change(tenant, 'contact', 'lifecycle_stage')

Chat lists: workflows, lists, forms, sequences, emails using it; highlights whether usage is read vs write; proposes a safely‑sequenced migration (e.g., duplicate → backfill → swap → deprecate → delete).

C) Data hygiene & governance

“What properties are unused / low‑fill? Where are orphan records?”

graph.property_fill_rates(tenant, 'company') → sorted by fill% asc.

graph.find_unassociated_records(tenant,'deal','company') → list deals to fix.

Chat outputs remediation playbooks (lists to build, import templates, validation via property rules).

D) Pipeline performance & bottlenecks (near real time)

“Where are deals getting stuck this week?”

events.stage_durations(tenant,'deal','P7D') and events.bottlenecks(tenant,'deal','P7D',5)

Chat explains variance vs baseline (from graph.stage_dwell_baseline) and proposes workflows tweaks (e.g., SLA reminders) or stage design changes.

E) Conformance / rework detection

“Are reps skipping mandatory qualification?”

events.conformance(tenant,'deal','P30D') compares actual event sequences to the pipeline’s designed order.

events.rework_loops(...) surfaces loops (A→B→A).

Chat outputs human‑readable root‑cause hypotheses (owner, segment, source) and suggests specific guardrails (required fields, workflows gates).

F) Marketing → Sales handoff analysis

“How long from MQL (form submit) → SQL (deal created) by source/owner?”

Use Form submission events + deal.created events, correlated by contact/company association.

events.variance_by_owner('deal','qualifiedtobuy','P30D') for handoff times; join to campaign/campaign member where modeled.

G) Support operations

“Which ticket states are breaching SLA right now?”

events.stage_durations(tenant,'ticket','P1D') filtered by ticketstatus stages; Chat flags cases exceeding SLA thresholds and offers bulk actions (lists, alerts).

H) “Explain this record” inside HubSpot (UI Extension panel)

From a contact/deal page, the extension sends tenant, object_type, object_id as context.

Chat uses events.case_timeline to produce a timeline (touches, ownership changes, stage moves) and recommends next best actions.

7) Example tool contracts (so you can implement rapidly)

Graph Tool (OpenAI‑style function schema, abridged)

{
  "name": "graph.find_assets_referencing_property",
  "description": "Find workflows/lists/forms/emails that reference a property.",
  "parameters": {
    "type": "object",
    "properties": {
      "tenant": { "type": "string" },
      "object_type": { "type": "string", "enum": ["contact","company","deal","ticket","custom"] },
      "property_name": { "type": "string" }
    },
    "required": ["tenant","object_type","property_name"]
  }
}


Event Tool

{
  "name": "events.stage_durations",
  "description": "Average dwell per stage within a window.",
  "parameters": {
    "type": "object",
    "properties": {
      "tenant": { "type": "string" },
      "object_type": { "type": "string" },
      "window": { "type": "string", "description": "ISO8601 duration like P7D" }
    },
    "required": ["tenant","object_type","window"]
  }
}


The backend binds these to prepared Cypher/SQL or event‑store queries. The LLM never writes arbitrary queries.

8) How answers are grounded (and auditable)

Every Chat answer includes a grounding trace:

Which tool calls were made (function + args, not raw credentials).

Counts & timestamps (e.g., “events analyzed: 4,512 in P7D”).

Links into Ameide (graph nodes, dashboards) so a consultant can click to verify.

This makes guidance explainable for stakeholders.

9) Performance & SLOs you can safely promise

Event freshness: near real time (seconds to <1 minute) because HubSpot webhooks → Ameide queue → Event Tool index; no polling.

Graph reads: typically sub‑second for metadata; multi‑second if a large impact query touches many assets—still bounded by parameterized queries and proper indexing.

Fairness: per‑tenant rate budgets ensure one noisy portal doesn’t starve others.

10) “Day‑one” checklist to enable this in Ameide

Tool registry: register the Graph & Event tools with the contracts above.

Allow‑listed queries: implement the Cypher behind each tool; add tenant guards.

Event stream: ensure the webhook normalizer is producing the canonical envelope and the Event Tool can query it.

ArchiMate mapping dictionary: freeze v1 (object → element, relation → relationship, property type mapping).

UI Extension (optional): embed a Chat panel inside HubSpot to pass {tenant, object_type, object_id} context.

11) Typical prompts consultants use (and what the tools do)

“List risky properties whose changes would impact automations.”
→ graph.property_fill_rates, graph.find_assets_referencing_property across core objects.

“Show the last 20 events for deal 123 and where it’s stuck.”
→ events.case_timeline(…, 'deal', '123', 'P30D') + events.stage_durations('deal','P7D').

“Which workflows update contact.owner and how often did they run last week?”
→ graph.find_assets_referencing_property('contact','owner_id') + event‑count by workflows action if you model custom workflows actions as events.

“What’s the conformance rate to the ideal pipeline for SMB vs ENT?”
→ events.conformance('deal','P30D') grouped by segment property, cross‑checked with graph pipeline.

Bottom line

Ameide Chat interfaces with the ArchiMate‑standardized graph via safe, named read tools and with the process‑mining stream via near‑real‑time event tools.

That combination yields explainable, instantly actionable guidance for HubSpot consulting: inventory, impact analysis, governance, and operational tuning—all without brittle scraping or slow batch jobs.

If you’d like, I can provide a starter set of 12 named queries (Cypher) and 8 event aggregations that map exactly to the workflows above, ready to drop behind your Graph/Event tools.

You said:
Given this ultimate goal, would you revisit anything of all the above 
ChatGPT said:
Request for GPT-5 Pro
Reasoned for 2m 17s

Absolutely—given the end state (Ameide Chat driving HubSpot consulting on top of an ArchiMate‑standardized enterprise graph with near‑real‑time process‑mining), I would revise and harden several parts of the plan. Below are the concrete changes I recommend, why, and what they enable.

0) TL;DR — What I’d change

Make webhooks the source of truth and treat REST only as bootstrap/repair.

Adopt an object‑centric event log (OCEL‑style) rather than a single‑case event model.

Separate “meta graph” from “operational graph” and add temporal validity on key edges (e.g., stage membership).

Tighten the ArchiMate mapping (pipelines = Business Process; stages = Business Process steps or states) and publish a fixed mapping dictionary.

Drop GraphQL for the connector data plane (keep it for UI extensions only).

Introduce a capability‑discovery handshake per tenant to switch streams on/off without code changes.

Add a schema registry & versioned contracts for events and graph upserts.

Strengthen multitenancy isolation (per‑tenant quotas, topics, encryption domain).

Harden real‑time SLOs & backpressure (ack‑first webhook ingress, idempotent replay).

Expand the Chat tool surface to include impact simulation and conformance checks as first‑class tools, not just queries.

Minimize PII at the edge (hash/tokenize; field‑level encryption for any unavoidable PII).

Operationalize “uninstall & DSAR” (data deletion end‑to‑end) for marketplace compliance.

Everything below explains these, with actionable specifics.

1) Event model: OCEL‑style, not single‑case

Why: HubSpot processes (Sales, Support, Marketing) are inherently multi‑object (deal↔company↔contact, ticket↔thread↔message). A classic “one case ID” log forces lossy joins.

Change: Normalize to an Object‑Centric Event Log:

Each event can reference multiple objects ({type,id} pairs).

Keep event type, actor, ts, attributes (e.g., fromStage, toStage, associationLabel).

Maintain object–event tables/edges so you can ask: “all events touching this deal/company/contact” without re‑joining the world.

Impact: Conformance analysis, rework loops, and cross‑domain traces “just work” without fragile correlation code.

2) Webhooks as the authoritative rail

Why: You want near‑real‑time and predictable load. Polling makes freshness vs. cost a trade‑off.

Change:

Ingress verifies signatures, acks immediately, and enqueues.

Workers enrich only when needed (e.g., to resolve a stage label) and never call large fan‑out reads inline.

Backfill is a separate control‑plane job (one‑time) with resumable checkpoints.

Add a reconciliation job (daily) that checks webhook completeness against record counts for drift/holes.

Impact: Sub‑minute latency with bounded HubSpot API usage.

3) Graph shape: meta vs operational + temporal edges

Why: Consulting questions span structure (“what depends on what?”) and runtime (“where do things get stuck?”). One graph trying to do both becomes noisy and slow.

Change:

Meta graph (stable): ArchiMate elements for application, processes (pipelines/stages), data objects (object types), properties, and allowed association types.

Operational graph (thin): only record references and temporal edges needed for analytics:

(:Record {type,id})-[:IN_STAGE {from,to}]->(:Stage)

(:Record)-[:ASSOCIATED {typeId,label,from,to}]->(:Record)

Keep rich event detail outside the graph in the OCEL stream/index; link with eventId.

Impact: Faster, simpler queries for Chat; time‑travel is explicit; you can purge old runtime edges without touching the meta model.

4) ArchiMate mapping (lock it)

Change (opinionated, consistent):

Pipelines → Business Process (designed flow).

Stages → Business Process (child) or Business State—pick one and stick with it; if you choose “state,” add state machine semantics (allowed transitions).

Workflows/Sequences → Application Process; relate with Serving to Business Processes they influence.

CRM object type → Data Object; add optional Business Object (e.g., “Customer”) with Realization → Data Object.

Associations → Association; add Flow only where information transfer is modeled.

Impact: Clean, defensible views for architecture boards; less rework later.

5) Connector data plane: REST only (GraphQL for UI)

Why: REST CRM/Associations/Exports are the stable, capacity‑planned surface for connectors. GraphQL “collector” is great for UI reads, but it’s not the right rail for high‑volume ingestion.

Change:

REST for bootstrap, enrich, and reconciliation.

GraphQL reserved for UI extension (snappy, nested reads for the card), never on the hot path.

6) Capability discovery per tenant

Why: Portals vary wildly (Free vs. Enterprise; enabled hubs; Beta access). You need zero‑touch onboarding.

Change: On install & daily:

Probe scopes/features (e.g., Conversations, Campaigns, Sequences) and write a capability ledger per tenant.

The pipeline activates streams accordingly; Chat tools read the ledger to avoid offering features the tenant doesn’t have.

Impact: One build runs on all tiers gracefully.

7) Contracts: schema registry & versioning

Why: You’ll evolve event payloads and graph upsert shapes.

Change:

Maintain a schema registry (JSON Schema/Avro) for: ocel.event.v1, ocel.event.v2, graph.upsert.v1, etc.

Producers and consumers negotiate versions; Chat declares what versions it understands.

Impact: Safe evolution, easier multi‑team development.

8) Multitenancy isolation & quotas

Change:

Per‑tenant topics/partitions; per‑tenant token buckets respecting HubSpot’s public‑app limits; per‑tenant encryption keys for OAuth tokens and any PII.

Row‑level security in the graph/event APIs via mandatory tenant predicates; no wildcard reads.

Impact: Blast‑radius containment; predictable performance.

9) SLOs, backpressure & replay

Change:

SLOs: p95 ingest‑to‑available < 60s; drop rate 0; reconciliation drift < 0.1%.

Ingress acks immediately; all heavy lifting happens after the queue.

Replay controls: 7–14‑day retention per tenant; idempotency keys (tenant, eventId); DLQ with auto‑triage.

API backoff with jitter and per‑endpoint concurrency (Search endpoints are tighter than CRUD).

Impact: You can promise (and observe) freshness and correctness.

10) Chat tool surface (beyond “query”)

Add these tool types so consultants can act, not just analyze:

impact.simulate_property_change() → resolve all dependent assets from meta graph and produce a safe migration plan.

process.compare_baseline(window) → compute variance vs. baseline dwell/conformance and output targeted recommendations.

data.governance_report() → unused properties, low‑fill fields, orphan records, suggested validation rules.

ops.health() → tenant ingest lag, webhook success, reconciliation drift (so Chat can say “your data is fresh/stale”).

Each tool maps to one or more parameterized queries + lightweight rules—no ad‑hoc graph writes from Chat.

11) PII minimization & privacy

Change:

Hash emails/phones at ingress (stable hash per tenant); store cleartext only if the consulting use case demands it, and then field‑encrypt.

PII redaction middleware for Chat responses by default; allow explicit “unmask” only for permitted roles.

Uninstall & DSAR automation: delete tenant tokens, wipe graph slices, tombstone events, and emit a completion receipt.

Impact: Market‑ready privacy posture and smoother HubSpot app review.

12) What to de‑emphasize or remove

No scraping or undocumented endpoints—marketplace apps must pass review.

No GraphQL on ingestion; it’s a UI convenience, not an ETL surface.

Avoid heavy “record nodes” in the graph; keep IDs + minimal attributes and push behavior to the event store.

13) Implementation checklist (delta from prior plan)

 Switch event envelope to OCEL‑style and publish v1 schema.

 Introduce temporal edges and split meta/operational graphs.

 Build capability discovery and tenant ledger.

 Replace any ingestion‑path GraphQL calls with REST batches; keep GraphQL only for the UI card.

 Add schema registry + version pinning; add reconciliation job.

 Add ops.health tool and metrics: ingest lag, 429 rates, replay counts.

 Finalize ArchiMate mapping dictionary and lock it in code (constants, not ad‑hoc).

 Wire privacy controls (hash/tokenize, field‑level encryption) and uninstall path end‑to‑end.

 Update Chat tool contracts to include simulate, compare_baseline, governance_report, ops.health.

What this buys you

Better consulting UX (impact analysis, conformance, hygiene) with one‑click explanations.

Lower runtime cost (event‑first; fewer HubSpot calls).

Higher confidence (temporal edges + replay + reconciliation).

Market‑ready posture (privacy, uninstall, no scraping).

True “works on all tiers” behavior (capability discovery + optional scopes).

If you want, I’ll turn these revisions into: (1) an updated component diagram, (2) the OCEL v1 JSON Schema, and (3) the ArchiMate mapping dictionary you can bake into the projector.

You said:
Focus on a consulting firm hubspot implementation journey with ameide
ChatGPT said:
Request for GPT-5 Pro
Reasoned for 2m 52s

Below is a field‑tested implementation journey tailored to a consulting firm rolling out HubSpot, with Ameide as the backbone for design, governance, and near‑real‑time operations. It’s structured as phased work you can run end‑to‑end or adapt to your cadence. For each phase you’ll see: goals, key activities, Ameide’s role (Graph + Events + Chat), deliverables, and exit criteria—plus industry‑specific patterns for consulting firms.

Phase 0 — Engagement setup & success criteria (Week 0)

Goals

Align on business outcomes (pipeline velocity, win rate, utilization pre‑signals, CSAT).

Decide “system of record” boundaries (HubSpot vs PSA/ERP).

Lock roles and governance.

Activities

Stakeholder map: Managing Partners, Practice Leads, RevOps, MOPs, Sales Ops, Marketing, Delivery, Finance.

Define target KPIs & reporting grain (by practice, region, deal type).

Security model: who can see what (by practice/region); PII posture.

Ameide’s role

ArchiMate baseline canvas: capture current processes (“Business Development”, “Opportunity Mgmt.”, “Proposal”, “Contracting”, “Engagement Handoff”) and systems landscape.

Chat prompts (examples):

“List capabilities we need in Sales Hub for a B2B consulting pipeline.”

“Draft an RBAC model by practice and geography.”

Deliverables

Project charter, RACI, KPI tree, access model.

ArchiMate “as‑is” (lightweight) and “to‑be” capability map.

Exit criteria

KPIs & access model approved; scope risks logged; cadence agreed.

Phase 1 — Blueprint & configuration design (Weeks 1–3)

Goals

Design the target HubSpot configuration to match how a consulting firm sells and hands off to delivery.

Freeze critical schemas and pipelines before build.

Activities

Pipelines (at least two):

New Business (Lead → MQL → SQL → Proposal → Legal/Redlines → Won)

Expansion/Renewal (for ongoing accounts)

Stage exit criteria (checklist per stage), required fields by stage.

Properties pack:

Deal: practice, service line, deal type (advisory/implementation/managed), sourcing (partner/referral/RFP), pricing model (T&M, fixed), forecast category.

Contact: buying role, seniority, practice alignment.

Company: parent/child for roll‑ups, industry, region.

Associations & labels: Deal↔Company (primary), Deal↔Contacts (Decision Maker, Champion), Deal↔Ticket (for pre‑sale technical due diligence), Company↔Company (Parent/Child).

Marketing: lifecycle model, lead scoring (content + intent), MQL definition per practice.

Service (optional at launch): basic ticket pipeline for pre‑sale requests; later integration with PSA.

Ameide’s role

Meta graph (ArchiMate):

Business Processes = pipelines & stages,

Data Objects = CRM objects,

relations = allowed associations.

Impact simulator: run “what breaks if we rename lifecycle_stage?” against workflows/lists/forms (once they exist).

Chat: turn policy into concrete config (“Generate the property list for Deal with internal names and validations”).

Deliverables

Configuration specification (pipelines, properties, associations, permissions).

ArchiMate “to‑be” with traceability from processes → data.

Exit criteria

Spec signed; config freeze + change‑control agreed.

Phase 2 — Build & governance controls (Weeks 3–6)

Goals

Build HubSpot to spec, with governance to prevent drift.

Activities

Create pipelines/stages with stage exit criteria as property validations.

Properties with help text, options, and owner/required rules.

Lists, forms, key workflows (lead routing, SLA reminders, data hygiene), sequences (BD outreach).

Teams/permissions by practice & region; naming conventions; audit dashboards.

Ameide’s role

Config‑as‑graph: continuously sync HubSpot config → Ameide meta graph.

Policy checks (Chat‑driven):

“Show properties added outside spec in last 7 days.”

“List workflows that write to dealstage—should be read‑only here.”

Drift alerts: if a new property bypasses review, Chat flags impact (workflows/lists/forms that would be affected).

Deliverables

HubSpot portal configured, governance rules published, review dashboards.

“Day‑in‑the‑life” UAT scripts.

Exit criteria

UAT pass on discovery→proposal workflows; governance checks green.

Phase 3 — Data migration & hygiene (Weeks 4–7, overlaps P2)

Goals

Move clean, minimal data from legacy tools (CRM, spreadsheets, email tools).

Activities

Field mapping (Company/Contact/Deal), dedupe (by domain + legal name + country).

Backfill buying committee roles on associated contacts.

Import historical stage history if available (or compute first‑seen/last‑seen proxy).

Lists for re‑engagement; sequence suppression rules.

Ameide’s role

Data quality scorecard: fill‑rates, duplicate clusters by practice/region.

Chat: “Generate a CSV import template for deals with required fields per stage.”

Reconciliation: after imports, Ameide checks counts and key metrics (e.g., pipeline coverage).

Deliverables

Clean import files, migration report, reconciliation sign‑off.

Exit criteria

Fill‑rates ≥ thresholds (e.g., 95% for mandatory); dedupe residual < target; stakeholder sign‑off.

Phase 4 — Integrations & event streaming (Weeks 6–9)

Goals

Establish near‑real‑time event feed for process mining; connect essential tools.

Activities

Install the Ameide HubSpot marketplace app (public app).

Enable webhooks for Contacts/Companies/Deals/Tickets/associationChange; optional: Conversations, Marketing Emails.

(Optional) Connect PSA/PM (Jira, Asana, Monday, Harvest) for delivery handoff and “first activity after deal won”.

Ameide’s role

Event normalizer (OCEL‑style): object‑centric events like deal.stage_changed, association.added, contact.owner_changed, thread.message_created.

Process projector: create case timelines for deals and tickets in seconds.

Chat “ops.health”: event lag, 429 rates, webhook delivery success.

Deliverables

Event feeds online, dashboards for velocity/bottlenecks, integration runbooks.

Exit criteria

p95 event freshness < 60s; no data loss; core KPIs updating in near real time.

Phase 5 — Training, cutover & hypercare (Weeks 8–10)

Goals

Go live with users; stabilize; prove value with the first insights.

Activities

Role‑based training: BD/Partners, AEs, Solutions, Marketing, RevOps, Practice Leads.

“First 14 days” operational cadence: pipeline review with conformance & dwell insights.

Cutover support: lead routing checks, sequence throttling, permission escalations (temporary).

Ameide’s role

In‑HubSpot UI extension: “Ameide Assistant” panel on records → timeline, stuck‑reason, next best action.

Chat playbooks:

“Why is Deal #123 stuck?”

“Which proposals wait on Legal > 5 days?”

“Which leads became MQL with no owner within SLA?”

Deliverables

Training recordings, quick‑reference guides, hypercare channel, daily health reports.

Exit criteria

Adoption ≥ targets (logins, activity), SLA breaches trending down, first value stories captured.

Phase 6 — Continuous improvement & scale (Weeks 10+)

Goals

Iterate on process design; unlock advanced capabilities (ABM, CPQ, CSAT).

Activities

Bottleneck remediation sprints (e.g., automate proposal approvals; pre‑populate scoping fields).

Conformance governance: ensure reps follow mandated sequences (no skipping qualification).

ABM for key accounts: intent + content; partner sourcing reporting.

Revenue predictability: forecast category hygiene, scenario boards by practice.

Ameide’s role

Baselines: historical dwell/conversion by practice/segment; compare current week vs baseline.

Impact simulation: propose “small config changes → predicted effect” (e.g., required field, routing tweak).

Change control: every change reviewed for impact (workflows/lists/forms) before publish.

Deliverables

Quarterly “RevOps health” report; roadmap; backlog with quantified ROI.

Exit criteria

KPI uplift vs baseline (e.g., +15–25% win rate on targeted segments; −20–30% proposal cycle time); audit shows controlled config drift.

Consulting‑firm patterns to bake in (ready‑made content packs)
1) Pipelines & stage definitions (starter)

New Business: Prospecting → Discovery → Solutioning → Proposal → Legal → Waiting on Client → Won/Lost.

Exit checks: Discovery (meeting logged + stakeholders mapped), Proposal (price & scope attached), Legal (redlines artifact).

Expansion: Identify Opportunity → Expansion Proposal → Decision → Won/Lost.

2) Buying committee roles

Decision Maker / Budget Holder / Executive Sponsor / Champion / Influencer / Procurement.

Enforce at Proposal stage: at least Decision Maker + Champion associated.

3) Lead‑to‑opportunity rules

MQL by practice (content + role fit + intent score).

SLA: MQL → first contact within 1 business day; escalations via workflows & Ameide alerts.

4) Data governance pack

Required fields by stage, per deal type; picklists locked; tooltip help text.

Quarterly deprecation policy: unused properties flagged, candidates removed after backup.

5) Ops dashboards (near real time)

Stage dwell (p50/p90) by practice/segment; leakages; rework loops.

Conformance (ideal vs observed transition graph).

Handover: Won → first delivery activity (if PSA connected).

How Ameide Chat accelerates the engagement (examples)

Discovery

“List all Deal properties not in the spec.”

“Show automations referencing dealstage or lifecycle_stage.”

Migration

“Generate a CSV mapping for Salesforce → HubSpot for Company/Deal; highlight unmapped fields.”

“Report fill‑rates for ‘practice’ after import.”

Cutover

“Which MQLs are unowned in the last 24h?”

“What are the top 3 bottleneck stages this week for Advisory deals?”

Governance

“Simulate renaming practice to service_line: affected workflows/lists/forms.”

“Show config drift in the last 7 days.”

Roles & RACI (condensed)
Task	Exec Sponsor	Practice Leads	RevOps	Sales Ops	MOPs	Delivery/PS	Ameide Team
Charter, KPIs	A	C	R	C	C	C	C
Blueprint (Pipelines/Props)	C	A	R	R	C	C	C
Build & Governance	C	C	A	R	R	C	R
Migration	C	C	A	R	R	C	R
Integrations & Events	C	C	R	C	C	R	A
Training & Cutover	C	C	A	R	R	C	R
Continuous Improvement	A	A	R	C	C	C	R

(A=Accountable, R=Responsible, C=Consulted)

Risks & mitigations (high‑leverage)

Skipping stage exit definitions → inconsistent data, poor forecasts
→ Mitigate: define exit criteria; make required fields; monitor with conformance checks.

Config sprawl (properties, workflows)
→ Mitigate: Ameide drift alerts; monthly deprecation; change‑control via impact simulator.

Late data during cutover
→ Mitigate: event pipeline with DLQ/replay; reconciliation job; Chat “ops.health”.

Practice silos (different processes)
→ Mitigate: shared core stages + practice‑specific sub‑stages; reporting roll‑ups in Ameide.

90‑day outcome targets (typical for consulting firms)

Lead→SQL conversion +20–30% (by routing + SLA governance).

Proposal cycle time −20–30% (stage exit clarity + bottleneck removal).

Forecast accuracy ±10% at T‑30 days (owner hygiene + stage conformance).

Adoption: ≥80% reps using sequences; ≥90% activities logged.

Data quality: required field fill‑rate ≥95%; dedupe residual <2%.

What you get “out of the box” from Ameide in this journey

ArchiMate meta graph continuously mirroring HubSpot config.

OCEL‑style event stream from HubSpot webhooks (sub‑minute freshness).

Governance guardrails (drift detection, impact simulation).

Consulting‑ready Chat tools that answer “why”, “what next”, and “what breaks if…”.

If you want, I can convert this into a week‑by‑week workplan with task owners, acceptance criteria, and a starter set of Chat tool prompts you can paste into your runbook.

You said:
Rebase on togaf adm 
ChatGPT said:
Request for GPT-5 Pro
Reasoned for 2m 43s

Below is a TOGAF ADM–based delivery plan for a consulting firm’s HubSpot implementation with Ameide providing the enterprise‑architecture backbone (ArchiMate graph, event streaming, and Chat‑assisted governance). It keeps the event‑first posture (near real time via webhooks), multi‑tenancy, and the ArchiMate standard you adopted.

Phase map (ADM → HubSpot with Ameide)
ADM Phase	Objective (HubSpot context)	Key Ameide Assets	Primary Outputs / Gates
Preliminary	Establish architecture capability, principles, and EA governance specific to RevOps	EA charter, principles, reference maps	EA Charter approved; Architecture Board stood up; principles ratified
A — Vision	Define value, scope, and KPIs for Sales/Marketing/Service; secure sponsorship	Motivation views; value streams; capability map	Architecture Vision; stakeholder map; KPI tree; go/no‑go
B — Business	Model pipelines, lifecycle, roles, SLAs, and handoffs	ArchiMate Business Process & Capability views	Target Business Architecture; stage exit criteria; governance policies
C — Data	Standardize CRM object model, properties, associations, data quality	ArchiMate Data Object views; property catalogs	Target Data Architecture; DQ rules; retention/privacy policy
C — Application	Place HubSpot among apps (PSA/ERP/Docs/e‑sig), integration contracts	Application Cooperation, Interface matrices	Target Application Architecture; integration contracts
D — Technology	Land the runtime (Ameide K8s pod, graph, streaming, observability)	Technology portfolios; deployment views	Target Technology Architecture; non‑functional SLOs
E — Opportunities & Solutions	Group ABBs/SBBs into releases; define transition architectures	Solution building blocks + roadmaps	Solution Options; Transition Architecture(s)
F — Migration Planning	Sequence work packages, risks, costs, and benefits	Work package portfolio; dependency/benefit matrices	Detailed migration plan; backlog; release plan
G — Implementation Governance	Control drift; assure compliance; manage waivers	Architecture Contracts; conformance checks	Conformance assessments; gated change control
H — Architecture Change Mgmt	Evolve safely; re‑baseline by metrics	Change log; capability ledger; proposals	Approved change backlog; periodic re‑baseline
Requirements Mgmt (central)	Trace requirements ↔ models ↔ tests ↔ KPIs	Requirements graph in Ameide	Bi‑directional traceability; CR process
Phase by phase—what to do, what to produce, how Ameide helps
Preliminary: Establish the EA capability for RevOps

Focus: Principles, roles, governance.

Principles (examples): Event‑first; Single source of truth per domain; Least PII; Optional scopes; Versioned contracts; Observability by default.

Ameide: Seed the Architecture Repository with an EA Charter, Principles Catalog, and Reference Models (ArchiMate).

Deliverables: EA Charter, Architecture Governance framework, initial graph (ArchiMate element set, naming conventions), decision log.

Gate: Architecture Board formed; tool access and RACI set.

Phase A — Architecture Vision

Focus: Business outcomes & scope.

Activities: Stakeholder map (Partners, Practice Leads, RevOps), KPI tree (Win rate, cycle time, forecast accuracy, SLA breaches), scope boundaries (HubSpot vs PSA/ERP).

Ameide: Motivation view (Drivers, Goals), Value Stream “Lead→Cash”, initial Capability Map by practice.

Outputs: Architecture Vision, high‑level migration sketch, risk register.

Gate: Vision signed; funding & timebox approved.

Phase B — Business Architecture

Focus: Processes, roles, policy.

Activities:

Pipelines: New Business & Expansion; stage exit criteria & SLAs.

Handoffs: Marketing→Sales (MQL→SQL), Sales→Delivery (Won→Kickoff).

RBAC: teams by practice/region; buying committee roles (Decision Maker, Champion, etc.).

ArchiMate views: Business Process (pipeline with stages), Organization/Role, Value Stream to Process mapping.

Ameide: Store Business Processes; attach policy rules (required fields per stage) to stages; Chat proposes missing policies.

Outputs: Target Business Architecture; policy handbook (SLA, stage exits); conformance rules.

Gate: Business design frozen; conformance rules accepted.

Phase C — Information Systems Architecture
C.1 Data Architecture

Focus: CRM schema, quality, retention.

Activities:

Object model: Contact, Company, Deal, Ticket, Custom; associations with labels.

Properties: internal names, datatypes, picklists, validations.

Data Quality: fill‑rates, dedupe keys, golden‑record rules, retention & privacy (hashing, field encryption).

Event model: adopt OCEL‑style envelope (object‑centric events) for process mining.

ArchiMate views: Data Objects catalog; Data Dissemination; Information Structure.

Ameide: Meta graph with Property nodes/HAS_PROPERTY edges; DQ rules catalog; event schema registry ocel.event.v1.

Outputs: Target Data Architecture; DQ policy; retention/privacy controls.

Gate: Schema locked; DQ thresholds set; privacy review passed.

C.2 Application Architecture

Focus: App landscape & integration.

Activities:

Position HubSpot as Application Component with services (Sales, Marketing, Service).

Map integrations: PSA/PM (Jira/Asana/…); ERP; e‑signature; BI; email/calendar.

Define contracts: REST/webhooks to Ameide pod; association and property scopes; error/backoff.

ArchiMate views: Application Cooperation, Application Communication, Interface catalog.

Ameide: Integration contract templates; UI Extension spec for “Ameide Assistant” panel in HubSpot.

Outputs: Target Application Architecture; interface matrix; security model (scopes, secrets).

Gate: Contracts signed; security approved.

Phase D — Technology Architecture

Focus: Runtime & NFRs.

Activities:

Ameide K8s: Deployment (ingress, pod, queue, cache, graph DB, event store), secrets, HPA.

SLOs: p95 ingest‑to‑available < 60s; 0 data loss; tenant fairness (per‑tenant token buckets); resilience (DLQ/replay).

Observability: event lag, 429 rate, queue depth, conformance drift.

ArchiMate views: Technology portfolio, Deployment & Infrastructure diagrams.

Outputs: Target Technology Architecture; NFR sheet; runbooks.

Gate: Platform readiness review passed.

Phase E — Opportunities & Solutions

Focus: Package options; transition architectures.

ABBs (Architecture Building Blocks): OCEL schema; ArchiMate mapping dictionary; webhook validator; association resolver; DQ evaluator; impact simulator.

SBBs (Solution Building Blocks): HubSpot public app; Ameide exporter pod; UI Extension; PSA connector; BI semantic model.

Transition architectures:

T1: Core pipelines + DQ + near‑real‑time events (Deals/Tickets/Associations).

T2: Marketing touchpoints + Conversations.

T3: PSA/ERP integration, advanced conformance.

Outputs: Option analysis, selected solution, transition views.

Gate: Architecture Board selects option & transition roadmap.

Phase F — Migration Planning

Focus: Sequencing & value.

Activities: Work packages, dependencies, benefit realization matrix, cost & risk; training/cutover plan.

Ameide: Roadmap visualization; benefit tracking linked to KPIs; Chat generates work‑package charters with acceptance criteria.

Outputs: Release plan (90‑day slices), RAID log, budget.

Gate: Plan & budget approved.

Phase G — Implementation Governance

Focus: Assure compliance & control drift.

Mechanisms:

Architecture Contracts: between Architecture Board and implementation teams, with measurable conformance criteria (e.g., no workflows write dealstage except listed IDs).

Conformance assessments: Ameide checks actual config vs meta graph weekly.

Change control: Impact Simulator must show safe sequence (duplicate→backfill→swap→deprecate→delete) before schema changes.

Deviation waivers: formalize and log with expiry.

Outputs: Conformance reports; waivers; “go live” checklists; DR/BC plans.

Gate: Go‑live / promotion approvals per release.

Phase H — Architecture Change Management

Focus: Evolve safely by metrics.

Activities: Intake new requirements; analyze impact; update models; run re‑baseline quarterly; capture realized benefits; deprecate unused properties/workflows.

Ameide: ops.health (ingest lag/drift), governance_report (unused props, low fill, orphans), compare_baseline (dwell/conformance vs last quarter).

Outputs: Change backlog; revised standards; benefit realization reports.

Gate: Quarterly ARB review; updated roadmap.

Requirements Management (central, continuous)

Repository: User stories & non‑functionals with traceability to ArchiMate elements and KPIs.

Lifecycle: Prioritize → implement → test → verify impact (KPIs) → close or iterate.

Ameide: Each requirement links to graph elements and to Chat “checks” that verify success (e.g., fill‑rate ≥ 95% for practice).

Catalogs, Matrices, and Diagrams you’ll maintain (TOGAF deliverables)

Business

Capability Map (by practice)

Business Process catalog (pipelines/stages) + Stage Exit Criteria matrix

Organization/Role catalog; Role→Process Assignment matrix

Data

Data Object catalog (objects/properties/associations)

DQ rules catalog (fill‑rate thresholds, validation, dedupe keys)

Data Lifecycle & Retention matrix

Application

Application Portfolio catalog (HubSpot, PSA, ERP, e‑sig, BI)

Application Cooperation matrix (interfaces, direction, protocol)

CRUD matrix (which app is SoR / SoT per entity)

Technology

Technology Portfolio catalog (K8s, DBs, queues, observability)

Environment & Deployment diagrams (dev/sandbox/prod tenants)

NFR checklist (SLOs, capacity, security)

Implementation & Migration

Work Package portfolio; Dependency matrix; Benefit realization matrix; Migration Roadmap

Governance gates & RACI (ADM‑aligned)
Gate	Decision Body	Must show	Typical Tools
A.Vision	Exec sponsor + ARB	Vision, KPI tree, scope, risks	Vision deck; Capability Map
B.Bus.Arch	ARB + Practice leads	Pipelines & policy; role model	Process & Role views
C.Data/App	ARB + Security + RevOps	Schema, DQ, retention; interfaces & scopes	Data catalogs; Interface matrix
D.Tech	ARB + Platform	NFRs; deployment; observability	Deployment & SLO sheets
E.Options	ARB	Option analysis; transition arch	Option matrix; roadmap
F.Migrate	PMO + ARB	Work packages; costs/benefits; risks	Roadmap; RAID; benefits
G.Impl	ARB	Conformance; waivers; go-live checklists	Conformance report; Impact sims
H.Change	ARB	Benefits; drift; proposed changes	ops.health; governance_report

RACI (condensed)

A/B/C: ARB (A), RevOps (R), Practice Leads (C), Security (C).

D: Platform/Infra (R), ARB (A).

E/F: PMO (R), ARB (A), Finance (C).

G: ARB (A), Implementers (R).

H: ARB (A), All domain owners (R).

Event‑first posture inside ADM

Where events show up:

B defines conformance rules and SLAs (what you measure).

C freezes OCEL event schema and DQ rules.

D delivers the streaming platform and SLOs.

G enforces conformance (near real‑time alerts, drift checks).

H uses event trends to justify change (e.g., bottleneck removal).

Architecture Building Blocks (ABBs) you’ll reuse every time

OCEL event envelope (v1), idempotency & replay policy

ArchiMate mapping dictionary (HubSpot→ArchiMate)

Webhook validator (signature v3) & association resolver

DQ evaluator (fill‑rates, dedupe clusters)

Impact simulator (property/workflows/list/form dependencies)

Conformance analyzer (pipeline model vs observed transitions)

ops.health (lag, 429s, drift), governance_report (hygiene)

Solution Building Blocks (SBBs)

HubSpot public app (OAuth, optional scopes)

Ameide exporter pod (API + worker) on K8s

UI Extension in HubSpot (Ameide Assistant card)

PSA/ERP/iPaaS connectors

BI semantic layer (subject areas: Pipeline, Marketing, Support)

90‑day release plan aligned to ADM

A/B (Weeks 1–3): Vision; Business Architecture frozen (pipelines, roles, SLAs).

C/D (Weeks 3–6): Data/Application/Technology target states; OCEL schema; K8s pod deployed.

E/F (Weeks 6–8): Select solution; migration plan; work packages seeded.

G (Weeks 8–10): Implementation governance live; first release to prod; conformance checks enabled.

H (Weeks 10–13): Re‑baseline; benefits report; backlog reprioritized.

What changes versus a non‑ADM plan

Traceability: Every property, workflows, or stage change traces to a requirement and benefit in the repo.

Decision audit: ARB gates make “who approved what” explicit.

Reuse: ABB/SBB libraries reduce variance across practices/regions.

Risk control: Architecture Contracts + near real‑time conformance prevent silent drift.

Want the starter pack?

I can provide, in ADM format: (1) a Principle set (10 concise principles), (2) the ArchiMate mapping dictionary for HubSpot, (3) the OCEL v1 event schema, and (4) a minimal set of TOGAF catalogs/matrices pre‑templated for Ameide so your team can populate Phase A–D quickly.

You said:
How do you bind hubspot objects with graph so to make sure we don’t pollute the graph? How to we handle versions and as Is vs to be in the graphs 
ChatGPT said:
Request for GPT-5 Pro
Reasoned for 4m 47s

Great question. Below is a concrete, battle‑ready binding + versioning strategy that keeps your Ameide graph clean while giving you full as‑is vs to‑be control. It assumes:

ArchiMate meta graph for structure/governance

Operational (thin) graph for “current state” pointers

OCEL‑style event stream for detailed history (outside the graph)

Neo4j‑like capabilities for examples (adapt similarly for JanusGraph/Neptune)

1) Binding: how HubSpot objects become graph elements (without pollution)
1.1 Stable identity & namespacing (multi‑tenant)

Create a source‑scoped, tenant‑scoped external key for every HubSpot record:

ekey = "hs://" + portalId + "/" + objectType + "/" + objectId


Invariants

Every node/edge carries tenant (portalId) and source='hubspot'.

Never key on mutable attributes (email, name). Always on ekey.

Neo4j constraints (examples)

CREATE CONSTRAINT unique_record IF NOT EXISTS
FOR (r:Record) REQUIRE (r.tenant, r.ekey) IS UNIQUE;

CREATE CONSTRAINT unique_property IF NOT EXISTS
FOR (p:Property) REQUIRE (p.tenant, p.objectType, p.name) IS UNIQUE;

CREATE CONSTRAINT unique_stage IF NOT EXISTS
FOR (st:Stage) REQUIRE (st.tenant, st.pipelineId, st.stageId) IS UNIQUE;


Why this prevents pollution: duplicates can’t be created by replays or retries; attempts to upsert the same HubSpot entity fold into one node.

1.2 Whitelist mapping (“dictionary”), not ad‑hoc creation

Put a mapping dictionary between HubSpot and ArchiMate. The projector creates only what the dictionary allows.

YAML sketch

objects:
  contact:  {element: DataObject,  properties: [ownerId, lifecycle_stage] }
  company:  {element: DataObject,  properties: [domain, industry, region] }
  deal:     {element: DataObject,  properties: [dealstage, pipeline, amount] }
  ticket:   {element: DataObject,  properties: [hs_pipeline_stage, priority] }
associations:
  - from: deal
    to: company
    hsTypeId: 5
    edge: ASSOCIATED
    attrs: [label]         # only store label if present
pipelines:
  deal:
    element: BusinessProcess
    stageElement: BusinessProcess # or BusinessState, but pick one and standardize


Behavior

Anything not in the dictionary goes to quarantine (a log/queue), not the graph.

You can update the dictionary via change control; Ameide Chat can surface “unknowns”.

1.3 Minimalism for the operational graph

To avoid bloat:

Records: store only ekey, objectType, and a handful of current pointers (e.g., current stage).

Do not dump every property into the graph. Detailed history lives in the event store.

Edges represent relationships and current state pointers, not raw events.

Record node example

MERGE (r:Record {tenant:$t, ekey:$ekey, objectType:$type})
SET r.updatedAt = $ts;           // last seen

2) Temporal modeling & versions (no more graph drift)
2.1 Bitemporal edges (validity intervals)

Use time‑bounded edges to model changing states and associations:

[:IN_STAGE {from, to, eventId, sysFrom, sysTo}]

[:ASSOCIATED {typeId, label, from, to, eventId, sysFrom, sysTo}]

Fields

from/to = business (event) time from HubSpot webhook/event

sysFrom/sysTo = system time when Ameide wrote/rewrote the edge (for audit)

Upsert pattern (idempotent)

// Close previous interval if open
MATCH (d:Record {tenant:$t, ekey:$deal})
MATCH (st:Stage {tenant:$t, stageId:$toStage})
OPTIONAL MATCH (d)-[rel:IN_STAGE]->(:Stage)
WHERE rel.to IS NULL
SET rel.to = $eventTs, rel.sysTo = timestamp()

// Open new interval
MERGE (d)-[new:IN_STAGE {eventId:$evId}]->(st)
SET new.from = $eventTs, new.sysFrom = timestamp(), new.to = NULL, new.sysTo = NULL


Why this helps: you can time‑travel (“as of date X”) or compute dwell time precisely without overwriting past states.

2.2 Meta graph versioning (configuration changes)

Treat configuration (pipelines, stages, properties) as SCD‑Type‑2:

Add version, validFrom, validTo on meta nodes (Property, Stage, Pipeline).

When HubSpot config changes, close the old meta node (validTo) and create a new one (validFrom=now, version+1).

Property example

MATCH (p:Property {tenant:$t, objectType:$o, name:$name})
WHERE p.validTo IS NULL
SET p.validTo = $ts;

CREATE (p2:Property {tenant:$t, objectType:$o, name:$name, version:p.version+1,
                     fieldType:$fieldType, validFrom:$ts, validTo:NULL})

2.3 Snapshots and diffs

Create snapshot anchors for “as‑is” at points in time:

MERGE (s:Snapshot {tenant:$t, id:$snapId})
SET s.createdAt = timestamp(), s.note = $note


Link snapshot to the versions of meta nodes valid at that time ((:Snapshot)-[:INCLUDES]->(:Property {validFrom<=snapTs<validTo}) etc.). Diffs become simple set comparisons between snapshots.

3) As‑Is vs To‑Be: layered contexts, safe promotion
3.1 Context tags (or named graphs)

Every node/edge carries a ctx (context) tag:

ctx='ASIS' for synchronized, production reality

ctx='TOBE/<scenarioId>' for design scenarios

To‑Be lives alongside As‑Is but never mixes: context is part of all query predicates and unique keys.

Constraints

CREATE CONSTRAINT unique_meta_version IF NOT EXISTS
FOR (p:Property) REQUIRE (p.tenant, p.objectType, p.name, p.version, p.ctx) IS UNIQUE;


Queries

As‑Is: MATCH ... WHERE ctx='ASIS'

Scenario: MATCH ... WHERE ctx='TOBE/scenario-42'

3.2 Scenario creation & promotion pipeline

Fork: copy the As‑Is meta subgraph into ctx='TOBE/scenario-42'.

Edit: architects change pipelines, properties, rules in the To‑Be context.

Impact simulate: resolve downstream dependencies (workflows/lists/forms) against To‑Be.

Approve: Architecture Board gate.

Promote:

Apply config to HubSpot (if desired) via API, or

Advance As‑Is meta to new versions mirroring the approved To‑Be.

Archive: mark scenario closed.

Key guardrail: operational Record nodes remain AS‑IS only. We do not model hypothetical records.

4) Hygiene guards to prevent graph pollution

Allow‑list dictionary (above): only mapped elements are created.

Schema validation: webhook payloads must match JSON Schemas (OCEL v1). Invalids → DLQ, never the graph.

Quarantine bucket: unknown object types/associations are logged for review, not ingested.

TTL policies: if you choose to mirror short‑lived things (e.g., messages), add TTL or archive to the event store after N days.

Orphan sweeps: scheduled job finds records with no required associations (deal without company) and raises governance alerts (not auto‑deleting).

Merge handling: when HubSpot emits merge (e.g., contacts), keep a canonical node and create (:Record)-[:ALIAS_OF]->(:Record) for the loser; rewire edges lazily on access or eagerly via a rewire job.

Soft deletes: when HubSpot deletes a record, close intervals and stamp deletedAt; don’t hard‑delete unless policy dictates.

5) Putting it all together (practical patterns)
5.1 Record binding (idempotent upsert)
// Upsert record shell
MERGE (r:Record {tenant:$t, ekey:$ekey, objectType:$type, ctx:'ASIS'})
ON CREATE SET r.createdAt = $ts
SET r.updatedAt = $ts

5.2 Association change → time‑bounded edge
MATCH (a:Record {tenant:$t, ekey:$from, ctx:'ASIS'})
MATCH (b:Record {tenant:$t, ekey:$to,   ctx:'ASIS'})
// close previous if removing
FOREACH (x IN CASE WHEN $op='remove' THEN [1] ELSE [] END |
  MATCH (a)-[r:ASSOCIATED {typeId:$typeId, label:$label}]->(b)
  WHERE r.to IS NULL
  SET r.to=$eventTs, r.sysTo=timestamp()
)
// open new if adding
FOREACH (x IN CASE WHEN $op='add' THEN [1] ELSE [] END |
  MERGE (a)-[r:ASSOCIATED {typeId:$typeId, label:$label, eventId:$evId}]->(b)
  ON CREATE SET r.from=$eventTs, r.sysFrom=timestamp()
)

5.3 “As of” query (time travel)
MATCH (d:Record {tenant:$t, ekey:$deal, ctx:'ASIS'})
MATCH (d)-[r:IN_STAGE]->(st:Stage {tenant:$t, ctx:'ASIS'})
WHERE r.from <= $asOf AND (r.to IS NULL OR r.to > $asOf)
RETURN st.name AS stage;

5.4 Snapshot diff (meta)
// Properties present in B but not A
MATCH (sb:Snapshot {tenant:$t, id:$B})-[:INCLUDES]->(p:Property {ctx:'ASIS'})
WHERE NOT EXISTS {
  MATCH (:Snapshot {tenant:$t, id:$A})-[:INCLUDES]->(p2:Property)
  WHERE p2.objectType=p.objectType AND p2.name=p.name AND p2.version=p.version
}
RETURN p.objectType, p.name, p.version;

6) What goes where (clear separation of concerns)
Concern	Store	Why
Event history (full detail)	Event stream/index (OCEL)	Cheap append, replayable, near‑real‑time
Current state pointers (stage, associations)	Operational graph (time‑bounded edges)	Fast “what is now” and time travel
Configuration & design (pipelines, properties)	Meta graph (versioned nodes)	Governance, impact analysis, diffs
Scenarios (to‑be)	Meta graph with ctx='TOBE/<id>'	Safe design sandbox
7) Versioning policy (simple and enforceable)

Events: immutable; ocel.event.v1 in a schema registry.

Operational edges: append‑only with interval close/open; no overwrites (preserves history).

Meta nodes: SCD‑2 (version, validFrom, validTo); never mutate in place.

Scenarios: separate ctx; no mixing with AS‑IS; promotion is a new meta version (with a snapshot).

8) Governance hooks for Ameide Chat

ops.health(tenant) → alerts if quarantine grows, DLQ fills, or open intervals exceed thresholds.

graph.schema_diff(A,B) → supports change boards.

impact.simulate_property_change(objectType, property) → computes downstream risk against meta graph only (no touching operational data).

asof(stage_dwell, dateRange) → does time‑travel queries over operational edges.

Summary

Bind HubSpot ➜ ArchiMate with strict external keys, allow‑listed mapping, and unique constraints to prevent duplication.

Keep the graph clean by separating meta (versioned) and operational (time‑bounded) layers; push raw history to the event store.

Manage versions with SCD‑2 on meta nodes and bitemporal edges on operational links.

Support as‑is vs to‑be using context tags (or named graphs), with a safe promotion pipeline and snapshotting for diffs.

If you’d like, I can produce a small reference repo with: the mapping dictionary scaffold, JSON Schemas for events, and the Cypher migrations to create the constraints and example procedures above.
---

## 123-billing.md (old)

Here’s the quick way to think about it:

* The **Vercel Chatbot** template on GitHub is a Next.js app built on the **Vercel AI SDK**; you hook your token metering right where the SDK streams or finishes a response. ([GitHub][1])
* The AI SDK already **exposes token usage** for you; you just need to read it and store it. Use `onFinish` / `onStepFinish` (and optionally `onAbort`) or the `totalUsage` helper. ([ai-sdk.dev][2])

Below are the three practical ways to measure tokens with this stack (plus copy-pasteable snippets).

---

# 1) Server-side (recommended): record usage from the AI SDK

### A. In your threads route (e.g. `app/api/threads/route.ts`)

```ts
import { openai } from '@ai-sdk/openai';
import {
  streamText,
  convertToModelMessages,
  type UIMessage,
} from 'ai';
import { db } from '@/lib/db'; // your persistence
import { usageLog } from '@/lib/schema'; // your table

export async function POST(req: Request) {
  const { messages, tenantId }: { messages: UIMessage[]; tenantId: string } =
    await req.json();

  const result = streamText({
    model: openai('gpt-4o'),           // any provider works here
    messages: convertToModelMessages(messages),
    // Called if the stream is aborted (user closed tab, clicked stop, etc.)
    onAbort: async ({ steps }) => {
      // Optional: sum per-step usage to capture partial consumption
      const totals = steps.reduce(
        (acc, s) => {
          const u = s.usage ?? { promptTokens: 0, completionTokens: 0, totalTokens: 0 };
          acc.promptTokens += u.promptTokens ?? u.inputTokens ?? 0;
          acc.completionTokens += u.completionTokens ?? u.outputTokens ?? 0;
          acc.totalTokens += u.totalTokens ?? 0;
          return acc;
        },
        { promptTokens: 0, completionTokens: 0, totalTokens: 0 },
      );
      if (totals.totalTokens > 0) {
        await db.insert(usageLog).values({
          tenantId,
          model: 'openai/gpt-4o',
          promptTokens: totals.promptTokens,
          completionTokens: totals.completionTokens,
          totalTokens: totals.totalTokens,
          wasAborted: true,
        });
      }
    },
  });

  // Stream the response to the client and record *final* usage on completion:
  return result.toUIMessageStreamResponse({
    async onFinish({ totalUsage, isAborted }) {
      if (isAborted) return; // already handled in onAbort
      const { promptTokens, completionTokens, totalTokens } = totalUsage;
      await db.insert(usageLog).values({
        tenantId,
        model: 'openai/gpt-4o',
        promptTokens,
        completionTokens,
        totalTokens,
        wasAborted: false,
      });
    },
  });
}
```

* `onFinish` receives **`totalUsage`** (sum across multi-step/tool-calling runs) so your numbers are correct even when the model calls tools. In AI SDK v5, `usage` is per last step; use **`totalUsage`** for billing. ([ai-sdk.dev][3])
* If the user **aborts** a stream, `onFinish` may not fire; use **`onAbort`** and sum `steps[i].usage` to capture partial usage. ([ai-sdk.dev][3])

> Docs that show this pattern and the shape of the usage object (prompt/completion/total): ([ai-sdk.dev][2])

---

# 2) Pre-flight estimates (before you send the request)

Sometimes you want to **estimate** tokens to enforce limits or quote costs before sending:

* **OpenAI-family models:** use a local tokenizer (e.g., `tiktoken` or `gpt-tokenizer`) to estimate context tokens.

  ```ts
  import { encode } from 'gpt-tokenizer';
  const estimate = encode(text).length; // rough estimate for GPT-style models
  ```

  (Estimates can differ slightly from billed usage.) ([OpenAI Cookbook][4])

* **Anthropic (Claude):** they provide an official **Count Tokens API**—ask for token count without generating. ([Anthropic][5])

* **Google Gemini (Vertex/AI Studio):** use **CountTokens** to precompute input tokens. ([Google Cloud][6])

Pre-flight is great for **guardrails**, but for **billing** you should still record the **server-reported usage** from §1.

---

# 3) Zero-code metering via **Vercel AI Gateway** (optional but nice)

If you route your AI calls through **Vercel AI Gateway**, you’ll automatically get **Input/Output Token Counts** and spend charts in the Vercel dashboard. Two easy ways:

* In AI SDK v5, you can specify models as plain strings (e.g., `'openai/gpt-4o'`) and set `AI_GATEWAY_API_KEY`. The SDK speaks to the Gateway provider and you get token metrics in **Observability → AI Gateway**. ([Vercel][7])
* Or create a Gateway provider with `@ai-sdk/gateway` and use it explicitly. Default base URL: `https://ai-gateway.vercel.sh/v1/ai`. ([ai-sdk.dev][8])

This is a solid complement to your own DB logs (nice for dashboards and cross-project rollups). ([Vercel][9])

---

## Where this plugs into the **vercel/ai-threadsbot** repo

In that template, the AI call happens in the server layer that streams threads responses (a Next.js route or server action that calls `streamText`). Add the `onFinish` / `onAbort` logic shown above and persist usage next to your messages table. ([GitHub][1])

---

## TL;DR billing flow you can ship

1. **On every response**: read `totalUsage` in `onFinish` and insert `{tenantId, model, promptTokens, completionTokens, totalTokens}` into a `usage_ledger`. Handle `onAbort` for partial usage. ([ai-sdk.dev][3])
2. **Convert tokens → credits** using your price map per model (e.g., tokens / 1000 × price) and **debit** the customer’s prepaid balance.
3. (Optional) **Pre-flight** check with a tokenizer / provider count-tokens endpoint to reject/trim prompts that would exceed the customer’s balance. ([OpenAI Cookbook][4], [Anthropic][5], [Google Cloud][6])
4. (Optional) Turn on **Vercel AI Gateway** for a second source of truth and dashboards. ([Vercel][9])

If you want, I can sketch the minimal Drizzle table + a helper `recordUsage(tenantId, totalUsage, model)` wired into your route.

[1]: https://github.com/vercel/ai-threadsbot "GitHub - vercel/ai-threadsbot: A full-featured, hackable Next.js AI threadsbot built by Vercel"
[2]: https://ai-sdk.dev/cookbook/rsc/stream-ui-record-token-usage "React Server Components: Record Token Usage after Streaming User Interfaces"
[3]: https://ai-sdk.dev/docs/advanced/stopping-streams?utm_source=threadsgpt.com "Advanced: Stopping Streams"
[4]: https://cookbook.openai.com/examples/how_to_count_tokens_with_tiktoken?utm_source=threadsgpt.com "How to count tokens with Tiktoken"
[5]: https://docs.anthropic.com/en/api/messages-count-tokens?utm_source=threadsgpt.com "Count Message tokens"
[6]: https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/list-token?utm_source=threadsgpt.com "List and count tokens | Generative AI on Vertex AI"
[7]: https://vercel.com/docs/ai-gateway/authentication?utm_source=threadsgpt.com "Authentication"
[8]: https://ai-sdk.dev/providers/ai-sdk-providers/ai-gateway?utm_source=threadsgpt.com "AI Gateway Provider"
[9]: https://vercel.com/docs/ai-gateway/observability?utm_source=threadsgpt.com "Observability"

---

## 124-hubspot-integration-core.md (old)

> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# 124: HubSpot Integration Core - OAuth App & Webhook Infrastructure

## Status
**Status**: 🔴 Not Started  
**Priority**: P1 - Critical  
**Size**: L (3-5 sprints)  
**Type**: Feature  

## Context
Build the foundational HubSpot integration infrastructure including OAuth marketplace app, webhook processing, and multi-tenant event streaming following AMEIDE architecture patterns.

## Architecture Alignment

### Package Structure
- **`packages/int_hubspot/`** - Pure business logic for HubSpot integration
  - `/webhook/` - Webhook verification and event processing logic
  - `/oauth/` - OAuth flow and token management protocols
  - `/models/` - HubSpot data models and types
  - `/config/` - Integration configuration and mappings

### Service Structure  
- **`services/int_hubspot/`** - Runtime infrastructure
  - HTTP server for webhook endpoints
  - Keycloak integration for OAuth token storage
  - Kafka producer for event streaming
  - Health checks following `core-platform-common` patterns

## Requirements

### OAuth Integration with Keycloak
- [ ] HubSpot OAuth app registration and configuration
- [ ] Store OAuth tokens as Keycloak user attributes (encrypted)
- [ ] Multi-tenant token management via Keycloak realms/groups
- [ ] Service-to-service auth using Keycloak client credentials
- [ ] Token refresh mechanism with automatic retry

### Webhook Infrastructure
- [ ] Webhook endpoint at `/webhooks` with X-HubSpot-Signature-V3 verification
- [ ] Pure verification logic in `packages/int_hubspot/webhook/`
- [ ] Event normalization to OCEL format in package layer
- [ ] Kafka publishing to `hubspot.events` topic
- [ ] Dead Letter Queue (DLQ) for poison events
- [ ] Backpressure handling (ack fast, process async)

### Multi-Tenancy
- [ ] Tenant isolation using HubSpot portalId
- [ ] Per-tenant Kafka partitions
- [ ] Row-level security in graph/event stores
- [ ] Per-tenant rate limiting with token buckets
- [ ] Keycloak-based tenant management

### Kubernetes Deployment
- [ ] Helm chart: `infra/kubernetes/charts/platform/int-hubspot/`
- [ ] HTTPRoute: `hubspot.dev.ameide.io` → `services/int_hubspot:8080`
- [ ] Standard health checks (`/health`, `/ready`)
- [ ] Observability (OpenTelemetry metrics, structured logging)
- [ ] SLOs: p95 webhook-to-kafka < 60s, zero data loss

## Technical Decisions
- **Package-Service Separation**: Business logic in `int_hubspot` package, runtime in service
- **Auth**: Keycloak for all OAuth token storage (no separate PostgreSQL storage)
- **Events**: Existing Kafka cluster (`kafka-kafka-bootstrap:9092`)
- **Gateway**: Envoy Gateway HTTPRoute for webhook ingress
- **Storage**: Event store in `postgres-event-rw`, projections in `postgres-platform-rw`

## Dependencies
- Keycloak (`keycloak:443`) - OAuth token storage
- Kafka (Strimzi-managed) - Event streaming
- PostgreSQL clusters - Event and platform databases
- Neo4j (`neo4j:7687`) - Graph storage
- Envoy Gateway - Webhook ingress

## Acceptance Criteria
- [ ] OAuth flow integrates with Keycloak identity management
- [ ] Webhooks are verified using pure package logic
- [ ] Events flow through Kafka with proper partitioning
- [ ] Multi-tenant isolation verified with test tenants
- [ ] Rate limiting prevents 429 errors
- [ ] Grafana dashboard shows key metrics
- [ ] Uninstall cleans up Keycloak tokens and tenant data

## References
- HubSpot API Documentation
- AMEIDE Package-Service Architecture (`packages/README.md`)
- Keycloak Integration Pattern (`services/www_ameide_platform/README.md`)
- Ameide-HubSpot-Blueprint/02_System-Architecture-and-Deployment.md

---

## 128-bpmn-artifact-v3-overview.md (old)

# V3 — Server-Gated BPMN Editor (Single-User, No Collaboration)

## 1. Scope & Goals
- Build a **server-authoritative** BPMN 2.0 editor.
- **All edits are gated** by backend; UI applies only **accepted** changes.
- Maintain **XSD compliance** (OMG BPMN 2.0) and **bpmnlint** best-practice checks.
- Persist **event-sourced semantics** with **separate layout** stream.
- Expose clean **gRPC-Web → Envoy → gRPC** boundary.
- Provide **read models** (PostgreSQL + Graph) and **Agent HITL** suggestions.

## 2. Non-Goals
- No real-time multi-user collaboration (no CRDT/Yjs).
- No server DOM rendering; **BpmnModeler is never run on server**.

## 3. Principles
- **Backend is source of truth.**
- **Pessimistic gating**: UI never mutates model without server permit.
- **One user action = one atomic transaction** (version++ once).
- **Semantics and layout are distinct concerns** (dual stream).
- **Spec before style**: XSD is authoritative; lint is advisory.

## 4. High-Level Architecture
```
Browser (bpmn-js)
├─ Rules Gate (canExecute) → blocks local command
├─ CommandProposal (gRPC-Web)
▼
Envoy (gRPC-Web ↔ gRPC)
▼
Command Service (gRPC)
├─ AuthZ / Tenancy
├─ Validate (XSD + bpmnlint + domain)
├─ Event Store (semantic v+1; layout base_v)
├─ Moddle Projection → XML
├─ Snapshots
└─ Ack: AppliedTransaction {version, id_map, applied[], [xml?]}
    ▲
    └─ Agents (HITL proposals) share the same API
```

## 5. Data Views
- **Event Store**: append-only semantic/layout events.
- **Read Models**
  - SQL (elements, flows, stats) for simple queries.
  - Graph (Neo4j/AGE) for paths/compliance/context.

## 6. UX Contract (summary)
- While pending: **ghost/preview** only, no canvas commit.
- On **accept**: UI **applies** server-returned operations/IDs/positions.
- On **reject**: UI shows reason; **no change** to canvas.
---

## 129-bpmn-artifact-v3-adrs.md (old)

# ADRs — V3

## ADR-001: Server-Gated Commands (Pessimistic)
**Decision**: Client sends CommandProposal; server validates/commits; UI applies only AcceptedTransaction.  
**Rationale**: Eliminate local/remote divergence and rebase complexity.

## ADR-002: No Collaboration / No CRDT
**Decision**: Single-user workflows; no Yjs/OT.  
**Rationale**: Reduce moving parts; authority stays server-side.

## ADR-003: Dual Streams (Semantic vs Layout)
**Decision**: Semantic events advance `version`; layout events reference `base_version = version_at_commit`.  
**Rationale**: Clean separation of business model vs DI/waypoints.

## ADR-004: Version per Transaction
**Decision**: One atomic batch per user action increments version once.  
**Rationale**: Predictable undo unit and concurrency control.

## ADR-005: Spec Compliance via XSD
**Decision**: Python `lxml` validates BPMN XSD; `bpmnlint` is advisory.  
**Rationale**: XSD defines correctness; lint improves ergonomics.

## ADR-006: Moddle-Only Server Projection
**Decision**: Use `bpmn-moddle` sidecar for deterministic model manipulation/export.  
**Rationale**: No DOM; stays compatible with BPMN tooling.

## ADR-007: ID Authority & Rewrites
**Decision**: Server owns final IDs; ack returns `id_map` for client remap.  
**Rationale**: Prevent collisions, ensure consistency.

## ADR-008: Transport
**Decision**: Browser uses **gRPC-Web**; Envoy bridges to internal gRPC.  
**Rationale**: Strong contracts, streaming support, standardized edge.

## ADR-009: Read Models
**Decision**: SQL for denormalized lookups; Graph for traversal/compliance; both updated on accept.  
**Rationale**: Fit-for-purpose query latency.

## ADR-010: Agents HITL First
**Decision**: Agents submit proposals; humans approve. Same API path; server validates.  
**Rationale**: Safety, auditability, gradual autonomy later.
---

## 130-bpmn-artifact-v3-interfaces.md (old)

# Interfaces & Schemas (contract-level)

## 1) gRPC Service (proto sketch)
```proto
service BpmnCommandService {
  rpc Propose(CommandProposal) returns (AppliedOrRejected);
  rpc GetSnapshot(GetSnapshotRequest) returns (Snapshot);
  rpc GetEvents(GetEventsRequest) returns (GetEventsResponse); // debug/ops
}
```

## 2) Messages (essential fields only)

### CommandProposal (client → server)
- `tenant_id`, `diagram_id`, `actor_id`
- `expected_version` (uint64)
- `commands[]` (atomic batch)
  - Semantic: `CreateElement`, `UpdateProperties`, `CreateConnection`, `UpdateLabel`, `DeleteElements`
  - Layout: `MoveElement`, `ResizeElement`, `UpdateWaypoints`
- *Notes*: client IDs/positions are suggestions; server may rewrite.

### AppliedOrRejected (server → client)
- oneof:
  - **AppliedTransaction**
    - `version` (new semantic version)
    - `applied[]` (ordered list of canonical events: semantic first, then layout for that version)
    - `id_map {client_id -> server_id}`
    - `warnings[]` (lint or non-fatal notes)
    - optional `xml_snapshot` (size-guarded)
  - **Rejection**
    - `reason_code` (e.g., `XSD_ERROR`, `LINT_FAIL`, `SEM_RULE_VIOLATION`, `VERSION_MISMATCH`, `AUTHZ_DENIED`)
    - `details[]`

### EventEnvelope (store)
- `stream_id` = `tenant-{tid}:diagram-{id}:{semantic|layout}`
- `stream_seq` (monotonic per stream)
- `global_seq` (store total order)
- `semantic_version` (only for semantic)
- `base_version` (only for layout)
- `ts_unix_ms`, `origin {type: human|agent, id}`
- `payload` (typed event)

## 3) Canonical Event Types (payload)

**Semantic**
- `CreatedElement {id, type, parent_id, props{}}`
- `UpdatedProperties {id, changes{}}`
- `CreatedConnection {id, type, source_id, target_id, props{}}`
- `UpdatedLabel {id, text}`
- `DeletedElements {ids[]}`

**Layout**
- `MovedElement {id, x, y, parent_id?}`
- `ResizedElement {id, width, height}`
- `UpdatedWaypoints {id, points[]}`

## 4) Read Models (minimal schemas)

### SQL (PostgreSQL)
- `bpmn_elements(diagram_id, element_id, element_type, name, parent_id, properties JSONB, version, created_at, updated_at)`
- `bpmn_flows(diagram_id, flow_id, source_id, target_id, condition_expression, condition_language)`
- `bpmn_statistics(diagram_id PK, task_count, gateway_count, event_count, flow_count, last_updated)`

### Graph (Neo4j/AGE)
- Nodes: `(:Process) (:Task) (:Gateway) (:Event) (:DataObject)`
- Rels: `(:A)-[:SEQUENCE_FLOW {id, condition}]->(:B)`, `(:Event)-[:BOUNDARY_OF {interrupting}]->(:Task)`, etc.

## 5) Validation Outputs
- **XSD**: `valid: bool`, `errors[] {line, col, message}`
- **Lint**: `errors[] {rule, message, element_id}`, `warnings[]`

## 6) Security / Tenancy (header fields)
- `x-tenant-id`, `x-actor-id`, `x-request-id`, `x-run-id` (loop prevention)
---

## 131-bpmn-artifact-v3-flows.md (old)

# Processing & Flows

## 1) UI Gate & Apply (single user)
- **Rules Gate** (bpmn-js `RuleProvider` / `commandStack.*.canExecute`):
  - Gate: `shape.create`, `elements.move`, `connection.create`, `connection.reconnect`, `shape.resize`, `elements.delete`.
  - Default deny unless a short-lived **permit** exists (set upon accept).
- **Proposal**: Build CommandProposal (batch), send via gRPC-Web.
- **Pending UX**: ghost/preview only (no model write).
- **Ack Handling**:
  - **Applied**: remap IDs via `id_map`, then **programmatically apply** operations through the modeling API (one undo step).
  - **Rejected**: show toast with `reason_code`; do nothing to canvas.

## 2) Validation Pipeline (server)
1) **AuthZ & Tenancy** → reject early if needed.  
2) **Version check** (`expected_version == current`) → else `VERSION_MISMATCH`.  
3) **Semantic checks** (domain rules).  
4) **Projection (moddle)** to temporary model; materialize both semantic and layout.  
5) **Export XML**; **XSD validate** (authoritative).  
6) **Lint** (advisory); attach warnings.  
7) **Commit** to Event Store (semantic v+1; layout base_v=v+1); write snapshot if thresholds met.  
8) **Return AppliedTransaction** (final IDs/positions, version, warnings [,xml?]).

## 3) Ordering & Merge Rules
- **Within a transaction**:
  - Apply **semantic** first ⇒ `version = v+1`.
  - Apply **all layout** with `base_version = v+1`.
- **Across transactions**:
  - Consumers replay semantic by version asc; interleave layout when `base_version` matches.

## 4) Sequences (mermaid)

### A) Accept
```mermaid
sequenceDiagram
autonumber
actor U
participant UI as bpmn-js (Rules Gate)
participant W as Envoy (gRPC-Web)
participant S as Command Service
participant E as Event Store
participant P as Moddle Projection
participant X as XSD/Lint

U->>UI: Drag/Create (intent)
UI->>UI: canExecute? → DENY (no local commit)
UI->>W: CommandProposal
W->>S: forward
S->>S: AuthZ + Version + Domain
S->>P: Apply to moddle (temp)
P-->>S: Temp model
S->>X: XSD + Lint
X-->>S: OK (+warnings)
S->>E: Append semantic (v+1) + layout (base_v=v+1)
E-->>S: Committed
S-->>W: AppliedTransaction {version, id_map, applied[]}
W-->>UI: Ack
UI->>UI: Apply programmatically; one undo step
```

### B) Reject
```mermaid
sequenceDiagram
autonumber
U->>UI: Edit intent
UI->>W: CommandProposal
W->>S: forward
S->>S: AuthZ/Version/Domain/XSD
S-->>W: Rejection {reason_code, details}
W-->>UI: Rejection
UI->>UI: Show error; no change
```

## 5) Snapshotting & Retrieval
- **Write snapshot** when: events ≥ N, or Δt ≥ T, or major change.
- **GetSnapshot** returns `{xml, version, created_at}`.
- Optionally include snapshot in the **accept ack** when a boundary is crossed.

## 6) Agents (HITL)
- Agents send **CommandProposals** with `origin.type=agent`.
- Same validation and commit path.
- UI receives an **AppliedTransaction** only when accepted by reviewer (HITL gate is policy-controlled in server).

## 7) Failure Modes & UI Behavior
- **Network**: keep pending; allow cancel; timeouts show retry UI.
- **VERSION_MISMATCH**: UI fetches snapshot; prompts user to retry action.
- **XSD/Lint fail**: show rule message; no canvas change.

## 8) Performance Notes
- UI: propose on **dragend** or throttle to 150–300 ms; accept **last ack** only.
- Server: batch layout ops; compress waypoints; defer heavy lint rules behind size guard.
---

## 132-bpmn-artifact-v3-deployment.md (old)

# Deployment, Ops & Security

## 1) Components
- **Web App**: bpmn-js editor + Rules Gate + gRPC-Web client.
- **Envoy**: terminates gRPC-Web; routes to internal gRPC.
- **Command Service**: validation, projection, event append, snapshotting.
- **Moddle Sidecar**: `bpmn-moddle` service (stateless).
- **XSD Validator**: Python `lxml` service (stateless).
- **Event Store**: append-only (e.g., Postgres streams / Kafka+DB).
- **Read Models**: PostgreSQL + Neo4j/AGE.
- **Agent Runtime**: submits proposals (HITL by policy).

## 2) Envoy (edge) — essentials
- HTTP/2, gRPC-Web filter enabled.
- mTLS to backend gRPC.
- Headers propagated: `x-tenant-id`, `x-actor-id`, `x-request-id`, `x-run-id`.

## 3) Scalability
- Scale **Command Service** horizontally; enforce idempotency via `transaction_id`.
- Make **Moddle/XSD** sidecars autoscalable; cold-start tolerant.
- Snapshots reduce replay time; tune thresholds per diagram size.

## 4) Observability
- **Structured logs** with `diagram_id`, `tenant_id`, `version`, `transaction_id`.
- **Metrics**: accept latency (p50/p95), XSD error rate, lint warning density, event append rate, snapshot size.
- **Tracing**: proposal → validation → append → ack (W3C tracecontext).

## 5) Security & Tenancy
- **AuthN**: OIDC/JWT at edge; short-lived tokens.
- **AuthZ**: RBAC/ABAC evaluated in Command Service.
- **Row-level** or stream-level isolation per `tenant_id`.
- **Input validation**: schema-checked proposals; hard caps on batch size, XML size, waypoint counts.

## 6) Backups & Recovery
- Event store is primary; schedule **logical backups** of streams.
- Periodic **snapshot archival** (compressed XML).
- Disaster recovery: rebuild read models from streams + latest snapshot.

## 7) Compatibility & Migration
- Version events; support **projection version** flag.
- Migrate layout independently of semantics.
- Keep **id_map** behavior stable across releases.

## 8) SLAs / SLOs (suggested)
- Accept latency p95 ≤ 300 ms for small changes.
- XSD validation budget ≤ 80 ms average (<100 KB XML).
- Snapshot write ≤ 500 ms p95 (async).

## 9) Test Strategy (architectural)
- **Contract tests**: Propose → Applied/Rejected (codes, id_map, version).
- **Projection fidelity**: round-trip events → XML → XSD valid.
- **Rules Gate**: canExecute denies without permit; allows with permit.
- **Throughput**: sustained append & snapshot under load.

## 10) Rollout Plan (phases)
1) **Gate & Transport**: rules gate + Propose/Ack path live.
2) **Validation & Store**: XSD+lint pipeline, event append, snapshots.
3) **Apply Strategy**: programmatic modeling path stable; one undo step/tx.
4) **Read Models**: SQL + Graph consumers; analytics endpoints.
5) **Agents (HITL)**: proposal submission, reviewer approval, audit logs.

## 11) Operational Runbook (abridged)
- **Hot issue**: `VERSION_MISMATCH` spikes → investigate concurrent edits; adjust UX to propose on dragend.
- **XSD error bursts**: check model templates, engine-specific profiles.
- **Slow accept**: profile moddle sidecar and XSD service; scale; enable snapshot in ack for large diffs.
---

## 133-bpmn-artifact-v3-gaps.md (old)

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

---

## 134-text-editor-v3-gaps.md (old)

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
---

## 135-bpmn-v3-optimistic-overview.md (old)

# V3 (Optimistic) — BPMN Editor with In-Editor AI, Periodic Snapshots, Save-Time Versioning

**Status**: SDK Complete, Backend Implementation Pending (2025-08-20)
**Architecture Decision**: Use Protobuf/gRPC/Connect stack instead of REST/OpenAPI (see 141-core-api-decision.md)

## Scope
- **No backend gating**: bpmn-js applies edits immediately.
- **Snapshots**: auto-saved at configurable intervals (drafts).
- **Versioning**: bump version only on **explicit Save**.
- **AI collaboration in the editor**: agent suggests/applies edits via the same command stack.
- Keep **XSD compliance** at Save; **bpmnlint** in-editor for instant feedback.

## High-Level Flow
```
User edit → bpmn-js updates canvas (local)
│
├─ Autosnapshot (draft) every N seconds / idle Ms
│
Explicit Save → Build XML → Client prelint (optional) → Send to backend
→ Server XSD validate → Persist new version → Return commit metadata
```

## Components (logical)
- **Editor (browser)**: bpmn-js, Lint Overlay, Command Capture, AutoSnapshot, AI Plugin.
- **Backend (services)**: Save API (XSD validation, persistence, versioning), optional Read Models (SQL/Graph).
- **AI Runtime**: invoked by editor (through a server proxy), returns **structured suggestions/commands**.

## Key Properties
- **Latency**: Edits are instant; network used for autosnap upload (optional) and Save.
- **Safety**: Validation gates apply at Save; AI suggestions are **local** and reversible (undo/redo).
- **Simplicity**: No CRDT/OT; single active editor assumed (or last-write-wins on Save).

## Non-Goals
- Realtime multi-user collaboration.
- Server-side rendering/modeler.

## MVP Definition (4 weeks)

### Phase 1: Core Editor Enhancement (Week 1) ✅ COMPLETE
**Goal**: Functional BPMN editor with local persistence
- ✅ Current bpmn-js integration (fully working in platform)
- ✅ BpmnJSAdapter bridging bpmn-js with SDK operations
- ✅ Dynamic imports to prevent SSR/chunk loading issues
- ✅ Command capture via event bus interception
- ✅ Auto-save with configurable intervals (30s default)

**Implementation Complete** (2025-08-21):
- ✅ Created `bpmn_editor_service.proto` with streaming support
- ✅ Defined operation-based delta system for efficient updates
- ✅ Added real-time collaboration support (WatchDraft streaming)
- ✅ Generated TypeScript types (protoc-gen-es v2)
- ✅ ameide_sdk_ts exposes raw BPMNEditorService (no wrappers)
- ✅ Mock transport with realistic BPMN XML generation
- ✅ SDK tests complete with comprehensive coverage
- ✅ SDK fully aligned with Connect v2 best practices
- ✅ BpmnJSAdapter implementation with full operation mapping
- ✅ Integration in www_ameide_platform with proper providers
- ✅ Fixed all Next.js/webpack issues for bpmn-js modules
- ⏳ Backend service handlers pending (core_grpc package)

### Phase 2: Backend Save & Versioning (Week 2)
**Goal**: Server persistence with version control
- [ ] ~~POST /diagrams/{id}/save endpoint~~ → Using gRPC SaveDraft instead
- [ ] XSD validation service (Python lxml)
- [ ] Version storage (PostgreSQL)
- [ ] Return version metadata to client
- [ ] Basic conflict detection (~~409~~ → expected_version check)

**Implementation Progress**:
- ✅ Defined SaveDraft/GetDraft/WatchDraft RPCs with optimistic concurrency
- ✅ Added blob storage references for large diagrams
- ✅ SDK integration in progress - raw Connect clients exposed
- ⏳ Backend service handlers not implemented yet
- ✅ Designed conflict resolution with operation history
- ⏳ Next: Implement storage layer integration

### Phase 3: Draft Recovery (Week 3)
**Goal**: Never lose work
- [ ] Draft persistence endpoint (optional)
- [ ] Local draft vs saved version comparison
- [ ] Recovery modal on page reload
- [ ] Draft cleanup after successful save
- [ ] ETag-based conflict resolution for drafts

**Implementation Progress**:
- ✅ Defined draft management with ETags and versioning
- ✅ Added WatchDraft for real-time conflict detection
- ✅ SDK provides raw streaming with AbortSignal support
- ⏳ Conflict resolution UI components pending
- ⏳ Next: Implement IndexedDB integration

### Phase 4: Basic AI Integration (Week 4)
**Goal**: AI assistance in editor
- [ ] AI context extraction (selection + lint issues)
- [ ] Backend AI proxy endpoint
- [ ] Structured command validation
- [ ] Apply suggestions via modeling API
- [ ] Undo/redo for AI operations

**Implementation Progress**:
- ✅ Defined GetAISuggestions as server-streaming RPC
- ✅ Created progressive AI event structure (header → operations → complete)
- ✅ SDK exposes raw AI streaming (no wrappers)
- ⏳ AI backend integration with LangGraph pending
- ✅ Added ApplyAISuggestion for selective operation application
- ⏳ Next: Integrate with LLM provider (Claude/GPT-4)

### MVP Success Criteria
- **User can**: Create, edit, save BPMN diagrams with version history
- **System provides**: XSD validation, draft recovery, basic AI suggestions
- **Performance**: <100ms local operations, <500ms save
- **Data safety**: No work lost via autosnapshots
- **AI value**: At least 3 useful operations (add labels, fix disconnected, add defaults)

### Out of MVP Scope
- ~~Remote draft synchronization~~ → Actually included via WatchDraft streaming
- Graph database projections
- Advanced AI operations
- Multi-tenant isolation
- Authentication/authorization
- Export formats beyond XML
- ~~Collaborative features~~ → Basic collaboration included (cursors, presence)

## Implementation Notes

### Technology Stack
- **Protocol**: Protobuf with gRPC services
- **Web Client**: Connect-ES v2 with TypeScript
- **Streaming**: Server-streaming for AI and collaboration
- **Operations**: Delta-based updates instead of full XML
- **Storage**: Blob references for large diagrams

### Key Files Created
- `packages/ameide_core_proto/proto/ameide_core_proto/bpmn/v1/bpmn_editor_service.proto` - Complete service definition with streaming
- `packages/ameide_sdk_ts/src/mock-transport.ts` - Full mock implementation for all RPCs
- `packages/ameide_sdk_ts/src/__tests__/*.test.ts` - Minimal test suite following Connect v2 principles
- `backlog/141-core-api-decision.md` - Architecture decision record

### Completed (2025-08-20)
1. ✅ Created comprehensive proto definition with operation-based deltas
2. ✅ Generated TypeScript types with protoc-gen-es v2
3. ✅ Updated `ameide_sdk_ts` with BPMNEditorService client
4. ✅ Implemented complete mock transport with streaming support
5. ✅ Added minimal test suite (exports, mock transport, interceptors)
6. ✅ Updated SDK documentation with usage examples

### Next Steps
1. Implement service handlers in `core_grpc` package
2. Integrate Connect-ES client in `www_ameide_platform`
3. Set up blob storage for large diagrams
4. Connect to LLM provider for AI suggestions (Claude/GPT-4)
5. Implement IndexedDB draft storage in frontend
---

## 136-bpmn-v3-optimistic-adrs.md (old)

# Architecture Decision Records (V3 Optimistic)

**Status**: Decisions Implemented (2025-08-21)
**Implementation**: Complete SDK + Platform Integration with Protobuf/gRPC/Connect stack

## ADR-001 — Optimistic UI (No Backend Gating)
**Decision**: bpmn-js executes commands locally; no server round-trip before paint.
**Rationale**: Lowest latency, simplest UX; aligns with single-user and AI-in-editor.
**Trade-offs**: No server authority until Save; validation errors discovered late.

## ADR-002 — Periodic Snapshots (Drafts)
**Decision**: Auto-snapshot current XML at configurable intervals: { intervalMs: 30000, idleMs: 5000, maxDraftAge: 7d }.
**Rationale**: Crash recovery and draft continuity without version churn.
**Implementation**: IndexedDB always, remote draft API optional.

## ADR-003 — Version Bump on Explicit Save
**Decision**: Only a user-initiated Save creates a new immutable version (v = v+1).
**Rationale**: Clean audit points; users control commits.
**Consequence**: Autosaves are drafts only, not versions.

## ADR-004 — Validation: Lint Live, XSD on Save
**Decision**: Run bpmnlint continuously in the editor; run authoritative XSD validation server-side on Save.
**Rationale**: Fast feedback + spec compliance at persistence boundary.
**Trade-off**: Invalid diagrams can exist locally until Save attempt.

## ADR-005 — In-Editor AI Collaboration
**Decision**: AI plugin runs inside the editor process; generates **structured operations**; applies via command stack (HITL default, Autopilot optional).
**Rationale**: Keep agent and human in the same undo/redo domain; progressive streaming for UX.
**Implementation**: ✅ Server-streaming GetAISuggestions RPC with typed Operation messages.

## ADR-006 — Optional Event Streams (Local Only)
**Decision**: Keep **typed command capture** client-side (for analytics/telemetry). Server receives only snapshots at Save; event sourcing is optional server-side.
**Rationale**: Reduce backend complexity; leave room for later ES adoption.
**Future**: Can add event streaming without breaking changes.

## ADR-007 — Persistence Model
**Decision**: Backend stores: versions (immutable XML), latest draft (optional), metadata. Read models (SQL/Graph) update on Save.
**Rationale**: Save-time consistency; avoid drift between projections and editor state.
**Tables**: diagram_versions, diagram_drafts, read projections.

## ADR-008 — Transport
**Decision**: Editor uses Connect-ES v2 (gRPC-Web/Connect protocol) for all operations; AI calls stream through backend.
**Rationale**: Type safety, binary efficiency, native streaming support.
**Services**: BPMNEditorService with SaveDraft, WatchDraft, GetAISuggestions RPCs.
**Implementation**: ✅ Proto defined in `bpmn_editor_service.proto`

## ADR-009 — IDs & Layout
**Decision**: Client owns IDs and layout; server does not rewrite on Save unless policy enforces normalization.
**Rationale**: Deterministic user experience; normalization can be an optional tool.
**Exception**: Server may enforce unique IDs if duplicates detected.

## ADR-010 — MVP Scope
**Decision**: MVP includes local drafts, server saves with XSD, basic AI, no collaboration.
**Rationale**: Deliver value in 4 weeks; establish foundation for enhancement.
**Defer**: Multi-tenancy, auth, remote drafts, graph projections, advanced AI.

## ADR-011 — AI Safety
**Decision**: HITL (Human-in-the-Loop) by default; Autopilot requires explicit allowlist.
**Rationale**: Prevent runaway AI edits; maintain user trust.
**Allowlist Examples**: add-labels, fix-disconnected, add-default-flows.

## ADR-012 — Conflict Resolution
**Decision**: Optimistic concurrency with expected_version/etag; conflicts return conflicting operations.
**Rationale**: Client can rebase operations intelligently.
**Implementation**: ✅ SaveDraftResponse includes ConflictInfo with operation history.

## ADR-013 — Operation Deltas (NEW)
**Decision**: Send granular operations instead of full XML for drafts.
**Rationale**: Smaller payloads (10-100x), better conflict resolution, natural for collaborative features.
**Implementation**: ✅ Operation message with typed oneofs (CreateElementOp, UpdateElementOp, etc.)

## ADR-014 — Streaming Architecture (NEW)
**Decision**: Use server-streaming for AI suggestions and draft watching.
**Rationale**: Progressive UI updates, cancellation support, real-time collaboration.
**Implementation**: ✅ WatchDraft and GetAISuggestions return streams.
---

## 137-bpmn-v3-optimistic-client.md (old)

# Client Architecture — Editor, Snapshots, AI

**Status**: Full Integration Complete (2025-08-21)
**Stack**: Connect-ES v2 with Protobuf-generated types  
**Progress**: 
- ✅ SDK exposes raw BPMNEditorService (no wrappers)
- ✅ Full Connect v2 alignment (gRPC codes, modern APIs, ESM-only)
- ✅ Comprehensive tests for streaming, chunking, conflicts
- ✅ README updated with correct transport patterns and bpmn-js notes
- ✅ BpmnJSAdapter translation layer complete (operations ↔ bpmn-js commands)
- ✅ Platform integration with proper TransportProvider setup
- ✅ Dynamic imports for SSR compatibility

## Modules

### Core Editor
- **Modeler**: bpmn-js + modeling API (existing)
- **Lint Overlay**: bpmnlint active; non-blocking visual feedback
- **Command Capture (local)**: semantic/layout typed events for telemetry, AI context, undo-friendly

### AutoSnapshot System (Updated for gRPC)
```typescript
import { createClient } from '@connectrpc/connect';
import { BPMNEditorService } from '@ameideio/ameide-sdk-ts';

class AutoSnapshot {
  private intervalTimer: Timer;  // Every 30s default
  private idleTimer: Timer;      // After 5s idle
  private operations: Operation[] = [];
  private client = createClient(BPMNEditorService, transport);
  
  async snapshot() {
    if (operations.length === 0) return; // Skip if no changes
    
    // Always save locally
    await indexedDB.put('drafts', { 
      diagramId,
      operations: [...operations],
      timestamp: Date.now(),
      version: currentVersion,
      isDraft: true 
    });
    
    // Sync to server with operations (not full XML)
    if (config.remoteDrafts) {
      const response = await client.saveDraft({
        artifactId: diagramId,
        operations,
        expectedVersion: currentVersion,
        etag
      });
      
      if (response.conflict) {
        // Handle conflict with operation rebase
        await this.rebaseOperations(response.conflict);
      }
      
      etag = response.etag;
      operations = []; // Clear after successful save
    }
  }
}
```

### Explicit Save (Using gRPC Snapshot)
```typescript
import { createClient } from '@connectrpc/connect';
import { BPMNEditorService } from '@ameideio/ameide-sdk-ts';

const client = createClient(BPMNEditorService, transport);

async function save() {
  const { xml } = await modeler.saveXML();
  
  // Optional client-side prelint
  if (config.prelintOnSave) {
    const response = await client.validateDiagram({
      xmlContent: xml,
      profile: ValidationProfile.MODELING
    });
    
    if (!response.isValid && !confirm('Save with errors?')) {
      return;
    }
  }
  
  // Create snapshot (immutable version)
  try {
    const response = await client.createSnapshot({
      artifactId: diagramId,
      description: 'User save',
      draftVersion: currentVersion,
      markAsRelease: false
    });
    
    currentVersion = response.version;
    markClean();
    showToast(`Saved as version ${response.version}`);
    
    // Clear local operations after snapshot
    operations = [];
  } catch (error) {
    if (error.code === 'FAILED_PRECONDITION') {
      // Version conflict
      const conflict = error.details;
      if (confirm(`Conflict: server at v${conflict.serverVersion}. Resolve?`)) {
        await resolveConflict(conflict.conflictingOperations);
      }
    }
  }
}
```

### AI Plugin Architecture (Streaming)
```typescript
import { createClient } from '@connectrpc/connect';
import { BPMNEditorService } from '@ameideio/ameide-sdk-ts';

class AIPlugin {
  private client = createClient(BPMNEditorService, transport);
  private controller?: AbortController;
  
  // Context extraction
  getContext(): AIContext {
    return {
      selectedElementIds: getSelectedElements().map(e => e.id),
      lintIssues: getCurrentLintIssues(),
      cursorPosition: getCursorPosition(),
      baseVersion: currentVersion
    };
  }
  
  // Stream AI suggestions with progressive rendering
  async *streamSuggestions(prompt: string) {
    this.controller = new AbortController();
    
    const stream = this.client.getAISuggestions({
      artifactId: diagramId,
      prompt,
      baseVersion: currentVersion,
      selectedElementIds: this.getContext().selectedElementIds,
      lintIssues: this.getContext().lintIssues,
      options: {
        temperature: 0.7,
        maxOperations: 50,
        explainReasoning: true
      }
    }, { signal: this.controller.signal });
    
    for await (const event of stream) {
      switch (event.event.case) {
        case 'header':
          yield { type: 'start', summary: event.event.value.summary };
          break;
        case 'operation':
          yield { type: 'operation', op: event.event.value.operation };
          break;
        case 'complete':
          yield { type: 'done', stats: event.event.value.metrics };
          break;
      }
    }
  }
  
  // Apply operations to canvas
  async applyOperations(operations: Operation[]) {
    modeling.startBatch(); // Group for single undo
    
    for (const op of operations) {
      switch (op.op.case) {
        case 'createElement':
          modeling.createShape(op.op.value);
          break;
        case 'updateElement':
          modeling.updateProperties(op.op.value);
          break;
        case 'createConnection':
          modeling.connect(op.op.value);
          break;
        // ... other operations
      }
    }
    
    modeling.endBatch();
    
    // Track operations for next draft save
    this.operations.push(...operations);
  }
}
```

## Sequences

### A) Edit + Autosnapshot (Operations)
```mermaid
sequenceDiagram
autonumber
User->>Editor: Move/Create/Update
Editor->>Editor: Immediate canvas update (local)
Editor->>Operations: Capture as typed Operation
loop every 30s or 5s idle
  Timer->>AutoSnapshot: Trigger
  AutoSnapshot->>IndexedDB: Store operations
  opt remoteDrafts enabled
    AutoSnapshot->>Backend: gRPC SaveDraft(operations)
    Backend-->>AutoSnapshot: SaveDraftResponse(etag, version)
    opt conflict detected
      Backend-->>AutoSnapshot: ConflictInfo(operations)
      AutoSnapshot->>Rebase: Resolve conflicts
    end
  end
end
```

### B) Explicit Save (Snapshot)
```mermaid
sequenceDiagram
autonumber
User->>Editor: Click Save
Editor->>Backend: gRPC ValidateDiagram(xml)
Backend-->>Editor: ValidationResponse
opt has errors
  Editor->>User: Confirm save with errors?
end
Editor->>Backend: gRPC CreateSnapshot(artifactId, version)
Backend->>XSD: Full validation
Backend->>Store: Create immutable version
Backend-->>Editor: SnapshotResponse(version, checksum)
Editor->>UI: Mark clean, show version
Editor->>Operations: Clear operation buffer
```

### C) AI Suggest & Apply (Streaming)
```mermaid
sequenceDiagram
autonumber
User->>AI: "Add default flows"
AI->>Context: Extract selection + issues
AI->>Backend: gRPC GetAISuggestions (stream)
Backend->>LLM: Generate operations
Backend-->>AI: Stream: Header
AI->>UI: Show summary + confidence
Backend-->>AI: Stream: Operation 1
AI->>UI: Render operation preview
Backend-->>AI: Stream: Operation 2..N
AI->>UI: Update preview progressively
Backend-->>AI: Stream: Complete
User->>AI: Click Apply Selected
AI->>Modeling: Execute operations
AI->>Operations: Track for draft
```

## Configuration

```typescript
interface EditorConfig {
  autosave: {
    intervalMs: number;      // Default: 30000
    idleMs: number;          // Default: 5000
    remoteDrafts: boolean;   // Default: false (MVP)
    maxDraftSize: number;    // Default: 5MB
  };
  
  ai: {
    enabled: boolean;        // Default: true
    mode: 'HITL' | 'AUTO';  // Default: HITL
    maxCommands: number;     // Default: 50
    contextLimit: number;    // Default: 75KB
    allowlist: string[];     // Auto-apply whitelist
  };
  
  lint: {
    active: boolean;         // Default: true
    profile: string;         // Default: 'recommended'
    showOverlay: boolean;    // Default: true
  };
  
  save: {
    prelintOnSave: boolean;  // Default: false
    confirmOverwrite: boolean; // Default: true
  };
}
```

## MVP Implementation (4 weeks)

### Week 1: Core Editor
```typescript
// Existing bpmn-js works as-is
// Add:
- [ ] bpmnlint integration (2 days)
- [ ] IndexedDB draft storage (1 day)
- [ ] Autosnapshot timer (1 day)
- [ ] Save button + local export (1 day)
```

### Week 2: Backend Integration
```typescript
- [ ] Save endpoint client (1 day)
- [ ] Version tracking (1 day)
- [ ] Conflict handling UI (1 day)
- [ ] Draft recovery modal (2 days)
```

### Week 3: Polish
```typescript
- [ ] Loading states (1 day)
- [ ] Error handling (1 day)
- [ ] Toast notifications (1 day)
- [ ] Keyboard shortcuts (1 day)
- [ ] Performance optimization (1 day)
```

### Week 4: AI Integration
```typescript
- [ ] Context extraction (1 day)
- [ ] Backend proxy client (1 day)
- [ ] Command validation (1 day)
- [ ] Apply UI + animations (1 day)
- [ ] Testing + refinement (1 day)
```

## UX Guarantees

1. **No blocking**: All edits apply instantly
2. **No data loss**: Autosnapshots every 30s to IndexedDB
3. **Clear saves**: Explicit user action creates versions
4. **AI transparency**: All AI edits are badged and undoable
5. **Conflict resolution**: Clear messaging on version conflicts
6. **Draft recovery**: Modal on reload if draft newer than saved

## Performance Targets

- Local edit latency: <16ms (one frame)
- Autosnapshot: <100ms (async, non-blocking)
- Save operation: <500ms for diagrams <500KB
- AI suggestion: <2s end-to-end
- Draft recovery: <200ms to detect and prompt
---

## 138-bpmn-v3-optimistic-backend.md (old)

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
---

## 139-bpmn-v3-optimistic-ops.md (old)

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
---

## 140-azure-ad-control-plane.md (old)

# Azure AD SSO for Control Plane

## Context
- Strategic direction: every control-plane UI must authenticate with Azure AD across all environments (local, staging, production), while customer-facing or tenant-scoped applications continue to rely on Keycloak for federation and per-tenant directories.
- Kubernetes Dashboard already used Azure AD in production; local and staging clusters now consume the same oauth2-proxy configuration backed by Azure Key Vault so all environments behave consistently.
- Argo CD is delivered via Helmfile (OCI chart) with Azure AD OIDC, the admin account disabled, and RBAC mapped to the shared `ameide-aks-admins` Azure AD group adopted across environments.
- Grafana and the rest of the observability surface (Prometheus, Alertmanager, Loki, Tempo), Temporal Web, and pgAdmin are all fronted by oauth2-proxy (or native OIDC) with Microsoft Entra ID; every environment now sources credentials from Azure Key Vault.
- Secrets flow from Azure Key Vault across all environments. The manifest of expected keys lives in `charts/values/platform/azure-keyvault-secrets.yaml`, and local clusters read from the `ameidelocalkv` vault inside the `Ameide-Local` resource group. Environment overlays currently map all control-plane roles to the shared Azure AD group `ameide-aks-admins` until dedicated read-only groups are provisioned.
- Documentation, runbooks, and GitOps references have been updated to reflect the Azure-only access pattern and the retirement of the legacy Argo CD install script.

## Goals
- Enforce Azure AD OIDC on every control-plane surface in each environment, including local development clusters, with matching redirect URIs (`*.ameide.io`, `*.staging.ameide.io`, `*.dev.ameide.io`).
- Stand up a delegated `staging.ameide.io` DNS zone (Azure Public DNS) mirroring production records so staging control-plane traffic stays isolated while following the same Azure AD and TLS patterns.
- Keep tenant and end-user applications bound to Keycloak for identity, federation, and per-customer SSO across environments.
- Deliver reproducible deployments using Helmfile-managed charts and Azure Key Vault + ExternalSecret across every environment without ad-hoc manual edits.
- Document Azure AD app-registration requirements (redirect URIs, group claims, automation strategy) and map them cleanly to Helm values so teams can audit and re-create the configuration.

## Control-Plane Surface Inventory

| Service | Purpose | Production FQDN | Staging FQDN | Local FQDN (CoreDNS) | Azure AD Status |
| --- | --- | --- | --- | --- | --- |
| Argo CD | GitOps orchestration | `argocd.ameide.io` | `argocd.staging.ameide.io` | `argocd.dev.ameide.io` | Helmfile-managed (OCI) with Azure AD OIDC; `ameide-aks-admins` mapped as admin/read-only; awaiting Key Vault secret seeding |
| Grafana | Observability dashboards | `grafana.ameide.io` | `grafana.staging.ameide.io` | `grafana.dev.ameide.io` | Azure AD enabled via `grafana.ini`; secrets sourced from Azure Key Vault in every environment (local draws from `ameidelocalkv`) |
| Kubernetes Dashboard (oauth2-proxy) | Cluster UI | `k8s-dashboard.ameide.io` | `k8s-dashboard.staging.ameide.io` | `k8s-dashboard.dev.ameide.io` | Azure AD in all environments; Azure Key Vault wiring complete |
| Postgres Admin (pgAdmin) | Database administration | `pgadmin.ameide.io` | `pgadmin.staging.ameide.io` | `pgadmin.dev.ameide.io` | Azure AD-native Helm chart & Gateway routes committed; secrets pending |
| Temporal Web UI | Workflow visibility | `temporal.ameide.io` | `temporal.staging.ameide.io` | `temporal.dev.ameide.io` | oauth2-proxy config shipped with `ameide-aks-admins`; blocked on Azure AD client secret rotation |
| Alertmanager / Prometheus consoles | Alerting & metrics | `alertmanager.ameide.io`, `prometheus.ameide.io` | `alertmanager.staging.ameide.io`, `prometheus.staging.ameide.io` | `alertmanager.dev.ameide.io`, `prometheus.dev.ameide.io` | oauth2-proxy + Gateway + Key Vault entries enforced by `ameide-aks-admins`; rollout pending credential seeding |
| Loki / Tempo consoles | Logs & traces | `loki.ameide.io`, `tempo.ameide.io` | `loki.staging.ameide.io`, `tempo.staging.ameide.io` | `loki.dev.ameide.io`, `tempo.dev.ameide.io` | oauth2-proxy + Gateway routes committed with `ameide-aks-admins`; awaiting secret materialisation |
| Keycloak Admin Console | Tenant identity management | `auth.ameide.io` | `auth.staging.ameide.io` | `auth.dev.ameide.io` | Remains Keycloak-native (no Azure AD changes) |

_Note:_ Any new ingress in Azure must fit under the wildcards `*.ameide.io` and `*.staging.ameide.io`; CoreDNS already maps `*.dev.ameide.io` to the local Envoy gateway.

## Latest Progress
- [x] Promoted Argo CD to a Helmfile-managed OCI release with Azure AD OIDC (UI + CLI), disabled the legacy admin account, and updated GitOps/infra documentation.
- [x] Fronted Grafana, Prometheus, Alertmanager, Loki, Tempo, and Temporal Web with oauth2-proxy (Azure provider) across every environment and added the corresponding Gateway routes/TLS wiring.
- [x] Added a pgAdmin chart with native Azure AD configuration and integrated it into Helmfile, Gateway, and Key Vault manifests.
- [x] Expanded `azure-keyvault-secrets` to include all control-plane client IDs, secrets, and cookie keys, and aligned local development with the Ameide-Local Key Vault.
- [x] Standardised on the `ameide-aks-admins` Azure AD group for control-plane admin/read-only roles across local, staging, and production until bespoke groups are available, and made Helmfile fail fast when IDs are missing.
- [x] Deprecated the shell-based Argo CD installer and replaced docs/runbooks with the Azure-only workflows and Key Vault seeding guidance.

## Remaining Work

### Azure AD & Key Vault
- [ ] Audit the `ameide-aks-admins` membership (admin vs. read-only personas) and decide whether dedicated read-only groups are required for long-term least privilege.
- [ ] Finalise Azure AD app registrations for every control-plane surface (UI + CLI), capturing client IDs/secrets in automation-friendly form.
- [ ] Seed Azure Key Vault (`ameide`, `ameide-staging`) with the committed secret names (`argocd-oidc-*`, `grafana-azuread-*`, `*-oauth2-proxy-*`, `pgadmin-oidc-*`) and validate ExternalSecret reconciliation.

### Secret Management & Automation
- [ ] Automate Azure AD app registration lifecycle (creation, consent, rotation) via CLI/Terraform/Bicep.
- [ ] Document and test the rotation workflows (Key Vault update ➜ ExternalSecret refresh ➜ pod restart) for each oauth2-proxy/native client.
- [ ] Encrypt and commit final `*.sops.yaml` payloads once Azure secrets are issued for local development parity.

### Deployment Validation
- [ ] Run `helmfile -e staging diff/sync` once secrets and group IDs are in place; validate Azure AD login flows for Argo CD, Grafana, Prometheus, Alertmanager, Loki, Tempo, Temporal, and pgAdmin.
- [ ] Promote to production with a controlled change window and verify group-based RBAC plus logout flows.
- [ ] Capture a break-glass procedure (temporary Azure group membership, admin re-enable) in the operations runbook and socialise with on-call staff.

### Operational Guardrails
- [ ] Publish the Azure AD incident response flow (expired secret, group membership drift, outage) in `docs/operations/azure-ad-control-plane.md`.
- [ ] Add automated alerts that flag oauth2-proxy refresh token expiry or ExternalSecret reconciliation failures.
- [ ] Establish quarterly reviews to ensure Azure AD app registrations stay aligned with DNS/TLS records and Helm values.

### Documentation & DNS Follow-Up
- [ ] Complete the `staging.ameide.io` DNS delegation work and ensure cert-manager DNS01 automation is in place.
- [ ] Publish the finalised runbooks covering Azure AD access, secret rotation, and emergency procedures.

## Risks & Mitigations
| Risk | Impact | Mitigation |
| --- | --- | --- |
| Azure AD secret rotation missed | Control-plane UIs reject logins | Automate secret rotation via CLI/Terraform and schedule reminders; expose rotation runbook |
| Group membership drift (`ameide-aks-admins`) | Loss of access or privilege escalation | Add Azure AD audit alerts, document break-glass, plan for dedicated read-only groups |
| ExternalSecret lag after Key Vault update | Stale credentials in pods | Monitor ExternalSecret status, add readiness probes that surface expired tokens, document manual restart path |
| TLS/DNS drift between environments | Login redirects fail | Track DNS zones in IaC, gate Helmfile sync on explicit hostnames, add staging delegation SOP |

## Timeline (Target)
- **Week 1:** Finalise Azure AD registrations and seed Key Vaults (production, staging, Ameide-Local).
- **Week 2:** Validate Helmfile sync in staging, complete documentation, cut operations runbook.
- **Week 3:** Production rollout with change window, publish post-deployment validation, close out risk register.

---

