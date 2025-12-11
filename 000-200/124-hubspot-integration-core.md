> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

# 124: HubSpot Integration Core - OAuth App & Webhook Infrastructure

## Status
**Status**: 🔴 Not Started  
**Priority**: P1 - Critical  
**Size**: L (3-5 sprints)  
**Type**: Feature  

## Context
Build the foundational HubSpot integration infrastructure including OAuth marketplace app, webhook processing, and multi-tenant event streaming following AMEIDE architecture patterns.

## Architecture Alignment

### Package Structure
- **`packages/int_hubspot/`** - Pure business logic for HubSpot integration
  - `/webhook/` - Webhook verification and event processing logic
  - `/oauth/` - OAuth flow and token management protocols
  - `/models/` - HubSpot data models and types
  - `/config/` - Integration configuration and mappings

### Service Structure  
- **`services/int_hubspot/`** - Runtime infrastructure
  - HTTP server for webhook endpoints
  - Keycloak integration for OAuth token storage
  - Kafka producer for event streaming
  - Health checks following `core-platform-common` patterns

## Requirements

### OAuth Integration with Keycloak
- [ ] HubSpot OAuth app registration and configuration
- [ ] Store OAuth tokens as Keycloak user attributes (encrypted)
- [ ] Multi-tenant token management via Keycloak realms/groups
- [ ] Service-to-service auth using Keycloak client credentials
- [ ] Token refresh mechanism with automatic retry

### Webhook Infrastructure
- [ ] Webhook endpoint at `/webhooks` with X-HubSpot-Signature-V3 verification
- [ ] Pure verification logic in `packages/int_hubspot/webhook/`
- [ ] Event normalization to OCEL format in package layer
- [ ] Kafka publishing to `hubspot.events` topic
- [ ] Dead Letter Queue (DLQ) for poison events
- [ ] Backpressure handling (ack fast, process async)

### Multi-Tenancy
- [ ] Tenant isolation using HubSpot portalId
- [ ] Per-tenant Kafka partitions
- [ ] Row-level security in graph/event stores
- [ ] Per-tenant rate limiting with token buckets
- [ ] Keycloak-based tenant management

### Kubernetes Deployment
- [ ] Helm chart: `infra/kubernetes/charts/platform/int-hubspot/`
- [ ] HTTPRoute: `hubspot.dev.ameide.io` → `services/int_hubspot:8080`
- [ ] Standard health checks (`/health`, `/ready`)
- [ ] Observability (OpenTelemetry metrics, structured logging)
- [ ] SLOs: p95 webhook-to-kafka < 60s, zero data loss

## Technical Decisions
- **Package-Service Separation**: Business logic in `int_hubspot` package, runtime in service
- **Auth**: Keycloak for all OAuth token storage (no separate PostgreSQL storage)
- **Events**: Existing Kafka cluster (`kafka-kafka-bootstrap:9092`)
- **Gateway**: Envoy Gateway HTTPRoute for webhook ingress
- **Storage**: Event store in `postgres-event-rw`, projections in `postgres-platform-rw`

## Dependencies
- Keycloak (`keycloak:443`) - OAuth token storage
- Kafka (Strimzi-managed) - Event streaming
- PostgreSQL clusters - Event and platform databases
- Neo4j (`neo4j:7687`) - Graph storage
- Envoy Gateway - Webhook ingress

## Acceptance Criteria
- [ ] OAuth flow integrates with Keycloak identity management
- [ ] Webhooks are verified using pure package logic
- [ ] Events flow through Kafka with proper partitioning
- [ ] Multi-tenant isolation verified with test tenants
- [ ] Rate limiting prevents 429 errors
- [ ] Grafana dashboard shows key metrics
- [ ] Uninstall cleans up Keycloak tokens and tenant data

## References
- HubSpot API Documentation
- AMEIDE Package-Service Architecture (`packages/README.md`)
- Keycloak Integration Pattern (`services/www_ameide_platform/README.md`)
- Ameide-HubSpot-Blueprint/02_System-Architecture-and-Deployment.md
