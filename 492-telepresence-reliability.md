# 492 – Telepresence Reliability (Connect + Intercept + Verification)

**Status:** Deprecated (historical; superseded by 650 suite)  
**Owner:** Platform / Developer Experience  
**Related backlogs:** [435-remote-first-development.md](435-remote-first-development.md), [434-unified-environment-naming.md](434-unified-environment-naming.md), [491-auto-contexts.md](491-auto-contexts.md), [492-telepresence-verification.md](492-telepresence-verification.md), [581-parallel-ai-devcontainers.md](581-parallel-ai-devcontainers.md)

> Superseded by the internal-only, Coder-based model:
> - `backlog/650-agentic-coding-overview.md`
> - `backlog/653-agentic-coding-test-automation.md`
> - `backlog/654-agentic-coding-cli-surface.md`
>
> This document is preserved for historical context only; Telepresence is not a platform dependency in the 650 suite.

## Scope & Purpose

This backlog covers the full Telepresence experience for the remote-first dev workflow:

- **Connectivity** – bootstrapping kube contexts, invoking `telepresence connect`, and ensuring the traffic-manager can be reached from every DevContainer.
- **Intercepts** – keeping RBAC and workloads in sync so Telepresence intercepts are reliable for the CLI-owned inner loop.
- **Verification & Troubleshooting** – automated health checks, logging, and runbooks so issues surface quickly in both developer workflows and CI. (See [492-telepresence-verification.md](492-telepresence-verification.md) for the step-by-step playbook.)

Remote-first on the shared AKS cluster (backlog 435) remains the default workflow. Local k3d (`ameide-local`) is also supported for local GitOps convergence, but requires extra cluster-side networking allowances for admission webhooks (see “Local k3d notes” below). This backlog tracks the verification tooling, observability, and troubleshooting required to keep Telepresence healthy for every developer session.

## Components & responsibilities

| Component | Location | Responsibility |
|-----------|----------|----------------|
| **DevContainer** | `.devcontainer/` | Installs Telepresence CLI (pinned v2.25.1) plus prerequisites (iptables/sshfs). |
| **Ameide CLI** | `packages/ameide_core_cli/` | Canonical dev entrypoints that own Telepresence connect/intercept/cleanup with stable URLs (`ameide dev inner-loop up|down|verify`, `ameide dev inner-loop-test`). |
| **Traffic Manager** | `argocd/{env}-traffic-manager` (chart `sources/charts/third_party/telepresence/telepresence/2.25.1`) | Handles session negotiation, intercept orchestration, Traffic Agent injection. |
| **RBAC & GitOps** | `ameide-gitops` repo | ClusterRole/Role/Bindings for the traffic-manager (`pods/eviction`, serviceCIDRs), namespace defaults, Helm values. |

## Prerequisites & configuration

- Target selection is explicit via `AMEIDE_TELEPRESENCE_TARGET` (`ameide-aks` or `ameide-local`).
- AKS: Azure login (`az login --use-device-code`) and `kubelogin` conversion are required before Telepresence runs; `tools/dev/bootstrap-contexts.sh --target ameide-aks` establishes kube contexts.
- Local: `tools/dev/bootstrap-contexts.sh --target ameide-local` establishes the local kube context.
- Telepresence environment variables:
  - `TELEPRESENCE_CONTEXT`, `TELEPRESENCE_NAMESPACE` (default `ameide-dev`, override for staging/prod).
  - Phase 3 / inner-loop commands read `AGENT_CI_*` and `AMEIDE_INNER_LOOP_*` env vars (advanced; normally not set manually).

## Architecture snapshot

```
┌────────────────────────────────────────────────────────────────────────────┐
│  DevContainer (VS Code)                                                    │
│  ├── tools/dev/bootstrap-contexts.sh (backlog 491)                         │
│  └── ameide CLI (inner-loop up/verify/test)                                │
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
- **Traffic Agent**: Injected into workloads during intercept (GitOps policy controls whether injection is allowed in each environment).
- **Registry**: Images are published to `ghcr.io/ameideio/...` and referenced by immutable digest; Telepresence intercepts do not rely on local registries.

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
2. **Verification** – `ameide dev inner-loop verify` validates Telepresence header-filtered routing against the stable ingress URL (`https://platform.local.ameide.io`) and fails fast with actionable diagnostics.

### Developer workflow

1. `./ameide dev inner-loop verify`
2. `./ameide dev inner-loop up` (hot reload + intercept)
3. When done: `./ameide dev inner-loop down`

### Cluster-side configuration

- Traffic Manager installs into each environment namespace; ArgoCD Applications reference the pinned chart version.
- RBAC requirements:
  - `create` on `pods/eviction`
  - `get/list/watch` on workloads/services
  - `patch/update` on workloads (for Traffic Agent injection)
  - `list` `networking.k8s.io/servicecidrs` (Kubernetes 1.33+)
- In local/dev, only explicit intercept targets should be interceptable. In staging/prod, keep workloads non-interceptable by default.

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
- If you rely on Telepresence mutating pod templates for injection, ArgoCD self-heal can revert those mutations. Prefer dedicated intercept targets that are explicitly annotated for injection in GitOps; otherwise use the bootstrap toggles only as a stopgap.

**What’s left to do (GitOps-side):**
- Make the `allow-control-plane-webhooks` policy part of the local environment GitOps base so Argo owns it (no manual/bootstrap patching).
- Decide (and document) the explicit intercept targets per environment, and ensure they are annotated for injection (`telepresence.io/inject-traffic-agent: enabled`) without requiring Telepresence to patch pod templates.
- Decide whether to:
  - Extend `deny-cross-environment` (local only) with an `ipBlock` allow for the control-plane CIDR, or
  - Keep `deny-cross-environment` as-is and add explicit allow policies for webhook servers (Telepresence + Vault injector).

## Update (2026-01-06): GitOps-owned webhook allowlist (local)

Local clusters hit recurring apiserver proxy `502` errors when calling admission webhooks (Telepresence `agent-injector` and Vault injector) because the k3s apiserver runs as a host process and does not match namespace-based allowlists.

**Remediation**: the local environment now renders GitOps-owned NetworkPolicies from `foundation-namespaces` to allow the control-plane source CIDRs to reach:
- Telepresence traffic-manager webhook port `8443`
- Vault agent-injector pod port `8080` (Service `443` → targetPort `8080`)

**Why earlier remediations weren’t effective**:
- In k3d, webhook calls can originate from the node IP range (e.g. `172.19.0.0/16`), not only the pod CIDR (`10.42.0.0/16`). Allowing only the pod CIDR can still result in `connect: connection refused` (policy REJECT) even though the webhook pods are healthy.
- Bootstrap/manual patches don’t survive GitOps resync/recreate; Argo must own these policies to prevent drift.

This keeps the fix persistent across resyncs/recreates and reduces cascading leader-election loss/timeouts in controllers under load.

## Update (2026-01-11): Namespaced traffic-manager scoping (per environment)

We hit a real-world failure mode where Telepresence was configured for **namespaced RBAC** (`managerRbac.namespaced: true`) but the traffic-manager still attempted to compute its managed namespaces from a **cluster-wide namespace selector**.

**Symptoms**
- Argo apps `local-traffic-manager` / `{env}-traffic-manager` show `Progressing` and the traffic-manager Deployment can CrashLoop.
- `kubectl -n <env> logs deploy/traffic-manager` shows:
  - `unable to determine a static list of namespaces from given namespace selector`
  - `error listing namespaces: namespaces is forbidden ... cannot list resource "namespaces" ... at the cluster scope`

**Root cause**
- The traffic-manager reads `namespace-selector.yaml` from `ConfigMap/traffic-manager`.
- The vendor default selector (`NotIn kube-system,kube-node-lease`) requires the controller to list namespaces to resolve the final set. With namespaced RBAC, listing namespaces is forbidden, so the traffic-manager can fail fast.

**GitOps fix (permanent, non-operational)**
- When `managerRbac.namespaced: true`, render `namespace-selector.yaml` as an explicit allowlist:
  - `operator: In`
  - `values: [ameide-<env>]`
- This is implemented by templating `sources/charts/third_party/telepresence/telepresence/2.25.1/templates/trafficManager-configmap.yaml` from `.Values.managerRbac.namespaces`.
- Each environment declares its own namespace list (single namespace):
  - `sources/values/env/local/platform/traffic-manager.yaml`
  - `sources/values/env/dev/platform/traffic-manager.yaml`
  - `sources/values/env/staging/platform/traffic-manager.yaml`
  - `sources/values/env/production/platform/traffic-manager.yaml`

**Expected behavior after fix**
- Traffic-manager logs may still print `Listing namespaces is not permitted` (debug/info), but it must proceed with `Using fixed set of namespaces [ameide-<env>]` and stay `Ready`.

## Recent improvements

- **RBAC alignment** – Telepresence Traffic Manager ClusterRole/Role templates now grant `create` on `pods/eviction`, matching the vendor guidance for v2.25.1 (mirrored in `sources/charts/third_party/telepresence/telepresence/2.25.1`). ArgoCD syncs `dev-traffic-manager`/`staging-traffic-manager` against the versioned chart path.
- **AKS role assignment automation** – `infra/terraform/azure` now creates the `Ameide Telepresence Developers` Entra ID group (object ID `f29f3bba-96e3-4faf-a6f5-6d8c595fe2d1`) and grants it the built-in **Azure Kubernetes Service RBAC Writer** role on the `ameide` cluster, replacing the manual “cluster-user” assignment attempts that failed earlier. Running `terraform -chdir=infra/terraform/azure apply` and a follow-up `plan` both return no drift.
- **CLI resilience** – `ameide dev inner-loop*` and `ameide dev inner-loop-test`:
  - Always clean Telepresence state first (`telepresence quit -s`, stale socket cleanup).
  - Require HTTP header filtered intercepts (Telepresence/traffic-manager >= 2.25).
  - Write logs/artifacts under `artifacts/agent-ci/` for incident correlation.
- **Namespace alignment** – Telepresence CLI v2.25 removed the `--namespace` flag from `telepresence intercept`. The CLI uses `telepresence connect --context ... --namespace ... -- <cmd>` so intercepts inherit the correct namespace without relying on removed flags.
- **Linux prerequisites** – DevContainers running Linux now get a fast-fail when `iptables` is missing. The helper checks for the binary before running Telepresence so we stop chasing `no active session` errors that were ultimately caused by the root daemon being unable to install DNS rules.
- **DevContainer packages** – `.devcontainer/Dockerfile` bakes `iptables` and `sshfs` into the base image and `.devcontainer/postCreate.sh` re-installs them when older images are reused. This keeps Telepresence DNS/routing healthy and removes the noisy “remote volume mounts are disabled” warning.

## Known issues (Dec 2025)

| Issue | Description | Impact | Next Actions |
|-------|-------------|--------|--------------|
| **NO-SESSION-1** | `telepresence intercept` returns `rpc error: code = Unavailable desc = no active session` even after a successful `connect`. | `ameide dev inner-loop*` and `ameide dev inner-loop verify` fail. | Pull `kubectl -n ameide-dev logs deploy/traffic-manager` to inspect session registration errors. Confirm traffic-manager and CLI share version v2.25.1; escalate to vendor if sessions still fail. |
| **CAP-NET-ADMIN** | `telepresence connect …` fails immediately with `connector.Connect: NewTunnelVIF: netlink.RuleAdd: operation not permitted` | DevContainer lacks the `CAP_NET_ADMIN` capability Telepresence needs to install routing rules, so even `sudo` fails. | Run the runbook from the host (or any shell with NET_ADMIN), or switch Telepresence to Docker mode once `docker` is available. Keep using `kubectl` + traffic-manager logs for validation when containerized tooling is constrained. |
| **STALE-SOCKET-1** | `telepresence connect` fails with `dial to socket /var/run/telepresence-daemon.socket ... permission denied` and mentions an unresponsive socket. | Telepresence cannot start because the root daemon socket is stale (common after container restarts). | Run `sudo rm -f /var/run/telepresence-daemon.socket` and retry; the `ameide` CLI also attempts best-effort socket cleanup via `sudo -n` before connecting. |
| **TP-PORT-1** | Intercepts cause readiness/liveness probes to hang (especially when `curl 127.0.0.1` works but `curl <podIP>` times out). This can show up as ArgoCD **Synced** but **Progressing**. | Argo health flaps; intercepts appear “randomly broken”. | Ensure intercepted Services use **named `targetPort`** (not numeric) so Telepresence can avoid the iptables redirect path; see the “named ports” contract in the GitOps refactor section below. |
| **RBAC fallback** | Namespaces using namespaced RBAC must also include the `pods/eviction` rule (added, but monitor). | Without it, intercept fails with RBAC errors. | Keep ArgoCD apps in sync; add regression test in the helper (detect RBAC failure vs session failure). |
| **Traffic-agent drift** | Some workloads (e.g., `kafka-entity-operator`) report `unable to engage (traffic-agent not installed): Unknown`. | Noise in Tilt CLI; may hide legitimate issues. | Document safe intercept targets (local/dev where injection is allowed) and keep staging/prod injection disabled by default. |
| **LINUX-IPTABLES** | `connector.CreateIntercept: ... no active session` paired with daemon logs `exec: "iptables": executable file not found in $PATH`. | Root daemon cannot configure DNS/routing inside DevContainer, so sessions immediately die. | Install `iptables` in the DevContainer (`sudo apt-get install -y iptables`) and rerun `ameide dev inner-loop verify`. |
| **CLIENT-CONFIG** | Traffic-manager logs print `client.yaml: ... unknown object member name "connectionTTL"` on every connect. | Cosmetic warning; intercepts still work, but we need confirmation from Telepresence maintainers. | `kubectl -n ameide-dev logs deploy/traffic-manager --since=30m | grep connectionTTL` captures the error. We confirmed our chart only sets `CLIENT_CONNECTION_TTL` via env vars (`kubectl -n ameide-dev get deploy traffic-manager -o jsonpath='{range ... }'`) and there is no `connectionTTL` string in this repo. Share the log + env list with the Telepresence team for follow-up. |

## GitOps refactor: Telepresence-safe baseline

Telepresence can inject Traffic Agents without changing GitOps manifests, which is why ArgoCD can remain **Synced** while workloads become **Progressing**. GitOps still controls two high-leverage knobs that align with vendor guidance and keep baseline workloads stable:

1. **Services use named `targetPort`** (avoid numeric `targetPort`)
   - Contract: container port is named (e.g. `http`), Service port is named `http`, and Service `targetPort` is the **string** `http`.
   - We enforce this in app charts so Telepresence can avoid the iptables init-container path that can interfere with Pod-IP probes.
   - Caveat: avoid intercepting **headless Services** (`clusterIP: None`) as Telepresence may still need the iptables init-container path for those targets.
2. **Traffic-agent injection is opt-in (GitOps posture)**
   - Contract: the traffic-manager uses `agentInjector.injectPolicy=WhenEnabled`, so workloads only receive traffic-agent sidecars when they are explicitly annotated.
   - Baseline workloads may still set `telepresence.io/inject-traffic-agent: disabled` as an explicit guardrail (even though injection is already opt-in).
   - Local + dev enable injection only on explicit intercept targets (workloads annotated with `telepresence.io/inject-traffic-agent: enabled`). The CLI verifies/enforces this for the `www-ameide-platform` target before intercepting.
   - Staging/prod keep injection disabled unless there is an explicit, reviewed reason to enable it.
   - GitOps implementation:
     - `sources/values/_shared/platform/platform-telepresence.yaml` sets `agentInjector.injectPolicy: WhenEnabled`.
     - `sources/values/_shared/platform/traffic-manager.yaml` mirrors the same setting (legacy shim; also pins `agentInjector.webhook.port: 8443`).

### ArgoCD enforcement gotcha: `ignoreDifferences` + `RespectIgnoreDifferences`

If ArgoCD is configured to ignore pod template annotations globally (e.g. `resource.customizations.ignoreDifferences.apps_Deployment` includes `/spec/template/metadata/annotations`) and Applications set `RespectIgnoreDifferences=true`, Argo will also skip applying changes to those fields during sync. That breaks the “baseline injection disabled” contract because the vendor-required annotation never lands on existing workloads.

**GitOps contract:** do not ignore the entire template annotations map. If Argo needs to ignore drift, narrow it to specific keys, not `/spec/template/metadata/annotations`.

**Where this is enforced:** `ameide-gitops` sets empty ignore rules in `sources/values/common/argocd.yaml` so template annotations remain enforceable.

### Expected noise (not a Telepresence failure)

- **Redis Sentinel auth warning:** `default user does not require a password, but a password was supplied` can occur if Sentinel itself is unauthenticated while Redis master requires auth. Treat as noise unless you also see Redis connection failures (`NOAUTH`, `ECONNREFUSED`, timeouts).
- **Next.js `NODE_ENV` warning during intercept:** if you intercept a production-like workload but run `next dev`, Next may warn about inconsistent env. Ensure your target environment sets `NODE_ENV=development` (or unset it) when running dev servers.

## Telepresence verification checklist

The quick-hit list below mirrors the detailed playbook in [492-telepresence-verification.md](492-telepresence-verification.md):

1. `ameide dev inner-loop verify` – validates Telepresence + header-filtered intercept routing against `https://platform.local.ameide.io`.
2. `telepresence status` – confirm daemon + traffic-manager connection.
3. `kubectl -n ameide-dev logs deploy/traffic-manager` – capture errors when intercept/session fails.
4. `argocd app get dev-traffic-manager` – ensure the app synced the RBAC fixes from `ameide-gitops`.

## Next steps (connect + intercept + observability)

1. **Diagnose NO-SESSION-1** – Collect traffic-manager logs during the helper’s intercept attempt; verify session IDs, look for authentication errors, and share with the Telepresence maintainers if necessary.
2. **Telemetry hooks** – Extend the `ameide` CLI to emit structured logs (JSON) for future integration with CI and support dashboards.
3. **Automated regression test** – Consider a lightweight GitHub Actions step that runs `ameide dev inner-loop verify` against a controlled namespace to catch context drift or RBAC regressions after infra changes.

## References

- [434-unified-environment-naming.md](434-unified-environment-naming.md) – Single AKS cluster, namespace naming rules, migration status.
- [435-remote-first-development.md](435-remote-first-development.md) – Remote-first architecture and Telepresence rationale.
- [491-auto-contexts.md](491-auto-contexts.md) – Auto context script details; referenced by `postCreate.sh`.
- [492-telepresence-verification.md](492-telepresence-verification.md) – Full verification flow and failure remediation steps.
- [581-parallel-ai-devcontainers.md](581-parallel-ai-devcontainers.md) – Parallel slot conventions (intercept naming, collision avoidance).
- `telepresence` (vendor CLI) – for debugging (`telepresence status`, `telepresence gather-logs`).
