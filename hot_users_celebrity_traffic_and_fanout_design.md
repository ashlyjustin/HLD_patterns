# Hot Users, Celebrity Traffic, and Fan-out Design

> **What this doc covers:** How to architect systems that stay performant and correct when a small number of users — celebrities, influencers, viral accounts — generate traffic that is orders of magnitude higher than average. Covers the fan-out problem in depth (why naive approaches explode at scale), the hybrid fan-out strategy used by Twitter and Instagram, cache stampede prevention, notification delivery at 50M-follower scale, write hot spots from viral content, and the tricky edge cases like celebrity account deletion and real-time leaderboards.

---

### 📋 Outline

| # | Section | What You'll Learn |
|---|---------|-------------------|
| 1 | What Is the Hot User Problem? | Why power-law user distribution breaks systems built for average load |
| 2 | Why Standard Approaches Break | Why fan-out on write fails for celebrities; why fan-out on read fails for regular users |
| 3 | The Hybrid Fan-out Strategy | How Twitter/Instagram solve this: push for regular users, pull for celebrities — with full pseudocode |
| 4 | Notification Fan-out for Hot Users | Multi-layer queue partitioning for delivering 50M notifications; priority lanes; staleness checks |
| 5 | Caching Strategy for Hot Users | Pre-warming, thundering herd prevention (mutex, XFetch), tiered caching |
| 6 | Write Hot Spots from Hot Users | Redis INCR + async DB flush for viral like/view counts |
| 7 | Hot Users in Search and Ranking | Pre-computed materialized views, Redis sorted sets for leaderboards |
| 8 | Regular Scenarios | Instagram celebrity post, YouTube viral video |
| 9 | Tricky Scenarios | Celebrity joins mid-traffic, unfollow cascade, CDN origin stampede, real-time leaderboard coalescing |
| 10 | Architecture Summary | Full write and read path diagrams |
| 11 | Reference Case Studies | Twitter, Instagram, Facebook TAO, YouTube, Discord |

---

## 1. What Is the Hot User Problem?

In most social platforms, user activity follows a **power-law distribution**:
- Top 0.01% of users (celebrities, brands) may have 10–500M followers
- A single post from Elon Musk (~180M followers) or Taylor Swift (~280M followers) triggers:
  - Millions of notification deliveries
  - Millions of timeline inserts (fan-out on write)
  - Millions of cache invalidations
  - Massive read amplification

This is also called the **Celebrity Problem**, **Thundering Herd from a Single User**, or the **Fan-out Problem**.

---

## 2. Why Standard Approaches Break

### Fan-out on Write (Precomputed Timelines)
Works great for regular users. When a regular user posts:
1. Write post to posts table
2. For each of N followers, insert post_id into their timeline table
3. Read is instant — just fetch user's timeline rows

**Breaks for celebrities:** If a celebrity has 50M followers and posts 3 times a day, that's **150M timeline inserts per day from one user**. At even 10k celebrity posts/day across all celebrities, you're doing billions of inserts just for fan-out.

### Fan-out on Read (Pull at Query Time)
Works for celebrities. When a user opens their feed:
1. Fetch list of people they follow
2. For each followed account, fetch latest N posts
3. Merge and sort

**Breaks for regular users at scale:** Following 500 people means 500 DB reads per feed refresh. Multiply by 100M users refreshing feeds → catastrophic.

---

## 3. The Hybrid Fan-out Strategy

The industry-standard solution, used by Twitter, Instagram, and Facebook:

```
User posts:
  ├─ Is this user a "celebrity" (followers > threshold)?
  │   └─ YES: Fan-out on READ (pull-based). Don't write to timelines.
  │   └─ NO:  Fan-out on WRITE (push-based). Write to follower timelines.
  
User reads their feed:
  1. Pull pre-computed timeline from cache (fan-out on write for regular users)
  2. Merge in latest posts from celebrities they follow (fan-out on read)
  3. Sort and deduplicate merged results
  4. Return to user
```

**Threshold:** Twitter historically used ~1M followers as the "celebrity" cutoff.

```python
CELEBRITY_THRESHOLD = 1_000_000

def on_post_created(post_id, author_id):
    author = get_user(author_id)
    
    if author.follower_count < CELEBRITY_THRESHOLD:
        # Fan-out on write: push to follower timelines
        followers = get_followers(author_id)  # manageable count
        for follower_id in followers:
            timeline_cache.lpush(f"timeline:{follower_id}", post_id)
            timeline_cache.ltrim(f"timeline:{follower_id}", 0, 999)
    # Celebrities: no fan-out. Pull at read time.

def get_timeline(user_id):
    # Step 1: Get pre-computed timeline (non-celebrity posts)
    timeline = timeline_cache.lrange(f"timeline:{user_id}", 0, 99)
    
    # Step 2: Get celebrity posts the user follows
    celebrity_follows = get_celebrity_follows(user_id)  # small list
    celebrity_posts = []
    for celeb_id in celebrity_follows:
        posts = post_store.get_recent(celeb_id, limit=20)
        celebrity_posts.extend(posts)
    
    # Step 3: Merge and sort
    all_posts = timeline + celebrity_posts
    return sorted(all_posts, key=lambda p: p.timestamp, reverse=True)[:100]
```

---

## 4. Notification Fan-out for Hot Users

### Problem
When a celebrity posts, you need to notify millions of followers. A naive approach:
```python
for follower_id in get_all_followers(celebrity_id):  # 50M rows
    send_notification(follower_id, post_id)  # 50M notifications
```

This takes hours, and the celebrity's next post may arrive before the first one finishes.

### Solution: Multi-Layer Fan-out with Queue Partitioning

```
Celebrity Post
     │
     ▼
Fan-out Service
     │
     ├──► Kafka topic: "notifications" (100 partitions)
     │         │
     │    [Fan-out Workers × 100]
     │         │
     │    Each worker handles 1 partition
     │    Processes ~500k followers/partition
     │
     ▼
Push Notification Service (APNs, FCM)
     │
     ├── Batch size: 1000 per API call
     ├── Rate limited to FCM quotas
     └── Async with retries
```

**Key techniques:**
- **Partitioned queues:** Spread fan-out work across many workers
- **Batching:** Group notification sends (1000 per API call vs 1 per call)
- **Priority lanes:** High-priority users (active now, paid tier) get notifications first
- **Staleness check:** If notification delivery takes >1hr, skip — user won't care about a 2-hour-old post notification

---

## 5. Caching Strategy for Hot Users

### Problem
A hot user's profile, post counts, and latest posts are requested millions of times/minute. A cache miss → DB stampede.

### Pattern 1: Pre-warm Cache on Celebrity Posts
When a celebrity posts, proactively push data to CDN edge nodes and application caches.

```python
def on_celebrity_post(post_id, celebrity_id):
    # Pre-warm: push post to all cache layers
    redis_cluster.setex(f"post:{post_id}", ttl=3600, value=serialize(post))
    cdn.purge_and_reprime(f"/users/{celebrity_id}/feed")
    
    # Fan-out via queue (not blocking the post API)
    queue.publish("celebrity_post_fanout", {
        "post_id": post_id,
        "celebrity_id": celebrity_id
    })
```

### Pattern 2: Thundering Herd on Cache Miss — Mutex / Probabilistic Early Expiry

When a cache entry expires for a hot user, thousands of requests simultaneously miss and hammer the DB.

**Mutex approach (cache stampede prevention):**
```python
def get_celebrity_profile(user_id):
    cached = redis.get(f"profile:{user_id}")
    if cached:
        return cached
    
    # Try to acquire lock
    lock_acquired = redis.set(f"lock:profile:{user_id}", 1, nx=True, ex=5)
    if lock_acquired:
        # We are the designated fetcher
        profile = db.get_user(user_id)
        redis.setex(f"profile:{user_id}", 300, serialize(profile))
        return profile
    else:
        # Another request is fetching, wait briefly and retry from cache
        time.sleep(0.1)
        return redis.get(f"profile:{user_id}")  # Should be populated now
```

**XFetch (probabilistic early expiry) — smarter approach:**
Refresh the cache slightly before it expires to avoid stampede:
```python
import math, random

def get_with_xfetch(key, ttl_seconds, fetch_fn, beta=1.0):
    value, expiry = cache.get_with_expiry(key)
    if value is None:
        value = fetch_fn()
        cache.setex(key, ttl_seconds, value)
        return value
    
    time_to_expire = expiry - time.now()
    # Probabilistically refresh earlier as expiry approaches
    if -beta * math.log(random.random()) > time_to_expire:
        value = fetch_fn()  # Refresh early
        cache.setex(key, ttl_seconds, value)
    return value
```

> 📖 **Reference:** [Optimal Probabilistic Cache Stampede Prevention (XFetch paper)](https://cseweb.ucsd.edu/~avattani/papers/cache_stampede.pdf)

---

### Pattern 3: Tiered Caching for Celeb Content

```
Request for @elonmusk's profile
         │
         ▼
    [CDN Cache] ──hit──► Return (0ms, no DB)
         │ miss
         ▼
  [Redis L1 Cache] ──hit──► Return (2ms, no DB)
         │ miss
         ▼
  [Read Replica DB] ──hit──► Return (10ms, no primary DB load)
         │ miss (very rare)
         ▼
  [Primary DB] ──► Populate all cache layers
```

For celebrities, keep TTL very short (5–30s) rather than relying on explicit invalidation — stale celebrity data for 30s is acceptable.

---

## 6. Write Hot Spots from Hot Users

### Problem
A celebrity's post gets 10M likes in an hour. `UPDATE posts SET like_count = like_count + 1 WHERE post_id = X` is a single-row hot spot.

### Solution: Write Buffering + Eventual Aggregation

```
Like Action
     │
     ▼
Redis INCR  (atomic, in-memory)
     │
     ▼ (background job, every 5 seconds)
Flush to DB: UPDATE posts SET like_count = like_count + N
             WHERE post_id = X
```

**Implementation:**
```python
# On like:
pipeline = redis.pipeline()
pipeline.incr(f"likes:post:{post_id}")
pipeline.expire(f"likes:post:{post_id}", 86400)
pipeline.execute()

# Background flusher (every 5s):
for key in redis.scan("likes:post:*"):
    post_id = key.split(":")[-1]
    count = redis.getdel(key)  # atomic get-and-delete
    if count:
        db.execute("UPDATE posts SET like_count = like_count + %s WHERE post_id = %s",
                   [count, post_id])
```

**Tradeoff:** Like count shown may be slightly stale (5s). Acceptable for most products; TikTok, YouTube, Instagram all use this pattern.

---

## 7. Hot User in Search / Ranking

### Problem
Trending topic pages and "Who to follow" surfaces involving celebrities need real-time data. Query `SELECT followers FROM users WHERE id = celebrity ORDER BY follower_count` might have cache hits, but ranking algorithms involving 50k candidates are expensive.

### Solution: Pre-computed materialized views + async updates

```sql
-- Materialized "top followed users" — refreshed every 5 minutes
CREATE MATERIALIZED VIEW top_users AS
SELECT id, username, follower_count, verified
FROM users
WHERE follower_count > 100000
ORDER BY follower_count DESC;

-- Refresh async
REFRESH MATERIALIZED VIEW CONCURRENTLY top_users;
```

Or in application layer: a background job maintains a Redis sorted set.
```python
# Updated on every follow event:
redis.zadd("top_users", {celebrity_id: new_follower_count})

# Read:
top_100 = redis.zrevrange("top_users", 0, 99, withscores=True)
```

---

## 8. Regular Scenarios

### Scenario 1: Instagram — Post from a Verified Celebrity
Selena Gomez (430M followers) posts a new photo.

**Flow:**
1. Photo uploaded → stored in S3, CDN URL generated
2. Post record written to Cassandra (post store)
3. Fan-out service checks: follower count > 1M → celebrity path
4. **No timeline inserts** — timeline for her followers fetches on read
5. Push notification queued → Kafka with 500 partitions → 500 notification workers
6. Notification workers batch-deliver to FCM/APNs
7. Feed read requests merge her post on-the-fly from post store

---

### Scenario 2: YouTube — Viral Video
Mr. Beast uploads a video. 50M subscribers.

**Write path:** Video metadata → PostgreSQL. Video file → S3/CDN. Subscriber notification → SNS-style fan-out queue.

**Read path:** Video page is cached at CDN edge (ttl=60s). View count increment via Redis batching. Recommendation engine adds it to "trending" bucket.

**Hot key mitigation:** If the Redis key `views:video:abc123` gets 100k ops/sec:
- Use Redis Cluster → key shards across nodes
- If still hot: use local in-process buffer (accumulate increments for 100ms, then flush to Redis in one command)

---

## 9. Tricky Scenarios

### Tricky Scenario 1: Celebrity Joins Your Platform
A platform (e.g., a new social app) suddenly attracts a major celebrity who announces a live session.

**Problem:** No caches are warm, no pre-computed timelines exist, followers see empty feeds or massive delays.

**Solution: Pre-warm on rapid follower growth detection**
```python
# Triggered when follower count crosses celebrity threshold
def on_follower_milestone(user_id, follower_count):
    if follower_count == CELEBRITY_THRESHOLD:
        # Switch user to celebrity mode
        user_store.set_celebrity_flag(user_id, True)
        # Pre-warm recent posts into all follower timelines asynchronously
        async_job.enqueue("backfill_celebrity_posts", user_id)
        # Warm their profile cache aggressively
        cache.setex(f"profile:{user_id}", 3600, fetch_profile(user_id))
```

---

### Tricky Scenario 2: Celebrity Unfollow Cascade
A celebrity (10M followers) deletes their account or is banned. 

**Problem:** Naively removing them from 10M timelines via fan-out on delete would take hours. Meanwhile users see posts from a deleted/banned account.

**Solution:**
1. Immediately add to a `blocked_users` set in Redis (checked on every read)
2. Background job asynchronously cleans up timelines
3. Feed rendering skips any post from blocked accounts

```python
def delete_user(user_id):
    # Immediate: add to blocked set
    redis.sadd("deleted_users", user_id)
    
    # Feed rendering:
    posts = get_timeline(viewer_id)
    posts = [p for p in posts if p.author_id not in redis.smembers("deleted_users")]
    
    # Background cleanup (hours later):
    async_job.enqueue("remove_from_timelines", user_id)
```

---

### Tricky Scenario 3: Hot User Causes Cascading Cache Invalidation
When a celebrity's data changes (bio update, new follower count), cache invalidation fires.

**Problem:** If 1000 CDN edge nodes each try to warm the celebrity cache from origin simultaneously, you get an **origin stampede** even with a warm cache in front.

**Solution: Staggered invalidation + CDN shield**
```
Request → Edge Node → [miss] → Shield Node (single regional POP) → [miss] → Origin
                                    └── Edge nodes all hit shield, not origin
                                    └── Only 1 origin request per cache miss
```

Configure CDN with an "origin shield" — a single regional node that sits in front of origin. All edge nodes pull from shield, not directly from origin.

---

### Tricky Scenario 4: Real-Time Leaderboard with a Celebrity
A gaming platform has a celebrity streamer with 5M followers. Their score updates must appear on a public leaderboard in real-time.

**Problem:** Each score update triggers 5M websocket pushes.

**Solution:** Push to subscribers via pub/sub, but:
1. **Sample, don't push every update.** If score updates every second, push every 5th update (client interpolates)
2. **Coalesce updates.** Buffer 500ms of score changes, send latest
3. **Virtual fan-out:** Group subscribers into fan-out trees. One broadcaster → 100 relay nodes → 50k each

```python
# Leaderboard update with coalescing
def on_score_update(celebrity_id, new_score):
    current = redis.get(f"score_queue:{celebrity_id}")
    redis.setex(f"score_queue:{celebrity_id}", 1, new_score)  # 1s TTL debounce
    
    if current is None:  # First update in window
        asyncio.create_task(flush_score_after_delay(celebrity_id, delay=0.5))

async def flush_score_after_delay(celebrity_id, delay):
    await asyncio.sleep(delay)
    score = redis.get(f"score_queue:{celebrity_id}")
    if score:
        await push_to_subscribers(celebrity_id, score)
```

---

## 10. Architecture Summary

```
                     Celebrity Post Flow
                     ──────────────────
Post Created ──► Identify: Celebrity?
                  │ YES                    │ NO
                  ▼                        ▼
          Post Store Only          Post Store +
          (Cassandra/S3)           Fan-out to follower
                  │                timelines (Redis)
                  ▼
          Notification Queue
          (Kafka, partitioned)
                  │
                  ▼
          Batch Delivery
          (APNs / FCM / Email)

                    Feed Read Flow
                    ──────────────
User Opens Feed
       │
       ├──► Fetch pre-computed timeline (Redis)
       │         (non-celebrity posts)
       │
       ├──► For each celebrity followed:
       │    fetch latest posts from Post Store
       │
       └──► Merge, sort, paginate → Return
```

---

## 11. Reference Case Studies

1. **Twitter's Timeline Architecture (Fan-out evolution)**  
   [https://www.infoq.com/presentations/Twitter-Timeline-Scalability/](https://www.infoq.com/presentations/Twitter-Timeline-Scalability/)

2. **Instagram: Scaling to 14 billion queries per day**  
   [https://instagram-engineering.com/what-powers-instagram-hundreds-of-billions-of-photos-and-videos-54cb56f597cb](https://instagram-engineering.com/what-powers-instagram-hundreds-of-billions-of-photos-and-videos-54cb56f597cb)

3. **Facebook: TAO — The Power of the Graph**  
   [https://www.usenix.org/system/files/conference/atc13/atc13-bronson.pdf](https://www.usenix.org/system/files/conference/atc13/atc13-bronson.pdf)

4. **Cache Stampede Prevention (XFetch)**  
   [https://cseweb.ucsd.edu/~avattani/papers/cache_stampede.pdf](https://cseweb.ucsd.edu/~avattani/papers/cache_stampede.pdf)

5. **YouTube Architecture and Scaling**  
   [https://highscalability.com/youtube-architecture/](https://highscalability.com/youtube-architecture/)

6. **Discord: How We Scaled to Millions of Users**  
   [https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users](https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users)
