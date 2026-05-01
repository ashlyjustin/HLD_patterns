# Handling Transactions

> **Goal:** Understand how database transactions work, when they break, and how to design systems that remain correct under concurrency, failures, and distributed environments.

---

## 1. What Is a Transaction?

A transaction is a unit of work that must be treated as atomic — either all operations succeed together, or none of them do.

**ACID Properties:**

| Property | Meaning | Example |
|----------|---------|---------|
| **Atomicity** | All-or-nothing: either all operations commit, or all roll back | Transfer $100: debit AND credit, never just one |
| **Consistency** | DB moves from one valid state to another; constraints are maintained | Account balance cannot go negative (if you enforce it) |
| **Isolation** | Concurrent transactions don't see each other's partial state | Two users booking the last seat see a consistent view |
| **Durability** | Committed data survives crashes | Committed payment isn't lost on DB restart |

These four properties are the contract that ACID databases give you. Understanding *exactly* what each isolation level guarantees (and doesn't) is critical to writing correct concurrent code.

---

## 2. Isolation Levels (The Most Misunderstood Part)

SQL standard defines four isolation levels. Each prevents certain anomalies:

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Write Skew |
|----------------|-----------|-------------------|--------------|------------|
| READ UNCOMMITTED | ✅ Possible | ✅ Possible | ✅ Possible | ✅ Possible |
| READ COMMITTED | ❌ Prevented | ✅ Possible | ✅ Possible | ✅ Possible |
| REPEATABLE READ | ❌ Prevented | ❌ Prevented | ✅ Possible* | ✅ Possible |
| SERIALIZABLE | ❌ Prevented | ❌ Prevented | ❌ Prevented | ❌ Prevented |

*MySQL's REPEATABLE READ also prevents phantom reads via next-key locking. Postgres's does not.

### Anomaly Definitions

**Dirty Read:** Transaction A reads uncommitted data from Transaction B. B later rolls back → A read data that never existed.
```
T1: UPDATE accounts SET balance = 1000 WHERE id = 1;  -- not committed
T2: SELECT balance FROM accounts WHERE id = 1;  -- reads 1000 ← dirty!
T1: ROLLBACK;  -- balance never was 1000
```

**Non-Repeatable Read:** Transaction A reads a row twice; T2 updates it in between → A gets different values.
```
T1: SELECT balance FROM accounts WHERE id = 1;  -- reads 500
T2: UPDATE accounts SET balance = 1000 WHERE id = 1; COMMIT;
T1: SELECT balance FROM accounts WHERE id = 1;  -- reads 1000 ← different!
```

**Phantom Read:** Transaction A runs a range query twice; T2 inserts a new row matching the range → A sees different rows.
```
T1: SELECT * FROM orders WHERE amount > 500;  -- sees 5 rows
T2: INSERT INTO orders (amount) VALUES (600); COMMIT;
T1: SELECT * FROM orders WHERE amount > 500;  -- sees 6 rows ← phantom!
```

**Write Skew:** Two transactions each read a shared condition, both find it valid, both write — combined result violates the invariant.
```
Invariant: at least one doctor must be on call at all times.

T1 (Doctor Alice): SELECT COUNT(*) FROM on_call WHERE shift = 'night'; -- sees 2
T2 (Doctor Bob):   SELECT COUNT(*) FROM on_call WHERE shift = 'night'; -- sees 2
T1: DELETE FROM on_call WHERE doctor = 'alice' AND shift = 'night';
T2: DELETE FROM on_call WHERE doctor = 'bob' AND shift = 'night';
-- Both committed: 0 doctors on call! ← Write skew
```

---

## 3. Default Isolation in Popular Databases

| Database | Default Isolation | Notes |
|----------|------------------|-------|
| PostgreSQL | READ COMMITTED | Use SERIALIZABLE or explicit locks for write skew |
| MySQL InnoDB | REPEATABLE READ | Next-key locking prevents phantoms |
| SQL Server | READ COMMITTED | READ COMMITTED SNAPSHOT ISOLATION (RCSI) available |
| Oracle | READ COMMITTED | Uses MVCC — no dirty reads by default |
| SQLite | SERIALIZABLE | Single-writer model; effectively serialized |
| CockroachDB | SERIALIZABLE | Default, strongest guarantee |

**Key insight:** Postgres READ COMMITTED (the default) does NOT prevent non-repeatable reads or write skew. Most applications run at READ COMMITTED and are unknowingly vulnerable to write skew bugs.

---

## 4. MVCC (Multi-Version Concurrency Control)

Most modern databases use MVCC to implement isolation without reader-writer blocking.

**How it works:**
- Every row has a transaction ID (xmin) for when it was created and (xmax) for when it was deleted/updated
- Readers see a **snapshot** of the database as of their transaction start time
- Readers never block writers; writers never block readers

```
Timeline:
  T100 writes row: balance=500 (xmin=100)
  T101 starts (sees snapshot at T101)
  T102 writes row: balance=1000 (xmin=102, old row xmax=102)
  T101 reads row: sees balance=500 (xmin=100 is visible to T101)
  T101 commits
  T103 starts (sees snapshot at T103): sees balance=1000
```

**MVCC garbage collection:** Old row versions accumulate. Postgres `VACUUM` cleans dead tuples. Poor VACUUM hygiene → table bloat → slow queries.

---

## 5. Transaction Design Patterns

### Pattern 1: The Unit of Work

Group all related DB operations for a single business action into one transaction. Either all succeed or all roll back.

```python
def transfer_money(from_account, to_account, amount):
    with db.transaction():
        # Check balance
        balance = db.scalar(
            "SELECT balance FROM accounts WHERE id = %s FOR UPDATE",
            [from_account]
        )
        if balance < amount:
            raise InsufficientFundsError()
        
        # Debit
        db.execute(
            "UPDATE accounts SET balance = balance - %s WHERE id = %s",
            [amount, from_account]
        )
        
        # Credit
        db.execute(
            "UPDATE accounts SET balance = balance + %s WHERE id = %s",
            [amount, to_account]
        )
        
        # Audit log
        db.execute(
            "INSERT INTO ledger (from_acct, to_acct, amount) VALUES (%s, %s, %s)",
            [from_account, to_account, amount]
        )
    # Transaction commits here
```

---

### Pattern 2: Savepoints (Partial Rollback)

Allows rolling back part of a transaction without aborting the whole thing.

```sql
BEGIN;
INSERT INTO orders (user_id, total) VALUES (42, 100);
SAVEPOINT after_order;

INSERT INTO payments (order_id, amount) VALUES (99, 100);
-- Something fails
ROLLBACK TO SAVEPOINT after_order;  -- Only undoes the payment insert

-- Try alternative payment method
INSERT INTO payments (order_id, amount, method) VALUES (99, 100, 'credit');
COMMIT;  -- Order + credit payment committed
```

**Use case:** Complex workflows where one optional step fails but you want to preserve the rest.

---

### Pattern 3: Advisory Locks (Application-Level Locks)

Postgres provides advisory locks — application-defined locks that don't lock any actual table row. Useful for preventing concurrent execution of a specific operation.

```sql
-- Acquire lock for user ID 42 (non-blocking)
SELECT pg_try_advisory_lock(42);  -- returns true if acquired, false if already held

-- Use case: prevent duplicate job execution
BEGIN;
SELECT pg_advisory_xact_lock(job_id);  -- auto-released at transaction end
-- Only one worker can hold this lock; others wait
INSERT INTO job_runs ...;
COMMIT;
```

**Use case at GitLab:** Advisory locks prevent concurrent migrations on the same table.

> 📖 **Reference:** [PostgreSQL Advisory Locks](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS)

---

### Pattern 4: Multi-Step Transactions — Designing for Correctness at Every Step

Before reaching for distributed protocols, understand what multi-step transactions look like within a single database boundary, and how to model the state machine that drives them.

#### The State Machine Model

Every multi-step transaction is a state machine. Each step moves the entity forward; any step can fail and must have a defined rollback or recovery path.

```
Order State Machine:

  CREATED ──► INVENTORY_RESERVED ──► PAYMENT_CHARGED ──► SHIPPED ──► COMPLETE
      │               │                     │                │
      ▼               ▼                     ▼                ▼
  CANCELLED    INVENTORY_FAILED      PAYMENT_FAILED    SHIP_FAILED
                     (compensate:           (compensate:
                   nothing to undo)    release inventory)
```

**Key rule:** The state must be persisted durably before starting the next step. Never hold the state only in memory across steps — a crash will leave you with no idea where you are.

```python
# Store state in DB at every transition
def process_order(order_id):
    order = db.get(order_id)  # status = "created"

    # Step 1
    if order.status == "created":
        result = inventory_service.reserve(order_id)
        if result.ok:
            db.update_order_status(order_id, "inventory_reserved")  # ← persist before continuing
        else:
            db.update_order_status(order_id, "inventory_failed")
            return

    # Step 2 — re-read status so crash recovery can resume from here
    order = db.get(order_id)
    if order.status == "inventory_reserved":
        result = payment_service.charge(order_id)
        if result.ok:
            db.update_order_status(order_id, "payment_charged")
        else:
            inventory_service.release(order_id)  # compensate
            db.update_order_status(order_id, "payment_failed")
            return

    # Step 3
    order = db.get(order_id)
    if order.status == "payment_charged":
        shipping_service.ship(order_id)
        db.update_order_status(order_id, "complete")
```

On any crash and restart, a background job scans for orders stuck in intermediate states and resumes from the last persisted step. This is the **process manager** pattern.

#### Try-Confirm-Cancel (TCC)

TCC is a pattern for multi-step transactions where you want each resource to be reserved tentatively before committing all at once.

```
Phase 1 — Try (reserve, don't commit):
  PaymentService.try_charge(order_id, amount)    → creates a hold on the card
  InventoryService.try_reserve(order_id, sku)    → marks stock as tentatively held
  ShippingService.try_book(order_id, address)    → tentatively books a shipment slot

  All three succeed? → Phase 2
  Any fail?          → Phase 3

Phase 2 — Confirm (finalize all holds):
  PaymentService.confirm(order_id)     → capture the charge
  InventoryService.confirm(order_id)   → decrement stock
  ShippingService.confirm(order_id)    → dispatch

Phase 3 — Cancel (release all holds):
  PaymentService.cancel(order_id)      → void the hold
  InventoryService.cancel(order_id)    → release tentative reservation
  ShippingService.cancel(order_id)     → free the slot
```

**Key property of TCC:** Each participant's `try` operation is idempotent and time-limited (the hold expires automatically after e.g. 10 minutes if `confirm` never arrives). This avoids the infinite lock-holding problem of 2PC.

```python
class TccCoordinator:
    def execute(self, participants, payload):
        # Phase 1: Try
        tried = []
        for p in participants:
            result = p.try_operation(payload)
            if result.failed:
                # Cancel all successfully tried participants
                for already_tried in tried:
                    already_tried.cancel(payload)
                raise TransactionFailed(f"Try failed at {p}")
            tried.append(p)

        # Phase 2: Confirm all (best-effort; retried until success)
        for p in tried:
            retry_until_success(p.confirm, payload)
```

The `confirm` step is retried indefinitely because all participants already agreed to commit in Phase 1 — eventual confirmation is guaranteed. This is fundamentally different from 2PC, where the coordinator could crash between prepare and commit.

---

### Pattern 5: Two-Phase Commit (2PC) — The Atomic Commitment Protocol

2PC is the classical distributed transaction protocol. It coordinates multiple independent resource managers to commit or abort atomically.

#### The Full Protocol

```
                    Coordinator
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
      Participant A  Participant B  Participant C
      (DB shard 1)   (DB shard 2)   (DB shard 3)
```

**Phase 1 — Voting (Prepare):**
```
1. Coordinator writes "BEGIN PREPARE" to its own WAL (durable)
2. Coordinator sends PREPARE to all participants

Each participant:
  a. Executes the transaction locally (but does not commit)
  b. Writes a PREPARE record to its own WAL → now it CAN commit even after a crash
  c. Acquires all locks needed for commit
  d. Responds VOTE-COMMIT or VOTE-ABORT to coordinator
```

**Phase 2 — Decision (Commit or Abort):**
```
If ALL participants voted COMMIT:
  a. Coordinator writes "COMMIT DECISION" to its own WAL (durable — point of no return)
  b. Coordinator sends COMMIT to all participants
  c. Each participant commits, releases locks, acks coordinator
  d. Coordinator writes "COMPLETE" to WAL

If ANY participant voted ABORT:
  a. Coordinator writes "ABORT DECISION" to WAL
  b. Coordinator sends ABORT to all participants
  c. Each participant rolls back, releases locks
```

**The Point of No Return:** The moment the coordinator writes the commit decision to its WAL is when the transaction becomes permanent — even if the coordinator crashes immediately after, it will re-send COMMIT on recovery because the decision is in its log.

#### Failure Scenarios (This Is Where 2PC Gets Hard)

**Scenario A: Participant crashes before sending VOTE**
```
Coordinator: waiting for vote from Participant B (timeout)
→ Coordinator treats timeout as VOTE-ABORT
→ Sends ABORT to all
→ Safe: B never prepared, so it has nothing to roll back
```

**Scenario B: Participant crashes after sending VOTE-COMMIT but before receiving COMMIT**
```
Coordinator: receives all votes, writes COMMIT to log, sends COMMIT
Participant B: crashes in PREPARED state
→ B's WAL contains a PREPARE record
→ On B's restart: "I have a prepared transaction, I do not know the coordinator's decision"
→ B queries coordinator: "What was the decision for transaction X?"
→ Coordinator responds COMMIT → B commits
→ OR coordinator is also down → B is BLOCKED (in-doubt transaction)
```

**Scenario C: Coordinator crashes after writing COMMIT to log but before sending to participants**
```
Participants: in PREPARED state, holding locks, waiting
Coordinator: down
→ All participants are BLOCKED — they cannot commit or abort without the coordinator
→ Locks are held for the duration of the coordinator outage
→ This is the fundamental blocking nature of 2PC
```

**Scenario D: Network partition between coordinator and one participant**
```
Coordinator sends COMMIT to A and B, then network to C is cut
A commits, B commits
C: still in PREPARED state, locks held, unable to proceed
→ When partition heals, coordinator retries COMMIT to C → C commits
→ Window of inconsistency: A and B see committed data, C does not
```

#### In-Doubt Transactions — The Most Dangerous State

An in-doubt transaction is one where a participant has prepared but cannot determine the coordinator's decision. This is the most operationally dangerous state in distributed systems.

```sql
-- Postgres: see in-doubt prepared transactions
SELECT gid, prepared, owner, database FROM pg_prepared_xacts;
-- gid = global transaction ID
-- prepared = when it was prepared (could be hours ago — this is a problem)

-- Manually commit an in-doubt transaction (emergency only, after confirming coordinator decision)
COMMIT PREPARED 'transaction-xyz-123';

-- Manually abort an in-doubt transaction
ROLLBACK PREPARED 'transaction-xyz-123';
```

**Operational alert:** Set `max_prepared_transactions > 0` in Postgres to allow prepared transactions. Monitor `pg_prepared_xacts` — any transaction prepared for more than a few minutes is a red flag.

#### 2PC Performance Profile

```
Network round trips required: 2
  Phase 1: Coordinator → Participants (send PREPARE) + wait for all votes
  Phase 2: Coordinator → Participants (send COMMIT/ABORT)

For cross-region 2PC (US-East ↔ EU-West, ~80ms RTT):
  Total latency: 2 × 80ms = 160ms minimum for the commit
  + local transaction processing time

Throughput ceiling:
  Each transaction holds locks across the entire Phase 1 → Phase 2 window
  High latency = locks held longer = more contention = lower throughput
  This is why 2PC is avoided for cross-region high-throughput workloads
```

#### When 2PC Is Appropriate vs. When to Avoid It

| Situation | Use 2PC? | Reason |
|-----------|----------|--------|
| Two databases in the same datacenter, must be atomic | ✅ Yes | Low RTT, short lock window |
| Updating a database + sending a message to Kafka | ❌ No | Use Outbox pattern instead |
| Cross-region distributed transaction | ❌ No | High latency, long lock hold |
| Enrolling XA resources (JTA, Java EE) | ✅ Yes | XA is standardized 2PC |
| Micro-services across teams | ❌ No | Use Saga; 2PC creates tight coupling |
| Single DB, atomic write to two tables | ❌ Unnecessary | Normal ACID transaction handles this |

---

### Pattern 5b: Three-Phase Commit (3PC) — The Non-Blocking Alternative

3PC was designed specifically to fix 2PC's blocking failure mode. It adds a third phase — a pre-commit — to eliminate the situation where participants are stuck not knowing the coordinator's decision.

#### The Three Phases

```
Phase 1 — CAN-COMMIT (same as 2PC's Prepare, but no locking yet):
  Coordinator → all: "Can you commit?"
  Participants: respond YES/NO (but do NOT acquire locks yet)

Phase 2 — PRE-COMMIT (the new phase):
  If all said YES:
    Coordinator → all: "Pre-commit" (write to WAL, acquire locks)
    Participants: acquire locks, write PRE-COMMIT to WAL, ack
  If any said NO:
    Coordinator → all: "Abort"

Phase 3 — COMMIT (same as 2PC's Phase 2):
  Coordinator → all: "Commit"
  Participants: commit, release locks
```

#### Why Pre-Commit Eliminates the Blocking Problem

The key insight: **if a participant has received a PRE-COMMIT, it knows every other participant also voted YES in Phase 1.** Therefore, if the coordinator crashes after PRE-COMMIT and before COMMIT, a participant can contact any other participant:
- If any participant did NOT receive PRE-COMMIT → the coordinator hadn't decided yet → safe to abort
- If all participants received PRE-COMMIT → the coordinator had decided to commit → safe to commit

```
2PC — Participant in PREPARED state when coordinator crashes:
  "I don't know if others prepared. I cannot safely commit OR abort without coordinator."
  → BLOCKED

3PC — Participant in PRE-COMMITTED state when coordinator crashes:
  "I know all others received PRE-COMMIT (because coordinator sent it to all before crashing)."
  "I can elect a new coordinator from among the participants and commit safely."
  → NOT BLOCKED
```

#### Why 3PC Is Rarely Used in Practice

Despite solving 2PC's blocking problem, 3PC has a fatal flaw: **it assumes no network partitions.** Under a network partition, 3PC can make two separated groups of participants reach different decisions — one half commits, the other aborts.

```
Network partition during 3PC:
  Group 1 (Coordinator + A + B): sees PRE-COMMIT → elects new coordinator → COMMITS
  Group 2 (C + D): never received PRE-COMMIT → times out → ABORTS

Result: A, B committed; C, D aborted. Database is split-brained.
```

2PC is **safe under partitions** (it blocks, but never makes wrong decisions).  
3PC is **live under crashes** (no blocking) but **unsafe under partitions** (split brain).

In real-world distributed systems, network partitions happen. This is why 3PC is mostly a theoretical construct and almost no production database uses it. Instead, the industry moved to:
- **Paxos / Raft** — consensus protocols that are both safe and live (up to a majority of nodes)
- **Saga pattern** — avoid distributed atomicity entirely
- **Spanner's TrueTime + Paxos** — strong consistency via clock-bounded commit wait

#### 3PC vs 2PC vs Paxos

```
                    2PC         3PC         Paxos/Raft
─────────────────────────────────────────────────────
Network rounds        2           3           2-3
Blocking on crash   YES          NO          NO
Safe under partition YES         NO          YES
Used in production  YES (XA)    Rarely       YES (Spanner, etcd, CockroachDB)
Coordinator role    Required    Required     Rotatable leader
Lock duration       Phase 1→2   Phase 2→3   Short (per round)
```

**Bottom line:** If you need non-blocking distributed atomicity that also handles network partitions, use a Raft-based database (CockroachDB, etcd, Spanner) rather than implementing 3PC yourself.

---

### Pattern 6: Saga Pattern — Distributed Transactions Without Atomicity

The Saga pattern accepts that you cannot have ACID atomicity across service boundaries. Instead, it breaks the transaction into a sequence of local transactions — each durable on its own — linked by events and compensating transactions.

#### What a Saga Gives You (and What It Doesn't)

Sagas provide **ACD** — but NOT full ACID:

| Property | In Saga | Notes |
|----------|---------|-------|
| **Atomicity** | Eventual | Either all steps complete, or compensations bring system back to consistent state — but there's a window of inconsistency |
| **Consistency** | Eventual | Invariants may be temporarily violated between steps |
| **Isolation** | ❌ None | Concurrent sagas can see each other's partial state (see "lost updates" below) |
| **Durability** | ✅ Yes | Each local transaction commits durably to its DB |

**The critical missing property is Isolation.** Two sagas operating on the same entity can interleave and corrupt each other. This requires careful defensive design.

#### Saga Type 1: Choreography (Event-Driven)

Services emit events. Other services react to events and emit their own. No central brain.

```
OrderService        InventoryService      PaymentService        ShippingService
     │                     │                    │                     │
     │─── OrderCreated ───►│                    │                     │
     │                     │── InventoryReserved►│                    │
     │                     │                    │─── PaymentCharged ──►│
     │                     │                    │                     │── OrderShipped
     │◄──────────────────────────────────────────────── OrderShipped ──┘

Failure path (PaymentFailed):
PaymentService ──► PaymentFailed event
                        │
         ┌──────────────┘
         ▼
InventoryService reacts to PaymentFailed:
  InventoryService.on_payment_failed(event) → release_reservation()
  emits: InventoryReleased
         │
         ▼
OrderService reacts to InventoryReleased:
  OrderService.on_inventory_released(event) → mark_order_cancelled()
```

**Pros:** Loose coupling. Services don't know about each other — only about events.  
**Cons:** The overall flow is implicit — spread across many services. Debugging "why did this order never complete?" requires tracing events across all services. Adding a new step means every downstream service changes its compensation logic.

**Choreography works best when:** The workflow has ≤4 steps and the team owns all services.

#### Saga Type 2: Orchestration (Central Coordinator)

A dedicated workflow service drives each step and knows the entire flow.

```python
# Temporal workflow — durable orchestrator
@workflow.defn
class PlaceOrderSaga:
    @workflow.run
    async def run(self, input: OrderInput) -> OrderResult:

        # Step 1: Reserve inventory
        try:
            reservation_id = await workflow.execute_activity(
                reserve_inventory,
                args=[input.order_id, input.items],
                start_to_close_timeout=timedelta(seconds=10),
                retry_policy=RetryPolicy(max_attempts=3)
            )
        except ApplicationError as e:
            # Inventory step failed — nothing to compensate yet
            await workflow.execute_activity(cancel_order, args=[input.order_id])
            return OrderResult(status="failed", reason="inventory_unavailable")

        # Step 2: Charge payment
        try:
            charge_id = await workflow.execute_activity(
                charge_payment,
                args=[input.order_id, input.total_amount],
                start_to_close_timeout=timedelta(seconds=30),
                retry_policy=RetryPolicy(max_attempts=3)
            )
        except ApplicationError as e:
            # Payment failed — compensate step 1
            await workflow.execute_activity(release_inventory, args=[reservation_id])
            await workflow.execute_activity(cancel_order, args=[input.order_id])
            return OrderResult(status="failed", reason="payment_declined")

        # Step 3: Dispatch shipping
        try:
            tracking_id = await workflow.execute_activity(
                dispatch_shipping,
                args=[input.order_id, input.address],
                start_to_close_timeout=timedelta(minutes=5),
                retry_policy=RetryPolicy(max_attempts=5)
            )
        except ApplicationError:
            # Shipping failed — compensate steps 1 and 2
            await workflow.execute_activity(refund_payment, args=[charge_id])
            await workflow.execute_activity(release_inventory, args=[reservation_id])
            await workflow.execute_activity(cancel_order, args=[input.order_id])
            return OrderResult(status="failed", reason="shipping_unavailable")

        return OrderResult(
            status="complete",
            tracking_id=tracking_id,
            charge_id=charge_id
        )
```

**What Temporal adds:** If the workflow process crashes mid-execution, Temporal replays the workflow from the beginning using its event history, but skips already-completed activities (their results are stored). The workflow resumes exactly where it left off — the application developer never writes retry/crash-recovery code.

**Pros:** The entire flow is visible in one place. Easy to debug, add steps, and change compensation logic.  
**Cons:** The orchestrator is a single point of coordination — it must be highly available. Temporal/Conductor handles this; a hand-rolled orchestrator often doesn't.

#### Designing Compensating Transactions

The hardest part of Sagas. Compensations are not simply `ROLLBACK` — by the time compensation runs, the local transaction has already committed and its effects may have cascaded.

**Rules for compensating transactions:**
1. **Must be idempotent:** Compensation may be retried. Running it twice must produce the same result.
2. **Must be eventually executable:** The compensating operation must be able to succeed even if the service is temporarily down — use a durable queue, not a synchronous call.
3. **Cannot always fully undo:** If a charge was captured and the customer was already emailed, compensation refunds the charge but cannot un-send the email. Design for this.
4. **Track compensation status:** Store compensation results in the DB. Know whether a compensation succeeded, is pending, or failed permanently.

```python
# Compensating transaction — idempotent refund
def refund_payment(charge_id, order_id):
    # Check if already refunded (idempotency)
    existing = db.query(
        "SELECT id FROM refunds WHERE charge_id = %s", [charge_id]
    )
    if existing:
        logger.info(f"Refund for charge {charge_id} already processed")
        return existing[0].id  # Return existing refund ID, don't double-refund

    refund_id = payment_gateway.refund(charge_id)

    db.insert(
        "INSERT INTO refunds (charge_id, order_id, refund_id, created_at) VALUES (%s, %s, %s, now())",
        [charge_id, order_id, refund_id]
    )
    return refund_id
```

#### The Isolation Problem: Countermeasures

Since Sagas have no isolation, two concurrent Sagas on the same entity can cause:

- **Lost Update:** Saga 1 reads a value, Saga 2 modifies it, Saga 1 overwrites Saga 2's change.
- **Dirty Read:** Saga 2 reads data written by Saga 1 mid-flight. Saga 1 compensates → Saga 2's read was of phantom data.
- **Phantom:** Saga 1 makes a decision based on a count. Saga 2 adds an item. Saga 1's decision is now wrong.

**Countermeasures:**

| Countermeasure | How | Example |
|----------------|-----|---------|
| **Semantic Lock** | Mark the entity as "in-progress" at saga start; other sagas reject or wait | `orders.status = "in_progress"` — reject concurrent checkout of same order |
| **Commutative Updates** | Design updates that can be applied in any order | Use `balance += amount` instead of `balance = new_value` — order-independent |
| **Pessimistic View** | Re-read data at each saga step; don't rely on data read in earlier steps | At payment step, re-verify inventory is still available |
| **Re-read Before Write** | Check current value before applying write in compensation | Before releasing inventory, verify it's still in "reserved" state |
| **Version Flag** | Track saga instance on the entity; reject writes from stale sagas | `orders.saga_id = current_saga_id` — mismatched saga_id → reject |

```python
# Semantic lock: prevent concurrent checkout of same order
def start_checkout_saga(order_id):
    rows_updated = db.execute("""
        UPDATE orders SET status = 'in_progress', saga_id = %s
        WHERE order_id = %s AND status = 'created'
    """, [saga_id, order_id])

    if rows_updated == 0:
        raise OrderAlreadyInProgress(order_id)

    # Now safe to proceed — other sagas are blocked by status check
```

---

---

### Pattern 6: Outbox Pattern (Transactional Messaging)

**Problem:** You want to write to DB AND publish an event to Kafka atomically. If the DB write succeeds but Kafka publish fails, your system is inconsistent.

```python
# BAD: Two operations, non-atomic
db.execute("INSERT INTO orders ...")
kafka.publish("order.created", order_data)  # ← what if this fails?
```

**Solution: Outbox table**
```python
# Transactional outbox: write to DB and outbox in same transaction
with db.transaction():
    order_id = db.insert("INSERT INTO orders ...")
    db.insert("""
        INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload, status)
        VALUES ('order', %s, 'OrderCreated', %s, 'pending')
    """, [order_id, json.dumps(order_data)])
# Transaction committed atomically

# Background relay process (separate service):
def relay_outbox_events():
    events = db.query("SELECT * FROM outbox WHERE status = 'pending' ORDER BY id LIMIT 100")
    for event in events:
        kafka.publish(event.event_type, event.payload)
        db.execute("UPDATE outbox SET status = 'sent' WHERE id = %s", [event.id])
```

**Guarantee:** Even if the relay crashes mid-way, it will re-process events on restart (idempotency needed at the consumer).

**Tools:** Debezium can automatically relay outbox events via CDC (no relay process needed).

> 📖 **Reference:** [Microservices Patterns: The Outbox Pattern (Chris Richardson)](https://microservices.io/patterns/data/transactional-outbox.html)

---

## 6. When to Use What: Decision Guide

This is the question that matters most in system design. The right protocol is determined by three axes: **how many databases are involved**, **what failure tolerance you need**, and **how much latency you can accept**.

### Decision Tree

```
Is the entire transaction within a SINGLE database?
  └─ YES → Use a local ACID transaction. No distributed protocol needed.
            Set isolation level based on your anomaly risk.

Is the transaction across MULTIPLE databases/services?
  └─ YES
      Can you tolerate eventual consistency (seconds of inconsistency)?
        └─ YES → Use SAGA (choreography or orchestration)
        └─ NO (must be atomic) →
            Are all participants in the SAME datacenter?
              └─ YES, low latency →
                  Are you using XA-compatible databases (Postgres, MySQL)?
                    └─ YES → 2PC via XA is viable
                    └─ NO  → Use TCC pattern
              └─ NO, cross-region →
                  Are you building infrastructure or application logic?
                    └─ Application → Saga is the only practical choice
                    └─ Infrastructure/DB → Use a NewSQL DB (Spanner, CockroachDB)
                                          which handles distribution internally
```

### Comparison Table

| Protocol | Consistency | Availability | Latency | Complexity | Lock Duration | Failure Mode |
|----------|-------------|--------------|---------|------------|---------------|--------------|
| **Local ACID** | Strong | High | Low (1ms) | Low | Short | DB crash only |
| **2PC (same DC)** | Strong | Medium | Medium (5-20ms) | Medium | Phase 1 → Phase 2 | Blocking on coordinator crash |
| **2PC (cross-region)** | Strong | Low | High (160ms+) | High | Very long | Blocking; locks held across regions |
| **3PC** | Strong | High (crash) | High (3 RTTs) | Very High | Phase 2 → Phase 3 | Split-brain on partition |
| **TCC** | Strong (eventual confirm) | High | Medium | High | Try TTL only | Expired holds if confirm delayed |
| **Saga (choreography)** | Eventual | Very High | Low (async) | Medium | None (local txns) | Partial completion visible |
| **Saga (orchestration)** | Eventual | High | Low (async) | High | None | Orchestrator is SPOF if unmanaged |
| **Raft-based NewSQL** | Strong | High | Medium (Raft RTT) | Low (for app) | Per-range | Leader election window |

### Concrete Scenarios → Protocol Mapping

| Scenario | Recommended Protocol | Why |
|----------|---------------------|-----|
| Bank transfer within one DB | Local ACID + `SERIALIZABLE` | Single DB, strong consistency needed |
| Cross-bank wire transfer (two banks, same DC) | 2PC via ISO 20022 / XA | Industry standard, same-DC latency acceptable |
| E-commerce order (inventory + payment + shipping) | Saga (orchestrated) | Three separate services, eventual is OK, need recovery |
| Hotel + Flight booking (must book both or neither) | TCC | Both services must be reserved before confirming |
| Multi-region financial transaction (Stripe/PayPal scale) | Spanner (internal Paxos) | Handles distribution internally, app sees single ACID DB |
| Sending email after DB write | Outbox pattern | Exactly-once delivery without distributed transaction |
| Distributed cache invalidation across regions | Eventual + TTL | Strong consistency not worth the cost |
| Inventory reservation across warehouses | Saga + semantic lock | Reserve from nearest, compensate on failure |
| Distributed cron job (run exactly once) | Advisory lock + DB-backed schedule | No distributed protocol, single writer |

---

## 7. Common Errors in Global Distributed Databases

This section covers the errors that appear in production with distributed transaction systems — most of them subtle, expensive, and hard to reproduce in test environments.

---

### Error 1: The In-Doubt Transaction (2PC Coordinator Crash)

**What happens:** Coordinator writes the commit decision to its log and crashes before sending COMMIT to all participants. Participants are PREPARED — they have acquired locks and are waiting. The coordinator is down for 10 minutes during incident response.

**Visible symptom:**
```sql
-- On a participant (Postgres), 10 minutes after crash:
SELECT * FROM pg_prepared_xacts;
-- gid = 'txn-payment-2025-03-01-482931'
-- prepared = '2025-03-01 14:22:17'  ← 10 minutes ago
-- This row is holding locks. Other transactions are blocked.
```

**Impact:** Every query touching the locked rows blocks until the coordinator recovers or an operator manually resolves the in-doubt transaction. This can cascade into a full service outage.

**Resolution procedure:**
```sql
-- Step 1: Determine the coordinator's decision from its logs
-- (Check coordinator's WAL or transaction log for 'COMMIT' or 'ABORT' record)

-- Step 2a: If coordinator decided COMMIT
COMMIT PREPARED 'txn-payment-2025-03-01-482931';

-- Step 2b: If coordinator decided ABORT, or you cannot determine
ROLLBACK PREPARED 'txn-payment-2025-03-01-482931';

-- WARNING: Step 2b can cause data inconsistency if other participants already committed
-- Always check ALL participants before rolling back
```

**Prevention:**
- Use a highly available coordinator (run coordinator in a Raft group, not a single process)
- Set `lock_timeout` on the coordinator to avoid indefinite lock-holding
- Monitor `pg_prepared_xacts` with an alert on age > 2 minutes
- Prefer Saga + Outbox over 2PC wherever possible

---

### Error 2: Saga Compensation Failure (The Half-Refunded Order)

**What happens:** Order saga charges the customer ($150), dispatches shipping, then the shipping service returns a permanent error. The saga tries to refund — but the payment gateway is also down.

```
Saga step 1: InventoryReserved   ✅
Saga step 2: PaymentCharged      ✅  ($150 captured)
Saga step 3: ShippingDispatched  ❌  (permanent failure)

Compensation step 1: RefundPayment   ❌  (payment gateway down)
  → retry in 60s
Compensation step 1: RefundPayment   ❌  (still down)
  → retry in 120s
... (payment gateway down for 4 hours) ...
Compensation step 1: RefundPayment   ✅  (finally succeeds)
```

**Impact during those 4 hours:** Customer was charged $150 and their order was never created. The order shows "processing." This is not a data integrity violation — the saga will eventually compensate — but it's a severe user experience and support issue.

**Solutions:**

1. **Pivot transaction:** When compensation cannot complete, pivot the saga forward instead of backward. Can shipping be retried with a different provider? Can the item be fulfilled manually?

2. **Dead letter queue for compensations:** Failed compensations go to a DLQ. A human operator or background processor works through them with retries and manual overrides.

3. **Saga status dashboard:** Every saga instance has a visible status. Support team can see "payment charged, compensation pending" and take action.

4. **SLA-based escalation:** If a compensation has been retrying for >30 minutes, page the on-call engineer.

```python
# Track compensation status explicitly
def execute_compensation_with_tracking(saga_id, step_name, compensation_fn, *args):
    db.upsert("saga_compensations",
        saga_id=saga_id,
        step=step_name,
        status="pending",
        attempts=0,
        created_at=now()
    )

    for attempt in range(MAX_ATTEMPTS):
        try:
            result = compensation_fn(*args)
            db.update("saga_compensations",
                where={"saga_id": saga_id, "step": step_name},
                set={"status": "completed", "result": result, "completed_at": now()}
            )
            return result
        except Exception as e:
            db.update("saga_compensations",
                where={"saga_id": saga_id, "step": step_name},
                set={"attempts": attempt + 1, "last_error": str(e), "last_attempt_at": now()}
            )
            time.sleep(exponential_backoff(attempt))

    # Max retries exceeded → human intervention required
    db.update("saga_compensations",
        where={"saga_id": saga_id, "step": step_name},
        set={"status": "needs_manual_review"}
    )
    alert_oncall(f"Saga {saga_id} compensation '{step_name}' requires manual review")
```

---

### Error 3: Dual Write Without Outbox (Guaranteed Data Loss Under Crash)

**What happens:** Developer writes to DB and then publishes to Kafka. The process crashes between the two. The DB write is committed but the Kafka message is never published.

```python
# This code has a guaranteed data loss window
def create_order(order_data):
    order = db.insert("INSERT INTO orders ...")  # DB committed ✅
    # --- PROCESS CRASHES HERE ---
    kafka.publish("order.created", order)  # Never happens ❌

# Downstream services (payment, shipping, notification) never hear about this order.
# The order sits in "pending" forever.
# Customer paid but nothing happens.
```

**Why this is insidious:** Under normal operation, the crash window is microseconds — it almost never triggers in testing. Under a deployment, OOM kill, or infrastructure issue, it triggers exactly when you least want it to.

**Detection:** Monitor for orders stuck in "pending" > 10 minutes. This is your early warning signal.

**Fix:** Outbox pattern (as shown in Pattern 6/7). The DB write and the outbox event are in the same transaction. The relay will publish regardless of when the crash happens.

---

### Error 4: Clock Skew Causing Incorrect Transaction Ordering

**What happens:** Two microservices each write to their own databases, using `NOW()` as the timestamp. Service A's clock is 50ms ahead of Service B's. Events from Service A appear to have happened "after" Service B's events even if they logically preceded them.

**Concrete failure:**
```
Real event sequence (wall clock):
  T=0ms:   User updates email in ProfileService  → profile_events: ts=1000050
  T=10ms:  AuthService reads profile to build JWT → reads email from snapshot at ts=1000000
           (AuthService clock is 50ms behind) → reads OLD email

JWT is issued with wrong email. User cannot receive verification emails.
```

**More dangerous version (financial):**
```
T=0ms:   FX rate updated: EUR/USD = 1.08
T=5ms:   Trade executed at EUR/USD = 1.07 (stale rate due to clock skew)
         → incorrect pricing, financial loss
```

**Solutions:**

```python
# 1. Use logical clocks (Lamport timestamps) for causal ordering
class LamportClock:
    def __init__(self):
        self.counter = 0

    def tick(self):
        self.counter += 1
        return self.counter

    def update(self, received_ts):
        self.counter = max(self.counter, received_ts) + 1
        return self.counter

# Each event carries a Lamport timestamp
event = {"data": ..., "lamport_ts": clock.tick()}

# On receive: update local clock to be at least as large as received
clock.update(incoming_event["lamport_ts"])
```

```python
# 2. Use NTP + bounded clock sync check (reject writes if clock is too far off)
import ntplib

def assert_clock_sync(max_offset_ms=100):
    c = ntplib.NTPClient()
    response = c.request("pool.ntp.org")
    offset_ms = abs(response.offset * 1000)
    if offset_ms > max_offset_ms:
        raise ClockDriftTooLarge(f"Clock offset {offset_ms}ms exceeds limit {max_offset_ms}ms")

# 3. Use a single authoritative time source for all order-sensitive events
# (e.g., all timestamps from a central time service, or Spanner's TrueTime)
```

---

### Error 5: Split Brain — Two Leaders Accept Writes Simultaneously

**What happens:** In a primary-replica setup, a network partition causes the replica to believe the primary is dead. The replica promotes itself. Now two nodes both think they are the primary and both accept writes.

```
Normal:
  Primary (NYC) ←── replication ──► Replica (LA)

Network partition (NYC ↔ LA severed):
  NYC: "I am primary, accepting writes"   ← clients in NYC write here
  LA:  "Primary is dead, I am now primary" ← clients in LA write here

Same user's account balance:
  NYC: balance = $1000 - $200 = $800
  LA:  balance = $1000 - $300 = $700

Partition heals:
  Which balance is correct? $800? $700? $500?
  Both nodes have diverged. There is no automatic answer.
```

**This is the most catastrophic distributed systems failure — data corruption without an obvious error.**

**How modern systems prevent it:**

```
1. Raft / Paxos consensus:
   A node can only accept writes if it is the current Raft leader.
   Raft ensures at most one leader at any time (by requiring a quorum of votes).
   No quorum → no leader → no writes → system blocks (CP, not AP).

2. Fencing tokens:
   Every time a new leader is elected, it gets a higher-numbered fencing token.
   All writes include the token. Storage layer rejects writes with stale tokens.

   Leader 1: token=100, writes "balance=800" with token=100
   Leader 2 (after re-election): token=101
   Leader 1 recovers and tries to write with token=100
   → Storage rejects: "token 100 < current token 101"

3. Quorum writes (Cassandra):
   Write with QUORUM consistency → must reach majority of replicas.
   Under partition, the minority partition cannot achieve quorum → writes fail.
   The majority partition continues normally.
```

**Detection in production:**
```python
# Monitor replication lag — sustained high lag is a partition warning
alert_if(
    metric="replication_lag_seconds",
    condition="> 30 for 2 minutes",
    action="page_oncall",
    message="Possible network partition or primary overload"
)

# Monitor for two primaries (split brain detection)
# Most HA tools (Patroni, Orchestrator) have built-in split-brain protection
```

---

### Error 6: Idempotency Violation on Retry (Phantom Charges)

**What happens:** A payment request times out at the network layer. The client retries. The first request actually completed on the server but the response was lost. The retry creates a second charge.

```
Client → POST /charge {amount: $100}  → (response lost, client times out)
Client → POST /charge {amount: $100}  → (server processes again)

Customer account: charged $200 instead of $100
```

**This happens constantly in distributed systems** — network timeouts, load balancer retries, Kubernetes pod restarts, mobile clients retrying.

**Fix: Idempotency keys, properly implemented**

```python
# Client: always send an idempotency key
import uuid

idempotency_key = str(uuid.uuid4())  # Generated once per user action, stored locally
# If the app restarts before getting a response, the SAME key is reused for the retry

response = http.post("/charge",
    json={"amount": 100},
    headers={"Idempotency-Key": idempotency_key}
)

# Server: deduplicate based on idempotency key
def charge(request):
    key = request.headers["Idempotency-Key"]

    # Check if already processed
    existing = redis.get(f"idempotency:{key}")
    if existing:
        return json.loads(existing)  # Return stored response, don't charge again

    # Process
    result = payment_gateway.charge(request.json["amount"])

    # Store result with TTL (24 hours is typical)
    redis.setex(f"idempotency:{key}", 86400, json.dumps(result))
    return result
```

**The subtle bug:** Storing the idempotency key in Redis *after* the charge creates a window where two concurrent requests both miss the cache and both charge. Fix: store a "processing" marker first, then the result:

```python
def charge(request):
    key = request.headers["Idempotency-Key"]

    # Atomic: set only if not exists (NX), with short TTL for the "in-flight" state
    acquired = redis.set(f"idempotency:{key}", "processing", nx=True, ex=30)

    if not acquired:
        # Another request with this key is in flight or already completed
        result = wait_for_result(key)  # Poll or block briefly
        return result

    try:
        result = payment_gateway.charge(request.json["amount"])
        redis.setex(f"idempotency:{key}", 86400, json.dumps(result))  # Store final result
        return result
    except Exception as e:
        redis.delete(f"idempotency:{key}")  # Release so client can retry
        raise
```

---

### Error 7: Read-Your-Own-Writes Violation in Multi-Region

**What happens:** User in Tokyo updates their profile (writes to Tokyo replica, which syncs to US primary). User's next request hits a different Tokyo replica that hasn't received the update yet (replication lag 200ms). User sees their old profile.

```
Timeline:
  T=0ms:   User writes new bio → Tokyo write replica → US primary (sync commit)
  T=10ms:  Tokyo read replica 1 has the update (fast sync)
  T=10ms:  Tokyo read replica 2 does NOT have the update yet (lag 150ms)
  T=20ms:  User reads profile → hits replica 2 → sees old bio
  T=170ms: Replica 2 catches up
  T=170ms: User reads again → sees new bio
```

**The user experience:** "I changed my bio but it shows the old one. Did it save? Let me save again." → double writes, confusion, support tickets.

**Solutions ranked by correctness vs. cost:**

```
Option 1 — Session affinity (cheapest):
  After a write, set a cookie/header: "primary_read_until = now + 10s"
  During that window, route reads to the primary (or the replica that confirmed the write)
  After the window, route to any replica

Option 2 — Read your own writes via token:
  Write returns a "replication token" (a timestamp or LSN — Log Sequence Number)
  Reads include this token
  Replica checks: "Have I applied at least up to this LSN?"
    YES → serve read
    NO  → wait (up to Xms), or proxy to primary

Option 3 — Sync replication for important writes:
  For writes the user immediately reads back (profile, settings),
  use synchronous replication: wait for at least one replica to confirm
  Latency cost: +1 RTT to Tokyo replica (~5ms intra-DC)

Option 4 — Always read from primary after write:
  Simple but expensive at scale — negates the value of replicas
  Viable for low-traffic write operations (user settings)
```

---

### Error 8: Thundering Herd on Database After Partition Heals

**What happens:** DB goes down for 5 minutes. Application servers queue up work (retries, buffers). When DB comes back, 500k queued operations hit the newly recovered DB simultaneously — immediately overloading it and causing another outage.

```
DB down for 5 minutes:
  API servers: buffering / queuing failed requests
  Background workers: all stuck, accumulating backlog

DB recovers:
  All 500k queued requests fire simultaneously
  DB connections saturate (max 500 connections, 100k trying to connect)
  Query queue depth → infinite
  DB falls over again under load spike → oscillating failure
```

**Solutions:**

```python
# 1. Controlled replay with exponential backoff and jitter
# Kafka consumers: don't rush to catch up after partition heals

def consume_with_backpressure(kafka_consumer, db):
    rate_limiter = TokenBucket(rate=1000)  # Start conservatively: 1000 ops/sec

    for message in kafka_consumer:
        rate_limiter.acquire()  # Block if over rate limit

        try:
            db.write(message)
            rate_limiter.on_success()  # Gradually increase rate on success
        except DBOverloadError:
            rate_limiter.on_failure()  # Reduce rate on overload signal
            kafka_consumer.pause()     # Stop pulling from Kafka temporarily
            time.sleep(rate_limiter.backoff())
            kafka_consumer.resume()

# 2. Circuit breaker: if DB is responding slowly, stop sending it more work
@circuit(failure_threshold=10, recovery_timeout=30, expected_exception=DBError)
def write_to_db(data):
    return db.execute(data)

# 3. Gradual traffic ramp: after DB restart, don't route 100% of traffic immediately
#    Route 5% → monitor metrics → 25% → 50% → 100% over 5 minutes
```

---

### Error 9: Cascading Transaction Rollback Across Services

**What happens:** A long saga has 8 steps. Step 8 fails. The saga attempts to compensate steps 7, 6, 5... but the compensation for step 3 calls an external API that is now rate-limiting. The compensation queue backs up. Thousands of orders are stuck mid-saga.

**This is the distributed systems equivalent of a deadlock — but across time.**

```
1,000 orders each reach step 8 simultaneously → step 8 failure spike
1,000 compensations attempt step 7 → step 7 service handles fine
1,000 compensations attempt step 6 → slightly slower
...
1,000 compensations attempt step 3 → external API rate limit: 100 req/min
→ 900 compensations are queued
→ New orders keep failing at step 8
→ Queue depth grows without bound
→ System memory exhausted on compensation queue
```

**Solutions:**

1. **Compensation queue with rate limiting and back-pressure:**
```python
# Compensation queue — bounded, with rate limiting
compensation_queue = BoundedQueue(max_size=10000)

def enqueue_compensation(saga_id, step, payload):
    if compensation_queue.is_full():
        # Shed load: mark saga as "pending_manual_review" instead
        db.update_saga_status(saga_id, "compensation_overloaded")
        alert_oncall(f"Compensation queue full — saga {saga_id} needs manual review")
        return
    compensation_queue.enqueue((saga_id, step, payload))

# Worker with rate limiter
def compensation_worker():
    rate_limiter = RateLimiter(rate=80)  # Stay under external API's 100 req/min limit
    while True:
        item = compensation_queue.dequeue(timeout=1)
        rate_limiter.acquire()
        execute_compensation(item)
```

2. **Timeout each compensation step:** Don't let a slow external service hold up all compensations. Each step has a deadline; exceeded = mark as pending and move on.

3. **Dead letter + SLA:** Compensations older than X hours go to a DLQ with a page to engineering. Some compensations genuinely require human judgment.

---

## 8. Regular Scenarios

### Scenario 1: E-commerce Checkout
**Requirement:** Atomically: reserve inventory, create order, charge payment. All three succeed or none.

**Architecture using Saga + Outbox:**
1. API creates order in DB with status="pending" + outbox event "OrderCreated"
2. Background relay publishes "OrderCreated" to Kafka
3. Payment service consumes, charges, emits "PaymentCharged"  
4. Inventory service consumes, reserves, emits "InventoryReserved"
5. Order service consumes both events, marks order="confirmed"

On any step failure, compensation events trigger rollbacks.

---

### Scenario 2: Bank Transfer
**Requirement:** Move $100 from Alice to Bob. Zero money created or destroyed. Concurrent transfers safe.

```python
def transfer(from_id, to_id, amount):
    # Lock in consistent order to prevent deadlocks
    lock_order = sorted([from_id, to_id])
    
    with db.transaction(isolation="REPEATABLE READ"):
        # Lock both rows in consistent order
        accounts = db.query(
            "SELECT * FROM accounts WHERE id = ANY(%s) ORDER BY id FOR UPDATE",
            [lock_order]
        )
        
        sender = next(a for a in accounts if a.id == from_id)
        receiver = next(a for a in accounts if a.id == to_id)
        
        if sender.balance < amount:
            raise InsufficientFunds()
        
        db.execute("UPDATE accounts SET balance = balance - %s WHERE id = %s", [amount, from_id])
        db.execute("UPDATE accounts SET balance = balance + %s WHERE id = %s", [amount, to_id])
        db.execute("INSERT INTO ledger (from_id, to_id, amount, ts) VALUES (%s, %s, %s, now())",
                   [from_id, to_id, amount])
```

---

### Scenario 3: Booking System (Doctor Appointments)
**Requirement:** Prevent double-booking. No phantom reads or write skew.

```python
def book_appointment(doctor_id, slot_datetime, patient_id):
    with db.transaction(isolation="SERIALIZABLE"):
        # Under SERIALIZABLE, this SELECT acts as a predicate lock
        # Prevents concurrent inserts for the same (doctor_id, slot_datetime)
        existing = db.query(
            "SELECT id FROM appointments WHERE doctor_id = %s AND slot = %s",
            [doctor_id, slot_datetime]
        )
        if existing:
            raise SlotUnavailable()
        
        db.execute(
            "INSERT INTO appointments (doctor_id, patient_id, slot) VALUES (%s, %s, %s)",
            [doctor_id, patient_id, slot_datetime]
        )
```

Under `SERIALIZABLE`, Postgres SSI (Serializable Snapshot Isolation) detects the write skew and aborts one of the concurrent transactions — guaranteeing no double-booking.

---

## 9. Tricky Scenarios

### Tricky Scenario 1: Long-Running Transaction Holding Locks
**Problem:** Transaction starts, performs some computation, then writes. Holds row lock for seconds.

```python
# BAD: DB lock held for the duration of all computation
with db.transaction():
    user = db.get_user(user_id)  # Lock acquired here
    report = compute_heavy_report(user)  # 2 seconds of CPU!
    db.save(report)  # Finally releases lock after 2 seconds
```

**Fix: Do computation outside the transaction**
```python
user = db.get_user(user_id)  # No transaction
report = compute_heavy_report(user)  # 2 seconds, no lock held

with db.transaction():  # Lock held for <10ms
    # Re-check that user data hasn't changed if critical
    db.save(report)
```

---

### Tricky Scenario 2: Nested Transactions and Rollback Semantics
**Problem:** Inner transaction rolls back. Does the outer transaction also roll back?

In most databases (Postgres, MySQL), **nested transactions aren't truly nested** — they're flat. A ROLLBACK in an inner BEGIN block rolls back the entire outer transaction.

```sql
BEGIN;  -- outer
INSERT INTO orders ...;

BEGIN;  -- this is IGNORED — there's only one transaction
INSERT INTO payments ...;
ROLLBACK;  -- THIS ROLLS BACK EVERYTHING, including the order!

-- orders row is gone too!
```

**Correct approach: Use savepoints for inner scopes**
```sql
BEGIN;
INSERT INTO orders ...;

SAVEPOINT payment_attempt;
INSERT INTO payments ...;
ROLLBACK TO SAVEPOINT payment_attempt;  -- Only undoes payment

-- orders row is still here
COMMIT;
```

---

### Tricky Scenario 3: Distributed Clock Skew in Timestamps
**Problem:** Using `created_at = NOW()` for ordering events across microservices. Clock drift between servers means event B (which logically happened after A) has an earlier timestamp.

```
Server 1 clock: 12:00:00.000 → emits Event A at 12:00:00.001
Server 2 clock: 11:59:59.980 (20ms behind) → emits Event B at 11:59:59.982

Event B.timestamp < Event A.timestamp, but Event B happened after!
```

**Solutions:**
1. **Use a monotonic logical clock (Lamport timestamps):** Each event carries a counter incremented on send/receive
2. **Google TrueTime (Spanner):** GPS and atomic clocks, bounded uncertainty of ~7ms → Spanner waits 7ms before committing to ensure causality
3. **Hybrid Logical Clocks (HLC):** Combines physical time with logical counters (used by CockroachDB, YugabyteDB)

> 📖 **Reference:** [Spanner: Google's Globally Distributed Database (TrueTime section)](https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf)

---

### Tricky Scenario 4: The Halloween Problem (UPDATE reads its own writes)
**Problem in Postgres (historical — fixed, but instructive):** An `UPDATE` on an indexed column could re-update rows it already updated because the index scan finds the updated rows again.

```sql
UPDATE employees SET salary = salary * 1.1 WHERE salary < 50000;
-- If scan uses the salary index, newly-updated rows (now salary > 50000 after 10% raise)
-- might be re-encountered and updated again!
```

Modern databases solve this with snapshot isolation — the UPDATE reads from the snapshot before any updates in the same statement. But be aware of this pattern when writing multi-step update scripts.

---

### Tricky Scenario 5: ORM-Level Transaction Gotchas
**Problem:** ORMs like Django, Rails, and SQLAlchemy manage transactions implicitly. Common mistake: assuming a try/except block inside a transaction rolls back cleanly.

**Django:**
```python
# BAD: After IntegrityError, the transaction is "aborted"
# Any further DB operations will fail with "InAbortedTransactionError"
with transaction.atomic():
    try:
        Order.objects.create(...)
        Payment.objects.create(...)  # IntegrityError here
    except IntegrityError:
        # Transaction is still "aborted" — you cannot run DB queries here!
        logger.error("Payment failed")  # Fine
        User.objects.get(id=user_id)    # This FAILS with TransactionManagementError
```

**Fix: Nested atomic blocks (savepoints)**
```python
with transaction.atomic():  # outer
    order = Order.objects.create(...)
    
    try:
        with transaction.atomic():  # inner — uses savepoint
            Payment.objects.create(...)
    except IntegrityError:
        # Savepoint rolled back; outer transaction still active
        logger.error("Payment failed")
        order.status = "payment_failed"
        order.save()  # This works — outer transaction still open
```

> 📖 **Reference:** [Django Docs: Controlling Transactions Explicitly](https://docs.djangoproject.com/en/stable/topics/db/transactions/)

---

### Tricky Scenario 6: Serialization Failures Under Load (Postgres SERIALIZABLE)
**Problem:** You set isolation to SERIALIZABLE for correctness. Under high load, you get frequent `ERROR: could not serialize access due to concurrent update` errors.

**This is expected and correct behavior.** Postgres SSI aborts transactions that would violate serializability.

**Your application must handle it:**
```python
MAX_RETRIES = 5

def book_with_serializable(doctor_id, slot, patient_id):
    for attempt in range(MAX_RETRIES):
        try:
            with db.transaction(isolation="SERIALIZABLE"):
                if appointment_exists(doctor_id, slot):
                    raise SlotUnavailable()
                create_appointment(doctor_id, slot, patient_id)
            return  # Success
        except SerializationFailure:
            if attempt == MAX_RETRIES - 1:
                raise
            time.sleep(0.01 * (2 ** attempt))  # Exponential backoff
```

**Performance tip:** For write-heavy serializable workloads, reduce transaction scope as much as possible. Fewer operations = fewer conflicts = fewer serialization failures.

---

## 10. Transaction Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Transaction spanning HTTP calls | Lock held during network I/O; deadlock risk; high latency | Do I/O outside transaction; lock only for DB writes |
| Implicit transaction in loop | One failed item rolls back all | Use savepoints; commit in batches |
| Ignoring serialization errors | Data corruption on concurrent access | Always retry on serialization failure |
| Using READ UNCOMMITTED | Dirty reads → garbage data | Never use in production |
| Auto-commit per statement | No atomicity across statements | Explicit transaction for multi-step operations |
| SELECT without FOR UPDATE before write | TOCTOU race condition | Use `SELECT FOR UPDATE` or optimistic locking |
| Long transactions in Postgres | Table bloat from dead tuples; replication slot lag | Keep transactions short; monitor transaction age |

---

## 11. Monitoring Transactions in Production

```sql
-- Postgres: find long-running transactions (> 5 minutes)
SELECT pid, now() - xact_start AS duration, state, query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
AND now() - xact_start > interval '5 minutes'
ORDER BY duration DESC;

-- Postgres: find transactions holding locks
SELECT pid, granted, mode, relation::regclass, query
FROM pg_locks l
JOIN pg_stat_activity a USING (pid)
WHERE relation IS NOT NULL
ORDER BY granted;

-- Alert thresholds:
-- Transaction age > 10 minutes → page on-call
-- Lock wait > 30s → investigate
-- Deadlock rate > 1/min → review lock ordering
```

---

## 12. Reference Case Studies

1. **Stripe: Designing Robust Systems with Idempotency**  
   [https://stripe.com/blog/idempotency](https://stripe.com/blog/idempotency)

2. **Microservices Patterns: Saga and Outbox (Chris Richardson)**  
   [https://microservices.io/patterns/data/saga.html](https://microservices.io/patterns/data/saga.html)

3. **Temporal: Durable Execution for Distributed Transactions**  
   [https://temporal.io/blog/workflow-engine-principles](https://temporal.io/blog/workflow-engine-principles)

4. **Designing Data-Intensive Applications — Chapter 7 (Kleppmann)**  
   [https://dataintensive.net/](https://dataintensive.net/) ← Best single resource for transactions

5. **Google Spanner: TrueTime and External Consistency**  
   [https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf](https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf)

6. **CockroachDB: Serializable Isolation**  
   [https://www.cockroachlabs.com/blog/serializable-lockless-distributed-isolation-cockroachdb/](https://www.cockroachlabs.com/blog/serializable-lockless-distributed-isolation-cockroachdb/)

7. **Postgres: Transaction Isolation Documentation**  
   [https://www.postgresql.org/docs/current/transaction-iso.html](https://www.postgresql.org/docs/current/transaction-iso.html)

8. **Debezium: Change Data Capture for the Outbox Pattern**  
   [https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/](https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/)
