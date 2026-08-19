# 同一项目的多环境 Helm Values 与构建结果指南

## 1. 这份文档解决什么问题

同一个 `K8S-Deploying-Java` 项目可以部署到虚拟机环境，也可以部署到云服务器环境。两套环境使用同一份 Java 源码、同一份 Helm Chart 和同一套 Jenkins 流水线，但访问域名和 TLS Secret 不同：

| 环境 | Ingress 域名 | TLS Secret | 是否修改项目 Chart |
| --- | --- | --- | --- |
| 虚拟机 | `app.k8s.lab` | `k8s-lab-tls` | 不修改，使用 Chart 默认值 |
| 云服务器 | `app.cloud.k8s.lab` | `k8s-cloud-lab-tls` | 不修改，通过环境 values 覆盖 |

这里的关键思想是：

> 项目代码和 Chart 描述“应用是什么”，环境 values 描述“这个应用在当前环境中使用什么域名和入口配置”。

这样做不会为了云服务器复制一份项目，也不会为了虚拟机再维护一份 Chart。每次构建时，Helm 根据同一份 Chart、当前环境 values 和本次构建参数，生成当前环境的 Kubernetes Manifest。

## 2. 先看最终结果

两套环境执行的是同一个项目构建流程：

```text
同一份 K8S-Deploying-Java 源码
        +
同一份 deploy/charts/spring-app
        +
当前环境的 values 输入
        +
本次构建产生的镜像仓库和镜像摘要
        ↓
Helm 渲染出当前环境的 Manifest
        ↓
部署到当前环境的 Kubernetes 集群
```

最终只有环境相关字段不同：

```text
虚拟机：
  ingress.host       = app.k8s.lab
  ingress.tlsSecret  = k8s-lab-tls

云服务器：
  ingress.host       = app.cloud.k8s.lab
  ingress.tlsSecret  = k8s-cloud-lab-tls
```

其他配置继续来自同一份 Chart 默认值，例如：

- `ingress.enabled: true`
- `ingress.className: traefik`
- Service 类型为 `ClusterIP`
- Service 端口为 `8080`
- 应用副本数为 `2`
- Spring Profile 为 `k8s`
- 数据库 Secret 名称为 `app-db`

## 3. Chart 默认值是共同起点

项目 Chart 中的 [`deploy/charts/spring-app/values.yaml`](https://github.com/sunweisheng/K8S-Deploying-Java/blob/v1.0.9/deploy/charts/spring-app/values.yaml) 保存两套环境共同使用的默认值：

```yaml
fullnameOverride: spring-app
replicaCount: 2

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: true
  className: traefik
  host: app.k8s.lab
  tlsSecret: k8s-lab-tls

runtime:
  springProfile: k8s
  existingDatabaseSecret: app-db
```

Chart 默认值不是云服务器专用配置，也不是虚拟机专用配置。它只是所有环境的共同起点。

## 4. 环境 values 只写差异

云服务器只需要改变域名和 TLS Secret，因此环境 values 可以写成：

```yaml
ingress:
  host: app.cloud.k8s.lab
  tlsSecret: k8s-cloud-lab-tls
```

它不是完整的 Chart 配置，也不需要重复写 `replicaCount`、Service 端口或数据库 Secret。没有写出的字段继续使用 Chart 默认值。

虚拟机不创建环境覆盖 ConfigMap。脚本找不到覆盖文件时会生成：

```yaml
{}
```

空对象表示“本环境没有额外覆盖”，因此虚拟机直接使用 Chart 默认的 `app.k8s.lab` 和 `k8s-lab-tls`。

## 5. 四个相似文件和对象要分开

这个机制中会出现多个名称相似的文件或对象：

| 名称 | 所在位置 | 作用 |
| --- | --- | --- |
| `deploy-overrides-values.yaml` | 云服务器 Master 的 `$HOME/k8s-platform/` | 创建 ConfigMap 时的输入文件 |
| `ConfigMap/deploy-overrides` | Kubernetes 的 `ci` 命名空间 | 保存环境 values，供新的 Agent Pod 挂载 |
| `/etc/helm/deploy-overrides/values.yaml` | 临时 Agent Pod 的 Helm 容器 | ConfigMap 挂载后的只读文件 |
| `.jenkins-json-build/deploy-overrides-values.yaml` | 临时 Agent Pod 的共享工作区 | 脚本复制或生成的文件，供 Helm 的 `--values` 读取 |
| `deploy/charts/spring-app/values.yaml` | Git 项目仓库 | Chart 的共同默认值 |

它们的关系不是 Master 文件直接挂载到 Pod，而是：

```text
Master 普通文件
  ↓ kubectl --from-file
ConfigMap/deploy-overrides
  ↓ kubelet 挂载
/etc/helm/deploy-overrides/values.yaml
  ↓ prepare-helm-values.sh 复制
.jenkins-json-build/deploy-overrides-values.yaml
  ↓ Helm --values
Helm 渲染结果
```

云服务器 Master 上的普通文件只参与创建或更新 ConfigMap。Pod 不会直接访问 Master 的文件系统，Pod 也可能被调度到任意 Worker。

## 6. 临时 Agent Pod 中的执行顺序

Jenkins 每次需要执行构建时，会创建一个临时 Agent Pod。项目 [`ci/jenkins-agent.yaml`](https://github.com/sunweisheng/K8S-Deploying-Java/blob/v1.0.9/ci/jenkins-agent.yaml) 定义 `jnlp`、Maven、BuildKit 和 Helm 四个容器，并给 Helm 容器挂载可选的 `ConfigMap/deploy-overrides`。

一次部署阶段的实际顺序是：

```text
1. Jenkins 读取 Jenkinsfile 和 ci/jenkins-project.json
2. Kubernetes 插件根据 ci/jenkins-agent.yaml 创建临时 Agent Pod
3. Pod 就绪后，流水线把 Git 仓库 checkout 到共享工作区
4. deploy 阶段进入同一个 Pod 的 helm 容器
5. prepare-helm-values.sh 读取挂载文件，复制或生成工作区 values 文件
6. helm lint 使用这份文件检查 Chart
7. helm template 使用这份文件生成最终 Manifest
8. helm upgrade --install 使用同一份文件提交部署
9. 构建结束，临时 Agent Pod 被删除
```

项目配置中的变量如下：

```json
{
  "HELM_OVERRIDE_SOURCE_FILE": "${HELM_OVERRIDE_MOUNT_PATH}/values.yaml",
  "HELM_OVERRIDE_VALUES_FILE": ".jenkins-json-build/deploy-overrides-values.yaml",
  "HELM_OVERRIDE_PREPARE_SCRIPT": "ci/prepare-helm-values.sh"
}
```

第一条部署命令展开后相当于：

```bash
ci/prepare-helm-values.sh \
  "/etc/helm/deploy-overrides/values.yaml" \
  ".jenkins-json-build/deploy-overrides-values.yaml"
```

脚本的行为只有两种：

```text
挂载的 values.yaml 存在且非空
  → 复制到 .jenkins-json-build/deploy-overrides-values.yaml

ConfigMap 不存在，或挂载文件为空
  → 生成内容为 {} 的 .jenkins-json-build/deploy-overrides-values.yaml
```

所以 Helm 不需要知道云服务器 Master 上的路径，也不需要知道 ConfigMap 的来源。Helm 只读取 Agent 工作区内的统一目标路径。

## 7. Values 的合并顺序

本项目的配置可以按下面的顺序理解：

```text
Chart 默认 values.yaml
        ↓
环境覆盖 values 文件
        ↓
本次构建的 --set-string 参数
        ↓
Helm 渲染最终 Manifest
```

### 7.1 Chart 默认值

所有环境共同的默认值来自 Chart 的 `values.yaml`。

### 7.2 环境覆盖文件

云服务器环境 values 覆盖：

```text
ingress.host: app.cloud.k8s.lab
ingress.tlsSecret: k8s-cloud-lab-tls
```

虚拟机环境没有覆盖文件，脚本生成 `{}`，因此仍使用：

```text
ingress.host: app.k8s.lab
ingress.tlsSecret: k8s-lab-tls
```

### 7.3 本次构建的镜像参数

镜像仓库和镜像摘要由 Jenkins 在本次构建完成镜像后注入：

```text
--set-string image.repository=ghcr.io/sunweisheng/spring-app
--set-string image.digest=sha256:...
```

这两个参数位于后面，因此不能被环境 values 文件改成另一个镜像。云服务器和虚拟机可以使用不同的域名，但仍然部署本次构建产生的同一个不可变镜像摘要。

## 8. 三个 Helm 动作分别做什么

项目使用同一个 values 文件执行 `lint`、`template` 和 `upgrade`，这样检查、预览和真正部署的输入保持一致。

### 8.1 `helm lint`

`lint` 检查 Chart、模板和 values 是否可以正常解析。它不会创建或更新 Kubernetes 对象。

云服务器和虚拟机虽然 values 不同，但都必须通过各自环境的 `lint`。这样可以在真正访问集群前发现 YAML 缩进、模板字段和必填值问题。

### 8.2 `helm template`

`template` 把 Chart 和当前环境 values 渲染成普通 Kubernetes Manifest。在本项目中，它用于确认最终生成的域名、TLS Secret、镜像仓库和镜像摘要是否正确，不会直接修改集群。

云服务器的 Ingress 渲染结果应包含：

```yaml
spec:
  tls:
    - hosts:
        - app.cloud.k8s.lab
      secretName: k8s-cloud-lab-tls
  rules:
    - host: app.cloud.k8s.lab
```

虚拟机的渲染结果应包含：

```yaml
spec:
  tls:
    - hosts:
        - app.k8s.lab
      secretName: k8s-lab-tls
  rules:
    - host: app.k8s.lab
```

### 8.3 `helm upgrade --install`

这是实际提交部署的动作：

- Release 不存在时安装；
- Release 已存在时升级；
- Ingress、Service、Deployment 和 Chart 管理的 ConfigMap 都按本次渲染结果进行创建或更新；
- 配置没有变化的对象保持不变；
- Deployment 的镜像摘要变化时，Deployment 会触发新的 ReplicaSet 和 Pod。

`ConfigMap/deploy-overrides` 不在 Spring Boot Chart 中，所以不由这个 Helm Release 更新。它只是 Helm values 的输入来源。

## 9. 同一项目生成不同环境结果的示例

假设同一次构建产生镜像：

```text
ghcr.io/sunweisheng/spring-app@sha256:abc123...
```

### 9.1 虚拟机

输入：

```text
Chart 默认值
+ {}
+ image.repository=ghcr.io/sunweisheng/spring-app
+ image.digest=sha256:abc123...
```

渲染结果：

```text
Deployment 镜像：ghcr.io/sunweisheng/spring-app@sha256:abc123...
Ingress 域名：app.k8s.lab
TLS Secret：k8s-lab-tls
Service：ClusterIP:8080
```

### 9.2 云服务器

输入：

```text
Chart 默认值
+ ingress.host=app.cloud.k8s.lab
+ ingress.tlsSecret=k8s-cloud-lab-tls
+ image.repository=ghcr.io/sunweisheng/spring-app
+ image.digest=sha256:abc123...
```

渲染结果：

```text
Deployment 镜像：ghcr.io/sunweisheng/spring-app@sha256:abc123...
Ingress 域名：app.cloud.k8s.lab
TLS Secret：k8s-cloud-lab-tls
Service：ClusterIP:8080
```

可以看到：

- 应用代码相同；
- Chart 相同；
- 镜像摘要相同；
- Service 配置相同；
- 只有环境需要不同的 Ingress 域名和 TLS Secret 不同。

## 10. 什么会更新，什么不会更新

每次 `main` 构建都会执行 Helm 发布阶段，但实际变化取决于合并后的 values 和 Chart 模板：

| 变化 | 可能产生的结果 |
| --- | --- |
| 镜像摘要变化 | Deployment 更新，创建新的 ReplicaSet 和 Pod |
| Ingress 域名变化 | Ingress 更新，Traefik 重新读取路由 |
| TLS Secret 名称变化 | Ingress 更新，Traefik 使用新的 Secret |
| Service 端口变化 | Service 更新，应用后端连接方式改变 |
| Spring Profile 或 JVM 参数变化 | `spring-app-runtime` ConfigMap 更新；Deployment 的 checksum 注解变化后触发滚动更新 |
| 所有输入都不变 | Helm 仍执行 upgrade，但 Kubernetes 对象通常保持不变 |

Helm 提交的是“期望状态”，不是每次都删除后重建所有对象。Kubernetes 发现对象不存在或实际状态与期望状态不同，才会创建或更新对象。

## 11. 云服务器 values 的更新边界

云服务器 Master 上的文件只是创建 ConfigMap 的输入：

```bash
kubectl -n "$CI_NAMESPACE" create configmap deploy-overrides \
  --from-file=values.yaml="$HOME/k8s-platform/deploy-overrides-values.yaml" \
  --dry-run=client -o yaml | kubectl apply -f -
```

修改普通文件不会自动改变集群中的 ConfigMap。修改后必须再次执行上面的命令，之后新创建的 Agent Pod 才会使用新的 values 输入。

这个 ConfigMap 只应保存非敏感环境配置，例如域名和 TLS Secret 名称。数据库密码、TLS 私钥、GHCR Token 等内容应继续保存为 Kubernetes Secret，不写入 values 文件。

## 12. 检查每个环境的最终结果

### 12.1 检查渲染结果

在相应项目工作区内，使用当前环境生成的 values 文件执行：

```bash
helm lint deploy/charts/spring-app \
  --values .jenkins-json-build/deploy-overrides-values.yaml

helm template spring-app deploy/charts/spring-app \
  --namespace spring-app \
  --values .jenkins-json-build/deploy-overrides-values.yaml
```

检查输出中的：

- `spec.rules[0].host`；
- `spec.tls[0].secretName`；
- Deployment 的镜像仓库和摘要；
- Service 端口和类型。

### 12.2 检查集群实际对象

```bash
kubectl -n spring-app get deployment,service,ingress,configmap
kubectl -n spring-app get ingress spring-app \
  -o jsonpath='{.spec.rules[0].host}{"\n"}{.spec.tls[0].secretName}{"\n"}'
```

云服务器应看到：

```text
app.cloud.k8s.lab
k8s-cloud-lab-tls
```

虚拟机应看到：

```text
app.k8s.lab
k8s-lab-tls
```

## 13. 一句话总结

> 同一个项目不需要为每个环境复制一份源码或 Chart；每次构建时，把 Chart 默认值、当前环境的可选 values 和本次构建的镜像参数合并，再由 Helm 生成并提交该环境自己的 Kubernetes 期望状态。
