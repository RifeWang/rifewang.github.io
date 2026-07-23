# Building a CDC System with PostgreSQL WAL: Principles and Engineering Practice

## A practical guide to WAL, logical decoding, replication slots, snapshot handoff, correctness, and the differences from MySQL binlog CDC

![Architecture of a CDC system built on PostgreSQL WAL](https://raw.githubusercontent.com/RifeWang/images/refs/heads/master/postgresql/00-pg-wal-cdc-cover.png)

## Preface

When a row changes in a database, the impact rarely remains confined to the database:

- Changes to products and content must promptly update Elasticsearch indexes.
- Changes to user profiles or configuration must refresh Redis and derived views.
- Business data must continuously flow into Kafka, a data warehouse, or a data lake.
- Changes to critical data such as accounts and permissions must leave an audit trail and trigger downstream processes.

Although these scenarios have different downstream systems, they all depend on the same capability: continuously capturing `INSERT`, `UPDATE`, and `DELETE` operations in the database and reliably propagating those changes. This is **`CDC` (Change Data Capture)**.

The real challenge is not “reading a change once.” It is continuing to guarantee the following when consumers restart, networks fail, full scans run, downstream systems become congested, or primary/standby failover occurs:

1. No committed data is lost.
2. Changes to the same row have a controllable order.
3. Consumers can resume from the correct position.
4. Existing data and subsequent incremental changes join seamlessly.

Starting from these problems, this article progressively breaks down the principles and engineering implementation of PostgreSQL WAL CDC, with a comparison to the more familiar MySQL binlog CDC.

![CDC use cases and implementation approaches](https://raw.githubusercontent.com/RifeWang/images/refs/heads/master/postgresql/01-cdc-use-cases-and-approaches.png)

---

## 1. Why Choose Log-Based CDC

Suppose database changes need to be synchronized to a search engine, cache, or message queue. The most intuitive approach is for the application to write to the downstream system after writing to the database. As soon as failures are considered, however, the problem becomes less straightforward: if the database transaction has committed but sending the message fails, which system is authoritative?

There are four common `CDC` approaches:

| Approach | How it works | Advantages | Main problems |
|---|---|---|---|
| Application-level delivery | The application actively delivers a message after writing to the database | Can carry business semantics | Creates a dual-write consistency problem and struggles to cover batch SQL or database changes that bypass the application |
| Query polling | Periodically queries by `updated_at` | Simple and requires few privileges | Adds latency and load to the primary; detecting deletions usually requires soft-delete markers such as `deleted_at` in the data model |
| Database triggers | Creates triggers on business tables that synchronously write to a change table when rows change | Real-time and transactional | Intrusive to the application, causes write amplification, and adds work to the database |
| Log parsing | Reconstructs changes from the database transaction log | Low intrusion, low latency, and complete coverage | Depends on the database logging mechanism and requires a more complex consumer |

Log parsing has become the dominant approach for modern `CDC` because the database has already completed the hardest step for us: **recording every committed change in a recoverable order**.

In MySQL, that log is the binlog. In PostgreSQL, it is the `WAL` (Write-Ahead Log). The two serve similar goals, but their underlying semantics and engineering responsibilities differ.

---

## 2. A Comprehensive Comparison of PostgreSQL and MySQL CDC

If you are already familiar with MySQL binlog CDC, the following table provides a useful frame of reference for PostgreSQL.

| Dimension | MySQL binlog CDC | PostgreSQL WAL CDC | Engineering impact |
|---|---|---|---|
| Log foundation | The binlog was originally designed primarily for replication and point-in-time recovery | The `WAL` was originally designed primarily for crash recovery and physical replication | PostgreSQL must first perform logical decoding, whereas MySQL ROW events already have strong logical semantics |
| `CDC` prerequisites | `log_bin=ON`, `binlog_format=ROW` | `wal_level=logical` | Both require instance-level configuration, and changing some parameters may require a restart |
| Decoding method | Parse binlog events | Logical decoding + output plugin | PostgreSQL can use plugins to select the output protocol |
| Built-in output format | ROW event | `pgoutput` binary protocol | Mature client libraries are normally used to parse both |
| Data scope | Usually filtered by database and table on the consumer side | A publication declares tables and operation scope on the server; PG 15+ supports row filters and column lists | PostgreSQL can narrow the publication scope at the source |
| Consumer position | Binlog file/position or GTID | LSN | Both must be persisted and used for resumption, but their semantics are not identical |
| Progress storage | Usually the consumer stores its position | A replication slot stores the confirmed watermark on the server | PostgreSQL is more consumer-friendly, but makes the database responsible for resource retention |
| Log retention | Binlogs usually expire by time or storage limit | A slot retains the `WAL` still required by its consumer | PostgreSQL requires monitoring the effect of a slot on `WAL` retention; MySQL requires handling resumption after logs expire |
| Old values for UPDATE/DELETE | `binlog_row_image` controls row images | `REPLICA IDENTITY` is configured per table | Before launch, old-value requirements must be checked table by table in PostgreSQL |
| Full/incremental handoff | Consistent snapshot + binlog position/GTID | Consistent-point LSN + exported snapshot | The principle is the same: establish the log boundary first, then read existing data from the same point in time |
| DDL | The binlog usually contains DDL Query Events | Logical replication does not propagate DDL by default | PostgreSQL requires a separate schema-evolution design |
| Large fields | Affected by row-image configuration | Consumers must recognize unchanged TOAST | PostgreSQL consumers must not interpret “unchanged” as `NULL` |
| Failover | Requires attention to GTID continuity and topology changes | Requires attention to the timeline, slot continuity, and the new primary's starting LSN | Neither system can determine whether resumption is safe merely by reconnecting to an address |

![Comparison of MySQL binlog CDC and PostgreSQL WAL CDC](https://raw.githubusercontent.com/RifeWang/images/refs/heads/master/postgresql/02-mysql-postgresql-cdc-comparison.png)

The most important distinction to remember is:

> MySQL CDC is more like “a consumer catching up with a single binlog retained globally by the database.” PostgreSQL CDC is more like “the database recording progress for a consumer through a slot and retaining the unconfirmed `WAL` for it.”

This design makes resumption more direct in PostgreSQL. Correspondingly, when a consumer stops advancing for an extended period, the amount of `WAL` required by the slot grows and must be addressed through capacity planning and monitoring.

---

## 3. Understanding PostgreSQL CDC in One Diagram

Before examining protocol details, a PostgreSQL `CDC` pipeline can be summarized as follows:

![End-to-end PostgreSQL WAL CDC architecture](https://raw.githubusercontent.com/RifeWang/images/refs/heads/master/postgresql/03-pg-wal-cdc-architecture.png)

A transaction follows roughly this path from database execution to change delivery:

1. Before modifying a data page, PostgreSQL first writes the relevant `WAL`; at commit time, it writes a commit record.
2. After the transaction is confirmed as committed, logical decoding reconstructs the low-level changes as row-level semantics.
3. An output plugin, usually `pgoutput`, encodes the changes as protocol messages.
4. The `CDC` consumer continuously reads through a replication slot.
5. The consumer transforms, routes, and delivers the changes.
6. Only after downstream acknowledgment succeeds does the consumer report a safe LSN to PostgreSQL.
7. PostgreSQL advances the slot watermark and recycles `WAL` that is no longer needed.

This is a closed loop. If the consumer reads without acknowledging, the slot must retain an ever-growing amount of `WAL`. If it acknowledges before delivery has succeeded, data can be lost permanently.

---

## 4. How `WAL` Becomes Row-Level Events

The `WAL` is the foundation of PostgreSQL durability and crash recovery. Before a data page is flushed to disk, the corresponding changes must first be written to the `WAL`—hence the term “Write-Ahead.”

Raw `WAL` is a low-level physical or physiological log organized around data pages and internal operations. It is suitable for:

- Crash recovery: replaying `WAL` after a restart to restore data to a consistent state.
- Physical streaming replication: sending `WAL` to a standby to create a copy of the primary's data pages.

For example, a raw `WAL` record might express that “a particular data page changed,” but it cannot directly tell a consumer that “the price of the product with primary key 42 in `public.products` changed to 99.” The former is a low-level storage change; the latter is the row-level business meaning required by `CDC`.

To convert these low-level storage changes into row-level events consumable by `CDC`, PostgreSQL has provided **logical decoding** since version 9.4. The server reads the `WAL` and combines it with transaction and table-schema information to reconstruct changes with table and row semantics.

For the `WAL` to carry the information required by logical decoding, the instance parameter `wal_level` must be set to `logical`.

Logical decoding only reconstructs changes. The final output format is determined by the **output plugin**:

| Plugin | Format | Distribution | Suitable use |
|---|---|---|---|
| `pgoutput` | PostgreSQL binary logical replication protocol | Built into PG 10+ | Preferred for production: standard, efficient, and requires no extension |
| `wal2json` | JSON | Third-party plugin | Easy to debug and straightforward for consumers to integrate |
| `decoderbufs` | Protobuf | Third-party plugin | Binary output; used by early Debezium implementations |
| `test_decoding` | Text | Provided with PostgreSQL contrib, depending on the package | Learning and testing, not production |

New systems should generally prefer `pgoutput`. Its disadvantage is that it is not human-readable, but client libraries should solve that problem. Long-term operational costs for a third-party plugin should not be accepted merely for debugging convenience.

---

## 5. Four Core Concepts

Before consuming a logical replication stream, you need to understand four related concepts:

| Concept | Think of it as | Question it answers |
|---|---|---|
| Publication | Capture manifest | Which tables and operations should be emitted? |
| Replication slot | Consumer bookmark | How far has this consumer read? |
| LSN | `WAL` coordinate | Where in the log did a change occur? |
| Replica identity | Old-row identification rule | How is the previous row identified for UPDATE and DELETE? |

Together, they describe a logical replication session: the publication determines the capture scope, the LSN marks the position of each change, the replication slot uses an LSN to save consumer progress, and the replica identity determines which old values UPDATE and DELETE can carry.

![Four core concepts in PostgreSQL logical replication](https://raw.githubusercontent.com/RifeWang/images/refs/heads/master/postgresql/04-pg-cdc-core-concepts.png)

**Publication: Which changes to capture**

A publication is PostgreSQL's server-side declaration of which tables participate in logical replication.

Publications have supported selecting tables and operation types since their earliest versions. PG 15+ additionally supports row filters and column lists:

```sql
-- Publish all tables
CREATE PUBLICATION cdc_pub FOR ALL TABLES;

-- Publish only specified tables
CREATE PUBLICATION cdc_pub
FOR TABLE public.users, public.products;

-- Publish only INSERT and UPDATE
CREATE PUBLICATION cdc_pub
FOR TABLE public.users
WITH (publish = 'insert, update');

-- PG 15+: When a row filter is used for UPDATE/DELETE,
-- its filter columns must be included in the replica identity
ALTER TABLE public.orders REPLICA IDENTITY FULL;

CREATE PUBLICATION paid_orders_pub
FOR TABLE public.orders WHERE (status = 'paid');

-- PG 15+: Publish only specified columns
CREATE PUBLICATION product_price_pub
FOR TABLE public.products (id, name, price);
```

Compared with the consumer-side filtering common in MySQL, a publication can narrow the scope at the source. It is not, however, an authorization system: publication scope, user read access, and replication privileges must still be configured separately.

**Replication slot: How far the consumer has read**

Imagine that a data warehouse undergoes two hours of maintenance and the `CDC` consumer is temporarily offline. Where should it resume afterward? Who retains the logs generated during the outage?

A replication slot is the server-side bookmark PostgreSQL maintains for a consumer. A logical slot primarily tracks two watermarks:

- `restart_lsn`: the earliest `WAL` position the slot may still require.
- `confirmed_flush_lsn`: the position the consumer has explicitly confirmed as safely persisted.

As long as retention limits such as `max_slot_wal_keep_size` are not exceeded, PostgreSQL retains the `WAL` still required by the slot, allowing the consumer to resume from its previous position. If the limit is exceeded, the slot may become invalid because required `WAL` has been removed. When a consumer stops advancing for an extended period, the growth in retained `WAL` must also be monitored.

This illustrates the difference between PostgreSQL and MySQL log-retention mechanisms:

> When a MySQL consumer stops advancing for a long time, the concern is whether the binlog will expire. When a PostgreSQL consumer stops advancing for a long time, the concern is how much `WAL` will be retained.

**LSN: Where the change occurred**

An LSN (Log Sequence Number) is a position in the `WAL`, represented in text as a value such as `16/B374D848`. Within the normal `WAL` sequence of the same cluster, it advances as `WAL` is written.

A `CDC` system encounters several LSNs at the same time:

- Where the server has currently written.
- How far the replication stream has sent.
- How far the consumer has processed.
- How far the downstream system has safely acknowledged.

Only the last of these may be reported to the slot.

**Replica identity: How many old values are visible**

After a DELETE there is no new row, and an UPDATE may need the pre-update partition key. PostgreSQL must decide which old values to retain in the `WAL`. This is controlled by the per-table `REPLICA IDENTITY` property.

| Setting | Old values available for UPDATE/DELETE | Typical use |
|---|---|---|
| `DEFAULT` | DELETE carries the old primary key; UPDATE carries the old key only if the primary key changes | Locate downstream rows by primary key |
| `USING INDEX` | DELETE carries the old index values; UPDATE carries the old key only if the indexed key changes | Tables without a primary key but with a suitable unique key |
| `FULL` | All columns | Auditing, computing differences, and cleaning up old sharded data |
| `NOTHING` | No old key is provided | Cannot satisfy replication requirements when publishing UPDATE/DELETE; the relevant operations fail |

```sql
ALTER TABLE public.accounts REPLICA IDENTITY FULL;
```

MySQL controls row images primarily through `binlog_row_image`, whereas PostgreSQL configures replica identity per table. Do not mechanically set every table to `FULL`: it provides a more complete before image, but it also increases the `WAL` volume for UPDATE and DELETE. Choose according to downstream requirements.

---

## 6. The Logical Replication Streaming Protocol

A `CDC` consumer connects to PostgreSQL using a replication connection and continuously receives data through a `COPY`-based streaming protocol.

![PostgreSQL logical replication protocol interaction](https://raw.githubusercontent.com/RifeWang/images/refs/heads/master/postgresql/05-pg-logical-replication-protocol.png)

The key protocol operations are:

- `IDENTIFY_SYSTEM`: obtains the cluster identifier, timeline, and current `WAL` position.
- `START_REPLICATION SLOT ... LOGICAL <lsn>`: starts consuming from a specified slot and LSN.
- `XLogData`: carries logically decoded data.
- Primary Keepalive: a server heartbeat that may request an immediate consumer response.
- Standby Status Update: reports the positions the consumer has received, flushed, and applied.

When using `pgoutput`, the consumer must also parse logical messages such as `Begin`, `Commit`, `Relation`, `Insert`, `Update`, `Delete`, and `Truncate`. A typical transaction can be understood as:

```text
Begin → Relation (when needed) → Insert/Update/Delete... → Commit
```

In the default mode, the logical replication stream emits transactions in commit order, and everything between `Begin` and `Commit` belongs to the same transaction. With streaming of in-progress large transactions enabled in PG 14+, one transaction may be divided into multiple `Stream Start/Stop` segments and eventually receive a `Stream Commit` or `Stream Abort`. The consumer should group them by xid and apply them only after commit has been confirmed. A production implementation should also use a mature client library to handle CopyData and the plugin protocol; it must not assume that one network read corresponds exactly to one business change.

One point requires particular clarification: a normal PostgreSQL primary/standby failover generally remains within the same database cluster, so the `systemid` does not necessarily change. During recovery, the consumer must also verify the timeline, whether the logical slot has been synchronized to the new primary, and whether the requested LSN remains valid.

---

## 7. Server Configuration and a Minimal Example

Before discussing snapshots and engineering implementation further, first establish a minimal working pipeline with a clearly defined set of example objects. All subsequent commands use the following names:

| Object | Example value |
|---|---|
| Database | `appdb` |
| Schema | `public` |
| Business table | `products` |
| Replication user | `cdc_user` |
| Publication | `cdc_pub` |
| Replication slot | `demo_slot` |
| Client network | `10.0.0.0/8` |

**Configure the PostgreSQL instance**

```conf
# postgresql.conf
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

These values must at least cover the actual number of slots and replication connections, with additional operational headroom. The instance must be restarted after changing startup parameters such as `wal_level`.

**Prepare the database, example table, and replication user**

First create the example database, then use the `psql` `\connect` command to switch to it. Every object in the following example resides in `appdb`:

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

-- The later message example needs to show the complete before image
ALTER TABLE public.products REPLICA IDENTITY FULL;

CREATE USER cdc_user
WITH REPLICATION LOGIN PASSWORD 'replace-with-a-secret';

GRANT CONNECT ON DATABASE appdb TO cdc_user;
GRANT USAGE ON SCHEMA public TO cdc_user;
GRANT SELECT ON TABLE public.products TO cdc_user;
```

The `REPLICATION` privilege is required to establish a replication connection, while the `SELECT` privilege is required to read published tables and perform a full snapshot.

A logical replication connection must specify the actual database. The `pg_hba.conf` entry should therefore allow the example database `appdb`, rather than using the special `replication` database keyword used for physical replication connections:

```conf
# TYPE  DATABASE  USER      ADDRESS       METHOD
host    appdb     cdc_user  10.0.0.0/8    scram-sha-256
```

After making the change, run `SELECT pg_reload_conf();` or reload the configuration.

**Production pipeline: Create the publication**

```sql
CREATE PUBLICATION cdc_pub FOR TABLE public.products;
```

When building a production pipeline with `pgoutput`, this command declares that changes to `public.products` should be captured. The production consumer subscribes to `cdc_pub` through the replication protocol, and the publication determines which tables and operations enter the logical replication stream.

**Local validation pipeline: Observe changes with test_decoding**

To make the changes directly readable, the following example separately uses the text output plugin `test_decoding`. This local validation pipeline does not read the `cdc_pub` publication above and is not equivalent to the production `pgoutput + Publication` pipeline.

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

The three statements above produce one INSERT, UPDATE, and DELETE change respectively. Now read the decoded contents of the slot:

```sql
SELECT lsn, xid, data
FROM pg_logical_slot_get_changes('demo_slot', NULL, NULL);
```

`pg_logical_slot_get_changes` consumes changes and advances the position. `pg_logical_slot_peek_changes` only inspects changes without consuming them, making it better suited to repeated debugging.

Promptly remove the test slot when finished:

```sql
SELECT pg_drop_replication_slot('demo_slot');
```

This example uses `test_decoding` and SQL functions to make the changes human-readable. Production systems normally use `pgoutput`, a publication, and the replication protocol for continuous consumption rather than polling the SQL functions above.

---

## 8. Handing Off from an Existing-Data Snapshot to Incremental Changes

Logical replication can provide only the incremental changes within the range retained by the slot. It cannot automatically reconstruct all data that existed in a table long before the slot was created. The initial startup of `CDC` therefore requires two tasks:

1. Read the existing data to establish a complete downstream baseline.
2. Begin consuming subsequent incremental changes from a known LSN.

The real difficulty is not performing the full scan and incremental consumption separately, but making them share the same boundary. Think of this boundary as a cut:

> The snapshot is responsible for data state before the boundary, while the logical replication stream is responsible for changes committed after the boundary.

For example, importing all rows from `public.products` in `appdb` into a search engine may take an hour, while product prices continue to change during the scan. If the incremental starting point is determined only after the scan finishes, changes committed in between may be missed. Without controlled application order, older snapshot data may also overwrite newer incremental results.

![PostgreSQL CDC snapshot-to-stream handoff](https://raw.githubusercontent.com/RifeWang/images/refs/heads/master/postgresql/06-pg-cdc-snapshot-stream-handoff.png)

PostgreSQL establishes this boundary through two corresponding values:

| Return value | Purpose |
|---|---|
| `consistent_point` | A safe starting LSN for the logical replication stream, from which changes committed after the boundary can be read |
| `snapshot_name` | An MVCC snapshot corresponding to that boundary, allowing one or more scan connections to see data from the same point in time |

**Complete handoff procedure**

1. **Establish the boundary:** Create a logical slot through the replication protocol and request an exported snapshot, obtaining `consistent_point` and `snapshot_name`.
2. **Import the snapshot:** One or more regular database connections begin read-only `REPEATABLE READ` transactions and import the same `snapshot_name` before executing any queries.
3. **Scan existing data:** The connections read `public.products` concurrently and write the snapshot data downstream. Do not apply later incremental events until all snapshot data has been acknowledged.
4. **Continue with incremental changes:** Start logical replication from `consistent_point` and read changes committed after the boundary was established.

Example for a snapshot scan connection:

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ READ ONLY;
SET TRANSACTION SNAPSHOT '00000003-0000001A-1';

SELECT id, name, price, updated_at
FROM public.products
WHERE id >= 1 AND id < 100000;

COMMIT;
```

An exported snapshot has a limited lifetime. Until every scan connection has successfully executed `SET TRANSACTION SNAPSHOT`, keep the replication connection that created the snapshot open and avoid issuing any other replication command on it. Client libraries usually encapsulate the protocol details of slot creation and snapshot export, but callers must still guarantee this ordering.

**Scanning large tables concurrently**

After every worker imports the same snapshot, scan tasks can be divided with different predicates while retaining a consistent data view. Common partitioning methods include:

- Primary-key ranges: `WHERE id >= ? AND id < ?`; simple, but affected by the primary-key distribution.
- Business partitions: scan concurrently by date, tenant, or native table partition.
- CTID page ranges: closer to physical order and suitable for large tables without a uniformly distributed primary key.

CTIDs change when operations such as `VACUUM FULL` and `CLUSTER` rewrite a table. If partitioning by CTID, avoid these operations during the snapshot and validate the scan plan against the actual PostgreSQL version and table structure.

**Application order and idempotency**

The easiest implementation to verify waits until all snapshot data has been written and acknowledged, then applies incremental changes sequentially from `consistent_point`. Price changes made during the scan remain buffered in the `WAL` retained by the slot and are applied in order after the snapshot completes.

If snapshot and incremental events are processed simultaneously to reduce latency, the implementation must ensure that an old snapshot row cannot overwrite newer incremental state—for example, by establishing a phase barrier. When concurrent application is required, `consistent_point` can serve as the snapshot baseline version, with the transaction commit LSN and an intra-transaction sequence number used to order incremental changes.

Log-based `CDC` usually provides **at-least-once** delivery, so reconnects and delivery retries can still produce duplicate events. Downstream systems should therefore UPSERT by primary key and implement idempotency using the transaction commit LSN, an intra-transaction sequence number, or an event ID. When scanning a large table takes a long time, monitor both the `WAL` retained for the slot and the effect of the long-running snapshot transaction on VACUUM and table bloat.

If the publication uses a row filter or column list, the snapshot query should apply equivalent filtering and projection so that the existing-data baseline matches the incremental scope.

MySQL uses different primitives, but the principle is the same: first obtain a consistent snapshot and its corresponding binlog position/GTID, then read existing data and continue with incremental changes.

---

## 9. The Correctness Core of an Engineering Implementation

A robust `CDC` consumer must do more than parse the protocol. It must handle acknowledgment, concurrency, ordering, and exceptional data correctly. The core problems can be summarized as follows:

| Engineering problem | Handling principle |
|---|---|
| When to acknowledge an LSN | Report a transaction's `Commit.end_lsn` only after the complete transaction has been successfully acknowledged downstream |
| Concurrent completion and safe watermark | Advance only through a contiguous sequence of completed transaction boundaries; pause reads when the in-flight window is full to create backpressure |
| Ordering changes to the same row | Use `schema.table.primary_key` as a stable routing key, with the same hashing algorithm, encoding, and seed across all instances |
| Unchanged TOAST | Preserve the “field unchanged” semantics and merge with the old value downstream; never treat it as `NULL` or an empty value |
| Large transactions | Limit in-flight data and implement backpressure; with PG 14+, combine client and plugin support for streaming the decoding of large transactions |
| Oversized individual rows | Use chunking, store content externally with Claim Check, or query the source database if timing differences are acceptable |
| DDL and schema evolution | Synchronize structural changes through Event Triggers, `pg_catalog`, schema history, or database migration events |

![Correctness loop and operational signals for production CDC](https://raw.githubusercontent.com/RifeWang/images/refs/heads/master/postgresql/07-pg-cdc-correctness-operations.png)

The most important distinction is between positions that have been “received,” “processed,” and “safely acknowledged.” The recommended advancement sequence is:

```text
Receive → Decode → Deliver → Downstream acknowledgment → Record the safe watermark (if needed) → Report the safe LSN
```

Concurrency does not change this principle. If a later-committing transaction has completed while an earlier transaction is still being retried, the safe watermark must remain before the earlier transaction. Overall delivery usually uses **at-least-once** semantics, while downstream idempotency absorbs duplicate events caused by reconnects and retries.

PostgreSQL logical replication does not propagate DDL by default. The MySQL binlog usually exposes DDL Query Events, but both systems require explicit handling of ordering and compatibility between schema events and row data.

---

## 10. Message Model Design

Snapshot and incremental events should preferably share the same structure, distinguished only by `op`. A practical event typically includes:

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

The main design considerations are:

- **Stable routing:** The key must consistently identify the same row and preserve order within a partition.
- **Comparable positions:** Include the transaction commit LSN and an intra-transaction sequence number to support troubleshooting, ordering, and deduplication.
- **Transaction traceability:** Preserve the xid and, when needed, support transaction-level aggregation.
- **Explicit old-value semantics:** Distinguish missing fields, `NULL`, and unchanged TOAST.
- **Evolvable schema:** Follow compatibility rules when using Protobuf or Avro.
- **Isomorphic snapshot and incremental events:** Downstream systems should not need two entirely different processing paths.

A more complete message format is not automatically better. A full before image, column metadata, and transaction information all increase message size and should be driven by downstream requirements.

---

## 11. Open-Source Tools and Clients

The following PostgreSQL `CDC` tools use OSI-approved licenses:

| Tool | Form/language | Primary use | Open-source license |
|---|---|---|---|
| Debezium + Kafka Connect | Java · Connector runtime | Captures changes from multiple databases and writes them to Kafka | Apache-2.0 |
| Debezium Server | Java · Standalone runtime | Sends database changes directly to multiple messaging systems | Apache-2.0 |
| Apache Flink CDC | Java · Distributed data pipeline | Ingests database snapshots and incremental changes into Flink data pipelines | Apache-2.0 |
| Apache SeaTunnel | Java · Data integration platform | Synchronizes batch and incremental data across multiple sources and destinations | Apache-2.0 |
| `xataio/pgstream` | Go · CLI/library | PostgreSQL to Kafka, OpenSearch, Webhook, or PostgreSQL | Apache-2.0 |
| `ConduitIO/conduit` | Go · Connector framework | Connects PostgreSQL to other data systems through connectors | Apache-2.0 |
| PeerDB | Go/Rust · Replication platform | PostgreSQL to analytical systems, queues, and object storage | AGPL-3.0 |
| Sequin | Elixir · Self-hosted service | PostgreSQL to queues, search engines, and Webhooks | MIT |

Most Airbyte code uses ELv2, while the current version of Materialize uses BSL 1.1. Their source code is visible, but those licenses are not OSI-approved open-source licenses, so they are not included in the open-source tool list above. PeerDB changed to AGPL-3.0 in 2026 and is open-source software, but users must comply with its stronger copyleft terms.

For learning and troubleshooting, `pg_recvlogical`, `test_decoding`, or `wal2json` can be used to inspect decoded output directly.

Common client libraries and APIs for custom implementations can also be compared directly:

| Language | Client/API | Role | Open-source license |
|---|---|---|---|
| Go | `jackc/pglogrepl` + `jackc/pgx` | Replication protocol, `pgoutput` messages, and PostgreSQL connections | MIT |
| Java | PostgreSQL JDBC `PGReplicationStream` | Logical replication interface built into the JDBC driver | BSD-2-Clause |
| Python | `psycopg2` | Logical replication interfaces such as `LogicalReplicationConnection` | LGPL-3.0-or-later |
| Rust | `supabase/etl` | Rust framework for PostgreSQL logical replication and real-time data pipelines | Apache-2.0 |
| Node.js | `kibae/pg-logical-replication` | Supports output plugins including `pgoutput` and `wal2json` | MIT |
| C/C++ | `libpq` | PostgreSQL's official low-level client library | PostgreSQL License |

---

## 12. Monitoring, Failure Recovery, and Launch Checklist

Production operations focus on three questions: whether the consumer is keeping up, how much `WAL` the slot needs to retain, and whether the system can recover from the correct position after a failure.

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

`restart_lsn` estimates the range of `WAL` the slot currently needs to retain, while `confirmed_flush_lsn` shows the consumer's acknowledgment progress. Beyond these, retain the following metric groups:

| Monitoring area | Key metrics |
|---|---|
| Slot status | `active`, retained `WAL` size, and the rate at which the confirmed LSN advances |
| End-to-end latency | P95/P99 from event commit to downstream acknowledgment |
| Consumer status | Number of in-flight events, retry count, time under backpressure, and reconnect count |
| Primary resources | Size of the `WAL` directory and remaining disk space |

Failure recovery cannot be limited to reconnecting after a dropped connection. Before resuming, verify that the consumer is still connected to the expected cluster, that the timeline and slot state remain valid, that the required `WAL` still exists, and that the local safe watermark is consistent with `confirmed_flush_lsn`. If they are inconsistent, stop consumption and raise an alert. A normal primary/standby failover does not necessarily change the `systemid`, so the slot-synchronization or failover-slot approach for the PostgreSQL version in use must be validated in advance.

The launch checklist can be summarized in six items:

- Instance parameters, replication-user privileges, and publication scope are correct.
- Each table's primary key and `REPLICA IDENTITY` satisfy downstream requirements.
- The snapshot-to-stream handoff has been validated under concurrent writes.
- The LSN advances contiguously and only after downstream acknowledgment.
- There are explicit strategies for unchanged TOAST, DDL, large transactions, and duplicate events.
- Alerts for the slot, end-to-end latency, and disk usage are configured, and procedures for primary/standby failover and abandoned-slot cleanup have been rehearsed.

---

## Summary

The core of PostgreSQL WAL CDC is turning database changes into an event stream that can be continuously consumed, acknowledged, and recovered. The `WAL` provides the source of changes, logical decoding reconstructs row-level semantics, the publication determines the capture scope, the LSN identifies log positions, the replication slot stores consumer progress, and `REPLICA IDENTITY` determines which old values UPDATE and DELETE can provide.

The initial startup of `CDC` must also handle the boundary between existing data and incremental changes: the exported snapshot covers data state before the boundary, while the logical replication stream covers changes committed after it. As long as both use the same consistent starting point, application writes do not need to be paused.

The most important engineering principle is that the safe LSN can advance only after the complete transaction has been successfully acknowledged downstream. Concurrent consumption may acknowledge only contiguous completed transaction boundaries, changes to the same row must preserve order, and duplicate events must be made idempotent through primary-key UPSERTs combined with the transaction commit LSN, an intra-transaction sequence number, or an event ID. The system also needs explicit handling for unchanged TOAST, DDL, large transactions, oversized messages, and primary/standby failover.

PostgreSQL and MySQL use similar principles for log-based `CDC`, but their log-retention mechanisms differ: MySQL must account for whether the binlog expires before a consumer resumes, whereas PostgreSQL must account for changes in `WAL` retention caused by slot progress.

Ultimately, the reliability of a `CDC` system comes down to three criteria: there are no gaps between the full snapshot and incremental stream, the acknowledged data position supports safe recovery, and the downstream state still converges correctly in the presence of duplicates, delays, and out-of-order delivery.
