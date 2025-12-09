# 492 – Telepresence Reliability (Connect + Intercept + Verification)

**Status:** In Progress  
**Owner:** Platform / Developer Experience  
**Related backlogs:** [429-devcontainer-bootstrap.md](429-devcontainer-bootstrap.md), [434-unified-environment-naming.md](434-unified-environment-naming.md), [435-remote-first-development.md](435-remote-first-development.md), [491-auto-contexts.md](491-auto-contexts.md)

## Scope & Purpose

This backlog covers the full Telepresence experience for the remote-first dev workflow:

- **Connectivity** – bootstrapping kube contexts, invoking `telepresence connect`, and ensuring the traffic-manager can be reached from every DevContainer.
- **Intercepts** – keeping RBAC, workloads, and Tilt resources in sync so `telepresence intercept` works for `svc-*` targets without manual tweaks.
- **Verification & Troubleshooting** – automated health checks, logging, and runbooks so issues surface quickly in both developer workflows and CI. (See [492-telepresence-verification.md](492-telepresence-verification.md) for the step-by-step playbook.)

Teleporting the inner loop to the shared AKS cluster is now the only supported workflow (see backlog 435). This backlog tracks the verification tooling, observability, and troubleshooting required to keep Telepresence healthy for every developer session. It consolidates the improvements we made to `tools/dev/telepresence.sh` and enumerates the remaining gaps (intercept failures, RBAC regressions, session handling).

## Components & responsibilities

| Component | Location | Responsibility |
|-----------|----------|----------------|
| **DevContainer** | `.devcontainer/` | Installs Telepresence CLI (pinned v2.25.1), fetches AKS credentials, seeds environment variables. |
| **Helper script** | `tools/dev/telepresence.sh` | Opinionated wrapper for connect/intercept/status/quit/verify with logging and AZ context bootstrap. |
| **Traffic Manager** | `argocd/{env}-traffic-manager` (chart `sources/charts/third_party/telepresence/telepresence/2.25.1`) | Handles session negotiation, intercept orchestration, Traffic Agent injection. |
| **Tilt resources** | `Tiltfile` (`verify-telepresence`, `svc-*`) | Invokes helper script, starts dev servers, ensures Telepresence env files are sourced automatically. |
| **RBAC & GitOps** | `ameide-gitops` repo | ClusterRole/Role/Bindings for the traffic-manager (`pods/eviction`, serviceCIDRs), namespace defaults, Helm values. |

## Prerequisites & configuration

- Azure login (`az login --use-device-code`) and `kubelogin` conversion are required before Telepresence runs (wired in `.devcontainer/postCreate.sh`).
- Contexts `ameide-dev`, `ameide-staging`, `ameide-prod` all point at the same AKS API server; defaults are stored in `~/.devcontainer-mode.env`.
- Telepresence environment variables:
  - `TELEPRESENCE_CONTEXT`, `TELEPRESENCE_NAMESPACE` (default `ameide-dev`, override for staging/prod).
  - `TELEPRESENCE_VERIFY_WORKLOAD`, `TELEPRESENCE_VERIFY_PORT`, `TELEPRESENCE_SKIP_INTERCEPT`, `TELEPRESENCE_VERIFY_script`.
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

## Current workflow

1. **Context bootstrap** – `.devcontainer/postCreate.sh` and `tools/dev/bootstrap-contexts.sh` refresh `ameide-{dev,staging,prod}` contexts and set Telepresence defaults (backlog 491).
2. **Telepresence helper** – `tools/dev/telepresence.sh` wraps `connect`, `intercept`, `status`, `quit`, and `verify`.  
   - `verify` is now idempotent, timestamped, and auto-creates missing kube contexts via `az aks get-credentials`.  
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

## Recent improvements

- **RBAC alignment** – Telepresence Traffic Manager ClusterRole/Role templates now grant `create` on `pods/eviction`, matching the vendor guidance for v2.25.1 (mirrored in `sources/charts/third_party/telepresence/telepresence/2.25.1`). ArgoCD syncs `dev-traffic-manager`/`staging-traffic-manager` against the versioned chart path.
- **AKS role assignment automation** – `infra/terraform/azure` now creates the `Ameide Telepresence Developers` Entra ID group (object ID `f29f3bba-96e3-4faf-a6f5-6d8c595fe2d1`) and grants it the built-in **Azure Kubernetes Service RBAC Writer** role on the `ameide` cluster, replacing the manual “cluster-user” assignment attempts that failed earlier. Running `terraform -chdir=infra/terraform/azure apply` and a follow-up `plan` both return no drift.
- **Script resilience** – `tools/dev/telepresence.sh verify` now:
  - Logs timestamps for every phase.
  - Ensures `az aks get-credentials` is invoked automatically when `ameide-{env}` contexts are missing.
  - Runs `telepresence status` + `telepresence list` even when intercept fails, capturing diagnostics for Tilt/CI logs.
  - Supports environment overrides (`TELEPRESENCE_VERIFY_WORKLOAD`, `TELEPRESENCE_VERIFY_PORT`, `TELEPRESENCE_VERIFY_SCRIPT`, `TELEPRESENCE_SKIP_INTERCEPT`).

## Known issues (Dec 2025)

| Issue | Description | Impact | Next Actions |
|-------|-------------|--------|--------------|
| **NO-SESSION-1** | `telepresence intercept` returns `rpc error: code = Unavailable desc = no active session` even after a successful `connect`. | Tilt `svc-*` commands and the `verify` helper fail. | Pull `kubectl -n ameide-dev logs deploy/traffic-manager` to inspect session registration errors. Confirm traffic-manager and CLI share version v2.25.1; escalate to vendor if sessions still fail. |
| **CAP-NET-ADMIN** | `telepresence connect …` fails immediately with `connector.Connect: NewTunnelVIF: netlink.RuleAdd: operation not permitted` | DevContainer lacks the `CAP_NET_ADMIN` capability Telepresence needs to install routing rules, so even `sudo` fails. | Run the runbook from the host (or any shell with NET_ADMIN), or switch Telepresence to Docker mode once `docker` is available. Keep using `kubectl` + traffic-manager logs for validation when containerized tooling is constrained. |
| **RBAC fallback** | Namespaces using namespaced RBAC must also include the `pods/eviction` rule (added, but monitor). | Without it, intercept fails with RBAC errors. | Keep ArgoCD apps in sync; add regression test in the helper (detect RBAC failure vs session failure). |
| **Traffic-agent drift** | Some workloads (e.g., `kafka-entity-operator`) report `unable to engage (traffic-agent not installed): Unknown`. | Noise in Tilt CLI; may hide legitimate issues. | Document safe intercept targets (only `*-tilt` workloads) and ensure baseline workloads ignore Telepresence annotations (backlog 455). |

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
