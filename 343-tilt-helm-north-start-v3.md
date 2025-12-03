> Note: Chart and values paths are now under gitops/ameide-gitops/sources (charts/values); any infra/kubernetes/charts references below are historical.

> **⚠️ DEPRECATED – SUPERSEDED BY [435-remote-first-development.md](435-remote-first-development.md)**
>
> This document describes the **local k3d + Tilt/Helm** workflow which is no longer used.
> The project has migrated to a **remote-first model** where:
> - **No local k3d cluster** – All development targets shared AKS dev cluster
> - **Telepresence intercepts** – Replace Tilt's `helm_resource` hot-swap with traffic routing
> - **Tilt for tests/SDKs only** – Service development uses Telepresence, not Tilt Helm releases
>
> See [435-remote-first-development.md](435-remote-first-development.md) for the current approach.

# 343 – Tilt + Helm North Star v3

**Status:** Draft for review  
**Owner:** Platform DX / Developer Experience  
**Updated:** 2025-11-04

## Why this doc exists

`backlog/142-tilt-helm-north-star-v2.md` still reflects the “digest file” flow that wrote per-service Helm values to disk and disabled Tilt’s automatic image injection. Since the layered Helmfile refactor and recent Tiltfile changes, the team has converged on a lighter-weight model where Tilt injects `image.repository`/`image.tag` on the fly. This note captures:

1. What the v2 workflow looked like (and why it was safe but cumbersome).  
2. How the current v3 approach works, including expectations for charts and Tilt resources.  
3. Migration guidance so new services avoid the old pattern.

---

## Recap: v2 (“tilt-values” digest files)

**Key ideas**

- Tilt built each image with a timestamp tag, pushed to `docker.io/ameide`, resolved the digest, then wrote a Helm values file under `infra/kubernetes/environments/local/tilt-values/<service>.values.yaml`.
- Charts expected `image.graph` + `image.digest`; `helm_resource()` was called with `image_keys=[]` to stop Tilt from adding `--set image.*`.
- Deployment used `helm upgrade --install --values … --values tilt-values/<service>.values.yaml --wait --atomic`.

**Example (from v2)**

```python
# Tiltfile
local_resource(
    'inference-build',
    cmd='bash -lc "set -euo pipefail; docker build -t $EXPECTED_REF services/inference; '
        'docker push $EXPECTED_REF; DIGEST=$(crane digest --insecure $EXPECTED_REF); '
        'mkdir -p infra/kubernetes/environments/local/tilt-values; '
        'cat <<EOF > infra/kubernetes/environments/local/tilt-values/inference.values.yaml\n'
        'image:\n  graph: docker.io/ameide/inference\n  digest: %s\n'
        'EOF\n"' % '${DIGEST}',
    deps=['services/inference'],
)

helm_resource(
    'inference',
    chart='infra/kubernetes/charts/platform/inference',
    namespace='ameide',
    flags=[
        '--values=infra/kubernetes/values/platform/inference.yaml',
        '--values=infra/kubernetes/environments/local/platform/inference.yaml',
        '--values=infra/kubernetes/environments/local/tilt-values/inference.values.yaml',
        '--wait',
    ],
    image_keys=[],  # Disable automatic injection
)
```

```yaml
# Chart snippet (deployment.yaml)
{{- $repo := .Values.image.graph -}}
{{- if .Values.image.digest }}
image: "{{ $repo }}@{{ .Values.image.digest }}"
{{- else if .Values.image.tag }}
image: "{{ $repo }}:{{ .Values.image.tag }}"
{{- else }}
image: "{{ $repo }}"
{{- end }}
```

**Pros**
- Guaranteed digest pinning (immutable deploys).
- Charts stayed digest-first compatible for future CD flows.

**Cons / Pain**
- Tilt had to maintain files on disk; stale files could cause confusing rollbacks.
- `image_keys=[]` meant losing the ergonomic Tilt rebuild → redeploy feedback loop.
- Every new service needed custom scripting (wrap `crane`, manage output dirs).
- Running `tilt down`/`tilt up` sometimes found missing digest files until a manual rebuild ran.

---

## Current v3 workflow (Oct 2025 onward) \*superseded by [backlog/373](./373-argo-tilt-helm-north-start-v4.md)

**Guiding principles**

- Keep Helm authoritative for manifests, but let Tilt handle image injection dynamically.
- Reuse the extension’s `image_keys=[('image.repository','image.tag')]` so the latest rebuild always reaches the Helm release.
- Ensure charts expose `image.repository` and `image.tag` (or adjust `image_keys` to match real field names).
- Fail fast by calling `required "image.repository is required"` (usually through shared helpers) and keep digest-first rendering logic available for downstream CD.
- Push to the embedded k3d registry (`k3d-dev-reg:5000/ameide/...`) so pods can pull without hitting the public internet.
- When Argo CD remains enabled for dev namespaces, keep the relevant Application in **manual** sync or use `automated: { prune:false, selfHeal:false }` so Tilt’s out-of-band changes aren’t immediately healed (see backlog/364 for YAML snippets); Argo will still flag drift for visibility.
- CLI helper scripts deploy with `helm upgrade --install --wait --atomic` so failed releases automatically roll back.
- Registry reachability, insecure-registry wiring, and the `localhost:5000` ↔ `k3d-dev-reg:5000` push/pull expectations are documented in `backlog/358-insecure-registry.md`; `.devcontainer/postCreateCommand.sh` runs `verify_registry_host_and_cluster_flow` to prove both paths on every bootstrap so Tilt’s v3 flow has a guaranteed registry target.
- The concrete k3d configuration that exposes the registry and mirrors (see `infra/docker/local/k3d.yaml` and `scripts/infra/ensure-k3d-cluster.sh`) lives alongside the `backlog/358` runbook; keep those files in sync with this doc when renaming registries or changing ports.

**Template pattern**

```yaml
{{- $repo := required "image.repository is required" .Values.image.repository -}}
{{- if .Values.image.digest }}
image: "{{ $repo }}@{{ .Values.image.digest }}"
{{- else if .Values.image.tag }}
image: "{{ $repo }}:{{ .Values.image.tag }}"
{{- else }}
image: "{{ $repo }}"
{{- end }}
```

Centralize this logic in shared helpers where possible so every template (deployments, jobs, init containers, sidecars) inherits the same digest-first behavior.

**Example (current pattern)**

```python
# Tiltfile
docker_build(
    registry_image('agents'),
    context='services/agents',
    dockerfile='services/agents/Dockerfile.release',
)

helm_resource(
    name='agents',
    chart='gitops/ameide-gitops/sources/charts/apps/agents',
    namespace='ameide',
    values=[
        'infra/kubernetes/values/platform/agents.yaml',
        'infra/kubernetes/environments/local/platform/agents.yaml',
    ],
    image_deps=[registry_image('agents')],
    image_keys=[('image.repository', 'image.tag')],
    resource_deps=INFRA_CRITICAL_PATH,
)
```

```yaml
# Chart snippet (deployment.yaml)
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

When Tilt rebuilds `agents`, it emits a unique tag (e.g., `tilt-aa4e0c3198976779`), pushes the image, and patches the Helm upgrade command:

```
helm upgrade --install \
  --set image.repository=k3d-dev-reg:5000/ameide/agents \
  --set image.tag=tilt-aa4e0c3198976779 \
  ...
```

Kubernetes then pulls from the mirrored registry inside k3d (`k3d-dev-reg`). This happens automatically; developers never edit values files by hand.

**Advantages**
- Zero filesystem artifacts; Tilt manages the values it sets.
- Fast rebuild/deploy loop: saving code → Tilt rebuilds → Helm upgrades with the new tag.
- Matches the layered Helmfile/Tilt split introduced in 2025—Helm still owns manifests, but Tilt remains the rapid inner loop.
- Works for services *and* integration test jobs alike; resources such as `test-agents` or `e2e-playwright` use `helm_resource(..., image_deps=[...], image_keys=[('image.repository','image.tag')])` so rebuilt job images roll out automatically before the Helm job runs.

**Caveats**
- Charts must expose `image.repository` + `image.tag`. If a chart still uses an older field (e.g., `image.graph`), update it or adjust `image_keys`.
- Registry wiring must exist: `k3d cluster create … --registry-create` and the `registry-mirror` component ensure both host and cluster see the same image.
- Tags are mutable in the dev loop; for promotion workflows we still rely on digest pinning downstream (ArgoCD/CI).

---

## Migration checklist (image.graph → image.repository)

1. **Retire v2 artifacts**
   - Remove `infra/kubernetes/environments/local/tilt-values/*.values.yaml` from git and delete the `local_resource` jobs that generated them.

2. **Fix shared helpers first**
   - Add lightweight helpers (`{{ include "service.image" . }}` style) that call `required "image.repository is required" …` and emit the digest/tag fallback snippet.
   - Tekton/operator bundles should provide dedicated helper entry points so all templates inherit the new contract immediately.

3. **Tighten every manifest**
   - Replace inline `image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"` (and similar) with the helper block above for deployments, CronJobs, init containers, and sidecars.
   - For charts with nested blocks (`migrations.image.*`, `jobs.<name>.image.*`), duplicate the pattern so each container has repository/tag enforcement.
   - Migrate high-impact services (graph, inference, workflows, www) first, then expand to job charts and operators in small batches to simplify troubleshooting.

4. **Rename values across the stack**
   - Move defaults from `image.graph` → `image.repository` in chart `values.yaml`, shared overlays (`charts/values/platform/<service>.yaml`), and every environment override (`environments/<env>/platform/<service>.yaml`).
   - Update Helmfile overrides and any scripts or docs that referenced `image.graph`.
   - Regenerate README tables/onboarding snippets so copy/paste examples match the new schema.

5. **Align Tilt wiring**
   - Ensure each `helm_resource` uses `image_keys=[('image.repository','image.tag')]`; add extra tuples for additional containers (e.g., `('migrations.image.repository','migrations.image.tag')`).
   - Keep `image_deps` pointing at the canonical image reference so Tilt triggers the right release.

6. **Validate chart output**
   - Run `helm template <release> ...` after each batch to catch missing `image.repository` values early.
   - Execute `tilt trigger <resource>` (or `helm upgrade --install` in a sandbox) and confirm the resulting pod references `k3d-dev-reg:5000/...:tilt-<hash>` or a supplied digest.

7. **Guardrail in CI**
   - Extend `scripts/infra/lint-image-values.py` (or add Helm unit tests) to fail fast when `image.graph` survives or when `image.repository`/`image.tag` are missing.
   - Keep `.Values.image.digest` support so production automation can continue pinning immutable images.

## Local package resources: keep the workspace immutable

- Local package pipelines such as `packages:python-publish` should avoid mutating watched sources while Tilt is running. Self-editing a dependency (e.g., rewriting `pyproject.toml`) causes the resource to flip back to `Pending` even after success.
- Prefer dynamic version metadata: expose `__version__` from a module (`ameide_sdk.version`) that reads `AMEIDE_PYTHON_SDK_VERSION` (with `AMEIDE_SDK_VERSION` as a temporary alias), mark the pyproject version as dynamic, and let the publish script set the environment variable (`0.1.0.dev<timestamp>` locally, release tags in CI).
- Standardize SDK overrides on `AMEIDE_{LANG}_SDK_VERSION` so every publish helper (Go, Python, TypeScript) reads a predictable variable and can share documentation/CI examples.
- With that pattern, both Tilt and GitHub Actions run the same command (`scripts/tilt-packages-python-publish.sh` → `release/publish_python.sh`) without touching the repo tree, eliminating retrigger loops and keeping the local_resource status stable.
- Harmonize logging across all local resources by sourcing `scripts/lib/logging.sh` (or the sibling Node/Python adapters) so Tilt panes lead with `[resource]`-prefixed `➡️ /✅ /❌` lines. This makes it obvious which script emitted a failure when multiple package pipelines run in parallel; see `backlog/346-tilt-packages-logging-harmonization.md` for the shared checklist.

---

## FAQ

**Q: Can we still pin digests for production?**  
A: Yes. The v3 flow only affects the dev/Tilt inner loop. CI/CD pipelines continue to build images, resolve digests, and update production values/Argo CD manifests. Charts should keep the digest-first logic (`if .Values.image.digest …`) so downstream environments can supply digests while local dev sticks with tags.

**Q: What about multi-arch builds?**  
A: Tilt currently builds for the local architecture. For arm64/x86 parity, use BuildKit with `--platform` in CI; Tilt will eventually adopt the same pattern once we require cross-platform dev builds.

**Q: Do integration test jobs follow the same rules?**  
A: Yes. The Tiltfile already wires `image_deps`/`image_keys` for resources like `test-agents`, so rebuilding the service automatically refreshes the integration job image before Helm applies the job chart.

---

## Next steps (before v4)

- **Status (Nov 4 2025)**  
  - ✅ Agents + core platform charts (graph, inference, workflows, www, threads, transformation, runtime variants) now render from `image.repository`/`image.tag`; Tilt injects `k3d-dev-reg` coordinates without fallbacks.  
  - ✅ Supporting jobs/hooks (`db-migrations`, ClickHouse/Plausible bootstrap, Tekton operator helpers) require the new fields and guard baselining logic for fresh schemas.  
  - ✅ `scripts/infra/lint-image-values.py` fails fast on legacy keys, and shared docs/readmes reference `image.repository` / `image.tag`.  
- ⏳ Remaining clean-up: archive the v2 backlog once every historical reference is updated and extend onboarding docs to point here.  
- ✅ `ops-migrations-image` now participates in the v3 flow: the Tiltfile builds `ameide-migrations` via `docker build` + push to the default registry instead of `k3d image import`, so Helm pulls the refreshed tag automatically and Flyway jobs no longer run stale SQL.
- ⏭️ v4 (see backlog/373) restores the full Tiltfile, removes infra guardrails, and keeps migrations squarely in the Argo/Vault path.

- [ ] Update `backlog/142-tilt-helm-north-star-v2.md` to reference this document (or archive v2 once every service migrates).  
- [x] Audit all charts for lingering `image.graph` fields and convert them to `image.repository`/`image.tag`.  
- [x] Add a lint check (`scripts/infra/lint-image-values.py`) that fails when `image.graph` lingers or when `image.repository`/`image.tag` are absent.  
- [ ] Extend developer onboarding docs to point at this backlog entry for the definitive Tilt/Helm coordination pattern.

---

## Field report: mistakes we actually made (Nov 2025) and how to avoid them

The recent manifest-not-found outage surfaced several concrete anti-patterns. Keep this list handy when onboarding new services or debugging registry issues.

| # | Mistake | Symptoms | Remediation |
|---|---------|----------|-------------|
| 1 | **Dual-tag/push in custom builds.** We minted `tilt-build-*`, `*-tilt_build_base`, and manual `tilt-<sha>` tags inside shell scripts. | Pods pulled `tilt-<sha>` that never existed because Tilt wasn’t allowed to retag/push; registry only stored our bootstrap tag. | Use the “good” `custom_build` (or plain `docker_build`) pattern: `docker build -t $EXPECTED_REF …` and **stop pushing**. Leave a local image so Tilt retags/pushes the final `tilt-<sha>`. If a remote builder must push, adopt `skips_local_docker=True`, `disable_push=True`, and `outputs_image_ref_to='ref.txt'` so Tilt deploys exactly what was pushed. |
| 2 | **Hard-coded registry prefixes in image names.** We baked `k3d-dev-reg:5000/<repo>` into every `registry_image()` call. Tilt then re-prefixed the ref and our host builds/pushes never matched the cluster pull target. | Log spam with encoded names like `localhost:5000/k3d-dev-reg_5000_ameide_agents…`; random tags existed locally but not under the exact name Helm injected. | Keep image names registry-neutral (`registry_image('service')` → `ameide/service`). Configure the registry rewrite once via `default_registry(host='localhost:5000', host_from_cluster='k3d-dev-reg:5000')`. |
| 3 | **Mismatched registry instances.** We occasionally ran a standalone `registry:2` on port 5000 while k3d was also running its managed registry. Tilt pushed to the host registry; pods pulled from the in-cluster registry; catalogs were different → `manifest unknown` every time. | Pods constantly pulled tags that supposedly existed (per host curl) but the same tag 404’d from `k3d-dev-reg:5000`. | Use a single registry instance. Prefer the k3d-managed registry (`k3d registry create k3d-dev-reg --port 0.0.0.0:5000`) and map it via `local-registry-hosting` ConfigMap. Verify with `curl` from both host and an in-cluster pod—the catalogs must match. |
| 4 | **Stale registry contents after flow changes.** We left old manifests/tags around while refactoring the build pipeline, so even after fixing the Tiltfile, the registry still lacked the tags Helm expected. | Confusing 404s even after switching to the “good” build command. | When overhauling the build/tag strategy, wipe the dev registry (`docker run --rm --volumes-from k3d-dev-reg alpine sh -c 'rm -rf /var/lib/registry/docker/registry/v2/*'`) before the first rebuild. Document that this is safe only for disposable dev clusters. |
| 5 | **Force-update Helm resources without a confirmed image push.** The `helm_resource` extension’s force-update step kept re-running `docker push …` commands we thought we had removed, reintroducing bad tags. | Even after editing `_docker_build_command`, Step 3 still pushed `tilt-e820…` because the extension reused the legacy command. | After changing the build command, restart Tilt and confirm `.tilt/tilt.log` shows Tilt’s own “Pushing …k3d-dev-reg_5000…” message, not the hard-coded push. If necessary, clear Tilt state (`tilt down`) so it picks up the new command. |
| 6 | **Skipping the registry health check.** We assumed `localhost:5000` and `k3d-dev-reg:5000` were wired the same and never validated. | Hours lost chasing “mysterious” 404s. | Make it muscle memory: `curl http://localhost:5000/v2/_catalog` (host) and `kubectl run --rm -i regcheck --image=alpine --restart=Never -- curl -s http://k3d-dev-reg:5000/v2/_catalog` (cluster). If outputs differ, fix the registry plumbing before touching code. |

Rebuilding the entire stack after we addressed all six issues resulted in every service pulling the tags Tilt advertised. آینده’s TL;DR: **one image name, one registry instance, one build command.** Let Tilt do the retag/push work it was designed for.



Notes:

Short version: you’ve nailed the symptom (pods pulling `…:tilt-<sha>` that doesn’t exist), but there’s one key misunderstanding: **`$EXPECTED_REF` is a bootstrap ref, not the final `tilt-<sha>` tag.** You shouldn’t try to “make `$EXPECTED_REF` equal the hashed tag.” Tilt computes that hash *after* the build, then retags and pushes it. If your script pushes to the bootstrap ref and doesn’t leave a **local** image for Tilt to retag—or you skip telling Tilt what you actually pushed—Tilt will tell Helm to use `tilt-<sha>` that never got uploaded.

Here’s how to fix it (pick ONE pattern and stick to it):

## 1) Preferred: let Tilt build/push (local Docker path)

* Your script should **not push** and should **only** tag to `$EXPECTED_REF`.
* Tilt inspects the local image, creates `tilt-<sha>`, pushes it, and injects that exact tag into Helm.

```python
# Tiltfile
default_registry(host='localhost:5000', host_from_cluster='k3d-dev-reg:5000')

# Best: drop the wrapper entirely
docker_build('ameide/threads', context='services/threads', dockerfile='services/threads/Dockerfile.release')

# If you MUST keep custom_build, use the "good" pattern and DO NOT PUSH:
custom_build(
  'ameide/threads',
  'docker build -t $EXPECTED_REF services/threads',
  deps=['services/threads'],
  # no skips_local_docker, no disable_push, no pushes in the script
)
```

**Gotcha:** If you use `docker buildx build --push` (or a remote builder), the image does **not** land in the local Docker daemon—Tilt can’t retag/push. Use plain `docker build` (or `buildx … --load`) so the image is available locally.

## 2) Remote builder / no local image: you push, tell Tilt the exact ref

If you can’t load locally (BuildKit remote, Kaniko, CI, multi-arch), then **you** push and **tell Tilt which ref you produced**:

```python
custom_build(
  'ameide/threads',
  # Example with buildx pushing directly to registry
  'docker buildx build --push -t $EXPECTED_REF services/threads && echo $EXPECTED_REF > ref.txt',
  deps=['services/threads'],
  skips_local_docker=True,   # we don’t have a local image
  disable_push=True,         # prevent Tilt from trying to push again
  outputs_image_ref_to='ref.txt',  # tell Tilt the ref we actually pushed
)
```

Even better: if your builder yields a digest, write the **digest ref** you pushed (e.g., `k3d-dev-reg:5000/ameide_threads@sha256:…`) into `ref.txt`. Tilt will deploy exactly that.

---

## Sanity checklist (do these once):

* **No extra tags.** Don’t mint your own `tilt-build-*` or `*-tilt_build_base` tags anymore.
* **Don’t hard-code registry in image names.** Use `default_registry(host='localhost:5000', host_from_cluster='k3d-dev-reg:5000')`. Tilt will push to `localhost:5000/k3d-dev-reg_5000_…` and inject `k3d-dev-reg:5000/…` for pods—that underscore form is expected on the **push** side.
* **Helm wiring:** make sure Tilt knows how to inject `image.repository`/`image.tag` (e.g., via `helm_resource(image_keys=[('image.repository','image.tag')])`), which you already have.
* **Verify after a trigger:**

  * `curl http://localhost:5000/v2/ameide_threads/tags/list` contains `tilt-<sha>`.
  * `curl http://localhost:5000/v2/ameide_threads/manifests/tilt-<sha>` returns a manifest (not 404).
  * `kubectl -n <ns> get pod <pod> -ojsonpath='{.spec.containers[*].image}'` matches that tag.

---

### Bottom line

* I agree with your diagnosis of *why* you’re getting `manifest unknown`.
* The fix is **not** to make `$EXPECTED_REF` equal the final `tilt-<sha>`; you can’t pre-know that.
* The supported fixes are: (1) let Tilt retag & push by leaving a **local** image and not pushing in your script; or (2) if you must push yourself, **declare the pushed ref** via `outputs_image_ref_to` (and set `skips_local_docker=True` + `disable_push=True`).

Do that, and the `tilt-<sha>` tags Helm deploys will actually exist in the registry, and the pods will pull cleanly.



You’re super close. The logs show two separate problems tangled together:

## 1) Your push-side registry ≠ the cluster pull-side registry

* Step 3 pushes to **`localhost:5000/ameide_agents:tilt-e820…`** (host view).
* Helm tells pods to pull **`k3d-dev-reg:5000/ameide_agents:tilt-e820…`** (in-cluster view).
* If `localhost:5000` is **not the same registry instance** as `k3d-dev-reg:5000`, the tag you just pushed won’t exist from the cluster’s perspective → `manifest unknown`.

### Fix it (pick one, but make it consistent):

**A. Use the k3d-managed registry end-to-end (recommended)**

1. Ensure there is **only one** registry bound to port 5000 on your host (stop any ad-hoc `docker run -p 5000:5000 registry:2` you might have started).
2. Create/verify the k3d registry and host mapping:

   * `k3d registry list` → you should see `k3d-dev-reg` with `Host: 0.0.0.0:5000`.
   * If missing:
     `k3d registry create k3d-dev-reg --port 0.0.0.0:5000`
     `k3d cluster create dev --registry-use k3d-dev-reg:5000` (or recreate to adopt).
3. In your **Tiltfile**:

   ```python
   default_registry(host='localhost:5000', host_from_cluster='k3d-dev-reg:5000')
   ```

   Then reference images **without** a registry (e.g., `'ameide/agents'`). Tilt will:

   * push to `localhost:5000/k3d-dev-reg_5000_ameide_agents:tilt-…`
   * inject `k3d-dev-reg:5000/ameide/agents:tilt-…` (or `…/ameide_agents` depending on your naming)
     Both point to the **same** registry instance + manifest.

**B. If you insist on a standalone host registry**

* Make the cluster pull **the exact same** registry you push to:

  * Add a ClusterIP Service + Endpoint in the cluster named `k3d-dev-reg` that points to the host registry (or use k3d’s built-in registry feature instead).
  * Or change Helm to pull from `localhost:5000` via the node’s host networking (messy).
    The clean path is A.

> Quick sanity checks
>
> * From your **host**: `curl http://localhost:5000/v2/` → OK?
> * From **inside the cluster** (e.g., a debug pod): `curl http://k3d-dev-reg:5000/v2/` → must hit the same catalog as the host. If these show different catalogs/tags, you have two registries.

---

## 2) Ensure Tilt actually has a local image to retag & push (or tell it the pushed ref)

Your Step 2 is now correct (builds to `$EXPECTED_REF` and doesn’t push). Step 3 shows **Tilt** pushing `tilt-e820…` with the Docker client — perfect.

**Keep it this way**:

* Custom build command: `docker build -t $EXPECTED_REF …` **only** (no push).
* No `disable_push=True`.
* No `skips_local_docker=True`.
* No extra `tilt-build-*` tags minted by your script.

**Only if you use a remote builder** (no local image), switch patterns:

```python
custom_build(
  'ameide/agents',
  'docker buildx build --push -t $EXPECTED_REF services/agents && echo $EXPECTED_REF > ref.txt',
  deps=['services/agents'],
  skips_local_docker=True,
  disable_push=True,
  outputs_image_ref_to='ref.txt',
)
```

(Then Tilt deploys exactly what you pushed.)

---

## What “good” looks like in your logs

* **Step 2**: names to `localhost:5000/k3d-dev-reg_5000_ameide_agents:tilt-build-…` (or just `…/ameide_agents:tilt-build-…` if you’re not using the encoded path—either is fine *as long as it’s the same registry instance*).
* **Step 3**: “Pushing `localhost:5000/k3d-dev-reg_5000_ameide_agents:tilt-<sha>`” (or the unencoded variant) — succeeds.
* **Helm**: `image.repository=k3d-dev-reg:5000/ameide_agents` (or `/ameide/agents`) and `image.tag=tilt-<sha>`.
* **Pod** pulls that exact tag and starts.

---

## Verify after one trigger

1. Host: `curl http://localhost:5000/v2/ameide_agents/tags/list` → includes `tilt-e820…`
2. Cluster: `kubectl -n ameide exec -it <some-pod> -- curl -s http://k3d-dev-reg:5000/v2/ameide_agents/manifests/tilt-e820…` → returns JSON, not 404.
3. `kubectl -n ameide get pod -ojsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'` → shows the same `tilt-<sha>` tag.

---

### Bottom line

* Your build/retag flow is now correct; the remaining blocker is **registry wiring**. You’re pushing to one registry name (`localhost:5000`) and the cluster is pulling from another (`k3d-dev-reg:5000`). Make them the **same instance** (via k3d’s registry + `default_registry`) and the `manifest unknown` errors will stop.
