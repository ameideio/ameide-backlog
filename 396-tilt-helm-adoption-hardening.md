# Tilt Helm Adoption Hardening (396) **DEPRECATED** (superseded by backlog/424-tilt-www-ameide-separate-release.md)

**Status**: Draft  
**Intent**: Make Tilt deploys resilient by adopting pre-existing resources automatically and failing fast on ownership drift, eliminating manual fixes for Helm ownership errors.

## What Changed
- **Inline adoption per apply**: Every apps-* Helm deploy now sets `AMEIDE_HELM_ADOPT_CMD`, so `helm-apply-helper.py` runs `tilt/tilt-helm-adopt.sh` immediately before `helm upgrade --install` (no separate adopt resource, no race).
- **Orphan check is strict and broad**: `tilt/tilt-orphan-check.sh` now enumerates all namespaced resource kinds via `kubectl api-resources` and fails on any missing/incorrect `meta.helm.sh/*` or `app.kubernetes.io/managed-by` labels (no warning-only path, no fallbacks).
- **Guardrail simplification**: Removed the extra `apps-*-helm-adopt` local_resource; only the Argo guard remains, paired with the inline adoption hook.
- **Skip transparency**: Adoption can still be skipped via `AMEIDE_TILT_SKIP_HELM_ADOPT`, but runs are explicit—adoption is the default, not optional.

## Expected Behavior
- Tilt preflight (`preflight-orphans`) blocks immediately when any namespaced resource lacks correct Helm ownership metadata.
- Helm applies no longer fail on “invalid ownership metadata” for pre-existing resources because adoption runs right before each apply.
- Developers see a clear error if `kubectl api-resources` cannot run (no silent fallback to a narrow kind list).

## Follow-Ups
- CI: add a small test that shells `tilt/tilt-orphan-check.sh` in a mocked `kubectl` context to guard regressions.
- Observability: optional periodic non-mutating drift check to surface newly orphaned resources without blocking.
- Docs: update Tilt README to mention inline adoption and the strict orphan check expectations.

## Risks / Mitigations
- **kubectl perms**: If the dev context lacks `api-resources` or `get` rights, preflight will fail loudly—ensure local env matches the target namespace.  
- **SSA/Helm 4**: Keep Helm pinned to v3.19.2 with SSA disabled to avoid Argo conflicts during adoption.
