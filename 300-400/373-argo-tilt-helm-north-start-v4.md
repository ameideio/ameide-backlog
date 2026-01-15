> **⚠️ DEPRECATED – SUPERSEDED BY [435-remote-first-development.md](435-remote-first-development.md)**

> **UI performance:** See `backlog/681-argocd-ui-performance.md` for tuning notes with ~400 Applications.
>
> This document describes the **Argo + Tilt + k3d** local cluster workflow.
> The project has migrated to a **remote-first model** where:
> - **No local k3d cluster** – All development targets shared AKS dev cluster
> - **Telepresence intercepts** – Replace Tilt Helm resource swapping with traffic routing
> - **ArgoCD in AKS** – Single Argo instance manages all environments (dev/staging/prod)
> - **Bootstrap CLI relocated** – The GitOps/bootstrap script moved from `tools/bootstrap/bootstrap-v2.sh` to `ameide-gitops/bootstrap/bootstrap.sh`; DevContainers now only run `tools/dev/bootstrap-contexts.sh` to connect to the shared cluster.
>
> The `-tilt` release isolation pattern from this backlog is preserved in the new architecture.
> See [435-remote-first-development.md](435-remote-first-development.md) for the current approach.

# 373 – Argo + Tilt + Helm North Star v4 **DEPRECATED**

**Status:** ❌ DEPRECATED (superseded by remote-first development)
**Owner:** Platform DX / Developer Experience
**Updated:** 2025-11-21

## Intent

v4 captures the new equilibrium after reintroducing the full Tilt orchestration while keeping the Argo CD “apps-dev” ApplicationSet for visibility only. The goals:

> **Two independent build paths (clarity)**  
> Argo CD consumes images built during devcontainer bootstrap (scripts under `tools/bootstrap` and `scripts/build-all-images-dev.sh`) and deploys them from the dev registry while running inside the cluster. Tilt is a separate, inner-loop tool running inside the devcontainer: it builds/pushes locally and hot-swaps images on existing resources. Keep the values/Helm config aligned, but remember the build/deploy loops are intentionally separate.

1. Keep Argo CD authoritative for GitOps (foundation + per-service health) but leave auto-sync/self-heal **disabled** on dev app workloads so Tilt can mutate pods freely (manual sync required to reconcile drift).
2. Use a single Tiltfile to build every first-party service, wire Helm charts via `image.repository`/`image.tag`, and expose live-update loops again.
3. Keep infra GitOps-owned (Helmfile layers, secret guardrails, migrations) while exposing optional Tilt hooks for *manual* inner-loop runs (same images/values as Argo) without changing desired state.
4. Centralize SDK builds, linting, integration/e2e runners inside Tilt’s local resources for a real inner-loop hub.
5. Avoid resource ownership conflicts: Argo creates/owns the Kubernetes objects; Tilt should only swap images on those existing resources (no new resource creation when Argo already applied them).

> **Proto / SDK imports**  
> All services follow backlog/393-sdk-import-policy.md: proto types always come from the checked-in `packages/ameide_core_proto` outputs during dev/tests, while runtime images install the SDK packages at the CI-stamped SemVer. This ensures devcontainer/Tilt/Argo share the same contract as the publishing pipeline described in backlog/388.

## Build & deploy paths

There are two independent build/deploy paths:

1. **Argo CD / GitOps path (cluster-side)**
   - Images are built and pushed by the devcontainer bootstrap scripts to a dev registry.
   - Argo CD (running in the k3d cluster) deploys those images based on GitOps values.
   - This path is owned by the bootstrap + Argo/CD pipeline.

2. **Tilt inner-loop path (devcontainer-side)**
   - Tilt runs inside the devcontainer and provides local, fast inner-loop development.
   - Tilt uses its own registry settings (currently `registry.localhost:32000` via `AMEIDE_REGISTRY_HOST[_FROM_CLUSTER]`), and applies resources directly to the cluster.
   - Tilt does **not** depend on images produced by the bootstrap/Argo path, and vice versa.

Changes to the Argo dev registry (e.g., `k3d-ameide.localhost:5001/ameide`) do not automatically change Tilt’s registry; Tilt’s registry configuration is owned here.

### Environment expectations (dev / staging / prod)

- **Dev (k3d):** Argo deploys images from the local dev registry (`k3d-ameide.localhost:5001/ameide`) built by bootstrap; Tilt is optional for inner-loop and may point at its own registry defaults. Pull secrets are disabled in dev values.
- **Staging:** Argo-only. Uses cloud registry (GHCR/ACR) with pull secrets enabled; Tilt is not part of the flow. Env-specific values (`values/staging/...`) must set image repos/tags and secrets explicitly.
- **Prod:** Same pattern as staging—Argo-only, cloud registry with pull secrets, no Tilt. Prod values own the registry host/tags and secret refs; dev-only overlays (k3d registry, no pull secrets) must not be reused.

### Registry/push notes for dev

- Hyphenated image names: build and GitOps values must use hyphens (`agents-runtime`, `inference-gateway`, `workflow-worker`, `workflows-runtime`, `www-ameide`, `www-ameide-platform`) to match containerd pulls and avoid `manifest unknown` from underscore repos.
- Pushing from devcontainer: use the host-published registry port (`AMEIDE_REGISTRY_PUSH_HOST=localhost:5001`) so pushes reach the k3d registry; the registry content is still stored under `k3d-ameide.localhost:5001/ameide/<service>:dev`.
### “Prod-parity first, Tilt second” contract

To keep the developer experience crystal clear, v4 formalizes a two-phase workflow:

1. **Bootstrap = prod.** Opening the devcontainer must converge the entire cluster via Argo before Tilt is even considered. `.devcontainer/postCreate.sh` used to call `tools/bootstrap/bootstrap-v2.sh` with the defaults we rely on (`--reset-k3d --show-admin-password --port-forward`) when the bootstrap CLI lived in this repo; that flow now exists only as `ameide-gitops/bootstrap/bootstrap.sh`, applying the root `ameide` Application and its RollingSync ApplicationSet with the exact same charts + value files that staging/production use, so the baseline dev cluster mirrors prod bit-for-bit.
2. **Tilt = optional overlay.** When hot reload is needed, developers run `tilt up` *after* the Argo baseline exists. Tilt only overrides `image.repository`/`image.tag` via live builds; every other Helm value stays identical to prod. Auto-sync/self-heal are disabled on the apps tier (the ApplicationSet patch overwrites `syncPolicy` and drops the `automated` block), so Argo will not revert Tilt edits until you manually “Sync” the Application.

This sequencing keeps configuration parity airtight (no dev-only manifests, no Tilt-specific values) while still unlocking inner-loop speed when desired.

## Architecture snapshot

| Layer | Tool | Notes |
| --- | --- | --- |
| Foundation / Platform / Data | Argo CD ApplicationSets | Managed from `gitops/ameide-gitops/environments/dev/argocd/...`. Fully automated sync. |
| First-party apps (`apps-*`) | Argo CD RollingSync + Tilt | `ameide-dev` generates one Application per service. The ApplicationSet patch rewrites `syncPolicy`, so the generated apps have **no** `automated` block (no auto-sync/self-heal). Drift from Tilt persists until a manual Sync. |
| Inner-loop builds, SDKs, tests | Tilt | `Tiltfile` builds Docker images via `docker_custom_build`, injects tags via `helm_resource`, and exposes local resources for packages + tests. |
| Migrations / secret guardrails | Argo CD (Helm hooks) | Removed from Tilt; still enforced via Argo/Vault and recovery scripts. |

## Tilt v4 layout

Key sections in the restored Tiltfile:

1. **Guardrails** – `version_settings`, `secret_settings`, `watch_settings`, `docker_prune_settings`, and the banner reminding devs to reuse the shared Tilt session (`http://localhost:10350`).
2. **Registry wiring** – defaults to the Tilt inner-loop registry (`default_registry(host='registry.localhost:32000', host_from_cluster='registry.localhost:32000')`) with env overrides (`AMEIDE_REGISTRY_HOST[_FROM_CLUSTER]`). Point these at `k3d-ameide.localhost:5001` only if you intentionally want Tilt builds to push to the Argo dev registry.
3. **Service catalog** – Explicit `helm_app` calls exist for every `apps-*` release (agents, agents-runtime, inference, inference-gateway, platform, graph, threads, transformation, workflows, workflows-runtime, www apps), wired to the gitops charts/values. `--only` flags use those `apps-*` names.
4. **Live updates per language** – helpers for Go/Node/Python ensure file sync + commands (`pnpm install --no-frozen-lockfile`, `go build`, `uv pip sync`). This restores the sub-second inner loop from v3.
5. **Packages & SDKs** – local resources build/publish `packages/ameide_core_proto`, `packages/ameide_sdk_*`, run lint scripts, and surface `packages:health`.
6. **Integration/E2E orchestration** – `test-job-runner` releases + helper commands (`tilt-e2e-run.sh`, playwright feature triggers, integration cleanup).
7. **Recovery tools** – logs + scripted helpers for e2e/integration pipelines remain. Migrations are GitOps-owned via Argo hooks; Tilt exposes manual helpers (`ops-migrations-image`, `ops-db-migrate-platform`, `ops-flyway-cli`) that build/import the same Flyway image and run once against the cluster when a dev opts in (no desired-state drift).
8. **Values parity** – For each service Tilt passes the same Helm value files Argo uses: shared `_shared/apps/apps-*.yaml` plus the `values/dev/apps/apps-*.yaml` overlay. Dev overlays now exist for every `apps-*` component (most are intentionally empty to defer to shared defaults), so there are no Tilt-only overrides.
9. **Helm version pin** – We pin local tooling to Helm v3.19.2 because Helm 4.0’s default server-side apply trips Argo-managed workloads (`metadata.managedFields` conflicts). Tilt still forces client-side apply (`--server-side=false`) and exposes `AMEIDE_HELM_SERVER_SIDE=true|auto` for intentional SSA experiments once upstream fixes (helm/helm#31515) land.

## Argo alignment

* `.devcontainer/postCreate.sh` historically shelled into `tools/bootstrap/bootstrap-v2.sh` without filters; today the GitOps bootstrap handles the root `ameide` Application, RollingSync ApplicationSet, and all `apps-*` Applications. The template patch replaces `syncPolicy` to add `syncOptions`, which removes the `automated` block—apps do not auto-sync or self-heal until you click Sync.
* Tilt auto-pauses Argo for apps via `argocd-sync-guard` (see `Tiltfile`). When Tilt starts it patches every `apps-*` Application (and parent ApplicationSets) to remove `syncPolicy.automated`, preventing Argo from reinstalling while Tilt owns the loop. On shutdown it restores the policy and runs `argocd app sync <apps>` using the devcontainer `argocd` wrapper, which auto-logs in with the `argocd-initial-admin-secret`, so the final reconcile is authenticated and hands control back to Argo.
* Tilt’s Helm adoption helper details (logging + optional pruning of duplicate workloads) live in `backlog/373-argo-tilt-helm-north-start-v4-helm-adoption.md`.
* Components live under `gitops/ameide-gitops/environments/dev/components/apps/**/component.yaml`. Tilt resource names (`apps-<component>`) match the RollingSync outputs, making drift easy to correlate.
* Developers can still delete the generated `apps-*` Applications if they want a Tilt-only experience, but the recommended path keeps Argo auto-syncing (with `selfHeal=false`) for dashboards + audit trails.

The `argocd-sync-guard` flow works as follows: Tilt launches a `local_resource` that immediately runs `tilt/tilt-argocd-sync.sh pause <apps…>`, which PATCHes each Application to drop `spec.syncPolicy.automated` and also snapshots/restores the RollingSync strategy on parent ApplicationSets (`TILT_ARGOCD_STATE_DIR` stores the serialized strategies). The resource’s `serve_cmd` keeps a long-lived process with a cleanup trap; when Tilt stops, the trap runs `tilt/tilt-argocd-sync.sh resume <apps…>` to re-enable `automated` with `prune/selfHeal`, restores the saved ApplicationSet strategies, and finally executes `argocd app sync <apps>` to reconcile any Helm/Tilt drift. That last call goes through the devcontainer `argocd` wrapper (default context `k3d-ameide`, server `localhost:8443`, uses `argocd-initial-admin-secret`), so it is already logged in and cannot create unauthenticated sync churn.

## What changed vs v3

| Topic | v3 | v4 |
| --- | --- | --- |
| Tilt scope | Some services migrated, but Tiltfile was replaced with a stub during Argo migration. | Full Tiltfile restored, trimmed to exclude infra; migrations stay Argo-owned with optional manual Tilt helpers. |
| Live updates | Removed in stub. | Re-enabled for Go/Node/Python services. |
| SDK/test automation | Previously inline; stub removed them. | Re-added (packages, integration, e2e). |
| Infra guardrails | Tilt used to trigger Helmfile layers + secret bootstrap. | Stripped out; developers run Helmfile/Argo independently. |
| Argo apps layer | Optional `app-ameide` Application. | Canonical `apps-dev` ApplicationSet + component files, auto-applied during bootstrap. |

## Migration checklist

1. **Devcontainer rebuild** – pick up buf CLI + other base image changes.
2. **Bootstrap** – reopen in DevContainer; `.devcontainer/postCreate.sh` installs k3d/tilt/argocd/bicep/yq and applies `apps-dev` via `bootstrap-v2.sh` defaults.
3. **Update `.env`** – include `GITHUB_TOKEN`, `BUF_TOKEN`, registry creds.
4. **Tilt usage**  
   * `tilt up` (all services) or `tilt up -- --only=apps-www-ameide,apps-inference` to filter (use `apps-*` names).  
   * Keep `BUF_TOKEN` exported so `ameide_core_proto` builds succeed.  
   * Use Tilt UI for integration/E2E triggers instead of bespoke scripts.
5. **Argo sync** – Apps do not auto-sync/self-heal (the generated Applications lack `syncPolicy.automated`), so click “Sync” after you stop Tilt to reconcile drift back to Git.
6. **Resource ownership guardrail** – Tilt is an overlay, not an installer. Before every `apps-*` Helm apply, Tilt now runs `tilt/tilt-helm-adopt.sh <release>` automatically (wired as a per-release `local_resource` dependency in the Tiltfile) to scan *all* namespaced API resources for `app.kubernetes.io/instance=<release>` and apply the `meta.helm.sh/*` annotations + managedFields scrub. This keeps Helm + Argo in lockstep even when Argo re-syncs or Tilt forces an uninstall, without relying on background loops. Set `AMEIDE_TILT_SKIP_HELM_ADOPT=true` to bypass (not recommended outside debugging).

## Open items

- Add doc snippet showing how to override `AMEIDE_REGISTRY_HOST[_FROM_CLUSTER]` for remote clusters (k3d defaults documented above, but non-k3d flows still need an example).
- Rebuild the README Tilt section to highlight the `-- --services=` flag and how it maps to `apps-dev`.
- Document the `tilt/tilt-helm-adopt.sh` workflow (when it runs automatically, environment flags, and how to re-run it when taking over an existing cluster).

## References

- `Tiltfile` (new full implementation)
- `gitops/ameide-gitops/environments/dev/argocd/apps/apps.yaml`
- `.devcontainer/postCreate.sh`
- `backlog/343-tilt-helm-north-start-v3.md` (historical context)
- `backlog/364-argo-configuration-v5.md` (Argo manual-sync guidance)
