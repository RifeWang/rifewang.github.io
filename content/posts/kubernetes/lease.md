+++
draft = false
date = 2023-12-26T11:42:30+08:00
title = "Kubernetes Lease 及分布式选主"
description = "Kubernetes Lease 及分布式选主"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

## 分布式选主

在分布式系统中，应用服务常常会通过多个节点（或实例）的方式来保证高可用。然而在某些场景下，有些数据或者任务无法被并行操作，此时就需要由一个特定的节点来执行这些特殊的任务（或者进行协调及决策），这个特定的节点也就是领导者（Leader），而在多个节点中选择领导者的机制也就是分布式选主（Leader Election）。

如今诸多知名项目也都使用了分布式选主，例如：
- `Etcd`
- `Kafka`
- `Elasticsearch`
- `Zookeeper`

常用算法包括：
- `Paxos`：一种著名的分布式共识算法，原理和实现较为复杂（此算法基本就是共识理论的奠基之作，曾有人说："世界上只有一种共识协议，就是 Paxos，其他所有共识算法都是 Paxos 的退化版本"）。
- `Raft`：目前最广泛使用的分布式共识算法之一，Etcd 使用的就是 `Raft`，Elasticsearch 和 Kafka 在后来的版本中也都抛弃了早期的算法并转向了 `Raft`。
- `ZAB（Zookeeper Atomic Broadcast`）：Zookeeper 使用的一致性协议，也包括选主机制。


## Kubernetes Lease

在 Kubernetes 中，诸如 `kube-scheduler` 和 `kube-controller-manager` 等核心组件也需要使用分布式选主，因为其需要确保任一时刻只有一个调度器在做出调度决策，同一时间只有一个控制管理器在处理资源对象。

然而，除了核心组件，用户的应用服务很可能也有类似分布式选主的需求，为了满足这种通用需求，kubernetes 提供了 `Lease`（翻译为“租约”）这样一个特殊的资源对象。

![](https://raw.githubusercontent.com/RifeWang/images/master/k8s/k8s-lease.png)

如上图所示，在 k8s 中选主是通过争抢一个分布式锁（`Lease`）来实现的，抢到锁的实例成为 leader，为了确认自己持续存活，leader 需要不断的续签这个锁（`Lease`），一旦 leader 挂掉，则锁被释放，其他候选人便可以竞争成为新的 leader。

`Lease` 的结构也很简单：
```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  # object
spec:
  acquireTime: # 当前租约被获取的时间
  holderIdentity: # 当前租约持有者的身份信息
  leaseDurationSeconds: # 租约候选者需要等待才能强制获取它的持续时间
  leaseTransitions: # 租约换了多少次持有者
  renewTime: # 当前租约持有者最后一次更新租约的时间
```

`Lease` 本质上与其它资源并无区别，除了 `Lease`，其实也可以用 configmap 或者 endpoint 作为分布式锁，因为在底层都是 k8s 通过资源对象的 `resourceVersion` 字段进行 compare-and-swap，也就是通过这个字段实现的乐观锁。当然在实际使用中，建议还是用 `Lease`。

### 使用示例

使用 `Lease` 进行分布式选主的示例如下：
```golang
import (
    "context"
    "time"

    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/rest"
    "k8s.io/client-go/tools/leaderelection"
    "k8s.io/client-go/tools/leaderelection/resourcelock"
)

func main() {
    config, err := rest.InClusterConfig()
    if err != nil {
        panic(err.Error())
    }

    clientset, err := kubernetes.NewForConfig(config)
    if err != nil {
        panic(err.Error())
    }

    // 配置 Lease 参数
    leaseLock := &resourcelock.LeaseLock{
        LeaseMeta: metav1.ObjectMeta{
            Name:      "my-lease",
            Namespace: "default",
        },
        Client: clientset.CoordinationV1(),
        LockConfig: resourcelock.ResourceLockConfig{
            Identity: "my-identity",
        },
    }

    // 配置 Leader Election
    leaderElectionConfig := leaderelection.LeaderElectionConfig{
        Lock:          leaseLock,
        LeaseDuration: 15 * time.Second,
        RenewDeadline: 10 * time.Second,
        RetryPeriod:   2 * time.Second,
        Callbacks: leaderelection.LeaderCallbacks{
            OnStartedLeading: func(ctx context.Context) {
                // 当前实例成为 Leader
                // 在这里执行 Leader 专属的逻辑
            },
            OnStoppedLeading: func() {
                // 当前实例失去 Leader 地位
                // 可以在这里执行清理工作
            },
            OnNewLeader: func(identity string) {
                // 有新的 Leader 产生
            }
        },
    }

    leaderElector, err := leaderelection.NewLeaderElector(leaderElectionConfig)
    if err != nil {
        panic(err.Error())
    }

    // 开始 Leader Election
    ctx := context.Background()
    leaderElector.Run(ctx)
}
```

---

参考资料：

- *https://kubernetes.io/docs/concepts/architecture/leases/*
- *https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/lease-v1/*
- *https://pkg.go.dev/k8s.io/client-go@v0.29.0/tools/leaderelection*