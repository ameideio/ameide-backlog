# 508 – Scrum Protos (authoritative file list)

**Status:** Active baseline for Scrum proto locations  
**Audience:** Proto/SDK owners, Transformation/Process implementers, agents  
**Scope:** Defines the canonical **paths**, **packages**, and **topic mappings** for the Scrum contract. The repo `.proto` files are the source of truth.

## Canonical proto sources (no exceptions)

Scrum domain (Transformation) protos live under:

- `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/transformation_scrum_common.proto`
- `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/transformation_scrum_artifacts.proto`
- `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/transformation_scrum_intents.proto`
- `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/transformation_scrum_facts.proto`
- `packages/ameide_core_proto/src/ameide_core_proto/transformation/scrum/v1/transformation_scrum_query.proto`

Scrum process facts (Process/Temporal) protos live under:

- `packages/ameide_core_proto/src/ameide_core_proto/process/scrum/v1/process_scrum_facts.proto`

If this backlog’s text contradicts those files, the `.proto` sources win.

## Packages

- `ameide_core_proto.transformation.scrum.v1` (Scrum artifacts, intents, facts, query service)
- `ameide_core_proto.process.scrum.v1` (Scrum process facts)

Proto naming and versioning rules are owned by `backlog/509-proto-naming-conventions.md`.

## Topic mappings (runtime seam)

The canonical Scrum runtime seam is defined in `backlog/506-scrum-vertical-v2.md` and uses exactly:

- `scrum.domain.intents.v1` → `ameide_core_proto.transformation.scrum.v1.ScrumDomainIntent`
- `scrum.domain.facts.v1` → `ameide_core_proto.transformation.scrum.v1.ScrumDomainFact`
- `scrum.process.facts.v1` → `ameide_core_proto.process.scrum.v1.ScrumProcessFact`

## Key envelope invariants (summary)

- Messages carry the required envelope fields (see `ScrumMessageMeta` in `transformation_scrum_common.proto`).
- Domain facts carry a monotonic per-aggregate version (`ScrumAggregateRef.version`) for idempotent downstream consumption (Temporal/workflows).
- The seam has exactly three message classes (domain intents, domain facts, process facts); do not introduce additional “extra” topics that duplicate Scrum state.

## How to change Scrum contracts

1. Edit the `.proto` sources under `packages/ameide_core_proto/src/**`.
2. Run `buf generate` to refresh SDKs and any generated glue/tests (see `backlog/520-primitives-stack-v2.md`).
3. Keep CI regen-diff strict so contract drift is caught at compile/test time.

## Historical note

Older revisions of this backlog embedded full proto text. That content is intentionally removed to prevent drift; the repo `.proto` files are the authoritative contract.

