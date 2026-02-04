## Intuition

At SDE-3, LLD is not just “classes.” It’s:
* Boundaries : You split the system into clear parts: “core business logic” vs “outside world” (DB, HTTP, Kafka, filesystem). Each part has a clear responsibility.

* Dependency direction : Your core logic should not depend on concrete tech (Postgres, Redis, Stripe SDK). Instead it depends on **interfaces**. The outside layer depends on the core, not the other way around. (This is DIP in action.)

* Testability : Because core depends on interfaces, you can test core logic with **fakes/mocks**—no real DB/network required.

* Change isolation : When something changes outside (new DB, new payment provider, new API), you only change the **adapter layer**, not the domain logic.

* Deployment/infra effects without leaking into domain : Infra concerns like retries, timeouts, tracing, caching, circuit breakers, connection pools belong in adapters/clients/config, not sprinkled inside business rules. The domain stays clean and deterministic.


**Overall goal**  
Keep the **core logic stable and boring**, while allowing the outer world (DB/HTTP/queues/vendors) to evolve freely.

Real example:  
If tomorrow you switch from **Postgres → DynamoDB**, only `OrderRepoDynamo` changes; `CheckoutService` stays the same.

# 13.1 Layered architecture (domain/application/infra separation)

## Intuition

Layering prevents “everything depends on everything.”  
Each layer has one job and dependencies flow inward.

## Concepts (3-layer minimal model)

1. **Domain**: entities, value objects, invariants, policies (no DB/HTTP). **Rule:** Domain should not know Postgres, HTTP, Kafka, Redis, timeouts, etc.

2. **Application**: use-cases orchestration (calls domain + ports). This layer answers: **“What happens when user does X?”** It contains flow logic, not heavy business rules.
 - coordinates steps: validate → call domain rules → persist → publish event
- depends on **ports/interfaces** like `IOrderRepo`, `IPaymentGateway`

3. **Infrastructure**: adapters (DB, HTTP clients, messaging, caching, retries, logging, tracing). It’s allowed to change often, because vendors and infra evolve.

## Dependency rule (interview must-say)

> Dependencies point inward: Infra → App → Domain

## Architecture (ASCII)

```less
[ Infrastructure ]
   (DB, HTTP, MQ)
         |
         v
[ Application / Use Cases ]
         |
         v
[ Domain Model ]
(Entities, VOs, invariants)
```

## The key interview rule: “dependencies point inward”

**Infra → App → Domain**

Meaning:
- Domain depends on nothing else.
- Application depends on Domain + interfaces (ports).
- Infra depends on Application/Domain to implement those ports.

So if you swap Postgres → Mongo, only Infra changes.


## Practical example

`PlaceOrderUseCase` (Application) does:
1. create/validate order using Domain rules
2. call `IOrderRepo.save(order)` (interface)
3. call payment policy / gateway through an interface

`PostgresOrderRepo` (Infrastructure) is the concrete class that implements `IOrderRepo` using SQL.

## Real-world use cases

- swapping DB vendors
- adding caching without touching domain
- testing domain without infra

## Interview questions

**Q. What goes in domain vs application?**
Domain contains invariants, entities, value objects and policies, whereas application layer answers - "What should happen when users perform a certain action?". It basically contains of flow logic, not heavy business rules.

**Q. Can domain depend on repository?**
No—domain depends on abstraction/port only, usually from app boundary


# 13.2 Hexagonal (Ports & Adapters) in C++

## Intuition

Hexagonal (Ports & Adapters) is a way to design so your **core logic stays stable**, and everything that talks to the outside world is **replaceable**.

## The core idea

Your system has a **core** (domain + use cases). The core should not care whether data comes from Postgres, Mongo, an HTTP service, or a mock. So the core talks to the outside world only through **ports** (interfaces). The outside world connects through **adapters** (implementations).

## Concepts

- **Port**: interface required by core (e.g., `IOrderRepo`, `IPaymentGateway`)
- **Adapter**: concrete implementation (Postgres repo, Stripe gateway)
- Core doesn’t know which adapter is used.

## Architecture (ASCII)

```
UI/API  ->  Application/Core  ->  Domain
                 |
                 v
               Ports  <-  Adapters (DB/HTTP/MQ)
```
## C++ mapping (simple)

- Ports: `struct IOrderRepo { ... }`
- Adapters: `class PostgresOrderRepo : public IOrderRepo { ... }`
- Wiring: done in main/composition root

## Real-world use cases
- clean dependency injection
- plugin architecture
- microservices adapters

## Interview questions

**Q. Why is this better than calling DB from domain classes?**
Because it keeps domain logic **pure and testable**, avoids coupling business rules to a specific DB/ORM, and isolates change so swapping DB/caching/retries affects only adapters, not core rules.
 
**Q. Where do ports live (domain/app)? (usually app boundary or domain boundary; keep core stable)**
Ports live at the **core boundary**—most commonly in the **application layer** (use-case needs), sometimes in the **domain boundary** if it’s a true domain concept; either way, adapters implement them and the core stays stable.
 


# 13.3 Clean architecture mapping: entities/usecases/gateways

## Intuition

Clean Architecture refines hexagonal:
- Entities = domain
- Use Cases = application orchestrators
- Gateways = ports/adapters to external systems

## Concepts (mapping)

- **Entities**: core business rules
- **Use cases**: application-specific rules (workflow)
- **Interface adapters/gateways**: translate external to internal

## Architecture (ASCII)

```
Entities (Domain)
   ^
   |
Use Cases (Application)
   ^
   |
Gateways/Adapters (Infra)

```

## Practical example

`CreateOrderUseCase`:
- validates input, calls domain to create order
- uses `IOrderRepo` gateway to persist
- uses `IPaymentGateway` to charge
- returns DTO

## Interview questions

**Q. What is a “use case” class vs domain service?**
Use case is the application workflow/orchestration for a user action, while a domain service holds core business rules that don’t fit an entity and stays free of DB/HTTP (called by the use case).
 
**Q. Where do DTOs live? (outside domain)**
 


# 13.4 Event-driven architecture at LLD level (events as contracts)

## Intuition

Events reduce coupling:
- producer doesn’t know consumers
- consumers evolve independently

But events are **contracts**: once published, they are hard to change.

## Think of it like a restaurant bell 🔔

### Direct calls (tight coupling)

Customer pays → waiter must personally:
- tell kitchen to make invoice
- tell staff to send email
- tell manager to update analytics

So the waiter **knows everyone** and must coordinate everything. If email system is down, the whole payment flow can get stuck.

### Event-driven (loose coupling)

Customer pays → waiter just rings a bell: **“ORDER_PAID!”**  
Now:
- invoice team hears bell → generates invoice
- email team hears bell → sends receipt
- analytics team hears bell → logs event

The waiter doesn’t know who listens. They can change independently.

That’s literally what an **event** is: a “bell ring” announcing a fact.

## “Events are contracts” — why people say this

Once you ring a bell with a message like:

`OrderPaid { orderId, amount, paidAt }`

other teams start depending on those fields.  
If you suddenly remove `amount` or rename it, those consumers break — just like changing a public API.

So events must be treated as stable **contracts** (often versioned).

## Concepts

- **Domain events**: reflect business facts (`OrderPaid`)    
- **Integration events**: cross-service messages (versioned)
- **Outbox pattern** (conceptually): don’t lose events on DB commit boundaries

## Architecture (ASCII)

```
UseCase -> Domain -> emits Event -> EventBus -> Handlers
```

## Practical example

`OrderPaid` triggers:
- invoice generation
- email notification
- analytics

## If you remember only 3 lines

1. Event = “a fact happened” broadcast (producer doesn’t call consumers directly).
2. Events are contracts → hard to change → version them.
3. Outbox ensures you don’t lose events across DB commit + publish failures.
## Interview questions

**Q. Event vs command?
A command is an instruction/intent (“do this”) and usually targets a specific handler (e.g., ChargeCard, ShipOrder). An event is a fact that already happened (“did happen”) and can be consumed by many listeners (e.g., OrderPaid, OrderShipped).

**Q. How do you version events safely?**
Versioning is about **schema evolution without breaking consumers**: keep old versions working and introduce new versions in a compatible way (typically **add fields, don’t remove/rename**, or publish `OrderPaid.v2` while still emitting `v1` until all consumers migrate).


# 13.5 CQRS at LLD level (when it’s worth it)

## Intuition

CQRS means you **separate the write side from the read side** because they have different goals.

CQRS separates:
- **Commands** (writes): enforce invariants
- **Commands (writes)**: operations that change state like `PlaceOrder`, `CancelOrder`.  
Here you care about **business rules/invariants**: “can’t ship before paid”, “stock can’t go negative”.

- **Queries** (reads): optimized projections
- **Queries (reads)**: operations that only fetch data like `GetOrderSummary`.  
Here you care about **fast, convenient reading**, not enforcing invariants.

Use it when:
- reads and writes have very different models
- complex query needs don’t fit domain aggregates

Don’t use it when:
- system is small; adds complexity

## Architecture (ASCII)

```cpp
Command API -> Write Model (Domain aggregates) -> DB
Query API   -> Read Model (projections/cache)  -> optimized store/view
```

## Practical example

- Write: `PlaceOrder`
- Read: `GetOrderSummary` from a denormalized view

## Interview questions

**Q. What complexity does CQRS introduce?**
Two models/stores to maintain, projection-building logic, handling eventual consistency (stale reads), backfills/rebuilds of projections, and more ops/monitoring around the pipeline.
 
**Q. How do you keep read model consistent? (eventual consistency)**
You update the read model asynchronously from the write model (events/CDC/outbox) and accept **eventual consistency**; make updates idempotent, track offsets/versioning, and support replay/backfill to rebuild projections.