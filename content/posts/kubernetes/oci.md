+++
draft = false
date = 2024-09-26T14:30:07+08:00
title = "OCI 简介：Kubernetes 环境下从代码到容器的全流程"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
disableComments = true
+++

## OCI 简介

在容器化技术的演进中，`OCI`（Open Container Initiative）提供了一套标准化的规范，帮助统一容器的构建、分发和运行。`OCI` 规范包含三个部分：
- `OCI Image-spec`：定义了容器镜像的结构，确保镜像可以被各种工具识别和管理。
- `OCI Distribution-spec`：定义了容器镜像的分发方式，保证镜像可以通过标准化的 HTTP API 与镜像仓库交互。
- `OCI Runtime-spec`：定义了容器的运行标准，确保容器运行时可以统一执行和管理容器生命周期。

下文将结合从代码到容器的完整 CICD 流程，详细讲解每个 OCI 规范的具体作用。

## 从代码到容器

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/k8s-oci-flow.drawio.png)

### 1. Code -> OCI Image

代码被集成之后，便可以使用 `docker`、`podman`/`buildah` 或者 `kaniko` 等工具来构建容器镜像。

此时，`OCI Image-spec` 便发挥了重要的作用，该规范定义了镜像的组成部分，使得不同工具能够跨平台识别和使用同一个镜像。

由于 docker 对行业的影响力，部分从业者会认为 image 就是 docker image，这其实并不对，准确来说，docker 只是工具之一，不管是使用什么工具构建出来的镜像应该统称为 OCI image。

`OCI Image-spec` 规范定义了镜像的结构，一个 image 至少包括：
- `Manifest`：元数据文件，描述镜像的组成部分。
- `Layers`：层，镜像由多层堆叠组成，每一层表示文件系统的增量变化。
- `Configuration`：配置文件，描述环境变量、启动命令、工作目录等信息。

让我们看看 busybox 这个镜像的结构：

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/k8s-OCI-Image.drawio.png)

从上图中可以看到，manifest 元数据文件、config 配置文件、layers 层文件一应俱全。

一个 image 其实也只是一堆文件和文件夹的集合。

### 2. OCI Image -> Registry

在镜像构建完成后，需要推送到镜像仓库。`OCI Distribution-spec` 规范则定义了镜像如何通过标准的 HTTP API 在仓库中存储、拉取和分发。

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/oci-distribution-spec-api.png)

`OCI Distribution-spec` 规范确保了不同的仓库和客户端可以使用统一的 API 进行镜像的分发。这样仓库既可以使用公共仓库（如 Docker Hub），也可以使用私有仓库（如 Harbor）。而客户端只要使用规范中的 HTTP 请求则都可以上传或下载镜像。

### 3. Kube-API-Server -> Kubelet -> Containerd/CRI-O

镜像上传完成之后，便可以创建或更新应用了。所有请求统一先由 `kube-api-server` 处理，然后 `scheduler` 将 Pod 调度到节点上，随后节点上的 `kubelet` 负责后续处理。

`kubelet` 通过 `CRI gRPC` 与具体的组件 `Containerd` 或 `CRI-O` 进行交互，由后者完成镜像的下载和容器的创建。

### 4. Containerd/CRI-O -> Runc/Kata-containers

`Containerd` 和 `CRI-O` 本身并不直接负责容器的创建和运行，它们会通过 `OCI Runtime-spec` 与底层容器执行工具 `runc` 或 `kata-containers` 交互，由 `runc` 或 `kata-containers` 完成容器的执行。

`Runc` 和 `Kata-containers`：
- `runc` 是最常见的容器执行工具，直接与 Linux 内核交互，通过 cgroups 和 namespaces 实现容器的隔离和资源管理。
- `kata-containers` 则是在虚拟机中运行容器，提供更强的隔离性，适用于需要高安全性的场景。它仍然遵循 OCI runtime-spec，但运行环境更加虚拟化。

`OCI runtime-spec` 规范规定了：
- 容器的配置：通过 config.json 配置文件描述了容器的进程、挂载、hooks 钩子、资源限制等等信息。
- 执行环境：如何保证环境的一致性，包括将镜像解压到运行时 filesystem bundle 文件系统包中。
- 生命周期：明确了容器从创建到消失期间的详细过程。

如何查看 `OCI runtime-spec` 规定的 config.json 文件？
- 先找到 Pod 所在的 Node 节点，然后 SSH 登录到该节点
- 再使用 CRI 工具，如 `crictl ps` 找到容器 ID。
- 如果是 containerd 则文件位置是 `/run/containerd/io.containerd.runtime.v2.task/k8s.io/<container-id>/config.json`；在 cri-o 中则可能是 `/run/containers/storage/<container-id>/userdata/config.json`。

我创建了一个 busybox ，它的 `OCI runtime-spec` config.json 如下图所示：

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/oci-runtime-spec-config.png)

而这个 `OCI runtime-spec` 规定的 config.json 其实就是 `Containerd` 或 `CRI-O` 根据 Pod 的定义（命令及参数、环境变量、资源限制等等）和上下文等信息生成的。


## 总结

从代码到容器实际运行，整个过程中多个组件和标准紧密协作，本文重点关注了 OCI 的三个核心规范（`image-spec`、`distribution-spec` 和 `runtime-spec`），这些规范确保了容器在不同环境中的兼容性和可移植性，正如 OCI 这个名字的含义一样 —— 开放容器倡议。


(关注我，无广告，专注于技术，不煽动情绪)

---

参考资料：

- *https://opencontainers.org/about/overview/*
- *https://github.com/opencontainers/image-spec/blob/main/spec.md*
- *https://github.com/opencontainers/runtime-spec/blob/main/spec.md*
- *https://github.com/opencontainers/distribution-spec/blob/main/spec.md*
- *https://github.com/opencontainers/runc*
