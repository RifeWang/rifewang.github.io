+++
draft = false
date = 2024-12-19T15:06:08+08:00
title = "我对 OpenAI Kubernetes 集群故障的追问与疑惑"
description = "我对 OpenAI Kubernetes 集群故障的追问与疑惑"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

2024 年 12 月 11 号，`OpenAI` 的 `Kubernetes` 集群发生故障，`API`、`ChatGPT`、`Sora` 等服务都受到了影响，且时间长达 4 小时 22 分钟。

关于这次故障，官方有公开复盘，也有很多媒体博主追踪报导。然而，我对此并不满意，**本文我将会提出自己的疑惑与追问**。

## 故障复盘

> At 3:12 PM PST, we deployed a new telemetry service to collect detailed Kubernetes control plane metrics. 

> Telemetry services have a very wide footprint, so this new service’s configuration unintentionally caused every node in each cluster to execute resource-intensive Kubernetes API operations whose cost scaled with the size of the cluster. With thousands of nodes performing these operations simultaneously, the Kubernetes API servers became overwhelmed, taking down the Kubernetes control plane in most of our large clusters. This issue was most pronounced in our largest clusters, so our testing didn’t catch it – and DNS caching made the issue far less visible until the rollouts had begun fleet-wide.

> The Kubernetes data plane can operate largely independently of the control plane, but DNS relies on the control plane – services don’t know how to contact one another without the Kubernetes control plane. 

> In short, the root cause was a new telemetry service configuration that unexpectedly generated massive Kubernetes API load across large clusters, overwhelming the control plane and breaking DNS-based service discovery.

根据以上官方信息所述，故障的根本原因在于新部署了一个 telemetry 服务，这个服务的本意是收集详细的控制平面的指标，却意外地导致集群中的每个 `node` 节点都执行了资源密集型的 `Kubernetes API` 操作，集群中的 `node` 节点数量越多，对 `kube-apiserver` 的冲击越大，最终导致 `kube-apiserver` 瘫痪。

对此，我的疑问是：
- **telemetry 服务到底做了什么会导致 node 节点更频繁的调用了 kube-apiserver ？**
- **telemetry 服务的本意是收集 control plane 控制平面的指标，为什么会牵扯到 node 节点？**

node 节点只是一个范围，真正执行操作的应该是节点上的某个组件，比如 `kubelet`、`kube-proxy`、`CNI`、`Pod/Container`。

- **到底是哪个/哪些具体的组件执行了更加频繁的对 kube-apiserver 的操作？**

集群内的服务使用了 `DNS-based service discovery`，DNS 在缓存失效后需要与 `kube-apiserver` 交互，而后者已经瘫痪，因此集群内的服务通信异常，而这种异常也很容易辨别（集群内部域名无法解析），所以也能很快地定位到故障。

> DNS caching mitigated the impact temporarily by providing stale but functional DNS records. However, as cached records expired over the following 20 minutes, services began failing due to their reliance on real-time DNS resolution. This timing was critical because it delayed the visibility of the issue, allowing the rollout to continue before the full scope of the problem was understood. Once the DNS caches were empty, the load on the DNS servers was multiplied, adding further load to the control plane and further complicating immediate mitigation.

再看故障分析里对 DNS 的描述，由于 DNS 存在缓存，它延迟了问题的显现，所以工程师刚开始没发现问题，继续推进了部署工作。而在 DNS 缓存失效后，控制平面 `kube-apiserver` 的负载又被加大。

DNS 存在缓存机制这点应该是一种常识，部署的时候忽略了也可以理解，但是：
- **DNS 不应该背锅，机制就是这样，怎么使用和配置完全是工程师的职责。**
- **等到 DNS 挂了才发现问题就有点晚了，既然知道部署的新服务与控制平面交互，那么对控制平面的监控呢？**

> In order to make that fix, we needed to access the Kubernetes control plane – which we could not do due to the increased load to the Kubernetes API servers.

为了修复问题，需要访问 kube-apiserver，但是后者挂掉了因此无法做到。

听上去有些道理，但是 `kube-apiserver` 专门提供了 `--max-requests-inflight` 和 `--max-mutation-requests-inflight` 两个参数用于设置最大并发请求数。**工程师对此一无所知吗？**

另外，`kube-apiserver` 还提供了 `APF`（`API Priority and Fairness`）机制，用来划分请求的优先级，保障集群管理员在过载的情况下仍然能够控制 `kube-apiserver`。

> We do not yet have a mechanism to ensure access to the API server when the data plane is putting too much pressure on the control plane. We’ll implement break-glass mechanisms to ensure that engineers can access Kubernetes API servers in any circumstances.

然而，从披露的信息看，**OpenAI 的工程师似乎之前对这些缺乏深入了解，是使用的 kubernetes 版本过于老旧，还是基于 kubernetes 魔改了很多东西？**

**他们说的 `break-glass` 机制又是什么？这不就是 `APF` 干的事情吗？**


我对 OpenAI Kubernetes 集群故障的疑惑很多，但是这些细节恐怕只有他们内部工程师知道了。


## 总结

`kube-apiserver` 是一个非常重要的组件，不只是文中提到的 DNS 服务会与它交互，因此对集群的监控一定要额外注意它。另外很多组件的数据存储除了支持 `kubernetes API` 之外，还可以直连 `etcd`，因此做高可用的时候，也可以考虑让组件使用另外的 `etcd`，从而减小对控制平面的依赖。


---

(我是凌虚，关注我，无广告，专注技术，欢迎交流或为我推荐工作)

---

参考资料：

- *https://status.openai.com/incidents/ctrsv3lwd797*
- *https://kubernetes.io/docs/concepts/cluster-administration/flow-control/*