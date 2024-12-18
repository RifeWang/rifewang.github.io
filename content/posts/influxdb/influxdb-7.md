+++
draft = false
date = 2019-11-17T13:43:48+08:00
title = "时序数据库 InfluxDB（七）"
description = "时序数据库 InfluxDB（七）"
slug = ""
authors = []
tags = ["Database", "InfluxDB"]
categories = ["Database"]
externalLink = ""
series = []
disableComments = true
+++

---
# 单点故障和容灾备份
---

InfluxDB 开源的社区版本面临的最大的问题就是单点故障和容灾备份，有没有一个简单的方案去解决这个问题呢？

既然有单点故障的可能，那么索性写入多个节点，同时也解决了容灾备份的问题：


![double write](https://raw.githubusercontent.com/RifeWang/images/master/influxdb/double-write.png)

1、在不同的机器上配置多个 InfluxDB 实例，写入数据时，直接由客户端并发写入多个实例。（为什么不用代理，因为代理自身就是个单点）。

2、当某个 InfluxDB 实例故障而导致写入失败时，记录失败的数据和节点，这些失败的数据可以临时存储在数据库、消息中间件、日志文件等等里面。

3、通过自定义的 worker 拉取上一步记录的失败的数据然后重写这些数据。

4、多个 InfluxDB 中的数据最终一致。


当然你需要注意的是：

1、由于是并发写入多个节点，且不同机器的状况不一，所以写入数据应该设置一个超时时间。

2、写入失败的数据必须要与节点相对应，同时你应该考虑如何去定义失败的数据：由于格式不正确或者权限问题导致的 4xx 或者 InfluxDB 本身异常导致的 5xx ，这些与 InfluxDB 宕机等故障导致的失败显然是不同的。

3、由于失败的数据需要临时存储在一个数据容器中，你应该考虑所使用的数据容器能否承载故障期间写入的数据压力，以及如果数据要求不可丢失，那么数据容器也需要有对应的支持。

4、失败数据的重写是一个异步的过程，所以写入的数据应该由客户端指定明确的时间戳，而不是使用 InfluxDB 写入时默认生成的时间戳。

5、故障期间多个 InfluxDB 可能存在数据不一致的情况。

---
相关文章：
- [时序数据库 InfluxDB（一）](/influxdb-1/)
- [时序数据库 InfluxDB（二）](/influxdb-2/)
- [时序数据库 InfluxDB（三）](/influxdb-3/)
- [时序数据库 InfluxDB（四）](/influxdb-4/)
- [时序数据库 InfluxDB（五）](/influxdb-5/)
- [时序数据库 InfluxDB（六）](/influxdb-6/)
- [时序数据库 InfluxDB（七）](/influxdb-7/)
