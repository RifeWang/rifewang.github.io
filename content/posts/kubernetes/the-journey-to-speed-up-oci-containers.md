+++
draft = false
date = 2023-03-28T16:16:36+08:00
title = "提速 30 倍！OCI 容器启动优化的历程"
description = "提速 30 倍！OCI 容器启动优化的历程"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

---

*文本翻译自: https://www.scrivano.org/posts/2022-10-21-the-journey-to-speed-up-oci-containers/*

---

> 原文作者是 Red Hat 工程师 [Giuseppe Scrivano](https://github.com/giuseppe/) ，其回顾了将 OCI 容器启动的时间提速 30 倍的历程。

当我开始研究 crun (https://github.com/containers/crun) 时，我正在寻找一种通过改进 OCI 运行时来更快地启动和停止容器的方法，OCI 运行时是 OCI 堆栈中负责最终与内核交互并设置容器所在环境的组件。

OCI 运行时的运行时间非常有限，它的工作主要是执行一系列直接映射到 OCI 配置文件的系统调用。

我很惊讶地发现，如此琐碎的任务可能需要花费这么长时间。

***免责声明***：对于我的测试，我使用了 Fedora 安装中可用的默认内核以及所有库。除了这篇博文中描述的修复之外，这些年来可能还有其他可能影响整体性能的修复。

以下所有用于测试的 crun 版本都是相同的。

对于所有测试，我都使用 [hyperfine](https://github.com/sharkdp/hyperfine)，它是通过 cargo 安装的。

## 2017年的情况如何

要对比我们与过去相差多大，我们需要回到 2017 年，或者只安装一个旧的 Fedora 映像。对于下面的测试，我使用了基于 Linux 内核 4.5.5 的 Fedora 24。

在新安装的 Fedora 24 上，运行从主分支构建：

```shell
# hyperfine 'crun run foo'
Benchmark 1: 'crun run foo'
  Time (mean ± σ):     159.2 ms ±  21.8 ms    [User: 43.0 ms, System: 16.3 ms]
  Range (min … max):    73.9 ms … 194.9 ms    39 runs
```

> 用户时间和系统时间指的是进程分别在用户态和内核态的耗时。

160 毫秒很多，据我所知，这与我五年前观察到的情况相似。

对 OCI 运行时的分析立即表明，大部分用户时间都花在了 libseccomp 上来编译 seccomp 过滤器。

为了验证这一点，让我们尝试运行一个具有相同配置但没有 seccomp 配置文件的容器：

```shell
# hyperfine 'crun run foo'
Benchmark 1: 'crun run foo'
  Time (mean ± σ):     139.6 ms ±  20.8 ms    [User: 4.1 ms, System: 22.0 ms]
  Range (min … max):    61.8 ms … 177.0 ms    47 runs
```

我们使用了之前所需用户时间的 1/10（43 ms -> 4.1 ms），整体时间也有所改善！

所以主要有两个不同的问题：1) 系统时间相当长，2) 用户时间由 libseccomp 控制。我们需要同时解决这两个问题。

现在让我们专注于系统时间，稍后我们将回到 seccomp。


## 系统时间

### 创建和销毁 network 命名空间

创建和销毁网络命名空间曾经非常昂贵，只需使用该 `unshare` 工具即可重现该问题，在 Fedora 24 上我得到：

```shell
# hyperfine 'unshare -n true'
Benchmark 1: 'unshare -n true'
  Time (mean ± σ):      47.7 ms ±  51.4 ms    [User: 0.6 ms, System: 3.2 ms]
  Range (min … max):     0.0 ms … 190.5 ms    365 runs
```

这算是很长的耗时！

我试图在内核中修复它并提出了一个 [patch 补丁](https://www.spinics.net/lists/netfilter-devel/msg50394.html)。Florian Westphal 以更好的方式将其进行了重写，并合并到了 Linux 内核中：

```text
commit 8c873e2199700c2de7dbd5eedb9d90d5f109462b
Author: Florian Westphal
Date:   Fri Dec 1 00:21:04 2017 +0100

    netfilter: core: free hooks with call_rcu

    Giuseppe Scrivano says:
      "SELinux, if enabled, registers for each new network namespace 6
        netfilter hooks."

    Cost for this is high.  With synchronize_net() removed:
       "The net benefit on an SMP machine with two cores is that creating a
       new network namespace takes -40% of the original time."

    This patch replaces synchronize_net+kvfree with call_rcu().
    We store rcu_head at the tail of a structure that has no fixed layout,
    i.e. we cannot use offsetof() to compute the start of the original
    allocation.  Thus store this information right after the rcu head.

    We could simplify this by just placing the rcu_head at the start
    of struct nf_hook_entries.  However, this structure is used in
    packet processing hotpath, so only place what is needed for that
    at the beginning of the struct.

    Reported-by: Giuseppe Scrivano
    Signed-off-by: Florian Westphal
    Signed-off-by: Pablo Neira Ayuso

commit 26888dfd7e7454686b8d3ea9ba5045d5f236e4d7
Author: Florian Westphal
Date:   Fri Dec 1 00:21:03 2017 +0100

    netfilter: core: remove synchronize_net call if nfqueue is used

    since commit 960632ece6949b ("netfilter: convert hook list to an array")
    nfqueue no longer stores a pointer to the hook that caused the packet
    to be queued.  Therefore no extra synchronize_net() call is needed after
    dropping the packets enqueued by the old rule blob.

    Signed-off-by: Florian Westphal
    Signed-off-by: Pablo Neira Ayuso

commit 4e645b47c4f000a503b9c90163ad905786b9bc1d
Author: Florian Westphal
Date:   Fri Dec 1 00:21:02 2017 +0100

    netfilter: core: make nf_unregister_net_hooks simple wrapper again

    This reverts commit d3ad2c17b4047
    ("netfilter: core: batch nf_unregister_net_hooks synchronize_net calls").

    Nothing wrong with it.  However, followup patch will delay freeing of hooks
    with call_rcu, so all synchronize_net() calls become obsolete and there
    is no need anymore for this batching.

    This revert causes a temporary performance degradation when destroying
    network namespace, but its resolved with the upcoming call_rcu conversion.

    Signed-off-by: Florian Westphal
    Signed-off-by: Pablo Neira Ayuso
```

这些补丁产生了巨大的差异，现在创建和销毁网络命名空间的时间已经下降到了一个难以置信的地步，以下是一个现代 5.19.15 内核的数据：

```shell
# hyperfine 'unshare -n true'
Benchmark 1: 'unshare -n true'
  Time (mean ± σ):       1.5 ms ±   0.5 ms    [User: 0.3 ms, System: 1.3 ms]
  Range (min … max):     0.8 ms …   6.7 ms    1907 runs
```


### 挂载 mqueue

挂载 mqueue 也是一个相对昂贵的操作。

在 Fedora 24 上，它曾经是这样的：

```shell
# mkdir /tmp/mqueue; hyperfine 'unshare --propagation=private -m mount -t mqueue mqueue /tmp/mqueue'; rmdir /tmp/mqueue
Benchmark 1: 'unshare --propagation=private -m mount -t mqueue mqueue /tmp/mqueue'
  Time (mean ± σ):      16.8 ms ±   3.1 ms    [User: 2.6 ms, System: 5.0 ms]
  Range (min … max):     9.3 ms …  26.8 ms    261 runs
```

在这种情况下，我也尝试修复它并提出一个 [补丁](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg1545810.html)。它没有被接受，但 Al Viro 想出了一个更好的版本来解决这个问题：

```text
commit 36735a6a2b5e042db1af956ce4bcc13f3ff99e21
Author: Al Viro
Date:   Mon Dec 25 19:43:35 2017 -0500

    mqueue: switch to on-demand creation of internal mount

    Instead of doing that upon each ipcns creation, we do that the first
    time mq_open(2) or mqueue mount is done in an ipcns.  What's more,
    doing that allows to get rid of mount_ns() use - we can go with
    considerably cheaper mount_nodev(), avoiding the loop over all
    mqueue superblock instances; ipcns->mq_mnt is used to locate preexisting
    instance in O(1) time instead of O(instances) mount_ns() would've
    cost us.

    Based upon the version by Giuseppe Scrivano ; I've
    added handling of userland mqueue mounts (original had been broken in
    that area) and added a switch to mount_nodev().

    Signed-off-by: Al Viro
```

在这个补丁之后，创建 mqueue 挂载的成本也下降了：

```shell
# mkdir /tmp/mqueue; hyperfine 'unshare --propagation=private -m mount -t mqueue mqueue /tmp/mqueue'; rmdir /tmp/mqueue
Benchmark 1: 'unshare --propagation=private -m mount -t mqueue mqueue /tmp/mqueue'
  Time (mean ± σ):       0.7 ms ±   0.5 ms    [User: 0.5 ms, System: 0.6 ms]
  Range (min … max):     0.0 ms …   3.1 ms    772 runs
```

### 创建和销毁 IPC 命名空间

我将加速容器启动时间的事推迟了几年，并在 2020 年初重新开始。我意识到的另一个问题是创建和销毁 IPC 命名空间的时间。

与网络命名空间一样，仅使用以下 `unshare` 工具即可重现该问题：

```shell
# hyperfine 'unshare -i true'
Benchmark 1: 'unshare -i true'
  Time (mean ± σ):      10.9 ms ±   2.1 ms    [User: 0.5 ms, System: 1.0 ms]
  Range (min … max):     4.2 ms …  17.2 ms    310 runs
```

与前两次尝试不同，这次我发送的补丁被上游接受了：

```text
commit e1eb26fa62d04ec0955432be1aa8722a97cb52e7
Author: Giuseppe Scrivano
Date:   Sun Jun 7 21:40:10 2020 -0700

    ipc/namespace.c: use a work queue to free_ipc

    the reason is to avoid a delay caused by the synchronize_rcu() call in
    kern_umount() when the mqueue mount is freed.

    the code:

        #define _GNU_SOURCE
        #include
        #include
        #include
        #include

        int main()
        {
            int i;

            for (i = 0; i < 1000; i++)
                if (unshare(CLONE_NEWIPC) < 0)
                    error(EXIT_FAILURE, errno, "unshare");
        }

    goes from

            Command being timed: "./ipc-namespace"
            User time (seconds): 0.00
            System time (seconds): 0.06
            Percent of CPU this job got: 0%
            Elapsed (wall clock) time (h:mm:ss or m:ss): 0:08.05

    to

            Command being timed: "./ipc-namespace"
            User time (seconds): 0.00
            System time (seconds): 0.02
            Percent of CPU this job got: 96%
            Elapsed (wall clock) time (h:mm:ss or m:ss): 0:00.03

    Signed-off-by: Giuseppe Scrivano
    Signed-off-by: Andrew Morton
    Reviewed-by: Paul E. McKenney
    Reviewed-by: Waiman Long
    Cc: Davidlohr Bueso
    Cc: Manfred Spraul
    Link: http://lkml.kernel.org/r/20200225145419.527994-1-gscrivan@redhat.com
    Signed-off-by: Linus Torvalds
```

有了这个补丁，创建和销毁 IPC 的时间也大大减少了，正如提交消息中所概述的那样，在我现在得到的现代 5.19.15 内核上：

```shell
# hyperfine 'unshare -i true'
Benchmark 1: 'unshare -i true'
  Time (mean ± σ):       0.1 ms ±   0.2 ms    [User: 0.2 ms, System: 0.4 ms]
  Range (min … max):     0.0 ms …   1.5 ms    1966 runs
```

## 用户时间

内核态时间现在似乎已得到控制。我们可以做些什么来减少用户时间？

正如我们之前已经发现的，libseccomp 是这里的罪魁祸首，因此我们需要首先解决它，这发生在内核中对 IPC 的修复之后。

libseccomp 的大部分成本都是由系统调用查找代码引起的。OCI 配置文件包含一个按名称列出系统调用的列表，每个系统调用通过 `seccomp_syscall_resolve_name` 函数调用进行查找，该函数返回给定系统调用名称的系统调用编号。

libseccomp 用于通过系统调用表对每个系统调用名称执行线性搜索，例如，对于 x86_64，它看起来像这样：

```c
/* NOTE: based on Linux v5.4-rc4 */
const struct arch_syscall_def x86_64_syscall_table[] = { \
	{ "_llseek", __PNR__llseek },
	{ "_newselect", __PNR__newselect },
	{ "_sysctl", 156 },
	{ "accept", 43 },
	{ "accept4", 288 },
	{ "access", 21 },
	{ "acct", 163 },
.....
    };

int x86_64_syscall_resolve_name(const char *name)
{
	unsigned int iter;
	const struct arch_syscall_def *table = x86_64_syscall_table;

	/* XXX - plenty of room for future improvement here */
	for (iter = 0; table[iter].name != NULL; iter++) {
		if (strcmp(name, table[iter].name) == 0)
			return table[iter].num;
	}

	return __NR_SCMP_ERROR;
}
```

通过 libseccomp 构建 seccomp 配置文件的复杂度为 `O(n*m)`，其中 `n` 是配置文件中的系统调用数量，`m` 是 libseccomp 已知的系统调用数量。

我遵循了代码注释中的建议，并花了一些时间尝试修复它。2020 年 1 月，我为 libseccomp 开发了一个 [补丁](https://github.com/seccomp/libseccomp/pull/204)，以使用完美的哈希函数查找系统调用名称来解决这个问题。

libseccomp 的补丁是这个：

```text
commit 9b129c41ac1f43d373742697aa2faf6040b9dfab
Author: Giuseppe Scrivano
Date:   Thu Jan 23 17:01:39 2020 +0100

    arch: use gperf to generate a perfact hash to lookup syscall names

    This patch significantly improves the performance of
    seccomp_syscall_resolve_name since it replaces the expensive strcmp
    for each syscall in the database, with a lookup table.

    The complexity for syscall_resolve_num is not changed and it
    uses the linear search, that is anyway less expensive than
    seccomp_syscall_resolve_name as it uses an index for comparison
    instead of doing a string comparison.

    On my machine, calling 1000 seccomp_syscall_resolve_name_arch and
    seccomp_syscall_resolve_num_arch over the entire syscalls DB passed
    from ~0.45 sec to ~0.06s.

    PM: After talking with Giuseppe I made a number of additional
    changes, some substantial, the highlights include:
    * various style tweaks
    * .gitignore fixes
    * fixed subject line, tweaked the description
    * dropped the arch-syscall-validate changes as they were masking
      other problems
    * extracted the syscalls.csv and file deletions to other patches
      to keep this one more focused
    * fixed the x86, x32, arm, all the MIPS ABIs, s390, and s390x ABIs as
      the syscall offsets were not properly incorporated into this change
    * cleaned up the ABI specific headers
    * cleaned up generate_syscalls_perf.sh and renamed to
      arch-gperf-generate
    * fixed problems with automake's file packaging

    Signed-off-by: Giuseppe Scrivano
    Reviewed-by: Tom Hromatka
    [PM: see notes in the "PM" section above]
    Signed-off-by: Paul Moore
```

该补丁已合并并发布，现在构建 seccomp 配置文件的复杂度为 `O(n)`，其中 n 是配置文件中系统调用的数量。

改进是显着的，在足够新的 libseccomp 下：

```shell
# hyperfine 'crun run foo'
Benchmark 1: 'crun run foo'
  Time (mean ± σ):      28.9 ms ±   5.9 ms    [User: 16.7 ms, System: 4.5 ms]
  Range (min … max):    19.1 ms …  41.6 ms    73 runs
```

用户时间仅为 16.7ms。以前是 40ms 以上，完全不用 seccomp 的时候是 4ms 左右。

所以使用 4.1ms 作为没有 seccomp 的用户时间成本，我们有：

```shell
time_used_by_seccomp_before = 43.0ms - 4.1ms = 38.9ms
time_used_by_seccomp_after = 16.7ms - 4.1ms = 12.6ms
```

快 3 倍以上！系统调用查找只是 libseccomp 所做工作的一部分，另外相当多的时间用于编译 BPF 过滤器。

## BPF 过滤器编译

我们还能做得更好吗？

BPF 过滤器编译由 `seccomp_export_bpf` 函数完成，它仍然相当昂贵。

一个简单的观察是，大多数容器一遍又一遍地重复使用相同的 seccomp 配置文件，很少进行自定义。

因此缓存编译结果并在可能的情况下重用它是有意义的。

有一个[新的运行特性](https://github.com/containers/crun/pull/1035) 来缓存 BPF 过滤器编译的结果。在撰写本文时，该补丁尚未合并，尽管它快要完成了。

有了这个，只有当生成的 BPF 过滤器不在缓存中时，编译 seccomp 配置文件的成本才会被支付，这就是我们现在所拥有的：

```shell
# hyperfine 'crun-from-the-future run foo'
Benchmark 1: 'crun-from-the-future run foo'
  Time (mean ± σ):       5.6 ms ±   3.0 ms    [User: 1.0 ms, System: 4.5 ms]
  Range (min … max):     4.2 ms …  26.8 ms    101 runs
```

## 结论

五年多来，创建和销毁 OCI 容器所需的总时间已从将近 160 毫秒加速到略多于 5 毫秒。

这几乎是 30 倍的改进！

