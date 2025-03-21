+++
draft = false
date = 2019-11-06T13:36:39+08:00
title = "时序数据库 InfluxDB（六）"
description = "时序数据库 InfluxDB（六）"
slug = ""
authors = []
tags = ["Database", "InfluxDB"]
categories = ["Database"]
externalLink = ""
series = []
disableComments = true
+++

---
# CQ 连续查询
---


连续查询 Continuous Queries（ CQ ）是 InfluxDB 很重要的一项功能，它的作用是在 InfluxDB 数据库内部自动定期的执行查询，然后将查询结果存储到指定的 measurement 里。

配置文件中的相关配置：
```
[continuous_queries]
  enabled = true
  log-enabled = true
  query-stats-enabled = false
  run-interval = "1s"
```

* enabled = true ：开启CQ
* log-enabled = true ：输出 CQ 日志
* query-stats-enabled = false ：关闭 CQ 执行相关的监控，不会将统计数据写入默认的监控数据库 _internal
* run-interval = "1s" ：InfluxDB 每隔 1s 检查是否有 CQ 需要执行



---
# 基本语法
---


#### 一 、


基本语法：
```
CREATE CONTINUOUS QUERY <cq_name> ON <database_name>
BEGIN
  <cq_query>
END
```
在某个数据库上创建一个 CQ ，而查询的具体内容 cq_query 的语法为：
```
SELECT <function[s]>
  INTO <destination_measurement>
  FROM <measurement>
  [WHERE <stuff>]
  GROUP BY time(<interval>)[,<tag_key[s]>]
```

* SELECT function[s] : 连续查询并不只是简单的查询原始数据，而是基于原始数据进行聚合、特选、转换、预测等处理，所以 CQ 必须要有一个或多个数据处理函数。
* INTO <destination_measurement> : 将 CQ 的结果存储到指定的 measurement 中。
* FROM <measurement> : 原始数据的来源 measurement 。
* [WHERE <stuff>] : 可选项，原始数据的筛选条件。
* GROUP BY time(<interval>)[,<tag_key[s]>] : 连续查询不是查一次就完了，而是每次查询指定时间范围内的数据，不断周期性的执行下去。


定位一个 measurement 的完整格式是：
```
<database>.<RP>.<measurement>
```
使用当前数据库和默认 RP 的情况就只需要 measurement 。



InfluxDB 支持的时长单位：
* ns      :   纳秒
* u / µ  :   微秒
* ms     :   毫秒
* s        :   秒
* m       :   分钟
* h        :   小时
* d        :   天
* w       :   周




#### 二、


##### 1、CQ 在何时执行？


CQ 在何时执行取决于 CQ 创建完成的时间点、GROUP BY time() 设置的时间间隔、以及 InfluxDB 数据库预设的时间边界（这个预设的时间边界其实就是 1970.01.01 00:00:00 UTC 时间，对应 Unix timestamp 的 0 值）。




假设我在 2019.11.05（北京时间）创建好了一个 GROUP BY time(30d) 的 CQ（也就是时间间隔为 30 天），那么这个 CQ 会在什么时间点执行？


首先，2019.11.05 号转换为 timestamp 是 1572883200 秒；
再算1572883200 距离 0 值隔了多少个 30 天（一天是 86400 秒），1572883200/86400/30 = 606.8 ；
那么下一个 30 天就是 606.8 向上取整 607 ，607*86400*30 = 1573344000 ，转换为对应的日期就是 2019.11.10 号，这也就是第一次执行 CQ 的时间，之后每次执行就是往后推 30 天。



如果每次都这样算就很麻烦，但其实我们更常使用的时间间隔没有那么长，通常都是秒、分钟、小时单位，这种情况下直接从 0 速算就可以了，比如：

* 在时间点 16:09:35 创建了 CQ ，GROUP BY time(30s) ，那么 CQ 的执行时间就是 16:10:00、16:10:30、16:11:00 以此类推（从 0s 开始速算）。
* 在时间点 16:16:08 创建了 CQ ，GROUP BY time(5m) ，那么 CQ 的执行时间就是 16:20:00、16:25:00、16:30:00 以此类推（从 0m 开始速算）。
* 在时间点 16:38:27 创建了 CQ ，GROUP BY time(2h) ，那么 CQ 的执行时间就是 18:00:00 、20:00:00 、22:00:00 以此类推（从 0h 开始速算）。




##### 2、CQ 执行的数据范围？


连续查询会根据 GROUP BY time() 的时间间隔确定作用的数据，每次执行所针对的数据的时间范围是 [ now() - GROUP BY time() ，now() ) 。


例如，GROUP BY time(1h) ：

* 在 8:00 执行时，数据是时间大于等于 7:00，小于 8:00，即 [ 7:00 , 8:00 )  范围内的数据。
* 在 9:00 执行时，数据是时间大于等于 8:00，小于 9:00，即 [ 8:00 , 9:00 ) 范围内的数据。


你可以使用 WHERE 去过滤数据，但是 WHERE 里指定的时间范围会被忽略掉。




##### 3、CQ 的执行结果？


CQ 会将执行结果存储到指定的 measurement ，但是存储的具体字段有哪些呢？首先 time 是必不可少的，time 写入的是 CQ 执行时数据范围的开始时间点；其次就是 function 的处理结果，如果只有单一字段，那么 field key 就是 function 的名称，如果有多个字段，那么 field key 就是 function 名称_作用字段。


例如，GROUP BY time(30m) ，UTC 7:30 执行：
单一字段：
```
SELECT mean("field")
  INTO "result_measurement"
  FROM "source_measurement"
  GROUP BY time(30m)
```
CQ 结果：
```
time                      mean
2019-11-05T07:00:00Z      7
```
多字段：
```
SELECT mean("*")
  INTO "result_measurement"
  FROM "source_measurement"
  GROUP BY time(30m)
```
CQ 结果：
```
time                      mean_field1    mean_field2
2019-11-05T07:00:00Z      7              6.5
```
这里的 mean 对应的是 function 里的平均值函数。




#### 三、


GROUP BY time() 的完整格式是：
```
GROUP BY time(<interval>[,<offset_interval>])
```
第二个参数 offset_interval 偏移量是可选的，这个偏移量会对 CQ 的执行时间和数据范围产生影响。


如果 GROUP BY time(1h) ，在 8:00 执行，数据范围是 [ 7:00 , 8:00 ) 。
那么 GROUP BY time(1h, 15m) 会使 CQ 的执行时间向后推迟 15m ，即在 8:15 执行，数据范围也就变成了 [ 7:15 , 8:15 ) 。



---
# 高级语法
---


高级语法：
```
CREATE CONTINUOUS QUERY <cq_name> ON <database_name>
RESAMPLE EVERY <interval> FOR <interval>
BEGIN
  <cq_query>
END
```
与基本语法不同的是，高级语法多了
```
RESAMPLE EVERY <interval> FOR <interval>
```

#### 1、RESAMPLE  EVERY


EVERY 定义了 CQ 执行的间隔：
```
RESAMPLE EVERY 30m
```
意思就是每隔 30m 执行一次 CQ 。


示例：
```
CREATE CONTINUOUS QUERY "cq_every" ON "db"
RESAMPLE EVERY 30m
BEGIN
  SELECT mean("field")
    INTO "result_measurement"
    FROM "source_measurement"
    GROUP BY time(1h)
END
```
如果没有 RESAMPLE EVERY 30m ，只有 GROUP BY time(1h) 将会：

* 在 8:00 执行 CQ ，数据范围是 [ 7:00 , 8:00 )
* 在 9:00 执行 CQ ，数据范围是 [ 8:00 , 9:00 )


增加了 RESAMPLE EVERY 30m 之后，每 30m 执行一次 CQ ：

* 在 8:00 执行 CQ ，数据范围是 [ 7:00 , 8:00 )
* 在 8:30 执行 CQ ，数据范围是 [ 8:00 , 9:00 )
* 在 9:00 执行 CQ ，数据范围是 [ 8:00 , 9:00 ) ，由于执行结果的 time 字段是 8:00 与上一次 CQ 一致，因此会覆盖上一次 CQ 的结果。


当 EVERY 的时间间隔小于 GROUP BY time() 时，会增加 CQ 的执行频率（如上述示例）。

当 EVERY 与 GROUP BY time() 的时间间隔一致时，无影响。


当 EVERY 的时间间隔大于 GROUP BY time() 时，CQ 执行时间和数据范围完全由 EVERY 控制，例如 EVERY 30m ，GROUP BY time(10m) ：

* 在 8:00 执行 CQ ，数据范围是 [ 7:30 , 8:00 )
* 在 8:30 执行 CQ ，数据范围是 [ 8:00 , 8:30 )




#### 2、RESAMPLE  FOR


FOR 定义了数据的时间范围：
```
RESAMPLE FOR 1h
```
意思就是每次 CQ 的数据的时间范围是 1h 。


示例：
```
CREATE CONTINUOUS QUERY "cq_for" ON "db"
RESAMPLE FOR 1h
BEGIN
  SELECT mean("field")
    INTO "result_measurement"
    FROM "source_measurement"
    GROUP BY time(30m)
END
```

如果没有 RESAMPLE FOR 1h ，只有 GROUP BY time(30m) 将会：

* 在 8:00 执行 CQ ，数据范围是 [ 7:30 , 8:00 )
* 在 8:30 执行 CQ ，数据范围是 [ 8:00 , 8:30 )


增加了 RESAMPLE FOR 1h 之后，每次 CQ 的时间范围是 1h ，但是因为  GROUP BY time(30m) ，每次 CQ 将会按照 30m 写入两点数据：

* 在 8:00 执行 CQ ，数据总范围是 [ 7:00 , 8:00 ) ，实际会拆分成两点 [ 7:00 , 7:30 ) 和 [ 7:30 , 8:00 )
* 在 8:30 执行 CQ ，数据总范围是 [ 7:30 , 8:30 ) ，实际会拆分成两点 [ 7:30 , 8:00 ) 和 [ 8:00 , 8:30 )




当 FOR 的时间间隔大于 GROUP BY time() 时，每次 CQ 的时间范围被扩大，但是每一个点仍然按照 GROUP BY time() 的时间间隔，因此每次 CQ 会写入多个点（如上述示例）。


当 FOR 与 GROUP BY time() 的时间间隔一致时，无影响。


当 FOR 的时间间隔小于 GROUP BY time() 时，创建 CQ 时报错，不允许这种情况。




#### 3、EVERY ... FOR


EVERY 和 FOR 可以一起使用。


示例：
```
CREATE CONTINUOUS QUERY "cq_every_for" ON "db"
RESAMPLE EVERY 1h FOR 90m
BEGIN
  SELECT mean("field")
    INTO "result_measurement"
    FROM "source_measurement"
    GROUP BY time(30m)
END
```

EVERY 1h 大于 GROUP BY time(30m)，因此 CQ 每隔 1h 执行一次；FOR 90m ，每次 CQ 执行的时间范围是 90m，按照 30m 拆分成三个点：

* 在 8:00 执行 CQ ，数据总范围 [ 6:30 , 8:00 ) ，实际会拆分为三个点  [ 6:30 , 7:00 )、 [ 7:00 , 7:30 )、 [ 7:30 , 8:00 )
* 在 9:00 执行 CQ ，数据总范围 [ 7:30 , 9:00 ) ，实际会拆分为三个点  [ 7:30 , 8:00 )、 [ 8:00 , 8:30 )、 [ 8:30 , 9:00 )



最后，CQ 只能创建和删除，无法修改。

---
相关文章：
- [时序数据库 InfluxDB（一）](/influxdb-1/)
- [时序数据库 InfluxDB（二）](/influxdb-2/)
- [时序数据库 InfluxDB（三）](/influxdb-3/)
- [时序数据库 InfluxDB（四）](/influxdb-4/)
- [时序数据库 InfluxDB（五）](/influxdb-5/)
- [时序数据库 InfluxDB（六）](/influxdb-6/)
- [时序数据库 InfluxDB（七）](/influxdb-7/)
