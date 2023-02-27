# 同步 MySQL 数据至 Elasticsearch/Redis/MQ 等的五种方式


# 同步 MySQL 数据至 Elasticsearch/Redis/MQ 等的五种方式

在实际应用中，我们经常需要把 MySQL 的数据同步至其它数据源，也就是在对 MySQL 的数据进行了新增、修改、删除等操作后，把该数据相关的业务逻辑变更也应用到其它数据源，例如：
- MySQL -> Elasticsearch ，同步 ES 的索引
- MySQL -> Redis ，刷新缓存
- MySQL -> MQ (如 Kafka 等) ，投递消息

本文总结了五种数据同步的方式。

## 1. 业务层同步

![业务层同步](https://raw.githubusercontent.com/RifeWang/images/master/sync-from-mysql-service.png)

由于对 MySQL 数据的操作也是在业务层完成的，所以在业务层同步操作另外的数据源也是很自然的，比较常见的做法就是在 ORM 的 hooks 钩子里编写相关同步代码。

这种方式的缺点是，当服务越来越多时，同步的部分可能会过于分散从而导致难以更新迭代，例如对 ES 索引进行不兼容迁移时就可能会牵一发而动全身。


## 2. 中间件同步

![中间件同步](https://raw.githubusercontent.com/RifeWang/images/master/sync-from-mysql-middleware.png)

当应用架构演变为微服务时，各个服务里可能不再直接调用 MySQL ，而是通过一层 middleware 中间件，这时候就可以在中间件操作 MySQL 的同时同步其它数据源。

这种方式需要中间件去适配，具有一定复杂度。

## 3. 定时任务根据 updated_at 字段同步

![定时任务根据 updated_at 同步](https://raw.githubusercontent.com/RifeWang/images/master/sync-from-mysql-updated_at.png)

在 MySQL 的表结构里设置特殊的字段，如 updated_at（数据的更新时间），根据此字段，由定时任务去查询实际变更的数据，从而实现数据的增量更新。

这种方式你可以使用开源的 Logstash 去完成。

当然缺点也很明显，就是无法同步数据的删除操作。

## 4. 解析 binlog 同步

![解析 binlog 同步](https://raw.githubusercontent.com/RifeWang/images/master/sync-from-mysql-canal.png)

比如著名的 [canal](https://github.com/alibaba/canal) 。

通过伪装成 slave 去解析 MySQL 的 binary log 从而得知数据的变更。

这是一种业界比较成熟的方案。

这种方式要求你将 MySQL 的 `binlog-format` 设置为 `ROW` 模式。

## 5. 解析 binlog -- mixed / statement 格式

MySQL 的 `binlog` 有三种格式：
- `ROW` 模式，binlog 按行的方式去记录数据的变更；
- `statement` 模式，binlog 记录的是 SQL 语句；
- `mixed` 模式时，混合以上两种，记录的可能是 SQL 语句或者 `ROW` 模式的每行变更；

某些情况下，可能你的 MySQL `binlog` 无法被设置为 `ROW` 模式，这种时候，我们仍然可以去统一解析 binlog ，从而完成同步，但是这里解析出来的当然还是原始的 SQL 语句或者 `ROW` 模式的每行变更，这种时候是需要我们去根据业务解析这些 SQL 或者每行变更，比如利用正则匹配或者 AST 抽象语法树等，然后根据解析的结果再进行数据的同步。

这种方式的限制也很明显，一是需要自己适配业务解析 SQL ，二是批量更新这种场景可能很难处理，当然如果你的数据都是简单的根据主键进行修改或者删除则能比较好的适用。


## 结语

最后列举几个 binlog 解析的开源库：
- [canal](https://github.com/alibaba/canal)
- [go-mysql](https://github.com/siddontang/go-mysql)
- [zongji](https://github.com/nevill/zongji)



