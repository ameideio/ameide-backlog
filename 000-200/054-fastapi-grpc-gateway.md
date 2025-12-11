# 054: FastAPI-gRPC Gateway Service

## Overview

Implement a unified API gateway service that exposes all proto services via both gRPC and REST protocols. This gateway will serve as the primary entry point for all client applications, providing protocol flexibility, authentication, and observability.

## Goals

- Single service exposing both gRPC (port 50051) and REST (port 8000)
- Auto-generate REST endpoints from proto definitions
- WebSocket support for real-time updates
- Unified authentication and authorization
- Full observability with OpenTelemetry
- Service discovery and routing

## Current State

- ✅ Proto services defined (IPA, Workflows, Agents, Users, BPMN)
- ✅ Observability patterns established
- ✅ FastAPI usage in existing services
- ✅ Unified gateway service implemented (core-api-grpc-rest)
- ✅ gRPC server implementation with dual-protocol support
- ✅ BPMN service fully accessible via both protocols
- ⚠️ Other services have placeholder implementations

## Proposed Solution

### Service Architecture

```
services/core-api-grpc-rest/      # ✅ Implemented
├── main.py                       # ✅ FastAPI app with gRPC server
├── config.py                     # ✅ Configuration management
├── grpc_server.py               # ✅ gRPC server setup
├── grpc_services/               # ✅ gRPC service implementations
│   ├── __init__.py              # ✅
│   ├── base.py                  # ✅ Base service class
│   ├── bpmn_service.py          # ✅ Fully implemented
│   ├── workflows_service.py      # ⚠️ Placeholder
│   ├── agent_service.py         # ⚠️ Placeholder
│   ├── ipa_service.py           # ⚠️ Placeholder
│   └── user_service.py          # ⚠️ Placeholder
├── rest_gateway/                # ✅ REST API routes
│   ├── __init__.py              # ✅
│   ├── bpmn_routes.py           # ✅ Full BPMN REST API
│   ├── design_routes.py         # ⚠️ Placeholder routes
│   ├── runtime_routes.py        # ⚠️ Placeholder routes
│   └── websocket.py             # ✅ WebSocket handlers
├── middleware/
│   ├── __init__.py              # ✅
│   ├── auth.py                  # ✅ JWT authentication
│   ├── logging.py               # ✅ Request logging
│   └── error_handler.py         # ✅ Global error handling
├── BUILD.bazel                  # ✅
├── Dockerfile                   # ✅
└── README.md                    # ✅
```

### Key Components

1. **Dual Protocol Server**
   ```python
   # Concurrent gRPC and HTTP servers
   - FastAPI on port 8000
   - gRPC on port 50051
   - Shared service instances
   - Common middleware stack
   ```

2. **REST Gateway Generation**
   - Use proto annotations for REST mapping
   - Auto-generate OpenAPI documentation
   - Support for streaming via SSE
   - File upload/download handling

3. **WebSocket Support**
   - Real-time workflows execution updates
   - Agent status streaming
   - Collaborative editing notifications
   - Pub/sub for model changes

4. **Service Routing**
   ```python
   # Dynamic service discovery
   - Service registry integration
   - Health check endpoints
   - Circuit breaker patterns
   - Load balancing support
   ```

## Technical Approach

### gRPC-REST Mapping
```proto
service WorkflowService {
  rpc GetWorkflow(GetWorkflowRequest) returns (Workflow) {
    option (google.api.http) = {
      get: "/api/v1/workflows/{id}"
    };
  }
  
  rpc ListWorkflows(ListWorkflowsRequest) returns (ListWorkflowsResponse) {
    option (google.api.http) = {
      get: "/api/v1/workflows"
    };
  }
}
```

### Authentication Flow
1. JWT tokens for REST API
2. mTLS for gRPC (optional)
3. API keys for service-to-service
4. OAuth2 integration ready

### Observability Integration
- OpenTelemetry for traces
- Prometheus metrics
- Structured logging
- Health check endpoints
- Request/response logging

## Success Criteria

- [x] Both gRPC and REST protocols working
- [x] BPMN service fully exposed via both protocols
- [ ] All other proto services exposed (Agent, Workflow, IPA, User)
- [x] WebSocket real-time updates functional
- [x] Authentication/authorization implemented (JWT-based)
- [x] OpenAPI documentation generated (auto-generated at /docs)
- [ ] Performance: <10ms overhead per request (needs testing)
- [ ] 99.9% uptime target (needs monitoring)

## Dependencies

- All proto service definitions
- Service implementations
- Authentication framework
- Observability infrastructure

## Estimated Effort

- Basic gateway setup: 2 days ✅ DONE
- gRPC service implementations: 3 days (1 day done, 2 days remaining)
- REST route generation: 2 days (1 day done, 1 day remaining)
- WebSocket handlers: 2 days ✅ DONE
- Authentication/middleware: 2 days ✅ DONE
- Testing and documentation: 3 days (0.5 days done, 2.5 days remaining)
- **Total: 14 days** (7.5 days completed, 6.5 days remaining)

## Remaining Work

### High Priority
1. **Implement remaining gRPC services**
   - Agent service with full CRUD and execution
   - Workflow service with full CRUD and execution
   - IPA service with full CRUD, deployment, and execution
   - User service with authentication logic

2. **Complete REST gateway routes**
   - Full implementation of design_routes.py
   - Full implementation of runtime_routes.py
   - Add missing transformation endpoints

3. **Integration and testing**
   - Unit tests for all services
   - Integration tests for dual-protocol scenarios
   - End-to-end tests with real database
   - Performance benchmarking

### Medium Priority
4. **Enhanced features**
   - Service discovery integration
   - Circuit breaker patterns
   - Rate limiting
   - Request retry logic
   - Batch operations support

5. **Production readiness**
   - mTLS support for gRPC
   - OAuth2 integration
   - API key management
   - Metrics and dashboards
   - Deployment configurations

### Low Priority
6. **Advanced features**
   - GraphQL support
   - Server-sent events for streaming
   - API versioning strategy
   - Multi-region support

## Risks

- Protocol conversion overhead (mitigated by shared service instances)
- Streaming complexity (basic implementation done, needs optimization)
- Authentication consistency (JWT implemented, needs testing)
- Service discovery challenges (not yet implemented)
- Performance at scale (needs load testing)