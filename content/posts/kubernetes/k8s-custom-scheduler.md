+++
draft = false
date = 2024-06-15T20:04:06+08:00
title = "Kubernetes scheduler 概述及自定义调度器"
description = "Kubernetes scheduler 概述及自定义调度器"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

## kube-scheduler

`kube-scheduler` 是 k8s 集群中控制平面的一个重要组件，其负责的工作简单且专一：给未分配的 pod 分配一个 node 节点。

调度器的大致工作过程可以分为以下几步：
- 监听到未绑定 node 的 pod。
- 过滤节点：挑选出来适合分配这个 pod 的 node 节点（可能有多个）。
- 节点打分：给过滤出来的节点进行打分。
- 最后选择得分最高的那个 node 与 pod 绑定（如果最高得分有多个 node 则随机选择一个）。

更加详细的步骤则参考下图所示：

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/scheduling-framework-extensions.png)

### 扩展点和默认插件

扩展点：调度器内部的工作流程划分为了一系列的步骤，为了满足可扩展性，某些步骤被设计成了扩展点，用户可以自己编码实现接口插入到这些扩展点，从而实现自定义调度器。

默认插件：Kubernetes 已经定义好了很多默认插件，这些插件实现了一个或者多个扩展点，默认调度器就是由这些默认插件组合而成。

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/scheduler-point-plugin.png)

如上图所示，蓝色部分是扩展点，绿色部分是默认插件。

扩展点是按照顺序依次执行的，每个插件可以作用于一个或者多个扩展点。

调度通过以下扩展点的一系列阶段进行：
- `queueSort`: 这些插件提供排序功能，用于对调度队列中的待处理 Pod 进行排序。每次只能启用一个队列排序插件。
- `preFilter`: 这些插件用于在过滤之前预处理或检查有关 Pod 或集群的信息。它们可以将 Pod 标记为不可调度。
- `filter`: 这些插件相当于调度策略中的谓词，用于过滤无法运行 Pod 的节点。过滤插件按配置的顺序调用。如果没有节点通过所有过滤器，Pod 将被标记为不可调度。
- `postFilter`: 当没有找到可行节点来调度 Pod 时，这些插件按配置顺序被调用。如果任何 postFilter 插件将 Pod 标记为可调度，则不会调用剩余插件。
- `preScore`: 这是一个信息性扩展点，可用于在评分之前进行工作。
- `score`: 这些插件为通过过滤阶段的每个节点提供一个分数，然后调度器会选择得分总和最高的节点。
- `reserve`: 这是一个信息性扩展点，通知插件何时为特定 Pod 保留了资源。插件还实现一个 Unreserve 调用，如果在保留期间或之后发生失败，将调用该函数。
- `permit`: 这些插件可以阻止或延迟 Pod 的绑定。
- `preBind`: 这些插件执行 Pod 绑定前所需的任何工作。
- `bind`: 这些插件将 Pod 绑定到节点。绑定插件按顺序调用，一旦其中一个完成绑定，剩余插件将被跳过。至少需要一个绑定插件。
- `postBind`: 这是一个信息性扩展点，在 Pod 绑定后调用。

默认插件非常多，不过你看它的名字就知道它们干了啥，本文不多赘述。

## 自定义调度器

根据是否编写代码，我把自定义调度器的方式分为了两种：
- 不写代码，调整组合已有的默认插件，从而定义新的调度器。
- 实现接口代码，然后定义调度器。

本文将会描述第一种方式，通过调整默认插件的方式快速定义一个新的调度器。

### 自定义调度器示例

默认插件 `NodeResourcesFit` 有三种评分策略：`LeastAllocated`(默认)、`MostAllocated` 和 `RequestedToCapacityRatio`，这三种策略的目的分别是优先选择资源使用率最低的节点、优先选择资源使用率较高的节点从而最大化节点资源使用率、以及平衡节点的资源使用率。

默认插件 `VolumeBinding` 绑定卷的默认超时时间是 600 秒。

示例中，我将自定义一个调度器，将 `NodeResourcesFit` 的评分策略配置为 `MostAllocated`，`VolumeBinding` 的超时时间配置为 60 秒。

#### 配置 `KubeSchedulerConfiguration`

首先，通过 `KubeSchedulerConfiguration` 对象自定义了一个调度器，叫做 my-custom-scheduler：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: my-custom-scheduler # 调度器名称
  plugins:
    score:
      enabled:
      - name: NodeResourcesFit
        weight: 1
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: MostAllocated
        resources:
        - name: cpu
          weight: 1
        - name: memory
          weight: 1
  - name: VolumeBinding
    args:
      bindTimeoutSeconds: 60
```

由于 `KubeSchedulerConfiguration` 对象本质上是 kube-scheduler 的配置文件，为了后续部署的时候方便使用，可以通过定义一个 `ConfigMap` 包含 `KubeSchedulerConfiguration` 的内容：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-scheduler-config
  namespace: kube-system
data:
  my-scheduler-config.yaml: |
    apiVersion: kubescheduler.config.k8s.io/v1
    kind: KubeSchedulerConfiguration
    profiles:
    - schedulerName: my-custom-scheduler # 调度器名称
      plugins:
        score:
          enabled:
          - name: NodeResourcesFit
            weight: 1
      pluginConfig:
      - name: NodeResourcesFit
        args:
          scoringStrategy:
            type: MostAllocated
            resources:
            - name: cpu
              weight: 1
            - name: memory
              weight: 1
      - name: VolumeBinding
        args:
          bindTimeoutSeconds: 60
```

#### 部署 Deployment 应用 kube-scheduler

然后，我们需要部署 kube-scheduler 应用我们自定义的调度器，为此可以定义一个 Deployment：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-custom-kube-scheduler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      component: my-custom-kube-scheduler
  template:
    metadata:
      labels:
        component: my-custom-kube-scheduler
    spec:
      # serviceAccountName 注意需要配置权限
      containers:
      - command:
        - kube-scheduler
        - --leader-elect=false
        - --config=/etc/kubernetes/my-scheduler-config.yaml
        - -v=5
        image: registry.k8s.io/kube-scheduler:v1.30.0
        name: kube-scheduler
        volumeMounts:
        - name: my-scheduler-config
          mountPath: /etc/kubernetes/my-scheduler-config.yaml
          subPath: my-scheduler-config.yaml
      volumes:
      - name: my-scheduler-config
        configMap:
          name: my-scheduler-config
```

注意：这里我们直接使用了 kube-scheduler 官方镜像，只是传递了不同的配置文件而已。

到这里你一定会问：自己部署的自定义调度器与已经存在的默认调度器会有冲突吗？只要 `schedulerName` 不同就不会有冲突，两个调度器各跑各的。通过自己部署的方式也避免了对默认调度器的任何干预。

至此，简单的两步就实现了自定义调度器。

#### 验证

我们通过部署两个 pod ，分别使用默认调度器 default-scheduler 和我们的自定义调度器 my-custom-scheduler ：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-default
spec:
  # schedulerName 默认就是使用 default-scheduler
  containers:
  - image: nginx
    name: nginx

---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-custom
spec:
  schedulerName: my-custom-scheduler
  containers:
  - image: nginx
    name: nginx
```

然后观察自定义调度器的日志：

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/scheduler-log.jpg)

从图中可以看到，自定义调度器工作正常，顺利完成了 pod 的调度，且与默认调度器互不影响。

---

参考资料：

- *https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/*
- *https://kubernetes.io/docs/reference/scheduling/config/*
- *https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/*
- *https://arthurchiao.art/blog/k8s-scheduling-plugins-zh/*