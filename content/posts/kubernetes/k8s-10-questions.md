+++
draft = false
date = 2024-11-16T15:14:17+08:00
title = "Kubernetes 10 问，测测你对 k8s 的理解程度"
description = "Kubernetes 10 问，测测你对 k8s 的理解程度"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

## Kubernetes 10 问

1. 假设集群有 2 个 node 节点，其中一个有 pod，另一个则没有，那么新的 pod 会被调度到哪个节点上？

2. 应用程序通过容器的形式运行，如果 OOM（Out-of-Memory）了，是容器重启还是所在的 Pod 被重建？

3. 应用程序配置如环境变量或者 `ConfigMap` 可以不重建 Pod 实现动态更新吗？

4. pod 被创建后是稳定的吗，即使用户不进行任何操作？

5. 使用 `ClusterIP` 类型的 `Service` 能保证 TCP 流量的负载均衡吗？

6. 应用日志要怎么采集，会不会有丢失的情况？

7. 某个 HTTP Server Pod 的 `livenessProbe` 正常是否就一定没问题？

8. 应用程序如何扩展以应对流量波动的情况？

9. 当你执行 `kubectl exec -it <pod> -- bash` 之后是登录到了 pod 里吗？

10. 如果 Pod 里的容器反复退出并重启，如何排查？

以上问题你是否能够快速回答，并注意到关键点。

## 解答

#### 1. 假设集群有 2 个 node 节点，其中一个有 pod，另一个则没有，那么新的 pod 会被调度到哪个节点上？

调度过程需要经过一系列处理阶段：

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/scheduling-framework-extensions.png)

各个阶段的配置、以及 Pod 的配置（强制调度、亲和性、污点容忍等等）可能都会影响调度的结果。

然而，在不考虑各种配置、节点资源也相同的情况下，默认插件 `NodeResourcesFit` 的评分策略会起到比较关键的作用，`NodeResourcesFit` 有三种评分策略：`LeastAllocated`（默认，优先选择资源使用率最低的节点）、`MostAllocated`（优先选择资源使用率较高的节点）和 `RequestedToCapacityRatio`（平衡节点的资源使用率）。这也就是说，在使用 `MostAllocated` 策略的时候，新的 pod 会被调度到已经有 pod 的节点上，而使用另外两种策略时，则调度到没有 pod 的节点。


#### 2. 应用程序通过容器的形式运行，如果 OOM（Out-of-Memory）了，是容器重启还是所在的 Pod 被重建？

当容器 OOM 了，一般情况下根据 Pod 的 `RestartPolicy` 配置（默认是 Always），容器会被重启，Pod 不会被重建。但在特殊情况下，节点内存压力很大时，可能会触发 Pod 驱逐，从而导致 Pod 被重建。


#### 3. 应用程序配置如环境变量或者 `ConfigMap` 可以不重建 Pod 实现动态更新吗？

环境变量无法动态更新，`ConfigMap` 通过挂载的方式可以动态更新，但要求挂载时不能使用 `subPath`，挂载的同步延时受 kubelet 的配置 `syncFrequency`（默认 1 分钟）和 `configMapAndSecretChangeDetectionStrategy` 影响。


#### 4. Pod 被创建后是稳定的吗，即使用户不进行任何操作？

即使用户不操作，也可能发生节点资源不足或者网络异常导致 Pod 被驱逐。


#### 5. 使用 `ClusterIP` 类型的 `Service` 能保证 TCP 流量的负载均衡吗？

`ClusterIP` 类型的 `Service` 不论使用的是 `iptables` 还是 `ipvs`，其都依赖于 Linux 内核的 Netfilter，而其内部的 `connection tracking` 机制会跟踪和记录每个连接的状态，进而让已经建立了 TCP 连接的双方一直使用这条连接。所以，针对长连接，是有可能出现负载不均衡的情况。


#### 6. 应用日志要怎么采集，会不会有丢失的情况？

应用日志一般输出为 `stdout/stderr`，或者写入日志文件。针对前者，容器日志会被保存到节点上的特定位置，此时可以使用日志代理（如 `Fluentd`、`Filebeat`）并部署为 `Daemonset` 的方式进行采集，但是存在日志丢失的可能，因为一旦 pod 被删除，对应的容器日志文件也会被删除，而日志代理可能还未完成全部日志的采集。通过挂载持久化存储并写入日志文件的方式则可以避免丢失。


#### 7. 某个 HTTP Server Pod 的 livenessProbe 正常是否就一定没问题？

站在应用的角度看，`livenessProbe` 只检查应用是否存活，无法验证其功能是否正常，例如应用可能进入了非健康但仍存活的状态。
站在网络的角度看，`livenessProbe`（假设配置的是 httpGet）是由本机的 kubelet 发起的请求，无法保证跨节点的网络是正常的。


#### 8. 应用程序如何扩展以应对流量波动的情况？

Kubernetes 提供了水平扩展（`HPA`）和垂直扩展（`VPA`）两种机制，由于 `VPA` 不是原地资源扩展，会删除 Pod 再创建，使用场景往往受限，所以使用更多是 `HPA`，可以根据指标（如 CPU 使用率、请求速率、其它自定义指标）动态调整 Pod 数量。
当然，也可以在外部监测指标然后向 kube-apiserver 发起请求扩展 Pod 的数量。


#### 9. 当你执行 `kubectl exec -it <pod> -- bash` 之后是登录到了 pod 里吗？

首先，使用 `kubectl exec` 需要指定容器，当 Pod 只有一个容器时则可以忽略。其次，Pod 是一组独立的 Linux 命名空间，容器本质上是一个进程，Pod 内容器共享 Network、IPC、UTS namespace，而每个容器的 PID、Mount namespace 则是独立的。"登录"的说法并不准确，`kubectl exec -it <pod> -- bash` 既不是进入了 Pod，也不是进入了容器，而是在目标容器的隔离环境中创建了一个新的 bash 进程。


#### 10. 如果 Pod 里的容器反复退出并重启，如何排查？

如果 Pod 里的容器反复退出并重启，是无法使用 `kubectl exec` 的。此时，除了检查节点或容器状态和日志之外，还可以使用 `kubectl debug` 的方式在 Pod 上启动一个临时容器，用于检查环境和依赖。


---

(我是凌虚，关注我，无广告，专注技术，不煽动情绪，欢迎与我交流)