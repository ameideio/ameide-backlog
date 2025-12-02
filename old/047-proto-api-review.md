# 047 – Critical Review of Backlog Items 043-046

Created: 2025-07-25  
Author: platform-architecture  
Status: **DRAFT**

This document consolidates the peer-review feedback for the following backlog items:

* 043 – IPA Architecture: Core Domain Model to Runtime Deployment
* 044 – Proto-Based APIs Implementation
* 045 – TypeScript SDK Generation
* 046 – Portal Backend Integration

Each section lists strengths, key concerns and actionable next steps.  The goal is to capture review outcomes in the product backlog so that follow-up work can be planned and tracked.

---

## 1. 043 – IPA Architecture

### Strengths
• Layered design removes file-based bias.  
• Mixed-runtime deployment acknowledged.  
• Comprehensive migration plan with passing tests.

### Key Concerns
1. **Timeline inconsistency** – creation/updated dates are reversed.  
2. **Cross-runtime orchestration** – no description of data sharing, compensation or correlation IDs.  
3. **Domain events missing** – but later backlogs depend on them.  
4. **Hot compilation load** – missing debounce/rate-limit strategy.  
5. **Versioning duplication** – `version` field vs DB column.  
6. **Tenant isolation** – component references lack tenant context.  
7. **Builder becoming God-object** – plug-in strategy required.

### Action Items
* Fix date metadata.  
* Move domain-event design into Phase 1.  
* Add “Cross-Runtime Data Contract” section.  
* Introduce `BuilderStrategy` plug-in registry.  
* Decide on single source of truth for versioning.  
* Embed `tenant_id` in component references.

---

## 2. 044 – Proto-Based APIs

### Strengths
• Clean-architecture layering with shared ErrorMapper.  
• Explicit non-functional budgets.  
• Detailed service directory structures.

### Key Concerns
1. **Scope creep** – document covers multiple programmes; split into MVP slices.  
2. **REST generation strategy open** – choose grpc-gateway or alternative now.  
3. **Streaming scalability** – ingress/back-pressure not addressed.  
4. **Error mapping incomplete** – unauthenticated/deadline etc. missing.  
5. **Saga pattern unspecified** – UnitOfWork shown but orchestration undecided.  
6. **Proto versioning rules** – Buf breaking-change policy not formalised.

### Action Items
* Decompose backlog into smaller deliverables.  
* Finalise gateway tooling choice.  
* Extend ErrorMapper coverage to 100 %.  
* Document ingress proxy limits for streams.  
* Define proto versioning & directory scheme (e.g., `api/<svc>/v<N>`).

---

## 3. 045 – TypeScript SDK Generation

### Strengths
• Uses mainstream generators.  
• Strong focus on developer experience.  
• Clear graph layout.

### Key Concerns
1. **Generator undecided** – ts-proto vs protobuf-ts implies different runtimes.  
2. **Module/bundle format** – need dual ESM/CJS builds and typings.  
3. **SDK versioning** – relation to proto versions unspecified.  
4. **Authentication** – only bearer token, no refresh-token support.  
5. **Streaming support** – browser vs Node details unaddressed.

### Action Items
* Select generator (recommend ts-proto) and document pros/cons.  
* Add publishing & CI strategy (npm, provenance).  
* Introduce `TokenProvider` interface for pluggable auth.  
* Specify streaming abstraction per environment.

---

## 4. 046 – Portal Backend Integration

### Strengths
• Code snippets for both server and client components.  
• Real-time updates via SSE.  
• NextAuth flow with token refresh considered.

### Key Concerns
1. **Data-fetch/cache layer undefined** – risk of duplication and hydration bugs.  
2. **SSE vs WebSocket decision** – SSE is one-way; may need bidirectional transport.  
3. **Token exposure** – clarify scope/audience, prevent refresh-token leakage.  
4. **File upload/download** – no strategy (direct S3 vs proxy).  
5. **SSR hydration issues** – mixing RSC, context, Zustand.

### Action Items
* Choose TanStack Query (or similar) for unified caching.  
* Abstract `RealtimeClient` behind transport interface.  
* Document token-handling rules (no refresh token in browser).  
* Add signed-URL upload flow.  
* Provide SSR hydration guideline.

---

## 5. Cross-Backlog Dependencies

1. Domain events (043) → API streaming (044) → SDK event typings (045) → Portal live updates (046).  
2. Stable proto contracts (044) must precede SDK generation (045) which precedes Portal integration (046).

---

## 6. Next Steps & Ownership

| Item | Owner | Due |
|------|-------|-----|
| Fix metadata + domain events in 043 | Core Domain Team | **T+3 days** |
| Split 044 into MVP epics, finalise gateway tool | API Team | **T+5 days** |
| Generator decision & CI pipeline for SDK | Developer-Experience | **T+7 days** |
| Portal data-layer & realtime transport decision | Frontend Team | **T+10 days** |

Once these blockers are addressed, implementation phases can proceed in the following order: 043 → 044 → 045 → 046.

