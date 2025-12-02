# RPC Runtime Alignment Plan

**Status**: Draft  
**Owner**: Platform Architecture  
**Last Updated**: 2025-10-26

---

## 1. Problem Statement

Our services currently mix Connect-based servers/clients (Node/TypeScript, Go gateway) with plain gRPC servers/SDKs (Platform service, Go/Python clients). The split increases maintenance cost, confuses integrators, and complicates observability and tooling.

---

## 2. Goals

1. Pick and document a single primary RPC runtime (Connect *or* vanilla gRPC) for internal services.
2. Provide migration guides and timelines for services that need to switch stacks.
3. Align generated SDKs (TS/Go/Python) and transports with the chosen runtime.
4. Add CI and observability guardrails so new code follows the standard.

---

## 3. Proposed Phases

### Phase A – Decision & RFC (Week 1)
- Draft short RFC comparing Connect vs gRPC (performance, feature parity, tooling).
- Architecture review sign-off.
- Publish “runtime standard” doc with target dates.

### Phase B – Service Migration (Weeks 2-5)
- For each service, capture current stack and required changes.
- Update infrastructure manifests (Envoy routes, helm charts) if endpoints change.
- Add smoke tests hitting each service through the canonical runtime.

### Phase C – SDK & Client Updates (Weeks 4-6)
- Regenerate and publish TS/Go/Python SDKs with the standard runtime.
- Deprecate legacy client constructors; add migration notes.
- Update documentation/snippets across repos.

### Phase D – Guardrails (Weeks 5-7)
- CI checks to block mixed runtimes (e.g., lint for @grpc/grpc-js imports if Connect wins).
- Logging/metrics tags indicating runtime version for visibility.
- Roll up dashboard/alert changes.

---

## 4. Service Inventory (Initial Scan)

| Service / Package | Current Runtime | Notes |
|-------------------|-----------------|-------|
| `services/graph` | Connect (`@connectrpc/connect-node`) | Already serves gRPC & gRPC-Web via Connect. |
| `services/threads` | Connect (`@connectrpc/connect-node`) | Streams inference via Connect clients. |
| `services/inference_gateway` (Go) | Connect (connectrpc.com) | Wraps Python inference. |
| `services/platform` | Plain gRPC (`@grpc/grpc-js`) | Needs migration if Connect chosen. |
| `services/workflows` | REST/Express | Not directly affected. |
| `packages/ameide_sdk_ts` | Connect transports | Align once decision made. |
| `packages/ameide_sdk_go` | Plain gRPC (`google.golang.org/grpc`) | Requires adapters or regeneration. |
| `packages/ameide_sdk_python` | Plain gRPC (`grpcio`) | Requires regeneration. |

---

## 5. Open Questions

1. Do we need to support both Connect and gRPC long term (e.g., external clients)?
2. How do we handle Go/Python SDK consumers if Connect becomes mandatory?
3. What is the impact on Envoy routing rules and existing TLS/offload settings?
4. Can we deliver incremental migrations without downtime?

---

## 6. Next Actions

1. Draft runtime RFC (assignee: TBD) and schedule platform review. **Due:** 2025-11-01.
2. Produce detailed migration matrix for each service and SDK.
3. Set up tracking issue(s) per phase with owners and dates.
4. Update this backlog item as decisions and migrations land.

---

## 7. References

- `services/graph/src/server.ts`
- `services/platform/src/server.ts`
- `packages/ameide_sdk_ts/src/client/index.ts`
- `packages/ameide_sdk_go/client.go`
- `packages/ameide_sdk_python/src/ameide_sdk/options.py`
