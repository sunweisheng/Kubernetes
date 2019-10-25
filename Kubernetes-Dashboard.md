# 部署 Kubernetes Dashboard

## 在Node1上下载Docker镜像

因为k8s.gcr.io访问不了我们只能手动拉取：

```shell
docker pull bluersw/kubernetes-dashboard-amd64:v1.10.1 #替代k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
docker tag bluersw/kubernetes-dashboard-amd64:v1.10.1 k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
```

## 在Master上部署Dashboard

安装部署请看[安装Kubernetes V1.16.2](Install.md)

参照[官网](https://github.com/kubernetes/dashboard)在master上执行：

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

执行后看看Master将Dashboard分配给哪个Node了，执行：

```shell
kubectl get pod --namespace=kube-system -o wide | grep dashboard
```

![Alt text](http://static.bluersw.com/images/Kubernetes/Kubernetes-Dashboard-01.png)  
从上图可以看出Master分配给Node1进行部署了，其实我们只有Node1这个Node节点，Master是不负责业务负载的。

## 在Master上进行Dashboard配置

```shell
wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

编辑kubernetes-dashboard：

```shell
vi kubernetes-dashboard.yaml

#找到这部分
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort #增加NodePort后可以从Node的IP进行访问
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30005 #增加30005端口
  selector:
    k8s-app: kubernetes-dashboard
```

保存退出后执行：  

```shell
#修改配置
kubectl apply -f kubernetes-dashboard.yaml

#查看结果
kubectl get service -n kube-system | grep dashboard
kubectl get -n kube-system pods -o wide
```

我们通过Node1的IP进行访问：https://192.168.0.7:30005,打开页面后我们用Token登录。  

## 在Master上创建用户并获得Token

为用户分配权限：

```shell
#创建文件
vi dashboard-adminuser.yaml

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system

#保存退出后执行
kubectl create -f dashboard-adminuser.yaml
```

查看用户Token：

```shell
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep kubernetes-dashboard | awk '{print $1}')
```

![Alt text](http://static.bluersw.com/images/Kubernetes/Kubernetes-Dashboard-02.png)  
注意token:后面的空格也是要一起复制的（其实那不是空格），复制到页面中登录：  
![Alt text](http://static.bluersw.com/images/Kubernetes/Kubernetes-Dashboard-03.png)  
初次登录啥都没有，我需要将命名空间改成kube-system：
