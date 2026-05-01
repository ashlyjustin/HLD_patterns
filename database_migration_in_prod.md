# Database Migrations in Production: Zero-Downtime Patterns and Techniques

> **What this doc covers:** How to safely evolve large production databases — schema changes, data backfills, type changes, index creation, table renames — without taking an outage. Covers the mental model (Expand-Contract), unsafe vs safe operations per database, the right tooling (gh-ost, pg_repack, pt-osc), multi-service migration choreography, database version upgrades, and the failure modes that have taken down production systems.

### 📋 Outline

| # | Section | What You'll Learn |
|---|---------|-------------------|
| 1 | Why Zero-Downtime Migrations Are Hard | Locking behaviour, deploy/DB coupling, why "just ALTER TABLE" fails |
| 2 | The Expand-Contract Pattern | The universal mental model for backwards-compatible schema evolution |
| 3 | Safe vs Unsafe DDL Operations | Per-operation lock analysis for Postgres, MySQL, and distributed DBs |
| 4 | Online Schema Change Tools | gh-ost, pt-osc, pg_repack — internals, trade-offs, when to use each |
| 5 | Column Operations In Depth | Add, rename, change type, drop — step-by-step for each |
| 6 | Constraint Operations In Depth | NOT NULL, foreign keys, unique — how to add each without locking |
| 7 | Index Operations In Depth | CONCURRENTLY, invisible indexes, covering indexes, partial indexes |
| 8 | Large-Scale Data Backfills | Chunked batching, backpressure, idempotency, progress tracking |
| 9 | Multi-Service Migration Choreography | Coordinating schema changes across microservices with feature flags |
| 10 | Database Version Upgrades | Major version upgrades, logical replication cutover, blue-green DB |
| 11 | Regular Scenarios | Adding a column, extracting a table, adding a unique constraint |
| 12 | Tricky Scenarios | Rename with traffic, BIGINT overflow, multi-tenant schema split |
| 13 | Common Errors and Failure Modes | The 10 mistakes that have caused production outages |
| 14 | Reference Case Studies | GitHub, Shopify, Stripe, GitLab, Notion |

---

## 1. Why Zero-Downtime Migrations Are Hard

### 1.1 The Naive Approach and Why It Fails

A common beginner mistake:

```sql
-- ❌ This will lock a 500-million-row table for 45 minutes
ALTER TABLE orders ADD COLUMN discount_code VARCHAR(50);
```

On Postgres <11 (and still on MySQL without online DDL), adding a column with a default requires rewriting the entire table. During the rewrite, an `ACCESS EXCLUSIVE` lock is held — no reads, no writes, complete outage.

Even on modern databases where this specific case is fast, the instinct to "just ALTER TABLE" leads to trouble because:

1. **Lock escalation** — even a brief metadata lock blocks all queries queueing behind it. 10,000 req/s × 2 seconds of blocking = 20,000 queued connections, OOM crash.
2. **Deploy/DB coupling** — your application deploy and your schema change are two separate events. Between them, both old and new code run simultaneously against the same database.
3. **Data migrations** — adding a column is one thing; filling 500M rows with correct values is another. You cannot do it in a single transaction.
4. **Rollback surface** — a schema change that touches live data creates a much larger rollback blast radius than a code deploy alone.

### 1.2 The Fundamental Constraint: Simultaneous Code Versions

During any rolling deploy, version `N` and version `N+1` of your application run at the same time. Your schema must be simultaneously compatible with both.

```
Time ──────────────────────────────────────────────────────▶
      │  v1 only  │  v1 + v2 mixed  │  v2 only  │
      └───────────┴─────────────────┴───────────┘
      ↑           ↑                 ↑
   deploy         DB schema        old pods
   starts         change           drained
```

Any schema that breaks `v1` cannot be applied while `v1` pods are still running.

### 1.3 Lock Types Reference (Postgres)

| Lock Mode | Who Takes It | Who It Blocks |
|-----------|--------------|---------------|
| `ACCESS SHARE` | SELECT | ACCESS EXCLUSIVE only |
| `ROW SHARE` | SELECT FOR UPDATE | EXCLUSIVE, ACCESS EXCLUSIVE |
| `ROW EXCLUSIVE` | INSERT, UPDATE, DELETE | SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE |
| `SHARE UPDATE EXCLUSIVE` | VACUUM, CREATE INDEX CONCURRENTLY | Itself, and heavier locks |
| `ACCESS EXCLUSIVE` | ALTER TABLE, DROP TABLE, TRUNCATE | Everything — full table lock |

The danger: even if your DDL runs quickly, it still queues behind existing long-running transactions, and all subsequent queries queue behind your DDL. A 100ms ALTER can cause a cascading outage.

```sql
-- Safe pattern: always set a lock timeout before DDL
SET lock_timeout = '2s';
ALTER TABLE users ADD COLUMN preferences JSONB;
-- If it can't acquire the lock in 2s, it errors instead of blocking
```

---

## 2. The Expand-Contract Pattern

Also called **Parallel Change** or **Blue-Green Schema Migration**. This is the universal mental model for all zero-downtime migrations.

### 2.1 Three Phases

```
EXPAND ──────▶ MIGRATE ──────▶ CONTRACT
(additive)    (backfill)      (cleanup)
```

**Phase 1 — Expand (backwards-additive change):**
Add the new thing (column, table, index) alongside the old one. Both old and new code can run simultaneously. This phase is deployed and left running.

**Phase 2 — Migrate:**
Backfill existing data into the new structure. Write new data into both old and new locations (dual write). This phase runs as a background job, may take hours or days.

**Phase 3 — Contract (backwards-incompatible removal):**
Remove the old thing. By now all code has been updated to use only the new structure. This phase can only be deployed after all old-code pods are gone and all data is migrated.

### 2.2 Why This Works

Between phases 1 and 3, any rollback is safe because both old and new structures exist. You're never in a state where the running code references a column that doesn't exist.

### 2.3 Example: Renaming a Column

```
Phase 1 (Expand):   Add new column `full_name`. Code writes to both `name` and `full_name`.
Phase 2 (Migrate):  Backfill `full_name = name` for all existing rows. 
Phase 3 (Contract): Drop old column `name`. Code only writes to `full_name`.
```

Full walkthrough in §5.2.

---

## 3. Safe vs Unsafe DDL Operations

### 3.1 Postgres

| Operation | Safe? | Lock Taken | Notes |
|-----------|-------|-----------|-------|
| `ADD COLUMN` (no default) | ✅ Safe | Brief ACCESS EXCLUSIVE | Postgres 11+: even with default, no rewrite |
| `ADD COLUMN NOT NULL` (no default) | ❌ Unsafe | ACCESS EXCLUSIVE + rewrite | Use nullable + backfill + constraint |
| `ADD COLUMN DEFAULT` (volatile fn) | ❌ Unsafe | ACCESS EXCLUSIVE + rewrite | `now()` requires rewrite pre-PG11 |
| `DROP COLUMN` | ✅ Safe | Brief ACCESS EXCLUSIVE | Column is marked invisible, no rewrite |
| `RENAME COLUMN` | ❌ Unsafe* | ACCESS EXCLUSIVE | Safe only if no code uses old name |
| `ALTER COLUMN TYPE` | ❌ Unsafe | ACCESS EXCLUSIVE + rewrite | Unless using `USING` to varchar(larger) |
| `ADD INDEX` | ❌ Unsafe | SHARE (blocks writes) | Use `CREATE INDEX CONCURRENTLY` |
| `ADD INDEX CONCURRENTLY` | ✅ Safe | SHARE UPDATE EXCLUSIVE | Allows reads+writes, takes longer |
| `ADD FOREIGN KEY` | ❌ Unsafe | SHARE ROW EXCLUSIVE on both tables | Use `NOT VALID` then `VALIDATE` |
| `ADD UNIQUE CONSTRAINT` | ❌ Unsafe | ACCESS EXCLUSIVE | Create unique index CONCURRENTLY first |
| `ADD NOT NULL` | ❌ Unsafe | ACCESS EXCLUSIVE + scan | Use `ADD CONSTRAINT ... NOT VALID` then `VALIDATE` |
| `DROP INDEX` | ✅ Safe | ACCESS EXCLUSIVE (brief) | Instant metadata change |
| `DROP INDEX CONCURRENTLY` | ✅ Safer | SHARE UPDATE EXCLUSIVE | No ACCESS EXCLUSIVE needed |
| `TRUNCATE` | ❌ Unsafe | ACCESS EXCLUSIVE | Never on live tables |

### 3.2 MySQL / InnoDB

MySQL's online DDL (introduced in 5.6, improved in 8.0) runs many operations in-place with concurrent DML.

| Operation | Algorithm | Lock | Notes |
|-----------|-----------|------|-------|
| `ADD COLUMN` | INSTANT (8.0) or INPLACE | NONE | INSTANT for most cases in 8.0 |
| `ADD COLUMN FIRST/AFTER` | COPY | SHARED | Requires table rewrite in some versions |
| `DROP COLUMN` | INPLACE | NONE | Fast in 8.0 |
| `RENAME COLUMN` | INSTANT | NONE | 8.0 only |
| `ADD INDEX` | INPLACE | NONE | Allows concurrent reads/writes |
| `ADD FULLTEXT INDEX` | INPLACE (first one: COPY) | SHARED | |
| `ADD FOREIGN KEY` | INPLACE | SHARED | |
| `CHANGE COLUMN TYPE` | COPY | SHARED | Table rewrite |
| `ADD PRIMARY KEY` | COPY | SHARED | Table rewrite |

```sql
-- MySQL: explicitly request online DDL
ALTER TABLE orders 
  ADD COLUMN discount_code VARCHAR(50),
  ALGORITHM=INPLACE, LOCK=NONE;
-- If INPLACE is not supported, MySQL errors rather than falling back to COPY
```

### 3.3 Distributed Databases (CockroachDB, Spanner, Vitess)

CockroachDB uses **schema change jobs** — DDL is non-blocking by design. Changes are versioned and rolled out via a two-version protocol across all nodes. A column is not removed until all nodes have acknowledged the new schema version.

```sql
-- CockroachDB: shows schema change job
SHOW JOBS;
-- job_id | job_type | description | status | fraction_completed
```

Spanner uses a similar two-phase schema propagation with a 1-7 minute rollout window.

Vitess/PlanetScale uses the `ONLINE DDL` subsystem which runs gh-ost or pt-osc under the hood, orchestrated across shards.

---

## 4. Online Schema Change Tools

When the database's built-in DDL is not safe enough, use an online schema change (OSC) tool. These work by creating a shadow table, copying data, and atomically swapping.

### 4.1 gh-ost (GitHub Online Schema Transmogrifier)

**How it works:**

```
┌──────────────┐     binary log     ┌──────────────┐
│  original    │ ─────────────────▶ │  ghost table │
│  table       │                    │  (new schema)│
└──────────────┘                    └──────────────┘
       ↑                                    │
  all DML continues                    row copy +
  on original                         binlog replay
                                            │
                                  ┌─────────▼──────────┐
                                  │  LOCK + RENAME SWAP │
                                  │  (~1-2 seconds)     │
                                  └────────────────────┘
```

1. Creates `_orders_gho` (ghost table) with new schema
2. Reads binary log (not triggers) to capture all DML on original
3. Copies data in chunks (configurable row count/chunk size)
4. Applies binlog events to keep ghost in sync
5. When copy is complete, acquires a brief lock (< 2 seconds) and atomically renames tables

**Why gh-ost is preferred over pt-osc:**
- pt-osc uses triggers → adds write overhead to every INSERT/UPDATE/DELETE during migration
- gh-ost reads the binlog directly → no trigger overhead
- gh-ost is pausable/throttleable without schema state corruption
- gh-ost has a `--test-on-replica` mode for dry runs

```bash
gh-ost \
  --user="ghost" \
  --password="..." \
  --host="replica.internal" \
  --database="myapp" \
  --table="orders" \
  --alter="ADD COLUMN discount_code VARCHAR(50)" \
  --allow-on-master \
  --execute \
  --max-load="Threads_running=50" \
  --critical-load="Threads_running=100" \
  --chunk-size=1000 \
  --default-retries=120 \
  --panic-flag-file=/tmp/ghost.panic \
  --postpone-cut-over-flag-file=/tmp/ghost.postpone
```

Key flags:
- `--max-load` — pause migration when DB load exceeds threshold
- `--critical-load` — abort migration if this threshold is hit
- `--postpone-cut-over-flag-file` — delay the final rename swap indefinitely (touch this file = hold; rm this file = proceed)
- `--panic-flag-file` — touch this file to immediately halt and roll back

**Monitoring gh-ost progress:**

```bash
# gh-ost exposes a unix socket for live inspection
echo "status" | nc -U /tmp/gh-ost.orders.sock
# Output: # Migrating myapp.orders; Ghost table is myapp._orders_gho
# Migrated 45,231,445/200,000,000 rows (22.6%)
# ETA: 3h41m remaining
```

### 4.2 pt-online-schema-change (Percona Toolkit)

Uses triggers. Simpler to use, but triggers add write amplification.

```bash
pt-online-schema-change \
  --alter "ADD COLUMN discount_code VARCHAR(50)" \
  --host=primary.internal \
  --user=root \
  --password=... \
  D=myapp,t=orders \
  --execute \
  --chunk-size=1000 \
  --max-lag=1  # pause if replica lag > 1s
```

**Use pt-osc when:**
- MySQL < 5.6 (no online DDL)
- gh-ost is not available
- You need pt-osc's `--recursion-method` for complex replication topologies

**Avoid pt-osc when:**
- Table has other triggers (only one trigger per event per table in MySQL 5.5)
- Very high write throughput (trigger overhead is significant)

### 4.3 pg_repack (Postgres)

Postgres's `CLUSTER` and `VACUUM FULL` require an `ACCESS EXCLUSIVE` lock. `pg_repack` rewrites tables and indexes online.

```bash
pg_repack --host=localhost --dbname=myapp --table=orders
```

Use cases:
- Reclaim space after large deletes (table bloat)
- Re-cluster a table on a different index
- Repack a bloated index

How it works: similar to gh-ost but uses a trigger to capture changes during copy, then does a brief lock-swap. Because it uses triggers, avoid on very high-write tables.

### 4.4 Tool Comparison

| Tool | DB | Mechanism | Trigger overhead | Pausable | Cut-over time |
|------|----|-----------|-----------------|----|--------------|
| gh-ost | MySQL | Binlog replay | None | Yes | ~1-2 seconds |
| pt-osc | MySQL | Triggers | Yes | Yes | ~1-2 seconds |
| pg_repack | Postgres | Triggers | Yes | No | Brief lock |
| CockroachDB DDL | CRDB | Schema jobs | None | No | Online/gradual |

---

## 5. Column Operations In Depth

### 5.1 Adding a Column (Safe)

**Postgres 11+:**

```sql
-- Safe: no table rewrite, brief metadata lock
SET lock_timeout = '2s';
ALTER TABLE orders ADD COLUMN discount_code VARCHAR(50);
```

**Postgres 10 and below (or MySQL with non-INSTANT DDL):**

Use gh-ost or pt-osc (see §4).

**Adding a column with a non-null default (Postgres 11+):**

```sql
-- Safe on Postgres 11+ — stored as a catalog default, no rewrite
ALTER TABLE users ADD COLUMN is_verified BOOLEAN NOT NULL DEFAULT false;
```

On Postgres < 11, this required a table rewrite. The workaround was:

```sql
-- Step 1: Add nullable column (no default, no rewrite)
ALTER TABLE users ADD COLUMN is_verified BOOLEAN;

-- Step 2: Backfill in batches (see §8)
UPDATE users SET is_verified = false WHERE id BETWEEN 1 AND 100000;
-- ... repeat for all rows ...

-- Step 3: Set default for new rows
ALTER TABLE users ALTER COLUMN is_verified SET DEFAULT false;

-- Step 4: Add NOT NULL constraint (see §6.1)
ALTER TABLE users ADD CONSTRAINT users_is_verified_not_null 
  CHECK (is_verified IS NOT NULL) NOT VALID;
VALIDATE CONSTRAINT users_is_verified_not_null;
```

### 5.2 Renaming a Column (The Hard Way)

You cannot simply `ALTER TABLE users RENAME COLUMN name TO full_name` — this breaks any running `v1` code that reads `name`.

**Full Expand-Contract walkthrough:**

```
Deploy  Code Change             DB Change
──────  ──────────────────────  ────────────────────────────────────────
 v1     reads `name`            column `name` exists
        writes to `name`
        
 v2     reads `full_name`       ADD COLUMN full_name VARCHAR(255)
        writes to both          (backwards-compatible: v1 still works)
        `name` AND `full_name`  
        
backfill job runs:              UPDATE users SET full_name = name
                                WHERE full_name IS NULL (batched)
                                
 v3     reads `full_name`       DROP COLUMN name
        writes only to          (v2 no longer running)
        `full_name`
```

**v2 dual-write code example:**

```python
def update_user_name(user_id: int, name: str):
    db.execute("""
        UPDATE users 
        SET name = %s, full_name = %s  -- write to both
        WHERE id = %s
    """, (name, name, user_id))

def get_user_name(user_id: int) -> str:
    row = db.fetchone("SELECT full_name, name FROM users WHERE id = %s", (user_id,))
    # read new column, fall back to old if null (handles rows not yet backfilled)
    return row['full_name'] or row['name']
```

**Migration timing:**
- v2 code + `ADD COLUMN full_name` → deploy together
- Backfill job runs → may take hours
- Verify 0 NULL rows in `full_name`
- v3 code + `DROP COLUMN name` → deploy when v2 pods are all gone

### 5.3 Changing a Column Type

This is the most dangerous operation. A type change almost always requires a table rewrite.

**Case 1: Widening a VARCHAR (safe in Postgres)**

```sql
-- VARCHAR(50) → VARCHAR(255): only a catalog change, no rewrite
ALTER TABLE users ALTER COLUMN username TYPE VARCHAR(255);
```

**Case 2: INT → BIGINT (requires rewrite — use gh-ost)**

```bash
# Use gh-ost to rewrite with new type
gh-ost \
  --alter="MODIFY COLUMN id BIGINT NOT NULL AUTO_INCREMENT" \
  --table=users \
  --execute
```

For the INT overflow emergency scenario (§12.2), see the detailed tricky scenario.

**Case 3: VARCHAR → ENUM or JSONB → normalized column**

Always use Expand-Contract: add new column with correct type, dual-write, backfill, drop old.

### 5.4 Dropping a Column

Dropping is safe (Postgres marks it invisible without rewriting), but you must ensure no code references the column first.

**Safe drop sequence:**

```sql
-- 1. Verify no live code references the column
--    (search codebase, ORM models, views, functions, triggers)

-- 2. Remove all DEFAULT values first (optional but clean)
ALTER TABLE users ALTER COLUMN legacy_field DROP DEFAULT;

-- 3. Drop the column
SET lock_timeout = '2s';
ALTER TABLE users DROP COLUMN legacy_field;
```

**Postgres physical reclaim:** Dropping a column doesn't reclaim disk space. The column data is still physically stored, just hidden. Run `VACUUM FULL` or `pg_repack` to reclaim space (but this locks the table — plan accordingly).

---

## 6. Constraint Operations In Depth

### 6.1 Adding NOT NULL Without Locking (Postgres)

`ALTER TABLE t ALTER COLUMN c SET NOT NULL` takes `ACCESS EXCLUSIVE` and does a full table scan.

**Safe pattern using `CHECK CONSTRAINT NOT VALID`:**

```sql
-- Step 1: Add constraint as NOT VALID (no scan, brief lock)
-- This applies to NEW rows only
ALTER TABLE users 
  ADD CONSTRAINT users_email_not_null 
  CHECK (email IS NOT NULL) NOT VALID;

-- Step 2: Backfill any NULL values
UPDATE users SET email = 'unknown@example.com' WHERE email IS NULL;

-- Step 3: VALIDATE — scans table but takes only SHARE UPDATE EXCLUSIVE
-- (allows concurrent reads and writes)
ALTER TABLE users VALIDATE CONSTRAINT users_email_not_null;

-- Step 4: (Optional) Convert to true NOT NULL
-- Postgres is smart: if a validated CHECK (col IS NOT NULL) exists,
-- it will skip the scan for SET NOT NULL
ALTER TABLE users ALTER COLUMN email SET NOT NULL;
ALTER TABLE users DROP CONSTRAINT users_email_not_null;
```

### 6.2 Adding a Foreign Key Without Locking

```sql
-- WRONG: takes SHARE ROW EXCLUSIVE on both tables
ALTER TABLE orders ADD CONSTRAINT fk_orders_user 
  FOREIGN KEY (user_id) REFERENCES users(id);

-- RIGHT: add as NOT VALID first (skips scan of existing rows)
ALTER TABLE orders ADD CONSTRAINT fk_orders_user 
  FOREIGN KEY (user_id) REFERENCES users(id) NOT VALID;

-- Then validate separately (SHARE UPDATE EXCLUSIVE — allows DML)
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_user;
```

### 6.3 Adding a UNIQUE Constraint

```sql
-- WRONG: takes ACCESS EXCLUSIVE
ALTER TABLE users ADD CONSTRAINT users_email_unique UNIQUE (email);

-- RIGHT: create a unique index concurrently first
CREATE UNIQUE INDEX CONCURRENTLY users_email_unique_idx ON users(email);

-- Then promote to constraint (brief lock, no scan — index is already built)
ALTER TABLE users ADD CONSTRAINT users_email_unique 
  UNIQUE USING INDEX users_email_unique_idx;
```

---

## 7. Index Operations In Depth

### 7.1 Creating Indexes Safely

```sql
-- WRONG: SHARE lock — blocks all writes during index build
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- RIGHT: CONCURRENTLY — allows reads AND writes during build
-- Takes 2-3x longer but zero write blocking
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
```

**CONCURRENTLY caveats:**
- Cannot be run inside a transaction block
- If it fails halfway, leaves a `INVALID` index — drop it and retry
- Postgres 12+ can `REINDEX CONCURRENTLY` to rebuild without locking

```sql
-- Check for invalid indexes
SELECT schemaname, tablename, indexname, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
JOIN pg_index USING (indexrelid)
WHERE NOT indisvalid;
```

### 7.2 Invisible / Hidden Indexes (MySQL 8.0+)

Before dropping an index, make it invisible. If nothing breaks, then drop it.

```sql
-- Make index invisible to optimizer (still maintained, just not used)
ALTER TABLE orders ALTER INDEX idx_orders_legacy INVISIBLE;

-- Monitor for a few days — no query plan changes
SHOW INDEX FROM orders;

-- Safe to drop
DROP INDEX idx_orders_legacy ON orders;
```

Postgres doesn't have invisible indexes, but you can use `pg_hint_plan` to force queries to avoid an index for testing.

### 7.3 Index Strategies for Large Tables

**Partial indexes** — index only a subset of rows, dramatically reducing index size:

```sql
-- Index only unprocessed orders (assume 99% of orders are 'completed')
CREATE INDEX CONCURRENTLY idx_orders_pending 
  ON orders(created_at) 
  WHERE status = 'pending';
```

**Covering indexes** — include non-key columns to avoid heap fetches:

```sql
-- Query: SELECT user_id, total FROM orders WHERE created_at > '2024-01-01'
-- Without covering: index scan → heap fetch for total
-- With covering: index-only scan
CREATE INDEX CONCURRENTLY idx_orders_date_covering 
  ON orders(created_at) INCLUDE (user_id, total);
```

**Index on expression** — for case-insensitive searches:

```sql
CREATE INDEX CONCURRENTLY idx_users_email_lower 
  ON users(LOWER(email));

-- Requires matching query:
SELECT * FROM users WHERE LOWER(email) = LOWER('User@Example.com');
```

---

## 8. Large-Scale Data Backfills

### 8.1 The Wrong Way

```sql
-- ❌ Single giant UPDATE — holds row locks for minutes/hours
-- Replication lag will spike, replicas fall behind
-- If it fails midway, you've only partially updated
UPDATE users SET full_name = name WHERE full_name IS NULL;
```

### 8.2 Chunked Batching

Break the work into small, independently-committable chunks. Each chunk is a short transaction.

```python
import time
import psycopg2

def backfill_full_name(conn, chunk_size=1000, sleep_ms=100):
    """
    Backfill users.full_name from users.name in chunks.
    Idempotent: uses WHERE full_name IS NULL so it's safe to restart.
    """
    cursor = conn.cursor()
    total_updated = 0
    
    while True:
        cursor.execute("""
            UPDATE users
            SET full_name = name
            WHERE id IN (
                SELECT id FROM users
                WHERE full_name IS NULL
                LIMIT %s
                FOR UPDATE SKIP LOCKED  -- skip rows locked by concurrent updates
            )
        """, (chunk_size,))
        
        rows_updated = cursor.rowcount
        conn.commit()
        total_updated += rows_updated
        
        print(f"Updated {rows_updated} rows (total: {total_updated})")
        
        if rows_updated == 0:
            break  # done
            
        # Backpressure: check replica lag before continuing
        if get_replica_lag() > 5:  # seconds
            print("Replica lag high, pausing...")
            time.sleep(5)
        else:
            time.sleep(sleep_ms / 1000)
    
    print(f"Backfill complete. Total rows updated: {total_updated}")

def get_replica_lag():
    # Query replica lag from your monitoring system or pg_stat_replication
    pass
```

### 8.3 Keyset (Cursor-Based) Batching for Large Tables

Using `LIMIT + OFFSET` degrades at high offsets. Use keyset pagination instead:

```python
def backfill_by_id(conn, chunk_size=1000, sleep_ms=50):
    cursor = conn.cursor()
    
    # Start from the minimum unprocessed ID
    cursor.execute("SELECT MIN(id) FROM users WHERE full_name IS NULL")
    last_id = cursor.fetchone()[0]
    
    if last_id is None:
        print("Nothing to backfill")
        return
    
    while True:
        cursor.execute("""
            UPDATE users
            SET full_name = name
            WHERE id IN (
                SELECT id FROM users
                WHERE full_name IS NULL
                  AND id >= %s
                ORDER BY id
                LIMIT %s
            )
            RETURNING id
        """, (last_id, chunk_size))
        
        updated_ids = [row[0] for row in cursor.fetchall()]
        conn.commit()
        
        if not updated_ids:
            break
            
        last_id = max(updated_ids) + 1
        print(f"Last processed ID: {last_id}, chunk size: {len(updated_ids)}")
        time.sleep(sleep_ms / 1000)
```

### 8.4 Progress Tracking and Idempotency

Always design backfills to be:

1. **Idempotent** — safe to run multiple times. Use `WHERE new_column IS NULL` or a `backfilled = false` flag.
2. **Resumable** — track `last_processed_id` in a `migration_progress` table.
3. **Observable** — emit progress metrics.

```sql
CREATE TABLE migration_progress (
    migration_name TEXT PRIMARY KEY,
    last_processed_id BIGINT,
    total_rows BIGINT,
    processed_rows BIGINT,
    started_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);

-- Update progress after each chunk
INSERT INTO migration_progress (migration_name, last_processed_id, total_rows, processed_rows)
VALUES ('backfill_full_name', :last_id, :total, :processed)
ON CONFLICT (migration_name) DO UPDATE
  SET last_processed_id = EXCLUDED.last_processed_id,
      processed_rows = EXCLUDED.processed_rows,
      updated_at = now();
```

### 8.5 Backfill Backpressure

Backfills compete with production traffic for I/O, CPU, and replication bandwidth. Use these backpressure controls:

```python
class BackfillController:
    def __init__(self, max_replica_lag_sec=3, max_threads_running=80):
        self.max_replica_lag = max_replica_lag_sec
        self.max_threads_running = max_threads_running
    
    def should_pause(self, conn) -> bool:
        # Check replica lag
        with conn.cursor() as cur:
            cur.execute("""
                SELECT GREATEST(COALESCE(
                    EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())), 0
                ), 0) AS lag_seconds
            """)
            lag = cur.fetchone()[0]
        
        if lag > self.max_replica_lag:
            return True
        
        # Check active connections
        with conn.cursor() as cur:
            cur.execute("SELECT count(*) FROM pg_stat_activity WHERE state = 'active'")
            active = cur.fetchone()[0]
        
        return active > self.max_threads_running
```

---

## 9. Multi-Service Migration Choreography

### 9.1 The Multi-Service Problem

When multiple services share a database (common in monolith-to-microservice transitions), a schema change must be coordinated across all services simultaneously.

```
     Service A (v1)          Service B (v1)
          │                        │
          └─────────┐  ┌──────────┘
                    ▼  ▼
              Shared Database
```

If you rename a column, Service A's v2 may deploy before Service B's v2. During the gap, Service B's v1 writes to the old column name — which no longer exists if you've already run the migration. **System outage.**

### 9.2 Feature Flag-Gated Migrations

Use feature flags to decouple code readiness from migration execution:

```python
# Service A and Service B both deploy this code first
def get_user_name(user_id):
    if feature_flags.is_enabled('use_full_name_column'):
        return db.query("SELECT full_name FROM users WHERE id = %s", user_id)
    else:
        return db.query("SELECT name FROM users WHERE id = %s", user_id)
```

Migration sequence:
1. Deploy new code to all services (flag OFF — old behaviour)
2. Run schema migration (add `full_name`, backfill, etc.)
3. Enable feature flag (all services now use `full_name`)
4. Verify — monitor for errors
5. Remove old code path + feature flag in next release

### 9.3 Schema Ownership in Microservices

The root cause of cross-service migration pain is shared databases. The correct long-term fix:

```
BAD:                           GOOD:
Service A ─┐                  Service A → Database A
           ├─▶ Shared DB      Service B → Database B
Service B ─┘                  (expose data via API, not direct DB access)
```

When you cannot split the DB immediately:

1. **Create a schema-per-service convention** — `service_a.users`, `service_b.orders`
2. **Use database views** for cross-service reads (views can be updated independently)
3. **Route writes through the owning service's API** — prevents direct cross-service schema coupling

### 9.4 Coordinating with ORMs

ORMs (ActiveRecord, SQLAlchemy, Hibernate) cache column metadata at startup. After a column rename, old pods will fail if they try to INSERT/SELECT the old name.

Mitigation:
- Dual-write phase ensures both columns exist
- Add `SELECT old_col, new_col` to queries during transition (explicit column list — never `SELECT *`)
- Drain old pods before removing old column

---

## 10. Database Version Upgrades

### 10.1 In-Place Upgrade Risks

Most databases support in-place major version upgrades (e.g., `pg_upgrade`). These are fast but risky:

- No easy rollback once upgrade is complete
- Incompatible extensions may fail to start
- Query plans may change (different planner statistics)
- Replication protocol changes may break standbys

### 10.2 Logical Replication Cutover (Zero-Downtime)

The safest pattern for major version upgrades:

```
┌──────────────────────────────────────────────────────────────┐
│  Step 1: Set up new DB (target version) as logical replica   │
│                                                              │
│  Old DB (PG 14) ──── logical replication ────▶ New DB (PG 16)│
│                                                              │
│  Step 2: Let replication catch up (lag < 100ms)             │
│                                                              │
│  Step 3: Stop writes to old DB (maintenance mode, seconds)   │
│                                                              │
│  Step 4: Apply last WAL changes to new DB                    │
│                                                              │
│  Step 5: Update connection strings → point to new DB         │
│                                                              │
│  Step 6: Verify, then remove old DB                         │
└──────────────────────────────────────────────────────────────┘
```

**Setting up logical replication for PG upgrade:**

```sql
-- On old DB (publisher):
ALTER SYSTEM SET wal_level = logical;
-- Restart required

CREATE PUBLICATION upgrade_pub FOR ALL TABLES;

-- On new DB (subscriber):
CREATE SUBSCRIPTION upgrade_sub
  CONNECTION 'host=old-db dbname=myapp user=replicator password=...'
  PUBLICATION upgrade_pub;

-- Monitor replication lag:
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

**Logical replication limitations:**
- Sequences are not replicated (update sequences manually on cut-over)
- DDL changes during replication will break the subscription
- Large objects are not replicated

### 10.3 Blue-Green Database Cutover

For the most conservative cutover with a rollback window:

```
                     ┌─── Load Balancer ───┐
                     │                     │
              Application pods             │
                     │                     │
               ┌─────▼──────┐       ┌──────▼──────┐
               │  Blue DB   │  sync  │  Green DB   │
               │ (current)  │◀──────▶│ (new ver.)  │
               └────────────┘       └─────────────┘
```

1. Green DB is a fully migrated copy of Blue, kept in sync via replication
2. Traffic switches to Green
3. Blue is kept alive for rollback window (hours/days)
4. If rollback needed: switch traffic back to Blue (any writes to Green are lost or must be replayed)

**Cut-over atomicity challenge:** The hardest part is ensuring no writes land in Blue after you start the switch to Green. Typical approach: brief maintenance page (< 10 seconds) or double-write to both during the switch.

### 10.4 Extension Compatibility Checklist

```sql
-- List installed extensions and versions before upgrade
SELECT name, default_version, installed_version 
FROM pg_available_extensions 
WHERE installed_version IS NOT NULL;

-- Common extensions that require extra steps during major upgrades:
-- PostGIS: usually needs pg_upgrade --link or dump/restore
-- pg_partman: check compatibility matrix
-- TimescaleDB: use timescaledb-tune migration guide
-- pgvector: recompile against new major version
```

---

## 11. Regular Scenarios

### Scenario 1: Adding a New Column to a High-Traffic Table (Postgres 14)

**Context:** `users` table, 50M rows, 5,000 writes/second. Need to add `preferences JSONB`.

**Solution:**

```sql
-- One-liner, safe on PG 11+
-- No table rewrite, brief metadata lock, default stored in catalog
SET lock_timeout = '2s';
ALTER TABLE users ADD COLUMN preferences JSONB DEFAULT '{}' NOT NULL;
```

**Why it works:** Postgres 11+ stores the default value in `pg_attrdef`, not in each row. Existing rows return the default without any I/O. The lock held is extremely brief (milliseconds).

**Verification:**

```sql
-- Confirm no table rewrite happened (row count should be unchanged in pg_class)
SELECT relname, reltuples FROM pg_class WHERE relname = 'users';

-- Test default value for existing rows
SELECT preferences FROM users LIMIT 5;
-- Should return '{}' for all
```

---

### Scenario 2: Extracting a Column Into a New Table (Table Decomposition)

**Context:** `products` table has a `metadata JSONB` column that has grown to 8KB average. Reads are slow because Postgres fetches the whole row. Need to move `metadata` to a `product_metadata` table.

**Phase 1 — Expand:**

```sql
-- Create new table
CREATE TABLE product_metadata (
    product_id BIGINT PRIMARY KEY REFERENCES products(id),
    metadata JSONB NOT NULL DEFAULT '{}'
);

-- Application: start writing to BOTH tables on every update
```

**Phase 2 — Backfill:**

```sql
-- Copy existing data (chunked)
INSERT INTO product_metadata (product_id, metadata)
SELECT id, metadata FROM products
WHERE id NOT IN (SELECT product_id FROM product_metadata)
ORDER BY id
LIMIT 10000;  -- repeat until done
```

**Phase 3 — Verify and switch:**

```sql
-- Verify row counts match
SELECT 
    (SELECT COUNT(*) FROM products) AS products_count,
    (SELECT COUNT(*) FROM product_metadata) AS metadata_count;

-- Application: stop writing to products.metadata, read only from product_metadata
```

**Phase 4 — Contract:**

```sql
-- Drop old column (after verifying zero reads from products.metadata)
SET lock_timeout = '2s';
ALTER TABLE products DROP COLUMN metadata;
```

---

### Scenario 3: Adding a Unique Constraint to an Existing Column

**Context:** `users.email` has duplicate values from a historical import. Need a unique constraint.

```sql
-- Step 1: Find duplicates
SELECT email, COUNT(*) 
FROM users 
GROUP BY email 
HAVING COUNT(*) > 1;

-- Step 2: Resolve duplicates (application-specific dedup logic)
-- E.g., keep the oldest account, merge or delete duplicates

-- Step 3: Create unique index concurrently (allows reads + writes during build)
CREATE UNIQUE INDEX CONCURRENTLY users_email_unique_idx ON users(email);

-- Step 4: If the CONCURRENTLY build fails (check for INVALID index)
SELECT indexname FROM pg_indexes 
JOIN pg_index ON (indexrelid = (SELECT oid FROM pg_class WHERE relname = indexname))
WHERE NOT indisvalid AND tablename = 'users';

-- If invalid, drop and retry:
DROP INDEX CONCURRENTLY users_email_unique_idx;

-- Step 5: Promote to constraint
ALTER TABLE users 
  ADD CONSTRAINT users_email_unique 
  UNIQUE USING INDEX users_email_unique_idx;
```

---

## 12. Tricky Scenarios

### Tricky Scenario 1: Renaming a Column With 50,000 Requests/Second

**Context:** `orders.customer_id` needs to be renamed to `orders.user_id` because the `customers` table is being renamed to `users`. The system processes 50K req/s and has 8 microservices all reading/writing `customer_id`.

**Why it's hard:** Any of the 8 services can be in any deploy state at any time. You can't coordinate a simultaneous deploy of all 8. A naive rename will take the system down.

**Solution: Expand-Contract over 3 deployments**

```
Deploy 0 (baseline):   All 8 services read/write customer_id
Deploy 1 (Expand):     ADD COLUMN user_id BIGINT
                       All 8 services dual-write customer_id AND user_id
                       Backfill job: SET user_id = customer_id WHERE user_id IS NULL
Deploy 2 (Cut over):   All 8 services read from user_id (with NULL fallback to customer_id)
                       All 8 services still write to both (safety net)
Deploy 3 (Contract):   All 8 services read and write user_id only
                       DROP COLUMN customer_id
```

**The dual-write middleware pattern** — instead of modifying every ORM model:

```python
class DualWriteMiddleware:
    """
    Intercepts writes to orders table and duplicates customer_id → user_id.
    Applied at the connection pool level, not per-service.
    """
    RENAME_MAP = {
        ('orders', 'customer_id'): 'user_id'
    }
    
    def rewrite_query(self, sql: str, params: tuple) -> tuple[str, tuple]:
        # Parse the SQL and inject dual-writes for mapped columns
        # Use sqlparse to safely handle this
        ...
```

**Monitoring the transition:**

```sql
-- Track how many rows have been successfully backfilled
SELECT 
    COUNT(*) FILTER (WHERE user_id IS NULL) AS not_migrated,
    COUNT(*) FILTER (WHERE user_id IS NOT NULL) AS migrated,
    ROUND(100.0 * COUNT(*) FILTER (WHERE user_id IS NOT NULL) / COUNT(*), 2) AS pct
FROM orders;
```

---

### Tricky Scenario 2: INT Overflow — Migrating a Primary Key From INT to BIGINT

**Context:** `orders.id` is `INT` (max ~2.1 billion). Current value: 1.95 billion. At 200 inserts/second, you have ~10 days before `INSERT` fails with "integer out of range". **This is an emergency.**

**Why it's catastrophically hard:**
- The `id` column is the primary key — changing it rebuilds the entire table AND all foreign keys pointing to it
- Other tables have `order_id INT` — they all need changing too
- At 200 rows/second, the ghost table copy will be racing against inserts for hours

**Step 1: Buy time with sequence manipulation (MySQL)**

```sql
-- MySQL: temporarily slow down the auto-increment to buy time
-- NOT a real fix, just buys hours
ALTER TABLE orders AUTO_INCREMENT = 1950000000;
```

**Step 2: gh-ost the primary key change**

```bash
gh-ost \
  --alter="MODIFY COLUMN id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT" \
  --table=orders \
  --execute \
  --chunk-size=500 \
  --max-load="Threads_running=60" \
  # Set postpone flag: manually control the cut-over window
  --postpone-cut-over-flag-file=/tmp/ghost.postpone.orders
```

**Step 3: Update foreign keys**

After orders.id is BIGINT, update all FK columns:

```bash
# Repeat gh-ost for each table with order_id INT
for table in order_items shipments refunds; do
  gh-ost --alter="MODIFY COLUMN order_id BIGINT UNSIGNED NOT NULL" \
         --table=$table --execute
done
```

**Step 4: Cut over**

```bash
# Remove the postpone flag at a low-traffic time to execute final rename
rm /tmp/ghost.postpone.orders
```

**Prevention:** Add an alert when sequence value exceeds 75% of INT max:

```sql
-- Postgres
SELECT 
    sequence_name,
    last_value,
    (last_value::numeric / 2147483647 * 100)::int AS pct_used
FROM information_schema.sequences
WHERE data_type = 'integer'
AND (last_value::numeric / 2147483647) > 0.75;
```

---

### Tricky Scenario 3: Zero-Downtime Multi-Tenant Schema Split

**Context:** A SaaS app has all tenants in a single `public` schema. You need to move large tenants into their own schemas for isolation. The `orders` table has 2 billion rows. You cannot afford a full outage.

**Phase 1: Route by tenant at the application layer**

```python
def get_db_schema(tenant_id: int) -> str:
    if tenant_id in ISOLATED_TENANTS:
        return f"tenant_{tenant_id}"
    return "public"

def execute_query(tenant_id: int, query: str, params: tuple):
    schema = get_db_schema(tenant_id)
    with get_connection(schema=schema) as conn:
        conn.execute(f"SET search_path TO {schema}, public")
        conn.execute(query, params)
```

**Phase 2: Create tenant schema and set up replication**

```sql
-- Create new schema for tenant
CREATE SCHEMA tenant_42;

-- Create shadow table in tenant schema
CREATE TABLE tenant_42.orders (LIKE public.orders INCLUDING ALL);

-- Create replication trigger on public.orders
CREATE OR REPLACE FUNCTION replicate_to_tenant_42()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.tenant_id = 42 THEN
        INSERT INTO tenant_42.orders VALUES (NEW.*)
        ON CONFLICT (id) DO UPDATE SET ... = EXCLUDED.*;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_replicate_tenant_42
AFTER INSERT OR UPDATE ON public.orders
FOR EACH ROW EXECUTE FUNCTION replicate_to_tenant_42();
```

**Phase 3: Backfill existing data**

```sql
INSERT INTO tenant_42.orders
SELECT * FROM public.orders WHERE tenant_id = 42
ON CONFLICT DO NOTHING;
```

**Phase 4: Cut over**

1. Feature flag: route tenant 42 reads to `tenant_42.orders`
2. Monitor for 1 hour
3. Remove trigger
4. Delete tenant 42 rows from `public.orders` (chunked DELETE)

---

## 13. Common Errors and Failure Modes

### Error 1: Lock Queue Poisoning

**What happens:** You run `ALTER TABLE` with no lock timeout. It queues behind a long-running SELECT (30 seconds). Every subsequent query — even simple SELECTs — queues behind your ALTER. 5,000 connections accumulate, PostgreSQL OOMs, crash.

**Diagnosis:**

```sql
SELECT pid, wait_event_type, wait_event, state, query_start, query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
ORDER BY query_start;
```

**Fix:**

```sql
-- Always set lock_timeout before DDL
SET lock_timeout = '3s';
ALTER TABLE ...;
-- If it times out, retry during low-traffic window
```

```sql
-- If already blocked: find the blocking PID and kill it if it's a safe idle transaction
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity 
WHERE pid IN (
    SELECT UNNEST(pg_blocking_pids(pg_backend_pid()))
);
```

---

### Error 2: Replica Lag Explosion During Backfill

**What happens:** Backfill runs with chunk size 50,000. Each commit is a large WAL event. Replicas fall 60+ seconds behind. Read replicas start returning stale data. Monitoring alerts fire. On-call scrambles.

**Diagnosis:**

```sql
-- On primary: check replica lag
SELECT client_addr, state, 
       pg_size_pretty(pg_wal_lsn_diff(sent_lsn, write_lsn)) AS write_lag,
       pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS replay_lag
FROM pg_stat_replication;
```

**Fix:** Reduce chunk size (1,000–5,000 rows), add sleep between chunks, implement backpressure (§8.5).

---

### Error 3: Ghost Table Accumulation

**What happens:** A gh-ost or pt-osc run is killed midway. The `_orders_gho` shadow table is left behind. Days later, disk is 40% consumed by ghost tables.

**Diagnosis:**

```sql
-- MySQL: find ghost tables
SELECT TABLE_NAME, TABLE_ROWS, 
       ROUND(DATA_LENGTH/1024/1024/1024, 2) AS data_gb
FROM information_schema.TABLES 
WHERE TABLE_SCHEMA = 'myapp'
AND TABLE_NAME LIKE '\_%\_gho'
ORDER BY DATA_LENGTH DESC;
```

**Fix:** Clean up manually. Add a cleanup step to your CI/CD pipeline after any failed migration run.

---

### Error 4: `CONCURRENTLY` Index Build Failure Leaving Invalid Index

**What happens:** `CREATE INDEX CONCURRENTLY` fails because of a constraint violation (duplicate value) or transaction conflict. An `INVALID` index is left behind — it takes up space and is maintained on every write, but never used by the planner.

**Diagnosis:**

```sql
SELECT schemaname, tablename, indexname
FROM pg_stat_user_indexes
JOIN pg_index USING (indexrelid)
WHERE NOT indisvalid;
```

**Fix:**

```sql
DROP INDEX CONCURRENTLY idx_users_email_unique_idx;
-- Resolve the underlying data issue (deduplicate)
-- Then recreate
CREATE UNIQUE INDEX CONCURRENTLY idx_users_email_unique_idx ON users(email);
```

---

### Error 5: Schema Migration in Wrong Transaction Isolation

**What happens:** Backfill runs inside `REPEATABLE READ` isolation. It reads a consistent snapshot of the table from the start of the transaction. Rows added after the snapshot started are never backfilled. Migration appears to complete, but new rows have NULL values.

**Fix:** Run backfills in `READ COMMITTED` (default). Each chunk reads the latest committed state.

---

### Error 6: Dual-Write Code Deployed But Migration Not Run

**What happens:** New code is deployed that writes to `full_name`. But the `ALTER TABLE ADD COLUMN full_name` hasn't run yet. Every INSERT fails with `column "full_name" does not exist`.

**Fix:** **Always run the DB migration before deploying the code that uses it.** In CI/CD:

```
1. Run migration (non-destructive, additive)
2. Deploy new code
3. Run backfill
4. (Later) Deploy code cleanup + run destructive migration
```

---

### Error 7: DROP COLUMN Before All Code Is Updated

**What happens:** Code v3 deploys. `DROP COLUMN name` runs. But there are 3 lagging pods still on v2 that read `name`. They crash with `column "name" does not exist`. Load balancer keeps sending traffic to the crashing pods.

**Fix:** Verify zero v2 pods before running DROP COLUMN. Use:

```bash
kubectl get pods -l app=my-service -o jsonpath='{.items[*].spec.containers[*].image}'
# Ensure all pods are on v3 image before proceeding
```

---

### Error 8: Sequence Gaps After Backfill

**What happens:** After an `INSERT INTO new_table SELECT * FROM old_table`, the sequence counter on `new_table.id` is still at 1 (it doesn't know about the copied rows). New inserts fail with duplicate key violations.

**Fix:**

```sql
-- After copying data, reset the sequence
SELECT setval('new_table_id_seq', (SELECT MAX(id) FROM new_table));
```

---

### Error 9: NOT VALID Constraint Never Validated

**What happens:** Team adds `ADD CONSTRAINT ... NOT VALID` as step 1. Backfills the data. Forgets to run `VALIDATE CONSTRAINT`. Months later, the constraint is assumed to be enforced but isn't. Bad data enters.

**Fix:** Track constraints awaiting validation:

```sql
SELECT conname, conrelid::regclass, contype, convalidated
FROM pg_constraint
WHERE NOT convalidated
AND contype = 'c';
```

Add an alert or CI check that fails if any unvalidated constraints exist for more than X days.

---

### Error 10: Migrating While a Streaming Replication Slot Is Stale

**What happens:** gh-ost reads the binlog via a replication slot. The migration is paused for 2 days. The replication slot keeps WAL from being recycled. Disk fills up. Database crashes.

**Diagnosis:**

```sql
-- Postgres: check replication slot WAL retention
SELECT slot_name, 
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots
WHERE active = false;  -- inactive slots are dangerous
```

**Fix:**

```sql
-- Drop stale slot (you will need to restart the consumer)
SELECT pg_drop_replication_slot('ghost_slot');
```

**Prevention:**

```ini
# postgresql.conf — limit WAL retention per slot (Postgres 13+)
max_slot_wal_keep_size = 10GB
```

---

## 14. Reference Case Studies

1. **GitHub Engineering — Renaming a Database Column** — The original post describing the Expand-Contract pattern applied to a live GitHub column rename with 60M rows. https://github.blog/2019-05-11-renaming-a-database-column/

2. **Stripe — Online Migrations at Scale** — How Stripe performs multi-step migrations across hundreds of millions of rows using dual writes and background workers. https://stripe.com/blog/online-migrations

3. **Shopify — Zero-Downtime Rails DB Migrations** — The strong_migrations gem philosophy: automatically detecting unsafe migrations in CI. https://shopify.engineering/zero-downtime-migrations-rails

4. **GitLab — Large Table Migration Strategy** — GitLab's documented approach using batched background migrations that run as sidekiq jobs, retryable and observable. https://docs.gitlab.com/ee/development/database/batched_background_migrations.html

5. **Notion — Herding Elephants: Lessons Learned From Sharding Postgres at Notion** — How Notion broke a single 5TB Postgres database into tenant-aware shards with zero downtime. https://www.notion.so/blog/herding-elephants

6. **Braintree — Safe DB Migrations** — Braintree's four-phase migration pattern (column exists in both old + new code). https://www.braintreepayments.com/blog/safe-operations-for-high-volume-postgresql

7. **gh-ost Documentation — GitHub Online Schema Transmogrifier** — Internals, flags, and production patterns. https://github.com/github/gh-ost

8. **Postgres Docs — Alter Table Locking** — Official documentation on lock levels taken by each DDL operation. https://www.postgresql.org/docs/current/explicit-locking.html

9. **PlanetScale — Non-Blocking Schema Changes** — How PlanetScale uses Vitess's online DDL to make all schema changes non-blocking by default. https://planetscale.com/blog/non-blocking-schema-changes

10. **AWS RDS — Best Practices for Upgrading** — Major version upgrade strategies for RDS Postgres including logical replication cutover. https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_UpgradeDBInstance.PostgreSQL.html
