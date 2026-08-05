+++
draft = false
date = 2026-08-05T11:40:00+08:00
title = "Kubernetes 编年史：从 Borg 到云原生操作系统"
description = "用故事串起 Kubernetes 十余年的关键特性与节点事件：它从何而来、为何胜出、又如何长成今天几乎无处不在的云原生底座。"
slug = ""
authors = []
tags = ["Kubernetes", "云原生", "容器", "CNCF", "历史"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

> 如果把数据中心想象成一座港口，容器是标准化的货柜，那么 `Kubernetes` 就是港口调度系统：决定哪艘船停哪个泊位、货柜怎么装卸、坏了怎么换、忙了怎么扩。它并不制造货柜，却决定货柜能不能规模化地运转起来。

---

## 开篇：为什么普通人也会撞上 `Kubernetes`

你不一定写过 `YAML`，但很可能已经在用 `Kubernetes`。

打开手机里的某个 App、刷短视频、问一句 ChatGPT——这些服务背后，很大概率跑在某个 `Kubernetes` 集群上。它几乎成了云时代的“默认操作系统”：不负责写业务逻辑，却负责把成千上万个服务摆上机器、保持存活、按流量伸缩。

名字来自希腊语，本意是“舵手”或“领航员”。社区常把它缩写成 **K8s**（K 与 s 之间夹着 8 个字母）。Logo 是一个七边形——这不是随便画的，后文会讲到。

这篇编年史不打算把每个版本的 changelog 抄一遍。我想讲清楚三件事：

1. `Kubernetes` 解决的到底是什么问题
2. 它一路上踩过哪些关键节点、长出了哪些真正改变行业的能力
3. 为什么最终是它，而不是 Swarm、Mesos 或其他选手，成了行业默认答案

即使你从没碰过 `kubectl`，读完也应该能明白个大概。

---

## 前传：Google 内部的 Borg（约 2003 年起）

故事要从 Google 讲起。

早在公开云和 Docker 流行之前，Google 已经在用**容器**把一台物理机切成很多隔离的小房间，让搜索、Gmail、YouTube 这类服务挤在一起跑，把机器利用率压到极高。管理这些“房间”的系统叫 **Borg**。

Borg 不是给外面人用的产品，而是 Google 内部的集群管家。它要回答的问题很朴素：

- 这个任务该放到哪台机器上？
- 机器挂了，任务怎么自动迁走？
- 高峰来了，怎么多开几份副本？
- 不同优先级的任务抢资源时，谁让谁？

后来 Google 又做了 **Omega**，继续探索更灵活的调度与共享状态模型。Borg / Omega 的论文（ACM Queue，《Borg, Omega, and Kubernetes》）后来几乎成了容器编排领域的必读文献。

对普通人来说，记住一句就够了：

> `Kubernetes` 不是从零空想出来的，它是 Google 把十几年“怎么大规模跑容器”的教训，重新用开源方式讲给全世界听。

它继承了 Borg 的很多直觉——最典型的是把一组紧密协作的容器当成一个调度单元（Borg 里叫 `alloc`，`Kubernetes` 里叫 `Pod`）。与此同时，它也刻意改进了 Borg 的限制：Borg 主要靠比较僵硬的 `Job` 来分组，`Kubernetes` 则用灵活的标签（`Label`）和选择器来组织对象；并且更强调声明式的“期望状态”——你描述“我想要什么”，系统自己去凑齐。

在控制面的访问方式上，它也不像后来的 Omega 那样把共享存储直接敞开给可信组件，而是对外只暴露一套带校验与策略的 REST API。

---

## 2013—2014：Docker 点燃容器，调度问题突然变成全民问题

2013 年，Docker 把“容器”这件事做得人人能用。以前容器更像运维专家的黑科技；Docker 之后，开发者可以在笔记本上打包应用，几乎原样搬到服务器。

货柜标准化了，下一题立刻出现：

**一台机器上的几个容器好管；一百台、一千台机器上的几千个容器，谁来管？**

于是“编排”（orchestration）成了新战场：自动部署、扩缩容、服务发现、故障恢复。谁能把这件事做成通用平台，谁就有机会成为云时代的基础设施标准。

2013 年夏天，Google 的 Joe Beda、Brendan Burns、Craig McLuckie 向公司高层推销一个想法：做一个开源的容器管理系统，把 Borg / Omega 的经验带到外面的世界。项目内部代号叫 **Project Seven of Nine**——向《星际迷航》里的“七九号”致敬。这也是 `Kubernetes` Logo 有七个边的原因。

2014 年 6 月 6 日，`Kubernetes` 正式开源，首个 commit 落库。业界通常把这一天当作它的生日。

那时它还不完美，甚至有点粗糙。但方向是对的：面向开发者体验，声明式 API，可扩展，而且从第一天就选择开源。

---

## 2015：1.0 发布，并把方向盘交给 CNCF

开源一年多后，2015 年 7 月 21 日，`Kubernetes` **1.0** 发布。Google 同时宣布：把这个项目捐赠给新成立的 **CNCF**（Cloud Native Computing Foundation，云原生计算基金会），由 Linux Foundation 托管。

这件事的象征意义很大。

如果 `Kubernetes` 一直是“Google 的开源项目”，竞争对手心里会打鼓；捐给中立基金会之后，Red Hat、IBM、Intel、VMware，甚至后来的 AWS、Microsoft，才更愿意把未来押在它上面。CNCF 的目标也不只是养大 `Kubernetes`，而是推动一整套“云原生”技术栈：容器化、动态调度、微服务化。

约一个月后（2015 年 8 月 26 日），Google 的托管服务 **Google Container Engine**（后来的 GKE）也进入正式可用。云厂商开始把“帮你管好 `Kubernetes` 控制面”当成产品。

到 1.0 时，核心故事已经成形：

| 概念 | 可以怎么理解 |
| --- | --- |
| `Pod` | 最小调度单位，像一个小隔间，里面可以放一个或多个紧密协作的容器 |
| `Node` | 一台工作机器（物理机或虚拟机） |
| `Service` | 给一组会变来变去的 `Pod` 提供稳定的访问入口 |
| `Label` / `Selector` | 给对象贴标签，再按标签筛选——像给快递包裹打分类码 |
| 声明式 API | 你说“我要 3 个副本一直活着”，系统自己去凑齐，而不是你手工 `SSH` 开机 |

很多后来人尽皆知的能力——`Deployment`、`DaemonSet`、`StatefulSet`、`Ingress`、完善的 `RBAC`——其实是 1.0 之后才陆续长出来的。1.0 更像一个能用的最小闭环：先证明“编排这件事可以做成平台”。

---

## 2015—2017：编排战争，以及 `Kubernetes` 为什么赢了

那段时间，市场上至少有三拨主力选手：

- **Docker Swarm**：跟 Docker 集成最紧，上手最快
- **Apache Mesos + Marathon**：更早出现，擅长超大规模、异构负载
- **`Kubernetes`**：学习曲线陡，但模型完整、可扩展性强

还有 Nomad 等其他方案。媒体喜欢叫它“编排之战”。

`Kubernetes` 最终胜出，通常不是因为“某一个开关特别好用”，而是几股力量叠在一起：

1. **Google 实战背书 + CNCF 中立治理**，降低了厂商押注的政治风险
2. **API 与扩展点设计得好**，后来的 `CRD`、`Operator`、各类接口都建立在这上面
3. **生态自我强化**：监控、网络、存储、CI/CD、安全工具都优先对接 `Kubernetes`
4. **云厂商集体站队**：一旦 AWS、Azure、GCP 都提供托管 `Kubernetes`，企业选型就几乎被锁定

高潮出现在 2017 年 10 月的 DockerCon Europe：Docker 宣布在自己的平台里原生支持 `Kubernetes`，与 Swarm 并存。舆论几乎一边倒地认为——编排战争的胜负已分。

Swarm 并没有立刻消失，Mesos 也还在特定场景里活着；但“默认答案是谁”这件事，行业已经有了共识。

---

## 2016—2018：从“能跑起来”到“能当真用”

如果把 1.0 看成学会走路，接下来几年就是长肌肉。几项真正改变日常使用的能力，大致长在这个阶段。

### `Deployment`：让发布不再像拆炸弹

早期大家更多直接摆弄 `Pod` 或 `ReplicationController`。后来 **`Deployment`** 成了无状态应用的标准姿势：滚动升级、回滚、声明副本数。

你可以把它理解成：

> 不要再手工决定“先杀哪个容器、再起哪个容器”，而是告诉系统“新版本长这样，请平滑替换过去”。

`DaemonSet`（每个节点跑一份，适合日志/监控 Agent）、`StatefulSet`（有稳定身份和存储的有状态应用）、`ReplicaSet` 等核心工作负载 API，在 `Kubernetes` **1.9**（2018 年初）一起升到了正式稳定版（`apps/v1`）。

这意味着：常见应用形态终于有了“官方推荐且稳定”的一等公民 API。

### `RBAC`：谁能对集群做什么

集群一旦进生产，权限就不能再是“所有人都能 `kubectl` 为所欲为”。

**`RBAC`**（Role-Based Access Control，基于角色的访问控制）在 **1.8**（2017）成为正式能力。它回答的是：这个账号能不能看 `Secret`、能不能删 `Namespace`、能不能改节点。

对非安全专家，一句话就够：

> `RBAC` 把 `Kubernetes` 从“共享 root 密码的服务器群”变成了“有门禁的系统”。

### 网络与策略：`Service` 之外，还要会“拒绝”

`Kubernetes` 默认很“友善”——`Pod` 之间往往能互相连通。这对演示友好，对生产危险。

**`NetworkPolicy`** 让你写规则：谁可以访问谁。配合 `CNI` 网络插件（Calico、Cilium、Flannel 等），集群网络从“能通”走向“可控”。`networking.k8s.io/v1` 下的 `NetworkPolicy` 在 **1.7**（2017）升到正式稳定；**1.8** 又补上了更实用的出口（`egress`）策略。旧的 `extensions/v1beta1` 后来在 1.16 停止服务。

### 包管理与“应用交付”

光有 API 还不够，人还得把复杂应用装上去。**Helm** 之类的包管理工具在这段时间迅速普及，把一组 `YAML` 收成可安装、可升级的 Chart。`Kubernetes` 负责“资源怎么变成现实”，Helm 负责“这一整套应用怎么交付”。

---

## 2016—2019：真正的杀手锏——可扩展性

如果说编排战争拼的是“谁能调度容器”，`Kubernetes` 后半场赢在另一句话：

> **把平台做成可以长出新平台的东西。**

### `CRI` / `CNI` / `CSI`：不绑死任何一家实现

`Kubernetes` 很早就意识到：自己不该既当裁判又当所有运动员。

- **`CRI`**（Container Runtime Interface）：容器怎么创建/删除，交给 runtime。2016 年底随 **1.5** 以 Alpha 形式引入，先把“运行时可插拔”这条路铺开；随后几年，`containerd`、`CRI-O` 等才陆续把 `CRI` 实现做成熟。
- **`CNI`**（Container Network Interface）：容器网络怎么接，交给网络插件。
- **`CSI`**（Container Storage Interface）：持久化存储怎么挂，交给存储插件。`CSI` 在 **1.13**（2019 年初）正式 GA。

这三条接口的战略意义，比任何一个单点功能都大：

> 网络厂商、存储厂商、运行时项目都可以在不改 `Kubernetes` 内核的前提下加入赛道。生态越大，标准越稳；标准越稳，生态越大。

### `CRD`：让 `Kubernetes` 听懂新名词

**`CustomResourceDefinition`（`CRD`）** 允许你在集群里登记新的 API 类型——比如 `PostgresCluster`、`Certificate`、`Prometheus`。登记之后，`kubectl get` 也能管它们，就像管原生的 `Pod` 一样。

`CRD` 的前身是 `ThirdPartyResources`；重新设计后在 **1.7** 进入 beta，并在 **1.16**（2019）正式 GA（`apiextensions.k8s.io/v1`）。

对新手，这几乎是理解现代 `Kubernetes` 的钥匙：

> 今天你在集群里看到的很多“高级能力”，并不是 `Kubernetes` 内核突然变魔法了，而是有人用 `CRD` 教集群认识了新对象，再用控制器去实现它们。

### `Operator`：把运维经验写成程序

2016 年 11 月，CoreOS 提出 **`Operator`** 模式：把“怎么部署、备份、故障转移、升级一个复杂系统”的人工经验，写成一直盯着自定义资源的控制器。那时 `CRD` 尚未正式稳定，早期实现更多建立在 `ThirdPartyResource` 等扩展机制上；等到 `CRD` 成熟后，`Operator` 几乎都转到了 `CRD` 这条路上。

最早开源的例子包括 `etcd Operator`、`Prometheus Operator`。之后这个模式爆火——数据库、消息队列、证书、机器学习平台，都能以 `Operator` 的方式“住进”集群。

一句话比喻：

> `Deployment` 会保证“有 3 个一模一样的 Web 副本”；`Operator` 会保证“这套有状态系统按专家的方式活着”。

---

## 2018：三大云齐聚，`Kubernetes` 成为云的通用语言

- **GKE**：2015 年就正式可用，起步最早
- **Amazon EKS**：2018 年 6 月正式可用
- **Azure AKS**：同样在 2018 年 6 月正式可用

至此，AWS、Azure、GCP 都提供托管 `Kubernetes`。对企业来说，这意味着：

> 学会一套模型，就能在多家云上说话；差异更多落在周边服务、网络和权限，而不是“要不要学第三种编排器”。

`Kubernetes` 从一个开源项目，变成了云计算的**可移植层**——不完全消灭锁定，但显著降低了“应用怎么跑”这一层的锁定。

更早一点，CNCF 在 2017 年 11 月推出了 **Certified `Kubernetes` Conformance**（一致性认证）：自称 `Kubernetes` 的发行版，必须通过一组测试，保证核心 API 行为一致。这避免了当年 Unix / Android 式的严重分裂，也让 2018 年各大云的托管服务更容易被企业当成“同一门语言的不同口音”。

---

## 2020—2022：断舍离与成年礼

任何活过十年的系统，都要学会扔掉旧船票。

### 告别 `dockershim`

很长一段时间，很多人以为“`Kubernetes` = 用 Docker 跑容器”。其实 `Kubernetes` 真正依赖的是**容器运行时接口**；Docker Engine 只是早期最流行的一种实现。

为了兼容 Docker，`kubelet` 里曾内置 **`dockershim`** 这层垫片。它在 **1.20** 被正式弃用，并在 **1.24**（2022 年 4 月）被移除。

社区建议转向 `containerd`、`CRI-O` 等 `CRI` 兼容运行时。对终端用户，镜像构建仍可用 Docker；变化主要在集群节点“到底用谁来跑容器”。

这是一次心理上的成年礼：

> `Kubernetes` 明确告诉世界——容器生态的标准是 `OCI` / `CRI`，不再是某一个商业公司的工具链。

### 安全模型重做：从 `PSP` 到 Pod Security Admission

**`PodSecurityPolicy`（`PSP`）** 曾是限制 `Pod` 权限的内建机制，但难用、难演进。它在 **1.21** 弃用，并在 **1.25**（2022）移除。

取而代之的是更简单的 **Pod Security Admission**：按命名空间贴标签，套用 `Privileged` / `Baseline` / `Restricted` 三档标准。不够细的场景，再交给外部策略引擎（如 `OPA` / Gatekeeper、`Kyverno`）。

方向很清晰：默认更安全，机制更少魔法。

### 控制面也在长“成人功能”

这些年还有一批不那么出圈、但对大规模集群要命的能力：API Priority and Fairness（过载时仍保住管理员入口）、更精细的调度与拓扑感知等。它们不太适合写进广告语，却决定了“几千节点时还能否睡得着觉”。

---

## 2023—2026：入口演进，以及 AI 把新压力塞进旧港口

### `Gateway API`：下一代流量入口

对外暴露 HTTP/HTTPS 服务，过去多年靠 **`Ingress`**。它简单，但扩展方式碎片化——注解满天飞，行为因实现而异。

**`Gateway API`** 在 2023 年 10 月发布 **v1.0**，`Gateway`、`GatewayClass`、`HTTPRoute` 等核心资源升到正式稳定。它把“基础设施怎么提供入口”和“应用怎么声明路由”拆得更清楚，也更适合多团队、多网关实现共存。

`Ingress` 不会瞬间消失，但新项目越来越默认看向 `Gateway API`。这很像当年从 `ReplicationController` 走向 `Deployment`：旧的还能用，新的才是方向。

### AI / GPU：老船长遇上新货种

大模型训练与推理，把 GPU、高速网络、超大作业调度重新推到台前。`Kubernetes` 并不是为 LLM 发明的，但它已经是多数公司唯一够通用的集群底座，于是 Device Plugin、拓扑感知调度、批量作业、队列系统都在往上叠。

2023 年 11 月前后，公开报道显示 Google 曾借助 GKE 等能力，调度约 5 万颗 TPU v5e 规模的分布式训练作业。这说明：

> 它未必是 AI 基础设施的最终形态，但眼下它是最现成、最能被组织消化的那一层。

与此同时，安全、可观测性、平台工程（Internal Developer Platform）继续把 `Kubernetes` 藏到自助服务后面——开发者不一定直接碰集群，但平台团队几乎都在其上构建。

---

## 一条时间线（方便对照）

| 时间 | 事件 |
| --- | --- |
| ~2003 起 | Google 内部 Borg 大规模管理容器化任务 |
| 2013 | Docker 让容器成为开发者日常工具；`Kubernetes` 以 Project Seven of Nine 立项 |
| 2014-06-06 | `Kubernetes` 开源 |
| 2015-07-21 | `Kubernetes` 1.0；CNCF 成立并接收该项目 |
| 2015-08-26 | Google Container Engine（后来的 GKE）正式可用 |
| 2016 | `CRI` Alpha（1.5）；CoreOS 提出 `Operator`；生态加速 |
| 2017 | `NetworkPolicy` GA（1.7）；`RBAC` GA（1.8）；Docker 宣布支持 `Kubernetes`；CNCF 推出一致性认证 |
| 2018 | 核心工作负载 API GA（1.9）；EKS / AKS 正式可用 |
| 2019 | `CSI` GA（1.13）；`CRD` GA（1.16） |
| 2020—2022 | `dockershim` 弃用（1.20）并移除（1.24）；`PSP` 移除，Pod Security Admission 稳定（1.25） |
| 2023-10 | `Gateway API` v1.0，核心资源正式稳定 |
| 2024 | 开源十周年；继续向 AI、多集群、平台工程场景延伸 |
| 2025—2026 | `Gateway API` 持续把实验特性推入稳定；K8s 仍是默认的云原生底座 |

---

## 它到底凭什么成为“编年史主角”

回看这十余年，`Kubernetes` 的胜利可以压缩成四个词：

1. **时机**：Docker 解决了打包，它补上了规模化运营
2. **经验**：Borg 十年踩坑，换来更清醒的抽象
3. **治理**：交给 CNCF，换来竞品也愿意共建
4. **接口**：`CRI`/`CNI`/`CSI` + `CRD`/`Operator`，让别人能在它身上继续盖楼

它也从不“简单”。学习曲线陡、`YAML` 又臭又长、生产事故可以很戏剧——这些吐槽都成立。但基础设施很少因为“优雅无缺点”而胜出；它们因为**足够通用、足够可扩展、足够被生态托住**而胜出。

如果只记住一个比喻，我想是这个：

> Docker 让软件变成标准货柜；`Kubernetes` 让港口学会自动调度。云原生后来长出的监测、服务网格、GitOps、平台工程，大多是在这座港口上加建的码头、吊车与海关。

往前看，港口不会消失，但它会越来越不像今天开发者每天直接面对的那片码头。更多团队会把它藏进平台工程与自助服务之后；更多负载——从微服务到批处理，再到 AI 训练与推理——会把这座港口当成默认停靠点；`Gateway API`、策略即代码、多集群与边缘调度，则可能继续把“怎么对外暴露、怎么治理、怎么跨地域放置”变成更标准化的航线。

`Kubernetes` 未必永远是故事的最后一章，但它已经把云原生的共同语言写进了基础设施。未来十年更可能发生的，不是另起一座全新港口，而是在这座港口上继续加高、加深、让更多人可以安心靠岸。


---

## 参考资料

- [Borg, Omega, and Kubernetes - ACM Queue](https://queue.acm.org/detail.cfm?id=2898444)
- [How Kubernetes came to be - Google Cloud Blog](https://cloud.google.com/blog/products/containers-kubernetes/from-google-to-the-world-the-kubernetes-origin-story)
- [CNCF 成立与 Kubernetes 捐赠公告（2015-07-21）](https://www.cncf.io/announcements/2015/06/21/new-cloud-native-computing-foundation-to-drive-alignment-among-container-technologies/)
- [Introducing CRI in Kubernetes](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/)
- [Core Workloads API GA（1.9）](https://kubernetes.io/blog/2018/01/core-workloads-api-ga/)
- [CSI for Kubernetes GA](https://kubernetes.io/blog/2019/01/15/container-storage-interface-ga/)
- [Kubernetes 1.16：CRD GA](https://kubernetes.io/blog/2019/09/18/kubernetes-1-16-release-announcement/)
- [Dockershim removal（1.24）](https://kubernetes.io/blog/2022/01/07/kubernetes-is-moving-on-from-dockershim/)
- [Pod Security Admission stable / PSP removed（1.25）](https://kubernetes.io/blog/2022/08/25/pod-security-admission-stable/)
- [Gateway API v1.0: GA Release](https://kubernetes.io/blog/2023/10/31/gateway-api-ga/)
- [KuberTENes：开源十年回顾 - The Register](https://www.theregister.com/2024/06/12/kubertenes_decade_anniversary/)
