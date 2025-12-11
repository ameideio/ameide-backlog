# Cluster Rightsizing Notes

## Context
- Goal is to run the production AKS cluster on a 3–4 node pool.
- Current pool uses `Standard_D2as_v6` nodes (2 vCPU, 8 Gi RAM each).
- We aggressively reduced resource requests across workloads (Neo4j, Temporal, MinIO, inference, gateway, observability, Argo CD, etc.).
- Azure-managed add-ons (`ama-metrics`, `retina-agent`, `azure-ip-masq`, `kube-proxy`) remain enabled.

## Consolidated Resource Footprint (Post-Tuning)
- Application namespaces (`ameide`, `argocd`, etc.): **≈3.7 vCPU**, **≈14 Gi RAM** requested.
- System namespaces (`kube-system`, `cert-manager`, `redis-operator`, etc.): **≈6.3 vCPU**, **≈9 Gi RAM** requested.
- **Total requested** across the cluster: **≈9.9 vCPU**, **≈24 Gi RAM** (captured via `kubectl get pods -A -o json` and parsing into `/tmp/pods.json`).
- Key contributors:
  - Neo4j: 0.1 vCPU / 1 Gi RAM (cannot realistically go lower without risking JVM OOM).
  - Azure Monitor (`ama-metrics`, 2 pods): ~0.34 vCPU / 1 Gi RAM total.
  - Retina network observability (DaemonSet): ~0.6 vCPU / 1.2 Gi RAM total.
  - Argo CD stack: ~0.35 vCPU / 1.5 Gi RAM.

### Per-Service Requests/Limits (production overrides)
| Service / Component | Replicas | CPU Request (per pod) | Memory Request (per pod) | CPU Limit (per pod) | Memory Limit (per pod) | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| Neo4j (statefulset) | 1 | 100 m | 1 Gi | 200 m | 2 Gi | Minimum viable for JVM; 100 Gi PVC |
| Redis Failover – Redis | 1 | 75 m | 192 Mi | 150 m | 384 Mi | Reduced to single replica |
| Redis Failover – Sentinel | 1 | 50 m | 128 Mi | 120 m | 256 Mi | |
| MinIO | 1 | 75 m | 512 Mi | 150 m | 768 Mi | 100 Gi PVC |
| Inference API | 1 | 100 m | 768 Mi | 200 m | 1.5 Gi | |
| Inference Gateway | 1 | 50 m | 192 Mi | 150 m | 384 Mi | |
| Temporal Frontend | 1 | 50 m | 384 Mi | 150 m | 768 Mi | |
| Temporal History | 1 | 50 m | 384 Mi | 150 m | 768 Mi | |
| Temporal Matching | 1 | 50 m | 384 Mi | 150 m | 768 Mi | |
| Temporal Worker | 1 | 50 m | 384 Mi | 150 m | 768 Mi | |
| Temporal Web | 1 | 40 m | 128 Mi | 120 m | 320 Mi | |
| Temporal AdminTools | 1 | 40 m | 128 Mi | 120 m | 256 Mi | |
| OpenTelemetry Collector | 2 | 50 m | 256 Mi | 150 m | 512 Mi | |
| Keycloak (statefulset) | 1 | 75 m | 384 Mi | 150 m | 768 Mi | |
| Keycloak Operator | 1 | 100 m | 256 Mi | 250 m | 512 Mi | |
| Grafana | 1 | 40 m | 160 Mi | 120 m | 320 Mi | |
| Loki (statefulset) | 1 | 75 m | 320 Mi | 150 m | 640 Mi | |
| Tempo (statefulset) | 1 | 75 m | 320 Mi | 150 m | 640 Mi | |
| Envoy Gateway control plane | 1 | 75 m | 256 Mi | 150 m | 512 Mi | |
| Envoy data plane | 1 | 75 m | 256 Mi | 150 m | 512 Mi | |
| Kafka NodePool (broker/controller) | 1 | 100 m | 384 Mi | 250 m | 768 Mi | |
| Kafka Entity Operator | 1 | 75 m | 256 Mi | 150 m | 512 Mi | |
| Argo CD Controller | 1 | 200 m | 512 Mi | 400 m | 1 Gi | StatefulSet |
| Argo CD Repo Server | 2 | 75 m | 192 Mi | 200 m | 384 Mi | Includes cmp-helmfile sidecar |
| Argo CD ApplicationSet | 2 | 75 m | 192 Mi | 200 m | 384 Mi | |
| Argo CD API Server | 1 | 100 m | 256 Mi | 250 m | 512 Mi | |
| Cert-Manager (controller) | 1 | 150 m | 220 Mi | 300 m | 440 Mi | |
| kube-system: ama-metrics (per pod) | 2 | 170 m | 530 Mi | 300 m | 768 Mi | Azure Monitor add-on |
| kube-system: retina-agent (per pod) | 6 | 100 m | 200 Mi | 200 m | 400 Mi | Azure network observability |
| kube-system: azure-ip-masq (per pod) | 6 | 25 m | 40 Mi | 80 m | 64 Mi | |
| kube-system: kube-proxy (per pod) | 6 | 50 m | 60 Mi | 100 m | 120 Mi | |
| kube-system: metrics-server (per pod) | 2 | 75 m | 128 Mi | 120 m | 200 Mi | |

## Why 3–4 × D2as Still Fails
- Four `Standard_D2as_v6` nodes provide **8 vCPU** and **32 Gi RAM**.
- Even with reduced requests, scheduler still needs ~10 vCPU; when we scaled the pool to four nodes, Grafana, Kafka, Loki, Neo4j, Temporal, and cert-manager failed to schedule.
- Memory is now under 32 Gi, but vCPU remains the hard limit.
- Neo4j’s 1 Gi request is already the minimum advisable (per vendor guidance); making it smaller risks instability, yet CPU pressure would still exist even if Neo4j were zero.

## Alternative Paths Considered
| Option | Pros | Cons / Impact |
| --- | --- | --- |
| Keep D2as, tighten requests further | Already at “bare minimum” for core services; risk of thrash and pod eviction; still can’t offset Azure add-ons. |
| Switch to memory-optimized SKU (e.g., E2as_v5) | More RAM per node | Still only 2 vCPU per node; CPU bottleneck persists. |
| Disable Azure add-ons (`ama-metrics`, `retina`, etc.) | Recovers ~1.5+ vCPU | Sacrifices platform observability; may violate ops requirements. |
| Split workloads across pools | Keeps system pool small | Adds operational overhead; doesn’t solve if all pods must run on system pool. |
| Move to higher-vCPU SKU (e.g., D4as_v6) | 4 vCPU / 16 Gi per node; 3–4 nodes provide 12–16 vCPU | Requires Bicep update + maintenance window; slightly higher cost. |

## Recommendation
- **Adopt `Standard_D4as_v6` for the production system node pool.**
  - Update Bicep (`infra/bicep/managed-application/modules/aks.bicep`) to use D4as_v6 with autoscaler bounds 3–4 for production.
  - Redeploy Bicep to resize the pool; plan a maintenance window for node drain.
- Maintain the tuned resource requests captured in Git to keep headroom.
- Optionally remove or offload Azure add-ons if future cost/footprint reductions are required.

## Follow-Up Actions
1. Schedule Bicep deployment to apply the node SKU change.
2. Monitor workloads post-resize; confirm autoscaler can sustain 3–4 node window.
3. Re-run `kubectl get pods -A -o json` and validate totals stay well below the new capacity (12–16 vCPU, 48–64 Gi RAM).
4. Document any future decision to disable Azure extensions or reintroduce replicas.
