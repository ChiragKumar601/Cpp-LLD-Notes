
# 4.1 Pattern Families + The real purpose of patterns

## Intuition

A design pattern is a **named solution** to a **repeatable design pain**.  
You use it when it makes a change **safer/easier**.
A design pattern is **not a badge**. It’s a **reusable fix for a recurring design pain**.

In LLD interviews, you’re judged on:
- _Why_ you chose a pattern (the pain)
- _What trade-off_ you accepted
- _How cleanly_ it maps to C++ (ownership, lifetimes, interfaces)

> Pattern = **stable structure** + **extension points** to isolate change.

## Core Concepts

#### The 3 families (what they actually solve)
1. **Creational** → _How objects are created_ (hide construction complexity, choose type at runtime)
    - Examples: Factory, Builder, Prototype, Singleton (rarely recommended)
        
2. **Structural** → _How objects are connected_ (compose behaviors, wrap, simplify, adapt)
    - Examples: Adapter, Decorator, Composite, Facade, Proxy, Bridge
        
3. **Behavioral** → _How objects collaborate_ (control flow, state, notifications, commands)
    - Examples: Strategy, Observer, State, Command, Chain of Responsibility


### Practical Example (Identify pattern by pain)

#### Pain: “if/else ladder grows when new payment types are added”

✅ Pattern: **Strategy (+ Factory if runtime selection)**
```lua
CheckoutService ---> IPaymentMethod
                      /   |   \
                 Card    UPI   Wallet
```
**What this achieves**
- `CheckoutService` stays stable (closed for modification)
- Add new payment method by adding new class (open for extension)

## Real-World Use Cases (pattern family mapping)

### Creational (object creation volatility)
- Different payment gateways based on config → Factory
- Construct complex objects with many optional fields → Builder
- Clone pre-configured templates (e.g., default rules) → Prototype

### Structural (wrapping/composing)

- Add logging/retry to a gateway without editing it → Decorator
- Expose simplified API over complex subsystem → Facade
- Treat “folder and file” uniformly → Composite
- Convert one interface to another → Adapter

### Behavioral (workflow complexity)

- State-driven machines (vending/elevator/order lifecycle) → State
- Notify multiple subscribers (pub-sub) → Observer
- Encapsulate requests (undo/redo, queue) → Command
- Pipeline of checks (fraud rules, validations) → Chain of Responsibility

## Interview-Level Practice Questions

### Q. For each family, give one real example from a system you’ve built.
You can answer with any credible project-style example (even if hypothetical). A clean template:

- **Creational (Factory/Builder):** “We used a **Factory** to choose `PaymentGateway` (Razorpay/Stripe/Cashfree) at runtime based on tenant/config.”
- **Structural (Decorator/Adapter/Facade):** “We used a **Decorator** around an HTTP client to add retry + timeout + metrics without changing the core client.”
- **Behavioral (Strategy/State/Observer):** “We used **Strategy** for Splitwise split types (equal/exact/percent) so adding new split types didn’t touch `addExpense` flow.”

### Q. You see `switch(type)` growing weekly — which family/pattern and why?
This is a classic **Behavioral → Strategy (or Factory + Strategy)** problem.

**Why:**
- `switch(type)` grows because **behavior varies by type** and new types keep coming.
- Replace switch with a **registry/map: `type -> handler/strategy`**.
- Add a new type by adding a new class and registering it (OCP).

**Good one-liner:**
“Growing `switch(type)` is a signal for Strategy/Polymorphism: move each case into its own handler and dispatch via a registry.”

When Decorator _is_ appropriate: when you want to **stack optional features** (logging, retry, cache), not when you’re selecting one of many algorithms.



### Q. If requirements are state-heavy (many “statuses”), which family and why?
Behavior changes by status and transitions are constrained. State pattern localizes state-specific rules and makes transitions explicit, avoiding scattered conditionals

### Q. When would you choose Decorator over inheritance?
Choose **Decorator** when:
- You need to add behavior **dynamically** (per instance, runtime), not for all subclasses.
- You want to avoid **subclass explosion** (Logging+Retry+Cache combinations).
- You’re adding **cross-cutting concerns** (metrics, tracing, auth, retry, caching).

Inheritance is okay when:
- Variation is small and stable,
- “is-a” relationship is real,
- behavior is fixed at compile-time.

One-liner:
“Decorator for composable optional features; inheritance for true ‘is-a’ specialization.”

### Q. Why are patterns “named abstractions” rather than “mandatory solutions”?
Because:
- A pattern is a **shared vocabulary** for a trade-offed solution (“use Strategy here” conveys structure + intent fast).
- It’s not mandatory because the same problem can be solved **simpler** depending on scale (YAGNI).
- Overusing patterns creates unnecessary indirection.

One-liner:
“Patterns are names for proven trade-offs, not rules—use them only when the pain exists.”

## Mini “Pattern Smell → Likely Pattern” Cheatsheet

- Growing `if/else` on type → **Strategy / Factory**
- “Do X, then optional Y, then optional Z” → **Decorator**
- Complex subsystem, want a simple API → **Facade**
- Tree structure + uniform operations → **Composite**
- Too many state checks → **State**
- Need notification fan-out → **Observer**
- Pipeline of rules/checks → **Chain of Responsibility**
- Need undo/queue/replay → **Command**

# Pattern Selection Decision Tree (Pick patterns from pain, not memory)

### Intuition

In interviews, the correct answer is rarely “use pattern X”.  
The correct answer is:
> “I see _this specific change/complexity_, so I’ll introduce _this specific seam_.”

A pattern is just a **named seam**.

So the process is:
1. identify the **axis of change / pain**
2. choose the **smallest** pattern that removes that pain
3. keep object creation and ownership clean (C++ matters)

## Step 1 — Identify the pain (what’s growing / what’s risky)

### Pain A: “new types keep getting added”

Examples:
- payment types
- discount rules
- notification channels
- file parsers

✅ Likely patterns:
- **Strategy** (vary behavior)
- **Factory Method / Abstract Factory** (create correct type)
- **Registry** (plugin-like runtime extension)


---

### Pain B: “I need to add behavior without editing existing class”

Examples:

- logging, metrics, retries, caching, rate limiting

✅ Likely patterns:

- **Decorator** (wrap to add behavior)
- **Proxy** (control access / lazy / remote)
- **Adapter** (wrap to match interface)

---

### Pain C: “Complex subsystem, callers want simple API”

Examples:

- video encoding pipeline
- payment + invoice + inventory
- DB + cache + fallback

✅ Likely patterns:
- **Facade** (simple front over complex subsystems)


---

### Pain D: “Tree/hierarchy where you apply operations uniformly”

Examples:
- file system (file/folder)
- UI components (container/leaf)
- org chart

✅ Likely patterns:
- **Composite**


---

### Pain E: “Behavior depends on state; rules differ per state”

Examples:
- vending machine
- order lifecycle
- elevator door logic

✅ Likely patterns:
- **State**
- (simple alternative: enum + switch in one place)


---

### Pain F: “Many objects must react to a change/event”

Examples:

- notifications
- UI updates
- cache invalidation triggers
- pub-sub

✅ Likely patterns:

- **Observer**
- (scaled: event bus / pub-sub)

---

### Pain G: “You need to queue/undo/retry/replay a request”

Examples:

- undo/redo editor
- job queue
- retries with backoff

✅ Likely patterns:

- **Command**

---

### Pain H: “You have a pipeline of checks/handlers”

Examples:

- validation rules
- fraud rules
- authentication/authorization chain

✅ Likely patterns:

- **Chain of Responsibility**

## Step 2 — The Decision Tree (Interview-friendly)

Use this exact mental flow:

```sql
Q1: What changes frequently?
    |
    +-- New TYPES? --------------> Strategy (+ Factory/Registry)
    |
    +-- New STEPS around same call?
    |         |
    |         +-- Add behavior without modifying? -> Decorator
    |         +-- Control access/lazy/remote?     -> Proxy
    |
    +-- Too much COMPLEXITY exposed to clients? -> Facade
    |
    +-- Is it a TREE structure? ---------------> Composite
    |
    +-- Many STATE-dependent rules? -----------> State (or enum+switch)
    |
    +-- Many dependents on EVENTS? ------------> Observer
    |
    +-- Need UNDO/QUEUE/REPLAY? --------------> Command
    |
    +-- Pipeline of CHECKS? -------------------> Chain of Responsibility
```

## Step 3 — Choose the “smallest” valid solution (YAGNI applied)

Interviewers hate pattern soup. Show restraint.
### Example: Payment types = 2 today, might become 10

✅ Good approach:
- Start with **Strategy interface** (`IPaymentMethod`)
- If selection is dynamic → add **Factory** later
- If plugins → add **Registry** later

**What you say in interview:**

> “I’ll isolate the switch in one place first. If types grow, I can replace it with a registry without touching CheckoutService.”


## Step 4 — Map pattern choice to C++ ownership correctly (common fail point)

### Strategy / Observer / State typically use polymorphism

So store them as:

- `std::unique_ptr<IState>`
- `std::shared_ptr<IObserver>` if shared by many owners
- non-owning pointers if lifetime guaranteed

**Rule:**

- If one object owns the strategy → `unique_ptr`
- If many subscribe to one bus → usually non-owning + explicit unsubscribe OR weak_ptr


## Trade-offs (what interviewer expects you to mention)

### Strategy
✅ Pros: OCP, testability  
⚠️ Cons: more classes, object creation needed

### Decorator
✅ Pros: add cross-cutting behavior without modification  
⚠️ Cons: many wrappers, order of decorators matters

### Observer
✅ Pros: decoupled notification  
⚠️ Cons: memory leaks, unsubscribe, ordering, async concerns

### State
✅ Pros: eliminates state if/else jungle  
⚠️ Cons: many classes if small problem, transitions must be managed carefully

### Facade
✅ Pros: simplifies usage  
⚠️ Cons: can become God class if it does too much


## Interview Practice Questions

### Q. “Payment methods keep getting added.” Pick pattern(s) and explain growth path.
2 methods today, might become 10 - Start with **Strategy interface** (`IPaymentMethod`) - If selection is dynamic → add **Factory** later - If plugins → add **Registry** later

Strategy is the main one; factory/registry is for **selection/creation**.

### Q. “Need to add logging + retries to API calls without editing gateway code.” Pattern?
Decorator

### Q. “Folders and files must support `size()` uniformly.” Pattern?
Composite

### Q. “Order lifecycle has many statuses with different allowed actions.” Pattern?
“If only 3–4 states, `enum+switch` is okay; if rules grow, State pattern avoids scattered conditionals.”

### Q. “Fraud checks are applied sequentially, and new checks will be added.” Pattern?
“Each check either rejects or forwards; new checks plug into the chain without changing existing ones.”


# 4.3 Pattern Trade-offs + Anti-patterns (How patterns make designs worse, and how to defend choices)

### Intuition

Patterns are tools. Every pattern has a cost:
- more classes
- indirection
- harder debugging
- runtime overhead (virtual calls, allocations)

Interviewers want you to show:

> “I chose X because it reduces _this pain_, and I accept _this cost_.”

## 4.3.1 The 3 Pattern Costs (always mention at least one)

### Cost A — Indirection

More layers = harder to trace flow.
- Strategy/Decorator/Proxy add “one more hop”.

### Cost B — More objects + allocation

In C++ especially:
- virtual polymorphism often implies heap allocation (`unique_ptr`), vtable, etc.

### Cost C — Abstraction tax

Too many interfaces = slower development + confusion.

**Interview phrase:**

> “I’m keeping the abstraction only at the volatility boundary; everything else stays concrete.”


## 4.3.2 Pattern-by-pattern trade-off notes (interview-ready)

| Pattern                        | When to Use (Pain)                            | What You Gain                                  | Trade-offs (What You Accept)                 |
| ------------------------------ | --------------------------------------------- | ---------------------------------------------- | -------------------------------------------- |
| **Strategy**                   | New types/algorithms keep getting added       | OCP, clean polymorphism, no switch explosion   | More classes, indirection                    |
| **Factory / Abstract Factory** | Object creation logic is complex or varies    | Decouples creation, hides constructors         | Extra layer, harder to trace instantiation   |
| **Registry**                   | Plugins grow, avoid editing factory/switch    | Zero-touch extension, runtime registration     | Global state risk, registration order issues |
| **Decorator**                  | Add optional behaviors without modifying core | Composable features, avoids subclass explosion | Wrapper nesting, harder debugging            |
| **Proxy**                      | Control access, lazy load, remote call        | Access control, performance optimization       | Behavior less obvious, extra hop             |
| **Facade**                     | Clients see too much subsystem complexity     | Simple API, reduced coupling                   | Hides power/flexibility of subsystem         |
| **Composite**                  | Tree/part-whole structure                     | Uniform treatment of leaf & composite          | Harder to restrict operations on leaves      |
| **State**                      | Many state-dependent rules/transitions        | Localized state logic, explicit transitions    | More classes, transition complexity          |
| **Observer**                   | Many dependents on events                     | Loose coupling, scalable notifications         | Hard to debug flow, ordering issues          |
| **Command**                    | Need undo/queue/replay/log actions            | Actions as objects, flexible execution         | Object proliferation                         |
| **Chain of Responsibility**    | Pipeline of checks/handlers                   | Easy insertion/removal of steps                | Flow becomes implicit, harder tracing        |


## 4.3.3 Anti-patterns (what NOT to do)

### Anti-pattern 1: Pattern Soup

**Symptom:** every class is interface + factory + builder + strategy for no reason.
Why it’s bad:
- no one understands the system
- changes take longer, not shorter

**Interview defense:**

> “I’m adding only one abstraction at the axis of change. The rest remains concrete.”

---

### Anti-pattern 2: God Object / Manager Class

**Symptom:** `XManager`, `SystemController`, `AppService` with 30 methods.

Why it happens:
- fear of splitting responsibilities
- no clear ownership boundaries

Fix:
- apply SRP + CRC cards
- split by use-case or domain boundary

---

### Anti-pattern 3: Inheritance for Reuse (LSP violation magnet)

**Symptom:** “I’ll just make everything extend BaseService”.

Why it’s bad:
- breaks substitutability
- deep inheritance becomes fragile

Fix:
- composition + small interfaces
- Decorator for cross-cutting concerns

---

### Anti-pattern 4: Service Locator (hidden dependencies)

**Symptom:**
`auto& logger = ServiceLocator::get<ILogger>();`

Why bad:
- makes testing hard
- dependencies invisible
- global coupling

Fix: constructor injection (DIP + DI)

---

### Anti-pattern 5: Anemic Domain Model

**Symptom:** domain classes are just structs; all logic in services.

Why bad:
- invariants not enforced
- services become god objects

Fix:
- move state transitions to domain objects (`order.markPaid()`)

## 4.3.4 “Pattern Justification Script” (say this in interviews)

When you introduce a pattern, say:
1. **Pain:** “We expect new X types frequently.”
2. **Seam:** “So I isolate it behind an interface.”
3. **Choice:** “Strategy fits because behavior varies by type.”
4. **Trade-off:** “Adds indirection and classes, but reduces risk and improves tests.”
5. **YAGNI note:** “I’ll keep creation simple now; add a registry only if needed.”

This is an SDE-2/SDE-3 sounding explanation.


## 4.3.5 Practical Example: When patterns make it worse

### Scenario
Only 2 payment types, no config-based selection.
❌ Overengineering:
- `PaymentFactory`
- `PaymentRegistry`
- `AbstractPaymentFactory`
- `PaymentBuilder`

✅ Better:
- Keep a simple seam:
```cpp
IPaymentMethod
CardPayment, UpiPayment
```

- If selection is needed, keep one `switch` in a `PaymentDispatcher` (single place).

##  Interview Practice Questions

### Give an example where a pattern harmed readability and how you’d fix it.
We introduced Strategy and Factory for payment methods when we only had UPI and Cash. This added multiple classes and indirection without real benefit, making the flow harder to read. In this case, an enum + switch in one place was clearer. I’d start simple, keep the switch isolated, and refactor to Strategy only when new payment methods start getting added frequently

### How do you decide between enum+switch vs State pattern?
I use enum + switch when the number of states is small and state-dependent logic lives in one or two places. I move to the State pattern when behavior varies by state across multiple operations, transitions have rules or guards, and adding a new state requires touching many switches. State pattern localizes state rules and avoids scattered conditionals.

### How do you prevent Decorator “ordering bugs”?
Decorator ordering bugs happen when wrappers are applied in the wrong sequence. I prevent this by composing decorators in one place using a builder or factory that enforces the correct order. I also add integration tests on the final pipeline so incorrect ordering is caught early.

### Why is Service Locator considered an anti-pattern?
Service Locator hides dependencies, making it hard to understand what a class needs by reading its constructor. It also makes testing harder because tests must manipulate global state, and it creates tight global coupling. Constructor injection fixes this by making dependencies explicit and improving testability and reasoning.

### How do you avoid pattern soup while still being extensible?
I start with the simplest design and introduce patterns only when there’s a clear axis of change. I keep abstractions around volatile parts, not stable code. I also prefer one explicit seam instead of spreading patterns everywhere, and refactor incrementally when requirements actually grow.

# 4.4 Anti-Pattern Detection + Refactoring Playbook (Interview “Fix this design” mastery)

### Intuition

Many LLD rounds are secretly:
- “Here’s a messy design—can you spot what’s wrong?”
- “Can you refactor it to something extensible and testable?”

So you need a **repeatable playbook**:
1. Detect the anti-pattern quickly
2. Name the pain (change axis)
3. Apply the smallest refactor (seam + pattern)
4. Explain trade-offs

## 4.4.1 The 10 High-Signal Anti-Patterns (and what to do)
| #   | Anti-Pattern                    | Symptom (How you spot it)                                           | Why it’s bad                               | Correct Fix (What to say/do)                                                           |
| --- | ------------------------------- | ------------------------------------------------------------------- | ------------------------------------------ | -------------------------------------------------------------------------------------- |
| 1   | **God Object / God Service**    | One `*Service/*Manager` does everything; huge constructor & methods | Low cohesion, high coupling, hard to test  | Apply **SRP + CRC**: split into orchestration service, domain objects, policies, repos |
| 2   | **Anemic Domain Model**         | Domain = getters/setters; all logic in services                     | Invariants not enforced; rules scattered   | Move behavior into domain: `order.markPaid()`, `ticket.close()`                        |
| 3   | **Inheritance for Reuse**       | `BaseService/BaseController` with random shared logic               | Silent LSP violations; fragile hierarchies | Prefer **composition**; use **Decorator** for cross-cutting concerns                   |
| 4   | **Switch / If-Else Explosion**  | `switch(type)` grows weekly                                         | Violates OCP; core logic changes often     | Growth path: isolate switch → **Strategy** → **Factory/Registry**                      |
| 5   | **Boolean / Flag Arguments**    | `process(x, true, false)`                                           | Control coupling; unreadable; untestable   | Split methods or use **Command / Strategy**                                            |
| 6   | **Service Locator / Globals**   | `ServiceLocator::get()` everywhere                                  | Hidden deps; global coupling; poor tests   | **Constructor Injection** + composition root                                           |
| 7   | **Feature Envy**                | Class A pulls data from B and does B’s job                          | LoD violation; wrong responsibility        | Move behavior to data owner (“**Tell, don’t ask**”)                                    |
| 8   | **Train Wreck (LoD violation)** | `a.getB().getC().doX()`                                             | High coupling; brittle to change           | Add domain method at correct level or mapper                                           |
| 9   | **Tight Coupling to I/O**       | Domain imports DB/HTTP/SDK directly                                 | Business logic tied to infra               | Introduce **ports/interfaces** (`Repo`, `Gateway`, `Clock`)                            |
| 10  | **Temporal Coupling**           | “Must call A then B then C”                                         | Easy to misuse; hidden order dependency    | Enforce invariants in constructor or model with **State pattern**                      |

## 4.4.2 The Refactoring Playbook (Do this in interviews)

### Step 1: State the pain + axis of change

Example:

> “Payment types keep increasing, so switch-case will keep changing.”

### Step 2: Identify responsibilities (CRC)

- who should own rules?    
- who is orchestrating?
- who is a detail?

### Step 3: Introduce the smallest seam

- add an interface or extract collaborator
- avoid introducing 4 patterns at once

### Step 4: Choose minimal pattern

- types vary → Strategy
- cross-cutting wrapper → Decorator
- state-driven rules → State

### Step 5: Re-check principles

- SRP: each class has one reason to change
- DIP: policy depends on abstractions
- LoD: no deep chains
- Testability: can I unit test core logic with fake


## 4.4.3 Worked Mini Refactor (Most common interview scenario)
### Before (anti-pattern soup)
```cpp
class CheckoutService {
public:
    void checkout(Order& order, int paymentType) {
        // compute total
        // save to db
        // send invoice
        if (paymentType == 1) { /* card */ }
        else if (paymentType == 2) { /* upi */ }
        // logging, retries...
    }
};
```

### Diagnose (say this)
- God service (SRP violation)
- switch explosion (OCP violation)
- mixes policy + persistence + presentation (DIP violation)

### After (clean, minimal)
```rust
CheckoutService (orchestrator)
  -> PricingPolicy
  -> IOrderRepo
  -> IPaymentMethod (strategy)
  -> InvoiceSender
```

```cpp
class IPaymentMethod {
public:
    virtual ~IPaymentMethod() = default;
    virtual void pay(double amount) = 0;
};

class CheckoutService {
    PricingPolicy& pricing;
    IOrderRepo& repo;
public:
    CheckoutService(PricingPolicy& p, IOrderRepo& r) : pricing(p), repo(r) {}

    void checkout(Order& order, IPaymentMethod& method) {
        double amount = pricing.total(order);
        method.pay(amount);
        order.markPaid();
        repo.save(order);
    }
};
```

That’s enough. Don’t add registry unless asked.
