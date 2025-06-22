# Ubuntu Server 安装配置Kubernetes 1.30版本

## 环境准备
虚拟机软件是KVM，使用Bridge网络，3台安装了Ubuntu Server 24.04.2版本的虚拟机，其中一台Master节点两台Node节点，
IP地址是：192.168.0.30（Master）、192.168.0.31（Node）、192.168.0.32（Node）。

## 在所有节点上安排软件和进行配置
````shell
sudo apt update && sudo apt upgrade -y
sudo apt install htop wget  telnet net-tools ntp curl vim git -y

# 对齐时间否则会出问题
ntpq -p
````

修改 GRUB 内核启动参数：
- 向 Linux 内核的启动参数中添加两个选项：
    - swapaccount=1：启用对 swap 使用情况的监控与统计（用于 kubelet 检测 swap 是否开启）
    - cgroup_enable=memory：启用 memory cgroup，允许 Kubernetes 对容器进行内存限制管理
从 Kubernetes v1.22 开始，默认使用 systemd 作为 Cgroup 驱动，并且需要 memory 子系统启用
````shell
# 续改之前默认是GRUB_CMDLINE_LINUX="quiet splash"
sudo sed -i 's/GRUB_CMDLINE_LINUX="\(.*\)"/GRUB_CMDLINE_LINUX="\1 swapaccount=1 cgroup_enable=memory"/' /etc/default/grub

# 应用上面修改的内核参数到GRUB配置文件中，使重启后依然生效。
sudo update-grub
````

Swap分区
````shell
# 禁用所有 Swap 分区（临时）
sudo swapoff -a
# 注释掉 fstab 中的 Swap 挂载点（永久）
sudo sed -i '/swap/s/^/#/' /etc/fstab
````

安排容器运行时（containerd）
````shell
sudo apt install -y containerd

# 创建 containerd 配置目录
sudo mkdir -p /etc/containerd

# 使用 containerd config default 生成一个默认的配置文件内容，然后通过 tee 命令将这些内容写入到 /etc/containerd/config.toml
containerd config default | sudo tee /etc/containerd/config.toml

# 修改 containerd 的 CRI 插件配置，启用 SystemdCgroup ，将SystemdCgroup = false 替换为 SystemdCgroup = true，因为从Kubernetes v1.22 开始，默认使用systemd作为Cgroup驱动
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
````

设置containerd的代理
````shell
sudo mkdir -p /etc/systemd/system/containerd.service.d
sudo nano /etc/systemd/system/containerd.service.d/http-proxy.conf
````

如果是Docker则（与containerd二选其一）
````shell
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo nano /etc/systemd/system/docker.service.d/http-proxy.conf
````

````
[Service]
Environment="HTTP_PROXY=http://192.168.0.10:7897"
Environment="HTTPS_PROXY=http://192.168.0.10:7897"
Environment="NO_PROXY=localhost,127.0.0.1,::1,.local,192.168.0.0/24,docker-registry.example.com,.intra"
````

完成上述配置之后重启容器运行时
````shell
# 重启 systemd 本身（仅当你修改了 systemd 核心配置）
sudo systemctl daemon-reexec
# 重载 systemd unit 文件（当你修改了服务定义文件）
sudo systemctl daemon-reload
# 重启 containerd 服务（使 config.toml 生效）
sudo systemctl restart containerd
# 设置 containerd 开机自启
sudo systemctl enable containerd
````

配置APT代理
````shell
sudo tee /etc/apt/apt.conf.d/proxy.conf <<EOF
Acquire::http::Proxy "http://192.168.0.10:7897";
Acquire::https::Proxy "http://192.168.0.10:7897";
EOF
````

配置代理在SSH环境变量下
````shell
# 编辑非登录ssh环境变量
sudo nano ~/.bashrc

# 设置代理 对wget等命令有效
export http_proxy="http://192.168.0.10:7897"
export https_proxy="http://192.168.0.10:7897"
export no_proxy="localhost,127.0.0.1,::1,.local,192.168.0.0/24,docker-registry.example.com,.intra"

# 重新加载
source ~/.bashrc
````

安装K8S1.30的软件安装源
````shell
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/  /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update

# 查看可用版本 1.30的最高版本是1.30.13-1.1
apt-cache madison kubelet
   kubelet | 1.30.13-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubelet | 1.30.12-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubelet | 1.30.11-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubelet | 1.30.10-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubelet | 1.30.9-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubelet | 1.30.8-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubelet | 1.30.7-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubelet | 1.30.6-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubelet | 1.30.5-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubelet | 1.30.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubelet | 1.30.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubelet | 1.30.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubelet | 1.30.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
   kubelet | 1.30.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.30/deb  Packages
````

安装K8S
````shell
# 在Master和Node上都执行
sudo apt install -y kubelet=1.30.13-1.1 kubeadm=1.30.13-1.1 kubectl=1.30.13-1.1

# 锁定版本防止自动升级 在Master和Node上都执行
sudo apt-mark hold kubelet kubeadm kubectl
````

安装ipvsadm和ipset
- ipvsadm:是一个用于管理 Linux 内核中的 IPVS（IP Virtual Server）  的命令行工具,IPVS 是 Linux 提供的一个高性能负载均衡机制，Kubernetes 使用它来实现 Service 的负载均衡（默认模式）。
- ipset:是一个用于管理大规模 IP 地址集合的工具，通常与 IPVS 配合使用,它可以高效地处理大量 IP 规则，提升 kube-proxy 使用 IPVS 模式时的性能

````shell
sudo apt install -y ipvsadm ipset
````

完成安装
````shell
sudo systemctl enable kubelet
sudo systemctl start kubelet
````

网络配置
````shell
# 启用系统的 IP 转发功能 ，这是 Kubernetes 和容器网络（CNI）正常工作的必要条件，net.ipv4.ip_forward=1这是一个 Linux 内核参数，用于控制是否允许系统进行 IP 转发（即让一个网络接口接收到的数据包转发到另一个接口）。在 Kubernetes 中，Pod 之间的通信依赖 IP 转发，CNI 插件（如 Flannel、Calico）需要它来实现跨节点网络互通
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf

# modprobe 是一个 Linux 命令，用于动态加载或卸载内核模块（kernel modules）,它允许你在系统运行时启用某些内核功能.
# 加载 Linux 内核模块 br_netfilter，用于启用桥接网络与 iptables 的集成功能
sudo modprobe br_netfilter

# 重启有效
sudo tee /etc/modules-load.d/br_netfilter.conf <<EOF
br_netfilter
EOF

# 启用 Linux 内核中桥接网络（bridge）与 iptables 的集成规则 ，确保 Kubernetes 和 CNI 插件（如 Flannel、Calico）在网络转发和策略控制方面正常工作
# net.bridge.bridge-nf-call-iptables = 1：允许桥接流量经过 iptables 规则处理
# net.bridge.bridge-nf-call-ip6tables = 1：允许桥接流量经过 ip6tables（IPv6）规则处理
# 使用如下技术时，设置是必要的，Flannel 的 VXLAN 模式、Docker 或 containerd 使用桥接网络、kube-proxy 使用 iptables 或 IPVS 模式实现 Service 转发
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-flannel.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# 这条命令会加载所有位于 /etc/sysctl.d/ 目录下的 .conf 文件，并应用它们到当前系统的内核参数中
sudo sysctl --system
````

## 在Master节点上配置

### 初始化K8S集群

````shell
# 在主节点上执行,10.244.0.0/16是CNI网络组件Flannel的默认网络配置，如果这里改了会导致Flannel不能启动
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
````

初始化完成之后显示：
````shell
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.30:6443 --token af9w3f.806oswyo0jflo701 \
	--discovery-token-ca-cert-hash sha256:dbfa6e85ec7b6e9c23e3b45b366180786eca9952d2bc87281e23bafe04bd0582 
````

执行：
````shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
````
注意：不要执行export KUBECONFIG=/etc/kubernetes/admin.conf这句，否则不是没权限访问API就是访问的地址不对，因为不是root用户。

### 配置Flannel组件

Kubernetes 使用CNI插件管理Pod网络，如 Flannel、Calico，二者选择其一进行安装。
````shell
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

kubectl  apply -f kube-flannel.yml
````

### kube-proxy启用IPVS模式
````shell
# 找到 mode: ""，改为 mode: "ipvs"
kubectl edit configmap -n kube-system kube-proxy

# 重启kube-proxy Pod
kubectl delete pod -n kube-system -l k8s-app=kube-proxy
````

## 在Node节点上执行

在所有Node上执行:
````shell
sudo kubeadm join 192.168.0.30:6443 --token af9w3f.806oswyo0jflo701 \
	--discovery-token-ca-cert-hash sha256:dbfa6e85ec7b6e9c23e3b45b366180786eca9952d2bc87281e23bafe04bd0582
````

## 在Master上查看结果

验证结果：
````shell
kubectl get nodes
NAME         STATUS   ROLES           AGE   VERSION
mac-master   Ready    control-plane   45h   v1.30.13
mac-node1    Ready    <none>          45h   v1.30.13
mac-node2    Ready    <none>          45h   v1.30.13

kubectl get pods -A -o wide
NAMESPACE      NAME                                 READY   STATUS    RESTARTS        AGE     IP             NODE         NOMINATED NODE   READINESS GATES
kube-flannel   kube-flannel-ds-bkqp8                1/1     Running   22 (153m ago)   45h     192.168.0.32   mac-node2    <none>           <none>
kube-flannel   kube-flannel-ds-k24br                1/1     Running   26 (153m ago)   45h     192.168.0.31   mac-node1    <none>           <none>
kube-flannel   kube-flannel-ds-rx7fh                1/1     Running   23 (153m ago)   45h     192.168.0.30   mac-master   <none>           <none>
kube-system    coredns-55cb58b774-4hk6h             1/1     Running   3 (153m ago)    45h     10.244.0.8     mac-master   <none>           <none>
kube-system    coredns-55cb58b774-fdwph             1/1     Running   3 (153m ago)    45h     10.244.0.9     mac-master   <none>           <none>
kube-system    etcd-mac-master                      1/1     Running   5 (153m ago)    45h     192.168.0.30   mac-master   <none>           <none>
kube-system    kube-apiserver-mac-master            1/1     Running   5 (153m ago)    45h     192.168.0.30   mac-master   <none>           <none>
kube-system    kube-controller-manager-mac-master   1/1     Running   5 (153m ago)    45h     192.168.0.30   mac-master   <none>           <none>
kube-system    kube-proxy-8n7sk                     1/1     Running   1 (153m ago)    6h24m   192.168.0.30   mac-master   <none>           <none>
kube-system    kube-proxy-sbdxl                     1/1     Running   1 (153m ago)    6h24m   192.168.0.32   mac-node2    <none>           <none>
kube-system    kube-proxy-zvbhm                     1/1     Running   1 (153m ago)    6h24m   192.168.0.31   mac-node1    <none>           <none>
kube-system    kube-scheduler-mac-master            1/1     Running   5 (153m ago)    45h     192.168.0.30   mac-master   <none>           <none>
````


