
## Intuition

In interviews, you rarely need full DB design—but you _do_ need storage thinking:
- what data you store
- how you access it efficiently
- how you isolate persistence from domain logic
- what consistency guarantees your design needs


# 16.1 Repository pattern (when it’s legit vs overkill)

## Intuition

A **repository** is a design pattern that makes persistence look like you’re working with an in-memory collection, so your core logic doesn’t deal with SQL/ORM details.

## When it’s legit

- You have **aggregates** and you want to load/save them cleanly
- Multiple persistence backends possible (in-memory for tests, DB for prod)
- You want to hide query/ORM specifics behind an interface (port)

## When it’s overkill

- Simple CRUD service with no domain logic
- Only one trivial storage strategy and no need for isolation
- You end up making `Repo` = thin wrapper around DB with no value


## Practical shape

```cpp
struct IOrderRepo {
  virtual ~IOrderRepo() = default;
  virtual std::optional<Order> findById(const OrderId&) = 0;
  virtual void save(const Order&) = 0;
};
```

## Interview questions

**Q. What should a repository return—domain objects or DTOs? (domain aggregates)**
Repositories should return **domain aggregates/entities** (domain types), not API DTOs—DTOs are for transport and should be mapped at the boundary.
 
**Q. Where do complex queries go? (read model / query service)**
Put them in a **read model / query service** (or separate query repo) that returns projection DTOs, so your write-side repository stays focused on aggregates and invariants.


# 16.2 In-memory vs persistent store abstraction

## Intuition

In-memory storage is great for(data lives in RAM):
- super fast
- easy to set up
- great for **unit tests** (no DB)
- good for **prototypes** and sometimes as a **cache**

Persistent storage is needed for:
- **durability** (survives crashes/restarts) 
- **multi-instance scaling** (two app instances can read/write the same shared data)
- backup/restore, indexing, queries


## Architecture

```
Application -> IRepo (port) -> InMemoryRepo (test)
                          -> PostgresRepo (prod)
```

## Practical example

- Use `unordered_map<OrderId, Order>` for in-memory repo
- In prod, repo wraps DB client/ORM

## Interview questions

**Q. What changes when moving from memory to DB? (transactions, concurrency, persistence)**
You introduce **persistence/durability**, need **transactions** for atomic multi-step updates, handle **concurrency** across threads/processes, plus DB realities like latency, failures, and consistency/locking.

**Q. How do you keep semantics consistent across both?**
Define a clear repo contract (what `save`/`find` guarantees) and make both implementations follow it—keep invariants in the domain, and in-memory uses locks/version checks to behave like the DB.


# 16.3 Index-thinking for data access patterns

## Intuition

Indexes exist because queries exist. Start from access patterns.

## How to think (LLD way)

For each endpoint/use case, list:
- lookup keys
- sorting
- time range queries
- pagination strategy

Then design keys/indexes accordingly.

## Practical examples

**Orders**
- `GET /orders/{id}` → index on `orderId` (PK)
- `GET /users/{id}/orders?cursor=` → composite index `(userId, createdAt desc)`
- status dashboards → index `(status, updatedAt)`

## Interview questions

**Q. What’s the “main” index for your list endpoint?**
The composite index that matches the list query’s **filters + sort**—typically `(filterKey, sortKey)` like `(userId, createdAt DESC)` for “list a user’s orders”.

**Q. Cursor pagination implies what index order?**
Cursor pagination needs an index in the **same sort order as the cursor**, e.g. `ORDER BY createdAt DESC, id DESC` → index on `(createdAt DESC, id DESC)` (or `(userId, createdAt DESC, id DESC)` if you also filter by user).


# 16.4 Caching + invalidation strategies

## Intuition

Cache improves latency; invalidation protects correctness.

## Strategies

- **Read-through cache** 
	Your code always asks the cache first. If it’s missing, the cache loads from DB, stores it, returns it.  
	So your business code stays simple: “get(key)”.

- **Write-through**
	When you update data, you update **DB and cache immediately**.  
    This keeps cache fresh but makes writes a bit slower and requires careful ordering.

- **TTL (time-to-live)**
	Every cached entry expires after a time (e.g., 5 minutes).  
	Simple, but you might serve **stale** data until expiry.

- **Event invalidation**
	When data changes, you publish an update event and caches evict/update that key immediately.  
	More correct, but needs a reliable event mechanism.  

Common best practice: **TTL + event invalidation** (events for freshness, TTL as safety net).

## Practical cache placement

### Where to place the cache

**Per-service in-memory cache**
- fastest (in-process)
- but each instance has its own cache (not shared), so duplicates + inconsistency across instances
- great for small hot data or request-local acceleration

**Distributed cache (Redis)**
- shared across instances
- better hit rate and consistency
- slightly slower than in-memory and adds network dependency

## Thread-safety note

A cache is **shared mutable state** inside a process:
- multiple threads may read/write the same map  

 So you need:
- a mutex, or
- **striped/sharded locks** to reduce contention, and keep critical sections small.

## Interview questions

**Q. TTL vs event invalidation—when choose which?**
Use **TTL** when some staleness is acceptable and you want simplicity; use **event invalidation** when updates must reflect quickly and you can publish reliable update events—often the best is **event invalidation + TTL as a safety net**.

**Q. How do you avoid cache stampede? (single-flight, jittered TTL)**
the “cache stampede” problem is when **many requests hit the cache at the same time, see a miss/expired key, and all rush to the DB**. That can overload DB and cause cascading failures.

### 1) Single-flight (one loader)

Idea: for a given key, **only one request is allowed to fetch from DB**, others wait for that result.

Flow:
- Request A misses cache → becomes the “leader” → loads from DB → writes cache
- Requests B, C, D miss cache too → but they **don’t** load DB → they wait for A’s result (or briefly serve stale if you allow)

So DB sees **1 query instead of 100**.

### 2) Jittered TTL (avoid synchronized expiry)

If you set TTL=60s for everything, many keys can expire around the same time (especially if loaded together), causing a burst of misses.

Jitter means: instead of exact TTL, add randomness:

- TTL = 60s ± random(0..10s)

So keys expire **spread out**, not all at once, reducing “thundering herd” spikes.

In one line: **single-flight limits DB load per key; jittered TTL spreads expiries to avoid spikes.**


# 16.5 Consistency models: strong vs eventual (LLD implications)

## Intuition

Consistency is a product requirement, not a DB feature.
This note is about choosing **how “fresh” reads must be after a write**, and how that choice changes your LLD.

## Strong consistency

After you write something, any read immediately reflects it.
- You update `Order.status = PAID`
- Next read shows `PAID` right away

**LLD implication:** easier to reason about rules/invariants, but may require tighter coordination (locks/transactions) and can cost latency/availability in distributed setups

## Eventual consistency

After a write, some reads might still show the old value for a short time.
- `Order` is updated to `PAID`
- A dashboard/read-model may still show `CREATED` for a few seconds until it catches up

**LLD implication:** you must design for:
- **idempotency** (events/commands can be delivered twice)
- **retries** (temporary failures)
- **compensation** (undo/fix when async steps fail)

### 1) Idempotency

Events/commands may be delivered **more than once** (network glitches, at-least-once delivery).  
So handlers must be safe to run twice without double effects.

Example: `OrderPaid` event arrives twice → invoice generator should not create 2 invoices.

### 2) Retries

Calls can fail transiently (timeouts, 5xx). In async pipelines you will retry.  
So your system must be designed so retries don’t corrupt state (idempotency helps).

Example: email send fails → retry later.

### 3) Compensations

Sometimes a later step fails after an earlier step succeeded, and you **can’t “rollback” across services**.  
So you do a compensation action to restore business correctness.

Example: payment succeeded but inventory reservation failed → you might **refund** payment or mark order as “PAYMENT_OK_BUT_FAILED” and trigger manual fix.

## LLD implications

- If you enforce invariants across aggregates/services → you often accept eventual consistency and use events
- For money/payment-like flows → prefer strong consistency in the aggregate boundary

## Practical examples

- Order placement: strong consistency for order state transitions
- Analytics dashboards: eventual consistency OK

## Interview questions

**Q. What can be eventually consistent in your design and why?**
Things like **analytics, notifications/emails, search indexes, read projections (CQRS summaries), and dashboards** can be eventually consistent because being a few seconds/minutes stale doesn’t break core correctness.

**Q. How do you handle duplicates in eventual flows?**
By using idempotency keys and a dedupe record/unique constraint, handlers can safely ignore replays and avoid double side effects.

A **dedupe record** is a small piece of stored state that says **“I’ve already processed this event/request.”**

**Simple example**
An event comes with `eventId = e-123`.

Your handler does:
1. Check storage (DB/Redis): “Have I processed `e-123`?”
2. If yes → return (do nothing)
3. If no → process it, then insert a record like:

`processed_events(eventId = "e-123", processedAt = ...)`