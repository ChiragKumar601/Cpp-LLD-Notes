## Intuition

Most LLD failures are not syntax—they’re **bad domain boundaries**:
- invariants scattered everywhere
- “God objects” doing everything
- anemic models (only getters/setters; logic in services)
- identity/equality bugs
- time/money modeled as primitives

A strong domain model makes correctness _hard to break_.

# 14.1 Entities vs Value Objects vs Aggregates

## Intuition

You must decide what things are:
- “a distinct thing that persists” → Entity
- “a concept defined only by its data” → Value Object
- “a consistency boundary” → Aggregate (with a root)

## Concepts

### Entity
- Has **identity** (`id`)
- Changes over time
- Equality by id

Examples: `Order`, `User`, `ParkingTicket`

### Value Object (VO)
- No identity; equality by value
- Usually immutable
- Safer than primitives

Examples: `Money`, `Email`, `Color`, `Pincode`, `TimeRange`

### Aggregate
- Cluster of entities/VOs treated as a single unit for invariants
- Has one entry point: **Aggregate Root**

Examples:
- `Order` aggregate root containing `OrderItems`
- `Wallet` root containing `Transactions`

## Architecture view

```
AggregateRoot (entity)
   |
   +-- entities/VOs
(invariants enforced inside)
```

## Practical example

`Money as VO avoids “amount + currency” bugs:

```cpp
struct Money {
    long long paise;
    std::string currency; // or enum
    bool operator==(const Money&) const = default;
};
```

## Interview questions

**Q. Why is `Money` a VO and not `double`?**
Because `double` causes rounding errors and loses currency rules; a `Money` value object stores exact minor units (paise), includes currency, and enforces invariants like non-negative/consistent currency.
 
**Q. What is the aggregate root responsible for?**
 The **aggregate root** is the “gatekeeper” of the aggregate:
- owns the aggregate’s entities/VOs
- enforces **all invariants** and valid state transitions
- is the **only entry point** for modifications (outside code shouldn’t mutate internals directly)
- defines what gets loaded/saved as one consistency unit (transaction boundary)


# 14.2 Invariants and aggregate boundaries

## Intuition

An invariant is a rule that must **always** hold.  
Aggregates exist to enforce invariants in one place.

## Concepts

- Invariants must be enforced **inside** the aggregate root
- External code should not mutate internal entities directly
- Cross-aggregate invariants are handled via:
    - application layer orchestration
    - eventual consistency (events)

## Practical example (Order)

Invariant:
- “Cannot ship an unpaid order”  
    So `Order::ship()` checks state internally.

## Architecture

```
App Service
  -> loads Order aggregate
  -> calls Order.pay()/ship()
  -> saves Order
```

## Interview questions

**Q. What belongs inside aggregate vs application service?**
 An **aggregate** contains the data + methods that enforce its **invariants and valid state transitions** (the rules that must always stay true for that aggregate). An **application service / use case** contains the **workflow**: load aggregates from repos, call aggregate methods, coordinate multiple aggregates, call external ports (payment/DB/events), and commit/save.
 
**Q. Why avoid invariants across aggregates?**
Because enforcing a rule that spans multiple aggregates usually needs **cross-aggregate locking or distributed transactions**, which is slow and hard to scale; instead you keep aggregates small and coordinate cross-aggregate rules via **events/sagas** with eventual consistency.


# 14.3 Identifiers, equality, hashing rules

## Intuition

Identity bugs are silent and deadly (cache keys, maps, dedup).

## Rules

- Entities: equality by **id**
- Value objects: equality by **all fields**
- Never use pointer address as identity in domain
- Hashing must match equality

## Practical example

```cpp
struct OrderId {
    std::string value;
    bool operator==(const OrderId&) const = default;
};

struct OrderIdHash {
    size_t operator()(const OrderId& id) const {
        return std::hash<std::string>{}(id.value);
    }
};
```

## Interview questions

**Q. Why do we wrap IDs instead of using string/int everywhere?**
 To get **type safety** and clearer domain intent: `OrderId` vs raw `string` prevents accidental mix-ups, centralizes validation/formatting, and makes equality/hashing consistent for maps/caches.
 
**Q. How do you avoid mixing userId with orderId?**
Use **strongly typed ID wrappers** (or distinct types) like `struct UserId{...}; struct OrderId{...};` so the compiler won’t let you pass one where the other is expected.


# 14.4 Modeling time, money, states, and transitions

## Intuition

Primitives cause ambiguity:
- time zones
- currency
- partial states
- invalid transitions

Model them explicitly.

## Concepts

### Money
- avoid `double`
- include currency
- define operations: add/subtract only if same currency

### Time

- use a clock interface `IClock`
- store timestamps in UTC internally
- define `TimeRange` VO for intervals

### State machines

- use explicit enums + transition methods
- or State pattern if complex

**Rule:** state transitions happen in one place (aggregate root).

## Practical example

```cpp
enum class OrderStatus { CREATED, PAID, SHIPPED, CANCELLED };

class Order {
    OrderStatus st;
public:
    void pay() {
        if (st != OrderStatus::CREATED) throw std::logic_error("bad transition");
        st = OrderStatus::PAID;
    }
};
```

## Interview questions

**Q. How do you prevent invalid transitions?**
Model state explicitly (enum) and allow changes only through transition methods on the aggregate root that validate “from → to” (otherwise reject/throw).

**Q. Why inject a clock?**
 So time-based logic is **testable and deterministic** (no flaky `now()`), and you can control time in tests and avoid coupling domain logic to the system clock/timezone.
 

# 14.5 Avoiding anemic domain models

## Intuition

Anemic model = objects with fields + getters/setters, while rules live in “god services”.  
It becomes:
- hard to ensure invariants
- easy to misuse

## Concepts

- Put invariants & business rules **inside** entities/aggregates
- Application services orchestrate, they don’t contain core rules
- Keep domain objects cohesive

## Practical example

Bad:
- `OrderService` checks everything; `Order` is a dumb struct

Good:
- `Order.pay()`, `Order.ship()`, `Order.addItem()` enforce rules internally

## Interview questions

**Q. Where should business rules live?**
 Core business rules should live in the **domain layer** (inside entities/aggregates/domain services), not in controller/god services.
 
**Q. How do you keep domain objects testable?**
Keep them **pure and deterministic**: no DB/HTTP/static globals, inject time/random via interfaces (`IClock`, `IRng`), and expose behavior through methods so you can unit test invariants and transitions with simple in-memory objects.