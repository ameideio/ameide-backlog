# 536 — MCP Write Optimizations (Agent-Friendly Commands)

**Status:** Draft (partial adapter scaffolds exist; write optimizations not implemented)
**Audience:** Architecture, platform engineering, AI/agent teams
**Scope:** Define write-side optimizations for MCP — enabling AI agents to create and modify data with simplified inputs, schema discovery, and meaningful responses.

**Use with:**
- MCP protocol adapter: `backlog/534-mcp-protocol-adapter.md`
- MCP read optimizations: `backlog/535-mcp-read-optimizations.md`
- Transformation domain: `backlog/527-transformation-domain.md`
- Transformation process (governance): `backlog/527-transformation-process.md`
- Capability definition: `backlog/528-capability-definition.md`
- Agentic memory contract (v6): `backlog/656-agentic-memory-v6.md`

---

## Implementation progress (current)

Implemented in repo (enablers):
- [x] MCP adapter scaffolding + verification exists (`ameide scaffold integration mcp-adapter`, and `ameide primitive verify --check mcp`).
- [x] Shared MCP runtime spine exists (`packages/ameide_mcp`) for HTTP transport, origin policy, and auth middleware.

Not yet implemented (still design-level in this backlog):
- [ ] Proto-driven schema discovery tools (e.g. “getElementTypes”, “getRelationshipTypes”) generated from proto + metamodel.
- [ ] Agent-friendly write tools that map simplified inputs → valid Domain intents (ID generation, metadata derivation, enum mapping, defaults).
- [ ] Proposal-first write flow integrated with Process governance (create proposal → review/approve → commit) as the default, with explicit escape hatches.
- [ ] Rich error responses with actionable suggestions (valid enums, missing fields, nearest matches) without leaking sensitive data.

Build-out checklist:
- [ ] Define “write tool surface” per capability (high-level templates vs low-level CRUD-ish tools) and keep it minimal.
- [ ] Implement deterministic ID + idempotency policy (idempotency keys, dedupe rules, replay-safe behavior).
- [ ] Standardize RequestContext propagation (trace/correlation/tenant) from MCP HTTP → Domain/Process RPCs.
- [ ] Add conformance tests that assert: schema discovery works, invalid inputs fail clearly, proposals are enforced by default, and committed writes emit expected facts.

## Clarification requests (next steps)

Decide/confirm:
- [ ] Default governance posture: “writes create proposals unless explicitly permitted” (and the exact policy knobs/roles that allow direct commits).
- [ ] Where “schema truth” comes from for write tools (proto annotations vs metamodel registry vs both) and how versions are handled.
- [ ] How to scope write operations (repository/path selection) without relying on the agent inventing IDs (context pinning, defaults, or required inputs).
- [ ] Which write operations are v1 for Transformation/SRE (minimal set) vs deferred (bulk imports, refactors, cross-model moves).
- [ ] How to handle optimistic concurrency/versioning for agent writes (ETags, expected version, or server-side merge strategy).

## Layer header (Application)

- **Primary ArchiMate layer(s):** Application.
- **Primary element types used:** Application Interface (MCP adapter), Application Service (write translation).
- **Out-of-scope layers:** Strategy/Business (capability design), Technology (domain internals).
- **Allowed nouns:** write optimization, schema discovery, input simplification, rich response.

## Implementation progress (repo)

- [x] Domain write services exist behind gRPC with `RequestContext` (capability-specific), providing the authoritative mutation surface.
- [x] SRE MCP adapter already performs basic “agent-friendly input → domain request” enrichment for a small set of tools (e.g., `createIncident` generates IDs/metadata in `primitives/integration/sre-mcp-adapter/internal/mcp/server.go`).
- [ ] No shared schema discovery tools (`getElementTypes`, `getRelationshipTypes`, `getWriteSchema`) implemented across adapters/capabilities.
- [ ] No standard “validate (dry-run) → confirm → commit” tool flow implemented in adapters.
- [ ] No standardized idempotency strategy for MCP writes (e.g., `client_request_id` + domain idempotency keys) implemented end-to-end.
- [ ] No enforcement that writes default to proposals / governance workflows (currently capability-specific and tool-specific).

## Clarifications requested (next steps)

- [ ] Define which write categories are allowed as direct Domain commands vs proposal-only (and how this maps to risk tiers and approvals).
- [ ] Standardize the MCP → gRPC metadata mapping: correlation/causation IDs, W3C trace context, and where these live (`RequestContext.metadata` vs gRPC headers).
- [ ] Choose the canonical idempotency contract for write tools: required `client_request_id`, how duplicates are detected, and the “repeat response” behavior.
- [ ] Decide where schema discovery truth lives: proto options → generated JSON schema, or an explicit “schema service” in Projection/Domain.
- [ ] Decide whether high-level template tools (e.g., `createCapabilityStructure`) are mandatory for complex writes to prevent agents from orchestrating many low-level calls.

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

### 1.1 Schema-as-contract for element internal structure

When a user says "create this capability", the platform must ensure the saved element matches a **specific internal structure** (required fields, required properties/profile, optional default sub-structure).

We achieve this by:

1. **Tool input schema defines the required structure**

   * Required fields/properties are required in the tool schema (or via a nested `profile` object).
   * Agents SHOULD treat this schema as the contract, not a prompt suggestion.
2. **(Recommended) Strict structured output at the client/model boundary**
   Configure the model/tooling so tool arguments must validate against the tool schema.
3. **Domain validation remains authoritative**
   The adapter may validate shape, but the Domain enforces invariants, defaults, governance, and rejects invalid writes.

Practical implication:

* If "internal structure" is **just fields/properties**, a single `createCapability` (or `createElement(type=Capability)`) is sufficient.
* If "internal structure" includes **standard decomposition/relationships**, expose a **high-level template tool** (e.g., `createCapabilityStructure`) so the platform builds the structure deterministically (agents should not orchestrate 5–10 low-level calls).

See also: `backlog/534-mcp-protocol-adapter.md` §3.3 for tool calling lifecycle.

### 1.2 Vendor compatibility and runtime configuration

**Client runtime configuration for reliable writes:**

Modern LLM vendors provide **structured output** capabilities that enforce tool schema conformance at the model boundary. Configure clients to use these features for write operations:

| Vendor | Feature | Configuration |
|--------|---------|---------------|
| **OpenAI** | Structured Outputs | `response_format: {"type": "json_schema", "strict": true}` |
| **Google Gemini** | Structured Outputs | `response_mime_type: "application/json"` with schema |
| **Azure OpenAI** | Structured Outputs | Same as OpenAI (recommended for function calling) |
| **Cohere** | Strict Tools | `strict_tools=True` (ensures exact schema conformance) |
| **Mistral** | Structured Outputs | `response_format: {"type": "json_object"}` + schema |
| **Amazon Nova** | Structured Output | `inferenceConfig: {structuredOutput: {format: "json"}}` |

**Temperature for write operations:**

Set **temperature=0** (greedy decoding) for write tools to maximize schema adherence and reduce hallucinations:

```python
# OpenAI / Azure OpenAI
completion = client.chat.completions.create(
    model="gpt-4",
    messages=[...],
    tools=[...],
    temperature=0  # Deterministic tool calling
)

# Amazon Nova
response = bedrock.converse(
    modelId="amazon.nova-pro-v1",
    messages=[...],
    toolConfig={...},
    inferenceConfig={"temperature": 0}
)
```

**Rationale:**
- **Google Vertex AI** recommends temperature `0` for function calling to reduce hallucinations
- **Amazon Nova** recommends greedy decoding (`temperature=0`) for both structured output and tool use
- Low temperature makes tool selection and argument generation deterministic

**JSON Schema subset limits:**

Not all vendors support full JSON Schema. Portable tool schemas should use:

- **Core types**: `object`, `properties`, `required`, `enum`, `array`, `items`
- **Validation**: `minLength`, `maxLength`, `minimum`, `maximum`, `pattern`
- **Nesting**: Keep ≤ 2 layers where possible (Amazon Nova best practice)
- **Size**: Schema counts toward token budget (Google Vertex AI note)

**Avoid** (vendor-specific or limited support):
- `anyOf`, `oneOf`, `allOf` (limited support in Vertex AI, Nova)
- Deep nesting (>2 levels degrades performance per Nova guidance)
- Exotic constraints (`dependencies`, `patternProperties`, etc.)

**Expected client behavior:**

Write tools MAY trigger explicit user confirmation in client UIs:
- **ChatGPT** shows confirmation modals before write/modify operations (per OpenAI Help Center)
- **Browser-based clients** should implement similar UX gating for governance

This aligns with the validate→confirm→commit pattern (see §6.0).

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

### 4.5 createCapability (domain-specific example)

Purpose: Create a Capability element with a required profile structure.

**Input (agent-facing):**

```json
{
  "name": "Fulfillment",
  "description": "Ability to fulfill customer orders",
  "profile": {
    "owner": "Supply Chain",
    "criticality": "High",
    "lifecycle": "Active"
  }
}
```

**Adapter responsibilities:**

* Derive `model_id`, `tenant_id`, timestamps, `correlation_id`
* Generate element `id` (or accept provided id)
* Map type string → enum (`Capability` → `CAPABILITY`)
* Apply defaults (if any)

**Domain responsibilities:**

* Validate required profile fields
* Enforce invariants (naming rules, uniqueness, governance)
* Persist + emit events

**Output:**

```json
{
  "success": true,
  "capability": {
    "id": "cap-abc-123",
    "type": "Capability",
    "name": "Fulfillment",
    "layer": "strategy",
    "revision": 1,
    "profile": {
      "owner": "Supply Chain",
      "criticality": "High",
      "lifecycle": "Active"
    }
  }
}
```

This example demonstrates how a domain-specific tool can enforce "internal structure" (required profile) through the tool schema contract. See §1.1 for details on schema-as-contract principle.

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

### 6.0 End-to-end write flow (chat → validate → commit)

In chat, "create capability" becomes one or more MCP tool calls.
The recommended pattern is:

* LLM produces tool arguments that match the tool schema.
* Agent runtime calls `validateWrite` first (dry run).
* Only if valid: call the write tool to commit.

```text
User: "Create capability X"
  │
  ▼
LLM: selects write tool + fills arguments (schema-shaped)
  │
  ▼
1) validateWrite(operation="createCapability", payload={...})
  │      ├─ valid=false → return errors/suggestions → LLM fixes args → retry
  │      └─ valid=true  → proceed
  ▼
2) createCapability(payload={...}, client_request_id=...)
  │
  ▼
Tool result returns canonical created capability (id, revision, defaults applied)
```

**Why return canonical created objects:** projections/search indexes may lag; the tool result should be sufficient to confirm what was created without an immediate read-after-write.

See also: `backlog/535-mcp-read-optimizations.md` §2.2.1 for read-after-write expectations.

---

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

---

## Appendix A — Vendor patterns: Structured Outputs + Tool Calling for MCP Writes

**Last verified:** 2025-12-16
**Why this appendix exists:** MCP write tools are only "agent-friendly" if the *model can reliably produce arguments that match your tool schema* (and if the host/client implements a safe write UX). Most LLM vendors now provide some form of schema-constrained decoding for either (a) tool arguments or (b) text responses. This appendix captures the cross-vendor patterns that matter for MCP write.

### A.1 Terminology mapping (same idea, different names)

| Concept | MCP | OpenAI | Anthropic | Google (Gemini/Vertex) | AWS (Nova/Bedrock) | Cohere | Mistral | Snowflake |
|---|---|---|---|---|---|---|---|---|
| "Tool calling" | tool invocation | function / tool calling | tool use | function calling | tool use (function calling) | tools | function calling | n/a (focus is structured output) |
| "Schema-constrained outputs" | JSON Schema in `inputSchema` | Structured Outputs / JSON Schema | `input_schema` for tools | Structured Outputs | output schema / constrained decoding | Structured Outputs / `strict_tools` | (tool params JSON schema) | structured outputs (`response_format`) |
| "Write safety / confirmation" | tool annotations + client UX | confirmation UI for write connectors | app-controlled | app-controlled | app-controlled | app-controlled | app-controlled | app-controlled |

### A.2 Core design pattern: "Schema-as-contract" for MCP write tools

**Goal:** When a user says "create this capability / element", the model should emit a tool call like:

- Tool: `createElement`
- Args: `{ "type": "...", "name": "...", "description": "...", ... }`

…and those args should always validate against your tool schema.

**How it composes:**
1. **You define the contract** in the tool schema (MCP tool `inputSchema`).
2. **Vendor tool calling** (or structured outputs) constrains the model's output to that schema.
3. **MCP adapter still validates** inputs server-side (never trust the client).
4. Adapter enriches + translates into Domain intent (IDs, correlation, tenant, versioning, etc.).

### A.3 Portable schema subset (works across more vendors)

Because vendors support different subsets of JSON Schema, prefer a conservative "portable" subset for tool inputs:

**Recommended**
- Top-level: `{ "type": "object", "properties": {...}, "required": [...] }`
- Use `additionalProperties: false` (both for correctness and to prevent "schema drift")
- Use enums for "closed sets" (types, relationship kinds)
- Keep nesting shallow (≤ 2 levels) for best performance (esp. on providers that advise it)
- Use strings for IDs and "opaque references"
- Prefer arrays of objects for bulk operations (but cap sizes)

**Avoid unless you know the vendor supports it**
- Deep recursion and overly nested schemas (performance + context costs)
- Complex combinators (`oneOf`/`anyOf`/`allOf`) and advanced JSON Schema features
- Overly large schemas (some vendors impose limits across all tools / fields)

**Server-side rule:** even with strict mode, always validate the tool args again in the MCP adapter.

### A.4 Write-safety pattern: "Confirm before execute" for consequential tools

For writes that mutate persistent state (create/update/delete), implement:

1. **Client confirmation UI** before execution (especially for destructive tools).
2. **Tool annotations** to drive UX:
   - `readOnlyHint: true/false`
   - `destructiveHint: true/false`
   - `idempotentHint: true/false`

**Important:** tool annotations are UX hints, not security boundaries. Always enforce authz + policy server-side.

### A.5 Vendor notes you can rely on (what each vendor explicitly recommends)

#### A.5.1 OpenAI (tool calling + structured outputs)
**What to use**
- Use **function/tool calling** when you want the model to produce tool arguments.
- Use **Structured Outputs** when you want schema-constrained JSON in the model's *text* response.

**Notable implementation details (from docs)**
- Structured Outputs is designed to ensure outputs adhere to a supplied JSON schema.
- Structured Outputs via `text.format` uses `type: "json_schema"` with `strict: true` (Responses API).
- JSON mode is weaker than Structured Outputs; when using function calling, JSON mode is on by default.

**Write safety**
- In ChatGPT connectors, OpenAI states the UI shows explicit confirmation before executing write actions.

**Implications for MCP write**
- If your MCP host uses OpenAI models, lean on tool calling for argument shaping, but keep a server validator and confirmation gate for mutations.

#### A.5.2 Microsoft Azure OpenAI
**What to use**
- Azure positions "structured outputs" as stricter than older JSON mode and recommends it for function calling and multi-step workflows.

**Implications**
- Same "schema-as-contract" approach; confirm writes; validate server-side.

#### A.5.3 Google Gemini API + Vertex AI
**Gemini Structured Outputs**
- Gemini supports JSON Schema constrained responses and positions it as guaranteeing predictable, parsable, type-safe results and enabling programmatic detection of refusals.

**Vertex function calling best practices**
- Use system instructions to provide missing context (date/time/location).
- Use low temperature (0 or similar) to reduce hallucinations.
- Validate with the user before executing calls that have significant consequences (orders, DB updates).
- Use structured output together with function calling for consistently formatted non-tool responses.

**Implications**
- Vendor explicitly endorses: low temperature + confirm-before-execute for consequential writes.

#### A.5.4 Anthropic Claude
**Tool use flow**
- Claude tool use has an explicit "stop_reason: tool_use", then the client executes and returns a `tool_result`.

**Tool definition + examples**
- Tool `input_examples` must validate against the tool's `input_schema`; invalid examples cause API errors.
- Anthropic provides a "tool runner" (SDK feature) to manage execution loops + validation.

**Implications**
- If your MCP host uses Claude, invest in "tool definitions that teach" (good descriptions + valid examples), but watch token cost of examples.

#### A.5.5 AWS (Amazon Nova / Bedrock)
**Structured output guidance**
- AWS recommends providing an output schema and using `temperature=0` ("greedy decoding") for structured output.
- AWS notes outputs are not fully deterministic and may still vary from the output schema in pure prompting scenarios.

**Tool use guidance**
- Best practices include: temperature=0, limit JSON schemas to two layers of nesting, and make tool name/description/schema explicit.
- Amazon Nova "understanding" models support only a subset of JSON Schema for tool input schemas; the top-level must be an object, and only `type`, `properties`, `required` are supported at the top level.

**Implications**
- For portability to Nova: keep tool schemas shallow and simple; do not depend on advanced schema features.

#### A.5.6 Cohere
**Structured Outputs**
- Cohere supports Structured Outputs for JSON and for Tools; for Tools, `strict_tools=true` enforces tool calls that adhere exactly to the provided schema.
- Constraints: only supported in Chat API V2, at least one required parameter is needed, and Cohere caps total fields across all tools in a call.

**Implications**
- When using Cohere with an MCP host, `strict_tools` is the closest analogue to "strict tool argument shaping"; still validate server-side.

#### A.5.7 Mistral
**Function calling**
- Mistral function calling uses JSON schema for tool parameter specifications.
- Offers `tool_choice` modes and optional parallel tool calling.

**Implications**
- Same design: schemas must be clear, minimal, and stable; consider disabling parallel tool calls for write tools if you need serialized safety checks.

#### A.5.8 Snowflake Cortex (AI_COMPLETE structured outputs)
**Structured outputs**
- Snowflake supports structured outputs by supplying a JSON schema; it verifies each generated token against the schema to ensure conformance.
- Snowflake documents extra requirements when using OpenAI GPT models (e.g., `additionalProperties: false` in every node; `required` includes all properties).
- Snowflake also supports schema references (`$ref`) for structured outputs.

**Implications**
- If your architecture uses Snowflake as the LLM gateway, structured outputs can offload much of the "retry until valid JSON" logic, but you still must validate + govern writes.

### A.6 Concrete "MCP write" recipe (vendor-agnostic)

**Step 1: keep tool inputs semantic**
- Tool args are "what the model is good at":
  - `type`, `name`, `description`, `properties`, human-friendly references
- Adapter adds "what the model is bad at":
  - IDs, correlation, timestamps, versioning, policy state

**Step 2: enable strict/schema-constrained decoding when available**
- Prefer:
  - vendor tool calling with schema constraints
  - "strict tools" modes (where supported)
  - "structured outputs" for text responses that must be JSON

**Step 3: validate + normalize in the adapter**
- Validate tool args against schema
- Normalize enums ("ApplicationComponent" → `APPLICATION_COMPONENT`)
- Resolve references ("Billing Service" → lookup ID; if ambiguous, return "needs_disambiguation" error)

**Step 4: confirm before executing writes**
- For destructive tools: force explicit user confirmation
- For non-destructive creates: still consider confirmation if it writes to a shared model or has governance implications

**Step 5: return a rich, structured response**
- Always return:
  - `success`
  - created IDs
  - next-step suggestions (e.g., "create relationship", "add to view")

### A.7 Reference links (vendor docs)

**OpenAI**
- Structured outputs: https://platform.openai.com/docs/guides/structured-outputs
- Function calling: https://platform.openai.com/docs/guides/function-calling
- ChatGPT connector write confirmation (help center): https://help.openai.com/en/articles/12584461-developer-mode-apps-and-full-mcp-connectors-in-chatgpt-beta

**Microsoft Azure OpenAI**
- Structured outputs: https://learn.microsoft.com/en-us/azure/ai-foundry/openai/how-to/structured-outputs

**Google**
- Gemini structured outputs: https://ai.google.dev/gemini-api/docs/structured-output
- Gemini function calling: https://ai.google.dev/gemini-api/docs/function-calling
- Vertex AI function calling best practices: https://docs.cloud.google.com/vertex-ai/generative-ai/docs/multimodal/function-calling

**Anthropic**
- Tool use overview: https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
- Implement tool use (examples + tool runner): https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use

**AWS (Amazon Nova / Bedrock)**
- Nova structured output prompting: https://docs.aws.amazon.com/nova/latest/userguide/prompting-structured-output.html
- Nova tool use best practices: https://docs.aws.amazon.com/nova/latest/nova2-userguide/using-tools.html
- Nova tool definition / JSON Schema subset notes: https://docs.aws.amazon.com/nova/latest/userguide/tool-use-definition.html

**Cohere**
- Structured outputs: https://docs.cohere.com/docs/structured-outputs
- strict_tools changelog: https://docs.cohere.com/changelog/structured-outputs-tools

**Mistral**
- Function calling: https://docs.mistral.ai/capabilities/function_calling

**Snowflake**
- AI_COMPLETE structured outputs: https://docs.snowflake.com/en/user-guide/snowflake-cortex/complete-structured-outputs
- $ref support release note: https://docs.snowflake.com/en/release-notes/2025/other/2025-05-19-complete-structured-output-json-refs

**MCP spec (tool annotations)**
- Tool annotations: https://modelcontextprotocol.io/legacy/concepts/tools
