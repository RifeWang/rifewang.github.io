+++
draft = false
date = 2024-09-22T14:34:08+08:00
title = "kubectl 执行一条命令之后发生了什么？"
description = "kubectl 执行一条命令之后发生了什么？"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

`kubectl` 是与 Kubernetes 集群交互的命令行工具，用户通过它可以对集群资源进行操作和管理。你有没有想过，当我们执行一条 `kubectl` 命令之后，背后都发生了什么？

## 详细过程

### kubectl -> kube-api-server

根据通信类型，我把 `kubectl` 命令分为两类：单向通信和双向通信。
- 单向通信：`kubectl` 通过 `HTTPS` 向 `kube-api-server` 发起请求并接收响应。大部分 `kubectl` 命令都是单向通信的，比如资源的创建、删除、修改和查询等。
- 双向通信：在执行某些持续性操作时，例如 `exec`、`attach`、`port-forward`、`logs` 等等少数命令，`kubectl` 与 `kube-api-server` 之间会使用 `WebSocket` 进行双向通信，此时双方可以进行持续性的双向的消息传递。

对于大部分单向通信的命令，`kube-api-server` 处理（可能与 etcd 交互）之后就会响应给 `kubectl`，虽然集群内其它组件可能会有后续动作，但是 `kubectl` 到 `kube-api-server` 的过程则到这里就结束了。

然而，双向通信的命令，其处理的链路会更长。下文以 `kubectl exec -it nginx -- bash` 这条命令为例进行详细说明。

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/k8s-kubectl.drawio.png)

如上图所示，`kubectl` 会先向 `kube-api-server` 发起 HTTPS 请求，并协商升级为 `WebSocket`，至此 `kubectl` 与 `kube-api-server` 之间可以持续的收发消息。

### kube-api-server -> kubelet

`kubelet` 会启动以下三个 server：
- HTTPS server：这是 `kubelet` 的主要服务，提供了完整的与 `kube-api-server` 和其他组件交互的接口，处理和控制所有与容器、Pod 生命周期相关的重要操作，包括 /healthz 健康检查、/metrics 指标采集、/pods pod 管理，以及 /exec、/attach、/portForward 等等接口：
```sh
/exec/{podNamespace}/{podID}/{containerName}
/attach/{podNamespace}/{podID}/{containerName}
/portForward/{podNamespace}/{podID}/{containerName}
/containerLogs/{podNamespace}/{podID}/{containerName}
```
- HTTP Read-Only server：提供只读的 API，用于公开一些状态信息，出于安全考虑默认禁用。可以通过 `--read-only-port` 参数启用，但不建议在生产环境中使用。
- gRPC server：专门用于查询节点上 Pod 和容器的资源分配情况。

当 `kube-api-server` 收到 `kubectl exec` 请求后，会通过 `HTTPS` 向 `kubelet` 发起请求，由 `kubelet` 负责进一步处理。

### kubelet -> CRI

`kubelet` 通过 `gRPC` 调用 `CRI` 组件，向容器运行时传递指令。

`gRPC` 协议定义了 `RuntimeService` 和 `ImageService` 两种服务及多种方法，其中就包括：
- `RuntimeServer.ExecSync`
- `RuntimeServer.Exec`
- `RuntimeServer.Attach`
- `RuntimeServer.PortForward`

`CRI` 组件最终执行指令，并将执行结果（stdout/stderr）依次经由 `kubelet`、`kube-api-server` 最后返回给 `kubectl` 并在客户端显示。这就是完整的过程。

## 总结

文本讲述了 `kubectl` 命令执行背后的整个过程，以 `kubectl exec` 为例，涉及 `kube-api-server`、`kubelet`、`CRI` 等多个组件的协作，以及 `HTTPS`、`WebSocket`、`gRPC` 等多种通信协议。


(关注我，无广告，专注于技术，不煽动情绪)

---

参考资料：

- *https://kubernetes.io/docs/reference/kubectl/generated/*
- *https://erkanerol.github.io/post/how-kubectl-exec-works/*
- *https://github.com/kubernetes/kubernetes/blob/v1.31.1/pkg/kubelet/server/server.go*
