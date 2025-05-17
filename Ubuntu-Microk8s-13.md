# 在Ubuntu Server中部署MicroK8s-1.3版本

## 准备环境

两台KVM虚拟机都是Ubuntu Server版本操作系统，其中一台设为Master节点，另一台是Node节点，在两台虚拟机上都执行以下环境准备的命令：

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

## 同样处理其他镜像文件

````shell
#docker.io/calico/cni:v3.25.1
docker pull docker.io/calico/cni:v3.25.1

docker save -o cni-3.25.1.tar calico/cni:v3.25.1

scp cni-3.25.1.tar sunweisheng@192.168.0.50:~/

#在master上执行
sudo microk8s ctr images import cni-3.25.1.tar
````

````shell
#检查microk8s状态
sudo microk8s status --wait-ready -t 60

#显示以下内容表示部署完成
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

## 在Master节点上开启相关插件

开启dashboard、ingress、hostpath-storage、metrics-server、registry

````shell
#开启 dashboard插件，metrics-server会一同开启
sudo microk8s enable dashboard

sudo microk8s enable ingress

sudo microk8s enable hostpath-storage

#开启Docker私有库插件(本来想用私有库共享给其他节点使用单没有成功，容器的ctr images pull始终用https访问不能用http访问，最后放弃了)
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

需要一个一个处理：

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

## 设置私有镜像仓库并推送镜像（非必须）
经过后续试验发现push是可以的，但不管怎么设置pull拉取镜像时无论是Docker还是MicroK8s的containers都只访问https不能访问http所以这个私有仓库试验失败了。
在Master上查看私有Docker仓库设置是否正确：
````shell
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

因为不是HTTPS所以需要在可以下载镜像的电脑上修改daemon.json文件，在文件中增加：

````json
{
  "insecure-registries": [
    "192.168.0.50:32000"
  ]
}
````
重启Docker。

向私有仓库推送镜像：

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

#查看结果
curl http://192.168.0.50:32000/v2/_catalog
````
预期结果：
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
    "registry.k8s.io/pause"
  ]
}
````

## 显示加入microk8s集群的信息

microk8s的主节点本身就是一个worker节点。

````shell
sudo microk8s add-node

#显示
From the node you wish to join to this cluster, run the following:
microk8s join 192.168.0.50:25000/ae8d39d69921a96f34226e3e5ad8ea9b/5a4decc79ab3

Use the '--worker' flag to join a node as a worker not running the control plane, eg:
microk8s join 192.168.0.50:25000/ae8d39d69921a96f34226e3e5ad8ea9b/5a4decc79ab3 --worker

If the node you are adding is not reachable through the default interface you can use one of the following:
microk8s join 192.168.0.50:25000/ae8d39d69921a96f34226e3e5ad8ea9b/5a4decc79ab3
microk8s join 2408:8207:1950:d771:5054:ff:fec4:e61e:25000/ae8d39d69921a96f34226e3e5ad8ea9b/5a4decc79ab3
````

## 将Node节点加入MicroK8s集群

````shell
#安装
sudo snap install microk8s --classic --channel=1.30/stable

#安装后pod依旧不能启动
sudo microk8s kubectl get pods -A

NAMESPACE     NAME                                      READY   STATUS     RESTARTS   AGE
kube-system   calico-kube-controllers-796fb75cc-lmkf9   0/1     Pending    0          74s
kube-system   calico-node-67dcd                         0/1     Init:0/2   0          75s
kube-system   coredns-5986966c54-26zjz                  0/1     Pending    0          74s

#加入MicroK8s集群
sudo microk8s join 192.168.0.50:25000/ae8d39d69921a96f34226e3e5ad8ea9b/5a4decc79ab3

#查看所有node节点下的pod运行情况
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

#方便从Master节点链接Node节点传输文件
ssh-keygen -t rsa -b 4096 -C "sunweisheng"
ssh-copy-id sunweisheng@192.168.0.51

#从Master节点传输Node节点必要的镜像
scp pause-3.7.tar sunweisheng@192.168.0.51:~/
scp cni-3.25.1.tar sunweisheng@192.168.0.51:~/
scp calico-node-3.25.1.tar sunweisheng@192.168.0.51:~/
scp ingress-nginx-v1.11.5.tar sunweisheng@192.168.0.51:~/

#在Node节点上导入镜像
sudo microk8s ctr images import pause-3.7.tar 
sudo microk8s ctr images import cni-3.25.1.tar
sudo microk8s ctr images import calico-node-3.25.1.tar
sudo microk8s ctr images import ingress-nginx-v1.11.5.tar

#查看所有node节点下的pod运行情况
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

````shell
#创建登录Dashboard的token
microk8s kubectl create token kubernetes-dashboard -n kube-system

#显示
eyJhbGciOiJSUzI1NiIsImtpZCI6Ilk3UHlScTZvc0J1SEh0VU85Rjh2VUs5QTdaQkdtVTZ0aFFzYUJ2aDMtb0kifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjIl0sImV4cCI6MTc0NzUwMzk1OSwiaWF0IjoxNzQ3NTAwMzU5LCJpc3MiOiJodHRwczovL2t1YmVybmV0ZXMuZGVmYXVsdC5zdmMiLCJqdGkiOiJjNmQxZGUwNS0yM2EyLTQ1MTEtOTYzNy1jZTAxODNhODdkMmEiLCJrdWJlcm5ldGVzLmlvIjp7Im5hbWVzcGFjZSI6Imt1YmUtc3lzdGVtIiwic2VydmljZWFjY291bnQiOnsibmFtZSI6Imt1YmVybmV0ZXMtZGFzaGJvYXJkIiwidWlkIjoiY2MxZWU5MDctM2MwNS00ZTg4LThkMDgtOGVkMDcxNWQwMGNmIn19LCJuYmYiOjE3NDc1MDAzNTksInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTprdWJlcm5ldGVzLWRhc2hib2FyZCJ9.pZGx7aG7cYIKhZjw2rboVHASMZ3gP2Z3THxSjhfTcgdTklhhkSF38x2qwhvTdQxeRwP2FhdHDjWaQpZTClzyqgmluAORQME48sbbeyMCIS4Auu9jgAZNC6BW4e-AKsvDqhu1XB4vVVVfi6BQfbpSKRUp_04R5pSa21GmzB9r_bKlqljWAv-hhogEi3cSlhGyetiT3DFeEBY5t91nwomTVILThZObdFF5xSmrEOLOX109Fm0fEJDR-M3Qzrj9exduNPMrpEZnnT9OckNevjpGVK6LDqiZM6_GeV-MGt3ss9Av4ljqHjDrnfv_RAglInYG5SZVY9b_E-jDwMyLETv-Hw

#直接暴露服务（不安全，仅测试用）
microk8s kubectl patch svc kubernetes-dashboard -n kube-system -p '{"spec": {"type": "NodePort"}}'

microk8s kubectl get svc -n kube-system | grep dashboard

#显示
kubernetes-dashboard        NodePort    10.152.183.140   <none>        443:32137/TCP            9h

#用浏览器直接访问，输入上面的Token即可登录成功
https://192.168.0.50:32137/#/login
````



