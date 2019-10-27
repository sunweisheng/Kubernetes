# Kubernetes V1.16.2部署Dashboard V2.0(beta5)

## 在Master上部署Dashboard

集群安装部署请看[安装Kubernetes V1.16.2](Install.md)

```shell
kubectl get pods -A  -o wide
```

![Alt text](http://static.bluersw.com/images/Kubernetes/Kubernetes-Install-02-2.png)  

参照[官网安装说明](https://github.com/kubernetes/dashboard/blob/93474e53b9ce5a1de0acba93c5aff6295d91dd19/docs/user/installation.md)在master上执行：

```shell
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta5/aio/deploy/recommended.yaml
```

修改recommended.yaml文件内容：

```yaml
---
#增加直接访问端口
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort #增加
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30008 #增加
  selector:
    k8s-app: kubernetes-dashboard

---
#因为自动生成的证书很多浏览器无法使用，所以我们自己创建，注释掉kubernetes-dashboard-certs对象声明
#apiVersion: v1
#kind: Secret
#metadata:
#  labels:
#    k8s-app: kubernetes-dashboard
#  name: kubernetes-dashboard-certs
#  namespace: kubernetes-dashboard
#type: Opaque

---
```

创建证书：

```shell
mkdir dashboard-certs
cd dashboard-certs/

kubectl create namespace kubernetes-dashboard

# 创建key文件
openssl genrsa -out dashboard.key 2048

#证书请求
openssl req -days 36000 -new -out dashboard.csr -key dashboard.key -subj '/CN=dashboard-cert'

#自签证书
openssl x509 -req -in dashboard.csr -signkey dashboard.key -out dashboard.crt

#创建kubernetes-dashboard-certs对象
kubectl create secret generic kubernetes-dashboard-certs --from-file=dashboard.key --from-file=dashboard.crt -n kubernetes-dashboard
```

安装Dashboard：

```shell
kubectl create -f  ~/recommended.yaml

kubectl get pods -A  -o wide

kubectl get service -n kubernetes-dashboard  -o wide
```

创建dashboard管理员：

```shell
vi dashboard-admin.yaml
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: dashboard-admin
  namespace: kubernetes-dashboard
```

```shell
#保存退出后执行
kubectl create -f dashboard-admin.yaml
```

为用户分配权限：

```shell
vi dashboard-admin-bind-cluster-role.yaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin-bind-cluster-role
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: dashboard-admin
  namespace: kubernetes-dashboard
```

```shell
#保存退出后执行
kubectl create -f dashboard-admin-bind-cluster-role.yaml
```

查看用户Token：

```shell
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep dashboard-admin | awk '{print $1}')
```

访问：https://192.168.0.7:30008，谷歌浏览器不让访问，Safari浏览器可以。

----------------------------------
![Alt text](http://static.bluersw.com/images/Kubernetes/Kubernetes-Dashboard-02.png)  
注意token:后面的空格也是要一起复制的（其实那不是空格），复制到页面中登录：  
![Alt text](http://static.bluersw.com/images/Kubernetes/Kubernetes-Dashboard-03.png)  

在Node1上下载镜像文件：

```shell
docker pull bluersw/metrics-server-amd64:v0.3.6
docker tag bluersw/metrics-server-amd64:v0.3.6 k8s.gcr.io/metrics-server-amd64:v0.3.6  
```

在Master上执行安装：
在[Kubernetes Metrics Server](https://github.com/kubernetes-incubator/metrics-server)下载K8S对象文件：[1.8+](https://github.com/kubernetes-incubator/metrics-server/tree/master/deploy/1.8%2B)

```shell

mkdir metrics-server
cd metrics-server

wget https://raw.githubusercontent.com/kubernetes-incubator/metrics-server/master/deploy/1.8%2B/aggregated-metrics-reader.yaml

wget https://raw.githubusercontent.com/kubernetes-incubator/metrics-server/master/deploy/1.8%2B/auth-delegator.yaml

wget https://raw.githubusercontent.com/kubernetes-incubator/metrics-server/master/deploy/1.8%2B/auth-reader.yaml

wget https://raw.githubusercontent.com/kubernetes-incubator/metrics-server/master/deploy/1.8%2B/metrics-apiservice.yaml

wget https://raw.githubusercontent.com/kubernetes-incubator/metrics-server/master/deploy/1.8%2B/metrics-server-deployment.yaml

wget https://raw.githubusercontent.com/kubernetes-incubator/metrics-server/master/deploy/1.8%2B/metrics-server-service.yaml

wget https://raw.githubusercontent.com/kubernetes-incubator/metrics-server/master/deploy/1.8%2B/resource-reader.yaml

```

```shell
vi metrics-server-deployment.yaml
```

修改metrics-server-deployment.yaml脚本：

```yaml
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      name: metrics-server
      labels:
        k8s-app: metrics-server
    spec:
      serviceAccountName: metrics-server
      volumes:
      # mount in tmp so we can safely use from-scratch images and/or read-only containers
      - name: tmp-dir
        emptyDir: {}
      containers:
      - name: metrics-server
        image: k8s.gcr.io/metrics-server-amd64:v0.3.6
        imagePullPolicy: IfNotPresent #修改
        command: #增加
        - /metrics-server
        - --kubelet-preferred-address-types=InternalIP
        - --kubelet-insecure-tls
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp




vi /etc/kubernetes/manifests/kube-apiserver.yaml  
# - --requestheader-allowed-names=front-proxy-client
- --requestheader-allowed-names=
```

```shell
kubectl create -f ~/metrics-server

kubectl top nodes
```

安装 heapster

```shell
docker pull bluersw/heapster-influxdb-amd64:v1.5.2
docker tag bluersw/heapster-influxdb-amd64:v1.5.2 k8s.gcr.io/heapster-influxdb-amd64:v1.5.2

docker pull bluersw/heapster-amd64:v1.5.4
docker tag bluersw/heapster-amd64:v1.5.4 k8s.gcr.io/heapster-amd64:v1.5.4

docker pull bluersw/heapster-grafana-amd64:v5.0.4
docker tag bluersw/heapster-grafana-amd64:v5.0.4 k8s.gcr.io/heapster-grafana-amd64:v5.0.4
```

在[https://github.com/kubernetes-retired](https://github.com/kubernetes-retired)下载对象配置文件[influxdb](https://github.com/kubernetes-retired/heapster/tree/master/deploy/kube-config/influxdb)


apiVersion: extensions/v1beta1 改成 apiVersion: apps/v1 
还有增加selector
spec:
  selector:
    matchLabels:
     k8s-app: heapster
  replicas: 1
  