# 373 – Argo + Tilt + Helm North Star v4 – Implementation Checklist

This checklist tracks the concrete steps to make v4 reproducible: Argo owns the baseline with auto-sync + `selfHeal=false`, Tilt overlays images only, and leftover resources never block installs.

## Current status (2025-11-16)
- [x] Tilt `helm_app` names now match the Argo `apps-*` releases, and Tilt integration/test dependencies use the same names.
- [x] Argo ApplicationSet template renders with `prune=true` / `selfHeal=false`; each app inherits it.
- [x] Devcontainer bootstrap keeps the default ApplicationSets (`foundation,data-services,platform,apps`), so `apps-dev` is applied automatically.
- [x] `apps-transformation` now has a Tilt `helm_app` wired to the GitOps chart/values; integration deps updated.

## Argo alignment
- [x] Ensure `apps-dev` ApplicationSet uses `automated.prune=true` and `selfHeal=false` (template) for all apps (ApplicationSet patch applied).
- [x] Per-app Argo `Application` inherits `selfHeal=false` (verify after ApplicationSet render).
- [x] Confirm value files: shared → env `dev`; Tilt uses the same files (no Tilt-only values). Dev overlays now exist for every `apps-*` component (empty when not needed).

## Tilt resource alignment
- [x] Rename each Tilt `helm_app` to match the Argo release name (`apps-<component>`) using the same chart path/value files and setting `chart_name` when needed.
- [x] Override only `image.repository`/`image.tag` to the local registry; no other value overrides.
- [x] Keep `--only` filtering usable; ensure SERVICE_BASE_DEPS includes proto build.
- [x] Integration/E2E Tilt resources depend on `apps-*` service names (no mixed naming).
- [x] Add Tilt coverage or an explicit defer note for `apps-transformation` to keep the catalog complete.

## Orphan/adoption guardrails
- [x] Add a Tilt preflight local_resource to detect pre-existing resources (SA/Service/ConfigMap/ExternalSecret/HTTPRoute/Deployment) missing Helm ownership for each app and fail fast with a clear remediation.
- [ ] Optional: auto-adopt by annotating/labeling `meta.helm.sh/release-*` and `app.kubernetes.io/managed-by=Helm` before helm apply.
- [ ] In dev, provide a cleanup helper to delete stray resources when adoption is not desired.

## Web apps (www-ameide, www-ameide-platform)
- [x] Tilt resource names match Argo (`apps-www-ameide`, `apps-www-ameide-platform`); chart_name set to underlying chart.
- [ ] Baseline resources have Helm ownership; Argo auto-sync disabled for self-heal but prune on (dev).
- [ ] Verify pods come up with Tilt image (local registry) in dev; Argo sync reverts to GHCR when Tilt stops.

## Apps catalog (dev ApplicationSet)
Concrete list from `gitops/ameide-gitops/environments/dev/components/apps/**/component.yaml`:
- Runtime: `apps-agents`, `apps-agents-runtime`, `apps-inference`, `apps-inference-gateway`
- Core: `apps-graph`, `apps-platform`, `apps-threads`, `apps-transformation`, `apps-workflows`, `apps-workflows-runtime`
- Web: `apps-www-ameide`, `apps-www-ameide-platform`
- QA: `apps-platform-smoke`

For each app:
- [x] Tilt resource name matches above; chart path/value files aligned; `chart_name` set when needed.
- [x] `selfHeal=false` enforced (ApplicationSet and rendered Application) with auto-sync prune on.
- [x] Orphan detection/adoption guardrails applied (SA/Service/ConfigMap/ExternalSecret/HTTPRoute/Deployment).

## Devcontainer / bootstrap
- [x] Devcontainer postCreate runs Argo bootstrap (root `ameide` app with RollingSync ApplicationSet).
- [x] Argo applies apps-dev before Tilt runs; no manual cleanup required on fresh cluster.
- [ ] Buf/Tilt defaults: registry host `localhost:5001` / `k3d-ameide.localhost:5001`; `BUF_TOKEN` loaded.

## CI parity
- [ ] Tilt CI (if used) adheres to same release names and charts.
- [ ] Argo ApplicationSet and Tilt stay in lockstep; no drifted chart paths or value files.

## Verification steps
- [ ] Fresh devcluster: run bootstrap → Argo syncs apps → Tilt up; install succeeds without manual annotations.
- [ ] Trigger Tilt overlays (`--only` subsets) and confirm Argo does not revert images while selfHeal=false.
- [ ] After stopping Tilt, Argo sync returns workloads to GHCR tags on manual sync.

Document blockers or deviations here as they’re discovered and track to resolution.
