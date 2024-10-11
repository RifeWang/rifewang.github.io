+++
draft = false
date = 2024-10-11T16:46:19+08:00
title = "Kubernetes：Seccomp、AppArmor、SELinux & Pod 安全性标准和准入"
description = "Kubernetes：Seccomp、AppArmor、SELinux & Pod 安全性标准和准入"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

在云原生环境中，为确保容器化应用的安全运行，Kubernetes 利用了 Linux 内核的三大安全机制：`Seccomp`、`AppArmor` 和 `SELinux`，并引入了 Pod 安全性标准与准入控制来增强 Pod 的安全性。

## Seccomp、AppArmor、SELinux 简介

`Seccomp`、`AppArmor` 和 `SELinux` 是 Linux 内核提供的三种安全机制：
- `Seccomp`（Secure Computing）：限制程序的系统调用（syscall）。
- `AppArmor`：限制程序对特定资源的访问。
- `SELinux`（Security-Enhanced Linux）：使用标签和策略限制对资源的访问。

### Seccomp

在 Kubernetes 中，`Seccomp` 通过 Pod 或 Container 的 `securityContext.seccompProfile` 字段进行配置，有三种类型：
- `Unconfined`：无限制。
- `RuntimeDefault`：使用容器运行时（如 containerd / CRI-O）的默认配置。
- `Localhost`：使用节点本地的配置文件。

示例：
![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/security/seccomp.png)

### AppArmor

`AppArmor` 在 v1.30 之前是通过注解的方式，现在则是通过配置 Pod 或 Container 的 `securityContext.appArmorProfile` 字段，同样是三种类型：
- `Unconfined`：无任何限制。
- `RuntimeDefault`：使用容器运行时（如 containerd / CRI-O）的默认配置。
- `Localhost`：使用节点上的配置文件。

示例：
![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/security/apparmor.png)

上图右侧 `AppArmor` 的配置文件有自己特定的规则，所以看上去有点奇怪。

### SELinux

`SELinux` 的安全上下文格式为 `user:role:type:level`（用户、角色、类型、范围），在 k8s 中对应 Pod 或 Container 的 `securityContext.seLinuxOptions`：
```yaml
...
securityContext:
  seLinuxOptions:
    user: unconfined_u
    role: system_r
    type: container_t
    level: "s0:c123,c456"
```

以上就是 `Seccomp`、`AppArmor` 和 `SELinux` 的基本作用以及在 Kubernetes 中的使用方法。


##  Pod 安全性标准和准入控制

### Pod 安全性标准

Kubernetes 制定了 Pod 安全性标准（Pod Security Standard），并划分了三个不同的安全级别：
- `Privileged`：特权级，几乎无限制。
- `Baseline`：基准级，弱限制：
    - 禁止使用宿主机 hostNetwork、hostPID、hostIPC、hostPath、hostPort。
    - 禁止使用特权容器，只允许部分 capabilities 权能。
    - 对 `Seccomp`、`AppArmor` 和 `SELinux` 有要求。
- `Restricted`：限制级，强限制：
    - 包括 `Baseline` 的全部要求。
    - 只允许特定的 volumes 卷类型。
    - 容器必须以非 root 用户运行，并进一步限制 capabilities 权能。

不同安全级别对应的具体 spec 规则清单请参考官方文档。

### Pod 安全性准入控制

Kubernetes 提供了一个内置的 Pod 安全准入控制器来执行 Pod 安全性标准。 

用户可以在不同的命名空间设置不同的安全策略（通过配置标签），例如：
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-baseline-namespace
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: v1.31
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.31
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.31
```

其中，标签的格式统一为：
```yaml
# MODE 必须是 `enforce`、`audit`、`warn` 其中之一
# LEVEL 对应三个安全性标准，必须是 `privileged`、baseline`、`restricted` 其中之一
pod-security.kubernetes.io/<MODE>: <LEVEL>

# VERSION 必须是一个合法的 Kubernetes 小版本号或者 `latest`
pod-security.kubernetes.io/<MODE>-version: <VERSION>
```

当 MODE 是：
- `enforce`：不满足安全性标准规则的 Pod 会被拒绝。
- `audit`：接受 Pod，但会记录审计日志。
- `warn`：接受 Pod，但会显示警告信息。


用户也可以在准入控制器配置中设置 `exemptions` 豁免规则，从而绕过安全性标准的检查，例如：
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "privileged"
      enforce-version: "latest"
      audit: "privileged"
      audit-version: "latest"
      warn: "privileged"
      warn-version: "latest"
    exemptions:
      # 要豁免的已认证用户名列表
      usernames: []
      # 要豁免的运行时类名称列表
      runtimeClasses: []
      # 要豁免的名字空间列表
      namespaces: []
```

注意，这个配置需要通过 `——admission-control-config-file` 应用于 kube-apiserver。


## 总结

Kubernetes 使用 Linux 内核的 `Seccomp`、`AppArmor` 和 `SELinux` 安全机制，通过限制系统调用和资源访问来提升容器的安全性。
Pod 安全性标准则进一步为集群管理员提供了预定义的安全策略，用于限制不同 Pod 的权限和行为。结合 Pod 安全性准入控制器，管理员可以有效地管理集群中的 Pod 安全性，确保工作负载在云原生环境中安全运行。


(关注我，无广告，专注于技术，不煽动情绪)

---

参考资料：

- *https://kubernetes.io/zh-cn/docs/concepts/security/linux-kernel-security-constraints/*
- *https://kubernetes.io/docs/reference/node/seccomp/*
- *https://kubernetes.io/docs/tutorials/security/apparmor/*
- *https://kubernetes.io/zh-cn/docs/concepts/security/pod-security-standards/*