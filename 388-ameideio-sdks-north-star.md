Here’s a draft “north star” doc you can drop into `backlog/388-ameideio-sdks-north-star.md` or similar and tweak to taste.

---

# North Star: Cross-Ecosystem Versioning & Publishing for Ameide SDKs

## 1. Purpose

This document defines the **target model** for how we version and publish the Ameide SDKs and configuration artifacts across:

* Source code (TS, Go, Python, configuration)
* Build artifacts (npm, PyPI, Go modules, archives)
* Container images (GHCR)
* CI/CD pipelines
* Runtime deployments (Docker, Kubernetes)

The aim is to replace ad-hoc fixes (filters, skips, custom sanitizing) with a **coherent, boring, standard** approach that:

* Works with each ecosystem’s rules (SemVer, PEP 440, Go modules)
* Aligns all artifacts for a given change on a **single git commit SHA**
* Supports both **release** and **dev/snapshot** flows
* Makes rollbacks, debugging, and audit trails straightforward.

---

## 2. Goals & Constraints

### Goals

1. **Single source of truth for versioning**
   Every SDK/configuration release has one canonical semantic version.

2. **SHA-aligned artifacts**
   For any commit, we can answer: “Which npm / PyPI / Go / GHCR artifacts were built from this SHA?”

3. **Ecosystem compliance**

   * npm → SemVer
   * PyPI → PEP 440
   * Go → Go modules (module paths + tags or pseudo-versions)
   * OCI (GHCR) → arbitrary tags, but consistent scheme

4. **Simple mental model for engineers**
   Easy to see “this service uses SDK X.Y.Z at SHA <sha>.”

5. **Minimal hacks in pipelines**
   No special filters, no ad-hoc version sanitizing, no “skip if >v1”.

### Constraints

* Registries will **not** accept raw SHA as the only “version.”
* We must support at least:

  * `@ameideio/ameide-sdk-ts`
  * `ameide-sdk-go`
  * `ameide-sdk-python`
  * `ameide/configuration` (SDK-adjacent configuration)
* We want to keep GHCR tags `:<sha>` and `:dev` as first-class citizens.

---

## 3. Core Principles

1. **SemVer is the primary identity.**
   Human-facing versions are semantic (`MAJOR.MINOR.PATCH`). Git tags follow `vMAJOR.MINOR.PATCH`.

2. **SHA is the linking key.**
   Every artifact records or encodes the git SHA used to build it and is tagged in GHCR by that SHA.

3. **Two channels: Release and Dev.**

   * **Release**: immutable, tagged, versioned with SemVer.
   * **Dev/Snapshot**: branch/main builds with SHA-encoded prerelease/dev versions; safe to overwrite `dev` tags.

4. **Per-ecosystem compatibility, not uniform strings.**
   We derive different version strings per ecosystem from the same `(base semver, SHA, timestamp)` tuple, while keeping them obviously related.

---

## 4. Versioning Model

### 4.1 Canonical Version Source

* Introduce a single **canonical SDK version** per coordinated release:

  * Example: `0.1.0`
* Store that canonical version in **one place**:

  * The TypeScript SDK’s `packages/ameide_sdk_ts/package.json` holds the canonical base version (currently `0.1.0`) and every other artifact derives from it.
  * This value defines the release train for all SDKs and adjacent configuration; if only one ecosystem needs a hotfix, bump the canonical version anyway, publish the affected artifact(s), and leave the others untouched (or republish no-op builds) so that the version map stays unambiguous.

### 4.2 Release vs Dev versions

Given:

* `BASE_VERSION` = `0.1.0`
* `GIT_SHA` = `71e210beab37b68546d56dceccba18f09264ecc3`
* `SHORT_SHA` = `71e210b`
* `DATE` = `20251113`
* `DATETIME` = `20251113153211` (UTC, taken from `git show -s --format=%cd --date=format:%Y%m%d%H%M%S <sha>`, not the workflow start time)

**Release builds** (tagged with `v0.1.0`):

* npm: `0.1.0`
* PyPI: `0.1.0`
* Go:

  * Git tag: `v0.1.0`
  * Module path: `github.com/ameideio/ameide-sdk-go` (no `/v2` unless MAJOR ≥ 2)
* GHCR:

  * `ghcr.io/ameideio/ameide-sdk-ts:0.1.0`
  * `ghcr.io/ameideio/ameide-sdk-ts:0.1`
  * `ghcr.io/ameideio/ameide-sdk-ts:latest`
  * `ghcr.io/ameideio/ameide-sdk-ts:<sha>` (always)
  * Same pattern for Go, Python, `ameide/configuration`.

**Dev/Snapshot builds** (branch / main, no `vX.Y.Z` tag):

* npm:

  * `0.1.0-dev.20251113+g71e210b`
* PyPI (PEP 440):

  * `0.1.0.dev20251113+git.71e210b`
* Go:

  * Use a Go pseudo-version derived from `DATE+TIME` and `SHA`:

    * `v0.1.0-0.20251113153211-71e210beab37`
  * We do **not** need to tag git for pseudo-versions; Go tooling derives them from commit history.
* GHCR:

  * `ghcr.io/ameideio/ameide-sdk-ts:<sha>`
  * `ghcr.io/ameideio/ameide-sdk-ts:dev`
  * Same for Go, Python, configuration.

Dev channel artifacts are intentionally moving targets for main/dev environments. Use `:dev` or `<sha>` in non-prod; semver pins are reserved for prod (and usually staging).

---

## 5. Implementation Across Components

### 5.1 Source Code

#### TypeScript SDK (`ameide-sdk-ts`)

* `packages/ameide_sdk_ts/package.json`:

  * `version` is canonical for releases/dev.
  * For releases: `0.1.0`
  * For dev: set to derived `0.1.0-dev.<DATETIME>+g<SHORT_SHA>` in CI before publish.

* Other TS consumers (`services/www_ameide`, `www_ameide_platform`):

  * Depend explicitly on `@ameideio/ameide-sdk-ts` versions:

    * Release: `"@ameideio/ameide-sdk-ts": "0.1.0"`
    * For local dev, use workspace ranges (`workspace:^0.1.0`) but published artifacts should always depend on real versions.

* `pnpm-lock.yaml`:

  * Regenerated as part of release so lockfile contains exact released versions, not workspace placeholders.

#### Python SDK (`ameide-sdk-python`)

* `pyproject.toml` or `setup.cfg`:

  * Use a placeholder version (`0.0.0`) in source.
  * CI overwrites `version` at build time:

    * Release: `0.1.0`
    * Dev: `0.1.0.dev<DATE>+git.<SHORT_SHA>`

#### Go SDK (`ameide-sdk-go`)

* `go.mod`:

  * `module github.com/ameideio/ameide-sdk-go`
  * For `MAJOR >= 2`, change to `…/v2`, etc.

* Tagging:

  * Release: `git tag v0.1.0`, push tag.
  * Dev: no tag required; pseudo-version used by Go tooling.

#### Configuration (`ameide/configuration`)

* Treat configuration as a **versioned package** aligned to the SDK version:

  * For TS/Node consumption: publish to npm (or GitHub npm registry) under `@ameideio/configuration` with version `0.1.0` / `0.1.0-dev...`.
  * For GHCR: image tags follow same scheme as SDK images, plus SHA.

---

### 5.2 Build Artifacts

#### npm (TS SDK + configuration)

* For release tags (`vX.Y.Z`):

  * Set `package.json version = X.Y.Z`.
  * Publish to npm:

    * Dist-tag `latest` (or `next`, depending on your branching model).
* For dev builds:

  * Set `package.json version = BASE_VERSION-dev.<DATETIME>+g<SHORT_SHA>`.
  * Publish with dist-tag `dev`.

#### PyPI (Python SDK)

* For releases:

  * `version = X.Y.Z`
* For dev:

  * `version = X.Y.Z.dev<DATE>+git.<SHORT_SHA>`
* Ensure PEP 440 compliance; no extra sanitizing steps should be needed once this is standardized.

#### Go (Go SDK)

* For releases:

  * Tag repo with `vX.Y.Z`.
  * Consumers import as:

    * `github.com/ameideio/ameide-sdk-go@vX.Y.Z`
* For dev:

  * Consumers can use pseudo-versions if they want a specific SHA:

    * `github.com/ameideio/ameide-sdk-go@v0.1.0-20251113153211-71e210beab37`
  * CI can optionally publish a tarball / GH release asset for convenience but not strictly required for Go tooling.

---

### 5.3 GHCR & Docker Images

For each artifact (TS SDK, Go SDK, Python SDK, configuration):

* Build an image **per commit** that compiles/packs the SDK or configuration as needed.

* Tag scheme:

  For dev/main builds:

  * `ghcr.io/ameideio/ameide-sdk-ts:<sha>`
  * `ghcr.io/ameideio/ameide-sdk-ts:dev`

  For release tags (`vX.Y.Z`):

  * `ghcr.io/ameideio/ameide-sdk-ts:<sha>`
  * `ghcr.io/ameideio/ameide-sdk-ts:X.Y.Z`
  * `ghcr.io/ameideio/ameide-sdk-ts:X.Y`
  * `ghcr.io/ameideio/ameide-sdk-ts:latest` (optional / policy-driven)
  * Same for Go, Python, configuration.

* **Signing**:

  * Cosign all images at the `<sha>` tag.
  * Optional: also sign the semantic tags, but `<sha>` is the critical one.

---

### 5.4 CI/CD Pipelines (GitHub Actions)

Single `publish-packages.yml` should implement:

1. **Input classification**

   * If `github.ref` matches `refs/tags/vX.Y.Z` → **release**.
   * Else if `github.ref` is `main` or other development branch → **dev build**.
   * Feature branches can either:

     * skip external publishing, or
     * publish “PR-scoped” artifacts with an additional qualifier (out of scope here).

2. **Version computation step**

   * Read `BASE_VERSION` from `packages/ameide_sdk_ts/package.json`.
   * Compute `SHORT_SHA`, `DATE`, `DATETIME`.
   * Compute ecosystem-specific versions:

     * npm: `NPM_VERSION`
     * PyPI: `PYPI_VERSION`
     * Go: `GO_RELEASE_TAG` or pseudo-version base (no need to write it explicitly, Go handles it).
   * Expose these via `outputs` to subsequent jobs.

3. **Build & test**

   * Run tests/lint for TS/Go/Python/configuration.
   * On failure, **do not publish** anything.

4. **Publish order**

   For releases:

   * Tag repo: `vX.Y.Z` (if not already).
   * npm publish:

     * `@ameideio/ameide-sdk-ts@X.Y.Z`
     * `@ameideio/configuration@X.Y.Z`
   * PyPI:

     * `ameide-sdk-python==X.Y.Z`
   * Go:

     * Ensure `go.mod` / module path is correct.
     * `git tag vX.Y.Z`, push tag (if not present).
   * GHCR:

     * Build images for TS/Go/Python/configuration.
     * Tag `:<sha>`, `:X.Y.Z`, `:X.Y`, `:latest`.
     * Sign `:<sha>`.

   For dev:

   * npm:

     * `@ameideio/ameide-sdk-ts@NPM_VERSION` with `--tag dev`.
   * PyPI:

     * `ameide-sdk-python==PYPI_VERSION`.
   * Go:

     * No tag; pseudo-version works off commit.
   * GHCR:

     * Build images for TS/Go/Python/configuration.
     * Tag `:<sha>` and `:dev` only.
     * Sign `:<sha>`.

5. **Removal of temporary hacks**

   Once this north star is implemented:

   * Remove `--filter ./packages/** --filter ./tools/**` in `pnpm install`. Use `pnpm install --frozen-lockfile` at repo root.
   * Remove “skip go-get if major >1” hack. Instead, fix module path when `MAJOR >= 2`.
   * Remove Python version “sanitizing” step; the computed `PYPI_VERSION` is already PEP 440-safe.
   * Remove “skip sync if repo missing” once `ameideio/ameide-sdk-go` exists and tokens are correct.

---

### 5.5 Kubernetes & Runtime Usage

#### Tagging policy per environment

* **Prod**:

  * Deploy K8s manifests referencing **semver tags**, not `:dev`:

    * `image: ghcr.io/ameideio/ameide-sdk-ts:0.1.0`
    * Optionally, embed the SHA in annotations for audit:

      * `ameide.io/image-sha: 71e210beab37b68546d56dceccba18f09264ecc3`
  * Only roll deploys when version changes.

* **Staging**:

  * Either mirror prod behavior with a “candidate” version (`0.1.0-rc.1`), or
  * Allow `:dev` images, but capture exact SHA via:

    * `image: ghcr.io/ameideio/ameide-sdk-ts:<sha>`
    * A `:dev` alias can be used for exploratory deploys, but rollbacks should always reference a specific SHA tag.

* **Dev / ephemeral environments**:

  * It’s acceptable to use:

    * `image: ghcr.io/ameideio/ameide-sdk-ts:dev`
    * `image: ghcr.io/ameideio/ameide-sdk-ts:<sha>`
  * Dev/ephemeral builds intentionally **do not pin**; prod (and usually staging) pins SemVer. Always keep the SHA discoverable (image ID/tag or annotations) for rollbacks.

#### Configuration delivery

* `ameide/configuration` can be:

  * A library (npm/PyPI), pinned by version in the service lockfiles, and/or
  * An OCI image (`ghcr.io/ameideio/configuration:X.Y.Z` / `:<sha>`) mounted or consumed by services.

* In Kubernetes, prefer:

  * For immutable config bundles: reference `X.Y.Z` or `:<sha>` directly.
  * For dev environments: `:dev` pointing to the latest main commit.

---

## 6. Migration Plan from Current State

1. **Pick canonical base version** (e.g., `0.1.0`) and set `packages/ameide_sdk_ts/package.json` to that value (this file is the single source of truth for the rest of the repo).

2. **Publish `@ameideio/configuration`** to npm with `0.1.0` (or appropriate version), update all services to depend on that version, regenerate `pnpm-lock.yaml`.

3. **Implement version computation** in `publish-packages.yml`:

   * Compute `NPM_VERSION`, `PYPI_VERSION`, `IS_RELEASE`, `BASE_VERSION`, `SHORT_SHA`, etc.

4. **Re-enable npm/PyPI publishing** using the new version scheme (`X.Y.Z` for releases, `X.Y.Z-dev.DATE+gSHA` for dev).

5. **Fix Go module path & tagging**:

   * Ensure `module` path matches MAJOR.
   * Standardize on `vX.Y.Z` tags for releases.
   * Remove the “major >1 skip go-get” workaround.

6. **Complete GHCR tagging**:

   * Retain `:<sha>` and `:dev`.
   * Add semver tags for releases (X.Y.Z, X.Y, latest).

7. **Remove hacks** (filters, sanitizing, skip guards) once above steps are in place and validated.

---

## 7. Open Questions / Decisions

* Do we want to keep **npm / PyPI dev publishing** on every main commit, or only for certain branches?
* How strict do we want to be about **`latest`** tags in GHCR? (Always current stable? Only manually bumped?)
* Do we need a **migration story** for consumers already using old “2.x” or ad-hoc versions, or can we treat the first properly versioned release as a reset (e.g., `0.1.0`)?

---

If you’d like, I can follow this up with an example `publish-packages.yml` skeleton that implements the version computation + per-ecosystem publish logic described here.
