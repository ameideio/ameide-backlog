# 409 – Ameide CLI (BSR SDK consumption)

## Goals
- Ship a single CLI (one language) that consumes the BSR-generated SDK for protos.
- Keep AmeideClient (and helpers) in the language SDK package; services and the CLI import it rather than copying client code.
- No local proto generation: proto stubs come from the BSR SDK; lockfiles pin exact versions.
- End users shouldn’t need Buf installed or `.proto` files; only the CLI and its pinned deps.

## Decisions
- Language: choose one (Go binary for easiest distro; or Node/Python if ecosystem fit matters). Avoid multiple CLIs.
- Proto source: BSR SDK only; no workspace `packages/ameide_core_proto` generation in the CLI workflow.
- Client surface: lives in the Ameide SDK package; CLI depends on that package + the BSR SDK.
- Auth/registry: either bundle the SDK in the release artifact (Go vendoring / npm pack / wheel) or ensure the CLI has access to the BSR registry at install time.

## Work items
- Pick language/runtime and distribution target (Go binary vs npm package vs pip package).
- Wire CLI deps to the BSR SDK + AmeideClient package; remove any buf/protoc steps from CLI build scripts.
- Add lock/pin enforcement so CLI builds fail if SDK versions drift (e.g., go.sum/module pin; package-lock/pnpm-lock; uv.lock).
- Add install docs for registry auth (or bundle the SDK so no auth is needed at runtime).
- Add smoke tests: install CLI from its release artifact, run a trivial command that exercises the AmeideClient against a mock/fixture.
- Update policies/docs to note the CLI is a consumer of the SDK-only proto surface; no workspace proto imports.
