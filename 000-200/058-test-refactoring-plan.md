# 058: Test Suite Refactoring Plan

## Overview

This plan outlines the refactoring needed for `tests/e2e` and `tests/integration` to align with the new proto-based architecture and SDK enhancements.

## Current State Analysis

### E2E Tests
- `test_agent_pipeline.py` - Agent workflows testing
- `test_ipa_api_lifecycle.py` - IPA API lifecycle
- `test_ipa_lifecycle.py` - IPA operations
- `test_workflows_pipeline.py` - Workflow execution

### Integration Tests
- `test_proto_integration.py` - Proto serialization
- `test_services_integration.py` - Service communication
- `test_storage_integration.py` - Database operations
- Multiple IPA-specific tests

## Refactoring Tasks

### 1. Update Proto Integration Tests

**File**: `tests/integration/test_proto_integration.py`

```python
# Add new test cases
def test_bpmn_proto_serialization():
    """Test BPMN proto message serialization."""
    from ameide_core_proto.design.v1 import bpmn_pb2
    
    # Create BPMN document
    doc = bpmn_pb2.BPMNDocument(
        id="test-123",
        name="Test Process",
        content="<?xml version='1.0'?>..."
    )
    
    # Serialize and deserialize
    serialized = doc.SerializeToString()
    deserialized = bpmn_pb2.BPMNDocument()
    deserialized.ParseFromString(serialized)
    
    assert deserialized.id == doc.id
    assert deserialized.name == doc.name

def test_design_namespace_messages():
    """Test all design namespace proto messages."""
    # Test BPMN messages
    # Test future ArchiMate messages
    # Test transformation messages
```

### 2. Create BPMN Service Integration Tests

**New File**: `tests/integration/test_bpmn_service_integration.py`

```python
import pytest
from ameide_core_proto.design.v1 import bpmn_pb2, bpmn_pb2_grpc
import grpc

class TestBPMNServiceIntegration:
    """Integration tests for BPMN service."""
    
    @pytest.fixture
    def bpmn_stub(self):
        """Create gRPC stub for BPMN service."""
        channel = grpc.insecure_channel('localhost:50051')
        return bpmn_pb2_grpc.BPMNServiceStub(channel)
    
    def test_create_document(self, bpmn_stub):
        """Test creating BPMN document."""
        request = bpmn_pb2.CreateDocumentRequest(
            document=bpmn_pb2.BPMNDocument(
                name="Integration Test",
                content="<bpmn:definitions>...</bpmn:definitions>"
            )
        )
        response = bpmn_stub.CreateDocument(request)
        assert response.document.id
        assert response.document.name == "Integration Test"
    
    def test_list_documents(self, bpmn_stub):
        """Test listing BPMN documents."""
        request = bpmn_pb2.ListDocumentsRequest(
            page_size=10
        )
        response = bpmn_stub.ListDocuments(request)
        assert response.documents is not None
    
    def test_update_document(self, bpmn_stub):
        """Test updating BPMN document."""
        # Create document first
        # Update it
        # Verify changes
    
    def test_delete_document(self, bpmn_stub):
        """Test deleting BPMN document."""
        # Create document
        # Delete it
        # Verify it's gone
```

### 3. Create SDK Integration Tests

**New File**: `tests/integration/test_sdk_integration.py`

```python
import pytest
from unittest.mock import patch
import subprocess
import time

class TestSDKIntegration:
    """Integration tests for TypeScript SDK."""
    
    @pytest.fixture(scope="class")
    def services_running(self):
        """Ensure all services are running."""
        # Start services if needed
        subprocess.run(["docker-compose", "up", "-d"], check=True)
        time.sleep(5)  # Wait for services
        yield
        # Cleanup if needed
    
    def test_sdk_connectivity(self, services_running):
        """Test SDK can connect to services."""
        # Run TypeScript test via subprocess
        result = subprocess.run(
            ["pnpm", "test:integration"],
            cwd="packages/ameide_core-sdk-ts",
            capture_output=True,
            text=True
        )
        assert result.returncode == 0
    
    def test_sdk_auth_flows(self, services_running):
        """Test SDK authentication."""
        # Test API key auth
        # Test Bearer token auth
        # Test OAuth2 flow
    
    def test_sdk_bpmn_operations(self, services_running):
        """Test SDK BPMN operations."""
        # Create, read, update, delete via SDK
        # Test pagination
        # Test error handling
```

### 4. Update E2E Tests

**File**: `tests/e2e/test_bpmn_design_flow.py`

```python
import pytest
from tests.shared.test_helpers import create_test_client
from tests.shared.fixtures import sample_bpmn_content

class TestBPMNDesignFlow:
    """End-to-end tests for BPMN design workflows."""
    
    def test_complete_bpmn_lifecycle(self):
        """Test complete BPMN document lifecycle."""
        client = create_test_client()
        
        # 1. Create document
        doc = client.design.bpmn.create(
            name="E2E Test Process",
            content=sample_bpmn_content()
        )
        assert doc.id
        
        # 2. List documents
        docs = client.design.bpmn.list()
        assert any(d.id == doc.id for d in docs)
        
        # 3. Update document
        updated = client.design.bpmn.update(
            id=doc.id,
            content=sample_bpmn_content(version=2)
        )
        assert updated.version > doc.version
        
        # 4. Get specific document
        retrieved = client.design.bpmn.get(doc.id)
        assert retrieved.id == doc.id
        
        # 5. Delete document
        client.design.bpmn.delete(doc.id)
        
        # 6. Verify deletion
        with pytest.raises(NotFoundError):
            client.design.bpmn.get(doc.id)
```

### 5. Create UI E2E Tests

**New File**: `tests/e2e/test_ui_bpmn_integration.py`

```python
from playwright.sync_api import sync_playwright
import pytest

class TestUIBPMNIntegration:
    """E2E tests for UI BPMN integration."""
    
    @pytest.fixture
    def browser(self):
        """Setup browser for UI testing."""
        with sync_playwright() as p:
            browser = p.chromium.launch()
            yield browser
            browser.close()
    
    def test_create_diagram_via_ui(self, browser):
        """Test creating BPMN diagram through UI."""
        page = browser.new_page()
        
        # Navigate to app
        page.goto("http://localhost:3000")
        
        # Login if needed
        # page.fill("#username", "test@example.com")
        # page.fill("#password", "testpass")
        # page.click("#login-button")
        
        # Create new diagram
        page.click("button:has-text('New Diagram')")
        
        # Add BPMN elements
        page.drag_and_drop("#palette-start-event", "#canvas")
        page.drag_and_drop("#palette-task", "#canvas")
        page.drag_and_drop("#palette-end-event", "#canvas")
        
        # Save diagram
        page.fill("#diagram-name", "UI Test Diagram")
        page.click("button:has-text('Save')")
        
        # Verify save success
        assert page.locator(".success-message").is_visible()
        
        # Reload and verify persistence
        page.reload()
        assert page.locator("text=UI Test Diagram").is_visible()
```

### 6. Performance Test Suite

**New File**: `tests/performance/test_bpmn_performance.py`

```python
import pytest
import time
import concurrent.futures
from tests.shared.test_helpers import create_test_client

class TestBPMNPerformance:
    """Performance tests for BPMN operations."""
    
    def test_concurrent_operations(self):
        """Test concurrent BPMN operations."""
        client = create_test_client()
        
        def create_document(index):
            return client.design.bpmn.create(
                name=f"Perf Test {index}",
                content="<bpmn:definitions>...</bpmn:definitions>"
            )
        
        # Create 100 documents concurrently
        start_time = time.time()
        with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
            futures = [executor.submit(create_document, i) for i in range(100)]
            results = [f.result() for f in futures]
        
        elapsed = time.time() - start_time
        
        # Assert performance criteria
        assert elapsed < 30  # Should complete in 30 seconds
        assert len(results) == 100
        assert all(r.id for r in results)
    
    def test_large_document_handling(self):
        """Test handling of large BPMN documents."""
        client = create_test_client()
        
        # Create large BPMN content (1MB)
        large_content = generate_large_bpmn(size_mb=1)
        
        start_time = time.time()
        doc = client.design.bpmn.create(
            name="Large Document",
            content=large_content
        )
        create_time = time.time() - start_time
        
        # Retrieve large document
        start_time = time.time()
        retrieved = client.design.bpmn.get(doc.id)
        retrieve_time = time.time() - start_time
        
        # Assert performance criteria
        assert create_time < 2  # Should create in 2 seconds
        assert retrieve_time < 1  # Should retrieve in 1 second
        assert len(retrieved.content) == len(large_content)
```

## Implementation Timeline

### Phase 1: Foundation (Week 1)
1. Update `test_proto_integration.py` with BPMN tests
2. Create `test_bpmn_service_integration.py`
3. Set up SDK integration test framework

### Phase 2: E2E Tests (Week 2)
1. Create `test_bpmn_design_flow.py`
2. Implement UI integration tests with Playwright
3. Update existing workflows tests

### Phase 3: Performance & Security (Week 3)
1. Create performance test suite
2. Add security test cases
3. Set up continuous benchmarking

### Phase 4: CI/CD Integration (Week 4)
1. Update GitHub Actions workflows
2. Add test result reporting
3. Create test dashboards

## Test Infrastructure Updates

### 1. Docker Compose for Tests
```yaml
# tests/docker-compose.test.yml
version: '3.8'
services:
  postgres-test:
    image: postgres:15
    environment:
      POSTGRES_DB: ameide_test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    
  core-api-test:
    build: ../services/core-api-grpc-rest
    environment:
      DATABASE_URL: postgresql://test:test@postgres-test/ameide_test
      JWT_SECRET: test-secret
    depends_on:
      - postgres-test
    
  canvas-ui-test:
    build: ../services/www_ameide-portal-canvas
    environment:
      NEXT_PUBLIC_API_URL: http://core-api-test:8000
    depends_on:
      - core-api-test
```

### 2. Test Data Fixtures
```python
# tests/shared/fixtures.py
def sample_bpmn_content(version=1):
    """Generate sample BPMN content."""
    return f"""<?xml version="1.0" encoding="UTF-8"?>
    <bpmn:definitions xmlns:bpmn="http://www.omg.org/spec/BPMN/20100524/MODEL"
                      version="{version}">
      <bpmn:process id="Process_1" isExecutable="true">
        <bpmn:startEvent id="StartEvent_1"/>
        <bpmn:task id="Task_1" name="Sample Task"/>
        <bpmn:endEvent id="EndEvent_1"/>
      </bpmn:process>
    </bpmn:definitions>"""
```

### 3. Test Utilities
```python
# tests/shared/sdk_utils.py
import subprocess
import json

def run_sdk_test(test_name: str) -> dict:
    """Run TypeScript SDK test and return results."""
    result = subprocess.run(
        ["pnpm", "test", "--", test_name],
        cwd="packages/ameide_core-sdk-ts",
        capture_output=True,
        text=True
    )
    return {
        "success": result.returncode == 0,
        "output": result.stdout,
        "error": result.stderr
    }
```

## Success Criteria

1. **Coverage**: All new code has 80%+ test coverage
2. **Performance**: No regression in existing tests
3. **Reliability**: Tests are deterministic and repeatable
4. **Speed**: Full test suite runs in < 10 minutes
5. **Documentation**: All tests have clear documentation

## Next Steps

1. Create issue for each test file to be refactored
2. Set up test environment with new services
3. Begin with proto integration tests
4. Progressively update each test category
5. Add new test categories as needed