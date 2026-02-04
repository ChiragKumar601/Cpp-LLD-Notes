
# 8.1 Concurrency Fundamentals for LLD

## 8.1.1 Thread vs Task (what you should say in interviews)

### Intuition
- **Thread** = OS execution lane
- **Task** = unit of work; can run on a thread pool

In LLD, design around **tasks** and use a **thread pool** to execute them. Threads are a resource.

**Rule:** Don’t spawn a thread per request.


## 8.1.2 Two concurrency styles (choose consciously)

### A) Shared-state + locks (most common)

```vb
multiple threads
   |
   v
shared object + mutex
```

Pros: straightforward  
Cons: contention, deadlocks if sloppy

### B) Message passing (queues, actors)

```vb
Producer threads -> Queue -> Worker thread(s)
```

Pros: less shared state, fewer locks  
Cons: async complexity, queue management

**LLD heuristic:**
- If you can isolate mutation behind a queue/worker → do it.
- If you must share mutable structures → lock them with clear invariants.


## 8.1.3 The 3 safety properties (interview vocabulary)

### 1) Atomicity

“Operation appears indivisible.”
- e.g., `push+pop` must not interleave incorrectly.

### 2) Visibility

One thread’s writes must become visible to another.
- locks and atomics provide this (don’t rely on “it usually works”).

### 3) Ordering

Some operations must happen in a defined order.
- locks/condition_variable define ordering of “wait until state changes”.


## 8.1.4 Invariants (the single best interview habit)

### Intuition

Concurrency is easy when you define what must always be true.
Example: bounded queue invariant

- `0 <= size <= capacity`
- If `size == 0`, consumers must wait
- If `size == capacity`, producers must wait (or fail fast)

**Design rule:**

> Every shared object must state its invariant + the lock that protects it.


## 8.1.5 Minimal “Thread-safe object” template (C++)

### Architecture
```cpp
class X
  - mutex m
  - state S  (protected by m)
  - public methods lock m, check/modify S, unlock
```

### Practical Example: thread-safe counter (mutex)

```cpp
class Counter {
    std::mutex m;
    long long value = 0;   // invariant: value modified only under m
public:
    void inc() { std::lock_guard<std::mutex> lg(m); ++value; }
    long long get() const {
        // const method still needs synchronization
        std::lock_guard<std::mutex> lg(m);
        return value;
    }
};
```

**LLD note:** If it’s just a counter → prefer `std::atomic<long long>` (in 8.2).

---

## 8.1.6 How to add concurrency to any LLD (the playbook)

When asked “make it thread-safe / scalable”, do this:
1. **List shared state**
- cache map, queue, balances, slots, etc.

2. **Define invariants**
- e.g., “spot can’t be assigned to 2 vehicles”

3. **Choose synchronization**
- single mutex (small system)
- striped locks / RW lock (read-heavy)
- queue + single writer (high correctness)

4. **Pick lock granularity**
- coarse lock: simpler, less bug risk
- fine lock: more throughput, more complexity

5. **Never hold locks across**
- network calls, DB calls, disk I/O, callbacks


## 8.1.7 Interview-level practice questions

**Q. For an LRU cache, what’s the shared state? What lock strategy would you use?**
For an LRU, the shared state will be the unordered_map of key and Node. For this one mutex per key value pair will be the best.

**Q. For a rate limiter, what can be atomic and what needs a mutex?**
For a rate limiter either a call happens or it does not, so function calling the API will be atomic, and the number of hits in a given time frame needs a mutex.

**Q. For a parking lot, how do you prevent two threads from assigning the same spot?**
For sliding-window rate limiting, the shared state is a `unordered_map<userId, deque<timestamp>>` (or ring buffer) plus any window metadata. Since unordered_map and the per-user container aren’t thread-safe, the whole sequence—evict old timestamps → count → allow/reject → append now—must be done atomically under a lock. For scalability, don’t use one global mutex; use striped locks `(locks[hash(userId)%N]) `or a per-user lock to reduce contention.

**Q. For an observer/event bus, what’s the thread-safety story for subscribe/unsubscribe/notify?**
Protect the per-topic subscriber registry with a mutex/RW-lock, make `subscribe/unsubscribe` write-locked updates, and implement `notify` as **lock → snapshot subscribers → unlock → invoke callbacks** (never call user code while holding locks), optionally using per-topic/striped locks for scale.

**Q. How do you avoid deadlocks when multiple locks exist?**
Enforce a **global lock ordering** (or lock by increasing stripe/key order / use `std::scoped_lock`), keep critical sections small, and **never hold locks across callbacks or I/O** to avoid re-entrancy and lock-order inversion.

#  8.2 C++ Concurrency Primitives (what to use, when, and why)

## 8.2.1 `std::mutex` + `std::lock_guard`

**Intuition:** one lock protects one shared invariant.  
**Use when:** you need **exclusive** access.

```cpp
class SafeCounter {
    mutable std::mutex m;
    long long v = 0;               // invariant: protected by m
public:
    void inc() { std::lock_guard<std::mutex> lg(m); ++v; }
    long long get() const { std::lock_guard<std::mutex> lg(m); return v; }
};
```

**Rule:** keep lock scope tiny; never hold across I/O.


## 8.2.2 `std::unique_lock` (more flexible lock)

**Intuition:** lock you can unlock/lock manually; required for `condition_variable`.  
**Use when:** waiting, timed locking, or early unlock.

```cpp
std::unique_lock<std::mutex> lk(m);
// ... maybe unlock before slow work
lk.unlock();
```


## 8.2.3 `std::condition_variable` (wait for state change)

**Intuition:** “sleep until predicate becomes true”.  
**Rules (interview-critical):**
- always wait in a **loop** with a predicate (spurious wakeups)
- predicate checks the shared state **under the same mutex**

```cpp
cv.wait(lk, [&]{ return !q.empty() || stop; });
```


### 8.2.4 `std::shared_mutex` (RW lock)

**Intuition:** many readers, few writers.  
**Use when:** read-heavy maps/caches/config.

```cpp
mutable std::shared_mutex sm;

{ std::shared_lock lock(sm);  /* read */ }
{ std::unique_lock lock(sm);  /* write */ }
```

**Caveat:** can cause writer starvation depending on workload.


## 8.2.5 `std::atomic<T>` (lock-free for simple state)

**Intuition:** safe concurrent reads/writes for _single_ values.  
**Use when:** counters, flags, sequence numbers.

```cpp
std::atomic<long long> hits{0};
hits.fetch_add(1, std::memory_order_relaxed);
```

**Rule:** atomics don’t magically protect multi-step invariants  
(e.g., “check then insert” on a map still needs a lock).

## A simple decision cheat-sheet

- Protect **one shared object/invariant** → `mutex + lock_guard`
- Need **wait/notify** → `unique_lock + condition_variable`
- Mostly **reads**, few writes → `shared_mutex`
- Single numeric/flag state → `atomic`

**Interview Qs (8.2):**

**Q. `lock_guard` vs `unique_lock`?**
**`lock_guard`** is a lightweight RAII lock: it **locks in the constructor** and **automatically unlocks when it goes out of scope**, so it’s best when you just need simple scoped exclusive access.  
**`unique_lock`** is more flexible (and slightly heavier): it’s **movable**, you can **unlock early and relock later**, and it supports **`try_lock` / timed locks**—also it is **required for `condition_variable::wait()`** because wait needs to temporarily release and reacquire the mutex.

**Q. Why use `cv.wait(lk, predicate)` and not `cv.wait(lk)`?** 

**Q. When is `atomic` enough vs mutex required?**
**Atomic is enough** when the shared state is a **single variable** and the operation is a **single atomic read-modify-write** (counter/flag); **mutex is needed** when you must keep a **multi-step invariant** consistent (check-then-act, multiple fields together, or container updates).
- Atomic = “single independent update”
- Mutex = “needed when updates involve multiple steps or shared data structures”
 


# 8.3 Common Concurrency Bugs (and the design fixes)

## 8.3.1 Data race

**What:** two threads access same memory, at least one write, without sync. 

**Fix:** define the invariant + one lock (or atomic) that protects it.


## 8.3.2 Deadlock

**What:** circular wait on locks.
**Fix patterns (say these in interviews):**
1. **Global lock ordering** (always acquire A then B)
2. Use `std::scoped_lock` / `std::lock` to lock multiple safely:

```cpp
std::scoped_lock lk(m1, m2);
```

3. **Avoid nested locks** by redesign (single owner, message passing)
4. Use timeouts (`try_lock_for`) only when you have a recovery policy


## 8.3.3 Livelock

**What:** threads keep retrying and making no progress (polite spinning).  
**Fix:** backoff/jitter, or switch to blocking waits.



## 8.3.4 Starvation

**What:** a thread never gets CPU/lock/time.  
**Fix:** fairness policies, bounded queues, avoid reader-dominant RW locks if writers must progress.



## 8.3.5 “Lock held while doing slow/unsafe work” (most common real bug)

**Examples:** holding lock while:
- network/DB/disk I/O
- calling user callbacks
- logging with slow sink

**Fix:** snapshot under lock, then act without lock:

```cpp
// lock -> snapshot -> unlock -> call callbacks
```

## 8.3.6 Real life analogies of common concurrency bugs

### Data race

**Analogy:** Two people editing the **same Google Doc line** at the same time without coordination.  
One types “12”, the other types “13”, and the final text becomes “1” or some broken mix.

**Key idea:** shared thing + at least one writer + no coordination → corrupted result.

### Deadlock

**Analogy:** Two people in a narrow hallway.
- Person A holds Door Key 1 and needs Door Key 2.
- Person B holds Door Key 2 and needs Door Key 1.  
    Both refuse to let go → nobody moves forever.

**Key idea:** circular waiting for resources.


###  Livelock

**Analogy:** Two polite people trying to pass each other in a corridor.  
Both step left at the same time, then both step right at the same time… forever.  
They’re “active” but still not progressing.

**Key idea:** not blocked, but stuck in repeated retries/avoidance.


### Starvation

**Analogy:** A queue where VIPs keep cutting in front.  
One normal person keeps waiting and waiting and never gets served.

**Key idea:** someone never gets access to CPU/lock/time because others dominate.


### Holding a lock during slow/unsafe work

**Analogy:** Only one cashier has the key to the cash drawer.  
They take the key, then go to the back room to check inventory (slow), while everyone else waits—even though they only needed the drawer for 2 seconds.

**Key idea:** you held the “key” while doing unrelated slow work → everyone stalls.



## Interview Qs (8.3):

**Q. How do you avoid deadlocks when multiple locks exist?**
 
**Q. Why should you not hold locks across callbacks/I/O?**
 
**Q. Difference: deadlock vs livelock vs starvation?**
 

---

## **8.4 Thread-safe Data Structures (LLD templates you reuse)**

### 8.4.1 Bounded Blocking Queue (Producer/Consumer)

**Invariants:** `0 <= size <= cap`
- if empty → consumers wait
- if full → producers wait (or reject)

`template <typename T> class BlockingQueue {     std::mutex m;     std::condition_variable notEmpty, notFull;     std::deque<T> q;     const size_t cap;     bool closed = false;  public:     explicit BlockingQueue(size_t capacity) : cap(capacity) {}      bool push(T item) {         std::unique_lock<std::mutex> lk(m);         notFull.wait(lk, [&]{ return closed || q.size() < cap; });         if (closed) return false;         q.push_back(std::move(item));         notEmpty.notify_one();         return true;     }      bool pop(T& out) {         std::unique_lock<std::mutex> lk(m);         notEmpty.wait(lk, [&]{ return closed || !q.empty(); });         if (q.empty()) return false;           // closed + empty         out = std::move(q.front());         q.pop_front();         notFull.notify_one();         return true;     }      void close() {         std::lock_guard<std::mutex> lg(m);         closed = true;         notEmpty.notify_all();         notFull.notify_all();     } };`

**Use cases:** thread pools, async logging, pipelines, schedulers.

---

### 8.4.2 Concurrent Map Patterns (choose by load)

**A) Small system:** single mutex

`map + mutex`

**B) Read-heavy:** `shared_mutex`
- `shared_lock` for reads, `unique_lock` for writes

**C) High contention:** striped locks (common interview answer)
- split into N shards by hash
- lock only the shard

`shard = hash(key) % N locks[shard] protects maps[shard]`

**Why stripes work:** reduce contention without complex lock-free code.

**Interview Qs (8.4):**

1. Blocking queue invariants + why predicate wait?
    
2. How would you scale an `unordered_map` under concurrency?
    
3. RW lock vs striped locks—when to pick which?
    

---

## **8.5 Thread Pool + Executor (how services run work)**

### Intuition

A thread pool is:

- **N worker threads**
    
- a **(bounded) work queue**
    
- a **submit() API**
    
- a **shutdown policy**
    

**Goal:** avoid “thread per request”, provide throughput + backpressure.

---

### Architecture

`Producers (requests)       |       v  [ Bounded Queue ]  <-- backpressure point       |       v Worker threads (N) -> execute tasks`

### Backpressure policies (must mention)

- **Block** caller until space (default for internal systems)
    
- **Reject** with error (good for APIs)
    
- **Drop** (telemetry/logs sometimes)
    
- **Caller-runs** (push work back to caller thread)
    

---

### Minimal C++ ThreadPool (reuses BlockingQueue)

`class ThreadPool {     BlockingQueue<std::function<void()>> q;     std::vector<std::thread> workers;  public:     ThreadPool(size_t threads, size_t queueCap) : q(queueCap) {         workers.reserve(threads);         for (size_t i = 0; i < threads; ++i) {             workers.emplace_back([this] {                 std::function<void()> task;                 while (q.pop(task)) { task(); }             });         }     }      bool submit(std::function<void()> task) {         return q.push(std::move(task));   // false if closed     }      void shutdown() {         q.close();                        // stop pops         for (auto& t : workers) if (t.joinable()) t.join();     }      ~ThreadPool() { shutdown(); } };`

**Graceful shutdown options (interview-ready):**

- **Drain then stop:** stop accepting new tasks, process remaining, then exit
    
- **Stop immediately:** close queue and drop pending tasks (must be explicit)