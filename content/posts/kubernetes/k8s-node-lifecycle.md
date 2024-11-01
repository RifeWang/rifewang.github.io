+++
draft = false
date = 2024-11-01T20:42:58+08:00
title = "Kubernetes Node 节点的生命周期简述"
description = "Kubernetes Node 节点的生命周期简述"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

`Node` 节点是 Kubernetes 的核心组件之一，其生命周期可以简要概括为：注册、运行、下线。本文将简要介绍 `Node` 生命周期中发生的关键事件。

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/kubelet/node-lifecycle.drawio.png)

### 节点注册

每个 `node` 节点都需要运行 `kubelet`，`kubelet` 启动后会向 `kube-apiserver` 发起节点的注册请求，即创建一个新的 `node` 资源对象。

`Kubelet` 配置文件中的 `registerNode`（或命令行参数 `--register-node`）的值默认为 true，用来控制是否自动注册节点。如果你希望手动管理节点的注册行为，可以将此项设置为 false。

节点的名称 `nodename` 由以下因素决定：
- 如果配置了 cloud provider，则由云供应商提供名称。
- 否则使用本机的 `hostname`，而 `hostname` 也可以通过 `kubelet` 的配置项 `--hostname-override` 覆盖掉。

注册节点实质上是创建了一个新的 `node` 资源对象，此时 `kubelet` 便会收集有关节点的状态信息一并提交。该接口也可以重复提交，反复注册并不会有什么影响。

### 节点心跳机制

节点的心跳机制包括两部分：节点 `.status` 状态信息更新，以及节点对应的 `lease` 对象更新。

`Kubelet` 配置文件中的 `nodeStatusUpdateFrequency`（或命令行参数 `--node-status-update-frequency`）默认为 10 秒钟。这意味着当节点状态发生改变时，或者达到了 10 秒钟，kubelet 会向 kube-apiserver 发起请求，以更新节点的 `.status` 状态信息。

每个节点都会在 `kube-node-lease` 这个命名空间中维护一个同名的 `lease` 对象，更新频率为 `kubelet` 配置文件中的 `nodeLeaseDurationSeconds`（默认 40 秒）* 0.25，即 10 秒钟。

### 节点健康监控

`controller-manager` 中的 `node-controller`（准确说是 `node-lifecycle-controller`）负责监控节点的健康情况。如果一切正常，那自然万事大吉。

但是如果节点出现网络中断或者宕机等情况时，`node-controller` 便会发现节点的心跳信息长时间未更新，一旦超过 `controller-manager` 的配置项 `--node-monitor-grace-period` 设置的时长（默认 40 秒，在未来的 v1.32 版将会变更为 50 秒），`node-controller` 会将该节点的状态设置为 `Unknown`，并给节点打上 `Taint` 污点，避免新的 pod 被调度。随后再等待 5 分钟，如果节点仍未恢复心跳，则开始向 kube-apiserver 发起请求，驱逐节点上的 pod 等资源。

节点的正常下线也非常类似，标记污点、重新调度 pod、下线节点。


(我是凌虚，关注我，无广告，专注技术，不煽动情绪，欢迎与我交流)

---

参考资料：

- *https://kubernetes.io/docs/concepts/architecture/nodes/*
- *https://kubernetes.io/docs/reference/node/node-status/*
- *https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/*
- *https://github.com/kubernetes/kubernetes/pull/126287*