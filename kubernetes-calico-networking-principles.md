# Kubernetes 与 Calico 网络运行机制：Operator、CNI、Felix、BGP、IPIP 与 RouterOS

> 更新时间：2026-08-13  
> 文档定位：云服务器方案与本地虚拟机方案共用的外部原理附件。  
> 配套主文档：[云服务器方案](./kubernetes-jenkins-buildkit-github-springboot3-postgresql-cloud-server-guide.md)、[本地虚拟机方案](./kubernetes-jenkins-buildkit-github-springboot3-postgresql-vm-guide.md)。  
> 当前共同基线：Kubernetes 节点网段为 `192.168.0.0/24`，Pod 网段为 `10.244.0.0/16`，Calico 为 `v3.32.1`，节点 AS 为 `64512`。  
> 方案差异：云服务器使用 Calico 节点间 BGP + IPIP；本地虚拟机使用 Calico 节点间 BGP + 无封装，并与 RouterOS 建立 eBGP。

## 一、先掌握六个结论

1. CRD 只让 Kubernetes 认识新的资源类型，本身不会安装或运行网络组件。
2. Tigera Operator 是运行在 Pod 中的长期管理程序。它把 `Installation` 等高层配置转换成 DaemonSet、Deployment、权限和配置等 Kubernetes 资源，并持续维护整套 Calico 的安装状态。
3. 创建 Pod 时，kubelet 通过容器运行时调用 Calico CNI。CNI 给 Pod 分配地址、创建 veth、配置 Pod 网络空间，并登记网络端点。
4. veth 只把 Pod 接到本机 Linux 网络。它不能自己解决跨节点路由、底层网络转发、回程路径和网络策略。
5. Felix、BGP 组件与 CNI 负责配置网络；真正逐包转发的是 Linux 内核。`proto bird` 表示路由由 BIRD 写入内核，不表示数据包经过 BIRD 进程。
6. RouterOS 是否有价值不取决于网卡数量，而取决于是否需要在不同 IP 网段之间查路由并转发。单网卡 RouterOS 也能作为单臂路由器，把局域网设备的流量转发到正确的 Pod 所在节点。

## 二、Kubernetes 为什么需要 Operator

### 2.1 Kubernetes 的基本工作方式

Kubernetes API 中保存的是“期望状态”。例如：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example
spec:
  replicas: 3
```

`kube-apiserver` 负责认证、鉴权、字段校验和 API 访问，配置最终通过 API Server 保存到 etcd。API Server 不会亲自创建三个容器；Deployment Controller、ReplicaSet Controller、Scheduler 和 kubelet 会继续完成工作。

```text
用户提交 Deployment/replicas=3
  -> kube-apiserver 校验并保存
  -> Deployment Controller 创建 ReplicaSet
  -> ReplicaSet Controller 创建 Pod
  -> Scheduler 选择节点
  -> 节点 kubelet 通过容器运行时启动 Pod
```

控制器会持续比较：

```text
期望状态：3 个 Pod
实际状态：2 个 Pod
处理动作：再创建 1 个 Pod
```

Operator 使用的也是同一种控制机制，只是它管理的是 Kubernetes 原本不认识的业务领域。在这里，这个领域就是 Calico 的安装和网络能力。

### 2.2 CRD、Custom Resource 和 Operator 的关系

三个概念不能混在一起：

| 对象 | 作用 | 是否执行安装动作 |
| --- | --- | --- |
| CRD | 定义资源名称、字段、类型和校验规则 | 否 |
| Custom Resource | 按 CRD 格式填写一份期望配置 | 否 |
| Operator | 监听配置，创建和维护真正运行的组件 | 是 |

例如，应用 `v1_crd_projectcalico_org.yaml` 后，Kubernetes 才能认识：

```text
Installation.operator.tigera.io
APIServer.operator.tigera.io
BGPConfiguration.crd.projectcalico.org
BGPPeer.crd.projectcalico.org
IPPool.crd.projectcalico.org
```

创建下面的对象时，API Server 只会验证并保存它：

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
      - cidr: 10.244.0.0/16
        encapsulation: IPIP
```

API Server 不理解怎样创建 `calico-node`，也不会因为看见 `IPIP` 就配置 Linux 隧道。理解并实施这份高层配置的是 Tigera Operator。

可以把三者理解为：

```text
CRD                 = 定义配置表的结构
Installation/default = 填写期望的 Calico 安装方式
Tigera Operator      = 持续读取配置并实施
```

### 2.3 Operator 在集群中的真实身份

`tigera-operator.yaml` 创建的核心程序是一个普通 Deployment：

```text
Deployment/tigera-operator
  -> ReplicaSet
    -> Pod/tigera-operator-...
      -> operator 进程
```

它使用 ServiceAccount 和 ClusterRole，通过 Kubernetes API 获得监听、创建和更新相关资源的权限。Operator 不需要逐台 SSH 登录节点，也不直接修改每台服务器上的所有文件。

Operator 的持续工作链路是：

```text
watch Kubernetes API
  -> 发现 Installation、APIServer 等对象新增或变化
  -> 读取当前实际资源和状态
  -> 计算应该创建或修改什么
  -> 通过 Kubernetes API 提交 DaemonSet、Deployment、RBAC、ConfigMap 等资源
  -> 更新 TigeraStatus
  -> 等待下一次变化，再次检查
```

这个不断比较“期望状态”和“实际状态”的过程，通常称为控制循环。

### 2.4 Operator 和 Kubernetes 内置控制器怎样接力

Operator 不负责独自完成所有动作。它创建上层资源后，Kubernetes 内置控制器继续工作：

```text
Installation/default
  -> Tigera Operator
  -> DaemonSet/calico-node
  -> DaemonSet Controller
  -> 每个节点各有一个 calico-node Pod
  -> kubelet/containerd
  -> 运行节点网络组件
```

因此，删除一个 `calico-node` Pod 后，通常首先是 DaemonSet Controller 根据仍然存在的 DaemonSet 补建 Pod，不是 Operator 每次亲自补建。只有 DaemonSet 本身、Operator 管理的配置或整套安装资源偏离期望时，Operator 才在更上层进行修正。

各控制程序的职责如下：

| 程序 | 管理对象 | 是否在数据包路径中 |
| --- | --- | --- |
| Tigera Operator | Calico 的安装、升级、组件配置和总体状态 | 否 |
| Kubernetes 内置控制器 | DaemonSet、Deployment、Pod 等原生资源 | 否 |
| calico-kube-controllers | 节点、IPAM 等日常 Calico 控制任务 | 否 |
| Calico CNI | Pod 创建和删除时配置或清理网络 | 只在创建、删除时执行 |
| Felix | 持续配置每个节点的 Linux 网络和策略 | 否 |
| BIRD | 建立 BGP 会话、学习和发布路由 | 否 |
| Linux 内核 | 按路由、邻居表和策略实际转发数据包 | 是 |

如果 Tigera Operator 暂时停止，已经建立的数据路径通常仍能继续工作，因为数据包不经过 Operator。此时 Calico 的高层配置变更、安装维护、状态修正和升级会受到影响。

## 三、Pod 创建时怎样接入 Linux 网络

### 3.1 CNI 的调用链路

Pod 被调度到某个节点后，典型调用顺序是：

```text
kubelet
  -> 通过 CRI 请求 containerd 创建 Pod Sandbox
  -> containerd 按节点 CNI 配置调用 Calico CNI
  -> Calico IPAM 分配 Pod IP
  -> Calico CNI 创建 veth、配置路由并登记网络端点
  -> 容器开始使用网络
```

Calico 安装组件会把 CNI 可执行文件和配置放到节点的标准位置，典型位置包括：

```text
/opt/cni/bin/
/etc/cni/net.d/
```

CNI 是容器运行时在 Pod 网络建立和删除时调用的标准接口，不是一个处理每个数据包的常驻代理。

### 3.2 veth 到底解决了什么

Linux 为每个 Pod 创建独立网络空间。Calico CNI 创建一对 veth，可以理解成一根虚拟网线的两端：

```text
Pod 网络空间                      节点网络空间

eth0 10.244.1.2  <------------>  cali...
```

它解决的是：

```text
Pod 怎样把数据送入本机 Linux 网络
本机 Linux 怎样把数据送回这个 Pod
```

它没有解决：

```text
远端 Pod 位于哪台节点
数据怎样穿过节点之间的网络
回包怎样找到原 Pod
哪些通信应该被网络策略允许
```

虚拟机网卡看似“创建后就能通信”，是因为背后已经有虚拟交换机、网桥、物理网络、地址配置、ARP 和路由。veth 只相当于其中很短的一段连接。

### 3.3 同一节点上的 Pod 通信

两个 Pod 位于同一节点时，不需要 BGP，也不需要 IPIP：

```text
Pod A eth0
  -> veth
  -> 节点 Linux 路由和 Calico 策略
  -> 另一个 veth
  -> Pod B eth0
```

Linux 根据本机工作负载路由找到目标 `cali...` 接口。Calico 默认是三层路由思路，不应简单理解成所有 Pod 都接在同一个普通 Linux 网桥上。

## 四、Felix、BIRD 与 Linux 内核怎样配合

### 4.1 Felix 是节点网络配置程序

Felix 运行在每台节点的 `calico-node` 中，持续读取 Kubernetes 和 Calico 状态，然后配置节点数据面。它主要负责：

- 维护本机工作负载接口和路由；
- 设置 IP 转发和相关系统参数；
- 把 NetworkPolicy 转换成 iptables、nftables 或 eBPF 规则，具体方式取决于安装模式；
- 维护 NAT、端点状态和其他 Calico 数据面设置。

Felix 不是软件路由器进程。数据包不会先进入 Felix 用户态程序再被它逐个转发。Felix 提前把规则写好，随后由 Linux 内核按这些规则处理数据包。

### 4.2 BGP 组件负责交换路由信息

当前两份方案使用标准 Linux Calico BGP 模式。`calico-node` 中的 BGP 配置组件根据 Calico 资源生成 BIRD 配置，BIRD 通过 TCP `179` 与其他 BGP 邻居建立会话。

节点之间交换的核心信息是：

```text
节点 A：10.244.1.0/26 在我这里
节点 B：10.244.2.0/26 在我这里
节点 C：10.244.3.0/26 在我这里
```

BGP 不传输 Pod 业务数据，只交换“目标网段应该从哪里到达”的信息。

### 4.3 路由究竟记录在哪里

BGP 相关信息分为四层：

| 层次 | 记录内容 | 是否持久配置 |
| --- | --- | --- |
| Kubernetes API/etcd | `BGPConfiguration`、`BGPPeer`、`IPPool` 等期望配置 | 是 |
| BIRD 内存 | BGP 邻居、收到的候选路由、选中的 BGP 路由 | 否，重启后重新学习 |
| Linux 内核路由表 | 当前实际用于转发的路由 | 否，由组件重新写入 |
| RouterOS 配置库与路由表 | RouterOS 的 BGP 配置、会话以及学到的路由 | 配置持久；动态路由重新学习 |

BIRD 把选中的远端 Pod 路由写入 Linux 内核后，`ip route` 可能显示：

```text
10.244.2.0/26 via 192.168.0.11 dev k8s0 proto bird
```

或者：

```text
10.244.2.0/26 via 192.168.0.11 dev tunl0 proto bird onlink
```

输出是否包含 `onlink` 等附加字段取决于具体节点状态。这里最重要的是：

```text
10.244.2.0/26     目标 Pod 地址块
via 192.168.0.11  承载该地址块的下一跳节点
dev ...           内核选用的逻辑出口设备
proto bird        这条路由由 BIRD 协议写入内核
```

`proto bird` 只标记路由来源。实际数据包由 Linux 内核转发，不经过 BIRD 用户态进程。

## 五、无封装路由：`dev k8s0`

假设：

```text
节点 A：192.168.0.10，Pod A：10.244.1.2
节点 B：192.168.0.11，Pod B：10.244.2.3
```

节点 A 有一条典型路由：

```text
10.244.2.0/26 via 192.168.0.11 dev k8s0 proto bird
```

它的意思是：目标属于 `10.244.2.0/26` 时，保留原始 Pod IP 包，通过 `k8s0` 把二层帧交给下一跳节点 `192.168.0.11`。

原始 IP 头不会改变：

```text
源 IP：10.244.1.2
目标 IP：10.244.2.3
```

在当前本地虚拟机方案中，节点 A 与节点 B 的 `k8s0` 位于同一二层局域网。节点 A 可以通过 ARP 获得 `192.168.0.11` 的 MAC 地址，随后发送：

```text
以太网目标 MAC：节点 B 的 k8s0 MAC
IP 目标地址：Pod B 的 10.244.2.3
```

交换机只根据目标 MAC 把帧送到节点 B，不需要理解 `10.244.2.0/26`。节点 B 收到帧后，Linux 再根据本机路由把 IP 包交给 Pod B。

完整路径：

```text
Pod A eth0
  -> veth
  -> 节点 A Linux 路由
  -> k8s0
  -> 二层局域网
  -> 节点 B k8s0
  -> 节点 B Linux 路由和 Calico 策略
  -> veth
  -> Pod B eth0
```

无封装并不表示任何网络都能直接承载 Pod 地址。中间如果存在三层路由器、云平台源地址检查、防伪造限制或不允许 Pod 地址的数据面，就必须让底层网络认识 Pod 路由，或者改用隧道封装。

## 六、IPIP 路由：`dev tunl0`

云服务器方案中的典型路由是：

```text
10.244.2.0/26 via 192.168.0.11 dev tunl0 proto bird onlink
```

它表示原始 Pod 包先交给 `tunl0`，由 Linux 增加一个外层 IPv4 头。

原始内层数据包：

```text
源 IP：10.244.1.2
目标 IP：10.244.2.3
```

IPIP 封装后：

```text
外层 IPv4：
  源 IP：192.168.0.10
  目标 IP：192.168.0.11
  协议号：4

内层 IPv4：
  源 IP：10.244.1.2
  目标 IP：10.244.2.3
```

节点 A 会进行两次路由判断：

```text
第一次查询内层目标 10.244.2.3
  -> 命中远端 Pod 路由
  -> 进入 tunl0

tunl0 增加外层 IP 头

第二次查询外层目标 192.168.0.11
  -> 命中节点网段路由
  -> 从节点真实网卡发出
```

因此，`dev tunl0` 不代表数据最终从虚拟隧道设备直接进入网线。最终物理出口仍然是承载 `192.168.0.0/24` 的真实网卡。

完整路径：

```text
Pod A eth0
  -> veth
  -> 节点 A Linux 路由和 Calico 策略
  -> tunl0 封装
  -> 节点 A 真实网卡
  -> 阿里云 VPC
  -> 节点 B 真实网卡
  -> tunl0 解封装
  -> 节点 B Linux 路由和 Calico 策略
  -> veth
  -> Pod B eth0
```

阿里云 VPC 看到的是节点地址之间的通信：

```text
192.168.0.10 -> 192.168.0.11，IP 协议号 4
```

它不需要根据内层 `10.244.2.3` 决定应该送往哪台 ECS。当前云服务器方案因此固定使用 IPIP，并要求节点内部网络允许 IP 协议号 `4`。

### 6.1 两条路由的直接对比

| 项目 | `dev k8s0` | `dev tunl0` |
| --- | --- | --- |
| Calico 模式 | 无封装 | IPIP |
| 原始 Pod IP 包 | 直接发送 | 放在外层节点 IP 包内部 |
| 底层网络看到的源和目标 IP | Pod IP | 节点 IP |
| 最终物理出口 | `k8s0` | 仍是节点真实网卡 |
| 额外头部 | 无 | 外层 IPv4 头，通常增加 20 字节 |
| 中间网络要求 | 必须允许原始 Pod 地址通过，并具备正确路径 | 只需承载节点地址，但必须允许 IP 协议号 4 |
| 当前方案 | 本地虚拟机 | 阿里云 ECS |

一句话概括：

```text
dev k8s0：拿着原始 Pod 包，直接交给承载目标 Pod 的节点。
dev tunl0：先把原始 Pod 包放进节点 IP 包，再交给目标节点。
```

## 七、云服务器方案中的 BGP 记录在哪里

云服务器方案没有 RouterOS，也不会把 Calico Pod 路由写入阿里云 VPC 路由表。

```text
ECS Master 192.168.0.10 <---- iBGP ----> ECS Node1 192.168.0.11
          \                              /
           \----------- iBGP ----------/
                         |
                  ECS Node2 192.168.0.12
```

配置位置：

```yaml
apiVersion: crd.projectcalico.org/v1
kind: BGPConfiguration
metadata:
  name: default
spec:
  asNumber: 64512
  nodeToNodeMeshEnabled: true
```

运行时记录位置：

| 内容 | 位置 |
| --- | --- |
| AS 和节点 Mesh 配置 | Kubernetes API/etcd 中的 `BGPConfiguration/default` |
| Calico 选用的节点地址 | Kubernetes Node 的 `projectcalico.org/IPv4Address` 注解 |
| BGP 邻居和候选路由 | 各 ECS 的 `calico-node` 中的 BIRD |
| 实际转发路由 | 各 ECS 的 Linux 内核路由表 |
| IPIP 逻辑设备 | 各 ECS 的 `tunl0` |
| Calico Pod 路由 | 不存在于阿里云公网 BGP 或 VPC 路由表中 |

BGP 回答“目标 Pod 地址块位于哪台 ECS”，IPIP 负责让业务数据穿过 VPC。二者不是替代关系。

## 八、虚拟机方案中的 BGP 记录在哪里

虚拟机方案同时存在两组 BGP 关系：

```text
第一组：三个 Calico 节点之间的 iBGP Mesh
第二组：每个 Calico 节点与 RouterOS 之间的 eBGP
```

Calico 配置保存在 Kubernetes API/etcd：

```yaml
apiVersion: crd.projectcalico.org/v1
kind: BGPConfiguration
metadata:
  name: default
spec:
  asNumber: 64512
  nodeToNodeMeshEnabled: true
---
apiVersion: crd.projectcalico.org/v1
kind: BGPPeer
metadata:
  name: routeros-peer
spec:
  peerIP: 192.168.0.2
  asNumber: 65000
  nextHopMode: Keep
```

RouterOS 自己保存 BGP 实例、模板和监听连接配置。会话建立后，RouterOS 从 Calico 学到类似路由：

```text
10.244.1.0/26 -> 192.168.0.10
10.244.2.0/26 -> 192.168.0.11
10.244.3.0/26 -> 192.168.0.12
```

运行时记录位置：

| 内容 | 位置 |
| --- | --- |
| Calico AS、节点 Mesh | Kubernetes `BGPConfiguration/default` |
| RouterOS 邻居定义 | Kubernetes `BGPPeer/routeros-peer` |
| 节点 BGP 会话和路由 | 各节点 `calico-node` 中的 BIRD |
| 节点实际转发路由 | 各虚拟机 Linux 内核路由表 |
| RouterOS BGP 实例和连接 | RouterOS 配置库 |
| RouterOS 学到的 Pod 路由 | RouterOS BGP 路由表和 `main` 路由表 |
| Mac 汇总路由 | Mac 本机路由表中的静态路由，不是 BGP 路由 |

`nextHopMode: Keep` 用来保留地址块的真实下一跳。这样 RouterOS 收到某个 Pod 子网的路由时，仍能把它交给真正承载该地址块的 Kubernetes 节点。

## 九、单网卡 RouterOS 为什么仍然有作用

### 9.1 同一节点网段内，它确实不参与

三台服务器都是 `192.168.0.0/24` 时：

```text
192.168.0.10 -> 192.168.0.11
```

发送方根据 `/24` 掩码判断目标与自己在同一网段，通过 ARP 找到对方 MAC，然后经交换机直接发送。RouterOS 不在这条路径上。

三个节点之间的 Pod 通信也通常不经过 RouterOS，因为 Calico 节点 Mesh 已经让节点直接知道其他 Pod 地址块的下一跳：

```text
Pod A -> 节点 A -> 节点 B -> Pod B
```

### 9.2 局域网访问 Pod 网段时，它开始发挥作用

局域网和 Pod 使用两个逻辑 IP 网段：

```text
局域网/节点网段：192.168.0.0/24
Pod 网段：       10.244.0.0/16
```

Mac 不参加 BGP，只配置一条汇总静态路由：

```text
10.244.0.0/16 -> 192.168.0.2
```

RouterOS 再根据从 Calico 学到的更精确 `/26` 路由，把数据交给正确节点：

```text
Mac 192.168.0.5
  -> RouterOS 192.168.0.2
  -> 查询 BGP 学到的 10.244.x.0/26 路由
  -> K8S Node 192.168.0.10/.11/.12
  -> 目标 Pod 10.244.x.x
```

RouterOS 的数据可能从 `ether1` 进入，再从同一个 `ether1` 发出：

```text
ether1 收包
  -> RouterOS 查询 IP 路由表
  -> 修改二层目标 MAC并减少 IP TTL
  -> ether1 发给正确 Kubernetes 节点
```

这叫单臂路由。路由器不要求每个逻辑网段都对应一块独立物理网卡，也不要求 RouterOS 自己拥有 `10.244.0.0/16` 中的地址。它只需要：

- 收到发往 Pod 网段的数据包；
- 路由表知道正确下一跳；
- 开启转发并允许该流量；
- 返回路径正确。

### 9.3 RouterOS 在当前方案中的实际边界

| 通信 | RouterOS 是否参与 |
| --- | --- |
| 节点 `192.168.0.10` 访问节点 `192.168.0.11` | 否 |
| 同一节点上的两个 Pod 通信 | 否 |
| 不同节点的两个 Pod 通信 | 通常否，Calico Mesh 直接转发 |
| Mac 或其他局域网设备访问 Pod IP | 是 |
| Mac 代理向保留 Pod 源地址的连接返回数据 | 是 |
| Calico 向外发布 Pod 路由 | RouterOS 是 BGP 接收方 |

所以，RouterOS 不是当前 Kubernetes 集群内部 Pod 互通的必要组件。它的实验价值是：

```text
让普通局域网只维护一条 10.244.0.0/16 汇总路由，
再由 RouterOS 通过 BGP 自动学习每个 /26 Pod 地址块实际位于哪台节点。
```

如果不使用 RouterOS，可以改用 Service/Ingress/NAT，不直接访问 Pod IP；也可以让真实主路由参加 BGP，或者在需要访问 Pod 的设备和路由器上维护正确的静态路由。

## 十、两套方案的数据路径对照

### 10.1 云服务器跨节点 Pod 通信

```text
Pod A 10.244.1.2
  -> 节点 A veth
  -> Linux 查询 BIRD 写入的远端 Pod 路由
  -> tunl0 添加 IPIP 外层头
  -> ECS A 私网地址 192.168.0.10
  -> 阿里云 VPC
  -> ECS B 私网地址 192.168.0.11
  -> tunl0 解封装
  -> 节点 B 本机 Pod 路由
  -> Pod B 10.244.2.3
```

### 10.2 虚拟机跨节点 Pod 通信

```text
Pod A 10.244.1.2
  -> 节点 A veth
  -> Linux 查询 BIRD 写入的远端 Pod 路由
  -> k8s0 直接发送原始 Pod IP 包
  -> 局域网交换
  -> 节点 B k8s0
  -> 节点 B 本机 Pod 路由
  -> Pod B 10.244.2.3
```

### 10.3 Mac 向虚拟机 Pod 返回数据

```text
Mac 根据静态路由：10.244.0.0/16 -> 192.168.0.2
  -> RouterOS
  -> RouterOS 根据 BGP /26 路由选择真实节点
  -> 节点 k8s0
  -> 节点本机 Pod 路由
  -> 目标 Pod
```

这条路径可能与 Pod 到 Mac 的正向路径不同。Pod 所在节点可以直接把数据发给同一局域网的 Mac；Mac 返回 Pod 网段时再经过 RouterOS。普通无状态 IP 路由允许这种不对称路径，但防火墙、NAT 和连接跟踪设计必须与实际路径一致。

## 十一、按层验证，不要只看一个状态

### 11.1 验证 Kubernetes 和 Operator 层

```bash
kubectl get crd installations.operator.tigera.io
kubectl get installation default -o yaml
kubectl -n tigera-operator get deployment,pod
kubectl get tigerastatus
kubectl -n calico-system get daemonset,deployment,pod -o wide
```

这一层回答：CRD 是否存在、Operator 是否运行、期望配置是否被实施、Calico 组件是否就绪。

### 11.2 验证 Calico 配置层

```bash
kubectl get bgpconfiguration default -o yaml
kubectl get bgppeers.crd.projectcalico.org -o wide
kubectl get ippools.crd.projectcalico.org -o yaml
kubectl get nodes -o \
  'custom-columns=NAME:.metadata.name,INTERNAL_IP:.status.addresses[?(@.type=="InternalIP")].address,CALICO_BGP_IPV4:.metadata.annotations.projectcalico\.org/IPv4Address'
```

这一层回答：节点 AS、Mesh、RouterOS Peer、地址池、封装方式和节点 BGP 地址是否正确。

### 11.3 验证 BGP 会话层

```bash
for pod in $(kubectl -n calico-system get pod -l k8s-app=calico-node -o name); do
  echo "===== ${pod} ====="
  kubectl -n calico-system exec "$pod" -c calico-node -- calico-node -bird-ready
done
```

虚拟机方案还要在 RouterOS 查看：

```routeros
/routing/bgp/session/print detail
/routing/route/print detail where bgp=yes
```

这一层只能证明邻居和路由交换状态，不能单独证明实际 Pod 数据一定能通过。

### 11.4 验证 Linux 数据面

在节点上检查：

```bash
ip route show proto bird
ip route show | grep 10.244
ip route get 10.244.2.3
ip link show | grep -E 'cali|tunl0|k8s0'
ip -d link show tunl0
```

云服务器方案应重点确认远端 Pod 路由经过 `tunl0`；虚拟机方案应重点确认远端 Pod 路由直接经过 `k8s0`。

在受控实验环境抓包时，可以分别观察：

```bash
# 无封装：物理接口上直接看到 Pod IP。
sudo tcpdump -ni k8s0 net 10.244.0.0/16

# IPIP：物理接口上看到 IP 协议号 4 的节点间流量。
sudo tcpdump -ni k8s0 proto 4

# IPIP 隧道逻辑接口上观察内层 Pod 流量。
sudo tcpdump -ni tunl0 net 10.244.0.0/16
```

最后必须使用固定在不同节点的测试 Pod 做双向通信验证。只有“组件状态、BGP 会话、Linux 路由和实际数据”全部符合预期，才能确认网络链路正常。

## 十二、常见误解

### 12.1 “安装了 CRD 就安装了 Calico”

不正确。CRD 只注册资源类型；还需要 Operator、`Installation/default` 和后续运行组件。

### 12.2 “Operator 在转发 Pod 数据”

不正确。Operator 管理安装和生命周期，不在数据包路径中。

### 12.3 “Felix 或 BIRD 逐个转发数据包”

不正确。它们配置 Linux 内核；真正逐包处理的是内核网络栈。

### 12.4 “创建 veth 后跨节点自然就通了”

不正确。veth 只连接 Pod 与本机，还需要远端路由、底层承载、回程路径和策略。

### 12.5 “BGP 和 IPIP 二选一”

不正确。当前云方案中，BGP 提供路由信息，IPIP 提供底层承载，两者同时工作。

### 12.6 “路由器必须有两个物理接口”

不正确。两个接口是常见形式，不是路由成立的必要条件。单个接口也能承载多个逻辑网段并完成单臂路由。

### 12.7 “节点都在同一个 `/24`，RouterOS 完全没有作用”

只对节点之间的 `192.168.0.0/24` 通信成立。局域网设备访问独立的 `10.244.0.0/16` Pod 网段时，RouterOS 仍负责根据 BGP 路由选择正确节点。

## 十三、最终关系图

```text
安装与控制链路：

CRD
  -> Kubernetes 认识 Calico 资源
Custom Resource
  -> 保存期望配置
Tigera Operator
  -> 创建和维护 Calico 安装资源
Kubernetes 控制器和 kubelet
  -> 让组件在各节点真正运行

Pod 网络建立链路：

kubelet/containerd
  -> 调用 Calico CNI
  -> IPAM 分配地址
  -> veth 把 Pod 接入本机

持续网络配置链路：

Felix
  -> 配置本机接口、策略、NAT 和工作负载路由
BIRD/BGP
  -> 学习和发布远端 Pod 路由
Linux 内核
  -> 按路由与策略实际转发

跨节点承载：

本地虚拟机
  -> dev k8s0
  -> 原始 Pod IP 包直接交给下一跳节点

阿里云 ECS
  -> dev tunl0
  -> IPIP 封装后通过 VPC 交给目标节点

局域网访问 Pod：

Mac 静态汇总路由
  -> 单网卡 RouterOS
  -> BGP 学到的精确 Pod 子网路由
  -> 正确 Kubernetes 节点
  -> 目标 Pod
```

## 十四、资料来源

- [Calico Operator 安装参考](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)
- [Calico Installation API](https://docs.tigera.io/calico/latest/reference/installation/api)
- [Calico BGP 配置](https://docs.tigera.io/calico/latest/networking/configuring/bgp)
- [Calico IPIP 与 VXLAN](https://docs.tigera.io/calico/latest/networking/configuring/vxlan-ipip)
- [Calico CNI 插件](https://docs.tigera.io/calico/latest/reference/configure-cni-plugins)
- [Kubernetes Operator 模式](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Kubernetes CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [CNI 规范](https://www.cni.dev/docs/spec/)
