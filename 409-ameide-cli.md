# 409 – Ameide CLI (SDK-only contract consumption)

## Goals
- Ship a single CLI (one language) that consumes Ameide wrapper SDKs for contracts (same import surface as runtime services).
- Keep AmeideClient (and helpers) in the language SDK package; services and the CLI import it rather than copying client code.
- No local proto generation in the CLI: proto stubs are an SDK build pipeline concern; the CLI consumes the wrapper SDK surface with pinned deps.
- End users shouldn’t need Buf installed or `.proto` files; only the CLI and its pinned deps.

## Decisions
- Language: choose one (Go binary for easiest distro; or Node/Python if ecosystem fit matters). Avoid multiple CLIs.
- Proto source: SDK-only (wrapper SDK packages); no workspace `packages/ameide_core_proto` generation in the CLI workflow.
- Client surface: lives in the Ameide SDK package; CLI depends on that package (and any SDK internal generated code it re-exports).
- Auth/registry: either bundle the SDK in the release artifact (Go vendoring / npm pack / wheel) or ensure the CLI can install the SDK packages from standard registries. The CLI should not require BSR access at install time.

## Work items
- Pick language/runtime and distribution target (Go binary vs npm package vs pip package).
- Wire CLI deps to the wrapper SDK surface; remove any buf/protoc steps from CLI build scripts.
- Add lock/pin enforcement so CLI builds fail if SDK versions drift (e.g., go.sum/module pin; package-lock/pnpm-lock; uv.lock).
- Add install docs for registry auth if needed (or bundle the SDK so no auth is needed at runtime).
- Add smoke tests: install CLI from its release artifact, run a trivial command that exercises the AmeideClient against a mock/fixture.
- Update policies/docs to note the CLI is a consumer of the SDK-only proto surface; no workspace proto imports.

## v6 alignment

This doc aligns to:

- `backlog/715-v6-contract-spine-doctrine.md` (SDK-only contract consumption doctrine)
- `backlog/300-400/393-ameide-sdk-import-policy.md` (enforced import policy for runtime consumers)
