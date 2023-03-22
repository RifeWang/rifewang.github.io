+++
draft = false
date = 2023-03-20T11:09:26+08:00
title = "Kubernetes snapshots 快照是什么以及如何使用快照？"
description = "Kubernetes snapshots 快照是什么以及如何使用快照？"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

---

*文本翻译自: https://blog.palark.com/kubernetes-snaphots-usage/*

---

随着 Kubernetes 中快照控制器的引入，现在可以为支持此功能的 CSI 驱动程序和云提供商创建快照。

API 是通用的且独立于供应商，这对于 Kubernetes 来说是典型的，因此我们可以探索它而无需深入了解特定实现的细节。让我们仔细看看快照，看看它们如何使 Kubernetes 用户受益。

## 介绍

首先，让我们澄清什么是快照。快照是文件系统在特定时间点的状态。您可以保存它并在以后使用它来恢复该特定状态。创建快照的过程几乎是瞬时的。创建快照后，对原始文件系统的所有更改都将写入不同的块。

由于快照数据与原始数据存储在同一位置，因此快照不能替代备份。同时，基于快照而不是实时数据的备份更加一致。这是因为在创建快照时保证所有数据都是最新的。

必须安装 [snapshot-controller](https://github.com/kubernetes-csi/external-snapshotter) （所有 CSI driver 的通用组件），并且必须在 Kubernetes 集群中定义以下 CRD 才能使用快照功能：

- `VolumeSnapshotClass` – 相当于快照的 `StorageClass`；
- `VolumeSnapshotContent` – 相当于快照的 `PV`；
- `VolumeSnapshot` – 相当于快照的 `PVC`。

最重要的是，CSI 驱动程序必须支持快照创建并具有相关的 `csi-snapshotter` controller。

## 快照在 Kubernetes 中是如何工作的？

他们运作背后的逻辑很简单。有几个实体；`VolumeSnapshotClass` 描述快照创建的参数，例如 CSI driver。您还可以在那里指定其他设置，例如，快照是否应该是增量的以及它们应该存储在哪里。

创建 `VolumeSnapshot` 时，您必须指定将为其创建快照的 `PersistentVolumeClaim`。

拍摄快照时，CSI 驱动程序会在集群中创建一个 `VolumeSnapshotContent` 资源并设置其参数（通常是资源 ID）。

接下来，快照控制器绑定 `VolumeSnapshot` 到 `VolumeSnapshotContent`（就像 PV 和 PVC 一样）。

创建新的 `PersistentVolume` 时，您可以将先前创建的 `VolumeSnapshot` 设置为  `dataSource` 以使用其数据。

## 配置

`VolumeSnapshotClass` 允许您指定各种 `VolumeSnapshot` 属性，例如 CSI 驱动程序名称和其他云提供商/数据存储相关参数。下面提供了几个 `VolumeSnapshotClass` 资源定义示例的链接 ：

- [OpenStack](https://github.com/openshift/openstack-cinder-csi-driver-operator/blob/master/assets/volumesnapshotclass.yaml)
- [vSphere](https://docs.vmware.com/en/VMware-vSphere-Container-Storage-Plug-in/2.0/vmware-vsphere-csp-getting-started/GUID-E0B41C69-7EEB-450F-A73D-5FD2FF39E891.html)
- [AWS](https://aws.amazon.com/ru/blogs/containers/using-ebs-snapshots-for-persistent-storage-with-your-eks-cluster/)
- [Azure](https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/deploy/example/snapshot/storageclass-azuredisk-snapshot.yaml)
- [LINSTOR](https://github.com/piraeusdatastore/piraeus-operator/blob/master/doc/optional-components.md#using-snapshots)
- [GCP](https://cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/volume-snapshots#create-snapshotclass)
- [CephFS](https://github.com/ceph/ceph-csi/blob/devel/examples/cephfs/snapshotclass.yaml)
- [Ceph RBD](https://github.com/ceph/ceph-csi/blob/devel/examples/rbd/snapshotclass.yaml)

创建 `VolumeSnapshotClass` 后 ，您就可以开始拍摄快照了。让我们来看看一些典型的用例。

## 案例一：PVC templates

[![asciiccast](https://asciinema.org/a/yQG1fxy7KSSScoSAC2PunzxYU.svg)](https://asciinema.org/a/yQG1fxy7KSSScoSAC2PunzxYU)

假设我们想要一些包含数据的 PVC 模板，并在需要时克隆它。在以下情况下这可能会派上用场：

- 使用数据快速创建开发环境；
- 在不同节点上使用多个 Pod 同时处理数据。

这背后的魔力是创建一个标准 PVC，用你想要的数据填充它，然后创建另一个 PVC 以原始集作为其源：

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-worker1
spec:
  storageClassName: linstor-ssd-lvmthin-r2
  dataSource:
    name: pvc-template
    kind: PersistentVolumeClaim
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
    storage: 10Gi
```

您将获得包含所有数据的原始 PVC 的完整克隆，您可以立即使用。快照机制在这里是完全透明的，所以我们甚至不必使用上述任何资源。


## 案例二：用于测试的快照

[![asciiccast](https://asciinema.org/a/MY8gBYhdAO22HguUX7gMIzDBO.svg)](https://asciinema.org/a/MY8gBYhdAO22HguUX7gMIzDBO)

此案例展示了如何在不干扰生产的情况下安全地对实时数据进行数据库迁移建模。

我们必须克隆我们的应用程序使用的现有 PVC（就像在上面的示例中一样）以及具有克隆 PVC 的新应用程序版本来测试升级。如果遇到问题，您可以创建一个新的克隆并重试。

测试完成后，可以将新版本的应用程序部署到生产环境中。但首先，创建一个 `mypvc-before-upgrade` 快照，这样您就可以随时恢复到升级前的状态。快照是使用 `VolumeSnapshots` 资源创建的。在其中指定创建快照的目标 PVC：

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mypvc-before-upgrade
spec:
  volumeSnapshotClassName: linstor
  source:
    persistentVolumeClaimName: mypvc
```

`mypvc-before-upgrade` 切换到新版本后，您始终可以通过将快照指定为 PVC 源来恢复到升级前的状态：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
spec:
  storageClassName: linstor-ssd-lvmthin-r2
  dataSource:
    name: mypvc-before-upgrade
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
    storage: 10Gi
```


## 案例三：使用快照做一致性备份

[![asciiccast](https://asciinema.org/a/AOgMG6UdmIrHzj2aWhgXwxQJe.svg)](https://asciinema.org/a/AOgMG6UdmIrHzj2aWhgXwxQJe)

快照对于在运行环境中创建一致的备份是不可或缺的。没有它们，就没有办法在不先暂停应用程序的情况下进行 PVC 备份。

如果您尝试在应用程序运行时复制整个卷，则很可能会覆盖其中的某些部分。为避免这种情况，您可以拍摄快照并将其用于备份。

有多种工具可用于在 Kubernetes 中进行备份，这些工具尊重应用程序的逻辑且/或使用快照机制。其中一个工具 [Velero](https://velero.io/) 允许您自动使用快照，安排额外的挂钩将数据重置到磁盘，并暂停/恢复应用程序以获得更好的备份一致性。

同时，一些供应商提供了内置的备份功能。例如，[LINSTOR](https://linbit.com/linstor/) 允许您自动将快照上传到远程 S3 服务器，并支持完整和增量备份。

为了从此功能中受益，您需要创建一个专用的 `VolumeSnapshotClass` 包含访问远程 S3 服务器所需的所有参数：

```yaml
---
kind: VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
metadata:
  name: linstor-minio
driver: linstor.csi.linbit.com
deletionPolicy: Retain
parameters:
  snap.linstor.csi.linbit.com/type: S3
  snap.linstor.csi.linbit.com/remote-name: minio
  snap.linstor.csi.linbit.com/allow-incremental: "false"
  snap.linstor.csi.linbit.com/s3-bucket: foo
  snap.linstor.csi.linbit.com/s3-endpoint: XX.XXX.XX.XXX.nip.io
  snap.linstor.csi.linbit.com/s3-signing-region: minio
  snap.linstor.csi.linbit.com/s3-use-path-style: "true"
  csi.storage.k8s.io/snapshotter-secret-name: linstor-minio
  csi.storage.k8s.io/snapshotter-secret-namespace: minio
---
kind: Secret
apiVersion: v1
metadata:
  name: linstor-minio
  namespace: minio
immutable: true
type: linstor.csi.linbit.com/s3-credentials.v1
stringData:
  access-key: minio
  secret-key: minio123
```

新创建的快照现在将被推送到远程 S3 服务器：

```yaml
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mydb-backup1
spec:
  volumeSnapshotClassName: linstor-minio
  source:
    persistentVolumeClaimName: db-data
```

有趣的是，您可以在不同的 Kubernetes 集群中使用它们。为此，除了 `VolumeSnapshotClass` 之外，您还必须定义 `VolumeSnapshotContent` 和 `VolumeSnapshot`：

```yaml
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  name: example-backup-from-s3
spec:
  deletionPolicy: Delete
  driver: linstor.csi.linbit.com
  source:
    snapshotHandle: snapshot-0a829b3f-9e4a-4c4e-849b-2a22c4a3449a
  volumeSnapshotClassName: linstor-minio
  volumeSnapshotRef:
    apiVersion: snapshot.storage.k8s.io/v1
    kind: VolumeSnapshot
    name: example-backup-from-s3
    namespace: new-cluster
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: example-backup-from-s3
spec:
  source:
    volumeSnapshotContentName: example-backup-from-s3
  volumeSnapshotClassName: linstor-minio
```

请注意，您必须在 `VolumeSnapshotContent` 中通过 `snapshotHandle` 参数指定存储系统的快照 ID。

现在您可以使用备份快照作为数据源来创建新的 PVC：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-data
  namespace: new-cluster
spec:
  storageClassName: linstor-ssd-lvmthin-r2
  dataSource:
    name: example-backup-from-s3
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
    storage: 10Gi
```

## 结论

借助快照，您可以通过创建一致的备份和克隆卷来更有效地利用您的存储解决方案。它们还允许您避免在不必要时复制数据。这是快照，让您的生活更轻松、更美好！
