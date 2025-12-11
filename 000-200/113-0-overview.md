# Artifact-Centric Graph Architecture - Overview

## Executive Summary

Unified graph model where all platform content becomes **Artifacts** with **typed relationships**. Enables cross-model queries, impact analysis, and natural navigation across BPMN processes, ArchiMate models, documents, and custom content types.

**Key Innovation**: Everything is a node in a graph-scoped graph, enabling queries like "What processes implement this capability?" or "Show all artifacts impacted by this change."

## Goals

- **Universal Graph Model**: All content as graph nodes with typed relationships
- **Semantic Relationships**: Rich, queryable relationships between artifacts
- **Flexible Composition**: Artifacts can contain other artifacts (hierarchical/network)
- **Cross-Model Traceability**: Natural links between BPMN, ArchiMate, documents, etc.

## Benefits

- **Graph Queries**: "Find all BPMN processes that implement this ArchiMate capability"
- **Impact Analysis**: "What artifacts depend on this component?"
- **Natural Navigation**: Browse from strategy → capability → process → implementation
- **Semantic Search**: Find by meaning, not just keywords
- **Artifact Reuse**: Same artifact can appear in multiple contexts
- **AI Context**: Graph traversal provides rich context for AI assistants

## Why Graph vs Document-Centric

| Aspect | Document-Centric | Artifact Graph |
|--------|-----------------|----------------|
| Structure | Hierarchical only | Network + hierarchical |
| Relationships | Parent-child | Any typed relationship |
| Cross-references | Manual links | Native graph edges |
| Impact analysis | Limited to tree | Full graph traversal |
| Model integration | Separate silos | Unified graph |
| Query capability | SQL/document queries | Graph pattern matching |
| Traceability | Manual maintenance | Automatic via edges |
| AI context | Limited to document | Rich neighborhood context |

## High-Level Architecture

```
Event Store (Source of Truth)
    ↓
Transactional Outbox
    ↓
Debezium CDC → Kafka
    ↓
GraphProjector Service
    ↓
Neo4j (Read-Only Projection)
    ↓
Query APIs (gRPC/GraphQL/REST)
```

## Key Design Principles

1. **Event Store owns truth** - Graph is disposable projection
2. **Repository boundary** - Security enforced at graph level
3. **Dual edge patterns** - Direct edges for simple, reified nodes for rich relations
4. **Kill switch ready** - Feature flag + rebuild command for safe rollback
5. **No graph writes** - Only projector writes to graph, all changes via commands

## Document Structure

- [Architecture & ADRs](113-1-architecture.md) - Core architecture decisions
- [Domain Model](113-2-domain-model.md) - Artifact kinds and relationships
- [Technical Implementation](113-3-technical-implementation.md) - Kafka, Neo4j, projections
- [API Design](113-4-api-design.md) - Query patterns and security
- [UI/UX Design](113-5-ui-ux-design.md) - User interface and workflows
- [Observability](113-6-observability.md) - Monitoring and operations
- [Implementation Guide](113-7-implementation-guide.md) - Step-by-step instructions
- [Appendices](113-8-appendices.md) - Integration patterns and examples
- [Governance & Operations](113-9-governance-operations.md) - Kind registry, compliance, lifecycle
- [Human-in-the-Loop ChangeSet System](113-10-changeset-hitl.md) - Review and approval workflows