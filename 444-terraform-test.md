# 444 – Terraform Local Workflow Test Plan

Document the end-to-end checks required to validate the hardened local Terraform workflow (Docker guard, k3d drift repair, cert-manager networking, and Argo reconcilers).

## 1. DevContainer Sanity

1. Rebuild the DevContainer so the docker-outside-of-docker feature reinstates `/var/run/docker-host.sock`.
2. Stop Docker Desktop (or the host daemon), reopen the DevContainer, and verify `postStartCommand` exits with:
   ```
   [devcontainer] Docker daemon is not reachable...
   ```
3. Restart Docker, reopen the container, and confirm `docker ps` succeeds.

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

## 3. Drift Repair Simulation

1. Mimic lost Terraform state:
   ```
   terraform -chdir=infra/terraform/local state rm module.k3d.k3d_cluster.this
   ```
2. Rerun `./infra/scripts/tf-local.sh apply`.
3. Expect the wrapper to log:
   ```
   [tf-local] Deleting unmanaged k3d cluster ameide
   ```
4. Verify the cluster is recreated and apply completes.

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
- `tf-local.sh` guards against missing Docker, recreates the shared network, and prunes orphaned k3d clusters automatically.
- Full destroy/apply loop converges cert-manager, Vault injector TLS, and Envoy TLS locally with the hostNetwork adjustments.
- ArgoCD Applications reach `Synced/Healthy` without manual port-forwarding commands.
