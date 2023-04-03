+++
draft = false
date = 2023-04-03T15:12:11+08:00
title = "我们为何选择 Cilium 作为 Kubernetes 的网络接口"
description = "我们为何选择 Cilium 作为 Kubernetes 的网络接口"
slug = ""
authors = []
tags = ["Kubernetes", "CNI", "Cilium"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

---

*文本翻译自: https://blog.palark.com/why-cilium-for-kubernetes-networking/*

---

> 原文作者是 Palark 平台工程师 Anton Kuliashov，其说明了选择 Cilium 作为 Kubernetes 网络接口的原因以及喜爱 Cilium 的地方。


多亏了 CNI（容器网络接口），Kubernetes 提供了大量选项来满足您的网络需求。在多年依赖简单的解决方案之后，我们面临着对高级功能日益增长的客户需求。Cilium 将我们 K8s 平台中的网络提升到了一个新的水平。

## 背景

我们为不同行业、规模和技术堆栈的公司构建和维护基础设施。他们的应用程序部署到私有云和公共云以及裸机服务器。他们对容错性、可扩展性、财务费用、安全性等方面有不同的要求。在提供我们的服务时，我们需要满足所有这些期望，同时足够高效以应对新兴的与基础设施相关的多样性。

多年前，当我们构建基于 Kubernetes 的早期平台时，我们着手实现基于可靠开源组件的生产就绪、简单、可靠的解决方案。为实现这一目标，我们的 CNI 插件的自然选择似乎是 [Flannel](https://github.com/flannel-io/flannel)（与 kube-proxy 一起使用）。

当时最受欢迎的选择是 Flannel 和 Weave Net。Flannel 更成熟，依赖性最小，并且易于安装。我们的基准测试也证明它的性能很高。因此，我们选择了它，并最终对我们的选择感到满意。

同时，我们坚信有一天会达到极限。

## 随着需求的增长

随着时间的推移，我们获得了更多的客户、更多的 Kubernetes 集群以及对平台的更具体的要求。我们遇到了对更好的安全性、性能和可观测性的日益增长的需求。这些需求适用于各种基础设施元素，而网络显然是其中之一。最终，我们意识到是时候转向更高级的 CNI 插件了。

许多问题促使我们跳到下一阶段：

1. 一家金融机构执行了严格的“默认禁止一切”规则。
2. 一个广泛使用的门户网站的集群有大量的服务，这对 kube-proxy 产生了压倒性的影响。
3. PCI DSS 合规性要求另一个客户实施灵活而强大的网络策略管理，并在其之上具有良好的可观测性。
4. 在 Flannel 使用的 iptables 和 netfilter 中，遇到大量传入流量的多个其他应用程序面临性能问题。

我们不能再受现有限制的阻碍，因此决定在我们的 Kubernetes 平台中寻找另一个 CNI —— 一个可以应对所有新挑战的 CNI。

## 为什么选择 Cilium

今天有很多可用的 [CNI 选项](https://www.cni.dev/docs/#3rd-party-plugins)。我们想坚持使用 [eBPF](https://en.wikipedia.org/wiki/Berkeley_Packet_Filter)，它被证明是一项强大的技术，在可观测性、安全性等方面提供了许多好处。考虑到这一点，当您想到 CNI 插件时，会出现两个著名的项目：[Cilium](https://cilium.io/) 和 [Calico](https://www.tigera.io/project-calico/)。

总的来说，他们两个都非常棒。但是，我们仍然需要选择其中之一。Cilium 似乎在社区中得到了更广泛的使用和讨论：更好的 GitHub 统计数据（例如 stars、forks 和 contributors）可以作为证明其价值的某种论据。它也是一个 CNCF 项目。虽然它不能保证太多，但这仍然是一个有效的观点，所有事情都是平等的。

在阅读了关于 Cilium 的各种文章后，我们决定尝试一下，并在几个不同的 K8s 集群上进行了各种测试。事实证明，这是一次纯粹的积极体验，揭示了比我们预期更多的功能和好处。

## 我们喜欢的 Cilium 的主要功能

在考虑是否使用 Cilium 来解决我们遇到的上述问题时，我们喜欢 Cilium 的地方如下：

### 1. 性能

使用 bpfilter（而不是 iptables）进行路由[意味着](https://www.admin-magazine.com/Archive/2019/50/Bpfilter-offers-a-new-approach-to-packet-filtering-in-Linux)将过滤任务转移到内核空间，这会产生令人印象深刻的性能提升。这正是项目设计、大量文章和第三方基准测试所承诺的。我们自己的测试证实，与我们之前使用的 Flannel + kube-proxy 相比，处理流量速度有显着提升。

![](https://blog.palark.com/wp-content/uploads/2023/03/ebpf-host-routing-diagram.png)
*eBPF host-routing compared to using iptables. source: [“CNI Benchmark: Understanding Cilium Network Performance”](https://cilium.io/blog/2021/05/11/cni-benchmark/)*


有关此主题的有用资料包括：

1. [Why is the kernel community replacing iptables with BPF?](https://cilium.io/blog/2018/04/17/why-is-the-kernel-community-replacing-iptables/)
2. [BPF, eBPF, XDP and Bpfilter… What are These Things and What do They Mean for the Enterprise?](https://www.netronome.com/blog/bpf-ebpf-xdp-and-bpfilter-what-are-these-things-and-what-do-they-mean-enterprise/)
3. [kube-proxy Hybrid Modes](https://docs.cilium.io/en/v1.9/gettingstarted/kubeproxy-free/#kube-proxy-hybrid-modes)

### 2. 更好的网络策略

[CiliumNetworkPolicy CRD](https://docs.cilium.io/en/latest/network/kubernetes/policy/#ciliumnetworkpolicy) 扩展了 Kubernetes NetworkPolicy API。它带来了 L7（而不仅仅是 L3/L4）网络策略支持网络策略中的 ingress 和 egress 以及 [port ranges](https://github.com/cilium/cilium/issues/16622) 规范等功能。

正如 Cilium 开发人员所说：“理想情况下，所有功能都将合并到标准资源格式中，并且不再需要此 CRD。”

### 3. 节点间流量控制

借助 [CiliumClusterwideNetworkPolicy](https://docs.cilium.io/en/latest/network/kubernetes/policy/#ciliumclusterwidenetworkpolicy) ，您可以控制节点间流量。这些策略适用于整个集群（非命名空间），并为您提供将节点指定为源和目标的方法。它使过滤不同节点组之间的流量变得方便。

### 4. 策略执行模式

易于使用的 [策略执行模式](https://docs.cilium.io/en/latest/security/policy/intro/#policy-enforcement-modes) 让生活变得更加轻松。 *default* 模式适合大多数情况：没有初始限制，但一旦允许某些内容，其余所有内容都会受到限制*。*Always* 模式 —— 当对所有端点执行策略时 —— 对于具有更高安全要求的环境很有帮助。

### 5. Hubble 及其 UI

[Hubble](https://github.com/cilium/hubble) 是一个真正出色的网络和服务可观测性以及视觉渲染工具。具体来说，就是对流量进行监控，实时更新服务交互图。您可以轻松查看正在处理的请求、相关 IP、如何应用网络策略等。

现在举几个例子，说明如何在我的 Kubernetes 沙箱中使用 Hubble。首先，这里我们有带有 Ingress-NGINX 控制器的命名空间。我们可以看到一个外部用户通过 Dex 授权后进入了 Hubble UI。会是谁呢？...

![](https://blog.palark.com/wp-content/uploads/2023/03/cilium-hubble-dex-ingress.png)

现在，这里有一个更有趣的例子：Hubble 花了大约一分钟的时间可视化 Prometheus 命名空间如何与集群的其余部分通信。您可以看到 Prometheus 从众多服务中抓取了指标。多么棒的功能！在您花费数小时为您的项目绘制所有这些基础架构图之前，您应该已经知道了！

![](https://blog.palark.com/wp-content/uploads/2023/03/cilium-hubble-prometheus-services.png)

### 6. 可视化策略编辑器

[此在线服务](https://editor.cilium.io/) 提供易于使用、鼠标友好的 UI 来创建规则并获取相应的 YAML 配置以应用它们。我在这里唯一需要抱怨的是缺少对现有配置进行反向可视化的功能。

再此说明，这个列表远非完整的 Cilium 功能集。这只是我根据我们的需要和我们最感兴趣的内容做出的有偏见的选择。

## Cilium 为我们做了什么

让我们回顾一下我们的客户遇到的具体问题，这些问题促使我们开始对在 Kubernetes 平台中使用 Cilium 产生兴趣。

第一种情况下的“默认禁止一切”规则是使用上述策略执行方式实现的。通常，我们会通过指定此特定环境中允许的内容的完整列表并禁止其他所有内容来依赖 *default* 模式。

以下是一些可能对其他人有帮助的相当简单的策略示例。您很可能会有几十个或数百个这样的策略。

1. 允许任何 Pod 访问 Istio 端点：

```yaml
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: all-pods-to-istio-internal-access
spec:
  egress:
  - toEndpoints:
    - matchLabels:
        k8s:io.kubernetes.pod.namespace: infra-istio
    toPorts:
    - ports:
      - port: "8443"
        protocol: TCP
  endpointSelector: {}
```

2. 允许给定命名空间内的所有流量：

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-ingress-egress-within-namespace
spec:
  egress:
  - toEndpoints:
    - {}
  endpointSelector: {}
  ingress:
  - fromEndpoints:
    - {}
```

3. 允许 VictoriaMetrics 抓取给定命名空间中的所有 Pod：

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: vmagent-allow-desired-namespace
spec:
  egress:
  - toEndpoints:
    - matchLabels:
        k8s:io.kubernetes.pod.namespace: desired-namespace
  endpointSelector:
    matchLabels:
      k8s:io.cilium.k8s.policy.serviceaccount: victoria-metrics-agent-usr
      k8s:io.kubernetes.pod.namespace: vmagent-system
```

4. 允许 Kubernetes Metrics Server 访问 kubelet 端口：

```yaml
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: host-firewall-allow-metrics-server-to-kubelet
spec:
  ingress:
  - fromEndpoints:
    - matchLabels:
        k8s:io.cilium.k8s.policy.serviceaccount: metrics-server
        k8s:io.kubernetes.pod.namespace: my-metrics-namespace
    toPorts:
    - ports:
      - port: "10250"
        protocol: TCP
  nodeSelector:
    matchLabels: {}
```

至于其他问题，我们最初遇到的挑战是：

- 案例 #2 和 #4，由于基于 iptables 的网络堆栈性能不佳。我们提到的基准和我们执行的测试在实际操作中证明了自己。
- Hubble 提供了足够水平的可观测性，这在案例 #3 中是必需的。

## 下一步是什么？

总结这次经验，我们成功解决了与 Kubernetes 网络相关的所有痛点。

关于 Cilium 的总体未来，我们能说些什么？虽然它目前是[一个孵化的 CNCF 项目](https://www.cncf.io/projects/cilium/)，但它已于去年年底[申请毕业](https://cilium.io/blog/2022/10/27/cilium-applies-for-graduation/)。这需要一些时间才能完成，但这个项目正朝着一个非常明确的方向前进。最近，在 2023 年 2 月，Cilium [宣布](https://www.cncf.io/blog/2023/02/13/a-well-secured-project-cilium-security-audits-2022-published/) 通过了两次安全审计，这是进一步毕业的重要一步。

我们正在关注该项目的[路线图](https://docs.cilium.io/en/latest/community/roadmap/)，并等待一些功能和相关工具的实施或变得足够成熟。（没错，[Tetragon](https://github.com/cilium/tetragon) 将会很棒！）

例如，虽然我们在高流量集群中使用 Kubernetes [EndpointSlice CRD](https://docs.cilium.io/en/latest/network/kubernetes/ciliumendpointslice/)，但相关的[Cilium 功能](https://docs.cilium.io/en/latest/network/kubernetes/ciliumendpointslice_beta/) 目前处于 beta 阶段 —— 因此，我们正在等待其稳定发布。我们正在等待稳定的另一个测试版功能是 [本地重定向策略](https://docs.cilium.io/en/latest/network/kubernetes/local-redirect-policy/#local-redirect-policy-beta)，它将 Pod 流量本地重定向到节点内的另一个后端 Pod，而不是整个集群内的随机后端 Pod。

## 后记

在生产环境中确定了我们新的网络基础设施并评估了它的性能和新功能之后，我们很高兴决定采用 Cilium，因为它的好处是显而易见的。对于多样化且不断变化的云原生世界来说，这可能不是灵丹妙药，而且绝不是最容易上手的技术。然而，如果你有动力、知识和一点冒险欲望，那么它 100% 值得尝试，而且很可能会得到多方面的回报。
