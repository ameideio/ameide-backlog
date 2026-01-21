# 444 – Terraform Infrastructure (v2: CI-First, Multi-Cloud, Local k3d Stays)

**Created**: 2026-01-10  
**Status**: Draft (proposed direction)

This document describes the next iteration of Ameide infrastructure provisioning:

- **Cloud (AKS/EKS) is CI-owned**: apply/destroy happen only in GitHub Actions, with deterministic verification gates.
- **Local development stays supported** via **Terraform-managed k3d** (`infra/terraform/local`) for offline/inner-loop workflows.
- **Multi-cloud becomes a first-class contract**: same GitOps desired state, consistent outputs, cloud-specific plumbing.

Related:
- `backlog/444-terraform.md` (current Terraform reference)
- `backlog/435-remote-first-development.md` (remote-first workflow)
- `backlog/519-gitops-fleet-policy-hardening.md` (outputs → globals policy)
- `backlog/451-secrets-management.md` (secrets flow)

---

## Goals

1. **Predictable rollouts**: one writer per cloud target, no hidden manual steps, verifiers gate success.
2. **Idempotent infrastructure**: apply/destroy are reproducible; lock discipline prevents parallel writers.
3. **GitOps owns desired state**: Kubernetes workloads are delivered via ArgoCD/ApplicationSets; no hand-applied manifests.
4. **Local k3d stays viable**: developers can stand up a local cluster via Terraform without any cloud access.
5. **Cloud portability**: adding AWS (EKS) should not require rewriting the GitOps model; only the infrastructure adapters differ.

Non-goals:
- Making local and cloud identical at the infrastructure layer (they only need to be GitOps-compatible).
- Storing secret values in GitHub.

---

## High-Level Architecture

### Single source of truth

- **GitOps desired state**: `argocd/`, `environments/`, `sources/`
- **Infrastructure**: Terraform workspaces under `infra/terraform/`
- **Runtime facts** (addresses, identity references, endpoints): derived from Terraform outputs and published as **CI-generated inputs** (see “Runtime Facts” below).

### Runtime facts (don’t create a second source of truth)

v2 direction: separate “human desired state” from “machine-generated runtime facts”.

- **Human repo**: this repo (ApplicationSets, charts, values defaults).
- **Runtime facts repo/branch**: CI-owned output (fully regenerated on apply; never hand-edited).
- **ArgoCD composition**: Applications layer the runtime facts over desired state using **multiple sources** (so generated values don’t require commits to `main` of the human repo).

### Split ownership model

| Target | Provisioner | Who runs it | Expected use |
|---|---|---|---|
| `local` (k3d) | Terraform (`infra/terraform/local`) | developers | inner-loop / offline |
| `azure` (AKS) | Terraform (`infra/terraform/azure`) | CI only | shared remote-first cluster |
| `aws` (EKS) | Terraform (`infra/terraform/aws`, future) | CI only | shared remote-first cluster |

---

## CI-Only Cloud Deploy (the contract)

### “Only CI applies” policy

Cloud targets must satisfy:
- **Apply**: CI can create the cluster + prerequisites, bootstrap GitOps, and pass verifiers without manual intervention.
- **Destroy**: CI can tear down everything Terraform created, including handling Kubernetes drain blockers.
- **Locks/concurrency**: a single concurrency lane per target prevents parallel writers.

The current Azure workflows already embody this direction:
- `.github/workflows/terraform-azure-apply.yaml`
- `.github/workflows/terraform-azure-destroy.yaml`
- `.github/workflows/terraform-azure-plan.yaml`
- `.github/workflows/terraform-azure-force-unlock.yaml`

Local developer machines should run only:
- `terraform validate` / `terraform plan -backend=false` for review
- never `terraform apply` for cloud workspaces

### CI-only mutation policy (no manual fixes)

If we adopt “CI is the only responsible actor that changes configuration”, this must be true in practice:

- **Cloud infrastructure is changed only by CI** (Terraform apply/destroy workflows).
- **Runtime facts are written only by CI** (the `ameide-runtime-facts` repo/branch is CI-generated output; never hand-edited).
- **Cluster desired state is changed only via Git commits** (ArgoCD reconciles; humans do not apply/patch live resources to “fix” drift).

Hard rules (cloud clusters: AKS/EKS):

- No manual `kubectl apply/patch/delete` against shared cloud clusters for platform services (Gateway/Envoy, Argo apps, Coder, etc.).
- No manual Azure Portal changes for DNS record sets / load balancer IP association.
- No manual commits to the runtime-facts repo/branch.
- Read-only inspection is always allowed (`kubectl get/describe/logs`, Azure `az ... show/list`, etc.).

Rationale:

- Manual changes create a second source of truth; the next CI run (or ArgoCD reconciliation) will overwrite them, and “it worked yesterday” becomes untraceable.

Break-glass policy:

- If an emergency action is required (state lock, stuck destroy, broken deployment), it must be executed via the dedicated CI workflows (e.g. `terraform-azure-force-unlock`) and then followed by a normal CI apply that restores convergence.
- Record the event as a Git change (issue/backlog note) so the next investigation is not tribal knowledge.

### Deterministic verification gates

For cloud deploys, “Terraform succeeded” is not the finish line. CI should gate on:
- Public IP association to AKS/EKS load balancers
- TLS certificates ready
- Envoy/Gateway listeners exposing `443`
- DNS resolution correctness for critical hostnames (e.g. `*.dev.ameide.io` including `coder.dev.ameide.io`)
- SSO verifiers passing

Current scripts:
- `infra/scripts/verify-argocd-sso.sh`
- `infra/scripts/verify-platform-sso.sh`

### Runtime facts invariants (must be enforced)

For each environment `dev|staging|production`, CI must enforce:

- The published runtime fact `azure.envoyPublicIpAddress` equals the reserved Envoy Public IP resource (`ameide-<env>-envoy-pip`) in the correct Azure subscription.
- The environment DNS zone contains wildcard coverage (`@` and `*`) pointing at that Envoy public IP, so any declared route hostname resolves without adding per-app DNS records.

If these invariants fail, CI must fail the deploy (do not “fix it manually”).

---

## Backends and Portability (required changes)

### Principle: no hardcoded backend identities in Terraform code

To support “switch subscription” and “deploy to AWS”, backends must be **configured by CI**, not baked into:
- `infra/terraform/*/versions.tf`

v2 requirement:
- Terraform code contains only a backend *block* (type), while CI provides per-target backend settings via `terraform init -backend-config=...`.
  - **Azure**: resource group, storage account, container, key, Entra ID auth (OIDC / AAD auth), least-privilege role assignments
  - **AWS**: S3 bucket, key, region, **S3-native state locking** (`use_lockfile`), and role/session settings

This enables:
- multiple subscriptions/accounts/environments without code edits
- per-target isolation of state with predictable naming

### Output contract (cross-cloud)

Every cloud workspace must emit a stable output schema so GitOps stays cloud-agnostic. Minimum contract (provider-neutral):
- Cluster identity:
  - `cluster_name`
  - `cluster_type` (`azure` / `aws` / `local`)
  - `kube_context` (stable name used by bootstrap/verifiers)
- Edge routing:
  - `cluster_gateway_public_address` (IP or hostname)
  - `env_gateway_public_addresses.{dev,staging,prod}` (IP or hostname)
- DNS challenge identity (provider-neutral reference object):
  - `dns_challenge_identities.{argocd,dev,staging,prod}` where each entry can be:
    - Azure: `azure_client_id`
    - AWS: `aws_role_arn`
- Secrets bootstrap endpoint:
  - `secrets_source_uri` (e.g., Key Vault URI for Azure; placeholder for AWS)

CI publishes runtime facts from this output contract into the runtime-facts repo/branch (not into `main` of the human repo).

Implementation note:
- `infra/scripts/sync-globals.sh` is the current adapter for Azure and can be refactored or replaced with a `sync-runtime-facts.sh` that writes to the runtime-facts repo format.

Policy: values files consuming runtime facts must source them from globals, not hardcode them (see `backlog/519-gitops-fleet-policy-hardening.md`).

---

## Bootstrap Strategy (reduce Terraform side effects)

### Problem: Terraform `local-exec` is non-hermetic

Terraform resources that shell out to:
- `az aks get-credentials`
- `kubectl`
- `helm`

can be useful for local workflows, but are a source of nondeterminism for CI-only cloud provisioning.

v2 direction:
- Keep Terraform focused on cloud resources + stable outputs.
- Run Kubernetes bootstrap as explicit CI steps (or scripts invoked by CI), using Terraform outputs as inputs.

This creates a single, observable “pipeline”:
1) `terraform apply`  
2) publish runtime facts (CI-owned commit/push to runtime-facts repo/branch)  
3) bootstrap GitOps (install ArgoCD + apply root ApplicationSets)  
4) wait gates + verifiers

---

## Secrets: source-of-truth and transport

### Cloud

Cloud secret values must not be stored in GitHub. CI should source secret values from a cloud-native secret store:
- Azure: Key Vault (via OIDC)
- AWS: Secrets Manager / SSM (future)

Terraform may create the vault/identities, but secret *values* should be injected through the CI pipeline (or a separate secure seeding job), not checked into repo state.

Minimum secret set for dev AKS now includes Coder bootstrap (and workspace auth bootstrap):
- `coder-bootstrap-admin-username|email|password` (seed initial Coder owner; skip `/setup`)
- `coder-bootstrap-token` (one-time seed; used by in-cluster reconcilers)

### Local

Local uses `.env`/`.env.local` + local seeding:
- `infra/scripts/seed-local-secrets.sh`
- `deploy.sh local` remains supported as the canonical local entry point.

---

## Local k3d (must remain supported)

Local stays “Terraform-first”:
- Workspace: `infra/terraform/local`
- Entry: `infra/scripts/tf-local.sh` and/or `./infra/scripts/deploy.sh local`

Local requirements:
- Never touches Azure/AWS resources
- Creates cluster + ArgoCD + root ApplicationSets deterministically
- Seeds local secrets from `.env` and converges ExternalSecrets/Vault flow

---

## Scheduling / Node Pools (follow-up, not required for v2 cutover)

This repo previously relied on environment-specific node pools + taints/tolerations. v2 should move toward:
- **stateful on stable capacity**
- **stateless on reclaimable capacity (Spot)** with safe fallbacks

This is intentionally orthogonal to “CI-only cloud provisioning”. It can be staged after CI-only is stable.

---

## Migration Plan (from v1)

1. **CI is authoritative for cloud**: document and enforce “no local apply” for cloud workspaces.
2. **Backend portability**:
   - move backend identity details out of Terraform files
   - provide per-target backend config in workflows
3. **Bootstrap refactor**:
   - move cloud bootstrap out of Terraform `local-exec` into CI steps
   - keep local bootstrap convenience in `deploy.sh local`
4. **Cross-cloud outputs contract**:
   - align Azure outputs to the contract
   - scaffold `infra/terraform/aws` with matching outputs (even if initially partial)
5. **Verification gates**:
   - keep TLS/443/SSO verifiers as first-class CI gates
   - make failures actionable (logs, resource snapshots)
6. **Runtime facts repo/branch**:
   - switch from “CI commits globals to `main`” to “CI publishes runtime facts to dedicated repo/branch”
   - update ArgoCD Applications to use multiple sources to consume runtime facts

---

## Decisions (answers to v1 open questions)

1. **Runtime facts publication**: CI writes runtime facts to a dedicated repo/branch (CI-only), not to `main` of the human repo.
2. **AWS runtime facts schema**: use provider-neutral identity references (`aws_role_arn` vs `azure_client_id`) and allow “public address” fields to be IPs or hostnames.
3. **Per-env capacity isolation**: deferred; once env-specific pools are removed, replace isolation with Kubernetes primitives (quotas/priority classes) if required.
