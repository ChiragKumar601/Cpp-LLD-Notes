# 7.1 Observer (Publish–Subscribe inside your process)

## Intuition
You have a **subject** whose state changes, and **many other parts** might want to react to that change (today or later). You don’t want the subject to hardcode calls to every dependent.

Observer makes this happen by letting others **subscribe**.
## Roles
- **Subject (Publisher):** owns the state, and a list of subscribers.
- **Observers (Subscribers):** implement a common callback interface like `onUpdate(...)`.

## Mental model: “Subject doesn’t know who cares”

The subject only knows: _“I have a list of observers; when I change, I notify them.”_  
It does **not** know their concrete types or business purpose.

## Flow
1. Observers **subscribe** to the subject
2. Subject state changes
3. Subject calls `observer->onUpdate(...)` for each subscriber
4. Observers react independently

## Real-life examples

- UI: button click → multiple listeners (analytics, UI update, sound)
- Trading: price update → strategy module, risk module, display module
- Domain: order status changed → email notifier, inventory updater, audit logger

## Why it’s useful
- **Loose coupling:** subject depends only on the observer interface
- **Extensible:** add new reactions without changing the subject
- **Runtime flexibility:** subscribers can be added/removed dynamically

### Common pitfalls (quick)

- Memory/lifetime issues (dangling observer pointers) → use unsubscribe or weak references
- Notification storms / ordering dependencies
- Threading: notify on the right thread if observers touch UI/shared state

## Concepts (Rules + C++ gotchas)
### Participants
- **Subject**: holds list of observers, notifies them. the subject holds direct references to observers and calls them.
- **Observer**: receives updates

### Key design choices interviewers care about

#### Push vs Pull
- Push: subject sends data in event (`onEvent(data)`)
- Pull: observer queries subject (`onEvent()` then `subject.getState()`)
**Sync vs Async**
- Sync: notify immediately (simple)
- Async: queue events (scales better, avoids long subject blocking)
**Lifetime management (C++ critical)**    
- Avoid dangling pointers when observers are destroyed
- Prefer **subscription tokens** or **weak_ptr** approach

---

## Architecture View
```
         +-----------+
         |  Subject  |
         +-----------+
           |   |   |
           v   v   v
        Obs1 Obs2 Obs3

```

---

## Practical Example (C++ with safe unsubscribe token)

### Observer interface
```cpp
class IObserver {
public:
    virtual ~IObserver() = default;
    virtual void onEvent(int value) = 0;
};
```

### Subject with subscription token (RAII)

```cpp
class Subject {
    std::mutex m;
    std::unordered_map<int, IObserver*> obs;
    int nextId = 1;

public:
    class Subscription {
        Subject* subject;
        int id;
    public:
        Subscription(Subject* s, int i) : subject(s), id(i) {}
        Subscription(Subscription&& o) noexcept : subject(o.subject), id(o.id) {
            o.subject = nullptr;
        }
        Subscription& operator=(Subscription&&) = delete;
        ~Subscription() { if (subject) subject->unsubscribe(id); }
    };

    Subscription subscribe(IObserver& o) {
        std::lock_guard<std::mutex> lock(m);
        int id = nextId++;
        obs[id] = &o;
        return Subscription(this, id);
    }

    void unsubscribe(int id) {
        std::lock_guard<std::mutex> lock(m);
        obs.erase(id);
    }

    void notify(int value) {
        std::vector<IObserver*> snapshot;
        {
            std::lock_guard<std::mutex> lock(m);
            for (auto& [_, p] : obs) snapshot.push_back(p);
        }
        for (auto* o : snapshot) o->onEvent(value);
    }
};
```

This solves the typical lifecycle problem:
- observer unsubscribes automatically when `Subscription` is destroyed.


## Real-World Use Cases
- UI updates (MVC-ish)
- domain events (OrderPaid triggers invoice, email, metrics)
- cache invalidation listeners
- in-memory pub-sub in a service


## Interview Questions (Observer)
**Q. Observer vs Pub-Sub (distributed): difference?**
**Observer** is in-process: the subject holds direct references to observers and calls them.  
**Distributed Pub-Sub** uses a broker (Kafka/RabbitMQ/etc.): publishers and subscribers are decoupled in time/process, with delivery/ordering/retries handled by messaging infrastructure.

**Q. How do you prevent memory leaks / dangling pointers in C++?**
Use **RAII subscriptions** (token that auto-unsubscribes) or store observers as **`weak_ptr`** in the subject; avoid raw owning pointers, and break shared_ptr cycles with `weak_ptr`.

**Q. Push vs pull model—when do you choose which?**
Choose **push** when the event payload is small/clear and observers shouldn’t depend on the subject API; choose **pull** when state is large or observers need different slices, so they query the subject for what they need.

**Q. What if one observer is slow? (async, queue, timeouts, decouple)**
Don’t let the subject block: notify **asynchronously** (queue/executor), add **timeouts/backpressure**, and isolate failures so one slow observer can’t delay others.


---

# 7.2 State (Replace state-dependent if/else with state objects)

## Intuition

When behavior depends on internal state:
- the same event does different things in different states
- many `if (state == X)` checks appear

State pattern makes each state responsible for behavior, so adding states is OCP-friendly.

**Mental model:** “Move state-specific behavior into state classes.”

---

## Concepts (Rules)
- `Context` holds current `State`
- `State` interface defines actions/events
- Concrete states implement behavior and transitions

### When NOT to use
- only 2–3 states with simple logic → enum + switch in one place is fine (YAGNI)


## Architecture View
```
Context ----> IState
   |           /   \
   |        Idle  HasCredit  Dispensing
   +-- transitions change current state
```


## Practical Example (Vending machine mini)

### State interface
```cpp
class VendingMachine;

class IState {
public:
    virtual ~IState() = default;
    virtual void insertCoin(VendingMachine& m, int amount) = 0;
    virtual void selectItem(VendingMachine& m, int price) = 0;
};
```

### Context
```cpp
class VendingMachine {
    int credit = 0;
    std::unique_ptr<IState> state;

public:
    VendingMachine(std::unique_ptr<IState> s) : state(std::move(s)) {}

    void setState(std::unique_ptr<IState> s) { state = std::move(s); }
    int& getCredit() { return credit; }

    void insertCoin(int amt)  { state->insertCoin(*this, amt); }
    void selectItem(int price){ state->selectItem(*this, price); }
};
```

### Concrete states (2 states shown)
```cpp
class IdleState : public IState {
public:
    void insertCoin(VendingMachine& m, int amount) override;
    void selectItem(VendingMachine&, int) override {
        throw std::runtime_error("Insert coin first");
    }
};

class HasCreditState : public IState {
public:
    void insertCoin(VendingMachine& m, int amount) override;
    void selectItem(VendingMachine& m, int price) override;
};

```

Definitions:
```cpp
void IdleState::insertCoin(VendingMachine& m, int amount) {
    m.getCredit() += amount;
    m.setState(std::make_unique<HasCreditState>());
}

void HasCreditState::insertCoin(VendingMachine& m, int amount) {
    m.getCredit() += amount;
}

void HasCreditState::selectItem(VendingMachine& m, int price) {
    if (m.getCredit() < price) throw std::runtime_error("Insufficient credit");
    m.getCredit() -= price;
    m.setState(std::make_unique<IdleState>());
}
```



## Real-World Use Cases
- order lifecycle / payment lifecycle
- elevator door states
- retry workflows
- game character states
- connection states (connected/disconnected/reconnecting)


## Interview Questions (State)

**Q. State vs Strategy: difference?**
**State** lets an object change behavior based on **internal state and transitions**; **Strategy** swaps an algorithm chosen **externally** (config/client) and doesn’t model transitions.
 
**Q. When is enum+switch better than State pattern?**
Use **enum + switch** when states and transitions are **few, stable, and simple**, and actions are small; switch to **State pattern** when transitions/behaviors grow, you see frequent changes, or the switch becomes a sprawling conditional with duplicated logic.
 
**Q. Where do transitions live—inside states or in context?**
- **State decides transitions** = “the current mode knows what’s next.”
- **Context decides transitions** = “a central rulebook decides what’s next.”

Transitions inside the **State** (each state decides)
**Analogy: A traffic light controller inside each light mode**
- When the light is **Green**, it “knows” the next is **Yellow** after a timer.
- When it’s **Yellow**, it “knows” the next is **Red**.
- Each mode carries its own rule: “what happens next from here.”

Transitions inside the **Context** (central transition table)
**Analogy: A metro/train control room with a rulebook**
- The train itself doesn’t decide where to go next.
- A central control room has a big table:
    - “If train is at Station A and signal is green → go to Station B”
    - “If signal is red → stop”
- The control room decides transitions based on global rules, scheduling, priorities, safety.

 
**Q. How do you test state machines cleanly?**
To test a state machine cleanly, I write **table-driven tests** that drive the public events and assert **(next state + observable outputs/side-effects)** for each `(startState, event)` pair. I keep transitions deterministic, inject fakes/mocks for external effects, and cover edge cases like invalid transitions and errors—so the tests validate behavior without peeking into internal implementation details.

# 7.3 Strategy (Swap algorithms/behavior without changing callers)

### Intuition
When you have **one stable workflow**, but **one step varies** by choice/type, use Strategy.
**Mental model:** “Same input/output, different algorithm.”
Typical interview triggers:
- multiple pricing algorithms
- different routing/selection policies
- multiple sort/filter strategies
- different cache eviction policies

## Concepts
- Context uses a `IStrategy`
- Strategies are interchangeable
- Strategy object usually has **no state** (or only config)


## Architecture View

```
Context ---> IStrategy
              /   \
        StrategyA  StrategyB
```


### Practical Example (Discount strategy)

```cpp
class IDiscountStrategy {
public:
    virtual ~IDiscountStrategy() = default;
    virtual double apply(double subtotal) const = 0;
};

class NoDiscount : public IDiscountStrategy {
public:
    double apply(double subtotal) const override { return subtotal; }
};

class Flat10Percent : public IDiscountStrategy {
public:
    double apply(double subtotal) const override { return subtotal * 0.9; }
};

class PricingEngine {
    const IDiscountStrategy& discount; // non-owning, injected
public:
    explicit PricingEngine(const IDiscountStrategy& d) : discount(d) {}

    double total(double subtotal) const {
        return discount.apply(subtotal);
    }
};
```

---

## Real-World Use Cases
- Payment routing policy (cheapest gateway, failover)
- Spot allocation policy (nearest, random, compact)
- Cache eviction (LRU/LFU/FIFO)
- Load balancing algorithms

---

### Interview Questions (Strategy)

Q. How do you choose ownership of strategy in C++? (ref vs unique_ptr)

Q. When would you use registry/factory with Strategy?
Use a **factory/registry** when the strategy must be selected **at runtime** based on config/input/tenant/feature flag, so adding a new strategy becomes “register it” rather than editing a giant `switch`.



---

# 7.4 Command (Encapsulate a request as an object)

## Intuition
Use Command when you need:
- queue requests
- retry later
- undo/redo
- log/replay
- decouple invoker from receiver

**Mental model:** “Turn a method call into a first-class object.”

## Concepts
- **Command** interface: `execute()` (and optional `undo()`)
- **Invoker** triggers commands (button, queue worker)
- **Receiver** does the actual work

## Architecture View

```
Invoker -> ICommand -> Receiver
```


## Practical Example (Queueable commands)
```cpp
class ICommand {
public:
    virtual ~ICommand() = default;
    virtual void execute() = 0;
};

class Light {
public:
    void on() { /*...*/ }
    void off(){ /*...*/ }
};

class LightOnCommand : public ICommand {
    Light& light;
public:
    explicit LightOnCommand(Light& l) : light(l) {}
    void execute() override { light.on(); }
};

class CommandQueue {
    std::queue<std::unique_ptr<ICommand>> q;
public:
    void push(std::unique_ptr<ICommand> c) { q.push(std::move(c)); }
    void runOne() {
        auto c = std::move(q.front()); q.pop();
        c->execute();
    }
};
```

Now you can enqueue work, schedule it, retry it.


## Real-World Use Cases
- Job scheduling systems
- Undo/redo in editors
- Payment retry queue
- Batch processing pipelines
- Macro recording (replay actions)


## Interview Questions (Command)

Q. Command vs Strategy: difference?
Strategy selects algorithm for a step; Command represents an action/request.
  
Q. How do you add undo? What state must be stored?
 Add `undo()` to the command and store the **minimum information to reverse the effect**—typically the **before state** (previous value/snapshot) or the **inverse operation data**.
 
Q. How do you persist commands? (serialize intent/params)
 Persist the **command type + version + parameters + idempotency/correlation id + timestamp** (not pointers), so you can safely **replay/dispatch** it later.



# 7.5 Chain of Responsibility (Pipeline of handlers/checks)

## Intuition

Use it when you have a **sequence of checks/handlers**, and you want to add/remove/reorder them without modifying a big function.

Triggers:
- validation pipeline
- auth filters / middleware
- fraud detection rules
- request processing pipeline

**Mental model:** “Pass request along a chain until handled or rejected.”


## Concepts
- Each handler either:
    - handles and stops
    - passes to next
- You can implement as:
    - linked chain (classic)
    - vector pipeline (cleaner in modern code)


## Architecture View
```
Req -> H1 -> H2 -> H3 -> ... -> Hn
```


## Practical Example (Validation pipeline — vector style, recommended)

```cpp
struct Request { int userId; double amount; };

class IHandler {
public:
    virtual ~IHandler() = default;
    virtual void handle(const Request& r) = 0; // throw on failure
};

class AuthHandler : public IHandler {
public:
    void handle(const Request& r) override {
        if (r.userId <= 0) throw std::runtime_error("Unauthenticated");
    }
};

class LimitHandler : public IHandler {
public:
    void handle(const Request& r) override {
        if (r.amount > 10000) throw std::runtime_error("Limit exceeded");
    }
};

class Pipeline {
    std::vector<std::unique_ptr<IHandler>> handlers;
public:
    void add(std::unique_ptr<IHandler> h) { handlers.push_back(std::move(h)); }

    void run(const Request& r) {
        for (auto& h : handlers) h->handle(r);
    }
};
```

---

## Real-World Use Cases
- API middleware stack
- Payment fraud rule chain
- Request validation and enrichment
- Logging/auth/metrics filters in gateways


## Interview Questions (Chain of Responsibility)

**Q. CoR vs Decorator: difference?**    
**Decorator** wraps a component to add behavior while keeping the same interface; **Chain of Responsibility** routes a request through multiple handlers until one handles it (or the chain ends).

**Q. When do you stop the chain? (first handler that handles / on failure / always continue)**
You stop based on the contract: **stop at the first handler that handles the request**, or **run all handlers** for pipeline-style processing; for errors, either **fail-fast** or **continue with error collection** depending on use case.

Examples:
- Auth chain → stop on first definitive deny/allow
- Logging/metrics pipeline → always continue
- Validation → often collect all errors
 
Q. How do you make the chain configurable? (register handlers by config)
Build the chain in the **composition root** using configuration: register handlers (map key → factory), select/order them from config, and link them at startup.

---

## Quick “When to pick which” (7.3–7.5)

- **Strategy**: choose _one algorithm_ among many
- **Command**: represent _an action_ (queue/undo/retry)
- **CoR**: run _a sequence of checks/handlers_

# 7.6 Template Method (Fixed algorithm skeleton, customizable steps)
## Intuition
Use Template Method when you have:
- a **common workflow**
- with some steps that vary
- and you want to enforce the order of steps

**Mental model:** “Parent defines the recipe; subclasses customize steps.”
This is often used in frameworks and pipelines.


## Concepts
- Base class defines `run()` (the skeleton)
- Calls `step1()`, `step2()`... (hooks)
- Derived classes override hooks

**Important trade-off**
- It uses inheritance (can be rigid)
- Strategy is often more flexible (composition)  
    Use Template Method when the sequence itself must be enforced and is stable.


## Architecture View
```cpp
BaseWorkflow::run()
  -> stepA()
  -> stepB()  (override points)
  -> stepC()
```


### Practical Example (C++)
```cpp
class DataPipeline {
public:
    virtual ~DataPipeline() = default;

    void run() {                 // template method (fixed order)
        read();
        transform();
        write();
    }

protected:
    virtual void read() = 0;
    virtual void transform() = 0;
    virtual void write() = 0;
};

class CsvToDbPipeline : public DataPipeline {
protected:
    void read() override { /* read CSV */ }
    void transform() override { /* map fields */ }
    void write() override { /* insert DB */ }
};
```


## Real-World Use Cases
- ETL pipelines (read/transform/write)
- Authentication flow templates
- Test frameworks (setup → run → teardown)
- Network protocols handshake skeletons


## Interview Questions (Template Method)

Q. Template Method vs Strategy?    
Template Method varies steps via inheritance; Strategy varies behavior via composition.

Q. When is Template Method a bad idea? (too rigid, subclass explosion)
Q. How do you avoid fragile base class? (keep hooks minimal, document contracts)



---

# 7.7 Mediator (Reduce many-to-many communication)

## Intuition
When many components talk to each other directly, coupling explodes:
- each component knows too much
- changes ripple everywhere

Mediator centralizes interactions:

> Components communicate through a mediator, not directly.

**Mental model:** “Air traffic control.”


## Concepts
- Colleagues: components that interact
- Mediator: orchestrates interactions


## Architecture View

Before (tight coupling):

```
A <-> B
A <-> C
B <-> C
... (mess)
```

After:

```
A --> Mediator <-- B
C --> Mediator
```


## Practical Example (Chat room)

```cpp
class IUser {
public:
    virtual ~IUser() = default;
    virtual void receive(const std::string& msg) = 0;
};

class ChatMediator {
    std::vector<IUser*> users; // non-owning; mediator doesn't own users
public:
    void join(IUser& u) { users.push_back(&u); }

    void broadcast(const std::string& msg) {
        for (auto* u : users) u->receive(msg);
    }
};

class User : public IUser {
    std::string name;
public:
    explicit User(std::string n) : name(std::move(n)) {}
    void receive(const std::string& msg) override {
        // handle incoming
    }
};
```


## Real-World Use Cases
- UI form controls (checkbox changes enable/disable other widgets)
- Game systems (components interacting via event hub)
- Trading systems (order book mediator between buyers/sellers)
- Microservice gateway orchestrating calls (conceptually similar)

---

### Interview Questions (Mediator)

**Q. Mediator vs Facade?**
Facade simplifies subsystem for external clients; Mediator manages internal interactions between peers.

**Q. When does Mediator become a god object? How to prevent? (split mediators by domain)**

**Q. Mediator vs Observer? (Observer is broadcast eventing; Mediator is directed coordination)**



# 7.8 Iterator (Traverse a collection without exposing internals)

## Intuition
Iterator lets you traverse elements without knowing the underlying structure:
- vector vs tree vs custom graph
- keeps encapsulation
- provides uniform access

In interviews, it shows up with:
- Composite structures (tree traversal)
- custom containers
- streaming

## Concepts
- Iterator represents a traversal cursor
- C++ already has iterators; pattern is built-in
- In OO interviews, focus on “hide representation”


## Architecture View
```
Client -> Iterator -> Collection internals hidden
```

---

### Practical Example (C++ already supports)

For custom container, provide `begin()`/`end()`.
```cpp
class IntBag {
    std::vector<int> data;
public:
    void add(int x) { data.push_back(x); }

    auto begin() { return data.begin(); }
    auto end()   { return data.end(); }

    auto begin() const { return data.begin(); }
    auto end()   const { return data.end(); }
};
```

Usage:
```cpp
IntBag bag;
bag.add(1); bag.add(2);
for (int x : bag) { /*...*/ }
```



## Iterator with Composite (tree)

Once you have Composite (`Node` with children), you can provide DFS/BFS iterators (advanced; mention conceptually).


## Real-World Use Cases
- Traversing large trees (filesystem, DOM)
- Paginated iteration over DB results (cursor iterator)
- Streaming processing pipelines


## Interview Questions (Iterator)
Q. Why is iterator useful when the underlying structure changes?
Iterator gives a **stable traversal API** so client code doesn’t depend on whether data is stored as `vector`, `list`, tree, etc.—you can change internals without rewriting traversal logic.

Q. Iterator vs exposing `vector` directly: what’s the benefit?
 Iterators preserve **encapsulation and invariants** while still allowing traversal, and let you swap the underlying container or provide lazy/filtered traversal without exposing internal representation.
 
Q. How would you implement DFS iterator for a tree?
 
Q. Internal vs external iterators (push vs pull traversal)?



## Quick “When to use which” (7.6–7.8)
- **Template Method**: fixed workflow order, variable steps
- **Mediator**: too much peer-to-peer coupling
- **Iterator**: traverse without exposing representation

```cpp
struct Node { std::vector<Node*> children; int val; };

class DfsIterator {
    std::vector<Node*> st;
public:
    explicit DfsIterator(Node* root) { if (root) st.push_back(root); }

    bool hasNext() const { return !st.empty(); }

    Node* next() {
        Node* cur = st.back(); st.pop_back();
        // push children reverse so first child is visited first
        for (int i = (int)cur->children.size()-1; i >= 0; --i)
            st.push_back(cur->children[i]);
        return cur;
    }
};
```

# 7.9 Memento (Capture & restore state without exposing internals)

## Intuition
Use Memento when you need:
- **undo/redo**
- **rollback**
- **checkpoints**
- **restore previous state**

…while keeping encapsulation intact.

**Mental model:** “Take a snapshot you can later restore.”

## Concepts
- **Originator**: object whose state you want to save/restore
- **Memento**: snapshot of state (often immutable)
- **Caretaker**: stores mementos (history stack)

Key rule:
> Caretaker should not modify memento internals.


## Architecture View
```
Caretaker <---- stores ---- Memento
     ^
     |
Originator --- creates/restores ---> Memento
```

---

### Practical Example (C++: Text editor undo)

```cpp
class TextEditor {
public:
    class Memento {
        friend class TextEditor;
        std::string state;
        explicit Memento(std::string s) : state(std::move(s)) {}
    public:
        // no public setters: immutable snapshot
    };

    void setText(std::string t) { text = std::move(t); }
    const std::string& getText() const { return text; }

    Memento save() const { return Memento(text); }
    void restore(const Memento& m) { text = m.state; }

private:
    std::string text;
};

class History {
    std::vector<TextEditor::Memento> stack;
public:
    void push(TextEditor::Memento m) { stack.push_back(std::move(m)); }
    bool empty() const { return stack.empty(); }
    TextEditor::Memento pop() {
        auto m = std::move(stack.back());
        stack.pop_back();
        return m;
    }
};
```

Usage:
```cpp
TextEditor ed;
History h;

ed.setText("v1");
h.push(ed.save());

ed.setText("v2");
ed.restore(h.pop()); // back to v1
```


## Real-World Use Cases
- Undo/redo in editors
- Rollback config changes
- Game save checkpoints
- Transaction-like local rollback (not DB transactions)


### Interview Questions (Memento)

**Q. Memento vs Command for undo/redo: when choose which?**
Memento: snapshot-based undo
Command: operation-based undo (`execute/undo`)

**Q. How do you avoid huge memory usage? (diffs, compress, cap history)**
Q. How do you keep encapsulation? (memento hidden/immutable)


---


# 7.10 Visitor (Add new operations to a structure without modifying its classes)

## Intuition
Use Visitor when:
- you have a **stable object structure** (AST, file tree, expression tree)
- you frequently add **new operations** over that structure (print, evaluate, serialize, lint)
- you want to avoid adding methods to every node for every new operation

**Mental model:** “Add behaviors externally, per node type.”


## Concepts
- Each element has `accept(visitor)`
- Visitor has overloaded `visit(Type&)` for each concrete element
- Adding a new **operation** = add a new Visitor class (OCP for operations)
- Trade-off: adding a new **node type** requires updating all visitors (OCP not for structure)


## Architecture View
```scss
ElementA --accept--> Visitor.visit(ElementA)
ElementB --accept--> Visitor.visit(ElementB)
```


### Practical Example (C++: Expression AST)

Elements:
```cpp
class Visitor;

class Expr {
public:
    virtual ~Expr() = default;
    virtual void accept(Visitor& v) = 0;
};

class Number : public Expr {
public:
    double value;
    explicit Number(double v) : value(v) {}
    void accept(Visitor& v) override;
};

class Add : public Expr {
public:
    std::unique_ptr<Expr> left, right;
    Add(std::unique_ptr<Expr> l, std::unique_ptr<Expr> r)
        : left(std::move(l)), right(std::move(r)) {}
    void accept(Visitor& v) override;
};
```

Visitor:
```cpp
class Visitor {
public:
    virtual ~Visitor() = default;
    virtual void visit(Number& n) = 0;
    virtual void visit(Add& a) = 0;
};

void Number::accept(Visitor& v) { v.visit(*this); }
void Add::accept(Visitor& v)    { v.visit(*this); }
```

Concrete visitor (evaluate):
```cpp
class EvalVisitor : public Visitor {
public:
    double result = 0;

    void visit(Number& n) override { result = n.value; }

    void visit(Add& a) override {
        EvalVisitor lv; a.left->accept(lv);
        EvalVisitor rv; a.right->accept(rv);
        result = lv.result + rv.result;
    }
};
```

Usage:
```cpp
auto expr = std::make_unique<Add>(
    std::make_unique<Number>(2),
    std::make_unique<Number>(3)
);

EvalVisitor ev;
expr->accept(ev); // ev.result == 5
```


## Real-World Use Cases

- Compilers/interpreters (AST visitors)
- File system operations (size, print, permissions check)
- Serialization/deserialization for heterogeneous object graphs
- Analytics/reporting over a domain tree


### Visitor trade-offs (must say in interview)

✅ Pros:
- Add new operations without modifying element classes
- Keeps operation logic centralized per visitor

⚠️ Cons:
- Adding new element type forces edits in every visitor
- Can feel heavy if structure is small or changing often


## Interview Questions (Visitor)

Q. When is Visitor a good fit? (stable structure, many operations)

**Q. Visitor vs adding virtual methods on base class?**

**Q. What breaks if we add a new node type?**

**Q. How does Visitor relate to double dispatch?**