+++
draft = false
date = 2026-07-24T14:00:00+08:00
title = "面向 MySQL 用户的 PostgreSQL 快速上手指南"
description = "面向熟悉 MySQL 的开发者，从架构、连接、权限、类型、SQL、事务、Vacuum 和索引等方面对照 PostgreSQL，帮助你快速建立正确的开发心智模型。"
slug = ""
authors = []
tags = ["Database", "MySQL", "PostgreSQL"]
categories = ["Database"]
externalLink = ""
series = []
disableComments = true
+++

## 序言：它们不只是“语法略有不同”

如果你已经熟悉 MySQL，那么第一次使用 PostgreSQL 时，大部分内容看起来都很亲切：它们都是关系型数据库，都支持事务、索引、约束和 SQL，也都能被主流语言和 ORM 良好支持。

真正容易出问题的，恰恰是那些“看起来差不多”的地方：

- MySQL 的 database，到了 PostgreSQL 中应该对应 database 还是 schema？
- `AUTO_INCREMENT`、`DATETIME`、`TINYINT(1)` 和 `JSON` 应该换成什么类型？
- `INSERT IGNORE`、`REPLACE INTO`、`ON DUPLICATE KEY UPDATE` 如何改写？
- 两边都有 MVCC 和 Repeatable Read，它们的并发语义是否相同？
- 为什么 PostgreSQL 需要特别关注长事务、Vacuum 和连接数？

本文不讨论“MySQL 和 PostgreSQL 谁更好”，也不会把所有功能逐项罗列，而是先看清 PostgreSQL 的整体架构、连接模型和权限体系，再依次走过数据类型与约束、SQL 方言、事务并发、Vacuum 和索引。

需要记住的主线只有一条：

> MySQL 经验可以成为理解 PostgreSQL 的坐标，但不能成为替 PostgreSQL 做决定的默认答案。

---

## 1. 整体架构：多出来的一层 schema

### 1.1 从三级结构到四级结构

理解 PostgreSQL 的第一步，是先看清两者的对象层级。MySQL 通常是实例、database、table 三级，PostgreSQL 在 database 与 table 之间多出一层 schema，形成四级：

| MySQL | PostgreSQL | 说明 |
|---|---|---|
| instance | cluster / instance | 管理一组 database |
| database | database 或 schema | 如何映射取决于事务、权限和隔离需求 |
| 无独立对应层级 | schema | database 内部的命名空间 |
| table | table | table 必须属于某个 schema |

索引、sequence、类型和函数等对象也属于某个 schema。一个 PostgreSQL database 中可以有多个 schema，不同 schema 中可以存在同名表，例如 `account.users` 和 `audit.users`。

多出来的这一层会直接改变你划分业务的方式，因为 PostgreSQL 的一条连接只能进入一个 database，跨 database 的 JOIN 写不出来。两个模块如果需要在同一个事务里查询和修改，应该放进同一 database 的不同 schema；拆成多个 database 当然也行，但跨库访问需要额外使用 `postgres_fdw`、`dblink` 或应用层的多个连接。

因此在 PostgreSQL 里划分库和 schema 时，更实用的判断方式是：

- 需要跨模块事务和 JOIN：优先考虑同一 database 下的多个 schema；
- 需要强隔离或独立运维：考虑拆成多个 database；
- 不要仅因为在 MySQL 里习惯用 database 分隔业务，就在 PostgreSQL 里也建一堆 database。

### 1.2 `public` 只是默认 schema

新建 database 后通常会看到一个名为 `public` 的 schema。它不是 database，只是默认命名空间。

当你写 `SELECT * FROM users` 而不指定 schema 时，PostgreSQL 会按 `search_path` 依次查找：

```sql
SHOW search_path;
-- "$user", public
```

这很方便，但也意味着“这条 SQL 到底访问了哪张表”取决于会话状态。生产代码更稳妥的做法是显式写 `app.users`，不要依赖搜索路径。

### 1.3 连上去看一眼：`psql` 与 MySQL CLI 对照

上面这些层级，连上去挨个看一遍最直观。使用 Docker 可以快速得到一个开发实例：

```bash
docker run --name pg-dev \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=shop \
  -p 5432:5432 \
  -d postgres:18
```

连接串的形式和 MySQL 类似，`psql` 可以直接接受：

```bash
psql "postgresql://postgres:postgres@localhost:5432/shop"
```

`psql` 大概是从 MySQL 转过来第一个不适应的东西。在 MySQL 客户端里，`SHOW DATABASES`、`USE shop`、`SHOW TABLES`、`DESC users` 都是 SQL，敲下去带分号就能跑；到了 `psql`，对应的全是反斜杠开头的客户端命令，原来的肌肉记忆直接换来一个语法错误。

常用命令可以这样对照：

| 目的 | MySQL | `psql` |
|---|---|---|
| 查看数据库 | `SHOW DATABASES` | `\l` |
| 切换数据库 | `USE shop` | `\c shop`，本质是重新连接 |
| 查看表 | `SHOW TABLES` | `\dt` |
| 查看表结构 | `DESC users` | `\d app.users` |
| 查看用户 | 查询 `mysql.user` | `\du` |
| 查看当前连接 | `SELECT CONNECTION_ID()` | `SELECT pg_backend_pid()` |
| 展开结果 | `\G` | `\x` |

这些命令看着零碎，其实有规律：`d` 是 describe，后面跟对象类型的首字母，`\dt` 表、`\di` 索引、`\dn` schema、`\df` 函数，记住这条就不用背了，`\?` 会列出全部。

另外要分清，反斜杠命令是 `psql` 这个客户端在解析，不是 SQL。应用程序里查元数据得用 `information_schema` 或 `pg_catalog`，发 `\d` 过去数据库是不认的。

---

## 2. 连接模型：一条连接就是一个进程

| 对比项 | MySQL | PostgreSQL |
|---|---|---|
| 一条连接是什么 | 一个线程 | 一个独立的后端进程 |
| 连接开销 | 较低，上千并发连接不罕见 | 每个进程各有内存与调度成本 |
| 应用侧连接池 | 常规做法 | 常规做法，但池的大小要按全局预算 |
| 外部连接池 | 很少需要 | 连接规模大时通常再加一层 |

按全局预算的意思是：所有应用实例的池大小加起来，再加上运维、监控和后台任务要用的连接，总量必须留在 `max_connections` 之内，并给管理员留出余量。池大小没有“CPU 核数乘二”这样的普适答案，应通过连接获取等待、CPU、I/O、锁竞争、查询延迟和吞吐压测决定。

连接一多，还得知道每条连接是谁的。应用侧的连接串建议带上可识别的 `application_name`：

```text
postgresql://user:password@localhost:5432/shop?application_name=order-api
```

它会出现在 `pg_stat_activity` 里，排查慢查询和锁等待时能直接看出问题连接来自哪个服务。

最后，如果前面还加了 PgBouncer 这类外部连接池，注意它常用的 transaction pooling 模式只在一个事务内绑定同一条后端连接，跨事务残留的东西都不可靠——`SET` 设的会话状态（包括 `search_path`）、临时表、`currval()` 都在此列，用 `SET LOCAL` 和 `RETURNING` 代替即可。

---

## 3. 认证授权：只有 role，权限要提前声明

PostgreSQL 不区分“用户”和“角色”两类对象，只有 role。能否登录由 `LOGIN` 属性决定，`CREATE USER` 只是 `CREATE ROLE ... LOGIN` 的别名。与 MySQL 的 `'user'@'host'` 不同，PostgreSQL 的 role 不包含来源主机；来源地址、数据库、用户和认证方式由 `pg_hba.conf` 或云数据库的访问控制管理。

一种常见划分是让对象归属和应用访问分开：一个不能登录的 owner 持有所有表，应用账号只做增删改查。

```sql
CREATE ROLE shop_owner NOLOGIN;
CREATE ROLE shop_app LOGIN PASSWORD 'replace-me';

CREATE SCHEMA app AUTHORIZATION shop_owner;

GRANT USAGE ON SCHEMA app TO shop_app;
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app TO shop_app;
```

这里有个容易踩的坑：`ON ALL TABLES` 只对执行这条语句时已经存在的表生效，之后新建的表不会自动继承权限。要让后续对象也带上权限，得用 `ALTER DEFAULT PRIVILEGES` 提前声明，并且它是按“由谁创建”来匹配的：

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE shop_owner IN SCHEMA app
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO shop_app;
```

也就是说，后续建表都应该以 `shop_owner` 的身份执行，这条声明才会生效。

---

## 4. 数据类型：不是简单的一一替换

大部分 MySQL 类型都能在 PostgreSQL 里找到对应。可以照搬的这些不必多想：

| MySQL | PostgreSQL |
|---|---|
| `TINYINT` / `SMALLINT` | `smallint` |
| `INT` / `BIGINT` | `integer` / `bigint` |
| `FLOAT` / `DOUBLE` | `real` / `double precision` |
| `TEXT` / `LONGTEXT` | `text` |
| `DATE` / `TIME` | `date` / `time` |
| `BLOB` | `bytea` |

剩下的换过去之前得先看一眼：

| MySQL | PostgreSQL | 差别在哪 |
|---|---|---|
| `BIGINT AUTO_INCREMENT` | `bigint GENERATED BY DEFAULT AS IDENTITY` | 背后是独立的 sequence 对象 |
| `INT UNSIGNED` | `bigint` 加 `CHECK` 限范围 | 没有无符号整数，换成 `integer` 可用范围少一半；`BIGINT UNSIGNED` 只能退到 `numeric(20, 0)` |
| `TINYINT(1)` / `BOOL` | `boolean` | 只接受 `true`、`false` 和 `NULL`，整数不会隐式转换，`VALUES (1)` 直接报类型错误 |
| `DECIMAL(m, n)` | `numeric(m, n)` | 两边都别省略精度：MySQL 裸写 `DECIMAL` 是 `DECIMAL(10, 0)`，会把小数截掉；PostgreSQL 裸写 `numeric` 则是任意精度 |
| `VARCHAR(n)` | `varchar(n)` 或 `text` | 两者性能相同，`varchar(n)` 的价值是表达业务约束，而不是“更短更快” |
| `DATETIME` / `TIMESTAMP` | `timestamp` / `timestamptz` | 分界线是“墙上时间”还是“绝对时间点” |
| `JSON` | `jsonb` | `json` 只存文本，`jsonb` 才能比较、检索和建 GIN 索引 |
| `ENUM(...)` | `text` 加 `CHECK`，或原生 enum | 原生 enum 增删取值要改类型定义，不适合多变的业务状态 |
| 用 `CHAR(36)` 存 UUID | 原生 `uuid` | 当主键时优先时间有序的 UUIDv7，不要用完全随机的 v4 |
| 无对应 | `array`、`range`、`inet`、`tsvector` 等 | 好用，但不该拿来替代正常的表设计 |

其中三处值得展开。

### 4.1 identity 只保证唯一，不保证连续

`BY DEFAULT` 允许显式写入自己指定的 ID，行为更接近 `AUTO_INCREMENT`；`GENERATED ALWAYS` 更严格，显式写值需要额外写 `OVERRIDING SYSTEM VALUE`，批量导入数据时这个区别会很明显。旧代码中常见的 `bigserial` 仍然可用，但它是“列 + sequence + default”的历史简写，新表优先使用符合 SQL 标准的 identity。

更需要调整的是对 ID 本身的预期。identity 的底层是一个独立的 sequence 对象，失败插入、事务回滚、`ON CONFLICT`、缓存和崩溃都可能消耗序列值：

> 自增 ID 用来保证唯一，不用来保证连续，也不能精确表示事务提交顺序。

### 4.2 `timestamp` 与 `timestamptz` 的分界线

`timestamp`（全称 `timestamp without time zone`）保存“墙上时间”，不代表全球时间线上的唯一时刻；`timestamptz`（全称 `timestamp with time zone`）保存一个绝对时间点，输入和输出会根据会话时区转换。要注意它并不保存原始时区名称：写入 `2026-07-24 09:00 Asia/Shanghai` 后，数据库留下的是对应的那个时间点，换一个 `TimeZone` 读取，显示值就会变。

按经验，MySQL 的 `TIMESTAMP` 更接近 `timestamptz`，`DATETIME` 更接近 `timestamp`，但最终仍要逐列按业务语义判断。订单创建时间、支付时间应该用 `timestamptz`；生日、每天 09:00 开门、未来某地的日程则可能需要 `date`、`time`、`timestamp`，甚至额外保存 `Asia/Shanghai` 这样的 IANA 时区名——这类按当地语义建模的字段，记得测试 DST 重复和不存在的当地时间。

选对类型只是第一步，绝对时间点还要在整条链路上保持一致：数据库连接统一设置 `TimeZone=UTC`，API 使用带 offset 的 ISO 8601，表中使用 `timestamptz`，展示层再转换到用户时区。

另外，MySQL 的 `ON UPDATE CURRENT_TIMESTAMP` 在 PostgreSQL 里没有对应写法。想让 `updated_at` 自动跟着变，要么写一个触发器，要么在每条 `UPDATE` 里显式赋值。

### 4.3 `jsonb` 与几个 MySQL 没有的类型

`json` 原样保存输入文本，保留空白和键顺序；`jsonb` 保存解析后的二进制结构。MySQL 的 `JSON` 过来通常应该选 `jsonb`，因为只有它支持包含判断和 GIN 索引：

```sql
SELECT *
FROM app.orders
WHERE attributes @> '{"channel": "ios"}'::jsonb;

CREATE INDEX orders_attributes_gin_idx
ON app.orders USING gin (attributes);
```

但 JSONB 不是“免设计表结构”的理由。稳定、高频过滤、需要约束和关联的数据仍应建成普通列。

表格末尾那几个 MySQL 没有对应的类型也是同样的分寸：值得用，但不该拿来绕开建模。UUID 当主键时之所以推荐 UUIDv7，是因为完全随机的 v4 会让插入散落到索引各处，这个代价 MySQL 开发者大多不陌生（较新版本已内置 `uuidv7()`，旧版本可以在应用侧生成）；array 适合标签、权限这类边界清晰的小集合，不应替代正常的一对多关系；range 表示一个区间，配合排斥约束可以表达“不允许重叠”，下一节会具体展开。

---

## 5. 约束：让数据库自己保证正确性

`PRIMARY KEY`、`UNIQUE`、`FOREIGN KEY`、`CHECK`、`NOT NULL` 两边都有，写法也基本一致，差别集中在下面几处：

| 对比项 | MySQL | PostgreSQL |
|---|---|---|
| `UNIQUE` 遇上 `NULL` | 允许多个 `NULL`，无法改变 | 默认相同，但可以声明 `NULLS NOT DISTINCT`，让 `NULL` 之间也算重复 |
| 只对部分行唯一 | 没有对应写法 | 条件唯一索引：`CREATE UNIQUE INDEX ... WHERE ...` |
| 外键列的索引 | InnoDB 自动创建 | 不会自动创建，要自己加 |
| 排斥约束 | 无法声明 | `EXCLUDE`，表达“任意两行不能同时满足某组条件” |

一个两边一致、但容易忘的细节：`CHECK` 表达式结果为 `TRUE` 或 `NULL` 都算通过，所以 `CHECK (amount >= 0)` 并不阻止写入 `NULL`，不允许为空还得另外声明 `NOT NULL`。

### 5.1 唯一性可以带条件

想让 `NULL` 之间也互相冲突，把约束写成 `UNIQUE NULLS NOT DISTINCT (provider, external_id)` 即可。

更常用的是另一种：只对满足条件的行唯一。软删除场景下，“同一个邮箱只能有一个未删除的账号”用普通 UNIQUE 表达不出来，条件唯一索引可以：

```sql
CREATE UNIQUE INDEX users_active_email_uq
ON app.users (lower(email))
WHERE deleted_at IS NULL;
```

只有 `deleted_at IS NULL` 的行进入索引，被软删除的记录自动退出唯一性判断，同一个邮箱可以重新注册。索引建在 `lower(email)` 上则顺带让唯一性对大小写不敏感。

### 5.2 `EXCLUDE`：MySQL 里无法声明的约束

还有一类约束 MySQL 根本没有对应写法：不允许同一房间的预订时间段重叠。

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE app.reservations (
    id          bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    room_id     bigint NOT NULL,
    during      tstzrange NOT NULL,
    EXCLUDE USING gist (
        room_id WITH =,
        during WITH &&
    )
);
```

这里声明的是“任意两行都不能同时满足房间号相同且时间区间有交集”。与“先查有没有重叠，再插入”相比，它能在并发下真正保证正确性——为什么查询挡不住，第 7 节会具体解释。

### 5.3 外键列不会自动建索引

InnoDB 建外键时会顺手在子表引用列上建索引，PostgreSQL 不会。这件事很容易被忽略，因为不建索引外键照样能用，只是父表执行 `DELETE`、`UPDATE` 时需要扫描子表查找引用行，父表记录更新频繁的话就会积累出扫描开销和锁等待。

实践中不必为它单独建一个索引，让外键列成为某个复合索引的第一列就够了——比如订单表上以 `user_id` 开头的索引，既承担引用检查，也服务于“查某个用户的订单”。

反过来，主键和 UNIQUE 已经自带索引，不要再重复建一份相同的。

---

## 6. 写 SQL：CRUD 与方言差异

下面的例子用 `$1`、`$2` 表示参数占位符，这是 PostgreSQL 服务端协议的写法。应用里实际写成什么由驱动或 ORM 决定，有些 Python 驱动就用 `%s`。

### 6.1 用 `RETURNING` 取回写入结果

MySQL 常在插入后调用 `LAST_INSERT_ID()`。PostgreSQL 可以直接返回 identity、默认值和触发器处理后的列：

```sql
INSERT INTO app.users (email, display_name)
VALUES ($1, $2)
RETURNING id, email, created_at;
```

`RETURNING` 同样适用于更新和删除：

```sql
UPDATE app.orders
SET status = 'paid',
    updated_at = now()
WHERE id = $1
  AND status = 'pending'
RETURNING id, status, updated_at;
```

应用可根据是否返回行判断状态流转是否成功，不必先查再改。

### 6.2 `ON CONFLICT` 比“忽略所有错误”更明确

对应 MySQL 的 `ON DUPLICATE KEY UPDATE` 的是 `INSERT ... ON CONFLICT`，但它要求写明冲突目标，且这个目标必须与某个唯一约束或唯一索引匹配。前面 `app.users` 上那个带 `lower(email)` 和条件的部分唯一索引，就要把表达式和谓词都写出来：

```sql
INSERT INTO app.users (email, display_name)
VALUES ($1, $2)
ON CONFLICT ((lower(email))) WHERE deleted_at IS NULL
DO UPDATE SET display_name = EXCLUDED.display_name
RETURNING id, email, display_name;
```

`ON CONFLICT DO NOTHING` 只忽略相应的唯一性或排斥冲突。它不等同于 MySQL `INSERT IGNORE`，因为后者还可能把某些非法值、截断和约束错误降级为警告。

同样，`ON CONFLICT DO UPDATE` 也不等同于 `REPLACE INTO`。MySQL `REPLACE` 通常是删除旧行再插入，可能触发删除、外键级联和新的默认值；PostgreSQL 的 upsert 是更新冲突行。改写这类 SQL 前，先确认业务真正需要的是“更新”还是“删除再插入”。

### 6.3 `UPDATE ... FROM` 和 `DELETE ... USING`

MySQL 的 `UPDATE ... JOIN` 在 PostgreSQL 里写成 `UPDATE ... FROM`，关联条件放进 `WHERE`：

```sql
UPDATE app.orders AS o
SET status = 'canceled',
    updated_at = now()
FROM app.users AS u
WHERE u.id = o.user_id
  AND u.deleted_at IS NOT NULL;
```

联表删除同理，把 `FROM` 换成 `USING`。

有两点和 MySQL 不同。一是要确保一条目标记录最多匹配一条来源记录，若 `FROM` 产生多条匹配，PostgreSQL 会任取其一，结果不应被依赖。二是这条语句只修改一个目标表，并不等于 MySQL 的多表更新或多表删除。

### 6.4 常用函数和表达式差异

日常 SQL 中还有一些高频改写：

| MySQL | PostgreSQL |
|---|---|
| `IFNULL(value, default)` | `COALESCE(value, default)` |
| `GROUP_CONCAT(name ORDER BY name SEPARATOR ',')` | `string_agg(name, ',' ORDER BY name)` |
| `DATE_FORMAT(created_at, '%Y-%m-%d')` | `to_char(created_at, 'YYYY-MM-DD')` |
| `LIMIT offset, count` | `LIMIT count OFFSET offset` |
| `CAST(value AS type)` | 同样使用 `CAST`，也可写成 `value::type` |

最容易被忽略的是字符串比较。MySQL 的默认排序规则不区分大小写（`utf8mb4_0900_ai_ci` 结尾的 `ci` 就是 case-insensitive），所以 `WHERE email = 'Foo@example.com'` 能匹配到库里存的 `foo@example.com`。PostgreSQL 没有这个默认，`text` 比较严格区分大小写，同一条 SQL 什么都查不到：

```sql
-- 区分大小写，大小写不一致就查不到
SELECT id FROM app.users WHERE email = $1;

-- 不区分大小写，同时能命中 lower(email) 索引
SELECT id FROM app.users WHERE lower(email) = lower($1);
```

这也是前面 `app.users` 上那个唯一索引建在 `lower(email)` 而不是 `email` 上的原因。除了两侧都套 `lower()`，另一种做法是把列换成 `citext` 类型，让比较本身不区分大小写，代价是多依赖一个扩展。

两边都有 `concat()`，但遇到 `NULL` 时语义不同：MySQL 的 `concat('a', NULL)` 返回 `NULL`，PostgreSQL 则忽略 `NULL` 并返回 `'a'`。需要 NULL 传播行为的话，改用 `||` 运算符，它遇到 `NULL` 会让整个表达式变成 `NULL`。

整数除法尤其容易静默改变结果。MySQL 中 `SELECT 1 / 2` 得到 `0.5000`，PostgreSQL 中两个 integer 相除仍是 integer，结果为 `0`，需要小数得先转换类型：`1::numeric / 2`。

两边对 `NULL` 的默认排序也相反：MySQL 升序时通常把 `NULL` 放在前面，PostgreSQL 升序时默认放在后面。需要稳定语义时显式写 `ORDER BY deleted_at ASC NULLS LAST`。

### 6.5 批量更新不能直接加 `LIMIT`

PostgreSQL 没有 MySQL 风格的 `UPDATE ... LIMIT` 和 `DELETE ... LIMIT`。可靠做法是先按稳定顺序选主键，再更新：

```sql
WITH batch AS (
    SELECT id
    FROM app.jobs
    WHERE status = 'ready'
      AND run_at <= now()
    ORDER BY run_at, id
    FOR UPDATE SKIP LOCKED
    LIMIT 100
)
UPDATE app.jobs AS j
SET status = 'running',
    worker_id = $1,
    started_at = now()
FROM batch
WHERE j.id = batch.id
RETURNING j.id, j.payload;
```

这也是一个并发任务队列的最小实现：多个工作进程同时领取任务，`SKIP LOCKED` 跳过已被其他进程锁住的行。

### 6.6 用 CTE、窗口函数和 LATERAL 减少应用层代码

这几样 MySQL 8 也支持，但从 MySQL 过来的代码往往没把它们用起来，习惯把数据拉回应用再算。

查询每个用户最近三笔订单：

```sql
WITH ranked_orders AS (
    SELECT o.*,
           row_number() OVER (
               PARTITION BY user_id
               ORDER BY created_at DESC, id DESC
           ) AS rn
    FROM app.orders AS o
)
SELECT *
FROM ranked_orders
WHERE rn <= 3;
```

这类 Top-N、累计值、排名和相邻记录计算不需要在应用中把全部数据拉回内存。

`LATERAL` 则允许右侧子查询引用左侧行，适合“每组取一条”：

```sql
SELECT u.id,
       u.email,
       latest.id AS latest_order_id,
       latest.amount
FROM app.users AS u
LEFT JOIN LATERAL (
    SELECT o.id, o.amount
    FROM app.orders AS o
    WHERE o.user_id = u.id
    ORDER BY o.created_at DESC, o.id DESC
    LIMIT 1
) AS latest ON true;
```

配合 `(user_id, created_at DESC, id DESC)` 索引，它可以高效完成这类查询，避免在应用里对每个用户循环发一次 SQL。

---

## 7. 事务与并发：最容易“代码能跑、语义却变了”的地方

### 7.1 默认隔离级别不同

这是两者之间最容易被忽略的差异之一：

- MySQL InnoDB 默认通常是 `REPEATABLE READ`；
- PostgreSQL 默认是 `READ COMMITTED`。

两者都可以改配置，所以实际值应该查一下，不要凭印象。

在 PostgreSQL Read Committed 中，每条语句开始时获得一个新快照。也就是说，同一个事务里连续查两次同一行，如果中间有别的事务提交了更新，两次结果可以不同——这在 InnoDB 的 Repeatable Read 下不会发生。

如果改用 Repeatable Read，也不能直接套用 InnoDB 的经验。两边都会在事务里固定快照，但对写冲突的处理完全不同：InnoDB 的 `UPDATE`、`DELETE` 会基于较新的已提交版本继续执行，PostgreSQL 则在目标行已被其他事务改过并提交时，直接以 SQLSTATE `40001` 失败。Serializable 的 SSI 检测也会返回同一个错误。

所以在 PostgreSQL 用 Repeatable Read 或 Serializable，前提是应用能捕获 `40001`，回滚并从 `BEGIN` 重试整个事务。另外它固定快照的方式本质上是 Snapshot Isolation，仍可能出现 write skew，不要把 Repeatable Read 当成 Serializable 用。

### 7.2 不要照搬 InnoDB gap lock

InnoDB 在 Repeatable Read 的范围锁定读、更新和删除中，常通过 next-key lock 锁住记录及间隙，阻止范围内插入。

PostgreSQL 没有等价的常规 gap lock。`SELECT ... FOR UPDATE` 只锁实际返回的行，查询没有返回结果，并不意味着“未来满足条件的插入”也被锁住。

这正是第 5 节强调约束的原因：在 PostgreSQL 里，约束不是应用校验之外的补充，而是并发正确性的主要手段。如果业务依赖“先查不存在，再插入”，应当改用：

- UNIQUE / EXCLUDE 等数据库约束；
- `INSERT ... ON CONFLICT`；
- Serializable 隔离级别并正确重试；
- 必要时使用 advisory lock，但要自行设计锁键和生命周期。

不要用一次查询结果替代约束。

### 7.3 `NOWAIT` 与 `SKIP LOCKED`

行锁除了熟悉的 `FOR UPDATE` 和 `FOR SHARE`，还可以加两个修饰：`NOWAIT` 在无法立即取得锁时直接报错，而不是排队等待；`SKIP LOCKED` 跳过已被锁住的行。后者返回的结果有意不一致，适合任务队列这种“谁抢到算谁的”场景，不适合普通报表或账户余额查询。

### 7.4 sequence 不回滚

sequence 的取值不受事务控制。事务里插入一行再 `ROLLBACK`，这次分配掉的 ID 通常不会被下次插入重用；`setval()` 对 sequence 状态的修改同样不会随事务回滚而撤销。

如果业务要求“发票号绝对连续”，那是一个需要串行化、审计和补偿的业务编号问题，不能直接拿 identity 主键解决。

### 7.5 DDL 可以放进事务，但仍然会加锁

PostgreSQL 可以这样执行：

```sql
BEGIN;
ALTER TABLE app.orders ADD COLUMN note text;
UPDATE app.orders SET note = '';
COMMIT;
```

失败时可以整体回滚。MySQL 8.x 的 atomic DDL 主要保证单条 DDL 自身原子完成，并不等同于可以与多条 DML 一起事务回滚。

但可事务化不等于不阻塞，也不是所有 DDL 都能进事务。大表上最常用的 `CREATE INDEX CONCURRENTLY` 就是例外：它不会长时间阻塞正常写入，代价是更慢、消耗更多 CPU 和 I/O，而且不能放在事务块中（`VACUUM`、`CREATE DATABASE` 也一样），变更脚本要把它当作独立步骤。它失败后还会留下一个无效索引，需要先 `DROP` 再重建。

### 7.6 超时：给锁和语句设上限

PostgreSQL 有几个容易混淆的超时参数：

- `lock_timeout`：单次等待锁的最长时间；
- `statement_timeout`：整条语句的最长执行时间，包含锁等待；
- `transaction_timeout`：一个事务从开始到结束的最长时间，超过就中止；
- `idle_in_transaction_session_timeout`：事务打开后长期不再执行语句的会话上限；
- `deadlock_timeout`：等待多久后开始做死锁检测，不是锁等待上限。

实践中通常让 `lock_timeout` 小于 `statement_timeout`，两者又都小于应用侧的请求截止时间。发生死锁时 PostgreSQL 会中止其中一个事务，应用应尽量统一加锁顺序。

### 7.7 出错后重试整个事务

PostgreSQL 显式事务中的一条语句报错后，事务通常进入 aborted 状态。后续语句会收到 `25P02`，必须执行 `ROLLBACK`，或者回滚到事先建立的 savepoint。这也决定了重试的粒度：能重发的只有整个事务，不是出错的那一条语句。

要判断一个错误该不该重试，靠的是 PostgreSQL 提供的五位 SQLSTATE，而不是错误文本：

| SQLSTATE | 含义 | 通常如何处理 |
|---|---|---|
| `23505` | unique violation | 返回冲突，或按明确业务语义处理 |
| `23503` | foreign key violation | 返回数据关系错误 |
| `40001` | serialization failure | 从 `BEGIN` 重试整个事务 |
| `40P01` | deadlock detected | 有上限、带退避地重试整个事务 |
| `55P03` | lock not available | 视竞争场景限次重试 |
| `57014` | query canceled | 先判断是超时、用户取消还是管理操作 |

不要匹配“duplicate key value”之类错误文本，它可能随语言、版本和上下文改变。

重试本身还有两条纪律：使用最大次数、指数退避和随机抖动；数据库外的 HTTP、消息、文件等副作用必须幂等，或采用事务发件箱（transactional outbox）。

如果网络在 `COMMIT` 附近断开，客户端可能无法判断事务是否已经提交。此时盲目重放写操作并不安全，需要业务幂等键或结果查询能力。

---

## 8. Vacuum：MySQL 开发者必须补上的一课

### 8.1 快照的代价

上一节反复出现的“快照”，落到存储上就是多版本。两边保存旧版本的方式不同：InnoDB 把历史版本放进 undo 日志，由 purge 清理；PostgreSQL 则把新版本直接写在表里——`UPDATE` 在 heap 中追加一个新 tuple，旧 tuple 在不再对任何事务可见后变成 dead tuple，`DELETE` 同样留下待回收的版本。

差别的后果是：PostgreSQL 的垃圾堆在表自己身上，必须有人来收。这个人就是 Vacuum，它的职责包括：

- 回收 dead tuple 占用的空间，供同一张表后续复用；
- 维护 visibility map，让部分查询可以只读索引、不回表；
- 清理索引中的无效引用；
- 冻结旧事务 ID，避免 transaction ID wraparound。

`ANALYZE` 则收集统计信息，供查询规划器估算数据分布。Autovacuum 会按阈值自动执行 Vacuum 和 Analyze，它不是可有可无的后台优化，而是 PostgreSQL 正常工作的组成部分。

### 8.2 普通 Vacuum 通常不会缩小文件

普通的 `VACUUM` 主要把空间变成表内可复用空间，通常不会把文件立即归还操作系统。所以删掉大量数据后，磁盘占用不降反而可能继续涨——这是从 MySQL 过来最容易困惑的现象之一。

`VACUUM FULL` 会重写整张表并缩小文件，但需要额外磁盘空间和 `ACCESS EXCLUSIVE` 锁，不能当作日常定时任务。出现膨胀时，应先找出为什么回收跟不上，而不是固定周期执行 Full。

### 8.3 长事务会阻止旧版本回收

只要某个旧快照仍可能看到历史 tuple，Vacuum 就不能清理它。最常见的两个原因，是长时间运行的事务，以及 `BEGIN` 之后迟迟不提交、挂在 `idle in transaction` 状态的会话。这类会话在 MySQL 下会撑大 undo，在 PostgreSQL 下则直接让整个库的垃圾回收停滞。

所以应用侧最重要的规则是：

> 事务应尽可能短，不要在事务中等待用户输入、远程 HTTP、消息队列或长时间业务计算。

代码之外还该有一道兜底：上一节的 `transaction_timeout` 和 `idle_in_transaction_session_timeout` 正是为这两种情况准备的，它们能保证个别失控的事务不会一直拖着整个库的回收。

想知道有没有这种事务，查 `pg_stat_activity`，按 `xact_start` 排序就能看到最老的事务开了多久，`state` 列会标出 `idle in transaction`。想知道回收有没有跟上，查 `pg_stat_user_tables` 的 `n_dead_tup` 和 `last_autovacuum`，注意前者只是估计值，不等于精确的膨胀大小。

---

## 9. 索引与查询优化：从 B-Tree 走向 PostgreSQL 的工具箱

### 9.1 B-Tree 经验仍然有效

B-Tree 仍然是默认索引，适合等值、范围、排序和 `IS NULL`。复合索引也要关注左侧列。以“翻某个用户的订单列表”为例：

```sql
CREATE INDEX orders_user_created_idx
ON app.orders (user_id, created_at DESC, id DESC)
INCLUDE (status, amount);

SELECT id, created_at, status, amount
FROM app.orders
WHERE user_id = $1
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

InnoDB 的二级索引记录会自动包含主键，PostgreSQL 普通二级索引则没有这一保证。这里的 `id` 参与排序、决定翻页位置，所以必须作为索引键；只用于返回的 `status`、`amount` 则通过 `INCLUDE` 附带。该索引不适合只按 `created_at` 查询全站订单，不要因为索引包含某列，就认为任何涉及该列的查询都能高效使用它。

### 9.2 三种很实用的 B-Tree 变体

表达式索引可以让 `WHERE lower(display_name) = $1` 这类查询走索引，而不必额外维护一个冗余列：

```sql
CREATE INDEX users_display_name_lower_idx
ON app.users (lower(display_name));
```

部分索引只索引满足条件的行，适合数据分布极不均匀、而查询只关心少数状态的场景：

```sql
CREATE INDEX orders_pending_created_idx
ON app.orders (created_at, id)
WHERE status = 'pending';
```

要注意查询条件必须能被规划器证明包含索引谓词。参数化成 `status = $1` 时，规划器未必能确认 `$1` 一定是 `'pending'`，因此可能用不上这个部分索引。

覆盖索引则由 `INCLUDE` 表达，上一节那个 `orders_user_created_idx` 就是一例。`INCLUDE` 列不参与搜索、排序和唯一性，只是让 Index Only Scan 有机会直接取得返回值。能不能真的不回表，还取决于上一节说的 visibility map，所以“加了 INCLUDE 就一定不回表”并不成立。

### 9.3 GIN、GiST、BRIN 各自解决不同问题

| 类型 | 适合场景 | 特点 |
|---|---|---|
| GIN | JSONB、数组、全文检索 | 倒排索引，查询强，写入和构建成本较高 |
| GiST | range、几何、近邻搜索 | 可扩展框架，能力取决于 operator class |
| BRIN | 超大时序表、与物理顺序高度相关的列 | 很小但有损，需要 recheck |

前面出现过的两个例子正好各属一类：`jsonb` 上的 GIN 索引，以及排斥约束背后的 GiST。

BRIN 不是“更省空间的通用 B-Tree”。只有当表足够大，而且索引列与数据的物理写入顺序高度相关（典型例子是一直追加的时间列）时，它才能用很小的索引快速排除大量数据块。

### 9.4 用真实执行计划说话

单独的 `EXPLAIN` 只给出规划器的估算，真正有用的是加上 `ANALYZE` 和 `BUFFERS` 实际跑一遍：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM app.orders
WHERE user_id = 1
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

阅读时抓三件事：耗时主要花在哪个节点、估算行数与实际行数是否严重偏离、以及数据是从缓存命中还是从磁盘读的。估算严重偏离通常意味着统计信息过期或数据分布超出了规划器的假设，这也是执行计划变差最常见的源头。

另外要注意，`cost` 是规划器的成本单位而不是毫秒；`EXPLAIN ANALYZE` 会真实执行语句，对写操作尤其要小心。

PostgreSQL 没有内置的通用 query hint，也就没有 MySQL 那样的 `FORCE INDEX` 可以强制走某个索引。遇到“为什么没走索引”，先确认查询条件和索引是否真的匹配——隐式类型转换、包在列上的函数、不一致的排序规则都会让索引失效；其次才是考虑表规模和选择率是否值得走索引。

---

## 10. 扩展：PostgreSQL 的另一半能力

前面用到的 `btree_gist` 就是一个扩展。这是 PostgreSQL 与 MySQL 差别最大的地方之一：很多在 MySQL 里要引入外部组件才能做的事，在 PostgreSQL 里是一条 `CREATE EXTENSION`。

扩展也不只是“内置函数库”。它能往数据库里加类型、运算符、索引访问方法、后台进程，甚至挂进查询规划器——上一节说 PostgreSQL 没有内置的 query hint，而 `pg_hint_plan` 这个扩展就把它补上了。能改的东西和内核能改的东西差不太多，所以 PostgreSQL 的能力边界比它自带的那份功能列表宽得多。

### 10.1 扩展是 database 级别的

装扩展的语句本身很简单，一句 `CREATE EXTENSION IF NOT EXISTS pg_trgm` 就够了，但有两点容易踩。

第一，扩展装在**某一个 database** 里，不是整个实例。同一实例的另一个 database 要用，得再装一次。这和第 1 节的层级是一致的：扩展带来的类型和函数都属于某个 schema，自然也就属于某个 database。

第二，装扩展通常需要管理员权限。一部分被标记为 trusted 的扩展（`pg_trgm`、`btree_gist`、`citext`、`pgcrypto` 等）允许有建对象权限的普通用户安装，但更多扩展仍然要超级用户出手。因此扩展属于初始化脚本的内容，不该让应用在运行时按需创建。

看当前装了哪些用 `\dx`，看这个实例还能装什么查 `pg_available_extensions`。

### 10.2 值得知道的几类

| 扩展 | 解决什么 | 在 MySQL 里通常怎么做 |
|---|---|---|
| `pg_stat_statements` | 按 SQL 指纹聚合调用次数和耗时 | 慢查询日志加 `performance_schema` |
| `pg_trgm` | 让 `LIKE '%keyword%'` 和相似度匹配走索引 | 全表扫，或上 ngram 全文索引 |
| `citext` | 大小写不敏感的文本类型 | 默认排序规则本来就不区分大小写 |
| `pgcrypto` | 哈希与加解密函数 | 应用层实现，或 `AES_ENCRYPT` |
| `postgres_fdw` | 把别的库的表当本地表查询 | `FEDERATED` 引擎，基本没人用 |
| `pg_cron` | 数据库内的定时任务 | `EVENT` 调度器 |
| `pgvector` | 向量类型与近邻检索 | 外挂一套向量数据库 |
| `PostGIS` | 完整的地理空间能力 | 内置 spatial 类型，能力差距很大 |
| `TimescaleDB` | 时序场景的分区、压缩与预聚合 | 手工分区加归档 |

其中 `pg_stat_statements` 值得第一个装（它需要预先加载，通常由管理员配置，托管数据库多半已默认开启）。它按语句指纹聚合调用次数、总耗时和平均耗时，是上一节那套执行计划分析的入口：先用它找出哪些语句值得看，再对具体语句跑 `EXPLAIN (ANALYZE, BUFFERS)`。只盯着单条慢查询，很容易漏掉那些单次很快但被调用几十万次的语句。

`pg_trgm` 解决的是一个 MySQL 开发者很熟的痛点。`LIKE '%keyword%'` 没有前缀可用，B-Tree 索引指望不上，两边默认都只能全表扫。`pg_trgm` 把字符串切成三字符片段建 GIN 索引，让这类模糊匹配和 `similarity()` 相似度排序都能用上索引。

`pgvector` 则说明了另一件事：PostgreSQL 常常靠扩展接住一整类新需求。它加了 `vector` 类型和近邻索引，让 embedding 检索留在业务库里，省掉再同步一套外部向量库。

### 10.3 代价：它成了运行环境的一部分

装扩展不像加一个依赖包那么轻。

托管数据库通常只开放一份扩展白名单，各家云厂商还不一样。所以在把核心功能压在某个扩展上之前，先确认目标环境装得上，PostGIS、TimescaleDB、pgvector 这类改动较深的尤其要先问清楚。

扩展也有自己的版本，跟随数据库大版本升级时可能需要执行 `ALTER EXTENSION ... UPDATE`，个别扩展还会卡住升级路径。选型时值得看一眼它的维护活跃度。

最后一条判断标准：内置能力够用时，不要为此引入扩展。生成 UUID 用内置的 `gen_random_uuid()` 就行，不必再装 `uuid-ossp`；键值存储用 `jsonb` 就够，`hstore` 现在基本只在维护老代码时才会碰到。

---

## 11. 结尾：把 MySQL 经验变成坐标，而不是枷锁

从 MySQL 转向 PostgreSQL，不需要把已有经验全部推倒重来。

可以继续沿用的，是关系模型、事务意识、索引基础、约束思维和用执行计划定位性能问题的方法论。需要重新建立的，是下面这些默认认知：

| MySQL 经验 | PostgreSQL 中应建立的新认知 |
|---|---|
| database 就是 schema | database 与 schema 是不同层级 |
| 一条连接是一个线程 | 一条连接是一个进程 |
| `AUTO_INCREMENT` 是列的属性 | identity 背后是独立的 sequence 对象 |
| `VARCHAR(255)` 是稳妥默认值 | 没有业务上限时使用 `text` |
| `INSERT IGNORE` / `REPLACE` | `ON CONFLICT` 语义更窄且更明确 |
| 默认 InnoDB RR | PostgreSQL 默认 RC，应明确并发语义 |
| 范围锁可以阻止插入 | PostgreSQL 没有常规 gap lock，依靠约束、upsert 或 Serializable |
| 旧版本放在 undo 里，由 purge 清理 | 旧版本留在表里，靠 Vacuum 回收 |
| 索引基本只用 B-Tree | 根据操作符和数据分布选择 B-Tree、GIN、GiST、BRIN |
| 数据库不够用就外挂组件 | 模糊匹配、地理、向量等能力多半有对应扩展 |

这些差异单独看都不大，麻烦在于它们大多不会报错：SQL 照样执行，代码照样上线，问题要等到并发上来、数据变大或者某个长事务挂住之后才浮出来。所以从 MySQL 转过来，真正要做的不是背下语法映射，而是在每一个“我以为它本来就该这样”的地方停一下，确认那个来自 MySQL 的假设在 PostgreSQL 里是否仍然成立。

---

## 参考资料

- [PostgreSQL Documentation](https://www.postgresql.org/docs/current/)
- [MySQL 8.4 Reference Manual: InnoDB Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
