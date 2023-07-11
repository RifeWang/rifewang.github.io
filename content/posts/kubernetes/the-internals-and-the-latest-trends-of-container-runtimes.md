+++
draft = false
date = 2023-07-11T10:07:05+08:00
title = "容器运行时的内部结构和最新趋势（2023）"
description = "容器运行时的内部结构和最新趋势（2023）"
slug = ""
authors = []
tags = ["Container", "Docker", "Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

# 容器运行时的内部结构和最新趋势（2023）

原文为 Akihiro Suda 在日本京都大学做的在线讲座，完整的 PPT 可 [点击此处下载](https://github.com/AkihiroSuda/AkihiroSuda/raw/5d9f0b1cd9b8c37cb1951768a3bebdb08a3a469e/slides/2023/20230615%20%5BKyoto%20University%5D%20The%20internals%20and%20the%20latest%20trends%20of%20container%20runtimes.pdf)

本文内容分为以下三个部分：

1. 容器简介
2. 容器运行时的内部结构
3. 容器运行时的最新趋势

-----

## 1. 容器简介

### 什么是容器？

容器是一组用于隔离文件系统、CPU 资源、内存资源、系统权限等的各种轻量级方法。容器在很多意义上类似于虚拟机，但它们比虚拟机更高效，而安全性则往往低于虚拟机。

![](https://miro.medium.com/1*OVsYlSmH_L15vparu4VrWg.png)

有趣的是，“*容器*”目前还没有严格的定义。当虚拟机提供类似容器的接口时，例如，当它们实现 [OCI（开放容器）规范](https://specs.opencontainers.org/) 时，甚至虚拟机也可以被称为“*容器*”。这种“非容器”的容器将在后面的第三部分中讨论。

### Docker

[Docker](https://www.docker.com/) 是最流行的容器引擎。Docker 本身支持 Linux 容器和 Windows 容器，但 Windows 容器不在本次讨论的范围之内。

启动 Docker 容器的典型命令行如下：

```shell
docker run -p 8080:80 -v .:/usr/share/nginx/html nginx:1.25
```

执行该命令后，可以在 `http://<the host’s IP>:8080/` 中看到当前目录下 `index.html` 的内容。

命令中的 `-p 8080:80` 部分指定将主机的 TCP 8080 端口转发到容器的 80 端口。

命令中的 `-v .:/usr/share/nginx/html` 部分指定将主机上的当前目录挂载到容器中的 `/usr/share/nginx/html`。

命令中的 `nginx:1.25` 指定使用 [Docker Hub](https://hub.docker.com/) 上的 [官方 nginx 镜像](https://hub.docker.com/_/nginx)。Docker 镜像与虚拟机镜像有些相似，但是它们通常不包含额外的诸如 systemd 和 sshd 等守护进程。

您也可以在 [Docker Hub](https://hub.docker.com/search) 上找到其他应用程序的官方镜像。您还可以使用称为 `Dockerfile` 的语言自行构建自己的镜像：

```Dockerfile
FROM debian:12
RUN  apt-get update && apt-get install -y openjdk-17-jre
COPY myapp.jar /myapp.jar
CMD  ["java", "-jar", "/myapp.jar"]
```

可以使用 [docker build](https://docs.docker.com/engine/reference/commandline/build/) 命令构建镜像，并使用 [docker push](https://docs.docker.com/engine/reference/commandline/push/) 命令将其推送到 Docker Hub 或其它镜像仓库。


### Kubernetes

[Kubernetes](https://kubernetes.io/) 将多个容器主机（例如（但不限于）Docker 主机）集群化，以提供负载平衡和容错功能。

![](https://miro.medium.com/1*An2qZhoR6_OGaz7Z4mG8Zg.png)

值得注意的是，Kubernetes 也是一个抽象框架，用于与 [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)（始终在同一主机上共同调度的容器组）、[Services](https://kubernetes.io/docs/concepts/services-networking/service/)（网络连接实体）和 [其它类型的对象](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) 进行交互，但是本次演讲不会深入介绍 kubernetes。

### Docker 与 Docker 之前的容器

虽然容器直到 2013 年 Docker 发布才受到太多关注，但 Docker 并不是第一个容器平台：

- **1999**：[FreeBSD Jail](https://svnweb.freebsd.org/base?view=revision&revision=46155)
- **2000**：[Linux 虚拟环境系统（Virtuozzo 和 OpenVZ 的前身）](https://lkml.iu.edu/hypermail/linux/kernel/0008.2/0042.html)
- **2001**：[Linux Vserver](https://www.cs.helsinki.fi/linux/linux-kernel/2001-40/1065.html)
- **2002**：[Virtuozzo](https://wiki.openvz.org/History)
- **2004**：[BSD Jail for Linux](https://lkml.iu.edu/hypermail/linux/kernel/0409.1/0994.html)
- **2004**：[Solaris Containers（显然，“容器”这个词就是这次创造的）](https://web.archive.org/web/20041116174148/http://www.sun.com/smi/Press/sunflash/2004-11/sunflash.20041115.2.html)
- **2005**：[OpenVZ](https://wiki.openvz.org/History)
- **2008**：[LXC](https://github.com/lxc/lxc/tree/5e97c3fcce787a5bc0f8ceef43aa3e05195b480a)
- **2013**：[Docker](https://www.youtube.com/watch?v=9xciauwbsuo)

人们普遍认为 FreeBSD Jail（大约 1999 年）是类 Unix 操作系统的第一个实用容器实现，尽管“容器”这个术语并不是在那时创造的。

从那时起，Linux 上也出现了几种实现。然而，Docker 之前的容器与 Docker 容器有本质上的不同。前者专注于模仿整个机器，其中包含 System V init、sshd、syslogd 等。当时经常将 Web 服务器、应用服务器、数据库服务器和所有内容放入一个容器中。

Docker 改变了整个范式。就 Docker 而言，一个容器通常只包含一个服务，因此容器可以是无状态且不可变的。这种设计显着降低了维护成本，因为容器现在是一次性的；当需要更新某些内容时，您只需删除容器并从最新镜像重新创建它即可。您也不再需要在容器内安装 sshd 和其他实用程序，因为您永远不需要对其进行 shell 访问。这也简化了多主机集群的负载平衡和容错。

![](https://miro.medium.com/1*Xzpc72NV3fxpZfDrUfCHCA.png)

---

## 2. 容器运行时的内部结构

本节假设使用 Docker v24 及其默认配置，但大多数部分也适用于非 Docker 容器。

### Docker 底层

Docker 由客户端程序（`docker` CLI）和守护进程（`dockerd`）组成。`docker` CLI 通过 Unix 套接字 (`/var/run/docker.sock`) 连接到 `dockerd` 守护进程来创建容器。

然而，`dockerd` 守护进程本身并不创建容器，它将控制权委托给 [`containerd`](https://containerd.io/) 守护进程来创建容器。但 `containerd` 也不创建容器，而是进一步将控制权委托给 [`runc`](https://github.com/opencontainers/runc) 运行时，它包含了多个 Linux 内核功能，例如 Namespaces、Cgroups 和 Capabilities，以实现“*容器*”的概念。Linux 内核中并没有“*容器*”对象。

![](https://miro.medium.com/1*RWzcHdOheUfu_cdEwCRRmQ.png)


### Namespace 命名空间

[Namespace 命名空间](https://man7.org/linux/man-pages/man7/namespaces.7.html) 将资源与主机和其他容器隔离。

最知名的命名空间是 [mount namespace](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html)。Mount 命名空间隔离文件系统视图，以便容器可以使用 [`pivot_root(2)`](https://man7.org/linux/man-pages/man2/pivot_root.2.html) 系统调用将 rootfs 更改为 `/var/lib/docker/.../<container's rootfs>`。该系统调用类似于传统的 [`chroot(2)`](https://man7.org/linux/man-pages/man2/chroot.2.html) 但 [更安全](https://tbhaxor.com/pivot-root-vs-chroot-for-containers/)。

容器的 rootfs 与主机的结构非常相似，但它对 `/proc`、`/sys` 和 `/dev` 有一些限制。例如，

- `/proc/sys` 目录被重新挂载为只读绑定以禁止 sysctl。
- 通过挂载 `/dev/null` 来屏蔽  `/proc/kcore` 文件（RAM）。
- 通过挂载空的只读 tmpfs 来屏蔽 `/sys/firmware` 目录（固件数据）。
- 对 `/dev` 目录的访问受到 Cgroup 的限制（稍后讨论）。

![](https://miro.medium.com/1*T-hPJqFAR6UIZ-yDHETOMQ.png)


[Network namespace](https://man7.org/linux/man-pages/man7/network_namespaces.7.html) 允许为容器分配专用 IP 地址，以便它们可以通过 IP 相互通信。

![](https://miro.medium.com/1*fDuES0pJVmlZ-JLDNM1gSw.png)


[PID namespace](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html) 隔离进程树，以便容器无法控制其外部的进程。

![](https://miro.medium.com/1*ZZdXqUyVmpRb1ZK9yk8OBQ.png)

[User namespace](https://man7.org/linux/man-pages/man7/user_namespaces.7.html)（不要与[用户空间](https://en.wikipedia.org/wiki/User_space_and_kernel_space) 混淆）通过将主机上的非 root 用户映射到容器中的伪 root 来隔离 root 权限。伪 root 可以像容器中的root 一样运行 `apt-get`、`dnf` 等，但它没有对容器外部资源的特权访问。

用户命名空间显着减轻了潜在的容器突破攻击，但 [Docker 中默认不使用它](https://docs.docker.com/engine/security/userns-remap/)。

![](https://miro.medium.com/1*yUzEpHCWi-vw1suk5ncFaw.png)

其他命名空间：

- [**IPC命名空间**](https://man7.org/linux/man-pages/man7/ipc_namespaces.7.html)：隔离 System V 进程间通信对象等。
- [**UTS 命名空间**](https://man7.org/linux/man-pages/man7/uts_namespaces.7.html)：隔离主机名。"UTS"（Unix Time Sharing system）似乎对这个命名空间来说是个用词不当的称呼。
- [**（可选）Cgroup 命名空间**](https://man7.org/linux/man-pages/man7/uts_namespaces.7.html)：隔离 `/sys/fs/cgroup` 层次结构。
- [**（可选）Time 命名空间**](https://man7.org/linux/man-pages/man7/time_namespaces.7.html)：隔离时钟。[大多数容器尚未使用](https://github.com/opencontainers/runtime-spec/pull/1151)。


### Cgroups

[Cgroups](https://man7.org/linux/man-pages/man7/cgroups.7.html)（控制组）施加多种资源配额，例如 CPU 使用率、内存使用率、block I/O 以及容器中的进程数量。

Cgroup 还控制对设备节点的访问。[Docker默认配置](https://github.com/opencontainers/runtime-spec/blob/v1.0.2/config-linux.md#default-devices) 允许无限制访问 `/dev/null`、`/dev/zero`、`/dev/urandom` 等，不允许访问 `/dev/sda`（磁盘设备）、`/dev/mem`（内存）等。


### Capabilities

在 Linux 上，root 权限由 [64-bit capability](https://man7.org/linux/man-pages/man7/capabilities.7.html) 标记。目前使用了 [41 位](https://github.com/torvalds/linux/blob/v6.3/include/uapi/linux/capability.h#L420)。

Docker 的默认配置删除了系统范围的管理功能，例如 `CAP_SYS_ADMIN`。

[保留的能力](https://github.com/moby/moby/blob/v24.0.2/oci/caps/defaults.go)包括：

- `CAP_CHOWN`：用于在容器内运行 `chown`。
- `CAP_NET_BIND_SERVICE`：用于绑定容器内 1024 以下的 TCP 和 UDP 端口。
- `CAP_NET_RAW`：用于运行需要制作原始以太网数据包的[旧版 `ping` 实现](https://github.com/moby/moby/issues/41886#issuecomment-1590736893)。这种功能非常危险，因为它允许在容器网络中进行[ARP 欺骗和 DNS 欺骗](https://blog.aquasec.com/dns-spoofing-kubernetes-clusters)。Docker 的未来版本可能会[默认禁用它](https://github.com/moby/moby/issues/41886)。


### （可选）Seccomp

[Seccomp](https://man7.org/linux/man-pages/man2/seccomp.2.html)（安全计算）允许指定系统调用的显式允许列表（或拒绝列表）。Docker 的默认配置允许大约 [350 个系统调用](https://github.com/moby/moby/blob/v24.0.2/profiles/seccomp/default.json)。

Seccomp 用于[*纵深防御*](https://en.wikipedia.org/wiki/Defense_in_depth_(computing))；对于容器来说这并不是硬性要求。为了向后兼容，Kubernetes 仍然默认不使用 seccomp，并且[在可预见的将来可能永远不会改变默认配置](https://github.com/kubernetes/enhancements/issues/2413#issuecomment-1581231097)。用户仍然可以通过 [KubeletConfiguration](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/#kubelet-config-k8s-io-v1beta1-KubeletConfiguration) 选择启用 seccomp 。


### （可选）AppArmor 或 SELinux

[AppArmor](https://apparmor.net/) 和 [SELinux](https://github.com/SELinuxProject)（安全增强型 Linux）是 [LSM](https://www.kernel.org/doc/html/v6.3/admin-guide/LSM/index.html)（Linux 安全模块），可提供更细粒度的配置旋钮。

这些是相互排斥的；由主机操作系统发行商（而不是容器镜像发行商）选择：

- **AppArmor**：Debian、Ubuntu、SUSE 等选择的。
- **SELinux**：由 Fedora、Red Hat Enterprise Linux 和类似的主机操作系统发行版选择。

为了进行纵深防御，Docker 的 [默认 AppArmor 配置文件](https://github.com/moby/moby/blob/v24.0.2/profiles/apparmor/template.go) 几乎与其功能、挂载掩码等默认配置重叠。用户可以添加自定义设置以提高安全性。

但 SELinux 的情况则不同。要在 [selinux-enabled](https://docs.docker.com/engine/reference/commandline/dockerd/) 模式下运行容器，您必须在绑定挂载上附加选项 `:z`（小写字符）或 `:Z`（大写字符），或者自己运行复杂的 `chcon` 命令避免权限错误。

`:z`（小写字符）选项用于类型强制。类型强制通过为进程和文件分配“类型”来保护主机文件免受容器的影响。以 `container_t` 类型运行的进程可以读取 `container_share_t` 类型的文件，并读/写 `container_file_t` 类型的文件，但无法访问其他类型的文件。

![](https://miro.medium.com/1*KoTwjHe3dUEYQRzfOl_Q2A.png)

`:Z`（大写字符）选项用于多类别安全性。多类别安全性通过为进程和文件分配类别号来保护一个容器免受另一个容器的影响。例如，类别 42 的进程无法访问标记为类别 43 的文件。

![](https://miro.medium.com/1*bQoe2Cca_wWLXrYBlj_t1w.png)


### 适用于 Mac/Win 的 Docker

[Docker Desktop](https://www.docker.com/products/docker-desktop/) 产品支持在 Mac 和 Windows 上运行 Linux 容器，但它们只是在底层运行 Linux 虚拟机来在其上运行容器。这些容器不直接在 macOS 和 Windows 上运行。

-----

## 3.容器运行时的最新趋势

### Docker 的替代品（作为 Kubernetes 运行时）

Kubernetes 的第一个版本（2014 年）是专门为 Docker 制作的。Kubernetes [v1.3](https://kubernetes.io/blog/2016/07/kubernetes-1-3-bridging-cloud-native-and-enterprise-workloads/) (2016) 添加了对名为 `rkt` 的替代容器运行时的临时支持，但 `rkt` 已于[2019 年](https://www.cncf.io/blog/2019/08/16/cncf-archives-the-rkt-project/)退役。支持替代容器运行时的努力在 Kubernetes [v1.5](https://github.com/kubernetes/kubernetes/blob/v1.5.0/docs/devel/container-runtime-interface.md) (2016) 中产生了容器运行时接口 `CRI` API。CRI 首次亮相后，业界已趋同于使用 [containerd](https://containerd.io/) 和 [CRI-O](https://cri-o.io/) 这两种运行时其中之一：。

![](https://miro.medium.com/1*04Edb0wEnXci2c5ye9F8dA.png)

Kubernetes 仍然内置了对 Docker 的支持，但最终在 Kubernetes [v1.24](https://kubernetes.io/blog/2022/03/31/ready-for-dockershim-removal/)（2022年）中被删除。Docker 仍然继续作为第三方运行时为 Kubernetes 工作（通过 [`cri-dockerd`](https://github.com/Mirantis/cri-dockerd) shim），但 Docker 现在在 Kubernetes 中的使用率越来越低。

![](https://miro.medium.com/1*ttq05nTH21UT577xW-FIRg.png)

业界知名大厂已经从 Docker 转向了 containerd 或者 CRI-O：

- **containerd 的采用者**：[Amazon Elastic Kubernetes Service (EKS)](https://docs.aws.amazon.com/eks/latest/userguide/dockershim-deprecation.html)、[Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/cluster-configuration#container-runtime-configuration)、[Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine/docs/how-to/migrate-containerd)、[k3s](https://docs.k3s.io/advanced#configuring-containerd) 等（很多）。
- **CRI-O 的采用者**：[Red Hat OpenShift](https://docs.openshift.com/container-platform/4.13/architecture/architecture.html#architecture-custom-os_architecture)、[Oracle Container Engine for Kubernetes (OKE)](https://docs.oracle.com/en-us/iaas/Content/ContEng/Concepts/contengaboutk8sversions.htm) 等。

Containerd 注重可扩展性，支持非 Kubernetes 工作负载以及 Kubernetes 工作负载。相比之下，CRI-O 注重简单性，并且仅支持 Kubernetes。


### Docker 的替代方案（作为 CLI）

尽管 Kubernetes 已成为多节点生产集群的标准，但用户仍然希望使用类似 Docker 的 CLI 在笔记本电脑上本地构建和测试容器。Docker 基本上满足了这个需求，但是社区中的运行时开发人员希望构建自己的“实验室”CLI，以先于 Docker 和 Kubernetes 孵化新功能，因为通常很难向 Docker 和 Kubernetes 提出新功能，由于一些技术/技术因素原因。

[Podman](https://podman.io/)（以前称为 kpod ）是由 Red Hat 等公司创建的兼容 Docker 的独立容器引擎。它与 Docker 的主要区别在于它默认没有守护进程。此外，Podman 的独特之处在于它为管理 Pod（共享相同网络命名空间的容器组，通常共享同一主机上的数据卷以实现高效通信）以及容器提供一流的支持。然而，大多数用户似乎只将 Podman 用于非 Pod 容器。

[nerdctl](https://github.com/containerd/nerdctl)（我于 2020 年创立）是一个适用于 containerd 的兼容 Docker 的 CLI。nerdctl 最初是为了试验新功能，例如延迟拉取（稍后讨论），但它对于调试运行 containerd 的 Kubernetes 节点也很有用。


### 在 Mac 上运行容器

[Docker Desktop](https://www.docker.com/products/docker-desktop/) 的 Mac 和 Windows 产品是专有的。Windows 用户可以在 WSL2 中运行 Docker 的 Linux 版本（Apache License 2.0，无图形界面），但迄今为止，Mac 用户还没有相应的解决方案。

[Lima](https://lima-vm.io/)（也是我于 2021 年创立的）是一个命令行工具，用于在 macOS 上创建类似 WSL2 的环境来运行容器。Lima 默认使用 nerdctl，但它也支持 Docker 和 Podman。

Lima 还被 [colima](https://github.com/abiosoft/colima) (2021)、[Rancher Desktop](https://rancherdesktop.io/) (2021) 和 [Finch](https://github.com/runfinch/finch) (2022)等第三方项目采用。

Podman 社区发布了 [Podman Machine](https://docs.podman.io/en/latest/markdown/podman-machine.1.html)（命令行工具，2021 年）和 [Podman Desktop](https://podman-desktop.io/)（GUI，2022 年）作为 Docker Desktop 的替代品。Podman Desktop 也支持 Lima（可选）。


### Docker 正在重构

containerd 主要提供两个子系统：运行时子系统和镜像子系统。然而，后者并未被Docker使用。这是一个问题，因为 Docker 自身的传统镜像子系统远远落后于 containerd 的现代镜像子系统（这也导致我启动了nerdctl项目）：

- 不支持 [lazy-pulling 惰性拉取](https://github.com/containerd/stargz-snapshotter)（按需镜像拉取）
- [对多平台镜像的有限支持](https://github.com/moby/moby/issues/44582)（例如 AMD64/ARM64 双平台镜像）
- [OCI 规范的有限合规性](https://github.com/moby/moby/issues/25779)

这个长期存在的问题终于得到解决。Docker v24 (2023) 在 `/etc/docker/daemon.json` 中添加了对使用 containerd 的镜像子系统和 [undocumented option](https://github.com/moby/moby/blob/v24.0.2/daemon/daemon.go#L801) 的实验性支持：

```json
{"features":{"containerd-snapshotter": true}}
```

Docker 的未来版本（2024？2025？）很可能默认使用 containerd 的镜像子系统。


### Lazy-pulling 惰性拉取

容器镜像中的大多数文件从未被使用：

> **“拉取包占容器启动时间的 76%，但其中只有 6.4％ 的数据被读取”**
> 摘自“ [Slacker：使用 Lazy Docker 容器进行快速分发](https://www.usenix.org/conference/fast16/technical-sessions/presentation/harter)”（Harter 等人，FAST 2016）

“惰性拉取”是一种通过按需拉取部分镜像内容来减少容器启动时间的技术。对于 [OCI 标准 tar.gz 镜像](https://github.com/opencontainers/image-spec/blob/v1.0.2/layer.md) 来说这是不可能的，因为它们不支持 `seek()` 操作。人们提出了几种替代格式来支持惰性拉取：

- [**eStargz**](https://github.com/containerd/stargz-snapshotter) (2019) ：优化 seek() 能力的 gzip 粒度；向前兼容 OCI v1 tar.gz。
- [**SOCI**](https://github.com/awslabs/soci-snapshotter) (2022)：捕获 tar.gz 解码器状态的检查点；向前兼容 OCI v1 tar.gz。
- [**Nydus**](https://github.com/containerd/nydus-snapshotter) (2022)：另一种图像格式；
    与 OCI v1 tar.gz 不兼容。
- [**OverlayBD**](https://github.com/containerd/overlaybd) (2021)：将块设备作为容器镜像；与 OCI v1 tar.gz 不兼容。

下图显示了 eStargz 的基准测试结果。惰性拉动（+额外优化）可以将容器启动时间减少到 1/9。

![](https://miro.medium.com/1*MoC4Bvx7V4t6gtRD9UbGkg.png)


### 扩大 User namespace 的采用

尽管 Docker 自 [v1.9](https://github.com/moby/moby/pull/12648)（2015）以来一直支持用户命名空间，但在 Docker 和 Kubernetes 生态系统中仍然很少使用。

原因之一是 “chowning” 容器 rootfs 作为伪根的复杂性和开销。Linux 内核 [v5.12](https://kernelnewbies.org/Linux_5.12#ID_mapping_in_mounts) (2021) 添加了 “idmapped mounts” 以消除 chown 的必要性。计划在 [runc v1.2](https://github.com/opencontainers/runc/pull/3717) 中支持这一点。

runc v1.2 发布后，用户命名空间预计将在 Docker 和 Kubernetes 中更加流行，而 Docker 和 Kubernetes 刚刚在 [v1.25](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.25.md)（2022）中添加了对用户命名空间的 [初步支持](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/127-user-namespaces/README.md)。出于兼容性考虑，Kubernetes 不太可能默认启用用户命名空间。然而，Docker [将来](https://github.com/moby/moby/pull/38795) 仍有可能默认启用用户命名空间。不过，一切还没有决定。


### Rootless 容器

[Rootless 容器](https://rootlesscontaine.rs/) 是一种将容器运行时以及容器放置在由非 root 用户创建的用户命名空间中的技术，以减轻运行时的潜在漏洞。

即使容器运行时存在允许攻击者逃离容器的错误，攻击者也无法拥有对其他用户的文件、内核、固件和设备的特权访问权限。

以下是 rootless 容器的简史：

- **2014**：[LXC v1.0](https://stgraber.org/2014/01/17/lxc-1-0-unprivileged-containers/) 引入了对 rootless 容器的支持。当时 rootless 容器被称为“非特权容器”。LXC 的非特权容器与现代 rootless 容器略有不同，因为它们需要 [SETUID 二进制文件](https://man7.org/linux/man-pages/man1/lxc-user-nic.1.html) 来 [启动网络](https://man7.org/linux/man-pages/man5/lxc-usernet.5.html)。
- **2017**：runc [v1.0-rc4](https://github.com/opencontainers/runc/releases/tag/v1.0.0-rc4) 获得对 rootless容器的初步支持。
- **2018**：一些工具已经开始支持，[containerd](https://twitter.com/_AkihiroSuda_/status/953231819008180224)、[BuildKit](https://twitter.com/_AkihiroSuda_/status/955698849560997888)（`docker build`的后端）、[Docker](https://github.com/AkihiroSuda/docker/commit/588a4e91fc8cb99af040dcde795ba6722a162127)、[Podman](https://github.com/containers/podman/commit/19f5a504ffb1470991f331db412be456e41caab5)。[slirp4netns](https://github.com/rootless-containers/slirp4netns) 被我自己创建，以通过转换以太网来允许 SETUID-less 网络数据包发送至非特权套接字系统调用。
- **2019**：Docker [v19.03](https://docs.docker.com/engine/release-notes/19.03/#19030) 发布，对 rootless 容器提供实验性支持。Podman [v1.1](https://github.com/containers/podman/releases/tag/v1.1.0) 也在今年发布，具有相同的功能，略领先于 Docker v19.03。
- **2020**：Docker [v20.10](/nttlabs/docker-20-10-59cc4bd59d37) 发布，rootless 容器全面可用。

![](https://miro.medium.com/1*41peAl7SSpEZqQGpkRmmUw.png)

从 2020 年到 2022 年，我们还致力于 [bypass4netns](https://github.com/rootless-containers/bypass4netns)，通过在容器内挂钩套接字文件描述符并在容器外重建它们来消除 slirp4netns 的开销。所实现的吞吐量甚至比 “rootful” 容器更快。

![](https://miro.medium.com/1*h5_2ZjoGixRrOfZdOu8ysA.png)

Rootless 容器已经成功普及，但也有人对 rootless 容器提出批评。特别是，是否应该允许非root用户创建运行无根容器所需的用户命名空间是有争议的。对于容器用户，我的回答是“是”，因为无根容器至少比以根身份运行所有内容要安全得多。但是，对于不使用容器的人，我宁愿回答“否”，因为用户命名空间也可能是攻击面。例如，[**CVE-2023–32233 漏洞**：“Privilege escalation in Linux Kernel due to a Netfilter nf_tables vulnerability.”](https://www.tarlogic.com/blog/cve-2023-32233-vulnerability/)。

社区已经在寻求解决这一困境的方法。Ubuntu（自 13.10 起）和 Debian 提供了一个 sysctl 设置 `kernel.unprivileged_userns_clone=<bool>` 来指定是否允许或禁止创建非特权用户命名空间。然而，他们的[补丁](https://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux/+git/jammy/commit/kernel/user_namespace.c?id=342276469714b5a307745d1a3b9bdc146c804e4e)并没有合并到上游 Linux 内核中。

相反，上游内核在 Linux [v6.1](https://github.com/torvalds/linux/commit/7cd4c5c2101cb092db00f61f69d24380cf7a0ee8) (2022) 中引入了新的 LSM（Linux 安全模块）钩子 `userns_create` ，以便 LSM 可以动态决定是否允许或禁止创建用户命名空间。该钩子可从 [eBPF (`bpf_program__atttach_lsm()`)](https://docs.kernel.org/bpf/prog_lsm.html) 调用，因此预计将有一个不依赖于 AppArmor 或 SELinux 的细粒度且非特定于发行版的旋钮。然而，eBPF + LSM 的用户空间实用程序尚未成熟，无法为此提供良好的用户体验。


### 更多 LSM

[Landlock](https://landlock.io/) LSM 已合并到 Linux [v5.13](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=17ae69aba89dbfa2139b7f8024b757ab3cc42f59) (2021) 中。Landlock 与 AppArmor 类似，它通过路径（`LANDLOCK_ACCESS_FS_EXECUTE`、`LANDLOCK_ACCESS_FS_READ_FILE` 等）限制文件访问，但 Landlock 不需要 root 权限来设置新配置文件。Landlock 也与 OpenBSD 的 [`promise(2)`](https://man.openbsd.org/pledge.2) 非常相似。

Landlock 仍然 [不受 OCI Runtime Spec 支持](https://github.com/opencontainers/runtime-spec/pull/1111)，但我猜它可以包含在 OCI Runtime Spec v1.2 中。


### Kata Containers


正如我在第一部分中提到的，“容器”并不是一个定义明确的术语。任何东西只要能与现有的容器生态系统提供良好的兼容性，就可以称为“容器”。

[Kata Containers](https://katacontainers.io/) (2017) 就是这样一种“容器”，实际上并不是狭义上的容器。Kata 容器实际上是虚拟机，但支持 OCI 运行时规范。Kata 容器比 runc 容器安全得多，但是它们在性能方面存在缺陷，并且在不支持嵌套虚拟化的典型非裸机 IaaS 实例上无法正常工作。

Kata Containers 作为一个 containerd 运行时插件，并接收与 runc 容器相同的镜像和运行时配置。它的用户体验与 runc 容器几乎没有区别。


### gVisor

[gVisor](https://gvisor.dev/) (2018) 是另一个奇特的容器运行时。gVisor 捕获系统调用并在 Linux 兼容的用户模式内核中执行它们以减轻攻击。gVisor 目前具有 [三种](https://gvisor.dev/docs/architecture_guide/platforms/) 捕获系统调用的模式：

- **KVM 模式**：很少使用，但是裸机主机的最佳选择
- **ptrace 模式**：最常见的选项，但速度较慢
- **SIGSYS trap 模式**（自 2023 年起）：预计最终取代 ptrace 模式

gVisor 已用于 Google 的多个产品中，包括 Google Cloud Run。然而，Google Cloud Run 已于 2023 年从 gVisor 转向 microVM。这意味着 gVisor 的性能和兼容性问题对于他们的业务来说是不可忽视的。


### WebAssembly

[WebAssembly (WASM) ](https://webassembly.org/) 是一种独立于平台的字节代码格式，最初于 [2015 年](https://blog.mozilla.org/luke/2015/06/17/webassembly/) 为 Web 浏览器设计。WebAssembly 与 Java applet (1995) 有点相似，但它更注重可移植性和安全性。WebAssembly 的一个有趣的方面是它将代码地址空间与数据地址空间分开；没有像 `JMP <immediate>` 和 `JMP *<reg>` 这样的指令。它仅支持 [跳转到在编译时解析的标签](https://webassembly.github.io/spec/core/syntax/instructions.html#control-instructions)。这种设计减少了任意代码执行错误，尽管它也牺牲了 JIT 将其他字节代码格式编译为 WebAssembly 的可行性。

WebAssembly 作为容器的潜在替代品也受到关注。为了在浏览器之外运行 WebAssembly，[WASI](https://wasi.dev/)（WebAssembly 系统接口）于 2019 年提出，提供低级 API（例如 [`fd_read()`、`fd_write()`、`sock_recv()`、`sock_send()`](https://github.com/WebAssembly/WASI/blob/main/legacy/preview1/docs.md)）可用于在其上实现类似 POSIX 的层。containerd 在 2022 年添加了 [runWASI](https://github.com/containerd/runwasi) 插件，将 WASI 工作负载视为容器。

2023年，[WASIX](https://wasix.org/docs/api-reference) 被提议扩展 WASI 以提供更方便（也有些争议）的功能：

- **线程**：[`thread_spawn()`](https://wasix.org/docs/api-reference/wasix/thread_spawn), [`thread_join()`](https://wasix.org/docs/api-reference/wasix/thread_join)`, ...
- **进程：** [`proc_fork()`](https://wasix.org/docs/api-reference/wasix/proc_fork), [`proc_exec()`](https://wasix.org/docs/api-reference/wasix/proc_exec), ...
- **套接字**：[`sock_listen()`](https://wasix.org/docs/api-reference/wasix/sock_listen), [`sock_connect()`](https://wasix.org/docs/api-reference/wasix/sock_connect), ...

最终，这些技术可能会取代很大一部分（但不是 100%）的容器。Docker 的创始人 Solomon Hykes 表示：“*如果 WASM+WASI 在 2008 年就存在，我们就不需要创建 Docker 了* ”。


## 总结

- 容器比虚拟机更高效，但安全性往往也更低。人们正在引入许多安全技术来强化容器。（用户命名空间、无根容器、Linux 安全模块……）
- Docker 的替代品不断涌现（containerd、CRI-O、Podman、nerdctl、Finch 等），但 Docker 并没有消失。
- “Non-container” 容器也是趋势。（**Kata**：基于 VM，**gVisor**：用户模式内核，**runWASI**：WebAssembly，...）

下图显示了著名的运行时的概况。

![](https://miro.medium.com/1*_x0ujgxNUyzBIco_J6O-mw.png)

更多内容另请参阅 [PPT](https://github.com/AkihiroSuda/AkihiroSuda/raw/5d9f0b1cd9b8c37cb1951768a3bebdb08a3a469e/slides/2023/20230615%20%5BKyoto%20University%5D%20The%20internals%20and%20the%20latest%20trends%20of%20container%20runtimes.pdf) 的其余部分，了解本文中无法涵盖的其他主题。


---

*文本翻译自: https://medium.com/nttlabs/the-internals-and-the-latest-trends-of-container-runtimes-2023-22aa111d7a93*

---