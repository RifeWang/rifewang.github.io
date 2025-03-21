+++
draft = false
date = 2019-10-25T13:02:58+08:00
title = "时序数据库 InfluxDB（一）"
description = "时序数据库 InfluxDB（一）"
slug = ""
authors = []
tags = ["Database", "InfluxDB"]
categories = ["Database"]
externalLink = ""
series = []
disableComments = true
+++

---
数据库种类有很多，比如传统的关系型数据库 RDBMS（ 如 MySQL ），NoSQL  数据库（ 如 MongoDB ），Key-Value 类型（ 如 redis ），Wide column 类型（ 如 HBase ）等等等等，当然还有本系列文章将会介绍的时序数据库 TSDB（ 如 InfluxDB ）。


---
#### 时序数据库 TSDB
---

不同的数据库针对的应用场景有不同的偏重。TSDB（ time series database ）时序数据库是专门以时间维度进行设计和优化的。
TSDB 通常具有以下的特点：
* 时间是不可或缺的绝对主角（就像 MySQL 中的主键一样），数据按照时间顺序组织管理
* 高并发高吞吐量的数据写入
* 数据的更新很少发生
* 过期的数据可以批量删除


InfluxDB 就是一款非常优秀的时序数据库，高居 DB-Engines TSDB rank 榜首。

InfluxDB 分为免费的社区开源版本，以及需要收费的闭源商业版本，目前只有商业版本支持集群。

InfluxDB 的底层数据结构从 LSM 树到 B+ 树折腾了一通，最后自创了一个 TSM 树（ Time-Structured Merge Tree ），这也是它性能高且资源占用少的重要原因。

InfluxDB 由 go 语言编写而成，没有额外的依赖，它的查询语言 InfluxQL 与 SQL 极其相似，使用特别简单。


---
#### InfluxDB 基本概念
---

InfluxDB 有以下几个核心概念：
1、database ：
数据库。


2、measurement
类似于表。


3、retention policy（ 简称 RP ）
保留策略，由以下三个部分构成：
* DURATION：数据的保留时长。
* REPLICATION：集群模式下数据的副本数，单节点无效。
* SHARD DURATION：可选项，shard group 划分的时间范围。



4、timestamp
时间戳，就像是所有数据的主键一样。


5、tag
tag key = tag value 键值对存储具体的数据，会构建索引有利于查询。tag set 就是 tag key-value 键值对的不同组合。


6、field
field key = field value  键值对也是存储具体的数据，但不会被索引。类似的 field set 就是 field key-value 的组合。


7、series
一个 series 序列是由同一个 RP 策略下的同一个 measurement 里的同一个 tag set 构成的数据集合。


8、point
一个 point 点代表了一条数据，由 measurement、tag set、field set、timestamp 组成。一个 series 上的某个 timestamp 时间对应唯一一个 point 。




##### Line protocol 行协议
行协议指定了写入数据的格式：

```
<measurement>[,<tag-key>=<tag-value>...] <field-key>=<field-value>[,<field2-key>=<field2-value>...] [unix-nano-timestamp]
```

符号 [] 代表可选项，符号 ... 代表可以有多个，符号 ，用来分隔相同 tag 或者 field 下的多个数据，符号空格分隔 tag、field、timestamp 。


示例：

![image](https://raw.githubusercontent.com/RifeWang/images/master/influxdb/line-protocol.png)


怎么去理解 series 和 point ？先看下图：

![image](https://raw.githubusercontent.com/RifeWang/images/master/influxdb/series-point.webp)


这张图选取了三种时序数据库的历年排名得分情况。首先，整个图表可以看成是一个 measurement ，它包含了许多数据；然后我们根据 db 名称构建 tag ，把 score 排名得分作为 field ，那么所有数据行就类似于：
```
measurement,db=InfluxDB score=5 timestamp
measurement,db=Kdb+ score=1 timestamp
measurement,db=Prometheus score=0.2 timestamp
...
```
上文说过 tag set 就是 tag key = tag value 的不同组合，因此这里的 tag set 有以下三种：
```
db=InfluxDB
db=Kdb+
db=Prometheus
```
三个 tag set 构成了三个 series ，每个 series 就可以看成是图中的一条线（一个维度），而每个 point 点就是 series 上具体某个 timestamp 对应的点。




---
#### 与传统数据库的不同
---

InfluxDB 就是被设计用于处理时间序列的数据。传统SQL数据库虽然也可以处理时间序列数据，但并不是专门以此为目标的。InfluxDB  可以更加高效快速的存储大量时间序列数据并对这些数据进行实时分析。


在 InfluxDB 中，时间是绝对的主角，就像是SQL数据库中的主键一样，如果你不指定则会默认为系统当前时间，时间必须是 UNIX epoch ( GMT ) 或者 RFC3339 格式。

InfluxDB 不需要预先定义好数据的结构，你可以随时改变你的数据结构。InfluxDB 支持 continuous queries（连续查询，就是以时间划分范围自动定期执行某个查询）和 retention policies（保留策略）。InfluxDB 不支持跨 measurement 的 JOIN 查询。

InfluxDB 中的查询语言叫 InfluxQL ，语法与 SQL 极其相似，就是 select from where 那一套。

InfluxDB 并不是 CRUD，更像是 CR-ud ，意思就是更新和删除数据跟传统SQL数据库明显不一样：
* 更新某个 point 数据，只需向原来的 measurement，tag set，timestamp 重写数据即可。
* 你可以删除 series ，但是不能基于 field 值去删除独立的 points ，解决方法是，你需要先查询 field 值的时间戳，然后根据时间戳去删除。
* 无法更新或重命名 tags ，因为 tags 会构建索引，你只能创建新的 tags 并导入数据然后删除老的。
* 无法通过 tag key 或者 tag value 去删除 tags 。


---
#### 设计与权衡之道
---

InfluxDB 为了更高的性能做了一些设计与权衡之道：
1、对于时间序列用例，即使相同的数据被发送多次也会被认为是同一笔数据。

* 优点：简化了冲突，提高了写入性能。
* 缺点：不能存储重复数据，可能会在极少数情况下覆盖数据。


2、删除是罕见的，当它们发生时肯定是针对大量的旧数据。

* 优点：提高了读写性能。
* 缺点：删除功能受到了很大限制。


3、更新是罕见的，持续或者大批量的更新不会发生。时间序列的数据主要是永远也不会更新的新数据。

* 优点：提高了读写性能。
* 缺点：更新功能受到了很大限制。


4、绝大多数写入都是接近当前时间戳的数据，并且是按时间递增顺序添加。
* 优点：按时间递增的顺序写入数据更高效。
* 缺点：随机时间写入的性能要低很多。

5、数据规模至关重要，数据库必须能够处理大量的读写。

* 优点：数据库可以处理大批量数据的读写。
* 缺点：被迫做出的一些权衡去提高性能。

6、能够写入和查询数据比具有强一致性更重要。

* 优点：多个客户端可以在高负载的情况下完成查询和写入操作。
* 缺点：如果负载过高，查询结果可能不包含最近的点。

7、许多时间序列都是短暂的。时间序列可能只有几个小时然后就没了，比如一台新的主机开机，监控数据写入一段时间，然后关机了。

* 优点：InfluxDB 善于管理不连续的数据。
* 缺点：无模式设计意味着不支持某些数据库功能，例如没有 join 交叉表连接。

8、No one point is too important 。

* 优点：InfluxDB 具有非常强大的工具去处理聚合数据和大数据集。
* 缺点：Points 数据点没有传统意义上的 ID ，它们被时间戳和 series 区分。

---
相关文章：
- [时序数据库 InfluxDB（一）](/influxdb-1/)
- [时序数据库 InfluxDB（二）](/influxdb-2/)
- [时序数据库 InfluxDB（三）](/influxdb-3/)
- [时序数据库 InfluxDB（四）](/influxdb-4/)
- [时序数据库 InfluxDB（五）](/influxdb-5/)
- [时序数据库 InfluxDB（六）](/influxdb-6/)
- [时序数据库 InfluxDB（七）](/influxdb-7/)
