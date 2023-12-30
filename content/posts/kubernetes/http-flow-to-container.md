+++
draft = false
date = 2023-12-30T16:38:11+08:00
title = "Kubernetes 外部 HTTP 请求到达 Pod 容器的全过程"
description = "Kubernetes 外部 HTTP 请求到达 Pod 中的应用容器的全过程"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

`Kubernetes` 集群外部的 HTTP/HTTPS 请求是如何达到 Pod 中的 `container` 的？


## HTTP 请求流转过程概述

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/k8s-http-flow-simple.png)

如上图所示，全过程大致为：
1. 用户从 web/mobile/pc 等客户端发出 HTTP/HTTPS 请求。
2. 由于应用服务通常是通过域名的形式对外暴露，所以请求将会先进行 `DNS` 域名解析，得到对应的公网 `IP` 地址。
3. 公网 `IP` 地址通常会绑定一个 `Load Balancer` 负载均衡器，此时请求会进入此负载均衡器。
    - `Load Balancer` 负载均衡器可以是硬件，也可以是软件，它通常会保持稳定（固定的公网 IP 地址），因为如果切换 IP 地址会因为 DNS 缓存的原因导致服务某段时间内不可达。
    - `Load Balancer` 负载均衡器是一个重要的中间层，对外承接公网流量，对内进行流量的管理和转发。
4. `Load Balancer` 再将请求转发到 `kubernetes` 集群的某个流量入口点，通常是 `ingress`。
    - `ingress` 负责集群内部的路由转发，可以看成是集群内部的网关。
    - `ingress` 只是配置，具体进行流量转发的是 `ingress-controller`，后者有多种选择，比如 Nginx、HAProxy、Traefik、Kong 等等。
5. `ingress` 根据用户自定义的路由规则进一步转发到 `service`。
    - 比如根据请求的 path 路径或 host 做转发。
![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/ingress-fanout.png)
![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/ingress-namebased.png)
6. `service` 根据 selector（匹配 label 标签）将请求转发到 `pod`。
    - `service` 有多种类型，集群内部最常用的类型就是 `ClusterIP`。
    - `service` 本质上也只是一种配置，这种配置最终会作用到 node 节点上的 `kube-proxy` 组件，后者会通过设置 `iptables/ipvs` 来完成实际的请求转发。
    - `service` 可能会对应多个 `pod`，但最终请求只会被随机转发到一个 `pod` 上。
7. `pod` 最后将请求发送给其中的 `container` 容器。
    - 同一个 `pod` 内部可能有多个 `container`，但是多个容器不能共用同一个端口，因此这里会根据具体的端口号将请求发给对应的 `container`。

以上就是一种典型的集群外部 HTTP 请求如何达到 Pod 中的 `container` 的全过程。

需要注意的是，由于网络配置灵活多变，以上请求流转过程并不是唯一的方式，例如：
- 如果你使用的是云服务，那么可以通过使用 `LoadBalancer` 类型的 `service` 直接绑定一个云服务商提供的负载均衡器，然后再接 ingress 或者其它 service。
- 你也可以通过 `NodePort` 类型的 `service` 直接使用节点上的端口，通过这些节点自建负载均衡器。
- 如果你的服务特别简单，没啥内部流量需要管理的，这时不用 `ingress` 也是可以的。

## 容器技术的底座

容器技术的底座有三样东西：
- `Namespace`（这里是指 Linux 系统内核的命名空间）
- `Cgroups`
- `UnionFS`

正是 Linux 内核的 namespace 实现了资源的隔离。因为每个 pod 有各自的 Linux namespace，所以不同的 pod 是资源隔离的。namespace 有多种，包括 PID、IPC、Network、Mount、Time 等等。其中 PID namespace 实现了进程的隔离，因此 pod 内可以有自己的 1 号进程。而 Network namespace 则让每个 pod 有了自己的网络。

Pod 有自己的网络，node 节点也有自己的网络，那么流量是如何从 node 节点到 pod 的呢？

## HTTP 请求流转过程补充

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/k8s-http-flow.png)

每个 node 节点上都有：
- `kubelet`：节点的小管家。
- `kube-proxy`：操作节点的 iptables/ipvs 。
- plugins:
    - `CRI`：容器运行时接口
    - `CNI`：容器网络接口
    - `CSI`（可选）：容器存储接口

每个 node 节点有自己的 root namespace，其中也包括网络相关的 root netns，每个 pod 有自己的 pod netns，从 node 到 pod 则可以通过 `veth pairs` 的方式连通，流量也正是通过此通道进行的流转。而构建 `veth pairs`、设置 pod network namespace、为 pod 分配 IP 地址等等工作则正是 `CNI` 的任务。

至此，一个典型的 `kubernetes` 集群外部的 HTTP/HTTPS 请求如何达到 Pod 中的 `container` 的全过程就是这样了。

---

参考资料：

- *https://kubernetes.io/docs/concepts/services-networking/*
- *https://learnk8s.io/kubernetes-network-packets*
