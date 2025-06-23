# 单独安装和使用Kubectl工具管理多个集群

## 环境介绍
- 使用Kubectl的环境是一台Mac Air M2处理器
- 集群1是Mac上用Multipass搭建的Microk8s集群（一台Master一台Node）
- 集群2是在另一台Mac Mini上用Virtual box搭建的原生kubernetes集群（一台Master和两台Node）

## 操作步骤

### 在Mac Air上执行操作

下载指定版本比如v1.30.13
````shell
# Mac Air下载kubectl工具
curl -LO https://dl.k8s.io/v1.30.13/bin/darwin/arm64/kubectl

# 赋予执行权限
chmod +x kubectl

sudo mv kubectl /usr/local/bin/

# 查看是否安装成功
kubectl version --client

# 显示
Client Version: v1.30.13
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3

# 在Mac上创建配置目录
mkdir -p $HOME/.kube
````

### 在Multipassd搭建的Microk8s集群上执行操作

````shell
# 创建目录
mkdir -p $HOME/.kube

# 将ubuntu用户添加到microk8s组里,Multipassd默认用户名:ubuntu，密码:ubuntu
sudo usermod -a -G microk8s ubuntu

# 将~/.kube的所有者改成ubuntu
sudo chown -R ubuntu ~/.kube

# SSH登录账户用户应用microk8s组权限
newgrp microk8s

# 复制配置信息到指定目录，这样执行就不用sudo了，比如microk8s kubectl get pod -A
sudo microk8s config > ~/.kube/config_microk8s

# 浏览集群配置信息并复制
cat  ~/.kube/config_microk8s

# 在Mac Air里新建config_microk8s文件并粘贴信息
vi ~/.kube/config_microk8s
````

### 在原生kubernetes集群上执行操作

````shell
# 浏览原生K8S集群配置信息并复制
cat ~/.kube/config

# 在Mac Air里新建config_k8s文件并粘贴信息
vi ~/.kube/config_k8s
````

### 在Mac Air上执行操作

````shell
vi ~/.zshrc 

# Kubernetes 支持通过环境变量 KUBECONFIG 指定多个配置文件，用冒号 : 分隔
export KUBECONFIG=~/.kube/config_k8s:~/.kube/config_microk8s

source ~/.zshrc

# 显示可链接的K8S集群
kubectl config get-contexts

# 显示内容
kubectl config get-contexts
CURRENT   NAME                          CLUSTER            AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes         kubernetes-admin   
          microk8s                      microk8s-cluster   admin   

# 切换到microk8s集群
kubectl config use-context microk8s

kubectl get pod -A

# 显示
NAMESPACE            NAME                                         READY   STATUS    RESTARTS       AGE
container-registry   registry-5776c58776-kshpk                    1/1     Running   3 (39m ago)    30d
ingress              nginx-ingress-microk8s-controller-8p4wb      1/1     Running   13 (39m ago)   30d
ingress              nginx-ingress-microk8s-controller-hsj49      1/1     Running   2 (39m ago)    30d
kube-system          calico-kube-controllers-796fb75cc-x69xv      1/1     Running   3 (39m ago)    30d
kube-system          calico-node-f8hcc                            1/1     Running   13 (39m ago)   30d
kube-system          calico-node-l5njq                            1/1     Running   2 (39m ago)    30d
kube-system          coredns-5986966c54-75dxb                     1/1     Running   3 (39m ago)    30d
kube-system          dashboard-metrics-scraper-795895d745-ld4s9   1/1     Running   13 (39m ago)   30d
kube-system          hostpath-provisioner-7c8bdf94b8-xqqx2        1/1     Running   3 (39m ago)    30d
kube-system          kubernetes-dashboard-6796797fb5-n4xbg        1/1     Running   13 (39m ago)   30d
kube-system          metrics-server-7c5c4b56bd-xw49g              1/1     Running   13 (39m ago)   30d

kubectl config use-context kubernetes-admin@kubernetes

kubectl get pod -A
# 显示
NAMESPACE      NAME                                 READY   STATUS    RESTARTS       AGE
kube-flannel   kube-flannel-ds-bkqp8                1/1     Running   23 (44m ago)   4d15h
kube-flannel   kube-flannel-ds-k24br                1/1     Running   27 (44m ago)   4d15h
kube-flannel   kube-flannel-ds-rx7fh                1/1     Running   24 (44m ago)   4d15h
kube-system    coredns-55cb58b774-4hk6h             1/1     Running   4 (44m ago)    4d16h
kube-system    coredns-55cb58b774-fdwph             1/1     Running   4 (44m ago)    4d16h
kube-system    etcd-mac-master                      1/1     Running   6 (44m ago)    4d16h
kube-system    kube-apiserver-mac-master            1/1     Running   6 (44m ago)    4d16h
kube-system    kube-controller-manager-mac-master   1/1     Running   6 (44m ago)    4d16h
kube-system    kube-proxy-8n7sk                     1/1     Running   2 (44m ago)    3d
kube-system    kube-proxy-sbdxl                     1/1     Running   2 (44m ago)    3d
kube-system    kube-proxy-zvbhm                     1/1     Running   2 (44m ago)    3d
kube-system    kube-scheduler-mac-master            1/1     Running   6 (44m ago)    4d16h

# 将集群名称改成更好理解的名字
kubectl config rename-context kubernetes-admin@kubernetes mac-kubernetes
kubectl config rename-context microk8s multipass-microk8s

kubectl config get-contexts

# 显示
CURRENT   NAME                 CLUSTER            AUTHINFO           NAMESPACE
*         mac-kubernetes       kubernetes         kubernetes-admin   
          multipass-microk8s   microk8s-cluster   admin 
````