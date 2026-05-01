# High-Level Design Reference Library

A personal reference library for staff-level system design. Each document is a standalone deep dive — not a cheat sheet, but a guide detailed enough to make you expert in the pattern. Every doc includes internal architecture explanations, real-world scenarios (regular and tricky), production code examples, common failure modes, and references to case studies from companies like Google, Netflix, Stripe, GitHub, and Shopify.

---

## 📚 Documents

### 🗄️ Databases

| File | What It Covers |
|------|---------------|
| [`choosing_the_right_database_for_your_workload.md`](./choosing_the_right_database_for_your_workload.md) | How to pick the right DB for any workload. Covers RDBMS, document, key-value, wide-column, graph, search, time-series, blob, and NewSQL — including how each is internally built for its use case (B+ trees, LSM trees, inverted indexes, Raft, etc.). Decision framework + tricky scenarios. |
| [`database_contention_detection_and_resolution.md`](./database_contention_detection_and_resolution.md) | Detecting and resolving lock contention, deadlocks, hot rows, and connection pool exhaustion. Covers MVCC, SELECT FOR UPDATE SKIP LOCKED, optimistic locking, and queue-table patterns. |
| [`database_replication_strategies_and_consistency_models.md`](./database_replication_strategies_and_consistency_models.md) | All replication topologies: single-leader, multi-leader, leaderless (Dynamo-style). Quorum math, CRDTs, Raft/Paxos, per-DB deep dives (Postgres WAL, Cassandra, MongoDB, Redis). Eventual consistency spectrum + CAP/PACELC. |
| [`database_migration_in_prod.md`](./database_migration_in_prod.md) | Zero-downtime schema migrations on large databases. The Expand-Contract pattern, safe vs unsafe DDL per DB, gh-ost/pt-osc internals, column/constraint/index operations, large-scale backfills, multi-service coordination, major version upgrades. |
| [`scaling_high_throughput_write_paths.md`](./scaling_high_throughput_write_paths.md) | Handling millions of writes per second. Write batching, write-behind cache, event sourcing, sharding strategies, CQRS, LSM tree write path, connection pooling, and the write amplification problem. |
| [`scaling_read_heavy_systems_with_caching_and_replicas.md`](./scaling_read_heavy_systems_with_caching_and_replicas.md) | Scaling read-heavy systems 100x+. Cache-aside, read-through, write-through patterns; read replicas; CDN; materialized views; denormalization; the N+1 problem; keyset pagination; hot key in Redis. |

### ⚙️ Distributed Systems

| File | What It Covers |
|------|---------------|
| [`transaction_patterns_2pc_3pc_saga_and_distributed_systems.md`](./transaction_patterns_2pc_3pc_saga_and_distributed_systems.md) | Everything about distributed transactions. Local ACID, 2PC (deep: WAL interaction, 4 failure scenarios), 3PC, TCC, Saga (choreography + orchestration with Temporal code), outbox pattern, when to use what, and 9 common distributed transaction errors. |
| [`async_task_queues_and_workflow_engine_patterns.md`](./async_task_queues_and_workflow_engine_patterns.md) | Building reliable async systems. Task queues, fan-out, pipeline/chain, DAG workflows, priority queues, rate limiting, at-least-once vs exactly-once delivery, idempotency, and observability. |

### 🚀 Traffic and Scale

| File | What It Covers |
|------|---------------|
| [`hot_users_celebrity_traffic_and_fanout_design.md`](./hot_users_celebrity_traffic_and_fanout_design.md) | The celebrity/hot user problem (Twitter, Instagram model). Push vs pull fanout, hybrid fanout, pre-computed timelines, handling Beyoncé posting to 100M followers, follower graph sharding. |

### 🗂️ Storage

| File | What It Covers |
|------|---------------|
| [`large_file_and_blob_storage_patterns.md`](./large_file_and_blob_storage_patterns.md) | Storing and serving large files. Object store internals (erasure coding, consistent hashing, strong consistency), presigned URLs, multipart upload, TUS resumable uploads, CDN integration, access control, media processing pipelines, deduplication, storage lifecycle policies. |

### 🛳️ Deployment

| File | What It Covers |
|------|---------------|
| [`large_scale_deployment_with_no_downtime.md`](./large_scale_deployment_with_no_downtime.md) | Deploying to production without downtime. Rolling, blue-green, canary strategies; graceful shutdown; health checks (liveness vs readiness vs startup); feature flags and dark launches; stateful service handling; multi-region ring-based rollouts; SLO-gated deploys and automated rollback. |

---

## 🧭 How to Use This Library

**Preparing for a system design interview:**
Start with `choosing_the_right_database_for_your_workload.md` — it gives you the vocabulary and decision framework used across all other docs. Then read the doc most relevant to the problem you're designing for.

**Debugging a production issue:**
Go directly to the relevant doc's *Common Errors* or *Failure Modes* section. Each doc has a dedicated section cataloguing real failure patterns with diagnosis queries and fixes.

**Designing a new system:**
Read the relevant doc's *Tricky Scenarios* section. These are the edge cases that only become visible at scale, and knowing them upfront prevents future pain.

**Learning a new concept:**
Each doc starts with *Why X Is Hard* — explaining the problem before diving into solutions. Read sequentially from the top for the full mental model.

---

## 🗺️ Concept Map

```
                         ┌─────────────────────┐
                         │  Choose the right   │
                         │      database       │
                         └──────────┬──────────┘
                                    │
              ┌─────────────────────┼──────────────────────┐
              ▼                     ▼                      ▼
    ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
    │  Scale reads     │  │  Scale writes    │  │  Replication &   │
    │  (caching,       │  │  (batching,      │  │  consistency     │
    │   replicas)      │  │   sharding)      │  │  (quorum, Raft)  │
    └──────────────────┘  └──────────────────┘  └──────────────────┘
              │                     │                      │
              └─────────────────────┼──────────────────────┘
                                    ▼
                         ┌─────────────────────┐
                         │  Transactions &     │
                         │  consistency        │
                         │  (2PC, Saga, TCC)   │
                         └──────────┬──────────┘
                                    │
              ┌─────────────────────┼──────────────────────┐
              ▼                     ▼                      ▼
    ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
    │  DB contention   │  │  DB migration    │  │  Async task      │
    │  & hot rows      │  │  (zero downtime) │  │  queues & DAGs   │
    └──────────────────┘  └──────────────────┘  └──────────────────┘
              
    ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
    │  Hot users &     │  │  Blob & file     │  │  Zero-downtime   │
    │  celebrity       │  │  storage         │  │  deployments     │
    │  fanout          │  │  (S3, CDN, TUS)  │  │  (canary, flags) │
    └──────────────────┘  └──────────────────┘  └──────────────────┘
```

---

## 📖 Key Concepts at a Glance

| Concept | Where to Learn It |
|---------|------------------|
| B+ tree vs LSM tree internals | `choosing_the_right_database` §3 |
| CAP theorem and PACELC | `database_replication_strategies` §11 |
| Expand-Contract migration pattern | `database_migration_in_prod` §2 |
| Quorum reads/writes (W+R>N) | `database_replication_strategies` §4 |
| Saga choreography vs orchestration | `transaction_patterns` §5 |
| Two-phase commit failure modes | `transaction_patterns` §4 |
| Presigned URL security model | `large_file_and_blob_storage` §6 |
| Canary deploy with Argo Rollouts | `large_scale_deployment` §2.3, §7.5 |
| Feature flags and dark launches | `large_scale_deployment` §4 |
| Cache stampede prevention | `scaling_read_heavy_systems` + `large_scale_deployment` §10 |
| gh-ost online schema change | `database_migration_in_prod` §4.1 |
| INT → BIGINT overflow emergency | `database_migration_in_prod` §12.2 |
| Celebrity fanout on social graph | `hot_users_celebrity_traffic` |
| Replication lag and WAL slots | `database_replication_strategies` §6.1 |
| Idempotency in async consumers | `async_task_queues` + `large_scale_deployment` §5.3 |

---

## 🔗 Real-World Case Studies Referenced

| Company | Topic | Document |
|---------|-------|----------|
| GitHub | Online schema changes, Scientist library | `database_migration_in_prod`, `large_scale_deployment` |
| Stripe | Online migrations at scale, idempotency | `database_migration_in_prod`, `transaction_patterns` |
| Netflix | Canary analysis with Kayenta, Chaos Engineering | `large_scale_deployment`, `database_replication_strategies` |
| Shopify | Zero-downtime Rails migrations, strong_migrations | `database_migration_in_prod` |
| GitLab | Batched background migrations | `database_migration_in_prod` |
| Notion | Sharding Postgres (herding elephants) | `database_migration_in_prod` |
| Dropbox | Chunk-level deduplication (Magic Pocket) | `large_file_and_blob_storage` |
| Twitter/X | Fanout on write vs read, timeline design | `hot_users_celebrity_traffic` |
| Amazon | Deployment rings, avoiding fallback | `large_scale_deployment` |
| Cloudflare | Global PoP deploys, R2 object storage | `large_scale_deployment`, `large_file_and_blob_storage` |
| Facebook | Haystack paper (blob storage), TAO (graph) | `large_file_and_blob_storage`, `choosing_the_right_database` |
| Google | Spanner (TrueTime), SRE release engineering | `choosing_the_right_database`, `large_scale_deployment` |
