+++
draft = false
date = 2024-10-20T12:56:58+08:00
title = "Kubernetes GPU 调度和 Device Plugin、CDI、NFD、GPU Operator 概述"
description = "Kubernetes GPU 调度和 Device Plugin、CDI、NFD、GPU Operator 概述"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

随着人工智能（`AI`）和机器学习（`ML`）的快速发展，`GPU` 已成为 Kubernetes 中不可或缺的资源。然而，Kubernetes 最初设计的调度机制主要针对 CPU 和内存等常规资源，未对异构硬件（如 GPU）提供原生支持。

为了高效管理和调度 `GPU` 以及其他硬件资源，Kubernetes 引入了一系列扩展机制，包括 `Device Plugin`、`Container Device Interface (CDI)`、`Node Feature Discovery (NFD)` 和 `GPU Operator`。

本文将以 GPU 调度为例，概述这些扩展机制的工作原理和应用。


## Device Plugin

`Device Plugin` 是 Kubernetes 用于管理特殊硬件资源的插件机制。它通过将设备（如 `GPU`、`FPGA`、`NIC`、`InfiniBand` 等）抽象为 Kubernetes 可识别的资源，实现设备的发现、分配和调度。

`Device Plugin API` 使用 gRPC 协议，定义了 kubelet 与设备插件之间的交互方式，包含两个 service：
- `Registration` service：设备插件通过 `Register` 方法向 kubelet 注册自己。
- `DevicePlugin` service 包括五个方法：
    - `GetDevicePluginOptions`：查询设备插件的可选项。
    - `ListAndWatch`：kubelet 通过此接口监听设备状态的变化。
    - `GetPreferredAllocation`：kubelet 可能会调用该接口来询问设备插件最优的分配方案（如多 GPU 任务如何选择最佳的 GPU 组合）。
    - `Allocate`：kubelet 调用此接口为容器分配设备。
    - `PreStartContainer`：可以调用此接口让设备插件在容器启动前做一些操作。

以使用 NVIDIA GPU 为例：

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/k8s-GPU.drawio.png)

1. 设备插件通过 `Register` 向 kubelet 注册自己。kubelet 通过 `ListAndWatch` 监听设备状态，并将设备信息上报给 kube-apiserver，随后控制平面才能知道节点的 GPU 资源情况。
2. 用户创建了一个使用 GPU 资源的 pod，调度器将 pod 分配到有空闲 GPU 资源的节点上。
3. 节点上的 kubelet 监听到有 pod 被调度至此，开始后续容器的创建工作：
    - kubelet 通过 `Allocate` 接口请求设备插件分配硬件资源。
    - kubelet 通过 CRI gRPC 接口与容器运行时组件（如 `containerd`、`cri-o`）进行交互。
    - CRI 组件（如 `containerd`、`cri-o`）调用更底层的运行时，一般是 `runc` 或 `kata-container`，但此时必须要使用 `nvidia-container-runtime`，由 `nvidia-container-runtime` 与 GPU 驱动交互，然后才能使用 GPU 资源。`nvidia-container-runtime` 其实就是在 `runc` 的基础上注入了 NVIDIA 的特定代码。

这个流程完整展示了 Kubernetes 如何管理和调度 GPU 资源。


## Container Device Interface

由于缺乏第三方设备标准，设备供应商通常必须为不同的运行时编写和维护多个插件，甚至直接在运行时中注入特定于供应商的代码（如 `nvidia-container-runtime` 在 `runc` 的基础上魔改）。

因此，社区提出了 `Container Device Interface（CDI）`，希望解耦并标准化容器运行时与设备插件的交互。

`CDI` 是容器运行时支持第三方设备的规范。它定义了设备的描述文件（JSON 格式），该文件用于描述特定设备的属性、环境变量、挂载点等等信息。

`CDI` 的工作流大致如下：
1. 设备插件或供应商提供 `CDI` 描述文件。
2. 设备名称被传递给容器运行时。
3. 容器运行时按照 `CDI` 文件内容更新容器配置。

`CDI` 并不是对 `Device Plugin` 的替代，而是协同工作。容器运行时就像使用 `CNI` 一样使用 `CDI`。

(感兴趣可以阅读我的往期文章《Kubernetes 之 kubelet 与 CRI、CNI 的交互过程》)


## Node Feature Discovery

在某些场景中，应用可能需要节点具有特定的硬件特性。例如：需要使用特定的 CPU 指令集（如 AVX、SSE）来加速某些计算工作；依赖硬件加速器（如 GPU、FPGA）；需要在特定的硬件架构（如 ARM、x86）上运行。

默认的 Kubernetes 调度器对这些特性并不了解，因此无法做出相应的调度决策。`Node Feature Discovery` 旨在通过自动化检测和标签机制填补这一空白。

`Node Feature Discovery (NFD)` 的工作流程如下：
1. 节点特性检测：`NFD` 以 `DaemonSet` 的形式运行在每个节点上，自动检测节点的硬件和软件特性。
2. 特性标签化：检测到的特性会以 `Labels` / `Annotations` 的形式添加到节点上。
3. 基于节点标签调度 Pod：用户可以利用已有的标签选择（如 nodeSelector 或 nodeAffinity）调度 Pod。

`NFD` 只是负责发现节点的特性，具体如何利用这些标签或注解则由用户决定。


## GPU Operator

在上文 `Device Plugin` 一节中可以看到，为了使用 GPU，需要 `GPU driver`、`device plugin`、`nvidia-container-runtime`、以及监控等等工具。手动管理这些组件非常复杂、容易出错。`GPU Operator` 的目的就是自动化这一过程，通过 `Operator` 模式统一管理和配置 GPU 相关的组件。

`Operator` 也是 Kubernetes 的一种扩展机制，用户可以自定义资源和控制器，从而覆盖更多的使用场景。


## 总结

通过 `Device Plugin`、`CDI`、`NFD` 和 `Operator` 机制，Kubernetes 实现了对 GPU 等特殊硬件资源的自动化管理与高效调度。然而，特定厂商的硬件仍然存在差异化的部署与配置，使用时需要额外关注。


(关注我，无广告，专注技术，不煽动情绪，也欢迎与我交流)

---

参考资料：

- *https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/*
- *https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/3573-device-plugin*
- *https://github.com/NVIDIA/k8s-device-plugin*
- *https://github.com/cncf-tags/container-device-interface*
- *https://github.com/kubernetes-sigs/node-feature-discovery*
- *https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/overview.html*