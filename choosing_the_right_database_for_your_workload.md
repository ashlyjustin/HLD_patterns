# Choosing the Right Database for Your Workload

> **What this doc covers:** A structured guide to selecting the right database — or combination of databases — for any system design problem. Goes beyond surface-level comparisons by explaining *why* each database excels at its workload through internal architecture: storage engines, data structures, replication models, and concurrency mechanisms. Includes a decision framework, real-world polyglot examples, and anti-patterns.

---

### 📋 Outline

| # | Section | What You'll Learn |
|---|---------|-------------------|
| 1 | Five Questions Before You Choose | A mental model that eliminates wrong categories before picking a DB name |
| 2 | Database Taxonomy | Overview of all 9 DB categories with best-fit use cases and when to avoid each |
| 3 | How Each Database Is Built for Its Use Case | Internal architecture deep-dive: B-trees, LSM trees, inverted indexes, MVCC, skip lists, erasure coding, Raft — and why these choices make each DB excel at what it does |
| 4 | Decision Framework | Flowchart logic to arrive at the right DB from first principles |
| 5 | Regular Scenarios | E-commerce, social media, ride-sharing, IoT — solved with rationale |
| 6 | Tricky Scenarios | "We need SQL and horizontal scale", polyglot trap, "just use DynamoDB", CAP during failover |
| 7 | Anti-Patterns to Avoid | Redis as primary store, Elasticsearch as primary store, premature sharding |
| 8 | Real-World Choices at Scale | Netflix, Uber, Airbnb, Discord, Stripe, GitHub — what they chose and why |
| 9 | Reference Case Studies | Links to original engineering blog posts and research papers |
| 10 | Quick Reference Cheat Sheet | One-line "need X → use Y" lookup table |

---

## 1. The Mental Model: Five Questions Before You Choose

Before touching any DB name, answer these five questions. The answers eliminate entire categories of databases.

| # | Question | Why it matters |
|---|----------|---------------|
| 1 | What is the **shape** of your data? | Relational, document, graph, time-series, blob? |
| 2 | What is the **access pattern**? | Point lookups, range scans, full-text, aggregations, graph traversals? |
| 3 | What are the **scale requirements**? | QPS, dataset size, write vs read ratio, hot keys? |
| 4 | What are the **consistency requirements**? | Strong, eventual, causal, linearizable? |
| 5 | What are the **operational constraints**? | Team expertise, cloud provider lock-in, budget, compliance (GDPR, HIPAA)? |

---

## 2. Database Taxonomy

### 2.1 Relational (RDBMS)
**Examples:** PostgreSQL, MySQL, CockroachDB, Aurora, Spanner  
**Best for:** Structured data, complex JOINs, ACID transactions, financial ledgers, user profiles with relationships  
**Avoid when:** Schema changes too frequently, data is unstructured, horizontal write scale is needed without careful sharding

### 2.2 Document Store
**Examples:** MongoDB, DynamoDB (document mode), Firestore, Couchbase  
**Best for:** Hierarchical / nested objects, flexible schema evolution, catalog data, user-generated content  
**Avoid when:** You need cross-document transactions frequently, data is highly relational, or you need complex aggregations

### 2.3 Key-Value Store
**Examples:** Redis, DynamoDB (KV mode), Memcached, etcd  
**Best for:** Caching, sessions, rate-limiting counters, leaderboards, feature flags  
**Avoid when:** You need range queries, complex filtering, or relational integrity

### 2.4 Wide-Column Store
**Examples:** Cassandra, HBase, ScyllaDB, Bigtable  
**Best for:** Time-series, write-heavy workloads, multi-datacenter replication, IoT sensor data, activity feeds  
**Avoid when:** You need ad-hoc queries, JOINs, or ACID transactions across rows

### 2.5 Graph Database
**Examples:** Neo4j, Amazon Neptune, JanusGraph, TigerGraph  
**Best for:** Fraud detection, social graphs, recommendation engines, knowledge graphs, access control  
**Avoid when:** Relationships are simple (1-2 hops), data volume is enormous (graph DBs struggle at petabyte scale)

### 2.6 Search Engine
**Examples:** Elasticsearch, OpenSearch, Typesense, Meilisearch  
**Best for:** Full-text search, faceted filtering, log analytics (ELK stack), e-commerce search  
**Avoid when:** You need primary-store guarantees (search engines are typically secondary indexes)

### 2.7 Time-Series Database
**Examples:** InfluxDB, TimescaleDB, Prometheus, QuestDB, TDengine  
**Best for:** Metrics, monitoring, financial ticks, IoT telemetry  
**Avoid when:** Data is not time-ordered or you need complex relational queries alongside metrics

### 2.8 Object / Blob Store
**Examples:** S3, GCS, Azure Blob, MinIO  
**Best for:** Images, videos, large files, backups, ML training datasets  
**Avoid when:** You need structured querying — blob stores have no query engine

### 2.9 NewSQL / Distributed SQL
**Examples:** CockroachDB, YugabyteDB, Google Spanner, TiDB  
**Best for:** Need for global ACID transactions + horizontal scalability; replaces sharded MySQL/Postgres  
**Avoid when:** Latency sensitivity is extreme (distributed commits add RTTs), or your scale doesn't justify complexity

---

## 3. How Each Database Is Built for Its Use Case

This is the section that separates engineers who *use* databases from those who truly *understand* them. The reason a database excels at its workload is almost always traceable to a specific internal design decision — a data structure, a concurrency model, or a network protocol. Understanding these internals lets you predict behavior under load, debug performance problems, and make principled trade-offs.

---

### 3.1 Relational Databases (PostgreSQL, MySQL/InnoDB)

#### The core problem they solve
Structured data where relationships matter, writes must be atomic across multiple tables, and query patterns aren't known in advance.

#### How they're built for it

**B-Tree as the universal index structure**

The B-tree (specifically B+ tree in most RDBMS implementations) is the foundation of every table and index. All rows are stored in a balanced tree where every leaf node contains actual row data (clustered index in InnoDB) or a pointer to it (heap file in Postgres).

```
                    [50 | 100]                ← internal node (keys only)
                   /     |     \
           [20|30]    [60|80]   [110|150]     ← internal nodes
           /  |  \    /  |  \   /   |   \
         [..][..][..][..][..][..][..] [..][..] ← leaf nodes (actual rows)
                          ↕    ↕    ↕
                    linked list → enables range scans
```

Why this matters:
- `WHERE id = 42` → O(log N) tree traversal, ~3-4 disk reads for a table with millions of rows
- `WHERE created_at BETWEEN X AND Y` → find start leaf, follow linked list → efficient range scan
- Every `INSERT`/`UPDATE`/`DELETE` must maintain the tree → write amplification, but predictable

**WAL (Write-Ahead Log) for Durability and Atomicity**

Before any data page is modified, the change is first written sequentially to the WAL (also called redo log in MySQL). On crash, the database replays the WAL to recover to a consistent state.

```
Client COMMIT
      │
      ▼
Write change to WAL (sequential, fast — just append)
      │
      ▼
fsync WAL to disk ← only this must hit disk before returning to client
      │
      ▼
Return success to client
      │
      ▼ (asynchronously, later)
Apply change to data pages (random I/O — can be batched)
```

Sequential WAL writes are why Postgres can sustain thousands of TPS on spinning disk — random I/O is minimized. This also enables **streaming replication**: replicas replay the primary's WAL to stay in sync.

**MVCC (Multi-Version Concurrency Control) for Isolation Without Reader/Writer Blocking**

Instead of locking rows for readers, Postgres keeps multiple versions of each row. Each row has `xmin` (the transaction that created it) and `xmax` (the transaction that deleted/updated it).

```
Row version history for account_id=42:
  xmin=100  balance=500  xmax=200   ← created by TX 100, overwritten by TX 200
  xmin=200  balance=600  xmax=NULL  ← current version, created by TX 200
```

A reader with snapshot at TX 150 sees `balance=500`. A reader at TX 250 sees `balance=600`. No locks. Readers and writers never block each other. This is how Postgres achieves high read throughput alongside concurrent writes.

The cost: old row versions accumulate as "dead tuples." `VACUUM` cleans them up. Poor VACUUM → table bloat → index bloat → slow queries. This is a uniquely Postgres operational concern.

**Query Planner and Cost-Based Optimizer**

SQL is a *declarative* language — you say what you want, not how to get it. The query planner tries every possible execution plan (join order, index choice, scan type) and picks the cheapest one based on statistics stored in `pg_statistic`.

```sql
EXPLAIN ANALYZE SELECT * FROM orders o JOIN users u ON o.user_id = u.id WHERE u.country = 'US';

-- Planner might choose:
-- Option A: Seq scan orders → Hash join with users filtered by country  (if few US users)
-- Option B: Index scan users WHERE country='US' → Nested loop join orders  (if many US users)
-- It runs cost formulas: seq_page_cost, random_page_cost, cpu_tuple_cost
-- and picks the cheaper one
```

When statistics are stale (after a large data load), the planner makes bad choices → `ANALYZE` table to refresh them.

---

### 3.2 Document Stores (MongoDB)

#### The core problem they solve
Application objects are hierarchical and nested. Forcing them into flat rows requires either many JOINs or awkward schema gymnastics. Document stores let you store and retrieve an entire object as one unit.

#### How they're built for it

**WiredTiger Storage Engine (MongoDB's default since v3.2)**

WiredTiger uses a **B-tree for its storage** — but with one critical difference from RDBMS: the document (BSON) is stored as a single opaque blob. The entire object is read or written atomically, no matter how many nested fields it has.

```
WiredTiger B-Tree:
  Key:   _id (ObjectId)
  Value: {"name":"Alice","address":{"city":"NY","zip":"10001"},"orders":[...]}
         ← entire document stored as one value in the leaf node
```

This means: `db.users.findOne({_id: "alice"})` → one B-tree lookup, return the entire object. No JOIN needed to get her address or orders — they're already embedded. This is the core performance win for document stores.

**BSON (Binary JSON)**

MongoDB stores documents as BSON, not text JSON. BSON is:
- **Length-prefixed:** Each field and document starts with its byte length → you can skip over fields you don't need without parsing them
- **Type-aware:** Native types for int32, int64, double, date, ObjectId, binary — no string-to-number conversion overhead
- **Traversable in O(fields)** rather than O(characters) like text JSON

When you project `db.users.find({}, {name: 1, city: 1})`, MongoDB uses BSON's length prefixes to skip over the `orders` array without deserializing it.

**Collection-Level Locking → Document-Level Locking**

Early MongoDB (pre-2.2) used a global write lock — one write at a time across the entire DB. A major bottleneck. WiredTiger moved to **document-level MVCC concurrency**: multiple documents can be read/written concurrently, even within the same collection. Each document modification is atomic (you cannot observe a partially-written document), but two concurrent writes to different documents don't block each other.

**Oplog (Operations Log) for Replication**

MongoDB's replication works via an **oplog** — a capped collection in every replica set member that records every write operation as an idempotent command. Secondaries tail the primary's oplog and replay operations to stay in sync.

```
Primary oplog:
  { op: "i", ns: "mydb.users", o: {_id: ..., name: "Bob"} }  ← insert
  { op: "u", ns: "mydb.users", o: {$set: {name: "Robert"}}, o2: {_id: ...} }  ← update
  
Secondary: continuously reads oplog, replays operations
```

Why capped? If a secondary falls too far behind, the oplog wraps around and the secondary must do a full resync. This is a critical ops concern for MongoDB clusters under heavy write load.

**Aggregation Pipeline**

MongoDB's aggregation pipeline is a declarative transformation DSL executed entirely inside the database engine. It uses a **pipeline of stages** processed sequentially, each transforming the document stream:

```
Collection scan/index → $match → $lookup (join) → $group → $sort → $project → result
```

The engine pushes down `$match` and `$sort` stages to use indexes before loading full documents into memory. `$lookup` is a left outer join — useful but expensive; MongoDB recommends embedding related data instead of relying on `$lookup` for frequently-accessed paths.

---

### 3.3 Key-Value Stores (Redis)

#### The core problem they solve
Microsecond-latency reads and writes for simple lookups, counters, caches, and small data structures. No disk I/O on the hot path.

#### How they're built for it

**Everything Lives in RAM (by design)**

Redis is architected as an **in-memory data structure server**. The primary copy of every key lives in a hash table in RAM. Persistence (RDB snapshots, AOF log) is secondary — it exists so you can survive a restart, not to be the primary I/O path.

```
Redis memory layout:
  Global hash table: {key → redisObject pointer}
  redisObject:       {type, encoding, ptr → actual data}

  e.g., "user:42:session" → {type: STRING, encoding: EMBSTR, ptr → "token_abc"}
       "leaderboard"      → {type: ZSET, encoding: SKIPLIST, ptr → skiplist}
```

This hash table lookup + pointer dereference is how Redis achieves `GET` in ~100 nanoseconds — it's essentially two pointer follows in RAM.

**Single-Threaded Command Execution (I/O Multiplexing, not blocking)**

Redis processes commands on a **single thread** using an event loop (similar to Node.js). This eliminates all lock contention between commands. No matter how many clients, only one command executes at a time.

```
Event loop:
  while True:
      events = epoll_wait(client_sockets, timeout=1ms)
      for event in events:
          read command from socket
          execute command (atomically, no preemption)
          write response to socket
```

Because RAM operations are nanoseconds, the single thread can process ~100k-500k simple commands/second. Network I/O is the actual bottleneck, handled by async I/O multiplexing.

This also means: **Redis commands are atomic by design.** `INCR`, `SETNX`, `LPUSH` — each is guaranteed to run without interruption. No locks needed in your application for simple counters.

**Specialized Data Structures Built for Their Access Patterns**

Redis doesn't store everything as strings. It has purpose-built data structures, each with specific internal implementations:

| Type | Internal Structure | Why |
|------|--------------------|-----|
| String (small, ≤44 bytes) | `embstr` — object + string in one allocation | No pointer dereference, cache-friendly |
| String (large) | `raw` — separate string allocation | Handles arbitrary size |
| Hash (small) | `listpack` (formerly ziplist) — flat array | Fits in one cache line; faster than hash table for small N |
| Hash (large) | Hash table | O(1) lookup at scale |
| List (small) | `listpack` | Contiguous memory |
| List (large) | Doubly linked list | O(1) head/tail push, O(N) middle |
| Sorted Set (small) | `listpack` | Linear scan still fast for small N |
| Sorted Set (large) | **Skip list + hash table** | O(log N) rank queries + O(1) score lookup |
| Set (small, integer) | `intset` — sorted integer array | Binary search, very compact |
| Set (large) | Hash table | O(1) member test |

The **skip list** for sorted sets deserves special mention. Redis chose skip list over a balanced BST because skip lists allow O(log N) range queries (`ZRANGEBYSCORE`) with simpler implementation and better cache performance during traversal. Each node in the skip list contains forward pointers at multiple heights — probabilistically balanced, no rotations needed.

```
Skip list for ZSET (leaderboard):
Level 3: [head] ─────────────────────────────────► [score:900]
Level 2: [head] ──────────► [score:500] ──────────► [score:900]
Level 1: [head] ──► [score:200] ──► [score:500] ──► [score:900]
          score:   100      200      500      700      900
```

`ZRANK leaderboard user` → O(log N) traversal from head, counting skipped nodes.

**Persistence: RDB vs AOF**

```
RDB (snapshot):
  - Fork the process → child writes point-in-time snapshot to disk
  - Parent continues serving requests (copy-on-write memory)
  - Risk: up to N minutes of data loss on crash (N = snapshot interval)
  - Fast restart (load one file)

AOF (Append-Only File):
  - Every write command is appended to a log file
  - On restart, replay the log to rebuild state
  - Risk: smaller data loss window (configurable: every write, every second, or OS-buffered)
  - Slower restart (must replay all commands), larger file size
```

Production recommendation: AOF with `appendfsync everysec` (at most 1 second of data loss) + periodic RDB for fast restarts.

---

### 3.4 Wide-Column Stores (Apache Cassandra, ScyllaDB, Google Bigtable)

#### The core problem they solve
Write-heavy, append-dominant workloads at massive scale — where you can't afford the write amplification of B-tree updates, where data is naturally partitioned (by user, by time, by region), and where multi-datacenter replication is a first-class requirement.

#### How they're built for it

**LSM Tree (Log-Structured Merge Tree) — The Heart of Cassandra**

The core insight: random writes to a B-tree are slow because they require reading a page from disk, modifying it, and writing it back — scattered random I/O. The LSM tree converts random writes into sequential writes.

```
Write path:
  Client write → CommitLog (sequential WAL on disk, for durability)
               → MemTable (in-memory sorted tree, for fast reads)
               
  When MemTable is full → flush to immutable SSTable file on disk
  
SSTable files accumulate → Compaction merges them periodically

Read path:
  Check MemTable (newest data)
  Check each SSTable (oldest to newest)
  Merge results (last-write-wins by timestamp)
```

```
Write amplification comparison:
  B-tree INSERT:   read page + modify + write page (random I/O, ~4ms on HDD)
  LSM tree INSERT: append to MemTable + append to CommitLog (sequential, ~0.1ms)
  
  LSM tree is 10-100x faster for write-heavy workloads.
```

**Bloom Filters — Avoiding Unnecessary SSTable Reads**

A read might need to check many SSTable files (compaction helps but doesn't eliminate this). To avoid reading SSTables that definitely don't contain a key, Cassandra maintains a **bloom filter** for each SSTable — a compact probabilistic data structure that answers "is this key in this SSTable?" with zero false negatives (it may have false positives, but never misses a key that exists).

```
Without bloom filter: read 10 SSTables to find one key → 10 disk reads
With bloom filter:    check 10 bloom filters in memory (nanoseconds)
                      → only 1 SSTable actually contains the key → 1 disk read
```

Bloom filter space is tunable. 10 bits per key → ~1% false positive rate. This is the difference between Cassandra reads being slow (without) and fast (with).

**Consistent Hashing Ring — Masterless, Linearly Scalable**

Unlike RDBMS where writes go to a single primary, Cassandra uses a **distributed hash ring** where every node is equal. Data is partitioned across nodes by hashing the partition key.

```
Ring (4 nodes, each responsible for a range of hash values):

               Node A (0-25%)
              /               \
    Node D (75-100%)        Node B (25-50%)
              \               /
               Node C (50-75%)

Write: partition key "user_42" → hash → 30% → goes to Node B
       With replication factor 3: also replicated to C and D
```

Adding a new node: it joins the ring at a random position and takes responsibility for a portion of the range from its neighbors. No central coordinator, no single point of failure. This is why Cassandra scales linearly — adding nodes adds proportional write capacity.

**Tunable Consistency via Quorums**

Cassandra lets you choose your consistency level **per query**:

```
Replication factor = 3 (data exists on 3 nodes)

CONSISTENCY ONE:    write to 1 node → return success (fastest, may lose data)
CONSISTENCY QUORUM: write to 2/3 nodes → return success (balanced)
CONSISTENCY ALL:    write to 3/3 nodes → return success (strongest, slowest)

Read CONSISTENCY QUORUM + Write CONSISTENCY QUORUM:
  Together they overlap: at least one node has the latest write → strong consistency
  Formula: R + W > N  →  2 + 2 > 3  ✓
```

This is what makes Cassandra flexible. Critical data can use QUORUM; high-throughput analytics can use ONE. This tunability is not available in traditional RDBMS.

**Gossip Protocol — Decentralized Cluster State**

Cassandra nodes don't rely on a central registry to know the cluster topology. Instead, each node periodically "gossips" with a few random neighbors, sharing its knowledge of which nodes are alive, their load, and their token positions. Within seconds, new information propagates to all nodes via **epidemic spreading**:

```
Node A talks to B → B now knows A's state
B talks to C → C now knows A's and B's state
C talks to D → D now knows A, B, C's state
...
```

This eliminates single points of failure and lets Cassandra handle network partitions gracefully — nodes continue operating independently, reconciling state when the partition heals.

---

### 3.5 Graph Databases (Neo4j)

#### The core problem they solve
Queries that traverse relationships — "find all people within 3 hops of Alice who share a mutual friend with Bob." In a relational DB, each hop requires a JOIN. Traversing 5 hops = 5 JOINs on potentially millions of rows. Graph databases make multi-hop traversal fast by design.

#### How they're built for it

**Index-Free Adjacency — The Key Architectural Difference**

In an RDBMS, finding a node's neighbors requires a JOIN against an edge table:
```sql
-- RDBMS: 3-hop traversal = 3 JOINs, full index scan on edges table each time
SELECT e3.to_node FROM edges e1
JOIN edges e2 ON e1.to_node = e2.from_node
JOIN edges e3 ON e2.to_node = e3.from_node
WHERE e1.from_node = 'alice';
-- Cost grows with total graph size, not with neighborhood size
```

Neo4j stores relationships differently: **each node directly contains physical pointers to its adjacent edges**. There is no global edges table to scan.

```
Node record (Alice):
  [node_id | label | properties_ptr | first_relationship_ptr]
                                            ↓
Relationship record:
  [rel_id | type | from_node_ptr | to_node_ptr | next_rel_ptr(from) | next_rel_ptr(to)]
                        ↓                ↓              ↓
                     Alice's         Bob's         Alice's next
                     record          record         relationship
```

Traversal is pointer chasing in a linked list — no index lookup, no table scan. The cost of a traversal depends only on the **size of the neighborhood visited**, not on the total number of nodes or edges in the graph. A 10-hop traversal touching 100 nodes costs the same whether the graph has 10k or 10B nodes.

This property — called **index-free adjacency** — is why graph databases are fundamentally faster for traversal than any relational database, regardless of indexing strategy.

**The Property Graph Model**

Neo4j uses the **labeled property graph model**:
- **Nodes** have labels (`:Person`, `:Product`) and properties (`{name: "Alice", age: 30}`)
- **Relationships** have types (`[:FOLLOWS]`, `[:PURCHASED]`) and properties (`{since: "2020-01-01"}`)
- Both nodes and relationships are first-class citizens with their own storage

This is why Cypher (Neo4j's query language) looks like ASCII art of a graph — it's pattern matching directly against this structure:

```cypher
// Find Alice's friends' purchases in the last 30 days
MATCH (alice:Person {name: "Alice"})-[:FOLLOWS]->(friend:Person)-[:PURCHASED]->(product:Product)
WHERE product.purchased_at > datetime() - duration("P30D")
RETURN friend.name, product.title, product.price
ORDER BY product.purchased_at DESC
```

Each `-[:FOLLOWS]->` in the query is a direct pointer follow in storage — no JOIN, no index lookup.

**Page Cache — Keeping Hot Graph Portions in Memory**

Neo4j's storage engine maps node and relationship records to fixed-size pages (8KB). Frequently traversed subgraphs (e.g., the social neighborhood of active users) tend to be accessed repeatedly and stay in the page cache. Cold parts of the graph (inactive users, old data) fall out of cache. This locality property means that for most social graph workloads, the working set fits in RAM and traversals become pure memory operations.

---

### 3.6 Search Engines (Elasticsearch / Apache Lucene)

#### The core problem they solve
Finding documents that contain specific words, phrases, or that match complex relevance criteria — across billions of documents in milliseconds. A B-tree index on a text column cannot answer "find all documents where *any* field contains 'distributed systems' near 'consensus'" efficiently.

#### How they're built for it

**Inverted Index — The Foundation**

A traditional index maps `row → values`. A search engine builds the inverse: `word → list of documents containing that word`.

```
Documents:
  doc1: "distributed systems use consensus algorithms"
  doc2: "consensus is hard in distributed environments"
  doc3: "algorithms power modern systems"

Inverted index:
  "distributed"  → [doc1, doc2]
  "systems"      → [doc1, doc3]
  "consensus"    → [doc1, doc2]
  "algorithms"   → [doc1, doc3]
  "environments" → [doc2]

Query: "distributed AND consensus"
  → [doc1, doc2] ∩ [doc1, doc2] → {doc1, doc2}
  → Intersection via merge of sorted posting lists → O(N) where N = result size
```

The **posting list** for each term is a sorted list of document IDs. Intersection is a fast merge of sorted arrays. For common words ("the", "a"), the posting list is millions of entries — Elasticsearch uses skip pointers within posting lists to jump past large ranges quickly.

**BM25 Relevance Scoring**

Search engines don't just return matching documents — they rank them by relevance. BM25 (Best Match 25) is the current state-of-the-art formula:

```
BM25(term, doc) = IDF(term) × TF_normalized(term, doc)

IDF (Inverse Document Frequency):
  Rare terms score higher. "consensus" in 100 of 10M docs → high IDF
  Common terms score lower. "the" in 9M of 10M docs → near-zero IDF

TF (Term Frequency) — normalized to avoid stuffing:
  A document with "consensus" 100 times doesn't score 100x a doc with it once.
  TF saturates logarithmically.
```

This is computed at query time using pre-computed per-term statistics stored in the index.

**Lucene Segments — Immutable, Append-Only Index Shards**

Lucene (the library underlying Elasticsearch) builds its index as a collection of immutable **segments**. When new documents are indexed, they go into a new in-memory segment. When the buffer is full, it's flushed to a new immutable segment file on disk. Periodically, smaller segments are merged into larger ones.

```
Indexing flow:
  Write → In-memory buffer → Flush → Segment 1 (immutable)
                                    Segment 2 (immutable)
                                    ...
  Background merge: Segment 1 + Segment 2 → Segment 3 (larger, immutable)
```

Why immutable? Because updating an inverted index in-place is extremely expensive (the word might appear in thousands of documents). Immutable segments avoid this — updates are implemented as deletes (tombstones in a `.del` file) + new inserts. Merges clean up tombstones.

**Near-Real-Time (NRT) Search**

The in-memory buffer is refreshed to a searchable in-memory segment every 1 second (configurable). This is why Elasticsearch advertises "near-real-time" search — there's up to 1 second of indexing lag. You can reduce this but at the cost of more frequent flushing and reduced indexing throughput.

**Distributed Sharding + Replicas**

An Elasticsearch index is split into **shards** (each shard is a full Lucene index). Shards are distributed across nodes:

```
Index "products" with 5 primary shards + 1 replica each = 10 shards total
  Node 1: [P0] [P3] [R1] [R4]
  Node 2: [P1] [P4] [R0] [R3]
  Node 3: [P2] [R2]

Query: fan out to all 5 primary shards → gather results → merge-sort by score → return top N
```

This is why Elasticsearch scales horizontally for both write throughput (more shards) and query throughput (more replicas).

---

### 3.7 Time-Series Databases (TimescaleDB, InfluxDB)

#### The core problem they solve
Data arrives as a continuous stream of `(timestamp, metric, value)` tuples. Queries are almost always range scans over time ("give me CPU usage for the last 6 hours") and aggregations ("average per 5-minute window"). Standard databases become slow as the table grows into billions of rows.

#### How they're built for it

**Automatic Time-Based Partitioning (Hypertables in TimescaleDB)**

TimescaleDB is a Postgres extension. It introduces the **hypertable** — a logical table that is transparently partitioned into **chunks**, each covering a fixed time interval (e.g., 1 week).

```
Hypertable: metrics (device_id, time, value)
  Chunk 1: time ∈ [2025-01-01, 2025-01-08)  → 200M rows
  Chunk 2: time ∈ [2025-01-08, 2025-01-15)  → 200M rows
  Chunk 3: time ∈ [2025-01-15, 2025-01-22)  → 200M rows  ← active chunk
  
Query: WHERE time > now() - interval '7 days'
  → Only touches Chunk 3 (constraint exclusion)
  → 200M rows scanned, not 600M
```

Without TimescaleDB: all data is in one table. Adding an index on `time` helps for point queries but range scans still grow linearly with total table size over time.

**Column-Oriented Compression (Gorilla + Delta Encoding)**

Time-series data has high regularity — sensor values change slowly, timestamps increment predictably. Time-series databases exploit this for extreme compression:

- **Delta encoding for timestamps:** Store differences, not absolute values. Timestamps [100, 101, 102, 103] → store [100, +1, +1, +1] → compresses to 2 bytes instead of 32
- **Gorilla compression for float values (Facebook's algorithm):** XOR consecutive floats — if the value changes little, the XOR has many leading zeros → compress via run-length encoding
- **Dictionary encoding for repeated strings:** device_id "sensor_42" repeated 1M times → store as integer code, dictionary maps code → string

Result: TimescaleDB achieves 90-95% compression on typical time-series. InfluxDB claims similar ratios. A 1TB dataset in Postgres becomes 50-100GB in TimescaleDB.

**Continuous Aggregates — Pre-Computed Rollups**

Dashboard queries like "show me hourly averages for the last 30 days" must aggregate billions of raw data points. Time-series databases solve this with **continuous aggregates** — materialized views that are automatically maintained as new data arrives:

```sql
-- Define once
CREATE MATERIALIZED VIEW hourly_avg
WITH (timescaledb.continuous) AS
SELECT device_id,
       time_bucket('1 hour', time) AS bucket,
       AVG(value) AS avg_value,
       MAX(value) AS max_value
FROM metrics
GROUP BY device_id, bucket;

-- TimescaleDB automatically updates this view as new rows arrive
-- (only recomputes the buckets affected by new data)

-- Dashboard query: instant, reads from materialized view
SELECT * FROM hourly_avg WHERE bucket > now() - interval '30 days';
```

**InfluxDB's TSM (Time-Structured Merge Tree)**

InfluxDB built a custom storage engine called TSM, inspired by LSM trees but optimized for time-series:
- Data is organized by `(measurement, tag set, field key)` — the **series key**
- Within a series, values are stored in time order, enabling efficient sequential reads
- Compression is applied per-series, per-block (1000 values per block by default)
- Compaction merges smaller TSM files into larger ones, maintaining time-sorted order within each series

This means reading `cpu_usage{host="web-01"}` for the last hour is a sequential read through one compressed block — extremely fast.

---

### 3.8 Object / Blob Stores (Amazon S3, Google Cloud Storage)

#### The core problem they solve
Store and retrieve arbitrarily large files (bytes to terabytes) reliably and cheaply, at any scale, without complex schema design. A RDBMS with `BYTEA` columns doesn't scale past gigabytes-per-row and is expensive per GB. You need a system designed around large, immutable objects.

#### How they're built for it

**Flat Namespace with Consistent Hashing**

S3 does not have a real directory hierarchy — it's a flat key-value store where the "key" is the full object path (`bucket/prefix/filename.jpg`). The perceived folder structure is just key prefix matching.

Objects are distributed across storage servers via **consistent hashing on the key**:

```
Object key: "photos/user_42/profile.jpg"
  → hash(key) → assigned to storage node ring position
  → replicated to 2 additional nodes for durability (11 nines durability)
```

This means writes to `photos/user_42/` and `photos/user_43/` go to different nodes — no hot spot. A common mistake is using sequential prefixes (`2025/01/01/00001.jpg`, `2025/01/01/00002.jpg`) — early S3 hashed on the prefix and these would all land on the same shard. The fix (still relevant for some systems) is to add a random hash prefix: `a7f2/2025/01/01/00001.jpg`.

**Erasure Coding for Durability Without Full Replication**

Storing 3 full copies of every object for durability costs 3x storage. S3 uses **erasure coding** (Reed-Solomon) instead for most storage tiers:

```
Erasure coding (example: 6 data shards + 3 parity shards):
  Original file → split into 6 chunks + compute 3 parity chunks → store 9 across 9 nodes
  
  Can lose any 3 nodes and still reconstruct the original file from the remaining 6.
  Storage overhead: 9/6 = 1.5x  (vs 3x for full replication)
```

For very frequently accessed objects (hot tier), full replication is preferred (lower reconstruction latency). For infrequently accessed data (S3 Glacier), erasure coding with slow retrieval (minutes-to-hours) saves cost.

**Multipart Upload for Large Objects**

Uploading a 10GB video as a single HTTP request is fragile — any network blip requires starting over. S3 supports **multipart upload**: split into parts (minimum 5MB each), upload parts in parallel, and complete by sending the list of part ETags.

```
1. InitiateMultipartUpload → get upload_id
2. UploadPart(upload_id, part_number=1, data=chunk1) → ETag1  ┐ parallel
   UploadPart(upload_id, part_number=2, data=chunk2) → ETag2  ┤
   UploadPart(upload_id, part_number=3, data=chunk3) → ETag3  ┘
3. CompleteMultipartUpload(upload_id, [ETag1, ETag2, ETag3]) → final object
```

S3 assembles the parts on their end. If a part upload fails, only that part is retried — not the entire file. This is also how S3 Transfer Acceleration achieves high upload speeds: parts are uploaded in parallel to nearby edge nodes, which then fan-in to S3 over AWS's internal network.

**Strong Read-After-Write Consistency (Since December 2020)**

Early S3 had eventual consistency: you could upload an object and immediately get a 404 on a GET — because the GET hit a replica that hadn't received the update yet. In 2020, AWS upgraded to **strong read-after-write consistency** for all S3 operations. The mechanism uses a distributed, strongly-consistent metadata service (similar to a Paxos/Raft cluster) to track object state, so a GET after a successful PUT is guaranteed to see the new object.

---

### 3.9 NewSQL / Distributed SQL (CockroachDB, Google Spanner)

#### The core problem they solve
You need the full SQL interface, ACID transactions, and relational integrity — but across multiple physical machines, potentially in multiple datacenters, with no single point of failure and no manual sharding.

#### How they're built for it

**Range-Based Sharding with Automatic Rebalancing**

Unlike hash-based sharding (which requires manual resharding when adding nodes), CockroachDB divides the keyspace into **ranges** — contiguous spans of the sorted key space, each typically 512MB.

```
Keyspace (sorted by primary key):
  Range 1: [/Table/users/001 → /Table/users/10000)   → stored on Node 1, 2, 3
  Range 2: [/Table/users/10000 → /Table/users/50000) → stored on Node 2, 3, 4
  Range 3: [/Table/users/50000 → ...)                → stored on Node 3, 4, 5

Range 1 grows beyond 512MB → splits into Range 1a and 1b automatically
Node 1 is overloaded → Range 1a migrated to Node 6 automatically (rebalancer)
```

This is a key operational advantage over manual sharding: CockroachDB handles shard splits, merges, and migrations automatically. The application is unaware.

**Raft Consensus Per Range**

Each range is replicated across N nodes (default 3) using the **Raft consensus algorithm**. Raft ensures that all replicas agree on every write before it's committed — providing strong consistency.

```
Range 1 Raft group (Node 1 = leader, Node 2 and 3 = followers):

Client write → Node 1 (Raft leader)
  → AppendEntries to Node 2 and Node 3 (in parallel)
  → Wait for quorum (2 of 3 acknowledge)
  → Commit log entry
  → Return success to client
  → Node 2 and 3 apply asynchronously
```

Writes require a quorum (2/3) to acknowledge before committing. This means any single node failure doesn't lose data or block writes (the other two nodes still form a quorum).

**Multi-Version Timestamps for Serializable Isolation**

CockroachDB implements **Serializable Snapshot Isolation** (the strongest isolation level) without global locks. Every transaction gets a timestamp from a **Hybrid Logical Clock (HLC)** — a clock that combines physical wall time with a logical counter to ensure causality.

```
HLC timestamp: (physical_time_ms, logical_counter)
  e.g., (1704067200000, 3)  ← ms since epoch, logical counter for same-ms events
  
Transaction T1 at time (100, 1): reads row versions ≤ (100, 1)
Transaction T2 at time (101, 0): reads row versions ≤ (101, 0)
  → T2 sees T1's committed writes (since 100 < 101)
  → T1 does NOT see T2's writes (since T2 commits at a later time)
```

If two transactions try to commit conflicting changes, CockroachDB compares their timestamps. The one with the earlier timestamp wins; the later one must restart with a new timestamp. No locks, no waiting — pure optimistic MVCC with timestamp ordering.

**Google Spanner: TrueTime for Global External Consistency**

Spanner takes this further. Instead of HLC (which has bounded uncertainty), Google uses **TrueTime** — GPS receivers and atomic clocks in every datacenter, providing a wall clock with a guaranteed uncertainty bound of ε (typically 1-7ms).

```
TrueTime API:
  TT.now() → [earliest, latest]  where latest - earliest = 2ε ≈ 14ms

Spanner commit wait:
  1. Choose commit timestamp T_commit = TT.now().latest
  2. Wait until TT.now().earliest > T_commit  (i.e., wait 2ε)
  3. Now EVERY server in the world knows T_commit is in the past
  4. Return to client
```

This wait (the "commit wait," ~7-14ms) guarantees **external consistency**: if transaction T1 commits before T2 starts in real time, T1's commit timestamp is strictly less than T2's. No server can see a causally later transaction with an earlier timestamp. This is stronger than serializable — it's **linearizable** across a globally distributed database.

The 7-14ms commit wait is why Spanner has higher write latency than a single-node database. You pay milliseconds for global, clock-synchronized strong consistency.

---

### 3.10 Summary: Internal Architecture at a Glance

| Database Type | Core Data Structure | Write Strategy | Read Strategy | The Design Choice That Defines It |
|---------------|--------------------|-----------------|--------------------|----------------------------------|
| RDBMS (Postgres) | B+ Tree | WAL → heap/index pages | B-tree traversal, MVCC snapshots | MVCC lets readers and writers not block each other |
| Document (MongoDB) | B-Tree (WiredTiger) | Oplog → WiredTiger pages | Single document fetch (no JOIN) | Document as atomic unit means no JOIN for object retrieval |
| Key-Value (Redis) | Hash Table + specialized per-type | In-memory only, async persist | Hash lookup, pointer dereference | Single-threaded event loop = zero lock contention |
| Wide-Column (Cassandra) | LSM Tree (MemTable → SSTable) | Append to MemTable + CommitLog | Bloom filter → SSTable scan | LSM converts random writes to sequential writes |
| Graph (Neo4j) | Linked node/relationship records | Append to transaction log | Pointer chasing (index-free adjacency) | Index-free adjacency makes traversal O(hops), not O(graph size) |
| Search (Elasticsearch) | Inverted Index (Lucene segments) | In-memory buffer → immutable segment | Posting list intersection + BM25 rank | Inverted index turns text search from O(N) to O(result size) |
| Time-Series (TimescaleDB) | B-Tree + time chunks | Partition routing + compression | Chunk pruning by time, continuous aggregates | Time partitioning limits scan to the relevant time window |
| Object Store (S3) | Flat key → consistent-hash ring | Erasure-coded multipart storage | Direct object fetch by key | Erasure coding gives 11-nines durability at 1.5x storage cost |
| NewSQL (CockroachDB) | B-Tree per range + Raft log | Raft consensus per range | Distributed SQL execution, MVCC | Raft per range gives per-shard strong consistency with no global lock |
| NewSQL (Spanner) | B-Tree + Raft + TrueTime | TrueTime commit wait | Globally consistent MVCC | TrueTime makes commit timestamps globally unambiguous |

---

## 4. Decision Framework (Flowchart Logic)

```
Is data primarily blobs/files?
  └─ YES → Object Store (S3/GCS)
  └─ NO
      Is it time-series / metrics?
        └─ YES → Time-Series DB (InfluxDB, TimescaleDB)
        └─ NO
            Is data graph-structured (many many-to-many hops)?
              └─ YES → Graph DB (Neo4j, Neptune)
              └─ NO
                  Do you need full-text search?
                    └─ YES (primary need) → Search Engine (Elasticsearch)
                    └─ NO
                        Is schema rigid and relational integrity required?
                          └─ YES
                              Does it need horizontal write scale?
                                └─ YES → NewSQL (Spanner, CockroachDB)
                                └─ NO  → RDBMS (Postgres, MySQL)
                          └─ NO (flexible / nested schema)
                              Is it write-heavy, time-ordered, or huge scale?
                                └─ YES → Wide-Column (Cassandra, Bigtable)
                                └─ NO  → Document Store (MongoDB, DynamoDB)
```

For **caching / sessions / real-time counters**, layer Redis on top of any of the above.

---

## 4. Regular Scenarios

### Scenario 1: E-commerce Platform
**Requirements:** Product catalog, user accounts, orders, inventory, payment records  
**Traffic:** 10k QPS reads, 500 QPS writes

**Solution:**
- **PostgreSQL** for users, orders, inventory (ACID, JOINs, foreign keys)
- **Redis** for session tokens, cart (ephemeral, millisecond latency)
- **Elasticsearch** for product search with facets (category, price range, rating)
- **S3** for product images

**Why not MongoDB for orders?** Orders involve multi-table consistency (decrement inventory + create order + charge payment). ACID transactions in PostgreSQL handle this atomically without sagas.

---

### Scenario 2: Social Media Feed (Twitter-like)
**Requirements:** Posts, followers, timelines, likes, DMs  
**Traffic:** 300k QPS reads, 6k QPS writes, celebrity users with 50M followers

**Solution:**
- **Cassandra** for posts/tweets (write-heavy, time-ordered, no JOINs needed)
- **Redis** for pre-computed timelines of active users (fan-out on write for regular users)
- **PostgreSQL** for user accounts and metadata (low write rate, needs ACID for auth)
- **Elasticsearch** for tweet search and hashtag trends
- **S3** for media

**Why Cassandra for posts?** Posts are append-only, time-ordered, partitioned by user_id. Cassandra's SSTable architecture and LSM tree excel here.

---

### Scenario 3: Ride-Sharing (Uber-like)
**Requirements:** Real-time driver location, trip matching, pricing, payment  
**Traffic:** 1M drivers updating location every 4 seconds, 500k active trips

**Solution:**
- **Redis** (Geo commands / Redis Cluster) for real-time driver locations (`GEOADD`, `GEORADIUS`)
- **Cassandra** for trip history (append-only, high write, partition by city+date)
- **PostgreSQL** for user profiles, payment methods
- **Kafka** as the event backbone between microservices

**Why Redis for location?** Redis `GEORADIUS` is O(N+log M) and operates in memory — essential for sub-100ms matching. Postgres PostGIS works but can't sustain 250k location updates/sec on a single node.

---

### Scenario 4: IoT Sensor Platform
**Requirements:** 10,000 sensors each writing 1 data point/second, dashboard showing trends  
**Traffic:** 10k writes/sec, 1k dashboard queries/sec over 90-day windows

**Solution:**
- **TimescaleDB** (PostgreSQL extension) for sensor data — automatic partitioning by time, compression (90-95% space savings), continuous aggregates for dashboards
- **Redis** for latest sensor value per device (real-time dashboard)
- **PostgreSQL** (shared with TimescaleDB cluster) for device metadata, alerts config

**Why not plain Postgres?** A single `metrics` table would grow to billions of rows in 90 days. TimescaleDB hypertables auto-partition by time and compress cold chunks, maintaining query performance.

---

## 5. Tricky Scenarios

### Tricky Scenario 1: "We need SQL *and* massive horizontal scale"
**Context:** FinTech startup, growing 10x per year, currently on MySQL, hitting write bottleneck  
**Trap:** Teams often reach for MongoDB because it "scales horizontally" — but then lose ACID and spend months building compensation logic.

**Right answer:** Migrate to **CockroachDB or PlanetScale** (MySQL-compatible distributed SQL).  
- CockroachDB: Strong consistency, multi-region, geo-partitioning for data residency  
- PlanetScale: MySQL-compatible, branch-based schema migrations, horizontal sharding without application changes

**When this still isn't enough:** At the scale of PayPal or Stripe, you'll have domain-specific sharding strategies on top of distributed SQL.

> 📖 **Case Study:** [Cockroach Labs — How Bose migrated from MySQL to CockroachDB](https://www.cockroachlabs.com/customers/bose/)

---

### Tricky Scenario 2: The Polyglot Trap
**Context:** Team adds MongoDB for flexibility, Redis for cache, Postgres for transactions, Elasticsearch for search. Now data is in 4 places and inconsistency bugs appear.

**Problem:** Data synchronization between stores is eventual. A product price updated in Postgres may show stale in Elasticsearch for seconds. A deleted user may still appear in MongoDB.

**Solutions:**
1. **CDC (Change Data Capture):** Use Debezium to stream Postgres changes to Kafka → sync to Elasticsearch and Redis automatically
2. **Single Source of Truth:** Postgres is primary. Elasticsearch and Redis are derived read models that get invalidated on write
3. **CQRS:** Separate write model (Postgres) from read model (Elasticsearch), with an event bus keeping them in sync

**Key insight:** Polyglot persistence is valid, but you must own the synchronization story from day one.

> 📖 **Case Study:** [Airbnb's Data Infrastructure with Kafka and Flink](https://medium.com/airbnb-engineering/avoiding-double-payments-in-a-distributed-payments-system-2981f6b070bb)

---

### Tricky Scenario 3: "Just use DynamoDB for everything"
**Context:** AWS shop, team decides DynamoDB for all data since it's "infinitely scalable"

**Traps:**
- DynamoDB requires knowing your access patterns *before* schema design. Adding a new query that doesn't match the partition key requires a full table scan ($$$) or a Global Secondary Index (extra cost, eventual consistency)
- Transactions across multiple items are expensive (2x write units)
- No native full-text search, no JOINs, limited aggregation support
- Hot partitions if partition key is poorly chosen

**Right approach:** DynamoDB is excellent for **known, stable access patterns** at massive scale. Use it alongside Postgres (for complex queries) and Elasticsearch (for search). Don't force everything into DynamoDB.

> 📖 **Reference:** [DynamoDB Best Practices — AWS Docs](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)

---

### Tricky Scenario 4: Consistency vs. Availability During a Database Failover
**Context:** Primary Postgres goes down. Read replica is 500ms behind. Do you serve stale reads?

**CAP Theorem implications:**
- If you pick CP (Consistency + Partition tolerance): Return errors until primary is back. Users see 503.
- If you pick AP (Availability + Partition tolerance): Serve stale reads from replica. Users may see slightly outdated data.

**Decision rule:** Depends on domain.
- **Banking/payments:** CP. Never show stale balance.
- **Social feed/recommendations:** AP. A 5-second-old feed is acceptable.
- **Inventory:** It depends. Shopping cart — AP acceptable. Final checkout deduct — CP required.

**Pattern:** Use **two-tier reads**: serve from cache/replica normally, but re-read from primary when `requires_strong_consistency=true` flag is set (e.g., post-payment page).

---

### Tricky Scenario 5: Choosing Between Cassandra and DynamoDB
Both are wide-column, both handle massive write throughput. How to choose?

| Factor | Cassandra | DynamoDB |
|--------|-----------|----------|
| Hosting | Self-managed or DataStax Astra | Fully managed (AWS) |
| Consistency model | Tunable (quorum, eventual, strong) | Eventually consistent (strong optional, costs more) |
| Query flexibility | CQL with secondary indexes, ALLOW FILTERING | GSIs needed for non-PK queries |
| Cost model | Compute-based (predictable) | Capacity unit-based (unpredictable at spiky load) |
| Multi-region | Built-in multi-DC replication | Global Tables (extra cost) |
| Best for | Already in hybrid/on-prem, need tunable consistency | Pure AWS, serverless/unpredictable load |

**Rule of thumb:** AWS-native shop with bursty traffic → DynamoDB. Multi-cloud or on-prem with predictable high throughput → Cassandra/ScyllaDB.

---

## 6. Anti-Patterns to Avoid

| Anti-Pattern | Why it's wrong | Better approach |
|---|---|---|
| Using Redis as primary DB | Data loss on crash without AOF; no complex queries | Use Redis as cache, Postgres as source of truth |
| Mongo for everything | No ACID across documents, poor for relational data | Use Postgres for relational, Mongo where schema is truly flexible |
| Elasticsearch as primary store | No ACID, designed for search not storage, can lose data | Elasticsearch as secondary read index, primary in Postgres/Cassandra |
| Premature sharding | Complexity before it's needed | Start with a single well-indexed Postgres instance (can handle 10k+ QPS) |
| Ignoring data locality | Global users hitting single-region DB | Use geo-partitioning or multi-region replication (Spanner, CockroachDB) |

---

## 7. Real-World Database Choices at Scale

| Company | Workload | Database Choice | Reason |
|---------|----------|----------------|--------|
| **Netflix** | Viewing history, metadata | Cassandra | High write throughput, multi-region, eventual consistency acceptable |
| **Uber** | Driver locations | Redis Geo | Sub-millisecond in-memory geospatial queries |
| **Airbnb** | Listings, bookings | MySQL (sharded) → migrating to Vitess | ACID for payments, horizontal scale via Vitess |
| **Twitter/X** | Tweets | Manhattan (custom) + MySQL | Custom KV built on RocksDB for tweet storage |
| **LinkedIn** | Social graph | Espresso (custom) + MySQL | Graph on top of RDBMS with custom replication |
| **Discord** | Messages | Cassandra → ScyllaDB | 10B+ messages/day; Cassandra GC pauses forced migration to ScyllaDB (C++) |
| **Stripe** | Payments | Postgres + Docstore | ACID critical; custom docstore for flexible objects |
| **Notion** | Pages/blocks | Postgres | Rich document model fits JSONB; team scale doesn't require NoSQL yet |
| **GitHub** | Repositories, issues | MySQL (Vitess) | Complex relational queries; Vitess for horizontal scale |

---

## 8. Reference Case Studies

1. **Discord's Migration from Cassandra to ScyllaDB**  
   [https://discord.com/blog/how-discord-stores-trillions-of-messages](https://discord.com/blog/how-discord-stores-trillions-of-messages)

2. **Instagram's PostgreSQL at Scale**  
   [https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c](https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c)

3. **Uber's Database Architecture**  
   [https://eng.uber.com/schemaless-part-one-mysql-datastore/](https://eng.uber.com/schemaless-part-one-mysql-datastore/)

4. **Netflix: Cassandra as a Service**  
   [https://netflixtechblog.com/revisiting-1-million-writes-per-second-c191a84864cc](https://netflixtechblog.com/revisiting-1-million-writes-per-second-c191a84864cc)

5. **Airbnb: Data Infrastructure Evolution**  
   [https://medium.com/airbnb-engineering/data-infrastructure-at-airbnb-8adfb34f169c](https://medium.com/airbnb-engineering/data-infrastructure-at-airbnb-8adfb34f169c)

6. **Google Spanner: Globally Distributed Database (Paper)**  
   [https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf](https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf)

7. **Facebook TAO: Social Graph**  
   [https://www.usenix.org/system/files/conference/atc13/atc13-bronson.pdf](https://www.usenix.org/system/files/conference/atc13/atc13-bronson.pdf)

8. **DynamoDB: Amazon's Highly Available Key-Value Store (Original Paper)**  
   [https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)

---

## 9. Quick Reference Cheat Sheet

```
Need ACID + JOINs?              → PostgreSQL / MySQL
Need ACID + global scale?       → CockroachDB / Spanner
Need flexible schema?           → MongoDB / DynamoDB
Need blazing fast KV/cache?     → Redis / Memcached
Need time-ordered data?         → Cassandra / TimescaleDB / InfluxDB
Need full-text search?          → Elasticsearch / Typesense
Need graph traversals?          → Neo4j / Neptune
Need files/blobs?               → S3 / GCS
Need metrics/monitoring?        → Prometheus / InfluxDB
```
