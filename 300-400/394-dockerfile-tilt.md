Context/relationships: This is the implementation guide for Dockerfile.dev + Tilt structure. It assumes the policy layer defined in backlog/388-ameide-sdks-north-star.md (versioning), backlog/390-ameide-sdk-versioning.md / backlog/391-resilient-cd-packages-workflow.md (cd-packages outputs), backlog/393-ameide-sdk-import-policy.md (proto/SDK runtime rules), and the overarching Docker/Tilt strategy in backlog/395-sdk-build-docker-tilt-north-star.md.

Here’s how I’d approach refactoring this Tiltfile, step‑by‑step, without breaking the current dev workflow and while lining it up cleanly with your SDK/proto/import policies (388/390/393).

Think of this as a migration plan you can execute over a few PRs, not a single giant change.

---

## 0. Goals (so we don’t lose the plot)

Before refactoring anything, agree on the goals for this Tiltfile:

1. **Tilt is dev-only orchestration**

   * Tilt always uses `Dockerfile.dev` (or explicit “dev” Dockerfiles) for services.
   * CI/CD (`cd-service-images`) owns prod Dockerfiles + publish flows.

2. **Keep guardrails, reduce complexity**

   * Keep: proto/SDK guard resources, Argo guards, secret checks, orphan checks.
   * Reduce: copy-pasted `helm_app` calls, custom `docker` shell plumbing where Tilt APIs suffice.

3. **Align with SDK import/version policy**

   * Inner loop can use workspace (`pnpm install --no-frozen-lockfile`, `go.work`, editable installs).
   * The Tiltfile must not accidentally teach people to use prod Dockerfiles or published SDKs for local dev; that’s CI’s job. 

### 0.1 Buf credentials are bootstrapped automatically

As of 2025‑02‑22 the devcontainer hooks (`.devcontainer/postCreate.sh`, `.devcontainer/postStartCommand.sh`) source `scripts/lib/buf_netrc.sh`, which:

* Loads `BUF_TOKEN` from `.env` (or the live env),
* Creates `~/.netrc_buf` with the Buf credentials, exports `NETRC`, and appends it to `.bashrc`.

This gives every Tilt/PNPM subprocess the same Buf auth without re-sourcing `.env`, so `pnpm --filter @ameide/core-proto build` / `buf generate` succeed even when invoked from guardrails or automation. Keep this assumption in mind whenever you add proto/SDK guards—Ring 0 now has deterministic credentials.

**Ring cheat sheet (see backlog/405-docker-files.md for full examples):**

- Ring 0: `packages/ameide_core_proto` is the schema module; Buf generates language stubs into the SDK package trees. Docker never runs Buf.
- Ring 1 (Tilt/PR/dev): `Dockerfile.dev` builds from workspace SDK packages (which already contain generated stubs). TS uses `pnpm install --no-frozen-lockfile`; Go uses `GOWORK=auto`; Python uses `uv sync` + editable installs. No registry/BSR stub packages required.
- Ring 2 (prod/staging images built in-repo): `Dockerfile.release` copies/vendores workspace SDK packages (with generated stubs) and uses strict third-party locks (`--frozen-lockfile`, `uv sync --frozen`, `GOWORK=off` or vendoring). Ring 2 does not install Ameide SDKs from registries.

Published SDK artifacts are used only by the **SDK product track** (out-of-tree smoke validation and external consumers), per `backlog/715-v6-contract-spine-doctrine.md` and `backlog/300-400/393-ameide-sdk-import-policy.md`.

---

## 1. Structural split: break the Tiltfile into modules

Right now everything lives in one monster file. First refactor is **purely organizational**:

### 1.1 Create sub‑Tiltfiles

Under `tilt/`, create:

* `Tiltfile.core` – guardrails, env, secrets, registry, shared helpers (`helm_app`, `docker_custom_build`, `ghcr_build_*`).
* `Tiltfile.packages` – `ameide_core_proto`, `ameide_sdk_ts/go/python` local_resources.
* `Tiltfile.services` – all `apps-*` services that are part of the platform.
* `Tiltfile.tests` – Playwright E2E, integration jobs, `test-all-*` resources.
* `Tiltfile.ops` – migrations, flyway, orphan checks, Argo/Helm adoption helpers (auto-runs `tilt/tilt-helm-adopt.sh <release>` before each `apps-*` Helm apply to stamp `meta.helm.sh/*` ownership).

Then shrink the root `Tiltfile` to just:

```python
version_settings(constraint='>=0.35.2')
min_tilt_version('0.35.2')

load('tilt/Tiltfile.core', 'setup_core')
load('tilt/Tiltfile.packages', 'setup_packages')
load('tilt/Tiltfile.services', 'setup_services')
load('tilt/Tiltfile.tests', 'setup_tests')
load('tilt/Tiltfile.ops', 'setup_ops')

cfg = config.parse()  # Parsed once so sub-Tiltfiles can read cfg_only/contexts
setup_core(cfg)
setup_packages()
setup_services()
setup_tests()
setup_ops()
```

Each `setup_*` function just encapsulates the chunk you already have, so you’re not changing behavior yet, only moving code.

### 1.2 Keep config parsing & context selection in `core`

* `config.define_string_list('only')` and `config.define_string_list('contexts')` stay in `Tiltfile.core`.
* `config.set_enabled_resources(cfg_only_effective)` stays there too, so resource filtering still works no matter how many modules you load.
* Call `config.parse()` exactly once (inside `setup_core` or before invoking it) so every sub-Tiltfile sees the parsed values.

---

## 2. Normalize container build strategy (dev vs “prod-style”)

You already do the right *thing* (services use `Dockerfile.dev`), but it’s buried in each `helm_app` call.

### 2.1 Make “dev image only” explicit

In `Tiltfile.core` (or `Tiltfile.services`), define a simple convention:

* Service images **always** come from `services/<name>/Dockerfile.dev`.
* Prod images **never** get built by Tilt; they’re built by `cd-service-images` using `services/<name>/Dockerfile`.

You already use `.dev` for every `apps-*` Dockerfile passed to `helm_app`; codify that as a rule in comments + doc, so nobody sneaks a prod Dockerfile into Tilt by accident.

### 2.2 Simplify `docker_custom_build`

Right now:

* `_docker_build_command` builds up a `docker build` CLI with `--secret`, `--add-host`, `--build-arg`, etc.
* `docker_custom_build` wraps that with `custom_build(...)`.

This is a good pattern when you need BuildKit secrets (Tilt’s `docker_build` doesn’t surface `--secret` in the API). ([Tilt][1])

Refactor it, but keep the same semantics:

* Move `_docker_build_command` & `docker_custom_build` into `Tiltfile.core`.
* Document them as **“dev-only image builder”** helpers; CI must never depend on them.
* Add a `build_tag_suffix` arg if you want to distinguish dev images (`-tilt-dev` etc.), but keep the image **name** the same as Helm uses so Tilt still wires images correctly.

Don’t change `custom_build` → `docker_build` unless you’re OK losing `--secret` – that’s a separate decision.

### 2.3 Go containers: workspace in Tilt, pinned in prod

Make the Go stance explicit so Tilt (Ring 1) and prod (Ring 2) stay aligned with 402/404:

- Dev/Tilt images copy `go.work` + `packages/ameide_core_proto` + `packages/ameide_sdk_go` and run with `GOWORK=auto` so proto changes break immediately with workspace stubs/SDK (no Buf GOPROXY).
- Prod/`cd-service-images` builds drop `go.work`, set `GOWORK=off`, and rely on tagged `github.com/ameideio/ameide-sdk-go` + go.sum to validate the published module graph.
- If a Tilt build needs cross-service `COPY` hacks to satisfy `go.work`, fix the workspace graph instead of reverting to Buf GOPROXY.

---

## 3. Tighten and centralize `helm_app`

`helm_app` is doing a *lot*:

* Builds an image (`docker_custom_build`)
* Figures out shared vs env values files
* Calls `helm_resource` (with image_deps)
* Calls `k8s_resource` with labels/portforwards/etc.

### 3.1 Split into two helpers

Refactor into:

```python
def build_service_image(name, image, context, dockerfile, live_update, build_args=None, build_secrets=None, only=None, ignore=None):
    docker_custom_build(...)

def deploy_service_chart(name, image, chart_name, namespace, values, resource_deps, image_keys, k8s_labels, port_forwards, links, trigger_mode):
    helm_resource(...)
    k8s_resource(...)
```

Then `helm_app` can just be:

```python
def helm_app(...):
    # compute chart/values/resource_deps/etc
    build_service_image(...)
    deploy_service_chart(...)
```

This makes it *much* easier to:

* Reuse `build_service_image` for integration jobs that need the same image,
* Or eventually add a `helm_app_prod` variant that skips the build and only deploys prebuilt images (for CI/preview envs).

### 3.2 Encode service metadata in a table instead of many calls

Right now you hand‑code:

```python
helm_app(
    name='apps-graph',
    image=registry_image('graph'),
    dockerfile='services/graph/Dockerfile.dev',
    only=[...],
    ignore=[...],
    live_update=[...],
    ...
)
```

for each service.

Refactor into a metadata map in `Tiltfile.services`:

```python
SERVICES = {
  'apps-graph': dict(
    image='graph',
    kind='node',
    dockerfile='services/graph/Dockerfile.dev',
    context='.',
    only=[...],
    ignore=[...],
    live_update=[...],
    chart_name='graph',
  ),
  'apps-agents': dict(
    image='agents',
    kind='go',
    dockerfile='services/agents/Dockerfile.dev',
    ...
  ),
  ...
}

for name, cfg in SERVICES.items():
    helm_app(
        name=name,
        image=registry_image(cfg['image']),
        dockerfile=cfg['dockerfile'],
        context=cfg.get('context', '.'),
        only=cfg.get('only', []),
        ignore=cfg.get('ignore', []),
        live_update=cfg.get('live_update', []),
        chart_name=cfg.get('chart_name', name.replace('apps-', '')),
        # resource_deps defaults from RESOURCE_DEPENDENCIES
    )
```

This removes a *ton* of repetition and makes adding a new service a one‑liner in `SERVICES`.

---

## 4. Package + proto/SDK guardrails

These are already pretty good; just make the intent sharper and slightly reorganize.

### 4.1 `ameide_core_proto` and SDK local_resources → `Tiltfile.packages`

Move:

* `local_resource('ameide_core_proto', ...)`
* `local_resource('ameide_sdk_ts', ...)`
* `local_resource('ameide_sdk_go', ...)`
* `local_resource('ameide_sdk_python', ...)`

into `Tiltfile.packages`.

Adjust a bit:

* Keep `ameide_core_proto` as **auto_init + TRIGGER_MODE_AUTO** (it already is by default), since everything else depends on it.
* For the SDKs:

  * Keep `trigger_mode=TRIGGER_MODE_MANUAL` (they’re heavy), but consider `auto_init=False` so the shared Tilt instance doesn’t slam your laptop by default.
  * Add a comment: “Run these manually when working on SDKs; normal service dev doesn’t need them every run.”

This still honors 393’s “proto package is the single source of truth” and “Tilt mirrors CI steps” intentions, but doesn’t punish every dev session. 

### 4.2 Make resource dependencies explicit and DRY

You already have:

```python
SERVICE_BASE_DEPS = ['ameide_core_proto']
SERVICE_GUARDRAIL_DEPS = SERVICE_BASE_DEPS + ['ameide_sdk_ts', 'ameide_sdk_go', 'ameide_sdk_python']
RESOURCE_DEPENDENCIES = { ... }
```

Refactor so:

* `RESOURCE_DEPENDENCIES` is computed with some helpers, e.g. “all apps-* get `SERVICE_BASE_DEPS` by default; only the ones that really need SDK guardrails get `SERVICE_GUARDRAIL_DEPS`”.
* `helm_app`’s `resolved_resource_deps` uses those defaults unless explicitly overridden.

That gives you a single place to tune “how strict are we for local dev” vs “just give me proto builds”.

---

## 5. Integration & E2E tests cleanup

The bottom third of the Tiltfile is integration/E2E; it’s powerful but a bit wild.

### 5.1 Group all test images in `Tiltfile.tests`

Move all `docker_custom_build(...)` for:

* `playwright-e2e`
* `transformation-integration-test`
* `inference-integration`
* `ameide-sdk-*` integration images

into `Tiltfile.tests` and give them a small helper:

```python
def test_image(name, dockerfile, context='.', only=None, ignore=None, extra_hosts=None):
    docker_custom_build(
        registry_image(name),
        context=context,
        dockerfile=dockerfile,
        only=only or [],
        ignore=ignore or [],
        extra_hosts=extra_hosts or [],
    )
```

### 5.2 Normalize integration jobs

You already have `integration_jobs = [...]` and a loop; that’s good. A tiny cleanup:

* Move the `docker_custom_build` definitions for those images next to their corresponding job config (or at least into the same module).
* Ensure each integration image uses the **same Dockerfile** as the service it’s testing when you want “prod‑style” tests (e.g. maybe `transformation-integration-test` should use `services/transformation/Dockerfile.release`, not `.dev`, if the goal is to match CI behavior).

Tie that back to 393: integration tests that exercise published SDKs + pinned deps should reuse the prod Dockerfile; dev‑only shims can keep `.dev`. 

---

## 6. Config ergonomics & modes

Your `--only` flag is already quite sophisticated. Two small improvements:

### 6.1 Add a `mode` config to toggle tests vs services

At the top of `Tiltfile.core`:

```python
cfg_mode = config.define_string('mode', default='services', args=False)
cfg = config.parse()
mode = cfg.get('mode', 'services')
```

Then in the root Tiltfile:

* `setup_services()` only runs if `mode` in `['services', 'all']`.
* `setup_tests()` only runs if `mode` in `['tests', 'all']`.

This lets people do:

* `tilt up -- --mode=services`
* `tilt up -- --mode=tests`

without editing the Tiltfile or starting multiple Tilt sessions (which your header explicitly forbids).

### 6.2 Annotate guardrails in the UI

You’re already using labels like `['ops.guardrails']`, `['test.integration']`. You can:

* Add a short section at top of `Tiltfile.ops` explaining these label groups.
* Encourage people to filter in the Tilt UI by label when debugging (helps avoid nuking guardrails by accident).

---

## 7. Verification plan

Once refactoring is done, sanity‑check:

1. **Behavior parity**

   * `tilt up` with no flags: same set of services come up as today.
   * `tilt up -- --only=apps-www-ameide-platform` still auto‑enables baseline deps and works.
   * E2E / integration commands still run from the same Tilt buttons.

2. **Tilt API usage**

   * All `docker_build` / `custom_build` calls still match the API doc (`dockerfile`, `context`, `build_args`, `ignore`, `live_update`). ([Tilt][1])
   * `local_resource` and `helm_resource` use `resource_deps` only on resource objects, not on `docker_build`.

3. **Policy alignment (388/390/393)**

   * Tilt never references prod Dockerfiles (except where explicitly desired for integration images).
   * All dev service builds still use workspace dependencies (`pnpm install --no-frozen-lockfile`, `go mod tidy`, `uv sync`) and *don’t* try to hit registries by default. 
   * Guardrails for `ameide_core_proto` & SDKs remain in place but are less intrusive for non‑SDK dev work.

---

“Dockerfiles & Tilt” doc

Especially:

.dev → workspace + --no-frozen-lockfile

prod/integration → published SDKs + frozen installs

Go: private module auth pattern and SDK import rule.

[1]: https://docs.tilt.dev/api.html?utm_source=chatgpt.com "Tiltfile API Reference | Tilt"
