+++
draft = false
date = 2024-09-07T14:46:34+08:00
title = "Kubernetes 之 kubelet 与 CRI、CNI 的交互过程"
description = "Kubernetes 之 kubelet 与 CRI、CNI 的交互过程"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

## 序言

当一个新的 Pod 被提交创建之后，`Kubelet`、`CRI`、`CNI` 这三个组件之间进行了哪些交互？

## Kubelet -> CRI -> CNI

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/kubelet-cri-cni-sequence.png)

如上图所示：
1. `Kubelet` 从 kube-api-server 处监听到有新的 pod 被调度到了自己的节点且需要创建。
2. `Kubelet` 创建 sandbox 并配置好 Pod 的环境，其中包括：
    - `Kubelet` 通过 gRPC 调用 `CRI` 组件创建 sandbox。
    - `CRI` 通过命令行调用 `CNI` 设置 pod 的网络。
3. `Kubelet` 创建 container 阶段：
    - 调用 `CRI` 拉取镜像。
    - 调用 `CRI` 创建 container。
    - 调用 `CRI` 启动 container。

注意：
- 先创建一个 sandbox 就是为了先设置好 pod 的网络命名空间，因为用户容器可能面临启动失败等各种异常情况。
- 从 kubernetes v1.24 版本开始，`Kubelet` 不再管理 `CNI`，而是由 `CRI` 负责调用 `CNI`。

`CRI` 的具体实现有 `containerd`，`cri-o`，`docker` 等几种。

`containerd` 的架构图如下：
![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/containerd-architecture.png)

`cri-o` 的架构图如下：
![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/crio-architecture.png)

从图中也可以看到 `CNI` 由 `CRI` 负责调用。


再进一步看看细节一点的流程：

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/kubelet-cri-cni-flow.png)

`Kubelet` 在 SyncPod 阶段同步 Pod：
1. 创建 sandbox，其中会进行两个 gRPC 方法的调用：
    - `Kubelet` 调用 `RuntimeService.RunPodSandbox`，`CRI` 开始创建 pod 的各种命名空间（隔离环境），然后再拉起 sandbox 容器，接着 `CRI` 调用 `CNI` 设置 pod 网络环境，包括分配 pod IP 地址。
    - `Kubelet` 调用 `RuntimeService.PodSandboxStatus` 确认 pod sandbox 状态。
2. 进行容器创建阶段（按照 ephemeral、init、normal 的顺序），此时涉及三个 gRPC 调用：
    - `Kubelet` 调用 `ImageService.PullImage` 由 `CRI` 拉取镜像。
    - `Kubelet` 调用 `RuntimeService.CreateContainer` 由 `CRI` 创建容器，这里主要是配置好环境。
    - `Kubelet` 调用 `RuntimeService.StartContainer` 由 `CRI` 启动容器，至此容器才跑起来。

新建 Pod 时 `Kubelet` 与 `CRI`、`CNI` 之间的交互大致如上所述。

## CRI & CNI

Kubernetes 通过定义标准接口的方式，与下层具体实现进行了解耦。其中 `CRI` 是容器运行时接口，通信协议使用的是 `gRPC`；`CNI` 是容器网络接口，交互方式则是命令行二进制可执行文件。

`CRI` 的 gRPC proto 如下，定义了 `RuntimeService` 和 `ImageService` 两种服务以及多种方法：

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/CRI-v1-api.png)

`CNI` 则只有六种操作：

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/CNI-proto.png)


## 总结

其实不管是新建 Pod 还是其它场景，`Kubelet`、`CRI`、`CNI` 的调用过程都是 `Kubelet` 调用 `CRI`，`CRI` 调用 `CNI`。


---

参考资料：

- *https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/*
- *https://github.com/kubernetes/cri-api/blob/master/pkg/apis/runtime/v1/api.proto*
- *https://github.com/containernetworking/cni/blob/main/SPEC.md*