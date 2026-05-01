# Scaling Database Writes

> **Goal:** Design write paths that sustain high throughput without sacrificing durability, correctness, or latency — as load grows 10x, 100x, and beyond.

---

## 1. Why Writes Are Hard to Scale

Reads can be scaled by adding replicas — 10 replicas means 10x read capacity. **Writes must go through the primary (or be coordinated)** — you can't simply add more primaries without careful design.

Write bottlenecks manifest as:
- High write latency (queue depth growing)
- Lock contention on hot rows
- WAL (Write-Ahead Log) replication lag
- Disk I/O saturation
- Connection pool exhaustion
- CPU saturation from index maintenance

---

## 2. The Write Scaling Ladder

Start with the simplest solution and climb only when needed.

```
Level 0: Single primary (good up to ~5k writes/sec with Postgres)
  └─ Optimize: Indexes, query tuning, connection pooling (PgBouncer)

Level 1: Write batching / buffering
  └─ Reduce write frequency, batch 100 ops into 1

Level 2: Vertical scaling
  └─ Bigger machine, NVMe SSDs, more RAM for write buffer

Level 3: Connection pooling + async I/O
  └─ PgBouncer, HikariCP — reduce connection overhead

Level 4: Write-ahead / WAL optimization
  └─ Tune fsync, checkpoint settings, fill factor

Level 5: Table partitioning
  └─ Range partitions reduce lock scope, enable parallel inserts

Level 6: Sharding (horizontal partitioning)
  └─ Split by customer_id, region, date — each shard has its own primary

Level 7: CQRS + Event Sourcing
  └─ Separate write model from read model entirely

Level 8: NewSQL (Spanner, CockroachDB, YugabyteDB)
  └─ Distributed ACID, unlimited horizontal write scale
```

---

## 3. Core Techniques

### Technique 1: Write Batching

Accumulate writes in memory and flush periodically instead of writing each operation individually.

**Without batching:** 10,000 separate `INSERT` statements → 10,000 round trips
**With batching:** One `INSERT` with 10,000 rows → 1 round trip

```python
# Bad: 10,000 individual inserts
for event in events:
    db.execute("INSERT INTO events (user_id, type, ts) VALUES (%s, %s, %s)",
               [event.user_id, event.type, event.ts])

# Good: Batch insert
db.executemany(
    "INSERT INTO events (user_id, type, ts) VALUES (%s, %s, %s)",
    [(e.user_id, e.type, e.ts) for e in events]
)

# Best: COPY (PostgreSQL bulk loader — 10-100x faster than INSERT)
with db.cursor() as cur:
    cur.copy_from(
        io.StringIO("\n".join(f"{e.user_id}\t{e.type}\t{e.ts}" for e in events)),
        "events",
        columns=("user_id", "type", "ts")
    )
```

**Performance:**
- Single row inserts: ~1,000 rows/sec (network-bound)
- Batch insert (1000 rows): ~50,000 rows/sec
- COPY: ~200,000-500,000 rows/sec

---

### Technique 2: Write-Behind Cache (Async Write)

Accept writes into a fast in-memory store (Redis) and asynchronously flush to the database.

```
Client ──► API ──► Redis (sync, <1ms) ──► Return 200 OK
                       │
                  Background Worker (every 500ms)
                       │
                   Batch flush to DB
```

```python
# Accept write — ultra fast
def record_page_view(user_id, page_id):
    redis.lpush("page_view_buffer", f"{user_id}:{page_id}:{time.time()}")
    return  # Return immediately, don't wait for DB

# Background flusher
def flush_page_views():
    while True:
        events = []
        # Pop up to 10,000 events atomically
        pipeline = redis.pipeline()
        for _ in range(10000):
            pipeline.lpop("page_view_buffer")
        events = [e for e in pipeline.execute() if e]
        
        if events:
            db.executemany(
                "INSERT INTO page_views (user_id, page_id, ts) VALUES (%s, %s, %s)",
                [e.split(":") for e in events]
            )
        time.sleep(0.5)
```

**Risk:** Redis crash before flush = data loss. Mitigate with Redis AOF persistence or replicated Redis.

**Real-world use:** YouTube view counts, Facebook likes, Twitter tweet counters — all use write-behind.

---

### Technique 3: Event Sourcing / Append-Only Log

Instead of updating rows in place, append every change as an immutable event. Current state is derived by replaying events.

```sql
-- Traditional approach (mutable state)
UPDATE accounts SET balance = balance - 100 WHERE id = 42;

-- Event sourcing (immutable log)
INSERT INTO account_events (account_id, type, amount, ts)
VALUES (42, 'DEBIT', 100, now());

-- Balance = SUM of events (or derived in a materialized view)
SELECT SUM(CASE WHEN type='CREDIT' THEN amount ELSE -amount END)
FROM account_events WHERE account_id = 42;
```

**Write benefits:**
- Append-only writes are the fastest possible DB operation (sequential, no index updates except B-tree append)
- No lock contention — writes never conflict (they're always new rows)
- Natural audit trail

**Read challenge:** Reconstructing current state requires aggregating all events. Solve with **snapshots** (periodic materialized views of current state) and **CQRS** (separate read store).

**Real-world use:** Banking systems, Kafka (log is the source of truth), Git (commit history), blockchain.

---

### Technique 4: Sharding (Horizontal Partitioning)

Split the dataset across multiple database instances. Each shard handles writes for a subset of data.

```
Writes for user_id % 4 = 0 → Shard 0
Writes for user_id % 4 = 1 → Shard 1
Writes for user_id % 4 = 2 → Shard 2
Writes for user_id % 4 = 3 → Shard 3
```

**Sharding strategies:**

| Strategy | Key | Pros | Cons |
|----------|-----|------|------|
| Hash-based | `user_id % N` | Even distribution | Re-sharding is painful |
| Range-based | `created_at` year | Easy time-range queries | Hot partitions (new data all on newest shard) |
| Directory-based | Lookup table | Flexible, re-shard without key change | Lookup table is a bottleneck |
| Geo-based | Region (US/EU/APAC) | Data residency, low latency | Uneven load if regions differ |

**Application-level sharding:**
```python
def get_shard(user_id):
    return shard_connections[user_id % NUM_SHARDS]

def write_user_event(user_id, event):
    db = get_shard(user_id)
    db.execute("INSERT INTO events ...", [user_id, event])
```

**Pitfall: Cross-shard queries**
```sql
-- This requires querying ALL shards and merging results:
SELECT * FROM events WHERE type = 'purchase' AND amount > 1000;
-- On each shard, then merge in application layer
```

For cross-shard analytics, maintain a separate analytics store (e.g., Redshift, BigQuery, ClickHouse) fed via CDC.

> 📖 **Case Study:** [Vitess — Sharding MySQL at YouTube/GitHub](https://vitess.io/docs/overview/)

---

### Technique 5: CQRS (Command Query Responsibility Segregation)

Separate the write model (command side) from the read model (query side).

```
Client                Write Side            Read Side
  │                       │                     │
  ├─ POST /orders ────►  Orders DB (Postgres)   │
  │                       │                     │
  │                       │ emit OrderCreated   │
  │                       └─── Kafka ──────────►│
  │                                             │ Update read
  │                                             │ model (Elastic,
  │                                             │ Redis, Mongo)
  │                                             │
  └─ GET /orders ──────────────────────────────►│
                                          Return denormalized
                                          view (fast)
```

**Benefits:**
- Write DB is optimized for transactions (normalized, indexed for writes)
- Read DB is optimized for queries (denormalized, indexed for reads)
- Scale them independently — 100 read replicas, 1 write primary

**Complexity:** You must manage eventual consistency between write and read models.

---

### Technique 6: Write-Optimized Data Structures

**LSM Tree (Log-Structured Merge Tree)** — used by Cassandra, RocksDB, LevelDB, HBase  
Sequential writes to an in-memory buffer (MemTable), flushed to immutable SSTable files. Compaction merges SSTables periodically.

```
Write → MemTable (memory) → flush when full → SSTable (disk)
                                                    │
                                             Background compaction
                                             (merge SSTables, remove tombstones)
```

- **Write amplification:** Lower than B-trees for random writes
- **Read amplification:** Higher than B-trees (must check multiple SSTables)
- **Best for:** Write-heavy workloads, time-series, event logs

**B-Tree (used by Postgres, MySQL InnoDB)**
- Better for mixed read/write, random access, range queries
- Higher write amplification (must update tree structure on each write)

---

### Technique 7: Database Connection Pooling

Every DB connection is expensive (~5MB RAM, ~10ms setup). Without pooling, high-write services exhaust connections.

```
Applications (100 pods × 10 threads = 1000 connections) 
     │
     ▼
PgBouncer (connection pooler)
  - Transaction mode: shares connections within a transaction
  - Maps 1000 app connections → 50 actual DB connections
     │
     ▼
PostgreSQL Primary (max_connections = 200)
```

```ini
# pgbouncer.ini
[databases]
mydb = host=postgres port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction  # or session, statement
max_client_conn = 10000  # application connections
default_pool_size = 50   # actual DB connections per database
```

**Without PgBouncer:** 1000 Rails connections → Postgres OOM
**With PgBouncer:** 1000 Rails connections → 50 Postgres connections, works fine

---

## 4. Regular Scenarios

### Scenario 1: Analytics Event Ingestion (10M events/day)
**Requirement:** High-volume write-only stream. No reads during ingestion.

**Architecture:**
1. **API servers** accept events, write to **Kafka** (durable, fast, at-least-once)
2. **Kafka consumers** batch 10,000 events and bulk-insert into **ClickHouse** or **BigQuery**
3. No transactional DB involved in the hot path

```python
# Producer (API)
kafka.produce("events", key=user_id, value=serialize(event))
return {"status": "accepted"}, 202  # Don't wait for DB write

# Consumer (batch inserter)
batch = []
for message in kafka.consume("events", batch_size=10000, timeout_ms=500):
    batch.append(deserialize(message.value))
    
if batch:
    clickhouse.insert("events", batch)  # Bulk insert
    kafka.commit()
```

**Throughput:** ClickHouse handles 1M+ inserts/sec with batching. No sharding needed for most analytics use cases.

---

### Scenario 2: User Activity Logging (Social Platform)
**Requirement:** Every user action (like, view, click, scroll) must be recorded. 500k events/sec peak.

**Architecture:**
- **Redis Streams** as write buffer (in-memory, fast, durable with AOF)
- **Background consumers** flush to Cassandra in batches
- Cassandra's LSM tree handles high write throughput natively

```python
# Write: 500k events/sec in Redis Stream
redis.xadd("activity_stream", {"user": user_id, "action": action, "ts": ts})

# Consumer: flush to Cassandra every second
events = redis.xreadgroup("activity-consumers", "worker-1", "activity_stream", ">", count=50000)
cassandra.execute_concurrent([
    ("INSERT INTO activity (user_id, action, ts) VALUES (?, ?, ?)", (e["user"], e["action"], e["ts"]))
    for e in events
])
redis.xack("activity_stream", "activity-consumers", *[e.id for e in events])
```

---

### Scenario 3: Financial Ledger (High Correctness Requirements)
**Requirement:** Every cent must be accounted for. No writes can be lost. Throughput: 10k TPS.

**Architecture:**
- **PostgreSQL** for ledger entries (ACID, append-only design)
- **PgBouncer** for connection pooling
- **Partitioning** by month (each month = one partition, reduces index size)
- **Async audit log** via Postgres trigger → Kafka (for compliance reporting)

```sql
-- Partitioned ledger
CREATE TABLE ledger (
    id          BIGSERIAL,
    account_id  BIGINT NOT NULL,
    amount      DECIMAL(18, 6) NOT NULL,
    type        TEXT NOT NULL,  -- DEBIT / CREDIT
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE ledger_2025_01 PARTITION OF ledger
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
-- Auto-create monthly partitions via pg_partman
```

---

## 5. Tricky Scenarios

### Tricky Scenario 1: Write Hot Spot — Sequential Auto-Increment IDs

**Problem:** With `id BIGSERIAL PRIMARY KEY`, all inserts go to the rightmost B-tree leaf page. Under high concurrency, this page becomes a hot spot (InnoDB: "index page latch contention").

**Solutions:**

Option A: **UUID v4** (random, spreads inserts across B-tree)
```sql
id UUID DEFAULT gen_random_uuid() PRIMARY KEY
```
*Downside:* UUIDs hurt B-tree performance due to fragmentation. Table size grows 2x.

Option B: **UUID v7** (time-ordered UUIDs — sequential enough to avoid fragmentation, but globally unique)
```sql
-- UUID v7: first 48 bits = millisecond timestamp
id UUID DEFAULT gen_uuid_v7() PRIMARY KEY  -- PostgreSQL 17+
```

Option C: **Snowflake IDs** (Twitter's approach — 64-bit integer, time-ordered, distributed)
```
64-bit Snowflake:
  [41 bits: timestamp ms] [10 bits: machine_id] [12 bits: sequence]
  → Time-ordered, globally unique, integer (fast B-tree)
  → Instagram, Discord, Twitter all use this
```

---

### Tricky Scenario 2: Schema Migration on a High-Write Table

**Problem:** Adding a column to a table with 500M rows. `ALTER TABLE` takes an exclusive lock for minutes, blocking all writes.

**Solution: Online Schema Change (OSC)**

1. **pt-online-schema-change** (Percona, MySQL) — creates shadow table, backfills, atomically renames
2. **pg_repack** (PostgreSQL) — rebuilds table online
3. **gh-ost** (GitHub, MySQL) — uses binlog to apply changes without locking

**Manual approach for Postgres:**
```sql
-- Step 1: Add nullable column (fast, no rewrite)
ALTER TABLE orders ADD COLUMN v2_status TEXT;  -- instant

-- Step 2: Backfill in batches (no lock)
UPDATE orders SET v2_status = status WHERE id BETWEEN 1 AND 100000;
-- ... repeat for all batches

-- Step 3: Add NOT NULL constraint using a check constraint (fast in PG 14+)
ALTER TABLE orders ADD CONSTRAINT orders_v2_status_nn 
    CHECK (v2_status IS NOT NULL) NOT VALID;  -- instant, no scan
ALTER TABLE orders VALIDATE CONSTRAINT orders_v2_status_nn;  -- concurrent scan

-- Step 4: Drop old column
ALTER TABLE orders DROP COLUMN status;
```

> 📖 **Case Study:** [GitHub: gh-ost — Online Schema Changes at GitHub](https://github.blog/engineering/infrastructure/gh-ost-github-s-online-schema-migrations-for-mysql/)

---

### Tricky Scenario 3: Write Amplification in Cassandra / Wide-Column
**Problem:** Every write to Cassandra also writes to the commit log, MemTable, and eventually SSTable. Compaction rewrites data repeatedly.

**Write Amplification Factor (WAF):** A single logical write may cause 10-30x the actual I/O.

**Mitigations:**
- Use **TWCS (TimeWindowCompactionStrategy)** for time-series data — groups SSTables by time window, reduces compaction I/O
- **Larger batch writes** to reduce commit log overhead per row
- **Avoid wide rows** (rows with thousands of columns) — breaks up into smaller column families

---

### Tricky Scenario 4: Multi-Region Write Conflicts
**Problem:** Users in US and EU can both update the same record (e.g., their profile). Both datacenters accept writes (active-active). How to reconcile conflicts?

**Strategies:**

| Strategy | How | Use When |
|----------|-----|----------|
| Last Write Wins (LWW) | Take the write with the latest timestamp | Non-critical updates (profile bio) |
| CRDT (Conflict-free Replicated Data Types) | Merge both values mathematically | Counters, sets, shopping carts |
| Operational Transform | Merge edits character by character | Collaborative text editing (Google Docs) |
| Application-level merge | Route writes to single region per user | Financial data, where LWW is unacceptable |

**CRDTs in practice:**
- Shopping cart: use OR-Set CRDT — adding items from two regions both survive, deletion is tracked
- Like counter: G-Counter CRDT — each node has its own counter, global sum = total likes
- Redis CRDT (Redis Enterprise): built-in CRDT types for multi-region

> 📖 **Reference:** [Riak: Why CRDTs are The Future](https://riak.com/posts/technical/why-crdts-are-the-future/index.html)

---

### Tricky Scenario 5: Thundering Herd on Write After Outage
**Problem:** DB is down for 30 minutes. During that time, 1M writes queue up in Kafka. When DB comes back, all 1M writes hit simultaneously → DB immediately overwhelmed again.

**Solution: Controlled replay with back-pressure**
```python
# Consumer with adaptive rate limiting
def replay_from_kafka():
    rate_limiter = AdaptiveRateLimiter(initial_rate=1000)  # writes/sec
    
    for message in kafka.consume():
        while True:
            try:
                db.write(message)
                rate_limiter.on_success()
                break
            except DBOverloadError:
                rate_limiter.on_failure()  # Reduce rate
                time.sleep(rate_limiter.backoff_seconds())
```

Also: configure Kafka consumer lag alerting. When lag spikes, alert — don't just let it pile up silently.

---

## 6. Write Scaling Techniques Summary

| Technique | Throughput Gain | Complexity | Durability Risk |
|-----------|----------------|------------|-----------------|
| Batch inserts | 10-100x | Low | None |
| Write-behind cache | 100x | Medium | Medium (Redis crash) |
| Connection pooling | 5-10x | Low | None |
| Append-only / event sourcing | 3-5x | High | None |
| Sharding | Nx (per shard) | High | None |
| CQRS | 10-50x (reads free writes) | Very High | Low |
| LSM-tree DB (Cassandra/RocksDB) | 5-20x over B-tree | Medium | Low |
| NewSQL (CockroachDB) | Nx | Medium | None |

---

## 7. Reference Case Studies

1. **Discord: Switching from Cassandra to ScyllaDB (write performance)**  
   [https://discord.com/blog/how-discord-stores-trillions-of-messages](https://discord.com/blog/how-discord-stores-trillions-of-messages)

2. **Shopify: Scaling MySQL for Black Friday**  
   [https://engineering.shopify.com/blogs/engineering/mysql-at-shopify](https://engineering.shopify.com/blogs/engineering/mysql-at-shopify)

3. **GitHub: gh-ost Online Schema Migrations**  
   [https://github.blog/engineering/infrastructure/gh-ost-github-s-online-schema-migrations-for-mysql/](https://github.blog/engineering/infrastructure/gh-ost-github-s-online-schema-migrations-for-mysql/)

4. **Instagram: Sharding IDs at Scale**  
   [https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c](https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c)

5. **Vitess: Scaling MySQL Horizontally**  
   [https://vitess.io/docs/overview/](https://vitess.io/docs/overview/)

6. **Cloudflare: How We Store Time-Series Data at Petabyte Scale**  
   [https://blog.cloudflare.com/http-analytics-for-6m-requests-per-second-using-clickhouse/](https://blog.cloudflare.com/http-analytics-for-6m-requests-per-second-using-clickhouse/)

7. **Cassandra at Netflix: Handling 1M Writes/Second**  
   [https://netflixtechblog.com/revisiting-1-million-writes-per-second-c191a84864cc](https://netflixtechblog.com/revisiting-1-million-writes-per-second-c191a84864cc)
