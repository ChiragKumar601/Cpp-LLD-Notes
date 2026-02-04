
# 10.1 Threads, tasks, thread pool design basics

## Intuition

A **thread** is an OS execution resource. A **task** is a unit of work.  
**Key line:** _“Threads are workers. Tasks are jobs.”_

In systems, you scale by **pooling threads** and scheduling **tasks**, not by spawning threads per request.

Why it’s bad:
- **Memory blowup:** each thread needs stack + OS bookkeeping → can hit OOM.
- **Context switching:** CPU wastes time switching between too many runnable threads.
- **Scheduling chaos:** latency gets worse, throughput drops, system can stall/crash.

## Concepts
- **Thread-per-request** ⇒ unbounded threads ⇒ context switching + memory blowups.
- **Thread pool** ⇒ fixed workers + bounded queue ⇒ throughput + backpressure.
- **Backpressure** is a design feature: block/reject/drop/caller-runs.

### One-line mental model

**Thread-per-request** = “unbounded workers” → crash under spikes.  
**Thread pool + bounded queue** = “bounded workers + bounded waiting room” → stable system.  
**Backpressure** = “what do we do when the waiting room is full?”

## Architecture (ASCII)

```bash
Clients -> submit(task) -> [ Bounded Queue ] -> Worker Threads (N) -> execute
                         (backpressure here)
```

## Practical example
- Async logging: producers push log events into queue; workers flush to sink.    
- API handler: enqueue heavy work; return `requestId` quickly.

## Real-world use cases
- job schedulers, async pipelines, background processing
- fan-out work (thumbnail generation, enrichment)

## Interview questions

**Q. Where do you enforce backpressure?**
Backpressure is enforced at the **work-admission boundary** (enqueue/submit into a bounded queue or acquire a permit), where overload triggers block/reject/drop/caller-runs.

**Q. What is graceful shutdown policy (drain vs drop)?**
- **Drain (graceful):** stop accepting new tasks, let workers finish **in-flight + queued** tasks, usually with a **timeout**.
- **Drop (fast shutdown):** stop accepting, **discard queued** tasks immediately, optionally cancel/interrupt in-flight, exit quickly.
- Rule of thumb: **drain for business-critical jobs**, **drop for best-effort work** (logs, telemetry).
 
**Q. Why avoid thread-per-request?**
- It creates **unbounded threads** as traffic spikes → **stack/OS overhead** → OOM risk.
- Too many runnable threads cause **context-switch thrash** → CPU wasted → latency/tail latency explodes.
- You lose **control**: no clear cap, no backpressure; a **thread pool bounds concurrency** and keeps the system stable.
 

# 10.2 Data race vs race condition

## Intuition

They sound similar but are different “bug classes”.
A **race condition** is when the _result depends on timing/interleaving_ of two actions; a **data race** is a specific kind of race condition where two threads access the **same memory location concurrently**, and at least one access is a **write**, without proper synchronization.

## Concepts

**Data race**

- unsynchronized concurrent access to same memory, at least one write
- causes **undefined behavior** in C++    

**Race condition**
- outcome depends on timing/order of events
- can happen even without a data race (e.g., bad check-then-act across threads with separate locks)

**Real-life analogy (kitchen whiteboard):**
- Two chefs update the same “remaining orders” number on a whiteboard.
- **Data race:** both write the number at the same time (no “only one person writes at a time” rule). The number can become garbage/incorrect because the writes collide.
- **Race condition (broader):** even if only one chef writes at a time, the workflow can still be wrong due to timing. Example: Chef A reads “2 orders left” and goes to cook; before he marks it, Chef B also reads “2 orders left” and cooks the same order → duplicate work. No write collision necessarily, but timing caused a wrong outcome.

**Quick memory hook:**  
Data race = _unsynchronized concurrent access to the same data._  
Race condition = _timing-dependent bug in logic (may or may not involve shared memory writes)._

## Practical example
- Data race: two threads write `x++` without lock/atomic.
- Race condition: “check slot free” then “assign slot” without atomicity → two assign same slot.

## Interview questions

**Q. Can you have a race condition without a data race? (Yes)**
Yes — a race condition can happen **without a data race** because a race condition is about **timing/order of operations**, while a data race specifically requires **unsynchronized concurrent access to the same memory location** (with at least one write).
 
**Q. Why are data races worse in C++?** 
Because in C++ a data race is **undefined behavior** under the language memory model — the compiler can assume it never happens and reorder/optimize aggressively, so you don’t just get “wrong values”; you can get totally unpredictable behavior.
 


# 10.3 Mutex / RWLock / Spinlock — when to use what

## Intuition

Pick the simplest tool that protects your invariant with acceptable contention.

## Concepts
**Mutex (`std::mutex`)**
- default choice for most shared invariants
- best general-purpose lock

**RW Lock (`std::shared_mutex`)**
- many readers, few writers
- but beware: “reads” that mutate (LRU `get()` updates recency) are **writes**

**Spinlock (conceptual)**
- busy-waits instead of sleeping
- only good when:
    - critical section is extremely tiny
    - contention is low
    - you’re on CPU-bound, low-latency code
- in interviews: mention sparingly; prefer mutex unless you can justify.

Imagine a **library** with a **single rare book register** (the shared state).
- **Mutex:** One person at a time can use the register—whether they’re reading entries or writing a new entry.
- **RW lock:** Many people can **read** the register at the same time, but if anyone needs to **write/update**, everyone else must stop and wait.
- **Spinlock:** You don’t take a token and wait; you just **stand at the counter staring** and checking every second if it’s free—fast if it frees immediately, terrible if it takes time (you waste effort).

### Practical example
- Config map: RW lock (read-heavy)
- LRU cache: mutex or sharded mutexes (because `get()` mutates)

## One-liners (interview-ready)
- **Mutex:** “Default lock—one thread at a time protects the invariant, simple and reliable.”
- **RW lock:** “Use when data is truly read-mostly; many readers in parallel, writers are exclusive.”
- **Spinlock:** “Busy-wait lock—only for ultra-short critical sections with low contention; otherwise wastes CPU.”

### Interview questions

**Q. Why is RW lock bad for LRU `get()`?** 
Because `get()` typically **mutates** the cache (updates recency / moves the node), so it’s effectively a **write**, meaning you can’t benefit from shared-read concurrency and RW locks add overhead without gain.
 
**Q. When is spinlock worse than mutex? (wastes CPU)**
When the lock can be held for more than a tiny time or contention is non-trivial (or the holder can be preempted/blocked) — spinning just **burns CPU** while a mutex would sleep and let others run.



**Note -** 
* **Trivial/low contention:** most of the time a thread grabs the lock immediately.
- **Non-trivial contention:** threads often find the lock already held and have to wait (queues form), causing delays and CPU waste (especially bad for spinlocks).
 


# 10.4 Condition variables & producer-consumer

## Intuition

Condition variables let threads **sleep** until a predicate becomes true—no polling.
In this context, a **predicate** is just a **boolean condition** that tells you “is it safe to continue?”

## Concepts (must-say rules)

- always wait with a **predicate** (spurious wakeups)
- predicate checks shared state under the same lock
- notify after state change

## Architecture

```scss
Producer -> push -> notify(notEmpty)
Consumer -> wait(notEmpty) -> pop -> notify(notFull)
```

## Practical example (bounded buffer)
- producers wait when full
- consumers wait when empty

## Interview questions

**Q. Why `cv.wait(lk, pred)` not `cv.wait(lk)`?**
`cv.wait(lk, pred)` is just the safe, built-in version of that loop. It guarantees you proceed only when `pred()` is true (checked under the lock).

**Q. What happens on shutdown? (close flag + notify_all)**
 On shutdown, you set a shared **`closed` flag** under the lock and then **`notify_all()`** to wake any threads stuck in `wait()`.  
Each woken thread re-checks the predicate and sees `closed==true`, so it **exits cleanly** (consumers stop waiting for items, producers stop waiting for space).

# 10.5 Lock ordering, deadlock prevention

## Intuition

Deadlocks happen when threads acquire locks in inconsistent order.

## Concepts

**Prevention toolkit**

- **Global lock ordering:** decide a universal order (e.g., `AccountLock -> LedgerLock`) and **never violate it** anywhere in code. This is the most “SDE3 clean” answer because it scales across teams.

- **`std::scoped_lock(m1, m2)`**: if you must take multiple locks, use it to lock them together safely (it uses deadlock-avoidance internally instead of you doing `lock(); lock();` manually).
 
- **Reduce the lock graph:** avoid nested locks and avoid calling external code while holding a lock (callbacks/IO/other services) because you don’t control what that code will try to lock → surprise cycles.

- **Timeouts only with a plan:** `try_lock` loops without a recovery policy often become “busy-wait with bugs.” If you do timeouts, you need a deterministic fallback (abort transaction, retry with backoff, etc.).

### Practical example

- Two resources: `AccountLock` then `LedgerLock` everywhere.

## Interview questions

Q. Give a lock ordering rule in your design.
I enforce a **global lock order** (e.g., `AccountLock -> LedgerLock`) and never acquire locks out of order; if I must take multiple, I use `std::scoped_lock(m1, m2)`.
 
Q. Why not hold locks while calling external callbacks?
 Because you don’t control what locks that code might take (can create **deadlock cycles**), and it can also block on slow work/IO, **holding the lock longer** and killing concurrency.

# 10.6 Atomic basics & memory ordering (LLD-relevant subset)

## Intuition
Atomics are for **single-value state**. Locks are for **multi-step invariants**.

## Concepts

**Use atomics for**
- counters (`requests++`)    
- flags (`closed=true`)
- sequence numbers (`nextId.fetch_add(1)`)

**Don’t use atomics for**
- container operations (push/pop/map insert) unless the structure is designed for concurrency
- “check then act” across multiple values (e.g., `if (x) { y++; z--; }`)

**Memory ordering (what’s enough for LLD)**

**Memory ordering** is the rule that defines **when writes done by one thread become visible to another thread, and in what order** those reads/writes are observed across threads.

- `relaxed`: counters/metrics where you only care about the number, not ordering.
- default `seq_cst`: safest if you’re not deliberately doing a publish/subscribe pattern.
- `release/acquire`: for “publish data then flip flag” patterns (mention only if interviewer goes deep). 

## Interview questions

**Q. Give an example where atomic is not enough.**
When you need a **multi-step invariant** across multiple values, e.g. a bounded queue: `if (size < cap) { push(item); size++; }` — making `size` atomic doesn’t make the **check-then-act + container update** atomic; you need a lock (or a designed concurrent structure).

**Q. When is relaxed ordering ok? (metrics counters)**
In case of counters/metrics where you only care about the number, not ordering.


# 10.7 Concurrent data structures (safe queues, bounded buffers)

## Intuition

Most interview DS are:
- thread-safe queue
- bounded buffer
- concurrent map with striped locks
- Most interview DS problems boil down to “protect an invariant correctly.”


## Concepts

Two big rules:

1. **Protect the invariant with one lock** (simple and correct). Example: for a queue, invariant is `items` + `size` consistency.

2. **Don’t hold locks while executing tasks/callbacks.** If you pop a task from a queue, release the lock first, then run it—otherwise one slow task blocks everyone.    

For scalability, use **sharding / striped locks**:

- Instead of one lock for the whole map, do `N` shards: `shard = hash(key) % N`
- Each shard has its own lock, so unrelated keys don’t contend.


## Architecture (striped map)

```bash
shard = hash(key) % N
locks[shard] protects maps[shard]
```

## Interview questions

**Q. Single mutex vs striped locks—trade-offs?**
A single mutex is easiest and least bug-prone, but if many threads hit it, everything waits and it becomes slow. Striped locks split the work across multiple locks so more threads can work in parallel, but it’s more complex (need hashing, hot keys can still bottleneck, and operations touching multiple keys are harder).
 
**Q. What is the invariant of your queue/buffer?**    
That the internal structure is consistent: `0 <= size <= capacity`, and the queue contents match `size` (push adds exactly one element, pop removes exactly one) with correct ordering (FIFO) and no lost/duplicated items.

# 10.8 Designing state machines under concurrency

## Intuition

State machines break under concurrency when two threads:
- read the same state
- both apply a transition, causing double-effects (double-ship, double-refund, illegal state).

## Concepts
**Option A: Single lock around transitions (most common)**
- protect `state` + related data with one mutex
- validate “is this transition allowed?” inside the lock; apply transition; record effects

**Option B: Single-writer (event loop) model**
All events go into a queue; exactly one worker applies transitions in order. This is often the easiest way to get correctness at scale, and it naturally avoids races.

A crucial SDE3 add-on: **separate state transition from side effects** (e.g., write “SHIPPED” + an “emit shipment event” record atomically, then perform external calls outside the lock / via worker). This avoids “state changed but side effect failed” inconsistencies.

### Architecture (single-writer recommended)

```
Events -> Queue -> StateMachineWorker -> apply transition -> emit effects
```

### Practical example

Order lifecycle:

- `CREATED -> PAID -> SHIPPED`  
    Prevent double-ship:
- transition guarded under lock OR single-writer event loop

### Interview questions

Q. How do you prevent two threads from shipping the same order?
 
Q. Lock-based vs event-loop approach—when choose which?