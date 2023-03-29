+++
draft = false
date = 2023-03-28T18:47:47+08:00
title = "Kubernetes Gateway API 介绍"
description = "Kubernetes Gateway API 介绍"
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
disableComments = true
+++

---

*文本翻译自: https://medium.com/geekculture/kubernetes-gateway-api-the-intro-you-need-to-read-80965f7acd82*

---

您以前听说过 [SIG-NETWORK](https://github.com/kubernetes/community/tree/master/sig-network) 的 Kubernetes Gateway API 吗？好吧，可能你们中的大多数人都是第一次遇到这个话题。尽管如此，无论您是第一次听说还是已经以某种方式使用过它，本博客的目的都是为您提供一个基本的和高度的概述来理解这个主题。

![](https://miro.medium.com/v2/1*xDcDFu4j7NrKstOD3vqPLg.png)

从了解对 Kubernetes Gateway API 的需求到探索其用例，本博客旨在为您提供全面的指南，介绍您需要了解的有关 Kubernetes 中服务网络革命性工具的所有信息。

**因此，在此博客中，我们将涵盖以下主题：**

- Ingress 资源的约束和限制
- 四层路由如何暴露服务？
- Kubernetes SIG-NETWORK 是啥，是什么推动了他们的目标？
- SIG-NETWORK 开启 Kubernetes Gateway API 项目的原因是什么？
- 全面了解 Kubernetes Gateway API（第二部分，后续文章会介绍）

## Ingress 资源的约束和限制

要了解对 Kubernetes Gateway API 的需求，我们需要了解 ingress 资源，该资源于 2015 年推出，并在 Kubernetes 1.19 中成为了稳定的 API。ingress 资源根据请求 host、path 或两者的组合管理对适当 Kubernetes 服务的外部流量访问。Ingress 资源有助于在同一个负载均衡器下公开多个服务，提供负载均衡、SSL 终止等。

虽然 ingress 资源是第 7 层路由（HTTP、HTTPS）的有效选择，但当它需要为第 4 层流量（TCP、UDP）提供服务时，它就显得不够用了，后者常用于公开诸如数据库、消息代理等服务.

## 四层路由如何暴露服务？

要为数据库、消息代理等提供 L4 流量，您有多种选择。

一种选择是使用 kubectl port-forward 供开发人员进行内部访问，以保持较低的云成本。

另一种选择是使用 LoadBalancer 类型的服务来对其他服务、开发人员或用户进行外部访问，这是第 4 层路由的简单解决方案。

此外，您可以使用 Kong 或 Istio 等服务网格提供商，它们提供通过单个负载均衡器 IP 地址路由第 4 层和第 7 层流量的功能。

然而，值得注意的是，Istio 和 Kong 等服务网格提供商拥有自己的专有 API，导致在服务第 4 层和第 7 层流量方面缺乏标准化。

## Kubernetes SIG-NETWORK 是什么？

SIG-NETWORK 是 Kubernetes 社区中的一个子社区，专注于 Kubernetes 中的网络。SIG-NETWORK 负责开发、维护和支持 Kubernetes 平台的网络相关组件。

SIG-NETWORK 旨在确保 Kubernetes 的网络功能稳健、可扩展，并能够满足各种用例的需求。

## SIG-NETWORK 开启 Kubernetes Gateway API 项目的原因是什么？

目前，Kubernetes 空间中的解决方案提供了自己的网关解决方案和特定的 API，允许它们将第 4 层和第 7 层流量路由到 Kubernetes 服务。

SIG-NETWORK 社区已经启动了 Kubernetes Gateway API，为四层和七层路由流量创建统一的 API 资源和标准。Kubernetes Gateway API 为 Kong 和 Istio 等不同的第三方解决方案提供了一个通用的接口。

虽然该项目目前处于测试版，但该领域的主要参与者已经采用。

[*Youtube video by kong on API Gateway and demo with their controller*](https://konghq.com/resources/webinar/understanding-the-new-kubernetes-gateway-api-vs-ingress)

[*Blog from Istio regards API Gateway*](https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/)


## 结语

总之，Kubernetes Gateway API 正在填补 Kubernetes Ingress 资源留下的标准化空白。尽管处于测试阶段，但它已经得到了 Istio 和 Kong 等知名工具的支持。这证明了 Kubernetes Gateway API 有潜力成为在 Kubernetes 环境中管理网络流量的广泛采用的解决方案。
