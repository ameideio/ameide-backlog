You’re not wrong to feel the approach has been flipping a bit. What you’re asking for is actually pretty crisp:

> **SDKs are product artifacts. Internal services should sit “on top of” protos, not “behind” SDK publishing. When I change a proto, my services should break *immediately* in dev/CI without waiting on cd‑packages or registry publishes.**
>
> **Dev and CI both use Buf-generated stubs emitted into the SDK packages (TS/Go/Py) from `packages/ameide_core_proto`; services consume the SDK surface only. Published SDK artifacts are for external consumers and out-of-tree smoke validation.**

You can absolutely have that **and** keep Ameide SDKs as products. It just means tightening the layering and being explicit about two paths:

* **Inner loop / internal services** → proto‑first, workspace‑only.
* **Outer loop / external consumers** → SDK‑as‑product, versioned, published.

**Fast map (rings ↔ builds):**

| Ring | Who | Inputs in builds | Registry requirement |
| --- | --- | --- | --- |
| 0 | `packages/ameide_core_proto` | Buf CLI, committed stubs, generated into SDK packages | Buf for plugins only |
| 1 | Services / PR CI / Tilt / dev images built in-repo | Workspace SDK packages (with generated stubs) + AmeideClient (TS/Go/Py) | None |
| 2 | Prod/staging service images built in-repo | Workspace SDK packages (with generated stubs) + AmeideClient; strict third-party locks | None |
| SDK product track | External consumers + publish smoke | Published SDKs (`@ameideio/…`, `ameide-sdk-*`) | npm/PyPI/Go module |

Naming note for this repo: every `Dockerfile.dev` implements Ring 1 (workspace-first dev/Tilt images), while the `Dockerfile.release` files implement Ring 2 (workspace SDKs, strict third-party locks; no Ameide packages pulled from registries). The SDK publish/smoke path is separate from these rings.

Related: backlog/388 (SDKs as products), backlog/390 (tag authority), backlog/393 (old import policy; superseded for Ring 1 by this doc + 403/404/405).

Below is how I’d shape that, in terms of architecture and what to tweak in the existing docs.

---

## 1. Core principle: two rings around the protos

Let’s define clear rings:

1. **Ring 0 – Schema (protos)**

   * `packages/ameide_core_proto` is the single schema module. Buf generates TS/Go/Py stubs directly into the SDK packages (including buf/validate).

2. **Ring 1 – Internal consumers (services, tests, tools)**

   * Services depend on the SDK proto surface only (TS/Go/Py) inside the monorepo.
   * When you change a proto and rebuild core_proto → SDKs, TS typecheck / `go build` / Python tests break immediately.
   * No external registry, no `cd-packages`, no published SDKs in this feedback loop.

3. **Ring 2 – Workspace-first prod/staging images**

   * Same imports as Ring 1, resolved to workspace SDK packages (copied/vendor in Dockerfiles), with strict locking for third-party deps.
   * No Ameide packages are pulled from registries in these images; they still break on proto drift without waiting for publishes.

4. **SDK product track (outer loop)**

   * `cd-packages` builds and publishes the SDKs for external consumers and runs out-of-tree smoke tests. This is not a ring for service images.

Rule of thumb:

> **Code imports proto/clients from the SDK barrels (`@ameideio/ameide-sdk-ts/proto.js`, `github.com/ameideio/ameide-sdk-go/gen/...`, `ameide_core_proto.*` from the Python SDK package). Rings 1 and 2 resolve those to workspace SDK packages; published SDKs are only for the outer-loop smoke and external consumers. Stubs are synced into SDKs from the BSR module (see 410 – BSR-native SDKs); services never import BSR stub packages directly.**

---

## 2. What this means concretely per language

### TypeScript

**Today (per 393):**

* Dev/tests: services import types from `@ameideio/ameide-sdk-ts/proto.js` but resolution often pointed straight at `packages/ameide_core_proto/dist/src/index.js`.
* Runtime: services are supposed to get Connect clients from `@ameideio/ameide-sdk-ts` (AmeideClient).
* CI/Docker guardrails try to force published SDK consumption, which is where you’re feeling pain.

**Adjusted model:**

* **Inner loop / CI on PRs:**

  * Services import **types + clients** from `@ameideio/ameide-sdk-ts/proto.js` (and SDK entrypoints). In Ring 1 this resolves to the **workspace** SDK build that already contains the generated stubs (buf/validate included).
  * No requirement that `@ameideio/ameide-sdk-ts` exists on npm / GHCR to get a passing build.
  * Breaking proto change → `pnpm --filter @ameide/core-proto build` (emits into SDK) → services fail typecheck or tests immediately.

* **Outer loop / release:**

  * `cd-packages` stamps `NPM_VERSION`, runs tests for `packages/ameide_sdk_ts`, and **publishes** it as `@ameideio/ameide-sdk-ts@X.Y.Z` using the SemVer scheme from 388/390.
  * Smoketests install it out of tree.
  * External consumers use the published package; internal services still use workspace.

Key tweak to docs: 393 should make clear the SDK surface is the only import path for services; resolution is to workspace SDKs in Rings 1/2 and to published artifacts only in the SDK product track.

---

### Go

**Today (docs vs reality conflict):**

* 388/390/393 say: Go services depend on `github.com/ameideio/ameide-sdk-go`, using tags/pseudo‑versions; CI/Docker set `GOWORK=off` and must not use replaces.
* 393 also mentions **Option A** from Buf: consume generated stubs via `buf.build/gen/go/...` with `go get`, no local caches. 
* You’ve bounced between those two, which is exactly the confusion you’re feeling.

**Adjusted model:**

* **Inner loop / PR CI:**

  * `go.work` maps `./packages/ameide_sdk_go` into the workspace (with stubs generated into the SDK tree).
  * Services import AmeideClient + stubs from the SDK module via `go.work`, not `packages/ameide_core_proto` paths.
  * `go build ./...` fails immediately when a proto change breaks a service—no `go get` from GitHub, no Buf registry dependency in this loop.

* **Outer loop / release:**

  * `cd-packages` tags and publishes `github.com/ameideio/ameide-sdk-go` at `vX.Y.Z` per 388/390.
  * Release validation and smoke tests use `GOWORK=off` + `go get github.com/ameideio/ameide-sdk-go@vX.Y.Z`.
  * External consumers use that module; internal services don’t care whether this publish has happened to keep iterating.

So Go gets the same story as TS:

> **Workspace AmeideClient + local stubs for dev/CI; tagged `ameide-sdk-go` for external consumers and release tests.**

Buf’s Option A (`go get buf.build/gen/go/...`) still applies as the underlying stub source; the only difference is *where* you add the AmeideClient layer and who is forced to use the published module.

---

### Python

**Adjusted model mirrors TS/Go:**

* Inner loop / PR CI:

  * `uv` installs the workspace `ameide_sdk_python` (with generated stubs) in editable mode.
  * Services import stubs from `ameide_sdk.proto.*` only; no `ameide_core_proto` alias.
  * Breaking proto change → tests break immediately.

* Outer loop / release:

  * `cd-packages` stamps `PYPI_VERSION`, builds/publishes `ameide-sdk-python`, and updates GHCR artifacts as in 388/390.
  * External consumers use the PyPI wheel; your release smoke tests ensure it still matches the current protos.

---

## 3. How this hits the “immediate error, no noise” requirement

With that split:

* **Proto change workflow becomes:**

  1. Edit `.proto` in `packages/ameide_core_proto`.
  2. Run `pnpm --filter @ameide/core-proto build` (local Buf generate + TS build; Go/Py codegen lives here too).
  3. Run your normal inner-loop checks (`pnpm test`, `go test ./...`, `uv run pytest`).

* Any service that:

  * imports a type that no longer exists,
  * relies on a field that changed type,
  * expects a method that was renamed,

  will **fail immediately** in the workspace build.

No `cd-packages`, no npm/PyPI/Go module publish, no Buf/BSR network round‑trips are required to see those errors.

The SDKs (as products) are **agnostic** to that inner loop:

* They’re just libraries in `packages/ameide_sdk_*` that you optionally build/run tests for.
* Publishing them is an outer‑loop concern (tag + `cd-packages`), not a prerequisite for making internal services compile.

That’s exactly the property you described.

---

## 4. What you should tweak in the docs to reflect this

To make this consistent and stop the flip‑flop, I’d explicitly change the docs like this:

### 388 – SDKs North Star 

* Keep it as the **“SDKs as products” doc**.
* Add a short note up top:

  > “This document describes how we version and publish the TS/Go/Py SDK *as artifacts* for external consumers and release validation. Internal services in the monorepo consume the SDK proto surface from the workspace SDK packages (generated from `packages/ameide_core_proto`); they are not required to consume published artifacts.”

### 389 – AmeideClient Parity 

* Clarify that the AmeideClient contract applies to:

  * the internal AmeideClient implementations in the repo,
  * and the SDKs that package them.
* Remove or soften any language that says “all services must consume the *published* SDK”; instead say “all clients of the platform (internal or external) must respect this API, but monorepo services may link the workspace implementation.”

### 390 – Versioning alignment 

* Make it explicit that:

  * **PR/branch CI** uses workspace dependencies only (no requirement to hit npm/PyPI/Go module registries).
  * The tag‑driven publish + SemVer model applies to **external SDK distribution and release images**, not to inner‑loop builds.

### 393 – SDK Import & Proto Consumption Policy 

This is the biggest one to tweak:

* Change the strong global statement:

  > “Runtime consumers never link the raw stubs directly; they import `@ameideio/ameide-sdk-ts`, `ameide-sdk-go`, or `ameide-sdk-python`.”

  into something like:

  > “**External** runtime consumers and SDK publish smoke use `ameide-sdk-*` as the public surface from registries.
  > **Internal** monorepo services (dev, PR CI, prod images built in-repo) consume the SDK proto surface from workspace SDK packages so proto changes fail fast without waiting for SDK publishes.”

* Keep the “one proto source” and “Buf outputs land in SDK packages” parts as‑is (they already give you the immediate‑error property).

* Narrow the Docker/CI rules so:
  * **Dev/Tilt** images use workspace SDK packages.
  * **Prod** images (built by `cd-service-images`) also copy/vendor workspace SDK packages; published SDKs are validated only in the SDK product track.

---

## 5. Sanity check against Buf/BSR

This is actually very close to Buf’s own story:

* Buf wants one schema module, codegen from that, and then:

  * internal consumers often use generated stubs directly;
  * vendors sometimes wrap those stubs in higher‑level SDKs for external use. 

You’re basically saying:

* **Follow Buf for internal dev** (schema‑first, stubs generated into SDK packages, immediate feedback).
* **Add a vendor‑style AmeideClient SDK layer for external consumers**, but treat it as product packaging in the SDK publish track, not as a prerequisite for internal service builds.

That’s a sane, vendor‑aligned split.

---
