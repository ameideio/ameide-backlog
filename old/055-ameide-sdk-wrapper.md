# 055: Enhance SDK with Design Services and Namespace Organization ✅ COMPLETED

## Overview

The existing TypeScript SDK (`packages/ameide_core-sdk-ts`) already has a solid foundation with `AmeideClient`, auth providers, retry logic, and transport infrastructure. However, it only includes runtime services (workflows, agents, IPA, users). This item adds design-time services (BPMN, ArchiMate, WfSpec) and organizes them into clear namespaces.

## Goals

- Add design-time service clients to existing SDK
- Organize services into design/runtime namespaces in `AmeideClient`
- Generate TypeScript clients for new proto services
- Add convenience methods for design-runtime workflows
- Maintain existing SDK patterns and quality

## Current State

**Existing SDK has:**
- ✅ `AmeideClient` with runtime services (workflows, agents, IPA, users)
- ✅ OAuth2, API Key, and Bearer token auth providers
- ✅ Retry logic with configurable sleepers
- ✅ Transport layer with interceptors
- ✅ Error handling with `AmeideError` and error codes
- ✅ Pagination utilities
- ✅ Generated proto clients for IPA namespace services

**Missing:**
- ✅ BPMN service client can now be generated
- ❌ No ArchiMate service clients (proto not defined)
- ❌ No WfSpec service clients (removed - not a standard)
- ❌ No namespace organization (design vs runtime)
- ❌ No transformation service clients
- ❌ Limited convenience methods for design workflows

## Proposed Enhancement

### 1. Generate New Proto Clients

Add to buf.gen.yaml targets:
```yaml
# Generate clients for new services
- bpmn/v1/*  # ✅ Proto exists, ready to generate
- ea/v1/*    # ✅ Proto exists (ArchiMate services)
- transformations/v1/*  # ❌ Proto not yet defined
```

### 2. Enhance AmeideClient

```typescript
// packages/ameide_core-sdk-ts/src/client/AmeideClient.ts
export class AmeideClient {
  private transport: Transport;

  // Organize existing services under runtime namespace
  readonly runtime: {
    workflows: Client<typeof WorkflowService>;
    agents: Client<typeof AgentService>;
    ipas: Client<typeof IPAService>;
  };

  // Add design namespace with new services
  readonly design: {
    bpmn: Client<typeof BPMNService>;
    archimate: Client<typeof ArchiMateService>;
    // transformations: Client<typeof TransformationService>; // Not yet defined
  };

  // Keep users at top level (cross-cutting concern)
  readonly users: Client<typeof UserService>;

  constructor(config: AmeideConfig) {
    // ... existing auth and transport setup ...

    // Initialize runtime services (existing)
    this.runtime = {
      workflows: createClient(WorkflowService, this.transport),
      agents: createClient(AgentService, this.transport),
      ipas: createClient(IPAService, this.transport),
    };

    // Initialize design services (new)
    this.design = {
      bpmn: createClient(BPMNService, this.transport),
      archimate: createClient(ArchiMateService, this.transport),
      wfspec: createClient(WfSpecService, this.transport),
      transformations: createClient(TransformationService, this.transport),
    };

    // Keep users separate
    this.users = createClient(UserService, this.transport);
  }

  // Add design-to-runtime convenience methods
  async importAndDeployBPMN(file: File, options?: DeployOptions) {
    // Import BPMN
    const content = await file.text();
    const model = await this.design.bpmn.importBPMN({ 
      bpmnXml: content,
      name: file.name,
    });
    
    // Convert to workflows
    const result = await this.design.transformations.transformBPMNToWorkflow({
      modelId: model.id,
      options: options?.transformOptions,
    });
    
    // Deploy if requested
    if (options?.autoDeploy) {
      await this.runtime.workflows.publishWorkflow({
        id: result.workflowsId,
      });
    }
    
    return result;
  }

  // Maintain backward compatibility by keeping existing methods
  async executeWorkflow(workflowsId: string, inputs: Record<string, any>) {
    // In v2, google.protobuf.Struct fields are JsonObject — pass plain objects
    return this.runtime.workflows.executeWorkflow({
      workflowsId,
      inputs,
    });
  }
}
```

### 3. Update Service Exports

```typescript
// packages/ameide_core-sdk-ts/src/services/design/index.ts (new)
export * from '../../generated/ameide_core_proto/bpmn/v1/bpmn_types_pb';
export * from '../../generated/ameide_core_proto/bpmn/v1/bpmn_service_pb';
// ... other design services (likewise from *_types_pb / *_service_pb) ...

// Update main index.ts to include design services
export * from './services/design';
```

### 4. Add Design-Specific Types

```typescript
// packages/ameide_core-sdk-ts/src/client/types.ts (addition)
export interface DeployOptions {
  autoDeploy?: boolean;
  transformOptions?: Record<string, any>;
  validateFirst?: boolean;
}

export interface ImportOptions {
  validate?: boolean;
  overwrite?: boolean;
}
```

## Success Criteria

- [x] BPMN proto services generated
- [x] AmeideClient reorganized with design/runtime namespaces
- [x] New convenience methods added
- [x] Documentation updated
- [x] SDK tests pass with new structure
- [x] Model editor successfully uses new SDK
- [ ] ArchiMate proto services generated (blocked - service not implemented)
- [ ] Examples include design workflows (partial - BPMN only)
- [ ] Integration tests written

## Current Progress

- ✅ BPMN proto service fully defined and implemented
- ✅ Gateway service provides REST/gRPC access to BPMN
- ✅ SDK updated with BPMN service in design namespace
- ✅ TypeScript clients generated for BPMN
- ✅ Convenience methods added for BPMN operations
- ✅ Documentation updated with examples
- ⚠️ ArchiMate proto exists but service not implemented
- ❌ Transformation service proto not defined

## Dependencies

- #051 BPMN Proto Service defined ✅ DONE
- #052 WfSpec Proto Service ❌ CANCELLED (not a standard)
- #053 Transformation services ❌ NOT STARTED
- #054 Gateway service running ✅ DONE (core-api-grpc-rest)

## Estimated Effort

- Proto generation setup: 1 day ✅ DONE
- Client reorganization: 1 day ✅ DONE
- Convenience methods: 1 day ✅ DONE
- Testing and examples: 2 days ⚠️ PARTIAL (1 day done)
- Documentation: 1 day ✅ DONE
- **Total: 6 days** (5 days completed)

## Risks

- Generated code size increase
- Namespace confusion for existing users
- Proto generation conflicts
