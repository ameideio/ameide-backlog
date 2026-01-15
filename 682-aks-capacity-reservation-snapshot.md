# AKS capacity / reservation snapshot

Generated (UTC): 2026-01-15T13:30:46Z
kubectl context: ameide

## Node inventory
```
NAME                             STATUS   ROLES    AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
aks-system-10377240-vmss000005   Ready    <none>   23h   v1.32.7   10.224.0.10    <none>        Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.29-1
aks-system-10377240-vmss000007   Ready    <none>   23h   v1.32.7   10.224.0.199   <none>        Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.29-1
aks-system-10377240-vmss000008   Ready    <none>   23h   v1.32.7   10.224.1.166   <none>        Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.29-1
aks-system-10377240-vmss00000b   Ready    <none>   18h   v1.32.7   10.224.0.69    <none>        Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.29-1
aks-system-10377240-vmss00000c   Ready    <none>   16h   v1.32.7   10.224.1.34    <none>        Ubuntu 22.04.5 LTS   5.15.0-1102-azure   containerd://1.7.29-1
```

## Node metrics (top)
```
NAME                             CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)   
aks-system-10377240-vmss000005   1108m        14%      19456Mi         65%         
aks-system-10377240-vmss000007   4430m        56%      11313Mi         38%         
aks-system-10377240-vmss000008   877m         11%      14688Mi         49%         
aks-system-10377240-vmss00000b   4484m        57%      17023Mi         57%         
aks-system-10377240-vmss00000c   751m         9%       12191Mi         40%         
```

## Pending / not-ready pods (all namespaces)
```
NAMESPACE                                        NAME                                                              READY   STATUS              RESTARTS         AGE     IP             NODE                             NOMINATED NODE   READINESS GATES
ameide-dev                                       codex-auth-refresher-0-29474100-wj8zg                             0/1     Completed           0                10h     10.224.2.10    aks-system-10377240-vmss00000c   <none>           <none>
ameide-dev                                       codex-auth-refresher-1-29474100-dzvk9                             0/1     Completed           0                10h     10.224.1.89    aks-system-10377240-vmss00000c   <none>           <none>
ameide-dev                                       platform-keycloak-realm-client-patcher-p6c79                      0/1     Completed           0                141m    10.224.2.21    aks-system-10377240-vmss00000c   <none>           <none>
ameide-dev                                       postgres-password-reconcile-29474720-t8n2j                        0/1     Completed           0                10m     10.224.0.202   aks-system-10377240-vmss000007   <none>           <none>
ameide-prod                                      keycloak-0                                                        0/1     Pending             0                34m     <none>         <none>                           <none>           <none>
ameide-prod                                      kubernetes-dashboard-oauth2-proxy-5f966d5694-c95qc                0/1     CrashLoopBackOff    10 (3m2s ago)    37m     10.224.0.71    aks-system-10377240-vmss00000b   <none>           <none>
ameide-prod                                      nifi-0-node47s7q                                                  0/1     Completed           0                19h     10.224.1.237   aks-system-10377240-vmss000008   <none>           <none>
ameide-prod                                      platform-backstage-666946744-r82nl                                0/1     CrashLoopBackOff    11 (3m14s ago)   37m     10.224.1.234   aks-system-10377240-vmss000008   <none>           <none>
ameide-prod                                      platform-langfuse-worker-5b7f94648c-sgprc                         0/1     CrashLoopBackOff    9 (5m9s ago)     37m     10.224.0.24    aks-system-10377240-vmss000007   <none>           <none>
ameide-prod                                      plausible-oauth2-proxy-75f76c4578-22f4z                           0/1     CrashLoopBackOff    9 (71s ago)      37m     10.224.0.234   aks-system-10377240-vmss000007   <none>           <none>
ameide-staging                                   platform-backstage-5c7bd9c95-2qvx4                                0/1     ContainerCreating   0                37m     <none>         aks-system-10377240-vmss00000b   <none>           <none>
argocd                                           preview-janitor-29474715-sjcqg                                    0/1     Completed           0                15m     10.224.0.63    aks-system-10377240-vmss000007   <none>           <none>
argocd                                           preview-janitor-29474730-ll56b                                    0/1     Completed           0                48s     10.224.0.211   aks-system-10377240-vmss000007   <none>           <none>
buildkit                                         buildkitd-1                                                       0/1     Pending             0                15h     <none>         <none>                           <none>           <none>
```

## Recent warning events (last 200)
```
ameide-prod                                      6m42s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474724-cg9dt              Created container: kubectl
ameide-prod                                      6m42s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474724-cg9dt              Container image "docker.io/alpine/k8s@sha256:aad13a02739dd0aee5ff376f75ef9bfcc4111ccf9c0f86eef9e096a995262505" already present on machine
ameide-prod                                      6m40s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474724-cg9dt              Started container kubectl
ameide-staging                                   6m40s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474724-4jjs2              Container image "docker.io/alpine/k8s@sha256:aad13a02739dd0aee5ff376f75ef9bfcc4111ccf9c0f86eef9e096a995262505" already present on machine
ameide-staging                                   6m39s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474724-4jjs2              Created container: kubectl
ameide-staging                                   6m38s       Normal    ClusterReconciling                nificluster/nifi                                                           NifiCluster starting reconciliation
ameide-staging                                   6m36s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474724-4jjs2              Started container kubectl
ameide-prod                                      6m35s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474724-cg9dt              Container image "docker.io/hashicorp/vault@sha256:62dd55c9ccbdc0af0a9269e87481a64650258907434d5ddb5e795e2eb2ac5780" already present on machine
ameide-prod                                      6m34s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474724-cg9dt              Created container: bootstrap
ameide-dev                                       6m34s       Warning   Unhealthy                         pod/kafka-kafka-pool-0                                                     Readiness probe failed:   % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current...
ameide-prod                                      6m32s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474724-cg9dt              Started container bootstrap
ameide-staging                                   6m31s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474724-4jjs2              Container image "docker.io/hashicorp/vault@sha256:62dd55c9ccbdc0af0a9269e87481a64650258907434d5ddb5e795e2eb2ac5780" already present on machine
ameide-staging                                   6m30s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474724-4jjs2              Created container: bootstrap
ameide-staging                                   6m27s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474724-4jjs2              Started container bootstrap
ameide-prod                                      6m21s       Normal    Started                           pod/plausible-oauth2-proxy-75f76c4578-22f4z                                Started container oauth2-proxy
argocd                                           6m19s       Normal    ResourceUpdated                   application/cluster-platform-envoy-crds                                    Updated health status: Healthy -> Unknown
ameide-dev                                       6m11s       Warning   Unhealthy                         pod/platform-camunda8-zeebe-0                                              Readiness probe failed: Get "http://10.224.0.223:9600/actuator/health/readiness": dial tcp 10.224.0.223:9600: connect: connection refused
ameide-prod                                      6m1s        Normal    Created                           pod/plausible-plausible-6b5c446b68-zzxpn                                   Created container: plausible-db-init
ameide-prod                                      6m1s        Normal    Pulled                            pod/plausible-plausible-6b5c446b68-zzxpn                                   Container image "ghcr.io/ameideio/mirror/plausible-analytics@sha256:cd5f75e1399073669b13b4151cc603332a825324d0b8f13dfc9de9112a3c68a1" already present on machine
ameide-prod                                      5m59s       Normal    Started                           pod/plausible-plausible-6b5c446b68-zzxpn                                   Started container plausible-db-init
ameide-dev                                       5m57s       Normal    Updated                           externalsecret/codex-account-status-sync-0                                 secret updated
ameide-staging                                   5m50s       Warning   Unhealthy                         pod/prometheus-staging-prometheus-prometheus-0                             Startup probe failed: HTTP probe failed with statuscode: 503
ameide-prod                                      5m48s       Normal    Killing                           pod/foundation-vault-bootstrap-vault-bootstrap-29474724-cg9dt              Stopping container bootstrap
ameide-prod                                      5m48s       Normal    SuccessfulCreate                  job/foundation-vault-bootstrap-vault-bootstrap-29474725                    Created pod: foundation-vault-bootstrap-vault-bootstrap-29474725-rpxqg
ameide-staging                                   5m48s       Normal    SuccessfulCreate                  job/foundation-vault-bootstrap-vault-bootstrap-29474725                    Created pod: foundation-vault-bootstrap-vault-bootstrap-29474725-nq99v
ameide-staging                                   5m48s       Normal    Killing                           pod/foundation-vault-bootstrap-vault-bootstrap-29474724-4jjs2              Stopping container bootstrap
ameide-staging                                   5m31s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474725-nq99v              Container image "docker.io/alpine/k8s@sha256:aad13a02739dd0aee5ff376f75ef9bfcc4111ccf9c0f86eef9e096a995262505" already present on machine
ameide-staging                                   5m29s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474725-nq99v              Created container: kubectl
ameide-prod                                      5m28s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474725-rpxqg              Container image "docker.io/alpine/k8s@sha256:aad13a02739dd0aee5ff376f75ef9bfcc4111ccf9c0f86eef9e096a995262505" already present on machine
ameide-prod                                      5m27s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474725-rpxqg              Created container: kubectl
ameide-staging                                   5m23s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474725-nq99v              Started container kubectl
ameide-prod                                      5m23s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474725-rpxqg              Started container kubectl
ameide-prod                                      5m23s       Normal    Pulled                            pod/plausible-plausible-6b5c446b68-zzxpn                                   Container image "ghcr.io/ameideio/mirror/plausible-analytics@sha256:cd5f75e1399073669b13b4151cc603332a825324d0b8f13dfc9de9112a3c68a1" already present on machine
ameide-staging                                   5m18s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474725-nq99v              Container image "docker.io/hashicorp/vault@sha256:62dd55c9ccbdc0af0a9269e87481a64650258907434d5ddb5e795e2eb2ac5780" already present on machine
ameide-staging                                   5m17s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474725-nq99v              Created container: bootstrap
ameide-prod                                      5m16s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474725-rpxqg              Container image "docker.io/hashicorp/vault@sha256:62dd55c9ccbdc0af0a9269e87481a64650258907434d5ddb5e795e2eb2ac5780" already present on machine
argocd                                           5m15s       Normal    ResourceUpdated                   application/production-platform-keycloak-crds                              Updated health status: Unknown -> Healthy
ameide-prod                                      5m15s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474725-rpxqg              Created container: bootstrap
ameide-staging                                   5m14s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474725-nq99v              Started container bootstrap
ameide-prod                                      5m12s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474725-rpxqg              Started container bootstrap
ameide-staging                                   4m48s       Normal    SuccessfulDelete                  cronjob/foundation-vault-bootstrap-vault-bootstrap                         (combined from similar events): Deleted job foundation-vault-bootstrap-vault-bootstrap-29474725
ameide-prod                                      4m48s       Normal    SuccessfulDelete                  cronjob/foundation-vault-bootstrap-vault-bootstrap                         (combined from similar events): Deleted job foundation-vault-bootstrap-vault-bootstrap-29474725
ameide-dev                                       4m48s       Normal    JobAlreadyActive                  cronjob/foundation-vault-bootstrap-vault-bootstrap                         Not starting job because prior execution is running and concurrency policy is Forbid
ameide-prod                                      4m48s       Normal    SuccessfulCreate                  job/foundation-vault-bootstrap-vault-bootstrap-29474726                    Created pod: foundation-vault-bootstrap-vault-bootstrap-29474726-prkwm
ameide-staging                                   4m48s       Normal    Killing                           pod/foundation-vault-bootstrap-vault-bootstrap-29474725-nq99v              Stopping container bootstrap
ameide-staging                                   4m48s       Normal    SuccessfulCreate                  job/foundation-vault-bootstrap-vault-bootstrap-29474726                    Created pod: foundation-vault-bootstrap-vault-bootstrap-29474726-sjh57
ameide-prod                                      4m48s       Normal    Killing                           pod/foundation-vault-bootstrap-vault-bootstrap-29474725-rpxqg              Stopping container bootstrap
ameide-staging                                   4m42s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474726-sjh57              Created container: kubectl
ameide-staging                                   4m42s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474726-sjh57              Container image "docker.io/alpine/k8s@sha256:aad13a02739dd0aee5ff376f75ef9bfcc4111ccf9c0f86eef9e096a995262505" already present on machine
ameide-staging                                   4m41s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474726-sjh57              Started container kubectl
ameide-prod                                      4m41s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474726-prkwm              Created container: kubectl
ameide-prod                                      4m41s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474726-prkwm              Container image "docker.io/alpine/k8s@sha256:aad13a02739dd0aee5ff376f75ef9bfcc4111ccf9c0f86eef9e096a995262505" already present on machine
ameide-prod                                      4m39s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474726-prkwm              Started container kubectl
argocd                                           4m38s       Normal    ResourceUpdated                   application/cluster-platform-envoy-crds                                    Updated health status: Unknown -> Healthy
ameide-staging                                   4m36s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474726-sjh57              Container image "docker.io/hashicorp/vault@sha256:62dd55c9ccbdc0af0a9269e87481a64650258907434d5ddb5e795e2eb2ac5780" already present on machine
ameide-staging                                   4m35s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474726-sjh57              Created container: bootstrap
ameide-prod                                      4m34s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474726-prkwm              Container image "docker.io/hashicorp/vault@sha256:62dd55c9ccbdc0af0a9269e87481a64650258907434d5ddb5e795e2eb2ac5780" already present on machine
ameide-staging                                   4m34s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474726-sjh57              Started container bootstrap
ameide-prod                                      4m33s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474726-prkwm              Created container: bootstrap
ameide-prod                                      4m31s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474726-prkwm              Started container bootstrap
ameide-prod                                      4m26s       Normal    Created                           pod/platform-langfuse-web-fdd77b54d-cx8vf                                  Created container: langfuse-web
ameide-prod                                      4m26s       Normal    Pulled                            pod/platform-langfuse-web-fdd77b54d-cx8vf                                  Container image "ghcr.io/langfuse/langfuse:3.120.0@sha256:7b8f7cb77fcbe25921c530f0f46a12c5fd5f911d9d5bf3f03b7b0fe00f1466ab" already present on machine
argocd                                           4m24s       Normal    ResourceUpdated                   application/production-plausible                                           Updated health status: Progressing -> Healthy
ameide-prod                                      4m24s       Normal    Started                           pod/platform-langfuse-web-fdd77b54d-cx8vf                                  Started container langfuse-web
ameide-dev                                       4m19s       Warning   KEDAScalerFailed                  scaledjob/workrequests-agentwork-coder                                     error creating kafka client: kafka: client has run out of available brokers to talk to: dial tcp 10.0.9.104:9092: connect: connection refused
ameide-dev                                       4m14s       Warning   KEDAScalerFailed                  scaledjob/workrequests-toolrun-verify-ui-harness                           error creating kafka client: kafka: client has run out of available brokers to talk to: dial tcp 10.0.9.104:9092: connect: connection refused
argocd                                           4m9s        Normal    ResourceUpdated                   application/dev-platform-camunda8                                          Updated health status: Progressing -> Healthy
argocd                                           4m9s        Normal    ResourceUpdated                   application/dev-platform-camunda8                                          Updated sync status: Synced -> Unknown
ameide-prod                                      4m9s        Warning   Unhealthy                         pod/chi-data-clickhouse-data-clickhouse-0-0-0                              Readiness probe failed: Get "http://10.224.1.121:8123/ping": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
ameide-dev                                       4m8s        Warning   KEDAScalerFailed                  scaledjob/workrequests-toolrun-generate                                    error creating kafka client: kafka: client has run out of available brokers to talk to: dial tcp 10.0.9.104:9092: connect: connection refused
ameide-prod                                      3m48s       Normal    Killing                           pod/foundation-vault-bootstrap-vault-bootstrap-29474726-prkwm              Stopping container bootstrap
ameide-prod                                      3m48s       Normal    SuccessfulCreate                  job/foundation-vault-bootstrap-vault-bootstrap-29474727                    Created pod: foundation-vault-bootstrap-vault-bootstrap-29474727-rxzvq
ameide-staging                                   3m48s       Normal    SuccessfulCreate                  job/foundation-vault-bootstrap-vault-bootstrap-29474727                    Created pod: foundation-vault-bootstrap-vault-bootstrap-29474727-x5pkv
ameide-staging                                   3m48s       Normal    Killing                           pod/foundation-vault-bootstrap-vault-bootstrap-29474726-sjh57              Stopping container bootstrap
ameide-prod                                      3m44s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474727-rxzvq              Container image "docker.io/alpine/k8s@sha256:aad13a02739dd0aee5ff376f75ef9bfcc4111ccf9c0f86eef9e096a995262505" already present on machine
ameide-staging                                   3m43s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474727-x5pkv              Container image "docker.io/alpine/k8s@sha256:aad13a02739dd0aee5ff376f75ef9bfcc4111ccf9c0f86eef9e096a995262505" already present on machine
ameide-prod                                      3m43s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474727-rxzvq              Created container: kubectl
ameide-staging                                   3m42s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474727-x5pkv              Created container: kubectl
ameide-prod                                      3m42s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474727-rxzvq              Started container kubectl
ameide-staging                                   3m41s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474727-x5pkv              Started container kubectl
ameide-prod                                      3m39s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474727-rxzvq              Container image "docker.io/hashicorp/vault@sha256:62dd55c9ccbdc0af0a9269e87481a64650258907434d5ddb5e795e2eb2ac5780" already present on machine
ameide-staging                                   3m39s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474727-x5pkv              Container image "docker.io/hashicorp/vault@sha256:62dd55c9ccbdc0af0a9269e87481a64650258907434d5ddb5e795e2eb2ac5780" already present on machine
ameide-prod                                      3m38s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474727-rxzvq              Created container: bootstrap
ameide-staging                                   3m38s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474727-x5pkv              Created container: bootstrap
ameide-dev                                       3m37s       Warning   KEDAScalerFailed                  scaledjob/workrequests-toolrun-verify                                      error creating kafka client: kafka: client has run out of available brokers to talk to: dial tcp 10.0.9.104:9092: connect: connection refused
ameide-staging                                   3m35s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474727-x5pkv              Started container bootstrap
ameide-prod                                      3m35s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474727-rxzvq              Started container bootstrap
default                                          3m30s       Normal    Valid                             clustersecretstore/ameide-vault-dev                                        store validated
ameide-prod                                      3m29s       Normal    Pulled                            pod/platform-backstage-666946744-r82nl                                     Container image "quay.io/janus-idp/backstage-showcase@sha256:81021071ebd48fc889ccf98ba3cb6c797c38edc11227fcf18fe52bd262a64c9a" already present on machine
ameide-dev                                       3m21s       Normal    Valid                             secretstore/ameide-vault                                                   store validated
ameide-ws-52841969-f3ca-4f53-a747-1864fd7e0699   3m18s       Normal    Updated                           externalsecret/codex-account-status-sync-1                                 secret updated
ameide-dev                                       3m18s       Warning   BackOff                           pod/codex-monitor-0-55d55fd454-6hcxk                                       Back-off restarting failed container monitor in pod codex-monitor-0-55d55fd454-6hcxk_ameide-dev(f574a1a1-cbed-46c8-8552-369e0aa0b3af)
ameide-staging                                   3m9s        Normal    Valid                             secretstore/ameide-vault                                                   store validated
ameide-prod                                      3m8s        Normal    Pulled                            pod/kubernetes-dashboard-oauth2-proxy-5f966d5694-c95qc                     Container image "quay.io/oauth2-proxy/oauth2-proxy:v7.6.0@sha256:dcb6ff8dd21bf3058f6a22c6fa385fa5b897a9cd3914c88a2cc2bb0a85f8065d" already present on machine
ameide-staging                                   2m48s       Normal    Killing                           pod/foundation-vault-bootstrap-vault-bootstrap-29474727-x5pkv              Stopping container bootstrap
ameide-prod                                      2m48s       Normal    Killing                           pod/foundation-vault-bootstrap-vault-bootstrap-29474727-rxzvq              Stopping container bootstrap
ameide-prod                                      2m48s       Normal    SuccessfulCreate                  job/foundation-vault-bootstrap-vault-bootstrap-29474728                    Created pod: foundation-vault-bootstrap-vault-bootstrap-29474728-lwmfp
ameide-staging                                   2m48s       Normal    SuccessfulCreate                  job/foundation-vault-bootstrap-vault-bootstrap-29474728                    Created pod: foundation-vault-bootstrap-vault-bootstrap-29474728-l85tw
ameide-staging                                   2m39s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474728-l85tw              Container image "docker.io/alpine/k8s@sha256:aad13a02739dd0aee5ff376f75ef9bfcc4111ccf9c0f86eef9e096a995262505" already present on machine
ameide-staging                                   2m38s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474728-l85tw              Created container: kubectl
ameide-prod                                      2m37s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474728-lwmfp              Container image "docker.io/alpine/k8s@sha256:aad13a02739dd0aee5ff376f75ef9bfcc4111ccf9c0f86eef9e096a995262505" already present on machine
eclipse-che                                      2m36s       Normal    Valid                             secretstore/ameide-vault                                                   store validated
ameide-prod                                      2m35s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474728-lwmfp              Created container: kubectl
ameide-prod                                      2m33s       Warning   BackOff                           pod/plausible-oauth2-proxy-75f76c4578-22f4z                                Back-off restarting failed container oauth2-proxy in pod plausible-oauth2-proxy-75f76c4578-22f4z_ameide-prod(3b006706-c7a5-4217-ab34-7a908824e5d1)
ameide-staging                                   2m33s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474728-l85tw              Started container kubectl
ameide-prod                                      2m32s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474728-lwmfp              Started container kubectl
argocd                                           2m31s       Normal    ResourceUpdated                   application/dev-platform-camunda8                                          Updated sync status: Unknown -> Synced
ameide-staging                                   2m28s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474728-l85tw              Container image "docker.io/hashicorp/vault@sha256:62dd55c9ccbdc0af0a9269e87481a64650258907434d5ddb5e795e2eb2ac5780" already present on machine
ameide-prod                                      2m28s       Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474728-lwmfp              Container image "docker.io/hashicorp/vault@sha256:62dd55c9ccbdc0af0a9269e87481a64650258907434d5ddb5e795e2eb2ac5780" already present on machine
ameide-prod                                      2m27s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474728-lwmfp              Created container: bootstrap
ameide-staging                                   2m27s       Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474728-l85tw              Created container: bootstrap
ameide-prod                                      2m25s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474728-lwmfp              Started container bootstrap
ameide-prod                                      2m24s       Warning   BackOff                           pod/platform-backstage-666946744-r82nl                                     Back-off restarting failed container backstage in pod platform-backstage-666946744-r82nl_ameide-prod(c1d16558-85c4-41e2-b1ad-f78a8f233a38)
ameide-staging                                   2m24s       Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474728-l85tw              Started container bootstrap
ameide-prod                                      2m22s       Warning   Unhealthy                         pod/platform-langfuse-web-fdd77b54d-cx8vf                                  Readiness probe failed: Get "http://10.224.0.25:3000/api/public/ready": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
argocd                                           2m21s       Normal    Valid                             secretstore/ameide-vault                                                   store validated
ameide-dev                                       2m17s       Warning   Unhealthy                         pod/postgres-ameide-1                                                      Readiness probe failed: Get "https://10.224.0.89:8000/readyz": net/http: request canceled (Client.Timeout exceeded while awaiting headers)
clickhouse-system                                2m17s       Normal    Valid                             secretstore/ameide-vault                                                   store validated
buildkit                                         2m16s       Normal    Valid                             secretstore/ameide-vault                                                   store validated
ameide-prod                                      2m14s       Normal    Valid                             secretstore/ameide-vault                                                   store validated
arc-runners                                      2m13s       Normal    Valid                             secretstore/ameide-vault                                                   store validated
ameide-system                                    2m9s        Normal    Valid                             secretstore/ameide-vault                                                   store validated
ameide-prod                                      2m3s        Warning   BackOff                           pod/platform-langfuse-worker-5b7f94648c-sgprc                              Back-off restarting failed container langfuse-worker in pod platform-langfuse-worker-5b7f94648c-sgprc_ameide-prod(b3fad784-6eb9-4c73-bf7a-23f7e8affa8e)
arc-systems                                      114s        Normal    Valid                             secretstore/ameide-vault                                                   store validated
ameide-staging                                   110s        Normal    Killing                           pod/prometheus-staging-prometheus-prometheus-0                             Container prometheus failed startup probe, will be restarted
ameide-dev                                       109s        Normal    ClusterReconciling                nificluster/nifi                                                           NifiCluster starting reconciliation
ameide-staging                                   108s        Normal    Killing                           pod/foundation-vault-bootstrap-vault-bootstrap-29474728-l85tw              Stopping container bootstrap
ameide-prod                                      108s        Normal    Killing                           pod/foundation-vault-bootstrap-vault-bootstrap-29474728-lwmfp              Stopping container bootstrap
ameide-prod                                      108s        Normal    SuccessfulCreate                  job/foundation-vault-bootstrap-vault-bootstrap-29474729                    Created pod: foundation-vault-bootstrap-vault-bootstrap-29474729-dznql
ameide-staging                                   108s        Normal    SuccessfulCreate                  job/foundation-vault-bootstrap-vault-bootstrap-29474729                    Created pod: foundation-vault-bootstrap-vault-bootstrap-29474729-bgbm2
ameide-prod                                      102s        Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474729-dznql              Container image "docker.io/alpine/k8s@sha256:aad13a02739dd0aee5ff376f75ef9bfcc4111ccf9c0f86eef9e096a995262505" already present on machine
ameide-staging                                   102s        Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474729-bgbm2              Created container: kubectl
ameide-staging                                   102s        Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474729-bgbm2              Container image "docker.io/alpine/k8s@sha256:aad13a02739dd0aee5ff376f75ef9bfcc4111ccf9c0f86eef9e096a995262505" already present on machine
ameide-prod                                      101s        Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474729-dznql              Created container: kubectl
ameide-staging                                   100s        Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474729-bgbm2              Started container kubectl
ameide-prod                                      100s        Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474729-dznql              Started container kubectl
ameide-prod                                      96s         Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474729-dznql              Container image "docker.io/hashicorp/vault@sha256:62dd55c9ccbdc0af0a9269e87481a64650258907434d5ddb5e795e2eb2ac5780" already present on machine
ameide-staging                                   96s         Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474729-bgbm2              Container image "docker.io/hashicorp/vault@sha256:62dd55c9ccbdc0af0a9269e87481a64650258907434d5ddb5e795e2eb2ac5780" already present on machine
ameide-prod                                      95s         Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474729-dznql              Created container: bootstrap
ameide-prod                                      95s         Warning   BackOff                           pod/kubernetes-dashboard-oauth2-proxy-5f966d5694-c95qc                     Back-off restarting failed container oauth2-proxy in pod kubernetes-dashboard-oauth2-proxy-5f966d5694-c95qc_ameide-prod(673e8774-92fc-4477-acdb-e32b24dd7b83)
ameide-staging                                   95s         Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474729-bgbm2              Created container: bootstrap
ameide-prod                                      93s         Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474729-dznql              Started container bootstrap
ameide-staging                                   93s         Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474729-bgbm2              Started container bootstrap
ameide-prod                                      86s         Warning   Unhealthy                         pod/platform-camunda8-connectors-77bf5d8bd9-tm5gl                          Readiness probe failed: HTTP probe failed with statuscode: 503
ameide-ws-aed4e053-e187-455d-a9d2-e3d530a9bf4d   79s         Normal    Updated                           externalsecret/codex-account-status-sync-1                                 secret updated
ameide-dev                                       78s         Warning   Unhealthy                         pod/kafka-kafka-pool-0                                                     Readiness probe failed: command timed out: "/opt/kafka/kafka_readiness.sh" timed out after 5s
ameide-prod                                      77s         Normal    Created                           pod/plausible-oauth2-proxy-75f76c4578-22f4z                                Created container: oauth2-proxy
ameide-prod                                      77s         Normal    Pulled                            pod/plausible-oauth2-proxy-75f76c4578-22f4z                                Container image "quay.io/oauth2-proxy/oauth2-proxy:v7.6.0@sha256:dcb6ff8dd21bf3058f6a22c6fa385fa5b897a9cd3914c88a2cc2bb0a85f8065d" already present on machine
ameide-ws-91648837-bed9-4fa3-b5f9-3e5f47099782   73s         Normal    Updated                           externalsecret/codex-account-status-sync-1                                 secret updated
argocd                                           69s         Normal    ResourceUpdated                   application/cluster-crds-prometheus                                        Updated health status: Healthy -> Unknown
ameide-dev                                       67s         Normal    Updated                           externalsecret/codex-account-status-sync-1                                 secret updated
argocd                                           50s         Warning   Unhealthy                         pod/envoy-argocd-cluster-18b70826-588978797f-6cdzt                         Readiness probe failed: HTTP probe failed with statuscode: 503
ameide-staging                                   48s         Normal    SuccessfulCreate                  job/foundation-vault-bootstrap-vault-bootstrap-29474730                    Created pod: foundation-vault-bootstrap-vault-bootstrap-29474730-6bhjs
ameide-prod                                      48s         Normal    Killing                           pod/foundation-vault-bootstrap-vault-bootstrap-29474729-dznql              Stopping container bootstrap
ameide-staging                                   48s         Normal    Killing                           pod/foundation-vault-bootstrap-vault-bootstrap-29474729-bgbm2              Stopping container bootstrap
argocd                                           48s         Normal    SuccessfulCreate                  job/preview-janitor-29474730                                               Created pod: preview-janitor-29474730-ll56b
argocd                                           48s         Normal    SuccessfulCreate                  cronjob/preview-janitor                                                    Created job preview-janitor-29474730
ameide-dev                                       48s         Normal    SuccessfulCreate                  job/postgres-password-reconcile-29474730                                   Created pod: postgres-password-reconcile-29474730-pxmh2
ameide-prod                                      48s         Normal    SuccessfulCreate                  job/foundation-vault-bootstrap-vault-bootstrap-29474730                    Created pod: foundation-vault-bootstrap-vault-bootstrap-29474730-xpvvx
ameide-dev                                       48s         Normal    SuccessfulCreate                  cronjob/postgres-password-reconcile                                        Created job postgres-password-reconcile-29474730
ameide-dev                                       46s         Normal    Pulled                            pod/codex-monitor-0-55d55fd454-6hcxk                                       Container image "ghcr.io/ameideio/codex-monitor@sha256:c6562a77d0d61c378653588f87b4cdaa4c24730fb87109c670e3bcecb88356f1" already present on machine
argocd                                           45s         Normal    ResourceUpdated                   application/cluster-crds-nifikop                                           Updated health status: Healthy -> Unknown
argocd                                           40s         Warning   Unhealthy                         pod/envoy-ameide-dev-ameide-7c898b73-d67994c48-k4b87                       Readiness probe failed: HTTP probe failed with statuscode: 503
argocd                                           40s         Normal    Pulled                            pod/preview-janitor-29474730-ll56b                                         Container image "docker.io/alpine/k8s@sha256:aad13a02739dd0aee5ff376f75ef9bfcc4111ccf9c0f86eef9e096a995262505" already present on machine
argocd                                           40s         Normal    Created                           pod/preview-janitor-29474730-ll56b                                         Created container: janitor
ameide-prod                                      39s         Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474730-xpvvx              Container image "docker.io/alpine/k8s@sha256:aad13a02739dd0aee5ff376f75ef9bfcc4111ccf9c0f86eef9e096a995262505" already present on machine
ameide-prod                                      38s         Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474730-xpvvx              Created container: kubectl
ameide-dev                                       37s         Normal    Pulled                            pod/postgres-password-reconcile-29474730-pxmh2                             Container image "bitnami/kubectl@sha256:554ab88b1858e8424c55de37ad417b16f2a0e65d1607aa0f3fe3ce9b9f10b131" already present on machine
ameide-dev                                       36s         Normal    Created                           pod/postgres-password-reconcile-29474730-pxmh2                             Created container: reconcile
argocd                                           36s         Normal    Started                           pod/preview-janitor-29474730-ll56b                                         Started container janitor
ameide-prod                                      35s         Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474730-xpvvx              Started container kubectl
ameide-dev                                       32s         Normal    Started                           pod/postgres-password-reconcile-29474730-pxmh2                             Started container reconcile
ameide-staging                                   32s         Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474730-6bhjs              Container image "docker.io/alpine/k8s@sha256:aad13a02739dd0aee5ff376f75ef9bfcc4111ccf9c0f86eef9e096a995262505" already present on machine
ameide-staging                                   31s         Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474730-6bhjs              Created container: kubectl
ameide-prod                                      29s         Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474730-xpvvx              Container image "docker.io/hashicorp/vault@sha256:62dd55c9ccbdc0af0a9269e87481a64650258907434d5ddb5e795e2eb2ac5780" already present on machine
ameide-staging                                   29s         Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474730-6bhjs              Started container kubectl
ameide-prod                                      28s         Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474730-xpvvx              Created container: bootstrap
ameide-dev                                       28s         Normal    KEDAJobsCreated                   scaledjob/workrequests-toolrun-generate                                    Created 1 jobs
ameide-dev                                       28s         Normal    SuccessfulCreate                  job/workrequests-toolrun-generate-7bqd8                                    Created pod: workrequests-toolrun-generate-7bqd8-lq5rl
buildkit                                         26s         Normal    WaitForPodScheduled               persistentvolumeclaim/buildkit-state-buildkitd-1                           waiting for pod buildkitd-1 to be scheduled
ameide-prod                                      25s         Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474730-xpvvx              Started container bootstrap
ameide-staging                                   25s         Normal    Pulled                            pod/foundation-vault-bootstrap-vault-bootstrap-29474730-6bhjs              Container image "docker.io/hashicorp/vault@sha256:62dd55c9ccbdc0af0a9269e87481a64650258907434d5ddb5e795e2eb2ac5780" already present on machine
ameide-staging                                   24s         Normal    Created                           pod/foundation-vault-bootstrap-vault-bootstrap-29474730-6bhjs              Created container: bootstrap
ameide-staging                                   24s         Warning   InvalidOwnership                  pooler/postgres-ameide-rw                                                  found invalid ownership for managed resources, check logs
ameide-prod                                      24s         Warning   InvalidOwnership                  pooler/postgres-ameide-rw                                                  found invalid ownership for managed resources, check logs
ameide-ws-91648837-bed9-4fa3-b5f9-3e5f47099782   23s         Normal    Updated                           externalsecret/codex-account-status-sync-0                                 secret updated
ameide-dev                                       23s         Warning   InvalidOwnership                  pooler/postgres-ameide-rw                                                  found invalid ownership for managed resources, check logs
ameide-staging                                   21s         Normal    Started                           pod/foundation-vault-bootstrap-vault-bootstrap-29474730-6bhjs              Started container bootstrap
ameide-ws-aed4e053-e187-455d-a9d2-e3d530a9bf4d   18s         Normal    Updated                           externalsecret/codex-account-status-sync-0                                 secret updated
argocd                                           18s         Warning   Unhealthy                         pod/envoy-ameide-staging-ameide-f462b9ba-f8c755fd7-6hqtx                   Readiness probe failed: HTTP probe failed with statuscode: 503
ameide-dev                                       17s         Normal    Pulled                            pod/workrequests-toolrun-generate-7bqd8-lq5rl                              Container image "public.ecr.aws/docker/library/busybox@sha256:6b219909078e3fc93b81f83cb438bd7a5457984a01a478c76fe9777a8c67c39e" already present on machine
ameide-dev                                       16s         Normal    Created                           pod/workrequests-toolrun-generate-7bqd8-lq5rl                              Created container: ghcr-auth-init
ameide-dev                                       13s         Normal    Started                           pod/workrequests-toolrun-generate-7bqd8-lq5rl                              Started container ghcr-auth-init
ameide-dev                                       9s          Normal    Pulled                            pod/workrequests-toolrun-generate-7bqd8-lq5rl                              Container image "ghcr.io/ameideio/primitive-integration-transformation-work-executor@sha256:8f919ea8a0569deeca2f573dbc54a5336e02b6e3302cb3e79078511cde1197bc" already present on machine
argocd                                           8s          Warning   Unhealthy                         pod/envoy-ameide-prod-ameide-270d1f22-5b65497db5-l47nh                     Readiness probe failed: HTTP probe failed with statuscode: 503
ameide-dev                                       8s          Normal    Created                           pod/workrequests-toolrun-generate-7bqd8-lq5rl                              Created container: runner
ameide-ws-52841969-f3ca-4f53-a747-1864fd7e0699   4s          Normal    Updated                           externalsecret/codex-account-status-sync-0                                 secret updated
ameide-dev                                       3s          Normal    Started                           pod/workrequests-toolrun-generate-7bqd8-lq5rl                              Started container runner
ameide-dev                                       0s          Normal    KEDAJobsCreated                   scaledjob/workrequests-toolrun-verify                                      Created 1 jobs
ameide-dev                                       0s          Normal    SuccessfulCreate                  job/workrequests-toolrun-verify-sfv9p                                      Created pod: workrequests-toolrun-verify-sfv9p-hkkng
```

## Node details (describe)

### aks-system-10377240-vmss000005
```
Name:               aks-system-10377240-vmss000005
Roles:              <none>
Labels:             agentpool=system
                    ameide.io/pool=general
                    beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=Standard_B8ms
                    beta.kubernetes.io/os=linux
                    failure-domain.beta.kubernetes.io/region=westeurope
                    failure-domain.beta.kubernetes.io/zone=0
                    kubernetes.azure.com/agentpool=system
                    kubernetes.azure.com/cluster=MC_Ameide_ameide_westeurope
                    kubernetes.azure.com/consolidated-additional-properties=d533fd41-f12d-11f0-a296-fa37edc4354b
                    kubernetes.azure.com/kubelet-identity-client-id=600965a1-36d8-4d7e-97f8-0400e56700d9
                    kubernetes.azure.com/kubelet-serving-ca=cluster
                    kubernetes.azure.com/localdns-state=disabled
                    kubernetes.azure.com/mode=system
                    kubernetes.azure.com/node-image-version=AKSUbuntu-2204gen2containerd-202512.18.0
                    kubernetes.azure.com/nodepool-type=VirtualMachineScaleSets
                    kubernetes.azure.com/os-sku=Ubuntu
                    kubernetes.azure.com/os-sku-effective=Ubuntu2204
                    kubernetes.azure.com/os-sku-requested=Ubuntu
                    kubernetes.azure.com/role=agent
                    kubernetes.azure.com/sku-cpu=8
                    kubernetes.azure.com/sku-memory=32768
                    kubernetes.azure.com/storageprofile=managed
                    kubernetes.azure.com/storagetier=Premium_LRS
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=aks-system-10377240-vmss000005
                    kubernetes.io/os=linux
                    node.kubernetes.io/instance-type=Standard_B8ms
                    storageprofile=managed
                    storagetier=Premium_LRS
                    topology.disk.csi.azure.com/zone=
                    topology.kubernetes.io/region=westeurope
                    topology.kubernetes.io/zone=0
Annotations:        alpha.kubernetes.io/provided-node-ip: 10.224.0.10
                    csi.volume.kubernetes.io/nodeid:
                      {"disk.csi.azure.com":"aks-system-10377240-vmss000005","file.csi.azure.com":"aks-system-10377240-vmss000005"}
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Wed, 14 Jan 2026 14:22:18 +0000
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  aks-system-10377240-vmss000005
  AcquireTime:     <unset>
  RenewTime:       Thu, 15 Jan 2026 13:30:48 +0000
Conditions:
  Type                          Status  LastHeartbeatTime                 LastTransitionTime                Reason                          Message
  ----                          ------  -----------------                 ------------------                ------                          -------
  FrequentDockerRestart         False   Thu, 15 Jan 2026 13:25:55 +0000   Wed, 14 Jan 2026 14:23:11 +0000   NoFrequentDockerRestart         docker is functioning properly
  FrequentContainerdRestart     False   Thu, 15 Jan 2026 13:25:55 +0000   Wed, 14 Jan 2026 14:23:11 +0000   NoFrequentContainerdRestart     containerd is functioning properly
  KernelDeadlock                False   Thu, 15 Jan 2026 13:25:55 +0000   Wed, 14 Jan 2026 14:23:11 +0000   KernelHasNoDeadlock             kernel has no deadlock
  KubeletProblem                False   Thu, 15 Jan 2026 13:25:55 +0000   Wed, 14 Jan 2026 14:23:11 +0000   KubeletIsUp                     kubelet service is up
  FilesystemCorruptionProblem   False   Thu, 15 Jan 2026 13:25:55 +0000   Wed, 14 Jan 2026 14:23:11 +0000   FilesystemIsOK                  Filesystem is healthy
  VMEventScheduled              False   Thu, 15 Jan 2026 13:25:55 +0000   Wed, 14 Jan 2026 14:23:11 +0000   NoVMEventScheduled              VM has no scheduled event
  FrequentUnregisterNetDevice   False   Thu, 15 Jan 2026 13:25:55 +0000   Wed, 14 Jan 2026 14:23:11 +0000   NoFrequentUnregisterNetDevice   node is functioning properly
  ReadonlyFilesystem            False   Thu, 15 Jan 2026 13:25:55 +0000   Wed, 14 Jan 2026 14:23:11 +0000   FilesystemIsNotReadOnly         Filesystem is not read-only
  FrequentKubeletRestart        False   Thu, 15 Jan 2026 13:25:55 +0000   Wed, 14 Jan 2026 14:23:11 +0000   NoFrequentKubeletRestart        kubelet is functioning properly
  ContainerRuntimeProblem       False   Thu, 15 Jan 2026 13:25:55 +0000   Wed, 14 Jan 2026 14:23:11 +0000   ContainerRuntimeIsUp            container runtime service is up
  MemoryPressure                False   Thu, 15 Jan 2026 13:30:35 +0000   Wed, 14 Jan 2026 14:22:18 +0000   KubeletHasSufficientMemory      kubelet has sufficient memory available
  DiskPressure                  False   Thu, 15 Jan 2026 13:30:35 +0000   Wed, 14 Jan 2026 14:22:18 +0000   KubeletHasNoDiskPressure        kubelet has no disk pressure
  PIDPressure                   False   Thu, 15 Jan 2026 13:30:35 +0000   Wed, 14 Jan 2026 14:22:18 +0000   KubeletHasSufficientPID         kubelet has sufficient PID available
  Ready                         True    Thu, 15 Jan 2026 13:30:35 +0000   Wed, 14 Jan 2026 14:22:52 +0000   KubeletReady                    kubelet is posting ready status
Addresses:
  InternalIP:  10.224.0.10
  Hostname:    aks-system-10377240-vmss000005
Capacity:
  cpu:                8
  ephemeral-storage:  129886128Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             32859916Ki
  pods:               110
Allocatable:
  cpu:                7820m
  ephemeral-storage:  119703055367
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             30453516Ki
  pods:               110
System Info:
  Machine ID:                 7f4a9e25ab784984b592865b81e9b4a4
  System UUID:                6938f422-dbd3-4654-9220-e4836e2f0243
  Boot ID:                    e281fb7a-48e4-4640-89cb-d1de3fda5fe8
  Kernel Version:             5.15.0-1102-azure
  OS Image:                   Ubuntu 22.04.5 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.7.29-1
  Kubelet Version:            v1.32.7
  Kube-Proxy Version:         v1.32.7
ProviderID:                   azure:///subscriptions/68ba6806-178c-41f1-84ec-d7a839799be1/resourceGroups/mc_ameide_ameide_westeurope/providers/Microsoft.Compute/virtualMachineScaleSets/aks-system-10377240-vmss/virtualMachines/5
Non-terminated Pods:          (74 in total)
  Namespace                   Name                                                    CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                                    ------------  ----------  ---------------  -------------  ---
  ameide-dev                  platform-loki-0                                         100m (1%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-dev                  postgres-ameide-2                                       500m (6%)     2 (25%)     1Gi (3%)         4Gi (13%)      23h
  ameide-prod                 data-minio-d847bc9f5-75644                              75m (0%)      150m (1%)   512Mi (1%)       768Mi (2%)     23h
  ameide-prod                 pgadmin-765f9b586d-45725                                100m (1%)     500m (6%)   256Mi (0%)       512Mi (1%)     23h
  ameide-prod                 platform-camunda8-elasticsearch-master-0                250m (3%)     1 (12%)     1Gi (3%)         2Gi (6%)       23h
  ameide-prod                 platform-camunda8-zeebe-0                               250m (3%)     1 (12%)     1Gi (3%)         2Gi (6%)       23h
  ameide-prod                 platform-grafana-68b74b9677-xtn5b                       40m (0%)      120m (1%)   160Mi (0%)       320Mi (1%)     23h
  ameide-prod                 platform-loki-0                                         100m (1%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-prod                 temporal-frontend-699f67d6c4-szvd7                      250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       21h
  ameide-prod                 temporal-history-777987c4dd-rww9v                       250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       21h
  ameide-prod                 temporal-matching-7ccd56bd5d-xxzk9                      250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       21h
  ameide-prod                 temporal-worker-7987489cbb-flhwl                        250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       21h
  ameide-prod                 threads-7757d9dd7d-l9jgt                                100m (1%)     200m (2%)   256Mi (0%)       512Mi (1%)     21h
  ameide-prod                 www-ameide-platform-59784df769-9xktb                    50m (0%)      150m (1%)   160Mi (0%)       320Mi (1%)     21h
  ameide-staging              alloy-logs-757fd756b5-7fbpx                             60m (0%)      500m (6%)   178Mi (0%)       256Mi (0%)     21h
  ameide-staging              data-minio-69b56b9945-m88cc                             50m (0%)      150m (1%)   384Mi (1%)       512Mi (1%)     23h
  ameide-staging              data-minio-console-84d76b546-4x5m6                      100m (1%)     150m (1%)   128Mi (0%)       192Mi (0%)     21h
  ameide-staging              inference-8588ddc55b-q9m8v                              100m (1%)     200m (2%)   512Mi (1%)       1Gi (3%)       21h
  ameide-staging              kafka-entity-operator-6f98dd987f-fnbm4                  200m (2%)     500m (6%)   512Mi (1%)       1Gi (3%)       21h
  ameide-staging              kafka-kafka-pool-0                                      100m (1%)     250m (3%)   384Mi (1%)       768Mi (2%)     20h
  ameide-staging              kubernetes-dashboard-api-b855c6774-mb2bt                100m (1%)     250m (3%)   200Mi (0%)       400Mi (1%)     21h
  ameide-staging              kubernetes-dashboard-auth-7ff56b864b-7c7j6              100m (1%)     250m (3%)   200Mi (0%)       400Mi (1%)     21h
  ameide-staging              pgadmin-65885f75d8-vsbnn                                100m (1%)     500m (6%)   256Mi (0%)       512Mi (1%)     23h
  ameide-staging              platform-camunda8-elasticsearch-master-0                250m (3%)     1 (12%)     1Gi (3%)         2Gi (6%)       23h
  ameide-staging              platform-camunda8-zeebe-0                               250m (3%)     1 (12%)     1Gi (3%)         2Gi (6%)       23h
  ameide-staging              platform-extensions-runtime-cffbf75c8-kcvhk             250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       21h
  ameide-staging              platform-langfuse-web-6f69f9fb94-6h248                  100m (1%)     0 (0%)      256Mi (0%)       1Gi (3%)       21h
  ameide-staging              platform-langfuse-worker-6f69db6cf9-c4s7v               100m (1%)     0 (0%)      256Mi (0%)       1Gi (3%)       21h
  ameide-staging              platform-loki-0                                         100m (1%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-staging              plausible-plausible-78ccc7bb75-qqz2d                    200m (2%)     500m (6%)   512Mi (1%)       768Mi (2%)     21h
  ameide-staging              postgres-ameide-2                                       500m (6%)     2 (25%)     1Gi (3%)         4Gi (13%)      23h
  ameide-staging              rfr-redis-0                                             25m (0%)      50m (0%)    50Mi (0%)        100Mi (0%)     23h
  ameide-staging              rfr-redis-1                                             25m (0%)      50m (0%)    50Mi (0%)        100Mi (0%)     23h
  ameide-staging              rfs-redis-654d8b497c-kgpzj                              25m (0%)      50m (0%)    50Mi (0%)        100Mi (0%)     21h
  ameide-staging              temporal-admintools-796d756458-59lmr                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         21h
  ameide-staging              temporal-frontend-6d776cb8d-kjzsp                       250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       21h
  ameide-staging              temporal-history-545987dcfd-74jvp                       250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       21h
  ameide-staging              temporal-matching-58c96bd6cb-4v279                      250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       21h
  ameide-staging              temporal-ui-589d9b5ddc-rzkwd                            0 (0%)        0 (0%)      0 (0%)           0 (0%)         21h
  ameide-staging              temporal-worker-86c548b76d-xth9j                        250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       21h
  ameide-staging              traffic-manager-54b776b96b-bqc8p                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         21h
  ameide-staging              transformation-v0-domain-67b8b6cc6c-hqlrg               0 (0%)        0 (0%)      0 (0%)           0 (0%)         21h
  ameide-staging              transformation-v0-projection-7fcbcb7884-dqghh           0 (0%)        0 (0%)      0 (0%)           0 (0%)         21h
  ameide-staging              vault-core-staging-agent-injector-85d66b4db7-lktjb      0 (0%)        0 (0%)      0 (0%)           0 (0%)         21h
  ameide-staging              workflows-79494546fc-6j8x8                              0 (0%)        0 (0%)      0 (0%)           0 (0%)         21h
  arc-systems                 arc-aks-696897c5-listener                               0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  argocd                      argocd-applicationset-controller-6c55d779dd-j66tr       0 (0%)        0 (0%)      0 (0%)           0 (0%)         21h
  argocd                      argocd-dex-server-f8b98dd59-wrtkd                       0 (0%)        0 (0%)      0 (0%)           0 (0%)         18h
  argocd                      argocd-notifications-controller-687cd7c44-rvqgr         0 (0%)        0 (0%)      0 (0%)           0 (0%)         21h
  argocd                      argocd-redis-5ff45f994-kzq76                            0 (0%)        0 (0%)      0 (0%)           0 (0%)         18h
  argocd                      argocd-repo-server-8455f5b4d5-vzb6t                     250m (3%)     2 (25%)     512Mi (1%)       1Gi (3%)       58m
  argocd                      envoy-ameide-dev-ameide-7c898b73-d67994c48-4vp5n        110m (1%)     0 (0%)      544Mi (1%)       0 (0%)         17h
  argocd                      envoy-ameide-prod-ameide-270d1f22-5b65497db5-kfvwh      110m (1%)     0 (0%)      544Mi (1%)       0 (0%)         17h
  argocd                      envoy-ameide-staging-ameide-f462b9ba-f8c755fd7-l4cmm    110m (1%)     0 (0%)      544Mi (1%)       0 (0%)         17h
  argocd                      envoy-argocd-cluster-18b70826-588978797f-7ks8f          110m (1%)     0 (0%)      544Mi (1%)       0 (0%)         15h
  argocd                      envoy-gateway-77dd576b87-vsjp2                          100m (1%)     0 (0%)      256Mi (0%)       512Mi (1%)     17h
  argocd                      nifikop-6c59f9d8cd-l9r82                                10m (0%)      1 (12%)     64Mi (0%)        512Mi (1%)     21h
  buildkit                    binfmt-skvdg                                            0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  cert-manager                cert-manager-7fbd79966f-8tmjj                           10m (0%)      200m (2%)   64Mi (0%)        256Mi (0%)     21h
  cert-manager                cert-manager-webhook-b575668fb-m9vxr                    10m (0%)      200m (2%)   64Mi (0%)        256Mi (0%)     21h
  clickhouse-system           clickhouse-operator-7797bd574d-qz52f                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         21h
  external-secrets            external-secrets-5c9bbdbb75-bg89s                       0 (0%)        0 (0%)      0 (0%)           0 (0%)         21h
  external-secrets            external-secrets-webhook-b9444bf8b-7rt2x                0 (0%)        0 (0%)      0 (0%)           0 (0%)         21h
  kube-system                 azure-cns-wlnph                                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  kube-system                 azure-ip-masq-agent-znthf                               50m (0%)      500m (6%)   36Mi (0%)        250Mi (0%)     23h
  kube-system                 azure-npm-mtrgk                                         50m (0%)      250m (3%)   300Mi (1%)       1000Mi (3%)    23h
  kube-system                 cloud-node-manager-cz4p6                                50m (0%)      0 (0%)      50Mi (0%)        512Mi (1%)     23h
  kube-system                 csi-azuredisk-node-9r77w                                30m (0%)      0 (0%)      60Mi (0%)        5800Mi (19%)   23h
  kube-system                 csi-azurefile-node-jbl96                                30m (0%)      0 (0%)      60Mi (0%)        600Mi (2%)     23h
  kube-system                 konnectivity-agent-56489d589b-g5jjm                     20m (0%)      1 (12%)     20Mi (0%)        1Gi (3%)       62m
  kube-system                 kube-proxy-6dks9                                        100m (1%)     0 (0%)      0 (0%)           0 (0%)         23h
  kube-system                 metrics-server-6fbdfd747b-269p4                         158m (2%)     253m (3%)   150Mi (0%)       420Mi (1%)     23h
  kube-system                 metrics-server-6fbdfd747b-2wlkw                         158m (2%)     253m (3%)   150Mi (0%)       420Mi (1%)     20h
  prometheus-system           prometheus-operator-prometheus-node-exporter-k2mfg      0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests       Limits
  --------           --------       ------
  cpu                7766m (99%)    30176m (385%)
  memory             21522Mi (72%)  51894Mi (174%)
  ephemeral-storage  150Mi (0%)     6Gi (5%)
  hugepages-1Gi      0 (0%)         0 (0%)
  hugepages-2Mi      0 (0%)         0 (0%)
Events:              <none>
```

### aks-system-10377240-vmss000007
```
Name:               aks-system-10377240-vmss000007
Roles:              <none>
Labels:             agentpool=system
                    ameide.io/pool=general
                    beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=Standard_B8ms
                    beta.kubernetes.io/os=linux
                    failure-domain.beta.kubernetes.io/region=westeurope
                    failure-domain.beta.kubernetes.io/zone=0
                    kubernetes.azure.com/agentpool=system
                    kubernetes.azure.com/cluster=MC_Ameide_ameide_westeurope
                    kubernetes.azure.com/consolidated-additional-properties=d533fd41-f12d-11f0-a296-fa37edc4354b
                    kubernetes.azure.com/kubelet-identity-client-id=600965a1-36d8-4d7e-97f8-0400e56700d9
                    kubernetes.azure.com/kubelet-serving-ca=cluster
                    kubernetes.azure.com/localdns-state=disabled
                    kubernetes.azure.com/mode=system
                    kubernetes.azure.com/node-image-version=AKSUbuntu-2204gen2containerd-202512.18.0
                    kubernetes.azure.com/nodepool-type=VirtualMachineScaleSets
                    kubernetes.azure.com/os-sku=Ubuntu
                    kubernetes.azure.com/os-sku-effective=Ubuntu2204
                    kubernetes.azure.com/os-sku-requested=Ubuntu
                    kubernetes.azure.com/role=agent
                    kubernetes.azure.com/sku-cpu=8
                    kubernetes.azure.com/sku-memory=32768
                    kubernetes.azure.com/storageprofile=managed
                    kubernetes.azure.com/storagetier=Premium_LRS
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=aks-system-10377240-vmss000007
                    kubernetes.io/os=linux
                    node.kubernetes.io/instance-type=Standard_B8ms
                    storageprofile=managed
                    storagetier=Premium_LRS
                    topology.disk.csi.azure.com/zone=
                    topology.kubernetes.io/region=westeurope
                    topology.kubernetes.io/zone=0
Annotations:        alpha.kubernetes.io/provided-node-ip: 10.224.0.199
                    csi.volume.kubernetes.io/nodeid:
                      {"disk.csi.azure.com":"aks-system-10377240-vmss000007","file.csi.azure.com":"aks-system-10377240-vmss000007"}
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Wed, 14 Jan 2026 14:22:22 +0000
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  aks-system-10377240-vmss000007
  AcquireTime:     <unset>
  RenewTime:       Thu, 15 Jan 2026 13:30:40 +0000
Conditions:
  Type                          Status  LastHeartbeatTime                 LastTransitionTime                Reason                          Message
  ----                          ------  -----------------                 ------------------                ------                          -------
  FrequentUnregisterNetDevice   False   Thu, 15 Jan 2026 13:26:07 +0000   Wed, 14 Jan 2026 14:22:55 +0000   NoFrequentUnregisterNetDevice   node is functioning properly
  KernelDeadlock                False   Thu, 15 Jan 2026 13:26:07 +0000   Wed, 14 Jan 2026 14:22:55 +0000   KernelHasNoDeadlock             kernel has no deadlock
  FilesystemCorruptionProblem   False   Thu, 15 Jan 2026 13:26:07 +0000   Wed, 14 Jan 2026 14:22:55 +0000   FilesystemIsOK                  Filesystem is healthy
  FrequentDockerRestart         False   Thu, 15 Jan 2026 13:26:07 +0000   Wed, 14 Jan 2026 14:22:55 +0000   NoFrequentDockerRestart         docker is functioning properly
  FrequentContainerdRestart     False   Thu, 15 Jan 2026 13:26:07 +0000   Wed, 14 Jan 2026 14:22:55 +0000   NoFrequentContainerdRestart     containerd is functioning properly
  ReadonlyFilesystem            False   Thu, 15 Jan 2026 13:26:07 +0000   Wed, 14 Jan 2026 14:22:55 +0000   FilesystemIsNotReadOnly         Filesystem is not read-only
  FrequentKubeletRestart        False   Thu, 15 Jan 2026 13:26:07 +0000   Wed, 14 Jan 2026 14:22:55 +0000   NoFrequentKubeletRestart        kubelet is functioning properly
  KubeletProblem                False   Thu, 15 Jan 2026 13:26:07 +0000   Wed, 14 Jan 2026 14:22:55 +0000   KubeletIsUp                     kubelet service is up
  ContainerRuntimeProblem       False   Thu, 15 Jan 2026 13:26:07 +0000   Wed, 14 Jan 2026 14:22:55 +0000   ContainerRuntimeIsUp            container runtime service is up
  VMEventScheduled              False   Thu, 15 Jan 2026 13:26:07 +0000   Wed, 14 Jan 2026 14:23:24 +0000   NoVMEventScheduled              VM has no scheduled event
  MemoryPressure                False   Thu, 15 Jan 2026 13:28:52 +0000   Thu, 15 Jan 2026 09:46:15 +0000   KubeletHasSufficientMemory      kubelet has sufficient memory available
  DiskPressure                  False   Thu, 15 Jan 2026 13:28:52 +0000   Thu, 15 Jan 2026 09:46:15 +0000   KubeletHasNoDiskPressure        kubelet has no disk pressure
  PIDPressure                   False   Thu, 15 Jan 2026 13:28:52 +0000   Thu, 15 Jan 2026 09:46:15 +0000   KubeletHasSufficientPID         kubelet has sufficient PID available
  Ready                         True    Thu, 15 Jan 2026 13:28:52 +0000   Thu, 15 Jan 2026 12:32:25 +0000   KubeletReady                    kubelet is posting ready status
Addresses:
  InternalIP:  10.224.0.199
  Hostname:    aks-system-10377240-vmss000007
Capacity:
  cpu:                8
  ephemeral-storage:  129886128Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             32864156Ki
  pods:               110
Allocatable:
  cpu:                7820m
  ephemeral-storage:  119703055367
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             30457756Ki
  pods:               110
System Info:
  Machine ID:                 456a22af8dfe40c8bfb753705ce89c2e
  System UUID:                97ceef8a-25d6-4c3a-a350-c71fb62694c3
  Boot ID:                    b1de35e0-2839-4627-8fb9-8ac5708b4092
  Kernel Version:             5.15.0-1102-azure
  OS Image:                   Ubuntu 22.04.5 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.7.29-1
  Kubelet Version:            v1.32.7
  Kube-Proxy Version:         v1.32.7
ProviderID:                   azure:///subscriptions/68ba6806-178c-41f1-84ec-d7a839799be1/resourceGroups/mc_ameide_ameide_westeurope/providers/Microsoft.Compute/virtualMachineScaleSets/aks-system-10377240-vmss/virtualMachines/7
Non-terminated Pods:          (39 in total)
  Namespace                   Name                                                               CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                                               ------------  ----------  ---------------  -------------  ---
  ameide-dev                  data-minio-5db69d469b-rmfdd                                        250m (3%)     375m (4%)   256Mi (0%)       384Mi (1%)     37m
  ameide-dev                  foundation-vault-bootstrap-vault-bootstrap-29474719-jnf2b          0 (0%)        0 (0%)      0 (0%)           0 (0%)         11m
  ameide-dev                  kafka-kafka-pool-0                                                 100m (1%)     250m (3%)   384Mi (1%)       768Mi (2%)     17m
  ameide-dev                  pgadmin-7b885f77c8-4m68j                                           100m (1%)     500m (6%)   256Mi (0%)       512Mi (1%)     37m
  ameide-dev                  platform-camunda8-elasticsearch-master-0                           250m (3%)     1 (12%)     1Gi (3%)         2Gi (6%)       34m
  ameide-dev                  platform-camunda8-zeebe-0                                          250m (3%)     1 (12%)     1Gi (3%)         2Gi (6%)       33m
  ameide-dev                  postgres-password-reconcile-29474730-pxmh2                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         50s
  ameide-dev                  workrequests-toolrun-generate-7bqd8-lq5rl                          0 (0%)        0 (0%)      0 (0%)           0 (0%)         30s
  ameide-dev                  workrequests-toolrun-verify-sfv9p-hkkng                            0 (0%)        0 (0%)      0 (0%)           0 (0%)         2s
  ameide-prod                 foundation-vault-bootstrap-vault-bootstrap-29474730-xpvvx          0 (0%)        0 (0%)      0 (0%)           0 (0%)         50s
  ameide-prod                 kubernetes-dashboard-metrics-scraper-production-646775fc4bzkcv7    100m (1%)     250m (3%)   200Mi (0%)       400Mi (1%)     37m
  ameide-prod                 kubernetes-dashboard-web-849fc4997f-7x6pq                          100m (1%)     250m (3%)   200Mi (0%)       400Mi (1%)     37m
  ameide-prod                 platform-camunda8-connectors-77bf5d8bd9-tm5gl                      250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       37m
  ameide-prod                 platform-extensions-runtime-587694db5b-5cdvv                       250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       37m
  ameide-prod                 platform-extensions-runtime-587694db5b-8b9np                       250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       37m
  ameide-prod                 platform-langfuse-web-fdd77b54d-cx8vf                              100m (1%)     0 (0%)      256Mi (0%)       1Gi (3%)       37m
  ameide-prod                 platform-langfuse-worker-5b7f94648c-sgprc                          100m (1%)     0 (0%)      256Mi (0%)       1Gi (3%)       37m
  ameide-prod                 plausible-oauth2-proxy-75f76c4578-22f4z                            100m (1%)     150m (1%)   128Mi (0%)       192Mi (0%)     37m
  ameide-prod                 plausible-plausible-6b5c446b68-zzxpn                               250m (3%)     1 (12%)     768Mi (2%)       1536Mi (5%)    37m
  ameide-staging              foundation-vault-bootstrap-vault-bootstrap-29474730-6bhjs          0 (0%)        0 (0%)      0 (0%)           0 (0%)         50s
  ameide-staging              www-ameide-77f7cfdd75-r6zbp                                        250m (3%)     1 (12%)     256Mi (0%)       1Gi (3%)       37m
  ameide-staging              www-ameide-77f7cfdd75-vptxv                                        250m (3%)     1 (12%)     256Mi (0%)       1Gi (3%)       37m
  argocd                      argocd-application-controller-1                                    250m (3%)     1 (12%)     2Gi (6%)         4Gi (13%)      34m
  argocd                      envoy-argocd-ameide-staging-grpc-internal-851fe813-74d47d7m4zgd    110m (1%)     0 (0%)      544Mi (1%)       0 (0%)         37m
  argocd                      preview-janitor-29474730-ll56b                                     0 (0%)        0 (0%)      0 (0%)           0 (0%)         50s
  buildkit                    binfmt-rsjt2                                                       0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  keda-system                 keda-operator-metrics-apiserver-5665f89bc7-z29js                   100m (1%)     1 (12%)     100Mi (0%)       1000Mi (3%)    37m
  keycloak-system             keycloak-operator-65864689db-jtflg                                 300m (3%)     700m (8%)   450Mi (1%)       450Mi (1%)     37m
  kube-system                 azure-cns-vk2q4                                                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  kube-system                 azure-ip-masq-agent-4x4kb                                          50m (0%)      500m (6%)   36Mi (0%)        250Mi (0%)     3h17m
  kube-system                 azure-npm-hc4d5                                                    50m (0%)      250m (3%)   300Mi (1%)       1000Mi (3%)    23h
  kube-system                 cloud-node-manager-j4q44                                           50m (0%)      0 (0%)      50Mi (0%)        512Mi (1%)     23h
  kube-system                 csi-azuredisk-node-ztlph                                           30m (0%)      0 (0%)      60Mi (0%)        5800Mi (19%)   23h
  kube-system                 csi-azurefile-node-h5mwp                                           30m (0%)      0 (0%)      60Mi (0%)        600Mi (2%)     23h
  kube-system                 kube-proxy-vznmc                                                   100m (1%)     0 (0%)      0 (0%)           0 (0%)         23h
  prometheus-system           prometheus-operator-kube-p-operator-5cb5bc77fc-56fd5               100m (1%)     200m (2%)   100Mi (0%)       200Mi (0%)     37m
  prometheus-system           prometheus-operator-prometheus-node-exporter-p46bn                 0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  redis-system                redis-operator-db6768894-9m74x                                     250m (3%)     1 (12%)     256Mi (0%)       512Mi (1%)     37m
  strimzi-system              strimzi-cluster-operator-5488759995-45zpm                          200m (2%)     1 (12%)     384Mi (1%)       384Mi (1%)     37m
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests       Limits
  --------           --------       ------
  cpu                4570m (58%)    15425m (197%)
  memory             11188Mi (37%)  30260Mi (101%)
  ephemeral-storage  150Mi (0%)     6Gi (5%)
  hugepages-1Gi      0 (0%)         0 (0%)
  hugepages-2Mi      0 (0%)         0 (0%)
Events:
  Type    Reason              Age                 From     Message
  ----    ------              ----                ----     -------
  Normal  NodeNotReady        59m (x13 over 18h)  kubelet  Node aks-system-10377240-vmss000007 status is now: NodeNotReady
  Normal  NodeReady           58m (x15 over 23h)  kubelet  Node aks-system-10377240-vmss000007 status is now: NodeReady
  Normal  NodeNotSchedulable  37m                 kubelet  Node aks-system-10377240-vmss000007 status is now: NodeNotSchedulable
  Normal  NodeSchedulable     22m                 kubelet  Node aks-system-10377240-vmss000007 status is now: NodeSchedulable
```

### aks-system-10377240-vmss000008
```
Name:               aks-system-10377240-vmss000008
Roles:              <none>
Labels:             agentpool=system
                    ameide.io/pool=general
                    beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=Standard_B8ms
                    beta.kubernetes.io/os=linux
                    failure-domain.beta.kubernetes.io/region=westeurope
                    failure-domain.beta.kubernetes.io/zone=0
                    kubernetes.azure.com/agentpool=system
                    kubernetes.azure.com/cluster=MC_Ameide_ameide_westeurope
                    kubernetes.azure.com/consolidated-additional-properties=d533fd41-f12d-11f0-a296-fa37edc4354b
                    kubernetes.azure.com/kubelet-identity-client-id=600965a1-36d8-4d7e-97f8-0400e56700d9
                    kubernetes.azure.com/kubelet-serving-ca=cluster
                    kubernetes.azure.com/localdns-state=disabled
                    kubernetes.azure.com/mode=system
                    kubernetes.azure.com/node-image-version=AKSUbuntu-2204gen2containerd-202512.18.0
                    kubernetes.azure.com/nodepool-type=VirtualMachineScaleSets
                    kubernetes.azure.com/os-sku=Ubuntu
                    kubernetes.azure.com/os-sku-effective=Ubuntu2204
                    kubernetes.azure.com/os-sku-requested=Ubuntu
                    kubernetes.azure.com/role=agent
                    kubernetes.azure.com/sku-cpu=8
                    kubernetes.azure.com/sku-memory=32768
                    kubernetes.azure.com/storageprofile=managed
                    kubernetes.azure.com/storagetier=Premium_LRS
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=aks-system-10377240-vmss000008
                    kubernetes.io/os=linux
                    node.kubernetes.io/instance-type=Standard_B8ms
                    storageprofile=managed
                    storagetier=Premium_LRS
                    topology.disk.csi.azure.com/zone=
                    topology.kubernetes.io/region=westeurope
                    topology.kubernetes.io/zone=0
Annotations:        alpha.kubernetes.io/provided-node-ip: 10.224.1.166
                    csi.volume.kubernetes.io/nodeid:
                      {"disk.csi.azure.com":"aks-system-10377240-vmss000008","file.csi.azure.com":"aks-system-10377240-vmss000008"}
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Wed, 14 Jan 2026 14:22:03 +0000
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  aks-system-10377240-vmss000008
  AcquireTime:     <unset>
  RenewTime:       Thu, 15 Jan 2026 13:30:41 +0000
Conditions:
  Type                          Status  LastHeartbeatTime                 LastTransitionTime                Reason                          Message
  ----                          ------  -----------------                 ------------------                ------                          -------
  FrequentContainerdRestart     False   Thu, 15 Jan 2026 13:30:21 +0000   Wed, 14 Jan 2026 14:22:38 +0000   NoFrequentContainerdRestart     containerd is functioning properly
  VMEventScheduled              False   Thu, 15 Jan 2026 13:30:21 +0000   Wed, 14 Jan 2026 14:22:38 +0000   NoVMEventScheduled              VM has no scheduled event
  KubeletProblem                False   Thu, 15 Jan 2026 13:30:21 +0000   Wed, 14 Jan 2026 14:22:38 +0000   KubeletIsUp                     kubelet service is up
  ContainerRuntimeProblem       False   Thu, 15 Jan 2026 13:30:21 +0000   Wed, 14 Jan 2026 14:22:38 +0000   ContainerRuntimeIsUp            container runtime service is up
  FrequentKubeletRestart        False   Thu, 15 Jan 2026 13:30:21 +0000   Wed, 14 Jan 2026 14:22:38 +0000   NoFrequentKubeletRestart        kubelet is functioning properly
  FrequentDockerRestart         False   Thu, 15 Jan 2026 13:30:21 +0000   Wed, 14 Jan 2026 14:22:38 +0000   NoFrequentDockerRestart         docker is functioning properly
  FrequentUnregisterNetDevice   False   Thu, 15 Jan 2026 13:30:21 +0000   Wed, 14 Jan 2026 14:22:38 +0000   NoFrequentUnregisterNetDevice   node is functioning properly
  KernelDeadlock                False   Thu, 15 Jan 2026 13:30:21 +0000   Wed, 14 Jan 2026 14:22:38 +0000   KernelHasNoDeadlock             kernel has no deadlock
  ReadonlyFilesystem            False   Thu, 15 Jan 2026 13:30:21 +0000   Wed, 14 Jan 2026 14:22:38 +0000   FilesystemIsNotReadOnly         Filesystem is not read-only
  FilesystemCorruptionProblem   False   Thu, 15 Jan 2026 13:30:21 +0000   Wed, 14 Jan 2026 14:22:38 +0000   FilesystemIsOK                  Filesystem is healthy
  MemoryPressure                False   Thu, 15 Jan 2026 13:27:47 +0000   Wed, 14 Jan 2026 14:22:03 +0000   KubeletHasSufficientMemory      kubelet has sufficient memory available
  DiskPressure                  False   Thu, 15 Jan 2026 13:27:47 +0000   Wed, 14 Jan 2026 14:22:03 +0000   KubeletHasNoDiskPressure        kubelet has no disk pressure
  PIDPressure                   False   Thu, 15 Jan 2026 13:27:47 +0000   Wed, 14 Jan 2026 14:22:03 +0000   KubeletHasSufficientPID         kubelet has sufficient PID available
  Ready                         True    Thu, 15 Jan 2026 13:27:47 +0000   Wed, 14 Jan 2026 14:22:17 +0000   KubeletReady                    kubelet is posting ready status
Addresses:
  InternalIP:  10.224.1.166
  Hostname:    aks-system-10377240-vmss000008
Capacity:
  cpu:                8
  ephemeral-storage:  129886128Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             32864152Ki
  pods:               110
Allocatable:
  cpu:                7820m
  ephemeral-storage:  119703055367
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             30457752Ki
  pods:               110
System Info:
  Machine ID:                 74156679085b45b0a88ada6a41f605c7
  System UUID:                0aca804f-f714-4bf1-a0d9-33bff75131da
  Boot ID:                    b206c13b-5fc5-4c86-be33-15810b59b133
  Kernel Version:             5.15.0-1102-azure
  OS Image:                   Ubuntu 22.04.5 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.7.29-1
  Kubelet Version:            v1.32.7
  Kube-Proxy Version:         v1.32.7
ProviderID:                   azure:///subscriptions/68ba6806-178c-41f1-84ec-d7a839799be1/resourceGroups/mc_ameide_ameide_westeurope/providers/Microsoft.Compute/virtualMachineScaleSets/aks-system-10377240-vmss/virtualMachines/8
Non-terminated Pods:          (110 in total)
  Namespace                   Name                                                               CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                                               ------------  ----------  ---------------  -------------  ---
  ameide-dev                  agents-54f464cdbf-962bb                                            0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  agents-runtime-7f6c886bbd-ppmj9                                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  alloy-logs-7c764845f5-m4gcn                                        60m (0%)      500m (6%)   178Mi (0%)       256Mi (0%)     23h
  ameide-dev                  ameide-dev-platform-reloader-reloader-5b45db694-p6lnc              10m (0%)      100m (1%)   32Mi (0%)        128Mi (0%)     23h
  ameide-dev                  data-minio-console-795546b6dd-d2kfx                                100m (1%)     150m (1%)   128Mi (0%)       192Mi (0%)     23h
  ameide-dev                  data-pgadmin-servers-pgadmin-servers-58f5f8855c-vkckq              0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  inference-5b85cc48d6-k6pgp                                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  inference-gateway-5c785c6877-vmqrn                                 100m (1%)     500m (6%)   128Mi (0%)       256Mi (0%)     23h
  ameide-dev                  kafka-entity-operator-86d4646845-tfvjf                             200m (2%)     500m (6%)   512Mi (1%)       1Gi (3%)       23h
  ameide-dev                  kubernetes-dashboard-api-5fd4d66f75-psvcz                          100m (1%)     250m (3%)   200Mi (0%)       400Mi (1%)     23h
  ameide-dev                  kubernetes-dashboard-auth-7c8dd97946-m6l7s                         100m (1%)     250m (3%)   200Mi (0%)       400Mi (1%)     23h
  ameide-dev                  kubernetes-dashboard-kong-85f664bd9c-hcdjn                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  kubernetes-dashboard-metrics-scraper-dev-ddb5b6bcb-s56vm           100m (1%)     250m (3%)   200Mi (0%)       400Mi (1%)     23h
  ameide-dev                  kubernetes-dashboard-oauth2-proxy-78d777f756-hz45c                 100m (1%)     150m (1%)   128Mi (0%)       192Mi (0%)     23h
  ameide-dev                  kubernetes-dashboard-web-8b96684bf-b7hgc                           100m (1%)     250m (3%)   200Mi (0%)       400Mi (1%)     23h
  ameide-dev                  otel-collector-b84dd4f9-rrxdg                                      0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  platform-54c5d4c958-md2hg                                          0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  platform-backstage-74b5946866-w4jtt                                100m (1%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-dev                  platform-camunda8-connectors-55c49b89cd-v6h7g                      250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-dev                  platform-camunda8-identity-5fdcff6856-lpmbl                        250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-dev                  platform-camunda8-web-modeler-restapi-54dbfb54d7-4x5hm             250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-dev                  platform-camunda8-web-modeler-webapp-5bb9d98868-6xm2z              250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-dev                  platform-camunda8-web-modeler-websockets-b6d98b8d6-8sgvj           250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-dev                  platform-extensions-runtime-7486f6c67-4mrf2                        250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-dev                  platform-extensions-runtime-7486f6c67-5mlh4                        250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-dev                  platform-langfuse-web-767f9ff6bd-wg8ss                             100m (1%)     0 (0%)      256Mi (0%)       1Gi (3%)       23h
  ameide-dev                  platform-langfuse-worker-b4fd8b77-d9fg6                            100m (1%)     0 (0%)      256Mi (0%)       1Gi (3%)       23h
  ameide-dev                  platform-prometheus-kube-state-metrics-df5d9c749-tpp7z             0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  plausible-oauth2-proxy-765b4b4dd4-pvvnj                            100m (1%)     150m (1%)   128Mi (0%)       192Mi (0%)     23h
  ameide-dev                  plausible-plausible-5f97d8dd58-m2p7r                               100m (1%)     500m (6%)   512Mi (1%)       768Mi (2%)     23h
  ameide-dev                  prometheus-dev-prometheus-prometheus-0                             0 (0%)        0 (0%)      0 (0%)           0 (0%)         19h
  ameide-dev                  rfr-redis-0                                                        25m (0%)      50m (0%)    50Mi (0%)        100Mi (0%)     33m
  ameide-dev                  rfs-redis-85b9c8b98c-9czgx                                         25m (0%)      50m (0%)    50Mi (0%)        100Mi (0%)     23h
  ameide-dev                  rfs-redis-85b9c8b98c-t9qtj                                         25m (0%)      50m (0%)    50Mi (0%)        100Mi (0%)     23h
  ameide-dev                  temporal-admintools-577f786c4b-5vd9s                               0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  temporal-frontend-745c5b47dd-bpvnh                                 250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-dev                  temporal-frontend-745c5b47dd-zt2cj                                 250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-dev                  temporal-history-d4d964ff8-fscgp                                   250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-dev                  temporal-history-d4d964ff8-tmwv5                                   250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-dev                  temporal-matching-67cbb6c464-247t5                                 250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-dev                  temporal-matching-67cbb6c464-2qvt7                                 250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-dev                  temporal-ui-9b65875b7-qx8hb                                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  temporal-worker-5687bd7784-7tcxp                                   250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-dev                  temporal-worker-5687bd7784-ptxlz                                   250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       23h
  ameide-dev                  threads-747db79645-g68xf                                           0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  traffic-manager-7cc98897c-tq7f4                                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  transformation-v0-domain-67b8b6cc6c-f9jdk                          0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  transformation-v0-domain-dispatcher-5f8bb57ff-kjrft                10m (0%)      250m (3%)   64Mi (0%)        256Mi (0%)     23h
  ameide-dev                  transformation-v0-process-77dbc64d67-lvg8v                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  transformation-v0-process-ingress-7d777fd5f-qrtjt                  10m (0%)      250m (3%)   64Mi (0%)        256Mi (0%)     23h
  ameide-dev                  transformation-v0-projection-7fcbcb7884-t2sx5                      0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  transformation-v0-projection-relay-964498dfd-x8glw                 10m (0%)      250m (3%)   64Mi (0%)        256Mi (0%)     23h
  ameide-dev                  vault-core-dev-agent-injector-66b757d695-v6gpf                     0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  workflows-6cc977c877-669hh                                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  workflows-runtime-77d65ff54-57zlv                                  0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  workrequests-workbench-54dc7cfcf6-4mssn                            50m (0%)      500m (6%)   128Mi (0%)       1Gi (3%)       23h
  ameide-dev                  www-ameide-5cc79f79cc-8fk8j                                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-dev                  www-ameide-platform-998d5c694-b7lr6                                0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-prod                 agents-6674c6ccbf-s9nvh                                            0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-prod                 agents-runtime-7886d55d47-xbgxd                                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-prod                 alertmanager-production-prometheus-alertmanager-0                  0 (0%)        0 (0%)      200Mi (0%)       0 (0%)         19h
  ameide-prod                 alloy-logs-b8559c749-rjc2x                                         60m (0%)      500m (6%)   178Mi (0%)       256Mi (0%)     23h
  ameide-prod                 ameide-prod-platform-reloader-reloader-549d46d986-vtpgw            10m (0%)      100m (1%)   32Mi (0%)        128Mi (0%)     23h
  ameide-prod                 data-minio-console-68f88f7db8-xbsg9                                100m (1%)     150m (1%)   128Mi (0%)       192Mi (0%)     23h
  ameide-prod                 data-pgadmin-servers-pgadmin-servers-794dc8b9fc-792t7              0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-prod                 inference-7cbfd4bb89-gw954                                         100m (1%)     200m (2%)   768Mi (2%)       1536Mi (5%)    23h
  ameide-prod                 inference-gateway-8446b6569f-mgc7v                                 100m (1%)     500m (6%)   128Mi (0%)       256Mi (0%)     23h
  ameide-prod                 kafka-entity-operator-5b4bb5cbd4-w5766                             200m (2%)     500m (6%)   512Mi (1%)       1Gi (3%)       23h
  ameide-prod                 kafka-kafka-pool-0                                                 100m (1%)     250m (3%)   384Mi (1%)       768Mi (2%)     23h
  ameide-prod                 kubernetes-dashboard-api-58bf846cd8-4jtt7                          100m (1%)     250m (3%)   200Mi (0%)       400Mi (1%)     23h
  ameide-prod                 kubernetes-dashboard-auth-9b655c45b-4rl77                          100m (1%)     250m (3%)   200Mi (0%)       400Mi (1%)     23h
  ameide-prod                 kubernetes-dashboard-kong-85f664bd9c-fjbws                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-prod                 otel-collector-7f9d4965bb-7l4r4                                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-prod                 platform-586888bdb4-ss9gm                                          0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-prod                 platform-backstage-666946744-r82nl                                 200m (2%)     1 (12%)     512Mi (1%)       1Gi (3%)       37m
  ameide-prod                 platform-extensions-runtime-587694db5b-lgxkh                       250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       37m
  ameide-prod                 platform-prometheus-kube-state-metrics-6d5748f85-lmlk5             0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-prod                 rfr-redis-1                                                        25m (0%)      50m (0%)    50Mi (0%)        100Mi (0%)     33m
  ameide-prod                 rfs-redis-99697f785-dplns                                          25m (0%)      50m (0%)    50Mi (0%)        100Mi (0%)     23h
  ameide-prod                 rfs-redis-99697f785-fkdt8                                          25m (0%)      50m (0%)    50Mi (0%)        100Mi (0%)     23h
  ameide-prod                 rfs-redis-99697f785-gbfk5                                          25m (0%)      50m (0%)    50Mi (0%)        100Mi (0%)     23h
  ameide-prod                 temporal-admintools-67c74c84dd-j8pzz                               0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-prod                 temporal-ui-79499b4849-8k2t2                                       0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-prod                 traffic-manager-9c677698d-s7qnc                                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-prod                 transformation-v0-domain-67b8b6cc6c-lvxc2                          0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-prod                 transformation-v0-projection-7fcbcb7884-9t88w                      0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-prod                 vault-core-prod-agent-injector-75d55f78bc-w5hpb                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-prod                 workflows-76656c647f-nrbtp                                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-prod                 workflows-runtime-7fd56db7d4-vg62w                                 0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-prod                 www-ameide-f856559f4-s25cw                                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-staging              agents-74dd8b67f9-6t4sr                                            0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-staging              agents-runtime-7f7698dff7-w6ldc                                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-staging              ameide-staging-platform-reloader-reloader-55856dc746-czkgw         10m (0%)      100m (1%)   32Mi (0%)        128Mi (0%)     23h
  ameide-staging              data-pgadmin-servers-pgadmin-servers-756d88d97c-2tm9d              0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  ameide-staging              vault-core-staging-0                                               0 (0%)        0 (0%)      0 (0%)           0 (0%)         20h
  argocd                      envoy-argocd-ameide-prod-grpc-internal-cee3432f-64dcd57fb-87v5j    110m (1%)     0 (0%)      544Mi (1%)       0 (0%)         37m
  buildkit                    binfmt-tt84v                                                       0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  eclipse-che                 che-gateway-57dbfb949d-btbb2                                       350m (4%)     0 (0%)      320Mi (1%)       5376Mi (18%)   58m
  kube-system                 azure-cns-b48b7                                                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
  kube-system                 azure-ip-masq-agent-tp4v7                                          50m (0%)      500m (6%)   36Mi (0%)        250Mi (0%)     23h
  kube-system                 azure-npm-8kj64                                                    50m (0%)      250m (3%)   300Mi (1%)       1000Mi (3%)    23h
  kube-system                 cloud-node-manager-9d4fs                                           50m (0%)      0 (0%)      50Mi (0%)        512Mi (1%)     23h
  kube-system                 coredns-6865d647c6-s6cng                                           100m (1%)     3 (38%)     70Mi (0%)        500Mi (1%)     23h
  kube-system                 coredns-6865d647c6-zkq69                                           100m (1%)     3 (38%)     70Mi (0%)        500Mi (1%)     23h
  kube-system                 coredns-autoscaler-fbbdff56-knf25                                  20m (0%)      200m (2%)   10Mi (0%)        1G (3%)        23h
  kube-system                 csi-azuredisk-node-s9xn8                                           30m (0%)      0 (0%)      60Mi (0%)        5800Mi (19%)   23h
  kube-system                 csi-azurefile-node-zk9w2                                           30m (0%)      0 (0%)      60Mi (0%)        600Mi (2%)     23h
  kube-system                 konnectivity-agent-autoscaler-6ff7779788-7fcz4                     20m (0%)      350m (4%)   10Mi (0%)        512M (1%)      23h
  kube-system                 kube-proxy-gkhrd                                                   100m (1%)     0 (0%)      0 (0%)           0 (0%)         23h
  prometheus-system           prometheus-operator-prometheus-node-exporter-xtc8p                 0 (0%)        0 (0%)      0 (0%)           0 (0%)         23h
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests       Limits
  --------           --------       ------
  cpu                7815m (99%)    33250m (425%)
  memory             17236Mi (57%)  51011078656 (163%)
  ephemeral-storage  200Mi (0%)     8Gi (7%)
  hugepages-1Gi      0 (0%)         0 (0%)
  hugepages-2Mi      0 (0%)         0 (0%)
Events:              <none>
```

### aks-system-10377240-vmss00000b
```
Name:               aks-system-10377240-vmss00000b
Roles:              <none>
Labels:             agentpool=system
                    ameide.io/pool=general
                    beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=Standard_B8ms
                    beta.kubernetes.io/os=linux
                    failure-domain.beta.kubernetes.io/region=westeurope
                    failure-domain.beta.kubernetes.io/zone=0
                    kubernetes.azure.com/agentpool=system
                    kubernetes.azure.com/cluster=MC_Ameide_ameide_westeurope
                    kubernetes.azure.com/consolidated-additional-properties=d533fd41-f12d-11f0-a296-fa37edc4354b
                    kubernetes.azure.com/kubelet-identity-client-id=600965a1-36d8-4d7e-97f8-0400e56700d9
                    kubernetes.azure.com/kubelet-serving-ca=cluster
                    kubernetes.azure.com/localdns-state=disabled
                    kubernetes.azure.com/mode=system
                    kubernetes.azure.com/node-image-version=AKSUbuntu-2204gen2containerd-202512.18.0
                    kubernetes.azure.com/nodepool-type=VirtualMachineScaleSets
                    kubernetes.azure.com/os-sku=Ubuntu
                    kubernetes.azure.com/os-sku-effective=Ubuntu2204
                    kubernetes.azure.com/os-sku-requested=Ubuntu
                    kubernetes.azure.com/role=agent
                    kubernetes.azure.com/sku-cpu=8
                    kubernetes.azure.com/sku-memory=32768
                    kubernetes.azure.com/storageprofile=managed
                    kubernetes.azure.com/storagetier=Premium_LRS
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=aks-system-10377240-vmss00000b
                    kubernetes.io/os=linux
                    node.kubernetes.io/instance-type=Standard_B8ms
                    storageprofile=managed
                    storagetier=Premium_LRS
                    topology.disk.csi.azure.com/zone=
                    topology.kubernetes.io/region=westeurope
                    topology.kubernetes.io/zone=0
Annotations:        alpha.kubernetes.io/provided-node-ip: 10.224.0.69
                    csi.volume.kubernetes.io/nodeid:
                      {"disk.csi.azure.com":"aks-system-10377240-vmss00000b","file.csi.azure.com":"aks-system-10377240-vmss00000b"}
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Wed, 14 Jan 2026 19:10:41 +0000
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  aks-system-10377240-vmss00000b
  AcquireTime:     <unset>
  RenewTime:       Thu, 15 Jan 2026 13:30:49 +0000
Conditions:
  Type                          Status  LastHeartbeatTime                 LastTransitionTime                Reason                          Message
  ----                          ------  -----------------                 ------------------                ------                          -------
  KubeletProblem                False   Thu, 15 Jan 2026 13:28:20 +0000   Wed, 14 Jan 2026 19:11:12 +0000   KubeletIsUp                     kubelet service is up
  FilesystemCorruptionProblem   False   Thu, 15 Jan 2026 13:28:20 +0000   Wed, 14 Jan 2026 19:11:12 +0000   FilesystemIsOK                  Filesystem is healthy
  FrequentUnregisterNetDevice   False   Thu, 15 Jan 2026 13:28:20 +0000   Wed, 14 Jan 2026 19:11:12 +0000   NoFrequentUnregisterNetDevice   node is functioning properly
  FrequentKubeletRestart        False   Thu, 15 Jan 2026 13:28:20 +0000   Wed, 14 Jan 2026 19:11:12 +0000   NoFrequentKubeletRestart        kubelet is functioning properly
  VMEventScheduled              False   Thu, 15 Jan 2026 13:28:20 +0000   Wed, 14 Jan 2026 19:11:12 +0000   NoVMEventScheduled              VM has no scheduled event
  ReadonlyFilesystem            False   Thu, 15 Jan 2026 13:28:20 +0000   Wed, 14 Jan 2026 19:11:12 +0000   FilesystemIsNotReadOnly         Filesystem is not read-only
  FrequentContainerdRestart     False   Thu, 15 Jan 2026 13:28:20 +0000   Wed, 14 Jan 2026 19:11:12 +0000   NoFrequentContainerdRestart     containerd is functioning properly
  KernelDeadlock                False   Thu, 15 Jan 2026 13:28:20 +0000   Wed, 14 Jan 2026 19:11:12 +0000   KernelHasNoDeadlock             kernel has no deadlock
  FrequentDockerRestart         False   Thu, 15 Jan 2026 13:28:20 +0000   Wed, 14 Jan 2026 19:11:12 +0000   NoFrequentDockerRestart         docker is functioning properly
  ContainerRuntimeProblem       False   Thu, 15 Jan 2026 13:28:20 +0000   Wed, 14 Jan 2026 19:11:12 +0000   ContainerRuntimeIsUp            container runtime service is up
  MemoryPressure                False   Thu, 15 Jan 2026 13:27:43 +0000   Wed, 14 Jan 2026 19:10:41 +0000   KubeletHasSufficientMemory      kubelet has sufficient memory available
  DiskPressure                  False   Thu, 15 Jan 2026 13:27:43 +0000   Wed, 14 Jan 2026 19:10:41 +0000   KubeletHasNoDiskPressure        kubelet has no disk pressure
  PIDPressure                   False   Thu, 15 Jan 2026 13:27:43 +0000   Wed, 14 Jan 2026 19:10:41 +0000   KubeletHasSufficientPID         kubelet has sufficient PID available
  Ready                         True    Thu, 15 Jan 2026 13:27:43 +0000   Wed, 14 Jan 2026 19:10:56 +0000   KubeletReady                    kubelet is posting ready status
Addresses:
  InternalIP:  10.224.0.69
  Hostname:    aks-system-10377240-vmss00000b
Capacity:
  cpu:                8
  ephemeral-storage:  129886128Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             32864156Ki
  pods:               110
Allocatable:
  cpu:                7820m
  ephemeral-storage:  119703055367
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             30457756Ki
  pods:               110
System Info:
  Machine ID:                 15447b12cf65499f97982e010eaaad44
  System UUID:                e0e716eb-66a4-46d8-a7d9-056231eda517
  Boot ID:                    cb8b3912-ceac-4417-ab68-04ae0b74b0f8
  Kernel Version:             5.15.0-1102-azure
  OS Image:                   Ubuntu 22.04.5 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.7.29-1
  Kubelet Version:            v1.32.7
  Kube-Proxy Version:         v1.32.7
ProviderID:                   azure:///subscriptions/68ba6806-178c-41f1-84ec-d7a839799be1/resourceGroups/mc_ameide_ameide_westeurope/providers/Microsoft.Compute/virtualMachineScaleSets/aks-system-10377240-vmss/virtualMachines/11
Non-terminated Pods:          (71 in total)
  Namespace                   Name                                                             CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                                             ------------  ----------  ---------------  -------------  ---
  ameide-dev                  alertmanager-dev-prometheus-alertmanager-0                       0 (0%)        0 (0%)      200Mi (0%)       0 (0%)         17h
  ameide-dev                  chi-data-clickhouse-data-clickhouse-0-0-0                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         17h
  ameide-dev                  keycloak-0                                                       0 (0%)        0 (0%)      1700Mi (5%)      2Gi (6%)       84m
  ameide-dev                  platform-keycloak-realm-master-bootstrap-ccc6p                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         37m
  ameide-dev                  platform-tempo-0                                                 0 (0%)        0 (0%)      0 (0%)           0 (0%)         17h
  ameide-dev                  postgres-ameide-1                                                500m (6%)     2 (25%)     1Gi (3%)         4Gi (13%)      17h
  ameide-dev                  rfr-redis-1                                                      25m (0%)      50m (0%)    50Mi (0%)        100Mi (0%)     17h
  ameide-prod                 chi-data-clickhouse-data-clickhouse-0-0-0                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         33m
  ameide-prod                 kubernetes-dashboard-oauth2-proxy-5f966d5694-c95qc               100m (1%)     150m (1%)   128Mi (0%)       192Mi (0%)     37m
  ameide-prod                 postgres-ameide-1                                                500m (6%)     2 (25%)     1Gi (3%)         4Gi (13%)      17h
  ameide-prod                 rfr-redis-0                                                      25m (0%)      50m (0%)    50Mi (0%)        100Mi (0%)     24m
  ameide-prod                 rfr-redis-2                                                      25m (0%)      50m (0%)    50Mi (0%)        100Mi (0%)     17h
  ameide-prod                 temporal-frontend-699f67d6c4-2lr7s                               250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       17h
  ameide-prod                 temporal-history-777987c4dd-7qlcc                                250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       17h
  ameide-prod                 temporal-matching-7ccd56bd5d-xd4gj                               250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       17h
  ameide-prod                 temporal-worker-7987489cbb-dndbs                                 250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       17h
  ameide-prod                 vault-core-prod-0                                                0 (0%)        0 (0%)      0 (0%)           0 (0%)         17h
  ameide-staging              alertmanager-staging-prometheus-alertmanager-0                   0 (0%)        0 (0%)      200Mi (0%)       0 (0%)         17h
  ameide-staging              chi-data-clickhouse-data-clickhouse-0-0-0                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         17h
  ameide-staging              inference-gateway-7b6bbb7948-s5p7w                               50m (0%)      150m (1%)   192Mi (0%)       384Mi (1%)     17h
  ameide-staging              keycloak-0                                                       500m (6%)     1 (12%)     768Mi (2%)       1536Mi (5%)    86m
  ameide-staging              kubernetes-dashboard-kong-85f664bd9c-bflf9                       0 (0%)        0 (0%)      0 (0%)           0 (0%)         17h
  ameide-staging              kubernetes-dashboard-metrics-scraper-staging-7d4df48586-6cz7l    100m (1%)     250m (3%)   200Mi (0%)       400Mi (1%)     17h
  ameide-staging              kubernetes-dashboard-oauth2-proxy-5f49b96c5d-dgq7m               100m (1%)     150m (1%)   128Mi (0%)       192Mi (0%)     17h
  ameide-staging              kubernetes-dashboard-web-db9d54fb7-h58q7                         100m (1%)     250m (3%)   200Mi (0%)       400Mi (1%)     17h
  ameide-staging              platform-556b8d6776-b6wnr                                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         17h
  ameide-staging              platform-backstage-5c7bd9c95-2qvx4                               100m (1%)     1 (12%)     512Mi (1%)       1Gi (3%)       37m
  ameide-staging              platform-camunda8-connectors-c9df8f8d-r8vpv                      250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       17h
  ameide-staging              platform-extensions-runtime-cffbf75c8-w6pmf                      250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       17h
  ameide-staging              platform-prometheus-kube-state-metrics-745584d685-q6lmq          0 (0%)        0 (0%)      0 (0%)           0 (0%)         17h
  ameide-staging              plausible-oauth2-proxy-844457945d-4s6xc                          100m (1%)     150m (1%)   128Mi (0%)       192Mi (0%)     17h
  ameide-staging              postgres-ameide-1                                                500m (6%)     2 (25%)     1Gi (3%)         4Gi (13%)      17h
  ameide-staging              prometheus-staging-prometheus-prometheus-0                       0 (0%)        0 (0%)      0 (0%)           0 (0%)         17h
  ameide-staging              rfs-redis-654d8b497c-xnbqd                                       25m (0%)      50m (0%)    50Mi (0%)        100Mi (0%)     17h
  ameide-staging              temporal-frontend-6d776cb8d-jspfr                                250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       17h
  ameide-staging              temporal-history-545987dcfd-ltglg                                250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       17h
  ameide-staging              temporal-matching-58c96bd6cb-tnd9w                               250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       17h
  ameide-staging              temporal-worker-86c548b76d-d4q2k                                 250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       17h
  ameide-staging              threads-949fccb84-7zrm6                                          100m (1%)     200m (2%)   256Mi (0%)       512Mi (1%)     17h
  ameide-staging              workflows-runtime-67d89cb448-sb98b                               0 (0%)        0 (0%)      0 (0%)           0 (0%)         17h
  ameide-staging              www-ameide-platform-856f88dd88-5dwz9                             0 (0%)        0 (0%)      0 (0%)           0 (0%)         17h
  ameide-system               ameide-operators-domain-75b56c647c-pmlmn                         50m (0%)      500m (6%)   128Mi (0%)       256Mi (0%)     17h
  ameide-system               ameide-operators-process-54c6b6df8-ddd4f                         50m (0%)      500m (6%)   128Mi (0%)       256Mi (0%)     17h
  ameide-system               ameide-operators-uisurface-6d4dbfb9fc-b9r2l                      50m (0%)      500m (6%)   128Mi (0%)       256Mi (0%)     17h
  ameide-system               temporal-operator-controller-manager-6c9c7d9dd4-p7k7l            10m (0%)      500m (6%)   64Mi (0%)        128Mi (0%)     17h
  arc-systems                 github-arc-controller-gha-rs-controller-6db79bbccf-z4smb         0 (0%)        0 (0%)      0 (0%)           0 (0%)         17h
  argocd                      argocd-application-controller-0                                  250m (3%)     1 (12%)     2Gi (6%)         4Gi (13%)      162m
  argocd                      argocd-applicationset-controller-6c55d779dd-pl6n4                0 (0%)        0 (0%)      0 (0%)           0 (0%)         17h
  argocd                      argocd-repo-server-8455f5b4d5-x4vpd                              250m (3%)     2 (25%)     512Mi (1%)       1Gi (3%)       3h58m
  argocd                      argocd-server-6b46bdf94c-7hljz                                   250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       18h
  argocd                      envoy-ameide-dev-ameide-7c898b73-d67994c48-k4b87                 110m (1%)     0 (0%)      544Mi (1%)       0 (0%)         17h
  argocd                      envoy-ameide-prod-ameide-270d1f22-5b65497db5-l47nh               110m (1%)     0 (0%)      544Mi (1%)       0 (0%)         17h
  argocd                      envoy-ameide-staging-ameide-f462b9ba-f8c755fd7-6hqtx             110m (1%)     0 (0%)      544Mi (1%)       0 (0%)         17h
  argocd                      envoy-argocd-cluster-18b70826-588978797f-6cdzt                   110m (1%)     0 (0%)      544Mi (1%)       0 (0%)         15h
  argocd                      envoy-gateway-77dd576b87-p5bcq                                   100m (1%)     0 (0%)      256Mi (0%)       512Mi (1%)     17h
  buildkit                    binfmt-hv8kr                                                     0 (0%)        0 (0%)      0 (0%)           0 (0%)         18h
  cert-manager                cert-manager-cainjector-5466759554-9vzfw                         10m (0%)      200m (2%)   64Mi (0%)        256Mi (0%)     17h
  cnpg-system                 cloudnative-pg-79cc8df686-pnx5q                                  0 (0%)        0 (0%)      0 (0%)           0 (0%)         17h
  devworkspace-controller     devworkspace-webhook-server-69fd9c49f6-s9gjv                     100m (1%)     200m (2%)   20Mi (0%)        300Mi (1%)     37m
  eclipse-che                 che-64fbd888c7-jtb9b                                             100m (1%)     0 (0%)      512Mi (1%)       1Gi (3%)       148m
  external-secrets            external-secrets-cert-controller-cdd97d9f9-22rk8                 0 (0%)        0 (0%)      0 (0%)           0 (0%)         17h
  keda-system                 keda-operator-84947d8c56-hfsrp                                   100m (1%)     1 (12%)     100Mi (0%)       1000Mi (3%)    3h23m
  kube-system                 azure-cns-lhg6n                                                  0 (0%)        0 (0%)      0 (0%)           0 (0%)         18h
  kube-system                 azure-ip-masq-agent-gnhhp                                        50m (0%)      500m (6%)   36Mi (0%)        250Mi (0%)     18h
  kube-system                 azure-npm-4s6l2                                                  50m (0%)      250m (3%)   300Mi (1%)       1000Mi (3%)    18h
  kube-system                 azure-wi-webhook-controller-manager-77c4b69f49-sbstk             100m (1%)     200m (2%)   20Mi (0%)        300Mi (1%)     37m
  kube-system                 cloud-node-manager-2wrg2                                         50m (0%)      0 (0%)      50Mi (0%)        512Mi (1%)     18h
  kube-system                 csi-azuredisk-node-x9cpq                                         30m (0%)      0 (0%)      60Mi (0%)        5800Mi (19%)   18h
  kube-system                 csi-azurefile-node-9246c                                         30m (0%)      0 (0%)      60Mi (0%)        600Mi (2%)     18h
  kube-system                 kube-proxy-fgrkx                                                 100m (1%)     0 (0%)      0 (0%)           0 (0%)         18h
  prometheus-system           prometheus-operator-prometheus-node-exporter-kq88p               0 (0%)        0 (0%)      0 (0%)           0 (0%)         18h
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests       Limits
  --------           --------       ------
  cpu                7520m (96%)    27850m (356%)
  memory             20178Mi (67%)  48402Mi (162%)
  ephemeral-storage  150Mi (0%)     6Gi (5%)
  hugepages-1Gi      0 (0%)         0 (0%)
  hugepages-2Mi      0 (0%)         0 (0%)
Events:              <none>
```

### aks-system-10377240-vmss00000c
```
Name:               aks-system-10377240-vmss00000c
Roles:              <none>
Labels:             agentpool=system
                    ameide.io/pool=general
                    beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=Standard_B8ms
                    beta.kubernetes.io/os=linux
                    failure-domain.beta.kubernetes.io/region=westeurope
                    failure-domain.beta.kubernetes.io/zone=0
                    kubernetes.azure.com/agentpool=system
                    kubernetes.azure.com/cluster=MC_Ameide_ameide_westeurope
                    kubernetes.azure.com/consolidated-additional-properties=d533fd41-f12d-11f0-a296-fa37edc4354b
                    kubernetes.azure.com/kubelet-identity-client-id=600965a1-36d8-4d7e-97f8-0400e56700d9
                    kubernetes.azure.com/kubelet-serving-ca=cluster
                    kubernetes.azure.com/localdns-state=disabled
                    kubernetes.azure.com/mode=system
                    kubernetes.azure.com/node-image-version=AKSUbuntu-2204gen2containerd-202512.18.0
                    kubernetes.azure.com/nodepool-type=VirtualMachineScaleSets
                    kubernetes.azure.com/os-sku=Ubuntu
                    kubernetes.azure.com/os-sku-effective=Ubuntu2204
                    kubernetes.azure.com/os-sku-requested=Ubuntu
                    kubernetes.azure.com/role=agent
                    kubernetes.azure.com/sku-cpu=8
                    kubernetes.azure.com/sku-memory=32768
                    kubernetes.azure.com/storageprofile=managed
                    kubernetes.azure.com/storagetier=Premium_LRS
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=aks-system-10377240-vmss00000c
                    kubernetes.io/os=linux
                    node.kubernetes.io/instance-type=Standard_B8ms
                    storageprofile=managed
                    storagetier=Premium_LRS
                    topology.disk.csi.azure.com/zone=
                    topology.kubernetes.io/region=westeurope
                    topology.kubernetes.io/zone=0
Annotations:        alpha.kubernetes.io/provided-node-ip: 10.224.1.34
                    csi.volume.kubernetes.io/nodeid:
                      {"disk.csi.azure.com":"aks-system-10377240-vmss00000c","file.csi.azure.com":"aks-system-10377240-vmss00000c"}
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Wed, 14 Jan 2026 20:46:45 +0000
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  aks-system-10377240-vmss00000c
  AcquireTime:     <unset>
  RenewTime:       Thu, 15 Jan 2026 13:30:43 +0000
Conditions:
  Type                          Status  LastHeartbeatTime                 LastTransitionTime                Reason                          Message
  ----                          ------  -----------------                 ------------------                ------                          -------
  KernelDeadlock                False   Thu, 15 Jan 2026 13:30:00 +0000   Wed, 14 Jan 2026 20:47:18 +0000   KernelHasNoDeadlock             kernel has no deadlock
  VMEventScheduled              False   Thu, 15 Jan 2026 13:30:00 +0000   Wed, 14 Jan 2026 20:48:02 +0000   NoVMEventScheduled              VM has no scheduled event
  ReadonlyFilesystem            False   Thu, 15 Jan 2026 13:30:00 +0000   Wed, 14 Jan 2026 20:47:18 +0000   FilesystemIsNotReadOnly         Filesystem is not read-only
  FrequentDockerRestart         False   Thu, 15 Jan 2026 13:30:00 +0000   Wed, 14 Jan 2026 20:47:18 +0000   NoFrequentDockerRestart         docker is functioning properly
  FrequentUnregisterNetDevice   False   Thu, 15 Jan 2026 13:30:00 +0000   Wed, 14 Jan 2026 20:47:18 +0000   NoFrequentUnregisterNetDevice   node is functioning properly
  FrequentKubeletRestart        False   Thu, 15 Jan 2026 13:30:00 +0000   Wed, 14 Jan 2026 20:47:18 +0000   NoFrequentKubeletRestart        kubelet is functioning properly
  FrequentContainerdRestart     False   Thu, 15 Jan 2026 13:30:00 +0000   Wed, 14 Jan 2026 20:47:18 +0000   NoFrequentContainerdRestart     containerd is functioning properly
  ContainerRuntimeProblem       False   Thu, 15 Jan 2026 13:30:00 +0000   Wed, 14 Jan 2026 20:47:18 +0000   ContainerRuntimeIsUp            container runtime service is up
  FilesystemCorruptionProblem   False   Thu, 15 Jan 2026 13:30:00 +0000   Wed, 14 Jan 2026 20:47:18 +0000   FilesystemIsOK                  Filesystem is healthy
  KubeletProblem                False   Thu, 15 Jan 2026 13:30:00 +0000   Wed, 14 Jan 2026 20:47:18 +0000   KubeletIsUp                     kubelet service is up
  MemoryPressure                False   Thu, 15 Jan 2026 13:29:34 +0000   Wed, 14 Jan 2026 20:46:45 +0000   KubeletHasSufficientMemory      kubelet has sufficient memory available
  DiskPressure                  False   Thu, 15 Jan 2026 13:29:34 +0000   Wed, 14 Jan 2026 20:46:45 +0000   KubeletHasNoDiskPressure        kubelet has no disk pressure
  PIDPressure                   False   Thu, 15 Jan 2026 13:29:34 +0000   Wed, 14 Jan 2026 20:46:45 +0000   KubeletHasSufficientPID         kubelet has sufficient PID available
  Ready                         True    Thu, 15 Jan 2026 13:29:34 +0000   Wed, 14 Jan 2026 20:46:58 +0000   KubeletReady                    kubelet is posting ready status
Addresses:
  InternalIP:  10.224.1.34
  Hostname:    aks-system-10377240-vmss00000c
Capacity:
  cpu:                8
  ephemeral-storage:  129886128Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             32864152Ki
  pods:               110
Allocatable:
  cpu:                7820m
  ephemeral-storage:  119703055367
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             30457752Ki
  pods:               110
System Info:
  Machine ID:                                     88017b53dd284cd98efee204763385f5
  System UUID:                                    d4fa2319-a215-4894-9ae6-d91efa6621e2
  Boot ID:                                        b137cf86-2b84-4eb7-b17e-09a654756c3e
  Kernel Version:                                 5.15.0-1102-azure
  OS Image:                                       Ubuntu 22.04.5 LTS
  Operating System:                               linux
  Architecture:                                   amd64
  Container Runtime Version:                      containerd://1.7.29-1
  Kubelet Version:                                v1.32.7
  Kube-Proxy Version:                             v1.32.7
ProviderID:                                       azure:///subscriptions/68ba6806-178c-41f1-84ec-d7a839799be1/resourceGroups/mc_ameide_ameide_westeurope/providers/Microsoft.Compute/virtualMachineScaleSets/aks-system-10377240-vmss/virtualMachines/12
Non-terminated Pods:                              (39 in total)
  Namespace                                       Name                                                               CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                                       ----                                                               ------------  ----------  ---------------  -------------  ---
  ameide-dev                                      coder-7fc6b5cc99-slcgk                                             200m (2%)     2 (25%)     512Mi (1%)       4Gi (13%)      8h
  ameide-dev                                      codex-monitor-0-55d55fd454-6hcxk                                   25m (0%)      500m (6%)   128Mi (0%)       512Mi (1%)     15h
  ameide-dev                                      codex-monitor-1-686cddbfd9-tjfwd                                   25m (0%)      500m (6%)   128Mi (0%)       512Mi (1%)     15h
  ameide-dev                                      platform-grafana-5d86f794db-zxqcd                                  0 (0%)        0 (0%)      0 (0%)           0 (0%)         37m
  ameide-dev                                      vault-core-dev-0                                                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         33m
  ameide-prod                                     platform-tempo-0                                                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         34m
  ameide-prod                                     postgres-ameide-2                                                  500m (6%)     2 (25%)     1Gi (3%)         4Gi (13%)      16h
  ameide-prod                                     prometheus-production-prometheus-prometheus-0                      0 (0%)        0 (0%)      0 (0%)           0 (0%)         34m
  ameide-staging                                  otel-collector-7cdb7666f-fn8lg                                     0 (0%)        0 (0%)      0 (0%)           0 (0%)         37m
  ameide-staging                                  platform-grafana-7879d679b7-b4c9j                                  0 (0%)        0 (0%)      0 (0%)           0 (0%)         37m
  ameide-staging                                  platform-tempo-0                                                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         33m
  ameide-staging                                  postgres-ameide-3                                                  500m (6%)     2 (25%)     1Gi (3%)         4Gi (13%)      16h
  ameide-staging                                  rfr-redis-2                                                        25m (0%)      50m (0%)    50Mi (0%)        100Mi (0%)     34m
  ameide-staging                                  rfs-redis-654d8b497c-k762l                                         25m (0%)      50m (0%)    50Mi (0%)        100Mi (0%)     37m
  ameide-ws-52841969-f3ca-4f53-a747-1864fd7e0699  coder-52841969-f3ca-4f53-a747-1864fd7e0699-8669fbdbb4-vfdfq        250m (3%)     2 (25%)     512Mi (1%)       15Gi (51%)     7h40m
  ameide-ws-91648837-bed9-4fa3-b5f9-3e5f47099782  coder-91648837-bed9-4fa3-b5f9-3e5f47099782-6c8d778677-qx29n        250m (3%)     8 (102%)    512Mi (1%)       16Gi (55%)     4h14m
  ameide-ws-aed4e053-e187-455d-a9d2-e3d530a9bf4d  coder-aed4e053-e187-455d-a9d2-e3d530a9bf4d-58c498b549-p2qjx        250m (3%)     2 (25%)     512Mi (1%)       16Gi (55%)     7h41m
  argocd                                          argocd-server-6b46bdf94c-hvn5n                                     250m (3%)     1 (12%)     512Mi (1%)       1Gi (3%)       4h8m
  argocd                                          envoy-argocd-ameide-dev-grpc-internal-fe61a74f-567d54d49-9pgp8     110m (1%)     0 (0%)      544Mi (1%)       0 (0%)         16h
  argocd                                          envoy-argocd-ameide-dev-grpc-internal-fe61a74f-567d54d49-g52cl     110m (1%)     0 (0%)      544Mi (1%)       0 (0%)         16h
  argocd                                          envoy-argocd-ameide-prod-grpc-internal-cee3432f-64dcd57fb-5c72m    110m (1%)     0 (0%)      544Mi (1%)       0 (0%)         16h
  argocd                                          envoy-argocd-ameide-staging-grpc-internal-851fe813-74d47d7xcr52    110m (1%)     0 (0%)      544Mi (1%)       0 (0%)         16h
  buildkit                                        binfmt-ktz2s                                                       0 (0%)        0 (0%)      0 (0%)           0 (0%)         16h
  buildkit                                        buildkitd-0                                                        4 (51%)       8 (102%)    8Gi (27%)        16Gi (55%)     15h
  devworkspace-controller                         devworkspace-controller-manager-cf9db9b68-9ds8h                    250m (3%)     3 (38%)     100Mi (0%)       5Gi (17%)      153m
  devworkspace-controller                         devworkspace-webhook-server-69fd9c49f6-g4v67                       100m (1%)     200m (2%)   20Mi (0%)        300Mi (1%)     152m
  eclipse-che                                     che-dashboard-7b475b6c4f-r8hs2                                     100m (1%)     0 (0%)      32Mi (0%)        1Gi (3%)       148m
  eclipse-che                                     che-operator-789f9cc494-5d89g                                      100m (1%)     500m (6%)   128Mi (0%)       2Gi (6%)       3h45m
  keda-system                                     keda-admission-webhooks-6955c54468-zt29b                           100m (1%)     1 (12%)     100Mi (0%)       1000Mi (3%)    3h23m
  kube-system                                     azure-cns-2f642                                                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         16h
  kube-system                                     azure-ip-masq-agent-xs952                                          50m (0%)      500m (6%)   36Mi (0%)        250Mi (0%)     16h
  kube-system                                     azure-npm-4f22p                                                    50m (0%)      250m (3%)   300Mi (1%)       1000Mi (3%)    16h
  kube-system                                     azure-wi-webhook-controller-manager-77c4b69f49-rgs47               100m (1%)     200m (2%)   20Mi (0%)        300Mi (1%)     10h
  kube-system                                     cloud-node-manager-6mgvq                                           50m (0%)      0 (0%)      50Mi (0%)        512Mi (1%)     16h
  kube-system                                     csi-azuredisk-node-ntv8f                                           30m (0%)      0 (0%)      60Mi (0%)        5800Mi (19%)   16h
  kube-system                                     csi-azurefile-node-4dn85                                           30m (0%)      0 (0%)      60Mi (0%)        600Mi (2%)     16h
  kube-system                                     konnectivity-agent-56489d589b-z5vwm                                20m (0%)      1 (12%)     20Mi (0%)        1Gi (3%)       10h
  kube-system                                     kube-proxy-ncjf9                                                   100m (1%)     0 (0%)      0 (0%)           0 (0%)         16h
  prometheus-system                               prometheus-operator-prometheus-node-exporter-prxjd                 0 (0%)        0 (0%)      0 (0%)           0 (0%)         16h
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests       Limits
  --------           --------       ------
  cpu                7820m (100%)   34750m (444%)
  memory             16258Mi (54%)  98026Mi (329%)
  ephemeral-storage  0 (0%)         0 (0%)
  hugepages-1Gi      0 (0%)         0 (0%)
  hugepages-2Mi      0 (0%)         0 (0%)
Events:              <none>
```

## Prometheus node-exporter status (prometheus-system)
```
prometheus-operator-prometheus-node-exporter-k2mfg     1/1     Running   0              23h   10.224.0.10    aks-system-10377240-vmss000005   <none>           <none>
prometheus-operator-prometheus-node-exporter-kq88p     1/1     Running   5              18h   10.224.0.69    aks-system-10377240-vmss00000b   <none>           <none>
prometheus-operator-prometheus-node-exporter-p46bn     1/1     Running   55 (45m ago)   23h   10.224.0.199   aks-system-10377240-vmss000007   <none>           <none>
prometheus-operator-prometheus-node-exporter-prxjd     1/1     Running   0              16h   10.224.1.34    aks-system-10377240-vmss00000c   <none>           <none>
prometheus-operator-prometheus-node-exporter-xtc8p     1/1     Running   0              23h   10.224.1.166   aks-system-10377240-vmss000008   <none>           <none>
```

## Focus: node-exporter on aks-system-10377240-vmss00000b
```
Name:             prometheus-operator-prometheus-node-exporter-kq88p
Namespace:        prometheus-system
Priority:         0
Service Account:  prometheus-operator-prometheus-node-exporter
Node:             aks-system-10377240-vmss00000b/10.224.0.69
Start Time:       Wed, 14 Jan 2026 19:10:41 +0000
Labels:           app.kubernetes.io/component=metrics
                  app.kubernetes.io/instance=prometheus-operator
                  app.kubernetes.io/managed-by=Helm
                  app.kubernetes.io/name=prometheus-node-exporter
                  app.kubernetes.io/part-of=prometheus-node-exporter
                  app.kubernetes.io/version=1.10.2
                  controller-revision-hash=6c5ffc768
                  helm.sh/chart=prometheus-node-exporter-4.49.2
                  jobLabel=node-exporter
                  pod-template-generation=1
                  release=prometheus-operator
Annotations:      cluster-autoscaler.kubernetes.io/safe-to-evict: true
Status:           Running
IP:               10.224.0.69
IPs:
  IP:           10.224.0.69
Controlled By:  DaemonSet/prometheus-operator-prometheus-node-exporter
Containers:
  node-exporter:
    Container ID:  containerd://5d4c0446c1e19667aba24b92a1a96085303d3f113382992febe0bc4ab2e34bb8
    Image:         quay.io/prometheus/node-exporter:v1.10.2
    Image ID:      quay.io/prometheus/node-exporter@sha256:337ff1d356b68d39cef853e8c6345de11ce7556bb34cda8bd205bcf2ed30b565
    Port:          9100/TCP (http-metrics)
    Host Port:     9100/TCP (http-metrics)
    Args:
      --path.procfs=/host/proc
      --path.sysfs=/host/sys
      --path.rootfs=/host/root
      --path.udev.data=/host/root/run/udev/data
      --web.listen-address=[$(HOST_IP)]:9100
      --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|run/containerd/.+|var/lib/docker/.+|var/lib/kubelet/.+)($|/)
      --collector.filesystem.fs-types-exclude=^(autofs|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|iso9660|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tracefs|erofs)$
    State:          Running
      Started:      Thu, 15 Jan 2026 13:17:07 +0000
    Ready:          True
    Restart Count:  5
    Liveness:       http-get http://:http-metrics/ delay=0s timeout=1s period=10s #success=1 #failure=3
    Readiness:      http-get http://:http-metrics/ delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:
      HOST_IP:  0.0.0.0
    Mounts:
      /host/proc from proc (ro)
      /host/root from root (ro)
      /host/sys from sys (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  proc:
    Type:          HostPath (bare host directory volume)
    Path:          /proc
    HostPathType:  
  sys:
    Type:          HostPath (bare host directory volume)
    Path:          /sys
    HostPathType:  
  root:
    Type:          HostPath (bare host directory volume)
    Path:          /
    HostPathType:  
QoS Class:         BestEffort
Node-Selectors:    kubernetes.io/os=linux
Tolerations:       :NoSchedule op=Exists
                   node.kubernetes.io/disk-pressure:NoSchedule op=Exists
                   node.kubernetes.io/memory-pressure:NoSchedule op=Exists
                   node.kubernetes.io/network-unavailable:NoSchedule op=Exists
                   node.kubernetes.io/not-ready:NoExecute op=Exists
                   node.kubernetes.io/pid-pressure:NoSchedule op=Exists
                   node.kubernetes.io/unreachable:NoExecute op=Exists
                   node.kubernetes.io/unschedulable:NoSchedule op=Exists
Events:
  Type     Reason     Age                 From     Message
  ----     ------     ----                ----     -------
  Warning  Unhealthy  16m (x25 over 87m)  kubelet  Readiness probe failed: Get "http://10.224.0.69:9100/": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
  Normal   Killing    14m (x5 over 77m)   kubelet  Container node-exporter failed liveness probe, will be restarted
  Normal   Pulled     13m (x5 over 77m)   kubelet  Container image "quay.io/prometheus/node-exporter:v1.10.2" already present on machine
  Normal   Created    13m (x5 over 76m)   kubelet  Created container: node-exporter
  Normal   Started    13m (x5 over 76m)   kubelet  Started container node-exporter
  Warning  Unhealthy  12m (x38 over 85m)  kubelet  Liveness probe failed: Get "http://10.224.0.69:9100/": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```
## Computed reservation (requests)

Notes: sums are based on non-terminal pods only (exclude Succeeded/Failed and deleting pods).

### By namespace (top 50)

| namespace | cpu requests (cores) | mem requests |
|---|---:|---:|
| buildkit | 8.000 | 16.00Gi |
| ameide-staging | 7.970 | 18.04Gi |
| ameide-dev | 7.700 | 19.47Gi |
| ameide-prod | 7.535 | 17.91Gi |
| argocd | 3.250 | 14.00Gi |
| kube-system | 2.346 | 3.00Gi |
| eclipse-che | 0.650 | 0.97Gi |
| devworkspace-controller | 0.450 | 0.14Gi |
| keda-system | 0.300 | 0.29Gi |
| keycloak-system | 0.300 | 0.44Gi |
| ameide-ws-52841969-f3ca-4f53-a747-1864fd7e0699 | 0.250 | 0.50Gi |
| ameide-ws-91648837-bed9-4fa3-b5f9-3e5f47099782 | 0.250 | 0.50Gi |
| ameide-ws-aed4e053-e187-455d-a9d2-e3d530a9bf4d | 0.250 | 0.50Gi |
| redis-system | 0.250 | 0.25Gi |
| strimzi-system | 0.200 | 0.38Gi |
| ameide-system | 0.160 | 0.44Gi |
| prometheus-system | 0.100 | 0.10Gi |
| cert-manager | 0.030 | 0.19Gi |
| arc-systems | 0.000 | 0.00Gi |
| clickhouse-system | 0.000 | 0.00Gi |
| cnpg-system | 0.000 | 0.00Gi |
| external-secrets | 0.000 | 0.00Gi |

### By node: requested vs allocatable

| node | cpu req (cores) | cpu alloc (cores) | cpu req% | mem req | mem alloc | mem req% | alloc pods |
|---|---:|---:|---:|---:|---:|---:|---:|
| aks-system-10377240-vmss000005 | 7.766 | 7.820 | 99.3% | 21.02Gi | 29.04Gi | 72.4% | 110 |
| aks-system-10377240-vmss000007 | 4.570 | 7.820 | 58.4% | 10.93Gi | 29.05Gi | 37.6% | 110 |
| aks-system-10377240-vmss000008 | 7.815 | 7.820 | 99.9% | 16.83Gi | 29.05Gi | 57.9% | 110 |
| aks-system-10377240-vmss00000b | 7.520 | 7.820 | 96.2% | 19.71Gi | 29.05Gi | 67.8% | 110 |
| aks-system-10377240-vmss00000c | 7.820 | 7.820 | 100.0% | 15.88Gi | 29.05Gi | 54.7% | 110 |

### Top 100 pods by CPU requests

| namespace | pod | node | phase | cpu req (cores) | mem req |
|---|---|---|---|---:|---:|
| buildkit | buildkitd-0 | aks-system-10377240-vmss00000c | Running | 4.000 | 8.00Gi |
| buildkit | buildkitd-1 |  | Pending | 4.000 | 8.00Gi |
| ameide-dev | postgres-ameide-1 | aks-system-10377240-vmss00000b | Running | 0.500 | 1.00Gi |
| ameide-dev | postgres-ameide-2 | aks-system-10377240-vmss000005 | Running | 0.500 | 1.00Gi |
| ameide-prod | keycloak-0 |  | Pending | 0.500 | 0.75Gi |
| ameide-prod | postgres-ameide-1 | aks-system-10377240-vmss00000b | Running | 0.500 | 1.00Gi |
| ameide-prod | postgres-ameide-2 | aks-system-10377240-vmss00000c | Running | 0.500 | 1.00Gi |
| ameide-staging | keycloak-0 | aks-system-10377240-vmss00000b | Running | 0.500 | 0.75Gi |
| ameide-staging | postgres-ameide-1 | aks-system-10377240-vmss00000b | Running | 0.500 | 1.00Gi |
| ameide-staging | postgres-ameide-2 | aks-system-10377240-vmss000005 | Running | 0.500 | 1.00Gi |
| ameide-staging | postgres-ameide-3 | aks-system-10377240-vmss00000c | Running | 0.500 | 1.00Gi |
| eclipse-che | che-gateway-57dbfb949d-btbb2 | aks-system-10377240-vmss000008 | Running | 0.350 | 0.31Gi |
| keycloak-system | keycloak-operator-65864689db-jtflg | aks-system-10377240-vmss000007 | Running | 0.300 | 0.44Gi |
| ameide-dev | data-minio-5db69d469b-rmfdd | aks-system-10377240-vmss000007 | Running | 0.250 | 0.25Gi |
| ameide-dev | platform-camunda8-connectors-55c49b89cd-v6h7g | aks-system-10377240-vmss000008 | Running | 0.250 | 0.50Gi |
| ameide-dev | platform-camunda8-elasticsearch-master-0 | aks-system-10377240-vmss000007 | Running | 0.250 | 1.00Gi |
| ameide-dev | platform-camunda8-identity-5fdcff6856-lpmbl | aks-system-10377240-vmss000008 | Running | 0.250 | 0.50Gi |
| ameide-dev | platform-camunda8-web-modeler-restapi-54dbfb54d7-4x5hm | aks-system-10377240-vmss000008 | Running | 0.250 | 0.50Gi |
| ameide-dev | platform-camunda8-web-modeler-webapp-5bb9d98868-6xm2z | aks-system-10377240-vmss000008 | Running | 0.250 | 0.50Gi |
| ameide-dev | platform-camunda8-web-modeler-websockets-b6d98b8d6-8sgvj | aks-system-10377240-vmss000008 | Running | 0.250 | 0.50Gi |
| ameide-dev | platform-camunda8-zeebe-0 | aks-system-10377240-vmss000007 | Running | 0.250 | 1.00Gi |
| ameide-dev | platform-extensions-runtime-7486f6c67-4mrf2 | aks-system-10377240-vmss000008 | Running | 0.250 | 0.50Gi |
| ameide-dev | platform-extensions-runtime-7486f6c67-5mlh4 | aks-system-10377240-vmss000008 | Running | 0.250 | 0.50Gi |
| ameide-dev | temporal-frontend-745c5b47dd-bpvnh | aks-system-10377240-vmss000008 | Running | 0.250 | 0.50Gi |
| ameide-dev | temporal-frontend-745c5b47dd-zt2cj | aks-system-10377240-vmss000008 | Running | 0.250 | 0.50Gi |
| ameide-dev | temporal-history-d4d964ff8-fscgp | aks-system-10377240-vmss000008 | Running | 0.250 | 0.50Gi |
| ameide-dev | temporal-history-d4d964ff8-tmwv5 | aks-system-10377240-vmss000008 | Running | 0.250 | 0.50Gi |
| ameide-dev | temporal-matching-67cbb6c464-247t5 | aks-system-10377240-vmss000008 | Running | 0.250 | 0.50Gi |
| ameide-dev | temporal-matching-67cbb6c464-2qvt7 | aks-system-10377240-vmss000008 | Running | 0.250 | 0.50Gi |
| ameide-dev | temporal-worker-5687bd7784-7tcxp | aks-system-10377240-vmss000008 | Running | 0.250 | 0.50Gi |
| ameide-dev | temporal-worker-5687bd7784-ptxlz | aks-system-10377240-vmss000008 | Running | 0.250 | 0.50Gi |
| ameide-prod | platform-camunda8-connectors-77bf5d8bd9-tm5gl | aks-system-10377240-vmss000007 | Running | 0.250 | 0.50Gi |
| ameide-prod | platform-camunda8-elasticsearch-master-0 | aks-system-10377240-vmss000005 | Running | 0.250 | 1.00Gi |
| ameide-prod | platform-camunda8-zeebe-0 | aks-system-10377240-vmss000005 | Running | 0.250 | 1.00Gi |
| ameide-prod | platform-extensions-runtime-587694db5b-5cdvv | aks-system-10377240-vmss000007 | Running | 0.250 | 0.50Gi |
| ameide-prod | platform-extensions-runtime-587694db5b-8b9np | aks-system-10377240-vmss000007 | Running | 0.250 | 0.50Gi |
| ameide-prod | platform-extensions-runtime-587694db5b-lgxkh | aks-system-10377240-vmss000008 | Running | 0.250 | 0.50Gi |
| ameide-prod | plausible-plausible-6b5c446b68-zzxpn | aks-system-10377240-vmss000007 | Running | 0.250 | 0.75Gi |
| ameide-prod | temporal-frontend-699f67d6c4-2lr7s | aks-system-10377240-vmss00000b | Running | 0.250 | 0.50Gi |
| ameide-prod | temporal-frontend-699f67d6c4-szvd7 | aks-system-10377240-vmss000005 | Running | 0.250 | 0.50Gi |
| ameide-prod | temporal-history-777987c4dd-7qlcc | aks-system-10377240-vmss00000b | Running | 0.250 | 0.50Gi |
| ameide-prod | temporal-history-777987c4dd-rww9v | aks-system-10377240-vmss000005 | Running | 0.250 | 0.50Gi |
| ameide-prod | temporal-matching-7ccd56bd5d-xd4gj | aks-system-10377240-vmss00000b | Running | 0.250 | 0.50Gi |
| ameide-prod | temporal-matching-7ccd56bd5d-xxzk9 | aks-system-10377240-vmss000005 | Running | 0.250 | 0.50Gi |
| ameide-prod | temporal-worker-7987489cbb-dndbs | aks-system-10377240-vmss00000b | Running | 0.250 | 0.50Gi |
| ameide-prod | temporal-worker-7987489cbb-flhwl | aks-system-10377240-vmss000005 | Running | 0.250 | 0.50Gi |
| ameide-staging | platform-camunda8-connectors-c9df8f8d-r8vpv | aks-system-10377240-vmss00000b | Running | 0.250 | 0.50Gi |
| ameide-staging | platform-camunda8-elasticsearch-master-0 | aks-system-10377240-vmss000005 | Running | 0.250 | 1.00Gi |
| ameide-staging | platform-camunda8-zeebe-0 | aks-system-10377240-vmss000005 | Running | 0.250 | 1.00Gi |
| ameide-staging | platform-extensions-runtime-cffbf75c8-kcvhk | aks-system-10377240-vmss000005 | Running | 0.250 | 0.50Gi |
| ameide-staging | platform-extensions-runtime-cffbf75c8-w6pmf | aks-system-10377240-vmss00000b | Running | 0.250 | 0.50Gi |
| ameide-staging | temporal-frontend-6d776cb8d-jspfr | aks-system-10377240-vmss00000b | Running | 0.250 | 0.50Gi |
| ameide-staging | temporal-frontend-6d776cb8d-kjzsp | aks-system-10377240-vmss000005 | Running | 0.250 | 0.50Gi |
| ameide-staging | temporal-history-545987dcfd-74jvp | aks-system-10377240-vmss000005 | Running | 0.250 | 0.50Gi |
| ameide-staging | temporal-history-545987dcfd-ltglg | aks-system-10377240-vmss00000b | Running | 0.250 | 0.50Gi |
| ameide-staging | temporal-matching-58c96bd6cb-4v279 | aks-system-10377240-vmss000005 | Running | 0.250 | 0.50Gi |
| ameide-staging | temporal-matching-58c96bd6cb-tnd9w | aks-system-10377240-vmss00000b | Running | 0.250 | 0.50Gi |
| ameide-staging | temporal-worker-86c548b76d-d4q2k | aks-system-10377240-vmss00000b | Running | 0.250 | 0.50Gi |
| ameide-staging | temporal-worker-86c548b76d-xth9j | aks-system-10377240-vmss000005 | Running | 0.250 | 0.50Gi |
| ameide-staging | www-ameide-77f7cfdd75-r6zbp | aks-system-10377240-vmss000007 | Running | 0.250 | 0.25Gi |
| ameide-staging | www-ameide-77f7cfdd75-vptxv | aks-system-10377240-vmss000007 | Running | 0.250 | 0.25Gi |
| ameide-ws-52841969-f3ca-4f53-a747-1864fd7e0699 | coder-52841969-f3ca-4f53-a747-1864fd7e0699-8669fbdbb4-vfdfq | aks-system-10377240-vmss00000c | Running | 0.250 | 0.50Gi |
| ameide-ws-91648837-bed9-4fa3-b5f9-3e5f47099782 | coder-91648837-bed9-4fa3-b5f9-3e5f47099782-6c8d778677-qx29n | aks-system-10377240-vmss00000c | Running | 0.250 | 0.50Gi |
| ameide-ws-aed4e053-e187-455d-a9d2-e3d530a9bf4d | coder-aed4e053-e187-455d-a9d2-e3d530a9bf4d-58c498b549-p2qjx | aks-system-10377240-vmss00000c | Running | 0.250 | 0.50Gi |
| argocd | argocd-application-controller-0 | aks-system-10377240-vmss00000b | Running | 0.250 | 2.00Gi |
| argocd | argocd-application-controller-1 | aks-system-10377240-vmss000007 | Running | 0.250 | 2.00Gi |
| argocd | argocd-repo-server-8455f5b4d5-vzb6t | aks-system-10377240-vmss000005 | Running | 0.250 | 0.50Gi |
| argocd | argocd-repo-server-8455f5b4d5-x4vpd | aks-system-10377240-vmss00000b | Running | 0.250 | 0.50Gi |
| argocd | argocd-server-6b46bdf94c-7hljz | aks-system-10377240-vmss00000b | Running | 0.250 | 0.50Gi |
| argocd | argocd-server-6b46bdf94c-hvn5n | aks-system-10377240-vmss00000c | Running | 0.250 | 0.50Gi |
| devworkspace-controller | devworkspace-controller-manager-cf9db9b68-9ds8h | aks-system-10377240-vmss00000c | Running | 0.250 | 0.10Gi |
| redis-system | redis-operator-db6768894-9m74x | aks-system-10377240-vmss000007 | Running | 0.250 | 0.25Gi |
| ameide-dev | coder-7fc6b5cc99-slcgk | aks-system-10377240-vmss00000c | Running | 0.200 | 0.50Gi |
| ameide-dev | kafka-entity-operator-86d4646845-tfvjf | aks-system-10377240-vmss000008 | Running | 0.200 | 0.50Gi |
| ameide-prod | kafka-entity-operator-5b4bb5cbd4-w5766 | aks-system-10377240-vmss000008 | Running | 0.200 | 0.50Gi |
| ameide-prod | platform-backstage-666946744-r82nl | aks-system-10377240-vmss000008 | Running | 0.200 | 0.50Gi |
| ameide-staging | kafka-entity-operator-6f98dd987f-fnbm4 | aks-system-10377240-vmss000005 | Running | 0.200 | 0.50Gi |
| ameide-staging | plausible-plausible-78ccc7bb75-qqz2d | aks-system-10377240-vmss000005 | Running | 0.200 | 0.50Gi |
| strimzi-system | strimzi-cluster-operator-5488759995-45zpm | aks-system-10377240-vmss000007 | Running | 0.200 | 0.38Gi |
| kube-system | metrics-server-6fbdfd747b-269p4 | aks-system-10377240-vmss000005 | Running | 0.158 | 0.15Gi |
| kube-system | metrics-server-6fbdfd747b-2wlkw | aks-system-10377240-vmss000005 | Running | 0.158 | 0.15Gi |
| argocd | envoy-ameide-dev-ameide-7c898b73-d67994c48-4vp5n | aks-system-10377240-vmss000005 | Running | 0.110 | 0.53Gi |
| argocd | envoy-ameide-dev-ameide-7c898b73-d67994c48-k4b87 | aks-system-10377240-vmss00000b | Running | 0.110 | 0.53Gi |
| argocd | envoy-ameide-prod-ameide-270d1f22-5b65497db5-kfvwh | aks-system-10377240-vmss000005 | Running | 0.110 | 0.53Gi |
| argocd | envoy-ameide-prod-ameide-270d1f22-5b65497db5-l47nh | aks-system-10377240-vmss00000b | Running | 0.110 | 0.53Gi |
| argocd | envoy-ameide-staging-ameide-f462b9ba-f8c755fd7-6hqtx | aks-system-10377240-vmss00000b | Running | 0.110 | 0.53Gi |
| argocd | envoy-ameide-staging-ameide-f462b9ba-f8c755fd7-l4cmm | aks-system-10377240-vmss000005 | Running | 0.110 | 0.53Gi |
| argocd | envoy-argocd-ameide-dev-grpc-internal-fe61a74f-567d54d49-9pgp8 | aks-system-10377240-vmss00000c | Running | 0.110 | 0.53Gi |
| argocd | envoy-argocd-ameide-dev-grpc-internal-fe61a74f-567d54d49-g52cl | aks-system-10377240-vmss00000c | Running | 0.110 | 0.53Gi |
| argocd | envoy-argocd-ameide-prod-grpc-internal-cee3432f-64dcd57fb-5c72m | aks-system-10377240-vmss00000c | Running | 0.110 | 0.53Gi |
| argocd | envoy-argocd-ameide-prod-grpc-internal-cee3432f-64dcd57fb-87v5j | aks-system-10377240-vmss000008 | Running | 0.110 | 0.53Gi |
| argocd | envoy-argocd-ameide-staging-grpc-internal-851fe813-74d47d7m4zgd | aks-system-10377240-vmss000007 | Running | 0.110 | 0.53Gi |
| argocd | envoy-argocd-ameide-staging-grpc-internal-851fe813-74d47d7xcr52 | aks-system-10377240-vmss00000c | Running | 0.110 | 0.53Gi |
| argocd | envoy-argocd-cluster-18b70826-588978797f-6cdzt | aks-system-10377240-vmss00000b | Running | 0.110 | 0.53Gi |
| argocd | envoy-argocd-cluster-18b70826-588978797f-7ks8f | aks-system-10377240-vmss000005 | Running | 0.110 | 0.53Gi |
| ameide-dev | data-minio-console-795546b6dd-d2kfx | aks-system-10377240-vmss000008 | Running | 0.100 | 0.12Gi |
| ameide-dev | inference-gateway-5c785c6877-vmqrn | aks-system-10377240-vmss000008 | Running | 0.100 | 0.12Gi |
| ameide-dev | kafka-kafka-pool-0 | aks-system-10377240-vmss000007 | Running | 0.100 | 0.38Gi |
| ameide-dev | kubernetes-dashboard-api-5fd4d66f75-psvcz | aks-system-10377240-vmss000008 | Running | 0.100 | 0.20Gi |
| ameide-dev | kubernetes-dashboard-auth-7c8dd97946-m6l7s | aks-system-10377240-vmss000008 | Running | 0.100 | 0.20Gi |

### Pods with no cpu/mem requests at all: 102 (first 200)

| namespace | pod | node | phase |
|---|---|---|---|
| ameide-dev | agents-54f464cdbf-962bb | aks-system-10377240-vmss000008 | Running |
| ameide-dev | agents-runtime-7f6c886bbd-ppmj9 | aks-system-10377240-vmss000008 | Running |
| ameide-dev | chi-data-clickhouse-data-clickhouse-0-0-0 | aks-system-10377240-vmss00000b | Running |
| ameide-dev | data-pgadmin-servers-pgadmin-servers-58f5f8855c-vkckq | aks-system-10377240-vmss000008 | Running |
| ameide-dev | foundation-vault-bootstrap-vault-bootstrap-29474731-gtf86 | aks-system-10377240-vmss000007 | Running |
| ameide-dev | inference-5b85cc48d6-k6pgp | aks-system-10377240-vmss000008 | Running |
| ameide-dev | kubernetes-dashboard-kong-85f664bd9c-hcdjn | aks-system-10377240-vmss000008 | Running |
| ameide-dev | otel-collector-b84dd4f9-rrxdg | aks-system-10377240-vmss000008 | Running |
| ameide-dev | platform-54c5d4c958-md2hg | aks-system-10377240-vmss000008 | Running |
| ameide-dev | platform-grafana-5d86f794db-zxqcd | aks-system-10377240-vmss00000c | Running |
| ameide-dev | platform-keycloak-realm-master-bootstrap-ccc6p | aks-system-10377240-vmss00000b | Running |
| ameide-dev | platform-prometheus-kube-state-metrics-df5d9c749-tpp7z | aks-system-10377240-vmss000008 | Running |
| ameide-dev | platform-tempo-0 | aks-system-10377240-vmss00000b | Running |
| ameide-dev | postgres-password-reconcile-29474730-pxmh2 | aks-system-10377240-vmss000007 | Running |
| ameide-dev | prometheus-dev-prometheus-prometheus-0 | aks-system-10377240-vmss000008 | Running |
| ameide-dev | temporal-admintools-577f786c4b-5vd9s | aks-system-10377240-vmss000008 | Running |
| ameide-dev | temporal-ui-9b65875b7-qx8hb | aks-system-10377240-vmss000008 | Running |
| ameide-dev | threads-747db79645-g68xf | aks-system-10377240-vmss000008 | Running |
| ameide-dev | traffic-manager-7cc98897c-tq7f4 | aks-system-10377240-vmss000008 | Running |
| ameide-dev | transformation-v0-domain-67b8b6cc6c-f9jdk | aks-system-10377240-vmss000008 | Running |
| ameide-dev | transformation-v0-process-77dbc64d67-lvg8v | aks-system-10377240-vmss000008 | Running |
| ameide-dev | transformation-v0-projection-7fcbcb7884-t2sx5 | aks-system-10377240-vmss000008 | Running |
| ameide-dev | vault-core-dev-0 | aks-system-10377240-vmss00000c | Running |
| ameide-dev | vault-core-dev-agent-injector-66b757d695-v6gpf | aks-system-10377240-vmss000008 | Running |
| ameide-dev | workflows-6cc977c877-669hh | aks-system-10377240-vmss000008 | Running |
| ameide-dev | workflows-runtime-77d65ff54-57zlv | aks-system-10377240-vmss000008 | Running |
| ameide-dev | workrequests-agentwork-coder-pvvb2-dv2sv | aks-system-10377240-vmss000007 | Pending |
| ameide-dev | workrequests-toolrun-generate-7bqd8-lq5rl | aks-system-10377240-vmss000007 | Running |
| ameide-dev | workrequests-toolrun-verify-5dfnh-rs8l8 | aks-system-10377240-vmss000007 | Pending |
| ameide-dev | workrequests-toolrun-verify-ui-harness-mc2zt-xrkhg | aks-system-10377240-vmss000007 | Running |
| ameide-dev | www-ameide-5cc79f79cc-8fk8j | aks-system-10377240-vmss000008 | Running |
| ameide-dev | www-ameide-platform-998d5c694-b7lr6 | aks-system-10377240-vmss000008 | Running |
| ameide-prod | agents-6674c6ccbf-s9nvh | aks-system-10377240-vmss000008 | Running |
| ameide-prod | agents-runtime-7886d55d47-xbgxd | aks-system-10377240-vmss000008 | Running |
| ameide-prod | chi-data-clickhouse-data-clickhouse-0-0-0 | aks-system-10377240-vmss00000b | Running |
| ameide-prod | data-pgadmin-servers-pgadmin-servers-794dc8b9fc-792t7 | aks-system-10377240-vmss000008 | Running |
| ameide-prod | foundation-vault-bootstrap-vault-bootstrap-29474732-q87kd | aks-system-10377240-vmss000007 | Pending |
| ameide-prod | kubernetes-dashboard-kong-85f664bd9c-fjbws | aks-system-10377240-vmss000008 | Running |
| ameide-prod | otel-collector-7f9d4965bb-7l4r4 | aks-system-10377240-vmss000008 | Running |
| ameide-prod | platform-586888bdb4-ss9gm | aks-system-10377240-vmss000008 | Running |
| ameide-prod | platform-prometheus-kube-state-metrics-6d5748f85-lmlk5 | aks-system-10377240-vmss000008 | Running |
| ameide-prod | platform-tempo-0 | aks-system-10377240-vmss00000c | Running |
| ameide-prod | prometheus-production-prometheus-prometheus-0 | aks-system-10377240-vmss00000c | Running |
| ameide-prod | temporal-admintools-67c74c84dd-j8pzz | aks-system-10377240-vmss000008 | Running |
| ameide-prod | temporal-ui-79499b4849-8k2t2 | aks-system-10377240-vmss000008 | Running |
| ameide-prod | traffic-manager-9c677698d-s7qnc | aks-system-10377240-vmss000008 | Running |
| ameide-prod | transformation-v0-domain-67b8b6cc6c-lvxc2 | aks-system-10377240-vmss000008 | Running |
| ameide-prod | transformation-v0-projection-7fcbcb7884-9t88w | aks-system-10377240-vmss000008 | Running |
| ameide-prod | vault-core-prod-0 | aks-system-10377240-vmss00000b | Running |
| ameide-prod | vault-core-prod-agent-injector-75d55f78bc-w5hpb | aks-system-10377240-vmss000008 | Running |
| ameide-prod | workflows-76656c647f-nrbtp | aks-system-10377240-vmss000008 | Running |
| ameide-prod | workflows-runtime-7fd56db7d4-vg62w | aks-system-10377240-vmss000008 | Running |
| ameide-prod | www-ameide-f856559f4-s25cw | aks-system-10377240-vmss000008 | Running |
| ameide-staging | agents-74dd8b67f9-6t4sr | aks-system-10377240-vmss000008 | Running |
| ameide-staging | agents-runtime-7f7698dff7-w6ldc | aks-system-10377240-vmss000008 | Running |
| ameide-staging | chi-data-clickhouse-data-clickhouse-0-0-0 | aks-system-10377240-vmss00000b | Running |
| ameide-staging | data-pgadmin-servers-pgadmin-servers-756d88d97c-2tm9d | aks-system-10377240-vmss000008 | Running |
| ameide-staging | foundation-vault-bootstrap-vault-bootstrap-29474732-lvldl | aks-system-10377240-vmss000007 | Pending |
| ameide-staging | kubernetes-dashboard-kong-85f664bd9c-bflf9 | aks-system-10377240-vmss00000b | Running |
| ameide-staging | otel-collector-7cdb7666f-fn8lg | aks-system-10377240-vmss00000c | Running |
| ameide-staging | platform-556b8d6776-b6wnr | aks-system-10377240-vmss00000b | Running |
| ameide-staging | platform-grafana-7879d679b7-b4c9j | aks-system-10377240-vmss00000c | Running |
| ameide-staging | platform-prometheus-kube-state-metrics-745584d685-q6lmq | aks-system-10377240-vmss00000b | Running |
| ameide-staging | platform-tempo-0 | aks-system-10377240-vmss00000c | Running |
| ameide-staging | prometheus-staging-prometheus-prometheus-0 | aks-system-10377240-vmss00000b | Running |
| ameide-staging | temporal-admintools-796d756458-59lmr | aks-system-10377240-vmss000005 | Running |
| ameide-staging | temporal-ui-589d9b5ddc-rzkwd | aks-system-10377240-vmss000005 | Running |
| ameide-staging | traffic-manager-54b776b96b-bqc8p | aks-system-10377240-vmss000005 | Running |
| ameide-staging | transformation-v0-domain-67b8b6cc6c-hqlrg | aks-system-10377240-vmss000005 | Running |
| ameide-staging | transformation-v0-projection-7fcbcb7884-dqghh | aks-system-10377240-vmss000005 | Running |
| ameide-staging | vault-core-staging-0 | aks-system-10377240-vmss000008 | Running |
| ameide-staging | vault-core-staging-agent-injector-85d66b4db7-lktjb | aks-system-10377240-vmss000005 | Running |
| ameide-staging | workflows-79494546fc-6j8x8 | aks-system-10377240-vmss000005 | Running |
| ameide-staging | workflows-runtime-67d89cb448-sb98b | aks-system-10377240-vmss00000b | Running |
| ameide-staging | www-ameide-platform-856f88dd88-5dwz9 | aks-system-10377240-vmss00000b | Running |
| arc-systems | arc-aks-696897c5-listener | aks-system-10377240-vmss000005 | Running |
| arc-systems | github-arc-controller-gha-rs-controller-6db79bbccf-z4smb | aks-system-10377240-vmss00000b | Running |
| argocd | argocd-applicationset-controller-6c55d779dd-j66tr | aks-system-10377240-vmss000005 | Running |
| argocd | argocd-applicationset-controller-6c55d779dd-pl6n4 | aks-system-10377240-vmss00000b | Running |
| argocd | argocd-dex-server-f8b98dd59-wrtkd | aks-system-10377240-vmss000005 | Running |
| argocd | argocd-notifications-controller-687cd7c44-rvqgr | aks-system-10377240-vmss000005 | Running |
| argocd | argocd-redis-5ff45f994-kzq76 | aks-system-10377240-vmss000005 | Running |
| buildkit | binfmt-hv8kr | aks-system-10377240-vmss00000b | Running |
| buildkit | binfmt-ktz2s | aks-system-10377240-vmss00000c | Running |
| buildkit | binfmt-rsjt2 | aks-system-10377240-vmss000007 | Running |
| buildkit | binfmt-skvdg | aks-system-10377240-vmss000005 | Running |
| buildkit | binfmt-tt84v | aks-system-10377240-vmss000008 | Running |
| clickhouse-system | clickhouse-operator-7797bd574d-qz52f | aks-system-10377240-vmss000005 | Running |
| cnpg-system | cloudnative-pg-79cc8df686-pnx5q | aks-system-10377240-vmss00000b | Running |
| external-secrets | external-secrets-5c9bbdbb75-bg89s | aks-system-10377240-vmss000005 | Running |
| external-secrets | external-secrets-cert-controller-cdd97d9f9-22rk8 | aks-system-10377240-vmss00000b | Running |
| external-secrets | external-secrets-webhook-b9444bf8b-7rt2x | aks-system-10377240-vmss000005 | Running |
| kube-system | azure-cns-2f642 | aks-system-10377240-vmss00000c | Running |
| kube-system | azure-cns-b48b7 | aks-system-10377240-vmss000008 | Running |
| kube-system | azure-cns-lhg6n | aks-system-10377240-vmss00000b | Running |
| kube-system | azure-cns-vk2q4 | aks-system-10377240-vmss000007 | Running |
| kube-system | azure-cns-wlnph | aks-system-10377240-vmss000005 | Running |
| prometheus-system | prometheus-operator-prometheus-node-exporter-k2mfg | aks-system-10377240-vmss000005 | Running |
| prometheus-system | prometheus-operator-prometheus-node-exporter-kq88p | aks-system-10377240-vmss00000b | Running |
| prometheus-system | prometheus-operator-prometheus-node-exporter-p46bn | aks-system-10377240-vmss000007 | Running |
| prometheus-system | prometheus-operator-prometheus-node-exporter-prxjd | aks-system-10377240-vmss00000c | Running |
| prometheus-system | prometheus-operator-prometheus-node-exporter-xtc8p | aks-system-10377240-vmss000008 | Running |
\n## Raw inputs for reproduction
```
kubectl get pods -A -o json
kubectl get nodes -o json
python3 <embedded script in this file>
```
