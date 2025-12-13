# 523 Commerce — Process component

## Intent

Define the deterministic workflows (Temporal) that make commerce operable at fleet scale:

- storefront BYOD domain onboarding (claim/verify/issue/attach),
- store-site provisioning and rollouts (scale unit),
- replication control (downsync/upsync, backfills, recovery),
- explicit handling of real-time required operations.

Process protos define workflow envelopes and process facts; operators remain operational only (`520-primitives-stack-v2.md`).

For the business-process view (value streams and “golden paths”), see `523-commerce-process-catalog.md`.

## Stack alignment (proto / generation / operator)

- Proto: workflow envelopes + process facts in `ameide_core_proto.process.commerce.v1`.
- Generation: Temporal worker wiring + determinism/versioning guardrails and tests.
- Operator: reconciles workers/HPAs/ingress and surfaces Temporal dependency status via conditions.

## Real-time required surface (v1)

Default is async replication. The system MUST label and handle “real-time required” operations explicitly:

- gift cards (if external)
- loyalty redemption/earn (if external)
- customer create/update (if centrally governed)
- cross-store returns/exchanges
- inventory adjustments that affect other sites
- fraud checks / risk holds (provider-dependent)

## Core workflows (sketch)

### A) Storefront custom domains

- `VerifyDomainClaim` (DNS TXT verification)
- `ProvisionDomainMapping` (cert issuance + gateway wiring + route)
- `RevokeDomainMapping` (day-2 cleanup)

Failure taxonomy (status/conditions should expose these):

- DNS target incorrect (wrong CNAME/A record)
- TXT mismatch or missing
- DNS propagation pending (retry with backoff)
- CAA blocks ACME issuance
- proxy/CDN interference
- rate limits / transient ACME errors

Transfers:

- revocation and re-claim MUST be explicit and auditable
- consider a cooldown period before a hostname can be claimed by a different tenant

### B) Store site provisioning (edge)

`ProvisionStoreSite`

- bootstraps store identity, credentials/certs
- configures replication bindings
- validates compatibility constraints
- emits a process fact on readiness

Channel binding:

- a store site MUST declare its `sales_channel_id` and replication bindings are derived from that channel.

### C) Replication control

- `RunDownsync(site_id)`
- `RunUpsync(site_id)`
- `RebuildStoreSiteProjection(site_id)` (replay/backfill)

## Compatibility + rollout policy (edge fleets)

Introduce an explicit compatibility contract:

- store site runs a bundle of component versions
- upgrades happen in a defined order (cloud domain → replication flows → edge domain/projection → POS UI)
- rollback is supported and observable

Deliverables:

- compatibility matrix (proto package majors + component versions)
- `RolloutStoreFleet(bundle, ring)` workflow with canary/observe/advance/rollback

## Reference analog (vendor serviceability constraints)

Commerce platforms impose real constraints on version compatibility and upgrade sequencing across “engine/runtime + storefront/POS + integrations”.
This backlog treats compatibility and rollout policy as first-class deliverables once edge fleets are introduced.
