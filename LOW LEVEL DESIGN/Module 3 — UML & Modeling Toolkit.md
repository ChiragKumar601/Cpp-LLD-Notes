
# Class Diagram Essentials (LLD Interview UML-lite)

**Intuition**
A class diagram is a **communication tool**:
- shows **what objects exist (entities + services + helpers)**
- **who owns whom(composition/lifecycle)**
- **who knows whom(dependencies/coupling)**
- **what can change independently(module boundaries, stable interfaces)**

Interviewers don’t care if you used the exact UML arrow head. They care if your diagram makes these 4 things **unambiguous**.

#### 1) “What objects exist”

Your diagram should answer: _“What are the nouns in this system?”_

##### Common buckets (LLD)
- **Domain / Entity**: `Order`, `User`, `Ride`, `Booking`
- **Value Objects**: `Money`, `Address`, `DateRange`
- **Services (business logic)**: `PricingService`, `MatchingService`
- **Repositories (persistence port)**: `OrderRepo`
- **Factories / Builders**: `OrderFactory`
- **Controllers / APIs**: `OrderController` (optional in LLD)

**Interview trick:** start with nouns (entities), then add verbs (services).

#### 2) “Who owns whom” (ownership / lifecycle)

This is the biggest source of confusion, so make it explicit.

**Ownership means:**
- Parent **creates/destroys** the child (or child’s meaning depends on parent)
- Child cannot reasonably exist alone (in your model)

**Strong has-a (composition)** examples:
- `Order` **owns** `OrderItem`
- `Car` **owns** `Engine` (in typical model)
- `House` **owns** `Room`

**Weak has-a (aggregation/reference)** examples:
- `Company` references `Employee`
- `Playlist` references `Song`
- `Team` references `Player`

**How to show it (interview-friendly, no UML perfection needed)**
- Use notes:
    - `Order *-- OrderItem (Order owns lifecycle)`
    - `Playlist --> Song (just references, no ownership)`

**Quick test:**
If parent is deleted, should child be deleted too?
- Yes → owned (composition)
- No → reference (aggregation)

#### 3) “Who knows whom” (dependency direction)

This is about **coupling**. “Knowing” means one class holds a reference to another or calls it directly.

##### Good interview principle:

**Depend on interfaces, not concrete classes** (DIP)

Bad:  
`PaymentService --> StripeGateway`

Good:  
`PaymentService --> IPaymentGateway`  
`StripeGateway implements IPaymentGateway`

 **How to communicate this clearly**
- Arrows should usually go **from caller to callee**
- Add constructor injection idea mentally:
    - if A knows B, A likely receives B in constructor

This helps the interviewer see: **what changes ripple where**.

#### 4) “What can change independently”

This is where you score big.
A good diagram silently answers:
- “If I replace DB, do I rewrite business logic?”
- “If I add a new payment provider, what changes?”
- “If pricing logic changes, do entities change?”

 **The technique: draw boundaries (mentally or with boxes)**

- **Domain**: Entities + pure rules
- **Application**: Use-cases/services orchestrating
- **Infrastructure**: DB, external APIs

Example:
- Domain doesn’t know infrastructure.
- Infrastructure depends on interfaces defined by domain/app.

That means: infra can change independently.

## Core Concepts (What you must know)

### 1) The 5 relationships that matter in LLD
1. **Generalization (Inheritance)**: `Derived is-a Base`
2. **Realization (Implements interface)**: `Concrete implements IInterface`
3. **Association (“knows about”)**: A has a reference to B (can be long-lived)
4. **Aggregation (weak “has-a”)**: A groups Bs but doesn’t own their lifetime
5. **Composition (strong “has-a”)**: A owns Bs’ lifetime (dies together)

### 2) Multiplicity (cardinality)

Multiplicity answers: **“For ONE object of A, how many objects of B can it be linked to?”**  

You write it **near the B end** of the relationship line.
- **`1` (exactly one)**  
    Each `A` must be linked to **exactly 1** `B`.  
    Example: `Order -> 1 Customer` (every order has exactly one customer)
- **`0..1` (optional one)**  
    Each `A` may have **0 or 1** `B`.  
    Example: `User -> 0..1 Address` (address optional)
- **`*` or `0..*` (zero or more)**  
    Each `A` may be linked to **any number**, including zero, of `B`.  
    Example: `User -> 0..* Orders`
- **`1..*` (one or more)**  
    Each `A` must be linked to **at least 1** `B`.  
    Example: `Order -> 1..* OrderItems` (an order must have at least one item)

**Tiny Example Diagram**

```sql
Order 1 -------- 1 Customer
Order 1 -------- 1..* OrderItem
User  1 -------- 0..* Order
User  1 -------- 0..1 Address
```

### 3) Mapping UML → C++ (this is the interview superpower)

| Relationship | Typical C++ representation                            | Lifetime meaning                                                                            |
| ------------ | ----------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Association  | `B*`, `B&`, `std::shared_ptr<B>`                      | A can use B; ownership depends                                                              |
| Aggregation  | `std::vector<B*>` / `std::vector<std::shared_ptr<B>>` | A “collects” Bs; **Deleting A should NOT imply deleting B.** <br>So `B` is **independent**. |
| Composition  | member object `B b;` OR `std::unique_ptr<B>`          | A owns B; B destroyed with A                                                                |
| Inheritance  | `class D : public B`                                  | substitutable polymorphism                                                                  |
| Implements   | `class X : public IFoo`                               | interface contract                                                                          |
**Rule of thumb**
- If A controls B’s lifetime → **composition** (`unique_ptr` or member object).
- If B can exist independently → **aggregation/association**.

**One-liner to say in interviews**
- **Implements** = “I promise I can do this” (contract)
- **Inheritance** = “I am a specialized version of this” (shared structure + behavior)

## Architecture View (UML-lite symbols you can write fast)

**Inheritance / Interface**
```
Circle --------|> Shape         (is-a)
CardPayment ---|> PaymentMethod (implements interface)
```

**Association (knows/uses)**
```
Order --------> Customer
```

**Composition (owns)**
```
Car *---- Engine
(Engine lifetime depends on Car)
```

**Aggregation**
```
Team o---- Player
(Players can exist without Team)
```

> In interviews, you can literally write:
- `*---` for composition (strong ownership)
- `o---` for aggregation (weak ownership)
- `-->` for association/dependency(here arrow pointer means navigability)

## Practical Example (LLD-friendly class diagram): Parking Lot (mini)

**Requirements slice**
- `ParkingLot` has many Floors
- Floor has many Spots
- Ticket references Vehicle + Spot
- Vehicle exists outside; Ticket doesn’t own Vehicle

**UML-lite diagram**
```
ParkingLot *---- 1..* ParkingFloor
ParkingFloor *--- 1..* ParkingSpot

Ticket  -----> Vehicle        (association: ticket refers to vehicle)
Ticket  -----> ParkingSpot    (association: ticket points to spot)
Ticket  -----> PaymentReceipt (optional)
```

**C++ mapping sketch (ownership clarified)**
```cpp
class ParkingLot {
    std::vector<std::unique_ptr<ParkingFloor>> floors; // composition
};

class ParkingFloor {
    std::vector<std::unique_ptr<ParkingSpot>> spots;   // composition
};

class Ticket {
    const Vehicle* vehicle;      // association (not owned)
    ParkingSpot* spot;           // association (spot owned by floor)
};
```
**Why this mapping is correct**
- Floors/spots should not outlive parking lot → composition
- Vehicle is external domain object → ticket references it, doesn’t own it

## Interview-Level Practice Questions

**Q. When would you use `shared_ptr` vs raw pointer in association?**
In associations I use raw pointers/references for non-owning links. If the class owns the object, I use `unique_ptr` (composition). I use `shared_ptr` only when ownership is genuinely shared, and `weak_ptr` for back-references to avoid cycles.

**Q. Convert this statement into UML:
“A Library has many Books, but Books can exist without a Library.”**

```
Library  0..*  o---->  Book  0..1
         (hollow diamond at Library)
```
**How to read it:**
- `Library o----> Book` = **aggregation** (Library “has” Books, but doesn’t own them)
- `0..*` near **Book** end = a Library can be linked to **many Books**
- `0..1` near **Library** end = a Book can exist with **no Library** (optional association)

For: “A Library has many Books, but Books can exist without a Library”

The UML you want is about these two questions:
Q1: For **one Library**, how many Books can it be linked to?
✅ **0..*** (a library can have zero or many books)

So near the **Book** end you put: **0..***
Q2: For **one Book**, how many Libraries can it be linked to?
That depends on your domain model:

 **Case A: Physical copy model (a copy belongs to at most one library at a time)**
 ✅ **0..1**
```
Library 1 -------- 0..* BookCopy
BookCopy 0..1 ----- 1   Library
```

Case B: Title/catalog model (same “Book” can be in many libraries)
✅ 0..* 
```
Library 0..* ------ 0..* BookTitle
```


**Q. What’s the difference between aggregation and composition in real systems?**
Both are “has-a”, but the difference is **ownership + lifecycle**.

**Composition (strong has-a)**
**Meaning:** A **owns** B. B’s lifetime is tied to A.
- If A is deleted, **B must be deleted**.
- B usually isn’t shared with other As.
- Invariants often live in A (“I guarantee my parts exist and are consistent”).

**Real system examples**
- `Order` **composes** `OrderItem`
- `Cart` **composes** `CartItem`
- `Document` **composes** `Page` (in many models)

**Implementation hint (C++)**
- typically `unique_ptr` or value member.

**Aggregation** (weak has-a)

**Meaning:** A **references** B. B exists independently.
- If A is deleted, **B still exists**.
- B can be shared / reused by multiple As.
- A doesn’t manage B’s lifecycle.

**Real system examples**
- `Playlist` **aggregates** `Song` (songs exist without playlist)
- `Team` **aggregates** `Player` (players can move teams)
- `Library` **aggregates** `BookTitle` (titles exist independent of a library)

**Implementation hint (C++)**
- typically raw pointer/reference (non-owning) or `weak_ptr` if shared_ptr-managed.

**Interview rule to decide fast**
Ask: **“If I delete A, should B be deleted too?”**
- Yes → **Composition**
- No → **Aggregation**
That’s the practical difference in real systems.

# 3.2 Multiplicity, navigability, ownership
### Intuition
Class diagrams are useful only if they answer these 3 questions clearly:
1. **How many?** → _Multiplicity_
2. **Who knows whom?** → _Navigability_
3. **Who owns lifetime?** → _Ownership_

In C++ LLD, **ownership is the big one** (because it decides pointers vs smart pointers vs values).

## 3.2.1 Multiplicity (Cardinality)
### Core multiplicities you must be fluent in
- `1` : exactly one
- `0..1` : optional
- `*` or `0..*` : many (including zero)
- `1..*` : at least one
- `m..n` : bounded range (rare, but useful)

### How to choose multiplicity (interview rule)

Multiplicity comes from **business constraints**, not from how you plan to store it.
Examples:
- An `Order` has **1** `Customer`
- A `Customer` has **0..*** `Orders`
- A `Ticket` has **0..1** `PaymentReceipt`
- A `ParkingSpot` has **0..1** current `Vehicle`

 **Example Diagram**
```sql
Customer 1 -------- 0..* Order
Order    1 -------- 1    Customer

Ticket   1 -------- 0..1 PaymentReceipt
Spot     1 -------- 0..1 Vehicle
```

### C++ mapping cheat-sheet

- `1` → member value or reference (or `unique_ptr` if polymorphic)
- `0..1` → `std::optional<T>` (value) OR nullable pointer OR `unique_ptr<T>`
- `0..*` → `std::vector<T>` / `std::vector<unique_ptr<T>>` / `unordered_map<Id, T>`
- `1..*` → same as many, but enforce invariant “non-empty” via constructor/validation

**Rule:** multiplicity is a _domain rule_, container choice is an implementation detail.

## 3.2.2 Navigability (Direction of “knowing”)

### Intuition
Navigability answers:

> Does A need a direct reference to B to do its job?

If not, don’t add the pointer/reference.  
Every extra reference increases coupling and makes ownership harder.

### Examples

#### One-way navigability (recommended by default)
```sql
Order  -----> Customer
```
Meaning: Order can reach Customer, but Customer does NOT need to hold all orders.

**C++**
```cpp
class Order {
    const Customer* customer; // or CustomerId
};
```

#### Two-way navigability (only when necessary)
```sql
Customer <-----> Order
```
Meaning: both sides can navigate directly.

**C++ (danger zone)**
```cpp
class Customer {
    std::vector<Order*> orders; // risk: invalid pointers, lifetime coupling
};

class Order {
    Customer* customer;
};
```

### Interview rule for navigability

✅ Prefer **unidirectional** + query via repository/service when collections can grow big.

Example: Customer with 50,000 orders
- Don’t store `vector<Order*>` inside Customer
- Fetch orders via `OrderRepository.findByCustomer(customerId, page)`

### Better pattern (scalable)
```
Customer (no orders list)
Order ---> CustomerId
OrderRepository ---> query orders by customerId
```

## 3.2.3 Ownership (Lifetime control)

### Intuition

Ownership answers:

> Who destroys the object?

In C++ this is non-negotiable.  
If you’re unclear, you’ll create leaks, dangling pointers, cycles, or shared_ptr abuse.

### The 3 ownership categories

#### A) **Composition (strong ownership)**

- “Part-of”
- child dies with parent
- parent controls lifetime

Diagram:
```
Car *---- Engine
```

C++ options:

1. Value member (best if concrete + small)
```cpp
class Car { Engine engine; };
```

 2. `unique_ptr` (best for polymorphism / heavy object / forward-declare)
```cpp
class Car { std::unique_ptr<Engine> engine; };
```


#### B) **Aggregation (weak ownership / grouping)**

- “Has-a, but does not own”
- objects exist independently

Diagram:
```
Team o---- Player
```

C++ (common):
- store IDs (best)
- or non-owning pointers/references
- or `shared_ptr` only if lifetimes truly shared

```cpp
class Team {
    std::vector<PlayerId> players;   // best
    // or std::vector<Player*> players; (non-owning)
};
```

#### C) **Association (just a reference)**
- “Uses/knows about”
- no lifetime ownership implied

Diagram:
```
Ticket -----> Vehicle
```
C++:
```cpp
class Ticket {
    const Vehicle* vehicle; // non-owning
};
```

### Ownership Decision Tree (use in interviews)

When you need a link A → B, ask:
**Does A own B’s lifetime?**
- Yes → `unique_ptr<B>` or `B` member (composition)
- No → go next

**Is B guaranteed to outlive A?**
- Yes → `B&` or `B*` (non-owning)
- No/unclear → prefer **ID reference** + lookup (repository)  
    (or `shared_ptr` only if shared ownership is truly required)

**Can cycles happen?**
- If using `shared_ptr`, cycles are common → break with `weak_ptr`

### Practical Example: Parking Lot Ownership (clean C++)

Diagram:
```
ParkingLot *---- 1..* Floor *---- 1..* Spot
Ticket -----> Vehicle
Ticket -----> Spot
```

C++ mapping:
```cpp
class ParkingLot {
    std::vector<std::unique_ptr<Floor>> floors;  // owns floors
};

class Floor {
    std::vector<std::unique_ptr<Spot>> spots;    // owns spots
};

class Ticket {
    VehicleId vehicleId;   // safer than pointer
    SpotId spotId;         // safer than pointer
};
```

**Why IDs are often best**
- avoids dangling pointer risks
- decouples lifetimes
- easy to persist/log


### Common Interview Traps (and how to answer)

#### Trap 1: “Why not always store pointers?”
In C++, a **raw pointer (`T*`) only says “I can point to a T.”**  
It does **not** say any of these critical things:
- **Who owns the object?** (Who is responsible to `delete` it?)
- **How long will it live?**
- **Can it be shared?**
- **Can it be null?**
- **Is it safe to store?** (Will it dangle later?)

That’s what people mean by **“pointers don’t express ownership.”**

Raw pointers don’t tell who owns the object, so they don’t prevent leaks or dangling references. In C++ we encode lifetime explicitly: `unique_ptr` for ownership, `shared_ptr/weak_ptr` for shared graphs, and references/IDs for non-owning links.

#### Trap 2: “Why not always use shared_ptr?”
Because shared_ptr:
- hides ownership decisions
- adds overhead(“Overhead” means **extra runtime + memory cost** you pay compared to `unique_ptr` / raw pointer / reference.)
- creates cycles  
    Use it only when ownership is genuinely shared.

#### Trap 3: Bidirectional links everywhere
Bidirectional links increase coupling and make lifetime management hard.  
Prefer unidirectional + repository queries.

### Interview Questions

#### 1) “Library has many Books, Books can exist independently.”

### Relation

✅ **Aggregation / non-owning association** (Library doesn’t own Book lifetime)

**Multiplicity (per instance):**
- 1 **Library** → `0..*` **Books**
- 1 **Book** → `0..1` **Library** (book may exist with no library)

Your line `0..1 Library o---- 0..* Books` is **correct** (just ensure the numbers are placed near the right class ends).

**C++ representation:**
Because Library does **not own** books:
- `Library` should store **non-owning** references:
    - `std::vector<Book*>` (optional link, non-owning), OR
    - `std::vector<BookId>` + lookup (often best in real systems)

Example:

```cpp
class Library {
  std::vector<BookId> bookIds; // non-owning, persistence-friendly
};

class Book {
  std::optional<LibraryId> libraryId; // 0..1
};
```

---

#### 2) “Order must have at least one OrderItem.”

**Multiplicity**
✅ `Order 1 ---- 1..* OrderItem`

### How to enforce in code (your code needs correction)

Issues in your snippet:
- `vector<OrderItem*>` implies **non-owning** items (wrong for composition).
- `virtual bool check() = order_list.size() > 0;` is invalid C++ and also the wrong way to enforce invariants (don’t rely on callers to call `check()`).

**Correct enforcement pattern:** enforce in **constructor / factory** and keep mutation controlled.

Example (value semantics, simplest):
```cpp
class Order {
  std::vector<OrderItem> items;

public:
  explicit Order(std::vector<OrderItem> items_) : items(std::move(items_)) {
    if (items.empty()) throw std::invalid_argument("Order must have at least one item");
  }

  const std::vector<OrderItem>& getItems() const { return items; }
};
```

If `OrderItem` must be polymorphic/heavy:
```cpp
class Order {
  std::vector<std::unique_ptr<OrderItem>> items;

public:
  explicit Order(std::vector<std::unique_ptr<OrderItem>> items_) : items(std::move(items_)) {
    if (items.empty()) throw std::invalid_argument("empty");
  }
};
```

---

#### 3) When is `std::optional<T>` better than `T*` for `0..1`?

Use **`std::optional<T>`** when:
- the object is a **value**, owned “inside” the class (not shared)
- no polymorphism needed
- you want **no heap allocation** + no dangling-pointer risk

Use **`T*` / `T&`** when:
- you are referencing an object **owned elsewhere** (non-owning association)
- you need polymorphism
- you need “points to external thing” semantics

Rule:
- `optional<T>` = “I may contain a T”
- `T*` = “I may point to someone else’s T”

---

#### 4) Show a case where `shared_ptr` creates a memory leak (cycle). How to fix?

❌ `shared_ptr` cycle leak 
```cpp
#include <iostream>
#include <memory>

class Child; // forward declaration

class Parent {
public:
    std::shared_ptr<Child> child;

    ~Parent() { std::cout << "~Parent\n"; }
};

class Child {
public:
    std::shared_ptr<Parent> parent; // ❌ creates cycle

    ~Child() { std::cout << "~Child\n"; }
};

int main() {
    auto p = std::make_shared<Parent>();
    auto c = std::make_shared<Child>();

    p->child = c;
    c->parent = p;

    // p and c go out of scope here
    // but ~Parent and ~Child will NOT print (cycle keeps them alive)
    return 0;
}
```

✅ Fix: break the cycle using `weak_ptr` on the back reference
```cpp
#include <iostream>
#include <memory>

class Child; // forward declaration

class Parent {
public:
    std::shared_ptr<Child> child;

    ~Parent() { std::cout << "~Parent\n"; }
};

class Child {
public:
    std::weak_ptr<Parent> parent; // ✅ no ownership, breaks cycle

    ~Child() { std::cout << "~Child\n"; }
};

int main() {
    auto p = std::make_shared<Parent>();
    auto c = std::make_shared<Child>();

    p->child = c;
    c->parent = p; // doesn't increase refcount

    // Now destructors WILL run at end of scope
    return 0;
}
```

---

#### 5) In LLD, when would you use IDs instead of pointers?

Use **IDs** when:
- the target object’s lifetime is **unclear / not in-memory owned**
- relationship crosses **aggregate boundaries** (DDD-style)
- you need **persistence + serialization** (DB, APIs)
- you want to avoid **object graphs + cycles** in memory

Example: `ExpenseSplit` stores `userId` (not `User*`) and resolves via `UserRepository`.


# 3.3 Sequence Diagrams for Behavior (LLD Gold)

### Intuition
Most candidates draw classes. Strong candidates show **runtime flow**.
A sequence diagram answers:
- **Who calls whom?**
- **In what order?**
- **What objects get created?**
- **Where do decisions happen?**
- **Where do state changes happen?**

In interviews, this proves you can design **behavior**, not just structures.

## Core Concepts (UML-lite, interview version)

### 1) Participants (lifelines)

**What it means:** the “actors/objects” that take part in a flow.

Typical interview participants:
- `User` (actor)
- `Controller` (API boundary)
- `Service` (use-case/business orchestration)
- `Repo` (DB access)
- `Gateway` (external API)

**LLD rule:** show only the 4–6 participants that matter for the use-case.

---

### 2) Messages (calls)

#### Sync call

Meaning: caller waits for completion.
```
A -> B : method(args)
B --> A : result
```
#### Async call (optional)

Meaning: caller does not wait (queue/event bus).
```
A ~> Queue : enqueue(msg)
Worker -> Service : process(msg)
```

**LLD rule:** Most CRUD flows are sync; use async for long-running work.

---

### 3) Activation (optional)

Activation box means: **“this participant is executing”**.  
In interviews, you can **skip** it unless someone asks.

If you do show it: it starts at call and ends at return.

---

### 4) Object creation

You show when a new object is created in the flow.

Typical patterns:
- Service creates domain object:

`Service -> Order : new Order(items)`
- Factory creates concrete:

`Service -> OrderFactory : create(cmd) OrderFactory --> Service : Order`

**LLD rule:** show creation only if it clarifies ownership/invariants.

---

### 5) Alternatives / branches (`alt`)

Use `alt` to show conditional flows (success/failure, retries).
Example (payment):

```rust
alt payment success
  Service -> Gateway : charge()
  Gateway --> Service : ok
else payment failed
  Service -> Repo : markFailed()
end
```

**LLD rule:** use `alt` for:
- validation failure (400)
- not found (404)
- external API failure + retry
- idempotency conflict

## Practical Example: **Parking Lot — Park Vehicle flow**

### Actors
- Driver
- `ParkingLot` (or `ParkingService`)
- `SpotFinder` strategy
- Spot
- `TicketService`
- Repository

### Sequence Diagram (ASCII UML-lite)
```rust
Driver -> ParkingService : park(vehicle)
ParkingService -> SpotFinder : findSpot(vehicleType)
SpotFinder --> ParkingService : spotId

alt [spotId not found]
  ParkingService --> Driver : "FULL"
else
  ParkingService -> SpotRepo : get(spotId)
  SpotRepo --> ParkingService : Spot*

  ParkingService -> Spot : assign(vehicleId)
  Spot --> ParkingService : ok

  ParkingService -> TicketService : createTicket(vehicleId, spotId)
  TicketService -> TicketRepo : save(ticket)
  TicketRepo --> TicketService : ticketId
  TicketService --> ParkingService : ticket

  ParkingService --> Driver : ticket
end
```

### ATM Withdrawal Sequence Diagram

**Participants**
`User, ATMController, AccountService, AccountRepo, CashDispenser`

```cpp
User -> ATMController : insertCard(cardNumber)
ATMController -> User : promptPin()

User -> ATMController : enterPin(pin)
ATMController -> AccountService : authenticate(cardNumber, pin)
AccountService -> AccountRepo : verifyPin(cardNumber, pin)

alt [invalid pin]
  AccountRepo --> AccountService : authFailed
  AccountService --> ATMController : error("Incorrect PIN")
  ATMController --> User : show("Incorrect PIN")
else [auth ok]
  AccountRepo --> AccountService : authOk(userId)
  AccountService --> ATMController : ok(sessionToken)
  ATMController --> User : show("Welcome")

  User -> ATMController : withdraw(amount)
  ATMController -> AccountService : withdraw(sessionToken, amount)

  AccountService -> CashDispenser : hasCash(amount)

  alt [ATM cash insufficient]
    CashDispenser --> AccountService : no
    AccountService --> ATMController : error("Insufficient cash in ATM")
    ATMController --> User : show("Insufficient cash in ATM")
  else [ATM cash ok]
    CashDispenser --> AccountService : yes

    AccountService -> AccountRepo : getBalance(userId)

    alt [insufficient account balance]
      AccountRepo --> AccountService : balance
      AccountService --> ATMController : error("Insufficient balance")
      ATMController --> User : show("Insufficient balance")
    else [enough balance]
      AccountRepo --> AccountService : balance

      AccountService -> AccountRepo : debit(userId, amount)   // atomic update

      alt [debit failed (race/lock)]
        AccountRepo --> AccountService : fail
        AccountService --> ATMController : error("Try again")
        ATMController --> User : show("Try again")
      else [debit ok]
        AccountRepo --> AccountService : success

        AccountService -> CashDispenser : dispense(amount)

        alt [dispense failed]
          CashDispenser --> AccountService : fail
          AccountService -> AccountRepo : creditBack(userId, amount) // rollback/compensation
          AccountService --> ATMController : error("Dispense failed, amount reversed")
          ATMController --> User : show("Dispense failed, amount reversed")
        else [dispense ok]
          CashDispenser --> AccountService : ok
          AccountService --> ATMController : success("Take cash")
          ATMController --> User : show("Take cash")
        end
      end
    end
  end
end

```
### What interviewers notice here

- Spot finding is delegated to `SpotFinder` (OCP)
- Storage is behind repos (DIP) 
- Error case is explicit (`alt` full)
- State change is clear (`Spot.assign()`)

## Sequence Diagram “Rules” (that make your design clean)

###  Rule 1: The caller should not “reach through” objects (LoD)

Don’t do long chains like `A.getB().getC().doX()`.  
It makes your caller depend on internal structure.

Do this instead:
- ask the **top object** for what you need (`order.shippingCity()`), or
- move the logic to the right object.

Bad:
```
Service -> Order -> Customer -> Address -> City
```
Better:
- ask `Order` for what you need OR move logic to appropriate object.

### Rule 2: Keep orchestration in one layer

Only one place should “coordinate” the whole flow.
- **Controller:** parse request, call service, return response
- **Service:** steps of the use-case (validate → repo → gateway → save)
- **Domain:** rules + state changes
- **Repo/Gateway:** DB/external calls only

### Rule 3: Use explicit state transitions

Don’t set fields from outside like `order.status = PAID`.
Prefer intent methods like:
- `order.markPaid()`
- `ticket.close()`
This keeps rules in one place ad prevents invalid states.

### Rule 4: Show `alt` for failures

Always show the main failure paths in sequence diagrams:
- invalid input
- not found
- resource unavailable (insufficient balance/cash)
- external failure (payment gateway/DB)

It proves your design handles real-world cases.


## Interview Questions

### Q. Where do you put validation in a sequence diagram (controller vs service vs domain)?
Yes — **mostly in the Service**, but split validation into 3 types (this is the LLD answer).
#### 1) Controller: **request-shape validation**
Put here:
- required fields present
- format/type checks (empty string, invalid number, malformed UUID)
- basic auth/header presence

Sequence diagram: show it as a quick guard before calling service (or a simple `alt [invalid request] return 400`).

#### 2) Service: **use-case/business validation**
Put here:
- “user is member of group”
- “withdraw amount > 0”
- “order is not already paid”
- “book is available”

This is the main place you were thinking of ✅

#### 3) Domain object: **invariant validation**

Put inside domain methods to prevent invalid state:
- `order.markPaid()` checks `status == CREATED`
- `ticket.close()` checks `not already closed`

In sequence diagram: show `Service -> Order : markPaid()` and the failure as `alt [invariant violated]`.

#### Interview rule of thumb
- **Controller validates syntax**
- **Service validates business rules**
- **Domain enforces invariants**

### Q. Draw for **Elevator request**: hall call → scheduler → elevator movement.

**Hall call → scheduler → elevator arrives**
```css
User -> HallPanel : pressUp(currentFloor)

HallPanel -> ElevatorScheduler : submitHallCall(currentFloor, UP)
ElevatorScheduler -> PriorityQueue : push(HallCall(currentFloor, UP))

ElevatorScheduler -> ElevatorScheduler : pickBestElevator(call)
ElevatorScheduler -> ElevatorController : assignStop(currentFloor)

ElevatorController -> ElevatorService : moveTo(currentFloor)
ElevatorService --> ElevatorController : arrived(currentFloor)
ElevatorController -> Doors : open()
```

**Inside elevator: destination request → scheduler → move**
```css
User -> CarPanel : selectFloor(destinationFloor)

CarPanel -> ElevatorScheduler : submitCarCall(elevatorId, destinationFloor)
ElevatorScheduler -> PriorityQueue : push(CarCall(elevatorId, destinationFloor))

ElevatorScheduler -> ElevatorController : assignStop(destinationFloor)

ElevatorController -> ElevatorService : moveTo(destinationFloor)
ElevatorService --> ElevatorController : arrived(destinationFloor)
ElevatorController -> Doors : open()
```

Hall call = the button you press outside the elevator (Up/Down) on a floor.
Example: on 5th floor you press UP → that’s a hall call.

Car call = the button you press inside the elevator car (destination floor).
Example: inside elevator you press 12 → that’s a car call.


### Q. In **Splitwise addExpense**, where do you compute balances? Show flow.
Yes: **`addExpense` belongs in `ExpenseService`**, and **balance computation should happen in the service/domain**, not in the controller.

Compute/update balances in a **Ledger/Balance component inside the service layer** (could be a method on `GroupLedger` domain object or a `BalanceService`). The controller should only map request/response.

```cs
Splitwise: AddExpense (compute balances in Service via ledger deltas)

Participants: User, ExpenseController, ExpenseService, GroupRepo, ExpenseRepo, TransactionRepo

User -> ExpenseController : POST /groups/{groupId}/expenses (req)
ExpenseController -> ExpenseService : addExpense(groupId, req)

ExpenseService -> GroupRepo : findById(groupId)

alt [group not found]
  GroupRepo --> ExpenseService : null
  ExpenseService --> ExpenseController : error(NotFound)
  ExpenseController --> User : 404
else [group found]
  GroupRepo --> ExpenseService : Group

  ExpenseService -> ExpenseService : validate(req)

  alt [invalid request/business rules]
    ExpenseService --> ExpenseController : error(BadRequest)
    ExpenseController --> User : 400
  else [valid]
    ExpenseService -> ExpenseService : new Expense(req)      // create domain object
    ExpenseService -> ExpenseRepo : save(expense)
    ExpenseRepo --> ExpenseService : expenseId

    ExpenseService -> ExpenseService : computeBalanceDeltas(expense)
    ExpenseService --> ExpenseService : List<Transaction>    // ledger entries: (fromUser,toUser,amount)

    ExpenseService -> TransactionRepo : saveAll(transactions)
    TransactionRepo --> ExpenseService : ok

    ExpenseService --> ExpenseController : success(expenseId)
    ExpenseController --> User : 201 Created
  end
end
```

### Q. Draw a sequence diagram for LRU Cache get/put (include eviction).
#### 1) LRU Cache: `get/put` with eviction

**Participants:** `Client, LRUCache, HashMap, DoublyLinkedList`

#### `get(key)`

```cs
Client -> LRUCache : get(key)
LRUCache -> HashMap : contains(key)?

alt [miss]
  HashMap --> LRUCache : false
  LRUCache --> Client : -1 / null
else [hit]
  HashMap --> LRUCache : Node*
  LRUCache -> DoublyLinkedList : moveToFront(node)   // mark as most recently used
  DoublyLinkedList --> LRUCache : ok
  LRUCache --> Client : node.value
end

```

#### `put(key, value)` (includes eviction)

```cs
Client -> LRUCache : get(key)
LRUCache -> HashMap : contains(key)?

alt [miss]
  HashMap --> LRUCache : false
  LRUCache --> Client : -1 / null
else [hit]
  HashMap --> LRUCache : Node*
  LRUCache -> DoublyLinkedList : moveToFront(node)   // mark as most recently used
  DoublyLinkedList --> LRUCache : ok
  LRUCache --> Client : node.value
end
```

---

## 2) Rate Limiter: `allowRequest()` (token refill + decision)

**Model:** Token Bucket  
**Participants:** `Client, RateLimiter, BucketRepo(Map), TokenBucket, Clock`

### `allowRequest(userId)`

```cs
Client -> RateLimiter : allowRequest(userId)
RateLimiter -> BucketRepo : get(userId)

alt [bucket not found]
  BucketRepo --> RateLimiter : null
  RateLimiter -> RateLimiter : create TokenBucket(capacity, refillRate)
  RateLimiter -> BucketRepo : put(userId, bucket)
  BucketRepo --> RateLimiter : ok
else [bucket found]
  BucketRepo --> RateLimiter : bucket
end

RateLimiter -> Clock : now()
Clock --> RateLimiter : tNow

RateLimiter -> TokenBucket : refill(tNow)
TokenBucket -> TokenBucket : tokens += (tNow - lastRefillTime)*refillRate (cap at capacity)
TokenBucket --> RateLimiter : ok

RateLimiter -> TokenBucket : tryConsume(1)

alt [tokens >= 1]
  TokenBucket -> TokenBucket : tokens -= 1
  TokenBucket --> RateLimiter : true
  RateLimiter --> Client : ALLOW
else [tokens < 1]
  TokenBucket --> RateLimiter : false
  RateLimiter --> Client : DENY
end
```


# 3. 4 State Diagrams for Stateful Systems (Where most LLD problems live)

## Intuition

If the system’s behavior depends on “what state it is in”, you need a **state model**.
State diagrams help you answer:
- What states exist?
- What events cause transitions?
- What actions happen on transitions?
- What transitions are illegal?

This prevents the classic bug:  
**“If/else jungle + scattered flags”**.

---

## Core Concepts (LLD interview version)

### 1) State

A named mode the system can be in.  
Examples: `Idle`, `HasMoney`, `Dispensing`, `OutOfService`

### 2) Event (Trigger)
Something that happens:
- user action: `insertCoin`, `selectItem`
- system action: `timeout`, `paymentSuccess`
- external event: `doorClosed`, `networkDown`

### 3) Transition
`StateA --(event)--> StateB`

### 4) Guard (Condition)
Transition only if condition holds:  
`HasMoney --(selectItem [stock>0])--> Dispensing`

### 5) Action (Side effect)
What happens during transition:  
`Dispensing / decrementStock, dispenseItem`


## When to use a state diagram (fast rule)

Use it if you see any of these:
- multiple `status` enums + many if/else checks
- workflows like payment/order/booking
- vending machine/elevator/traffic light/game

If state affects allowed operations → state diagram is mandatory.

## Architecture View: Why this ties directly to the **State Pattern**

- State diagram = behavior spec
- State pattern = implementation technique

```sql
Context (Machine/Order/Elevator)
   |
   +--> State (interface)
          |
          +--> ConcreteStateA
          +--> ConcreteStateB

```

---

## Practical Example 1: Vending Machine (classic interview)

### State diagram (ASCII)

```lua
[Idle]
  | insertCoin
  v
[HasCredit] <------------------+
  | selectItem [credit>=price] |
  v                            |
[Dispensing] --done--> [Idle]  |
  | refund                      |
  +-----------> [Idle] --------+
  
AnyState --outOfService--> [OutOfService]
[OutOfService] --restock--> [Idle]
```

### Key points interviewers look for
- Explicit illegal actions: selecting item in `Idle` should be rejected
- Guards: enough credit & stock availability
- Side effects: dispense reduces stock, credit resets

---

## Practical Example 2: Order Lifecycle (real-world)

### State diagram
```less
[CREATED]
   | paySuccess
   v
[PAID]
   | pack
   v
[PACKED]
   | ship
   v
[SHIPPED]
   | deliver
   v
[DELIVERED]

[CREATED] --cancel--> [CANCELLED]
[PAID]    --refund--> [REFUNDED]
[SHIPPED] --return--> [RETURN_REQUESTED] -> [RETURNED]
```

### Why state modeling matters

- It defines what transitions are allowed
- Prevents bugs like “refund after delivered without return”

 **(SDE-2/3 friendly): State pattern**
- Each state class defines behavior for events.
- Eliminates huge if/else.

**State pattern wins when:**
- many states
- many events
- rules differ per state
- you need OCP for adding state/event combinations


## Interview Questions

**Q. - In an online booking, model hold → confirmed → cancelled → expired.**
```scss
Online Booking (10/10 state diagram: states + events + guards only)

[IDLE]
  --(selectSeat [seatAvailable])--> [HOLD]

[HOLD]
  --(paySuccess [now <= holdExpiry])--> [CONFIRMED]
  --(payTimeout OR now > holdExpiry)--> [EXPIRED]
  --(cancelByUser)--> [CANCELLED]

[EXPIRED]
  --(paySuccess [now > holdExpiry])--> [REFUND_PENDING]   // late payment race

[CONFIRMED]
  --(cancelByUser [now < cancelCutoff])--> [CANCELLED]
  --(showStarted OR ticketUsed)--> [COMPLETED]
  --(eventCancelledByOrganizer)--> [REFUND_PENDING]

[REFUND_PENDING]
  --(refundSuccess)--> [REFUNDED]
  --(refundFailed)--> [REFUND_PENDING]   // retry loop

Terminal states: [CANCELLED], [EXPIRED], [COMPLETED], [REFUNDED]
```

**Q. For vending machine, where do you put “refund” and “timeout”?**
```scss
Idle --(insertCoin)--> HasMoney

HasMoney --(selectItem [credit>=price && stock>0])--> Dispensing
HasMoney --(timeout OR cancel)--> Idle
HasMoney --(selectItem [stock==0])--> HasMoney

Dispensing --(dispenseSuccess)--> Idle
Dispensing --(dispenseFailed)--> Idle   / refund(credit)
```

**Q.  Draw a state diagram for a traffic signal with pedestrian button.**
```scss
[NS_GREEN]
  --(timerElapsed)--> [NS_YELLOW]
  --(pedButtonPress)--> [NS_GREEN]         // sets pedRequest=true (note)

[NS_YELLOW]
  --(timerElapsed)--> [ALL_RED]

[ALL_RED]
  --(timerElapsed [pedRequest==true])--> [PED_WALK]
  --(timerElapsed [pedRequest==false])--> [EW_GREEN]

[PED_WALK]
  --(walkTimerElapsed)--> [PED_FLASH_DONT_WALK]

[PED_FLASH_DONT_WALK]
  --(flashTimerElapsed)--> [EW_GREEN]

[EW_GREEN]
  --(timerElapsed)--> [EW_YELLOW]
  --(pedButtonPress)--> [EW_GREEN]         // sets pedRequest=true (note)

[EW_YELLOW]
  --(timerElapsed)--> [ALL_RED]
```

**Q.  Draw for elevator: idle/moving/open/closing/overload/out-of-service.
```scss
[IDLE]
  --(hallCall OR carCall)--> [MOVING]

[MOVING]
  --(arrivedAtStop)--> [OPEN]
  --(faultDetected)--> [OUT_OF_SERVICE]

[OPEN]
  --(doorOpenTimerElapsed)--> [CLOSING]
  --(closeButton)--> [CLOSING]
  --(overloadDetected)--> [OVERLOAD]
  --(faultDetected)--> [OUT_OF_SERVICE]

[CLOSING]
  --(doorClosed)--> [IDLE]
  --(obstructionDetected)--> [OPEN]
  --(overloadDetected)--> [OVERLOAD]
  --(faultDetected)--> [OUT_OF_SERVICE]

[OVERLOAD]
  --(weightNormal)--> [OPEN]
  --(faultDetected)--> [OUT_OF_SERVICE]

[OUT_OF_SERVICE]
  --(maintenanceReset)--> [IDLE]
```

**Q.  Draw for rate limiter: tokens available vs exhausted + refill event.**
```scss
[AVAILABLE]
  --(allowRequest [tokens >= 1])--> [AVAILABLE]     // tokens--
  --(allowRequest [tokens < 1])--> [EXHAUSTED]

[EXHAUSTED]
  --(allowRequest)--> [EXHAUSTED]                   // deny
  --(refill [tokens >= 1])--> [AVAILABLE]
  --(refill [tokens < 1])--> [EXHAUSTED]
```

## Quick Checklist (use live in interviews)
- List states (as nouns/adjectives)
- List events (verbs)
- Add transitions with guards
- Add illegal transitions
- Add side effects/actions

### Example system: **Online Order** (Placed → Paid → Shipped → Delivered → Cancelled)

#### 1) List **states** (nouns/adjectives)
- `Created`
- `Paid`
- `Shipped`
- `Delivered`
- `Cancelled`
- `Expired` (optional: payment window)

#### 2) List **events** (verbs)
- `placeOrder`
- `paySuccess`
- `ship`
- `deliver`
- `cancel`
- `payTimeout`

#### 3) Add transitions with guards
```scss
Created --(paySuccess)--> Paid
Created --(cancel)--> Cancelled
Created --(payTimeout)--> Expired

Paid --(ship)--> Shipped
Paid --(cancel [beforeShipCutoff])--> Cancelled

Shipped --(deliver)--> Delivered
```

#### 4) Add illegal transitions (explicitly call them out)
- `Delivered --(cancel)-->` illegal
- `Cancelled --(paySuccess)-->` illegal
- `Expired --(paySuccess)-->` illegal (or goes to Refund flow)

(Interview note: “We reject these events with an error / ignore idempotently.”)

#### 5) Add side effects/actions (only at high level)
```scss
Created --(paySuccess)--> Paid      / capturePayment
Paid --(ship)--> Shipped           / assignCourier, generateTrackingId
Paid --(cancel)--> Cancelled       / initiateRefund
Shipped --(deliver)--> Delivered   / closeOrder
Created --(payTimeout)--> Expired  / releaseInventory
```

That’s the full checklist applied in order: **States → Events → Transitions + Guards → Illegal transitions → Actions.**

# 3.5 Translating Requirements → Objects → Responsibilities (CRC Cards)
''
## Intuition

Most LLD failures happen because people jump to classes too early.

CRC is a fast, interview-friendly method to go from:  
**requirements → correct objects → correct responsibilities → clean collaboration**

CRC = **Class – Responsibilities – Collaborators**

It prevents:
- god objects
- random utility classes
- mis-placed logic
- over-abstraction

## Practical Example: Parking Lot (CRC done properly)

### Requirement slice
- Park vehicle
- Unpark and pay
- Track spot availability
- 
### Step 1: Nouns → candidate classes
- ParkingLot, Floor, Spot
- Vehicle, Ticket
- PricingPolicy
- PaymentProcessor
- Repositories / Stores

### Step 2: CRC cards (clean version)

#### CRC 1

**Class:** `ParkingService`  
**Responsibilities:**
- `park(vehicle)` (orchestrate flow)
- `unpark(ticketId)` (orchestrate + pay)  
**Collaborators:**
- `SpotFinder`, `SpotRepo`, `TicketService`, `PricingPolicy`, `PaymentProcessor`

#### CRC 2

**Class:** `SpotFinder`  
**Responsibilities:**
- `findSpot(vehicleType)` (strategy-based allocation)  
**Collaborators:**
- `SpotRepo` (reads availability)

#### CRC 3

**Class:** `ParkingSpot`  
**Responsibilities:**
- `canFit(vehicleType)`
- `assign(vehicleId)`
- `release()`  
    **Collaborators:** none (pure domain)

#### CRC 4

**Class:** `TicketService`  
**Responsibilities:**
- `createTicket(vehicleId, spotId)`
- `closeTicket(ticketId)`  
**Collaborators:**
	- `TicketRepo`, `Clock` (optional)

#### CRC 5

**Class:** `PricingPolicy`  
**Responsibilities:**
- `calculateFee(entryTime, exitTime, spotType)`  
**Collaborators:** none (pure policy)
    

#### CRC 6

**Class:** `PaymentProcessor`  
**Responsibilities:**
- `charge(amount, paymentMethod)`  
**Collaborators:**    
- `IPaymentGateway` (DIP)
    

---

## ASCII “Responsibility flow” diagram

This is a fast interview artifact:
```rust
Driver
  |
  v
ParkingService
  |--> SpotFinder --> SpotRepo
  |--> ParkingSpot(assign/release)
  |--> TicketService --> TicketRepo
  |--> PricingPolicy
  |--> PaymentProcessor --> IPaymentGateway
```

This shows:
- Orchestration is centralized (service)
- Domain state transitions are inside domain objects
- Policies are isolated (SRP)
- Details are behind interfaces (DIP)

## How to decide “where does this method go?” (Rules)

### Rule 1: Data locality
If a method needs fields of `X`, it probably belongs in `X`.

### Rule 2: Policy vs Mechanism
- Business decision = policy class (`PricingPolicy`, `SpotAllocationPolicy`)
- Implementation detail = repo/gateway

### Rule 3: Orchestration belongs to services (not domain, not repo)
Service coordinates multiple domain objects.

### Rule 4: Avoid “manager of everything”
If `ParkingLotManager` has 30 methods → split by responsibility.

## Real-World Use Cases (CRC shines)

- Splitwise
- Elevator scheduler
- Vending machine
- Chess/game engines (piece behavior vs board rules)
- Order lifecycle + payment flow

## Interview-Level Practice Questions

### Q. - For Splitwise AddExpense, write CRC cards for: ExpenseService, SplitStrategy, Ledger, UserBalanceRepo
- For **LRU Cache**, CRC cards for:
    - LRUCache, DoublyLinkedList, HashMapIndex, Node
- For **Elevator**, CRC cards for:
    - ElevatorController, Scheduler, ElevatorCar, Door, Request
- In **Rate Limiter**, where does refill logic belong?
- Identify a god object in a design you’ve seen and show CRC-based refactor.


## CRC “5-minute interview workflow”

1. Read requirements → underline nouns & verbs
2. Write 4–8 CRC cards
3. Draw collaboration arrows
4. Convert CRC into class diagram
5. Validate with 2–3 key flows using sequence diagram