# System specs
```bash
[vagrant@rocky9 ~]$ cat /etc/os-release
NAME="Rocky Linux"
VERSION="9.3 (Blue Onyx)"
ID="rocky"
ID_LIKE="rhel centos fedora"
VERSION_ID="9.3"
PLATFORM_ID="platform:el9"
PRETTY_NAME="Rocky Linux 9.3 (Blue Onyx)"
ANSI_COLOR="0;32"
LOGO="fedora-logo-icon"
CPE_NAME="cpe:/o:rocky:rocky:9::baseos"
HOME_URL="https://rockylinux.org/"
BUG_REPORT_URL="https://bugs.rockylinux.org/"
SUPPORT_END="2032-05-31"
ROCKY_SUPPORT_PRODUCT="Rocky-Linux-9"
ROCKY_SUPPORT_PRODUCT_VERSION="9.3"
REDHAT_SUPPORT_PRODUCT="Rocky Linux"
REDHAT_SUPPORT_PRODUCT_VERSION="9.3"

[vagrant@rocky9 ~]$ uname -a
Linux rocky9.localdomain 5.14.0-362.13.1.el9_3.x86_64 #1 SMP PREEMPT_DYNAMIC Wed Dec 13 14:07:45 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux


[vagrant@rocky9 ~]$ free -h
               total        used        free      shared  buff/cache   available
Mem:           7.7Gi       1.8Gi       3.8Gi        68Mi       2.5Gi       6.0Gi
Swap:          2.0Gi          0B       2.0Gi
```

#  1. 請以kind ( https://kind.sigs.k8s.io/) 架設一個1個control-plane 節點，以及3個worker 節點 。
## Prerequisites
* Docker Engine -- follow Docker official website for installation https://docs.docker.com/engine/install/rhel/

## Instructions
* Follow kind official website to create cluster https://kind.sigs.k8s.io/docs/user/quick-start/
```bash
sudo kind create cluster --config kind-cluster-config.yaml
```

* Follow Kubernetes official website to install kubectl https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
## Results
```bash
[vagrant@rocky9 ~]$ sudo kind get clusters
kind


[vagrant@rocky9 ~]$ sudo kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:35099
CoreDNS is running at https://127.0.0.1:35099/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.


[vagrant@rocky9 ~]$ sudo kubectl get nodes
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   19m   v1.31.2
kind-worker          Ready    <none>          10m   v1.31.2
kind-worker2         Ready    <none>          10m   v1.31.2
kind-worker3         Ready    <none>          10m   v1.31.2
```

# 2. 節點分為2群角色或功能:

Infra node: 3個worker中的1個節點 作為infra node。

Application node: 3 個worker 中的2 個節點 作為 application node。

## Results
+ kind-worker is infra node
+ kind-worker2 is app node
+ kind-worker3 is app node

```bash
[vagrant@rocky9 ~]$ sudo kubectl get nodes --show-labels
NAME                 STATUS   ROLES           AGE   VERSION   LABELS
kind-control-plane   Ready    control-plane   17h   v1.31.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-control-plane,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
kind-worker          Ready    <none>          17h   v1.31.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-worker,kubernetes.io/os=linux,node-role.kubernetes.io=infra
kind-worker2         Ready    <none>          17h   v1.31.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-worker2,kubernetes.io/os=linux,node-role.kubernetes.io=app
kind-worker3         Ready    <none>          17h   v1.31.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kind-worker3,kubernetes.io/os=linux,node-role.kubernetes.io=app
```

#  3 . 安裝 MetalLB ，以L2 模式安裝，speaker 部署在 infra node上。
## Instructions
* Download metallb-native.yaml mentioned in https://metallb.universe.tf/installation/

* In the downloaded yaml, adjust speaker component kind from DaemonSet to Deployment

* In the speaker component section, add spec.replica: 1

* In the speaker component section, add spec.template.spec.nodeSelector.node-role.kubernetes.io: "infra"

* Apply this modified yaml

## Results
```bash
[vagrant@rocky9 ~]$ sudo kubectl get all -n metallb-system -o wide
NAME                              READY   STATUS    RESTARTS   AGE   IP            NODE           NOMINATED NODE   READINESS GATES
pod/controller-8694df9d9b-4h2kx   1/1     Running   0          13m   10.244.1.17   kind-worker3   <none>           <none>
pod/speaker-7ff6d8c7b4-67dzj      1/1     Running   0          13m   172.18.0.5    kind-worker    <none>           <none>

NAME                              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/metallb-webhook-service   ClusterIP   10.96.130.25   <none>        443/TCP   14m   component=controller

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                               SELECTOR
deployment.apps/controller   1/1     1            1           14m   controller   quay.io/metallb/controller:v0.14.8   app=metallb,component=controller
deployment.apps/speaker      1/1     1            1           14m   speaker      quay.io/metallb/speaker:v0.14.8      app=metallb,component=speaker

NAME                                    DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                               SELECTOR
replicaset.apps/controller-8694df9d9b   1         1         1       14m   controller   quay.io/metallb/controller:v0.14.8   app=metallb,component=controller,pod-template-hash=8694df9d9b
replicaset.apps/speaker-7ff6d8c7b4      1         1         1       14m   speaker      quay.io/metallb/speaker:v0.14.8      app=metallb,component=speaker,pod-template-hash=7ff6d8c7b4
```

# 4. 安裝 Prometheus, kube-state-metrics 在infra node 節點上，所有的節點安裝node exporter，Prometheus 收集node exporter, kube-state-metrics的效能數據。

## Prerequisites
* Configure firewall rules to allow incoming 9090/tcp
* Helm -- follow Helm official website for installation https://helm.sh/docs/intro/install/

## Instructions
* Install Prometheus using Docker https://hub.docker.com/r/prom/prometheus/
* Install node-exporter on Docker https://hub.docker.com/r/prom/node-exporter
   > The node_exporter is designed to monitor the host system. Deploying in containers requires extra care in order to avoid monitoring the container itself.
* Create compose.yaml to for prometheus service and node_exporter service
* docker compose up to start prometheus service and node_exporter service
```bash
version: '3'
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    restart: unless-stopped
    ports:
      - 9090:9090
  node-exporter:
    image: prom/node-exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped
    ports:
      - 9100:9100
```

* Install kube-state-metrics in the cluster using Helm

* Create Prometheus configuration file - prometheus.yml to collect metrics from node_exporter and from kube-state-metrics
code snippet
```bash
scrape_configs: 
  - job_name: "node"
    static_configs:
      - targets: ["node-exporter:9100"]
  - job_name: "kube-state-metrics"
    static_configs:
      - targets: ["localhost:8080"]
```

# 5. 安裝 Grafana 在kind叢集外，以docker或podman 執行，datasouce指向 Prometheus，並呈現3個效能監控儀表板。

## Prerequisites
* Configure firewall rules to allow incoming 3000/tcp

## Instructions
* Install Grafana using Docker https://hub.docker.com/r/grafana/grafana
* admin/admin log in http://localhost:3000

## Instructions
時間不足未答
* Follow Grafana Docker Hub for installation https://hub.docker.com/r/grafana/grafana

      5.1 效能監控儀表板(1): 呈現node的效能監控數據

      5.2 效能監控儀表板(2): 呈現 cluster的效能監控數據

      5.3 效能監控儀表板(3): 呈現 USE(Utilization, Saturation, Errors)角度的效能監控數據

      5.4 效能監控儀表板(4): 呈現 etcd的效能監控數據

      5.5 效能監控儀表板(5): 呈現 Prometheus效能監控數據

      5.6 請說明以上的效能監控儀表板的每個panel內容。

      5.7 請說明要如何以上建立的監控儀表板觀察CPU Throttling現象

             或是需要再新增新的監控panel或是儀表板來監控，請說明新增的原因。

# 6. 請部署一個容器應用程式在application node，建立一個hpa物件以cpu 使用率到達50%為條件，最多擴充到10個pod。
## Prerequisites
* Deploy a Metric Server in cluster as described here https://github.com/kubernetes-sigs/metrics-server
```bash
[vagrant@rocky9 ~]$ sudo kubectl get apiservices | grep metric
v1beta1.metrics.k8s.io                 kube-system/metrics-server   True        3h50m
```

## Instructions
* Deploy a busy depoloyment workloads
```bash
[vagrant@rocky9 ~]$ sudo kubectl describe deployment/deployment-on-app-node
Name:                   deployment-on-app-node
Namespace:              default
CreationTimestamp:      Sat, 30 Nov 2024 11:15:28 +0000
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               node-role.kubernetes.io=app
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  node-role.kubernetes.io=app
  Containers:
   deployment-on-app-node:
    Image:      alpine
    Port:       <none>
    Host Port:  <none>
    Command:
      /bin/sh
      -c
      while true; do echo $(date); done
    Limits:
      cpu:  500m
    Requests:
      cpu:         100m
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  node-role.kubernetes.io=app
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  deployment-on-app-node-6746d7d7d4 (0/0 replicas created)
NewReplicaSet:   deployment-on-app-node-7d8f849667 (2/2 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  87s   deployment-controller  Scaled up replica set deployment-on-app-node-7d8f849667 to 1
  Normal  ScalingReplicaSet  84s   deployment-controller  Scaled down replica set deployment-on-app-node-6746d7d7d4 to 1 from 2
  Normal  ScalingReplicaSet  84s   deployment-controller  Scaled up replica set deployment-on-app-node-7d8f849667 to 2 from 1
  Normal  ScalingReplicaSet  82s   deployment-controller  Scaled down replica set deployment-on-app-node-6746d7d7d4 to 0 from 1
```

* Deploy a horizontal pod autoscaler (HPA)
```bash
[vagrant@rocky9 ~]$ sudo kubectl describe horizontalpodautoscaler.autoscaling/deployment-on-app-node
Name:                                                  deployment-on-app-node
Namespace:                                             default
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Sat, 30 Nov 2024 10:45:09 +0000
Reference:                                             Deployment/deployment-on-app-node
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  45% (45m) / 50%
Min replicas:                                          1
Max replicas:                                          10
Deployment pods:                                       10 current / 10 desired
Conditions:
  Type            Status  Reason               Message
  ----            ------  ------               -------
  AbleToScale     True    ScaleDownStabilized  recent recommendations were higher than current one, applying the highest recent recommendation
  ScalingActive   True    ValidMetricFound     the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  True    TooManyReplicas      the desired replica count is more than the maximum replica count
Events:
  Type    Reason             Age    From                       Message
  ----    ------             ----   ----                       -------
  Normal  SuccessfulRescale  5m5s   horizontal-pod-autoscaler  New size: 4; reason: cpu resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  4m30s  horizontal-pod-autoscaler  New size: 8; reason:
  Normal  SuccessfulRescale  89s    horizontal-pod-autoscaler  New size: 10; reason: cpu resource utilization (percentage of request) above target
```

## Results
```bash
[vagrant@rocky9 ~]$ sudo kubectl get pods -o wide
NAME                                      READY   STATUS    RESTARTS   AGE     IP            NODE           NOMINATED NODE   READINESS GATES
deployment-on-app-node-7d8f849667-2lv5f   1/1     Running   0          21s     10.244.1.14   kind-worker3   <none>           <none>
deployment-on-app-node-7d8f849667-68g9g   1/1     Running   0          21s     10.244.3.13   kind-worker2   <none>           <none>
deployment-on-app-node-7d8f849667-6g9gd   1/1     Running   0          36s     10.244.1.13   kind-worker3   <none>           <none>
deployment-on-app-node-7d8f849667-7x5b7   1/1     Running   0          47s     10.244.1.11   kind-worker3   <none>           <none>
deployment-on-app-node-7d8f849667-dw895   1/1     Running   0          4m18s   10.244.3.9    kind-worker2   <none>           <none>
deployment-on-app-node-7d8f849667-m8b92   1/1     Running   0          36s     10.244.1.12   kind-worker3   <none>           <none>
deployment-on-app-node-7d8f849667-n7zr7   1/1     Running   0          4m15s   10.244.1.10   kind-worker3   <none>           <none>
deployment-on-app-node-7d8f849667-n8trj   1/1     Running   0          36s     10.244.3.11   kind-worker2   <none>           <none>
deployment-on-app-node-7d8f849667-nbr8x   1/1     Running   0          36s     10.244.3.12   kind-worker2   <none>           <none>
deployment-on-app-node-7d8f849667-tmb8d   1/1     Running   0          47s     10.244.3.10   kind-worker2   <none>           <none>


[vagrant@rocky9 ~]$ sudo kubectl top pods
NAME                                      CPU(cores)   MEMORY(bytes)
deployment-on-app-node-7d8f849667-2lv5f   130m         0Mi
deployment-on-app-node-7d8f849667-68g9g   135m         0Mi
deployment-on-app-node-7d8f849667-6g9gd   135m         0Mi
deployment-on-app-node-7d8f849667-7x5b7   134m         0Mi
deployment-on-app-node-7d8f849667-dw895   125m         1Mi
deployment-on-app-node-7d8f849667-m8b92   132m         0Mi
deployment-on-app-node-7d8f849667-n7zr7   130m         1Mi
deployment-on-app-node-7d8f849667-n8trj   132m         0Mi
deployment-on-app-node-7d8f849667-nbr8x   137m         0Mi
deployment-on-app-node-7d8f849667-tmb8d   137m         0Mi


[vagrant@rocky9 ~]$ sudo kubectl get all
NAME                                          READY   STATUS    RESTARTS   AGE
pod/deployment-on-app-node-7d8f849667-2lv5f   1/1     Running   0          82s
pod/deployment-on-app-node-7d8f849667-68g9g   1/1     Running   0          82s
pod/deployment-on-app-node-7d8f849667-6g9gd   1/1     Running   0          97s
pod/deployment-on-app-node-7d8f849667-7x5b7   1/1     Running   0          108s
pod/deployment-on-app-node-7d8f849667-dw895   1/1     Running   0          5m19s
pod/deployment-on-app-node-7d8f849667-m8b92   1/1     Running   0          97s
pod/deployment-on-app-node-7d8f849667-n7zr7   1/1     Running   0          5m16s
pod/deployment-on-app-node-7d8f849667-n8trj   1/1     Running   0          97s
pod/deployment-on-app-node-7d8f849667-nbr8x   1/1     Running   0          97s
pod/deployment-on-app-node-7d8f849667-tmb8d   1/1     Running   0          108s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   22h

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/deployment-on-app-node   10/10   10           10          155m

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/deployment-on-app-node-6746d7d7d4   0         0         0       155m
replicaset.apps/deployment-on-app-node-7d8f849667   10        10        10      5m19s

NAME                                                         REFERENCE                           TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/deployment-on-app-node   Deployment/deployment-on-app-node   cpu: 132%/50%   1         10        10         2m14s

```

# 7. 請製作架構圖說明(圖檔)以上的架構以及操作的方法、使用到的配置檔(yaml, helm 等)、說明檔(以markdown格式呈現圖、內容)，上傳到github，並將github網址於下週xxx前回傳。
