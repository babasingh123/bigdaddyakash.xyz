# SQL Databases (RDBMS)

A SQL database — or *relational* database — organizes data into tables of rows and columns with strict schemas, expresses relationships via foreign keys, and exposes a declarative query language (SQL) backed by an optimizer that picks an execution plan for you. The four headline guarantees are **ACID**: atomicity, consistency, isolation, durability.

## Why it exists

Before relational databases (Codd, 1970), applications encoded relationships into navigational links — pointers, hierarchies, or hand-rolled file formats. Every query required a programmer who understood the physical layout, and changing the schema meant rewriting the application. The relational model decoupled the *logical* data model from the *physical* storage. You describe **what** you want (`SELECT name FROM users WHERE age > 30`) and let the database figure out **how**.

The deeper reason SQL has lasted 50+ years is **ACID**. When you transfer money from account A to account B, you cannot tolerate the system crashing mid-transfer and leaving the money in neither account. Relational databases were the first systems to make this kind of correctness easy and the default. That guarantee, combined with a flexible query language and a mature ecosystem, is why SQL is still the right starting point for most designs.

!!! tip "When in doubt, start with Postgres"
    A common interview anti-pattern is reaching for NoSQL too fast. Unless you have a clear scale or access-pattern reason to leave SQL, start with Postgres or MySQL. They scale further than people assume.

## How it works

### ACID properties

Every relational database guarantees the same four properties at the transaction boundary:

- **Atomicity** — a transaction is all-or-nothing. If any statement in `BEGIN ... COMMIT` fails, the database rolls back every change made since `BEGIN`. The classic example is debiting one account and crediting another: both happen or neither does.
- **Consistency** — committed transactions leave the database in a valid state with respect to all declared constraints (foreign keys, unique indexes, check constraints, triggers). The application doesn't get to commit nonsense.
- **Isolation** — concurrent transactions don't see each other's intermediate state. The isolation *level* controls how strict this is (see below).
- **Durability** — once a transaction returns "committed", the data survives crashes. This is typically implemented via a write-ahead log (WAL) flushed to disk before acknowledgement.

!!! note "When ACID matters vs. when you can relax"
    ACID matters for **money, inventory, orders, identity** — anywhere a duplicate or lost write is a real-world bug. ACID is overkill for **likes, view counts, social feeds, analytics** — eventual consistency is fine and you can pick a faster store.

### Indexing

An index is a separate data structure that lets the database find rows without scanning the whole table. The trade-off is universal: indexes speed up reads and slow down writes (every insert/update/delete must also update the index).

| Index type | Structure | Best for |
|------------|-----------|----------|
| **B-tree** | Balanced sorted tree | Range queries, ordering, equality. The default. |
| **Hash** | Hash table | Equality only (`WHERE id = 42`). Faster than B-tree for that case, can't do `>` or `BETWEEN`. |
| **Composite** | B-tree on multiple columns | Queries that filter on several columns together. Column order matters. |
| **Covering** | Index contains all queried columns | The query plan never touches the heap — pure index scan. |

A *covering index* is worth highlighting: if your query is `SELECT name, email FROM users WHERE country = 'US'`, an index on `(country, name, email)` lets the database answer entirely from the index without dereferencing the row. That can be an order-of-magnitude speedup.

!!! tip "Learn EXPLAIN"
    Almost every SQL performance problem ends with someone running `EXPLAIN` (or `EXPLAIN ANALYZE` in Postgres) and discovering a sequential scan that should be an index seek. It's the single highest-leverage skill on this page.

### Normalization

Normalization is the process of removing redundancy by splitting wide tables into narrower related tables. The named "normal forms" are progressively stricter:

- **1NF** — every column holds a single, atomic value (no comma-separated lists, no nested structures).
- **2NF** — every non-key column depends on the *whole* primary key (no partial dependencies).
- **3NF** — every non-key column depends only on the primary key, not on other non-key columns (no transitive dependencies).

In practice, 3NF is the target for OLTP systems. Reporting and read-heavy systems often deliberately **denormalize** — copy data into wider tables — to avoid expensive joins. The trade-off is always the same: normalized schemas are easier to keep consistent on writes; denormalized schemas are faster to read.

### Joins

Joins combine rows from two or more tables based on a related column.

```sql
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON o.user_id = u.id
WHERE o.created_at > NOW() - INTERVAL '7 days';
```

| Join type | Returns |
|-----------|---------|
| `INNER JOIN` | Rows matching in **both** tables |
| `LEFT JOIN` | All rows from left, NULLs for unmatched right |
| `RIGHT JOIN` | All rows from right, NULLs for unmatched left |
| `FULL OUTER JOIN` | All rows from both, NULLs where no match |

Joins are cheap on small tables and expensive on large ones — especially when no index covers the join key, or when the tables live on different shards. **Cross-shard joins are something to avoid by design**: either co-locate the data, denormalize, or do the join in the application layer.

### Transactions and locking

Within a transaction, the database uses **locks** to keep concurrent transactions from corrupting each other:

- **Row-level locks** — only the affected rows are locked. The default in modern systems. Allows high concurrency.
- **Table-level locks** — the whole table is locked. Used for DDL (schema changes) or coarse operations.
- **Pessimistic locking** — `SELECT ... FOR UPDATE` grabs a lock immediately and holds it until commit. Safe but blocks others.
- **Optimistic locking** — read the row with a version number, do work, update only if the version is unchanged. Cheap unless conflicts are common.

When two transactions hold locks the other one needs, you get a **deadlock**. Modern engines detect cycles in the wait-for graph and abort one transaction with a deadlock error — your application code needs to retry.

#### Isolation levels

The SQL standard defines four isolation levels, from weakest to strongest:

| Level | Dirty read | Non-repeatable read | Phantom read |
|-------|-----------|---------------------|--------------|
| READ UNCOMMITTED | possible | possible | possible |
| READ COMMITTED (Postgres default) | prevented | possible | possible |
| REPEATABLE READ (MySQL default) | prevented | prevented | possible |
| SERIALIZABLE | prevented | prevented | prevented |

Stronger isolation means more locks and lower throughput. For most workloads, **READ COMMITTED** is the right default. Use SERIALIZABLE when you're doing something where phantom reads would cause real bugs (e.g. allocating sequential IDs without a sequence).

## When to use

- You have **complex relationships** between entities (users → orders → line items → products → categories).
- You need **ACID guarantees** — financial data, orders, inventory, anything where a partial write is a bug.
- Your data is **structured and the schema is stable** enough that a strict schema is an asset, not a burden.
- You'll need **ad-hoc analytical queries** that you can't predict at design time. SQL's expressiveness shines here.
- Your scale fits in a **single large box plus read replicas** (which is a *lot* — TB of data, tens of thousands of QPS).
- You want a mature ecosystem: connection pooling, ORMs, BI tools, monitoring, backup tooling — all assume SQL.

## Trade-offs

What you give up by choosing SQL:

- **Horizontal write scaling is hard.** A single Postgres primary handles writes; you can add read replicas easily, but to scale writes you have to shard (Vitess, Citus) or denormalize into a NoSQL store.
- **Strict schemas are friction.** Adding a column on a multi-TB table can lock the table or require careful online migration tools (gh-ost, pt-online-schema-change).
- **Joins on large tables are expensive**, especially if they don't fit in memory.
- **Object-relational impedance mismatch** — going from rows-and-columns to object graphs in the application is a known source of bugs and overhead (ORMs help, sometimes).

Alternatives to reach for when SQL is the wrong shape:

- Key-value cache for **session storage / hot reads** → Redis.
- Document store for **schemaless, hierarchical data** → MongoDB. See [NoSQL](nosql.md).
- Wide-column store for **time-series / write-heavy** → Cassandra.
- Search engine for **full-text / typo-tolerant search** → Elasticsearch. See [Search & Indexing](../search-indexing.md).
- Graph DB for **deeply connected relationship traversal** → Neo4j.

**Tools**: PostgreSQL, MySQL, Oracle, SQL Server, CockroachDB (NewSQL), Spanner.

## Linked problems

SQL is the foundation of the majority of HLD problems. A few where it's the *primary* store or where SQL-vs-NoSQL is the central decision:

- [URL Shortener](../../problems/url-shortener.md) — `urls` table with a B-tree index on the short code; the canonical "should this be SQL or NoSQL?" debate.
- [Pastebin](../../problems/pastebin.md) — small pastes in SQL, large blobs in S3, indexed `expires_at` for cleanup.
- [Instagram](../../problems/instagram.md) — `users`, `photos`, `follows`, `likes` tables, with sharding by `user_id`.
- [Dropbox](../../problems/dropbox.md) — file metadata in MySQL/Postgres sharded by `user_id`, content chunks in S3.
- [Uber](../../problems/uber.md) — payment flows demand ACID and the saga pattern across services.
