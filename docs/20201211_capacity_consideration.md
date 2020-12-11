# Capacity and Utilization

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=4 orderedList=false} -->
<!-- code_chunk_output -->

- [Objective](#objective)
- [Executive Summary](#executive-summary)
- [Workload](#workload)
    - [All nodes](#all-nodes)
    - [Utilization but not very accurate](#utilization-but-not-very-accurate)
    - [Allocation](#allocation)
    - [Grep "Allocated" only](#grep-allocated-only)
    - [Grep name | cpu | memory](#grep-name-cpu-memory)
    - [View the usage, requests and limits](#view-the-usage-requests-and-limits)
    - [Top CPU Usage](#top-cpu-usage)
    - [Top Memory Usage](#top-memory-usage)

<!-- /code_chunk_output -->

## Objective

Understand the workload to calculate and plan the capacity required for production deployment at client data center

## Executive Summary


Since all compute has been designed and allocated running on Kubernetes, therefore in term of compute (pod), storage (nfs), network (ingress-controller), security (image and registry) and so on, the environment is quite standard and it's easy to measure the workload

When the resources are deployed in this environment, the install follows standard `helm chart` from community. We don't tune the parameter in chart unless otherwise we highlight in document. It means the resource allocation comes from open source community without specifically changed

One of event driven application integrated in this solution has a pre-requisite of running on Kubernetes up to `1.15.11`.

## Workload

Currently what we have

#### All nodes

```sh
nxyw@nxyw205:~$ kubectl get node -o wide
NAME      STATUS   ROLES    AGE     VERSION    INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
nxyw147   Ready    <none>   5d19h   v1.15.11   192.168.90.147   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   docker://19.3.14
nxyw201   Ready    <none>   142d    v1.15.11   192.168.90.201   <none>        Ubuntu 18.04.2 LTS      4.15.0-118-generic       docker://19.3.8
nxyw202   Ready    <none>   142d    v1.15.11   192.168.90.202   <none>        Ubuntu 18.04.2 LTS      4.15.0-126-generic       docker://19.3.8
nxyw203   Ready    <none>   142d    v1.15.11   192.168.90.203   <none>        Ubuntu 18.04.2 LTS      4.15.0-118-generic       docker://19.3.13
nxyw204   Ready    <none>   142d    v1.15.11   192.168.90.204   <none>        Ubuntu 18.04.2 LTS      4.15.0-118-generic       docker://19.3.8
nxyw205   Ready    master   142d    v1.15.11   192.168.90.205   <none>        Ubuntu 18.04.4 LTS      4.15.0-118-generic       docker://19.3.12
nxyw206   Ready    <none>   142d    v1.15.11   192.168.90.206   <none>        Ubuntu 18.04.2 LTS      4.15.0-126-generic       docker://19.3.8
```

#### Utilization but not very accurate

```sh
nxyw@nxyw205:~$ kubectl top node
NAME      CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
nxyw147   477m         2%     19986Mi         62%       
nxyw201   538m         4%     18018Mi         56%       
nxyw202   625m         7%     8175Mi          51%       
nxyw203   2789m        23%    19890Mi         62%       
nxyw204   756m         9%     17117Mi         53%       
nxyw205   790m         19%    3085Mi          39%       
nxyw206   1770m        22%    4627Mi          58%    
```

We can see the usage of CPU isn't very high, but memory in average is pretty busy.

#### Allocation

List all allocated resource on specific node
```sh
kubectl describe nodes
```

```sh
Name:               nxyw206
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=nxyw206
                    kubernetes.io/os=linux
Annotations:        flannel.alpha.coreos.com/backend-data: {"VtepMAC":"d6:a1:7c:7c:bf:e7"}
                    flannel.alpha.coreos.com/backend-type: vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager: true
                    flannel.alpha.coreos.com/public-ip: 192.168.90.206
                    kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Tue, 21 Jul 2020 21:37:06 +0800
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  nxyw206
  AcquireTime:     <unset>
  RenewTime:       Fri, 11 Dec 2020 10:42:43 +0800
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Sat, 05 Dec 2020 17:08:40 +0800   Sat, 05 Dec 2020 17:08:40 +0800   FlannelIsUp                  Flannel is running on this node
  MemoryPressure       False   Fri, 11 Dec 2020 10:42:39 +0800   Sat, 05 Dec 2020 17:08:09 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Fri, 11 Dec 2020 10:42:39 +0800   Sat, 05 Dec 2020 17:08:09 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Fri, 11 Dec 2020 10:42:39 +0800   Sat, 05 Dec 2020 17:08:09 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Fri, 11 Dec 2020 10:42:39 +0800   Sat, 05 Dec 2020 17:08:09 +0800   KubeletReady                 kubelet is posting ready status. AppArmor enabled
Addresses:
  InternalIP:  192.168.90.206
  Hostname:    nxyw206
Capacity:
  cpu:                8
  ephemeral-storage:  61661996Ki
  hugepages-2Mi:      0
  memory:             8167432Ki
  pods:               110
Allocatable:
  cpu:                8
  ephemeral-storage:  56827695420
  hugepages-2Mi:      0
  memory:             8065032Ki
  pods:               110
System Info:
  Machine ID:                 23255e7c3ffd40c59c7dbaf62e899e7c
  System UUID:                42075F21-BFAC-E949-9D86-50557C3612DB
  Boot ID:                    fed059cd-57cb-47bc-8f7e-9e675f8f9f94
  Kernel Version:             4.15.0-126-generic
  OS Image:                   Ubuntu 18.04.2 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://19.3.8
  Kubelet Version:            v1.15.11
  Kube-Proxy Version:         v1.15.11
PodCIDR:                      10.244.5.0/24
Non-terminated Pods:          (11 in total)
  Namespace                   Name                                  CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                   ----                                  ------------  ----------  ---------------  -------------  ---
  aiops-dev                   kangpaas-job-854b594c8f-pslcg         0 (0%)        0 (0%)      512Mi (6%)       1Gi (13%)      2d21h
  aiops-dev                   kangpaas-systemmgnt-f68755cf-6r86l    0 (0%)        0 (0%)      512Mi (6%)       1Gi (13%)      3d23h
  aiops-support               redis-slave-1                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         5d17h
  aiops-system                scan-scripts-6d7d977bb6-v7hlq         5 (62%)       32 (400%)   1Gi (13%)        8Gi (104%)     45h
  elastic                     filebeat-filebeat-825nx               100m (1%)     1 (12%)     100Mi (1%)       200Mi (2%)     7d
  kube-system                 kube-flannel-ds-amd64-rbb7c           100m (1%)     100m (1%)   50Mi (0%)        50Mi (0%)      142d
  kube-system                 kube-proxy-555k5                      0 (0%)        0 (0%)      0 (0%)           0 (0%)         142d
  monitor                     black-exporter                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         3d23h
  monitor                     prometheus-node-exporter-qqp56        0 (0%)        0 (0%)      0 (0%)           0 (0%)         133d
  nginx-ingress               nginx-ingress-controller-vbpsm        0 (0%)        0 (0%)      0 (0%)           0 (0%)         24d
  tidb-aiops                  tidb-cluster-tidb-1                   20m (0%)      100m (1%)   5Mi (0%)         50Mi (0%)      5d17h
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                5220m (65%)   33200m (415%)
  memory             2203Mi (27%)  10540Mi (133%)
  ephemeral-storage  0 (0%)        0 (0%)
  hugepages-2Mi      0 (0%)        0 (0%)
Events:              <none>
```

#### Grep "Allocated" only

```sh
nxyw@nxyw205:~$ kubectl describe nodes | grep -A5 "Allocated"

Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests       Limits
  --------           --------       ------
  cpu                5390m (33%)    36350m (227%)
  memory             13540Mi (42%)  27477724364800m (82%)
--
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests       Limits
  --------           --------       ------
  cpu                7560m (63%)    14350m (119%)
  memory             18988Mi (59%)  31772691660800m (94%)
--
```

#### Grep name | cpu | memory

```sh
kubectl describe nodes | grep 'Name:\|  cpu\|  memory'
Name:               nxyw147
  cpu:                16
  memory:             32778628Ki
  cpu:                16
  memory:             32676228Ki
  cpu                5390m (33%)    36350m (227%)
  memory             13540Mi (42%)  27477724364800m (82%)
Name:               nxyw201
  cpu:                12
  memory:             32939136Ki
  cpu:                12
  memory:             32836736Ki
  cpu                7560m (63%)    14350m (119%)
  memory             18988Mi (59%)  31772691660800m (94%)
...
```

#### View the usage, requests and limits

> Credit > https://github.com/kubernetes/kubernetes/issues/17512

```sh
join -1 2 -2 2 -a 1 -a 2 -o "2.1 0 1.3 2.3 2.5 1.4 2.4 2.6" -e '<wait>' \
  <( kubectl top pods --all-namespaces | sort --key 2 -b ) \
  <( kubectl get pods --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,"CPU_REQ(cores)":.spec.containers[*].resources.requests.cpu,"MEMORY_REQ(bytes)":.spec.containers[*].resources.requests.memory,"CPU_LIM(cores)":.spec.containers[*].resources.limits.cpu,"MEMORY_LIM(bytes)":.spec.containers[*].resources.limits.memory | sort --key 2 -b ) \
  | column -t -s' '
```

```sh
argocd             argocd-application-controller-7b88944878-9wgb5          21m         <none>          <none>          74Mi           <none>             <none>
argocd             argocd-dex-server-7d7c867548-2vfpn                      1m          <none>          <none>          10Mi           <none>             <none>
argocd             argocd-redis-78c9595d44-lf6kr                           2m          <none>          <none>          8Mi            <none>             <none>
argocd             argocd-repo-server-7bc955644b-9bhnl                     1m          <none>          <none>          77Mi           <none>             <none>
argocd             argocd-server-679b5fc948-9ptcb                          3m          <none>          <none>          32Mi           <none>             <none>
monitor            black-exporter                                          38m         <none>          <none>          32Mi           <none>             <none>
default            busybox-78c88d76df-wxffl                                0m          <none>          <none>          0Mi            <none>             <none>
cert-manager       cert-manager-5d8d74bb4d-4wfw2                           11m         <none>          <none>          22Mi           <none>             <none>
cert-manager       cert-manager-cainjector-5db54b6b45-rbrb6                3m          <none>          <none>          40Mi           <none>             <none>
cert-manager       cert-manager-webhook-7cd5d4fdd7-4tmmb                   1m          <none>          <none>          8Mi            <none>             <none>
aiops-system       contract-management-7d47bc869f-f8s65                    1m          <none>          <none>          58Mi           <none>             <none>
aiops-dev          contract-management-846fcd45bd-b7ff9                    1m          <none>          <none>          62Mi           <none>             <none>
kube-system        coredns-5cd9d9c6fc-lrwh4                                17m         100m            <none>          20Mi           70Mi               170Mi
kube-system        coredns-5cd9d9c6fc-m4zdh                                20m         100m            <none>          15Mi           70Mi               170Mi
aiops-dev          d3-room-89d97f98f-tx6j2                                 1m          <none>          <none>          60Mi           <none>             <none>
aiops-system       d3-room-bbd665c7c-lf9zm                                 6m          <none>          <none>          64Mi           <none>             <none>
aiops-dev-support  dev-kafka-0                                             207m        <none>          <none>          1331Mi         <none>             <none>
aiops-dev-support  dev-kafka-zookeeper-0                                   2m          250m            <none>          366Mi          256Mi              <none>
elastic            elasticsearch-master-0                                  37m         1               1               1476Mi         2Gi                2Gi
elastic            elasticsearch-master-1                                  43m         1               1               1476Mi         2Gi                2Gi
elastic            elasticsearch-master-2                                  7m          1               1               1486Mi         2Gi                2Gi
kube-system        etcd-nxyw205                                            116m        <none>          <none>          91Mi           <none>             <none>
...

kube-system        kube-proxy-jnpvw                                        3m          <none>          <none>          22Mi           <none>             <none>
kube-system        kube-proxy-p7l6d                                        3m          <none>          <none>          15Mi           <none>             <none>
kube-system        kube-proxy-pp7v2                                        2m          <none>          <none>          28Mi           <none>             <none>
kube-system        kube-proxy-spscj                                        2m          <none>          <none>          27Mi           <none>             <none>
kube-system        kube-scheduler-nxyw205                                  6m          100m            <none>          16Mi           <none>             <none>
elastic            logstash-logstash-0                                     21m         100m            1               1174Mi         1536Mi             1536Mi
kube-system        metrics-server-6f9c567d7-hvw4c                          3m          <none>          <none>          25Mi           <none>             <none>
default            my-nfs-release-nfs-client-provisioner-774d8c7dc7-n8nc2  4m          <none>          <none>          11Mi           <none>             <none>
NAMESPACE          NAME                                                    CPU(cores)  CPU_REQ(cores)  CPU_LIM(cores)  MEMORY(bytes)  MEMORY_REQ(bytes)  MEMORY_LIM(bytes)
default            nginx-7bb7cd8db5-b2lkg                                  0m          <none>          <none>          4Mi            <none>             <none>
nginx-ingress      nginx-ingress-controller-98xc4                          24m         <none>          <none>          154Mi          <none>             <none>
...
istio-system       prometheus-56fc7774c-6vz2j                              57m         10m             <none>          1139Mi         <none>             <none>
monitor            prometheus-alertmanager-64b8485dc5-gjjmp                5m          <none>          <none>          16Mi           <none>             <none>
...
```
#### Top CPU Usage

```sh
kubectl top pods --all-namespaces | sort --key 3 --numeric --reverse

aiops-dev           scan-scripts-dbc5f55d8-cslj2                             1294m        2229Mi          
tidb-aiops          tidb-cluster-tikv-1                                      392m         3366Mi          
tidb-aiops          tidb-cluster-tikv-2                                      316m         4424Mi          
tidb-aiops          tidb-cluster-tidb-1                                      284m         211Mi           
kube-system         kube-apiserver-nxyw205                                   265m         341Mi           
tidb-aiops          tidb-cluster-pd-1                                        216m         53Mi            
tidb-aiops          tidb-cluster-tikv-0                                      201m         3556Mi          
tidb-aiops          tidb-cluster-tidb-0                                      184m         277Mi           
kube-system         etcd-nxyw205                                             111m         86Mi            
elastic             elasticsearch-master-1                                   107m         1470Mi          
aiops-dev-support   dev-kafka-0                                              100m         1345Mi          
aiops-dev           kangpaas-job-854b594c8f-pslcg                            92m          576Mi           
...
```

#### Top Memory Usage

```sh
kubectl top pods --all-namespaces | sort --key 4 --numeric --reverse
aiops-system        kangpaas-monitormgnt-7595b58dc5-gqn4k                    28m          4821Mi          
tidb-aiops          tidb-cluster-tikv-2                                      299m         4425Mi          
tidb-aiops          tidb-cluster-tikv-0                                      126m         3556Mi          
tidb-aiops          tidb-cluster-tikv-1                                      366m         3366Mi          
aiops-dev           kangpaas-monitormgnt-974958dcd-sb5cw                     27m          2270Mi          
aiops-dev           scan-scripts-dbc5f55d8-cslj2                             1032m        2160Mi          
aiops-support       kafka-0                                                  67m          1978Mi          
aiops-system        scan-scripts-6d7d977bb6-v7hlq                            58m          1790Mi          
aiops-system        kangpaas-standingbook-55bc55fd77-2hdt6                   10m          1630Mi          
aiops-dev           kangpaas-standingbook-84844f5896-w6v9h                   15m          1629Mi          
elastic             elasticsearch-master-2                                   7m           1486Mi          
elastic             elasticsearch-master-0                                   27m          1478Mi          
elastic             elasticsearch-master-1                                   56m          1470Mi          
aiops-dev-support   dev-kafka-0                                              110m         1344Mi          
elastic             logstash-logstash-0                                      15m          1175Mi          
istio-system        prometheus-56fc7774c-6vz2j                               61m          1124Mi          
aiops-system        kangpaas-gate-778888c5b7-bbb8x                           3m           1075Mi          
```
