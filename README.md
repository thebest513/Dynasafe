# Dynasafe

## System specs
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
Mem:           1.9Gi       431Mi       788Mi        10Mi       929Mi       1.5Gi
Swap:          2.0Gi          0B       2.0Gi
```

1. 請以kind ( https://kind.sigs.k8s.io/) 架設一個1個control-plane 節點，以及3個worker 節點 。
### Prerequisites
* Docker Engine -- follow Docker official website for installation https://docs.docker.com/engine/install/rhel/

### Instructions
Follow kind official website to create cluster https://kind.sigs.k8s.io/docs/user/quick-start/
```bash
sudo kind create cluster --config kind-cluster-config.yaml
```

Follow Kubernetes official website to install kubectl https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
### Results
```bash
[vagrant@rocky9 ~]$ sudo /usr/local/bin/kind get clusters
kind

[vagrant@rocky9 ~]$ sudo /usr/local/bin/kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:35099
CoreDNS is running at https://127.0.0.1:35099/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

[vagrant@rocky9 ~]$ sudo /usr/local/bin/kubectl get nodes
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   19m   v1.31.2
kind-worker          Ready    <none>          10m   v1.31.2
kind-worker2         Ready    <none>          10m   v1.31.2
kind-worker3         Ready    <none>          10m   v1.31.2
```

2. 節點分為2群角色或功能:

     Infra node: 3個worker中的1個節點 作為infra node。

     Application node: 3 個worker 中的2 個節點 作為 application node。

+ 3 . 安裝 MetalLB ，以L2 模式安裝，speaker 部署在 infra node上。

4. 安裝 Prometheus, kube-state-metrics 在infra node 節點上，所有的節點安裝node exporter，Prometheus 收集node exporter, kube-state-metrics的效能數據。

5. 安裝 Grafana 在kind叢集外，以docker或podman 執行，datasouce指向 Prometheus，並呈現3個效能監控儀表板。

      5.1 效能監控儀表板(1): 呈現node的效能監控數據

      5.2 效能監控儀表板(2): 呈現 cluster的效能監控數據

      5.3 效能監控儀表板(3): 呈現 USE(Utilization, Saturation, Errors)角度的效能監控數據

      5.4 效能監控儀表板(4): 呈現 etcd的效能監控數據

      5.5 效能監控儀表板(5): 呈現 Prometheus效能監控數據

      5.6 請說明以上的效能監控儀表板的每個panel內容。

      5.7 請說明要如何以上建立的監控儀表板觀察CPU Throttling現象

             或是需要再新增新的監控panel或是儀表板來監控，請說明新增的原因。

4. 請部署一個容器應用程式在application node，建立一個hpa物件以cpu 使用率到達50%為條件，最多擴充到10個pod。

5. 請製作架構圖說明(圖檔)以上的架構以及操作的方法、使用到的配置檔(yaml, helm 等)、說明檔(以markdown格式呈現圖、內容)，上傳到github，並將github網址於下週xxx前回傳。
