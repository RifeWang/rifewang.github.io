# 时序数据库 InfluxDB（三）


---
### 数据类型
---

InfluxDB 是一个无结构模式，这也就是说你无需事先定义好表以及表的数据结构。


InfluxDB 支持的数据类型非常简单：

* measurement : string
* tag key : string
* tag value : string
* field key : string
* field value : string , float , interger , boolean


你可以看到除了 field value 支持的数据类型多一点之外，其余全是字符串类型。


当然还有最重要的 timestamp ，InfluxDB 中的时间都是 UTC 时间，而且时间精度非常高，默认为纳秒。



---
### 数据结构设计
---

在实际使用中，数据都是存储在 tag 或者 field 中，这两者最重要的区别就是，tag 会构建索引（也就是说查询时，where 条件里的是 tag ，则查询性能更高），field 则不会被索引。


存储数据到底是使用 tag 还是 field ，参考以下原则：

* 常用于查询条件的数据存储为 tag 。
* 计划使用 GROUP BY() 的数据存储为 tag 。
* 计划使用 InfluxQL function 的数据存储为 field 。
* 数据不只是 string 类型的存储为 field 。


对于标识性的名称，如 database、RP、user、measurement、tag key、field key 这些应该避免使用 InfluxQL 中的关键字。


其它需要注意的原则：

* 不要有过于庞大的 series 。若在 tag 中使用 UUID、hash、随机字符串等将会导致数量庞大的 series ，这将会导致更高的内存使用率，尤其是系统内存有限的情况下需要额外注意。


* measurement 名称不应该包含具体的数据（表名就是一个单纯的表名），你应该使用不同的 tag 去区分数据，而不是 measurement 名称。


* 一个 tag 中不要放置多条信息，复杂的信息合理拆分为多个 tag 有助于简化查询并减少使用正则。



---
### 索引
---

InfluxDB 通过构建索引可以提高查询性能。InfluxDB 中的索引有两种：In-memory 和 TSI 。这两种索引只能选择一种，且无法动态更改，一旦更改必须重启 InfluxDB 。


In-memory ：索引被存储在内存中，这也是默认使用的方式，性能更高。


TSI（ Time Series Index ）：In-memory 索引可以支持千万级别的 series ，然而内存资源终归是有限的，为了支持亿级和十亿级别的 series 数据，TSI 应运而生，其会将索引映射到磁盘文件上。


索引相关配置项（默认的配置文件为 influxdb.conf ）：

* 索引方式，inmem 或者 tsi1 ：
```
index-version = "inmem"
```

* in-memory 相关设置：
```
max-series-per-database = 1000000
max-values-per-tag = 100000
```
max-series-per-database ：每个数据库允许的最大 series 数量，默认一百万，一旦达到上限，再写入新的 series 则会得到一个 500 错误，向已经存在的 series 写入数据不受影响。设置为 0 则意味着没有限制。


max-values-per-tag ：每个 tag key 允许的最大 tag values 数量，默认十万，类似的，一旦达到上限，无法写入新的 tag value ，而向已经存在的 tag value 写入数据不受影响。设置为 0 则意味着没有限制。


* TSI（ tsi1 ）相关设置：

```
max-index-log-file-size = "1m"
series-id-set-cache-size = 100
```
max-index-log-file-size ：预写日志的文件大小达到多大的阈值之后，将其压缩为索引文件，阈值越低，压缩越快，堆内存使用率越低，但会降低写入的吞吐量。


series-id-set-cache-size ：使用内存缓存的 series 集的大小，由于 TSI 索引存储在了磁盘文件中，因此使用时需要额外的计算工作，但如果将索引结果缓存起来的话就可以避免重复的计算，提高查询性能。默认缓存 100 个 series ，这个值越大则使用的堆内存越大，设置为 0 则不缓存。

---
相关文章：
- [时序数据库 InfluxDB（一）](/posts/influxdb/1/)
- [时序数据库 InfluxDB（二）](/posts/influxdb/2/)
- [时序数据库 InfluxDB（三）](/posts/influxdb/3/)
- [时序数据库 InfluxDB（四）](/posts/influxdb/4/)
- [时序数据库 InfluxDB（五）](/posts/influxdb/5/)
- [时序数据库 InfluxDB（六）](/posts/influxdb/6/)
- [时序数据库 InfluxDB（七）](/posts/influxdb/7/)

