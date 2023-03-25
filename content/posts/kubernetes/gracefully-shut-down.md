+++
draft = false
date = 2023-03-25T21:06:05+08:00
title = "Kubernetes 优雅终止 pod"
description = "Kubernetes 优雅终止 pod"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

---

*文本翻译自: https://itnext.io/how-do-you-gracefully-shut-down-pods-in-kubernetes-fb19f617cd67*

---

![](https://miro.medium.com/v2/0*95EBbIEaG9T2_N5V.png)

当你执行 `kubectl delete pod` 时，pod 被删除， endpoint 控制器从 service 和 etcd 中删除该 pod 的 IP 地址和端口。

你可以使用 `kubectl describe service` 观察到这一点。

![使用 kubectl describe 列出 endpoint](https://miro.medium.com/v2/0*u1dqT-9CWfJPzSz-.png)

但远不止如此！

**多个组件都会同步变更至本地 endpoint 列表：**

- kube-proxy 通过本地 endpoint 列表来编写 iptables 规则
- CoreDNS 使用 endpoint 重新配置 DNS

Ingress 控制器、Istio 等也是如此。

![Kubernetes 中的 endpoint 传播](https://miro.medium.com/v2/0*for72y-XTvmvKZuF.png)

所有这些组件都将（最终）删除以前的 endpoint，这样就再也没有流量可以到达它了。

同时，kubelet 也收到了变化的通知，并删除了 pod。

*当 kubelet 在其余组件之前删除 pod 时会发生什么？*

![](https://miro.medium.com/v2/0*7tK2_LUn-gwGEAad.png)

**不幸的是，你会遇到停机，** 因为 kube-proxy、CoreDNS、ingress 控制器等组件仍在使用该 IP 地址来路由流量。

*所以，你可以做什么？*

等待！

![kubelet 还没有传播 endpoint 便立即删除 pod](https://miro.medium.com/v2/0*Ew5wmjSUW1wQmti9.png)

**如果在删除 Pod 之前等待足够长的时间，飞行中的流量仍然可以解析，并且可以将新流量分配给其他 Pod。**

*你应该如何等待？*

![kubelet 应该在删除 pod 之前等待 endpoint 传播](https://miro.medium.com/v2/0*344P0iQtxQxJNzi8.png)

当 kubelet 删除一个 pod 时，它会经历以下步骤：

- 触发 `preStop` 钩子（如果有）。
- 发送 `SIGTERM`。
- 发送 `SIGKILL` 信号（默认 30 秒后）。

![kubelet 删除 pod 经历了 3 个步骤：preStop hook、SIGTERM 和 SIGKILL](https://miro.medium.com/v2/0*LrgZBq0nLMJIPfz4.png)

**你可以使用**`preStop`**挂钩来插入人工延迟。**

![你可以使用 preStop 挂钩来延迟删除 pod](https://miro.medium.com/v2/0*wItrocVhRzm4DYMm.png)

**你可以在你的应用程序中监听 SIGTERM 信号并等待。**

此外，你可以优雅地停止该过程并在等待完成后退出。

Kubernetes 给你 30 秒的时间来这样做（时长可配置）。

![你可以在你的应用程序中捕获 SIGTERM 信号并等待](https://miro.medium.com/v2/0*Lj2c8E2Cyb-TJwXP.png)

*你应该等待 10 秒、20 秒还是 30 秒？*

没有单一的答案。

虽然传播 endpoint 可能只需要几秒钟，但 Kubernetes 不保证任何时间，也不保证所有组件将同时完成。

![Kubernetes 中的 endpoint 传播时间线](https://miro.medium.com/v2/0*2qhqDUVV50rrTCYU.png)

如果你想探索更多，这里有一些链接：

- https://learnk8s.io/graceful-shutdown
- https://freecontent.manning.com/handling-client-requests-properly-with-kubernetes/
- https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods
- https://medium.com/tailwinds-navigator/kubernetes-tip-how-to-gracefully-handle-pod-deletion-b28d23644ccc
- https://medium.com/flant-com/kubernetes-graceful-shutdown-nginx-php-fpm-d5ab266963c2
- https://www.openshift.com/blog/kubernetes-pods-life
