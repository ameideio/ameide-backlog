# 044: Proto-Based APIs Implementation

## Overview

Implement production-ready gRPC and REST API services that expose core business functionality through well-defined proto contracts. Both services will use the same domain services and follow clean architecture principles, with code generation to minimize duplication and ensure consistency.

## MVP Scope Decomposition

This backlog item has been decomposed into the following deliverables:

### MVP 1: Core API Infrastructure (This Document)
- Authentication & authorization framework
- Error mapping and handling
- Basic CRUD operations for all entities
- gRPC service implementation
- REST API generation via grpc-gateway

### MVP 2: Advanced Features (Separate Backlog Item)
- Streaming APIs for real-time updates
- Saga pattern implementation
- Advanced query capabilities
- Batch operations

### MVP 3: Performance & Scale (Separate Backlog Item)
- Connection pooling optimization
- Caching layer
- Rate limiting enhancements
- Horizontal scaling patterns

## Background

The system currently has:
- Proto definitions in `packages/ameide_core_proto` with service contracts and common types
- Proto adapters in `packages/ameide_core_proto_adapters` for bidirectional conversion
- Domain services in `packages/ameide_core-services` (User, Workflow, Agent)
- Repository implementations in `packages/ameide_core-storage`
- Robust auth system supporting JWT, Auth0, Keycloak with multi-tenancy
- Observability standards with OpenTelemetry and structured logging

What's missing:
- IPA service implementation in core-services
- IPA graph in storage layer
- IPA proto adapter
- Complete gRPC and REST API implementations
- Consistent domain error hierarchy
- Transaction boundaries and saga patterns
- Repository pattern fixes (e.g., user roles storage)
- REST generation strategy decision (grpc-gateway vs alternatives)
- Streaming ingress/back-pressure handling
- Complete error mapping coverage
- Proto versioning rules and Buf breaking-change policy

## Requirements

### Functional Requirements

1. **gRPC API Service**
   - Implement all proto service contracts
   - Support streaming for real-time updates
   - Enable reflection for development
   - Handle large message sizes

2. **REST API Service**
   - RESTful endpoints matching gRPC functionality
   - OpenAPI documentation
   - JSON request/response format
   - CORS support for web clients

3. **Authentication & Authorization**
   - JWT token validation via OIDC standard
   - Single AuthProvider interface (Auth0/Keycloak as implementations)
   - Role-based access control (RBAC)
   - API key authentication for service accounts
   - Pure context DTOs (no transport objects in domain)

4. **Error Handling**
   - Consistent error responses
   - Domain error to API error mapping
   - Request validation with clear messages
   - Rate limiting and abuse prevention

### Non-Functional Requirements

1. **Performance**
   - < 100ms p95 latency for simple operations
   - < 500ms p95 for complex operations (with gzip for large messages)
   - Support 1000+ concurrent connections
   - Efficient proto serialization
   - Performance budget: max 10MB payload, 1000 RPS per instance

2. **Observability**
   - Structured logging with correlation IDs
   - OpenTelemetry traces and metrics
   - Business event tracking
   - Health check endpoints

3. **Security**
   - TLS/mTLS support
   - Input validation and sanitization
   - SQL injection prevention
   - XSS protection for REST API

4. **Scalability**
   - Horizontal scaling support
   - Connection pooling
   - Graceful shutdown
   - Circuit breakers for external calls

## Technical Design

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                             │
├──────────────────────┬──────────────────────────────────────────┤
│    gRPC Clients      │         REST Clients                     │
│  (Internal Services) │    (Web Apps, External)                  │
└──────────┬───────────┴───────────────┬──────────────────────────┘
           │                           │
           ▼                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API Gateway Layer                           │
├──────────────────────┬──────────────────────────────────────────┤
│   gRPC API Service   │         REST API Service                 │
│                      │                                          │
│ • Proto Services     │  • Generated Routes*                    │
│ • Interceptors       │  • Middleware                           │
│ • Streaming          │  • OpenAPI                             │
└──────────┬───────────┴───────────────┬──────────────────────────┘
           │                           │
           ▼                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                 Context & Error Mapping Layer                    │
│  • RequestContext DTO (no transport objects)                    │
│  • Central ErrorRegistry (domain → {grpc, http} codes)          │
│  • Correlation ID propagation                                   │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Proto Adapter Layer                            │
│         (Pure data conversion, no orchestration)                │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Service Layer                                 │
│              (Business Logic & Use Cases)                        │
└─────────────────────────────────────────────────────────────────┘

* REST routes generated from proto definitions via grpc-gateway or similar
```

### Service Structure

```
services/
├── api-base/                    # Shared base image
│   └── Dockerfile.base
├── api-common/                  # Shared modules
│   ├── error_registry.py        # Central error mapping
│   ├── context.py              # RequestContext DTO
│   ├── auth_provider.py        # AuthProvider interface
│   └── correlation.py          # Trace propagation
├── core-api-grpc/
│   ├── BUILD.bazel
│   ├── Dockerfile              # FROM api-base
│   ├── main.py
│   ├── config.py
│   ├── container.py            # Shared DI wiring
│   ├── grpc_server.py
│   ├── interceptors/
│   │   ├── auth.py
│   │   ├── errors.py
│   │   └── metrics.py
│   └── services/               # Proto service impls
└── core-api-rest/
    ├── BUILD.bazel
    ├── Dockerfile              # FROM api-base
    ├── main.py
    ├── config.py
    ├── container.py            # Shared DI wiring
    ├── generated/              # From proto
    │   ├── routes.py
    │   └── schemas.py
    └── middleware/
```

### REST API Structure

```
services/core-api-rest/
├── BUILD.bazel
├── Dockerfile
├── requirements.txt
├── main.py                # FastAPI app
├── config.py              # Configuration
├── container.py           # DI container
├── dependencies.py        # FastAPI dependencies
├── middleware.py          # Custom middleware
├── auth.py               # Authentication
├── routes/               # API routes
│   ├── __init__.py
│   ├── users.py
│   ├── workflows.py
│   ├── agents.py
│   └── ipas.py
├── schemas/              # Pydantic models
│   ├── __init__.py
│   ├── common.py
│   ├── users.py
│   ├── workflows.py
│   ├── agents.py
│   └── ipas.py
└── tests/
    ├── unit/
    ├── integration/
    └── fixtures/
```

### Key Components

#### 0. Error Mapping Infrastructure

```python
class ErrorMapper:
    """Central domain to API error mapping."""
    
    @staticmethod
    def to_grpc_status(domain_error: Exception) -> grpc.StatusCode:
        mapping = {
            NotFoundError: grpc.StatusCode.NOT_FOUND,
            ValidationError: grpc.StatusCode.INVALID_ARGUMENT,
            AuthorizationError: grpc.StatusCode.PERMISSION_DENIED,
            UnauthenticatedError: grpc.StatusCode.UNAUTHENTICATED,
            ConflictError: grpc.StatusCode.ALREADY_EXISTS,
            RateLimitError: grpc.StatusCode.RESOURCE_EXHAUSTED,
            ExternalServiceError: grpc.StatusCode.UNAVAILABLE,
            DeadlineExceededError: grpc.StatusCode.DEADLINE_EXCEEDED,
            DataLossError: grpc.StatusCode.DATA_LOSS,
            InternalError: grpc.StatusCode.INTERNAL,
        }
        return mapping.get(type(domain_error), grpc.StatusCode.UNKNOWN)
    
    @staticmethod
    def to_http_status(domain_error: Exception) -> int:
        mapping = {
            NotFoundError: 404,
            ValidationError: 400,
            AuthorizationError: 403,
            UnauthenticatedError: 401,
            ConflictError: 409,
            RateLimitError: 429,
            ExternalServiceError: 503,
            DeadlineExceededError: 504,
            DataLossError: 500,
            InternalError: 500,
        }
        return mapping.get(type(domain_error), 500)

class UnitOfWork:
    """Transaction management pattern."""
    
    def __init__(self, session_factory):
        self.session_factory = session_factory
        self._session = None
    
    async def __aenter__(self):
        self._session = self.session_factory()
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if exc_type:
            await self._session.rollback()
        else:
            await self._session.commit()
        await self._session.close()
    
    @property
    def session(self):
        return self._session
```

#### 1. Authentication & Validation Interceptors (gRPC)

```python
class AuthInterceptor(grpc.aio.ServerInterceptor):
    """JWT validation and context injection."""
    
    def __init__(self, auth_provider: AuthProvider, role_requirements: Dict[str, List[str]]):
        self.auth_provider = auth_provider
        self.role_requirements = role_requirements
    
    async def intercept_service(self, continuation, handler_call_details):
        # Extract token from metadata
        # Validate with auth provider
        # Inject user context as DTO
        # Check role requirements
        # Add request signing for service-to-service
        pass

class RequestValidationInterceptor(grpc.aio.ServerInterceptor):
    """Request validation and rate limiting."""
    
    def __init__(self, proto_validators: Dict[str, Callable]):
        self.validators = proto_validators
    
    async def intercept_service(self, continuation, handler_call_details):
        # Validate request context
        # Validate business rules
        # Check rate limits
        # Enforce field behaviors from proto
        pass
```

#### 2. Proto Service Implementation with UoW Pattern

```python
class UserGRPCServiceImpl(users_pb2_grpc.UserServiceServicer):
    """gRPC service implementation for users."""
    
    @inject
    def __init__(
        self, 
        user_service: UserService = Provide[Container.user_service],
        error_mapper: ErrorMapper = Provide[Container.error_mapper],
        uow_factory: UnitOfWorkFactory = Provide[Container.uow_factory]
    ):
        self.user_service = user_service
        self.error_mapper = error_mapper
        self.uow_factory = uow_factory
    
    async def CreateUser(self, request, context):
        try:
            # Context already injected as DTO by interceptor
            request_context: RequestContext = context.request_context
            
            # Transaction boundary
            async with self.uow_factory.create() as uow:
                user = await self.user_service.create_user(
                    name=request.user.name,
                    email=request.user.email,
                    roles=request.user.roles,  # Fixed: list support
                    context=request_context,
                    uow=uow
                )
            
            # Convert to proto
            return CreateUserResponse(user=UserAdapter.to_proto(user))
            
        except DomainError as e:
            # Consistent error mapping
            grpc_status = self.error_mapper.to_grpc_status(e)
            context.set_code(grpc_status)
            context.set_details(str(e))
            # Add error to response metadata for debugging
            context.set_trailing_metadata([
                ('error-type', type(e).__name__),
                ('error-id', str(uuid.uuid4()))
            ])
            raise
```

#### 3. REST Route Pattern

```python
@router.get("/users/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: UUID,
    current_user: User = Depends(get_current_user),
    user_service: UserService = Depends(get_user_service),
):
    """Get user by ID."""
    user = await user_service.get_user(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return UserResponse.from_domain(user)
```

## Streaming & Ingress Strategy

### Streaming Back-pressure
```python
class StreamingInterceptor(grpc.aio.ServerInterceptor):
    """Handle back-pressure for streaming RPCs."""
    
    def __init__(self, max_concurrent_streams: int = 1000):
        self.semaphore = asyncio.Semaphore(max_concurrent_streams)
        self.ingress_limiter = IngressRateLimiter()
    
    async def intercept_service(self, continuation, handler_call_details):
        if self._is_streaming(handler_call_details):
            async with self.semaphore:
                # Apply ingress limits
                await self.ingress_limiter.check_rate(handler_call_details)
                return await continuation(handler_call_details)
        return await continuation(handler_call_details)
```

### Proto Versioning Strategy
```yaml
# buf.yaml
version: v1
lint:
  use:
    - DEFAULT
  except:
    - PACKAGE_VERSION_SUFFIX  # We use directories for versioning
breaking:
  use:
    - FILE
  except:
    - FIELD_SAME_DEFAULT  # Allow changing defaults

# Directory structure:
# proto/
#   ameide/
#     user/v1/      # Stable v1 API
#     user/v1alpha/ # Alpha features
#     user/v2/      # Breaking changes
```

## Implementation Plan

### Phase 0: Foundation & Prerequisites (Week 1)

**Day 1-2: Missing Domain Infrastructure**
- [ ] Implement IPAService in core-services
- [ ] Create IPA graph with SqlIPARepository
- [ ] Implement IPA proto adapter
- [ ] Fix user graph role storage (string → list)
- [ ] Define domain error hierarchy

**Day 3: Common Components**
- [ ] Create api-common module with ErrorRegistry
- [ ] Define RequestContext DTO
- [ ] Implement AuthProvider interface
- [ ] Add UnitOfWork pattern for transactions
- [ ] Setup shared DI wiring module
- [ ] Choose REST generation strategy (grpc-gateway recommended)

**Day 4: Code Generation Setup**
- [ ] Setup Buf for proto linting/breaking changes
- [ ] Configure grpc-gateway for REST generation
- [ ] Create make targets for client generation
- [ ] Create api-base Docker image
- [ ] Define proto versioning directory scheme
- [ ] Setup Buf breaking-change CI pipeline

**Day 5: Error & Validation Framework**
- [ ] Implement domain error hierarchy
- [ ] Create ErrorMapper for domain→API mapping
- [ ] Add RequestValidationInterceptor
- [ ] Setup proto message validators

### Phase 1: gRPC Service (Week 1.5)

**Day 1-2: Authentication & Infrastructure**
- [ ] Implement auth interceptor using AuthProvider
- [ ] Create request validation interceptor
- [ ] Create context injection interceptor
- [ ] Add correlation ID propagation
- [ ] Setup error mapping with ErrorMapper
- [ ] Add request signing for service-to-service

**Day 3-4: Service Implementations**
- [ ] Complete UserGRPCServiceImpl
- [ ] Implement WorkflowGRPCServiceImpl
- [ ] Implement AgentGRPCServiceImpl
- [ ] Implement IPAGRPCServiceImpl

**Day 5: Testing & Documentation**
- [ ] Unit tests for interceptors
- [ ] Integration tests for services
- [ ] Performance benchmarks
- [ ] Update API documentation

### Phase 2: REST API (Week 2)

**Day 1-2: Core Infrastructure**
- [ ] Create FastAPI app structure
- [ ] Implement authentication dependencies
- [ ] Add middleware for logging/metrics
- [ ] Create base schemas

**Day 3-4: Route Implementations**
- [ ] Implement user routes
- [ ] Implement workflows routes
- [ ] Implement agent routes
- [ ] Implement IPA routes

**Day 5: Testing & Documentation**
- [ ] Unit tests for routes
- [ ] Integration tests
- [ ] Generate OpenAPI spec
- [ ] API usage examples

### Phase 3: Production Readiness (Week 3)

**Day 1-2: Observability**
- [ ] Add OpenTelemetry instrumentation
- [ ] Create custom business metrics
- [ ] Implement distributed tracing
- [ ] Add performance monitoring

**Day 3: Security Hardening**
- [ ] Security audit
- [ ] Rate limiting
- [ ] Input validation
- [ ] TLS configuration

**Day 4-5: Deployment**
- [ ] Update Dockerfiles
- [ ] Create Kubernetes manifests
- [ ] Add health checks
- [ ] Load testing

## Testing Strategy

### Unit Tests
- Mock repositories and external services
- Test business logic in isolation
- Verify error handling
- Check authorization rules

### Integration Tests
- Use test database
- Test full request/response cycle
- Verify proto conversions
- Check transaction handling

### Performance Tests
- Load testing with k6
- Measure latency percentiles
- Test concurrent connections
- Memory leak detection
- Flamegraph profiling for proto↔pydantic conversions
- Monitor allocation churn
- Connection pool efficiency
- Proto message caching effectiveness

### Security Tests
- OWASP API Security Top 10
- Authentication bypass attempts
- SQL injection tests
- Rate limit testing

## Deployment Considerations

### Container Image
- Shared api-base stage
- Multi-stage build FROM api-base
- Non-root user
- Minimal base image
- Security scanning
- Parametrized CMD for grpc/rest

### Kubernetes Resources
- Deployment with HPA
- Service with load balancing
- ConfigMap for configuration
- Secret for sensitive data

### Monitoring
- Prometheus metrics
- Grafana dashboards
- Alert rules
- SLO definitions

## Success Criteria

1. **Functionality**
   - All proto services implemented (including IPA)
   - REST endpoints generated from proto via grpc-gateway
   - Authentication via unified AuthProvider
   - Error handling via central mapper with 100% coverage
   - Idempotency keys for mutations
   - Pagination/filtering/field-masks consistent
   - Transaction boundaries properly managed
   - Repository inconsistencies fixed
   - Streaming with proper back-pressure handling
   - Proto versioning scheme enforced

2. **Performance**
   - Meeting latency targets
   - Handling concurrent load
   - Efficient resource usage
   - No memory leaks

3. **Quality**
   - 80%+ test coverage
   - No critical security issues
   - Clean code principles
   - Comprehensive documentation

4. **Operations**
   - Easy to deploy
   - Observable behavior
   - Graceful degradation
   - Quick recovery

## Dependencies

- Proto definitions must be stable
- Domain services fully implemented (including missing IPA service)
- Auth providers configured
- Database migrations complete
- Repository pattern inconsistencies resolved
- Domain error hierarchy defined

## Risks

1. **Proto Changes**: Breaking changes to proto definitions
   - Mitigation: Buf CI for breaking change detection, versioning (v1alpha → v1)

2. **Performance**: Not meeting latency requirements
   - Mitigation: Performance budget doc, profiling, uvloop for async

3. **Security**: Authentication bypass or data leaks
   - Mitigation: Single AuthProvider interface, security review

4. **Complexity**: Difficult to maintain two API surfaces
   - Mitigation: REST generation from proto, shared error registry

5. **Code Drift**: Diverging configurations between services
   - Mitigation: Shared DI wiring, common base image

## Governance

- Proto changes require architecture review
- Breaking changes need migration plan
- Performance budgets enforced in CI
- Contract tests with golden proto payloads
- Field deprecation strategy for proto evolution
- SLI/SLO dashboards from day one

## Architectural Review Findings (2025-07-26)

### Key Concerns Identified

1. **Scope creep** – One backlog covers MVP + streaming + saga + rate limiting
   - Impact: Impossible to deliver in one PI
   - Action: Focus on MVP 1 (CRUD + unary RPC only), defer streaming to MVP 2

2. **Gateway undecided** – grpc-gateway vs Connect-ES blocks proto to REST generation
   - Recommendation: **Connect-ES** for unified gRPC, REST, WS support
   - Action: Decide Day 0-5, slim down to MVP scope

3. **Back-pressure** – Streaming ingress limits hand-waved
   - Risk: Potential overload without proper flow control
   - Action: Standardize on Connect-ES or grpc-gateway v3 + HTTP/2 flow-control

4. **Error coverage** – Unauthenticated, deadline-exceeded, unknown still missing
   - Current: Only partial error mapping implemented
   - Action: Complete ErrorMapper with 100% domain error coverage

5. **Versioning rules** – Buf policy sketched but not wired into CI
   - Risk: Breaking changes slip through
   - Action: Add Buf breaking-change CI pipeline immediately (Day 0-2)

### Missing Prerequisites

- IPA service implementation in core-services
- IPA graph in storage layer
- IPA proto adapter
- Domain error hierarchy definition
- Repository pattern fixes (e.g., user roles storage)
- UnitOfWork pattern for transactions

### Immediate Actions (P0/P1)

- [ ] Fix duplicate enums, add Buf lint & breaking-change CI (Day 0-2)
- [ ] Decide gateway tool (recommend Connect-ES) and slim to MVP 1 (Day 0-5)
- [ ] Implement missing IPA service, graph, and adapter (Day 1-2)
- [ ] Complete domain error hierarchy and ErrorMapper (Day 4)
- [ ] Setup Buf breaking-change guard in CI pipeline

### Revised Implementation Timeline

**Phase 0: Foundation & Prerequisites (Week 1)**
- Complete missing domain infrastructure (IPA service, graph, adapter)
- Choose and setup REST generation strategy
- Implement error framework and validation

**Phase 1: MVP 1 - Core API (Week 1.5)**
- Basic CRUD operations for all entities
- Authentication & authorization
- Unary RPC only (no streaming yet)
- REST generation via chosen gateway

**Phase 2: MVP 2 - Advanced Features (Separate backlog)**
- Streaming APIs with back-pressure
- Saga pattern implementation
- Advanced query capabilities
- Batch operations

## Additional Considerations

1. **Graceful Shutdown**
   - Max 30s drain time
   - Close DB connection pools
   - Flush telemetry data
   - Complete in-flight requests

2. **Ingress Proxy Limits**
   - Max message size: 10MB (configurable)
   - Max concurrent streams: 1000 per connection
   - Stream idle timeout: 5 minutes
   - Connection idle timeout: 15 minutes

2. **Context Propagation Rules**
   - trace-id: OpenTelemetry standard
   - request-id: UUID per request
   - user-id: From JWT claims
   - tenant-id: From JWT or header

3. **Circuit Breakers**
   - Client-side for external calls
   - Hystrix-style with fallbacks
   - Saga patterns for distributed transactions
   - Compensation logic for failures

## References

- [gRPC Best Practices](https://grpc.io/docs/guides/performance/)
- [FastAPI Security](https://fastapi.tiangolo.com/tutorial/security/)
- [OpenTelemetry Python](https://opentelemetry.io/docs/instrumentation/python/)
- [Proto Style Guide](https://developers.google.com/protocol-buffers/docs/style)
- [Buf Schema Registry](https://buf.build/docs/)
- [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)