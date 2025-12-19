# 492 – Telepresence Reliability (Connect + Intercept + Verification)

**Status:** In Progress  
**Owner:** Platform / Developer Experience  
**Related backlogs:** [429-devcontainer-bootstrap.md](429-devcontainer-bootstrap.md), [434-unified-environment-naming.md](434-unified-environment-naming.md), [435-remote-first-development.md](435-remote-first-development.md), [491-auto-contexts.md](491-auto-contexts.md)

## Scope & Purpose

This backlog covers the full Telepresence experience for the remote-first dev workflow:

- **Connectivity** – bootstrapping kube contexts, invoking `telepresence connect`, and ensuring the traffic-manager can be reached from every DevContainer.
- **Intercepts** – keeping RBAC, workloads, and Tilt resources in sync so `telepresence intercept` works for `svc-*` targets without manual tweaks.
- **Verification & Troubleshooting** – automated health checks, logging, and runbooks so issues surface quickly in both developer workflows and CI. (See [492-telepresence-verification.md](492-telepresence-verification.md) for the step-by-step playbook.)

Remote-first on the shared AKS cluster (backlog 435) remains the default workflow. Local k3d (`ameide-local`) is also supported for local GitOps convergence, but requires extra cluster-side networking allowances for admission webhooks (see “Local k3d notes” below). This backlog tracks the verification tooling, observability, and troubleshooting required to keep Telepresence healthy for every developer session.

## Components & responsibilities

| Component | Location | Responsibility |
|-----------|----------|----------------|
| **DevContainer** | `.devcontainer/` | Installs Telepresence CLI (pinned v2.25.1), plus prerequisites (iptables/sshfs) and Tilt defaults. |
| **Helper script** | `tools/dev/telepresence.sh` | Opinionated wrapper for connect/intercept/status/quit/verify with logging and target-aware bootstrap. |
| **Traffic Manager** | `argocd/{env}-traffic-manager` (chart `sources/charts/third_party/telepresence/telepresence/2.25.1`) | Handles session negotiation, intercept orchestration, Traffic Agent injection. |
| **Tilt resources** | `Tiltfile` (`verify-telepresence`, `svc-*`) | Invokes helper script, starts dev servers, ensures Telepresence env files are sourced automatically. |
| **RBAC & GitOps** | `ameide-gitops` repo | ClusterRole/Role/Bindings for the traffic-manager (`pods/eviction`, serviceCIDRs), namespace defaults, Helm values. |

## Prerequisites & configuration

- Target selection is explicit via `TILT_TELEPRESENCE_TARGET` (`ameide-aks` or `ameide-local`).
- AKS: Azure login (`az login --use-device-code`) and `kubelogin` conversion are required before Telepresence runs; `tools/dev/bootstrap-contexts.sh --target ameide-aks` establishes kube contexts.
- Local: `tools/dev/bootstrap-contexts.sh --target ameide-local` establishes the local kube context.
- Telepresence environment variables:
  - `TELEPRESENCE_CONTEXT`, `TELEPRESENCE_NAMESPACE` (default `ameide-dev`, override for staging/prod).
  - `TELEPRESENCE_VERIFY_WORKLOAD`, `TELEPRESENCE_VERIFY_PORT`, `TELEPRESENCE_SKIP_INTERCEPT`, `TELEPRESENCE_VERIFY_SCRIPT`.
- Tilt sets `DEV_REMOTE_CONTEXT`, `TILT_REMOTE=1`, `TELEPRESENCE_ENV_ROOT=.telepresence-envs`.

## Architecture snapshot

```
┌────────────────────────────────────────────────────────────────────────────┐
│  DevContainer (VS Code)                                                    │
│  ├── tools/dev/bootstrap-contexts.sh (backlog 491)                         │
│  ├── tools/dev/telepresence.sh (connect/intercept/verify)                  │
│  └── Tilt (verify-telepresence + svc-* wrappers)                           │
│                   │                                                        │
│                   ▼ telepresence connect                                   │
└────────────────────────────────────────────────────────────────────────────┘
                    │
                    ▼
        ┌──────────────────────────┐
        │ AKS Cluster: ameide      │
        │  ├─ Namespace: ameide-dev│
        │  ├─ Namespace: ameide-stg│
        │  ├─ Namespace: ameide-prod
        │  ├─ Namespace: argocd    │
        │  └─ telepresence-traffic-manager (chart v2.25.1)                   │
        └──────────────────────────┘
```

- **Traffic Manager**: Deploys in each environment namespace via `argocd/{env}-traffic-manager`. Chart pinned at `sources/charts/third_party/telepresence/telepresence/2.25.1`.
- **Traffic Agent**: Injected into workloads during intercept (Tilt targets `apps-*-tilt` releases to avoid Argo drift).
- **Registry**: Tilt pushes images to `ghcr.io/ameideio/...` when `TILT_REMOTE=1`.

### RBAC verification & documentation

- **Source of truth** – RBAC templates live under `sources/charts/third_party/telepresence/telepresence/2.25.1/templates/trafficManagerRbac/`. Both the namespace- and cluster-scoped files grant `pods/eviction.create`, `services.create`, and `servicecidrs.get/list/watch`, so we no longer need ad-hoc patches when Telepresence upgrades requirements.
- **Argo ownership** – `dev|staging|prod-traffic-manager` Applications render those templates directly, so a `kubectl get gitopsapp dev-traffic-manager -o jsonpath='{.status.sync.status}'` returning `Synced` is the gate that RBAC landed.
- **Verification commands** – To prove RBAC exists and is wired to the service account, run:

  ```bash
  kubectl -n ameide-dev get role traffic-manager -o yaml | rg -C2 pods/eviction
  kubectl auth can-i --as system:serviceaccount:ameide-dev:traffic-manager create pods/eviction -n ameide-dev
  kubectl auth can-i --as system:serviceaccount:ameide-dev:traffic-manager get servicecidrs --all-namespaces
  ```

  These commands are referenced from the verification backlog so the “is this documented somewhere?” question always has a single answer. If any command fails, upgrade the chart copy under `sources/charts/third_party/telepresence/...` and let Argo sync the fix instead of patching roles in place.

## Current workflow

1. **Context bootstrap** – `.devcontainer/postCreate.sh` and `tools/dev/bootstrap-contexts.sh` refresh `ameide-{dev,staging,prod}` contexts and set Telepresence defaults (backlog 491).
2. **Telepresence helper** – `tools/dev/telepresence.sh` wraps `connect`, `intercept`, `status`, `quit`, and `verify`.  
   - `verify` is idempotent, timestamped, and target-aware. Missing contexts are handled via `tools/dev/bootstrap-contexts.sh` (auto-bootstrap for `ameide-aks`, hard error for `ameide-local`).  
   - By default it runs a full intercept using `TELEPRESENCE_VERIFY_WORKLOAD` (defaults to `www-ameide`) and logs `telepresence status/list` automatically when failures occur. Set `TELEPRESENCE_SKIP_INTERCEPT=1` for connectivity-only checks.
3. **Tilt integration** – The Tiltfile exposes a `verify-telepresence` local resource so developers (or CI) can trigger the scripted health check (backlog 429 § Telepresence mode).

### Developer workflow

1. `./tools/dev/telepresence.sh connect --context ameide-dev --namespace ameide-dev`
2. `tilt up svc-www-...` or manual `telepresence intercept ... --env-file .telepresence-envs/<service>.env -- bash -lc '<dev command>'`
3. `source .telepresence-envs/<service>.env && pnpm/go/uv run dev`
4. When done, `./tools/dev/telepresence.sh quit` (or `telepresence leave <service>` to stop intercept only).

### Cluster-side configuration

- Traffic Manager installs into each environment namespace; ArgoCD Applications reference the pinned chart version.
- RBAC requirements:
  - `create` on `pods/eviction`
  - `get/list/watch` on workloads/services
  - `patch/update` on workloads (for Traffic Agent injection)
  - `list` `networking.k8s.io/servicecidrs` (Kubernetes 1.33+)
- Workloads targeted by Tilt use the `*-tilt` releases to avoid Argo drift and keep Telepresence annotations isolated.

### Security & networking

- Telepresence uses Azure AD authentication via `kubelogin` and inherits the developer’s `kubectl` permissions.
- Connect commands run inside the DevContainer; no host-level VPN is needed.
- Traffic Manager pods run in the same namespace as the workloads (per environment) and respect the namespace’s network policies.

## Local k3d notes (ameide-local)

On k3d/k3s the Kubernetes API server runs as a host process (not a pod). With a default-deny ingress policy like `NetworkPolicy/deny-cross-environment`, admission webhooks can fail because the control-plane source IP does not match any namespaceSelector allowlist. Telepresence traffic-agent injection is an admission webhook (`agent-injector-webhook-*`), so intercepts can fail with:

- Telepresence: `connector.CreateIntercept ... DeadlineExceeded`
- k3s server logs: `Failed calling webhook ... agent-injector-*.getambassador.io ... dial tcp <podIP>:8443: connect: connection refused` (often surfaced as a 502 via the apiserver proxy)

**Stopgap (bootstrap-side):**
- `tools/dev/bootstrap-contexts.sh --target ameide-local` applies a narrow `NetworkPolicy/allow-control-plane-webhooks` to allow the k3d/k3s pod CIDR (default `10.42.0.0/16`, override via `AMEIDE_LOCAL_CONTROL_PLANE_WEBHOOK_CIDR`) to reach the Telepresence webhook port (`8443`).
- The same bootstrap script can patch `argocd-cm` ignoreDifferences for `Deployment`/`StatefulSet` pod templates so ArgoCD self-heal doesn’t instantly revert Telepresence traffic-agent injection (toggle via `AMEIDE_LOCAL_CONFIGURE_ARGO_FOR_TELEPRESENCE=0`).

**What’s left to do (GitOps-side):**
- Make the `allow-control-plane-webhooks` policy part of the local environment GitOps base so Argo owns it (no manual/bootstrap patching).
- Decide how local GitOps should treat Telepresence traffic-agent injection drift:
  - Local-only `argocd-cm` ignoreDifferences (preferred over bootstrap patching), or
  - Ensure all intercept targets are `*-tilt` workloads not reconciled by Argo, or
  - Disable self-heal for the specific Argo apps that back intercept targets (local only).
- Decide whether to:
  - Extend `deny-cross-environment` (local only) with an `ipBlock` allow for the control-plane CIDR, or
  - Keep `deny-cross-environment` as-is and add explicit allow policies for webhook servers (Telepresence + Vault injector).

## Recent improvements

- **RBAC alignment** – Telepresence Traffic Manager ClusterRole/Role templates now grant `create` on `pods/eviction`, matching the vendor guidance for v2.25.1 (mirrored in `sources/charts/third_party/telepresence/telepresence/2.25.1`). ArgoCD syncs `dev-traffic-manager`/`staging-traffic-manager` against the versioned chart path.
- **AKS role assignment automation** – `infra/terraform/azure` now creates the `Ameide Telepresence Developers` Entra ID group (object ID `f29f3bba-96e3-4faf-a6f5-6d8c595fe2d1`) and grants it the built-in **Azure Kubernetes Service RBAC Writer** role on the `ameide` cluster, replacing the manual “cluster-user” assignment attempts that failed earlier. Running `terraform -chdir=infra/terraform/azure apply` and a follow-up `plan` both return no drift.
- **Script resilience** – `tools/dev/telepresence.sh` now:
  - Logs timestamps for every phase and wraps critical commands with retry/backoff.
  - Ensures `az aks get-credentials` is invoked automatically when `ameide-{env}` contexts are missing and cleans up active sessions via traps.
  - Runs `telepresence status` + `telepresence list` even when intercept fails, capturing diagnostics for Tilt/CI logs.
  - Supports environment overrides (`TELEPRESENCE_VERIFY_WORKLOAD`, `TELEPRESENCE_VERIFY_PORT`, `TELEPRESENCE_VERIFY_SCRIPT`, `TELEPRESENCE_SKIP_INTERCEPT`).
- **Service runner scripts** – `scripts/telepresence/intercept_service.sh` and `scripts/telepresence/run_service.sh` wrap every Tilt `svc-*` resource. They source `.telepresence-envs/<service>.env`, log the number of exported variables, enforce required envs (`--required VAR`), and print the values being used before the dev server starts. Intercepts fail fast (with actionable logs) when GitOps hasn’t provided the expected environment variables (e.g., `NEXT_PUBLIC_DEFAULT_ORG`).
- **Namespace alignment** – Telepresence CLI v2.25 removed the `--namespace` flag from `telepresence intercept`. The helper now re-establishes the requested context/namespace before every intercept so verification/Tilt flows keep working even after the upstream change.
- **Linux prerequisites** – DevContainers running Linux now get a fast-fail when `iptables` is missing. The helper checks for the binary before running Telepresence so we stop chasing `no active session` errors that were ultimately caused by the root daemon being unable to install DNS rules.
- **DevContainer packages** – `.devcontainer/Dockerfile` bakes `iptables` and `sshfs` into the base image and `.devcontainer/postCreate.sh` re-installs them when older images are reused. This keeps Telepresence DNS/routing healthy and removes the noisy “remote volume mounts are disabled” warning.

## Known issues (Dec 2025)

| Issue | Description | Impact | Next Actions |
|-------|-------------|--------|--------------|
| **NO-SESSION-1** | `telepresence intercept` returns `rpc error: code = Unavailable desc = no active session` even after a successful `connect`. | Tilt `svc-*` commands and the `verify` helper fail. | Pull `kubectl -n ameide-dev logs deploy/traffic-manager` to inspect session registration errors. Confirm traffic-manager and CLI share version v2.25.1; escalate to vendor if sessions still fail. |
| **CAP-NET-ADMIN** | `telepresence connect …` fails immediately with `connector.Connect: NewTunnelVIF: netlink.RuleAdd: operation not permitted` | DevContainer lacks the `CAP_NET_ADMIN` capability Telepresence needs to install routing rules, so even `sudo` fails. | Run the runbook from the host (or any shell with NET_ADMIN), or switch Telepresence to Docker mode once `docker` is available. Keep using `kubectl` + traffic-manager logs for validation when containerized tooling is constrained. |
| **TP-PORT-1** | Intercepts cause readiness/liveness probes to hang (especially when `curl 127.0.0.1` works but `curl <podIP>` times out). This can show up as ArgoCD **Synced** but **Progressing**. | Argo health flaps; intercepts appear “randomly broken”. | Ensure intercepted Services use **named `targetPort`** (not numeric) so Telepresence can avoid the iptables redirect path; see the “named ports” contract in the GitOps refactor section below. |
| **RBAC fallback** | Namespaces using namespaced RBAC must also include the `pods/eviction` rule (added, but monitor). | Without it, intercept fails with RBAC errors. | Keep ArgoCD apps in sync; add regression test in the helper (detect RBAC failure vs session failure). |
| **Traffic-agent drift** | Some workloads (e.g., `kafka-entity-operator`) report `unable to engage (traffic-agent not installed): Unknown`. | Noise in Tilt CLI; may hide legitimate issues. | Document safe intercept targets (only `*-tilt` workloads) and ensure baseline workloads ignore Telepresence annotations (backlog 455). |
| **LINUX-IPTABLES** | `connector.CreateIntercept: ... no active session` paired with daemon logs `exec: "iptables": executable file not found in $PATH`. | Root daemon cannot configure DNS/routing inside DevContainer, so sessions immediately die. | Install the `iptables` package inside the DevContainer (`sudo apt-get install -y iptables`) and re-run `./tools/dev/telepresence.sh verify`. The helper now checks for the binary up front. |
| **CLIENT-CONFIG** | Traffic-manager logs print `client.yaml: ... unknown object member name "connectionTTL"` on every connect. | Cosmetic warning; intercepts still work, but we need confirmation from Telepresence maintainers. | `kubectl -n ameide-dev logs deploy/traffic-manager --since=30m | grep connectionTTL` captures the error. We confirmed our chart only sets `CLIENT_CONNECTION_TTL` via env vars (`kubectl -n ameide-dev get deploy traffic-manager -o jsonpath='{range ... }'`) and there is no `connectionTTL` string in this repo. Share the log + env list with the Telepresence team for follow-up. |

## GitOps refactor: Telepresence-safe baseline

Telepresence can inject Traffic Agents without changing GitOps manifests, which is why ArgoCD can remain **Synced** while workloads become **Progressing**. GitOps still controls two high-leverage knobs that align with vendor guidance and keep baseline workloads stable:

1. **Services use named `targetPort`** (avoid numeric `targetPort`)
   - Contract: container port is named (e.g. `http`), Service port is named `http`, and Service `targetPort` is the **string** `http`.
   - We enforce this in app charts so Telepresence can avoid the iptables init-container path that can interfere with Pod-IP probes.
2. **Baseline workloads disable injection by default**
   - Contract: Argo-managed releases set `telepresence.io/inject-traffic-agent: disabled` on the pod template.
   - Tilt-only releases (`*-tilt`) omit the annotation so Telepresence can inject on-demand during intercepts.

## Telepresence verification checklist

The quick-hit list below mirrors the detailed playbook in [492-telepresence-verification.md](492-telepresence-verification.md):

1. `./tools/dev/telepresence.sh verify` (optional env overrides). It should auto-create contexts if missing, connect, list workloads, and attempt intercept using the namespace-aware flags.
2. `telepresence intercept www-ameide --port 3000:3000 --env-file .telepresence-envs/www-ameide.env -- bash -lc 'pnpm --filter www-ameide dev'` – manual reproduction of Tilt’s service wrapper when deeper debugging is required.
3. `kubectl -n ameide-dev logs deploy/traffic-manager` – capture errors when the helper reports “no active session”.
4. `argocd app get dev-traffic-manager` – ensure the app synced the RBAC fixes from `ameide-gitops`.

## Next steps (connect + intercept + observability)

1. **Diagnose NO-SESSION-1** – Collect traffic-manager logs during the helper’s intercept attempt; verify session IDs, look for authentication errors, and share with the Telepresence maintainers if necessary.
2. **Telemetry hooks** – Extend `tools/dev/telepresence.sh` to emit structured logs (JSON) for future integration with CI and support dashboards.
3. **Automated regression test** – Consider a lightweight GitHub Actions step that runs the helper (with `TELEPRESENCE_SKIP_INTERCEPT=1`) to catch context drift or RBAC regressions after infra changes.

## References

- [429-devcontainer-bootstrap.md](429-devcontainer-bootstrap.md) – Telepresence mode description and helper wiring in Tilt.
- [434-unified-environment-naming.md](434-unified-environment-naming.md) – Single AKS cluster, namespace naming rules, migration status.
- [435-remote-first-development.md](435-remote-first-development.md) – Remote-first architecture and Telepresence rationale.
- [491-auto-contexts.md](491-auto-contexts.md) – Auto context script details; referenced by `postCreate.sh`.
- [492-telepresence-verification.md](492-telepresence-verification.md) – Full verification flow and failure remediation steps.
- `tools/dev/telepresence.sh` – Helper script (see git history for `telepresence verification` commits).
