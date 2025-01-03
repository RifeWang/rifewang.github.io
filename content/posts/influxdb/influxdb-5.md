+++
draft = false
date = 2019-10-30T13:33:30+08:00
title = "时序数据库 InfluxDB（五）"
description = "时序数据库 InfluxDB（五）"
slug = ""
authors = []
tags = ["Database", "InfluxDB"]
categories = ["Database"]
externalLink = ""
series = []
disableComments = true
+++

---
# 系统监控
---


InfluxDB 自带有一个监控系统，默认情况下此功能是开启的，每隔 10 秒中采集一次系统数据并把数据写入到 _internal 数据库中，其默认使用名称为 monitor 的 RP（数据保留 7 天），相关配置见配置文件中的：
```
[monitor]
  store-enabled = true
  store-database = "_internal"
  store-interval = "10s"
```


_internal 数据库与其它数据库的使用方式完全一致，其记录的统计数据分为多个 measurements ：

* cq ：连续查询
* database ：数据库
* httpd ：HTTP 相关
* queryExecutor ：查询执行器
* runtime ：运行时
* shard ：分片
* subscriber ：订阅者
* tsm1_cache ：TSM cache 缓存
* tsm1_engine ：TSM 引擎
* tsm1_filestore ：TSM filestore
* tsm1_wal ：TSM 预写日志
* write ：数据写入



比如查询最近一次统计的数据写入情况：
```
select * from "write" order by time desc limit 1
```


_internal 数据库里的这些 measurements 中具体有哪些 field ，每个 field 数据又代表了什么含义，请参考官方文档：

[https://docs.influxdata.com/platform/monitoring/influxdata-platform/tools/measurements-internal/#influxdb-internal-measurements-and-fields](https://docs.influxdata.com/platform/monitoring/influxdata-platform/tools/measurements-internal/#influxdb-internal-measurements-and-fields)




**InfluxDB 相关命令**：

* show stats
* show diagnostics



1、
```
SHOW STATS [ FOR '<component>' | 'indexes' ]
```
show stats 命令返回的系统数据与 _internal 数据库中的数据结构是一致的，这里的 component 其实就是对应 _internal 中的 measurement ，比如：
```
show stats for 'queryExecutor'
```


唯一例外的是：
```
show stats for 'indexes'
```
其会返回所有索引使用的内存大小预估值，且没有 _internal 中的 measurement 与之对应。



2、
```
SHOW DIAGNOSTIC
```
返回系统的诊断信息，包括：版本信息、正常运行时间、主机名、服务器配置、内存使用情况、Go 运行时等，这些数据不会存储到 _internal 数据库中。





**InfluxDB 也支持通过 HTTP 接口获取系统信息**：

* /metrics ：这个接口返回的数据是诸如垃圾回收、内存分配等的 Go 相关指标。
* /debug/vars ：这个接口返回的数据与 _internal 数据类似。






---
# 备份和恢复
---


InfluxDB 支持本地或远程的数据备份和恢复，其是通过 TCP 连接进行的，对于远程方式，你必须修改配置文件中的：
```
bind-address = "127.0.0.1:8088"
```
将其设置为本机在网络上可通信的对外地址，然后重启服务，执行命令时需要通过 -host 参数对应这个地址。



备份命令：

![backup](https://raw.githubusercontent.com/RifeWang/images/master/influxdb/backup.png)



恢复命令：

![restore](https://raw.githubusercontent.com/RifeWang/images/master/influxdb/restore.png)



备份和恢复的命令参数非常相似，参数的含义也是一目了然的，比如你可以备份指定的数据库、RP、shard，恢复到新的数据库、RP 。



由于备份的格式进行过不兼容的更新，-portable 就是指定使用新的备份格式（强烈建议使用），-online 就是老的备份格式。



所有备份都是全量备份，不支持增量备份。你可能会问，不是有 -start 和 -end 可以指定备份数据的时间范围吗？没错，是可以的，但是备份是在数据块上执行，并不是逐点执行，而数据块又是高度压缩的，你使用 -start 和 -end 时，其还会备份到同一个数据块中的其它数据点，也就是说：
* 备份和还原可能会包含指定时间范围之外的数据。
* 如果包含重复的数据点，再次写入则会覆盖现有数据点。


另外，恢复数据时，无法直接恢复到一个已经存在的数据库或者 RP 中，为此你只能先使用一个临时的数据库和 RP ，然后再重新将数据插入到已有的数据库中（比如使用 select ... into 语句）。

---
相关文章：
- [时序数据库 InfluxDB（一）](/influxdb-1/)
- [时序数据库 InfluxDB（二）](/influxdb-2/)
- [时序数据库 InfluxDB（三）](/influxdb-3/)
- [时序数据库 InfluxDB（四）](/influxdb-4/)
- [时序数据库 InfluxDB（五）](/influxdb-5/)
- [时序数据库 InfluxDB（六）](/influxdb-6/)
- [时序数据库 InfluxDB（七）](/influxdb-7/)
