+++
draft = false
date = 2023-03-31T16:11:20+08:00
title = "在 Kubernetes 中应该如何设置 CPU 的 requests 和 limits"
description = "在 Kubernetes 中应该如何设置 CPU 的 requests 和 limits"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

---

*文本翻译自: https://itnext.io/cpu-limits-and-requests-in-kubernetes-fa9d55948b7c*

---

![](https://miro.medium.com/v2/0*ikA_Xbhg7x4goOIW.png)

*在 Kubernetes 中，我应该如何设置 CPU 的 requests 和 limits？*

热门答案包括：

- 始终使用 limits !
- 永远不要使用 limits，只使用 requests !
- 都不用；可以吗？

让我们深入研究它。

在 Kubernetes 中，您有两种方法来指定一个 pod 可以使用多少 CPU：

1. **Requests** 通常用于确定平均消耗。
2. **Limits** 设置允许的最大资源数。

Kubernetes 调度器使用 requests 来确定 pod 应该分配到集群中的哪个节点。

由于调度器并不知道实际消耗（pod 尚未启动），它需要一个提示。

但它并没有就此结束。

![Kubernetes 调度器使用 requests 来决定如何将 pod 分配给节点](https://miro.medium.com/v2/0*2D2ERPTIbuEWNhff.png)

CPU requests 还用于将同一个节点上的 CPU 资源如何分配给不同的容器。

让我们看一个例子：

- 一个节点只有一个 CPU。
- 容器 A requests 0.1 个 vCPU。
- 容器 B requests 0.2 个 vCPU。

*当两个容器都尝试使用 100% 的可用 CPU 时会发生什么？*

![两个容器的 CPU 使用率](https://miro.medium.com/v2/0*9KcoDnah9KRBVKrN.png)

由于 CPU 请求不限制消耗，因此两个容器都将使用所有可用的 CPU。

但是，由于容器 B 的请求与另一个相比增加了一倍，因此最终的 CPU 分配是：**容器 1 使用 0.3vCPU，另一个使用 0.6vCPU（双倍数量）。**

![两个容器都使用所有可用的 CPU，但它们保持比例配额](https://miro.medium.com/v2/0*Uh640qLTFZ2GE5gm.png)

Requests 适用于：

- 设置基准（给我至少 X 数量的 CPU）。
- 设置 pod 之间的关系（这个 pod A 使用的 CPU 是另一个的两倍）。

但不影响硬性限制。

为此，您需要 CPU limits。

**设置 CPU limits 时，您定义了 period 周期和 quota 配额。**

例如：

- 周期：100000 微秒 (0.1s)。
- 配额：10000 微秒 (0.01s)。

我只能每 0.1 秒使用 CPU 0.01 秒。

这也缩写为“100m”。

![CPU 限制中的配额和周期](https://miro.medium.com/v2/0*F3yHnz65qLoEZvSD.png)

**如果你的容器有硬限制并且想要更多的 CPU，它必须等待下一个周期。**

您的进程受到限制。

![一个被 CPU 限制的进程](https://miro.medium.com/v2/0*t4-iWQLLq8T_jySr.png)

*那么您应该在 Pod 中如何设置 CPU requests 和 limits？*

一种简单（但不准确）的方法是将最小的 CPU 单元计算为：

```
REQUEST = NODE_CORES * 1000 / MAX_NUM_PODS_PER_NODE
```

对于 1 个 vCPU 节点和 10 个 Pod ，最小单元就是 `1 * 1000 / 10 = 100Mi`。

**将最小单位或其乘数分配给您的容器。**

![将 CPU 请求分配给 Pod 和容器](https://miro.medium.com/v2/0*QQ1lYpqKlNe18BMP.png)

例如，如果您不知道 Pod A 需要多少 CPU，但您确定它是 Pod B 的两倍，您可以设置：

- Request A：1 个单元
- Request B：2 个单位

如果容器使用 100% CPU，它们将根据它们的权重 (1:2) 重新分配 CPU。

![两个 Pod 竞争 CPU 资源](https://miro.medium.com/v2/0*-EryolQMqg8TRjld.png)

**更好的方法是监控应用程序并得出平均 CPU 利用率。**

您可以使用现有的监控基础设施来完成此操作，或者使用 [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) 来监视并报告平均请求值。

*你应该如何设置 limits？*

1. 您的应用可能已经有“硬性”限制。（例如单线程的应用即使分配了 2 个核，也最多只使用 1 个核）。
2. 你可以设置：limit = 99th 分位数 + 30–50%。

您应该分析应用程序（或使用 VPA）以获得更详细的答案。

![CPU 的第 99 百分位数](https://miro.medium.com/v2/0*OaOhsVD74uRv7Q1R.png)

*您应该始终设置 CPU requests 吗？*

**绝对没错。**

这是 Kubernetes 中的标准良好实践，可帮助调度器更有效地分配 pod。

*您应该始终设置 CPU limits 吗？*

这有点争议，但总的来说，我是这么认为的。

你可以进行更深入的了解：[https://dnastacio.medium.com/why-you-should-keep-using-cpu-limits-on-kubernetes-60c4e50dfc61](https://dnastacio.medium.com/why-you-should-keep-using-cpu-limits-on-kubernetes-60c4e50dfc61)

其它的一些相关链接：

- https://learnk8s.io/setting-cpu-memory-limits-requests
- https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b
- https://nodramadevops.com/2019/10/docker-cpu-resource-limits


