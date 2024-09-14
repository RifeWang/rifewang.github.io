+++
draft = false
date = 2024-09-13T12:11:23+08:00
title = "Kubernetes CNI 网络模型概览：VETH & Bridge / Overlay / BGP"
description = "Kubernetes CNI 网络模型概览：VETH & Bridge / Overlay / BGP"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

## 序言

网络是容器通信的基础，Kubernetes 本身并未提供开箱即用的网络互通功能，只提出了两点基本要求：

> - pods can communicate with all other pods on any other node without NAT
> - agents on a node (e.g. system daemons, kubelet) can communicate with all pods on that node

至于如何实现这些通信能力，通常依赖于 `CNI` 插件来完成。

## IPAM

Kubernetes 的网络模型要求每个 Pod 拥有唯一的 IP 地址，负责管理和分配这些 IP 地址的功能由 `IPAM`（IP Address Management）来实现。而 `IPAM` 也是 `CNI` 插件的重要组成部分。

一种常见的 `IPAM` 实现方式是先给每个 node 节点分配一个 CIDR（Classless Inter-Domain Routing 无类别域间路由），然后根据 node 节点的 CIDR 范围分配该节点上的 Pod IP 地址。

例如，每个 node 节点设置 CIDR：
```yaml
apiVersion: v1
kind: Node
metadata:
  name: node01
spec:
  podCIDR: 192.168.1.0/24
  podCIDRs:
  - 192.168.1.0/24

---

apiVersion: v1
kind: Node
metadata:
  name: node02
spec:
  podCIDR: 192.168.2.0/24
  podCIDRs:
  - 192.168.2.0/24
```

上例中：
- node01 的 `podCIDR` 是 `192.168.1.0/24`（地址范围是：192.168.1.0 ~ 192.168.1.255），那么该节点上 Pod 的 IP 地址范围是：`192.168.1.1 ~ 192.168.1.254`（首尾两个地址有其它特殊用途）。
- node02 的 `podCIDR` 是 `192.168.2.0/24`（地址范围是：192.168.2.0 ~ 192.168.2.255），那么该节点上 Pod 的 IP 地址范围是：`192.168.2.1 ~ 192.168.2.254`。

简单理解就是给每个 node 节点划分一个小的子网，同一个 node 节点上的 pod IP 地址处于同一个子网范围内。

注意！以上只是 `IPAM` 的实现方式之一，不同的 `CNI` 插件可能还会提供其它的实现方式。


## Linux VETH & Bridge

Pod 的 IP 地址确定后，接下来的问题是如何在集群内进行通信。

Linux 提供了丰富的虚拟网络接口类型来支持复杂的网络环境。`VETH`（virtual Ethernet）和 `Bridge` 就是其中的两种虚拟网络接口。

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/k8s-linux-veth-bridge.drawio.png)

如上图所示（你可能见过很多类似的图片）。每个 Pod 有自己的 network namespace（独立的网络命名空间），通过 veth pair 与 root namespace（主机网络命名空间）打通，然后再使用 bridge（通常叫 `cni0` 或 `docker0`）将所有 veth pair 连接在一起。

需要注意的是，`VETH` 和 `Bridge` 是相互独立的技术。如果我们使用 veth pair 直连两个 pod ，那么这两个 pod 就可以互相通信。但是如果只使用 veth pair，当 pod 数量越来越多时，为了确保两两互通，veth pair 的数量就会非常庞大进而难以管理，这也是使用 bridge 的原因之一。

在同一个 node 节点上的 pod 互相通信很容易理解，veth pair 和 bridge 在本机就处理掉了。

但如果跨越多个 node 节点的时候呢？我们似乎很想当然的以为，由 bridge 交给 root namespace 的 eth0 真实网卡去转发流量就行了，真的这么简单吗？

当然不是！node 节点可能是虚拟机或者物理机，node 节点可能在同一台物理机上通过虚拟网络互联，也可能是不同的物理机通过路由器或交换机相互连接。这里的问题就在于，**连接 node 节点的这个网络设备（虚拟/真实的路由器/交换机/其它设备）是否能够直接路由 Pod 的 MAC/IP 地址？**（路由 node 的 IP 地址很自然，但是 node 内部的 pod MAC/IP 地址又怎么知道呢？）

如果上面这个问题的答案是可以，那么 `CNI` 插件只使用 `VETH` 和 `Bridge` 就能完成集群内容器的互联互通，这就是本文想说的第一种 `CNI` 网络模型，在 `Cilium` 中，这种网络模型称为 `Native-Routing`，而在 `Flannel` 中则叫 `host-gw` 模式。


## Overlay

如前文所述，如果连接 node 节点的路由设备无法路由 pod MAC/IP，那么集群内的网络互通又如何保证？答案是使用 `Overlay` 网络。

各个 node 节点是互通的（node IP 可路由），那么将原始数据包封装一层再传输，例如 `VXLAN` 封包：

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/vxlan-frame.jpg)

如上图所示，原始的 pod IP 地址被封装在 Inner Ethernet Frame 里作为数据包，在外层添加 node 节点的 IP 地址，由于 node 节点互通，因此数据包会被正常转发到目标 node 节点，然后再解包并根据内层的 pod IP 转发给 pod。

`Overlay` 网络通过在现有网络的基础上封装额外的一层网络包，使得 Pod 可以跨节点进行通信。

`Overlay` 的实现方式有很多，`VXLAN` 是比较常见的一种，除此之外还有 `IP-in-IP` 等等。Linux 内核直接提供了 `VXLAN` 的功能，但是不同的 CNI 插件不一定直接使用的就是 Linux 内核功能，CNI 插件可能会绕过内核自己实现 `VXLAN` 的封包解包。

这是本文想介绍的第二种 `CNI` 网络模型 `Overlay`。


## BGP

当涉及到多集群或混合云场景时，某些情况下集群内的 Pod 需要能被外部直接访问。此时 `BGP`（Border Gateway Protocol 边界网关协议）便是解决方案之一。

`BGP` 已经广泛用于大规模数据中心或骨干网络的路由，它允许不同 AS（autonomous systems）自治系统（可以简单理解为两个独立的网络）之间交换路由信息。

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/BGP.jpeg)

在 Kubernetes 中，可以将集群视为一个 AS 自治系统，通过 `BGP` 交互路由信息（边界网关互相知道对方网络中的路由），也就是可以通过宣布 Pod 的 IP 地址，使得集群外能够直接路由到这些 Pod IP 地址，从而实现 Pod 的集群外部可达性。

既然能使用 `BGP` 交互路由信息，那么直接通过 `BGP` 去交换集群内所有 pod IP 信息，从而让集群内 pod IP 可路由，进而直接使用本文上述的第一种 `CNI` 网络模型 `Native-Routing` 是否可行？答案是可行，但是不建议这么做，因为 `BGP` 缺乏数据路径的可编程性（不建议使用 BGP 建立集群内部网络的可达性）。


## 总结

本文介绍了 `Native-Routing`、`Overlay`、`BGP` 三种 `CNI` 网络模型。使用 `Native-Routing` 的前提条件是集群内 Pod IP 可路由，然而很多时候无法满足此要求，因此众多 `CNI` 插件通过 `VXLAN`/`IP-in-IP` 等方式实现了 `Overlay` 网络，进而保证集群内容器的互通。在某些复杂场景下（如跨集群或跨云环境），集群内 Pod 需要支持集群外部的可访问性，此时则可以使用 `BGP` 的方式。

---

参考资料：

- *https://kubernetes.io/docs/concepts/services-networking/*
- *https://kubernetes.io/docs/concepts/cluster-administration/addons/*
- *https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking*
- *https://docs.cilium.io/en/stable/network/concepts/routing/*
- *https://docs.tigera.io/calico/latest/networking/configuring/vxlan-ipip*
- *https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md*
