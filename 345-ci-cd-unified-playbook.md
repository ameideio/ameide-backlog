# 345 – CI/CD Unified Playbook

> **Update (2026-02-17):** Buf Managed Mode now publishes SDKs directly through the Buf BSR. The sections that reference `*.packages.dev.ameide.io` mirrors document the pre-Managed-Mode architecture and are due for rewrite as the new pipelines land.

**Status:** Draft for review
**Owner:** Platform DX / Developer Experience
**Updated:** 2025-11-04
**Related**: [220-ci-automation-patterns.md](220-ci-automation-patterns.md), [450-argocd-service-issues-inventory.md](450-argocd-service-issues-inventory.md), [456-ghcr-mirror.md](456-ghcr-mirror.md)

## Known Issues

- **`main` tag not pushed to GHCR**: Staging/production environments use `main` tag for images, but CI/CD pipelines are not pushing this tag. See [450-argocd-service-issues-inventory.md](450-argocd-service-issues-inventory.md) for affected services.

## Why this doc exists

Multiple sources currently describe our CI/CD story: the GitHub Actions workflow files, `docs/ci-cd-playbook.md`, and various backlog notes. This backlog item consolidates the end-to-end strategy so engineers have a single north-star reference while we iterate. The goals:

1. Capture the contractual behaviour of the CI gate (quality, security, generated artefacts).  
2. Describe how delivery promotes images, Helm values, and SDK packages.  
3. Standardise environment variables and secrets so Tilt, CI, and release jobs share identical interfaces (notably the `AMEIDE_{LANG}_SDK_VERSION` convention).  
4. Enumerate outstanding work needed to keep CI/CD frictionless across languages and deployment targets.

---

## Current pipeline architecture

### Continuous Integration

- **Core quality gate** (`.github/workflows/ci-core-quality.yml`)  
  Runs on pushes/PRs. Installs dependencies via PNPM + `astral-sh/setup-uv`, executes lint/format/test suites:
  - JS/TS: `pnpm lint`, targeted Jest suites (`services/www_ameide_platform`, `packages/ameide_sdk_ts`).  
  - Python: Ruff (temporary `--exit-zero`), MyPy, pytest across `packages/`.  
  - Publishes coverage XML and uploads to Codecov (`codecov.yml` enforces ±2 % tolerance). Failures block merges.
- **Buf proto checks** (`.github/workflows/ci-proto.yml`)  
  Lints schemas, performs breaking-change detection, and regenerates bindings into `packages/ameide_core_proto/src`.
- **Security guardrails**  
  - Testcontainers-backed e2e suites ensure Postgres compatibility; Docker availability is required on runners.  
  - Repository scripts (`scripts/sdk_policy_enforce_proto_imports.sh`) prevent improper proto consumption and can be wired into CI jobs.

### Continuous Delivery

- **Service image pipeline** (`.github/workflows/cd-service-images.yml`)  
  - `verify` job repeats the core gate subset.  
  - `build` job authenticates to Azure via OIDC, leverages BuildKit + cache pushes, and deploys to environment-specific tags (`staging` for branches, `production` for semver tags).  
  - Generates SPDX SBOMs with Syft, signs images with Cosign keyless signing, and attaches the SBOM as a Cosign attestation for every build.  
  - Trivy scans built images (fail on HIGH/CRITICAL, ignore unfixed), uploads SARIF with severity limits, and surfaces results in GitHub Security.
- **Promotion workflow** (`.github/workflows/cd-cd-promote-release.yml`)  
  - Triggered by GitHub Releases with a `v*` tag.  
  - Waits for ACR availability, rewrites production Helm overrides with the tagged images and digests, and opens an automated promotion PR.
  - Verifies Cosign signatures + SPDX SBOM attestations and enforces Trivy HIGH/CRITICAL policy gates (with allowlist support) before touching manifests.

### Package publishing

- **Buf module push** (`.github/workflows/ci-proto.yml`)  
  On `main`, lints schemas, runs breaking-change detection, and executes `buf push` so the `buf.build/ameideio/ameide` module stays current.
- **Generated SDK verification** (`.github/workflows/cd-packages.yml`)  
  The TypeScript, Go, and Python jobs invoke the release helpers under `release/` (`publish.sh`, `publish_go.sh`, `publish_python.sh`). Each helper resolves the latest Buf Generated SDK version, downloads it from the Buf registry (npm/go/pypi endpoints), and records a manifest under `release/dist/<lang>/manifest.json`. No Verdaccio/devpi/Athens uploads remain.
- **Local/Tilt alignment**  
  Tilt resources call the same sync scripts (`packages/ameide_sdk_ts/scripts/sync-proto.mjs`, etc.) and surface Buf download failures immediately; `scripts/tilt-packages-*.sh` were deleted with the registry retirement.

### Secrets and environments

- GitHub Environments: `staging` (branch builds) and `production` (tagged releases) with dedicated secrets.  
- Local development relies on the k3d container registry mirror plus Buf’s BSR for SDK downloads; no language-specific package registries run in-cluster.  
- Dependabot covers npm, pip, Docker, and GitHub Action updates weekly.

---

## Standardised interfaces

### SDK version environment variables

- **Convention:** `AMEIDE_{LANG}_SDK_VERSION` governs snapshot/release versioning across all SDK publishers.  
  - Go: `AMEIDE_GO_SDK_VERSION` (alias for `GO_SDK_VERSION`, defaults to pseudo-version `v0.0.0-YYYYMMDDHHMMSS-<commit>`).  
  - Python: `AMEIDE_PYTHON_SDK_VERSION` (temporary alias `AMEIDE_SDK_VERSION` supported while we transition). `pyproject.toml` reads the value via `ameide_sdk.version.__version__`.  
  - TypeScript: `AMEIDE_TS_SDK_VERSION` (replacing `SDK_TS_SNAPSHOT_VERSION`). Publish script copies the workspace into a temp dir before mutating `package.json`, keeping Tilt’s filesystem watch stable.
- CI workflows should export the appropriate env var before invoking `release/publish_*`. Tilt helpers should do the same, so both paths stay congruent.

### Buf credentials

- Buf CLI authentication (`BUF_TOKEN`) is required for both module pushes and Generated SDK downloads across workflows.  
- Local development relies on `buf login` (DevContainer seeds `.netrc`); no `*.packages.dev.ameide.io` mirrors remain.  
- Azure credentials provided via OIDC continue to handle container pushes.

### Reporting artefacts

- Coverage uploads (Codecov), SBOM artifacts, and Trivy SARIF scans are mandatory outputs per run.  
- Package publish jobs should emit the published version as a build output (`tmp/tilt-packages/<lang>-version` locally; GitHub `step.outputs` in workflows).

---

## Outstanding work

1. **CI workflow alignment**  
   - TypeScript workflow now exports `AMEIDE_TS_SDK_VERSION` and uploads the published version artifact; replicate the pattern when introducing Go/Python publishers.  
   - Ensure future Go/Python workflows emit job outputs or artifacts mirroring the published snapshot.
2. **Artifact verification**  
   - Keep the Cosign/Attestation gate and Trivy policy allowlist documented; surface violations in PR summaries so reviewers know why a build failed.
3. **Documentation refresh**  
   - Merge this backlog snapshot into `docs/ci-cd-playbook.md` once the env-var changes ship.  
   - Add developer onboarding steps describing how to run local `publish → smoke` loops and how they mirror CI.
4. **Automation health metrics**  
   - Instrument the workflows (e.g., with GitHub Insights dashboards) to track failure rate, time to green, and publish success rates.

---

## Next steps

- [ ] Extend GitHub Actions with Go/Python publishers that export the new env vars (TypeScript done).  
- [ ] Document the CVE allowlist stewardship process and ensure promotion PRs summarize any waived findings.  
- [ ] Produce a single “CI/CD quickstart” section in `docs/ci-cd-playbook.md` referencing this backlog item.  
- [ ] Close this backlog item once the documentation and workflow updates ship.
