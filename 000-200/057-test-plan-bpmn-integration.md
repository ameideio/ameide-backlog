# 057: BPMN Integration Test Plan

## Overview

This document outlines a comprehensive test plan for verifying the end-to-end BPMN integration, including the SDK enhancements, UI updates, and proto-based architecture.

## Test Environments

1. **Local Development**
   - Run all services locally using docker-compose
   - Use test databases and mock data
   
2. **Integration Environment**
   - Deploy to test Kubernetes cluster
   - Use dedicated test database instances
   
3. **Live System**
   - Production-like environment
   - Real database connections
   - Full monitoring enabled

## Test Categories

### 1. Unit Tests ✅ COMPLETED
- **SDK Tests** (`packages/ameide_core-sdk-ts`)
  - [x] Auth providers
  - [x] Transport layer
  - [x] Client initialization
  - [x] Design namespace structure
  - [x] Convenience methods

### 2. Integration Tests 🔄 TO BE REFACTORED

#### A. Proto Integration (`tests/integration/test_proto_integration.py`)
- [ ] Update to test BPMN proto messages
- [ ] Verify serialization/deserialization
- [ ] Test gRPC communication
- [ ] Validate REST gateway translation

#### B. Service Integration (`tests/integration/test_services_integration.py`)
- [ ] Test BPMN service CRUD operations
- [ ] Verify database persistence
- [ ] Test concurrent access
- [ ] Validate error handling

#### C. SDK Integration (`tests/integration/sdk/`)
- [ ] Create new test suite for TypeScript SDK
- [ ] Test against running services
- [ ] Verify auth flows
- [ ] Test retry mechanisms

### 3. End-to-End Tests 🔄 TO BE CREATED

#### A. BPMN Design Flow (`tests/e2e/test_bpmn_design_flow.py`)
```python
def test_bpmn_design_flow():
    """Test complete BPMN design workflows."""
    # 1. Create BPMN document via SDK
    # 2. Update document content
    # 3. List and search documents
    # 4. Delete document
    # 5. Verify audit trail
```

#### B. UI Integration (`tests/e2e/test_ui_integration.ts`)
```typescript
describe('BPMN UI Integration', () => {
  it('should create and edit BPMN diagram', async () => {
    // 1. Launch UI
    // 2. Create new diagram
    // 3. Add BPMN elements
    // 4. Save to backend
    // 5. Reload and verify
  });
});
```

#### C. Design to Runtime (`tests/e2e/test_design_to_runtime.py`)
```python
def test_design_to_runtime_transformation():
    """Test BPMN to workflows transformation."""
    # 1. Create BPMN diagram
    # 2. Transform to workflows (when available)
    # 3. Deploy workflows
    # 4. Execute workflows
    # 5. Monitor execution
```

### 4. Performance Tests 🔄 TO BE CREATED

#### A. Load Testing (`tests/performance/load_test.py`)
- [ ] Concurrent BPMN operations
- [ ] Large diagram handling
- [ ] API rate limiting
- [ ] Database connection pooling

#### B. Stress Testing (`tests/performance/stress_test.py`)
- [ ] Maximum document size
- [ ] Concurrent user limits
- [ ] Memory usage under load
- [ ] Recovery from failures

### 5. Security Tests 🔄 TO BE CREATED

#### A. Authentication (`tests/security/test_auth.py`)
- [ ] JWT token validation
- [ ] API key authentication
- [ ] OAuth2 flows
- [ ] Permission boundaries

#### B. Authorization (`tests/security/test_authz.py`)
- [ ] Document access control
- [ ] User role validation
- [ ] Cross-tenant isolation
- [ ] API endpoint security

## Live System Verification Steps

### 1. Service Health Checks
```bash
# Check all services are running
curl http://localhost:8000/health  # Gateway
curl http://localhost:3000/api/health  # UI

# Verify gRPC services
grpcurl -plaintext localhost:50051 grpc.health.v1.Health/Check
```

### 2. BPMN Service Verification
```bash
# Create test document
curl -X POST http://localhost:8000/v1/design/bpmn/documents \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Process",
    "description": "Integration test",
    "content": "<?xml version=\"1.0\"?><bpmn:definitions>...</bpmn:definitions>"
  }'

# List documents
curl http://localhost:8000/v1/design/bpmn/documents \
  -H "Authorization: Bearer $TOKEN"
```

### 3. SDK Verification
```typescript
// Test SDK connectivity
const client = new AmeideClient({
  baseUrl: 'http://localhost:8000',
  auth: { type: 'bearer', token: process.env.TEST_TOKEN }
});

// Test BPMN operations
const documents = await client.design.bpmn.list();
console.log('Documents:', documents);
```

### 4. UI Verification
1. Open http://localhost:3000
2. Login with test credentials
3. Create new BPMN diagram
4. Save and reload
5. Verify data persistence

## Test Data Management

### 1. Test Fixtures
- Sample BPMN diagrams in `tests/fixtures/bpmn/`
- User credentials in `tests/fixtures/auth/`
- Mock data generators in `tests/fixtures/generators/`

### 2. Database Seeding
```python
# tests/fixtures/seed_db.py
def seed_test_data():
    """Seed database with test data."""
    # Create test users
    # Create sample documents
    # Set up permissions
```

### 3. Cleanup
```python
# tests/fixtures/cleanup.py
def cleanup_test_data():
    """Remove all test data."""
    # Delete test documents
    # Remove test users
    # Clear caches
```

## Continuous Integration

### 1. GitHub Actions Workflow
```yaml
# .github/workflows/bpmn-integration-tests.yml
name: BPMN Integration Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Start services
        run: docker-compose up -d
      - name: Run integration tests
        run: |
          pytest tests/integration/
          pnpm test:e2e
      - name: Upload test results
        uses: actions/upload-artifact@v3
```

### 2. Test Reports
- JUnit XML for Python tests
- Jest coverage for TypeScript
- Performance metrics
- Security scan results

## Rollback Plan

If issues are discovered:
1. Revert SDK to previous version
2. Restore UI to use old API client
3. Keep BPMN service running (backward compatible)
4. Document issues for resolution

## Success Metrics

- [ ] All unit tests pass
- [ ] Integration tests achieve 80%+ coverage
- [ ] E2E tests complete in < 5 minutes
- [ ] No performance regressions
- [ ] Zero security vulnerabilities
- [ ] UI functions correctly in all browsers

## Next Steps

1. **Immediate** (Today)
   - Run manual verification of live system
   - Update existing integration tests
   
2. **Short Term** (This Week)
   - Create E2E test suite
   - Add performance benchmarks
   
3. **Long Term** (This Month)
   - Implement security testing
   - Set up monitoring dashboards
   - Create chaos testing scenarios