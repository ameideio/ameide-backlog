gotcha—zooming out. here’s the **big picture** so this stays boring and reliable.

# North star

* **Tilt = local dev**: fast inner loop. builds, rewrites image refs, applies **dev manifests**, live-updates. no releases.
* **Helm/Helmfile = test/prod**: deploy **prebuilt** artifacts, **digest-pinned**, with rollback. no builds.

# End-to-end flow

```
DEV (laptop)                             CI/CD                                   CLUSTERS
-------------                            ----------------------------------      -------------------------
Tilt (docker_build/custom_build)  →      Build & push images (Buildx/Bazel) →    Staging/Prod via Helmfile
k8s_yaml/kustomize apply          →      Resolve digest: crane digest       →    Deploy digest-pinned charts
Logs/ports/live-update            →      Store + pass digest/env            →    --wait --atomic, rollback
```

# Contracts & conventions

**Images**

* Base chart values: `image.graph: <service name>` (no registry, no tag).
* **Helmfile (non-local)** must set:

  * `image.graph: <full-registry>/<service>`
  * `image.digest: sha256:...` (prefer digest; tag optional as a label).
* **Tilt (local)** deploys **raw manifests** with `image: <service name>`; Tilt rewrites to `docker.io/ameide/<service>:tilt-…`.

**Charts (release-only)**

* Template logic (digest-first, strict):

  ```yaml
  {{- $repo := required "image.graph is required" .Values.image.graph }}
  {{- if .Values.image.digest }}
  image: "{{ $repo }}@{{ .Values.image.digest }}"
  {{- else }}
  image: "{{ $repo }}:{{ required "image.tag or image.digest is required" .Values.image.tag }}"
  {{- end }}
  ```
* This is fine now because **Tilt no longer calls Helm**.

**Dev manifests**

* Keep simple YAML (or a kustomize overlay) per app:

  ```yaml
  containers:
    - name: www-ameide
      image: www-ameide   # no tag, no registry
      imagePullPolicy: IfNotPresent
  ```
* Tiltfile:

  ```python
  default_registry('docker.io/ameide')
  docker_build('www-ameide', 'services/www-ameide')
  k8s_yaml('manifests/dev/www-ameide.yaml')
  k8s_resource('www-ameide', port_forwards=['3000:3000'])
  ```

# CI/CD gates (make failures loud)

1. **Build & push** → `docker buildx build --push -t $IMAGE:$GIT_SHA …`
2. **Resolve digest** → `DIGEST=$(crane digest $IMAGE:$GIT_SHA)`
3. **Policy checks**

   * refuse `:latest`
   * ensure `DIGEST` is non-empty & pullable
4. **Deploy** → `helmfile -e staging apply --wait --atomic`, passing:

   * `WWW_AMEIDE_REPO=registry.example.com/www-ameide`
   * `WWW_AMEIDE_DIGEST=$DIGEST`
5. **Post-deploy** smoke: readiness, 200 on health, error budget alerts.

# What you’ve already fixed (nice!)

* Base values are clean (service names).
* Local overrides don’t inject registry.
* Helmfile skips app releases for `local`.
* Docs updated with the final pattern.
* Chart templates enforce “digest/tag required” (now safe since Tilt ≠ Helm).

# Remaining tidy-ups (quick wins)

* **Remove Helm from Tiltfile** for app services (use `k8s_yaml`/kustomize only).
* **CI env wiring**: expose `<APP>_DIGEST` to Helmfile `set:` or env values.
* **Rollback story**: `helm rollback` + promote previous digest from CI history.
* **Guardrails**: a pre-commit or CI check that rejects tags in base values and registry prefixes sneaking into them.

# Mental model (one-liners)

* *Dev speed needs mutability → Tilt owns it.*
* *Release safety needs immutability → Helmfile pins by digest.*
* *Charts don’t decide tags/registries; tools/environments do.*

If you want, paste one app’s current **dev manifest** and the **helmfile release** block; I’ll sanity-check them against this blueprint and tighten anything that could still bite later.
