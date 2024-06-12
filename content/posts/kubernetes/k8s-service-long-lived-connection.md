+++
draft = false
date = 2024-06-12T16:14:46+08:00
title = "Kubernetes Service 与 long-lived connections"
description = "Kubernetes Service 与 long-lived connections"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

本文将会介绍：
- 从 pod 到 service 再到 pod，kubernetes 中的流量是怎么走的？
- 对于 long-lived connection 长连接又是怎样的情况？

## 从 pod 到 service 再到 pod

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/service1.png)

如上图所示：

1、我们先创建一个多副本的 deployment，k8s 会通过 CNI（容器网络接口）给每个 pod 分配一个集群内可达的 IP 地址。

2、我们随后创建一个类型为 clusterIP 的 `service`，指向 deployment（即其所属的所有 pod ），此时 `service` 会被赋予一个 virtual IP（虚拟 IP 地址）。

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/service2.png)

然后，我们从另外的 pod 发起请求，请求地址是这个 `service` 的虚拟 IP 或者 `coreDNS` 的内部域名（随后也会转换成 IP）。`service` 会随机将请求转发到一个后端 pod，至此 pod 到 pod 的连接就建立完成了。

但是，流量真的是 `service` 转发的吗？显然并不是。`service` 只是一个配置项而已，并不负责转发流量的具体工作。

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/service3.png)

每个 node 节点上有一个重要的组件 `kube-proxy`，它会监听所有 `service`，然后配置 `iptables`（默认） 或者 `ipvs`（此处配置项详见 kube-proxy 的 [`--proxy-mode`](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)），由 `iptables`/`ipvs` 完成真正的流量转发。

## iptables 和 long-lived connections

`iptables` 是 Linux 系统中一个用于配置和管理网络规则的工具，用于对网络数据包进行过滤、转发、修改等操作，可以实现防火墙、网络地址转换（NAT）、负载均衡等功能。k8s 拿它来进行 L4 的 TCP/UDP 转发。

`iptables` 本身是支持 `random`（基于概率的随机转发）和 `nth`（round robin 轮询）两种转发策略的，但是 k8s 固定使用了 `random` 随机转发且无法配置。

对于 long-lived connections（例如 HTTP/1.1 keep-alive、HTTP/2、gRPC、WebSocket）呢？

这里必须说明，`iptables` 进行随机的包转发，这句话容易产生误解。更准确一点应该是，`iptables` 只会在 TCP 连接刚开始创建的时候随机选择一个目标 pod，此后通过内核的 `connection tracking` 机制跟踪和记录每个连接的状态，进而让已经建立了 TCP 连接的双方一直使用这条连接。

所以 long-lived connections 在 k8s 里是支持的。

但是，会存在负载不均衡的情况，比如 pod A 的长连接请求永远都是打到 pod B 上，而另一个 pod C 长期空闲。

如何解决负载不均衡？常见的方案有两种：

- 客户端自己做负载均衡，这意味着客户端需要获得 service 背后绑定的 pod IP，然后自己实现负载均衡算法，对客户端来说复杂度更高了。
- 使用中间层，例如 `service mesh` 服务网格专门去处理流量。


---

参考资料：

- *https://learnk8s.io/kubernetes-long-lived-connections*
- *https://scalingo.com/blog/iptables*
- *https://learnk8s.io/kubernetes-network-packets*
