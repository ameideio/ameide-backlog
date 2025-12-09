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
2. **Actionable logging** – Every phase is timestamped, and failures automatically emit `telepresence status`/`telepresence list` to the console.
3. **Namespace-accurate intercepts** – The helper always passes the namespace explicitly so verification covers staging/prod overrides instead of whichever context Telepresence cached previously.
4. **Failure matrices** – When verification fails, the backlog links to the remediation steps (context bootstrap, RBAC drift, traffic-manager bugs).

## Verification flow (helper script)

| Step | Command | Notes |
|------|---------|-------|
| 1 | `telepresence quit` | Ensures a clean slate; ignore errors. |
| 2 | `telepresence connect --context <ctx> --namespace <ns>` | Uses defaults from `TELEPRESENCE_CONTEXT/TELEPRESENCE_NAMESPACE` or overrides. |
| 3 | `telepresence status` | First health probe; failure increments the internal counter but does not abort. |
| 4 | `telepresence list` | Ensures workloads are visible before attempting intercepts. |
| 5 | `telepresence intercept <svc> --context <ctx> --namespace <ns> --port <mapping> --env-file <tmp> -- bash -lc "$TELEPRESENCE_VERIFY_SCRIPT"` | Skipped when `TELEPRESENCE_SKIP_INTERCEPT=1`. Namespace flag is required even if the kube context default differs. |
| 6 | `telepresence leave <svc>` | Best-effort cleanup; errors logged as warnings. |
| 7 | `telepresence quit` | Disconnect once verification completes. |

All stdout/stderr gets prefixed with `[telepresence-helper] <timestamp>` so multiple runs are easy to correlate.

## Environment variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `TELEPRESENCE_CONTEXT` | `ameide-dev` | Cluster context to connect to. |
| `TELEPRESENCE_NAMESPACE` | `ameide-dev` | Namespace passed to connect/intercept. |
| `TELEPRESENCE_VERIFY_WORKLOAD` | `www-ameide` (falls back to `TELEPRESENCE_DEFAULT_WORKLOAD`) | Target workload used for the intercept test. |
| `TELEPRESENCE_VERIFY_PORT` | `3000:3000` (`DEFAULT_PORT`) | Local→remote port mapping for the verify intercept. |
| `TELEPRESENCE_VERIFY_SCRIPT` | `echo telepresence-verify && sleep 1` | Command executed inside the intercept shell to prove envs propagate. Override to run service-specific smoke tests. |
| `TELEPRESENCE_VERIFY_ENV_ROOT` | `${TELEPRESENCE_ENV_ROOT:-.telepresence-envs}` | Directory storing temporary env files for verify runs. |
| `TELEPRESENCE_SKIP_INTERCEPT` | unset | When set (any value), verification stops after status/list. |

## Failure signatures and remediation

| Signature | Helper output | Most likely cause | Actions |
|-----------|---------------|-------------------|---------|
| `kube context 'ameide-dev' not found` | Logged during `ensure_context_exists` | DevContainer lost AKS credentials | Run `tools/dev/bootstrap-contexts.sh` or `az aks get-credentials --resource-group Ameide --name ameide`. |
| `telepresence connect verification failed` | Connect step exits non-zero | Azure credentials expired / traffic-manager unreachable | `az login --use-device-code`, then re-run verify. Inspect `kubectl -n ameide-dev logs deploy/traffic-manager`. |
| `telepresence status command failed` + `list` fails | Telepresence daemon stuck | `telepresence quit --all` (host) then retry; escalate if daemon keeps crashing. |
| `intercept ... failed (context=X, namespace=Y)` | Intercept error block with `status/list` dumps | RBAC regression, workload missing traffic-agent, traffic-manager bug | `kubectl auth can-i --as <telepresence SA> create pods/eviction -n <ns>`, verify the `*-tilt` workload exists, capture traffic-manager logs. |
| `no active session` | `rpc error: code = Unavailable desc = no active session` | Known upstream bug tracked in reliability backlog | Collect logs, reference NO-SESSION-1 in [492-telepresence-reliability.md](492-telepresence-reliability.md#known-issues-dec-2025). |

## Automation hooks

- **Tilt** – `tilt up verify-telepresence` invokes the helper with the same env vars used for service resources. The Tilt UI surfaces pass/fail alongside other local resources.
- **CI (future)** – Add a lightweight GitHub Actions job that runs `./tools/dev/telepresence.sh verify --context ameide-dev --namespace ameide-dev` with `TELEPRESENCE_SKIP_INTERCEPT=1` to catch kube context drift without requiring intercept permissions in CI.
- **Observability** – Because every log line already includes ISO timestamps, we can redirect helper output to Loki/Grafana once we emit JSON lines (see reliability backlog for telemetry follow-ups).

## Acceptance criteria

1. Documentation reflects the namespace-aware intercept changes (Dec 2025).
2. Engineers can follow this playbook to triage Telepresence failures without pinging platform immediately.
3. Tilt resource docs link here so future onboarding references a single source of truth.
4. Reliability backlog references this file for verification details instead of re-describing the same flow.

## References

- `tools/dev/telepresence.sh` (verify implementation)
- [429-devcontainer-bootstrap.md](429-devcontainer-bootstrap.md#current-bootstrap-split-2025-12)
- [491-auto-contexts.md](491-auto-contexts.md)
- [docs/dev-workflows/telepresence.md](../docs/dev-workflows/telepresence.md)
