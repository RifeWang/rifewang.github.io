# From MySQL to PostgreSQL: A Quick-Start Guide for Developers

## A practical comparison of PostgreSQL architecture, connections, permissions, types, SQL, transactions, Vacuum, and indexes for developers familiar with MySQL, designed to help you quickly build the right mental model

## Preface: They Differ in More Than Just Syntax

If you are already familiar with MySQL, most of PostgreSQL will look familiar at first: both are relational databases, both support transactions, indexes, constraints, and SQL, and both are well supported by mainstream programming languages and ORMs.

The places most likely to cause problems are precisely those that “look almost the same”:

- Should a MySQL database map to a PostgreSQL database or schema?
- What should replace `AUTO_INCREMENT`, `DATETIME`, `TINYINT(1)`, and `JSON`?
- How should `INSERT IGNORE`, `REPLACE INTO`, and `ON DUPLICATE KEY UPDATE` be rewritten?
- Both systems have MVCC and Repeatable Read, but do they provide the same concurrency semantics?
- Why do long-running transactions, Vacuum, and connection counts require special attention in PostgreSQL?

This article does not debate whether MySQL or PostgreSQL is “better,” nor does it enumerate every feature. Instead, it first clarifies PostgreSQL’s overall architecture, connection model, and permission system, then walks through data types and constraints, SQL dialect differences, transactional concurrency, Vacuum, and indexes.

There is only one central idea to remember:

> Your MySQL experience can serve as a frame of reference for understanding PostgreSQL, but it must not make decisions for PostgreSQL by default.

---

## 1. Overall Architecture: The Additional Schema Layer

### 1.1 From Three Levels to Four

The first step in understanding PostgreSQL is to understand the object hierarchy in each system. MySQL typically has three levels—instance, database, and table. PostgreSQL adds a schema layer between database and table, creating four levels:

| MySQL | PostgreSQL | Description |
|---|---|---|
| instance | cluster / instance | Manages a collection of databases |
| database | database or schema | The appropriate mapping depends on transaction, permission, and isolation requirements |
| no separate equivalent | schema | A namespace within a database |
| table | table | Every table must belong to a schema |

Objects such as indexes, sequences, types, and functions also belong to a schema. A PostgreSQL database can contain multiple schemas, and different schemas can contain tables with the same name, such as `account.users` and `audit.users`.

This additional layer directly changes how you divide a system into business domains, because a PostgreSQL connection can access only one database and cannot perform a cross-database JOIN. If two modules need to be queried and modified within the same transaction, place them in different schemas of the same database. You can certainly split them into multiple databases, but cross-database access then requires `postgres_fdw`, `dblink`, or multiple application-level connections.

A more practical way to decide between databases and schemas in PostgreSQL is:

- If you need cross-module transactions and JOINs, prefer multiple schemas in the same database.
- If you need strong isolation or independent operations, consider separate databases.
- Do not create many PostgreSQL databases merely because you are accustomed to separating business domains with databases in MySQL.

### 1.2 `public` Is Only the Default Schema

After creating a database, you will usually see a schema named `public`. It is not a database; it is merely the default namespace.

When you write `SELECT * FROM users` without specifying a schema, PostgreSQL searches the entries in `search_path` in order:

```sql
SHOW search_path;
-- "$user", public
```

This is convenient, but it also means that the table accessed by a SQL statement depends on session state. Production code is safer when it explicitly uses names such as `app.users` instead of relying on the search path.

### 1.3 Taking a Look: Comparing `psql` with the MySQL CLI

The most intuitive way to understand these levels is to connect and inspect them. Docker can quickly provide a development instance:

```bash
docker run --name pg-dev \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=shop \
  -p 5432:5432 \
  -d postgres:18
```

The connection string resembles its MySQL counterpart, and `psql` accepts it directly:

```bash
psql "postgresql://postgres:postgres@localhost:5432/shop"
```

`psql` is probably the first unfamiliar part of moving from MySQL. In the MySQL client, `SHOW DATABASES`, `USE shop`, `SHOW TABLES`, and `DESC users` are all SQL statements that run with a trailing semicolon. In `psql`, their counterparts are client commands beginning with a backslash, so your old muscle memory produces a syntax error.

Here is a comparison of common commands:

| Purpose | MySQL | `psql` |
|---|---|---|
| List databases | `SHOW DATABASES` | `\l` |
| Switch databases | `USE shop` | `\c shop`, which actually reconnects |
| List tables | `SHOW TABLES` | `\dt` |
| Describe a table | `DESC users` | `\d app.users` |
| List users | Query `mysql.user` | `\du` |
| Show the current connection | `SELECT CONNECTION_ID()` | `SELECT pg_backend_pid()` |
| Display expanded results | `\G` | `\x` |

These commands may look arbitrary, but they follow a pattern: `d` means describe, followed by the first letter of the object type. Thus `\dt` lists tables, `\di` indexes, `\dn` schemas, and `\df` functions. Remember that pattern instead of memorizing every command; `\?` lists them all.

Also remember that backslash commands are parsed by the `psql` client, not by PostgreSQL as SQL. Applications must query `information_schema` or `pg_catalog` for metadata; the database will not recognize a `\d` command sent through an application connection.

---

## 2. Connection Model: One Connection, One Process

| Dimension | MySQL | PostgreSQL |
|---|---|---|
| What one connection represents | One thread | One independent backend process |
| Connection overhead | Relatively low; thousands of concurrent connections are not unusual | Every process has its own memory and scheduling cost |
| Application-side connection pool | Standard practice | Standard practice, but pool sizes must follow a global budget |
| External connection pool | Rarely needed | Usually added as another layer at large connection scales |

A global budget means that the pool sizes of all application instances, plus connections needed by operations, monitoring, and background jobs, must remain within `max_connections`, with spare capacity reserved for administrators. There is no universal formula such as “twice the CPU core count” for pool sizing. Determine it through load testing while observing connection-acquisition waits, CPU, I/O, lock contention, query latency, and throughput.

When connection counts grow, you also need to know who owns each connection. Application connection strings should include an identifiable `application_name`:

```text
postgresql://user:password@localhost:5432/shop?application_name=order-api
```

It appears in `pg_stat_activity`, allowing you to identify the service behind a problematic connection while investigating slow queries and lock waits.

Finally, if you add an external pooler such as PgBouncer, remember that its commonly used transaction-pooling mode binds a backend connection only for the duration of one transaction. Anything expected to persist across transactions is unreliable—including session state set by `SET` (such as `search_path`), temporary tables, and `currval()`. Use `SET LOCAL` and `RETURNING` instead.

---

## 3. Authentication and Authorization: Roles Only, with Privileges Declared in Advance

PostgreSQL does not distinguish between separate “user” and “role” object types; it has roles only. The `LOGIN` attribute determines whether a role can log in, and `CREATE USER` is merely an alias for `CREATE ROLE ... LOGIN`. Unlike MySQL’s `'user'@'host'`, a PostgreSQL role does not include a source host. Source addresses, databases, users, and authentication methods are managed through `pg_hba.conf` or a cloud database’s access controls.

A common design separates object ownership from application access: a non-login owner owns all tables, while the application account receives only read and write privileges.

```sql
CREATE ROLE shop_owner NOLOGIN;
CREATE ROLE shop_app LOGIN PASSWORD 'replace-me';

CREATE SCHEMA app AUTHORIZATION shop_owner;

GRANT USAGE ON SCHEMA app TO shop_app;
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app TO shop_app;
```

There is an easy trap here: `ON ALL TABLES` affects only tables that exist when the statement runs. Tables created later do not automatically inherit those privileges. To grant privileges on future objects, declare them in advance with `ALTER DEFAULT PRIVILEGES`. Crucially, the rule is matched according to who creates the objects:

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE shop_owner IN SCHEMA app
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO shop_app;
```

In other words, subsequent tables must be created as `shop_owner` for this declaration to apply.

---

## 4. Data Types: Not Simple One-to-One Replacements

Most MySQL types have PostgreSQL counterparts. These can be carried over without much thought:

| MySQL | PostgreSQL |
|---|---|
| `TINYINT` / `SMALLINT` | `smallint` |
| `INT` / `BIGINT` | `integer` / `bigint` |
| `FLOAT` / `DOUBLE` | `real` / `double precision` |
| `TEXT` / `LONGTEXT` | `text` |
| `DATE` / `TIME` | `date` / `time` |
| `BLOB` | `bytea` |

The remaining types deserve closer inspection before migration:

| MySQL | PostgreSQL | Key difference |
|---|---|---|
| `BIGINT AUTO_INCREMENT` | `bigint GENERATED BY DEFAULT AS IDENTITY` | Backed by an independent sequence object |
| `INT UNSIGNED` | `bigint` with a `CHECK` limiting its range | There are no unsigned integers; using `integer` halves the available range, while `BIGINT UNSIGNED` must fall back to `numeric(20, 0)` |
| `TINYINT(1)` / `BOOL` | `boolean` | Accepts only `true`, `false`, and `NULL`; integers are not implicitly converted, so `VALUES (1)` raises a type error |
| `DECIMAL(m, n)` | `numeric(m, n)` | Do not omit precision in either system: bare `DECIMAL` in MySQL means `DECIMAL(10, 0)` and discards fractional digits, while bare `numeric` in PostgreSQL has arbitrary precision |
| `VARCHAR(n)` | `varchar(n)` or `text` | They have the same performance; the value of `varchar(n)` is expressing a business constraint, not being “shorter and faster” |
| `DATETIME` / `TIMESTAMP` | `timestamp` / `timestamptz` | The dividing line is “wall-clock time” versus an “absolute point in time” |
| `JSON` | `jsonb` | `json` stores only text; `jsonb` supports comparison, search, and GIN indexes |
| `ENUM(...)` | `text` with a `CHECK`, or a native enum | Adding or removing values from a native enum changes the type definition, making it unsuitable for frequently changing business states |
| UUID stored as `CHAR(36)` | Native `uuid` | For primary keys, prefer time-ordered UUIDv7 over fully random v4 |
| no equivalent | `array`, `range`, `inet`, `tsvector`, and others | Useful, but not substitutes for sound table design |

Three areas deserve additional explanation.

### 4.1 Identity Guarantees Uniqueness, Not Continuity

`BY DEFAULT` permits explicitly supplied IDs and behaves more like `AUTO_INCREMENT`. `GENERATED ALWAYS` is stricter: explicitly inserting a value requires `OVERRIDING SYSTEM VALUE`, a difference that becomes especially visible during bulk imports. The `bigserial` commonly found in older code still works, but it is a historical shorthand for “column + sequence + default.” Prefer the SQL-standard identity syntax for new tables.

More importantly, adjust your expectations of the ID itself. Identity is backed by an independent sequence object. Failed inserts, transaction rollbacks, `ON CONFLICT`, caching, and crashes can all consume sequence values:

> Auto-incrementing IDs guarantee uniqueness, not continuity, and they cannot precisely represent transaction commit order.

### 4.2 The Boundary Between `timestamp` and `timestamptz`

`timestamp` (fully, `timestamp without time zone`) stores a “wall-clock time” and does not represent a unique instant on the global timeline. `timestamptz` (fully, `timestamp with time zone`) stores an absolute point in time, converting input and output according to the session time zone. Note that it does not retain the original time-zone name: after writing `2026-07-24 09:00 Asia/Shanghai`, the database stores the corresponding instant. Reading it under a different `TimeZone` changes the displayed value.

As a rule of thumb, MySQL `TIMESTAMP` is closer to `timestamptz`, while `DATETIME` is closer to `timestamp`, but every column must ultimately be evaluated according to its business semantics. Order creation and payment times should use `timestamptz`. Birthdays, “opens daily at 09:00,” and future events at a particular location may require `date`, `time`, `timestamp`, or even a separately stored IANA time-zone name such as `Asia/Shanghai`. For fields modeled according to local-time semantics, remember to test duplicated and nonexistent local times around DST transitions.

Choosing the right type is only the first step. Absolute timestamps must remain consistent throughout the entire path: configure database connections with `TimeZone=UTC`, use ISO 8601 values with offsets in APIs, store values as `timestamptz`, and convert them to the user’s time zone only in the presentation layer.

MySQL’s `ON UPDATE CURRENT_TIMESTAMP` also has no PostgreSQL equivalent. To update `updated_at` automatically, either create a trigger or assign it explicitly in every `UPDATE`.

### 4.3 `jsonb` and Several Types MySQL Does Not Have

`json` stores the original input text, preserving whitespace and key order; `jsonb` stores a parsed binary structure. A MySQL `JSON` column should usually become `jsonb`, because only `jsonb` supports containment checks and GIN indexes:

```sql
SELECT *
FROM app.orders
WHERE attributes @> '{"channel": "ios"}'::jsonb;

CREATE INDEX orders_attributes_gin_idx
ON app.orders USING gin (attributes);
```

But JSONB is not an excuse to avoid designing a schema. Stable data that is frequently filtered, constrained, or related to other data should still be modeled as ordinary columns.

The same restraint applies to the types at the end of the table that lack MySQL equivalents: they are worth using, but not as shortcuts around proper modeling. UUIDv7 is recommended for primary keys because fully random v4 values scatter inserts across the index, a cost familiar to most MySQL developers. Recent PostgreSQL versions include `uuidv7()`; with older versions, generate it in the application. Arrays suit small, clearly bounded sets such as tags and permissions, but should not replace normal one-to-many relationships. Range types represent intervals and, together with exclusion constraints, can express “overlap is forbidden,” as the next section demonstrates.

---

## 5. Constraints: Let the Database Guarantee Correctness

Both systems support `PRIMARY KEY`, `UNIQUE`, `FOREIGN KEY`, `CHECK`, and `NOT NULL` with largely identical syntax. The main differences are:

| Dimension | MySQL | PostgreSQL |
|---|---|---|
| `NULL` under `UNIQUE` | Multiple `NULL` values are allowed, with no way to change the behavior | The default is the same, but `NULLS NOT DISTINCT` can make `NULL` values conflict |
| Uniqueness for only some rows | No corresponding syntax | A conditional unique index: `CREATE UNIQUE INDEX ... WHERE ...` |
| Indexes on foreign-key columns | Created automatically by InnoDB | Not created automatically; you must add them |
| Exclusion constraints | Cannot be declared | `EXCLUDE` expresses that no two rows may simultaneously satisfy a set of conditions |

One easily forgotten detail is the same in both systems: a `CHECK` passes when its expression evaluates to either `TRUE` or `NULL`. Consequently, `CHECK (amount >= 0)` does not prevent `NULL`; add `NOT NULL` if null values are forbidden.

### 5.1 Uniqueness Can Be Conditional

To make `NULL` values conflict with one another, simply declare `UNIQUE NULLS NOT DISTINCT (provider, external_id)`.

A more common requirement is uniqueness only for rows satisfying a condition. With soft deletion, an ordinary UNIQUE constraint cannot express “an email address can belong to only one account that has not been deleted,” but a conditional unique index can:

```sql
CREATE UNIQUE INDEX users_active_email_uq
ON app.users (lower(email))
WHERE deleted_at IS NULL;
```

Only rows where `deleted_at IS NULL` enter the index. Soft-deleted records automatically leave the uniqueness check, allowing the same email address to register again. Indexing `lower(email)` also makes uniqueness case-insensitive.

### 5.2 `EXCLUDE`: A Constraint MySQL Cannot Declare

Another class of constraint has no MySQL equivalent: preventing reservation periods for the same room from overlapping.

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

This declares that no two rows may have both the same room number and overlapping time ranges. Unlike “check for an overlap, then insert,” it genuinely guarantees correctness under concurrency. Section 7 explains why a query cannot provide the same protection.

### 5.3 Foreign-Key Columns Are Not Indexed Automatically

When InnoDB creates a foreign key, it also creates an index on the referencing columns in the child table. PostgreSQL does not. This is easy to overlook because the foreign key still works without an index. However, when the parent table executes a `DELETE` or `UPDATE`, PostgreSQL must scan the child table for referencing rows. Frequent changes to parent rows can therefore accumulate scan overhead and lock waits.

In practice, you need not create a dedicated index for every foreign key. It is enough for the foreign-key column to be the first column of a composite index. For example, an index on the orders table beginning with `user_id` can support both referential checks and queries for a user’s orders.

Conversely, primary keys and UNIQUE constraints already have indexes. Do not create duplicate indexes on the same columns.

---

## 6. Writing SQL: CRUD and Dialect Differences

The following examples use `$1` and `$2` as parameter placeholders, following the PostgreSQL server protocol. The actual syntax in an application depends on the driver or ORM; some Python drivers, for example, use `%s`.

### 6.1 Retrieve Written Values with `RETURNING`

MySQL applications often call `LAST_INSERT_ID()` after an insert. PostgreSQL can directly return identity values, defaults, and columns modified by triggers:

```sql
INSERT INTO app.users (email, display_name)
VALUES ($1, $2)
RETURNING id, email, created_at;
```

`RETURNING` also applies to updates and deletes:

```sql
UPDATE app.orders
SET status = 'paid',
    updated_at = now()
WHERE id = $1
  AND status = 'pending'
RETURNING id, status, updated_at;
```

The application can determine whether the state transition succeeded from whether a row was returned, without first querying and then updating.

### 6.2 `ON CONFLICT` Is More Explicit Than “Ignore All Errors”

The PostgreSQL counterpart to MySQL’s `ON DUPLICATE KEY UPDATE` is `INSERT ... ON CONFLICT`, but it requires an explicit conflict target that matches a unique constraint or unique index. For the earlier partial unique index on `app.users` involving `lower(email)` and a condition, both the expression and predicate must be specified:

```sql
INSERT INTO app.users (email, display_name)
VALUES ($1, $2)
ON CONFLICT ((lower(email))) WHERE deleted_at IS NULL
DO UPDATE SET display_name = EXCLUDED.display_name
RETURNING id, email, display_name;
```

`ON CONFLICT DO NOTHING` ignores only the corresponding uniqueness or exclusion conflict. It is not equivalent to MySQL’s `INSERT IGNORE`, which may also downgrade certain invalid values, truncation problems, and constraint errors to warnings.

Likewise, `ON CONFLICT DO UPDATE` is not equivalent to `REPLACE INTO`. MySQL `REPLACE` usually deletes the old row and inserts a new one, potentially triggering deletions, foreign-key cascades, and new default values. A PostgreSQL upsert updates the conflicting row. Before rewriting such SQL, confirm whether the business actually requires an “update” or a “delete followed by an insert.”

### 6.3 `UPDATE ... FROM` and `DELETE ... USING`

MySQL’s `UPDATE ... JOIN` becomes `UPDATE ... FROM` in PostgreSQL, with the join condition in `WHERE`:

```sql
UPDATE app.orders AS o
SET status = 'canceled',
    updated_at = now()
FROM app.users AS u
WHERE u.id = o.user_id
  AND u.deleted_at IS NOT NULL;
```

Joined deletion follows the same pattern, replacing `FROM` with `USING`.

There are two differences from MySQL. First, ensure that each target row matches at most one source row. If `FROM` produces multiple matches, PostgreSQL chooses one arbitrarily, and you must not rely on the result. Second, the statement modifies only one target table; it is not equivalent to MySQL’s multi-table update or multi-table delete.

### 6.4 Differences in Common Functions and Expressions

Everyday SQL requires several common rewrites:

| MySQL | PostgreSQL |
|---|---|
| `IFNULL(value, default)` | `COALESCE(value, default)` |
| `GROUP_CONCAT(name ORDER BY name SEPARATOR ',')` | `string_agg(name, ',' ORDER BY name)` |
| `DATE_FORMAT(created_at, '%Y-%m-%d')` | `to_char(created_at, 'YYYY-MM-DD')` |
| `LIMIT offset, count` | `LIMIT count OFFSET offset` |
| `CAST(value AS type)` | Also uses `CAST`, or can be written as `value::type` |

String comparison is the easiest difference to overlook. MySQL’s default collation is case-insensitive—the `ci` suffix in `utf8mb4_0900_ai_ci` means case-insensitive—so `WHERE email = 'Foo@example.com'` can match a stored value of `foo@example.com`. PostgreSQL does not behave this way by default. `text` comparison is strictly case-sensitive, so the same SQL returns nothing:

```sql
-- Case-sensitive; a difference in letter case prevents a match
SELECT id FROM app.users WHERE email = $1;

-- Case-insensitive and able to use the lower(email) index
SELECT id FROM app.users WHERE lower(email) = lower($1);
```

This is also why the earlier unique index on `app.users` indexes `lower(email)` rather than `email`. An alternative to wrapping both sides in `lower()` is changing the column to the `citext` type, which makes comparison itself case-insensitive at the cost of another extension dependency.

Both systems provide `concat()`, but their `NULL` semantics differ. MySQL’s `concat('a', NULL)` returns `NULL`, whereas PostgreSQL ignores `NULL` and returns `'a'`. If you need null-propagating behavior, use the `||` operator, which makes the whole expression `NULL` when an operand is `NULL`.

Integer division can silently change results. In MySQL, `SELECT 1 / 2` returns `0.5000`. In PostgreSQL, division between two integers remains integer division and returns `0`. Cast first when you need a fractional result: `1::numeric / 2`.

The default sort order for `NULL` is also reversed. MySQL usually places `NULL` first in ascending order, while PostgreSQL places it last by default. For stable semantics, write `ORDER BY deleted_at ASC NULLS LAST` explicitly.

### 6.5 Batch Updates Cannot Use `LIMIT` Directly

PostgreSQL has no MySQL-style `UPDATE ... LIMIT` or `DELETE ... LIMIT`. A reliable approach first selects primary keys in a stable order and then updates them:

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

This is also a minimal implementation of a concurrent job queue: multiple worker processes claim jobs simultaneously, and `SKIP LOCKED` skips rows already locked by another process.

### 6.6 Reduce Application Code with CTEs, Window Functions, and LATERAL

MySQL 8 supports these features too, but code migrated from MySQL often underuses them and instead pulls data into the application for computation.

To query each user’s three most recent orders:

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

Top-N queries, cumulative values, rankings, and adjacent-row calculations do not require loading the entire data set into application memory.

`LATERAL` allows a subquery on the right to reference rows on the left, making it useful for “one row per group”:

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

Combined with an index on `(user_id, created_at DESC, id DESC)`, this can execute such queries efficiently and avoid issuing one SQL statement per user in an application loop.

---

## 7. Transactions and Concurrency: Where Code Still Runs but Its Semantics Change

### 7.1 Different Default Isolation Levels

This is one of the most easily overlooked differences:

- MySQL InnoDB usually defaults to `REPEATABLE READ`.
- PostgreSQL defaults to `READ COMMITTED`.

Both can be reconfigured, so inspect the actual setting rather than relying on assumptions.

Under PostgreSQL Read Committed, each statement receives a new snapshot when it begins. This means two successive queries of the same row within one transaction can return different results if another transaction commits an update between them. That does not happen under InnoDB Repeatable Read.

Switching to Repeatable Read does not let you apply InnoDB experience directly. Both systems hold a fixed snapshot during the transaction, but they handle write conflicts very differently. InnoDB `UPDATE` and `DELETE` statements continue against a newer committed version. PostgreSQL instead fails with SQLSTATE `40001` if another transaction has modified and committed the target row. Serializable’s SSI detection returns the same error code.

Therefore, using Repeatable Read or Serializable in PostgreSQL requires the application to catch `40001`, roll back, and retry the entire transaction from `BEGIN`. PostgreSQL Repeatable Read is fundamentally Snapshot Isolation and can still exhibit write skew, so do not treat it as Serializable.

### 7.2 Do Not Carry Over InnoDB Gap-Lock Assumptions

For locking range reads, updates, and deletes under Repeatable Read, InnoDB commonly uses next-key locks to lock records and the gaps between them, preventing inserts into the range.

PostgreSQL has no equivalent ordinary gap lock. `SELECT ... FOR UPDATE` locks only rows that are actually returned. A query returning no rows does not mean that future inserts satisfying the condition are locked out.

This is why Section 5 emphasized constraints. In PostgreSQL, constraints are not merely a supplement to application validation; they are a primary mechanism for concurrency correctness. If the business logic relies on “verify that no row exists, then insert,” replace it with:

- Database constraints such as UNIQUE or EXCLUDE.
- `INSERT ... ON CONFLICT`.
- Serializable isolation with correct retries.
- An advisory lock when necessary, with a carefully designed lock key and lifecycle.

Do not substitute a query result for a constraint.

### 7.3 `NOWAIT` and `SKIP LOCKED`

In addition to the familiar `FOR UPDATE` and `FOR SHARE`, row locks support two modifiers. `NOWAIT` raises an error if the lock cannot be acquired immediately instead of waiting in a queue. `SKIP LOCKED` skips rows that are already locked. The latter deliberately returns an inconsistent view, which is suitable for work queues where any available worker may claim a job, but not for ordinary reports or account-balance queries.

### 7.4 Sequences Do Not Roll Back

Sequence values are not governed by transaction control. If a transaction inserts a row and then executes `ROLLBACK`, the allocated ID will usually not be reused by the next insert. Changes to sequence state made by `setval()` likewise are not undone when a transaction rolls back.

If the business requires invoice numbers with absolutely no gaps, that is a business-numbering problem requiring serialization, auditing, and compensation. It cannot be solved directly with an identity primary key.

### 7.5 DDL Can Be Transactional, but It Still Takes Locks

PostgreSQL allows the following:

```sql
BEGIN;
ALTER TABLE app.orders ADD COLUMN note text;
UPDATE app.orders SET note = '';
COMMIT;
```

The whole operation can be rolled back on failure. MySQL 8.x atomic DDL primarily guarantees that each individual DDL statement completes atomically; it does not mean that DDL can be rolled back transactionally together with multiple DML statements.

But being transactional does not mean being nonblocking, and not all DDL can run inside a transaction. The frequently used `CREATE INDEX CONCURRENTLY` on large tables is one exception. It avoids blocking normal writes for an extended period at the cost of taking longer and consuming more CPU and I/O, and it cannot run inside a transaction block (`VACUUM` and `CREATE DATABASE` cannot either). Migration scripts must treat it as a separate step. If it fails, it also leaves an invalid index that must be dropped before rebuilding.

### 7.6 Timeouts: Put Limits on Locks and Statements

PostgreSQL has several timeout parameters that are easy to confuse:

- `lock_timeout`: the maximum time spent waiting for any single lock.
- `statement_timeout`: the maximum execution time for an entire statement, including lock waits.
- `transaction_timeout`: the maximum duration of a transaction from beginning to end, after which it is aborted.
- `idle_in_transaction_session_timeout`: the maximum time a session may remain inactive after opening a transaction.
- `deadlock_timeout`: how long to wait before starting deadlock detection, not an upper limit on lock waits.

In practice, `lock_timeout` is usually shorter than `statement_timeout`, and both are shorter than the application request deadline. When a deadlock occurs, PostgreSQL aborts one of the transactions. Applications should use a consistent lock-acquisition order whenever possible.

### 7.7 Retry the Entire Transaction After an Error

After a statement fails inside an explicit PostgreSQL transaction, the transaction usually enters an aborted state. Subsequent statements receive `25P02`; you must execute `ROLLBACK` or roll back to a savepoint established in advance. This also determines retry granularity: only the entire transaction can be retried, not just the failed statement.

Use PostgreSQL’s five-character SQLSTATE—not error text—to decide whether an error is retryable:

| SQLSTATE | Meaning | Typical handling |
|---|---|---|
| `23505` | unique violation | Return a conflict, or handle it according to explicit business semantics |
| `23503` | foreign key violation | Return a data-relationship error |
| `40001` | serialization failure | Retry the entire transaction from `BEGIN` |
| `40P01` | deadlock detected | Retry the entire transaction with a limit and backoff |
| `55P03` | lock not available | Retry a limited number of times when appropriate for the contention scenario |
| `57014` | query canceled | First determine whether the cause was a timeout, user cancellation, or an administrative operation |

Do not match error text such as “duplicate key value”; it can change with language, version, and context.

Retries themselves require two disciplines: use a maximum attempt count, exponential backoff, and random jitter; and make side effects outside the database—HTTP calls, messages, files, and so on—idempotent, or use a transactional outbox.

If the network disconnects around `COMMIT`, the client may be unable to tell whether the transaction committed. Blindly replaying the write is unsafe; you need a business idempotency key or a way to query the result.

---

## 8. Vacuum: The Lesson Every MySQL Developer Must Learn

### 8.1 The Cost of Snapshots

The “snapshots” repeatedly discussed in the previous section become multiple row versions at the storage layer. The two systems retain old versions differently. InnoDB stores historical versions in the undo log and removes them through purge. PostgreSQL writes new versions directly into the table: an `UPDATE` appends a new tuple to the heap, and the old tuple becomes a dead tuple once it is no longer visible to any transaction. A `DELETE` likewise leaves a version awaiting reclamation.

The consequence is that PostgreSQL’s garbage accumulates in the table itself, so something must collect it. That process is Vacuum, whose responsibilities include:

- Reclaiming space occupied by dead tuples for later reuse by the same table.
- Maintaining the visibility map so some queries can read only the index without visiting the table.
- Removing invalid references from indexes.
- Freezing old transaction IDs to prevent transaction ID wraparound.

`ANALYZE` gathers statistics that the query planner uses to estimate data distribution. Autovacuum automatically runs Vacuum and Analyze according to configured thresholds. It is not an optional background optimization; it is part of normal PostgreSQL operation.

### 8.2 Ordinary Vacuum Usually Does Not Shrink Files

Ordinary `VACUUM` primarily makes space reusable within the table; it usually does not immediately return file space to the operating system. After deleting a large amount of data, disk usage may therefore fail to decrease and may even continue growing—one of the most confusing behaviors for developers arriving from MySQL.

`VACUUM FULL` rewrites the entire table and shrinks its file, but it needs additional disk space and an `ACCESS EXCLUSIVE` lock. It should not be a routine scheduled task. When bloat appears, first determine why reclamation cannot keep up instead of running Full at fixed intervals.

### 8.3 Long-Running Transactions Prevent Old Versions from Being Reclaimed

As long as an old snapshot might still see a historical tuple, Vacuum cannot remove it. The two most common causes are long-running transactions and sessions that execute `BEGIN` but do not commit, remaining `idle in transaction`. Under MySQL, such sessions enlarge undo; under PostgreSQL, they directly stall garbage collection across the database.

The most important application rule is therefore:

> Keep transactions as short as possible. Do not wait for user input, remote HTTP calls, message queues, or lengthy business computations inside a transaction.

There should also be a safeguard outside the code. The `transaction_timeout` and `idle_in_transaction_session_timeout` settings from the previous section exist for exactly these cases, ensuring that a single runaway transaction cannot indefinitely hold back reclamation across the database.

To find such transactions, query `pg_stat_activity` and sort by `xact_start` to see how long the oldest transaction has been open; the `state` column identifies `idle in transaction` sessions. To check whether reclamation is keeping up, inspect `n_dead_tup` and `last_autovacuum` in `pg_stat_user_tables`. Remember that `n_dead_tup` is only an estimate, not an exact measure of bloat.

---

## 9. Indexes and Query Optimization: Beyond B-Tree to the PostgreSQL Toolbox

### 9.1 B-Tree Experience Still Applies

B-Tree remains the default index and supports equality, ranges, ordering, and `IS NULL`. Composite indexes still depend on their leftmost columns. Consider paginating through a user’s orders:

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

InnoDB secondary-index entries automatically contain the primary key, but ordinary PostgreSQL secondary indexes provide no such guarantee. Here, `id` participates in sorting and determines the pagination position, so it must be an index key. The `status` and `amount` columns are needed only in the result and are attached with `INCLUDE`. This index does not suit querying all site-wide orders by `created_at` alone. An index containing a column does not mean every query involving that column can use it efficiently.

### 9.2 Three Highly Practical B-Tree Variants

An expression index allows queries such as `WHERE lower(display_name) = $1` to use an index without maintaining an additional redundant column:

```sql
CREATE INDEX users_display_name_lower_idx
ON app.users (lower(display_name));
```

A partial index contains only rows satisfying a condition. It is useful when data is distributed very unevenly and queries care about only a small subset of statuses:

```sql
CREATE INDEX orders_pending_created_idx
ON app.orders (created_at, id)
WHERE status = 'pending';
```

The planner must be able to prove that the query condition implies the index predicate. With a parameterized condition such as `status = $1`, the planner may be unable to confirm that `$1` is always `'pending'`, so it may not use the partial index.

Covering indexes are expressed with `INCLUDE`; the `orders_user_created_idx` from the previous subsection is one example. `INCLUDE` columns do not participate in searching, sorting, or uniqueness. They merely give an Index Only Scan the opportunity to obtain result values directly. Whether the scan can truly avoid visiting the table also depends on the visibility map discussed in the previous section, so “adding INCLUDE always eliminates table access” is false.

### 9.3 GIN, GiST, and BRIN Solve Different Problems

| Type | Suitable use cases | Characteristics |
|---|---|---|
| GIN | JSONB, arrays, full-text search | An inverted index with powerful querying but higher write and build costs |
| GiST | Ranges, geometry, nearest-neighbor search | An extensible framework whose capabilities depend on the operator class |
| BRIN | Very large time-series tables and columns highly correlated with physical order | Very small but lossy, requiring rechecks |

Two earlier examples belong to these categories: the GIN index on `jsonb` and the GiST index underlying the exclusion constraint.

BRIN is not merely “a general B-Tree that uses less space.” It can exclude large numbers of data blocks with a tiny index only when the table is sufficiently large and the indexed column is highly correlated with the physical insertion order—a continuously appended timestamp is the classic example.

### 9.4 Let Actual Execution Plans Speak

Plain `EXPLAIN` shows only the planner’s estimates. The truly useful approach adds `ANALYZE` and `BUFFERS` to execute the query:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM app.orders
WHERE user_id = 1
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Focus on three things: which node consumes most of the time, whether estimated row counts differ drastically from actual row counts, and whether data was found in cache or read from disk. Severe estimation errors usually mean statistics are stale or the data distribution violates the planner’s assumptions, which is also one of the most common causes of degraded execution plans.

Remember that `cost` is the planner’s cost unit, not milliseconds. `EXPLAIN ANALYZE` really executes the statement, so take particular care with write operations.

PostgreSQL has no built-in general-purpose query hints and therefore no MySQL-style `FORCE INDEX` for forcing a particular index. When asking “why didn’t it use the index?”, first verify that the query condition genuinely matches the index. Implicit type conversions, functions wrapped around columns, and incompatible collations can all prevent index use. Only then consider whether table size and selectivity make an index worthwhile.

---

## 10. Extensions: The Other Half of PostgreSQL’s Capabilities

The `btree_gist` used earlier is an extension. This is one of the greatest differences between PostgreSQL and MySQL: many tasks that require external components in MySQL are available in PostgreSQL through a single `CREATE EXTENSION`.

Extensions are more than “built-in function libraries.” They can add types, operators, index access methods, background processes, and even hooks into the query planner. The previous section noted that PostgreSQL has no built-in query hints, but the `pg_hint_plan` extension adds them. Extensions can change nearly everything that core code can, so PostgreSQL’s practical capability boundary is much broader than its built-in feature list.

### 10.1 Extensions Are Database-Scoped

Installing an extension is simple—`CREATE EXTENSION IF NOT EXISTS pg_trgm` is enough—but two details are easy to miss.

First, an extension is installed in **one particular database**, not across the entire instance. To use it in another database within the same instance, install it again. This follows the hierarchy described in Section 1: the types and functions added by an extension belong to a schema and therefore to a database.

Second, installing extensions usually requires administrative privileges. Some extensions marked as trusted—such as `pg_trgm`, `btree_gist`, `citext`, and `pgcrypto`—can be installed by ordinary users who have permission to create objects, but many others still require a superuser. Extensions therefore belong in initialization scripts; applications should not create them on demand at runtime.

Use `\dx` to see what is currently installed and query `pg_available_extensions` to see what can be installed on the instance.

### 10.2 Extension Categories Worth Knowing

| Extension | What it solves | Typical MySQL approach |
|---|---|---|
| `pg_stat_statements` | Aggregates call counts and execution time by SQL fingerprint | Slow query log plus `performance_schema` |
| `pg_trgm` | Makes `LIKE '%keyword%'` and similarity matching indexable | Full-table scan or an ngram full-text index |
| `citext` | A case-insensitive text type | The default collation is already case-insensitive |
| `pgcrypto` | Hashing, encryption, and decryption functions | Implement in the application or use `AES_ENCRYPT` |
| `postgres_fdw` | Queries tables in another database as though they were local | The `FEDERATED` engine, which is rarely used |
| `pg_cron` | Scheduled tasks inside the database | The `EVENT` scheduler |
| `pgvector` | A vector type and nearest-neighbor search | Deploy a separate vector database |
| `PostGIS` | Comprehensive geospatial capabilities | Built-in spatial types, with a large capability gap |
| `TimescaleDB` | Partitioning, compression, and preaggregation for time-series workloads | Manual partitioning plus archival |

`pg_stat_statements` is worth installing first. It must be preloaded and is usually configured by an administrator; managed databases often enable it by default. It aggregates call counts, total execution time, and average execution time by statement fingerprint, providing the entry point for the execution-plan analysis from the previous section. First use it to identify which statements deserve investigation, then run `EXPLAIN (ANALYZE, BUFFERS)` on specific statements. Focusing only on individually slow queries can easily miss statements that are fast per call but execute hundreds of thousands of times.

`pg_trgm` addresses a pain point familiar to MySQL developers. Because `LIKE '%keyword%'` has no usable prefix, a B-Tree cannot help, and both systems default to a full-table scan. `pg_trgm` splits strings into three-character fragments and builds a GIN index, allowing both fuzzy matching and `similarity()` ordering to use an index.

`pgvector` illustrates another point: PostgreSQL often uses extensions to absorb an entire new class of requirements. It adds a `vector` type and nearest-neighbor indexes, keeping embedding search in the business database and eliminating the need to synchronize a separate external vector store.

### 10.3 The Cost: Extensions Become Part of the Runtime Environment

Installing an extension is not as lightweight as adding a package dependency.

Managed databases usually expose only an allowlist of extensions, and the list differs by cloud provider. Before making core functionality depend on an extension, confirm that the target environment supports it. Ask especially early about extensions with deeper integration, such as PostGIS, TimescaleDB, and pgvector.

Extensions have their own versions. A major database upgrade may require `ALTER EXTENSION ... UPDATE`, and some extensions can even block an upgrade path. Their maintenance activity is worth checking during technology selection.

One final rule: when built-in functionality is sufficient, do not introduce an extension. Use the built-in `gen_random_uuid()` to generate UUIDs instead of installing `uuid-ossp`; `jsonb` is enough for key-value storage, and `hstore` is now encountered mainly when maintaining legacy code.

---

## 11. Conclusion: Make MySQL Experience a Reference Point, Not a Constraint

Moving from MySQL to PostgreSQL does not require discarding all your existing experience.

You can retain your understanding of relational modeling, transactional thinking, indexing fundamentals, constraints, and performance diagnosis through execution plans. What you must rebuild are the following default assumptions:

| MySQL experience | New understanding to build in PostgreSQL |
|---|---|
| A database is a schema | Databases and schemas are different levels |
| One connection is one thread | One connection is one process |
| `AUTO_INCREMENT` is a column property | Identity is backed by an independent sequence object |
| `VARCHAR(255)` is a safe default | Use `text` when there is no business-defined maximum |
| `INSERT IGNORE` / `REPLACE` | `ON CONFLICT` has narrower, more explicit semantics |
| InnoDB defaults to RR | PostgreSQL defaults to RC; concurrency semantics must be explicit |
| Range locks can prevent inserts | PostgreSQL has no ordinary gap lock; rely on constraints, upserts, or Serializable |
| Old versions live in undo and are removed by purge | Old versions remain in the table and are reclaimed by Vacuum |
| Indexes are essentially all B-Trees | Choose B-Tree, GIN, GiST, or BRIN according to operators and data distribution |
| Add an external component when the database falls short | Fuzzy matching, geospatial, vector, and many other capabilities have corresponding extensions |

Individually, none of these differences looks large. The difficulty is that most do not produce errors: SQL still runs and code still reaches production. Problems emerge only when concurrency rises, data grows, or a long-running transaction gets stuck. The real task when moving from MySQL is therefore not memorizing syntax mappings. It is pausing whenever you think, “surely it naturally works this way,” and verifying whether that MySQL-derived assumption still holds in PostgreSQL.

---

## References

- [PostgreSQL Documentation](https://www.postgresql.org/docs/current/)
- [MySQL 8.4 Reference Manual: InnoDB Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
