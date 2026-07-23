+++
draft = false
date = 2026-07-23T11:11:00+08:00
title = "基于 PostgreSQL WAL 构建 CDC 系统：原理与工程实现"
description = "从实际同步场景出发，系统讲解 PostgreSQL WAL、逻辑解码、复制槽、快照衔接与工程实现，并与 MySQL Binlog CDC 进行完整对比。"
images = ["https://raw.githubusercontent.com/RifeWang/images/refs/heads/master/postgresql/00-pg-wal-cdc-cover.png"]
featuredImage = "https://raw.githubusercontent.com/RifeWang/images/refs/heads/master/postgresql/00-pg-wal-cdc-cover.png"
featuredImagePreview = "https://raw.githubusercontent.com/RifeWang/images/refs/heads/master/postgresql/00-pg-wal-cdc-cover.png"
slug = ""
authors = []
tags = ["Database", "PostgreSQL", "CDC", "WAL"]
categories = ["Database"]
externalLink = ""
series = []
disableComments = true
+++

## 序言

数据库中的一行记录发生变化后，影响往往不只停留在数据库内部：

- 商品和内容发生变化，需要及时更新 Elasticsearch 索引；
- 用户资料或配置发生变化，需要刷新 Redis 和派生视图；
- 业务数据需要持续进入 Kafka、数据仓库或数据湖；
- 账户、权限等关键数据的修改，需要留下审计记录并触发后续流程。

这些场景的下游各不相同，却都依赖同一种能力：持续捕获数据库中的 `INSERT`、`UPDATE` 和 `DELETE`，并可靠地把变化传递出去。这就是 **`CDC`（Change Data Capture，变更数据捕获）**。

真正困难的不是“读到一次变化”，而是在消费者重启、网络中断、全量扫描、下游拥塞和主备切换时，仍然做到：

1. 已经提交的数据不丢失；
2. 同一行的变化顺序可控；
3. 消费者可以从正确位置恢复；
4. 存量数据与后续增量能够无缝衔接。

本文从这些问题出发，逐步拆解 PostgreSQL WAL CDC 的原理和工程实现，并与更为人熟悉的 MySQL Binlog CDC 进行对比。

![CDC 的应用场景与实现方式](https://raw.githubusercontent.com/RifeWang/images/refs/heads/master/postgresql/01-cdc-use-cases-and-approaches.png)

---

## 1. 为什么选择日志型 CDC

假设要把数据库变化同步到搜索引擎、缓存或消息队列，最直观的办法是让应用在写数据库后再写一次下游。但只要把问题放到故障场景中，就会发现事情没有那么简单：数据库已经提交，消息发送却失败了，该以谁为准？

常见 `CDC` 方案有四类：

| 方案 | 工作方式 | 优点 | 主要问题 |
|---|---|---|---|
| 应用层投递 | 应用写数据库后主动投递消息 | 能携带业务语义 | 存在双写一致性问题，也难以覆盖批量 SQL 和绕过应用的改库 |
| 查询轮询 | 定期按 `updated_at` 查询 | 简单、权限要求低 | 有延迟、增加主库压力；为感知删除，数据模型通常需要引入 `deleted_at` 等软删除标记 |
| 数据库触发器 | 在业务表上创建触发器，行数据变化时同步写入变更表 | 实时且与事务同步 | 侵入业务、产生写放大，数据库承担额外工作 |
| 日志解析 | 从数据库事务日志还原变化 | 低侵入、低延迟、覆盖完整 | 依赖数据库日志机制，消费端实现更复杂 |

日志解析之所以成为现代 `CDC` 的主流，是因为数据库已经替我们完成了最困难的一步：**把所有已提交变化按可恢复的顺序记录下来**。

在 MySQL 中，这份日志是 binlog；在 PostgreSQL 中，则是 `WAL`（Write-Ahead Log，预写日志）。两者目标相似，但底层语义和工程责任并不相同。

---

## 2. PostgreSQL 与 MySQL CDC 全景对比

如果已经熟悉 MySQL Binlog CDC，可以先通过下面这张表建立 PostgreSQL 的认知坐标。

| 对比维度 | MySQL Binlog CDC | PostgreSQL WAL CDC | 工程影响 |
|---|---|---|---|
| 日志基础 | binlog 原本主要服务于复制和增量恢复 | `WAL` 原本主要服务于崩溃恢复和物理复制 | PostgreSQL 需要先做逻辑解码，MySQL ROW event 已具有较强的逻辑语义 |
| `CDC` 前置配置 | `log_bin=ON`、`binlog_format=ROW` | `wal_level=logical` | 都需要实例级配置，变更部分参数可能需要重启 |
| 解码方式 | 解析 binlog event | Logical Decoding + Output Plugin | PostgreSQL 可通过插件选择输出协议 |
| 内置输出格式 | ROW event | `pgoutput` 二进制协议 | 两者通常都交给成熟客户端库解析 |
| 数据范围 | 通常在消费端按库表过滤 | Publication 在服务端声明表和操作范围；PG 15+ 支持行过滤与列列表 | PostgreSQL 能在源端缩小发布范围 |
| 消费位置 | binlog file/position 或 GTID | LSN | 都要持久化并用于断点恢复，但语义并不完全等价 |
| 进度保存 | 通常由消费者保存位点 | Replication Slot 在服务端保存确认水位 | PostgreSQL 对消费者更友好，但把资源保留责任放到了数据库 |
| 日志保留 | binlog 通常按时间或空间过期 | Slot 会保留消费者仍然需要的 `WAL` | PG 需要关注 Slot 对 `WAL` 保留量的影响；MySQL 需要关注日志过期后的续传问题 |
| UPDATE/DELETE 旧值 | `binlog_row_image` 控制行镜像 | 逐表设置 `REPLICA IDENTITY` | PostgreSQL 上线前要逐表检查旧值需求 |
| 全量与增量衔接 | 一致性快照 + binlog 位点/GTID | 一致点 LSN + 导出快照 | 原理相同：先确定日志边界，再读取同一时刻的存量 |
| DDL | binlog 通常包含 DDL Query Event | 逻辑复制默认不传播 DDL | PostgreSQL 需要单独设计 Schema 演进机制 |
| 大字段 | 受 row image 配置影响 | 需要识别 unchanged TOAST | PG 消费端不能把“未变化”误认为 `NULL` |
| 故障切换 | 关注 GTID 连续性和拓扑切换 | 关注 timeline、Slot 是否延续及新主库起始 LSN | 两者都不能只靠重连地址判断是否能安全续传 |

![MySQL Binlog CDC 与 PostgreSQL WAL CDC 对比](https://raw.githubusercontent.com/RifeWang/images/refs/heads/master/postgresql/02-mysql-postgresql-cdc-comparison.png)

最值得记住的区别是：

> MySQL CDC 更像“消费者追赶数据库统一保留的一条 binlog”；PostgreSQL CDC 更像“数据库通过 Slot 为消费者记录进度，并为它保留尚未确认的 `WAL`”。

这种设计让 PostgreSQL 的断点恢复更直接。相应地，当消费者长时间没有推进时，Slot 所需的 `WAL` 保留量会增加，因此需要配合容量规划和监控。

---

## 3. 一张图看懂 PostgreSQL CDC

先不看协议细节，一套 PostgreSQL CDC 链路可以概括为：

![PostgreSQL WAL CDC 整体链路](https://raw.githubusercontent.com/RifeWang/images/refs/heads/master/postgresql/03-pg-wal-cdc-architecture.png)

一次事务从数据库执行到变更投递，大致经历以下链路：

1. PostgreSQL 在修改数据页前先写入相关 `WAL`，提交时再写入提交记录；
2. 事务确认提交后，Logical Decoding 将底层变化还原为行级语义；
3. Output Plugin（通常是 `pgoutput`）将变化编码为协议消息；
4. `CDC` 消费者通过 Replication Slot 持续读取；
5. 消费者完成转换、路由和投递；
6. 下游确认成功后，消费者才向 PostgreSQL 上报安全 LSN；
7. PostgreSQL 推进 Slot 水位，并回收不再需要的 `WAL`。

这是一条闭环。只读不确认，Slot 需要保留的 `WAL` 会持续增加；尚未投递成功就提前确认，则可能永久丢数据。

---

## 4. `WAL` 如何变成行级事件

`WAL` 是 PostgreSQL 保证持久性和崩溃恢复的基础。数据页落盘前，相关修改必须先写入 `WAL`，这也是“Write-Ahead”的含义。

原始 `WAL` 是面向数据页和内部操作的物理/物理逻辑日志，适合：

- 崩溃恢复：重启后重放 `WAL`，使数据恢复到一致状态；
- 物理流复制：将 `WAL` 传给备库，得到主库的数据页副本。

例如，原始 `WAL` 记录可能表达的是“某个数据页发生了变化”，却不能直接告诉消费者“`public.products` 表中主键为 42 的商品价格变成了 99”。前者是底层存储变化，后者才是 `CDC` 需要的行级业务语义。

为了把这种底层存储变化转换成 `CDC` 能够消费的行级事件，PostgreSQL 从 9.4 开始提供 **Logical Decoding（逻辑解码）**。服务端读取 `WAL`，结合事务、表结构等信息，将其还原为带有表和行语义的变化。

要让 `WAL` 携带逻辑解码需要的信息，实例参数 `wal_level` 必须设为 `logical`。

逻辑解码只负责还原变化，最终输出格式由 **Output Plugin** 决定：

| 插件 | 格式 | 提供方式 | 适合场景 |
|---|---|---|---|
| `pgoutput` | PostgreSQL 二进制逻辑复制协议 | PG 10+ 内置 | 生产系统首选，标准、性能好、无需安装扩展 |
| `wal2json` | JSON | 第三方插件 | 调试友好，消费端容易接入 |
| `decoderbufs` | Protobuf | 第三方插件 | 二进制输出，Debezium 早期方案 |
| `test_decoding` | 文本 | 随 PostgreSQL contrib 提供，取决于安装包 | 学习和测试，不面向生产 |

新系统通常优先使用 `pgoutput`。它的缺点是人眼不可读，但这属于客户端库应解决的问题，不必为了调试便利而长期承担第三方插件的运维成本。

---

## 5. 四个核心概念

开始消费逻辑复制流之前，需要先理解四个彼此关联的概念：

| 概念 | 可以理解为 | 回答的问题 |
|---|---|---|
| Publication | 发布清单 | 哪些表和操作需要输出？ |
| Replication Slot | 消费者书签 | 这个消费者已经读到哪里？ |
| LSN | `WAL` 坐标 | 某次变化位于日志的什么位置？ |
| REPLICA IDENTITY | 旧记录识别规则 | UPDATE/DELETE 时如何定位变化前的记录？ |

它们共同描述了一次逻辑复制：Publication 确定捕获范围，LSN 标记每个变化的位置，Replication Slot 使用 LSN 保存消费者进度，REPLICA IDENTITY 决定 UPDATE 和 DELETE 能携带哪些旧值。

![PostgreSQL 逻辑复制的四个核心概念](https://raw.githubusercontent.com/RifeWang/images/refs/heads/master/postgresql/04-pg-cdc-core-concepts.png)

**Publication：捕获哪些变化**

Publication 是 PostgreSQL 对“哪些表参与逻辑复制”的服务端声明：

Publication 从早期版本起就能选择表和操作类型，PG 15+ 进一步支持行过滤与列列表：

```sql
-- 发布所有表
CREATE PUBLICATION cdc_pub FOR ALL TABLES;

-- 只发布指定表
CREATE PUBLICATION cdc_pub
FOR TABLE public.users, public.products;

-- 只发布 INSERT 和 UPDATE
CREATE PUBLICATION cdc_pub
FOR TABLE public.users
WITH (publish = 'insert, update');

-- PG 15+：行过滤用于 UPDATE/DELETE 时，过滤列必须包含在 Replica Identity 中
ALTER TABLE public.orders REPLICA IDENTITY FULL;

CREATE PUBLICATION paid_orders_pub
FOR TABLE public.orders WHERE (status = 'paid');

-- PG 15+：只发布指定列
CREATE PUBLICATION product_price_pub
FOR TABLE public.products (id, name, price);
```

与 MySQL 常见的消费端过滤相比，Publication 可以在源端缩小范围。不过它不是权限系统：发布范围、用户读取权限和复制权限仍需分别配置。

**Replication Slot：消费者读到哪里**

设想数据仓库维护两小时，`CDC` 消费者暂时离线。它恢复后应从哪里继续？离线期间的日志又由谁保留？

Replication Slot 就是 PostgreSQL 为消费者维护的服务端书签。逻辑 Slot 主要关注两个水位：

- `restart_lsn`：该 Slot 仍可能需要的最早 `WAL` 位置；
- `confirmed_flush_lsn`：消费者已经明确确认安全持久化的位置。

在未超过 `max_slot_wal_keep_size` 等保留限制时，PostgreSQL 会为 Slot 保留仍然需要的 `WAL`，因此消费者能够断点续传。如果超过限制，Slot 可能因所需 `WAL` 已被移除而失效；消费者长时间没有推进时，也需要关注 `WAL` 保留量的变化。

这体现了 PostgreSQL 与 MySQL 在日志保留机制上的差异：

> MySQL 消费者长时间没有推进时，需要关注 binlog 是否过期；PostgreSQL 消费者长时间没有推进时，需要关注 `WAL` 保留量的变化。

**LSN：变化发生在哪里**

LSN（Log Sequence Number）是 `WAL` 中的位置，文本形式类似 `16/B374D848`。在同一集群的正常 `WAL` 序列语境中，它随 `WAL` 写入向前推进。

`CDC` 系统会同时遇到多个 LSN：

- 服务端当前写到哪里；
- 复制流已经发送到哪里；
- 消费者已经处理到哪里；
- 下游已经安全确认到哪里。

真正允许上报给 Slot 的，只能是最后一个。

**REPLICA IDENTITY：旧值能看到多少**

DELETE 后已经没有新行，UPDATE 也可能需要知道修改前的分区键。PostgreSQL 必须决定在 `WAL` 中保留哪些旧值，这由逐表属性 `REPLICA IDENTITY` 控制。

| 设置 | UPDATE/DELETE 可用的旧值 | 典型用途 |
|---|---|---|
| `DEFAULT` | DELETE 携带旧主键；UPDATE 仅在主键变化时携带旧键 | 下游按主键定位 |
| `USING INDEX` | DELETE 携带索引旧值；UPDATE 仅在索引键变化时携带旧键 | 没有主键但有合适的唯一键 |
| `FULL` | 所有列 | 审计、计算差异、旧分片数据清理 |
| `NOTHING` | 不提供旧键 | 发布 UPDATE/DELETE 时无法满足复制要求，相关操作会报错 |

```sql
ALTER TABLE public.accounts REPLICA IDENTITY FULL;
```

MySQL 主要通过 `binlog_row_image` 控制行镜像，而 PostgreSQL 是逐表设置。不要机械地给所有表设为 `FULL`：它能提供更完整的 before image，也会增加 UPDATE/DELETE 的 `WAL` 体积。应根据下游需求选择。

---

## 6. 逻辑复制的流式协议

`CDC` 消费者使用复制连接连接 PostgreSQL，并通过基于 `COPY` 的流式协议持续接收数据。

![PostgreSQL 逻辑复制协议交互](https://raw.githubusercontent.com/RifeWang/images/refs/heads/master/postgresql/05-pg-logical-replication-protocol.png)

协议中的关键动作是：

- `IDENTIFY_SYSTEM`：获得集群标识、timeline 和当前 `WAL` 位置；
- `START_REPLICATION SLOT ... LOGICAL <lsn>`：从指定 Slot 和 LSN 开始消费；
- `XLogData`：承载逻辑解码数据；
- Primary Keepalive：服务端心跳，可能要求消费者立即回复；
- Standby Status Update：消费者上报已接收、已刷盘和已应用的位置。

使用 `pgoutput` 时，还需要解析 `Begin`、`Commit`、`Relation`、`Insert`、`Update`、`Delete`、`Truncate` 等逻辑消息。典型事务可以理解为：

```text
Begin → Relation（必要时）→ Insert/Update/Delete... → Commit
```

默认模式下，逻辑复制流按事务提交顺序输出，`Begin` 与 `Commit` 之间属于同一事务。PG 14+ 开启大事务流式解码后，一个事务可能分成多个 `Stream Start/Stop` 片段，并最终收到 `Stream Commit` 或 `Stream Abort`；消费端应按 xid 归组，并在确认提交后再应用。生产实现还应使用成熟客户端库处理 CopyData 和插件协议，不要假设一次网络读取恰好对应一条业务变更。

需要特别澄清的是：正常的 PostgreSQL 主备切换通常仍属于同一数据库集群，`systemid` 不一定变化。恢复时还必须检查 timeline、逻辑 Slot 是否已同步到新主库，以及请求的 LSN 是否仍然有效。

---

## 7. 服务端配置与最小示例

在继续讨论快照和工程实现之前，先用一组明确的示例对象跑通最小链路。后续命令统一使用以下名称：

| 对象 | 示例值 |
|---|---|
| 数据库 | `appdb` |
| Schema | `public` |
| 业务表 | `products` |
| 复制用户 | `cdc_user` |
| Publication | `cdc_pub` |
| Replication Slot | `demo_slot` |
| 客户端网段 | `10.0.0.0/8` |

**配置 PostgreSQL 实例**

```conf
# postgresql.conf
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

这些值至少应覆盖实际 Slot 和复制连接数量，并预留运维空间。修改 `wal_level` 等启动参数后需要重启实例。

**准备数据库、示例表和复制用户**

先创建示例数据库，再使用 `psql` 的 `\connect` 命令切换到该数据库。以下示例中的所有对象都位于 `appdb`：

```sql
CREATE DATABASE appdb;

\connect appdb

CREATE TABLE public.products (
    id         bigint PRIMARY KEY,
    name       text NOT NULL,
    price      numeric(12, 2) NOT NULL,
    attributes jsonb,
    updated_at timestamptz NOT NULL DEFAULT now()
);

-- 后续消息示例需要展示完整 before image
ALTER TABLE public.products REPLICA IDENTITY FULL;

CREATE USER cdc_user
WITH REPLICATION LOGIN PASSWORD 'replace-with-a-secret';

GRANT CONNECT ON DATABASE appdb TO cdc_user;
GRANT USAGE ON SCHEMA public TO cdc_user;
GRANT SELECT ON TABLE public.products TO cdc_user;
```

其中 `REPLICATION` 权限用于建立复制连接，`SELECT` 权限用于读取已发布表和执行全量快照。

逻辑复制连接需要指定实际数据库，因此 `pg_hba.conf` 应放行示例数据库 `appdb`，而不是物理复制连接使用的特殊数据库关键字 `replication`：

```conf
# TYPE  DATABASE  USER      ADDRESS       METHOD
host    appdb     cdc_user  10.0.0.0/8    scram-sha-256
```

修改后执行 `SELECT pg_reload_conf();` 或 reload 配置。

**生产链路：创建 Publication**

```sql
CREATE PUBLICATION cdc_pub FOR TABLE public.products;
```

使用 `pgoutput` 构建生产链路时，这条命令声明需要捕获 `public.products`。生产消费者通过复制协议订阅 `cdc_pub`，Publication 决定进入逻辑复制流的表和操作范围。

**本地验证链路：使用 test_decoding 观察变化**

为了让变化可以直接阅读，下面单独使用文本输出插件 `test_decoding`。这条本地验证链路不读取上面的 `cdc_pub`，也不等同于生产环境中的 `pgoutput + Publication` 链路。

```sql
SELECT *
FROM pg_create_logical_replication_slot('demo_slot', 'test_decoding');

INSERT INTO public.products (id, name, price)
VALUES (1, 'Mechanical Keyboard', 699.00);

UPDATE public.products
SET price = 649.00, updated_at = now()
WHERE id = 1;

DELETE FROM public.products WHERE id = 1;
```

上述三条语句分别产生一条 INSERT、UPDATE 和 DELETE 变化。现在读取 Slot 中已经解码的内容：

```sql
SELECT lsn, xid, data
FROM pg_logical_slot_get_changes('demo_slot', NULL, NULL);
```

`pg_logical_slot_get_changes` 会消费变化并推进位置；`pg_logical_slot_peek_changes` 只查看、不消费，更适合反复调试。

用完后及时清理测试 Slot：

```sql
SELECT pg_drop_replication_slot('demo_slot');
```

这个例子使用 `test_decoding` 和 SQL 函数是为了让变化肉眼可见。生产系统通常使用 `pgoutput`、Publication 和复制协议持续消费，而不是轮询上述 SQL 函数。

---

## 8. 存量快照与增量如何衔接

逻辑复制只能提供 Slot 所保留范围内的增量变化，不能自动还原表中早已存在的全部数据。因此，第一次启动 `CDC` 时需要完成两项工作：

1. 读取当前已有的数据，为下游建立一份完整基线；
2. 从一个确定的 LSN 开始消费此后的增量变化。

真正的难点不在于分别完成全量扫描和增量消费，而在于让两者共享同一个边界。可以把这个边界理解成一次切分：

> 边界之前的数据状态由快照负责，边界之后提交的变化由逻辑复制流负责。

例如，将 `appdb` 中的 `public.products` 全量导入搜索引擎可能需要一小时，而扫描期间商品价格仍在变化。如果等扫描结束后才临时确定增量起点，中间提交的变化就可能遗漏；如果没有控制应用顺序，较旧的快照数据也可能覆盖较新的增量结果。

![PostgreSQL CDC 存量快照与增量衔接](https://raw.githubusercontent.com/RifeWang/images/refs/heads/master/postgresql/06-pg-cdc-snapshot-stream-handoff.png)

PostgreSQL 通过两个相互对应的值建立这条边界：

| 返回值 | 作用 |
|---|---|
| `consistent_point` | 逻辑复制流的安全起始 LSN，边界之后提交的变化可以从这里读取 |
| `snapshot_name` | 与该边界对应的 MVCC 快照，让一个或多个扫描连接看到同一时刻的数据 |

**完整衔接过程**

1. **建立边界**：通过复制协议创建逻辑 Slot，并要求导出快照，获得 `consistent_point` 和 `snapshot_name`。
2. **导入快照**：一个或多个普通数据库连接开启只读的 `REPEATABLE READ` 事务，在执行查询前导入同一个 `snapshot_name`。
3. **扫描存量**：各连接并发读取 `public.products`，将快照数据写入下游；所有快照数据确认完成前，不应用更晚的增量事件。
4. **接续增量**：从 `consistent_point` 启动逻辑复制，读取建立边界之后提交的变化。

快照扫描连接的示例：

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ READ ONLY;
SET TRANSACTION SNAPSHOT '00000003-0000001A-1';

SELECT id, name, price, updated_at
FROM public.products
WHERE id >= 1 AND id < 100000;

COMMIT;
```

导出快照有生命周期限制。在所有扫描连接成功执行 `SET TRANSACTION SNAPSHOT` 之前，应保持创建快照的复制连接打开，并避免在该连接上继续执行其他复制命令。具体客户端库通常会封装创建 Slot 和导出快照的协议细节，但调用方仍需保证这个时序。

**大表如何并发扫描**

所有 worker 导入同一个快照后，可以用不同条件拆分扫描任务，同时保持一致的数据视图。常见分片方式包括：

- 按主键范围：`WHERE id >= ? AND id < ?`，简单，但受主键分布影响；
- 按业务分区：按日期、租户或原生分区表并发扫描；
- 按 CTID 页范围：更接近物理顺序，适合缺少均匀主键的大表。

CTID 会因 `VACUUM FULL`、`CLUSTER` 等表重写操作变化。若用 CTID 分片，快照期间应避免这些操作，并在实际版本和表结构上验证扫描计划。

**应用顺序与幂等**

最容易验证的实现是：先等待全部快照数据写入并确认，再从 `consistent_point` 顺序应用增量。这样，扫描期间发生的价格修改会暂存在 Slot 所保留的 `WAL` 中，并在快照完成后依次追平。

如果为了降低延迟而同时处理快照和增量，就必须保证旧的快照行不会覆盖更新的增量状态，例如设置阶段屏障；需要并发应用时，可将 `consistent_point` 作为快照基线版本，并使用事务提交 LSN 与事务内序号判断增量顺序。

日志型 `CDC` 通常采用 **at-least-once（至少一次）**，重连和投递重试仍可能产生重复事件。因此下游应按主键执行 UPSERT，并结合事务提交 LSN、事务内序号或事件 ID 实现幂等。大表扫描持续时间较长时，既要监控 Slot 所需的 `WAL` 保留量，也要关注长事务快照对 VACUUM 和表膨胀的影响。

如果 Publication 使用了行过滤或列列表，快照查询也应采用等价的过滤条件和字段投影，保证存量基线与增量范围一致。

MySQL 使用的具体原语不同，但原则相同：先取得一致性快照及其对应的 binlog position/GTID，再完成存量读取与增量接续。

---

## 9. 工程实现的正确性核心

健壮的 `CDC` 消费者不仅要能解析协议，还要正确处理确认、并发、顺序和异常数据。核心问题可以归纳如下：

| 工程问题 | 处理原则 |
|---|---|
| LSN 确认时机 | 完整事务在下游确认成功后，才能上报该事务的 `Commit.end_lsn` |
| 并发完成与安全水位 | 只推进连续完成的事务边界；在途窗口满时暂停读取，形成背压 |
| 同一行的变更顺序 | 使用 `schema.table.primary_key` 作为稳定路由键，多实例统一哈希算法、编码和种子 |
| unchanged TOAST | 保留“字段未变化”的语义并由下游合并旧值，不能将其当作 `NULL` 或空值 |
| 大事务 | 限制在途数据并做好背压；PG 14+ 可结合客户端和插件对流式解码大事务的支持 |
| 超大单行 | 使用 Chunking 分片、Claim Check 外置内容，或在接受时序差异的前提下回源查询 |
| DDL 与 Schema 演进 | 使用 Event Trigger、`pg_catalog`、Schema History 或数据库迁移事件同步结构变化 |

![生产级 CDC 的正确性闭环与运维信号](https://raw.githubusercontent.com/RifeWang/images/refs/heads/master/postgresql/07-pg-cdc-correctness-operations.png)

其中最关键的是区分“已经收到”“已经处理”和“已经安全确认”三个位置。推荐的推进顺序是：

```text
接收 → 解码 → 投递 → 下游确认 → 记录安全水位（如需要）→ 上报安全 LSN
```

并发不会改变这条原则。如果较晚提交的事务已经完成，而更早的事务仍在重试，安全水位必须停在较早事务之前。整体投递通常采用 **at-least-once（至少一次）**，再通过下游幂等吸收重连和重试产生的重复事件。

PostgreSQL 逻辑复制默认不传播 DDL；MySQL binlog 通常能看到 DDL Query Event，但两者都需要显式处理 Schema 事件与行数据之间的顺序和兼容性。

---

## 10. 消息模型设计

快照事件和增量事件最好使用同一套结构，只通过 `op` 区分。一个实用的事件通常包含：

```json
{
  "key": "public.products:42",
  "source": "production-pg",
  "op": "UPDATE",
  "commit_lsn": "16/B374D848",
  "event_index": 1,
  "xid": 123456,
  "commit_ts": "2026-07-23T10:00:00Z",
  "schema": "public",
  "table": "products",
  "primary_key": {"id": 42},
  "before": {"price": 100},
  "after": {"price": 99},
  "unchanged_toast": ["attributes"]
}
```

设计时重点考虑：

- **路由稳定**：key 能稳定标识同一行，保证分区内顺序；
- **位置可比较**：携带事务提交 LSN 和事务内序号，支持排错、排序和去重；
- **事务可追踪**：保留 xid，必要时支持事务级聚合；
- **旧值语义明确**：区分缺失、`NULL` 和 unchanged TOAST；
- **Schema 可演进**：使用 Protobuf/Avro 时遵守兼容性规则；
- **快照增量同构**：下游不需要维护两套完全不同的处理逻辑。

消息格式不是越完整越好。完整 before image、列元数据和事务信息都会增加体积，应由下游需求驱动。

---

## 11. 开源工具与客户端

以下列出采用 OSI 认可许可证、可用于 PostgreSQL CDC 的开源工具：

| 工具 | 形态/语言 | 主要用途 | 开源许可证 |
|---|---|---|---|
| Debezium + Kafka Connect | Java · Connector 运行时 | 捕获多种数据库的变化并写入 Kafka | Apache-2.0 |
| Debezium Server | Java · 独立运行时 | 将数据库变化直接输出到多种消息系统 | Apache-2.0 |
| Apache Flink CDC | Java · 分布式数据管道 | 将数据库快照和增量接入 Flink 数据管道 | Apache-2.0 |
| Apache SeaTunnel | Java · 数据集成平台 | 在多种数据源和目标之间同步批量与增量数据 | Apache-2.0 |
| `xataio/pgstream` | Go · CLI/库 | PostgreSQL 到 Kafka、OpenSearch、Webhook 或 PostgreSQL | Apache-2.0 |
| `ConduitIO/conduit` | Go · Connector 框架 | 通过 Connector 连接 PostgreSQL 与其他数据系统 | Apache-2.0 |
| PeerDB | Go/Rust · 复制平台 | PostgreSQL 到分析系统、队列和对象存储 | AGPL-3.0 |
| Sequin | Elixir · 自托管服务 | PostgreSQL 到队列、搜索引擎和 Webhook | MIT |

Airbyte 的主要代码采用 ELv2，Materialize 当前版本采用 BSL 1.1。二者源码可见，但许可证不是 OSI 认可的开源许可证，因此没有放入上述开源工具清单。PeerDB 已在 2026 年改为 AGPL-3.0，属于开源软件，但使用时需要遵守较强的 copyleft 条款。

学习和排障时，可以使用 `pg_recvlogical`、`test_decoding` 或 `wal2json` 直接观察解码结果。

用于自研的常见客户端也可以直接对比：

| 语言 | 客户端/API | 作用 | 开源许可证 |
|---|---|---|---|
| Go | `jackc/pglogrepl` + `jackc/pgx` | 复制协议、`pgoutput` 消息和 PostgreSQL 连接 | MIT |
| Java | PostgreSQL JDBC `PGReplicationStream` | JDBC 驱动内置的逻辑复制接口 | BSD-2-Clause |
| Python | `psycopg2` | `LogicalReplicationConnection` 等逻辑复制接口 | LGPL-3.0-or-later |
| Rust | `supabase/etl` | 构建 PostgreSQL 逻辑复制与实时数据管道的 Rust 框架 | Apache-2.0 |
| Node.js | `kibae/pg-logical-replication` | 支持 `pgoutput`、`wal2json` 等输出插件 | MIT |
| C/C++ | `libpq` | PostgreSQL 官方底层客户端库 | PostgreSQL License |

---

## 12. 监控、故障恢复与上线检查

生产运维主要关注三件事：消费者是否跟得上、Slot 需要保留多少 `WAL`，以及故障后能否从正确位置恢复。

```sql
SELECT
    slot_name,
    active,
    restart_lsn,
    confirmed_flush_lsn,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) AS retained_wal
FROM pg_replication_slots
WHERE slot_type = 'logical';
```

`restart_lsn` 用于估算 Slot 当前需要保留的 `WAL` 范围，`confirmed_flush_lsn` 用于观察消费者的确认进度。除此之外，保留以下几组指标即可：

| 监控方向 | 关键指标 |
|---|---|
| Slot 状态 | `active`、保留的 `WAL` 大小、确认 LSN 推进速度 |
| 端到端延迟 | 事件提交到下游确认的 P95/P99 |
| 消费者状态 | 在途数量、重试次数、背压时间和重连次数 |
| 主库资源 | `WAL` 目录大小和磁盘剩余空间 |

故障恢复不能只做断线重连。恢复前应确认仍连接到预期集群、timeline 和 Slot 状态有效、所需 `WAL` 仍然存在，并核对本地安全水位与 `confirmed_flush_lsn` 是否符合预期；不一致时应停止消费并告警。正常主备切换不一定改变 `systemid`，因此还需要提前验证所用 PostgreSQL 版本下的 Slot 同步或 failover slot 方案。

上线前可以归纳为六项检查：

- 实例参数、复制用户权限和 Publication 范围正确；
- 各表主键与 `REPLICA IDENTITY` 满足下游需求；
- 快照与增量衔接经过并发写入验证；
- LSN 只在下游确认后连续推进；
- unchanged TOAST、DDL、大事务和重复事件有明确处理策略；
- Slot、端到端延迟和磁盘告警已配置，主备切换与废弃 Slot 清理流程已演练。

---

## 小结

PostgreSQL WAL CDC 的核心，是把数据库中的变化转化为一条可以持续消费、确认和恢复的事件流。`WAL` 提供变化来源，Logical Decoding 恢复行级语义，Publication 确定捕获范围，LSN 标记日志位置，Replication Slot 保存消费者进度，`REPLICA IDENTITY` 则决定 UPDATE 和 DELETE 能提供哪些旧值。

第一次启动 `CDC` 时，还需要处理存量与增量的边界：导出快照负责边界之前的数据状态，逻辑复制流负责边界之后提交的变化。只要两者使用同一个一致性起点，就不需要暂停业务写入。

工程实现中最重要的原则，是完整事务在下游确认成功后才能推进安全 LSN。并发消费只能确认连续完成的事务边界，同一行的变化需要保持顺序，重复事件则通过主键 UPSERT、事务提交 LSN 与事务内序号或事件 ID 实现幂等。系统还需要明确处理 unchanged TOAST、DDL、大事务、超大消息和主备切换。

PostgreSQL 与 MySQL 的日志型 `CDC` 原理相近，但日志保留方式不同：MySQL 需要关注 binlog 是否在消费者恢复前过期，PostgreSQL 需要关注 Slot 进度带来的 `WAL` 保留量变化。

最终衡量一套 `CDC` 系统是否可靠，可以归结为三点：全量与增量之间没有遗漏，数据确认位置能够安全恢复，下游在重复、延迟和乱序情况下仍能收敛到正确状态。
