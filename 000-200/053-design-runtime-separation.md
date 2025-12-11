# 053: Design-Runtime Proto Service Alignment

## Overview

Align the proto service definitions with the existing model transformation architecture. We already have a clear separation between design models (BPMN, WfDesc) and runtime implementations (Temporal, LangGraph, Camunda, Airflow) with transformation packages. This item focuses on exposing these capabilities through proto services.

## Goals

- Expose existing transformation packages via proto services
- Create proto definitions that align with existing model packages
- Maintain consistency with current architecture patterns
- Enable gRPC/REST access to transformation capabilities
- Preserve the existing separation of concerns

## Current State

- ✅ Model packages for workflows and agents
- ✅ Transformation packages (bpmn2model, model2temporal, model2langgraph, etc.)
- ✅ Clear separation between models and implementations
- ❌ No proto service definitions for transformations
- ❌ Transformations not exposed via API

## Existing Architecture

**Design/Model Layer:**
- `core-workflows-bpmn2model` - BPMN to workflows model
- `core-agent-wfdesc2model` - WfDesc to agent model
- `core-workflows-model` - Abstract workflows model
- `core-agent-model` - Abstract agent model

**Transformation Layer:**
- `core-workflows-model2temporal` - Workflow model to Temporal
- `core-workflows-model2camunda` - Workflow model to Camunda
- `core-agent-model2langgraph` - Agent model to LangGraph
- `core-agent-model2airflow` - Agent model to Airflow
- `core-agent-model2networkx` - Agent model to NetworkX

**Build Layer:**
- `core-workflows-build` - Workflow building utilities
- `core-agent-build` - Agent building utilities

## Proposed Solution

### Proto Service Definitions

1. **Workflow Transformation Service**
   ```proto
   service WorkflowTransformationService {
     // Design to Model
     rpc ImportBPMN(ImportBPMNRequest) returns (WorkflowModelResponse);
     
     // Model to Runtime
     rpc GenerateTemporalWorkflow(GenerateTemporalRequest) returns (TemporalArtifacts);
     rpc GenerateCamundaProcess(GenerateCamundaRequest) returns (CamundaArtifacts);
     
     // Model Operations
     rpc ValidateWorkflowModel(ValidateModelRequest) returns (ValidationResponse);
     rpc OptimizeWorkflowModel(OptimizeModelRequest) returns (WorkflowModelResponse);
   }
   ```

2. **Agent Transformation Service**
   ```proto
   service AgentTransformationService {
     // Design to Model
     rpc ImportWfDesc(ImportWfDescRequest) returns (AgentModelResponse);
     
     // Model to Runtime
     rpc GenerateLangGraphAgent(GenerateLangGraphRequest) returns (LangGraphArtifacts);
     rpc GenerateAirflowDAG(GenerateAirflowRequest) returns (AirflowArtifacts);
     rpc GenerateNetworkXGraph(GenerateNetworkXRequest) returns (NetworkXArtifacts);
     
     // Model Operations
     rpc ValidateAgentModel(ValidateModelRequest) returns (ValidationResponse);
     rpc AnalyzeAgentGraph(AnalyzeGraphRequest) returns (GraphAnalysis);
   }
   ```

### Service Implementation Strategy

1. **Wrapper Services**
   - Create service wrappers around existing transformation packages
   - Maintain existing package interfaces
   - Add proto adapters for input/output

2. **Artifact Management**
   - Return generated code/configs as proto messages
   - Support artifact storage and retrieval
   - Version tracking for transformations

3. **Validation Integration**
   - Expose existing validation logic
   - Add transformation-specific validations
   - Return detailed error messages

## Success Criteria

- [ ] Proto services defined for all transformation packages
- [ ] Existing transformations accessible via gRPC/REST
- [ ] No changes to core transformation logic
- [ ] Full test coverage for service layer
- [ ] Documentation of transformation options
- [ ] Performance benchmarks established

## Dependencies

- Existing transformation packages
- #051 BPMN Proto Service
- #052 WfSpec Proto Service
- Proto infrastructure

## Estimated Effort

- Proto definitions: 2 days
- Service wrappers: 3 days
- Proto adapters: 2 days
- Testing: 2 days
- Documentation: 1 day
- **Total: 10 days**

## Risks

- Performance overhead of proto serialization
- Large artifact sizes in responses
- Maintaining backward compatibility
- Complex error handling across layers