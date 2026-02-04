## Intuition

Interviewers reward designs that are:
- easy to unit test
- have clear boundaries (what to mock vs test real)
- make correctness provable via invariants

Testability is not “extra”—it’s a sign your design has clean dependencies.


# 12.1 Designing for tests: seams, interfaces, dependency injection

## Intuition

A **seam** is a place where you can substitute a real dependency with a fake.

## Concepts (SDE2/3 rules)

- Depend on **interfaces**, not concrete types (DIP)
- Inject dependencies via **constructors**
- Keep object creation in a **composition root** (main/bootstrap)

## Practical example

```cpp
class CheckoutService {
  IPaymentGateway& gw;
  IOrderRepo& repo;
public:
  CheckoutService(IPaymentGateway& g, IOrderRepo& r) : gw(g), repo(r) {}
};
```

Now you can inject `FakePaymentGateway` in tests.

## Real-world use cases
- swap DB with in-memory repo    
- swap clock with deterministic clock
- swap random generator with fixed seed

### Interview questions

**Q. What seams exist in your design?**
 Seams are the dependencies I can swap in tests: payment gateway, order repo/DB, clock/time source, random/UUID generator, external clients (HTTP), and even queue/executor.
 
**Q. Where do you create concrete objects?**
Concrete objects are created in the **composition root** (main/bootstrap) and injected into services, not constructed inside business logic.


# 12.2 Unit vs integration boundaries (what to mock)

## Intuition

**Mocking** means replacing a real dependency (DB/HTTP/clock) with a **fake object** in a test, so you can control its behavior and verify how your code interacts with it.

Mocking everything makes tests meaningless. Mocking nothing makes tests slow/flaky.

## Rule of thumb
- **Unit tests**: test business logic; mock external I/O boundaries
- **Integration tests**: test adapters with real DB/network (or containerized)
- Keep the number of integration tests smaller but high value

## What to mock (good default)
- network clients (HTTP)
- time (`IClock`)
- randomness (`IRng`)
- message brokers
- filesystem

## What not to mock (usually)
- value objects
- pure functions
- in-memory domain logic
Because these are the things you _actually want to test_—they’re fast and deterministic already.

## Practical mental model

Unit test = “Given inputs, do we make the right decision?”  
Integration test = “Does our code actually talk to DB/network correctly?”

## Interview questions

**Q. What is a unit test for your service? What’s mocked?**
Unit test is about testing business logic by mocking external I/O boundaries.
 
**Q. What’s one integration test you’d definitely write?**
 An end-to-end test of the **adapter + real dependency**—e.g., the repository against a real Postgres (container) to verify schema/mappings/transactions (or the HTTP client against a stub server to verify request/response + retries/timeouts).


# 12.3 Fake vs mock vs stub in C++

## Intuition

They’re different tools; knowing the difference is senior-signal.

## Concepts

- **Stub**: gives **canned answers** so your code can run. You don’t care how it was called, only that it returns something predictable.  
    Example: `ClockStub` always returns `10:00 AM`.

- **Fake**: a **working but simplified** implementation, usually in-memory. Good when you want real behavior without real infra.  
    Example: `InMemoryOrderRepo` using `std::unordered_map`.

- **Mock**: used to **verify interactions**—that some method was called, with certain args, maybe in order.  
    Example: `PaymentGatewayMock` asserts `charge(500)` called once.

## Practical example
- Stub: `ClockStub` returns fixed time
- Fake: `InMemoryOrderRepo`
- Mock: `PaymentGatewayMock` asserts `charge()` called once

## Interview questions

**Q. When do you prefer fake over mock? (most of the time)**
Most of the time—when you want to test real behavior cheaply (e.g., an in-memory repo) and avoid coupling the test to exact call sequences.


**Q. What’s the risk of over-mocking? (brittle tests)** 
Tests become **brittle** and implementation-coupled: small refactors change interactions and break tests even though behavior is still correct, and you may end up “testing the mock” instead of real logic.


# 12.4 Contract tests for interfaces

## Intuition

If multiple implementations exist (plugins/adapters), you need a shared behavioral contract.
That is, When you have an interface with multiple implementations (S3 vs local FS vs Azure, or different payment providers), you want to ensure they all behave the same **for the behaviors you rely on**.
## Concepts

- Define a test suite that any implementation must pass    
- Run same tests against:
    - real implementation
    - fake implementation
    - alternative vendor adapter

## Practical example

Example for `IStorage` contract:
- `put(k,v)` then `get(k)` returns `v`
- overwrite behavior is defined (replace vs reject)
- missing key behavior is defined (null/exception)

## Interview questions

**Q. What are 3 contract tests for your key interface?**
- `put(k,v)` then `get(k)` returns `v`
- overwrite behavior is defined (replace vs reject)
- missing key behavior is defined (null/exception)
 
**Q. How do you ensure all plugins obey them?**
It’s correct and concise. To make it 10/10, add one small detail: you run the same suite **against each implementation in CI** so no new plugin can ship without passing.
**CI** means **Continuous Integration**. It’s an automated system (like GitHub Actions, Jenkins, GitLab CI) that runs whenever you push code or open a pull request.

# 12.5 Property-based thinking (invariants & edge cases)

## Intuition

Instead of testing specific examples only, test **properties** that must always hold.

## Concepts
- invariants: must be true for any valid sequence of operations
- generate many random sequences (even manually in interviews, describe it)

## Practical examples (properties)

**LRU cache**
- size never exceeds capacity
- most recently accessed key is at front
- if key exists, get returns latest value

**Rate limiter**
- never allows more than limit in window
- monotonic time does not decrease counts incorrectly

**Note -** **Monotonic time** means a clock that **only moves forward**—it never goes backward, even if the system clock is changed.

Why it matters for a rate limiter: if you use the normal wall clock and the time jumps backward (NTP correction, manual change), your window calculations can break (you might “reset” too early or count incorrectly). A monotonic clock avoids that because elapsed time is always non-negative.


## Interview questions

**Q. Give 2 invariants for your design.**
- **Capacity invariant:** `0 <= size <= capacity` (never overflow/underflow).
- **Exactly-once effect per requestId:** the same request/event is processed idempotently (no duplicate side effects).

**Q. What edge cases break most systems? (timeouts, retries, duplicates, clock skew)**
Timeouts and partial failures, retries causing duplicates, out-of-order delivery, and clock issues (clock skew / non-monotonic time) that break TTL/windows.