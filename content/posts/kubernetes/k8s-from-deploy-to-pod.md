+++
draft = false
date = 2023-12-23T18:26:15+08:00
title = "Kubernetes 从提交 deployment 到 pod 运行的全过程"
description = "Kubernetes 从提交 deployment 到 pod 运行的全过程"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++


当用户向 `Kubernetes` 提交了一个创建 `deployment` 的请求后，`Kubernetes` 从接收请求直至创建对应的 `pod` 运行这整个过程中都发生了什么呢？

## kubernetes 架构简述

在搞清楚从 `deployment` 提交到 `pod` 运行整个过程之前，我们有先来看看 `Kubernetes` 的集群架构：

![](/images/k8s/kubernetes-cluster-architecture.svg)

上图与下图相同：

![](/images/k8s/k8s-architecture.png)

如图所示，k8s 集群分为 `control plane` 控制平面和 `node` 节点。

`control plane` 控制平面（也称之为主节点）主要包含以下组件：
- `kube-api-server`: 顾名思义，负责处理所有 api，包括客户端以及集群内部组件的请求。
- `etcd`: 分布式持久化存储、事件订阅通知。只有 `kube-api-server` 直接操作 `etcd`，其它所有组件都是与 `kube-api-server` 进行相互。
- `scheduler`: 处理 pod 的调度，将 pod 绑定到具体的 node 节点。
- `controller manager`: 控制器，处理各种资源对象。
- `cloud controller manager`: 对接云服务商的控制器。

`node` 节点，专门部署用户的应用程序（与控制平面隔离，避免影响到 k8s 的核心组件），主要包含以下组件：
- `kubelet`: 管理节点上的 pod 以及状态检查和上报。
- `kube-proxy`: 进行流量的路由转发（目前是通过操作节点的 iptables 或者 ipvs 实现）。
- `CRI`: 容器运行时接口。

## 从 Deployment 到 Pod

从 `Deployment` 到 `Pod` 的整个过程如下图所示：

![](/images/k8s/from-deploy-to-pod/k8s-from-deploy-to-pod.png)


### 1. 请求发送到 `kube-api-server`

请求发送到 `kube-api-server`，然后会进行认证、鉴权、变更、校验等一系列过程，最后将 deployment 的数据持久化存储至 `etcd`。

![](/images/k8s/admission-controller-phases.png)

在这个过程我们可以通过 mutation admission 的 webhook 自主地对资源对象进行任意的变更，比如注入 sidecar 等等。

### 2. controller manager 处理

`controller manager` 组件针对不同的资源对象有不同的处理部分。

针对 `Deployment`，由于其并不直接管理 `Pod`，而是 `Deployment` 管理 `ReplicaSet`，`ReplicaSet` 再管理 `Pod`：

![](/images/k8s/from-deploy-to-pod/deploy-replicaset-pod.png)

因此其中涉及到 `controller manager` 中的两个部分：
- `deployment controller`
- `replicaset controller`

(1) 先是 `deployment controller` 监听到 `deployment` 的创建事件，然后进行相关的处理，最后创建 `replicaset`。

(2) 然后 `replicaset controller` 监听到 `replicaset` 的创建事件，进行相关处理后，最后创建 `pod`。

### 3. scheduler 调度

`scheduler` 接受到 pod 需要调度的事件后，进行一系列调度逻辑处理，最后选择一个合适的 node 节点，将 pod 绑定到这个节点上（所谓的节点调度在这里只是修改 pod 数据，对其中的 nodeName 进行赋值）。

具体的调度算法比较复杂，涉及强制性调度、亲和与反亲和、污点和容忍、以及硬件资源计算、优先级等等，本文不做展开。

### 4. 节点 kubelet 处理

调度完成后，`pod` 被绑定的 node 节点上的 `kubelet` 同样通过 `kube-api-server` 会接受到相应的事件，然后 `kubelet` 会进行 `pod` 的创建。

在这个过程中 `kubelet` 会分别调用 `CRI`、`CNI`、`CSI`：

- `CRI`（Container Runtime Interface）: 容器运行时接口，`CRI` 插件负责执行拉取镜像、创建、删除容器等操作。`CRI` 的几种常用插件：
    - `containerd`
    - `CRI-O`
    - `Docker Engine`

- `CNI`（Container Network Interface）: 容器网络接口，`CNI` 插件负责给 pod 分配 IP 地址，确保 pod 能够与集群内的其它 pod 进行通信。`CNI` 的几种常用插件：
    - `Cilium`
    - `Calico`

- `CSI`（Container Storage Interface）: 容器存储接口，`CSI` 插件负责与外部存储提供者通信，执行卷的附加、挂载等操作。

所谓的接口其实只是定义了通信的规范或者标准（使用的是 `grpc` 协议），具体的实现则是交给了插件。


至此，Kubernetes 从创建 deployment 到 pod 运行的全过程就是这样了。

![](/images/k8s/from-deploy-to-pod/k8s-from-deploy-to-pod.png)


---

参考资料：

- *https://kubernetes.io/docs/concepts/architecture/*
- *https://kubernetes.io/docs/concepts/scheduling-eviction/*
- *https://kubernetes.io/docs/setup/production-environment/container-runtimes/*
- *https://kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/*
