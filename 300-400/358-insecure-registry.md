# 358 – k3d Registry TLS Mismatch

## Summary

- **Symptom:** The `agents` deployment in the local `ameide` k3d cluster looped on `ImagePullBackOff` because the control plane attempted to pull `k3d-dev-reg:5000/ameide/agents:tilt-0e75bc57aa9591e1` over HTTPS. The embedded registry only serves HTTP, so containerd returned `http: server gave HTTP response to HTTPS client`.
- **Impact:** Any Tilt-built workload pushed to `k3d-dev-reg:5000` failed to roll out; developers could not iterate on the Agents service locally.
- **Trigger:** Tilt rebuilt the Agents image (`tilt-0e75bc57aa9591e1`) and deployed `helm_resource('agents')`. The cluster lacked an insecure-registry entry for `k3d-dev-reg`, causing the pull failure during the subsequent rollout.
- **Context:** This registry path underpins the Tilt/Helm workflow in `backlog/343-tilt-helm-north-start-v3.md`, where `image.repository`/`image.tag` are injected with the `k3d-dev-reg:5000/ameide/...` prefix defined in `Tiltfile:120-210`. Keeping the registry reachable is therefore a prerequisite for the entire v3 inner loop.
- **Observation (Nov 6 2025):** Even with mirrors configured, containerd still falls back to the implicit HTTPS endpoint (`https://k3d-dev-reg:5000/v2`). K3s exposes `--disable-default-registry-endpoint` to turn off that fallback; without the flag, every pull hits HTTPS first and fails.
- **Current approach (v7, Nov 6 2025):** Provision a mirrors-only registry plus the `--disable-default-registry-endpoint` extra arg on every k3s node so containerd never retries HTTPS. Devcontainer bootstrap validates this flow end-to-end.

## Root Cause Analysis

- k3d normally injects a `registries.yaml` into cluster nodes when `--registry-create` or `--registry-config` is used. Our declarative cluster config (`infra/docker/local/k3d.yaml`) did not define either option, so the nodes started with default containerd settings.
- containerd treats every mirror as HTTPS unless explicitly marked insecure. With no mirror config present, it attempted TLS to `k3d-dev-reg:5000`.
- The registry container (`registry:2`) bundled with k3d only serves plaintext HTTP; TLS will always fail without a terminating proxy or certificates.

## Timeline & Attempts

1. **Initial fix (2025‑11‑05)**  
   Added a `registries` block with mirrors plus a `configs` stanza setting `tls.insecure_skip_verify: true` for `k3d-dev-reg:5000`. At the time this appeared to work, but k3s rendered `/var/lib/rancher/k3s/agent/etc/containerd/certs.d/k3d-dev-reg:5000/hosts.toml` with:
   ```toml
   server = "https://k3d-dev-reg:5000/v2"
   skip_verify = true
   ```
   Containerd still issued an HTTPS HEAD before falling back to the HTTP mirror, so pulls continued to fail with `http: server gave HTTP response to HTTPS client`.

2. **Manual node patching (2025‑11‑06)**  
   We tried rewriting `hosts.toml` on each node to force `server = "http://k3d-dev-reg:5000/v2"`. This worked only until the next reconcile; k3s regenerates the files on every start, so the change was not durable.

3. **Permissions hiccup**  
   `scripts/infra/ensure-k3d-cluster.sh` writes its config hash to `/home/vscode/.config/ameide/k3d-config.sha`. The devcontainer ships that directory as `root:root`, causing the reconcile to fail after recreating the cluster. Fix with `sudo chown -R vscode:vscode /home/vscode/.config/ameide` before rerunning the script.

4. **Script hardening (2025‑11‑06)**  
   The reconcile script now auto-recovers from that scenario: it first tries the user-owned config directory, attempts a `sudo chown` if needed, and falls back to `/workspace/.k3d-state` when it cannot obtain write access. The fallback is git-ignored.

5. **Bootstrap guardrail (2025‑11‑06)**  
   `.devcontainer/postCreateCommand.sh` now curls `http://localhost:5000/v2/_catalog` (overridable via `K3D_REGISTRY_HEALTHCHECK_URL`) right after cluster reconciliation. Bootstrap aborts early if the registry cannot be reached, preventing Tilt from starting in a broken state.

6. **Approach v7 – mirrors + fallback disabled (2025‑11‑06)**  
  The devcontainer now reconciles a mirrors-only registry configuration *and* passes `--disable-default-registry-endpoint` to every k3s node so containerd never retries HTTPS. The relevant YAML snippet:
  ```yaml
  mirrors:
    "k3d-dev-reg:5000":
      endpoint:
        - "http://k3d-dev-reg:5000"
    "localhost:5000":
      endpoint:
        - "http://k3d-dev-reg:5000"
  ```
  This combination prevents containerd from synthesizing an HTTPS default endpoint, so all pulls use the configured HTTP mirrors.

## Git History Reference

- `ad2915a7` (2025‑11‑03): Introduced `infra/docker/local/k3d.yaml` with a k3d-managed registry and an HTTP mirror for `registry.dev.ameide.io:5000`.
- `b3f0906d` (2025‑11‑05): Added the `configs` stanza with `tls.insecureSkipVerify` for the registry mirrors—this change reintroduced the HTTPS probe described above.
- `923d387c` (2025‑11‑06): Switched mirrors to `k3d-dev-reg:5000` / `localhost:5000` and renamed the TLS flag to `insecure_skip_verify`, keeping the HTTPS-first behavior.
- Working tree (2025‑11‑06): Removed the `configs` stanza, leaving only mirrors so k3s emits an HTTP-only `hosts.toml`. Hardened `scripts/infra/ensure-k3d-cluster.sh` to auto-select a writable state directory, and added a devcontainer registry healthcheck.

## Current Resolution

- Keep the mirrors-only registry config in `infra/docker/local/k3d.yaml:5-17`.
- Append `options.k3s.extraArgs` with `--disable-default-registry-endpoint` for both server and agent nodes (see `infra/docker/local/k3d.yaml:35-41`) so containerd never retries the HTTPS default.
- Run `scripts/infra/ensure-k3d-cluster.sh` so every node picks up the new config (fix ownership on `/home/vscode/.config/ameide` first if needed).
- Restart Tilt or `kubectl rollout restart` affected deployments; pods now pull from the dev registry without TLS errors.
- Tilt/Tiltfile expectations for the registry (automatic tagging, Helm image injection) live in `backlog/343-tilt-helm-north-start-v3.md`; that document assumes the mirrors + health checks described here are in place.

### Verification (mirrors-only + fallback disabled)

- Devcontainer bootstrap exercises the new strategy end-to-end. The `verify_registry_host_and_cluster_flow` section in `.devcontainer/postCreateCommand.sh` announces “Registry strategy: mirrors-only + --disable-default-registry-endpoint,” pushes BusyBox to `localhost:5000/...`, and deploys a pod that pulls via `k3d-dev-reg:5000/...`.
- Because HTTPS fallback is disabled, any failure now means the HTTP mirror itself is unreachable (versus containerd silently retrying HTTPS). The bootstrap summary records the status under “registry e2e”.
- Manual spot checks are still documented for bare-metal environments:
  1. `docker pull busybox:latest && docker tag busybox:latest localhost:5000/test/busybox:dev && docker push localhost:5000/test/busybox:dev`
  2. `kubectl run registry-check --restart=Never -n ameide --image=k3d-dev-reg:5000/test/busybox:dev --command -- true`
  3. Clean up with `kubectl delete pod registry-check -n ameide --ignore-not-found`
- For verbose logs, inspect `/workspace/.devcontainer-setup.log` around “Registry host/container probe”; it now captures the host push, namespace creation, pod status, and any describe/log output if the probe fails.

## Follow-ups

1. Confirm every Tilt-managed chart uses the `AMEIDE_REGISTRY` default (`k3d-dev-reg:5000/ameide`) so the mirrored endpoint is exercised consistently.
2. Document the reconciliation step in developer onboarding (DevContainer post-create already calls the ensure script, but bare-metal setups should be reminded to re-run it after config edits).
3. Optional hardening: publish the registry over HTTPS with a dev-trust root CA to catch accidental plaintext usage in CI or future clusters.
