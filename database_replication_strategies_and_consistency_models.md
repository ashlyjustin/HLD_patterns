# Replication: Strategies, Consistency, and Common Failure Modes

> **Goal:** Deeply understand every replication topology, how each database implements it, what consistency model you get, and what goes wrong in production — so you can design systems that survive real-world failures.

---

## 1. What Is Replication and Why Does It Exist?

Replication is the act of keeping copies of the same data on multiple nodes. It exists for three reasons:

| Goal | Problem It Solves |
|------|------------------|
| **High Availability** | If the primary node dies, a replica takes over — no data loss, no downtime |
| **Read Scalability** | Route read queries to replicas — 10 replicas ≈ 10x read throughput |
| **Geographic Locality** | Serve users from the nearest datacenter — reduce latency |

Every replication strategy involves a fundamental trade-off: the stronger the consistency guarantee you want, the more network round trips (and therefore latency and reduced availability) are required.

---

## 2. The Replication Topology Map

```
                    ┌─────────────────────────────────────┐
                    │         REPLICATION TOPOLOGIES       │
                    └─────────────────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          ▼                           ▼                           ▼
  Single Leader               Multi-Leader                  Leaderless
  (Leader-Follower)         (Active-Active)               (Dynamo-style)
          │                           │                           │
   ┌──────┴──────┐            ┌───────┴───────┐          ┌───────┴───────┐
   ▼             ▼            ▼               ▼          ▼               ▼
Sync Async    Conflict    Same-DC         Cross-DC    Quorum         Anti-entropy
repl  repl    resolution  multi-leader    multi-leader reads/writes   gossip
```

---

## 3. Single Leader Replication (Leader-Follower)

### How It Works

One designated node — the **leader** (also called primary or master) — accepts all writes. The leader records every write to its **replication log** (WAL in Postgres, binlog in MySQL). Followers (replicas, standbys) continuously apply the leader's log, staying in sync.

```
Client writes ──► Leader (primary)
                       │
              WAL / binlog stream
              ┌────────┼────────┐
              ▼        ▼        ▼
          Follower1 Follower2 Follower3
          (US-East) (EU-West) (AP-South)

Client reads ──► Any follower (or leader for strong reads)
```

### Synchronous vs. Asynchronous Replication

This is the single most impactful setting in any replication setup. It controls what happens between "write accepted by leader" and "write visible on followers."

#### Asynchronous Replication

The leader writes to its local log, acknowledges the client, and **then** streams the change to followers in the background. The follower might be milliseconds or seconds behind.

```
Timeline:
  T=0ms:   Client sends WRITE
  T=1ms:   Leader commits to local WAL → returns SUCCESS to client ✅
  T=2ms:   Leader begins streaming change to Follower 1
  T=4ms:   Follower 1 applies change
  T=180ms: Follower 2 (in EU) applies change (network latency)

T=3ms, client reads from Follower 1: sees the write ✅
T=3ms, client reads from Follower 2: does NOT see the write ❌ (replication lag)
```

**Durability risk:** If the leader crashes at T=1.5ms (after ACKing the client but before any follower applies), the write is **lost permanently**. The followers don't have it, and the client was told it succeeded.

**When to use:** Highest write throughput, lowest write latency. Acceptable when losing the last few milliseconds of writes is tolerable (social posts, analytics, activity logs).

#### Synchronous Replication

The leader waits for at least one follower to confirm it has written the change before acknowledging the client.

```
Timeline:
  T=0ms:   Client sends WRITE
  T=1ms:   Leader commits to local WAL
  T=1ms:   Leader sends change to synchronous follower
  T=6ms:   Synchronous follower confirms ← leader waits here
  T=6ms:   Leader returns SUCCESS to client ✅
  T=180ms: Async follower 2 applies change (not waited for)
```

**Durability guarantee:** If the leader crashes after T=6ms, the synchronous follower has the data. No writes are lost.

**Risk:** If the synchronous follower crashes or becomes slow, **writes stall** — the leader cannot accept new writes until the follower is available again. This is why most systems designate **one** synchronous follower and the rest async (semi-synchronous).

**Postgres configuration:**
```ini
# postgresql.conf on the primary
synchronous_standby_names = 'FIRST 1 (replica1, replica2)'
# Wait for exactly 1 of the listed replicas to confirm before committing
# 'ANY 2 (r1, r2, r3)' would wait for any 2 of 3
```

**MySQL semi-synchronous:**
```sql
-- MySQL: at least one replica must acknowledge before commit
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_timeout = 1000; -- fallback to async after 1s if no ack
```

#### The Replication Lag Problem

Async replication lag is measured in bytes behind (replication slot lag) and time (seconds behind primary).

```sql
-- Postgres: measure replication lag
SELECT
    client_addr,
    state,
    sent_lsn - write_lsn  AS write_lag_bytes,
    sent_lsn - flush_lsn  AS flush_lag_bytes,
    sent_lsn - replay_lsn AS replay_lag_bytes,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;

-- On a replica: how far behind is this replica?
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

**Dangerous condition: replication slot lag**
```sql
-- Postgres replication slots retain WAL until the slot consumer catches up.
-- If a replica goes offline and the slot is not dropped, WAL accumulates forever.
SELECT slot_name, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag
FROM pg_replication_slots;
-- If this grows to tens of GB, the primary disk fills up → DB crash
-- ALERT when slot lag > 1GB; DROP REPLICATION SLOT if replica is permanently gone
```

### Failover: How Leader Election Works

When the leader fails, a follower must be promoted. This is harder than it sounds.

```
Detection:
  Health check agent (Patroni, Orchestrator, MHA) monitors leader
  Leader fails to respond to heartbeats for N seconds (e.g., 30s)
  → Failover triggered

The problem of choosing the right follower:
  Follower 1: 0 bytes behind leader  ← ideal candidate
  Follower 2: 5MB behind leader      ← 5MB of writes it hasn't applied
  Follower 3: 50MB behind leader     ← very lagged

  If Follower 2 is promoted, those 5MB of writes that were on the leader are LOST
  (they were async-replicated but the leader died before followers got them)

Promotion:
  Patroni elects Follower 1 (most up-to-date)
  Patroni updates the config service (etcd/ZooKeeper/Consul) with new leader address
  Application's connection pool reconnects to new leader
  Followers 2 and 3 start replicating from the new leader
```

**Split brain during failover:** The old leader recovers from a transient network partition. Now there are two nodes that think they are the primary. Patroni uses etcd/ZooKeeper as the source of truth — only the node that holds the distributed lock is permitted to accept writes. The old leader will see it no longer holds the lock and demote itself.

**Real-world tools:**
- **Postgres:** Patroni (etcd/ZooKeeper/Consul) — industry standard
- **MySQL:** Orchestrator + ProxySQL, or MySQL Group Replication
- **Redis:** Redis Sentinel (monitors + automatic promotion), Redis Cluster (built-in)

---

## 4. Multi-Leader Replication (Active-Active)

### How It Works

Multiple nodes accept writes simultaneously. Every leader replicates to every other leader. Useful when you need writes to be accepted in multiple datacenters.

```
                DC1 (US-East)              DC2 (EU-West)
                    │                           │
Client US ──► Leader A ◄──── replication ────► Leader B ◄──── Client EU
                 │   ▲                        ▲   │
                 │   └──────────────────────────┘ │
              Followers                       Followers
```

### The Fundamental Problem: Write Conflicts

Two clients in different datacenters update the same record simultaneously. Both writes are accepted by their respective leaders and are locally valid. When they replicate to each other, they conflict.

```
T=0ms: Client EU updates title: "Trip to Paris"  → Leader B accepts, commits
T=0ms: Client US updates title: "Trip to London" → Leader A accepts, commits

T=100ms: Leader A receives "Trip to Paris" from B
         Leader B receives "Trip to London" from A
         
         Both leaders: "I have two different values for the same row. What do I do?"
```

This is an **irreconcilable conflict** without a resolution policy.

### Conflict Resolution Strategies

#### Last Write Wins (LWW)

Each write is timestamped. The write with the latest timestamp wins. Concurrent conflicting writes: the later one overwrites the earlier one.

```python
def resolve_conflict(local_version, remote_version):
    if remote_version.timestamp > local_version.timestamp:
        return remote_version  # remote wins
    return local_version  # local wins
```

**Problem:** Requires synchronized clocks. With clock skew, a "later" timestamp may represent an actually earlier write. LWW **silently drops data** — the losing write disappears without any error or indication.

**Who uses it:** Cassandra's default conflict resolution is LWW per column (using timestamps set by the client). This is why Cassandra timestamp generation must use a monotonically increasing clock — application bugs setting wrong timestamps cause data corruption.

#### Last Write Wins — Safer Version (CAS + Fencing)

```python
# Safer: use a server-assigned monotonic version, not wall clock
def update_title(doc_id, new_title, expected_version):
    db.execute("""
        UPDATE docs SET title = %s, version = version + 1
        WHERE id = %s AND version = %s
    """, [new_title, doc_id, expected_version])
    # If 0 rows updated → conflict detected → retry or reject
```

#### Custom Conflict Resolution (Application Logic)

The application receives both conflicting versions and decides:

```python
# CouchDB / DynamoDB: expose conflicts to application
def resolve_title_conflict(versions):
    # Domain-specific: for a shopping cart, merge items from both versions
    all_items = set()
    for v in versions:
        all_items.update(v.items)
    return ShoppingCart(items=list(all_items))  # Union of both carts — no data loss

# For a bank account balance: this is NOT safe — you cannot simply take the highest value
# Multi-leader is inappropriate for financial ledgers
```

#### CRDTs (Conflict-free Replicated Data Types)

Data structures that are mathematically designed to always merge without conflicts, regardless of order.

```python
# G-Counter CRDT: each node has its own counter, total = sum of all
class GCounter:
    def __init__(self, node_id, num_nodes):
        self.node_id = node_id
        self.counts = [0] * num_nodes

    def increment(self):
        self.counts[self.node_id] += 1

    def value(self):
        return sum(self.counts)

    def merge(self, other):
        # Merge = element-wise max: always safe, commutative, associative
        self.counts = [max(a, b) for a, b in zip(self.counts, other.counts)]

# Node A: counts=[5, 0]  (incremented 5 times)
# Node B: counts=[0, 3]  (incremented 3 times)
# After merge on either node: counts=[5, 3], value = 8 ✅
# Order of merge doesn't matter — same result always
```

**CRDT types in practice:**
| CRDT | Use Case | Behavior on Conflict |
|------|----------|---------------------|
| G-Counter | View counts, like counts | Sum all nodes' counts |
| PN-Counter | Inventory (inc + dec) | Sum of increments minus sum of decrements |
| OR-Set (Observed-Remove Set) | Shopping cart, tags | Union of adds; removes tagged with unique ID |
| LWW-Register | Single value with timestamp | Latest timestamp wins |
| MV-Register | Wiki page edits | Preserve all concurrent versions; user resolves |

**Who uses CRDTs:** Riak, Redis Enterprise (multi-region), Automerge (collaborative editing), Figma (real-time design), Apple's CloudKit.

### Multi-Leader Topologies

```
Circular:  A → B → C → A
  Risk: one node failure breaks the chain; a write loops forever if not tracked

Star:      B
           ↑
  A ←──── Hub ────► C
           ↓
           D
  Risk: Hub is a bottleneck and single point of failure

All-to-all: Every node replicates to every other node
  Best fault tolerance; but write ordering can be skewed
  (A→C may arrive before A→B→C even though B→C happened after A→B)
```

**All-to-all with ordering problem:**

```
Leader A: INSERT order 1 (at T=1)
Leader B: UPDATE order 1 price (at T=2, after replicating from A)

At Leader C:
  Receives B's UPDATE at T=2 first (faster network path)
  Receives A's INSERT at T=3 (slower path)
  
  C tries to UPDATE order 1 — but the INSERT hasn't arrived yet!
  C rejects or applies incorrectly.
```

**Fix:** Version vectors (each node tracks which other nodes' writes it has seen) allow detection of out-of-order replication.

### When to Use Multi-Leader

✅ **Appropriate:** Multiple datacenters that must accept writes independently; offline-first mobile apps (device is its own leader, syncs when online); collaborative editing (Google Docs — each client is a leader, OT/CRDTs handle conflicts)

❌ **Avoid for:** Anything requiring strict consistency (payments, inventory, unique constraint enforcement) — conflicts are unavoidable and cannot always be resolved automatically

---

## 5. Leaderless Replication (Dynamo-Style)

### How It Works

No designated leader. Any node can accept reads and writes. The client (or a coordinator node) sends requests to multiple replicas simultaneously. **Quorum** determines success.

```
Client WRITE (w=2, n=3):
  Send to all 3 replicas simultaneously
  ┌──► Node 1: writes ✅
  ├──► Node 2: writes ✅   ← 2 of 3 confirmed → return success to client
  └──► Node 3: slow / down ← not waited for

Client READ (r=2, n=3):
  Send to all 3 replicas simultaneously
  ┌──► Node 1: returns v1 ✅
  ├──► Node 2: returns v1 ✅   ← 2 of 3 agree → return v1 to client
  └──► Node 3: returns v0 (stale, didn't get the write) ← stale!
```

The client reads multiple versions and **picks the newest** (using version vectors or timestamps). Stale replicas are repaired in the background.

### Quorum Mechanics

```
N = total replicas (replication factor)
W = number of replicas that must confirm a write
R = number of replicas that must respond to a read

Quorum condition for strong consistency: W + R > N

Common configurations:
  N=3, W=2, R=2: Strong consistency (2+2 > 3) ← "QUORUM" in Cassandra
  N=3, W=1, R=1: Weakest. Fast but stale reads possible ← "ONE" in Cassandra
  N=3, W=3, R=1: Writes to all, reads from one. Strong but writes slow if any node down
  N=3, W=1, R=3: Writes fast, reads slow. Only useful for analytics
```

**Why W + R > N guarantees overlap:**

```
N=3 nodes: [A, B, C]
W=2 write: write landed on [A, B]
R=2 read:  reads from [A, C]  ← A is in both sets! A has the latest write.
           reads from [B, C]  ← B is in both sets!
           reads from [A, B]  ← Both have it!
           
Any 2 nodes read must overlap with the 2 nodes written to.
At least one node in every R-set has the latest version → read returns latest.
```

**When the quorum condition is violated (stale reads are possible):**

```
N=3, W=1, R=1:
  Write to [A] only
  Read from [C] only → C never got the write → stale read ✅ possible
```

### Read Repair and Anti-Entropy

When a read returns different versions from different replicas, the coordinator detects the discrepancy and repairs the stale replica:

```python
def quorum_read(key, r=2):
    responses = []
    for node in random.sample(replicas, len(replicas)):
        responses.append(node.get(key))
        if len(responses) >= r:
            break

    # Detect stale replicas
    latest = max(responses, key=lambda x: x.version)
    stale_nodes = [r.node for r in responses if r.version < latest.version]

    # Read repair: asynchronously update stale nodes
    for node in stale_nodes:
        asyncio.create_task(node.put(key, latest.value, latest.version))

    return latest.value
```

**Anti-entropy process:** A background job continuously compares data between replicas using **Merkle trees** (hash trees). The root hash of a Merkle tree covers all the data; if two replicas have the same root hash, their data is identical. If different, walk the tree to find exactly which keys differ — highly efficient for large datasets.

```
Merkle tree (simplified):
  Root hash H(H(H1,H2), H(H3,H4))
       │
   ┌───┴───┐
 H(H1,H2) H(H3,H4)
   │         │
 H1  H2    H3  H4
 (key1-25) (key26-50) ...

Compare root hashes between replicas.
If root differs → compare children → find exactly which leaf (key range) differs
→ Sync only the differing keys
```

### Sloppy Quorum and Hinted Handoff

When `W` or `R` nodes are unavailable (network partition), should the operation:
- **Fail:** Wait for exactly W/R nodes from the preferred replica set (strict quorum)
- **Sloppy quorum:** Accept writes/reads from any available nodes, even outside the preferred replica set — and apply a **hinted handoff** to deliver the write to the correct replica when it recovers

```
Normal: write should go to nodes [A, B, C]. A and B are down.
Sloppy quorum: write to [C, D, E] instead (available nodes, not the "home" nodes)
               Store hint: "this data belongs to A and B when they recover"

When A recovers:
  D detects A is back (via gossip)
  D sends the hinted write to A
  A applies it
  D deletes the hint
```

**Trade-off:** Sloppy quorum increases availability but **does not provide quorum reads** — a read from [A, B] will not see the write that went to [D, E] during the outage. Sloppy quorum sacrifices the W+R > N consistency guarantee temporarily. Cassandra enables sloppy quorum by default.

---

## 6. Quorum in Consensus Protocols (Raft/Paxos)

Database replication quorums (Cassandra-style) should not be confused with **consensus quorums** (Raft/Paxos-style). They are different.

| | Cassandra Quorum | Raft/Paxos Quorum |
|--|--|--|
| **Purpose** | Overlap reads and writes to find latest version | Elect a leader; agree on a single value |
| **Conflict handling** | Last-write-wins or CRDT merge | Single serialized log; no conflicts possible |
| **Consistency** | Tunable (can be eventually consistent) | Always linearizable |
| **Leader** | None | Required (Raft leader, Paxos proposer) |
| **Used in** | Cassandra, DynamoDB, Riak | etcd, CockroachDB, Spanner, Kafka (KRaft) |

### How Raft Works

Raft is the consensus algorithm behind etcd, CockroachDB, TiDB, and Kafka's KRaft mode. It guarantees that all replicas agree on the same sequence of log entries — making the replicated state machine identical across all nodes.

```
Raft roles:
  Leader:    Handles all writes. Replicates entries to followers.
  Follower:  Applies entries from leader's log.
  Candidate: Seeking votes to become leader (during election).

Raft log replication:
  1. Client sends write to leader
  2. Leader appends entry to its log (uncommitted)
  3. Leader sends AppendEntries RPC to all followers in parallel
  4. Followers append entry to their logs, respond OK
  5. When leader receives majority (quorum) of OKs:
       → Leader marks entry as committed
       → Leader applies to state machine
       → Leader returns success to client
       → Leader notifies followers to commit (next AppendEntries)

Crash safety:
  If leader crashes after step 5 but before notifying followers:
    → New leader election
    → New leader has the committed entry (it was in the majority)
    → New leader replays it to followers
    → No data loss
```

**Raft leader election:**
```
All followers start an election timer (randomized 150-300ms).
If no heartbeat from leader before timer expires → start election:
  Node increments its term, votes for itself, requests votes from all others
  
A node wins if it receives votes from a majority (N/2 + 1).
  → A node only votes for a candidate if the candidate's log is at least as
    up-to-date as the voter's log (prevents electing a stale node as leader)

Randomized timers prevent split votes most of the time.
```

**When Raft blocks (minority partition):**
```
5-node cluster: [A, B, C, D, E]
Network partition: [A, B] | [C, D, E]

[C, D, E] partition: has majority (3/5) → elects a leader → continues normally
[A, B] partition:    only 2 nodes → cannot reach quorum → no writes accepted → blocked (CP)

When partition heals: A and B rejoin, replicate from new leader, discard their uncommitted entries
```

---

## 7. How Specific Databases Handle Replication

### 7.1 PostgreSQL

**Mechanism:** WAL (Write-Ahead Log) streaming replication. The primary continuously streams WAL records to standbys. Standbys apply WAL in sequence — they are physically identical to the primary at any point in time (byte-for-byte same data files, same WAL applied).

**Replication types:**
```
Streaming replication (default): standby connects to primary via TCP,
  streams WAL records continuously in near-real-time.
  
Logical replication: replicates at the SQL statement level (row changes),
  not at the binary WAL level. Allows:
  - Replication to a different Postgres version
  - Selective table replication
  - Replication to non-Postgres databases (via logical decoding plugins like Debezium)
```

**Consistency:** Synchronous standbys provide strong consistency (zero data loss on failover). Async standbys have replication lag (typically <100ms in same DC, seconds in cross-DC).

**Failover:** Patroni uses etcd/Consul to elect a leader. Only the node holding the distributed lock is the primary. Other nodes are standbys. Patroni calls `pg_promote()` to promote a standby to primary.

**Replication slots (important caveat):**
```sql
-- A replication slot makes Postgres retain WAL until the subscriber consumes it.
-- If a subscriber (replica or logical consumer like Debezium) goes offline,
-- WAL accumulates indefinitely → disk fills up → DB crash.

-- MONITOR THIS:
SELECT slot_name,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots
WHERE active = false;  -- Inactive slots are most dangerous

-- If a slot has been inactive for >1 hour and WAL > 5GB: drop it
SELECT pg_drop_replication_slot('dead_replica_slot');
```

---

### 7.2 MySQL / InnoDB

**Mechanism:** Binary log (binlog) replication. The primary records all changes to the binlog. Replicas read from the binlog and apply changes via a **SQL thread**.

**Replication modes:**
```
Statement-based (SBR): Log the SQL statement itself.
  Risk: Non-deterministic functions (NOW(), UUID(), RAND()) produce different
        results on replica → data divergence.

Row-based (RBR): Log the actual row values changed (before + after image).
  Safe: Always produces identical result on replica.
  More verbose: large batch UPDATEs generate large binlog entries.

Mixed: Use SBR for safe statements, fall back to RBR for non-deterministic ones.
  Default in MySQL 8.0.
```

**GTID (Global Transaction Identifiers):**
```sql
-- GTIDs assign a globally unique ID to every transaction.
-- Replicas track which GTIDs they've applied.
-- Failover: replica knows exactly which transactions it's missing.
-- Much simpler than position-based replication for failover.

-- Check GTID position:
SHOW MASTER STATUS;  -- On primary: shows current GTID position
SHOW SLAVE STATUS\G  -- On replica: shows which GTIDs have been applied, lag

-- Example GTID: 3E11FA47-71CA-11E1-9E33-C80AA9429562:1-23
-- Format: server_uuid:transaction_sequence_number
```

**MySQL Group Replication:** Built-in multi-master replication using Paxos-based consensus. Up to 9 nodes. Certifies transactions across the group before commit — provides conflict detection (not resolution). Conflicts cause one transaction to roll back. Basis for MySQL InnoDB Cluster.

---

### 7.3 Apache Cassandra

**Mechanism:** Leaderless replication with consistent hashing. Each row's partition key determines which N nodes are its "natural" replicas (determined by the hash ring). Any node can act as coordinator for a client request.

**Replication factor and consistency levels:**
```sql
-- Keyspace with RF=3 (3 copies of every partition)
CREATE KEYSPACE orders WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'us-east': 3,
    'eu-west': 2   -- 5 total replicas: 3 in US, 2 in EU
};

-- Per-query consistency level (can be different for reads vs writes):
-- ONE:           1 replica must confirm (fastest, least consistent)
-- QUORUM:        (RF/2)+1 replicas must confirm (balanced)
-- LOCAL_QUORUM:  quorum within the local DC only (avoids cross-DC latency)
-- EACH_QUORUM:   quorum in EVERY DC (strongest multi-DC consistency)
-- ALL:           all replicas must confirm (slowest, most consistent)
-- ANY:           even a coordinator hint counts as a write (loosest possible)
```

**Tombstones — Cassandra's silent killer:**
Cassandra cannot delete data immediately — a deleted row must be replicated to all replicas, but some may be offline. Cassandra uses **tombstones** — deletion markers that propagate through replication and suppress reads for the row.

```
Problem: If a replica was offline during the delete, it never gets the tombstone.
When it comes back online, the delete is already past the gc_grace_seconds TTL
(default: 10 days). The tombstone is garbage-collected on other nodes.
The offline replica's un-deleted row is now the only version → it resurfaces!
This is called "zombie data" or "tombstone resurrection."

Prevention:
  - Keep gc_grace_seconds long enough that all replicas sync before GC
  - Use node repair regularly: nodetool repair -full
  - Monitor replica availability — don't let nodes stay offline > gc_grace_seconds
```

**Consistency in multi-DC:**
```
Write with LOCAL_QUORUM (RF=3 per DC, 2 DCs):
  - Waits for 2/3 nodes in LOCAL DC only
  - Async to the other DC
  
Read with LOCAL_QUORUM:
  - Reads from 2/3 local DC nodes
  - Will NOT see writes that only exist on remote DC yet
  
This means LOCAL_QUORUM provides strong consistency within a DC
but eventual consistency across DCs.

For cross-DC strong consistency: use EACH_QUORUM (write) + EACH_QUORUM (read)
  - Must reach quorum in EVERY DC before returning
  - Adds cross-DC network latency (~80-150ms for US↔EU)
  - Both DCs must be available
```

---

### 7.4 MongoDB

**Mechanism:** Replica sets — a primary and multiple secondaries. Primary receives all writes, secondaries replicate from the primary's **oplog** (a capped collection of operations). Up to 50 members, typically 3 or 5 nodes.

**Write concern:**
```javascript
// Write concern: how many replicas must confirm before returning success
db.orders.insertOne(
    { orderId: 123, amount: 99.99 },
    { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
)
// w: "majority" → majority of replica set members must confirm
// j: true       → each confirming member must write to its journal (disk) first
// wtimeout: 5000 → if majority not reached in 5s → return error (don't block forever)
```

**Read preference:**
```javascript
// Control which nodes service reads
db.orders.find({userId: 42}).readPref("secondary")
// primary:            Always read from primary (strong consistency)
// primaryPreferred:   Primary if available, fallback to secondary
// secondary:          Always secondary (potentially stale)
// secondaryPreferred: Secondary if available, fallback to primary
// nearest:            Lowest latency node (ignores primary/secondary)
```

**Causal consistency sessions:**
```javascript
// MongoDB 3.6+: causally consistent sessions
const session = client.startSession({ causalConsistency: true });
// All operations in this session are guaranteed to be causally consistent:
// A read will always see the results of previous writes in the same session,
// even if reads go to a secondary with replication lag.
session.endSession();
```

---

### 7.5 Redis

**Mechanism:** Redis Sentinel (for HA with single-leader) or Redis Cluster (for sharding + replication).

**Redis Sentinel:**
```
Sentinel 1, 2, 3 monitor the primary Redis node.
If primary is down (majority of sentinels agree): failover.
Sentinel elects a replica as the new primary via its own quorum.
Sentinels reconfigure other replicas to follow the new primary.
Sentinels update the DNS/service discovery so clients find the new primary.
```

**Redis Cluster replication:**
```
16384 hash slots divided across master nodes.
Each master has 0-N replicas.
Writes go to the master for the relevant slot.
Replicas are async by default.

WAIT command: make a write synchronous to N replicas
  WAIT 1 100  ← wait for at least 1 replica to confirm, timeout 100ms
              ← returns number of replicas that confirmed
              ← useful for critical operations, but redis replication is still
                 best-effort (if replica crashes, wait may return before it persists)
```

**Redis replication lag risk:** Redis replication is **always asynchronous**. A primary crash loses the last N writes (typically <1ms worth). This is acceptable for caching but not for financial data. Never use Redis as the sole store for data that cannot be lost.

---

### 7.6 CockroachDB / Google Spanner (Raft-Based NewSQL)

**Mechanism:** Data is divided into ranges (key ranges). Each range is replicated across 3 or 5 nodes using **Raft consensus**. The Raft leader for each range accepts writes for that range.

```
Table: accounts (partitioned by account_id)
  Range 1 (accounts 1-10000):   Raft group [Node1(leader), Node2, Node3]
  Range 2 (accounts 10001-20000): Raft group [Node2(leader), Node3, Node4]
  Range 3 (accounts 20001-30000): Raft group [Node3(leader), Node4, Node5]

Each range has an independent Raft group.
A node can be the Raft leader for some ranges and a follower for others.
No single bottleneck — write throughput scales with number of ranges.
```

**Consistency:** Always linearizable (equivalent to SERIALIZABLE at the transaction level). Every committed write is immediately visible to all subsequent reads, anywhere in the cluster, regardless of which node serves the read.

**Cross-range transactions:** Use a two-phase protocol built on top of Raft: the transaction coordinator performs a Raft-committed prepare on all involved ranges, then a Raft-committed commit. All with distributed timestamps (HLC in CockroachDB, TrueTime in Spanner).

---

## 8. Eventual Consistency: Deep Dive

### What "Eventual" Actually Means

"Eventual consistency" is often cited but rarely defined precisely. The formal definition:

> **If no new updates are made to a data item, eventually all accesses to that item will return the last updated value.**

This tells you almost nothing practically useful. The actual questions are:
1. **How eventual?** Seconds, minutes, hours? (Typically: milliseconds to seconds in same-DC, seconds to minutes cross-DC or after network partition)
2. **What can go wrong in the meantime?** Lost updates? Stale reads? Write skew? Conflicting versions?
3. **Who resolves conflicts?** The DB automatically? Your application code?

### The Consistency Spectrum

```
Stronger ◄──────────────────────────────────────────────► Weaker
    │                                                         │
    │ Linearizable   Serializable   Causal    Monotonic   Eventual
    │ (Spanner)      (Postgres SSI) (Mongo    Read        (Cassandra
    │                               Sessions) Consistent   ANY/ONE)
    │
    │ Every read sees     Txns appear    A process     Reads of         May read
    │ latest committed    in some        sees its own  a key never      arbitrarily
    │ write, globally     serial order   writes        go backward      stale data
```

### Levels of Eventual Consistency

These are not formal standards but practically important distinctions:

#### Eventual Consistency (Weakest)

No guarantees about when a write becomes visible. A read after a write may return stale data indefinitely until the system converges.

```
Node A writes: user.email = "new@example.com"
Node B (replica, 2s behind): still returns "old@example.com"
Node C (replica, 500ms behind): returns "new@example.com"

Two requests from the same client going to different nodes:
  Request 1 → Node C: sees "new@example.com"
  Request 2 → Node B: sees "old@example.com"  ← went backward!
```

#### Monotonic Read Consistency

Once a client has seen a value, it will never see an older value in a later read. Reads may still be stale, but they won't go backward.

```python
# Implementation: route client to same replica for duration of session
# Or: track the client's read timestamp, route to replica that has caught up past it
def read(key, client_last_read_ts):
    # Route to a replica that has applied at least client_last_read_ts
    replica = pick_replica_with_ts_gte(client_last_read_ts)
    result = replica.get(key)
    return result, result.timestamp  # Return new ts for client to track
```

#### Read-Your-Own-Writes Consistency

A client always sees its own writes. Other clients may still see stale data.

```python
# After a write, store the write's timestamp in the client session
def write(key, value):
    ts = primary.put(key, value)
    session["write_ts"] = ts  # Remember when we wrote
    return ts

def read(key):
    if "write_ts" in session:
        # Must read from a replica that has applied up to write_ts
        return replica_with_ts(session["write_ts"]).get(key)
    return any_replica.get(key)
```

#### Causal Consistency

If operation B is causally dependent on operation A (B happened after A and possibly because of A), then any process that sees B also sees A. Unrelated operations may still be reordered.

```
Thread 1: posts a comment (op A)
Thread 2: "likes" the comment (op B — causally depends on A)

Causal consistency guarantees: no process sees the "like" without also seeing the comment.
```

MongoDB sessions provide causal consistency. Spanner and CockroachDB provide linearizability (strictly stronger than causal).

#### Strong Eventual Consistency (CRDTs)

All replicas that have received the same set of updates will have the same state — regardless of the order in which they received them. This is the guarantee CRDTs provide. No coordination needed; no conflicts possible.

### Practical Eventual Consistency Patterns

#### Pattern: Version Vectors (Vector Clocks)

Track which writes each node has seen to detect concurrent writes and causal ordering:

```python
# Each data item carries a version vector
class VersionVector:
    def __init__(self):
        self.clocks = {}  # {node_id: counter}

    def increment(self, node_id):
        self.clocks[node_id] = self.clocks.get(node_id, 0) + 1

    def dominates(self, other):
        """Self is strictly newer than other if all self counters >= other counters"""
        return all(self.clocks.get(n, 0) >= c for n, c in other.clocks.items())

    def concurrent_with(self, other):
        """Neither dominates → concurrent writes → conflict"""
        return not self.dominates(other) and not other.dominates(self)

# On write:
vv = VersionVector()
vv.increment("node_A")  # node A wrote this
db.put(key, value, vv)

# On merge (two versions found during read):
v1 = db.get_version(key, replica=1)  # vv: {A: 3, B: 2}
v2 = db.get_version(key, replica=2)  # vv: {A: 3, B: 1}

if v1.vv.dominates(v2.vv):
    return v1  # v1 is newer, use it
elif v2.vv.dominates(v1.vv):
    return v2
else:
    return resolve_conflict(v1, v2)  # Truly concurrent → application decides
```

#### Pattern: Read Repair on Divergence

```python
def read_with_repair(key, quorum=2):
    # Read from multiple replicas
    results = [replica.get(key) for replica in random.sample(replicas, 3)]
    results = [r for r in results if r is not None]

    if len(results) < quorum:
        raise InsufficientReplicas()

    # Find newest version
    latest = max(results, key=lambda r: r.version)
    stale = [r for r in results if r.version < latest.version]

    # Repair stale replicas asynchronously
    for stale_result in stale:
        asyncio.create_task(stale_result.node.put(key, latest.value, latest.version))

    return latest.value
```

---

## 9. Common Replication Problems and How to Diagnose Them

### Problem 1: Replication Lag Spike Causing Read-After-Write Failures

**Symptom:** After updating their profile, users intermittently see the old version. Happens more under high write load.

**Root cause:** Application reads from async replicas. Under write load, replication lag grows from 10ms to 5s.

```sql
-- Postgres: measure current lag per replica
SELECT client_addr, replay_lag FROM pg_stat_replication;
-- replay_lag = 00:00:04.312 ← 4 seconds behind on a loaded primary

-- Identify which queries are causing write load:
SELECT query, calls, total_exec_time, rows
FROM pg_stat_statements
WHERE query LIKE 'UPDATE%' OR query LIKE 'INSERT%'
ORDER BY total_exec_time DESC LIMIT 10;
```

**Fixes:**
1. Route writes' immediate reads to primary (session affinity, 30s window)
2. Tune replica: `max_standby_streaming_delay = 30s`, `hot_standby_feedback = on`
3. Reduce write load: batch writes, use write buffers
4. Use synchronous replication for the specific tables causing user-visible issues

---

### Problem 2: Replication Slot WAL Accumulation (Disk Full)

**Symptom:** Primary DB disk usage grows at an alarming rate. Eventually DB crashes.

**Root cause:** A replication slot was created for a replica (or Debezium CDC consumer) that went offline. Postgres retains all WAL since the slot's last confirmed LSN — indefinitely.

```sql
-- EMERGENCY: find inactive slots consuming disk
SELECT slot_name, active, restart_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots
ORDER BY restart_lsn;

-- slot_name='debezium_slot', active=false, retained='47 GB' ← DANGER

-- If the consumer is gone for good, drop the slot:
SELECT pg_drop_replication_slot('debezium_slot');

-- Prevention: set max_slot_wal_keep_size (Postgres 13+)
-- Postgres will drop the slot automatically if it would consume more than this:
ALTER SYSTEM SET max_slot_wal_keep_size = '10GB';
SELECT pg_reload_conf();
```

---

### Problem 3: MySQL Replica Drift (Silent Data Divergence)

**Symptom:** No error visible, but replica data silently differs from primary after weeks. Discovered when the replica is promoted during failover.

**Root causes:**
- Statement-based replication with non-deterministic functions
- Direct writes to the replica (breaking the replication invariant)
- `IGNORE` errors in replication thread skipping failed statements

```sql
-- Detect if replica has accepted direct writes (or has drifted):
-- On replica: this should show zero rows; if not, someone wrote directly
SHOW SLAVE STATUS\G
-- Look for: Seconds_Behind_Master, Last_Error, Last_SQL_Error

-- Pt-table-checksum: tool that checksums tables on primary vs replicas
pt-table-checksum --host primary.db --user root --password ...
-- Reports any tables that differ between primary and replica

-- Pt-table-sync: repair drifted replica (use with caution in production)
pt-table-sync --execute --sync-to-master replica.db
```

**Prevention:**
- Use RBR (row-based replication) — eliminates non-determinism
- Set `read_only = ON` on all replicas — prevents direct writes
- Set `slave_skip_errors = OFF` — never silently skip replication errors
- Use GTIDs — makes it impossible to "re-apply" already-applied transactions

---

### Problem 4: Cassandra Replica Inconsistency After Node Outage

**Symptom:** After a Cassandra node is offline for 3 days and comes back, reads return inconsistent data — sometimes old values, sometimes correct ones.

**Root cause:** Hints expired (default: 3 hours). The node missed writes that were hinted but the hints were discarded after the TTL. Read repair hasn't fully converged yet.

```bash
# Check if a node needs repair after coming back
nodetool status  # Look for nodes in DN (Down Normal) or leaving state

# Run full repair after bringing node back
nodetool repair -full -pr  # -pr = primary range only (faster, less network)
# Full repair (without -pr) repairs the entire keyspace including secondary replicas

# Check repair status
nodetool compactionstats  # Shows repair progress

# Monitor tombstone counts (high = potential zombie data)
nodetool tablestats keyspace.table | grep 'Tombstone'

# Set hint_window_persistent_period appropriately:
# In cassandra.yaml:
# max_hint_window: 3h  ← Increase if nodes can be offline longer
# hinted_handoff_enabled: true
```

---

### Problem 5: Redis Split Brain During Network Partition

**Symptom:** Two Redis masters are accepting writes after a Sentinel-triggered failover. When the network heals, one master's data is silently discarded.

**Root cause:** Old primary was slow (not crashed) — Sentinel declared it dead, promoted a replica. Old primary continues serving writes (it thinks it's still primary). When partition heals, the old primary is demoted and loses all writes it accepted while demoted.

```
T=0:  Primary (P) is slow due to GC pause. Sentinels timeout.
T=5s: Sentinel elects Replica R as new primary.
T=5s-T=30s: OLD Primary P resumes. Accepts writes from clients that didn't update config.
             NEW Primary R also accepts writes.
T=30s: P detects it's been demoted (via Sentinel). P becomes a replica of R.
       P's writes (T=5-30s) are DISCARDED.
```

**Fixes:**
```
1. Redis MIN-SLAVES-TO-WRITE:
   min-slaves-to-write 1  ← Primary refuses writes if it has 0 sync'd replicas
   min-slaves-max-lag 10  ← Primary refuses writes if replicas are >10s behind
   
   Effect: Old primary, now isolated from all replicas, stops accepting writes.
   Clients get errors, but no split-brain.

2. Application-side fencing:
   On receiving a connection error (old primary rejecting writes),
   application re-discovers primary via Sentinel and reconnects.
   
3. Use Redis Cluster instead of Sentinel:
   Redis Cluster uses its own consensus for failure detection,
   reducing the split-brain window significantly.
```

---

### Problem 6: Raft Election Storm (etcd / CockroachDB)

**Symptom:** Cluster becomes unavailable periodically (seconds at a time). etcd logs show frequent leader elections.

**Root causes:**
- Disk I/O too slow — leader cannot write to WAL fast enough → heartbeat timeout → followers start elections
- Network jitter causing heartbeat timeouts
- etcd on HDD instead of SSD — etcd requires `fsync` on every commit; HDD `fsync` ~5ms, but heartbeat timeout may be set to 100ms — a burst of writes can cause cascading timeouts

```bash
# Check etcd health
etcdctl endpoint health

# Check leader changes (too many = election storm)
etcdctl endpoint status --cluster

# Check disk latency (must be <10ms for etcd to be stable)
# etcd logs: "slow fdatasync" warning is a red flag
journalctl -u etcd | grep "slow fdatasync"

# etcd tuning for slow disks:
# --heartbeat-interval=250       ← increase from default 100ms
# --election-timeout=2500        ← 10x heartbeat interval
# --snapshot-count=5000          ← fewer snapshots = less disk I/O during normal operation
```

---

## 10. The CAP Theorem: What It Actually Means for Replication

The CAP theorem states that a distributed system can guarantee only 2 of 3:
- **C**onsistency: every read sees the most recent write
- **A**vailability: every request receives a response (not an error)
- **P**artition tolerance: the system continues operating despite network partitions

**The critical nuance:** Partition tolerance is not optional — network partitions happen in any distributed system. The real choice is between **C and A when a partition occurs**:

```
Partition happens:
  CP system: stop accepting writes to unavailable partition → consistent but unavailable
  AP system: continue accepting writes on both sides → available but potentially inconsistent
```

### PACELC: A Better Model

PACELC extends CAP with the non-partition case:

```
If Partition: choose between Availability or Consistency (CAP)
Else (no partition, normal operation): choose between Latency or Consistency

PACELC(PA/EL) = available during partitions + low latency during normal operation
  → Cassandra (ONE/ANY consistency), DynamoDB default

PACELC(PC/EC) = consistent during partitions + consistent during normal operation
  → Spanner, CockroachDB, etcd, Postgres sync replication

PACELC(PA/EC) = available during partitions + consistent during normal operation
  → MongoDB (w: majority with j: true)
```

| Database | Partition behavior | Normal operation | Classification |
|----------|--------------------|-----------------|----------------|
| Cassandra (QUORUM) | PA | EL (tunable) | PA/EL |
| DynamoDB (default) | PA | EL | PA/EL |
| Postgres (sync) | PC | EC | PC/EC |
| Spanner | PC | EC | PC/EC |
| CockroachDB | PC | EC | PC/EC |
| MongoDB (w:majority) | PC | EC | PC/EC |
| Redis (async) | PA | EL | PA/EL |

---

## 11. Choosing a Replication Strategy: Decision Guide

```
Is the data financial, inventory, or legally required to be consistent?
  └─ YES → Single-leader with sync replication or NewSQL (Raft-based)
           Never use multi-leader. Never use leaderless with ONE consistency.

Do you need writes from multiple regions simultaneously?
  └─ YES
      Can conflicts be resolved automatically (CRDT, LWW, merge)?
        └─ YES → Multi-leader with CRDT or LWW conflict resolution
        └─ NO  → Route all writes through one region (single-leader)
                 OR use NewSQL with geo-partitioning (data per region, strong consistency)

Is your workload read-heavy (>90% reads)?
  └─ YES → Single-leader + many async replicas
           Use replica routing for reads; primary for writes

Do you need sub-millisecond write latency?
  └─ YES → Async replication (acknowledge before replica confirms)
           Accept: potential loss of last few ms of writes on crash

Do you need zero data loss on any failure?
  └─ YES → Sync replication (at least one synchronous replica)
           Accept: writes block if synchronous replica is unavailable

Do you need to handle replicas going offline for hours/days?
  └─ YES → Leaderless (Cassandra/DynamoDB) with sloppy quorum + hinted handoff
           Accept: convergence time after recovery; run nodetool repair regularly
```

---

## 12. Reference Case Studies

1. **Amazon Dynamo — The Original Leaderless/Quorum Paper**
   [https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)

2. **Google Spanner — Externally Consistent Distributed Transactions**
   [https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf](https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf)

3. **Raft Consensus Algorithm (original paper — very readable)**
   [https://raft.github.io/raft.pdf](https://raft.github.io/raft.pdf)

4. **CockroachDB: Living Without Atomic Clocks**
   [https://www.cockroachlabs.com/blog/living-without-atomic-clocks/](https://www.cockroachlabs.com/blog/living-without-atomic-clocks/)

5. **Netflix: Active-Active for Multi-Regional Resiliency**
   [https://netflixtechblog.com/active-active-for-multi-regional-resiliency-c47719f6685b](https://netflixtechblog.com/active-active-for-multi-regional-resiliency-c47719f6685b)

6. **Designing Data-Intensive Applications — Chapters 5 & 9 (Kleppmann)**
   [https://dataintensive.net/](https://dataintensive.net/)  ← Chapters 5 (Replication) and 9 (Consistency) are the definitive reference

7. **Discord: How Discord Stores Trillions of Messages (Cassandra → ScyllaDB)**
   [https://discord.com/blog/how-discord-stores-trillions-of-messages](https://discord.com/blog/how-discord-stores-trillions-of-messages)

8. **MongoDB: Causal Consistency in Distributed Transactions**
   [https://www.mongodb.com/docs/manual/core/causal-consistency-read-write-concerns/](https://www.mongodb.com/docs/manual/core/causal-consistency-read-write-concerns/)

9. **Patroni: High Availability PostgreSQL**
   [https://patroni.readthedocs.io/en/latest/](https://patroni.readthedocs.io/en/latest/)

10. **PACELC: An Alternative to CAP (Daniel Abadi)**
    [https://cs-people.bu.edu/dna/talks/PACELC.pdf](https://cs-people.bu.edu/dna/talks/PACELC.pdf)

11. **etcd: Raft in Production (Lessons Learned)**
    [https://etcd.io/docs/v3.5/faq/](https://etcd.io/docs/v3.5/faq/)

12. **Cassandra: Replication Internals**
    [https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html](https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html)
