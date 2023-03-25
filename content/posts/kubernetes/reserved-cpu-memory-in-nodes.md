+++
draft = false
date = 2023-03-25T16:42:31+08:00
title = "Kubernetes 节点的预留资源"
description = "Kubernetes 节点的预留资源"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

---

*文本翻译自: https://medium.com/@danielepolencic/reserved-cpu-and-memory-in-kubernetes-nodes-65aee1946afd*

---

*在 Kubernetes 中，运行多个集群节点是否存在隐形成本？*

是的，因为[并非 Kubernetes 节点中的所有 CPU 和内存都可用于运行 Pod。](https://learnk8s.io/allocatable-resources)

在一个 Kubernetes 节点中，CPU 和内存分为：

1. 操作系统
2. Kubelet、CNI、CRI、CSI（+ 系统 daemons）
3. Pods
4. 驱逐阈值

![Kubernetes 节点中的可分配资源](https://miro.medium.com/v2/0*2JOWmHzYsGH_EV9a.png)

这些预留的资源取决于实例的大小，并且可能会增加相当大的开销。

*让我们举一个简单的例子。*

想象一下，你有一个具有单个 1GiB / 1vCPU 节点的集群。

以下资源是为 kubelet 和操作系统保留的：

- 255MiB 内存。
- 60m 的 CPU。

最重要的是，为驱逐阈值预留了 100MB。

那么总共**有 25% 的内存和 6% 的 CPU 不能使用。**

![在 1GiB / 1vCPU Kubernetes 节点中，25% 的内存由 kubelet 保留](https://miro.medium.com/v2/0*z9VdA6cYtCbusoQS.png)

*在云厂商中的情况又是如何？*

EKS 有一些（有趣的？）限制。

让我们选择一个具有 2vCPU 和 8GiB 内存的 m5.large 实例。

AWS 为 kubelet 和操作系统保留了以下内容：

- 574MiB 内存。
- 70m 的 CPU。

*这一次，你很幸运。*

**你可以使用大约 93% 的可用内存。**

![如果将 EKS 与 m5.large 节点一起使用，则可以使用 93% 的可用内存来运行 pod](https://miro.medium.com/v2/0*6OXCGq4oHIudaR1W.png)

*但这些数字从何而来？*

每个云厂商都有自己定义限制的方式，但对于 CPU，他们似乎都同意以下值：

- 第一个核心的 6%。
- 下一个核心的 1%（最多 2 个核心）。
- 接下来 2 个核心的 0.5%（最多 4 个核心）。
- 四核以上任何核的 0.25%。

至于内存限制，云厂商之间差异很大。

Azure 是最保守的，而 AWS 则是最不保守的。

[Azure 中 kubelet 的预留内存为](https://docs.microsoft.com/en-us/azure/aks/concepts-clusters-workloads#resource-reservations)：

- 前 4 GB 内存的 25%。
- 4 GB 以下内存的 20%（最大 8 GB）。
- 8 GB 以下内存的 10%（最大 16 GB）。
- 下一个 112 GB 内存的 6%（最多 128 GB）。
- 超过 128 GB 的任何内存的 2%。

这[对于 GKE 是相同的](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-architecture#memory_cpu)，除了一个值：逐出阈值在 GKE 中为 100MB，在 AKS 中为 750MiB。

[在 EKS 中，使用以下公式分配内存：](https://github.com/awslabs/amazon-eks-ami/blob/4b54ee95d42df8a2715add2a32f5150db097fde8/files/bootstrap.sh#L247-L251)

```
255MiB + (11MiB * MAX_NUMBER OF POD)
```

*不过，这个公式提出了一些问题。*

在前面的示例中，m5.large 保留了 574MiB 的内存。

*这是否意味着 VM 最多可以有 (574–255) / 11 = 29 个 pod？*

[如果你没有在 VPC-CNI 中启用 prefix 前缀分配模式，这是正确的。](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html)

**如果这样做，结果将大不相同。**

对于多达 110 个 pod，AWS 保留：

- 1.4GiB 内存。
- （仍然）70m 的 CPU。

这听起来更合理，并且与其他云厂商一致。

![如果你将 EKS 与 m5.large 节点和前缀分配模式一起使用，则 kubelet 会保留 1.4GiB 的内存](https://miro.medium.com/v2/0*12kfnJmIW4MLLmpU.png)

让我们看看 GKE 进行比较。

对于类似的实例类型（即 n1-standard-2，7.5GB 内存，2vCPU），kubelet 的预留如下：

- 1.7GB 内存。
- 70m 的 CPU。

![如果你将 GKE 与 n1-standard-2 节点一起使用，则 kubelet 会保留 1.7GiB 的内存](https://miro.medium.com/v2/0*lCKAy1gjsyJyOta2.png)

换句话说，23% 的实例内存无法分配给运行的 Pod。

> 如果实例每月花费 48.54 美元，那么**你将花费 11.16 美元来运行 kubelet。**

*其他云厂商呢？*

*你如何检查这些值？*

我们构建了一个简单的工具来检查 kubelet 的配置并提取相关细节。

你可以在这里找到它：[https://github.com/learnk8s/kubernetes-resource-inspector](https://github.com/learnk8s/kubernetes-resource-inspector)

如果你有兴趣探索更多关于节点大小的信息，我们还构建了一个简单的实例计算器，你可以在其中定义工作负载的大小，它会显示适合该大小的所有实例（及其价格）。

[https://learnk8s.io/kubernetes-instance-calculator](https://learnk8s.io/kubernetes-instance-calculator)

我希望你喜欢这篇关于 Kubernetes 资源预留的短文；在这里，你可以找到更多链接以进一步探索该主题。

- [Kubernetes instance calculator](https://learnk8s.io/kubernetes-instance-calculator)。
- [Allocatable memory and CPU in Kubernetes Nodes](https://learnk8s.io/allocatable-resources)
- [Allocatable memory and CPU resources on GKE](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-architecture#memory_cpu)
- AWS EKS AMI [reserved CPU](https://github.com/awslabs/amazon-eks-ami/blob/4b54ee95d42df8a2715add2a32f5150db097fde8/files/bootstrap.sh#L253-L262) and [reserved memory](https://github.com/awslabs/amazon-eks-ami/blob/4b54ee95d42df8a2715add2a32f5150db097fde8/files/bootstrap.sh#L247-L251)
- [AKS resource reservations](https://docs.microsoft.com/en-us/azure/aks/concepts-clusters-workloads#resource-reservations)
- 官方文档 [https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/)
- [Enabling prefix assignment in EKS and VPC CNI](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html)


