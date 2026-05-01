# Scaling Reads

> **Goal:** Design read paths that serve millions of queries per second with low latency — without the primary database becoming a bottleneck.

---

## 1. Why Reads Need Special Treatment

In most production systems, reads outnumber writes 10:1 to 1000:1.

- Twitter: 300k reads/sec vs 6k writes/sec (50:1)
- Facebook News Feed: 100M+ reads/min
- Netflix: 99% of traffic is reads (video streaming + metadata)

A single Postgres primary can handle ~10k–50k simple reads/sec. To serve millions, you need a read scaling strategy.

---

## 2. The Read Scaling Toolkit

```
Speed        Technique
(fastest)    ──────────────────────────────────────
  ↑          Local in-process cache (L1, μs latency)
  │          Distributed cache / Redis (ms latency)
  │          CDN edge cache (global, ms latency)
  │          Read replicas (same DC, 5-10ms latency)
  │          Materialized views (pre-computed queries)
  │          Database query optimization
  ↓          Vertical scaling
(slowest)    Primary DB (tens of ms latency)
```

You typically need multiple layers simultaneously.

---

## 3. Core Techniques

### Technique 1: Read Replicas

The simplest form of read scaling. Replicate data from the primary to N replica databases. Route read queries to replicas.

```
                   Writes
Client ──► API ──► Primary DB
                       │
              Streaming replication
                 ┌─────┴──────┐
                 ▼            ▼
            Replica 1    Replica 2   ← Route reads here
```

**Implementation:**
```python
# Django example
DATABASES = {
    "default": {"HOST": "primary.db"},    # Writes go here
    "replica": {"HOST": "replica.db"},   # Reads go here
}

# Using Django's database router
class PrimaryReplicaRouter:
    def db_for_read(self, model, **hints):
        return "replica"
    
    def db_for_write(self, model, **hints):
        return "default"
```

**Key consideration: Replication Lag**
Replicas are asynchronously replicated (usually 10ms–500ms behind). Problems arise when:
- User writes a post → reads it back from replica → post not there yet (read-your-writes violation)
- Inventory decremented on primary → replica shows old value → oversell

**Solutions to replication lag:**
```python
# Pattern 1: Read-your-writes — route the read after a write to primary
def create_post(user_id, content):
    post = primary_db.insert("INSERT INTO posts ...")
    # For the next 5 seconds, this user reads from primary
    session["use_primary_until"] = time.now() + 5
    return post

def get_user_posts(user_id):
    db = primary_db if session.get("use_primary_until", 0) > time.now() else replica_db
    return db.query("SELECT * FROM posts WHERE user_id = %s", [user_id])

# Pattern 2: Sync replication for critical reads (Postgres synchronous_standby_names)
# Pattern 3: Wait for replica to catch up (not recommended — adds latency)
```

**Scaling ceiling:** With 10 replicas, you can handle ~500k reads/sec. Beyond that, add a caching layer.

---

### Technique 2: Caching

The highest-leverage read scaling technique. A cache hit avoids the DB entirely.

#### 2a. Cache-Aside (Lazy Loading)

The most common pattern. Application checks cache, misses → loads from DB → populates cache.

```python
def get_user_profile(user_id):
    # 1. Check cache
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)
    
    # 2. Cache miss — load from DB
    user = db.query("SELECT * FROM users WHERE id = %s", [user_id])
    
    # 3. Populate cache
    redis.setex(f"user:{user_id}", 300, json.dumps(user))  # 5 min TTL
    
    return user
```

**Pros:** Simple, cache only stores what's requested  
**Cons:** First request always misses (cold start latency spike)

#### 2b. Read-Through Cache

Cache is in the read path. On miss, cache fetches from DB automatically. Application only talks to cache.

```python
# Conceptually: cache.get("user:42") fetches from DB if missing
# Implemented by: AWS ElastiCache DAX (for DynamoDB), or cache libraries
```

#### 2c. Cache Invalidation Strategies

| Strategy | When to invalidate | Risk |
|----------|-------------------|------|
| TTL-based expiry | After N seconds | Stale reads during TTL window |
| Write-through (invalidate on write) | When data changes in DB | Cache stampede on popular keys |
| Event-driven (CDC) | On DB change event via Kafka/CDC | Complexity but most accurate |

```python
# Write-through: invalidate cache when data changes
def update_user_bio(user_id, bio):
    db.execute("UPDATE users SET bio = %s WHERE id = %s", [bio, user_id])
    redis.delete(f"user:{user_id}")  # Invalidate immediately

# Event-driven: Debezium sends DB changes to Kafka → consumer invalidates cache
def handle_user_update_event(event):
    user_id = event["payload"]["after"]["id"]
    redis.delete(f"user:{user_id}")
```

---

### Technique 3: CDN (Content Delivery Network)

For static and semi-static content, serve from CDN edge nodes geographically close to users.

```
User (Tokyo) ──► CDN Edge (Tokyo) ──hit──► Return (5ms)
                        │ miss
                        ▼
               CDN Shield (Singapore)
                        │ miss
                        ▼
               Origin Server (US-East, 150ms away)
```

**What to cache in CDN:**
- Static assets (JS, CSS, images, videos) — long TTL (1 year, with cache-busting on deploy)
- API responses for public, non-personalized data (product pages, article content) — short TTL (60s)
- Profile pages of public accounts

**Cache-Control headers:**
```
Cache-Control: public, max-age=31536000, immutable   # Static assets (1 year)
Cache-Control: public, max-age=60, stale-while-revalidate=300  # API responses
Cache-Control: private, no-store  # User-specific data (never cache at CDN)
```

**Real-world:** Netflix serves video through CDN (Open Connect). For a popular movie, a single CDN node serves millions of users — origin never sees most requests.

---

### Technique 4: Materialized Views

Pre-compute expensive queries and store results. Refresh periodically or on-demand.

```sql
-- Expensive query: runs on 10M rows, takes 5 seconds
SELECT 
    u.country,
    COUNT(DISTINCT o.user_id) AS buyers,
    SUM(o.total) AS revenue,
    AVG(o.total) AS avg_order
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.created_at > now() - interval '30 days'
GROUP BY u.country;

-- Pre-compute as materialized view (refreshed every 5 minutes)
CREATE MATERIALIZED VIEW mv_country_revenue AS
SELECT ... (above query);

CREATE INDEX ON mv_country_revenue (country);

-- Fast read: microseconds
SELECT * FROM mv_country_revenue WHERE country = 'US';

-- Refresh (can be done concurrently, no table lock)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_country_revenue;
```

**When not to use:** If data must be real-time (not acceptable to have 5-minute stale data for this use case).

---

### Technique 5: Denormalization

Normalization reduces redundancy but requires expensive JOINs at read time. For read-heavy paths, denormalize: store redundant data to avoid JOINs.

```sql
-- Normalized: requires JOIN on every read
SELECT p.content, u.username, u.avatar_url
FROM posts p
JOIN users u ON p.author_id = u.id
WHERE p.id = 42;

-- Denormalized: all data in one row, no JOIN needed
CREATE TABLE posts_denormalized (
    post_id      BIGINT PRIMARY KEY,
    content      TEXT,
    author_id    BIGINT,
    author_name  TEXT,   -- denormalized from users
    author_avatar TEXT,  -- denormalized from users
    created_at   TIMESTAMPTZ
);
```

**Risk:** If the author changes their username, you must update all their posts. Acceptable if username changes are rare; not acceptable if email is the denormalized field and users change it frequently.

**Where this is used at scale:** Cassandra data model design is fundamentally denormalized (no JOINs). DynamoDB single-table design embeds related data together.

---

### Technique 6: Database Index Optimization

Indexes are the first line of defense for read scaling. A missing index turns an O(log N) query into O(N).

```sql
-- Slow: full table scan on 100M rows
SELECT * FROM orders WHERE user_id = 42 AND status = 'pending';
-- EXPLAIN shows: Seq Scan

-- Add composite index (column order matters!)
CREATE INDEX CONCURRENTLY idx_orders_user_status 
ON orders (user_id, status);
-- Now: Index Scan, microseconds

-- Partial index: index only the rows you actually query
CREATE INDEX CONCURRENTLY idx_orders_pending
ON orders (user_id, created_at)
WHERE status = 'pending';  -- Smaller index, faster queries for this case

-- Covering index: includes all columns needed by query (avoids table lookup)
CREATE INDEX CONCURRENTLY idx_orders_covering
ON orders (user_id, status) INCLUDE (order_total, created_at);
```

**Index design rules:**
1. Put equality conditions first in composite index: `(user_id, status)` for `WHERE user_id = X AND status = Y`
2. Put range/sort columns last: `(user_id, created_at)` for `WHERE user_id = X ORDER BY created_at DESC`
3. Don't over-index: each index slows down writes. Remove unused indexes.

---

### Technique 7: Query Result Caching (Application-Level)

For read-heavy endpoints with many users requesting the same data:

```python
# Scenario: 1M users all request the top 10 trending topics
# Without caching: 1M DB queries/minute → 17k QPS on DB

def get_trending_topics():
    cache_key = "trending:topics:v2"
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)
    
    topics = db.query("""
        SELECT hashtag, COUNT(*) as post_count
        FROM posts_hashtags
        WHERE created_at > now() - interval '1 hour'
        GROUP BY hashtag
        ORDER BY post_count DESC
        LIMIT 10
    """)
    
    # Cache for 60 seconds — all 1M users share this single cache entry
    redis.setex(cache_key, 60, json.dumps(topics))
    return topics

# With caching: 1M users hit Redis, DB sees only 1 query/minute
```

---

## 4. Regular Scenarios

### Scenario 1: E-commerce Product Page
**Requirements:** Product detail page. 99% of traffic is read. Product data changes infrequently (price, stock). 500k page views/day.

**Architecture:**
```
Browser ──► CDN Edge (HTML/CSS/JS) ──hit──► Serve in <10ms
                    │ miss
                    ▼
           API Server
                    │
         ┌──────────┴───────────┐
         ▼                      ▼
  Redis Cache             Redis Cache
  (product data, 60s TTL)  (inventory count, 5s TTL)
         │ miss                 │ miss
         ▼                      ▼
    Read Replica DB         Read Replica DB
```

**On inventory update:** Invalidate Redis inventory key immediately (write-through cache invalidation). Product data TTL of 60s is acceptable.

---

### Scenario 2: Social Feed
**Requirements:** User's feed of posts from followed accounts. 1B impressions/day.

**Architecture:**
- Pre-computed timeline stored in Redis (fan-out on write for non-celebrities)
- Feed read = Redis `LRANGE timeline:{user_id} 0 99` (~1ms)
- For celebrities: merge from Cassandra post store on read
- Redis TTL = 48 hours. Inactive users (>48h offline): re-compute feed from Cassandra on first read

```python
def get_feed(user_id, page=0, page_size=20):
    start = page * page_size
    end = start + page_size - 1
    
    post_ids = redis.lrange(f"timeline:{user_id}", start, end)
    
    if not post_ids and page == 0:
        # Cold start: user has no timeline — compute from scratch
        post_ids = compute_feed_cold_start(user_id)
    
    # Batch fetch post details (multi-get from Redis or Cassandra)
    posts = redis.mget([f"post:{pid}" for pid in post_ids])
    return [json.loads(p) for p in posts if p]
```

---

### Scenario 3: Global API (Multi-Region Reads)
**Requirements:** API serving users in US, EU, APAC. Current single-region setup has 200ms latency for APAC users.

**Architecture:**
- **Active-passive per region:** Primary in US-East. Read replicas in EU-West and APAC-Singapore.
- Reads routed to nearest region's replica
- Writes still go to US-East primary (acceptable for most use cases)
- For data residency (GDPR): EU users' data lives in EU-West primary, US users' in US-East primary (geo-partitioning)

```
APAC user reads → APAC replica → 10ms latency ✓
APAC user writes → US-East primary → 150ms latency (acceptable; writes are rare)
```

---

## 5. Tricky Scenarios

### Tricky Scenario 1: Cache Stampede (Thundering Herd)

**Problem:** A popular product's cache entry expires. 10,000 concurrent requests all miss and query the DB simultaneously.

**Solution 1: Mutex / Single-Writer**
```python
def get_product(product_id):
    cached = redis.get(f"product:{product_id}")
    if cached:
        return json.loads(cached)
    
    # Acquire lock — only 1 request regenerates cache
    lock = redis.lock(f"lock:product:{product_id}", timeout=5)
    if lock.acquire(blocking=True, timeout=2):
        try:
            product = db.get_product(product_id)
            redis.setex(f"product:{product_id}", 300, json.dumps(product))
            return product
        finally:
            lock.release()
    else:
        # Couldn't acquire lock — wait briefly and retry from cache
        time.sleep(0.1)
        return json.loads(redis.get(f"product:{product_id}"))
```

**Solution 2: Stagger TTLs** — Add random jitter to prevent many keys expiring simultaneously.
```python
ttl = 300 + random.randint(-30, 30)  # 270-330 seconds
redis.setex(f"product:{product_id}", ttl, json.dumps(product))
```

**Solution 3: XFetch (Probabilistic Early Expiry)** — Proactively refresh slightly before expiry:
```python
def should_refresh(ttl_remaining, delta, beta=1.0):
    """Returns True if we should proactively refresh this cache entry."""
    import math, random
    return (- beta * math.log(random.random()) * delta) > ttl_remaining
```

---

### Tricky Scenario 2: Read-After-Write Consistency

**Problem:** User changes profile picture. API writes to primary. User's next request hits replica → sees old picture. Frustrating UX.

**Solutions:**

Option A: **Session affinity** — For 10 seconds after a write, route reads to primary
```python
def update_avatar(user_id, image_url):
    primary_db.execute("UPDATE users SET avatar = %s WHERE id = %s", [image_url, user_id])
    # Set flag: route reads to primary for 30s
    redis.setex(f"read_primary:{user_id}", 30, "1")

def get_user(user_id):
    if redis.get(f"read_primary:{user_id}"):
        return primary_db.get_user(user_id)
    return replica_db.get_user(user_id)
```

Option B: **Sync replication for important data** — Postgres `synchronous_standby_names`
```sql
-- Postgres: promote one replica to synchronous standby
-- Writes only commit when the synchronous replica acknowledges
-- Tradeoff: write latency increases by 1 RTT (~1-10ms intra-DC)
SET synchronous_standby_names = 'FIRST 1 (replica1)';
```

Option C: **Write-through cache** — Primary writes also update Redis instantly. Reads hit Redis, not the replica.

---

### Tricky Scenario 3: Cache Coherence in Distributed Cache Cluster

**Problem:** You have 5 Redis nodes (not cluster mode). User 1 connected to node 1 sets `user:42` = Profile A. User 2 connected to node 3 reads `user:42` — gets nothing (miss) and reads stale data from replica.

**Solution:** Use **Redis Cluster** (consistent hashing) or **Redis Sentinel** for HA. With Redis Cluster:
```
Key "user:42" → hash slot 1234 → always routed to node 3, regardless of which client
```
All clients use the same node for the same key → no coherence issue.

**Alternative for very hot keys:** Use **local in-process LRU cache** as L1 (millisecond), back-filled from Redis. Accept slight staleness.
```python
from cachetools import TTLCache

local_cache = TTLCache(maxsize=10000, ttl=10)  # 10s in-process cache

def get_user(user_id):
    if user_id in local_cache:
        return local_cache[user_id]
    user = redis.get(f"user:{user_id}") or db.get_user(user_id)
    local_cache[user_id] = user
    return user
```

---

### Tricky Scenario 4: The N+1 Query Problem at Scale

**Problem:** Rendering a list of 100 posts. For each post, fetch the author's name → 100 separate DB queries.

```python
# Bad: N+1 queries
posts = db.query("SELECT * FROM posts LIMIT 100")
for post in posts:
    post.author = db.query("SELECT name FROM users WHERE id = %s", [post.author_id])
    # 100 queries!
```

**Fix: Batch loading (DataLoader pattern)**
```python
# Collect all author IDs, fetch in one query
posts = db.query("SELECT * FROM posts LIMIT 100")
author_ids = list(set(p.author_id for p in posts))
authors = {u.id: u for u in db.query("SELECT * FROM users WHERE id = ANY(%s)", [author_ids])}

for post in posts:
    post.author = authors[post.author_id]
# 2 queries total
```

**In GraphQL:** Use **DataLoader** (batches N resolver calls into 1 DB call automatically).

---

### Tricky Scenario 5: Paginating Large Result Sets

**Problem:** `OFFSET 100000 LIMIT 20` requires scanning 100,020 rows. At page 5000, it's extremely slow.

```sql
-- Bad: OFFSET-based pagination
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 100000;
-- EXPLAIN: Seq Scan + Sort, 100,020 rows examined
```

**Fix: Cursor-based pagination (keyset pagination)**
```sql
-- Good: Keyset pagination (uses index efficiently)
-- First page:
SELECT * FROM posts ORDER BY created_at DESC, id DESC LIMIT 20;

-- Next page: pass last row's (created_at, id) as cursor
SELECT * FROM posts
WHERE (created_at, id) < ('2025-03-01 12:00:00', 99500)
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- EXPLAIN: Index Scan on (created_at, id), 20 rows examined
```

**API design:**
```json
{
  "data": [...],
  "cursor": "eyJjcmVhdGVkX2F0IjogIjIwMjUtMDMtMDEiLCAiaWQiOiA5OTUwMH0="
  // Base64 encoded: {"created_at": "2025-03-01", "id": 99500}
}
```

---

### Tricky Scenario 6: Hot Key in Redis (Key Overload)

**Problem:** `trending:topics` key is accessed 1M times/second. All requests hit the same Redis slot → CPU bottleneck on one node.

**Solution: Key replication with local jitter**
```python
NUM_COPIES = 10

def get_trending_topics():
    # Spread reads across N copies of the same key
    shard = random.randint(0, NUM_COPIES - 1)
    key = f"trending:topics:shard:{shard}"
    
    cached = redis.get(key)
    if cached:
        return json.loads(cached)
    
    # Miss: one of the N shards needs to refetch
    topics = compute_trending_topics()
    
    # Populate all shards
    for i in range(NUM_COPIES):
        redis.setex(f"trending:topics:shard:{i}", 60 + i, json.dumps(topics))
    
    return topics
```

**Alternative: Local cache L1** (10s TTL in process) eliminates Redis for this access pattern entirely.

---

## 6. Read Scaling Patterns Summary

| Pattern | Latency | Scale | Complexity | Consistency |
|---------|---------|-------|------------|-------------|
| Read replicas | 5-10ms | 10-50x | Low | Eventual (ms lag) |
| Redis cache | 1-2ms | 100-1000x | Medium | Configurable TTL |
| CDN edge cache | 1-50ms | Unlimited | Low | Eventual (TTL) |
| Local in-process cache | <0.1ms | Unlimited | Medium | Stale (seconds) |
| Materialized views | 0.1-1ms | 100x | Medium | Periodic refresh |
| Denormalization | 1-5ms | 5-10x | Medium | Sync on write |
| Keyset pagination | N/A | N/A | Low | N/A |
| Read-through cache | 1-2ms | 100x | Low | TTL |

---

## 7. Reference Case Studies

1. **Facebook: TAO — Scaling the Social Graph**  
   [https://www.usenix.org/system/files/conference/atc13/atc13-bronson.pdf](https://www.usenix.org/system/files/conference/atc13/atc13-bronson.pdf)

2. **Netflix: CDN Architecture (Open Connect)**  
   [https://netflixtechblog.com/how-netflix-works-with-isps-around-the-globe-to-deliver-a-great-viewing-experience-long-term-9ddfnf91e648](https://netflixtechblog.com/open-connect-overview/)

3. **Instagram: Scaling Read Traffic**  
   [https://instagram-engineering.com/handling-growth-with-postgres-5-tips-from-instagram-84f86b0bb674](https://instagram-engineering.com/handling-growth-with-postgres-5-tips-from-instagram-84f86b0bb674)

4. **Slack: Caching at Scale**  
   [https://slack.engineering/caching-for-a-global-netflix/](https://slack.engineering/flannel-an-application-level-edge-cache-to-make-slack-scale/)

5. **Twitter: Timeline Cache Architecture**  
   [https://www.infoq.com/presentations/Twitter-Timeline-Scalability/](https://www.infoq.com/presentations/Twitter-Timeline-Scalability/)

6. **Shopify: Caching at Scale**  
   [https://engineering.shopify.com/blogs/engineering/caching-at-shopify-how-we-handle-a-billion-requests-per-day](https://engineering.shopify.com/blogs/engineering/caching-at-shopify-how-we-handle-a-billion-requests-per-day)

7. **XFetch: Optimal Probabilistic Cache Stampede Prevention (Research Paper)**  
   [https://cseweb.ucsd.edu/~avattani/papers/cache_stampede.pdf](https://cseweb.ucsd.edu/~avattani/papers/cache_stampede.pdf)

8. **GitHub: How We Optimized Our Read Replicas to Reduce p99 Latency**  
   [https://github.blog/engineering/infrastructure/how-we-scaled-github-with-read-replicas/](https://github.blog/engineering/infrastructure/how-we-scaled-github-with-read-replicas/)
