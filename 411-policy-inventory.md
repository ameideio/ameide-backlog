---
title: 411 - Policy & Guardrails Inventory
status: draft
owners:
  - platform
created: 2025-11-27
---

## Scope
Inventory of CI/guardrail scripts, what they enforce, and where they run (Tilt/CI/CD). Aligns with Option A (workspace-first SDKs with BSR sync + committed stubs).

## Policy scripts (Option A guardrails)
- Entry: `scripts/policy/run_all.sh` (default `SDK_POLICY_CHANNEL=workspace`; set `release` only for publish-time strictness if needed).
- `scripts/policy/enforce_proto_generation.sh` — enforce single Buf module (buf.gen* only under packages/ameide_core_proto).
- `scripts/policy/enforce_proto_imports.sh` — block service imports of workspace proto/BSR stubs; enforce SDK-only surface; scans dist/Docker for frozen-lock installs.
- `scripts/policy/check_proto_workspace_barrels.sh` — ensure proto barrels point at SDK surfaces.
- `scripts/policy/check_proto_source.sh` — validate proto-source.json alignment with the BSR module/ref.
- `scripts/policy/check_lock_hygiene.sh` — verify SDK lockfile/import hygiene (TS/Go/Py).
- `scripts/policy/check_go_option_a.sh` — guard Go services to use workspace SDK stubs (Option A).
- `scripts/policy/check_go_replaces.sh` — ensure Go replaces point at workspace SDK copies (workspace); in release channel, expect stamping instead.
- `scripts/policy/check_docker_sources.sh` — block Dockerfiles from copying packages/ameide_core_proto or fetching Ameide SDKs from registries in Rings 1/2.
- `scripts/policy/ts/rewrite_lock.sh` / `scripts/policy/ts/rewrite_manifests.py` — rewrite TS manifests/lock to enforce workspace SDK resolution for Rings 1/2.

## CI helper scripts (invoked by GitHub workflows)
- Helm lint: scripts/ci/lint_helm_charts.sh (ci-helm).
- Buf/BSR proto flow: scripts/ci/compute_sdk_versions.sh, define_download_helper.sh, generate_bsr_artifacts.sh, generate_*_sbom.sh (TS/Go/Python/config), setup_buf_go_proxy.sh.
- Workspace SDK installs/auth: scripts/ci/install_node_workspace_deps.sh, install_python_workspace_sdks.sh, install_ripgrep.sh, install_syft_cosign.sh, install_oras.sh, login_ghcr.sh, setup_go_auth.sh, setup_npmrc.sh.
- Packaging/publish/smoke/sign: scripts/ci/pack_* (TS/Go/Python/config), publish_* (python/ts), push_to_ghcr.sh, sign_oci_artifacts.sh, smoke_* (ghcr, pnpm, ts_registries), recreate_version_artifacts.sh, update_dev_tags.sh.
- Verification: scripts/ci/verify_* (ghcr_aliases, go_get, npm_publication, proto_bundle, ts_tgz).
  - Tagging/aliases: compute_sdk_versions.sh emits channel-aware tags; cd-service-images derives dev → dev+SHA, main → latest+SHA, tags → SemVer/major-minor/latest+SHA; push_to_ghcr.sh + verify_ghcr_aliases.sh apply the same tag matrix for SDK artifacts (cd-packages) and service images.

## Workflow coverage (high level)
- ci-proto.yml: buf format/lint/breaking; generate/push BSR; sdk-sync-bsr (no-diff after sync for Go/TS/Py); sdk-smoke.
- cd-service-images.yml: runs policy scripts (install_ripgrep → proto/import/barrel/lock/Docker source checks); builds service images from workspace SDKs.
- cd-packages*.yml: setup/auth, policy scripts via install_ripgrep, pack/publish SDKs for external consumers.
  - Tagging: dev → dev+SHA, main → latest+SHA, tags → SemVer/major-minor/latest+SHA (applied via push_to_ghcr.sh/verify_ghcr_aliases.sh).
- ci-core-quality.yml / ci-go-integration.yml / ci-integration-packs.yml: standard workspace-dependent CI (uses committed stubs); no direct BSR packages.
- ci-helm.yml: helm lint guardrail.

## Tilt (inner loop)
- SDK resources sync from BSR before tests (TS/Go/Py) and keep stubs committed.
- Services depend on workspace SDK copies; no direct BSR package consumption in Rings 1/2.

## Open items
- Add explicit scans to block buf.build/gen/... imports in Go/Py services; dist scans for BSR/core_proto paths where missing.
- Ensure policy scripts are wired into all relevant workflows (including new jobs, if added).
- Keep docs in 410/402–405 aligned with Option A.
