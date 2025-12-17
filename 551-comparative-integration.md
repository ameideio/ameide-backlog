# 551 Comparative Integration Primitive Analysis

**Status:** Reviewed 2025-12-17 (revalidated vs repo; corrected proto LOC and MCP adapter details)
**Type:** Cross-cutting Architecture Analysis
**Scope:** Sales, Commerce, Transformation, SRE integration primitives

## Executive Summary

Comparative analysis of integration primitive implementation maturity across four domains reveals significant variance in proto contracts, test coverage, security, and deployment readiness. No single domain is production-complete; each has complementary strengths and critical gaps.

## Maturity Overview

| Domain | Overall Score | Proto | Implementation | Tests | GitOps | Security |
|--------|--------------|-------|----------------|-------|--------|----------|
| **Commerce** | ★★★½ (3.5/5) | ✅✅ 154 LOC | ⚠️ Stubbed handlers (shape + validation) | ⚠️ Stub | ❌ None | ✅✅ TLS/mTLS |
| **Sales** | ★★½ (2.5/5) | ✅ 61 LOC | ✅ Business logic (2 actions + echo fallback) | ✅ PASS (basic) | ✅ CRD | ⚠️ Insecure |
| **Transformation** | ★★½ (2.5/5) | ❌ None | ⚠️ MCP server scaffold (no tools/resources) | ❌ Stub | ❌ None | ✅ Token |
| **SRE** | ★★ (2/5) | ❌ Generic | ⚠️ Echo only | ✅✅ 245 LOC | ❌ None | ✅ Strict TLS |

## Domain-Specific Findings

### Commerce Integration (`primitives/integration/commerce/`)

**Strengths:**
- **Best proto contracts**: 154 LOC across 3 files (`commerce_payments_integration.proto`, `commerce_hardware_integration.proto`, `commerce_replication.proto`)
  - Payment intent lifecycle: 9 states (pending, requires_action, processing, succeeded, failed, etc.)
  - Money type with currency handling (units, nanos)
  - Complete flow: Authorize → Capture → Refund
  - Hardware station integration (PingHardwareStation RPC)
- **Production security**: Optional TLS/mTLS via `GRPC_TLS_CERT_FILE`, `GRPC_TLS_KEY_FILE`, `GRPC_TLS_CLIENT_CA_FILE`
- **Complete handlers**: `CommercePaymentsIntegrationService` (Authorize, Capture, Refund), `CommerceHardwareIntegrationService`
- **Validation**: Comprehensive input validation, UUID-based provider IDs, idempotency key enforcement

**Gaps:**
- ❌ No GitOps deployment manifest (sales has one, commerce doesn't)
- ❌ Tests stub-only (6 LOC empty smoke test at `tests/smoke_test.go`)
- ⚠️ `tests/run_integration_tests.sh` exists but currently just runs `go test ./...` (no external-system integration assertions)
- README states "stub implementation to be replaced"

**Critical Paths:**
- Proto: `packages/ameide_core_proto/src/ameide_core_proto/commerce/integration/v1/`
- Handlers: `primitives/integration/commerce/internal/handlers/handlers.go`
- Main: `primitives/integration/commerce/cmd/main.go`

### Sales Integration (`primitives/integration/sales/`)

**Strengths:**
- **Well-structured EDA schema**: 61 LOC `integration_facts.proto`
  - Confluent schema registry conventions
  - Event types: ErpOrderCreated, QuoteEmailSent, ESignatureEnvelopeSent
  - Meta tracking (tenant_id, message_id), idempotency patterns (partition_key_fields, idempotency_key_fields)
- **Business logic implemented**: `sales.erp.create_order.v1`, `sales.quote.send_email.v1`
  - SHA256-based deterministic order ID generation
  - Email message ID formatting
- **GitOps ready**: Custom Integration CRD at `gitops/ameide-gitops/sources/values/_shared/apps/integration-sales-v0.yaml`
- **Good docs**: Development checklist (6 items), EDA/idempotency guidance (references 496)

**Gaps:**
- ⚠️ Tests are basic (20 LOC) but PASS at `internal/tests/execute_test.go`; README still says tests are RED
- ❌ No TLS security (insecure gRPC default)
- ❌ Single Dockerfile (no dev/release variants)
- README states "intentionally fails until real business logic implemented"
- ❌ Integration facts emission not wired (adapter returns outputs but doesn't publish facts)

**Critical Paths:**
- Proto: `packages/ameide_core_proto/src/ameide_core_proto/sales/integration/v1/integration_facts.proto`
- Handlers: `primitives/integration/sales/internal/handlers/handlers.go`
- GitOps: `gitops/ameide-gitops/sources/values/_shared/apps/integration-sales-v0.yaml`

### Transformation Integration (`primitives/integration/transformation-mcp-adapter/`)

**Strengths:**
- **Modern MCP architecture**: Official MCP Go SDK (`github.com/modelcontextprotocol/go-sdk`)
- **Centralized security**: Uses `packages/ameide_mcp` for transport/security
- **HTTP-based**: Port 8080 vs traditional gRPC 50051
- **Flexible auth**: Token verification (dev + keycloak), CORS origin policy, OAuth2, PRM integration
- Factory via `amcp.NewHTTPMux()` creating a `go-sdk` `mcp.Server` per request

**Gaps:**
- ❌ No domain-specific proto definitions
- ❌ No tools/resources implemented yet (server advertises `HasTools`/`HasResources` but handlers are placeholders)
- ❌ Tests stub-only (6 LOC empty smoke test at `tests/smoke_test.go`)
- ❌ No catalog-info.yaml (Backstage missing)
- ❌ No GitOps deployment manifest
- ❌ No README

**Architectural Note:**
Different paradigm from gRPC integrations. Uses Model Context Protocol (MCP) for tool/resource exposure vs traditional RPC.

**Critical Paths:**
- Main: `primitives/integration/transformation-mcp-adapter/cmd/main.go`
- Server: `primitives/integration/transformation-mcp-adapter/internal/mcp/server.go`

### SRE Integration

#### Standard (`primitives/integration/sre/`)

**Strengths:**
- **Strict security**: Requires TLS certs OR explicit `GRPC_INSECURE=true` flag via `serverCreds()` function
- **Passing tests**: Unit test `TestExecute_EchoesWithPrefix` at `internal/handlers/handlers_test.go` (19 LOC)

**Gaps:**
- ❌ **Most minimal implementation**: Single echo handler prefixing with "sre:"
- ❌ Explicitly marked scaffold/placeholder
- ❌ No domain-specific proto contracts (uses generic `IntegrationV0ServiceServer`)
- ❌ No integration test infrastructure (no `run_integration_tests.sh`)
- ❌ No GitOps deployment manifest
- ❌ No business logic

#### MCP Adapter (`primitives/integration/sre-mcp-adapter/`) ⭐ **BEST TEST COVERAGE**

**Strengths:**
- **⭐ Comprehensive tests**: 245+ LOC across 4 files (`auth_test.go`, `http_test.go`, `origin_test.go`, `oauth_test.go`)
  - Security: Scope enforcement (`TestHTTPHandler_ToolsCall_RequiresScopeInProd`), token validation, origin checking
  - Protocol: MCP tools/list, tools/call (`TestHTTPHandler_ToolsList_IncludesCapabilityTools`)
  - OAuth: Protected resource metadata (`TestProtectedResourceMetadataHandler`)
  - Error handling: Default deny pattern (`TestHTTPHandler_ToolsCall_DefaultDeny`)
  - Table-driven patterns for origin matching (`TestMatchOrigin`)
- **Production implementation**: Complete MCP server with hardcoded SRE tools (ping, searchIncidents, createIncident)
- **gRPC integration**: Direct clients to sre-domain and sre-projection services
- **Context propagation**: Identity from HTTP headers to gRPC context
- **Custom MCP protocol**: Full JSON-RPC implementation, stdio + HTTP transports

**Gaps:**
- ⚠️ Custom MCP implementation (not official SDK like transformation-mcp-adapter)
- ⚠️ Hardcoded tools in server.go (monolithic vs adapter pattern)
- ❌ No GitOps deployment manifest
- ❌ Missing catalog-info.yaml

**Critical Paths:**
- Server: `primitives/integration/sre-mcp-adapter/internal/mcp/server.go`
- Tests: `primitives/integration/sre-mcp-adapter/internal/mcp/*_test.go`

## Comparative Matrices

### Proto Contract Maturity

| Domain | Files | LOC | Event Schema | Service Contracts | Rating |
|--------|-------|-----|--------------|-------------------|--------|
| Commerce | 3 | 154 | Replication events | Payments + Hardware | ⭐⭐⭐⭐⭐ |
| Sales | 1 | 61 | EDA facts (3 types) | Generic integration | ⭐⭐⭐⭐ |
| Transformation | 0 | 0 | None | None | ⭐ |
| SRE | 0 | 0 | None | Generic only (shared `integration/v1`, 19 LOC) | ⭐ |

### Test Coverage Maturity

| Domain | Files | LOC | Coverage | Status |
|--------|-------|-----|----------|--------|
| SRE (MCP) | 4 | 245+ | Security, protocol, OAuth | ✅ PASS ⭐⭐⭐ |
| Sales | 1 | 20 | Handler echo + basic assertions | ✅ PASS |
| SRE (standard) | 1 | 19 | Handler echo | ✅ PASS |
| Commerce | 1 | 6 | None | ❌ STUB |
| Transformation | 1 | 6 | None | ❌ STUB |

### Security Implementation

| Domain | Approach | TLS | Auth | Origin Validation |
|--------|----------|-----|------|-------------------|
| Commerce | Optional TLS/mTLS | ✅ Configurable | N/A | N/A |
| SRE (standard) | Strict TLS | ✅ Required or explicit insecure | N/A | N/A |
| SRE (MCP) | Token + Origin | N/A | ✅ Bearer + scope | ✅ CORS |
| Transformation | Token + Origin | N/A | ✅ Dev + Keycloak | ✅ CORS |
| Sales | None | ❌ Insecure default | N/A | N/A |

### Deployment Readiness

| Domain | GitOps Manifest | CRD | Backstage | Status |
|--------|----------------|-----|-----------|--------|
| Sales | ✅ Yes | Custom Integration CRD | ✅ Yes | READY |
| Commerce | ❌ No | N/A | ✅ Yes | NOT READY |
| Transformation | ❌ No | N/A | ❌ No | NOT READY |
| SRE (both) | ❌ No | N/A | ⚠️ Standard only | NOT READY |

## Cross-Cutting Patterns

### Two Integration Paradigms

1. **gRPC-based**: Commerce, Sales, SRE (standard)
   - Traditional RPC with proto contracts
   - Direct service-to-service communication
   - TLS for security

2. **MCP-based**: Transformation, SRE (MCP adapter)
   - Model Context Protocol for tool/resource exposure
   - HTTP-based with token authentication
   - CORS/origin validation

### MCP Adapter Evolution

| Generation | Example | SDK | Auth | Tools |
|------------|---------|-----|------|-------|
| **First** | SRE-MCP | Custom protocol | Custom bearer | Hardcoded in server.go |
| **Second** | Transformation-MCP | Official SDK | `ameide_mcp` wrapper | Adapter layer (empty) |
| **Template** | CLI scaffold | Custom protocol | Custom bearer | Scaffold with adapter/ |

**Gap**: CLI template still generates first-generation pattern, not second-generation SDK-based approach.

## Critical Gaps Summary

### 1. Test Coverage Crisis (3 of 4 domains)
- Only SRE-MCP has comprehensive tests (245+ LOC)
- Commerce, Transformation: 6 LOC empty stubs
- Sales: 20 LOC basic unit test (PASS)
- “Integration test” runner scripts exist in some primitives, but currently just run unit tests (`go test ./...`) and do not exercise external systems

### 2. GitOps Deployment (3 of 4 domains)
- Only Sales has deployment manifest
- Commerce, Transformation, SRE cannot be deployed to cluster

### 3. Proto Contract Gaps (2 of 4 domains)
- Transformation and SRE: No domain-specific contracts
- Limits typed API exposure and code generation

### 4. Security Inconsistency
- Sales: Insecure gRPC by default (weakest)
- No standard security policy across domains
- MCP adapters have strong auth but gRPC variants inconsistent

### 5. CI/CD Integration (ALL domains)
- Integration primitives **NOT** in `.github/workflows/ci-integration-packs.yml`
- No automated testing in PR workflows
- Test infrastructure exists (`run_integration_tests.sh`) but unused

### 6. Dockerfile Variants (ALL domains)
- None implement separate dev/release Dockerfiles
- Single Dockerfile pattern everywhere

## Recommendations

### P0 (Immediate)

1. **Add GitOps manifests** for Commerce, Transformation, SRE
   - Template: `gitops/ameide-gitops/sources/values/_shared/apps/integration-sales-v0.yaml`
   - Enables cluster deployment

2. **Implement tests** for Commerce, Transformation
   - Reference: SRE-MCP test structure (`primitives/integration/sre-mcp-adapter/internal/mcp/*_test.go`)
   - Focus: Security (auth, origin), protocol compliance

3. **Define proto contracts** for Transformation, SRE
   - Transformation: Tool/resource schemas
   - SRE: Incident management, metrics, alerts

4. **Fix Sales security**
   - Add TLS support or strict insecure flag (match SRE pattern)
   - Never default to insecure in production

### P1 (Short-term)

5. **Update MCP scaffold template**
   - Generate second-generation pattern (SDK-based)
   - Match transformation-mcp-adapter architecture
   - Path: `packages/ameide_core_cli/internal/commands/templates/integration/mcp_adapter/`

6. **Add primitives to CI/CD**
   - Update `ci-integration-packs.yml` to include primitive test paths
   - Pattern: `primitives/integration/**/__tests__/integration/**`

7. **Complete Sales tests**
   - Move from RED to GREEN
   - Implement EDA/idempotency verification (see backlog 496)

8. **Standardize Dockerfile approach**
   - Decide: dev/release split vs single Dockerfile
   - Apply consistently

### P2 (Long-term)

9. **Consolidate MCP patterns**
   - Migrate SRE-MCP to SDK-based approach
   - Establish proto-first tool generation
   - Remove custom JSON-RPC implementations

10. **Create MCP adapters** for Sales, Commerce
    - Enable tool-based access patterns
    - Complement gRPC integrations

11. **Security baseline**
    - Enforce TLS by default
    - Standardize authentication (token + scope)
    - Require origin validation for HTTP

12. **Integration test infrastructure**
    - Implement actual tests in `run_integration_tests.sh`
    - Mock + cluster modes
    - JUnit XML reports for CI/CD

## File Reference Index

### Proto Definitions
- **Commerce**: `packages/ameide_core_proto/src/ameide_core_proto/commerce/integration/v1/`
  - `commerce_payments_integration.proto` (80 LOC)
  - `commerce_hardware_integration.proto` (31 LOC)
  - `commerce_replication.proto` (43 LOC)
- **Sales**: `packages/ameide_core_proto/src/ameide_core_proto/sales/integration/v1/`
  - `integration_facts.proto` (61 LOC)
- **Transformation**: None
- **SRE**: None (uses generic `integration/v1` only, 19 LOC)

### Integration Primitives
- Commerce: `primitives/integration/commerce/`
- Sales: `primitives/integration/sales/`
- Transformation (MCP): `primitives/integration/transformation-mcp-adapter/`
- SRE (standard): `primitives/integration/sre/`
- SRE (MCP): `primitives/integration/sre-mcp-adapter/`

### GitOps Manifests
- Sales: `gitops/ameide-gitops/sources/values/_shared/apps/integration-sales-v0.yaml`
- Others: **Missing**

### Test Files
- Commerce: `primitives/integration/commerce/tests/smoke_test.go` (6 LOC)
- Sales: `primitives/integration/sales/internal/tests/execute_test.go` (20 LOC)
- Transformation: `primitives/integration/transformation-mcp-adapter/tests/smoke_test.go` (6 LOC)
- SRE (standard): `primitives/integration/sre/internal/handlers/handlers_test.go` (19 LOC)
- SRE (MCP): `primitives/integration/sre-mcp-adapter/internal/mcp/*_test.go` (245+ LOC)

### Infrastructure
- MCP framework: `packages/ameide_mcp/`
- Integration test runner: `tools/integration-runner/`
- CLI scaffold template: `packages/ameide_core_cli/internal/commands/templates/integration/mcp_adapter/`

### CI/CD
- `.github/workflows/ci-integration-packs.yml` (does NOT include primitives)
- `.github/workflows/ci-core-quality.yml` (packages/services only)

## Related Backlog Items

- **496**: EDA/Idempotency patterns (referenced by Sales integration docs)
- **540-sales-integration.md**: Sales integration component status
- **526-sre-integration.md**: SRE integration component (if exists)
- **534-mcp-protocol-adapter.md**: MCP adapter architecture
- **537-primitive-testing-discipline.md**: Testing standards

## Conclusion

**Mixed maturity model** — no domain fully production-complete:

- **Commerce**: Leads in proto contracts + implementation, lacks tests + deployment
- **Sales**: Best docs + GitOps readiness, weak security
- **Transformation**: Future architecture, minimal implementation
- **SRE (MCP)**: Strongest test foundation, minimal business logic

**Path forward**: Consolidate patterns, fill gaps systematically, establish consistent standards across all domains.

**Production readiness checklist** (per domain):
1. ✅ Comprehensive test coverage (like SRE-MCP)
2. ✅ GitOps deployment manifests (like Sales)
3. ✅ Security by default (like SRE standard or Commerce)
4. ✅ Complete proto contracts (like Commerce)
