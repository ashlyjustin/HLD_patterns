# Large-Scale Deployments with Zero Downtime

> **What this doc covers:** How to deploy new versions of large distributed systems — hundreds of services, millions of active users, global infrastructure — without a maintenance window. Covers the core deployment strategies (rolling, blue-green, canary), the infrastructure primitives that make them possible (load balancers, feature flags, health checks), how to handle the hardest cases (stateful services, database-coupled deploys, multi-region rollouts), observability and rollback triggers, and the failure modes that have caused high-profile outages during deploys.

### 📋 Outline

| # | Section | What You'll Learn |
|---|---------|-------------------|
| 1 | Why Zero-Downtime Deploys Are Hard | Coupling, in-flight requests, state, the simultaneous-version problem |
| 2 | Core Deployment Strategies | Rolling, blue-green, canary — mechanics, trade-offs, when to use each |
| 3 | Infrastructure Primitives | Load balancers, health checks, readiness probes, connection draining |
| 4 | Feature Flags and Dark Launches | Decoupling deploy from release; targeting, gradual rollout, kill switches |
| 5 | Handling Stateful Services | Sticky sessions, database-coupled deploys, cache invalidation, WebSockets |
| 6 | Multi-Region and Global Rollouts | Ring-based deploys, region ordering, geo-routing, cross-region state |
| 7 | Kubernetes-Native Deployment Patterns | Deployments, PodDisruptionBudgets, lifecycle hooks, Argo Rollouts |
| 8 | Observability and Rollback Triggers | SLO-gated deploys, automated rollback, deployment events in dashboards |
| 9 | Regular Scenarios | Stateless service deploy, config change rollout, dependency upgrade |
| 10 | Tricky Scenarios | Schema + code coupled deploy, long-running jobs, thundering herd on restart |
| 11 | Common Failure Modes | The 10 mistakes that have caused production outages during deploys |
| 12 | Reference Case Studies | Netflix, Etsy, Slack, Amazon, Cloudflare, GitHub |

---

## 1. Why Zero-Downtime Deploys Are Hard

### 1.1 The Five Sources of Downtime During a Deploy

**1. In-flight request termination** — A pod is killed while it's mid-response. The client gets a connection reset.

**2. Startup gap** — The new version is starting up (loading models, warming caches, connecting to DBs) but hasn't passed health checks. If old pods are already gone, there's a window with no healthy pods.

**3. Simultaneous-version incompatibility** — During a rolling deploy, v1 and v2 run at the same time. If v2 produces messages/events/DB writes that v1 cannot understand, errors cascade.

**4. Dependency version skew** — Service A deploys a change that requires Service B to be on v2. But Service B is still on v1. A → B calls fail until B deploys.

**5. State migration** — A new version needs data in a different format (different cache key, different config shape, new schema). Old data is still there; new code breaks on it.

### 1.2 The Simultaneous-Version Constraint

```
Time ──────────────────────────────────────────────────────▶
      │  v1 only  │     v1 + v2 mixed     │  v2 only  │
      └───────────┴───────────────────────┴───────────┘
                  ↑                       ↑
              first new pod          last old pod
              goes live              drained
```

During the mixed window, **all of the following must be true simultaneously:**
- v1 can read any data that v2 writes
- v2 can read any data that v1 writes
- v1 and v2 can consume the same API contracts from dependencies
- External clients calling either v1 or v2 pods get correct results

Violating any one of these causes errors during the mixed window, which can last minutes to hours depending on deploy speed.

### 1.3 Graceful Shutdown: The First Thing to Get Right

Before tackling deployment strategy, every service needs correct graceful shutdown:

```
SIGTERM received
      │
      ▼
Stop accepting new connections (remove from load balancer first)
      │
      ▼
Wait for in-flight requests to complete (drain)
      │
      ▼
Close DB connections, flush buffers, checkpoint state
      │
      ▼
Exit 0
```

**Node.js example:**

```javascript
const server = app.listen(3000);

process.on('SIGTERM', async () => {
  console.log('SIGTERM received, starting graceful shutdown');
  
  // Stop accepting new connections
  server.close(async () => {
    console.log('HTTP server closed');
    
    // Close DB pool — waits for in-flight queries to complete
    await db.pool.end();
    
    // Close message queue consumer
    await consumer.stop();
    
    process.exit(0);
  });
  
  // Force exit if graceful shutdown takes too long
  setTimeout(() => {
    console.error('Graceful shutdown timed out, forcing exit');
    process.exit(1);
  }, 30000);
});
```

**Python/FastAPI:**

```python
import signal
import asyncio

async def shutdown(signal, loop):
    print(f"Received {signal.name}, shutting down...")
    tasks = [t for t in asyncio.all_tasks() if t is not asyncio.current_task()]
    [task.cancel() for task in tasks]
    await asyncio.gather(*tasks, return_exceptions=True)
    loop.stop()

loop = asyncio.get_event_loop()
for sig in (signal.SIGTERM, signal.SIGINT):
    loop.add_signal_handler(sig, lambda s=sig: asyncio.create_task(shutdown(s, loop)))
```

**Kubernetes: preStop hook** (gives Kubernetes a grace window before sending SIGTERM):

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]
# This 5-second sleep gives the load balancer time to deregister the pod
# before the process starts shutting down
terminationGracePeriodSeconds: 60
```

---

## 2. Core Deployment Strategies

### 2.1 Rolling Deployment

Replace instances one (or a few) at a time. At any point, a mix of old and new versions is running.

```
Start:   [v1][v1][v1][v1][v1][v1]
Step 1:  [v2][v1][v1][v1][v1][v1]
Step 2:  [v2][v2][v1][v1][v1][v1]
...
Done:    [v2][v2][v2][v2][v2][v2]
```

**How it works in Kubernetes:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # Spin up 2 extra pods above desired count
      maxUnavailable: 0  # Never kill old pods until new ones are ready
                         # This is the zero-downtime configuration
  template:
    spec:
      containers:
      - name: my-service
        image: my-service:v2
        readinessProbe:
          httpGet:
            path: /healthz/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
```

**`maxSurge: 2, maxUnavailable: 0` — the zero-downtime math:**
- Desired replicas: 10
- During rollout: up to 12 pods exist (10 old + 2 new starting)
- A new pod is only promoted when its readiness probe passes
- An old pod is only terminated after a new pod is ready
- At all times, exactly 10 ready pods serve traffic

**Trade-offs:**

| Pro | Con |
|-----|-----|
| No extra infrastructure needed | v1 + v2 mixed window — must be backwards-compatible |
| Gradual resource consumption | Slow rollout for large fleets |
| Works on any platform | Hard to target specific users |
| Easy rollback (just reverse the rollout) | Rollback is also gradual |

**When to use:** Default strategy for stateless services with backwards-compatible changes.

---

### 2.2 Blue-Green Deployment

Maintain two identical environments (Blue = live, Green = staging). Switch all traffic at once.

```
              ┌──────────────────┐
              │   Load Balancer  │
              └────────┬─────────┘
                       │ 100% traffic
            ┌──────────▼──────────┐
            │   Blue (v1) — LIVE  │
            └─────────────────────┘
            
            ┌─────────────────────┐
            │  Green (v2) — IDLE  │
            └─────────────────────┘
                       │
               fully tested, ready

// Switch:
            ┌──────────────────┐
            │   Load Balancer  │
            └────────┬─────────┘
                     │ 100% traffic
          ┌──────────▼──────────┐
          │  Green (v2) — LIVE  │
          └─────────────────────┘
          
          ┌─────────────────────┐
          │  Blue (v1) — IDLE   │ ← keep warm for rollback
          └─────────────────────┘
```

**Traffic switch (AWS ALB):**

```bash
# Shift 100% of traffic to Green target group
aws elbv2 modify-listener \
  --listener-arn arn:aws:elasticloadbalancing:... \
  --default-actions Type=forward,TargetGroupArn=$GREEN_TG_ARN
```

**Or with weighted routing for gradual cut-over:**

```bash
# Start: 90% Blue, 10% Green
aws elbv2 modify-listener \
  --listener-arn $LISTENER_ARN \
  --default-actions '[{
    "Type": "forward",
    "ForwardConfig": {
      "TargetGroups": [
        {"TargetGroupArn": "$BLUE_TG", "Weight": 90},
        {"TargetGroupArn": "$GREEN_TG", "Weight": 10}
      ]
    }
  }]'
```

**Database challenge:** Blue and Green share a database (usually). This means your schema must be compatible with both v1 and v2 simultaneously — use the Expand-Contract pattern from `database_migration_in_prod.md`.

**Trade-offs:**

| Pro | Con |
|-----|-----|
| Instant rollback (flip traffic back) | Double infrastructure cost during switch |
| No mixed-version window | Requires identical environment parity |
| Full pre-production testing on real infra | Database/state synchronisation complexity |
| Clean cut-over | Session stickiness issues (users switch mid-session) |

**When to use:** Major releases, compliance-sensitive systems, whenever instant rollback is required.

---

### 2.3 Canary Deployment

Route a small percentage of real traffic to the new version. Observe. Gradually increase.

```
              ┌──────────────────┐
              │   Load Balancer  │
              └────────┬─────────┘
                  95%  │  5%
           ┌──────────┘└──────────┐
           ▼                      ▼
    [v1][v1][v1][v1]          [v2]  ← canary
    (production fleet)      (1 pod)
```

**Canary progression:**

```
5% → observe (5 min) → 20% → observe (10 min) → 50% → observe → 100%
```

At each step, check: error rate, latency p99, business metrics (conversion rate, checkout success).

**Nginx canary split (Kubernetes ingress):**

```yaml
# Canary ingress — routes 5% of traffic to v2
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-service-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "5"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service-v2
            port:
              number: 80
```

**Argo Rollouts canary with analysis:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-service
spec:
  strategy:
    canary:
      steps:
      - setWeight: 5
      - pause: {duration: 5m}
      - setWeight: 20
      - pause: {duration: 10m}
      - analysis:
          templates:
          - templateName: success-rate-check
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 100
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate-check
spec:
  metrics:
  - name: success-rate
    interval: 60s
    successCondition: result[0] >= 0.99  # 99% success rate required
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{status=~"2..",deployment="my-service-canary"}[5m]))
          /
          sum(rate(http_requests_total{deployment="my-service-canary"}[5m]))
```

**Header-based canary (for deterministic testing):**

```yaml
annotations:
  nginx.ingress.kubernetes.io/canary: "true"
  nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
  nginx.ingress.kubernetes.io/canary-by-header-value: "true"
# curl -H "X-Canary: true" https://api.example.com/endpoint
# Routes to v2 regardless of weight — useful for QA testing in production
```

**Trade-offs:**

| Pro | Con |
|-----|-----|
| Real traffic testing | Complex routing logic |
| Limits blast radius to 5% of users | v1 + v2 mixed window (same as rolling) |
| Measurable, data-driven promotion | Slower full rollout |
| Easy rollback (reduce weight to 0) | Metrics need careful isolation per version |

**When to use:** High-risk changes, performance-sensitive code, any change touching money or data integrity.

---

### 2.4 Strategy Comparison

| Dimension | Rolling | Blue-Green | Canary |
|-----------|---------|-----------|--------|
| Mixed-version window | Yes (long) | No | Yes (small) |
| Blast radius if bad | 100% gradually | 0% (instant flip) | 5% initially |
| Rollback speed | Gradual (~same as deploy) | Instant | Instant (weight to 0) |
| Infrastructure overhead | None | 2x during switch | Minimal |
| Real traffic validation | No | No | Yes |
| Suitable for schema changes | With backwards compat | With backwards compat | With backwards compat |
| Best for | Stateless microservices | Major releases | High-risk / data changes |

---

## 3. Infrastructure Primitives

### 3.1 Health Checks: Liveness vs Readiness vs Startup

These three probes are distinct and are all required for zero-downtime:

```yaml
livenessProbe:
  httpGet:
    path: /healthz/live    # Am I running? (if fails → restart me)
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /healthz/ready   # Am I ready for traffic? (if fails → remove from LB)
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3      # 3 failures = 15s without traffic, then re-check

startupProbe:
  httpGet:
    path: /healthz/started # Am I done starting? (disables liveness during slow starts)
    port: 8080
  failureThreshold: 30     # Allow up to 30*10s = 5 minutes for slow start
  periodSeconds: 10
```

**Health check endpoint implementation:**

```python
from fastapi import FastAPI, Response
import asyncpg

app = FastAPI()

@app.get("/healthz/live")
async def liveness():
    # Minimal check: is the process alive and event loop running?
    return {"status": "ok"}

@app.get("/healthz/ready")
async def readiness(response: Response):
    checks = {}
    
    # Check DB connection
    try:
        await db.execute("SELECT 1")
        checks["db"] = "ok"
    except Exception as e:
        checks["db"] = f"error: {e}"
        response.status_code = 503
    
    # Check Redis connection
    try:
        await redis.ping()
        checks["redis"] = "ok"
    except Exception as e:
        checks["redis"] = f"error: {e}"
        response.status_code = 503
    
    # Check: is warm-up complete? (e.g., model loaded, cache warmed)
    if not app.state.warmup_complete:
        checks["warmup"] = "pending"
        response.status_code = 503
    
    return {"status": "ok" if response.status_code == 200 else "degraded", "checks": checks}
```

**The critical distinction:**
- **Liveness failure** → Kubernetes kills and restarts the pod (may not fix things if DB is down — restart loops are bad)
- **Readiness failure** → Kubernetes removes pod from load balancer, but does NOT kill it (correct response to dependency outage)

**Common mistake:** Putting DB health check in liveness instead of readiness. When DB goes down, all pods restart in a loop, making the outage worse.

### 3.2 Connection Draining

When a pod is deregistered from the load balancer, in-flight requests must be allowed to complete.

**AWS ALB deregistration delay:**

```bash
aws elbv2 modify-target-group-attributes \
  --target-group-arn $TG_ARN \
  --attributes Key=deregistration_delay.timeout_seconds,Value=60
# ALB keeps sending existing connections to the draining target for 60 seconds
# New connections are not sent to the draining target
```

**Kubernetes `terminationGracePeriodSeconds`** must be > your deregistration delay:

```yaml
spec:
  terminationGracePeriodSeconds: 90  # Must be > ALB deregistration delay (60s)
  containers:
  - lifecycle:
      preStop:
        exec:
          command: ["/bin/sleep", "65"]  # Wait for LB to stop sending new requests
```

**The full graceful shutdown sequence:**

```
1. Pod receives SIGTERM
2. preStop hook runs: sleep 65s (LB finishes deregistering)
3. LB stops sending new connections to this pod
4. App receives SIGTERM (after preStop completes)
5. App drains in-flight requests (up to 25s remaining)
6. App exits 0
7. Kubernetes confirms pod terminated
```

### 3.3 Load Balancer Behaviour During Deploys

**Layer 4 (TCP) LB:** Connections are pinned. If the backend pod dies, the connection breaks — client must reconnect.

**Layer 7 (HTTP) LB:** Each request is independently routed. Pod death only affects requests that are literally mid-flight on that pod.

**Implication:** For zero-downtime, use Layer 7 load balancing. Or, if using L4, ensure your clients retry on connection failure (which they should anyway).

**Session affinity (sticky sessions) and rolling deploys:** If you have sticky sessions pinned by cookie, users stuck to a pod will lose their session when that pod is killed. Solutions:
1. Externalize sessions to Redis (best)
2. Use connection draining with a long enough window
3. Issue new session cookies on reconnect

---

## 4. Feature Flags and Dark Launches

### 4.1 Deploy vs Release

The central insight of feature flags:

```
Deploy   = shipping the code to production
Release  = making the feature available to users

These can (and should) be independent events.
```

Code for a new feature can sit dormant in production for days before being enabled. This eliminates the "big bang" deploy risk.

### 4.2 Feature Flag Anatomy

```python
from flagsmith import Flagsmith

client = Flagsmith(environment_api_key="ser.abc123")

def should_use_new_checkout(user_id: int) -> bool:
    # Check flag — resolves in order:
    # 1. User-level override (kill switch for specific users)
    # 2. Segment membership (beta users, employees, specific org_ids)
    # 3. Percentage rollout (hash of user_id → stable bucket)
    # 4. Default (global on/off)
    flags = client.get_identity_flags(str(user_id))
    return flags.is_feature_enabled("new_checkout")

# Usage
if should_use_new_checkout(current_user.id):
    return new_checkout_flow(cart)
else:
    return legacy_checkout_flow(cart)
```

**Stable bucketing** (same user always gets same experience):

```python
import hashlib

def get_user_bucket(user_id: int, flag_name: str) -> int:
    """Returns 0-99, stable for the same user+flag combination."""
    key = f"{flag_name}:{user_id}"
    hash_bytes = hashlib.md5(key.encode()).digest()
    return int.from_bytes(hash_bytes[:4], 'big') % 100

def is_in_rollout(user_id: int, flag_name: str, rollout_pct: int) -> bool:
    return get_user_bucket(user_id, flag_name) < rollout_pct
```

### 4.3 Flag Types

| Type | Use Case | Example |
|------|----------|---------|
| **Boolean kill switch** | Emergency feature disable | `new_payment_processor: false` |
| **Percentage rollout** | Gradual release | `new_search: 10% → 50% → 100%` |
| **User targeting** | Beta users, employees | `dark_mode: segment=beta_users` |
| **Org/tenant targeting** | Enterprise feature rollout | `sso_login: org_id IN [42, 87]` |
| **A/B experiment** | Measure impact | `checkout_layout: variant_a / variant_b` |
| **Ops flag** | Infrastructure switch | `use_new_cache_cluster: false` |

### 4.4 Dark Launches

Deploy new code that runs silently alongside the old code, comparing results without affecting users.

```python
async def get_recommendations(user_id: int) -> list[Item]:
    # Old path — always used for response
    result = await old_recommender.get(user_id)
    
    # Dark launch: run new algorithm but don't use result
    if feature_flags.is_enabled("dark_launch_new_recommender"):
        asyncio.create_task(
            shadow_compare(user_id, result, new_recommender.get(user_id))
        )
    
    return result

async def shadow_compare(user_id, old_result, new_result_coro):
    try:
        new_result = await asyncio.wait_for(new_result_coro, timeout=0.5)
        
        # Log comparison without affecting user
        metrics.histogram("recommender.latency_diff", 
                         new_result.latency - old_result.latency)
        metrics.gauge("recommender.overlap", 
                     len(set(old_result.ids) & set(new_result.ids)) / len(old_result.ids))
    except asyncio.TimeoutError:
        metrics.increment("recommender.shadow_timeout")
    except Exception as e:
        metrics.increment("recommender.shadow_error", tags={"error": type(e).__name__})
```

### 4.5 Flag Lifecycle and Tech Debt

Feature flags that are never cleaned up become permanent complexity. Track them:

```sql
CREATE TABLE feature_flags (
    name TEXT PRIMARY KEY,
    description TEXT,
    type TEXT,            -- 'release', 'experiment', 'ops', 'kill_switch'
    created_at TIMESTAMPTZ DEFAULT now(),
    target_removal_date DATE,
    owner TEXT
);

-- Alert when flags are past their target removal date
SELECT name, owner, target_removal_date
FROM feature_flags
WHERE target_removal_date < CURRENT_DATE
  AND type = 'release';
```

---

## 5. Handling Stateful Services

### 5.1 Sticky Sessions and Session Draining

**Problem:** Users are routed to the same pod (sticky sessions via cookie). During a rolling deploy, pods are replaced. Users lose session.

**Solution 1 (preferred): Externalize sessions**

```python
# Store session in Redis, not in pod memory
from fastapi import Request
import redis.asyncio as redis

r = redis.Redis(host="redis-cluster")

async def get_session(request: Request) -> dict:
    session_id = request.cookies.get("session_id")
    data = await r.get(f"session:{session_id}")
    return json.loads(data) if data else {}

async def save_session(session_id: str, data: dict, ttl: int = 3600):
    await r.setex(f"session:{session_id}", ttl, json.dumps(data))
```

**Solution 2: Long drain window**

```yaml
# Keep pod in service long enough for sessions to expire naturally
terminationGracePeriodSeconds: 7200  # 2 hours = session TTL
# With preStop hook making the pod unresponsive to health checks
# so LB removes it from rotation, but existing connections finish
lifecycle:
  preStop:
    exec:
      command: ["/usr/local/bin/graceful-drain"]
      # This script: 1) marks /healthz/ready as 503, 2) waits for connections to drop to 0
```

### 5.2 Cache Invalidation During Deploy

New version may produce/read different cache key formats. Two failure modes:

**Poison:** v1 writes `cache:user:42 = {name: "Alice"}`. v2 expects `cache:user:42 = {name: "Alice", tier: "premium"}`. v2 reads stale structure → runtime error.

**Cold cache thundering herd:** New version uses different key pattern. All existing cache entries are misses. Every request goes to DB simultaneously → DB overload.

**Solutions:**

```python
# Cache key versioning — bump version when structure changes
CACHE_VERSION = "v3"

def user_cache_key(user_id: int) -> str:
    return f"user:{CACHE_VERSION}:{user_id}"

# Old keys (v2, v1) simply expire naturally — no stampede
# Warm the new key in the background during startup
async def warm_cache_on_startup():
    hot_user_ids = await db.fetchall(
        "SELECT id FROM users ORDER BY last_active_at DESC LIMIT 10000"
    )
    for batch in chunks(hot_user_ids, 100):
        await asyncio.gather(*[prime_user_cache(uid) for uid in batch])
        await asyncio.sleep(0.1)  # Don't overload DB during warmup
```

### 5.3 Message Queue Consumers During Deploy

**Problem:** A rolling deploy stops a consumer pod mid-processing. The message is re-queued and processed again by another pod. Non-idempotent handlers cause duplicate side effects.

**Requirement:** All message handlers must be idempotent.

```python
async def handle_order_shipped(message: Message):
    order_id = message.data["order_id"]
    
    # Idempotency check: has this message been processed?
    key = f"processed:{message.message_id}"
    already_processed = await redis.set(key, "1", nx=True, ex=86400)
    
    if not already_processed:
        # Already processed — safe to ACK and skip
        await message.ack()
        return
    
    # Process the message
    await send_shipping_notification(order_id)
    await update_order_status(order_id, "shipped")
    await message.ack()
```

**Graceful consumer shutdown:**

```python
class GracefulConsumer:
    def __init__(self):
        self.running = True
        self.in_flight = 0
        signal.signal(signal.SIGTERM, self.handle_sigterm)
    
    def handle_sigterm(self, signum, frame):
        print("SIGTERM received — draining consumer")
        self.running = False
        # Stop pulling new messages but finish current ones
    
    async def run(self):
        async for message in self.queue.subscribe():
            if not self.running:
                # Nack and requeue — let another consumer handle
                await message.nack(requeue=True)
                break
            
            self.in_flight += 1
            try:
                await self.process(message)
            finally:
                self.in_flight -= 1
        
        # Wait for all in-flight messages to finish
        while self.in_flight > 0:
            await asyncio.sleep(0.1)
        
        print("Consumer drained cleanly")
```

### 5.4 WebSocket Connections During Deploy

WebSockets are long-lived connections. When a pod is replaced, all WebSocket connections to it drop, causing client reconnects. At scale, this causes a thundering reconnect storm.

**Solution 1: Connection draining with long grace period**

```yaml
terminationGracePeriodSeconds: 600  # 10 minutes to drain WebSocket connections
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "
        # Mark pod as draining — load balancer stops sending new connections
        touch /tmp/draining
        # Broadcast 'server-going-down' message to all WebSocket clients
        curl -X POST http://localhost:8080/internal/drain-connections
        # Wait for clients to reconnect elsewhere
        sleep 60
      "]
```

**Solution 2: Client-side reconnect with jitter**

```javascript
class ReconnectingWebSocket {
  constructor(url) {
    this.url = url;
    this.attempts = 0;
    this.connect();
  }
  
  connect() {
    this.ws = new WebSocket(this.url);
    
    this.ws.onclose = (event) => {
      if (event.code === 1001) {
        // Server going away — reconnect with exponential backoff + jitter
        const delay = Math.min(1000 * 2 ** this.attempts, 30000);
        const jitter = Math.random() * 1000;
        setTimeout(() => this.connect(), delay + jitter);
        this.attempts++;
      }
    };
    
    this.ws.onopen = () => {
      this.attempts = 0; // Reset on successful connection
    };
  }
}
```

**Solution 3: Sticky-by-design with pub/sub backend**

```
WebSocket Gateway (stateless, route to any pod)
          │
          ▼
    Redis Pub/Sub  ← all pods subscribe to all topics
          ▲
          │
   Backend Services (send to Redis, not to specific pod)
```

Any pod can handle any WebSocket connection. Replacing a pod only affects reconnect, not message delivery.

---

## 6. Multi-Region and Global Rollouts

### 6.1 Ring-Based Deployment

Don't deploy to all regions simultaneously. Use a promotion ring:

```
Ring 0: Internal (employees only)  → deploy immediately
Ring 1: Canary region (1 region, 1% global traffic)  → promote after 30 min
Ring 2: Expansion (5 regions, 30% global traffic)   → promote after 2 hours
Ring 3: Global (all regions)       → promote after 24 hours
```

```yaml
# Argo Rollouts with ring-based analysis
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
      - setWeight: 1      # Ring 1: 1% traffic
      - pause:
          duration: 30m
      - analysis:
          templates:
          - templateName: error-rate
      - setWeight: 30     # Ring 2: 30% traffic
      - pause:
          duration: 2h
      - analysis:
          templates:
          - templateName: error-rate
          - templateName: latency-p99
      - setWeight: 100    # Ring 3: global
```

### 6.2 Region Ordering Strategy

Choose region order carefully:

```
Bad order:  us-east-1 first (most traffic → maximum blast radius if bad)

Good order:
  ap-southeast-1 (small, different timezone — failure doesn't affect peak US hours)
  → eu-west-1 (medium, validates in EU)
  → us-west-2 (secondary US)
  → us-east-1 (largest — deploy only after confidence established)
```

**Wait for signal between regions:**

```python
def deploy_to_region(region: str, version: str, wait_minutes: int):
    trigger_deploy(region, version)
    
    deadline = time.time() + wait_minutes * 60
    while time.time() < deadline:
        metrics = get_region_metrics(region, version, window="5m")
        
        if metrics.error_rate > 0.01:  # > 1% error rate
            alert(f"High error rate in {region}: {metrics.error_rate:.2%}")
            rollback_region(region)
            raise DeployAborted(f"Aborted after {region} errors")
        
        if metrics.p99_latency_ms > 500:
            alert(f"High latency in {region}: {metrics.p99_latency_ms}ms")
        
        time.sleep(60)
    
    print(f"Region {region} healthy for {wait_minutes} minutes — promoting")

regions = ["ap-southeast-1", "eu-west-1", "us-west-2", "us-east-1"]
for region in regions:
    deploy_to_region(region, version="v2.4.1", wait_minutes=30)
```

### 6.3 Cross-Region State During Rollout

**Problem:** v2 in `us-east-1` writes a new field to DynamoDB global table. v1 in `eu-west-1` reads the record and doesn't understand the new field.

**Solution:** Backwards-compatible writes during the mixed window.

```python
# v2 code:
def write_order(order: Order):
    item = {
        "id": order.id,
        "status": order.status,  # existing field
        # New field: only write if feature flag is 100% globally rolled out
        # During mixed-version window, DON'T write new fields
        # that v1 can't handle
    }
    
    if feature_flags.globally_enabled("new_order_fields"):
        item["fulfillment_type"] = order.fulfillment_type  # new field
    
    dynamodb.put_item(Item=item)
```

Flag is turned on globally only after all regions are on v2.

### 6.4 DNS and GeoDNS Cutover

For major infrastructure changes, use DNS TTL management:

```
72 hours before:   Lower TTL from 300s to 30s
                   (ensures fast DNS propagation when you switch)
                   
Deploy window:     Update DNS records to point to new infra
                   
Wait 5 minutes:    Most clients have picked up new DNS

Post-deploy:       Restore TTL to 300s
```

**Caution:** DNS-based routing has no connection draining. In-flight HTTP/1.1 keep-alive connections continue to hit old infra until the keep-alive timeout. For true zero-downtime, use load balancer weighted routing, not DNS switching.

---

## 7. Kubernetes-Native Deployment Patterns

### 7.1 PodDisruptionBudgets (PDB)

PDBs prevent Kubernetes from removing too many pods simultaneously during node upgrades, cluster autoscaler scale-down, or manual pod evictions.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-service-pdb
spec:
  # At all times, at least 90% of pods must be available
  minAvailable: "90%"
  # OR: no more than 1 pod may be unavailable at a time
  # maxUnavailable: 1
  selector:
    matchLabels:
      app: my-service
```

**Without PDB:** A node drain can evict all pods on that node simultaneously, including 5 out of your 6 replicas. Service goes down.

**With PDB:** Node drain is throttled. Pods are evicted one at a time, waiting for replacements to be ready.

### 7.2 Topology Spread Constraints

Ensure pods are spread across zones so a single zone failure or deploy doesn't take down all pods:

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: my-service
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: my-service
```

With 6 pods across 3 zones: `[2, 2, 2]`. During rolling deploy, skew is at most 1: `[2, 2, 1]` → `[2, 1, 2]` etc. Never `[3, 3, 0]`.

### 7.3 Resource Requests and Limits (Critical for Deploys)

```yaml
resources:
  requests:
    cpu: "500m"    # Scheduler uses this to place the pod
    memory: "512Mi"
  limits:
    cpu: "2000m"   # Burst allowed
    memory: "1Gi"  # Hard limit — OOMKill if exceeded
```

**Why this matters for deploys:** During a rolling deploy with `maxSurge: 2`, you temporarily run more pods than normal. If nodes don't have headroom for the surge pods, they stay `Pending`. The deploy stalls. Old pods are never replaced. **Set cluster autoscaler to pre-provision capacity before deploys.**

### 7.4 Init Containers for Pre-Start Tasks

Run database migrations, configuration validation, or dependency health checks before the main container starts:

```yaml
spec:
  initContainers:
  - name: db-migrate
    image: my-service:v2
    command: ["python", "manage.py", "migrate", "--run-syncdb"]
    # Runs schema migration before any v2 app pod starts
    # Safe only if migration is backwards-compatible (additive)
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: url
  
  - name: wait-for-dependencies
    image: busybox
    command: ["sh", "-c", "until nc -z redis-service 6379; do sleep 1; done"]
    # Don't start until Redis is reachable
  
  containers:
  - name: my-service
    image: my-service:v2
    # Main container only starts after all init containers succeed
```

**Caution:** Running migrations in initContainers means every pod replica tries to run migrations. Use a leader election pattern or an external migration job instead for non-idempotent migrations.

### 7.5 Argo Rollouts: Progressive Delivery

Argo Rollouts extends Kubernetes with richer deployment primitives:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-service
spec:
  replicas: 20
  strategy:
    canary:
      canaryService: my-service-canary  # separate service for canary pods
      stableService: my-service-stable  # separate service for stable pods
      trafficRouting:
        istio:
          virtualService:
            name: my-service-vsvc
      steps:
      - setWeight: 5
      - pause: {}          # Indefinite pause — human approval required
      - setWeight: 25
      - pause: {duration: 10m}
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 100
      analysis:
        startingStep: 2    # Start analysis after first pause
        templates:
        - templateName: my-service-analysis
```

---

## 8. Observability and Rollback Triggers

### 8.1 The Deployment Event Marker

The single most impactful observability improvement for deployments: mark every deploy on every dashboard.

```python
# Send deployment event to Datadog/Grafana/PagerDuty when deploy starts and completes
def mark_deployment(service: str, version: str, environment: str, status: str):
    # Datadog Events API
    requests.post(
        "https://api.datadoghq.com/api/v1/events",
        headers={"DD-API-KEY": DD_API_KEY},
        json={
            "title": f"Deployment: {service} → {version}",
            "text": f"Status: {status}\nEnvironment: {environment}",
            "tags": [f"service:{service}", f"version:{version}", f"env:{environment}"],
            "alert_type": "info" if status == "started" else "success",
        }
    )
```

With this in place, any anomaly on a dashboard is instantly correlated to a deploy event.

### 8.2 SLO-Gated Deploys

Define error budget burn rate thresholds that automatically pause or abort deploys:

```python
class DeployGate:
    def __init__(self, service: str):
        self.service = service
    
    def check_slo(self) -> DeployDecision:
        metrics = self.query_prometheus(f"""
            # 1-hour error rate
            sum(rate(http_requests_total{{service="{self.service}",status=~"5.."}}[1h]))
            /
            sum(rate(http_requests_total{{service="{self.service}"}}[1h]))
        """)
        
        error_rate = metrics[0].value
        
        if error_rate > 0.05:   # > 5% errors
            return DeployDecision.ABORT
        elif error_rate > 0.01: # > 1% errors
            return DeployDecision.PAUSE
        else:
            return DeployDecision.PROCEED
    
    def check_latency(self) -> DeployDecision:
        p99 = self.query_prometheus(f"""
            histogram_quantile(0.99, 
              rate(http_request_duration_seconds_bucket{{service="{self.service}"}}[5m]))
        """)
        
        if p99 > 2.0:    # > 2s p99
            return DeployDecision.ABORT
        elif p99 > 0.5:  # > 500ms p99
            return DeployDecision.PAUSE
        return DeployDecision.PROCEED
```

### 8.3 Automated Rollback

```bash
#!/bin/bash
# deploy.sh — deploy with automated rollback on SLO breach

SERVICE=$1
VERSION=$2
PREVIOUS_VERSION=$(kubectl get deployment $SERVICE -o jsonpath='{.spec.template.spec.containers[0].image}' | cut -d: -f2)

echo "Deploying $SERVICE:$VERSION (previous: $PREVIOUS_VERSION)"

# Apply new version
kubectl set image deployment/$SERVICE $SERVICE=$SERVICE:$VERSION

# Wait for rollout to complete
kubectl rollout status deployment/$SERVICE --timeout=5m

# Monitor for 10 minutes
echo "Monitoring for 10 minutes..."
for i in $(seq 1 10); do
    sleep 60
    ERROR_RATE=$(curl -s "http://prometheus:9090/api/v1/query" \
      --data-urlencode "query=sum(rate(http_requests_total{service=\"$SERVICE\",status=~\"5..\"}[2m]))/sum(rate(http_requests_total{service=\"$SERVICE\"}[2m]))" \
      | jq '.data.result[0].value[1]' -r)
    
    echo "Minute $i — error rate: $ERROR_RATE"
    
    if (( $(echo "$ERROR_RATE > 0.02" | bc -l) )); then
        echo "ERROR RATE TOO HIGH — rolling back to $PREVIOUS_VERSION"
        kubectl rollout undo deployment/$SERVICE
        kubectl rollout status deployment/$SERVICE --timeout=5m
        
        # Alert
        curl -X POST $SLACK_WEBHOOK -d "{
          \"text\": \"🔴 Auto-rollback triggered for $SERVICE. Error rate was $ERROR_RATE. Rolled back to $PREVIOUS_VERSION.\"
        }"
        
        exit 1
    fi
done

echo "✅ Deploy of $SERVICE:$VERSION successful"
```

### 8.4 What to Monitor During a Deploy

| Signal | Threshold to investigate | Threshold to rollback |
|--------|-------------------------|-----------------------|
| Error rate (5xx) | > 0.5% | > 2% |
| p99 latency | > 200ms increase | > 2x baseline |
| p50 latency | > 50ms increase | > 3x baseline |
| Connection pool saturation | > 80% | > 95% |
| Pod restart count | Any | > 3 per pod |
| DB error rate | > 0.1% | > 1% |
| Message queue depth | > 2x normal | > 5x normal |
| Business metric (checkout success) | Any drop | > 1% drop |

---

## 9. Regular Scenarios

### Scenario 1: Stateless Microservice Rolling Deploy

**Context:** `payment-processor` service, 20 replicas, handles 10K req/s, no local state.

**Checklist:**

```
Before deploy:
  ✅ All health check endpoints implemented (/healthz/live, /healthz/ready)
  ✅ Graceful shutdown handles SIGTERM + drains in-flight requests
  ✅ PodDisruptionBudget set to minAvailable: 90%
  ✅ Topology spread constraints ensure zone redundancy
  ✅ terminationGracePeriodSeconds > LB deregistration delay
  ✅ New version is backwards-compatible with any data written by old version
  ✅ New version can handle responses from old downstream services
  
Deploy config:
  maxSurge: 3
  maxUnavailable: 0
  
Monitor (5 minutes post-deploy each batch):
  - error_rate < 0.5%
  - p99_latency < baseline + 100ms
  
Rollback trigger:
  - kubectl rollout undo deployment/payment-processor
```

**Timeline:** 20 replicas with `maxSurge: 3, maxUnavailable: 0` → replace 3 pods at a time → ~7 batches → complete in ~15 minutes.

---

### Scenario 2: Configuration Change Rollout (Feature Flag)

**Context:** Switching the email provider from Sendgrid to SES. Zero risk of code error, but you want to catch integration issues early.

```
Step 1: Deploy code with flag check (flag OFF) → 0% using SES
Step 2: Enable flag for 5% of users (employees + beta)
Step 3: Monitor: delivery rate, bounce rate, spam score
Step 4: Ramp to 25%, 50%, 100% over 48 hours
Step 5: Remove old Sendgrid code path in next release
```

```python
async def send_email(to: str, subject: str, body: str, user_id: int):
    if feature_flags.is_enabled("use_ses", user_id=user_id):
        await ses_client.send(to, subject, body)
        metrics.increment("email.sent", tags={"provider": "ses"})
    else:
        await sendgrid_client.send(to, subject, body)
        metrics.increment("email.sent", tags={"provider": "sendgrid"})
```

---

### Scenario 3: Dependency Library Upgrade (Log4j-style Critical Patch)

**Context:** Critical security patch for a transitive dependency. Must deploy to 200 services within 24 hours. No functional changes — only the JAR version changes.

**Approach:**

1. **Automate the PR creation** — bot creates PRs for all affected services in one run
2. **Auto-merge if CI passes** — no review required for pure dependency bump with no functional change
3. **Staggered deploy with ring approach** — don't push all 200 services simultaneously (would saturate deployment pipeline and suppress monitoring signal)
4. **Track progress centrally:**

```bash
# Check which services still have old version
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}: {.spec.containers[0].image}{"\n"}{end}' \
  | grep "log4j:2.14" | wc -l
# → 147 pods still on vulnerable version
```

---

## 10. Tricky Scenarios

### Tricky Scenario 1: Coupled Schema + Code Deploy

**Context:** New feature requires both a new column and new application code that writes to it. The schema migration and the code must deploy together, but Kubernetes rolling deploys mean old code runs alongside new code for several minutes.

**Why it fails if done naively:**
- Deploy schema first: old code ignores `discount_code` column — OK for reads, but old INSERT statements don't include it. If column is `NOT NULL` without a default, old INSERTs fail.
- Deploy code first: new code tries to write to `discount_code` column that doesn't exist → immediate error.

**The correct sequence:**

```
Step 1: Add column as nullable (no default required)
        ALTER TABLE orders ADD COLUMN discount_code VARCHAR(50);
        (Old code ignores it, new code writes it — both work)
        
Step 2: Deploy new code
        (During rolling deploy: old pods write NULL, new pods write discount_code)
        
Step 3: (Optional) Backfill + add NOT NULL if required
        (After all old pods are gone)
```

**Or using init container for migration:**

```yaml
initContainers:
- name: run-migration
  image: my-service:v2
  command: ["python", "manage.py", "migrate"]
  # Runs before the first v2 pod starts
  # Migration adds nullable column → compatible with running v1 pods
```

---

### Tricky Scenario 2: Thundering Herd on Cache Miss After Restart

**Context:** `product-catalog` service, 50 replicas, caches all products in memory (200K items). Pods are being replaced (rolling deploy). New pods start with empty cache. For 2-3 minutes, all 50 pods have empty caches and hit Postgres for every product lookup. **DB query rate spikes from 500/s to 50,000/s. DB crashes.**

**Solution 1: Cache pre-warming in readiness probe**

```python
@app.on_event("startup")
async def warm_cache():
    """Called on startup. Pod is not marked ready until this completes."""
    app.state.warmup_complete = False
    
    # Load hot items into cache from Redis (fast, < 5s) 
    # rather than from DB (slow)
    hot_product_ids = await redis.smembers("hot_products")
    
    for batch in chunks(hot_product_ids, 100):
        products = await redis.mget([f"product:{pid}" for pid in batch])
        for pid, data in zip(batch, products):
            if data:
                app.state.product_cache[pid] = json.loads(data)
    
    app.state.warmup_complete = True
    # /healthz/ready now returns 200 → pod enters load balancer rotation

@app.get("/healthz/ready")
async def readiness():
    if not app.state.warmup_complete:
        return Response(status_code=503, content='{"status": "warming"}')
    return {"status": "ok"}
```

**Solution 2: Probabilistic early expiry (prevent stampede on expiry)**

```python
import math, random

async def get_product(product_id: int) -> Product:
    raw = await redis.get(f"product:{product_id}")
    
    if raw:
        cached = json.loads(raw)
        ttl = await redis.ttl(f"product:{product_id}")
        
        # Probabilistic early recompute: as TTL gets low, 
        # some fraction of requests recompute early to avoid stampede
        # Formula: recompute with probability = exp(-ttl / beta)
        beta = 5.0  # tuning parameter
        if random.random() < math.exp(-ttl / beta):
            # One request refreshes the cache early — others still hit cache
            asyncio.create_task(refresh_product_cache(product_id))
        
        return Product(**cached)
    
    # Cache miss — fetch from DB with a distributed lock to prevent stampede
    lock_key = f"lock:product:{product_id}"
    async with redis_lock(lock_key, timeout=5):
        # Double-check after acquiring lock
        raw = await redis.get(f"product:{product_id}")
        if raw:
            return Product(**json.loads(raw))
        
        product = await db.fetchone("SELECT * FROM products WHERE id = %s", product_id)
        await redis.setex(f"product:{product_id}", 300, json.dumps(product.dict()))
        return product
```

---

### Tricky Scenario 3: Long-Running Job Migration

**Context:** `report-generator` service processes reports that take 10-60 minutes each. A rolling deploy kills a pod mid-job. The job must restart from scratch. Customer's 45-minute report fails.

**Solution 1: Checkpoint-based resumable jobs**

```python
async def generate_report(job_id: str):
    checkpoint = await load_checkpoint(job_id)
    start_page = checkpoint.get("last_page", 0)
    
    for page in range(start_page, total_pages):
        rows = await fetch_page(page)
        process_rows(rows)
        
        # Save checkpoint every 100 pages
        if page % 100 == 0:
            await save_checkpoint(job_id, {"last_page": page})
    
    await mark_complete(job_id)

async def save_checkpoint(job_id: str, state: dict):
    await redis.setex(f"checkpoint:{job_id}", 7200, json.dumps(state))
```

**Solution 2: Drain long-running pods last**

```bash
# Before deploy: cordon the node running long-running jobs
# (don't schedule new pods, but don't evict existing ones)
kubectl cordon node/worker-7

# Wait for in-flight jobs to complete (poll until idle)
while kubectl exec deployment/report-generator -- \
    python -c "import api; print(api.active_job_count())" | grep -v "^0$"; do
    echo "Waiting for jobs to complete..."
    sleep 60
done

# Now safe to drain
kubectl drain node/worker-7 --ignore-daemonsets
```

**Solution 3: Job lease with distributed lock**

```python
LEASE_TTL = 300  # 5 minutes

async def run_with_lease(job_id: str):
    lease_key = f"lease:{job_id}"
    worker_id = socket.gethostname()
    
    # Acquire lease
    acquired = await redis.set(lease_key, worker_id, nx=True, ex=LEASE_TTL)
    if not acquired:
        return  # Another worker has this job
    
    # Keep renewing lease while working
    async def renew_lease():
        while True:
            await asyncio.sleep(LEASE_TTL // 2)
            owner = await redis.get(lease_key)
            if owner == worker_id:
                await redis.expire(lease_key, LEASE_TTL)
            else:
                raise LeaseLost("Another worker took over the job")
    
    renewer = asyncio.create_task(renew_lease())
    try:
        await do_work(job_id)
    finally:
        renewer.cancel()
        # Only release if we still own it
        if await redis.get(lease_key) == worker_id:
            await redis.delete(lease_key)
```

---

## 11. Common Failure Modes

### Failure 1: Readiness Probe Not Implemented — Silent Deploy Failure

**What happens:** New pods start, pass liveness (HTTP 200 from a stub endpoint), enter the load balancer immediately, but are still connecting to DB and warming up. 30% of requests fail with connection errors for 3 minutes.

**Fix:** Implement `/healthz/ready` that gates on actual dependency health and warmup completion. `maxUnavailable: 0` is useless without a proper readiness probe.

---

### Failure 2: terminationGracePeriodSeconds Too Short

**What happens:** `terminationGracePeriodSeconds: 30`. ALB deregistration delay: 60 seconds. Pod is killed after 30 seconds while ALB is still routing traffic to it. In-flight requests get connection reset errors for the remaining 30 seconds.

**Fix:** Always set `terminationGracePeriodSeconds` = ALB deregistration delay + max request duration + 10-second buffer.

---

### Failure 3: Missing PodDisruptionBudget During Node Upgrade

**What happens:** Cluster auto-upgrade drains a node. The node has 3 of your 4 replicas (bad scheduling distribution). All 3 are evicted simultaneously. Service is running on 1 pod, which is hammered. It OOMs. Service is down.

**Fix:** Set PDB + topology spread constraints. Always spread replicas across nodes AND zones.

---

### Failure 4: Backwards-Incompatible API Change Without Versioning

**What happens:** v2 of `user-service` changes `/api/users/{id}` response shape (removes `phone` field, renames `email` to `contact_email`). `order-service` on v1 calls `user-service` and tries to access `response.email` → `AttributeError`. All order creation fails during the mixed-version window.

**Fix:** API changes must be backwards-compatible during any mixed-version window. Additive changes (new fields) are safe. Removal or rename requires API versioning (`/api/v2/users/{id}`) or a dual-response period.

---

### Failure 5: Deploy Triggered During Peak Traffic

**What happens:** Deploy started at 2pm on a Monday. Surge of new pods causes node autoscaler to provision. New pods take 4 minutes to become ready. During this window, the system is running at reduced capacity and traffic is at peak. Latency spikes. Alerts fire. On-call interrupted.

**Fix:** Define deploy windows (non-peak hours). Use a "deploy freeze" mechanism for peak periods (Black Friday, product launches). Automate deploy scheduling in CI/CD.

---

### Failure 6: Image Pull Failure Blocks Rollout

**What happens:** Deploy starts. New pods fail with `ImagePullBackOff` — the container registry is rate-limiting or the image tag doesn't exist (typo). Old pods are still running (good), but the rollout is stuck. Automated tools interpret "rollout stuck" as requiring manual intervention.

**Fix:**
- `imagePullPolicy: Always` + image digest (not tag) in production
- Pre-check image exists before triggering deploy
- Set registry pull secrets and increase rate limits

```yaml
containers:
- name: my-service
  # Use digest for immutable references
  image: 123456789.dkr.ecr.us-east-1.amazonaws.com/my-service@sha256:abc123...
  imagePullPolicy: IfNotPresent  # Use cached image if present — avoids registry calls on every start
```

---

### Failure 7: Config Map / Secret Not Updated Before Deployment

**What happens:** New version requires `NEW_FEATURE_API_KEY` environment variable. Config map was not updated. Pod starts, crashes on startup with `KeyError: NEW_FEATURE_API_KEY`. Pod restarts in a CrashLoopBackOff. Deploy stalls. Old pods keep running.

**Fix:** Always deploy config changes before or with the code that requires them. Use a startup probe that validates required env vars:

```python
# startup.py — validate required config at startup
REQUIRED_ENV = ["DATABASE_URL", "REDIS_URL", "NEW_FEATURE_API_KEY"]

for key in REQUIRED_ENV:
    if not os.environ.get(key):
        print(f"FATAL: Missing required environment variable: {key}")
        sys.exit(1)  # Crash fast — don't start the server with broken config
```

---

### Failure 8: Non-Idempotent Startup Code Runs Twice

**What happens:** Init container runs `flask db upgrade`. Runs fine. But due to a flaky health check, the init container is restarted and runs again. The second migration run fails with "column already exists" and crashes. Pod never starts.

**Fix:** Always write migrations to be idempotent (`IF NOT EXISTS`, `ON CONFLICT DO NOTHING`). Use a separate migration job (not init container) for non-idempotent operations.

---

### Failure 9: Canary Capturing the Wrong Metric

**What happens:** Canary at 5% shows 0% error rate. Promoted to 100%. Error rate jumps to 8%. Investigation reveals: the canary error metric was measuring `4xx` client errors, not `5xx` server errors. The new version was silently returning wrong data (200 OK with error payload).

**Fix:**
- Measure business metrics, not just HTTP status codes
- Log and alert on application-level error fields in response bodies
- Include a dark launch / shadow comparison for data correctness

---

### Failure 10: Rollback Doesn't Fully Roll Back

**What happens:** v2 deploys. It writes a new field `v2_metadata` to Postgres. Rollback to v1 triggers. v1 doesn't write `v2_metadata` but also doesn't fail — it just ignores the field. However, a downstream consumer was already updated to expect `v2_metadata`. That consumer is now broken.

**Fix:** Track all consumers of every schema change. Rollback plans must include consumer rollback, not just the producer. This is the core reason that "you can never fully roll back a database change" — you can always roll back code, but data written by v2 persists.

---

## 12. Reference Case Studies

1. **Netflix — Deploying Changes at Netflix Scale** — How Netflix deploys 100+ times per day using canary analysis with Kayenta, automated metric comparisons, and progressive traffic shifting. https://netflixtechblog.com/automated-canary-analysis-at-netflix-with-kayenta-3260bc7acc69

2. **Etsy — Continuous Deployment at Etsy** — The origin story of trunk-based development and continuous deployment in production, with feature flags as the safety mechanism. https://www.etsy.com/codeascraft/continuous-deployment-at-etsy/

3. **Slack — Deploys at Slack** — How Slack handles deploys across hundreds of services while avoiding user disruption, including their "lazy" migration approach for WebSocket sessions. https://slack.engineering/deploys-at-slack/

4. **Amazon — Minimizing Blast Radius with Deployment Rings** — Amazon's ring-based deployment model that starts in one AZ within one region before expanding globally. https://aws.amazon.com/builders-library/automating-safe-hands-off-deployments/

5. **Cloudflare — How Cloudflare Deploys** — Deploying to 200+ PoPs globally with a health-check-gated promotion system where each PoP must pass before the next group is promoted. https://blog.cloudflare.com/how-cloudflare-tests-software-for-more-than-10-million-requests-per-second/

6. **GitHub — Scientist Library** — GitHub's approach to dark launches: run old and new code in parallel, compare results, log discrepancies, promote when error rate is zero. https://github.blog/2016-02-03-scientist-github-s-experiment-library/

7. **Argo Rollouts Documentation** — Kubernetes progressive delivery controller with canary, blue-green, and analysis templates. https://argoproj.github.io/rollouts/

8. **LaunchDarkly — Feature Flag Best Practices** — Comprehensive guide to flag lifecycle management, gradual rollouts, and kill switches. https://launchdarkly.com/blog/feature-flag-best-practices/

9. **Google — Site Reliability Engineering, Chapter 8: Release Engineering** — SRE principles for safe releases including hermetic builds, progressive rollouts, and rollback procedures. https://sre.google/sre-book/release-engineering/

10. **AWS Builder's Library — Avoiding Fallback in Distributed Systems** — Why rollbacks are harder than they seem and how to design for forward-only deployments. https://aws.amazon.com/builders-library/avoiding-fallback-in-distributed-systems/
