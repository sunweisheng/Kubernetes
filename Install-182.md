# 安装Kubernetes V1.18.2

## 准备虚拟机环境

[用KVM组建NAT网络虚拟机集群](https://github.com/sunweisheng/kvm/blob/master/Create-NAT-Cluster.md)

## 设置系统

```shell
#关闭防火墙（所有节点）
systemctl disable firewalld
systemctl stop firewalld

#关闭SELINUX（所有节点）
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

#设置iptables（所有节点）
cd /etc/sysctl.d/

cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system

#关闭swap分区（所有节点）
vi /etc/fstab

# /dev/mapper/centos-swap swap                    swap    defaults        0 0

swapoff -a
```

## 安装Docker

```shell
#在所有节点上安装
cd /etc/yum.repos.d/

yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

yum -y install docker-ce

#开机启动Docker
systemctl start docker
systemctl enable docker
```

## 安装kubernetes

```shell
#在所有节点上安装
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

#开机启动kubelet
systemctl enable kubelet
```

## 初始化K8S的Master节点

```shell
#初始化（要增加registry.aliyuncs.com/google_containers才能下载google的镜像文件，初始化时要下载镜像文件要耐心的等）
kubeadm init  --kubernetes-version=v1.18.2 --image-repository=registry.aliyuncs.com/google_containers  --apiserver-advertise-address=192.168.122.3 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.1.0.0/16

#按照成功安装后的提示执行
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

* apiserver-advertise-address：是Master节点上的管理API地址。
* pod-network-cidr：是pod的网络IP段，用于k8s的pod通信。
* service-cidr：是服务的虚IP网络段，用于使用服务资源。

## Node节点加入K8S集群

```shell
#从Master安装成功提示中找到下面命令，在所有Node节点上执行
kubeadm join 192.168.122.3:6443 --token t0ie01.1ya3iwrt0uofx45p \
    --discovery-token-ca-cert-hash sha256:14e36c483bb8180e740e953c8141d4e747a19e325a09007c675ba8fc64a95e27
```

## 在Master节点上安装flannel网络

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

## 查看安装结果

```shell
#查看所有pod
kubectl get pod -A

#查看所有NODE
kubectl get node -A
```

![Alt text](http://static.bluersw.com/images/Kubernetes/Kubernetes-Install-182-01.png)  
![Alt text](http://static.bluersw.com/images/Kubernetes/Kubernetes-Install-182-02.png)  
