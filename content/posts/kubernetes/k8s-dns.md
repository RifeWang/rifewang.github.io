+++
draft = false
date = 2024-09-17T19:50:36+08:00
title = "Kubernetes 集群内 DNS"
description = "Kubernetes 集群内 DNS"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

## DNS 简介

在互联网早期，随着连接设备数量的增加，IP 地址的管理与记忆变得越来越复杂。为了简化网络资源的访问，`DNS`（Domain Name System）应运而生。`DNS` 的核心作用是将用户可读的域名（如 www.example.com）解析为对应的 IP 地址（如 93.184.215.34），从而使用户无需记忆复杂的数字串，便能轻松访问全球各地的网络资源。

`DNS` 是一种应用层协议，与 HTTP(S) 等协议同属 OSI 网络模型中的最高层。它依赖于 client-server 模型进行工作，客户端向 DNS 服务器发出查询请求，服务器返回对应的 IP 地址。传统的 DNS 通过 UDP 或 TCP 传输，随着互联网安全需求的提升，`DNS over TLS`、`DNS over HTTPS (DoH)`、`DNS over QUIC` 等新协议也相继推出，以增强数据隐私和安全性。

要实现 DNS 查询，客户端需要知道 DNS 服务器地址，而这些信息通常配置在客户端的 `/etc/resolv.conf` 文件中。这个文件记录了 DNS 服务器的 IP 地址以及查询时的附加选项。接下来我们会详细探讨 Kubernetes 集群内的 `/etc/resolv.conf` 文件和相关的 `kubelet` 配置。


## K8S DNS 配置详解

在 Kubernetes 中，每个 Pod 都拥有一个由 `kubelet` 自动生成的 `/etc/resolv.conf` 文件，文件内容一般如下：
```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

其中：
- `nameserver`：指定了 DNS server 的集群内 IP 地址，DNS server 早期使用的是 `kube-dns`，现在则基本是 `CoreDNS`。如果 `nameserver` 设置了多个地址，则表示前一个地址无响应时依次向下一个地址发起 DNS 查询。
- `search`：定义了 DNS 查询时的域名后缀补全顺序。
- `options ndots:5`：这个选项规定域名中至少有 5 个 `.` 才会被认为是 FQDN 完全限定域名。如果查询的域名中点的数量不足，则会根据 `search` 补全后缀。例如，查询 `mysvc` 时，其会被补全为 `mysvc.default.svc.cluster.local.`，最后真正请求的是补全后的域名。

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/k8s-dns-config.png)

`/etc/resolv.conf` 文件是由 `kubelet` 在 Pod 启动时为每个 Pod 自动生成的。Kubelet 的配置（早期通过命令行参数的形式，后面则调整为了配置文件）决定了 Pod 中 `/etc/resolv.conf` 的具体内容，具体关联如下：
- `--cluster-dns` 或 `clusterDNS`：指定集群内 DNS 服务器的 IP 地址，映射到 `/etc/resolv.conf` 中的 `nameserver`。
- `--cluster-domain` 或 `clusterDomain`：用于配置集群的默认域名后缀（如 `cluster.local`），映射到 `/etc/resolv.conf` 中的 `search`。
- `options ndots:5` 这一项则在 kubelet 代码中写死了，参考 [kubelet 源码](https://github.com/kubernetes/kubernetes/blob/v1.31.1/pkg/kubelet/network/dns/dns.go#L43)。

补充说明：`search default.svc.cluster.local ...` 是 kubelet 的默认规则，对应 `<namespace>.svc.<clusterDomain>`，namespace 是 pod 所在的命名空间名称。

此外，Pod 的 DNS 行为也受 `dnsPolicy` 和 `dnsConfig` 这两个配置的影响。

`dnsPolicy` 决定 Pod 使用哪种 DNS 策略。有以下四种取值：
- `Default`：虽然名字叫 default 但并不是默认选型，此时 pod 的 `/etc/resolv.conf` 使用所在 node 节点的配置，而该配置又通过 kubelet 的配置项 resolvConf 指定。
- `ClusterFirst`：默认选项，使用集群内的 DNS server 进行域名解析。
- `ClusterFirstWithHostNet`：当 pod 使用 hostNetwork 时应该设置此项。
- `None`：忽略此前 DNS 设置，具体配置完全由 `dnsConfig` 决定。

`dnsConfig` 自定义 pod 的 `/etc/resolv.conf`，例如：
```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx
  dnsPolicy: "None"
  dnsConfig: # 自定义 pod 的 /etc/resolv.conf
    nameservers:
      - 192.0.2.1
    searches:
      - ns1.svc.cluster-domain.example
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
```

## 总结

DNS 是 client-server 请求-响应模型，日常所说的公网的 DNS 被称为 global DNS，而像 kubernetes 集群内的 DNS 则是 private DNS。在 Kubernetes 集群中，Pod 的 DNS 设置由 `kubelet` 管理，所有的 DNS 查询首先由集群内的 DNS 服务器（通常是 `CoreDNS`）处理。Pod 的 `/etc/resolv.conf` 文件内容则由 `kubelet` 的配置以及 Pod 的 `dnsPolicy` 和 `dnsConfig` 设置决定。掌握这些配置项的含义与用法，对于在 Kubernetes 集群中进行 DNS 问题排查和优化至关重要。

---

参考资料：

- *https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/*
- *https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/*
- *https://www.man7.org/linux/man-pages/man5/resolv.conf.5.html*
- *https://www.rfc-editor.org/rfc/rfc9499.html*
- *https://www.rfc-editor.org/rfc/rfc1034.txt*
- *https://www.rfc-editor.org/rfc/rfc1035.txt*
