# 314 – Full-Stack Integration Pipeline

## Summary
- Stand up a disposable k3d cluster per pipeline run.
- Build and load all service images into the cluster registry.
- Deploy platform infrastructure via Helmfile and wait for readiness gates.
- Execute the full integration suite inside the freshly provisioned cluster.
- Enforce hardened registry + package routing (`docker.io/ameide`, `packages.*`) before workloads start.
- Collect logs, JUnit, and diagnostics before tearing the cluster down.

## Goals
- Guarantee isolation between runs (no shared namespace state, no leaked PVCs).
- Mirror local `tilt up` behaviour so developers can reproduce failures quickly.
- Provide deterministic scripts that CI can invoke without manual steps.
- Surface rich failure artifacts (pod logs, events, Helm status) for debugging.
- Enforce registry and SDK routing guardrails introduced in backlogs 336/337 so CI matches developer environments.

## Non-Goals
- Creating mocked substitutes for external dependencies; the pipeline must use the real stack.
- Replacing lighter PR checks; this flow is a heavier gate intended for nightly or release branches.
- Automating cloud cluster creation (focus remains on k3d for speed and parity).

## Proposed Pipeline Flow
1. **Bootstrap**
   - Install Docker Buildx, k3d, Helm, and Tilt (if not already present in the runner image).
   - Create a k3d cluster with an embedded registry (`k3d cluster create ameide-ci --registry-create`).
2. **Registry & DNS Preflight**
   - Run `scripts/infra/helm-ops.sh preflight dns docker.io` to confirm the Envoy/CoreDNS routing is active.
   - Wait for the `registry-mirror` DaemonSet to report ready (e.g., `kubectl wait daemonset/registry-mirror -n ameide --for=condition=RegistryMirrorReady --timeout=5m`).
   - Fail fast if the hardened path is unavailable; the pipeline must halt rather than attempting alternate hosts or manual mirror scripts.
3. **Image Build**
   - Run `tilt build` or a Makefile wrapper to build every service image with caching enabled.
   - Push/tag images against `docker.io/ameide` so downstream Helm deploys can rely on digest-first values.
   - Use `k3d image import` (or registry push) so the cluster has the latest images.
4. **Infrastructure Deploy**
   - Execute `helmfile -e ci sync` to deploy shared dependencies (Postgres, Kafka, Keycloak, etc.).
   - Run `scripts/vault/ensure-local-secrets.py` (or a CI wrapper) once layer 15 completes to seed Vault with deterministic fixtures and refresh the ESO Kubernetes auth role.
   - Reuse `infra/kubernetes/scripts/infra-health-check.sh` with retries to gate progress, explicitly waiting on the secrets/certificates layer so External Secrets Operator reports Ready.
   - Fail the pipeline immediately if `kubectl get externalsecret -A` contains `Ready=False` after the health check; application deploys must not proceed without materialised secrets.
5. **Application Deploy**
   - Apply application charts or rely on `tilt ci --apply` to deploy services.
   - Confirm Verdaccio/npm/Go/PyPI mirrors (`packages.*`) are reachable so SDK imports follow the hardening rules (see backlog 336).
   - Double-check per-service secrets by re-running the ESO readiness sweep (or `infra/kubernetes/scripts/infra-health-check.sh --layers 15,40,45`). Abort if any workload ExternalSecret regresses.
   - Wait for readiness via `tilt wait --for=condition=Ready` or equivalent kubectl checks.
6. **Integration Execution**
   - Trigger `pnpm test:integration --all` (or matrixed `--filter <service>` runs) targeting the temporary cluster.
   - Stream logs in real time while tests run; abort early on infrastructure errors.
7. **Artifacts & Teardown**
   - Collect JUnit XML, summarized reports, `kubectl get events`, and Helm status into the CI artifact store.
   - Export pod logs for failed jobs.
   - Delete the k3d cluster (`k3d cluster delete ameide-ci`) to free resources.

## Tooling Deliverables
- `scripts/ci/setup-k3d.sh` – creates the cluster, configures registry, and installs supporting CRDs.
- `scripts/ci/build-images.sh` – canonical entry point for building all service images with caching hints.
- `scripts/ci/full-stack-integration.sh` – orchestrates Helmfile sync, Vault seeding, ExternalSecret validation, readiness checks, integration runner, and artifact capture.
- Vault bootstrap helper (wrapper around `scripts/vault/ensure-local-secrets.py`) that emits diagnostics on auth misconfiguration and surfaces missing fixture keys.
- Registry/package preflight helpers that wrap `scripts/infra/helm-ops.sh preflight dns`, wait for `registry-mirror`, and verify Verdaccio `packages:*` health endpoints before application deploys.
- Documentation update in `tests/integration/README.md` covering CI reproduction steps for developers.
- GitHub Actions (or alternative CI) workflows invoking the scripts with caching configured (BuildKit cache, npm cache, pnpm store).

## Open Questions
- How do we seed non-deterministic data (Keycloak users, Temporal namespaces) per run? Prefer idempotent Helm hooks or init jobs.
- Should the pipeline support parallel test batches (matrix) or run serially to simplify artifact capture?
- What resource limits should be applied to keep k3d memory/CPU within CI runner constraints?
- Do we need optional gates (smoke-only mode vs. full suite) toggled via workflows inputs?
