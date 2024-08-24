+++
draft = false
date = 2024-08-24T13:24:29+08:00
title = "图解计算机网络：一条 HTTP 请求的网络拓扑之旅"
description = "图解计算机网络：一条 HTTP 请求的网络拓扑之旅"
slug = ""
authors = []
tags = ["网络"]
categories = ["网络"]
externalLink = ""
series = []
disableComments = true
+++

## 引言

常见的网络拓扑结构如下图所示：

![](https://raw.githubusercontent.com/RifeWang/images/master/network/network-network-topology.drawio.png)

在此拓扑中，终端设备通过 WiFi 连接到路由器，路由器再连接到光猫（或终端设备通过移动网络 4G/5G 连接到基站），之后 ISP 网络服务提供商接管网络通信，将请求最终转发至应用服务器。

从用户设备发出的 HTTP 请求是如何穿越网络的？我们将深入探讨这一过程。


## HTTP 请求的网络旅途

### OSI 网络体系结构

先从计算机网络的基础架构开始：

![](https://raw.githubusercontent.com/RifeWang/images/master/network/network-osi-layer.drawio.png)

上图展示了五层简化版 OSI 网络模型。每层都对网络通信至关重要，特别是在 HTTP 请求的传递过程中。关键点包括：
- 传输层：TCP 头部包含的源端口和目标端口。
- 网络层：IP 头部包含的源 IP 地址和目标 IP 地址。
- 数据链路层：MAC 头部包含的源 MAC 地址和目标 MAC 地址。

接下来我们来看看，在网络设备的转发过程中，这些信息如何发生变化。

### HTTP 网络之旅

下图展示了完整的网络路径：

![](https://raw.githubusercontent.com/RifeWang/images/master/network/network-network-travel.drawio.png)

#### 一、用户终端设备 --> 路由器

![](https://raw.githubusercontent.com/RifeWang/images/master/network/from-user-to-router.png)

1. HTTP 请求基于 TCP 连接。用户通常会请求一个域名地址，首先必须通过 `DNS` 解析获取服务器的 IP 地址。`DNS` 的查询过程如下：
    - 浏览器 DNS 缓存（如果访问的是 web 网页）。
    - 本地操作系统 DNS 缓存。
    - 本地 `/etc/hosts` 文件是否有配置域名到 IP 的直接映射。
    - DNS 查询：
        - 向域名服务器（地址配置在 `/etc/resolv.conf` 文件）发起查询：
            - 域名服务器地址可以是：ISP 域名服务器或公共 DNS 服务器（如 Google 8.8.8.8 或 Cloudflare 1.1.1.1）。
            - 通常，终端设备通过路由器连接网络，这时 `/etc/resolv.conf` 的 `nameserver` 指向的就是路由器的 WAN IP（如 192.168.3.1），路由器将继续转发 DNS 查询。
        - 递归查询：根域名服务器 -> 顶级域名服务器 -> 权威域名服务器（存储真实的 DNS 记录）。
2. DNS 解析完毕后，数据开始从应用层向下传递并封装：
    - 传输层：封装 TCP 头，包含源端口（一般随机生成）和目标端口（HTTP 默认 80）。
    - 网络层：封装 IP 头，包含源 IP（用户设备的内网 IP）和目标 IP（远程服务器公网 IP）地址。
3. 用户设备通过 `ARP` 协议查找目标 MAC 地址（此时得到路由器的 MAC 地址）。在数据链路层则封装了 MAC 头部，其中包含了源 MAC（用户设备 MAC）地址和目标 MAC（路由器 MAC）地址。
4. 最后，数据在物理层通过无线电波（WiFi）传递二进制数据到路由器（如果是双绞线则是电信号，光纤则是光信号）。路由器接收到数据后，进入下一阶段处理。

#### 二、路由器 --> 光猫

![](https://raw.githubusercontent.com/RifeWang/images/master/network/from-router-to-oni.png)

路由器在物理层接收到二进制数据后，将数据解析成上一层的数据链路层格式，随后改写源 MAC 和目标 MAC 地址，将数据发送到下一跳设备（光猫）。

由于路由器没有公网 IP 地址，此时不会进行 NAT（网络地址转换），IP 头部中的源 IP 仍为用户设备的内网 IP。

#### 三、光猫 --> ISP NAT 设备

![](https://raw.githubusercontent.com/RifeWang/images/master/network/from-oni-to-nat.png)

光猫负责将数据通过一系列中间网络设备，最终传递到 ISP 的 NAT 设备。

#### 四、ISP NAT 设备 --> 服务器

![](https://raw.githubusercontent.com/RifeWang/images/master/network/from-nat-to-server.png)

由于公网 IPv4 地址的数量有限，ISP 通常会为同一区域的多个用户共享一个公网 IP。

此时，ISP 的 NAT 设备会将源 IP 地址转换为共享的公网 IP 地址。至此，数据才进入公网传输，并最终达到应用服务器。

服务器在接受到数据后，从下层往上依次解析数据，最终还原出来应用层的 HTTP 请求。

## 总结

![](https://raw.githubusercontent.com/RifeWang/images/master/network/network-network-travel.drawio.png)

本文介绍了 HTTP 请求从用户终端设备到应用服务器的全过程，并通过图示说明了请求如何在 OSI 模型的不同层次间传递和转化。