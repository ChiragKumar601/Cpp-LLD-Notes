# 9.1 Exceptions vs error codes vs `expected<T,E>`-style

## Intuition
You’re designing a **contract**: how failures travel across layers. Pick one style per boundary so callers aren’t confused.

## Concepts (when to use what)

**A) Exceptions**
- Best for: **programmer errors** (violated preconditions), truly exceptional failures, deep call stacks where bubbling is tedious.
- Trade-offs: hidden control flow, must be documented; avoid throwing in destructors.

**B) Error codes / status**
- Best for: **hot paths**, C-style APIs, when you need predictable control flow.
- Trade-offs: caller may ignore; error propagation is noisy.

**C) `expected<T,E>`-style**
- Best for: “failure is part of normal flow” (parse/validate/lookup), _and_ you want compiler-enforced handling.
- Trade-offs: some boilerplate; choose error type carefully.

## One-liner to remember
- **Exception** = emergency stop
- **Status code** = quick yes/no routine response
- **Expected** = outcome + reason packaged together, must be checked

## Architecture rule (LLD-ready)
- **Domain layer**: prefer `expected` (explicit) for business-rule failures
- **Infra layer**: exceptions for unexpected I/O failures OR convert infra failures into `expected` at boundary
- **API boundary**: map errors → stable error response model

## Practical example
- Parsing user input:
    - `expected<Order, ParseError>`
- Null pointer / violated invariant:
    - exception or assertion (depending on policy)

## Real-world use cases
- Parsers, validators, config loaders → `expected`
- DB/network transient failures → retryable exception mapped to domain error

## Interview questions

**Q. Where would you draw the boundary for exceptions vs expected**
We use `expected<T,E>` when failures are **part of normal flow** and must be handled explicitly (like validation/lookup/business rules). We use **exceptions** for **unexpected failures or contract/invariant violations** (often infra/I/O), and we **catch them at a boundary** to translate into a stable error response—rather than terminating the whole program.

**Q. Why are destructors not supposed to throw?**
Because if we throw in a destructor, during stack unwinding it will move the code into except resulting in calling of `std::runtime_error` and also possible memory leaks and undefined behavior 

**Q. How do you ensure errors carry enough context?**

# 9.2 Validation strategy: fail-fast vs accumulate errors

## Intuition

Validation answers: do we stop at the first bad thing, or collect everything wrong?

## Concepts
**Fail-fast**
- Stop on first error
- Best for: internal invariants, performance, security (don’t leak details)
- Simpler control flow

**Accumulate errors**
- Return list of issues
- Best for: user-facing forms, batch imports, config validation
- Requires an error aggregation model

## Architecture

```
Request -> Validator(s) -> Validated Command OR ErrorList
```

## Practical example (pipeline)

- Fail-fast in service:
    - `if (!isAuthorized) return Unauthorized;`

- Accumulate in UI validation: 
    - missing name + invalid phone + invalid pincode


## Real-world use cases

- API request validation: often fail-fast
- Bulk CSV import: accumulate to show all bad rows

## Interview questions

**Q. When is accumulate harmful?**
 
**Q. How do you structure validators (pipeline/CoR)?**
 
**Q. How do you avoid duplicated validation rules?**

# 9.3 Idempotency (critical for real systems)

## Intuition

If a client retries the same request (timeout/network), you must avoid duplicate effects.

**Idempotent operation:** repeating it produces the same final effect.

## Concepts

**Where it matters**
- payments, order creation, refunds
- async jobs and retries
- at-least-once delivery (queues)

**How to implement**
- client sends `idempotencyKey`
- server stores outcome by `(key, operation)` with TTL
- on retry: return same stored result

## Architecture

```cpp
Client -> API (idempotency key)
           |
           v
IdempotencyStore (key -> result/status)
           |
           v
Execute command exactly-once effect (best-effort)
```

## Practical example (order create)
- If `createOrder(key=abc)` is called twice:
    - second call returns same orderId

## Real-world use cases
- Payment gateway calls
- “place order” APIs
- job schedulers re-running tasks after crash

## Interview questions

Q. What store do you use for idempotency? (DB/Redis)
 
Q. What’s TTL strategy?
 
Q. Key scope: per-user vs global?


# 9.4 Retries/backoff (where belongs in design)

## Intuition
Retries are not a “random loop”. They are a **policy** tied to failure types.

## Concepts
**Retry only when**
- failure is transient (timeouts, 5xx, rate limit)
- operation is safe OR you have idempotency

**Backoff**
- exponential + jitter to avoid thundering herd
- cap max delay and max attempts
- **Exponential backoff**: wait 100ms, 200ms, 400ms, 800ms…
- **Jitter**: add randomness so everyone doesn’t retry at the same time
- **Caps**: limit max delay and max attempts so you don’t hang forever

**Where retries belong**
- **Infra boundary**: client adapter (HTTP/DB client) with policy
- **Not in domain logic** (domain shouldn’t know about network flakes)
- **Never retry non-idempotent effects without idempotency**

## Architecture

```
Service -> GatewayAdapter (retry/backoff) -> Remote
```
- **Gateway** = interface/port (what you need)
- **Adapter** = implementation (how you do it with Stripe/SMTP/etc.)
## Practical example
- Payment charge: retries only if gateway says “timeout/unknown” AND you used idempotency key.

## Real-world use cases
- calling downstream microservices
- queue consumers processing transient DB failures

## Interview questions

**Q. What failures are retryable vs not?**

**Q. Why is jitter important?**

**Q. Where do you centralize retry policy?**



# 9.5 Logging design: levels, contexts, correlation ids

## Intuition

Logs are not print statements. They are a **diagnostic contract**.

## Concepts
**Levels**
- `ERROR`: user-impacting failure, needs action
- `WARN`: suspicious but handled
- `INFO`: high-level lifecycle events (request start/end)
- `DEBUG`: internal details for troubleshooting
- `TRACE`: extremely verbose (rare in prod)

**Structured logging**  
Log key-value context (not just strings):
- `requestId`, `userId`, `orderId`, `tenantId`, `component`, `latencyMs`

**Correlation IDs**
In distributed systems, one user request triggers many service calls. A **correlationId** is the thread that stitches them together.
- generate at ingress (API/gateway)
- propagate through calls (headers)
- include in every log line

Note - **Ingress** means the **entry point** where traffic first comes into your system. In real systems
It’s the component that receives the incoming request before it reaches your services, like:
- an **API Gateway**
- a **load balancer** (ALB/NLB)
- a **reverse proxy** (Nginx, Envoy)
- Kubernetes **Ingress Controller**

## Architecture

```
Ingress -> sets correlationId in context
All layers -> log with same correlationId
Downstream -> propagate
```

- **Ingress:** Generate a new `correlationId` (or reuse incoming one) and store it in the per-request context.
- **All layers:** Include the same `correlationId` in every log line for that request (controller → service → repo → client).
- **Downstream:** Propagate the `correlationId` on all outgoing calls (headers) so other services continue the same logging chain.
## Practical example (what to log)
- at API boundary: requestId + endpoint + status + latency
- on error: errorCode + correlationId + relevant IDs (no secrets)

## Real-world use cases
- distributed tracing alignment (logs + traces)
- debugging retries/timeouts
- auditing business events (separate from debug logs)

### Interview questions

**Q. What must you never log? (PII/secrets)**
 
**Q. What’s the difference between audit logs and debug logs?**
 
**Q. How do you tie logs across services?**