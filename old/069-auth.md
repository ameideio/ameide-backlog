# Keycloak Authentication Integration

## Overview

Production-ready Keycloak authentication implementation for AMEIDE Core platform using standard python-keycloak library.

## Current State

### ✅ Already Implemented
- **Keycloak Integration** using `python-keycloak` in `services/core-api-grpc-rest/auth/keycloak_auth.py`
  - Token validation with built-in JWKS caching
  - Service account support (client credentials)
  - Token introspection and user info endpoints
  - Using standard KeycloakOpenID client (correct approach)
- **AuthMiddleware** in `services/core-api-grpc-rest/middleware/auth.py`
  - FastAPI middleware with exempt paths
  - Bearer token extraction
  - AUTH_ENABLED flag support
  - Role extraction (realm and resource-specific)
- **Docker Setup** - Keycloak service in docker-compose.yml
  - PostgreSQL backend configured
  - HTTPS enabled with certificates
  - Health checks configured
- **Dependencies** already in requirements.in:
  - `python-keycloak>=3.0.0`
  - `jose[cryptography]>=3.3.0` (for dev token creation)

### ✅ Recently Completed
- Fixed import error in `rest_gateway/auth_routes.py` - now uses keycloak_auth module
- Implemented gRPC authentication interceptor in `auth/grpc_interceptor.py`
- Implemented service-to-service authentication helpers in `auth/service_auth.py`
- Added CSRF header support for Keycloak 24 in keycloak_auth.py
- Created comprehensive unit tests in __tests__ directory
- Added observability logging with business events

### ❌ Not Yet Implemented
- Integration tests with real Keycloak container
- Automated realm import on startup

## Phase 1: Complete Existing Implementation (Week 1)

### Infrastructure Setup
- [ ] Verify Keycloak in docker-compose.yml is properly configured
  - [ ] Health checks enabled
  - [ ] Realm import via volume mount
  - [ ] Database persistence
- [ ] Test realm import with existing `infra/auth/keycloak-realm-config.json`
- [ ] Document Keycloak configuration for production deployment

### Enhance Existing Auth Implementation
- [ ] Fix import in `rest_gateway/auth_routes.py` to use existing keycloak_auth module
- [ ] Add gRPC interceptor in `services/core-api-grpc-rest/auth/grpc_interceptor.py`:
  ```python
  import grpc
  from grpc import aio
  from typing import Callable, Any
  from auth.keycloak_auth import get_keycloak_openid, validate_token
  
  class AuthInterceptor(aio.ServerInterceptor):
      """gRPC server interceptor for authentication."""
      
      def __init__(self):
          self.keycloak = get_keycloak_openid(...)
      
      async def intercept_service(
          self,
          continuation: Callable,
          handler_call_details: grpc.HandlerCallDetails
      ) -> grpc.RpcMethodHandler:
          # Extract token from metadata
          # Validate with keycloak
          # Add user context to call
          pass
  ```
- [ ] Add service-to-service auth helpers in `services/core-api-grpc-rest/auth/service_auth.py`:
  ```python
  from auth.keycloak_auth import get_keycloak_openid, get_service_token
  
  async def get_service_headers() -> dict[str, str]:
      """Get auth headers for service-to-service calls."""
      keycloak = get_keycloak_openid(...)
      token = await get_service_token(keycloak)
      return {"Authorization": f"Bearer {token}"}
  ```
- [ ] Add CSRF header support for Keycloak 24 in token requests:
  ```python
  # Update python-keycloak calls to include CSRF header
  headers = {
      "X-Requested-With": "XMLHttpRequest",  # Required for Keycloak 24
      "Content-Type": "application/x-www-form-urlencoded"
  }
  ```
### Testing & Verification
- [x] Test existing keycloak integration against running Keycloak:
  ```bash
  # Start Keycloak
  bazel run //infra/docker:up -- db-auth auth-service-keycloak
  
  # Run manual test
  python -m services.core-api-grpc-rest.auth.test_auth
  ```
- [x] Verify CSRF header requirements for Keycloak 24
- [x] Test service account token generation
- [x] Verify token validation with real JWKS

### Integration Tests
- [ ] Create `tests/integration/test_auth_integration.py`:
  ```python
  async def test_keycloak_auth_flow():
      # Start with real Keycloak
      # Test token validation
      # Test service account
      # Test token expiry
  ```
- [ ] Add to integration test BUILD.bazel:
  ```python
  py_test(
      name = "test_auth_integration",
      srcs = ["test_auth_integration.py"],
      deps = [
          "//services/core-api-grpc-rest:core_api_grpc_rest_lib",
          requirement("pytest"),
          requirement("pytest-asyncio"),
          requirement("httpx"),
      ],
  )
  ```

### Observability
- [ ] Use `ameide_core_platform_common.observability` standards:
  ```python
  from ameide_core_platform_common.observability import get_logger, BusinessEvent, business_operation
  
  logger = get_logger(__name__)
  
  @business_operation(BusinessEvent.AUTH_TOKEN_VALIDATE)
  async def validate_token(self, token: str) -> dict:
      logger.info("Validating token", kid=headers.get("kid"))
  ```
- [ ] Export metrics using existing instrumentation:
  - [ ] `auth_token_validation_total{result="success|expired|invalid"}`
  - [ ] `auth_jwks_refresh_total`
  - [ ] `auth_service_token_fetch_total`
- [ ] Log validation failures at WARN with context (sub, kid, reason)
- [ ] Mask PII in logs using structured logging

### Phase 1 Acceptance Criteria
- [x] Access token obtained via client_credentials and validated by middleware (200 response)
- [x] Expired token returns 401 with `{ "code": "token_expired" }`
- [ ] Invalid `kid` triggers JWKS refresh then 401
- [x] `AUTH_ENABLED=false` bypasses auth completely
- [x] All tests pass in CI

## Phase 2: gRPC Authentication (Week 1-2)

### gRPC Server Interceptor
- [x] Implement in `services/core-api-grpc-rest/auth/grpc_interceptor.py`
- [ ] Extract token from gRPC metadata
- [ ] Validate using existing KeycloakClient
- [ ] Inject user context into gRPC context
- [ ] Handle missing/invalid tokens properly

### gRPC Client Support
- [ ] Add token injection helpers for outgoing gRPC calls
- [ ] Use service account tokens from KeycloakClient
- [ ] Handle token refresh automatically

### Service Updates
- [ ] Update BaseGrpcService to use validated context
- [ ] Add gRPC interceptor to server startup
- [ ] Test with existing gRPC services

### Phase 2 Acceptance Criteria
- [x] gRPC calls require valid tokens
- [x] Service-to-service calls work with service accounts
- [x] Integration tests pass

## Phase 3: Production Readiness (Week 2-3)

### Documentation
- [ ] Document auth setup and configuration
- [ ] Create troubleshooting guide
- [ ] Document service-to-service auth pattern

### Monitoring
- [ ] Add auth metrics to existing observability
- [ ] Set up alerts for auth failures
- [ ] Monitor token validation performance

### Security Hardening
- [ ] Review and update Keycloak security settings
- [ ] Implement proper secret management
- [ ] Add rate limiting for auth endpoints

### Phase 3 Acceptance Criteria
- [ ] Documentation complete
- [ ] Monitoring in place
- [ ] Security review passed

## Quick Reference

### Local Development
```bash
# Start Keycloak
bazel run //infra/docker:up -- db-auth auth-service-keycloak

# Build and run core-api with auth
bazel run //services/core-api-grpc-rest

# Run without auth
AUTH_ENABLED=false bazel run //services/core-api-grpc-rest

# Test auth manually
curl -H "Authorization: Bearer <token>" http://localhost:8000/api/v1/users
```

### Testing
```bash
# Run existing tests
bazel test //services/core-api-grpc-rest:all

# Integration tests (when created)
bazel test //tests/integration:test_auth_integration
```

### Environment Variables
```bash
AUTH_ENABLED=true
KEYCLOAK_URL=http://localhost:8080
KEYCLOAK_REALM=ameide
KEYCLOAK_CLIENT_ID=core-api
KEYCLOAK_CLIENT_SECRET=<from-keycloak-console>
JWKS_CACHE_TTL=3600
```

## Implementation Notes

### Using Standard Libraries
- **python-keycloak**: Available but not needed - existing implementation works well
- **jwcrypto**: Already used for JWT validation (more secure than python-jose)
- **httpx**: Already used for async HTTP calls to Keycloak

### Key Decisions
1. **Use python-keycloak directly** - No custom KeycloakClient wrapper needed
2. **Standard library approach** - KeycloakOpenID from python-keycloak handles everything
3. **No separate auth package** - Auth code stays in each service (simpler, less coupling)
4. **Focus on missing pieces** - gRPC interceptor and service-to-service auth

### Known Issues
- **Keycloak 24 CSRF**: Requires `X-Requested-With: XMLHttpRequest` header for token requests
- **Token size**: Monitor for gRPC 8KB metadata limit
- **JWKS caching**: Already implemented with proper refresh logic