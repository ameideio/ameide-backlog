# 032 – API Extended Routes Implementation

Status: **In Progress**  
Created: 2025-07-24

## Overview

Implementation of the extended API routes for workflows, agents, and transformations based on proto definitions.

## Completed (2025-07-24)

### 1. Workflow Routes (`/v1/workflows/*`)
- ✅ `POST /workflows/execute` - Execute a workflows
- ✅ `GET /workflows/{workflows_id}/executions` - List workflows executions
- ✅ `POST /workflows/{execution_id}/cancel` - Cancel running workflows
- ✅ `GET /workflows/{execution_id}/status` - Get execution status

### 2. Agent Routes (`/v1/agents/*`)
- ✅ `POST /agents/execute` - Execute agent (non-streaming)
- ✅ `POST /agents/stream` - Execute agent with SSE streaming
- ✅ `POST /agents/{run_id}/interrupt` - Interrupt running agent
- ✅ `GET /agents/runs` - List agent runs with filtering
- ✅ `GET /agents/{run_id}/status` - Get agent run status

### 3. Transformation Routes (`/v1/transformations/*`)
- ✅ `POST /transformations/transform` - Transform model
- ✅ `POST /transformations/transform/file` - Transform uploaded file
- ✅ `POST /transformations/validate` - Validate model
- ✅ `GET /transformations/discover` - Discover available models
- ✅ `GET /transformations/capabilities` - Get transformation capabilities

### 4. Service Implementations
- ✅ `WorkflowService` - Mock implementation with in-memory storage
- ✅ `AgentService` - Mock implementation with streaming support
- ✅ `TransformationService` - Mock implementation with format detection

### 5. Dependency Injection
- ✅ Updated `dependencies.py` with new service providers
- ✅ Added service factory functions

## In Progress

### Proto-Based Generation
- [ ] Run `generate_routes.py` to create proto-derived routes
- [ ] Compare generated vs manual implementations
- [ ] Merge generated code appropriately

## Next Steps

### 1. Service Implementation Enhancement
- [ ] Connect WorkflowService to Temporal client
- [ ] Connect AgentService to LangGraph runtime
- [ ] Connect TransformationService to actual transpilers

### 2. Streaming Infrastructure
- [ ] Enhance SSE implementation for agents
- [ ] Add WebSocket support for bidirectional communication
- [ ] Implement proper backpressure handling

### 3. Error Handling
- [ ] Standardize error responses across all routes
- [ ] Add proper status codes for each error type
- [ ] Implement retry guidance in error responses

### 4. Testing
- [ ] Unit tests for each service
- [ ] Integration tests for routes
- [ ] End-to-end tests with mock clients

## Dependencies

- `core-proto` - All request/response types
- `core-workflows-model2temporal` - For workflows transformation
- `core-agent-model2langgraph` - For agent transformation
- External: Temporal client, LangGraph runtime

## Technical Decisions

1. **Mock First**: All services start with mock implementations
2. **Streaming**: Use SSE for unidirectional, WebSocket for bidirectional
3. **File Handling**: Temporary files for upload/download operations
4. **Pagination**: Consistent offset/limit pattern across all list endpoints

## Acceptance Criteria

1. All routes return proper proto-based responses
2. Streaming endpoints work with standard SSE clients
3. File upload/download handles large files efficiently
4. Error responses follow RFC 7807 (Problem Details)
5. All endpoints documented in OpenAPI spec