# Multi-Task Async Processing

> **Goal:** Design systems that execute multiple independent or dependent tasks concurrently, reliably, and observably — without blocking callers, losing work, or creating unmanageable complexity.

---

## 1. Why Async Processing?

Synchronous processing means: caller waits until the task completes before getting a response. This is fine for fast operations (<100ms). It breaks down when:

- Tasks are **slow** (video encoding, ML inference, report generation)
- Tasks are **fan-out** (send 10M notifications after a post)
- Tasks have **retries** (third-party API calls that sometimes fail)
- Tasks can be **deferred** (send a daily digest email at 8am)
- Tasks have **dependencies** (step 2 needs output of step 1)
- You need **back-pressure** (don't overwhelm downstream services)

---

## 2. Core Concepts

### 2.1 Task Queue
A data structure where producers enqueue tasks and workers dequeue and execute them.  
- **FIFO:** First-in, first-out (simple, fair)
- **Priority Queue:** Higher-priority tasks jump the line (urgent notifications before bulk mail)
- **Delayed Queue:** Tasks scheduled for a future time (crons, retries with backoff)

### 2.2 Worker
A process or thread that picks tasks from the queue and executes them. Workers can be:
- **Single-threaded process:** One task at a time, simple to reason about
- **Thread pool:** N concurrent tasks per process
- **Async event loop:** Thousands of concurrent I/O-bound tasks (Node.js, Python asyncio)

### 2.3 At-Least-Once vs Exactly-Once Delivery
| Semantic | Guarantee | Risk | Use When |
|----------|-----------|------|----------|
| At-most-once | Delivered 0 or 1 times | Task may be lost | Log events, analytics (losing some is OK) |
| At-least-once | Delivered ≥ 1 times | Task may run twice | Most use cases (make tasks idempotent) |
| Exactly-once | Delivered exactly 1 time | Complex, expensive | Payments, ledger operations |

Most queues provide **at-least-once**. Design tasks to be **idempotent** (safe to run multiple times with the same result).

### 2.4 Dead Letter Queue (DLQ)
When a task fails after max retries, it's moved to a DLQ for inspection. Never silently discard failed tasks.

---

## 3. Async Task Patterns

### Pattern 1: Simple Task Queue (Single-step)

```
Producer ──► Queue ──► Worker Pool ──► Execute Task
```

**Implementations:** Celery + Redis/RabbitMQ, Sidekiq (Ruby), BullMQ (Node), Resque, pg-boss (Postgres-backed)

**Example: Send welcome email after signup**
```python
# Producer (API handler)
@app.route("/signup", methods=["POST"])
def signup():
    user = create_user(request.json)
    # Don't send email inline — queue it
    tasks.send_welcome_email.delay(user.id)
    return {"user_id": user.id}, 201

# Worker (runs in separate process)
@celery.task(bind=True, max_retries=3)
def send_welcome_email(self, user_id):
    try:
        user = User.get(user_id)
        email_service.send(user.email, template="welcome")
    except EmailServiceError as exc:
        raise self.retry(exc=exc, countdown=60)  # retry in 60s
```

---

### Pattern 2: Fan-out (One Task → Many Tasks)

When one event triggers many parallel sub-tasks.

```
Post Created Event
        │
        ▼
  Fan-out Worker
        │
   ┌────┼────┐
   ▼    ▼    ▼
Send  Update  Index
Notifs Feed  Search
(queue) (queue) (queue)
```

**Example: Order placed → parallel processing**
```python
@celery.task
def handle_order_placed(order_id):
    # Launch all sub-tasks in parallel (Celery group)
    tasks = group(
        charge_payment.s(order_id),
        send_confirmation_email.s(order_id),
        update_inventory.s(order_id),
        notify_warehouse.s(order_id),
    )
    tasks.apply_async()
```

**Key consideration:** If any sub-task fails, others may have already succeeded. Each must be idempotent and you need a way to detect partial failure.

---

### Pattern 3: Pipeline / Chain (Sequential Steps)

Step 2 depends on Step 1's output. A failure at any step aborts the pipeline.

```
Step 1 → Step 2 → Step 3 → Step 4
```

**Example: Video processing pipeline**
```python
# Celery chain
pipeline = chain(
    download_video.s(video_id),       # Step 1: download from S3
    transcode_to_720p.s(),             # Step 2: transcode (uses Step 1 output)
    generate_thumbnail.s(),            # Step 3: thumbnail (uses video)
    update_video_status.s("ready"),    # Step 4: mark ready
)
pipeline.apply_async()
```

```
Download ──► Transcode ──► Thumbnail ──► Update DB
  |              |             |             |
 fail          fail           fail          fail
  └──────────────────────────────────────────┘
                  → DLQ + alert
```

---

### Pattern 4: DAG (Directed Acyclic Graph) — Complex Dependencies

Some steps depend on multiple upstream steps. Apache Airflow, Temporal, Prefect, and AWS Step Functions handle this.

```
       ┌── Fetch User Data ──┐
       │                     ▼
Start ─┤                  Merge ──► Personalize ──► Send
       │                     ▲
       └── Fetch Product ────┘
```

**Example: Data pipeline for personalized report**
```python
# Prefect DAG
@flow
def generate_report(user_id):
    user_data = fetch_user_data(user_id)       # parallel start
    product_data = fetch_product_data(user_id) # parallel start
    merged = merge_data(user_data, product_data)  # waits for both
    report = personalize_report(merged)
    send_report(user_id, report)
```

**Tools:**
- **Apache Airflow:** Python DAGs, scheduling, monitoring dashboard, widely used for data pipelines
- **Temporal:** Workflow engine for microservices; durable execution, handles failures, retries, signals
- **AWS Step Functions:** Managed, visual workflow designer, integrates with all AWS services
- **Prefect / Dagster:** Modern Airflow alternatives with better developer experience

---

### Pattern 5: Saga Pattern (Distributed Transactions)

When a multi-step transaction spans multiple services/databases, and you need rollback on failure.

**Choreography Saga (event-driven):**
```
Order Service        Payment Service       Inventory Service
     │                    │                      │
     │─── OrderCreated ──►│                      │
     │                    │─── PaymentCharged ──►│
     │                    │                      │─── InventoryReserved
     │◄── OrderConfirmed ─┘                      │
     
On failure:
Inventory fails ──► InventoryFailed event
                        ──► PaymentService reverses charge
                        ──► OrderService marks order failed
```

**Orchestration Saga (central coordinator):**
```python
# Temporal workflow (orchestrator)
@workflow.defn
class PlaceOrderWorkflow:
    @workflow.run
    async def run(self, order_id):
        # Step 1: Reserve inventory
        try:
            await workflow.execute_activity(reserve_inventory, order_id)
        except InventoryError:
            await workflow.execute_activity(cancel_order, order_id)
            return

        # Step 2: Charge payment
        try:
            await workflow.execute_activity(charge_payment, order_id)
        except PaymentError:
            await workflow.execute_activity(release_inventory, order_id)  # compensate
            await workflow.execute_activity(cancel_order, order_id)
            return

        # Step 3: Notify
        await workflow.execute_activity(send_confirmation, order_id)
```

---

### Pattern 6: Priority Queues

Different task types need different urgency levels.

```python
# Celery priority queues
app.conf.task_queues = (
    Queue("critical", routing_key="critical"),  # Fraud alerts, payments
    Queue("high",     routing_key="high"),       # Notifications, emails
    Queue("default",  routing_key="default"),    # Standard tasks
    Queue("low",      routing_key="low"),        # Analytics, reporting
    Queue("bulk",     routing_key="bulk"),       # Batch jobs
)

# Worker processes: run more workers on critical queue
# celery worker -Q critical --concurrency=16
# celery worker -Q high,default --concurrency=8
# celery worker -Q low,bulk --concurrency=4
```

---

### Pattern 7: Rate Limiting & Back-pressure

Prevent async tasks from overwhelming downstream services.

```python
@celery.task(rate_limit="100/m")  # Max 100 tasks/minute from this worker
def send_sms(phone, message):
    sms_gateway.send(phone, message)

# Or use a token bucket in Redis:
def acquire_rate_limit_token(service_name, max_per_second):
    key = f"rate_limit:{service_name}"
    current = redis.incr(key)
    if current == 1:
        redis.expire(key, 1)  # Reset each second
    if current > max_per_second:
        raise RateLimitExceeded()
```

---

## 4. Regular Scenarios

### Scenario 1: E-commerce Order Processing
**Requirement:** When an order is placed, charge payment, update inventory, send confirmation email, notify warehouse.

**Architecture:**
```
POST /orders
     │
     ▼ (synchronous)
Create order record (DB, status="pending")
     │
     ▼ (async fan-out)
Kafka topic: order.created
     │
     ├──► Payment Service consumer → charge card → emit payment.charged
     ├──► Inventory Service consumer → reserve items → emit inventory.reserved
     └──► Notification Service consumer → send email

Kafka topic: payment.charged + inventory.reserved
     │
     ▼ (join — both must succeed)
Order Service → update status = "confirmed"
```

**Idempotency:** Each service stores `processed_event_ids`. If an event is re-delivered (Kafka at-least-once), duplicate processing is a no-op.

---

### Scenario 2: Image Processing Platform (Bulk Upload)
**Requirement:** User uploads 500 photos. For each: resize to 3 sizes, generate thumbnail, run NSFW classifier, store in CDN.

```python
@celery.task
def process_uploaded_image(image_id):
    image = storage.download(image_id)
    
    # Parallel sub-tasks
    job = group(
        resize_image.s(image_id, "1080p"),
        resize_image.s(image_id, "720p"),
        resize_image.s(image_id, "thumbnail"),
        run_nsfw_classifier.s(image_id),
    )
    # Then update status after all complete
    chord(job)(update_image_status.s(image_id))

@celery.task
def process_bulk_upload(user_id, image_ids):
    # Don't process all 500 at once — chunk into batches
    for batch in chunks(image_ids, size=10):
        group(process_uploaded_image.s(img_id) for img_id in batch).apply_async()
```

**Back-pressure:** Chunk into batches of 10 to avoid flooding the image processing queue.

---

### Scenario 3: Scheduled Reports
**Requirement:** Every Monday 8am, generate and email analytics reports to 10k users.

```python
# Airflow DAG
@dag(schedule="0 8 * * 1")  # Every Monday 8am
def weekly_reports_dag():
    
    @task
    def get_report_subscribers():
        return db.query("SELECT user_id FROM report_subscriptions WHERE active=true")
    
    @task
    def generate_and_send_report(user_id):
        report = analytics.generate(user_id)
        email.send(user_id, report)
    
    users = get_report_subscribers()
    # Dynamic task mapping: one task instance per user
    generate_and_send_report.expand(user_id=users)
```

---

## 5. Tricky Scenarios

### Tricky Scenario 1: Exactly-Once Payment Processing
**Problem:** Network blip causes the "payment charged" message to re-deliver. Customer gets charged twice.

**Solution: Idempotency key + conditional processing**
```python
@celery.task
def charge_payment(order_id):
    idempotency_key = f"payment:order:{order_id}"
    
    # Check if already processed
    if redis.get(idempotency_key):
        logger.info(f"Payment for order {order_id} already processed, skipping")
        return
    
    # Charge the card
    result = stripe.charge(order_id)
    
    # Mark as processed (with TTL to handle rare edge cases)
    redis.setex(idempotency_key, 86400, result.charge_id)
    
    db.update_order_status(order_id, "paid", result.charge_id)
```

**Stripe's approach:** Client sends `Idempotency-Key` header. Stripe stores the result for 24h and returns the same response on duplicate requests.

---

### Tricky Scenario 2: Long-Running Task Crashes Mid-Way
**Problem:** A video encoding job runs for 2 hours. Worker crashes at 1h50m. Job is re-queued and starts from scratch.

**Solution: Checkpointing**
```python
@celery.task
def encode_video(video_id):
    checkpoints = redis.hgetall(f"encode_checkpoint:{video_id}")
    
    # Resume from last checkpoint
    start_segment = int(checkpoints.get("last_segment", 0))
    total_segments = get_total_segments(video_id)
    
    for segment in range(start_segment, total_segments):
        encode_segment(video_id, segment)
        # Save progress every segment
        redis.hset(f"encode_checkpoint:{video_id}", "last_segment", segment)
    
    # Stitch segments
    stitch_video(video_id)
    redis.delete(f"encode_checkpoint:{video_id}")  # cleanup
```

**Alternative:** Use **Temporal** workflows. Temporal persists workflow state durably. On worker crash, execution resumes exactly where it left off — no custom checkpointing needed.

---

### Tricky Scenario 3: Fan-out at Scale with Partial Failure
**Problem:** Sending 1M notifications. 100k fail due to FCM quota. How to retry only the failed ones?

**Solution: Granular tracking**
```python
@celery.task
def fanout_notification(post_id, follower_ids_batch):
    failed = []
    for follower_id in follower_ids_batch:
        try:
            push_notification(follower_id, post_id)
        except PushError:
            failed.append(follower_id)
    
    if failed:
        # Retry failed subset with exponential backoff
        fanout_notification.apply_async(
            args=[post_id, failed],
            countdown=60  # retry in 60s
        )
```

**Alternative:** Use Kafka with per-partition offsets. Only commit offset after successful processing. On failure, Kafka re-delivers from last committed offset — automatic partial retry.

---

### Tricky Scenario 4: Task Priority Inversion
**Problem:** Low-priority bulk email job floods the queue. High-priority fraud alert can't get a worker.

**Solution:**
1. **Separate queues by priority** — workers for critical queue run always; bulk workers scale to zero when critical queue has work
2. **Work-stealing:** Bulk workers check critical queue first before pulling bulk tasks
3. **Admission control:** When queue depth exceeds threshold, reject new low-priority tasks

```python
def celery_worker_loop():
    # Always check critical queue first
    task = critical_queue.pop() or high_queue.pop() or default_queue.pop() or bulk_queue.pop()
    if task:
        execute(task)
```

---

### Tricky Scenario 5: Poison Pill Task (Infinite Retry Loop)
**Problem:** A task fails every time (bug in code, bad data). With max_retries=None or a very high number, it floods the queue and clogs workers.

**Solution: Exponential backoff + DLQ + circuit breaker**
```python
@celery.task(
    bind=True,
    max_retries=5,
    default_retry_delay=60,  # seconds
)
def process_webhook(self, payload):
    try:
        handle(payload)
    except NonRetryableError:
        # Don't retry bad data errors — send to DLQ immediately
        dlq.send(payload, reason=str(exc))
        return
    except RetryableError as exc:
        # Exponential backoff: 60s, 120s, 240s, 480s, 960s
        raise self.retry(exc=exc, countdown=60 * (2 ** self.request.retries))
    
    # After 5 retries, task goes to DLQ automatically
```

**Circuit Breaker for downstream:**
```python
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=60)
def call_payment_api(payload):
    return requests.post(PAYMENT_API_URL, json=payload, timeout=5)
```

If 5 consecutive calls fail, circuit opens for 60s — no more tasks sent to broken downstream.

---

### Tricky Scenario 6: Distributed Cron — Avoid Duplicate Execution
**Problem:** You have 10 web servers, each running a cron job at midnight. The job runs 10 times.

**Solutions:**
1. **Designated scheduler:** Only one service is responsible for scheduling (Airflow, Kubernetes CronJob)
2. **Distributed lock:**
```python
def run_nightly_report():
    lock = redis.lock("cron:nightly_report", timeout=300)  # 5 min timeout
    if lock.acquire(blocking=False):  # Don't wait — if someone else has it, skip
        try:
            generate_nightly_report()
        finally:
            lock.release()
    # else: another instance already running this cron, skip
```
3. **Database-backed scheduling:** Store next_run_at in DB. Use `SELECT FOR UPDATE SKIP LOCKED` to claim the job.

---

## 6. Choosing the Right Tool

| Need | Tool |
|------|------|
| Simple background tasks, Python | Celery + Redis |
| Job queues with Postgres | pg-boss, Faktory |
| Ruby job queues | Sidekiq |
| Node.js job queues | BullMQ, Bee-Queue |
| Complex workflows / durable execution | Temporal, Conductor |
| Data pipelines with scheduling | Apache Airflow, Prefect, Dagster |
| Stream processing | Kafka + Flink / Spark Streaming |
| Serverless async | AWS SQS + Lambda, GCP Pub/Sub + Cloud Functions |
| Visual orchestration (AWS-native) | AWS Step Functions |
| High-throughput messaging | Apache Kafka, Pulsar, RabbitMQ |

---

## 7. Observability Checklist

Every async processing system must have:

- [ ] **Queue depth metric** — alert when queue grows unboundedly
- [ ] **Task processing time** — p50, p95, p99 latency
- [ ] **Task failure rate** — alert when >1% of tasks fail
- [ ] **DLQ depth** — alert when DLQ grows (means persistent failures)
- [ ] **Worker saturation** — alert when all workers are busy (need to scale out)
- [ ] **Retry rate** — spikes indicate downstream issues
- [ ] **End-to-end latency** — time from task enqueued to task completed
- [ ] **Distributed tracing** — correlate async tasks back to the original request

---

## 8. Reference Case Studies

1. **Robinhood: Task Processing at Scale with Celery and Redis**  
   [https://robinhood.engineering/robinhood-uses-python-for-everything-db91e95a8e1b](https://robinhood.engineering/robinhood-uses-python-for-everything-db91e95a8e1b)

2. **Temporal: Coinbase's Durable Workflows**  
   [https://temporal.io/case-studies/coinbase](https://temporal.io/case-studies/coinbase)

3. **Airbnb's Airflow at Scale**  
   [https://medium.com/airbnb-engineering/airflow-a-workflow-management-platform-46318b977fd8](https://medium.com/airbnb-engineering/airflow-a-workflow-management-platform-46318b977fd8)

4. **Slack: Scaling Background Tasks**  
   [https://slack.engineering/executing-tasks-at-scale-1000-celery-workers/](https://slack.engineering/executing-tasks-at-scale-1000-celery-workers/)

5. **Stripe: Designing Robust and Predictable APIs with Idempotency**  
   [https://stripe.com/blog/idempotency](https://stripe.com/blog/idempotency)

6. **Netflix: Workflow Orchestration**  
   [https://netflixtechblog.com/netflix-conductor-a-microservices-orchestrator-2e8d4771bf40](https://netflixtechblog.com/netflix-conductor-a-microservices-orchestrator-2e8d4771bf40)

7. **Uber: Cadence (predecessor to Temporal)**  
   [https://eng.uber.com/cadence-workflow-orchestration/](https://eng.uber.com/cadence-workflow-orchestration/)
