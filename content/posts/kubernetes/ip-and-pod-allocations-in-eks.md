+++
draft = false
date = 2023-03-23T18:34:51+08:00
title = "EKS 集群中的 IP 地址分配问题"
description = "EKS 集群中的 IP 地址分配问题"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

---

*文本翻译自: https://itnext.io/ip-and-pod-allocations-in-eks-5be6612b8325*

---

![](https://miro.medium.com/v2/0*5ASKzTbKkuNTFjOn.png)

运行 EKS 集群时，你可能会遇到两个问题：

- 分配给 pod 的 IP 地址用完了。
- 每个节点的 pod 数量少（由于 ENI 限制）。

在本文中，你将学习如何克服这些问题。

在我们开始之前，这里有一些关于节点内网络如何在 Kubernetes 中工作的背景知识。

创建节点时，kubelet 委托：

1. 创建容器到容器运行时。
2. 将容器连接到 CNI 的网络。
3. 将卷安装到 CSI。

![kubelet 将任务委托给 CRI、CNI 和 CSI](https://miro.medium.com/v2/0*eJxeBijeBd5Y_zht.png)

*让我们关注 CNI 部分。*

**每个 pod 都有自己独立的 Linux 网络命名空间，并连接到一个网桥。**

CNI 负责创建网桥、分配 IP 并将 veth0 连接到 cni0。

![大多数情况下，一个节点上的所有容器都连接到一个网桥上](https://miro.medium.com/v2/0*VCpGPOkWGAAqxcuh.png)

这通常会发生，但不同的 CNI 可能会使用其他方式将容器连接到网络。

**例如，可能没有 cni0 网桥。**

AWS-CNI 是此类 CNI 的一个示例。

![并非所有 CNI 都使用网桥连接同一节点上的容器](https://miro.medium.com/v2/0*PnhSgM-9PNPcHoTa.png)

在 AWS 中，每个 EC2 实例都可以有多个网络接口 (ENI)。

**你可以为每个 ENI 分配有限数量的 IP。**

例如，一个 `m5.large` 实例可以为 ENI 分配最多 10 个 IP。

在这 10 个 IP 中，你必须将一个分配给网络接口。

剩下的你可以不用管。

![弹性网络接口和 IP 地址](https://miro.medium.com/v2/0*vZ-nFSgNOQQ9wDyq.png)

**以前，你可以使用额外的 IP 并将它们分配给 Pod。**

但是有一个很大的限制：IP 地址的数量。

*让我们看一个例子。*

使用 `m5.large` 实例，你最多有 3 个 ENI，每个有 10 个 IP 私有地址。

由于保留了一个 IP，每个 ENI 还剩下 9 个（总共 27 个）。

这意味着你的 `m5.large` 实例最多可以运行 27 个 Pod。

*这不是很多。*

![你最多可以在 m5.large 中拥有 27 个 pod](https://miro.medium.com/v2/0*7_6qmTgfy27SVaLg.png)

**但是 AWS 发布了对 EC2 的更改，允许将“地址前缀”分配给网络接口。**

*地址前缀是什么？！*

简而言之，ENI 现在支持范围而不是单个 IP 地址。

**如果以前你可以拥有 10 个私有 IP 地址，那么现在你可以拥有 10 个 IP 地址槽。**

*地址槽有多大呢？*

默认情况下，16 个 IP 地址。

使用 10 个槽，你最多可以拥有 160 个 IP 地址。

这是一个相当显着的变化！

*让我们看一个例子。*

![EC2 中的地址前后对比](https://miro.medium.com/v2/0*_roqQcLfflfPaKPn.png)

使用 `m5.large` 实例，你有 3 个 ENI，每个有 10 个插槽（或 IP）。

由于为 ENI 保留了一个 IP，因此你还剩下 9 个插槽。

每个插槽是 16 个 IP，所以是 `9*16=144` 个 IP。

由于有 3 个 ENI，那就是 `144x3=432` 个 IP。

**你现在最多可以拥有 432 个 Pod（之前是 27 个）。**

![你最多可以在 m5.large 中拥有 432 个 pod](https://miro.medium.com/v2/0*rpcU1W03eKQtbC_Z.png)

AWS-CNI 支持插槽并将 Pod 的最大数量限制为 110 或 250，因此你最多可以在 m5.large 中拥有 432 个 pod 。

还值得指出的是，这不是默认启用的——即使在较新的集群中也是如此。

*可能是因为只有 nitro 实例支持它。*

分配插槽非常棒，直到你意识到 CNI 一次提供 16 个 IP 地址，而不是仅提供 1 个，这具有以下含义：

- 更快地耗尽 IP 空间。
- 碎片化。

*让我们回顾一下。*

![EC2 和 EKS 中的前缀问题](https://miro.medium.com/v2/0*Ya4QHmsFonBglalW.png)

一个 pod 被调度到一个节点。

AWS-CNI 分配 1 个 slot（16 个 IP），pod 使用一个。

现在想象一下有 5 个节点和一个包含 5 个副本的部署。

*会发生什么？*

![](https://miro.medium.com/v2/0*sO3OokcIZQ42Bycl.png)

**Kubernetes 调度程序更喜欢将 pod 分布在整个集群中。**

很可能，每个节点接收 1 个 pod，AWS-CNI 分配 1 个插槽（16 个 IP）。

你从你的网络分配了 `5*15=75` 个 IP，但仅使用了 5 个。

![使用 AWS CNI 分配 IP](https://miro.medium.com/v2/0*9M39EbGj42jB_XF3.png)

*但还有更多。*

**插槽分配一个连续的 IP 地址块。**

如果分配了一个新 IP（例如创建了一个节点），你可能会遇到碎片问题。

![](https://miro.medium.com/v2/0*9nDFF_RXoMlqgpcx.png)

*怎么解决这些问题呢？*

- [你可以为 EKS 分配一个次级 CIDR。](https://aws.amazon.com/premiumsupport/knowledge-center/eks-multiple-cidr-ranges/)
- [你可以在子网内保留 IP 空间供插槽独占使用。](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-cidr-reservation.html)


相关链接：

- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI
- https://aws.amazon.com/blogs/containers/amazon-vpc-cni-increases-pods-per-node-limits/
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-prefix-eni.html#ec2-prefix-basics

