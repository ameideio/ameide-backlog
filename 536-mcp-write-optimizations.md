# 536 — MCP Write Optimizations (Agent-Friendly Commands)

**Status:** Draft
**Audience:** Architecture, platform engineering, AI/agent teams
**Scope:** Define write-side optimizations for MCP — enabling AI agents to create and modify data with simplified inputs, schema discovery, and meaningful responses.

**Use with:**
- MCP protocol adapter: `backlog/534-mcp-protocol-adapter.md`
- MCP read optimizations: `backlog/535-mcp-read-optimizations.md`
- Transformation domain: `backlog/527-transformation-domain.md`
- Transformation process (governance): `backlog/527-transformation-process.md`
- Capability definition: `backlog/528-capability-definition.md`

---

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Interface (MCP adapter), Application Service (write translation).
- **Out-of-scope layers:** Strategy/Business (capability design), Technology (domain internals).
- **Allowed nouns:** write optimization, schema discovery, input simplification, rich response.

---

## 0) Problem

When AI agents write to capabilities via MCP, they face several challenges:

**Current write pattern requires too much:**
```json
{
  "intent": {
    "message_id": "???",           // Agent shouldn't generate
    "correlation_id": "???",       // Or this
    "aggregate_version": 1,        // Or know about versioning
    "create_element": {
      "id": "???",                 // Or generate UUIDs
      "model_id": "???",           // Need to know context
      "type": "ApplicationComponent",
      "layer": "application",      // Redundant if type implies layer
      ...
    }
  }
}
```

**What agents are good at:** Semantic content, natural structure
**What agents are bad at:** UUIDs, version numbers, protocol metadata, exact enum values

**The goal:** Agent sends semantic content → MCP adapter handles complexity → Domain receives valid intent

---

## 1) Design principles

| Principle | Implementation |
|-----------|----------------|
| **MCP communicates expected** | Schema discovery tools, valid types/relationships |
| **MCP abstracts complexity** | Generate IDs, derive metadata, handle versioning |
| **LLM sends what it's good at** | Semantic content in JSON, no protocol boilerplate |
| **MCP gives meaningful responses** | Success with created entity, errors with suggestions |

---

## 2) Input simplification

### 2.1 Before: Protocol-oriented input

```json
{
  "intent": {
    "meta": {
      "message_id": "msg-uuid-123",
      "correlation_id": "corr-uuid-456",
      "tenant_id": "tenant-uuid",
      "occurred_at": "2024-01-15T10:30:00Z"
    },
    "create_element": {
      "id": "elem-uuid-789",
      "model_id": "model-uuid-abc",
      "type": "APPLICATION_COMPONENT",
      "layer": "APPLICATION",
      "name": "OrderService",
      "description": "Handles order processing"
    }
  }
}
```

### 2.2 After: Agent-oriented input

```json
{
  "type": "ApplicationComponent",
  "name": "OrderService",
  "description": "Handles order processing"
}
```

### 2.3 What MCP adapter generates

| Field | Source |
|-------|--------|
| `message_id` | Generated UUID |
| `correlation_id` | From MCP request context |
| `tenant_id` | From auth token |
| `occurred_at` | Current timestamp |
| `id` | Generated UUID |
| `model_id` | Default model or from context |
| `type` enum | Mapped from string ("ApplicationComponent" → APPLICATION_COMPONENT) |
| `layer` | Derived from type (ApplicationComponent → APPLICATION) |

---

## 3) Schema discovery tools

Agents can discover valid inputs before writing:

### 3.1 getElementTypes

```yaml
- name: getElementTypes
  description: List valid element types grouped by layer
  output:
    layers:
      strategy:
        - { type: "Capability", description: "An ability the organization possesses" }
        - { type: "Resource", description: "An asset owned or controlled" }
        - { type: "CourseOfAction", description: "An approach to achieve goals" }
        - { type: "ValueStream", description: "A sequence of activities delivering value" }
      business:
        - { type: "BusinessProcess", description: "A sequence of business behaviors" }
        - { type: "BusinessService", description: "A service fulfilling business needs" }
        - { type: "BusinessActor", description: "An organizational entity" }
        # ...
      application:
        - { type: "ApplicationComponent", description: "A modular, deployable unit" }
        - { type: "ApplicationService", description: "A service exposed by components" }
        - { type: "ApplicationInterface", description: "A point of access to a service" }
        - { type: "DataObject", description: "Data structured for processing" }
      technology:
        - { type: "Node", description: "A computational resource" }
        - { type: "Device", description: "A physical resource" }
        - { type: "SystemSoftware", description: "Software enabling other software" }
```

### 3.2 getRelationshipTypes

```yaml
- name: getRelationshipTypes
  description: List valid relationship types with source/target rules
  input:
    source_type: { type: string, description: "Filter by source element type" }
    target_type: { type: string, description: "Filter by target element type" }
  output:
    relationships:
      - type: "Realization"
        description: "Source realizes target"
        valid_pairs:
          - { source: "ApplicationComponent", target: "ApplicationService" }
          - { source: "BusinessProcess", target: "BusinessService" }
        direction: "source → target"
      - type: "Serving"
        description: "Source serves target"
        valid_pairs:
          - { source: "ApplicationService", target: "BusinessProcess" }
          - { source: "ApplicationComponent", target: "ApplicationComponent" }
      - type: "Composition"
        description: "Source is composed of target"
        valid_pairs:
          - { source: "Capability", target: "Capability" }
          - { source: "ApplicationComponent", target: "ApplicationComponent" }
      # ...
```

### 3.3 getWriteSchema

```yaml
- name: getWriteSchema
  description: Get JSON schema for a write operation
  input:
    operation: { type: string, enum: [createElement, createRelationship, updateElement, deleteElement] }
  output:
    schema: { ... }  # Full JSON Schema
    examples:
      - description: "Create an application component"
        input: { type: "ApplicationComponent", name: "OrderService", description: "..." }
      - description: "Create with parent"
        input: { type: "ApplicationComponent", name: "OrderAPI", parent_id: "..." }
    required_fields: ["type", "name"]
    optional_fields: ["description", "properties", "parent_id", "model_id"]
```

---

## 4) Write tool design

### 4.1 createElement

```yaml
- name: createElement
  description: Create an ArchiMate element
  input:
    # Required (semantic)
    type:
      type: string
      required: true
      description: "Element type (e.g., ApplicationComponent, Capability)"
    name:
      type: string
      required: true
      description: "Element name"

    # Optional (semantic)
    description:
      type: string
      description: "Element description"
    properties:
      type: object
      description: "Custom properties as key-value pairs"

    # Optional (context - agent can provide if known)
    model_id:
      type: string
      description: "Model to add to. Defaults to tenant's active model"
    parent_id:
      type: string
      description: "Parent element ID for composition relationship"

    # NOT accepted (handled by adapter):
    # - id, message_id, correlation_id, tenant_id
    # - aggregate_version, occurred_at
    # - layer (derived from type)

  output:
    success: { type: boolean }
    element:
      id: { type: string }
      type: { type: string }
      name: { type: string }
      layer: { type: string }
      model_id: { type: string }
      version: { type: integer }
    suggestions:
      type: array
      description: "Optional suggestions for next steps"
```

### 4.2 createRelationship

```yaml
- name: createRelationship
  description: Create a relationship between elements
  input:
    type:
      type: string
      required: true
      description: "Relationship type (e.g., Realization, Serving, Composition)"
    source_id:
      type: string
      required: true
      description: "Source element ID"
    target_id:
      type: string
      required: true
      description: "Target element ID"
    name:
      type: string
      description: "Optional relationship name"
    description:
      type: string
      description: "Optional description"

  output:
    success: { type: boolean }
    relationship:
      id: { type: string }
      type: { type: string }
      source_id: { type: string }
      target_id: { type: string }
```

### 4.3 updateElement

```yaml
- name: updateElement
  description: Update an existing element
  input:
    id:
      type: string
      required: true
      description: "Element ID to update"
    name:
      type: string
      description: "New name (if changing)"
    description:
      type: string
      description: "New description (if changing)"
    properties:
      type: object
      description: "Properties to update (merged with existing)"

    # Versioning handled internally:
    # - Adapter fetches current version
    # - Retries on conflict (optimistic locking)

  output:
    success: { type: boolean }
    element: { ... }
    previous_version: { type: integer }
    new_version: { type: integer }
```

### 4.4 deleteElement

```yaml
- name: deleteElement
  description: Delete an element
  input:
    id:
      type: string
      required: true
    cascade:
      type: boolean
      default: true
      description: "Also delete relationships involving this element"

  output:
    success: { type: boolean }
    deleted_relationships:
      type: array
      description: "IDs of relationships that were also deleted"
```

---

## 5) Rich error responses

### 5.1 Error response structure

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable error message",
    "field": "type",
    "details": { ... },
    "suggestions": { ... }
  }
}
```

### 5.2 Error types with suggestions

**Invalid type:**
```json
{
  "success": false,
  "error": {
    "code": "INVALID_ELEMENT_TYPE",
    "message": "Element type 'AppComponent' is not valid",
    "field": "type",
    "suggestions": {
      "did_you_mean": ["ApplicationComponent", "ApplicationService"],
      "valid_types_for_context": ["ApplicationComponent", "ApplicationService", "ApplicationInterface", "DataObject"],
      "hint": "Application layer types start with 'Application' or 'Data'"
    }
  }
}
```

**Invalid relationship:**
```json
{
  "success": false,
  "error": {
    "code": "INVALID_RELATIONSHIP",
    "message": "Cannot create 'Realization' from Capability to ApplicationComponent",
    "field": "type",
    "suggestions": {
      "valid_relationships": ["Assignment", "Serving"],
      "hint": "Realization goes from implementation to specification (e.g., ApplicationComponent realizes ApplicationService)"
    }
  }
}
```

**Element not found:**
```json
{
  "success": false,
  "error": {
    "code": "ELEMENT_NOT_FOUND",
    "message": "Element 'elem-123' not found",
    "field": "source_id",
    "suggestions": {
      "similar_elements": [
        { "id": "elem-456", "name": "OrderService", "type": "ApplicationComponent" },
        { "id": "elem-789", "name": "OrderProcessor", "type": "ApplicationComponent" }
      ],
      "hint": "Use transformation.semanticSearch to find elements by description"
    }
  }
}
```

**Duplicate name:**
```json
{
  "success": false,
  "error": {
    "code": "DUPLICATE_NAME",
    "message": "Element 'OrderService' already exists in this model",
    "field": "name",
    "suggestions": {
      "existing_element": { "id": "elem-existing", "type": "ApplicationComponent" },
      "alternatives": ["OrderService2", "OrderServiceV2", "NewOrderService"],
      "hint": "Use updateElement to modify the existing element, or choose a different name"
    }
  }
}
```

---

## 6) Validation tool (dry run) — Default first step

**validateWrite SHOULD be the default first step for agent write workflows.** This ensures agents check validity before committing, reducing errors and improving UX.

```yaml
- name: validateWrite
  description: Validate a write operation without committing. Call this BEFORE actual write operations.
  hints:
    read_only: true
    destructive: false
    idempotent: true
  input:
    operation:
      type: string
      enum: [createElement, createRelationship, updateElement, deleteElement]
    payload:
      type: object
      description: "The input that would be sent to the write operation"

  output:
    valid: { type: boolean }
    errors:
      type: array
      items:
        code: { type: string }
        message: { type: string }
        field: { type: string }
    warnings:
      type: array
      items:
        code: { type: string }
        message: { type: string }
        suggestion: { type: string }
    suggestions:
      type: array
      description: "Proactive suggestions even if valid"
```

### 6.1 Recommended agent workflow

```
Agent wants to create element
       │
       ▼
┌─────────────────────────────┐
│ 1. validateWrite            │  ◄── Always call first
│    (dry run)                │
└──────────────┬──────────────┘
               │
        ┌──────┴──────┐
        │             │
     valid=true    valid=false
        │             │
        ▼             ▼
┌───────────────┐ ┌──────────────────┐
│ 2. createElement │ Fix errors based │
│    (commit)      │ on suggestions   │
└───────────────┘ └──────────────────┘
```

### 6.2 System prompt guidance

MCP adapters SHOULD include guidance in tool descriptions that instructs agents to call validateWrite first:

```json
{
  "name": "createElement",
  "description": "Create an ArchiMate element. TIP: Call validateWrite first to check for errors before committing.",
  ...
}
```

### 6.3 Example usage

```
Agent: I'll create an OrderService component. Let me validate first.

Agent: [invokes transformation.validateWrite]
{
  "operation": "createElement",
  "payload": {
    "type": "ApplicationComponent",
    "name": "OrderService"
  }
}

Response:
{
  "valid": true,
  "errors": [],
  "warnings": [
    {
      "code": "MISSING_DESCRIPTION",
      "message": "Element has no description",
      "suggestion": "Add a description for better discoverability"
    }
  ],
  "suggestions": [
    "Similar element 'OrderProcessor' exists - consider if this is a duplicate",
    "Consider adding relationships to 'OrderAPI' interface"
  ]
}

Agent: Validation passed with a warning about missing description. Let me add one.

Agent: [invokes transformation.createElement]
{
  "type": "ApplicationComponent",
  "name": "OrderService",
  "description": "Handles order processing and fulfillment"
}
```

---

## 7) Idempotency (two-layer strategy)

### 7.1 Problem: Duplicate requests

Network failures, retries, and agent re-runs can cause duplicate write operations:

```
Agent: [invokes createElement] → Network timeout
Agent: [retries createElement] → Creates duplicate element
```

**Single-layer cache is insufficient:**
- If adapter restarts → cached responses lost → duplicates possible
- If horizontally scaled → each instance has own cache → duplicates possible

### 7.2 Solution: Two-layer idempotency

Idempotency is enforced at **two layers**:

| Layer | Purpose | Storage | TTL | Authoritative |
|-------|---------|---------|-----|---------------|
| **Adapter cache** | Fast rejection of obvious duplicates | In-memory + Redis | 24h | No |
| **Domain boundary** | Durable, authoritative deduplication | Postgres (idempotency table) | 7d | **Yes** |

**Key principle:** The Domain is the single writer; idempotency enforcement MUST be at the Domain boundary for durability. The adapter cache is an optimization, not a guarantee.

### 7.3 client_request_id in tool input

All write operations accept an **optional `client_request_id`** for idempotency:

```yaml
# Added to all write tool inputs
client_request_id:
  type: string
  description: "Client-generated idempotency key. If provided, repeated requests with same key return cached response."
```

### 7.4 Domain-layer implementation

The Domain WriteService persists idempotency keys in a dedicated table:

```sql
-- Idempotency table (per aggregate type)
CREATE TABLE idempotency_keys (
    tenant_id        UUID NOT NULL,
    client_request_id TEXT NOT NULL,
    aggregate_type   TEXT NOT NULL,
    aggregate_id     UUID,
    response_payload JSONB NOT NULL,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at       TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (tenant_id, client_request_id, aggregate_type)
);

CREATE INDEX idx_idempotency_expires ON idempotency_keys (expires_at);
```

**Domain WriteService handler:**

```go
func (s *WriteService) SubmitIntent(ctx context.Context, req *SubmitIntentRequest) (*SubmitIntentResponse, error) {
    tenantID := auth.TenantIDFromContext(ctx)
    clientRequestID := req.Meta.ClientRequestId

    // 1. Check idempotency table (authoritative)
    if clientRequestID != "" {
        existing, err := s.idempotencyRepo.Get(ctx, tenantID, clientRequestID, req.AggregateType)
        if err == nil && existing != nil {
            // Return cached response — no side effects
            return &SubmitIntentResponse{
                Success:          true,
                IdempotentReplay: true,
                Response:         existing.ResponsePayload,
                OriginalTime:     existing.CreatedAt,
            }, nil
        }
    }

    // 2. Process intent (create element, emit facts, etc.)
    result, err := s.processIntent(ctx, req)
    if err != nil {
        return nil, err
    }

    // 3. Persist idempotency key (same transaction as domain write)
    if clientRequestID != "" {
        s.idempotencyRepo.Set(ctx, tenantID, clientRequestID, req.AggregateType, IdempotencyRecord{
            AggregateID:     result.AggregateID,
            ResponsePayload: result.ToJSON(),
            ExpiresAt:       time.Now().Add(7 * 24 * time.Hour),
        })
    }

    return result, nil
}
```

**Critical:** The idempotency key insertion MUST be in the same transaction as the domain write. This ensures atomicity — if the domain write succeeds, the idempotency key is guaranteed to be persisted.

### 7.5 Adapter-layer implementation (optimization)

The adapter cache provides fast rejection but is NOT authoritative:

```go
func (a *Adapter) HandleCreateElement(ctx context.Context, input AgentInput) (*Response, error) {
    // 1. Fast path: check adapter cache (optimization only)
    if input.ClientRequestID != "" {
        if cached, found := a.idempotencyCache.Get(input.ClientRequestID); found {
            return cached, nil
        }
    }

    // 2. Build domain intent with client_request_id
    intent := &TransformationArchitectureDomainIntent{
        Meta: &TransformationMessageMeta{
            MessageId:       messageID,
            CorrelationId:   correlationID,
            TenantId:        tenantID,
            ClientRequestId: input.ClientRequestID,  // Passed to Domain
            OccurredAt:      timestamppb.Now(),
        },
        // ...
    }

    // 3. Submit to domain (authoritative idempotency check happens here)
    result, err := a.domainClient.SubmitIntent(ctx, intent)
    if err != nil {
        return a.MapDomainError(err)
    }

    // 4. Populate adapter cache for fast future lookups
    if input.ClientRequestID != "" {
        response := a.BuildResponse(result)
        a.idempotencyCache.Set(input.ClientRequestID, response, 24*time.Hour)
    }

    return response, nil
}
```

### 7.6 Proto envelope update

The `TransformationMessageMeta` envelope includes `client_request_id`:

```protobuf
message TransformationMessageMeta {
  string message_id = 1;
  string correlation_id = 2;
  string tenant_id = 3;
  google.protobuf.Timestamp occurred_at = 4;

  // Idempotency key — if set, domain enforces exactly-once semantics
  string client_request_id = 5;
}
```

### 7.7 Idempotency key generation

**Agents SHOULD generate idempotency keys** for write operations. Recommended patterns:

```
# Format: <operation>-<timestamp>-<hash>
createElement-1702656000-sha256(name+type)[:8]

# Or: UUID
createElement-550e8400-e29b-41d4-a716-446655440000
```

**Example tool call with idempotency:**
```json
{
  "type": "ApplicationComponent",
  "name": "OrderService",
  "description": "Handles order processing",
  "client_request_id": "createElement-1702656000-a1b2c3d4"
}
```

### 7.8 Response on duplicate

When a request is identified as duplicate (at either layer):

```json
{
  "success": true,
  "element": { ... },
  "idempotent_replay": true,
  "original_request_time": "2024-12-15T10:30:00Z"
}
```

### 7.9 Layer properties

| Property | Adapter Layer | Domain Layer |
|----------|---------------|--------------|
| **Storage** | In-memory + Redis | Postgres |
| **TTL** | 24 hours | 7 days |
| **Key format** | `idempotency:{tenant_id}:{client_request_id}` | `(tenant_id, client_request_id, aggregate_type)` |
| **Survives restart** | No (Redis) / Yes (Redis persistent) | Yes |
| **Horizontal scaling** | Per-instance (not shared) | Shared (database) |
| **Authoritative** | No | **Yes** |

### 7.10 When to use client_request_id

| Use Case | Recommended |
|----------|-------------|
| User-initiated writes | Yes |
| Automated agent workflows | Yes |
| Batch imports | Yes (per-item key) |
| Schema discovery queries | No (queries are naturally idempotent) |

### 7.11 Failure modes

| Scenario | Adapter Cache | Domain Layer | Outcome |
|----------|---------------|--------------|---------|
| First request | Miss | Miss | Process normally, persist key |
| Retry (adapter warm) | Hit | N/A | Return cached (fast) |
| Retry (adapter cold, domain warm) | Miss | Hit | Return from domain |
| Adapter restart | Lost | Preserved | Domain catches duplicate |
| Horizontal scale (other instance) | Miss | Hit | Domain catches duplicate |
| Key expired | Miss | Miss | Process as new (acceptable after 7d) |

---

## 8) High-level operations (templates)

For common multi-step patterns, provide compound operations:

### 8.1 createCapabilityStructure

```yaml
- name: createCapabilityStructure
  description: Create a capability with its standard decomposition
  input:
    name: { type: string, required: true }
    description: { type: string }
    value_streams:
      type: array
      items: { type: string }
      description: "Names of value streams this capability realizes"
    sub_capabilities:
      type: array
      items: { type: string }
      description: "Names of sub-capabilities (composition)"

  # Creates:
  # 1. Capability element
  # 2. ValueStream elements (if provided)
  # 3. Realization relationships (Capability → ValueStreams)
  # 4. Sub-capability elements (if provided)
  # 5. Composition relationships (Capability ◇─ SubCapabilities)

  output:
    success: { type: boolean }
    capability: { ... }
    created_elements: [ ... ]
    created_relationships: [ ... ]
```

### 8.2 addComponentToCapability

```yaml
- name: addComponentToCapability
  description: Add an application component that realizes a capability
  input:
    capability_id: { type: string, required: true }
    component_name: { type: string, required: true }
    component_description: { type: string }
    services:
      type: array
      items: { type: string }
      description: "Application services this component provides"

  # Creates:
  # 1. ApplicationComponent
  # 2. Realization relationship (Component → Capability)
  # 3. ApplicationService elements (if provided)
  # 4. Realization relationships (Component → Services)

  output:
    success: { type: boolean }
    component: { ... }
    created_elements: [ ... ]
    created_relationships: [ ... ]
```

---

## 9) Adapter implementation

### 9.1 Input enrichment flow

```go
func (a *Adapter) HandleCreateElement(ctx context.Context, input AgentInput) (*Response, error) {
    // 1. Extract context
    tenantID := auth.TenantIDFromContext(ctx)
    correlationID := mcp.CorrelationIDFromContext(ctx)

    // 2. Generate IDs
    elementID := uuid.New().String()
    messageID := uuid.New().String()

    // 3. Derive layer from type
    layer, err := DeriveLayer(input.Type)
    if err != nil {
        return ErrorResponse("INVALID_ELEMENT_TYPE", err, SuggestTypes(input.Type))
    }

    // 4. Resolve model (default if not provided)
    modelID := input.ModelID
    if modelID == "" {
        modelID, err = a.GetDefaultModel(ctx, tenantID)
        if err != nil {
            return ErrorResponse("NO_DEFAULT_MODEL", err, nil)
        }
    }

    // 5. Map type string to enum
    typeEnum, err := MapTypeToEnum(input.Type)
    if err != nil {
        return ErrorResponse("INVALID_ELEMENT_TYPE", err, SuggestTypes(input.Type))
    }

    // 6. Build domain intent
    intent := &TransformationArchitectureDomainIntent{
        Meta: &TransformationMessageMeta{
            MessageId:     messageID,
            CorrelationId: correlationID,
            TenantId:      tenantID,
            OccurredAt:    timestamppb.Now(),
        },
        Intent: &CreateElementIntent{
            Id:          elementID,
            ModelId:     modelID,
            Type:        typeEnum,
            Layer:       layer,
            Name:        input.Name,
            Description: input.Description,
            Properties:  input.Properties,
        },
    }

    // 7. Submit to domain
    result, err := a.domainClient.SubmitIntent(ctx, intent)
    if err != nil {
        return a.MapDomainError(err)
    }

    // 8. Build rich response
    return &Response{
        Success: true,
        Element: &ElementResponse{
            ID:      elementID,
            Type:    input.Type,
            Name:    input.Name,
            Layer:   layer.String(),
            ModelID: modelID,
            Version: result.AggregateVersion,
        },
        Suggestions: a.GenerateSuggestions(ctx, input),
    }
}
```

### 9.2 Type derivation

```go
var typeToLayer = map[string]Layer{
    // Strategy
    "Capability":      LAYER_STRATEGY,
    "Resource":        LAYER_STRATEGY,
    "CourseOfAction":  LAYER_STRATEGY,
    "ValueStream":     LAYER_STRATEGY,

    // Business
    "BusinessProcess": LAYER_BUSINESS,
    "BusinessService": LAYER_BUSINESS,
    "BusinessActor":   LAYER_BUSINESS,

    // Application
    "ApplicationComponent": LAYER_APPLICATION,
    "ApplicationService":   LAYER_APPLICATION,
    "ApplicationInterface": LAYER_APPLICATION,
    "DataObject":           LAYER_APPLICATION,

    // Technology
    "Node":           LAYER_TECHNOLOGY,
    "Device":         LAYER_TECHNOLOGY,
    "SystemSoftware": LAYER_TECHNOLOGY,
}

func DeriveLayer(typeName string) (Layer, error) {
    layer, ok := typeToLayer[typeName]
    if !ok {
        return 0, fmt.Errorf("unknown element type: %s", typeName)
    }
    return layer, nil
}
```

---

## 10) Acceptance criteria

1. Write tools accept simplified, agent-friendly input (no IDs, no versioning).
2. MCP adapter generates required protocol fields (IDs, metadata, timestamps).
3. Layer is derived from element type automatically.
4. Schema discovery tools expose valid types and relationships.
5. Error responses include suggestions and hints for correction.
6. **Validation tool (validateWrite) is recommended as default first step** in agent workflows.
7. High-level operations exist for common multi-step patterns.
8. Transformation reference implementation demonstrates all patterns.
9. **Two-layer idempotency**: Adapter cache (optimization) + Domain boundary (authoritative, durable).
10. `client_request_id` is included in Domain intent envelope (`TransformationMessageMeta`).
11. Idempotency table exists in Domain database with proper indexes and TTL cleanup.

---

## 11) V1 scope

| Feature | V1 | Future |
|---------|-----|--------|
| Input simplification | Full | ✓ |
| ID generation | Full | ✓ |
| Layer derivation | Full | ✓ |
| Schema discovery | getElementTypes, getRelationshipTypes | getWriteSchema with full JSON Schema |
| Rich errors | Code + message + suggestions | Did-you-mean with fuzzy matching |
| Validation (dry run) | validateWrite as recommended first step | With proactive suggestions |
| Idempotency | Two-layer: Adapter cache (24h) + Domain boundary (7d, durable) | Cross-region idempotency |
| High-level operations | None | createCapabilityStructure, etc. |
| Conflict retry | None | Automatic retry on version conflict |
