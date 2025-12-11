# 051: BPMN Proto Service Implementation (v2) ✅ COMPLETED

## Overview

Create a focused BPMN storage and validation service. This service handles CRUD operations for BPMN 2.0 XML documents with validation capabilities. No transformations, no versioning, just reliable BPMN document management.

## Goals

- Store and retrieve BPMN 2.0 XML documents
- Validate BPMN against schema and business rules
- Provide search and listing capabilities
- Parse BPMN metadata on-demand for queries
- Support large files via streaming

## Current State

- ✅ BPMN ontology package with validation
- ✅ Proto infrastructure ready
- ✅ BPMN storage service implemented
- ✅ Full API for BPMN management (gRPC and REST)
- ✅ Streaming support for large files
- ✅ Validation integrated with ontology package

## Proposed Solution

### Proto Definition (Simplified)

```proto
service BPMNService {
  // Core CRUD
  rpc CreateBPMN(CreateBPMNRequest) returns (BPMNDocument);
  rpc GetBPMN(GetBPMNRequest) returns (BPMNDocument);
  rpc UpdateBPMN(UpdateBPMNRequest) returns (BPMNDocument);
  rpc DeleteBPMN(DeleteBPMNRequest) returns (google.protobuf.Empty);
  rpc ListBPMN(ListBPMNRequest) returns (ListBPMNResponse);
  
  // Validation
  rpc ValidateBPMN(ValidateBPMNRequest) returns (ValidateBPMNResponse);
  
  // Streaming for large files
  rpc UploadBPMN(stream UploadBPMNRequest) returns (BPMNDocument);
  rpc DownloadBPMN(DownloadBPMNRequest) returns (stream DownloadBPMNResponse);
}

message BPMNDocument {
  ameide.iomon.v1.ResourceMetadata metadata = 1;
  string name = 2;
  string description = 3;
  int64 size_bytes = 4;
  string checksum = 5;  // SHA-256 for deduplication
  
  // Extracted metadata (computed on-demand)
  BPMNMetadata extracted_metadata = 6;
}

message BPMNMetadata {
  repeated string process_ids = 1;
  repeated string process_names = 2;
  int32 element_count = 3;
  bool is_executable = 4;
}
```

### Storage Strategy

**PostgreSQL with dedicated table:**
- `id` (UUID) - Primary key
- `tenant_id` (string) - Tenant isolation
- `name` (string) - User-friendly name
- `description` (text) - Optional description
- `xml_content` (text) - BPMN XML content
- `checksum` (string) - SHA-256 hash
- `size_bytes` (bigint) - File size
- `metadata` (jsonb) - Extracted process info (cached)
- `created_at`, `updated_at` - Timestamps

**Constraints:**
- Unique index on (tenant_id, checksum) for deduplication
- Max file size: 50MB (configurable)
- GIN index on metadata for process search

### Error Handling

```python
class BPMNError(Exception):
    """Base BPMN error"""
    grpc_code = grpc.StatusCode.INTERNAL

class BPMNValidationError(BPMNError):
    """BPMN validation failed"""
    grpc_code = grpc.StatusCode.INVALID_ARGUMENT

class BPMNNotFoundError(BPMNError):
    """BPMN document not found"""
    grpc_code = grpc.StatusCode.NOT_FOUND

class BPMNSizeLimitError(BPMNError):
    """File too large"""
    grpc_code = grpc.StatusCode.RESOURCE_EXHAUSTED
```

### Service Operations

1. **Create**: Accept XML, validate, compute checksum, store
2. **Get**: Retrieve by ID, include cached metadata
3. **Update**: Replace XML, revalidate, update metadata
4. **Delete**: Soft delete with audit trail
5. **List**: Filter by name, process IDs, pagination
6. **Validate**: Parse and validate without storing

## Success Criteria

- [x] BPMN CRUD operations working
- [x] Validation using existing ontology package
- [x] Streaming for files >10MB
- [x] Deduplication via checksum
- [x] Search by process ID/name
- [ ] <100ms response for standard operations (needs performance testing)
- [ ] <500ms for validation (needs performance testing)

## Dependencies

- BPMN ontology package
- PostgreSQL database
- Proto infrastructure

## Estimated Effort

- Proto definition: 1 day ✅ DONE
- Domain model & graph: 2 days ✅ DONE
- Service implementation: 2 days ✅ DONE
- Streaming support: 1 day ✅ DONE
- Testing: 2 days ⚠️ PARTIAL (unit tests needed)
- **Total: 8 days** (7 days completed)

## Implementation Details

### What was built:

1. **Proto Definition** (`packages/ameide_core_proto/proto/ameide_core_proto/bpmn/v1/bpmn.proto`)
   - Full CRUD operations
   - Validation endpoint
   - Streaming upload/download

2. **Domain Model** (`packages/ameide_core-domain/src/ameide_core_domain/bpmn/`)
   - `BPMNDocument` with automatic checksum calculation
   - `BPMNMetadata` for extracted information
   - Full validation support

3. **Storage Layer** (`packages/ameide_core-storage/src/ameide_core_storage/sql/`)
   - `BPMNDocumentEntity` with proper indexes
   - `SqlBPMNRepository` with deduplication
   - PostgreSQL with GIN index for metadata search

4. **Service Implementation** (`services/bpmn-service/`)
   - Full gRPC service with all operations
   - Integrated validation using ontology package
   - Streaming support for large files
   - Proper error handling

5. **API Gateway** (`services/core-api-grpc-rest/`)
   - REST endpoints for all BPMN operations
   - File upload/download support
   - WebSocket notifications

### Remaining Work:

- Performance testing and optimization
- Comprehensive unit and integration tests
- Monitoring and metrics
- Documentation updates

## Out of Scope

- ❌ Workflow conversion
- ❌ Model versioning
- ❌ Model composition/imports
- ❌ Pattern detection
- ❌ Any transformation logic

These belong in separate, focused services.