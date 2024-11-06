+++
draft = false
date = 2024-11-06T19:30:34+08:00
title = "Kubernetes Extended Resource 扩展资源使用简介"
description = "Kubernetes Extended Resource 扩展资源使用简介"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

Kubernetes 除了提供基于 CPU 和内存的传统计算资源调度外，还支持自定义的 `Extended Resource` 扩展资源，以便调度和管理其它各种类型的资源。

## Extended Resource

`Extended Resource` 扩展资源的创建和使用过程如下图所示：

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/kubelet/k8s-kubelet-extended-resources.drawio.png)

1. 定义资源：用户或管理员通过 Kubernetes API 向特定的节点新增自定义的扩展资源（即修改节点的 status 信息）。例如，图中新增的资源 `example.com/dongle`，拥有 4 个单位。

2. 节点同步资源：`kubelet` 会在定时同步节点状态时，通过 `GET /api/v1/nodes/<nodeName>` 请求从 `kube-apiserver` 获取节点的资源信息，从而同步到该扩展资源的数据。

3. 调度和使用：用户创建一个请求 `example.com/dongle` 扩展资源的 Pod，`Scheduler` 会将该 Pod 调度到满足条件的节点上。随后该节点上的 `kubelet` 负责创建和启动 Pod。

一旦 `Extended Resource` 被添加到节点上，Kubernetes 将自动管理该资源在 Pod 调度和创建过程中的分配和使用情况，无需用户手动干预。

然而，**节点的 `status.allocatable` 中的扩展资源信息不会随着 Pod 的创建和删除实时更新**。例如，假设某个 Pod 使用了 `example.com/dongle` 的 3 个单位资源，但在查看节点状态时，`status.allocatable` 中的 `example.com/dongle` 仍然显示为 4 个单位。这种非实时更新设计的目的是为了减少对 `kube-apiserver` 和 `etcd` 的频繁更新，避免增加系统负载。

此时你可能会疑惑，如果可分配资源信息不实时更新，资源调度不会有问题吗？答案是不会，因为 `scheduler` 和 `kubelet` 都会在各自的进程中记录并追踪资源的使用情况。

## 扩展资源的局限性

虽然 `Extended Resource` 的设计看起来简单，似乎只需通过 API 为节点新增资源即可，但其应用往往伴随着以下挑战：
- 资源的配置和使用：例如，定义 GPU 作为 `Extended Resource` 后，Scheduler 可以正常完成调度，但容器实际使用 GPU 时，还需要适配驱动和运行时环境（如 NVIDIA 容器运行时）。这意味着扩展资源仅声明和调度是不够的，还需要配置支持相关硬件的容器环境。
- 自动化管理需求：手动为每个节点逐一添加或修改扩展资源并不实际，特别是在有多个节点或复杂硬件需求的场景中。依赖人工管理难以保证扩展资源的一致性和效率。

因此，在实际应用中，为了更好地管理和使用扩展资源，通常会借助 `Device Plugin` 和 `Operator`。`Device Plugin` 是 Kubernetes 提供的设备管理机制，通过它可以自动检测和管理扩展资源，如 GPU 等特殊硬件。`Operator` 则进一步简化了资源部署和配置管理流程，自动执行资源的配置和调度。

想深入了解这方面内容，可以参考我之前的文章 *《Kubernetes GPU 调度和 Device Plugin、CDI、NFD、GPU Operator 概述》*。

## 总结

`Extended Resource` 为 Kubernetes 提供了灵活的扩展能力，使集群能够支持更多样化的资源类型。然而在实践中，它仅解决了资源声明和调度的一部分问题。


(我是凌虚，关注我，无广告，专注技术，不煽动情绪，欢迎与我交流)

---

参考资料：

- *https://kubernetes.io/docs/tasks/administer-cluster/extended-resource-node/*
- *https://kubernetes.io/docs/tasks/configure-pod-container/extended-resource/*