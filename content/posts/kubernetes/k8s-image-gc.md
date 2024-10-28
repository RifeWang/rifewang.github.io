+++
draft = false
date = 2024-10-28T23:10:53+08:00
title = "Kubernetes Node 节点上的镜像管理"
description = "Kubernetes Node 节点上的镜像管理"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

本文将详细介绍 Kubernetes 如何管理节点上的镜像。

## 拉取镜像

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/kubelet/kubelet-image-gc.drawio.png)

`Kubelet` 通过 gRPC 协议与 `CRI` 组件（如 `containerd`、`cri-o`）进行交互。在创建新 Pod 时，`kubelet` 调用 gRPC 的 `ImageService.PullImage` 方法，由 `CRI` 组件将镜像下载到节点上。镜像在磁盘上的组织和管理由 `CRI` 组件负责，不同的 CRI 组件存在差异。

`Containerd` 的配置文件默认为 `/etc/containerd/config.toml`，其中的配置项 `root=/var/lib/containerd/` 指定了存储路径（这是一个根目录，内部会再细分子目录）。

`Cri-o` 存储相关的配置文件则是 `/etc/containers/storage.conf`，其中的配置项 `root=/var/lib/containers/storage` 指定存储路径，另外存在一个配置项 `imagestore` 可以单独指定镜像的存储位置。

## Image GC

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/kubelet/kubelet-image-gc.drawio.png)

`Kubelet` 通过垃圾回收策略来管理镜像的删除，相关的配置项包括：
- `imageMinimumGCAge`：默认值为 2 分钟，表示镜像的最小存在时间，避免在垃圾回收时删除过于新的镜像。
- `imageMaximumGCAge`：默认值为 0 秒（表示不开启此特性），表示镜像的最大存在时间（此特性在 v1.30 中引入）。如果镜像自上次使用以来的时间超过此值，该镜像将被删除。
- `imageGCHighThresholdPercent`：默认值是 85，当磁盘的使用率大于等于 85%，触发镜像的垃圾回收。
- `imageGCLowThresholdPercent`：默认值是 80，镜像垃圾回收过程中，一旦磁盘使用率小于等于 80%，则停止删除镜像。

`Kubelet` 每隔 5 分钟（这个值在代码中写死了），不停的循环以下对镜像的 GC 过程：
1. `imagesInEvictionOrder`：kubelet 向 CRI 发起 `ImageService.ListImages`、`RuntimeService.ListPodSandbox`、`RuntimeService.ListContainers` 三个 gRPC 请求，以获取该节点上所有镜像的使用信息，并按上次使用时间排序。
2. `freeOldImages`：如果镜像从上一次使用至今的时间间隔超过了 `imageMaximumGCAge`，kubelet 会向 CRI 发起 `ImageService.RemoveImage` 请求，删除这些镜像。
3. `ImageFsStats`：kubelet 发 CRI 发起 `ImageService.ImageFsInfo` 请求（cri-o 依赖于 cadvisor），以获取磁盘使用情况。
4. `freeSpace`：当磁盘使用率大于等于 `imageGCHighThresholdPercent` 高水位线时，触发镜像的删除行为，镜像会按照时间顺序从最久未被使用开始，一个接一个被删除，每删除完一个镜像都会判断一次磁盘使用率是否小于等于 `imageGCLowThresholdPercent` 低水位线，一旦降至低水位线，则停止删除镜像。

## 总结

在 node 节点上，镜像由 `CRI` 组件（如 `containerd`、`cri-o`）管理。kubelet 每 5 分钟进行一次镜像的垃圾回收。如果配置了 `imageMaximumGCAge` 镜像的最大存在时间，则不管磁盘使用率是否达到高水位线，都会删除符合条件的镜像。之后，镜像的删除将依据 `imageGCHighThresholdPercent`、`imageGCLowThresholdPercent` 高低水位线的配置来触发，具体删除操作由 CRI 组件执行，而 kubelet 仅负责发起 gRPC 调用。