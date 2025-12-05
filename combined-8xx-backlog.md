# Combined Backlog (8xx Series)

Generated on: 2025-12-05 15:47:49 UTC

---

## 800-k8s.md (old)

Below is an opinionated reference architecture for running **many customers (or internal business units) in a single Kubernetes cluster while guaranteeing hard data‑plane isolation and strong security boundaries**.  The design draws on patterns proven in regulated industries (PCI‑DSS, HIPAA, financial services) and in large SaaS platforms such as Salesforce Hyperforce and VMware Tanzu SaaS.

---

## 1 . Tenancy Model & Isolation Levels

| Isolation Plane   | Goal                                                                     | Mechanism in This Design                                                             |
| ----------------- | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------ |
| **Control‑plane** | Tenants must *never* see each other’s objects via the API.               | One namespace per tenant; strict RBAC; admission‑controller hard‑stops.              |
| **Compute**       | No cross‑tenant process interference.                                    | Node pools dedicated per sensitivity tier, runtime hardening (gVisor/Kata).          |
| **Network**       | Packets can’t cross tenant boundary except via allowed public endpoints. | Calico/Cilium NetworkPolicies + per‑tenant VPC sub‑CIDR or eBPF VXLAN ID.            |
| **Storage**       | Persistent data must be physically or cryptographically separated.       | StorageClass per tenant; CSI encryption with tenant‑scoped KMS key.                  |
| **Observability** | Logs/metrics/traces are isolated and access‑controlled.                  | Multi‑tenant Prometheus (Thanos receive) + Loki tenant ID + Tempo tenant ID.         |
| **Secrets**       | Keys/tokens are never visible outside tenant.                            | External secrets operator backed by one Vault namespace (or AWS KMS key) per tenant. |

---

## 2 . Cluster Building Blocks

1. **Namespaces**

   * Authoritative boundary; all objects carry a `tenant=<uuid>` label.
   * Mutating admission webhook rejects any object referencing another tenant’s namespace.

2. **RBAC Profiles**

   * `TenantAdmin`, `TenantDev`, `TenantView` cluster roles generated from a template.
   * RoleBindings are created by an **operator** when a namespace is provisioned.

3. **Node Pools (optional but recommended)**

   * `user‑workload‑standard` (shared), `user‑workload‑sensitive` (Kata/gVisor + encrypted swap).
   * Nodes are tainted `tenant‑tier=<tier>`; a namespace default `NodeSelector` pins pods.

4. **Network Policies**

   * Default deny on ingress/egress per namespace.
   * Only egress to whitelisted public services (S3, Stripe, etc.) via DNS selectors.
   * “Tenant gateway” (NGINX or Istio Ingress) lives in a *system* namespace and is the **only** cross‑tenant path.

5. **Storage Classes**

   * CSI provisioner label `parameters.kmsKeyId=arn:aws:kms:…:<tenant‑uuid>` (or per‑tenant vSAN policy).
   * RWX volumes (e.g., NFS) are **never** shared; use RWX‑capable CSI drivers that support volume‑level encryption.

6. **Admission Control Stack (OPA/Gatekeeper)**

   * *namespace‑scoped* policies:

     * `spec.volumes[].hostPath` = DENY
     * `metadata.namespace` must equal `request.user.extra.tenant_id`
     * disallow `LoadBalancer` Service type unless annotation `tenant-override=true` (forces use of shared Ingress).

7. **Service Mesh Partitioning (Istio ≥1.19 or Linkerd)**

   * **Trust domain per tenant** (`trustDomain=tenant‑<uuid>` in Sidecar CR).
   * Multi‑control‑plane (Istiod) is optional; most teams run a single control plane with \[Istio Tenant Isolation by Sidecar Scope].
   * mTLS is still cluster‑wide; root CA is kept by platform team.

8. **Secrets & Config**

   * \[External Secrets Operator] fetches from **Vault**:

     * Vault namespace = tenant UUID.
     * KV v2 mount + per‑tenant AppRole wrapped tokens.
   * Optional: AWS Secrets Manager with resource policy restricting to tenant IAM role.

9. **Observability**

   * **Prometheus Agent** sidecars scrape only the local namespace; remote‑write label `tenant=<uuid>`.
   * Thanos **receive** runs with `--tenant-label=tenant`; querier uses tenancy headers.
   * Loki multi‑tenant gateway; Tempo receive with separate TSDB per tenant.

10. **Quota & Fairness**

    * `ResourceQuota` + `LimitRange` per namespace to defend against noisy neighbours.
    * Kubernetes **PriorityClass** to give system pods pre‑emption rights over tenants.

---

## 3 . Provisioning Lifecycle (GitOps)

1. **Signup / “createTenant” API** creates a *Tenant Custom Resource* (CR) in the platform namespace.
2. **Tenant‑Operator** (a Kopf or controller‑runtime operator) reacts:

   ```text
   - generates namespace → tenant-<uuid>
   - applies RBAC & LimitRange
   - patches NetworkPolicy deny‑all
   - creates StorageClass tenant‑<uuid>
   - requests Vault namespace + KMS key
   - applies ArgoCD Application pointing to tenant Git repo path
   ```
3. ArgoCD syncs the tenant’s **app manifests** into their namespace.
4. On deletion, Tenant‑Operator cascades cleanup after a configurable retention period.

Everything is auditable: each step emits a Kubernetes Event and platform team logs a PROV‑O `Activity`.

---

## 4 . Data Isolation Deep Dive

| Control                      | Implementation Detail                                                                                                      |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **Encryption‑at‑rest**       | CSI driver (`EBS`, `GCP PD`, `vSphere`) with per‑PV key; key ID derived from tenant UUID.                                  |
| **Encryption‑in‑transit**    | Mesh mTLS **and** storage‑backend TLS.                                                                                     |
| **Backup Segregation**       | Velero runs in “platform” namespace but uses per‑tenant AWS IAM role + S3 prefix.                                          |
| **Failure Domain Isolation** | Kubernetes **PodDisruptionBudget** and topology spread constraints make sure one tenant can’t monopolise a single AZ/host. |
| **Metadata Hardening**       | Audit webhooks redact tenant IDs before shipping logs to central SIEM.                                                     |

---

## 5 . Security & Compliance Guard‑Rails

* **Pod Security Admission** set to `baseline` cluster‑wide; tenants cannot escalate to `privileged`.
* **Syscalls**: seccomp default profile; only `tenant‑sensitive` pool runs gVisor/Kata.
* **Auditing**: Audit Webhook copies all requests to an aggregator that injects `request.user.extra.tenant_id` into the record.
* **Image Policy**: Sigstore verification Gatekeeper constraint; tenants can only pull images signed by their CI or by the platform team.
* **Pen Testing Hook**: Platform exposes a “breakglass” role (`tenant‑pentest‑<uuid>`) that drops after 24 h.

---

## 6 . Operational Considerations

* **Scaling limits**: Most cloud CNI plug‑ins support ≤ 250 namespaces with dense policies before etcd latency spikes—benchmark with your workload.
* **Cost accounting**: Use `kubecost` or OpenCost with `aggregateBy: ["namespace","tenant"]`.
* **Upgrades**: Perform canary node‑pool upgrades by tier—start with `user‑workload‑standard`, hold 24 h, then sensitive pools.
* **Disaster Recovery**: Restore per‑tenant backups into a *clean* staging cluster first; run vulnerability scan before promoting restore to production.
* **Runbooks**: Write *one* SOP per plane—control‑plane, network, storage, observability—to keep cognitive load reasonable for on‑call.

---

## 7 . When to Split into Multiple Clusters

You can stay single‑cluster until one of these becomes true:

1. **Hard regulatory demand** (e.g., data‑residency per country) → clone cluster per region.
2. **> 4 k Namespaces / > 150 k objects** → etcd or CNI scaling pain.
3. **Kubernetes minor versions** must diverge (rare).
4. **Different platform‑level PSP / SELinux modes** per tenant group.

If you branch out, reuse the same operator & GitOps flow; only the bootstrap stack changes.

---

### Bottom Line

A **namespace‑per‑tenant pattern backed by strict admission control, network/secret segmentation, encrypted per‑tenant storage, and labeled observability pipelines** is the simplest architecture that:

* Scales to hundreds of tenants,
* Passes regulated‑industry audits,
* Keeps platform operational overhead modest (one control plane to patch).

All other components—service mesh, node‑pool segregation, side‑channel monitoring—are additive defence‑in‑depth layers you can incrementally enable as the attack surface or compliance bar rises.

---

