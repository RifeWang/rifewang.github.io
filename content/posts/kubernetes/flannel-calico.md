+++
draft = false
date = 2024-11-30T10:23:42+08:00
title = "Kubernetes 集群网络：Flannel 与 Calico 的区别"
description = "Kubernetes 集群网络：Flannel 与 Calico 的区别"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

有读者提问：`Flannel` 与 `Calico` 的区别。文本将解析一下这两个组件。

## Flannel

`Flannel` 的架构非常简单，只有两个组件：`flanneld` 和 `flannel-cni-plugin`。

在功能特性上，`Flannel` 有三个部分：
- 使用 `kubernetes API` 或 `etcd` 存储数据。
- `IPAM`，给节点的 Pod 分配 IP 地址范围（`podCIDR`）。
- 配置后端转发数据包。

`Flannel` 支持以下后端机制进行数据包转发：
- `VXLAN`：使用 Linux kernel 的 `VXLAN` 封包并转发，这种方式属于 `Overlay` 网络。
- `host-gw`：`Flannel` 会直接在节点上创建 IP 路由，这种方式依赖节点之间的 L2（二层网络）的连通性。
- `WireGuard`：使用 kernel 的 `WireGuard` 封装并加密数据包。
- `UDP`：仅调试使用。
- 还有一些实验性后端，本文不做赘述。

### 网络模型

#### `host-gw`

`host-gw` 单词缩写的意思是使用 host 主机的 gateway。这种方式不会额外地封装数据包，但要求节点之间的 L2 连通性。

使用 `host-gw` 时，`Flannel` 做的事情就是通过 `github.com/vishvananda/netlink` 这个第三方库去配置主机路由表，即执行 `ip route ...` 命令，例如：

```sh
ip route add <destination> via <gateway> dev <device>
```

假设我们有以下三个节点：
- `node01`：主机 IP 是 `192.168.1.1`，podCIDR 是 `10.244.1.0/24`。
- `node02`：主机 IP 是 `192.168.1.2`，podCIDR 是 `10.244.2.0/24`。
- `node03`：主机 IP 是 `192.168.1.3`，podCIDR 是 `10.244.3.0/24`。

那么使用 `host-gw` 时，`Flannel` 会把 `node01` 的主机路由表配置为类似以下形式：

```sh
# node01 本地路由
10.244.1.0/24 dev flannel.1 proto kernel scopee link

# node02 路由
10.244.2.0/24 via 192.168.1.2 dev eth0

# node03 路由
10.244.3.0/24 via 192.168.1.3 dev eth0
```

在 `node02` 节点上同样配置类似的主机路由表：

```sh
# node01 路由
10.244.1.0/24 via 192.168.1.1 dev eth0

# node02 本地路由
10.244.2.0/24 dev flannel.1 proto kernel scopee link

# node03 路由
10.244.3.0/24 via 192.168.1.3 dev eth0
```

其它节点类似。

当 `node01` 节点上的 Pod 想要向 `node02` 节点上的 Pod 发送数据时，会先根据以上路由表得知目标 IP 地址，然后通过 `ARP` 协议得到该地址对应的 `MAC` 地址（`L2` 数据链路层地址），再把数据包封装为数据帧传输。 

`L2` 的连通性要求节点之间能够在数据链路层直接通信，这通常意味着节点必须位于同一广播域（如同一个子网）。此时，数据帧可以通过二层设备（如交换机）基于目标 `MAC` 地址直接转发，而无需三层设备（如路由器）进行 IP 层转发。广播域的范围决定了 `L2` 通信的有效范围。

如果节点位于不同子网或通过路由器连接，尽管源节点可以通过路由表得知目标 IP 地址，但由于目标 IP 不在本地子网，源节点需要通过 `ARP` 获取下一跳路由器的 `MAC` 地址，而非目标节点的 `MAC` 地址。数据帧首先发送到路由器，路由器在转发时根据目标网络重新封装帧。这种场景下，`host-gw` 模式依赖的直接二层转发机制无法实现。

`host-gw` 模式数据转发上能得到更高的性能，没有额外的封包过程，L2 直连，但是由于每个节点都要配置全量的主机路由表，当节点数量庞大、节点上下线频繁时会有不小的挑战。

更多关于 L2 直连的规模问题，可以参考 [Concerns over Ethernet at scale](https://docs.tigera.io/calico/latest/reference/architecture/design/l2-interconnect-fabric)。


#### `VXLAN`

如果无法保证 `L2` 的直接连通性，则需要使用 `VXLAN` 这种 `Overlay` 的方式，在原有数据帧之外进行一层额外的封装。

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/network/VXLAN.drawio.png)

此时，即使节点位于不同的子网，或通过路由器等设备组网都没有关系，但是 `VXLAN` 要求 L3 必须连通，即通过节点的 IP 地址能够进行通信。因此，`VXLAN` 也称为 `L2 over L3`，不用管下层 L2 是什么样的，只要 L3 连通即可。

`Flannel` 在 `VXLAN` 模式下做的事情也是通过 `github.com/vishvananda/netlink` 这个第三方库去执行 `ip link ...` 命令：

```sh
ip link add <vxlan-name> type vxlan id <vxlan-id> dev <device> ...
```

这也就是说 `Flannel` 本身不进行任何数据包的封装和转发，是系统内核在做实际的工作。


## Calico

`Calico` 的架构则更加复杂，由多个组件构成。

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/network/calico-architecture.png)

其中：
- `Calico API server`：用户可以使用 kubectl 直接管理 Calico 资源。
- `Calico kube-controllers`：监控资源并执行操作。
- `Datastore plugin`：使用 `kubernetes API` 或 `etcd` 存储数据。
- `Typha`：作为数据存储和 `Felix` 实例之间的守护进程运行，减少数据存储的负载以增加集群规模。在大规模（100 个节点以上）的集群中必不可少。
- `Felix`：运行在每个节点上，控制节点的网络接口、路由、ACL 流量控制和安全策略、以及报告状态等。
- `BIRD`：在承载 `Felix` 的每个节点上运行，从 `Felix` 获取路由并分发给网络中的 `BGP` 对等点，用于主机间路由。
- `confd`：监控数据存储中 BGP 配置和 AS 号、日志级别和 IPAM 等全局默认值的变化，动态生成 `BIRD` 配置文件，并触发 BIRD 加载新的配置。
- `Dikastes`：可选，为 Istio 服务网格执行网络策略。作为 Istio Envoy 的 sidecar 代理在集群上运行。
- `CNI plugin` 和 `IPAM plugin`。
- `calicoctl`：命令行工具。


`Calico` 和 `Flannel` 一样使用 `kubernetes API` 或 `etcd` 存储数据，不同的是 `Calico` 在数据存储与消费者之间增加了一个中间层 `Typha`，减少了存储侧的负担，以支持更大规模的集群。

`Calico` 和 `Flannel` 一样都支持 `IPAM`（给节点分配 `podCIDR`），不同的是 `Flannel` 是静态的，而 `Calico` 可以动态分配。

### 网络模型

`Calico` 支持 `Overlay` 的网络模型，包括 `VXLAN` 和 `IP-in-IP`（`IPIP` 在 `Flannel` 中处于实验性阶段），它同样是使用 `vishvananda/netlink` 这个第三方库去配置系统内核，由内核进行封包和转发。换句话说，如果两者都使用 `VXLAN`，那么封包和转发性能没有任何差别。

`Calico` 支持 `non-overlay` 模式，类似于 `Flannel` 中的 `host-gw`，但 `Calico` 的实现方式有区别，每个节点负责路由自己的 Pod 子网，并通过 `BGP` 协议将路由信息分发给其他节点，能够实现更灵活的路由控制，支持更复杂的网络拓扑。

`Calico` 通过 `BGP` 协议可以支持 Pod IP 在集群外部可路由，而 `Flannel` 并不支持这点。

此外，`Calico` 针对不同的云供应商环境做了贴心的适配，并且额外支持 `eBPF`、`Prometheus` 监控、`network policy` 网络策略和流量管控（`Flannel` 并不支持这些）。


## 总结

`Flannel` 的架构非常简单，在小规模集群上能够很好的工作。

`Calico` 的架构更加复杂，不仅支持 `Flannel` 具备的所有功能，还提供了更多的网络模型（如 `IPIP`、`BGP`），并支持 `eBPF`、监控、网络策略和流量管控等诸多功能，适用于更大规模的集群。


---

(我是凌虚，关注我，无广告，专注技术，不煽动情绪，欢迎与我交流)

---

参考资料：

- *https://github.com/flannel-io/flannel*
- *https://docs.tigera.io/calico/latest/networking/determine-best-networking*