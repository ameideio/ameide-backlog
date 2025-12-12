# 444 – Terraform Local Workflow Test Plan

Document the end-to-end checks required to validate the hardened local Terraform workflow (Docker guard, k3d drift repair, cert-manager networking, and Argo reconcilers).

## 1. DevContainer Sanity

1. Rebuild the DevContainer so the docker-outside-of-docker feature reinstates `/var/run/docker-host.sock`.
2. Stop Docker Desktop (or the host daemon), reopen the DevContainer, and verify `postStartCommand` exits with:
   ```
   [devcontainer] Docker daemon is not reachable...
   ```
3. Restart Docker, reopen the container, and confirm `docker ps` succeeds.
4. Run `terraform version` and `argocd version` inside the container; they should report `Terraform v1.14.2` and `argocd: v3.2.1` (snapshot of the current stable release) with no manual install steps.
5. Inspect `.devcontainer/devcontainer.json` or `docker inspect` for the running container and confirm the `--add-host=host.docker.internal:host-gateway` runArg is present; the host mapping is required for the pinned k3d API endpoint.

## 2. Wrapper Preflight & Apply

1. Ensure a clean slate: `k3d cluster delete ameide || true`.
2. Run `./infra/scripts/tf-local.sh apply -auto-approve`.
3. Confirm the wrapper logs:
  ```
  [tf-local] Initializing Terraform modules
  [tf-local] Running terraform apply -auto-approve
  ```
4. After apply:
   - `lsof -i :8443` → shows the detached `kubectl port-forward`.
   - `kubectl -n argocd get applications local-foundation-cert-manager local-platform-cert-manager-config local-foundation-vault-webhook-certs`.
   - `kubectl config view -o jsonpath='{.clusters[?(@.name=="k3d-ameide")].cluster.server}'` returns `https://host.docker.internal:6550` (no `insecure-skip-tls-verify` entries should exist).

## 3. Drift Repair Simulation

1. With the cluster still running from the previous step, delete the local state artifacts:
   ```
   rm -f infra/terraform/local/terraform.tfstate*
   ```
2. Rerun `./infra/scripts/tf-local.sh apply`.
3. Expect the wrapper to log:
   ```
   [tf-local] Deleting unmanaged k3d cluster ameide
   [tf-local] Deleted local Terraform state; next apply will fully recreate the stack
   ```
4. Verify the cluster is recreated, the ArgoCD install re-runs (no missing CRD errors), and apply completes.

## 4. Destroy/Recreate Loop

1. `./infra/scripts/tf-local.sh destroy -auto-approve`
2. `./infra/scripts/tf-local.sh apply -auto-approve`
3. Confirm no manual Docker cleanup is required between the runs.

## 5. ArgoCD & TLS Smoke Checks

1. Wait for foundation apps:
   ```bash
   kubectl -n argocd app wait local-foundation-cert-manager --health --timeout 5m
   kubectl -n argocd app wait local-platform-cert-manager-config --health --timeout 5m
   kubectl -n argocd app wait local-foundation-vault-webhook-certs --health --timeout 5m
   ```
2. Inspect Issuers/Certificates:
   ```bash
   kubectl -n ameide-local get issuer,certificate
   kubectl -n ameide-local get secret vault-core-local-injector-tls
   ```
3. Ensure the cert-manager webhook endpoints exist:
   ```bash
   kubectl -n ameide-local get endpoints cert-manager-local-webhook
   ```
4. Validate the Vault/ExternalSecrets chain:
   ```bash
   kubectl -n ameide-local describe secretstore ameide-vault | grep -i ready
   kubectl -n ameide-local get externalsecret ghcr-pull-sync
   kubectl -n ameide-local get secret ghcr-pull
   ```
   `.env`/`.env.local` must define `GHCR_USERNAME` and `GHCR_TOKEN`; `infra/scripts/seed-local-secrets.sh` copies those values into `vault-bootstrap-local-secrets` so Vault bootstrap can make them available to ExternalSecrets.

## 6. Networking Assertions

1. Confirm the local webhook binding:
   ```bash
   kubectl -n ameide-local get pod -l app.kubernetes.io/name=cert-manager-local-webhook \
     -o jsonpath='{.items[0].spec.hostNetwork}'
   kubectl -n ameide-local get pod -l app.kubernetes.io/name=cert-manager-local-webhook \
     -o jsonpath='{.items[0].spec.containers[0].ports[0].containerPort}'
   ```
   Expect `true` and `10260`.
2. Optionally curl the webhook service from another pod to ensure 10260 is reachable.

## 7. Documentation Cross-Check

1. Verify this plan and the updated status banners are referenced from `backlog/444-terraform.md`.
2. Link the test results in sprint notes or the runbook so future engineers can trace the verification history.

## Success Criteria

- Docker access enforcement works (container refuses to start when Docker is unavailable).
- `tf-local.sh` guards against missing Docker, recreates the shared network, deletes unmanaged clusters, and wipes the local tfstate when drift is detected so the next apply is deterministic.
- Full destroy/apply loop converges cert-manager, Vault injector TLS, and Envoy TLS locally with the hostNetwork adjustments, and kubeconfig always points at `https://host.docker.internal:6550`.
- ArgoCD Applications reach `Synced/Healthy` without manual port-forwarding commands.
