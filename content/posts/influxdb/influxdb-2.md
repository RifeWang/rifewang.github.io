+++
draft = false
date = 2019-10-26T13:14:35+08:00
title = "时序数据库 InfluxDB（二）"
description = "时序数据库 InfluxDB（二）"
slug = ""
authors = []
tags = ["Database", "InfluxDB"]
categories = ["Database"]
externalLink = ""
series = []
disableComments = true
+++

---
#### RP
---


先回顾一下 RP 策略（ retention policy ），它由三个部分构成：

* DURATION：数据的保留时长。
* REPLICATION：集群模式下数据的副本数，单节点无效。
* SHARD DURATION：可选项，shard group 划分的时间范围。


前两个部分没啥好说的，而 shard duration 和 shard group 的概念你可能会感到比较陌生。


shard 是什么？


先来看数据的层次结构：

![image](https://raw.githubusercontent.com/RifeWang/images/master/influxdb/shard.webp)

如果所示，一个 database 对应一个实际的磁盘上的文件夹，该数据库下不同的 RP 策略对应不同的文件夹。



shard group 只是一个逻辑概念，并没有实际的磁盘文件夹，shard group 包含有一个或多个 shard 。



最终的数据是存储在 shard 中的，每个 shard 也对应一个具体的磁盘文件目录，数据是按照时间范围分割存储的，shard duration 也就是划分 shard group 的时间范围（例如 shard duration 如果是一周，那么第一周的数据就会存储到一个 shard group 中，第二周的数据会存储到另外一个 shard group 中，以此类推）。



另外，每个 shard 目录下都有一个 TSM 文件（后缀名为 .tsm ），正是这个文件存储了最后编码和压缩后的数据。shard group 下的 shard 是按照 series 来划分的，每个 shard 包含一组特定的 series ，换句话说特定 shard group 中的特定 series 上的所有 points 点都存储在同一个 TSM 文件中。



---
#### shard duration
---


shard 从属于唯一一个 shard group ，shard duration 和 shard group duration 是同一个概念。



如前文所述，数据按照时间范围分割存储，分割的时间范围由 RP 策略中的 shard group duration 指定。



默认情况下，shard group duration 根据 RP duration 的值来确定，对应关系如下图：

![image](https://raw.githubusercontent.com/RifeWang/images/master/influxdb/rp-duration.png)



RP 策略是不可或缺的，如果未设置则会使用默认的名称为 autogen 的 RP ，它的 duration 是 infinite 也就是数据不会过期，shard group duration 是 7 天（ duration 是 infinite 对应的就是 > 6 months 这一栏）。


shard group duration 设置为多久才最好？

* 长时间范围：有利于存储更多数据，整体性能更好。
* 短时间范围：灵活性更高，有利于删除过期数据和记录增量备份。删除过期数据是删除整个 shard group 而不是单个的 shard 。


默认配置对于大多数场景都运行的很好，然而，高吞吐量或长时间运行的实例将受益于更长的 shard group duration ，官方建议的配置如下：

![image](https://raw.githubusercontent.com/RifeWang/images/master/influxdb/rp-duration-config.webp)



其它一些需要考虑的因素：
* shard group 应该包含最频繁查询的最长时间范围的两倍。
* 每个 shard group 应该包含超过十万个 point 。
* shard group 中的每个 series 应该包含超过一千个 point 。


另外，批量插入长时间范围内的大量历史数据将会一次触发大量 shard 的创建，并发访问和写入成百上千的 shard 会导致性能降低和内存耗尽，对于这种情况建议临时设置较长的 shard group duration 比如 52 周。

RP 策略可以动态调整，删除一个 RP 将会删除其下的所有数据。

---
相关文章：
- [时序数据库 InfluxDB（一）](/influxdb-1/)
- [时序数据库 InfluxDB（二）](/influxdb-2/)
- [时序数据库 InfluxDB（三）](/influxdb-3/)
- [时序数据库 InfluxDB（四）](/influxdb-4/)
- [时序数据库 InfluxDB（五）](/influxdb-5/)
- [时序数据库 InfluxDB（六）](/influxdb-6/)
- [时序数据库 InfluxDB（七）](/influxdb-7/)
