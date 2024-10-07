+++
draft = false
date = 2024-10-07T19:06:17+08:00
title = "Kubernetes: kube-proxy 和 CNI 是如何协作的？"
description = "Kubernetes: kube-proxy 和 CNI 是如何协作的？"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

在 Kubernetes 中，`kube-proxy` 和 `CNI` 插件协同工作，确保集群内 Pod 之间的互联互通。

## Kube-proxy & CNI

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/network/kube-proxy&cni.png)

如上图所示，假设我们有一个类型为 `ClusterIP` 的 `Service`，它对应两个位于不同节点的 Pod。

当我们从 Pod A 对该 `Service` 发起请求时：
1. `Pod A: 192.168.0.2  -->  service-name`（通过域名访问 Service）。
2. CoreDNS 进行域名解析，返回 service-name 的 clusterIP 地址：
    `Pod A: 192.168.0.2  -->  10.111.13.31`
3. Linux 内核的 Netfilter 进行 DNAT（目标网络地址转换），选择一个真实的后端 Pod IP：
    `Pod A: 192.168.0.2  -->  192.168.1.4`
4. 如果请求的 Pod 位于不同节点，则 CNI 插件将根据其配置的网络模式：
    - 直接交由主机网络转发数据包，
    - 或者使用 `VXLAN` / `IPIP` 等方式封包之后再转发数据包。

以上是基本流程，但实际过程中涉及更多细节。

### kube-proxy

`kube-proxy` 通常部署为 DaemonSet，确保所有 worker node 都有一个 pod。它连接 kube-apiserver，监听所有 Service（以及相关的 Endpoints、EndpointSlices）对象，然后通过 `iptables` 或 `ipvs` 配置节点的 `Netfilter`。

`Netfilter` 是 Linux 内核中用于处理网络数据包的框架。它工作在网络协议栈的多个位置（如输入、输出、转发），允许对数据包进行过滤、修改、重定向等操作。它的钩子机制允许在数据包处理的不同阶段插入自定义规则。

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/network/linux-netfilter.png)

`iptables` 和 `ipvs` 都使用了 `Netfilter` 的功能。

Service 的 ClusterIP 是一个 virtual IP（虚拟 IP，简称 `VIP`），虚拟的含义是它本身没有实际的网络实体，只是被用作数据包的处理逻辑。

`kube-proxy` 使用 `iptables` 或者 `ipvs` 设置 Service ClusterIP（虚拟 IP）和 DNAT/SNAT 规则。当 Pod A 访问 ClusterIP 时，Linux 内核会通过这些规则将虚拟 IP 替换为真实的 Pod IP。

查看 `iptables` 的规则形如：

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/network/iptables.png)

查看 `ipvs` 的规则形如：

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/network/ipvs.png)


Service 可能会对应多个 Pod，此时就涉及到负载均衡算法。`iptables` 使用的是随机选择；而 `ipvs` 则支持多种算法，包括 `rr`（Round Robin 轮询，默认值）、`lc`（Least Connection 最少连接）、`sh`（Source Hashing 源 IP 哈希）等等十几种。

总之，`kube-proxy` 监听 Service 并使用 `iptables` 或者 `ipvs` 去配置 Linux 内核的数据包修改、转发规则。

### CNI

当 Linux 内核完成 DNAT/SNAT 之后，接下来的步骤是发包。如果 Pod 的 MAC/IP 在集群网络中可达，则 `CNI` 插件无需做额外操作，直接交由主机网络发包即可。然而，很多时候无法满足此要求，因此 `CNI` 插件提供了 `Overlay` 这种网络模式，通过 `VXLAN` 或 `IPIP` 等方式再次封装数据包，然后进行传输。

有关 CNI 插件的网络模型，可以参考我之前的文章《Kubernetes CNI 网络模型概览：VETH & Bridge / Overlay / BGP》。

常见的 `CNI` 插件（如 `Flanne`、`Calico`）使用的 `VXLAN` 或 `IPIP` 是基于 Linux 内核的功能，`CNI` 插件的主要作用是进行配置。

这意味着，`kube-proxy` 和 `CNI` 插件都只是为 Linux 内核提供配置，而实际的数据包处理仍由内核完成。（某些 CNI 插件如 `Cilium` 通过 `eBPF` 等技术可以绕过内核，直接在用户空间处理数据包。）


## 无需 kube-proxy

`kube-proxy` 存在一些局限性，例如随着集群中 Service 和 Pod 数量的增长，`iptables` 的规则匹配效率会下降，即使 `ipvs` 进行了优化，依然面临性能开销。此外，频繁的 Service 和 Pod 更新会导致规则重新应用，可能带来网络延迟或中断。`kube-proxy` 还依赖于 Linux 内核的 `Netfilter`，一旦内核配置不当或不兼容，可能会出现网络问题。

事实上，`kube-proxy` 的工作相对简单，如果网络插件能够实现数据包的高效转发，并提供与 `kube-proxy` 等效的功能，那么就不再需要 `kube-proxy` 了。例如，`Cilium` 通过 `eBPF` 实现了无代理的服务流量转发，完全可以替代 `kube-proxy`。


## 总结

在 Kubernetes 集群中，`kube-proxy` 和 `CNI` 插件通过配置 Linux 内核的网络组件（如 `Netfilter`、`VXLAN` 等），协同工作以确保 Pod 和 Service 之间的通信。随着 `eBPF` 等技术的发展，某些 CNI 插件能够实现无代理的服务转发，进一步优化了网络性能。


(关注我，无广告，专注于技术，不煽动情绪)

---

参考资料：

- *https://kubernetes.io/docs/reference/networking/virtual-ips/*
- *https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/*
