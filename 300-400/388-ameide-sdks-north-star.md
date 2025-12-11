# 388 – Ameide SDKs North Star

**Status:** Draft (requires CI wiring)  
**Owner:** Platform DX / SDK maintainers  
**Updated:** 2025-11-21

See also: backlog/389-ameide-sdk-ameideclient.md, backlog/390-ameide-sdk-versioning.md, backlog/391-resilient-cd-packages-workflow.md, backlog/393-ameide-sdk-import-policy.md, backlog/395-sdk-build-docker-tilt-north-star.md. This doc is the design-level north star for **outer-loop publishing**; 402/403/404 cover inner-loop workspace use of core_proto + AmeideClient for services, and 390/391/395 describe the publish/release workflows.

## Intent

Codify a boring, cross-ecosystem release system for the Ameide SDKs and adjacent configuration so every artifact (npm, PyPI, Go modules, GHCR images) can be traced back to the same git SHA and semantic version. This replaces ad-hoc filters and “skip the scary bit” workflows with one canonical playbook for release and dev channels.

**Scope:** This doc is about *publishing* SDK/config artifacts for external consumers and release images (Ring 2). Internal services in the monorepo use workspace `packages/ameide_core_proto` + workspace AmeideClient implementations per backlog/402/403/404; they are not blocked on published SDKs in dev/PR CI.

## Target Outcomes

1. **Single source of truth:** one SemVer tag (`vMAJOR.MINOR.PATCH`) per coordinated SDK/config release.
2. **SHA-aligned outputs:** for any SHA we can answer “which npm/PyPI/Go/GHCR bits came from this commit?” without spelunking logs.
3. **Two channels that behave predictably:** `release` (immutable semver) and `dev` (fast-moving, SHA-tagged snapshots).
4. **Minimal bespoke tooling:** no more per-language hacks, filters, or version sanitizers—the CI pipeline computes all ecosystem-specific strings.
5. **Prod/staging pin SemVer, dev pins SHA/dev tags:** guarantees traceability and rollback safety.
6. **Inner/outer loop ownership is explicit:** when Tilt (Ring 1) is running against a service, ArgoCD auto-sync is temporarily disabled for that Application (via `tilt/tilt-argocd-sync.sh`) so the workspace-first image stays in place; once the dev loop completes, auto-sync is re-enabled to hand control back to Ring 2. This prevents controllers from fighting over a Deployment while keeping GitOps the source of truth.

## Core Principles

1. **SemVer drives everything** – Git tags `vMAJOR.MINOR.PATCH` are the canonical signal. Source files keep placeholders (`0.0.0`) so CI can stamp versions dynamically.
2. **SHA is the connective tissue** – Every artifact encodes or stores the git SHA and GHCR always publishes `<sha>` tags plus semantic aliases.
3. **Release vs Dev:**  
   - *Release:* immutable, tagged, tested. Publishes SemVer to registries and GHCR `:latest`/`:X.Y`/`:X.Y.Z`.  
   - *Dev:* every main commit builds SHA-tagged artifacts plus dev/prerelease versions that are safe to overwrite.
4. **Ecosystem compliance over string uniformity** – npm sticks with SemVer + build metadata, PyPI uses PEP 440 dev versions, Go uses pseudo-versions, OCI uses free-form tags.
5. **Pipelines stay boring** – One workflow (`cd-packages.yml`) computes versions, runs tests, and publishes. All temporary hacks die once this is live.

## Reference Version Model

Canonical tuples are derived from git metadata:

```
BASE_VERSION = latest vX.Y.Z tag (or seed)
GIT_SHA      = 71e210beab37b68546d56dceccba18f09264ecc3
SHORT_SHA    = 71e210b
DATE         = 20251113
DATETIME     = 20251113153211 (UTC)
```

### Release Channel (git tag `v0.1.0`)

| Ecosystem | Version / Tag | Notes |
| --- | --- | --- |
| npm | `0.1.0` | Dist-tag `latest` (or `next` if desired). |
| PyPI | `0.1.0` | Wheel + sdist. |
| Go | Git tag `v0.1.0` | Module path `github.com/ameideio/ameide-sdk-go` (append `/vN` when `MAJOR >= 2`). |
| GHCR | `:0.1.0`, `:0.1`, `:latest`, `:<sha>` | All signed; `<sha>` is mandatory. |

### Dev / Snapshot Channel (main commit, no tag)

| Ecosystem | Version / Tag | Notes |
| --- | --- | --- |
| npm | `0.1.0-dev.20251113+g71e210b` | Published with `--tag dev`. |
| PyPI | `0.1.0.dev20251113` | SHA recorded in metadata/release notes (PEP 440 disallows `+`). |
| Go | (release tags only today) | Dev channel currently does not publish pseudo-versions; consume workspace/BSR directly. See backlog/390-ameide-sdk-versioning.md for the authoritative policy. |
| GHCR | `:<sha>`, `:dev` | Optional `:main` alias, but `:dev`/`:<sha>` are required. |

## Ecosystem Implementation

> **See also**: backlog/393-ameide-sdk-import-policy.md for the enforcement details that every repo/service must follow (where to import protos from, how workspace links behave locally, and how Docker/Helm installs resolve the published SDKs via the `sdk_policy_*` guardrails).

### TypeScript SDK (`packages/ameide_sdk_ts`)

- `package.json` stores `0.0.0`; CI rewrites a **temp copy** to `NPM_VERSION` before `pnpm pack` (repo manifests stay on `workspace:^`/`workspace:*`).
- Services/apps inside the monorepo depend on the SDK via `workspace:` ranges for dev/CI installs; pnpm’s publish step rewrites `workspace:` to the stamped semver for consumers automatically.
- Outside the monorepo, consumers pin normal semver (`"@ameideio/ameide-sdk-ts": "0.1.0"`). A post-build smoke job creates a throwaway project, runs `pnpm init -y && pnpm add @ameideio/ameide-sdk-ts@$NPM_VERSION`, and executes `node -e "require('@ameideio/ameide-sdk-ts')"` to prove the packed artifact installs cleanly.

### Python SDK (`packages/ameide_sdk_python`)

- `pyproject.toml` keeps `0.0.0`.  
- CI writes `PYPI_VERSION` into a build-only copy and publishes via `uv build`/`uv publish`.  
- SHA is embedded via wheel metadata or release notes for dev builds.

### Go SDK (`packages/ameide_sdk_go`)

- `go.mod` declares `module github.com/ameideio/ameide-sdk-go` (append `/vN` on major bumps).  
- Releases require pushing `vX.Y.Z` git tags. Dev channel currently does **not** publish pseudo-versions; use workspace + Buf/BSR locally, and publish only on tags per backlog/390-ameide-sdk-versioning.md.  
- No “skip if major > 1” hacks—fix module path instead.

### Configuration Package (`ameide/configuration`)

- Treated like any other SDK artifact.  
- Publish to npm as `@ameideio/configuration@VERSION` and optionally store the bundle in GHCR as `ghcr.io/ameideio/configuration:VERSION`.  
- Services pin SemVer; dev clusters can float on `:dev`/`:<sha>`.

## Build Artifacts & GHCR Images

- Every SDK/config artifact emits a GHCR image for traceability (even if consumers primarily use npm/PyPI). Canonical names: `ghcr.io/ameideio/ameide-sdk-{ts,go,python}:{tag}` and `ghcr.io/ameideio/configuration:{tag}`.  
- Tag policy: always push `:<sha>` (signed), plus `:dev` on dev builds and `:X.Y.Z`, `:X.Y`, `:latest` on release builds.  
- Cosign signs at least the `<sha>` tag; semantic aliases may be signed as policy dictates.

## CI/CD Blueprint (`.github/workflows/cd-packages.yml`)

1. **Trigger classification:**  
   - `refs/tags/vX.Y.Z` → release.  
   - `main` (and optionally other branches) → dev snapshot.  
   - Feature branches default to build/test only (no publish) unless explicitly allowed.
2. **Version computation step:**  
   - Read latest tag for `BASE_VERSION`.  
   - Compute `SHORT_SHA`, `DATE`, `DATETIME`.  
   - Derive `NPM_VERSION`, `PYPI_VERSION`, GHCR tags, `IS_RELEASE`.  
   - Expose via job outputs.
3. **Build & test:**  
   - Run lint/tests for TS/Go/Python/config packages plus any schema validation.  
   - Abort publishing on failure.
4. **Publish order (release):**  
   - Ensure git tag exists/pushed.  
   - `pnpm pack && npm publish @ameideio/ameide-sdk-ts@X.Y.Z`.  
   - `uv build && uv publish ameide-sdk-python==X.Y.Z`.  
   - Push GHCR images with semantic + SHA tags.  
   - Sign `<sha>` tags.  
   - Optional: attach tarballs/archives to a GitHub Release.
5. **Publish order (dev):**  
   - Publish npm/PyPI artifacts using dev versions + `--tag dev`.  
   - Skip git tags; rely on pseudo-version semantics for Go.  
   - Push GHCR `:<sha>` + `:dev` tags, sign `<sha>`.
6. **Artifact smoke (pnpm workspace-first):**  
   - Create a throwaway directory outside the workspace, run `pnpm init -y`, `pnpm add @ameideio/ameide-sdk-ts@$NPM_VERSION`, and `node -e "require('@ameideio/ameide-sdk-ts')"`. This proves the packed artifact installs without abusing `workspace:` or overrides.

Implementation note: backlog/391-resilient-cd-packages-workflow.md documents how the reusable workflow enforces these steps, and backlog/395-sdk-build-docker-tilt-north-star.md explains how service images consume the published outputs.

## Runtime Consumption & GitOps Alignment

- **Prod/Staging:** only deploy semantic tags (e.g., `ghcr.io/ameideio/ameide-sdk-ts:0.1.0`). Record SHAs via annotations (`ameide.io/image-sha: <sha>`).  
- **Dev/Ephemeral:** allowed to pull `:dev` or `:<sha>` but must log the SHA for debugging.  
- Argo CD manifests pin SemVer for prod and staging; Tilt/dev overlays can re-tag images to `<sha>` but should never reference floating semver in dev once a SHA is known.  
- Configuration bundles follow the same tagging policy; `ameide/configuration:dev` is acceptable only outside prod.

## Migration Plan

1. **Cut baseline SemVer** – choose `v0.1.0` (or similar) and tag/push without editing source versions.  
2. **Ship `@ameideio/configuration`** – publish to npm and update consuming services to pin the real version.  
3. **Implement version computation in CI** – add a reusable action or script that emits all derived versions.  
4. **Re-enable npm/PyPI publishing** – remove manual filters and rely on the computed versions.  
5. **Fix Go module path & tagging** – ensure module path matches major version; delete skip guards.  
6. **Expand GHCR tagging/signing** – always push `<sha>` plus semantic tags and cosign the digest.  
7. **Remove transitional hacks** once the above lands (pnpm filters, python sanitizing, go-get skips, etc.).

## Open Questions

- Should every main commit publish dev artifacts, or do we gate on a nightly cron?  
- Who owns bumping the `latest` GHCR tag—automatic on every release or a manual promotion step?  
- Do we need a migration story for legacy consumers stuck on bespoke `2.x` ranges, or do we declare `0.1.0` the new baseline and require opt-in upgrades?

## Current Status (discovery – Nov 2025)

- **Go SDK**: Dedicated repo now live at `github.com/ameideio/ameide-sdk-go` with tag `v0.1.0`; GHCR `ghcr.io/ameideio/ameide-sdk-go` now carries `v0.1.0`, `0.1`, `latest`, and the release digest. Services pin `v0.1.0` (no replaces) with `GOPRIVATE/GONOSUMDB=github.com/ameideio`. Dev/CI needs PAT for private module resolution.
- **TypeScript SDK**: Not found on npmjs (404 for `@ameideio/ameide-sdk-ts`); npm.pkg.github.com still 401 with current token, so likely unpublished/inaccessible. GHCR `ameide-sdk-ts` exists with only `dev`/hash tags (no SemVer aliases).
- **Python SDK**: PyPI has releases (`2.10.1` and dev builds), so SemVer exists publicly. GHCR `ameide-sdk-python` / `ameide-sdk-python-sdist` still carry only `dev`/hash tags (no SemVer/`latest`).
- **Configuration**: GHCR `configuration` package has only `dev`/hash tags; no SemVer/`latest`.
- **Net effect**: Go is now on-path (repo + SemVer tags + GHCR aliases), with CI guard for replaces and bufconn tests for retry/timeout/metadata/tracing. TS remains unpublished to npm; GHCR SemVer aliases are missing for TS/Python/Configuration; CI/CD needs to be wired to publish/tag consistently.
- **Workspace consumption**: Services use `workspace:*` for SDK/config deps during inner-loop; publish steps stamp versions on temp copies and pnpm’s publish rewrite converts `workspace:` to semver ranges. Add a post-publish smoke test outside the workspace (`pnpm init`, `pnpm add @ameideio/ameide-sdk-ts@$NPM_VERSION`, `node -e "require(...)";`) to assert registry installs without bending workspace semantics.
