# 在K8S集群中使用安装NGINX Ingress V1.7

[官方资料](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/)

## 安装NGINX Ingress

```shell
mkdir ~/nginx-ingress
cd ~/nginx-ingress

#创建Ingress控制器的命名空间和服务账号
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/common/ns-and-sa.yaml
kubectl apply -f ns-and-sa.yaml

#创建给服务账号绑定集群角色
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/rbac/rbac.yaml
kubectl apply -f rbac.yaml

#TLS证书（仅用于测试）
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/common/default-server-secret.yaml
#改上述文件的命名空间和名字可以用于其他命名空间，但也只是用于测试
kubectl apply -f default-server-secret.yaml

#MapConfig
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/common/nginx-config.yaml
kubectl apply -f nginx-config.yaml

#VirtualServer and VirtualServerRoute and TransportServer资源
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/common/ts-definition.yaml
kubectl apply -f ts-definition.yaml
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/common/vs-definition.yaml
kubectl apply -f vs-definition.yaml
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/common/vsr-definition.yaml
kubectl apply -f vsr-definition.yaml

#GlobalConfiguration
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/common/gc-definition.yaml
kubectl apply -f gc-definition.yaml
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/common/global-configuration.yaml
kubectl apply -f global-configuration.yaml

#使用DaemonSet方式部署到集群
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/daemon-set/nginx-ingress.yaml
kubectl apply -f nginx-ingress.yaml

#创建服务(NodePort)
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/service/nodeport.yaml
kubectl apply -f nodeport.yaml

#查看结果
kubectl get pods --namespace=nginx-ingress
```

## 创建dashboard-default-backend.yaml

```sehll
vi dashboard-default-backend.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: default-http-backend
  labels:
    app: default-http-backend
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  selector:
    matchLabels:
      app: default-http-backend
  template:
    metadata:
      labels:
        app: default-http-backend
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: default-http-backend
        # Any image is permissible as long as:
        # 1. It serves a 404 page at /
        # 2. It serves 200 on a /healthz endpoint
        image: registry.aliyuncs.com/google_containers/defaultbackend:1.4
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
---

apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
  namespace: kubernetes-dashboard
  labels:
    app: default-http-backend
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: default-http-backend
```





```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: default-http-backend
  labels:
    app: default-http-backend
  namespace: nginx-ingress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: default-http-backend
  template:
    metadata:
      labels:
        app: default-http-backend
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: default-http-backend
        # Any image is permissible as long as:
        # 1. It serves a 404 page at /
        # 2. It serves 200 on a /healthz endpoint
        image: registry.aliyuncs.com/google_containers/defaultbackend:1.4
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
---

apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
  namespace: nginx-ingress
  labels:
    app: default-http-backend
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: default-http-backend
```

```shell
kubectl apply -f dashboard-default-backend.yaml
```

## 创建TLS证书

PS：路由（Ingress）、服务（Service）、凭证（Secret）都要在一个命名空间下

```shell
cd ~/dashboard
mkdir dashboard-ingress-secret
cd ~/dashboard/dashboard-ingress-secret

# 创建key文件
openssl genrsa -out dashboard-ingress.key 2048

#证书请求
openssl req -days 36000 -new -out dashboard-ingress.csr -key dashboard-ingress.key -subj '/CN=*.dashboard.com'

#自签证书
openssl x509 -req -in dashboard-ingress.csr -signkey dashboard-ingress.key -out dashboard-ingress.crt

#创建kubernetes-dashboard-certs对象
kubectl create secret generic dashboard-ingress-secret --from-file=dashboard-ingress.key --from-file=dashboard-ingress.crt -n kubernetes-dashboard
```

## 创建kubernetes-dashboard的Ingress 路由配置文件

PS：路由（Ingress）、服务（Service）、凭证（Secret）都要在一个命名空间下

```shell
vi dashboard-ingress.yaml
```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.org/ssl-services: "kubernetes-dashboard"

spec:
  tls:
  - hosts:
    - dashboard.com
    - "*.dashboard.com"
    secretName: dashboard-ingress-secret
  rules:
  - host: www.dashboard.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
      - path: /healthz
          serviceName: default-http-backend
          servicePort: 80
```

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: default-backend-ingress
  namespace: nginx-ingress
  annotations:
    kubernetes.io/ingress.class: nginx

spec:
  rules:
   - http:
      paths:
      - path: /
        backend:
         serviceName: default-http-backend
         servicePort: 80
      - path: /healthz
        backend:
         serviceName: default-http-backend
         servicePort: 80
```

```shell
#添加路由信息
kubectl apply -f dashboard-ingress.yaml

#查看路由信息详情
kubectl describe ingress default-backend-ingress -n nginx-ingress

#在宿主机上实现端口转发
ssh -CNg -L 443:192.168.122.4:443 root@127.0.0.1

kubectl exec  nginx-ingress-drccd -n nginx-ingress -- cat /etc/nginx/conf.d/nginx-ingress-cafe-ingress.conf

curl --resolve cafe.example.com:443:10.244.1.12 https://cafe.example.com:443/tea --insecure

https://github.com/nginxinc/kubernetes-ingress/tree/master/examples/complete-example

```



## 卸载

```shell
kubectl delete namespace nginx-ingress
kubectl delete clusterrole nginx-ingress
kubectl delete clusterrolebinding nginx-ingress
```
