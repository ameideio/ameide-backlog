# 391 – BuildKit registry token handling for dev inner-loop images

## Problem
- Tilt/`docker build` for `services/www_ameide` fails when pulling private dependencies (`ghcr.io` tarballs or `npm.pkg.github.com` packages) because the build context lacks `PKG_GITHUB_COM_TOKEN`/`GHCR_TOKEN`.
- Prior Dockerfile patterns relied on `ARG`/`ENV` to pass tokens, which leaked secrets into layers and triggered lint warnings.
- Tokens were mistakenly considered runtime/Helm concerns even though Next.js pods do **not** need those credentials once the image is built.

## Vendor guidance (Docker + GitHub Packages)
- Treat registry/PAT credentials as **build-time secrets** only.
- Use BuildKit secret mounts (`RUN --mount=type=secret,...`) for install steps so values exist only inside that layer.
- Source the secret from local env/CI secrets and forward via `docker build --secret id=...`.
- Do not bake PATs into layers, `.npmrc` inside the image, or Helm manifests.

## Adopted pattern (2024‑??)
1. **Local dev setup**
   ```bash
   export PKG_GITHUB_COM_TOKEN=ghp_...
   export GHCR_TOKEN=$PKG_GITHUB_COM_TOKEN  # or separate PAT
   export GHCR_USERNAME=<github-handle>
   ```
   - Keep these in a gitignored `.env`.
   - `tilt`/scripts read and pass them to `docker build` via `--secret`.
2. **Dockerfile.dev**
   ```dockerfile
   RUN --mount=type=secret,id=pkg_github_token,required=false \
       --mount=type=secret,id=ghcr_token,required=false \
       sh -c 'TOKEN="${PKG_GITHUB_COM_TOKEN:-}"
       if [ -z "$TOKEN" ] && [ -f /run/secrets/p... ]; then TOKEN=$(cat /run/secrets/p...); fi
       export PKG_GITHUB_COM_TOKEN="$TOKEN"
       pnpm install --no-frozen-lockfile --ignore-scripts'
   ```
   - Token lives only for the install step; no `ARG PKG_GITHUB_COM_TOKEN` remains.
   - Applies to both dependency installs (pre/post source copy).
3. **Tiltfile / CI**
   - Wrap custom builds with `--secret id=pkg_github_token,env=PKG_GITHUB_COM_TOKEN --secret id=ghcr_token,env=GHCR_TOKEN`.
   - CI uses repo/org secrets (`PKG_GITHUB_COM_TOKEN`, `GHCR_TOKEN`) and mirrors the same invocation.

## What this unlocks
- Consistent inner-loop: `tilt up ameide/www-ameide` no longer fails on private npm/GHCR pulls.
- No Helm/Argo wiring: runtime pods remain secret-free for this use case.
- Compliance: secrets are absent from `docker history`, SBOMs, or push layers.

## Follow-ups / Guardrails
- [ ] Audit other service Dockerfiles for lingering token `ARG`/`ENV` usage; port to BuildKit secret mounts.
- [ ] Update developer onboarding docs (`README`, `backlog/388-ameideio-sdks.md`) to link to this entry for private registry installs.
- [ ] Add CI lint to block committed `.npmrc` files with inline tokens/logins.
- [ ] Extend Tilt helper to error loudly if required secrets are missing instead of surfacing opaque `401 Unauthorized`.
