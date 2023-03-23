+++
draft = false
date = 2023-03-22T17:47:12+08:00
title = "使用 Kubernetes API"
description = "使用 Kubernetes API 可以让您控制集群的各个方面。"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

---
*文本翻译自: https://itnext.io/working-with-the-kubernetes-api-587bc5941992*

---

![](/images/k8s/0_Th5qU5L4AOg9QC5Q.png)

Kubernetes 公开了一个强大的 API，可让您控制集群的各个方面。

大多数时候，它隐藏在 kubectl 后面，但没有人会阻止您直接使用它。

在本文中，您将学习如何使用 `curl` 或者您喜欢的编程语言向 Kubernetes API 发出请求。

但首先，让我们回顾一下 Kubernetes API 的工作原理。

当您键入命令时，kubectl：

- 客户端校验请求。
- 在文件上生成 YAML（例如`kubectl run`）。
- 构造运行时对象。

![Client side validation in kubectl](/images/k8s/0_v3ORz4nWAv-w5rKC.png)

此时，kubectl 还没有向集群发出任何请求。

下一步，它查询当前的 API 服务器并发现所有可用的 API 端点。

![OpenAPI descovery in kubectl](/images/k8s/0_wrHSiajOlLUZ9xX_.png)

**最后，kubectl 使用运行时对象和端点来协商正确的 API 调用。**

如果您的资源是 Pod，kubectl 会读取 `apiVersion` 和 `kind` 字段并确保它们在集群中可用和受支持。

然后它发送请求。

![API negotiation in kubectl](/images/k8s/0_doUoRiRC9UzsJOse.png)

**理解在 Kubernetes 中 API 是分组的这很重要的。**

为了进一步隔离多个版本，资源被版本化。

![Kubernetes API groups, versions and resources](/images/k8s/0_SZ2rP9HcCRWGKuoS.png)

现在您已经掌握了基础知识，让我们来看一个示例。

您可以使用 `kubectl proxy` 启动到 API 服务器的本地隧道。

*但是如何检索所有 deployments 呢？*

![Proxy to kubernetes API](/images/k8s/0_G-MtvhjbKRni6t58.png)

Deployments 属于 `apps` 组并且有一个 `v1` 版本。

您可以列出它们：

```sh
curl localhost:8001/apis/apps/v1/namespaces/{namespace}/deployments
```

![List deployments](/images/k8s/0_7SatHg1pCYATIash.png)

*列出所有正在运行的 pod 怎么样？*

Pod 属于 `""`（空）组并且有一个 `v1` 版本。

您可以列出它们：

```sh
curl localhost:8001/api/v1/namespaces/{namespace}/pods
```

![List pods](/images/k8s/0_s8LRlXObi7rHM2x6.png)

*group 为空看起来有点奇怪——还有更多例外吗？*

**好吧，现实是有一种更简单的方法来构建 URL。**

我通常使用 [Kubernetes API 参考文档](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.26/#list-pod-v1-core)，因为路径都整齐地列出了。

让我们看另一个示例，但这次是在 API 参考的帮助下。

*如果你想收到 pod 更改的通知怎么办？*

在 API 中称为 `watch`，命令为：

```sh
GET /api/v1/watch/namespaces/{namespace}/pods/{name}
```

![Watch pods](/images/k8s/0_lveeFi04s0mJEhMv.png)

*太好了，但这一切有什么意义呢？*

**直接访问 API 允许您构建脚本来自动执行任务。**

或者您可以构建自己的 kubernetes 扩展。

我来给你展示。

[这是一个约 130 行 Javascript 的小型 kubernetes 仪表板。](https://github.com/learnk8s/k8bit)

![](/images/k8s/0_6BJSpJC5oha9iK__.gif)

它调用了 2 个 API:

1. 列出所有 pod
2. watch pod 的变化

其余代码用于对节点进行分组和显示。

**在 Kubernetes 中，将列出和更新资源结合起来非常普遍，以至于它成为一种称为 shared informer 的模式。**

Javascript/Typescript API 有一个很好的 [shared informer](https://github.com/kubernetes-client/javascript/blob/master/examples/typescript/informer/informer.ts) 的例子.

但它只是 2 个 GET 请求（和一些缓存）的奇特名称。

**API 不止于读取资源。**

您还可以创建新资源并修改现有资源。

例如，您可以修改部署的副本：

```sh
PATCH /apis/apps/v1/namespaces/{namespace}/deployments/{name}
```

![使用 curl 修补 Kubernetes 部署](/images/k8s/0_hSWjviRpVrHk8fJF.png)

为了进行实验，我建造了一些非常规的东西。

[xlskubectl 是我尝试使用 Excel/Google 表格控制 kubernetes 集群。](https://github.com/learnk8s/xlskubectl)

![Sheetops — 使用电子表格控制 Kubernetes](/images/k8s/0_GZDAo6pMhT8Ru5wX.gif)

该代码与上述 Javascript 代码非常相似：

1. 它使用 shared informer
2. 它轮询 google sheets 的更新
3. 它将所有内容呈现为单元格

这个示例是一个好主意吗？ **可能并不是。**

希望它能帮助您实现直接使用 Kubernetes API 的潜力。

**这些代码都不是用 Go 编写的——您可以使用任何编程语言去调用 Kubernetes API。**


