# Database Contention: Detection and Resolution

> **What this doc covers:** A practical guide to identifying, diagnosing, and eliminating database contention — the silent killer of write throughput. Covers every contention type (row locks, hot rows, deadlocks, phantom reads), provides SQL queries to detect each in production, and maps seven battle-tested patterns to the exact scenarios where they apply. Includes real-world flash sale and banking examples, and the tricky concurrency bugs that hide behind standard isolation levels.

---

### 📋 Outline

| # | Section | What You'll Learn |
|---|---------|-------------------|
| 1 | What Is Database Contention? | Root causes: lock waiting, deadlocks, serialization failures, hot-spot degradation |
| 2 | Types of Contention | Row-level, table-level, index contention, hot row/counter, deadlock — each explained with mechanics |
| 3 | Detection | SQL queries for live lock inspection in Postgres and MySQL; application-layer signals |
| 4 | Core Patterns to Handle Contention | Optimistic locking, pessimistic locking, counter sharding, queue-based serialization, SKIP LOCKED, idempotency keys — with full code examples |
| 5 | Regular Scenarios | Banking transfers, flash sale inventory, viral post comment counts |
| 6 | Tricky Scenarios | Phantom reads/write skew, distributed deadlocks, Redis read-modify-write races, bulk update lock escalation |
| 7 | Contention Patterns Summary Table | Which pattern to apply for which contention type |
| 8 | Reference Case Studies | Stripe, Shopify, GitLab, PostgreSQL docs |

---

## 1. What Is Database Contention?

Contention occurs when multiple transactions or queries compete for the **same resource** — a row, a page, an index node, or a lock — at the same time. The result is one of:

- **Lock waiting:** A transaction blocks until another releases a lock
- **Deadlocks:** Two transactions each hold a lock the other needs → both abort
- **Serialization failures:** Optimistic reads detect a conflict at commit time → rollback
- **Hot spot degradation:** A single row/page absorbs disproportionate write load (e.g., a global counter)

Contention is the dominant bottleneck in *write-heavy, concurrent* systems long before raw hardware becomes a limit.

---

## 2. Types of Contention

### 2.1 Row-Level Lock Contention
Most common. Two transactions try to `UPDATE` the same row simultaneously. One blocks (or one is chosen as deadlock victim).

### 2.2 Table-Level Lock Contention
`ALTER TABLE`, `LOCK TABLE`, or MyISAM (non-InnoDB) operations. Can block all readers and writers.

### 2.3 Index Contention
Inserts into a sequential (auto-increment) primary key all hammer the *rightmost* B-tree leaf page — called **index page latch contention** or "right-edge contention."

### 2.4 Hot Row / Counter Contention
A single row represents a global counter, inventory count, or balance. Every concurrent operation must update it → serialized throughput.

### 2.5 Deadlock
Circular dependency between transactions. Classic pattern: T1 locks row A then wants row B; T2 locks row B then wants row A.

---

## 3. Detection

### In PostgreSQL
```sql
-- Active locks and what's waiting
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL;

-- Lock details
SELECT l.pid, l.granted, l.mode, c.relname
FROM pg_locks l
JOIN pg_class c ON l.relation = c.oid
WHERE NOT l.granted;

-- Deadlock logs
-- Set log_lock_waits = on, deadlock_timeout = 1s in postgresql.conf
```

### In MySQL / InnoDB
```sql
SHOW ENGINE INNODB STATUS\G
-- Look for LATEST DEADLOCK section
-- Check TRANSACTIONS section for lock wait chain
```

### In Application Code
Look for:
- `LockTimeoutException`, `SerializationFailure` (Postgres error code 40001), `Deadlock found` (MySQL 1213)
- p99 latency spikes on write endpoints without CPU/memory pressure
- Retry storms — multiple services retrying the same failed transaction

---

## 4. Core Patterns to Handle Contention

### Pattern 1: Optimistic Locking (Version Field / CAS)

**When to use:** Read-heavy, write-occasional. Conflicts are rare.  
**Mechanism:** Add a `version` column. On update, check the version hasn't changed since you read it.

```sql
-- Read
SELECT id, balance, version FROM accounts WHERE id = 42;
-- balance = 1000, version = 7

-- Update (conditional on version)
UPDATE accounts
SET balance = balance - 100, version = version + 1
WHERE id = 42 AND version = 7;

-- Check rows affected: if 0, someone else updated → retry
```

**Application layer:**
```python
for attempt in range(MAX_RETRIES):
    row = db.fetchone("SELECT id, balance, version FROM accounts WHERE id=%s", [user_id])
    rows_updated = db.execute(
        "UPDATE accounts SET balance=%s, version=%s WHERE id=%s AND version=%s",
        [row.balance - amount, row.version + 1, user_id, row.version]
    )
    if rows_updated == 1:
        break  # success
    time.sleep(exponential_backoff(attempt))
else:
    raise OptimisticLockException("Too many conflicts, giving up")
```

**Real-world use:** Hibernate ORM uses this by default (`@Version`). Elasticsearch uses `_seq_no` and `_primary_term`. DynamoDB `ConditionExpression`.

---

### Pattern 2: Pessimistic Locking (`SELECT FOR UPDATE`)

**When to use:** Write-heavy, conflicts are frequent, you need to guarantee no lost updates.  
**Mechanism:** Explicitly lock the row at read time. Other transactions block until you commit or rollback.

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 42 FOR UPDATE;
-- Now no other transaction can update this row until we commit
UPDATE accounts SET balance = balance - 100 WHERE id = 42;
COMMIT;
```

**Lock hints in Postgres:**
```sql
SELECT ... FOR UPDATE NOWAIT;        -- Fail immediately if locked (don't wait)
SELECT ... FOR UPDATE SKIP LOCKED;  -- Skip locked rows (useful for job queues)
```

**Real-world use:** Payment systems (bank transfers), inventory reservation, ticket booking.

---

### Pattern 3: Counter Sharding (Distributed Counters)

**When to use:** A single column is updated by thousands of concurrent writes (e.g., `likes_count`, `inventory`, `total_views`).

**Mechanism:** Split the counter into N shards. Each write goes to a random shard. Reads aggregate all shards.

```sql
-- Schema
CREATE TABLE counter_shards (
    counter_name TEXT,
    shard_id     INT,
    value        BIGINT,
    PRIMARY KEY (counter_name, shard_id)
);

-- Write: pick a random shard
UPDATE counter_shards
SET value = value + 1
WHERE counter_name = 'post_views' AND shard_id = floor(random() * 100);

-- Read: aggregate
SELECT SUM(value) FROM counter_shards WHERE counter_name = 'post_views';
```

**Tradeoff:** Reads become expensive (aggregate N rows). Use Redis `INCR` for real-time counters and sync to DB periodically if you need persistence.

> 📖 **Case Study:** Facebook's like counter uses a similar approach. Instagram stores likes in Cassandra counters partitioned by post ID.

---

### Pattern 4: Queue-Based Serialization

**When to use:** You need strict serial execution on a resource (e.g., a user's wallet), but want throughput for different resources.

**Mechanism:** Route all writes for a given entity to a single-partition queue. A consumer processes them serially per entity, but different entities are processed in parallel.

```
Write Request (user_id=42) → Kafka partition(42 % 32) → Consumer → DB write
Write Request (user_id=43) → Kafka partition(43 % 32) → Consumer → DB write
```

Each partition is a FIFO queue. All updates for the same user go to the same partition → no concurrent writes to the same row.

**Application:** Stripe, PayPal use per-account event queues for ledger operations. No lock contention because DB writes are serialized per account.

---

### Pattern 5: Avoid Long Transactions

Long-held locks are the root cause of most contention.

**Bad pattern:**
```python
BEGIN;
user = db.fetch("SELECT * FROM users WHERE id=%s FOR UPDATE", [uid])
# --- network call to external service (200ms) ---
result = call_payment_api(user)
# --- another DB query ---
db.execute("UPDATE users SET status=%s WHERE id=%s", [result, uid])
COMMIT;  # Lock held for 200ms+
```

**Fix: Minimize lock scope**
```python
# Do the slow work first
result = call_payment_api(pre_fetched_data)  # outside transaction

# Only lock for the actual write
BEGIN;
db.execute("UPDATE users SET status=%s WHERE id=%s", [result, uid])
COMMIT;  # Lock held for <5ms
```

**Rule:** Never do I/O (HTTP calls, sleeps, user input) inside a transaction.

---

### Pattern 6: SKIP LOCKED for Job Queues

Classic problem: multiple workers pull from the same `jobs` table. `SELECT FOR UPDATE` causes them to queue up.

```sql
-- Worker query (Postgres)
BEGIN;
SELECT id, payload FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;  -- Skip rows locked by other workers

UPDATE jobs SET status = 'processing', worker_id = $1 WHERE id = $2;
COMMIT;
```

`SKIP LOCKED` means each worker instantly gets a different row — no blocking. This is how pg-boss, Sidekiq (with Postgres), and other job queue libraries work.

---

### Pattern 7: Idempotency Keys

Prevent duplicate writes on retry (common when transactions are retried due to serialization failures).

```sql
CREATE TABLE payments (
    idempotency_key TEXT UNIQUE,
    amount          DECIMAL,
    status          TEXT,
    created_at      TIMESTAMPTZ DEFAULT now()
);

INSERT INTO payments (idempotency_key, amount, status)
VALUES ('client-generated-uuid-xyz', 100.00, 'completed')
ON CONFLICT (idempotency_key) DO NOTHING;
-- Second attempt with same key → no-op, returns existing row
```

Stripe uses idempotency keys on every payment API call. The key is stored in Redis with a TTL of 24 hours.

---

## 5. Regular Scenarios

### Scenario 1: Banking Transfer (Classic)
Two users transfer money simultaneously. Classic deadlock risk.

**Unsafe:**
```sql
-- T1: Alice → Bob
UPDATE accounts SET balance = balance - 100 WHERE id = 'alice';
UPDATE accounts SET balance = balance + 100 WHERE id = 'bob';

-- T2: Bob → Alice (concurrent)
UPDATE accounts SET balance = balance - 50 WHERE id = 'bob';
UPDATE accounts SET balance = balance + 50 WHERE id = 'alice';
-- T1 locks alice, T2 locks bob → DEADLOCK
```

**Safe — Always lock in a consistent order:**
```sql
-- Both transactions lock accounts in ascending ID order
BEGIN;
SELECT * FROM accounts WHERE id IN ('alice', 'bob') ORDER BY id FOR UPDATE;
-- Now update in any order — deadlock impossible because lock order is deterministic
UPDATE accounts SET balance = balance - 100 WHERE id = 'alice';
UPDATE accounts SET balance = balance + 100 WHERE id = 'bob';
COMMIT;
```

**Lesson:** Lock multiple rows in a globally consistent order to eliminate deadlocks.

---

### Scenario 2: Flash Sale Inventory
1 million users hit "Buy" simultaneously for 100 items.

**Bad approach:** `UPDATE inventory SET count = count - 1 WHERE product_id = 1`  
→ All 1M transactions serialize on a single row. Throughput: maybe 500 TPS.

**Better approach:**
```sql
-- Use a guarded update to prevent oversell
UPDATE inventory
SET count = count - 1
WHERE product_id = 1 AND count > 0;

-- Check rows_affected == 1; if 0, out of stock
```

**Best approach for extreme load:** Pre-allocate "slots" in Redis.
```python
available = redis.decr("inventory:product:1")
if available < 0:
    redis.incr("inventory:product:1")  # compensate
    raise OutOfStockError()

# Write sale record to DB asynchronously via queue
queue.publish("sale_event", {"product": 1, "user": user_id})
```

Redis `DECR` is atomic, serialized, and handles 100k ops/sec on a single node.

> 📖 **Case Study:** [Taobao's Double 11 (Singles' Day) inventory handling](https://www.alibabacloud.com/blog/alibaba-tech-double-11-2020_597110)

---

### Scenario 3: Comment Count on a Post
A viral post gets 10k comments per second. `UPDATE posts SET comment_count = comment_count + 1` is a hot-row bottleneck.

**Solution: Redis + async DB sync**
```python
# On new comment:
redis.hincrby(f"post:{post_id}:meta", "comment_count", 1)
# Flush to DB every 5 seconds via background job
```

Or use a separate `comment_counts` table (one row per post per shard) and aggregate on read.

---

## 6. Tricky Scenarios

### Tricky Scenario 1: Phantom Reads in Range Queries

**Context:** Two transactions both check "is there a doctor available between 2pm-3pm?" and both insert a booking.

```sql
-- T1 and T2 both run simultaneously
SELECT COUNT(*) FROM bookings WHERE doctor_id = 5 AND time_slot = '14:00'; -- both see 0
INSERT INTO bookings (doctor_id, time_slot, patient_id) VALUES (5, '14:00', ...); -- both succeed
-- Result: double-booked!
```

This is a **write skew** anomaly — neither transaction violated its own constraint, but the combined result is invalid.

**Fix:** Use `SERIALIZABLE` isolation level (Postgres SSI handles this), or use `SELECT FOR UPDATE` on the check query to lock the gap.

```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- OR use explicit locking:
SELECT * FROM bookings WHERE doctor_id = 5 AND time_slot = '14:00' FOR UPDATE;
-- If 0 rows → no conflict, safe to insert
INSERT INTO bookings ...;
COMMIT;
```

> 📖 **Reference:** [Kleppmann, "Designing Data-Intensive Applications" — Chapter 7: Transactions](https://dataintensive.net/)

---

### Tricky Scenario 2: Distributed Deadlock (Across Microservices)
Single-DB deadlocks are detected automatically. Distributed deadlocks (across services with their own DBs) are not.

**Scenario:** Service A locks User record → calls Service B → Service B tries to update User record (via its own DB connection or shared DB).

**Detection:** Implement distributed tracing and look for circular call graphs under lock. Use a global timeout on all cross-service calls inside a transaction context.

**Solution:** Redesign to avoid distributed locking. Use saga pattern (compensating transactions) rather than distributed ACID.

---

### Tricky Scenario 3: Read-Modify-Write in Distributed Cache + DB
```
Thread 1: read counter from Redis → 10
Thread 2: read counter from Redis → 10
Thread 1: write 11 to Redis
Thread 2: write 11 to Redis  ← Lost update! Should be 12
```

**Fix:** Use atomic Redis operations:
```python
# Not safe:
val = redis.get("counter")
redis.set("counter", int(val) + 1)

# Safe:
redis.incr("counter")  # Atomic increment
```

For complex read-modify-write in Redis: use **Lua scripts** (executed atomically) or **Redis transactions** (`MULTI/EXEC`).

---

### Tricky Scenario 4: Lock Escalation in Bulk Updates
```sql
UPDATE orders SET status = 'archived' WHERE created_at < '2020-01-01';
-- This touches 10 million rows
-- InnoDB escalates from row locks to table lock
-- All other writes blocked for minutes
```

**Fix:** Batch updates in small chunks with sleep between batches:
```python
while True:
    rows_affected = db.execute("""
        UPDATE orders SET status = 'archived'
        WHERE created_at < '2020-01-01' AND status != 'archived'
        LIMIT 1000
    """)
    if rows_affected == 0:
        break
    time.sleep(0.1)  # yield to other transactions
```

This is called **chunked migration** — a standard technique for schema changes on live production databases.

---

## 7. Contention Patterns Summary Table

| Contention Type | Pattern to Apply | When |
|----------------|-----------------|------|
| Hot row (counter) | Counter sharding / Redis INCR | High-frequency increments |
| Concurrent updates to same row | Optimistic locking (versioning) | Read-heavy, rare conflicts |
| Must guarantee no lost updates | Pessimistic locking (SELECT FOR UPDATE) | Write-heavy, conflicts frequent |
| Multi-row deadlocks | Lock in consistent order | Multi-entity transactions |
| Long transaction blocking others | Minimize lock scope; no I/O in TX | All systems |
| Job queue worker contention | SKIP LOCKED | Background job processing |
| Phantom reads / write skew | SERIALIZABLE isolation or range locks | Booking/scheduling systems |
| Distributed contention | Saga pattern, message queues | Microservices |
| Retry storms after failure | Exponential backoff + jitter + idempotency keys | All retry logic |

---

## 8. Reference Case Studies

1. **PostgreSQL Advisory Locks at Gitlab**  
   [https://about.gitlab.com/blog/2021/09/09/excluding-locks/](https://about.gitlab.com/blog/2021/09/09/excluding-locks/)

2. **Stripe: Designing Robust Distributed Systems**  
   [https://stripe.com/blog/idempotency](https://stripe.com/blog/idempotency)

3. **Shopify: Handling Millions of Concurrent Requests (Flash Sales)**  
   [https://engineering.shopify.com/blogs/engineering/under-pressure](https://engineering.shopify.com/blogs/engineering/under-pressure)

4. **MySQL InnoDB Locking and Transaction Model**  
   [https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-transaction-model.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-transaction-model.html)

5. **Designing Data-Intensive Applications — Chapter 7: Transactions (Kleppmann)**  
   [https://dataintensive.net/](https://dataintensive.net/)

6. **CockroachDB: Understanding and Avoiding Transaction Contention**  
   [https://www.cockroachlabs.com/docs/stable/performance-best-practices-overview.html](https://www.cockroachlabs.com/docs/stable/performance-best-practices-overview.html)
