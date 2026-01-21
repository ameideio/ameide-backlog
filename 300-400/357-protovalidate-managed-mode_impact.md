# 357 – Protovalidate Managed Mode Impact Assessment

> **Superseded for v6 service consumption:** v6 converged on **wrapper SDK-only** contract consumption for runtime services and scenario runners (no direct `@buf/*` or `buf.build/gen/*` imports outside the SDK build pipeline). Use:
> - `backlog/715-v6-contract-spine-doctrine.md` (doctrine: SDK-only; Kafka + CloudEvents)
> - `backlog/300-400/393-ameide-sdk-import-policy.md` (enforced import rules)
> - `backlog/410-bsr-native.md` and `backlog/408-workspace-first-ring-2.md` (workspace-first rings)
>
> This document is retained as historical context for the earlier “services consume Buf artifacts directly” phase.

## Purpose

Document how adopting Buf Managed Mode with Protovalidate (backlog 356) reshapes the most recent delivery workstreams, codify the gaps it opens, and itemize the coordination required across SDKs, CI, Tilt, and domain services before we flip the switch.

## Executive Takeaways

- Managed Mode aligns with existing goals around single-source proto governance and registry-backed SDK consumption, but it requires pruning the last bits of generated code that are still committed and tightening package version hand-offs.
- Protovalidate migration is already assumed by multiple domain reviews; the remaining risk is sequencing SDK cutovers so Org, Graph, Threads, and onboarding journeys pick up the new runtime without breaking in-flight refactors.
- Developer tooling (DevContainer, Tilt, package registries) is ready for the shift, yet workflows must be updated so local/CI bootstrap run `buf generate` or install Generated SDKs rather than leaning on committed artefacts.

## Impact by Backlog Area

### Identity & Onboarding

- Onboarding Temporal workers consume gRPC clients purely through `@ameideio/ameide-sdk-ts`; Managed Mode increases the importance of keeping those packages in sync with proto changes because no generated code will remain in-repo for hot fixes (backlog/319-onboarding-v3.md).
- Realm provisioning and RBAC backlogs continue to depend on proto-level invariants (realm immutability, tenant scoping). Migrating annotations to Protovalidate requires a coordinated release window so Keycloak/role orchestration does not drift during rollout (backlog/323-keycloak-realm-roles-v2.md, backlog/329-authz.md).

### Domain Services & Proto Governance

- Organization, Graph, and Threads reviews already plan to publish regenerated SDKs once their proto cleanups land; Managed Mode makes those releases mandatory because consumers will lose any fallback to committed bindings (backlog/351-proto-review-organization.md, backlog/352-proto-review-graph.md, backlog/353-proto-review-threads.md).
- Protovalidate migration must cascade through shared `RequestContext` helpers and domain visibility models or we risk runtime validation failures once annotations become enforced.

### SDK Distribution & Package Registries

- The publish → smoke → consume contract shifts to Buf BSR artifacts; Verdaccio/devpi/Athens have been removed, so CI/Tilt now install directly from Buf-generated bundles (backlog/356-protovalidate-managed-mode.md).
- SDK dependency hardening work eliminated direct proto imports from services; Managed Mode doubles down on that contract by removing any committed stubs (`packages/ameide_sdk_*` become the only supported entry point) (backlog/336-sdk-proto-dependency-hardening.md).
- Service test guidance warns against bypassing registries or skipping install scripts; Buf Managed Mode must slot into those helpers so tests keep exercising “real” package resolution, whether artifacts come from BSR or our mirrors (backlog/347-service-tests-refactorings.md).

### CI/CD & Tooling

- CI workflows that currently regenerate bindings into `packages/ameide_core_proto/src` need to switch to lint/breaking-only steps and rely on Managed Mode outputs for language-specific jobs (backlog/345-ci-cd-unified-playbook.md).
- Tilt logging and health resources will need minor tweaks to emit Managed Mode cache pulls or Buf CLI invocations so developers keep a coherent view of package generation (backlog/346-tilt-packages-logging-harmonization.md).

### Developer Environment

- DevContainer bootstrap and Tilt orchestration are already the single source of truth for registry image builds and Helm layer reconciliation. We must thread Buf CLI/Managed Mode bootstrap into those flows so a fresh container recovers without manual `buf generate` (backlog/338-devcontainer-startup-hardening.md, backlog/354-devcontainer-tilt.md).
- Docker Buildx, secrets harmonisation, and env bootstrap tasks stay untouched, but documentation needs to call out the new “no generated code in Git” reality during onboarding (backlog/348-envs-secrets-harmonization.md, backlog/355-secrets-layer-modularization.md).

## Required Workstreams

1. **Buf configuration & repo hygiene** – Enable Managed Mode, drop generated language directories from Git, and update `.gitignore`, linting, and local scripts.
2. **SDK publishing strategy** – Decide between Buf Generated SDKs or existing registries; codify the single pipeline per language and update publish smoke scripts accordingly.
3. **Runtime dependency updates** – Add Protovalidate runtimes to Org, Graph, Threads, and onboarding services; ensure lockfiles and container images pull the correct versions.
4. **CI/Tilt alignment** – Update workflows and Tilt resources to call `buf generate` or install published SDKs on-demand; remove bespoke proto sync scripts.
5. **Communication & documentation** – Refresh developer guides and backlog references to explain the Managed Mode flow, package sourcing, and validation changes.

## Risks & Mitigations

- **Version skew during rollout** – Without committed stubs, missed SDK bumps will break services; mitigate with automated smoke tests that import the latest published versions before services build.
- **Registry duplication** – Running both Buf Generated SDKs and existing registries introduces drift; pick one distribution model or automate mirroring so the source of truth remains singular.
- **Validation regressions** – Protovalidate annotations could reject payloads that legacy code emitted; stage rollouts per domain, and leverage feature flags plus expanded service tests to catch violations early.

## Open Questions

- Confirm whether any mirroring to external package registries is still required (BSR-native distribution is now the default).
- Which services generate code locally during image builds versus consuming SDKs, and do we enforce one pattern per language?
- How do we gate Proto + Protovalidate migrations across multi-tenant rollout windows (Org/Graph/Threads) without blocking ongoing feature delivery?

## Implementation Notes (2025-01-24)

- Buf module renamed to `buf.build/ameideio/ameide`; go SDK now regenerates via remote plugins (`buf.gen.sdk-go.yaml`) and Git ignores all generated artefacts.
- Tilt, CLI helpers, and CI smoke steps call Buf directly instead of bespoke go plugin binaries, keeping local/CI generation identical.
- Python and TypeScript SDK sync helpers remain intact, but their outputs are ignored by Git and validated through `sync_proto` commands (`pnpm --filter @ameideio/ameide-sdk-ts run build`, `uv run python scripts/sync_proto.py`).
- CI now pushes the module to BSR on `main`; Go (`ameide_sdk_go/scripts/sync-proto.mjs`) and TypeScript (`ameide_sdk_ts/scripts/sync-proto.mjs`) SDKs now require Buf Generated SDKs (no local regeneration fallbacks).
