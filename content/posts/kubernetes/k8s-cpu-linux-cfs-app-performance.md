+++
draft = false
date = 2024-12-11T15:44:11+08:00
title = "Kubernetes：CPU 配置、Linux CFS、编程语言的性能问题"
description = "Kubernetes：CPU 配置、Linux CFS、编程语言的性能问题"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

## Kubernetes CPU 配置 -> Linux CFS

在使用 Kubernetes 时，可以通过 `resources.requests` 和 `resources.limits` 配置资源的请求和限额，例如：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "250m"
      limits:
        cpu: "500m"
```

对容器的资源配置会通过 `CRI` 组件（如 `containerd`、`cri-o` 交由更底层的 `runc` 或 `kata-container`）去设置 Linux 的 cgroup。

在 cgroup v1 中（目前仍然是主流版本，v2 则正在发展）：
- `requests.cpu` 对应为 cgroup 的 `cpu.shares`。`cpu.shares = 1024` 表示一个核 CPU，`requests.cpu = 250m` 表示 0.25 核，对应的 `cpu.shares = 1024 * 0.25 = 256`。此项配置只作用在 CPU 繁忙时，决定如何给多个容器按比例分配 CPU 时间。

- `limits.cpu` 对应为 cgroup 的：
    - `cpu.cfs_period_us`：表示 CPU 调度周期，一般是 100000 us，即 100 ms。
    - `cpu.cfs_quota_us`：表示在一个周期内，容器最多可以使用的 CPU 时间。`limits.cpu = 500m` 表示 0.5 核，对应的 `cpu.cfs_quota_us = 100000 * 0.5 = 50000`。

以上配置可以进入该容器中的 `/sys/fs/cgroup/cpu/` 目录查看，也可以直接查看宿主机上的 `/sys/fs/cgroup/cpu/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod<uid>.slice/cri-containerd-<uid>.scope/` 目录。

此时可以看到，在容器环境下，对 CPU 的限额是通过 Linux 的 CFS 机制来完成的。由此很自然地引出了一个问题：容器里的应用程序/编程语言拿到的 CPU 核数是什么？


## 容器环境中的编程语言

如果通过 CPU 核数去设置协程/线程/进程数时，可能会发生意料之外的性能问题。

在 Golang 中，通过 `GPM`（`Goroutine-Processor-Machine`）模式调度 `goroutine`，而 processor 的数量取自 `GOMAXPROCS`，`GOMAXPROCS` 的默认值则是 `runtime.NumCPU()` 拿到的宿主机 CPU 核数。这可能导致应用程序出现更大的延迟，容器配额越小、宿主机资源越大时影响越糟糕。解决方式就是设置 `GOMAXPROCS=max(1, floor(cpu_quota))`，或者直接使用 `uber-go/automaxprocs` 这个库。更多信息可参考：[https://github.com/golang/go/issues/33803](https://github.com/golang/go/issues/33803)。

在 Java 中，JVM 的垃圾回收机制与 Linux CFS 调度器的相互作用，也可能导致更长的 STW（stop-the-word）。LinkedIn 工程师在博文 [Application Pauses When Running JVM Inside Linux Control Groups](https://www.linkedin.com/blog/engineering/archive/application-pauses-when-running-jvm-inside-linux-control-groups) 中则建议提供足够的 CPU 配额，并且应该根据场景调低 GC 线程。

更好的方式显然是消除模糊，根据容器的资源配置，明确地设置相关影响值。


## 总结

Kubernetes 工作负载的 CPU 配额决定了 Linux CFS 的行为，进而有可能导致编程语言意料之外的性能问题。


---

(我是凌虚，关注我，无广告，专注技术，不煽动情绪，欢迎与我交流)

---

参考资料：

- *https://github.com/golang/go/issues/33803*
- *https://www.linkedin.com/blog/engineering/archive/application-pauses-when-running-jvm-inside-linux-control-groups*
- *https://www.kernel.org/doc/Documentation/scheduler/sched-bwc.txt*