+++
draft = false
date = 2022-03-13T00:00:00+08:00
title = "加速 Kubernetes 镜像拉取"
description = ""
slug = ""
authors = []
tags = []
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

# 加速 Kubernetes 镜像拉取

Kubernetes pod 启动时会拉取用户指定的镜像，一旦这个过程耗时太久就会导致 pod 长时间处于 pending 的状态，从而无法快速提供服务。

镜像拉取的过程参考下图所示：

![k8s image pull](https://raw.githubusercontent.com/RifeWang/images/master/k8s/k8s-image-pull.jpg)

Pod 的 `imagePullPolicy` 镜像拉取策略有三种：
- `IfNotPresent`：只有当镜像在本地不存在时才会拉取。
- `Always`：kubelet 会对比镜像的 digest ，如果本地已缓存则直接使用本地缓存，否则从镜像仓库中拉取。
- `Never`：只使用本地镜像，如果不存在则直接失败。

说明：每个镜像的 digest 一定唯一，但是 tag 可以被覆盖。

---

从镜像拉取的过程来看，我们可以从以下三个方面来加速镜像拉取：
1. 缩减镜像大小：
    使用较小的基础镜像、移除无用的依赖、减少镜像 layer 、使用多阶段构建等等。
    推荐使用 [docker-slim](https://github.com/docker-slim/docker-slim)
1. 加快镜像仓库与 k8s 节点之间的网络传输速度。
1. 主动缓存镜像：
    Pre-pulled 预拉取镜像，以便后续直接使用本地缓存，比如可以使用 daemonset 定期同步仓库中的镜像到 k8s 节点本地。

---

题外话 1：本地镜像缓存多久？是否会造成磁盘占用问题？

本地缓存的镜像一定会占用节点的磁盘空间，也就是说缓存的镜像越多，占用的磁盘空间越大，并且缓存的镜像默认一直存在，并没有 TTL 机制（比如说多长时间以后自动过期删除）。

但是，k8s 的 GC 机制会自动清理掉镜像。当节点的磁盘使用率达到 `HighThresholdPercent` 高百分比阈值时（默认 85% ）会触发垃圾回收，此时 kubelet 会根据使用情况删除最旧的不再使用的镜像，直到磁盘使用率达到 `LowThresholdPercent`（默认 80% ）。


题外话 2：镜像 layer 层数真的越少越好吗？

我们经常会看到一些文章说在 Dockerfile 里使用更少的 RUN 命令之类的减少镜像的 layer 层数然后缩减镜像的大小，layer 越少镜像越小这确实没错，但是某些场景下得不偿失。首先，如果你的 RUN 命令很大，一旦你修改了其中某一个小的部分，那么这个 layer 在构建的时候就只能重新再来，无法使用任何缓存；其次，镜像的 layer 在上传和下载的过程中是可以并发的，而单独一个大的层无法进行并发传输。