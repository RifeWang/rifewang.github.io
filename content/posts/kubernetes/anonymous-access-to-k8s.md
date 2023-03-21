+++
draft = false
date = 2023-03-21T17:12:37+08:00
title = "谈谈 Kubernetes 的匿名访问"
description = "谈谈 Kubernetes 的匿名访问"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

---

*文本翻译自: https://raesene.github.io/blog/2023/03/18/lets-talk-about-anonymous-access-to-Kubernetes/*

---

本周有一些关于 [Dero Cryptojacking operation](https://www.bleepingcomputer.com/news/security/first-known-dero-cryptojacking-operation-seen-targeting-kubernetes/) 的文章，其中关于攻击者所实施的细节之一引起了我的注意。有人提到他们正在攻击允许匿名访问 Kubernetes API 的集群。究竟如何以及为什么可以匿名访问 Kubernetes 是一个有趣的话题，涉及几个不同的领域，所以我想我会写一些关于它的内容。

匿名访问如何工作？
---------

集群是否可以进行匿名访问由 [kube-apiserver](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/) 组件的标志 `--anonymous-auth` 控制，其默认为`true`，因此如果您在传递给服务器的参数列表中没有看到它，那么匿名访问将被启用。

然而，仅凭此项设置并不能给攻击者提供访问集群的很多权限，因为它只涵盖了请求在被处理之前通过的三个步骤之一（Authentication -> Authorization -> Admission Control ）。正如 Kubernetes [控制访问](https://kubernetes.io/docs/concepts/security/controlling-access/) 的文档中所示，在身份认证后，请求还必须经过授权和准入控制（认证 -> 授权 -> 准入控制）。


授权和匿名访问
-------

因此下一步是请求需要匹配授权策略（通常是 RBAC，但也可能是其他策略）。当然，为了做到这一点，请求必须分配一个身份标识，这个时候 `system:anonymous` 和 `system:unauthenticated` 权限组就派上了用场。这些身份标识被分配给任何没有有效身份验证令牌的请求，并用于匹配授权政策。

您可以通过查看Kubeadm 集群上的 `system:public-info-viewer` `clusterrolebinding` 来了解类似的工作原理。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
    annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
    labels:
    kubernetes.io/bootstrapping: rbac-defaults
    name: system:public-info-viewer
roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: system:public-info-viewer
subjects:
- apiGroup: rbac.authorization.k8s.io
    kind: Group
    name: system:authenticated
- apiGroup: rbac.authorization.k8s.io
    kind: Group
    name: system:unauthenticated
```

匿名访问有多常见
------

现在我们知道了匿名访问是如何工作的，问题就变成了“这有多常见？”。答案是大多数主要发行版都会默认启用匿名访问，并通常通过 `system:public-info-viewer` `clusterrole`提供一些对 `/version` 以及其他几个端点的访问权限。

要了解这适用于多少集群，我们可以使用 [censys](https://search.censys.io/) 或 [shodan](https://www.shodan.io/) 来查找返回版本信息的集群。例如，这个 [censys 查询](https://search.censys.io/search?resource=hosts&q=services.kubernetes.version_info.git_version%3D%22*%22) 显示返回版本信息的主机超过一百万，因此我们可以说这是一个相当常见的配置。

一个更严重的也更符合 dero 文章中提出的要点是，这些集群中有多少允许攻击者在其中创建工作负载。虽然您无法从 Censys 获得确切的信息，但它确实有一个显示集群的查询，允许匿名用户枚举集群中的 pod，在撰写本文时显示 [302 个集群节点](https://search.censys.io/search?resource=hosts&q=services.kubernetes.pod_names%3D%22*%22)。我猜其中一些/大部分是蜜罐，但也可能有几个就是高风险的易受攻击的集群。

禁用匿名访问
------

在非托管集群（例如 Rancher、Kubespray、Kubeadm）上，您可以通过将标志 `--anonymous-auth=false` 传递给 `kube-apiserver` 组件来禁用匿名访问。在托管集群（例如 EKS、GKE、AKS）上，您不能这样做，但是您可以删除任何允许匿名用户执行操作的 RBAC 规则。例如，在 Kubeadm 集群上，您可以删除`system:public-info-viewer` `clusterrolebinding`和`system:public-info-viewer` `clusterrole`，以有效阻止匿名用户从集群获取信息。

当然，如果您有任何依赖这些端点的应用程序（例如健康检查），它们就会中断，因此测试您对集群所做的任何更改非常重要。这里的一种选择是查看您的审计日志，看看是否有任何匿名请求向 API 服务器发出。

结论
--

允许某种级别的匿名访问是 Kubernetes 中的常见默认设置。这本身并不是一个很大的安全问题，但它确实意味着在许多配置中，阻止攻击者破坏您的集群的唯一方法是 RBAC 规则，因此一个错误可能会导致重大问题，尤其是当您的集群暴露在互联网上时。

