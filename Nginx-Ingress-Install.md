# 在K8S集群中使用安装NGINX Ingress V1.7

[官方资料](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/)

## 安装NGINX Ingress

```shell
mkdir ~/nginx-ingress
cd ~/nginx-ingress

#创建Ingress控制器的命名空间和服务账号
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/common/ns-and-sa.yaml

#创建给服务账号绑定集群角色
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/rbac/rbac.yaml

#TLS证书（仅用于测试）
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/common/default-server-secret.yaml

#改上述文件的命名空间和名字可以用于其他命名空间，但也只是用于测试

#MapConfig
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/common/nginx-config.yaml

#VirtualServer and VirtualServerRoute and TransportServer资源
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/common/ts-definition.yaml
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/common/vs-definition.yaml
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/common/vsr-definition.yaml

#GlobalConfiguration
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/common/gc-definition.yaml
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/common/global-configuration.yaml

#使用DaemonSet方式部署到集群
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/daemon-set/nginx-ingress.yaml

#创建服务(NodePort)
wget https://raw.githubusercontent.com/nginxinc/kubernetes-ingress/release-1.7/deployments/service/nodeport.yaml

#安装
kubectl apply -f .

#查看结果
kubectl get pods --namespace=nginx-ingress
```

## 创建TLS证书

```shell
mkdir dashboard-ingress-secret
cd dashboard-ingress-secret

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
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/secure-backends: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"

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
```

```shell
#添加路由信息
kubectl apply -f dashboard-ingress.yaml

#查看路由信息详情
kubectl describe ingress dashboard-ingress -n kubernetes-dashboard

#在宿主机上实现端口转发
ssh -CNg -L 443:192.168.122.4:443 root@127.0.0.1

```

## 卸载

```shell
kubectl delete namespace nginx-ingress
kubectl delete clusterrole nginx-ingress
kubectl delete clusterrolebinding nginx-ingress
```
