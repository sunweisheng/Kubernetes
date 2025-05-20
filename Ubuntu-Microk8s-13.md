# 在Ubuntu Server中部署MicroK8s-1.30版本

## 准备环境

两台KVM虚拟机都是Ubuntu Server本操作系统，其中一台设为Master节点，另一台是Node节点，在两台虚拟机上都执行以下环境准备的命令：

````shell
sudo apt update
sudo apt upgrade

#安装必要工具
sudo apt install htop wget  telnet net-tools ntp -y

#查看自动时间同步服务状态
sudo systemctl status ntp

#查看时间同步服务器
ntpq -p

#查看防火墙状态
sudo ufw status
#Status: inactive代表防火墙未启用，系统的网络端口完全开放，无任何过滤规则生效
````

## 部署MicroK8s

在Master节点虚拟机上执行：

````shell
#查看microk8s的可用版本，要安装/stable版本
sudo snap info microk8s

#安装 MicroK8s（指定 1.30 版本）
sudo snap install microk8s --classic --channel=1.30/stable

# 验证安装 超时60秒
sudo microk8s status --wait-ready -t 60

#结果应该是：microk8s is not running. Use microk8s inspect for a deeper inspection.
#查看原因
journalctl | grep microk8s

#原因是registry.k8s.io/pause:3.7镜像拉取不下来（没有科学上网导致）
````

## 下载需要的镜像文件并导出tar文件

找另一台可以拉取的电脑下载镜像之后导出tar文件:

````shell
#寻找并下载镜像
docker pull registry.k8s.io/pause:3.7

#查看下载结果
docker image ls

#pause:3.7
#导出tar文件
docker save -o pause-3.7.tar registry.k8s.io/pause:3.7

#将tar文件上传到虚拟机
scp pause-3.7.tar sunweisheng@192.168.0.50:~/
````

## 在master节点上导入tar文件

````shell
#导入registry.k8s.io/pause:3.7
sudo microk8s ctr images import pause-3.7.tar

#查看导入镜像情况
sudo microk8s ctr images ls
````

## 同样处理其他核心镜像文件

````shell
#docker.io/calico/cni:v3.25.1
docker pull docker.io/calico/cni:v3.25.1

docker save -o cni-3.25.1.tar calico/cni:v3.25.1

scp cni-3.25.1.tar sunweisheng@192.168.0.50:~/

#在master上执行
sudo microk8s ctr images import cni-3.25.1.tar
````

````shell
# 检查microk8s状态，不加60秒超时就会一直等
sudo microk8s status --wait-ready -t 60

# 显示以下内容表示部署完成，核心的镜像导入之后可显示部署完成，但实际上还有很多镜像文件下载不了
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
addons:
  enabled:
    dns                  # (core) CoreDNS
    ha-cluster           # (core) Configure high availability on the current node
    helm                 # (core) Helm - the package manager for Kubernetes
    helm3                # (core) Helm 3 - the package manager for Kubernetes
  disabled:
    cert-manager         # (core) Cloud native certificate management
    cis-hardening        # (core) Apply CIS K8s hardening
    community            # (core) The community addons repository
    dashboard            # (core) The Kubernetes dashboard
    gpu                  # (core) Alias to nvidia add-on
    host-access          # (core) Allow Pods connecting to Host services smoothly
    hostpath-storage     # (core) Storage class; allocates storage from host directory
    ingress              # (core) Ingress controller for external access
    kube-ovn             # (core) An advanced network fabric for Kubernetes
    mayastor             # (core) OpenEBS MayaStor
    metallb              # (core) Loadbalancer for your Kubernetes cluster
    metrics-server       # (core) K8s Metrics Server for API access to service metrics
    minio                # (core) MinIO object storage
    nvidia               # (core) NVIDIA hardware (GPU and network) support
    observability        # (core) A lightweight observability stack for logs, traces and metrics
    prometheus           # (core) Prometheus operator for monitoring and logging
    rbac                 # (core) Role-Based Access Control for authorisation
    registry             # (core) Private image registry exposed on localhost:32000
    rook-ceph            # (core) Distributed Ceph storage using Rook
    storage              # (core) Alias to hostpath-storage add-on, deprecated
````

## 在Master节点上开启需要的插件

开启dashboard、ingress、hostpath-storage、metrics-server、registry

````shell
#开启 dashboard插件，metrics-server会一同开启
sudo microk8s enable dashboard

sudo microk8s enable ingress

sudo microk8s enable hostpath-storage

sudo microk8s enable registry
````

## 修复不能启动的Pod

这个时候很多pod因为不能下载镜像还处在不正常的状态：

````shell
#查看所有Pod状态
sudo microk8s kubectl get pods -A

#显示内容
NAMESPACE            NAME                                         READY   STATUS              RESTARTS   AGE
container-registry   registry-5776c58776-mtzt6                    0/1     Pending             0          114m
ingress              nginx-ingress-microk8s-controller-589zq      0/1     ContainerCreating   0          118m
kube-system          calico-kube-controllers-796fb75cc-97tvl      0/1     ContainerCreating   0          5h14m
kube-system          calico-node-d8bvd                            0/1     ImagePullBackOff    0          5h14m
kube-system          coredns-5986966c54-fmg4w                     0/1     ContainerCreating   0          5h14m
kube-system          dashboard-metrics-scraper-795895d745-8bhsp   0/1     ContainerCreating   0          123m
kube-system          hostpath-provisioner-7c8bdf94b8-nmvtn        0/1     ContainerCreating   0          118m
kube-system          kubernetes-dashboard-6796797fb5-r7f4k        0/1     ContainerCreating   0          123m
kube-system          metrics-server-7cff7889bd-hnrz5              0/1     ContainerCreating   0          123m

````

需要一个一个处理，在可以下载镜像的电脑上把缺少的镜像都下载后通过tar导入到Master节点上（或需要这些镜像的服务器上），跳过从DockerHub上下载的过程。

````shell
#查看不能下载的镜像名称
sudo microk8s kubectl describe pod calico-node-d8bvd -n kube-system 
#在能下载镜像的机器上下载镜像
docker pull docker.io/calico/node:v3.25.1
#导出tar文件
docker save -o calico-node-3.25.1.tar calico/node:v3.25.1
#上传tar文件
scp calico-node-3.25.1.tar sunweisheng@192.168.0.50:~/
#在master上导入tar镜像文件
sudo microk8s ctr images import calico-node-3.25.1.tar

#查看不能下载的镜像名称
sudo microk8s kubectl describe pod coredns-5986966c54-fmg4w  -n kube-system
#在能下载镜像的机器上下载镜像
docker pull docker.io/coredns/coredns:1.10.1
#导出tar文件
docker save -o coredns-1.10.1.tar coredns/coredns:1.10.1
#上传tar文件
scp coredns-1.10.1.tar sunweisheng@192.168.0.50:~/
#在master上导入tar镜像文件
sudo microk8s ctr images import coredns-1.10.1.tar

#查看不能下载的镜像名称
sudo microk8s kubectl describe pod calico-kube-controllers-796fb75cc-97tvl  -n kube-system
#在能下载镜像的机器上下载镜像
docker pull docker.io/calico/kube-controllers:v3.25.1
#导出tar文件
docker save -o calico-kube-controllers-v3.25.1.tar calico/kube-controllers:v3.25.1
#上传tar文件
scp calico-kube-controllers-v3.25.1.tar sunweisheng@192.168.0.50:~/
#在master上导入tar镜像文件
sudo microk8s ctr images import calico-kube-controllers-v3.25.1.tar
 
#查看不能下载的镜像名称
sudo microk8s kubectl describe pod kubernetes-dashboard-6796797fb5-r7f4k  -n kube-system
#在能下载镜像的机器上下载镜像
docker pull kubernetesui/dashboard:v2.7.0
#导出tar文件
docker save -o kubernetesui-dashboard-v2.7.0.tar kubernetesui/dashboard:v2.7.0
#上传tar文件
scp kubernetesui-dashboard-v2.7.0.tar sunweisheng@192.168.0.50:~/
#在master上导入tar镜像文件
sudo microk8s ctr images import kubernetesui-dashboard-v2.7.0.tar
 
#查看不能下载的镜像名称
sudo microk8s kubectl describe pod metrics-server-7cff7889bd-hnrz5  -n kube-system
#在能下载镜像的机器上下载镜像
docker pull registry.k8s.io/metrics-server/metrics-server:v0.6.3
#导出tar文件
docker save -o metrics-server-v0.6.3.tar registry.k8s.io/metrics-server/metrics-server:v0.6.3
#上传tar文件
scp metrics-server-v0.6.3.tar sunweisheng@192.168.0.50:~/
#在master上导入tar镜像文件
sudo microk8s ctr images import metrics-server-v0.6.3.tar

#查看不能下载的镜像名称
sudo microk8s kubectl describe pod dashboard-metrics-scraper-795895d745-8bhsp  -n kube-system
#在能下载镜像的机器上下载镜像
docker pull kubernetesui/metrics-scraper:v1.0.8
#导出tar文件
docker save -o metrics-scraper-v1.0.8.tar kubernetesui/metrics-scraper:v1.0.8
#上传tar文件
scp metrics-scraper-v1.0.8.tar sunweisheng@192.168.0.50:~/
#在master上导入tar镜像文件
sudo microk8s ctr images import metrics-scraper-v1.0.8.tar

#查看不能下载的镜像名称
sudo microk8s kubectl describe pod hostpath-provisioner-7c8bdf94b8-nmvtn  -n kube-system
#在能下载镜像的机器上下载镜像
docker pull cdkbot/hostpath-provisioner:1.5.0
#导出tar文件
docker save -o hostpath-provisioner-1.5.0.tar cdkbot/hostpath-provisioner:1.5.0
#上传tar文件
scp hostpath-provisioner-1.5.0.tar sunweisheng@192.168.0.50:~/
#在master上导入tar镜像文件
sudo microk8s ctr images import hostpath-provisioner-1.5.0.tar

#查看不能下载的镜像名称
sudo microk8s kubectl describe pod hostpath-provisioner-k8s-master-kc9cw  -n kube-system
#在能下载镜像的机器上下载镜像
docker pull busybox:1.28.4
#导出tar文件
docker save -o busybox-1.28.4.tar busybox:1.28.4
#上传tar文件
scp busybox-1.28.4.tar sunweisheng@192.168.0.50:~/
#在master上导入tar镜像文件
sudo microk8s ctr images import busybox-1.28.4.tar

#查看不能下载的镜像名称
sudo microk8s kubectl describe pod nginx-ingress-microk8s-controller-589zq  -n ingress
#在能下载镜像的机器上下载镜像
docker pull registry.k8s.io/ingress-nginx/controller:v1.11.5
#导出tar文件
docker save -o ingress-nginx-v1.11.5.tar registry.k8s.io/ingress-nginx/controller:v1.11.5
#上传tar文件
scp ingress-nginx-v1.11.5.tar sunweisheng@192.168.0.50:~/
#在master上导入tar镜像文件
sudo microk8s ctr images import ingress-nginx-v1.11.5.tar

#查看不能下载的镜像名称
sudo microk8s kubectl describe pod registry-5776c58776-mtzt6  -n container-registry
#在能下载镜像的机器上下载镜像
docker pull registry:2.8.1
#导出tar文件
docker save -o registry-2.8.1.tar registry:2.8.1
#上传tar文件
scp registry-2.8.1.tar sunweisheng@192.168.0.50:~/
#在master上导入tar镜像文件
sudo microk8s ctr images import registry-2.8.1.tar
````

都导入完之后查看结果：

````shell
sudo microk8s kubectl get pods -A

#全部Pod都启动起来了
NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
container-registry   registry-5776c58776-mtzt6                    1/1     Running   0          163m
ingress              nginx-ingress-microk8s-controller-589zq      1/1     Running   0          167m
kube-system          calico-kube-controllers-796fb75cc-97tvl      1/1     Running   0          6h3m
kube-system          calico-node-d8bvd                            1/1     Running   0          6h3m
kube-system          coredns-5986966c54-fmg4w                     1/1     Running   0          6h3m
kube-system          dashboard-metrics-scraper-795895d745-8bhsp   1/1     Running   0          172m
kube-system          hostpath-provisioner-7c8bdf94b8-nmvtn        1/1     Running   0          166m
kube-system          kubernetes-dashboard-6796797fb5-r7f4k        1/1     Running   0          172m
kube-system          metrics-server-7cff7889bd-hnrz5              1/1     Running   0          172m
````

## 将另一个虚拟机加入MicroK8s集群成为Node（worker）节点

microk8s的主节点本身就是一个worker节点，现在Master节点上运行命令显示出加入集群的命令和密钥，密钥只使用一次每加入一台服务器都需要重新执行命令查看密钥。

````shell
# 在Master上执行
sudo microk8s add-node

# 显示
From the node you wish to join to this cluster, run the following:
microk8s join 192.168.0.50:25000/ae8d39d69921a96f34226e3e5ad8ea9b/5a4decc79ab3

Use the '--worker' flag to join a node as a worker not running the control plane, eg:
microk8s join 192.168.0.50:25000/ae8d39d69921a96f34226e3e5ad8ea9b/5a4decc79ab3 --worker

If the node you are adding is not reachable through the default interface you can use one of the following:
microk8s join 192.168.0.50:25000/ae8d39d69921a96f34226e3e5ad8ea9b/5a4decc79ab3
microk8s join 2408:8207:1950:d771:5054:ff:fec4:e61e:25000/ae8d39d69921a96f34226e3e5ad8ea9b/5a4decc79ab3
````

## 将Node节点加入MicroK8s集群

在Node节点服务器上执行如下命令：

````shell
#安装
sudo snap install microk8s --classic --channel=1.30/stable

#安装后pod依旧不能启动
sudo microk8s kubectl get pods -A

NAMESPACE     NAME                                      READY   STATUS     RESTARTS   AGE
kube-system   calico-kube-controllers-796fb75cc-lmkf9   0/1     Pending    0          74s
kube-system   calico-node-67dcd                         0/1     Init:0/2   0          75s
kube-system   coredns-5986966c54-26zjz                  0/1     Pending    0          74s

#加入MicroK8s集群，注意要增加--worker参数
sudo microk8s join 192.168.0.50:25000/ae8d39d69921a96f34226e3e5ad8ea9b/5a4decc79ab3 --worker

#显示
Contacting cluster at 192.168.0.50

The node has joined the cluster and will appear in the nodes list in a few seconds.

This worker node gets automatically configured with the API server endpoints.
If the API servers are behind a loadbalancer please set the '--refresh-interval' to '0s' in:
    /var/snap/microk8s/current/args/apiserver-proxy
and replace the API server endpoints with the one provided by the loadbalancer in:
    /var/snap/microk8s/current/args/traefik/provider.yaml
````

````shell
#在Master上查看所有node节点下的pod运行情况
sudo microk8s kubectl get pods -A -o wide

NAMESPACE            NAME                                         READY   STATUS     RESTARTS      AGE     IP             NODE         NOMINATED NODE   READINESS GATES
container-registry   registry-5776c58776-mtzt6                    1/1     Running    2             5h38m   10.1.235.216   k8s-master   <none>           <none>
ingress              nginx-ingress-microk8s-controller-589zq      1/1     Running    2             5h43m   10.1.235.211   k8s-master   <none>           <none>
kube-system          calico-kube-controllers-796fb75cc-97tvl      1/1     Running    2             8h      10.1.235.214   k8s-master   <none>           <none>
kube-system          calico-node-6v6g7                            1/1     Running    0             98m     192.168.0.50   k8s-master   <none>           <none>
kube-system          calico-node-qvm65                            0/1     Init:0/2   0             97m     192.168.0.51   k8s-node1    <none>           <none>
kube-system          coredns-5986966c54-fmg4w                     1/1     Running    2             8h      10.1.235.213   k8s-master   <none>           <none>
kube-system          dashboard-metrics-scraper-795895d745-8bhsp   1/1     Running    2             5h47m   10.1.235.212   k8s-master   <none>           <none>
kube-system          hostpath-provisioner-7c8bdf94b8-nmvtn        1/1     Running    3 (98m ago)   5h42m   10.1.235.217   k8s-master   <none>           <none>
kube-system          kubernetes-dashboard-6796797fb5-r7f4k        1/1     Running    2             5h47m   10.1.235.215   k8s-master   <none>           <none>
kube-system          metrics-server-7cff7889bd-hnrz5              1/1     Running    2             5h47m   10.1.235.210   k8s-master   <none>           <none>

#创建SSH登录密钥方便从Master节点链接Node节点传输文件
ssh-keygen -t rsa -b 4096 -C "sunweisheng"
ssh-copy-id sunweisheng@192.168.0.51

#从Master节点传输Node节点必要的镜像
scp pause-3.7.tar sunweisheng@192.168.0.51:~/
scp cni-3.25.1.tar sunweisheng@192.168.0.51:~/
scp calico-node-3.25.1.tar sunweisheng@192.168.0.51:~/
scp ingress-nginx-v1.11.5.tar sunweisheng@192.168.0.51:~/
````

````shell
#在Node节点上导入镜像
sudo microk8s ctr images import pause-3.7.tar 
sudo microk8s ctr images import cni-3.25.1.tar
sudo microk8s ctr images import calico-node-3.25.1.tar
sudo microk8s ctr images import ingress-nginx-v1.11.5.tar
````

````shell
#在Master下查看所有node节点下的pod运行情况
sudo microk8s kubectl get pods -A -o wide

#显示
NAMESPACE            NAME                                         READY   STATUS    RESTARTS      AGE    IP             NODE         NOMINATED NODE   READINESS GATES
container-registry   registry-5776c58776-mtzt6                    1/1     Running   3 (32m ago)   9h     10.1.235.219   k8s-master   <none>           <none>
ingress              nginx-ingress-microk8s-controller-589zq      1/1     Running   4 (31m ago)   9h     10.1.235.225   k8s-master   <none>           <none>
ingress              nginx-ingress-microk8s-controller-rptkg      1/1     Running   0             11m    10.1.36.65     k8s-node1    <none>           <none>
kube-system          calico-kube-controllers-796fb75cc-97tvl      1/1     Running   3 (32m ago)   12h    10.1.235.223   k8s-master   <none>           <none>
kube-system          calico-node-6v6g7                            1/1     Running   1 (32m ago)   5h7m   192.168.0.50   k8s-master   <none>           <none>
kube-system          calico-node-qvm65                            1/1     Running   0             5h6m   192.168.0.51   k8s-node1    <none>           <none>
kube-system          coredns-5986966c54-fmg4w                     1/1     Running   3 (32m ago)   12h    10.1.235.224   k8s-master   <none>           <none>
kube-system          dashboard-metrics-scraper-795895d745-8bhsp   1/1     Running   3 (32m ago)   9h     10.1.235.222   k8s-master   <none>           <none>
kube-system          hostpath-provisioner-7c8bdf94b8-nmvtn        1/1     Running   4 (32m ago)   9h     10.1.235.221   k8s-master   <none>           <none>
kube-system          kubernetes-dashboard-6796797fb5-r7f4k        1/1     Running   3 (32m ago)   9h     10.1.235.220   k8s-master   <none>           <none>
kube-system          metrics-server-7cff7889bd-hnrz5              1/1     Running   3 (32m ago)   9h     10.1.235.218   k8s-master   <none>           <none>

#查看节点信息
microk8s kubectl get nodes

#显示
NAME         STATUS   ROLES    AGE     VERSION
k8s-master   Ready    <none>   12h     v1.30.11
k8s-node1    Ready    <none>   5h12m   v1.30.11
````

## 配置和查看MicroK8s的仪表盘（Dashboard）

在Master节点执行：

````shell
#创建登录Dashboard的token
microk8s kubectl create token kubernetes-dashboard -n kube-system

#显示登录Token
eyJhbGciOiJSUzI1NiIsImtpZCI6Ilk3UHlScTZvc0J1SEh0VU85Rjh2VUs5QTdaQkdtVTZ0aFFzYUJ2aDMtb0kifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjIl0sImV4cCI6MTc0NzUwMzk1OSwiaWF0IjoxNzQ3NTAwMzU5LCJpc3MiOiJodHRwczovL2t1YmVybmV0ZXMuZGVmYXVsdC5zdmMiLCJqdGkiOiJjNmQxZGUwNS0yM2EyLTQ1MTEtOTYzNy1jZTAxODNhODdkMmEiLCJrdWJlcm5ldGVzLmlvIjp7Im5hbWVzcGFjZSI6Imt1YmUtc3lzdGVtIiwic2VydmljZWFjY291bnQiOnsibmFtZSI6Imt1YmVybmV0ZXMtZGFzaGJvYXJkIiwidWlkIjoiY2MxZWU5MDctM2MwNS00ZTg4LThkMDgtOGVkMDcxNWQwMGNmIn19LCJuYmYiOjE3NDc1MDAzNTksInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTprdWJlcm5ldGVzLWRhc2hib2FyZCJ9.pZGx7aG7cYIKhZjw2rboVHASMZ3gP2Z3THxSjhfTcgdTklhhkSF38x2qwhvTdQxeRwP2FhdHDjWaQpZTClzyqgmluAORQME48sbbeyMCIS4Auu9jgAZNC6BW4e-AKsvDqhu1XB4vVVVfi6BQfbpSKRUp_04R5pSa21GmzB9r_bKlqljWAv-hhogEi3cSlhGyetiT3DFeEBY5t91nwomTVILThZObdFF5xSmrEOLOX109Fm0fEJDR-M3Qzrj9exduNPMrpEZnnT9OckNevjpGVK6LDqiZM6_GeV-MGt3ss9Av4ljqHjDrnfv_RAglInYG5SZVY9b_E-jDwMyLETv-Hw

#直接暴露服务（不安全，仅测试用）
microk8s kubectl patch svc kubernetes-dashboard -n kube-system -p '{"spec": {"type": "NodePort"}}'

microk8s kubectl get svc -n kube-system | grep dashboard

#显示
kubernetes-dashboard        NodePort    10.152.183.140   <none>        443:32137/TCP            9h

#用浏览器直接访问，输入上面的Token即可登录成功
https://192.168.0.50:32137/#/login
````

## 开启nvidia插件

在有GPU的服务器上执行（该服务器操作系统是Ubuntu桌面版本）：

````shell
# 检查显卡驱动确认安装了NVIDIA显卡和驱动
nvidia-smi

#显示显卡信息和驱动信息
Sun May 18 13:54:27 2025       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.144.03             Driver Version: 550.144.03     CUDA Version: 12.4     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 3090        Off |   00000000:65:00.0 Off |                  N/A |
|  0%   29C    P8             22W /  390W |      26MiB /  24576MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
                                                                                         
+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A      1634      G   /usr/lib/xorg/Xorg                              9MiB |
|    0   N/A  N/A      1756      G   /usr/bin/gnome-shell                            8MiB |
+-----------------------------------------------------------------------------------------+
# 安装和加入集群
sudo snap install microk8s --classic --channel=1.30/stable

# 更换了加入集群的Token
sudo microk8s join 192.168.0.50:25000/01219019dbcf7c61dc5cd098499fb7a7/5a4decc79ab3 --worker
````

````shell
# 从Master节点复制镜像文件到GPU服务器（在Master节点执行）
scp *.tar admin@192.168.0.15:~/tar/
````

````shell
# 在GPU服务器上执行导入镜像
sudo microk8s ctr images import pause-3.7.tar 
sudo microk8s ctr images import cni-3.25.1.tar
sudo microk8s ctr images import calico-node-3.25.1.tar
sudo microk8s ctr images import ingress-nginx-v1.11.5.tar

# 将当前用户加入microk8s组并刷新组权限
sudo usermod -a -G microk8s $USER
newgrp microk8s
````

````shell
# 在Master节点执行启动nvidia插件
microk8s enable nvidia

#显示
Infer repository core for addon nvidia
Addon core/dns is already enabled
Addon core/helm3 is already enabled
Checking if NVIDIA driver is already installed
"nvidia" has been added to your repositories
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "nvidia" chart repository
Update Complete. ⎈Happy Helming!⎈
Deploy NVIDIA GPU operator
Using operator GPU driver
NAME: gpu-operator
LAST DEPLOYED: Sun May 18 13:05:06 2025
NAMESPACE: gpu-operator-resources
STATUS: deployed
REVISION: 1
TEST SUITE: None
Deployed NVIDIA GPU operator

# 检查部署情况
sudo microk8s kubectl get pods -A -o wide

# 又是一大堆镜像下载不了(部分显示信息)
NAMESPACE                NAME                                                          READY   STATUS             RESTARTS       AGE     IP             NODE         NOMINATED NODE   READINESS GATES
container-registry       registry-5776c58776-mtzt6                                     1/1     Running            5 (171m ago)   29h     10.1.235.236   k8s-master   <none>           <none>
gpu-operator-resources   gpu-operator-56b6cf869d-k7q9d                                 1/1     Running            0              87s     10.1.28.195    ai-home      <none>           <none>
gpu-operator-resources   gpu-operator-node-feature-discovery-gc-5fcdc8894b-z4hlm       0/1     ErrImagePull       0              87s     10.1.28.194    ai-home      <none>           <none>
gpu-operator-resources   gpu-operator-node-feature-discovery-master-7d84b856d7-kvxnr   0/1     ImagePullBackOff   0              87s     10.1.28.196    ai-home      <none>           <none>
gpu-operator-resources   gpu-operator-node-feature-discovery-worker-2t9wv              0/1     ErrImagePull       0              87s     10.1.28.197    ai-home      <none>           <none>
gpu-operator-resources   gpu-operator-node-feature-discovery-worker-nwwh8              0/1     ErrImagePull       0              87s     10.1.235.242   k8s-master   <none>           <none>
gpu-operator-resources   gpu-operator-node-feature-discovery-worker-rmqfz              0/1     ErrImagePull       0              87s     10.1.36.67     k8s-node1    <none>           <none>
````

````shell
# 查看镜像名称（registry.k8s.io/nfd/node-feature-discovery:v0.14.2）
sudo microk8s kubectl describe pod gpu-operator-node-feature-discovery-master-7d84b856d7-kvxnr  -n gpu-operator-resources
#在能下载镜像的机器上下载镜像
docker pull registry.k8s.io/nfd/node-feature-discovery:v0.14.2
#推送镜像到Master节点的registry私有仓库
docker tag registry.k8s.io/nfd/node-feature-discovery:v0.14.2 192.168.0.50:32000/registry.k8s.io/nfd/node-feature-discovery:v0.14.2
docker push 192.168.0.50:32000/registry.k8s.io/nfd/node-feature-discovery:v0.14.2
#Master本地是可以使用registry插件仓库，拉去镜像并修改tag
sudo microk8s ctr images pull localhost:32000/registry.k8s.io/nfd/node-feature-discovery:v0.14.2
sudo microk8s ctr images tag localhost:32000/registry.k8s.io/nfd/node-feature-discovery:v0.14.2 registry.k8s.io/nfd/node-feature-discovery:v0.14.2

#在能下载镜像的机器上导出tar文件
docker save -o node-feature-discovery-v0.14.2.tar registry.k8s.io/nfd/node-feature-discovery:v0.14.2
#上传tar文件(传到Node和GPU两个服务器上)
scp node-feature-discovery-v0.14.2.tar admin@192.168.0.15:~/tar/
scp node-feature-discovery-v0.14.2.tar sunweisheng@192.168.0.51:~/
#在GPU服务器（15）和Node节点（51）上都执行导入tar镜像文件命令
sudo microk8s ctr images import node-feature-discovery-v0.14.2.tar
````

因为nvidia-cuda-validator-vp2qj（Pod）错误，导致其他Pod无法启动，查日志发现因驱动有冲突（cgroup v2 与 NVIDIA 设备权限管理不兼容，导致设备节点无法正确创建或访问，造成了Pod错误），修改GPU Operator 的 ClusterPolicy，禁用符号链接验证。
````shell
# 在Master节点上执行
sudo microk8s kubectl edit clusterpolicy -n gpu-operator-resources
````
````yaml
validator:
    #添加内容开始
    driver:
      env:
      - name: DISABLE_DEV_CHAR_SYMLINK_CREATION
        value: "true"
    #添加内容结束
    image: gpu-operator-validator
    imagePullPolicy: IfNotPresent
    plugin:
      env:
      - name: WITH_WORKLOAD
        value: "false"
    repository: nvcr.io/nvidia/cloud-native
    version: v23.9.1
````
保存之后Pod自动重启，然后查看结果：

````shell
# 在Master上执行（看来nvidia的镜像下载站点是可以正常使用的）
sudo microk8s kubectl get pods -A -o wide

NAMESPACE                NAME                                                          READY   STATUS      RESTARTS        AGE     IP             NODE         NOMINATED NODE   READINESS GATES
container-registry       registry-5776c58776-mtzt6                                     1/1     Running     5 (4h49m ago)   31h     10.1.235.236   k8s-master   <none>           <none>
gpu-operator-resources   gpu-feature-discovery-n6w4n                                   1/1     Running     0               58m     10.1.28.204    ai-home      <none>           <none>
gpu-operator-resources   gpu-operator-56b6cf869d-k7q9d                                 1/1     Running     0               118m    10.1.28.195    ai-home      <none>           <none>
gpu-operator-resources   gpu-operator-node-feature-discovery-gc-5fcdc8894b-z4hlm       1/1     Running     0               118m    10.1.28.194    ai-home      <none>           <none>
gpu-operator-resources   gpu-operator-node-feature-discovery-master-7d84b856d7-kvxnr   1/1     Running     0               118m    10.1.28.196    ai-home      <none>           <none>
gpu-operator-resources   gpu-operator-node-feature-discovery-worker-2t9wv              1/1     Running     0               118m    10.1.28.197    ai-home      <none>           <none>
gpu-operator-resources   gpu-operator-node-feature-discovery-worker-nwwh8              1/1     Running     0               118m    10.1.235.242   k8s-master   <none>           <none>
gpu-operator-resources   gpu-operator-node-feature-discovery-worker-rmqfz              1/1     Running     0               118m    10.1.36.67     k8s-node1    <none>           <none>
gpu-operator-resources   nvidia-container-toolkit-daemonset-vr9ct                      1/1     Running     0               8m26s   10.1.28.200    ai-home      <none>           <none>
gpu-operator-resources   nvidia-cuda-validator-vp2qj                                   0/1     Completed   0               8m12s   10.1.28.205    ai-home      <none>           <none>
gpu-operator-resources   nvidia-dcgm-exporter-9nq7c                                    1/1     Running     0               58m     10.1.28.202    ai-home      <none>           <none>
gpu-operator-resources   nvidia-device-plugin-daemonset-n5fht                          1/1     Running     0               58m     10.1.28.201    ai-home      <none>           <none>
gpu-operator-resources   nvidia-operator-validator-27gpv                               1/1     Running     0               8m27s   10.1.28.203    ai-home      <none>           <none>
ingress                  nginx-ingress-microk8s-controller-589zq                       1/1     Running     6 (4h49m ago)   31h     10.1.235.234   k8s-master   <none>           <none>
ingress                  nginx-ingress-microk8s-controller-j2jdw                       1/1     Running     0               5h50m   10.1.28.193    ai-home      <none>           <none>
ingress                  nginx-ingress-microk8s-controller-knpqw                       1/1     Running     1 (4h49m ago)   5h59m   10.1.36.66     k8s-node1    <none>           <none>
kube-system              calico-kube-controllers-796fb75cc-97tvl                       1/1     Running     5 (4h49m ago)   34h     10.1.235.239   k8s-master   <none>           <none>
kube-system              calico-node-6v6g7                                             1/1     Running     3 (4h49m ago)   27h     192.168.0.50   k8s-master   <none>           <none>
kube-system              calico-node-79c68                                             1/1     Running     1 (4h49m ago)   5h59m   192.168.0.51   k8s-node1    <none>           <none>
kube-system              calico-node-rs8sf                                             1/1     Running     0               6h7m    192.168.0.15   ai-home      <none>           <none>
kube-system              coredns-5986966c54-fmg4w                                      1/1     Running     5 (4h49m ago)   34h     10.1.235.235   k8s-master   <none>           <none>
kube-system              dashboard-metrics-scraper-795895d745-8bhsp                    1/1     Running     5 (4h49m ago)   31h     10.1.235.237   k8s-master   <none>           <none>
kube-system              hostpath-provisioner-7c8bdf94b8-nmvtn                         1/1     Running     6 (4h49m ago)   31h     10.1.235.241   k8s-master   <none>           <none>
kube-system              kubernetes-dashboard-6796797fb5-r7f4k                         1/1     Running     5 (4h49m ago)   31h     10.1.235.238   k8s-master   <none>           <none>
kube-system              metrics-server-7cff7889bd-hnrz5                               1/1     Running     5 (4h49m ago)   31h     10.1.235.240   k8s-master   <none>           <none>
````

## 通过Registry仓库插件简化部署工作

在上面的操作步骤中因为下载不了镜像导致一台服务器加入节点或启动插件都需要传输大量镜像文件，以下内容是通过Registry插件简化镜像分发工作，另外该Registry仓库也是以后部署自己开发镜像的存储仓库，可以直接在Deployment中使用简化集群内部署。
如果要让集群内所有资源都能访问则Registry插件需要改为nodePort访问方式。
````shell
# 在Master上查看私有Docker仓库设置是否正确
sudo microk8s kubectl edit svc registry -n container-registry
````
将 spec.ports 中的 port: 32000 下方添加 nodePort: 32000，并修改 type: ClusterIP 为 type: NodePort：
````yaml
spec:
  ports:
  - port: 32000
    targetPort: 5000
    nodePort: 32000  # 手动指定节点端口
  type: NodePort     # 修改为 NodePort
````
保存后，其他机器可通过 <主节点IP>:32000 访问。

可以下载镜像的电脑安装的是Docker环境，所以要修改daemon.json设置，支持非HTTPS场景，在文件中增加配置内：

````json
{
  "insecure-registries": [
    "http://192.168.0.50:32000"
  ]
}
````
重启Docker。

在Docker环境中向私有仓库推送上述过程中下载的镜像：

````shell
docker tag calico/cni:v3.25.1 192.168.0.50:32000/calico/cni:v3.25.1
docker push 192.168.0.50:32000/calico/cni:v3.25.1

docker tag registry.k8s.io/pause:3.7 192.168.0.50:32000/registry.k8s.io/pause:3.7
docker push 192.168.0.50:32000/registry.k8s.io/pause:3.7

docker tag registry.k8s.io/ingress-nginx/controller:v1.11.5 192.168.0.50:32000/registry.k8s.io/ingress-nginx/controller:v1.11.5
docker push 192.168.0.50:32000/registry.k8s.io/ingress-nginx/controller:v1.11.5

docker tag cdkbot/hostpath-provisioner:1.5.0 192.168.0.50:32000/cdkbot/hostpath-provisioner:1.5.0
docker push 192.168.0.50:32000/cdkbot/hostpath-provisioner:1.5.0

docker tag registry:2.8.1 192.168.0.50:32000/registry:2.8.1
docker push 192.168.0.50:32000/registry:2.8.1

docker tag calico/kube-controllers:v3.25.1 192.168.0.50:32000/calico/kube-controllers:v3.25.1
docker push 192.168.0.50:32000/calico/kube-controllers:v3.25.1

docker tag calico/node:v3.25.1 192.168.0.50:32000/calico/node:v3.25.1
docker push 192.168.0.50:32000/calico/node:v3.25.1

docker tag registry.k8s.io/metrics-server/metrics-server:v0.6.3 192.168.0.50:32000/registry.k8s.io/metrics-server/metrics-server:v0.6.3
docker push 192.168.0.50:32000/registry.k8s.io/metrics-server/metrics-server:v0.6.3

docker tag coredns/coredns:1.10.1 192.168.0.50:32000/coredns/coredns:1.10.1
docker push 192.168.0.50:32000/coredns/coredns:1.10.1

docker tag kubernetesui/dashboard:v2.7.0 192.168.0.50:32000/kubernetesui/dashboard:v2.7.0
docker push 192.168.0.50:32000/kubernetesui/dashboard:v2.7.0

docker tag kubernetesui/metrics-scraper:v1.0.8 192.168.0.50:32000/kubernetesui/metrics-scraper:v1.0.8
docker push 192.168.0.50:32000/kubernetesui/metrics-scraper:v1.0.8

docker tag busybox:1.28.4 192.168.0.50:32000/busybox:1.28.4
docker push 192.168.0.50:32000/busybox:1.28.4

docker tag registry.k8s.io/nfd/node-feature-discovery:v0.14.2 192.168.0.50:32000/registry.k8s.io/nfd/node-feature-discovery:v0.14.2
docker push 192.168.0.50:32000/registry.k8s.io/nfd/node-feature-discovery:v0.14.2
#查看结果
curl http://192.168.0.50:32000/v2/_catalog
````
显示结果：
````json
{
  "repositories": [
    "busybox",
    "calico/cni",
    "calico/kube-controllers",
    "calico/node",
    "cdkbot/hostpath-provisioner",
    "coredns/coredns",
    "kubernetesui/dashboard",
    "kubernetesui/metrics-scraper",
    "registry",
    "registry.k8s.io/ingress-nginx/controller",
    "registry.k8s.io/metrics-server/metrics-server",
    "registry.k8s.io/nfd/node-feature-discovery",
    "registry.k8s.io/pause"
  ]
}
````
在Docker环境中测试一下拉取镜像

````shell
docker pull 192.168.0.50:32000/busybox:1.28.4
````

新增加一台Node服务器（服务器名称：db-home）用于测试使用Registry插件的情况。
在db-home服务器上安装MicroK8s并加入现有集群中：

````shell
#在db-home上执行安装
sudo snap install microk8s --classic --channel=1.30/stable

#安装后查看pod不能启动情况
sudo microk8s kubectl get pods -A

NAMESPACE     NAME                                      READY   STATUS     RESTARTS   AGE
kube-system   calico-kube-controllers-796fb75cc-szn46   0/1     Pending    0          3s
kube-system   calico-node-5mtmf                         0/1     Init:0/2   0          4s
kube-system   coredns-5986966c54-k9sj9                  0/1     Pending    0          3s
````

在db-home上增加私有仓库设置，因为MicroK8s默认是Containerd环境，所以设置方法与Docker环境设置方式差异很大。
在db-home上创建如下目录和文件内容：

````shell
# 配置所有未单独配置的Registry的默认镜像仓库。
# 默认的镜像仓库镜像（Registry Mirror），使得所有拉取镜像的请求都会被重定向到指定的镜像仓库。
# 行为：
# 当拉取镜像（如 XXX.XXX/library/nginx）时，Containerd 先尝试 192.168.0.50:32000。
# 如果 192.168.0.50:32000 没有该镜像，仍然会回退到原始 registry（如 XXX.XXX/library/nginx）。
sudo mkdir  /var/snap/microk8s/current/args/certs.d/_default/
sudo nano /var/snap/microk8s/current/args/certs.d/_default/hosts.toml
````
添加如下信息
````toml
# 非单独配置仓库先看看私有仓库汇总有没有镜像如果没有再从目标仓库拉取
[host."http://192.168.0.50:32000"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
````

修改docker.io仓库的配置内容增加新的镜像仓库：
````shell
sudo nano /var/snap/microk8s/current/args/certs.d/docker.io/hosts.toml
````
添加如下信息并放在[host...]内容的最上面，因为上下顺序代表了优先级:
````toml
server = "https://docker.io"
# 从docker.io下载镜像时先看看私有仓库汇总有没有镜像如果没有再从Docker仓库拉取
# 添加的内容开始
[host."http://192.168.0.50:32000"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
# 添加内容结束
  
[host."https://registry-1.docker.io"]
  capabilities = ["pull", "resolve"]
````
单独配置192.168.0.50:32000仓库信息
````shell
sudo mkdir /var/snap/microk8s/current/args/certs.d/192.168.0.50:32000/
sudo nano /var/snap/microk8s/current/args/certs.d/192.168.0.50:32000/hosts.toml
````
添加如下信息：
````toml
# 对私有仓库单独配置不走_default配置，其实走_default效果都一样，这样就是显式声明而已
server = "http://192.168.0.50:32000"

[host."http://192.168.0.50:32000"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
````

都创建完文件之后目录树如下：
````shell
sudo tree /var/snap/microk8s/current/args/certs.d/
/var/snap/microk8s/current/args/certs.d/
├── 192.168.0.50:32000
│   └── hosts.toml
├── _default
│   └── hosts.toml
├── docker.io
│   └── hosts.toml
└── localhost:32000
    └── hosts.toml
````

````shell
# 完成配置之后重启 microk8s
sudo microk8s stop
sudo microk8s start
````

在db-home上从私有仓库手工拉取registry.k8s.io/pause:3.7镜像(没有pause镜像，Pod无法启动，业务镜像也不会被拉取(因为Pod的"基础设施"未就绪))：
这是因为以上配置是为cri准备的（[plugins."io.containerd.grpc.v1.cri".registry]），因此只适用于cri客户端如crictl、kubectl ,如果使用ctr测试，可以使用--hosts-dir指定配置文件，否则会报HTTPS错误。
````shell
sudo microk8s ctr images pull --hosts-dir "/var/snap/microk8s/current/args/certs.d/" 192.168.0.50:32000/registry.k8s.io/pause:3.7
sudo sudo microk8s ctr images tag 192.168.0.50:32000/registry.k8s.io/pause:3.7 registry.k8s.io/pause:3.7
````

其他镜像自动从私有库拉取，查看自动拉取镜像效果：
````shell
# 在db-home上执行
sudo microk8s kubectl get pods -A

#显示
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-796fb75cc-szn46   1/1     Running   0          5h3m
kube-system   calico-node-5mtmf                         1/1     Running   0          5h3m
kube-system   coredns-5986966c54-k9sj9                  1/1     Running   0          5h3m
````

加入MicroK8s集群：
````shell
# 在db-home上执行，加入MicroK8s集群，注意要增加--worker参数
sudo microk8s join 192.168.0.50:25000/fcea486e14652f2bd260d3a8dcba736c/5a4decc79ab3 --worker
````

有两个镜像还是获取不到，原因还未查明，先解决问题吧。
````shell
# nvidia插件镜像自动拉取不了手工解决
sudo microk8s ctr images pull --hosts-dir "/var/snap/microk8s/current/args/certs.d/" 192.168.0.50:32000/registry.k8s.io/nfd/node-feature-discovery:v0.14.2
sudo microk8s ctr images tag 192.168.0.50:32000/registry.k8s.io/nfd/node-feature-discovery:v0.14.2 registry.k8s.io/nfd/node-feature-discovery:v0.14.2
# ingress插件镜像自动拉取不了手工解决
sudo microk8s ctr images pull --hosts-dir "/var/snap/microk8s/current/args/certs.d/" 192.168.0.50:32000/registry.k8s.io/ingress-nginx/controller:v1.11.5
sudo microk8s ctr images tag 192.168.0.50:32000/registry.k8s.io/ingress-nginx/controller:v1.11.5 registry.k8s.io/ingress-nginx/controller:v1.11.5

# 在Master上执行
sudo microk8s kubectl get pods -A -o wide

# 显示
NAMESPACE                NAME                                                          READY   STATUS      RESTARTS       AGE     IP             NODE         NOMINATED NODE   READINESS GATES
container-registry       registry-5776c58776-mtzt6                                     1/1     Running     6 (6h2m ago)   2d10h   10.1.235.248   k8s-master   <none>           <none>
gpu-operator-resources   gpu-feature-discovery-n6w4n                                   1/1     Running     1 (6h2m ago)   28h     10.1.28.213    ai-home      <none>           <none>
gpu-operator-resources   gpu-operator-56b6cf869d-k7q9d                                 1/1     Running     2 (6h2m ago)   29h     10.1.28.206    ai-home      <none>           <none>
gpu-operator-resources   gpu-operator-node-feature-discovery-gc-5fcdc8894b-z4hlm       1/1     Running     1 (6h2m ago)   29h     10.1.28.211    ai-home      <none>           <none>
gpu-operator-resources   gpu-operator-node-feature-discovery-master-7d84b856d7-kvxnr   1/1     Running     1 (6h2m ago)   29h     10.1.28.208    ai-home      <none>           <none>
gpu-operator-resources   gpu-operator-node-feature-discovery-worker-2t9wv              1/1     Running     1 (6h2m ago)   29h     10.1.28.209    ai-home      <none>           <none>
gpu-operator-resources   gpu-operator-node-feature-discovery-worker-n9wpj              1/1     Running     0              26m     10.1.200.66    db-home      <none>           <none>
gpu-operator-resources   gpu-operator-node-feature-discovery-worker-nwwh8              1/1     Running     1 (6h2m ago)   29h     10.1.235.246   k8s-master   <none>           <none>
gpu-operator-resources   gpu-operator-node-feature-discovery-worker-rmqfz              1/1     Running     1 (6h2m ago)   29h     10.1.36.69     k8s-node1    <none>           <none>
gpu-operator-resources   nvidia-container-toolkit-daemonset-vr9ct                      1/1     Running     1 (6h2m ago)   27h     10.1.28.210    ai-home      <none>           <none>
gpu-operator-resources   nvidia-cuda-validator-s8r2r                                   0/1     Completed   0              6h1m    10.1.28.216    ai-home      <none>           <none>
gpu-operator-resources   nvidia-dcgm-exporter-9nq7c                                    1/1     Running     1 (6h2m ago)   28h     10.1.28.212    ai-home      <none>           <none>
gpu-operator-resources   nvidia-device-plugin-daemonset-n5fht                          1/1     Running     1 (6h2m ago)   28h     10.1.28.214    ai-home      <none>           <none>
gpu-operator-resources   nvidia-operator-validator-27gpv                               1/1     Running     1 (6h2m ago)   27h     10.1.28.215    ai-home      <none>           <none>
ingress                  nginx-ingress-microk8s-controller-589zq                       1/1     Running     7 (6h2m ago)   2d10h   10.1.235.245   k8s-master   <none>           <none>
ingress                  nginx-ingress-microk8s-controller-j2jdw                       1/1     Running     1 (6h2m ago)   33h     10.1.28.207    ai-home      <none>           <none>
ingress                  nginx-ingress-microk8s-controller-knpqw                       1/1     Running     2 (6h2m ago)   33h     10.1.36.68     k8s-node1    <none>           <none>
ingress                  nginx-ingress-microk8s-controller-ttg7h                       1/1     Running     0              26m     10.1.200.65    db-home      <none>           <none>
kube-system              calico-kube-controllers-796fb75cc-97tvl                       1/1     Running     6 (6h2m ago)   2d14h   10.1.235.247   k8s-master   <none>           <none>
kube-system              calico-node-6v6g7                                             1/1     Running     4 (6h2m ago)   2d6h    192.168.0.50   k8s-master   <none>           <none>
kube-system              calico-node-79c68                                             1/1     Running     2 (6h2m ago)   33h     192.168.0.51   k8s-node1    <none>           <none>
kube-system              calico-node-rs8sf                                             1/1     Running     1 (6h2m ago)   33h     192.168.0.15   ai-home      <none>           <none>
kube-system              calico-node-shvzd                                             1/1     Running     0              26m     192.168.0.10   db-home      <none>           <none>
kube-system              coredns-5986966c54-fmg4w                                      1/1     Running     6 (6h2m ago)   2d14h   10.1.235.249   k8s-master   <none>           <none>
kube-system              dashboard-metrics-scraper-795895d745-8bhsp                    1/1     Running     6 (6h2m ago)   2d11h   10.1.235.243   k8s-master   <none>           <none>
kube-system              hostpath-provisioner-7c8bdf94b8-nmvtn                         1/1     Running     7 (6h2m ago)   2d10h   10.1.235.250   k8s-master   <none>           <none>
kube-system              kubernetes-dashboard-6796797fb5-r7f4k                         1/1     Running     6 (6h2m ago)   2d11h   10.1.235.244   k8s-master   <none>           <none>
kube-system              metrics-server-7cff7889bd-hnrz5                               1/1     Running     6 (6h2m ago)   2d11h   10.1.235.251   k8s-master   <none>           <none>
````
至此MicroK8s的全部部署配置内容完成。