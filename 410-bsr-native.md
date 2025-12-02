# 410 – BSR-native SDKs

## Objective
Option A (workspace-first SDKs): BSR is the source of truth for the proto module; SDKs sync stubs from BSR and keep them committed; services in all rings consume only the workspace SDK copies (never direct BSR packages), and published SDKs/BSR packages are for external consumers and smokes only.

## Principles
- **SDK definition:** SDK = Ameide’s language package per runtime (ameide-sdk-go, @ameideio/ameide-sdk-ts, ameide-sdk-python) that bundles synced stubs + helpers (AmeideClient, auth, telemetry, retries). Services import only these SDKs.
- **Single proto surface:** BSR owns schema + stubs; SDKs expose the surface. Services never import workspace `packages/ameide_core_proto` or direct BSR stub packages.
- **Workspace-only for Rings:** services/Docker builds copy workspace SDKs; no registry fetches in Rings 1/2/3.
- **Publish for third parties only:** published SDKs/BSR packages are for external consumers + external smokes, not for internal service builds.
- **Policy enforcement:** CI fails on any workspace proto imports or direct BSR stub imports in services; only SDK imports allowed.
- **Consumption split:** SDKs sync and bundle stubs from the BSR module; services depend only on SDKs (workspace copies in internal rings), never on BSR stub packages directly.

## Plan (phases)

### Phase 0 – Prep
- Confirm BSR module and labels (e.g., `buf.build/ameide/core-proto` with `main/dev/etc.`).
- Add `proto-source.json` to each SDK repo noting the BSR module/ref it syncs from.
- Keep SDK naming stable (Go: `github.com/ameideio/ameide-sdk-go`; TS: `@ameideio/ameide-sdk-ts`; Py: `ameide-sdk-python` with `ameide_core_proto.*` imports).
**Status:** proto-source.json added for Go/TS/Py (module `buf.build/ameideio/ameide:main`). Strategy: SDKs sync and bundle stubs from BSR; services consume SDKs only.
**Done:** proto-source.json added (Go/TS/Py); BSR endpoints identified for TS/Go/Py; strategy locked to “sync + bundle stubs in SDKs; services never import BSR stubs.”
**Gaps:** ensure all SDKs consistently sync from BSR with a no-diff check; keep stubs committed in the SDKs.

### Phase 1 – SDKs sync from BSR (no service generation)
- Add per-SDK sync scripts that run `buf generate buf.build/ameide/core-proto:<ref> --template <sdk-template> --output <sdk-gen-root>`.
  - Go: `scripts/sync_from_bsr_go.sh` → `gen/`.
  - TS: `scripts/sync_from_bsr_ts.sh` → `src/gen/`.
  - Py: `scripts/sync_from_bsr_py.sh` → `src/ameide_core_proto/`.
- Remove/archive service/Tilt proto-gen steps; generation lives only in SDK sync jobs.
- Add “no-diff after sync” CI per SDK: run sync, `git diff --exit-code`.
- Optionally validate the synced BSR ref matches `proto-source.json`.
**Status:** sync scripts + `buf.gen.bsr-*` templates added for Go/TS/Py; CI job `sdk-sync-bsr` added to `ci-proto` to run sync + diff; Tilt runs sync before SDK tests. TS/Go/Py sync scripts run clean; TS/Go/Py tests pass on bundled stubs.
**Done:** added BSR sync templates and scripts (Go/TS/Py); added proto-source.json; CI `sdk-sync-bsr` runs sync+diff; Tilt SDK resources run sync before tests; TS stubs regenerated locally and direct BSR dep removed; Go/Py sync + test passed.
**Gaps:** ensure sync scripts are the only path; clean up any lingering generation references; keep “sync from BSR + no-diff” as the steady state (not migrating to direct BSR packages).

### Phase 2 – Services consume SDKs only
- Imports:
  - Go: `github.com/ameideio/ameide-sdk-go/...`
  - TS: `@ameideio/ameide-sdk-ts/...` (or barrels)
  - Py: `ameide_sdk.*` / `ameide_core_proto.*` exposed by the SDK package (never from `packages/ameide_core_proto`).
- Internal rings resolve SDKs from workspace copies:
  - Go: `go.work` + `replace` to `packages/ameide_sdk_go`.
  - TS: workspace/paths to `packages/ameide_sdk_ts`.
  - Py: `tool.uv.sources`/path to `packages/ameide_sdk_python`.
- No registry installs in internal rings; published SDKs only for external smokes.
- Update Dockerfiles to copy SDKs (not `packages/ameide_core_proto`) and avoid registry fetches in internal rings.
**Status:** imports aligned; workspace resolution in place. TS proto gen references removed from Tilt; dev Dockerfiles now copy SDKs only (no core_proto builds). All SDKs bundle synced stubs; no direct BSR deps in services.
**Done:** services/imports mostly on SDK-only; workspace resolution via go.work/tsconfig/uv sources; Tilt SDK resources sync stubs and test; removed @ameide/core-proto builds from dev Dockerfiles.
**Gaps:** keep Dockerfile/dist scans enforced so services never import/copy workspace proto or BSR stubs; continue tightening Go/Py checks; keep SDK bundling in place; ensure Tilt/CI never pull BSR packages directly into services.

### Phase 3 – Policy enforcement
- Block imports in services that reference `packages/ameide_core_proto` or direct BSR stub packages (Go/TS/Py).
- Block service Dockerfiles that copy `packages/ameide_core_proto` or install SDKs from registries in internal rings.
- Add TS dist scans to fail on workspace proto paths; similar guards for Go/Py sources.
- Keep “no-diff after sync” checks in SDK repos.
**Status:** core policy script updated to new Python import rules; TS policy now blocks `@buf/...` in services; Docker policy blocks @buf/buf.build/gen/go/Python BSR wheels in images; generation guardrails enforce a single Buf module without rebuilding core_proto dist. Bundled stubs remain.
**Done:** Python import policy updated to prefer `ameide_core_proto.*` from SDK; SDK sync CI added; TS policy blocks `@buf/...` in services; Docker policy blocks BSR stub installs; TS dist scan blocks @buf/... in build outputs; generation policy now focuses on Buf module placement only.
**Gaps:** add scans to block `buf.build/gen/...` imports in services (Go/Py); add TS/Go/Py dist scans for BSR/core_proto paths where missing; keep policies aligned with bundled-stub SDKs.

### Phase 4 – Clean up and docs
- Delete/archive proto-generation scripts from services/Tilt; keep `packages/ameide_core_proto` schema-only.
- Document “how to bump proto”: bump BSR label → run SDK sync scripts → PR SDKs → update services if needed.
- Document “when to use AmeideClient vs raw stubs” per language (default: AmeideClient).
- Cross-link 407 (single proto surface) and this doc.
**Status:** docs refreshed toward SDK-only consumption; generation scripts/Tilt cleanup pending. BSR dependency endpoints recorded (see Phase 0). TS local stubs restored; Go/Py still bundle stubs. Need a “bump proto” runbook and AmeideClient vs raw stubs guidance per language.
**Done:** core docs (402/403/404/405/407) aligned to mention BSR-sync; TS gen references removed from Tilt; endpoints recorded; service/SDK docs updated to reflect SDK-only surfaces and committed BSR-synced stubs.
**Gaps:** remove any remaining service/Tilt generation paths; finalize bump-proto runbook inline with BSR sync; add AmeideClient vs raw stub guidance per language; finalize policy/docs with bundled-stub model.

### Bump proto runbook (SDK sync model)
1. Update `packages/ameide_core_proto` schema and push to BSR (`buf build && buf push`).
2. In each SDK:
   - Update `proto-source.json` if the label/ref changes.
   - Run `scripts/sync_from_bsr.sh <label>` (Go/TS/Py).
   - Verify `git diff --exit-code` after sync and commit regenerated stubs.
3. Run SDK tests (Tilt/CI) to ensure workspace SDKs are fresh.
4. Update services if imports changed; keep services pointed at workspace SDKs.

### AmeideClient vs raw stubs
- Default: services consume AmeideClient (or SDK helpers) from the SDK package.
- Raw stubs should be used only inside the SDK or advanced cases; forbid direct use of BSR stub packages in services.

## CI policy scripts (Option A guardrails)
- Entry: `scripts/policy/run_all.sh` (workspace channel by default; set `SDK_POLICY_CHANNEL=release` only for publish strictness)
- `scripts/policy/enforce_proto_generation.sh`: enforce single Buf module (buf.gen* only under packages/ameide_core_proto).
- `scripts/policy/enforce_proto_imports.sh`: block service imports of workspace proto/BSR stubs; enforce SDK-only surface; includes dist/Docker scans.
- `scripts/policy/check_proto_workspace_barrels.sh`: ensure proto barrels point at SDK surfaces.
- `scripts/policy/check_proto_source.sh`: validate proto-source.json alignment with the BSR module/ref.
- `scripts/policy/check_lock_hygiene.sh`: verify SDK lockfile/import hygiene (TS/Go/Py).
- `scripts/policy/check_go_option_a.sh`: guard Go services to use workspace SDK stubs (Option A).
- `scripts/policy/check_go_replaces.sh`: ensure Go replaces point at the workspace SDK copy.
- `scripts/policy/check_docker_sources.sh`: block Dockerfiles from copying packages/ameide_core_proto or fetching Ameide SDKs from registries in Rings 1/2.
- `scripts/policy/ts/rewrite_lock.sh` / `scripts/policy/ts/rewrite_manifests.py`: rewrite TS manifests/lock to enforce workspace SDK resolution for Rings 1/2.
- `scripts/ci/lint_helm_charts.sh`: Helm lint guardrail (used in ci-helm workflow).
- Other CI helpers invoked by GitHub workflows (publish/pack/smoke/sign/setup): `scripts/ci/compute_sdk_versions.sh`, `define_download_helper.sh`, `generate_bsr_artifacts.sh`, `generate_*_sbom.sh` (TS/Go/Python/config), `install_node_workspace_deps.sh`, `install_python_workspace_sdks.sh`, `install_ripgrep.sh`, `install_syft_cosign.sh`, `install_oras.sh`, `login_ghcr.sh`, `pack_*` (TS/Go/Python/config), `prepare_python_env.sh`, `publish_python*.sh`, `publish_ts.sh`, `push_to_ghcr.sh`, `recreate_version_artifacts.sh`, `setup_buf_go_proxy.sh`, `setup_go_auth.sh`, `setup_npmrc.sh`, `sign_oci_artifacts.sh`, `smoke_*` (ghcr, pnpm, ts_registries), `sync_go_repo.sh`, `update_dev_tags.sh`, `verify_*` (ghcr_aliases, go_get, npm_publication, proto_bundle, ts_tgz).

## Open questions
- Exact sync mechanism per language (buf remote generate vs fetch + generate) and cadence.
- Version alignment: how to record “SDK is synced to BSR ref X” (e.g., `proto-source.json`).
- Authoring flow: always push from `packages/ameide_core_proto` to BSR, then sync back? Or evolve `packages/ameide_core_proto` into a pure schema mirror.***
