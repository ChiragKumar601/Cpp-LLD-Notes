## Intuition
In LLD, performance is about **choosing the right shapes**:
- right data structure
- right ownership model (avoid churn)
- right hot-path behavior (avoid unnecessary virtual/alloc/copies)
- caching with correct invalidation

You should talk in **trade-offs**, not “premature optimization”.

# 11.1 Big-O where it matters in object interactions

## Intuition

Big-O matters when an operation happens:
- per request
- in a loop over many entities
- on a hot path (every event)

## What to evaluate (LLD checklist)

- lookup: `O(1)` hash vs `O(log n)` tree vs `O(n)` scan    
- updates: do you need stable ordering?
- deletes: do you need `O(1)` removal?

## Practical example

**LRU cache** needs:
- `get(key)` in `O(1)`
- `put(key)` in `O(1)`  
    So you choose:
	- `unordered_map<key, node*>` + doubly linked list

## Real-world use cases
- rate limiter per user
- scheduler job lookup by id
- parking lot spot lookup and allocation

## Interview questions

**Q. What operations must be `O(1)` in your design?**
`get()` and `put()` must be **O(1) average** (hash lookup + O(1) list move/insert in an LRU-style design).
 
**Q. Where is the true bottleneck: CPU or I/O?**
 It depends on what the work does: if it’s mostly computation/locking it’s **CPU-bound**; if it waits on DB/network/disk it’s **I/O-bound**—and that decides pool sizing (CPU ≈ cores, I/O can use more threads but needs backpressure).


# 11.2 Allocations & object lifetime: avoid churn (pools, flyweight)

## Intuition

Frequent heap allocations create:
- **latency spikes**: allocator sometimes has to search, split/merge, request pages from OS, etc.
- **fragmentation**: memory becomes scattered; you can have free memory but not in the right shapes.
- **allocator overhead that feels like GC**: even in C++/non-GC systems, the allocator does internal bookkeeping/cleanup that can pause or slow you.

## Practical rules

- prefer stack/value types for small objects
- Keep a set of reusable objects (buffers, request objects). Instead of free-ing, you return them to the pool. This reduces repeated heap work.
- If you have large **immutable** data used by many objects (e.g., config blobs, string tables, parsed schemas), store it once and let many objects reference it, rather than duplicating it.
- batch allocations when possible

## Practical example

- preallocate buffers for network messages(A **buffer** is just a chunk of memory used to **temporarily hold data** while you read it, write it, or process it.)
- reuse DB connections (connection pool)

## Real-world use cases

- high-throughput pipelines (logs/events)
- parsers/encoders in services
- game-like simulations

## Interview questions

**Q. When do you justify object pooling?**
When you’re on a **hot path** and objects are **expensive/frequent to allocate** (big buffers, high QPS, tail-latency sensitive), and reuse is safe—pooling reduces allocation churn and allocator contention

**Q. Flyweight vs pool—difference?**
**Flyweight** shares **immutable data** across many objects to avoid duplication; **Pool** reuses **mutable/temporary objects** by recycling instances instead of allocating/freeing repeatedly.
# 11.3 Copy vs move, pass-by-value vs ref

## Intuition

A lot of “slow” code is accidental copying.
### Copy vs move (what’s the real difference)

- **Copy** duplicates the data (e.g., copying a big `std::string`/`std::vector` can allocate and copy bytes).
- **Move** transfers ownership of the internal resources (e.g., steals the pointer of a vector/string) — usually **O(1)** and avoids allocation/copy.

## Rules (C++ interview-ready)

**For inputs:**
- **Small scalars** (`int`, `double`, small enums) → pass **by value** (cheap).
- **Large objects** (`std::string`, `std::vector`, big structs) → pass **`const T&`** to avoid copying when you only need to read.
- **If your function will store/own the input** → take **`T` by value** and then `std::move` it into your member.  
    This is great because:
    - callers with an lvalue pay **one copy** (expected)
    - callers with an rvalue can **move** (cheap)
    - your implementation stays simple (one function signature)

**For returns:**
- Return **by value**. Modern C++ will use **NRVO** (copy elision) or move; you don’t need `T*`/`T&` returns for performance in most cases.

**For storing:**
- Store by **value** when the object is logically part of your type.
- Store via **`unique_ptr`** when it’s large, optional, polymorphic, or you want stable addresses.
- Store by reference only if you can **guarantee lifetime** (otherwise you create dangling refs).
- An **lvalue** is a value that has a **stable identity/location in memory**, so you can take its address and it lives beyond the current expression.
- An **rvalue** is usually a **temporary** (no stable name), often used for move.

```cpp
int x = 10;        // x is an lvalue
std::string s;     // s is an lvalue

int y = x + 1;          // (x + 1) is an rvalue
setName(std::string("a")); // temporary is an rvalue
```
## Practical example

“sink” pattern:

```cpp
class User {
  std::string name;
public:
  void setName(std::string n) { name = std::move(n); } // caller can move
};
```

## Interview questions

**Q. When do you pass by value intentionally?**
You pass by value when the type is **cheap to copy** (scalars/small structs) **or** when you want a **sink/ownership parameter** (`T` by value) so callers can pass lvalues (copy) or rvalues (move) and you `std::move` into storage.
 
**Q. What does move guarantee? (valid but unspecified state)**
 After a move, the source object remains **valid but in an unspecified state** (destructible and assignable; you can safely reassign/clear it, but shouldn’t rely on its old value).


# 11.4 Hot-path design: virtual dispatch cost vs templates

## Intuition

Virtual calls aren’t “slow” in general, but on extremely hot paths they can matter. This is about choosing **runtime polymorphism (virtual)** vs **compile-time polymorphism (templates)** when performance matters.

### What “hot path” means

A hot path is code that runs **a huge number of times** (millions/sec), so even tiny overheads start showing up in latency/CPU.
## Concepts

- Virtual dispatch: 
  When you call a virtual function, the actual method is picked at **runtime** (via vtable).
    - **Pros:** clean design, easy extensibility (plugins), stable interfaces
	- **Cons (on hot paths):** small extra indirection + often **blocks inlining**, so the compiler can’t optimize as aggressively
- Templates/static polymorphism:
    - compile-time binding, can inline
    - compile time, bigger binaries (code bloat), and it “leaks types” (harder to hide implementation / ABI stability)     

## Interview framing

- default: virtual interfaces for clean LLD
- optimize only if hot path proven
- mention type erasure if you need ABI stability    

## Practical example

- strategy via interface (virtual) is fine 
- if millions of calls/sec and stable types → template policy can help
 

## Interview questions

**Q. When would you replace virtual strategy with template policy?**
When profiling shows the strategy call is on an extreme **hot path** (millions/sec) and the set of concrete strategies is **stable/known at compile time**, so inlining/optimization is worth it.

**Q. What’s the trade-off of templates? (bloat/compile time)**
Templates can be faster via inlining, but trade-offs are **code bloat (bigger binary / i-cache pressure)**, **longer compile times**, and **less runtime flexibility / leaky types** (harder to hide implementation or keep ABI stable). 


# 11.5 Caching strategies at class level (memoization, invalidation)

## Intuition

Caching is easy; **invalidation is the real problem**. The main message is: storing things is easy — keeping them correct is hard.

## Concepts

**Memoization**  
Inside an object, you remember the result of an expensive computation so next time you return it instantly.  
Example: `priceForUser(userId)` computed once → stored in a map.

**Read-through cache**  
Caller asks cache for `key`. If it’s missing, the cache itself loads from source (DB/service), stores it, and returns it.  
So callers don’t write “if miss then load” everywhere.

**Write-through vs write-back**

- **Write-through:** every update writes to DB and cache immediately → simpler correctness, slower writes.

- **Write-back:** update cache now, write to DB later (async) → faster, but riskier (crash can lose updates, needs retry/durability).    

**TTL vs explicit invalidation**

- **TTL:** cache entry expires after time → simple, but can serve stale data until expiry. **Stale data** means the cache is serving an **old value** that is no longer the latest truth in the source (DB/service).

- **Explicit invalidation:** when data changes, you actively evict/update cache (via events/version) → fresher, but more moving parts.

## Correctness rules

- define freshness requirement (stale allowed or not)
- define eviction policy (LRU/LFU/TTL)
- define invalidation triggers (events/version)

## Practical example

- cache product catalog by id with TTL + event invalidation on updates   

## Real-world use cases

- config caching
- computed pricing results
- derived fields (permissions)

## Interview questions

**Q. TTL vs event invalidation—when choose which?**
Use **TTL** when some staleness is acceptable and you want simple “eventual freshness”; use **event invalidation** when you need fresher data on updates (or correctness-critical values) and you can reliably publish update events—often the best answer is **TTL + event invalidation** together.
 
**Q. How do you make cache thread-safe without killing performance?**
Caching is easy; correctness is hard: decide how fresh data must be, how to evict (LRU/LFU/TTL), and how to invalidate (TTL and/or events—often both with TTL as fallback). Use memoization/read-through for fast reads, write-through for correctness, and write-back only with strong crash-safe retry/durability.