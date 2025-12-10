# 492 – Telepresence Verification Playbook

**Status:** Draft  
**Owner:** Platform / Developer Experience  
**Related backlogs:** [429-devcontainer-bootstrap.md](429-devcontainer-bootstrap.md), [434-unified-environment-naming.md](434-unified-environment-naming.md), [435-remote-first-development.md](435-remote-first-development.md), [491-auto-contexts.md](491-auto-contexts.md), [492-telepresence-reliability.md](492-telepresence-reliability.md)

## Problem

Remote-first development depends on Telepresence for every inner-loop. When it breaks we lose the ability to debug or ship changes. The `tools/dev/telepresence.sh verify` helper exists, but we lacked a durable playbook that:

- Describes the exact assertions the helper makes.
- Documents the environment variables that gate different modes (connect-only vs full intercept).
- Captures known failure signatures along with commands we expect engineers to run before escalating.
- Defines how Tilt and CI surface errors quickly.

## Objectives

1. **Deterministic verification** – `./tools/dev/telepresence.sh verify` should execute the same sequence locally, under Tilt (`tilt up verify-telepresence`), and in CI.
2. **Actionable logging** – Every phase is timestamped, commands retry with backoff, and failures automatically emit `telepresence status`/`telepresence list` to the console.
3. **Namespace-accurate intercepts** – The helper now re-connects with the requested namespace before every intercept (Telepresence v2.25 dropped the `--namespace` flag), so verification still covers staging/prod overrides instead of whichever context Telepresence cached previously.
4. **Failure matrices** – When verification fails, the backlog links to the remediation steps (context bootstrap, RBAC drift, traffic-manager bugs).

## Prerequisites

- **RBAC** – the developer must belong to the `Ameide Telepresence Developers` Entra group (or one of the principals passed through `developer_role_assignments`). Terraform/Bicep now create the group on demand and assign the built-in `Azure Kubernetes Service RBAC Writer` role to it, so membership is enough to call `kubectl`/Telepresence without bespoke role assignments.
- **Capabilities** – Telepresence requires `CAP_NET_ADMIN` to program routing rules. Codespaces-based GitOps containers do not expose this capability, so run the helper from your host (or a devcontainer with NET_ADMIN enabled) when you need a full intercept. Inside GitOps you can still collect diagnostics (`kubectl`, traffic-manager logs) and mark the run as “connectivity-only”.
- **Contexts** – the helper will auto-run `az aks get-credentials` the first time a context is missing, but `az login --use-device-code` must succeed beforehand; make sure `kubelogin` is already configured per [429-devcontainer-bootstrap.md](429-devcontainer-bootstrap.md#current-bootstrap-split-2025-12).

## Verification flow (helper script)

| Step | Command | Notes |
|------|---------|-------|
| 1 | `telepresence quit` | Ensures a clean slate; ignore errors. |
| 2 | `telepresence connect --context <ctx> --namespace <ns>` | Uses defaults from `TELEPRESENCE_CONTEXT/TELEPRESENCE_NAMESPACE` or overrides. |
| 3 | `telepresence status` | First health probe; failure increments the internal counter but does not abort. |
| 4 | `telepresence list` | Ensures workloads are visible before attempting intercepts. |
| 5 | `telepresence intercept <svc> --port <mapping> --env-file <tmp> -- bash -lc "$TELEPRESENCE_VERIFY_SCRIPT"` | Skipped when `TELEPRESENCE_SKIP_INTERCEPT=1`. The helper re-runs `telepresence connect --context <ctx> --namespace <ns>` first so the session inherits the correct namespace, and retries once before failing. |
| 6 | `telepresence leave <svc>` | Best-effort cleanup via a “safe leave” wrapper; “already removed” errors are treated as success. |
| 7 | `telepresence quit` | Disconnect once verification completes. |

All stdout/stderr gets prefixed with `[telepresence-helper] <timestamp>` so multiple runs are easy to correlate.

## Environment variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `TELEPRESENCE_CONTEXT` | `ameide-dev` | Cluster context to connect to; helper auto-runs `az aks get-credentials` if missing. |
| `TELEPRESENCE_NAMESPACE` | `ameide-dev` | Namespace passed to `telepresence connect`; intercepts reuse the active session. |
| `TELEPRESENCE_VERIFY_WORKLOAD` | `www-ameide` (falls back to `TELEPRESENCE_DEFAULT_WORKLOAD`) | Target workload used for the intercept test. |
| `TELEPRESENCE_VERIFY_PORT` | `3000:3000` (`DEFAULT_PORT`) | Local→remote port mapping for the verify intercept. |
| `TELEPRESENCE_VERIFY_SCRIPT` | `echo telepresence-verify && sleep 1` | Command executed inside the intercept shell to prove envs propagate. Override to run service-specific smoke tests. |
| `TELEPRESENCE_VERIFY_ENV_ROOT` | `${TELEPRESENCE_ENV_ROOT:-.telepresence-envs}` | Directory storing temporary env files for verify runs. |
| `TELEPRESENCE_SKIP_INTERCEPT` | unset | When set (any value), verification stops after status/list. |

### Organization slug invariants

`telepresence intercept` writes the intercepted pod’s ConfigMap-derived env vars into `.telepresence-envs/<workload>.env`. Because `services.www_ameide_platform.organization.defaultOrg` is the single GitOps source of truth, the verify helper should assert that this env file contains `AUTH_DEFAULT_ORG`, `NEXT_PUBLIC_DEFAULT_ORG`, and `WWW_AMEIDE_PLATFORM_ORG_ID` (all equal to the Keycloak slug such as `atlas`). Missing or mismatched values mean Helm values were not updated and should block verification until Argo syncs the fix.

## Failure signatures and remediation

| Signature | Helper output | Most likely cause | Actions |
|-----------|---------------|-------------------|---------|
| `kube context 'ameide-dev' not found` | Logged during `ensure_context_exists` | DevContainer lost AKS credentials | Run `tools/dev/bootstrap-contexts.sh` or `az aks get-credentials --resource-group Ameide --name ameide`. |
| `telepresence connect verification failed` | Connect step exits non-zero | Azure credentials expired / traffic-manager unreachable | `az login --use-device-code`, then re-run verify. Inspect `kubectl -n ameide-dev logs deploy/traffic-manager`. |
| `telepresence status command failed` + `list` fails | Telepresence daemon stuck | `telepresence quit --all` (host) then retry; escalate if daemon keeps crashing. |
| `intercept ... failed (context=X, namespace=Y)` | Intercept error block with `status/list` dumps | RBAC regression, workload missing traffic-agent, traffic-manager bug | `kubectl auth can-i --as <telepresence SA> create pods/eviction -n <ns>`, verify the `*-tilt` workload exists, capture traffic-manager logs. |
| `connector.CreateIntercept: ... no active session` + daemon logs `exec: "iptables": executable file not found in $PATH` | DevContainer doesn’t have `iptables`, so the root daemon can’t program DNS/routing | Install `iptables` (e.g., `sudo apt-get update && sudo apt-get install -y iptables`) inside the DevContainer; re-run verify once packages are present. |
| `telepresence intercept: error: unknown flag: --namespace` | Happens immediately after the CLI upgrade | Telepresence >=2.25 removed `--namespace` (and `--context`) flags from `intercept` | Update to the latest `tools/dev/telepresence.sh`, which now re-establishes the session via `telepresence connect --context ... --namespace ...` before starting the intercept. |
| `no active session` | `rpc error: code = Unavailable desc = no active session` | Known upstream bug tracked in reliability backlog | Collect logs, reference NO-SESSION-1 in [492-telepresence-reliability.md](492-telepresence-reliability.md#known-issues-dec-2025). |
| `connector.Connect: NewTunnelVIF: netlink.RuleAdd: operation not permitted` | Immediate failure during the connect step | Environment lacks `CAP_NET_ADMIN` (GitOps devcontainer) | Run the helper from a shell that exposes NET_ADMIN (host, privileged devcontainer). You can still capture `kubectl` + traffic-manager diagnostics inside GitOps, but mark the run as “connectivity only” and skip intercept assertions. |
| `ORG env vars missing` | Helper prints `AUTH_DEFAULT_ORG/NEXT_PUBLIC_DEFAULT_ORG/WWW_AMEIDE_PLATFORM_ORG_ID not found` | `services.www_ameide_platform.organization.defaultOrg` not set in Helm values | Update the relevant GitOps values (shared + per-env overlays) with the correct slug, let Argo sync, and rerun verify so Telepresence env files pick up the new ConfigMap. |

## Automation hooks

- **Tilt** – `tilt up verify-telepresence` invokes the helper with the same env vars used for service resources. The Tilt UI surfaces pass/fail alongside other local resources. Service resources themselves now run through `scripts/telepresence/intercept_service.sh` + `scripts/telepresence/run_service.sh`, so env-file logging and required-variable enforcement are consistent across verify and dev flows.
- **CI (future)** – Add a lightweight GitHub Actions job that runs `./tools/dev/telepresence.sh verify --context ameide-dev --namespace ameide-dev` with `TELEPRESENCE_SKIP_INTERCEPT=1` to catch kube context drift without requiring intercept permissions in CI.
- **Observability** – Because every log line already includes ISO timestamps, we can redirect helper output to Loki/Grafana once we emit JSON lines (see reliability backlog for telemetry follow-ups).

## Acceptance criteria

1. Documentation reflects the namespace-aware intercept changes (Dec 2025) and the upstream removal of the `--namespace` flag.
2. Engineers can follow this playbook to triage Telepresence failures without pinging platform immediately.
3. Tilt resource docs link here so future onboarding references a single source of truth.
4. Reliability backlog references this file for verification details instead of re-describing the same flow.

## References

- `tools/dev/telepresence.sh` (verify implementation)
- [429-devcontainer-bootstrap.md](429-devcontainer-bootstrap.md#current-bootstrap-split-2025-12)
- [491-auto-contexts.md](491-auto-contexts.md)
- [docs/dev-workflows/telepresence.md](../docs/dev-workflows/telepresence.md)
